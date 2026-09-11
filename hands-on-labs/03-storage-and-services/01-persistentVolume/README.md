# 💾 Kubernetes PersistentVolume — Deep Dive

> **Lab Series:** Storage & Services → 01 PersistentVolume  
> **Difficulty:** Beginner → Intermediate  
> **Estimated Time:** 30–45 minutes  
> **Focus:** Understand what PersistentVolumes are, how the PV/PVC binding model works, and when to use them

---

## 📌 Table of Contents

1. [Why PersistentVolumes Exist](#why-persistentvolumes-exist)
2. [The PV/PVC Model — How It Works](#the-pvpvc-model--how-it-works)
3. [PersistentVolume Manifest — Field by Field Breakdown](#persistentvolume-manifest--field-by-field-breakdown)
4. [Hands-On Lab](#hands-on-lab)
5. [Access Modes — Know the Difference](#access-modes--know-the-difference)
6. [Reclaim Policies — What Happens After Release](#reclaim-policies--what-happens-after-release)
7. [PersistentVolumes in Production — What Changes](#persistentvolumes-in-production--what-changes)
8. [Interview Q&A — Straight to the Point](#interview-qa--straight-to-the-point)
9. [Common Mistakes & Gotchas](#common-mistakes--gotchas)

---

## Why PersistentVolumes Exist

Containers are ephemeral by design — when a pod dies, everything written to its filesystem is gone. That's fine for stateless apps, but it's a problem for databases, file uploads, logs, or any workload that needs data to survive pod restarts.

Kubernetes solves this with **PersistentVolumes (PV)** — a piece of storage in the cluster that exists independently of any pod.

| Scenario | Without PV | With PV |
|---|---|---|
| Pod restarts | Data is lost ❌ | Data survives ✅ |
| Pod is rescheduled to another node | Data is lost ❌ | Data survives (depends on type) ✅ |
| Multiple pods need shared storage | Not possible ❌ | Possible with correct access mode ✅ |
| Storage managed by cluster admin | No separation of concerns ❌ | Admin provisions PV, dev claims it ✅ |

> 💡 **The Rule:** Any workload that needs data to outlive a pod needs a PersistentVolume. Classic examples: MySQL, PostgreSQL, Redis (with persistence), file upload services, and CI/CD artifact stores.

---

## The PV/PVC Model — How It Works

Kubernetes separates storage into two objects:

- **PersistentVolume (PV)** — the actual storage resource, provisioned by a cluster admin (or dynamically by a StorageClass)
- **PersistentVolumeClaim (PVC)** — a request for storage by a developer/pod. It says "I need 1Gi with ReadWriteOnce access"

```
┌─────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                       │
│                                                             │
│  Admin creates:          Developer creates:                 │
│  ┌─────────────┐         ┌─────────────┐                    │
│  │     PV      │◄────────│     PVC     │◄──── Pod uses PVC  │
│  │  (storage)  │  bound  │  (request)  │                    │
│  └─────────────┘         └─────────────┘                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

The binding process:
1. Admin creates a PV (defines capacity, access mode, storage backend)
2. Developer creates a PVC (requests capacity and access mode)
3. Kubernetes finds a matching PV and **binds** them together
4. Pod references the PVC — it never talks to the PV directly

```
kubectl apply -f pv.yml
        │
        ▼
┌───────────────┐
│  API Server   │  ← Stores PV in etcd, status: Available
└───────┬───────┘
        │
        ▼  (PVC is created)
┌────────────────────┐
│  PV Controller     │  ← Matches PVC to a compatible PV
│                    │    Binds them together
└───────┬────────────┘
        │
        ▼
┌────────────────────┐
│  Pod               │  ← Mounts the PVC as a volume
│  (references PVC)  │    Reads/writes to persistent storage
└────────────────────┘
```

> 💡 This separation of concerns is intentional. Admins manage the storage infrastructure (PVs). Developers just claim what they need (PVCs). The pod doesn't care where the storage comes from.

---

## PersistentVolume Manifest — Field by Field Breakdown

This is the `01-persistentVolume.yml` used in this lab:

```yaml
kind: PersistentVolume      # Resource type
apiVersion: v1              # PVs are in the core v1 API group
metadata:
  name: local-pv            # Name of the PV object
  labels:
    app: local              # Optional labels for selection

spec:
  capacity:
    storage: 1Gi            # How much storage this PV provides

  accessModes:
    - ReadWriteOnce         # Can be mounted read-write by one node at a time

  persistentVolumeReclaimPolicy: Retain   # What happens to the PV after the PVC is deleted

  storageClassName: local-storage         # Logical grouping — PVC must request this class

  hostPath:
    path: /mnt/data         # Actual path on the host node (for local/dev use only)
```

### Every Field Explained

| Field | Required | Purpose |
|---|---|---|
| `kind` | ✅ | Resource type |
| `apiVersion: v1` | ✅ | PVs are in the core API group, not `apps/v1` |
| `metadata.name` | ✅ | Unique name for the PV in the cluster (PVs are cluster-scoped, not namespaced) |
| `metadata.labels` | ❌ | Optional — can be used by PVCs with `matchLabels` for precise binding |
| `spec.capacity.storage` | ✅ | Size of the volume (e.g., `1Gi`, `500Mi`, `10Gi`) |
| `spec.accessModes` | ✅ | How the volume can be mounted — see [Access Modes](#access-modes--know-the-difference) |
| `spec.persistentVolumeReclaimPolicy` | ❌ | Defaults to `Retain` for manually provisioned PVs — see [Reclaim Policies](#reclaim-policies--what-happens-after-release) |
| `spec.storageClassName` | ❌ | Logical class name — PVC must request the same class to bind |
| `spec.hostPath.path` | ✅ (for hostPath) | Directory on the host node — **only for local dev/testing, never production** |

> ⚠️ **Critical:** PersistentVolumes are **cluster-scoped** — they don't belong to a namespace. PersistentVolumeClaims are **namespace-scoped**. A PVC in namespace `dev` can bind to a cluster-level PV.

> ⚠️ **hostPath Warning:** `hostPath` volumes tie your data to a specific node. If the pod is rescheduled to a different node, it won't find the data. Use only for local development or single-node clusters.

---

## Hands-On Lab

### Prerequisites
- A running Kubernetes cluster (Minikube, Kind, or any single-node cluster for `hostPath`)
- `kubectl` configured (`kubectl cluster-info` to verify)
- The directory `/mnt/data` exists on your node (or Minikube VM)

---

### Step 1 — Prepare the Host Directory

For `hostPath`, the directory must exist on the node:

```bash
# If using Minikube, SSH into the node first
minikube ssh

# Create the directory
sudo mkdir -p /mnt/data
echo "Hello from PersistentVolume" | sudo tee /mnt/data/index.html

# Exit minikube ssh
exit
```
<img width="329" height="99" alt="image" src="https://github.com/user-attachments/assets/ad25729e-e338-4aac-9de5-2d8c0702f878" /> </br>
<img width="355" height="43" alt="image" src="https://github.com/user-attachments/assets/77e60e4b-fd82-449a-b126-a99e8aea05e3" />

---

### Step 2 — Apply the PersistentVolume

```bash
kubectl apply -f 01-persistentVolume.yml
```
<img width="636" height="44" alt="image" src="https://github.com/user-attachments/assets/c85fd008-e957-4bc8-b5c4-54a5a3af5904" />

---

### Step 3 — Verify the PV Was Created

```bash
kubectl get pv
```
<img width="1266" height="99" alt="image" src="https://github.com/user-attachments/assets/67abb08f-614f-481f-8590-39202c9c8c00" />


Expected output:
```
NAME       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      STORAGECLASS    AGE
local-pv   1Gi        RWO            Retain           Available   local-storage   5s
```

Key columns:
- `STATUS: Available` → PV exists and is waiting for a PVC to claim it
- `ACCESS MODES: RWO` → ReadWriteOnce
- `RECLAIM POLICY: Retain` → data is kept after PVC is deleted

```bash
# Full details
kubectl describe pv local-pv
```

In the `describe` output, pay attention to:
- `Status` — `Available` means no PVC is bound yet
- `Claim` — empty until a PVC binds to it
- `Source` — shows the `hostPath` path on the node

<img width="592" height="361" alt="image" src="https://github.com/user-attachments/assets/e8b46c6c-1532-470f-84ae-e231b6fc7310" />

---

### Step 4 — Create a PersistentVolumeClaim

Create a PVC that requests this PV:

```yaml
# 02-persistentVolumeClaim.yml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: local-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: local-storage
```

```bash
kubectl apply -f 02-persistentVolumeClaim.yml
kubectl get pvc
```
<img width="681" height="41" alt="image" src="https://github.com/user-attachments/assets/f742ff00-fa00-4ec9-b4ad-89d62f2c759a" />    


<img width="1024" height="63" alt="image" src="https://github.com/user-attachments/assets/e312aab9-4041-4de6-998a-acbc9e54a150" />


Expected output:
```
NAME        STATUS   VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS    AGE
local-pvc   Bound    local-pv   1Gi        RWO            local-storage   3s
```

> 💡 `STATUS: Bound` means Kubernetes matched the PVC to `local-pv` and they are now linked. The PV status also changes from `Available` to `Bound`.

---

### Step 5 — Use the PVC in a Pod

```yaml
# 03-pod.yml
apiVersion: v1
kind: Pod
metadata:
  name: pv-test-pod
  namespace: default
spec:
  containers:
    - name: nginx
      image: nginx:1.27
      volumeMounts:
        - mountPath: /usr/share/nginx/html   # Where the volume is mounted inside the container
          name: local-storage
  volumes:
    - name: local-storage
      persistentVolumeClaim:
        claimName: local-pvc                 # References the PVC, not the PV directly
```

```bash
kubectl apply -f 03-pod.yml
kubectl get pod pv-test-pod
```
<img width="502" height="43" alt="image" src="https://github.com/user-attachments/assets/77a8a6ea-0f1c-43c3-8152-019d44b1b9ff" /> </br>
<img width="499" height="63" alt="image" src="https://github.com/user-attachments/assets/3968ea7d-726c-484d-a7c1-65156c4b47f9" />

---

### Step 6 — Verify Data Persistence

```bash
# Exec into the pod and check the mounted data
kubectl exec -it pv-test-pod -- cat /usr/share/nginx/html/index.html
```

Expected output:
```
Hello from PersistentVolume
```
<img width="689" height="59" alt="image" src="https://github.com/user-attachments/assets/161cadd5-11bb-406e-9c61-876b5b3b54ea" />
</br>

Now delete the pod and recreate it — the data should still be there:

```bash
kubectl delete pod pv-test-pod
kubectl apply -f 03-pod.yml
kubectl exec -it pv-test-pod -- cat /usr/share/nginx/html/index.html
```
<img width="532" height="44" alt="image" src="https://github.com/user-attachments/assets/f3dc918c-c82e-4473-a2f5-7be55822e9e1" /> </br>

<img width="532" height="44" alt="image" src="https://github.com/user-attachments/assets/b8b5afb5-4f5a-4c0f-b155-cbe5efa4ebb4" /> </br>

<img width="532" height="61" alt="image" src="https://github.com/user-attachments/assets/76010058-4a6f-4c75-8210-2569bee2330a" />



> 💡 The data persists because it lives on the host filesystem (`/mnt/data`), not inside the container. The pod is ephemeral; the PV is not.

---

### Step 7 — Expose the Pod via a Service

```bash
kubectl apply -f 04-service.yml
kubectl get svc nginx-service
```

Expected output:
```
NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
nginx-service   ClusterIP   10.96.x.x       <none>        89/TCP    3s
```

The service selects the pod via `app: nginx` label and forwards traffic from port `89` to the container's port `80`.

```bash
# Access the nginx page served from the PV-backed storage
kubectl run curl-test --image=curlimages/curl --restart=Never --rm -it -- curl http://nginx-service:89
```

Expected output:
```
Hello from PersistentVolume
```

> 💡 The pod is not accessed directly — the Service acts as a stable endpoint. Even if the pod is replaced, the Service keeps routing to the new pod as long as the label matches.

---

### Step 8 — Observe the Reclaim Policy in Action

Delete the PVC and observe what happens to the PV:

```bash
kubectl delete pvc local-pvc
kubectl get pv
```

Expected output:
```
NAME       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS     STORAGECLASS    AGE
local-pv   1Gi        RWO            Retain           Released   local-storage   5m
```

- `STATUS: Released` → the PVC is gone, but the PV still exists with its data intact
- The PV cannot be claimed by a new PVC yet — it needs to be manually reclaimed (see [Reclaim Policies](#reclaim-policies--what-happens-after-release))

---

### Step 9 — Clean Up

```bash
kubectl delete pod pv-test-pod
kubectl delete pvc local-pvc   # if not already deleted
kubectl delete pv local-pv
```

---

## Access Modes — Know the Difference

| Mode | Short | Meaning |
|---|---|---|
| `ReadWriteOnce` | RWO | Mounted read-write by **one node** at a time |
| `ReadOnlyMany` | ROX | Mounted read-only by **many nodes** simultaneously |
| `ReadWriteMany` | RWX | Mounted read-write by **many nodes** simultaneously |
| `ReadWriteOncePod` | RWOP | Mounted read-write by **one pod** at a time (Kubernetes 1.22+) |

```
ReadWriteOnce (RWO):              ReadWriteMany (RWX):
┌──────────┐                      ┌──────────┐  ┌──────────┐  ┌──────────┐
│  Node 1  │                      │  Node 1  │  │  Node 2  │  │  Node 3  │
│  Pod ✅  │◄── read/write        │  Pod ✅  │  │  Pod ✅  │  │  Pod ✅  │
└──────────┘         │            └──────────┘  └──────────┘  └──────────┘
┌──────────┐         │                    └──────────┬──────────┘
│  Node 2  │         ▼                               ▼
│  Pod ❌  │◄── blocked          ┌─────────────────────────┐
└──────────┘    (same PV)        │          PV             │ ← all can read/write
                                 └─────────────────────────┘
```

> ⚠️ Access modes are about **nodes**, not pods. `ReadWriteOnce` allows multiple pods on the **same node** to mount the volume. `ReadWriteOncePod` is stricter — only one pod cluster-wide.

> ⚠️ Not all storage backends support all access modes. `hostPath` only supports `RWO`. NFS supports `RWX`. Always check your storage driver's documentation.

---

## Reclaim Policies — What Happens After Release

When a PVC is deleted, the bound PV enters `Released` state. What happens next depends on the `persistentVolumeReclaimPolicy`:

| Policy | Behavior | Use Case |
|---|---|---|
| `Retain` | PV stays, data intact, status becomes `Released`. Admin must manually reclaim | Production — don't lose data accidentally |
| `Delete` | PV and underlying storage are automatically deleted | Dynamic provisioning with cloud storage (EBS, GCE PD) |
| `Recycle` | ⚠️ Deprecated. Ran `rm -rf` on the volume and made it `Available` again | Don't use |

```
PVC Deleted
     │
     ▼
PV Status: Released
     │
     ├── Retain  → PV exists, data intact, cannot be rebound until admin clears claimRef
     ├── Delete  → PV and storage backend are deleted automatically
     └── Recycle → (Deprecated) Data wiped, PV becomes Available again
```

To manually reclaim a `Retain` PV and make it available for a new PVC:

```bash
# Remove the claimRef from the PV spec
kubectl patch pv local-pv -p '{"spec":{"claimRef": null}}'

# PV status returns to Available
kubectl get pv local-pv
```

---

## PersistentVolumes in Production — What Changes

The `01-persistentVolume.yml` in this lab uses `hostPath` which is only suitable for local development. In production, you would use:

- **Cloud block storage:** AWS EBS, GCE Persistent Disk, Azure Disk (RWO only)
- **Cloud file storage:** AWS EFS, Azure Files, GCE Filestore (RWX capable)
- **Network storage:** NFS, CephFS, GlusterFS

In production, PVs are almost never created manually. Instead, a **StorageClass** with a provisioner handles dynamic provisioning:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: production-pv
spec:
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: gp3                  # Maps to an AWS EBS gp3 StorageClass
  csi:
    driver: ebs.csi.aws.com             # AWS EBS CSI driver
    volumeHandle: vol-0a1b2c3d4e5f      # Actual EBS volume ID
    fsType: ext4
```

> 💡 In most production clusters, you won't create PVs manually at all. You create a PVC, and the StorageClass + CSI driver provisions the underlying storage (EBS volume, EFS mount, etc.) automatically.

---

## Interview Q&A — Straight to the Point

**Q: What is a PersistentVolume?**
A PersistentVolume is a cluster-level storage resource that exists independently of any pod. It represents actual storage — a directory on a node, an NFS share, a cloud disk — and its lifecycle is decoupled from the pods that use it.

---

**Q: What is the difference between a PV and a PVC?**
A PV is the actual storage resource, provisioned by an admin. A PVC is a request for storage by a developer or pod. Kubernetes binds a PVC to a compatible PV. The pod references the PVC — it never interacts with the PV directly.

---

**Q: Are PersistentVolumes namespaced?**
No. PVs are cluster-scoped. PVCs are namespace-scoped. A PVC in any namespace can bind to a cluster-level PV.

---

**Q: What happens to a PV when its PVC is deleted?**
It depends on the `persistentVolumeReclaimPolicy`. `Retain` keeps the PV and data intact (status becomes `Released`). `Delete` removes the PV and the underlying storage. `Recycle` is deprecated.

---

**Q: What does `ReadWriteOnce` mean?**
The volume can be mounted read-write by one **node** at a time. Multiple pods on the same node can use it. Use `ReadWriteOncePod` if you need single-pod exclusivity.

---

**Q: What is a StorageClass?**
A StorageClass defines a "class" of storage with a specific provisioner and parameters. It enables dynamic provisioning — when a PVC is created, the StorageClass automatically provisions a matching PV. This eliminates the need for admins to manually create PVs.

---

**Q: What is the difference between static and dynamic provisioning?**
Static provisioning: an admin manually creates PVs ahead of time. Dynamic provisioning: a StorageClass + CSI driver automatically creates a PV when a PVC is submitted. Dynamic provisioning is the standard in production cloud environments.

---

**Q: Why should you not use `hostPath` in production?**
`hostPath` ties data to a specific node's filesystem. If the pod is rescheduled to a different node, it won't find the data. It also creates security risks by exposing the host filesystem. Use cloud storage or network storage in production.

---

**Q: What is a CSI driver?**
Container Storage Interface (CSI) is a standard for exposing storage systems to containerized workloads. CSI drivers (e.g., `ebs.csi.aws.com`) allow Kubernetes to dynamically provision, attach, and mount storage from external systems like AWS EBS or EFS.

---

## Common Mistakes & Gotchas

| Mistake | Why It's Wrong | Fix |
|---|---|---|
| Using `hostPath` in production | Data is tied to one node — pod rescheduling loses data | Use cloud storage or NFS with a proper CSI driver |
| PVC `storageClassName` doesn't match PV | PVC stays in `Pending` forever | Ensure both PV and PVC use the same `storageClassName` |
| PVC requests more storage than PV provides | No matching PV found, PVC stays `Pending` | PVC capacity must be ≤ PV capacity |
| Access modes don't match | PVC won't bind to PV | PVC access mode must be a subset of PV access modes |
| Expecting a `Released` PV to auto-rebind | `Retain` policy keeps the `claimRef` — new PVCs can't bind | Manually patch the PV to clear `claimRef` |
| Forgetting PVs are cluster-scoped | Trying to create a PV in a namespace with `-n` flag | PVs have no namespace — omit `-n` in kubectl commands |
| Deleting a PVC while a pod is still using it | PVC enters `Terminating` state but isn't deleted until the pod releases it | Delete the pod first, then the PVC |
| No `resources.requests.storage` in PVC | Kubernetes rejects the PVC manifest | Always specify storage request in PVC |

---

## Files in This Lab

```
01-persistentVolume/
├── 01-persistentVolume.yml      ← PV manifest (hostPath, 1Gi, ReadWriteOnce, Retain)
├── 02-persistentVolumeClaim.yml ← PVC manifest (requests 1Gi, ReadWriteOnce, local-storage)
├── 03-pod.yml                   ← Pod manifest (nginx, mounts PVC at /usr/share/nginx/html)
├── 04-service.yml               ← ClusterIP Service (port 89 → container port 80)
└── README.md                    ← This file
```

---

## ✍️ Author

**[Himanshu Kumar](https://www.linkedin.com/in/h1manshu-kumar/)** - Learning by building, documenting, and sharing 🚀

---
<div align="center">

**Built while learning Kubernetes hands-on.**  
*The best way to understand PersistentVolumes is to delete a pod and watch your data still be there.*

</div>
