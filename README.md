# Kubernetes Storage Deep Dive — PV, PVC, StorageClass & StatefulSets

This repository is a **hands-on demonstration of Kubernetes storage internals**, built and tested on a local multi-node Kind cluster.

The goal of this project is not just to *use* storage, but to **understand how Kubernetes actually provisions, binds, and persists data** across Pods — both in **static and dynamic provisioning scenarios**.

---

# What This Project Proves

By running this demo, I validated:

* How **PersistentVolumes (PV)** represent actual storage
* How **PersistentVolumeClaims (PVC)** request and bind storage
* How **StorageClass enables dynamic provisioning**
* How **StatefulSets provide stable storage per Pod**
* How **data persists across Pod restarts and scaling events**
* The **exact matching logic Kubernetes uses** for PV ↔ PVC binding

---

# Environment

* Local Kubernetes cluster using **Kind (multi-node)**
* Dynamic provisioning enabled via local-path provisioner
* Workloads deployed inside `storage-demo` namespace

---

# Project Structure

```id="c7kl6t"
sf-demo/
│
├── pv.yaml              # Static PV (manual storage definition)
├── pvc.yaml             # PVC for static binding
├── service.yaml         # Headless Service (required for StatefulSet)
├── statefulset.yaml     # StatefulSet with volumeClaimTemplates
└── screenshots/         # Execution proof (kubectl outputs)
```

---

# Core Concepts Demonstrated

## 1. PV vs PVC vs StorageClass

| Component    | Responsibility                                       |
| ------------ | ---------------------------------------------------- |
| PV           | Represents actual storage (disk, path, cloud volume) |
| PVC          | Requests storage with specific requirements          |
| StorageClass | Defines *how* storage should be created dynamically  |

---

## 2. Dynamic Provisioning (StatefulSet Case)

In the StatefulSet:

```id="ozu9dq"
volumeClaimTemplates → PVCs auto-created → PVs auto-provisioned
```

Flow:

```id="k8xq0h"
Pod → PVC → StorageClass → PV → Actual Storage
```

### Observations

* Each Pod (`web-0`, `web-1`, `web-2`) gets:

  * A unique PVC
  * A unique PV
* Storage is **automatically created** using StorageClass
* No manual PV creation required

---

## 3. Data Persistence Validation

Steps performed:

* Wrote different data into each Pod
* Deleted Pods manually
* Allowed StatefulSet to recreate them

### Result

* Data remained intact
* PVC remained bound to same PV
* Pod reattached to same storage

 Confirms **true persistence beyond Pod lifecycle**

---

## 4. Static Provisioning (Manual PV Binding)

Created:

* A manual **PersistentVolume (pv.yaml)**
* A matching **PersistentVolumeClaim (pvc.yaml)**

### Key Learning

Kubernetes does **not create storage** in this case.

Instead:

```id="y1q1sx"
PV (existing) → PVC → bind → Pod uses it
```

---

## 5. Critical Real-World Gotcha

Initially, PVC was stuck in `Pending`.

### Root Cause

* PVC was automatically assigned default StorageClass (`standard`)
* PV had no StorageClass

 Result: **Mismatch → No binding**

---

### Fix Applied

```yaml id="5cdsdr"
storageClassName: ""
```

### Insight

| Case              | Behavior                        |
| ----------------- | ------------------------------- |
| Not specified     | Default StorageClass is applied |
| "" (empty string) | Disables dynamic provisioning   |
| Specific name     | Matches that StorageClass       |

---

## 6. PV ↔ PVC Matching Logic (Deep Understanding)

Kubernetes binds PV and PVC based on:

1. StorageClass match
2. AccessModes compatibility
3. Capacity (PV ≥ PVC)
4. Volume availability
5. Optional label selectors

### Selection Strategy

If multiple PVs match:

```id="x8aqe2"
Kubernetes selects the smallest suitable PV
```

---

## 7. Where Data is Actually Stored

### In Kind (Local Setup)

* Using `hostPath`
* Data stored at:

```id="l2t2h9"
/mnt/data (inside Kind node container)
```

### In Cloud (Conceptual - EKS)

If using AWS EBS:

```id="4n0o6g"
PVC → StorageClass → EBS volume → attached to EC2 → mounted into Pod
```

 No filesystem path is exposed to user

---

# Key Experiments Performed

* ✔ Verified automatic PV creation via StorageClass
* ✔ Observed one-to-one mapping: Pod → PVC → PV
* ✔ Tested persistence across Pod deletion
* ✔ Scaled StatefulSet and observed new volume creation
* ✔ Tested static PV binding and debugged mismatch issue
* ✔ Verified actual storage location inside node

---

# Important Behavioral Insights

* PV is **cluster-scoped**, PVC is **namespace-scoped**
* StatefulSet **does not delete PVCs** automatically
* `hostPath` is node-specific (not production-safe)
* Dynamic provisioning depends entirely on StorageClass
* Kubernetes **does not manage storage directly** — it delegates to provisioners (CSI drivers)

---

# Final Mental Model

```id="q8l0o0"
PVC = request
PV  = supply
StorageClass = provisioner logic
Pod = consumer
```

---

# Conclusion

This project demonstrates a **complete lifecycle of Kubernetes storage**:

* From **manual storage allocation (static PV)**
* To **automated provisioning (StorageClass)**
* To **real-world usage with StatefulSets**

The hands-on experiments validate not just usage, but **internal decision-making and behavior of Kubernetes storage system**.

---

# Screenshots

All execution proofs (kubectl outputs, bindings, errors, fixes) are available in:

```id="bflvmt"
screenshots/
```

---

# Author

Kiran Kumatkar
