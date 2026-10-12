# 🗄️ Kubernetes StatefulSets — Deep Dive

> **Lab Series:** Advanced Features & Configs → 01 StatefulSets    
> **Difficulty:** Intermediate → Advanced    
> **Estimated Time:** 45–60 minutes    
> **Focus:** Understand what StatefulSets are, how they differ from Deployments, and when you must use them    

---

## 📌 Table of Contents

1. [Why StatefulSets Exist](#why-statefulsets-exist)
2. [StatefulSet vs Deployment — Know the Difference](#statefulset-vs-deployment--know-the-difference)
3. [How a StatefulSet Works — Identity, Ordering, and Storage](#how-a-statefulset-works--identity-ordering-and-storage)
4. [The Headless Service — Why It's Required](#the-headless-service--why-its-required)
5. [Manifest Breakdown — Field by Field](#manifest-breakdown--field-by-field)
6. [Hands-On Lab](#hands-on-lab)
7. [volumeClaimTemplates — Per-Pod Persistent Storage](#volumeclaimtemplates--per-pod-persistent-storage)
8. [StatefulSets in Production — What Changes](#statefulsets-in-production--what-changes)
9. [Interview Q&A — Straight to the Point](#interview-qa--straight-to-the-point)
10. [Common Mistakes & Gotchas](#common-mistakes--gotchas)
11. [What's Next](#whats-next)

---

## Why StatefulSets Exist

In the previous labs, you used Deployments and ReplicaSets. These work perfectly for **stateless** workloads — every pod is identical, interchangeable, and disposable. Lose one, create another, no problem.

But some applications fundamentally cannot work this way:

- A **MySQL** replica needs to know it's replica-2, not replica-0 (the primary)
- A **Kafka** broker needs a stable hostname so other brokers can find it after a restart
- An **Elasticsearch** node needs its own dedicated disk — it can't share data with another node
- A **ZooKeeper** node needs a persistent identity across rescheduling

These are **stateful** workloads. They need:

1. **Stable, unique network identity** — the pod name and hostname must not change across restarts
2. **Stable, persistent storage** — each pod gets its own PVC that follows it across rescheduling
3. **Ordered, graceful deployment and scaling** — pods start and stop in a predictable sequence

A StatefulSet provides all three. A Deployment provides none of them.

| Scenario | Deployment | StatefulSet |
|---|---|---|
| Pod gets a stable hostname | ❌ Random hash suffix | ✅ `pod-0`, `pod-1`, `pod-2` |
| Pod gets its own dedicated PVC | ❌ All pods share or use ephemeral storage | ✅ Each pod gets its own PVC via `volumeClaimTemplates` |
| Pods start in order | ❌ All start simultaneously | ✅ `pod-0` → `pod-1` → `pod-2` sequentially |
| Pods stop in reverse order | ❌ Arbitrary | ✅ `pod-2` → `pod-1` → `pod-0` |
| DNS record per pod | ❌ | ✅ `pod-0.service.namespace.svc.cluster.local` |
| Use case | Web servers, APIs, microservices | Databases, message queues, distributed systems |

> 💡 **The Rule:** If your application stores data on disk, has a concept of primary/replica, or needs peers to find it by a stable name — use a StatefulSet. Otherwise, use a Deployment.

---

## StatefulSet vs Deployment — Know the Difference

| Feature | Deployment | StatefulSet |
|---|---|---|
| Pod naming | Random: `nginx-7d4b9c-xkz2p` | Ordered: `mysql-statefulset-0` |
| Pod identity | Interchangeable | Unique and sticky |
| Storage | Shared volume or ephemeral | Per-pod PVC via `volumeClaimTemplates` |
| Startup order | Parallel | Sequential (0 → 1 → 2) |
| Shutdown order | Arbitrary | Reverse sequential (2 → 1 → 0) |
| Requires a Service | Optional (ClusterIP) | Required — must be a **Headless Service** |
| Rolling updates | Parallel by default | One pod at a time, in reverse order |
| PVC lifecycle | Deleted with pod | **PVC is NOT deleted when pod or StatefulSet is deleted** |

```
Deployment (stateless):
  ┌─────────────────────────────────────────────────────┐
  │  nginx-7d4b9c-xkz2p  │  nginx-7d4b9c-ab3mn  │ ...  │
  │  (interchangeable)   │  (interchangeable)   │      │
  └─────────────────────────────────────────────────────┘
         All share the same identity — any pod can serve any request

StatefulSet (stateful):
  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
  │  mysql-ss-0      │  │  mysql-ss-1      │  │  mysql-ss-2      │
  │  (Primary)       │  │  (Replica 1)     │  │  (Replica 2)     │
  │  PVC: data-0     │  │  PVC: data-1     │  │  PVC: data-2     │
  └──────────────────┘  └──────────────────┘  └──────────────────┘
         Each pod has a unique role, unique storage, unique DNS name
```

---

## How a StatefulSet Works — Identity, Ordering, and Storage

### Stable Network Identity

Every pod in a StatefulSet gets a predictable name: `<statefulset-name>-<ordinal>`.

For `mysql-statefulset` with `replicas: 3`:
- `mysql-statefulset-0`
- `mysql-statefulset-1`
- `mysql-statefulset-2`

This name is **sticky** — if `mysql-statefulset-1` is rescheduled to a different node, it comes back as `mysql-statefulset-1`, not a new random name.

Combined with a Headless Service, each pod also gets a stable DNS entry:
```
mysql-statefulset-0.mysql-service.mysql.svc.cluster.local
mysql-statefulset-1.mysql-service.mysql.svc.cluster.local
```

### Ordered Startup and Shutdown

```
Scale up (replicas: 0 → 3):
  mysql-statefulset-0 starts → becomes Ready
        ↓
  mysql-statefulset-1 starts → becomes Ready
        ↓
  mysql-statefulset-2 starts → becomes Ready

Scale down (replicas: 3 → 0):
  mysql-statefulset-2 terminates
        ↓
  mysql-statefulset-1 terminates
        ↓
  mysql-statefulset-0 terminates
```

This ordering matters for databases: the primary (`-0`) must be up before replicas (`-1`, `-2`) try to connect to it.

### The Reconciliation Loop

```
kubectl apply -f statefulset.yml
        │
        ▼
┌───────────────┐
│  API Server   │  ← Stores desired state in etcd
└───────┬───────┘
        │
        ▼
┌────────────────────┐
│  StatefulSet       │  ← Watches pod count, enforces ordering,
│  Controller        │    manages PVC creation per pod
└───────┬────────────┘
        │
        ▼
┌────────────────────┐
│   Scheduler        │  ← Assigns each pod to a node
└───────┬────────────┘
        │
        ▼
┌────────────────────┐
│  Kubelet (node)    │  ← Pulls image, mounts the pod's dedicated PVC
└────────────────────┘
```

---

## The Headless Service — Why It's Required

A StatefulSet **requires** a Headless Service (`clusterIP: None`). This is not optional.

A normal Service gives you a single virtual IP that load-balances across all pods. That's useless for a database cluster — you need to address individual pods directly (e.g., send writes only to the primary).

A Headless Service skips the virtual IP entirely. Instead, DNS returns the individual pod IPs directly, and each pod gets its own DNS A record.

```yaml
# service.yml — Headless Service
kind: Service
apiVersion: v1
metadata:
  name: mysql-service
  namespace: mysql
spec:
  clusterIP: None        # ← This is what makes it headless
  selector:
    app: mysql
  ports:
  - name: mysql
    protocol: TCP
    port: 3306
    targetPort: 3306
```

```
Normal ClusterIP Service:
  client → mysql-service (10.96.0.1) → random pod

Headless Service:
  client → mysql-statefulset-0.mysql-service.mysql.svc.cluster.local → pod-0 directly
  client → mysql-statefulset-1.mysql-service.mysql.svc.cluster.local → pod-1 directly
```

> ⚠️ The `serviceName` field in the StatefulSet spec must match the `metadata.name` of the Headless Service. This is how the StatefulSet knows which service to use for DNS registration.

---

## Manifest Breakdown — Field by Field

### statefulset.yml

```yaml
kind: StatefulSet
apiVersion: apps/v1
metadata:
  name: mysql-statefulset    # Pods will be named mysql-statefulset-0, -1, -2...
  namespace: mysql

spec:
  serviceName: mysql-service  # Must match the Headless Service name — used for DNS
  replicas: 1                 # Number of pods to maintain

  selector:
    matchLabels:
      app: mysql              # Must match template.metadata.labels

  template:
    metadata:
      labels:
        app: mysql            # Must match selector.matchLabels
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        ports:
        - containerPort: 3306
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:           # Password pulled from a Secret
              name: mysql-secret
              key: MYSQL_ROOT_PASSWORD
        - name: MYSQL_DATABASE
          valueFrom:
            configMapKeyRef:        # DB name pulled from a ConfigMap
              name: mysql-config-map
              key: MYSQL_DATABASE
        volumeMounts:
        - name: mysql-data
          mountPath: /var/lib/mysql  # Where MySQL stores its data files

  volumeClaimTemplates:              # Per-pod PVC — each pod gets its own
  - metadata:
      name: mysql-data               # Must match volumeMounts[].name
    spec:
      accessModes: ["ReadWriteOnce"] # One pod reads/writes at a time
      resources:
        requests:
          storage: 1Gi
```

### Every Field Explained

| Field | Required | Purpose |
|---|---|---|
| `kind: StatefulSet` | ✅ | Resource type |
| `apiVersion: apps/v1` | ✅ | StatefulSets live in the `apps` API group |
| `spec.serviceName` | ✅ | Name of the Headless Service — enables per-pod DNS |
| `spec.replicas` | ❌ | Defaults to `1` if omitted |
| `spec.selector.matchLabels` | ✅ | Must match `template.metadata.labels` |
| `spec.template` | ✅ | Pod blueprint — same as any other workload |
| `env[].valueFrom.secretKeyRef` | — | Pulls sensitive values from a Secret (not hardcoded) |
| `env[].valueFrom.configMapKeyRef` | — | Pulls config values from a ConfigMap |
| `volumeMounts[].mountPath` | — | Where the volume is mounted inside the container |
| `volumeClaimTemplates` | — | Defines a PVC template — one PVC is created per pod |
| `volumeClaimTemplates[].accessModes` | ✅ (if using VCT) | `ReadWriteOnce` = one node at a time |
| `volumeClaimTemplates[].resources.requests.storage` | ✅ (if using VCT) | Storage size per pod |

> ⚠️ **Critical:** `spec.selector.matchLabels` and `spec.template.metadata.labels` must match — same rule as ReplicaSets and Deployments.

> ⚠️ **Critical:** `volumeClaimTemplates[].metadata.name` must match the `volumeMounts[].name` in the container spec. This is how Kubernetes knows which PVC to mount where.

---

## Hands-On Lab

### Prerequisites
- A running Kubernetes cluster with a default StorageClass (Minikube, Kind, EKS, GKE, or AKS)
- `kubectl` configured (`kubectl cluster-info` to verify)
- Verify a StorageClass exists: `kubectl get storageclass`

---

### Step 1 — Create the Namespace

```bash
kubectl apply -f mysql/namespace.yml
```

Verify:
```bash
kubectl get namespace mysql
```
<img width="362" height="179" alt="image" src="https://github.com/user-attachments/assets/7c07b04c-6aaa-4ef9-97a6-e144ce0c8c67" />

---

### Step 2 — Create the Secret and ConfigMap

The StatefulSet references a Secret (`mysql-secret`) and a ConfigMap (`mysql-config-map`). Create them before applying the StatefulSet:

```bash
# Create the Secret for the root password
kubectl create secret generic mysql-secret \
  --from-literal=MYSQL_ROOT_PASSWORD=<your-password> \
  -n mysql

# Create the ConfigMap for the database name
kubectl create configmap mysql-config-map \
  --from-literal=MYSQL_DATABASE=myappdb \
  -n mysql
```

Verify:
```bash
kubectl get secret,configmap -n mysql
```
<img width="612" height="135" alt="image" src="https://github.com/user-attachments/assets/a4ecba88-9c09-419e-93f9-6ffaf7a909a0" />

---

### Step 3 — Apply the Headless Service

The Service must exist before the StatefulSet — pods register their DNS entries against it on startup.

```bash
kubectl apply -f mysql/service.yml
```
<img width="535" height="47" alt="image" src="https://github.com/user-attachments/assets/5e190df0-61e0-495e-b5b8-e3f947383432" />

Verify:
```bash
kubectl get service -n mysql
```
<img width="708" height="60" alt="image" src="https://github.com/user-attachments/assets/c5f4041d-bc0b-470c-9dc6-7d14d49f7e92" />

Expected output — note `CLUSTER-IP` is `None`:
```
NAME            TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
mysql-service   ClusterIP   None         <none>        3306/TCP   5s
```

---

### Step 4 — Apply the StatefulSet

```bash
kubectl apply -f mysql/statefulset.yml
```
<img width="573" height="62" alt="image" src="https://github.com/user-attachments/assets/4fe93c0e-119a-4c06-81e3-6b23e179adac" />

---

### Step 5 — Verify Everything Was Created

```bash
# Check the StatefulSet
kubectl get statefulset -n mysql

# Check the Pod — note the stable name: mysql-statefulset-0
kubectl get pods -n mysql

# Check the PVC — automatically created by volumeClaimTemplates
kubectl get pvc -n mysql
```

Expected output:
```
NAME                READY   AGE
mysql-statefulset   1/1     30s

NAME                    READY   STATUS    RESTARTS   AGE
mysql-statefulset-0     1/1     Running   0          30s

NAME                                   STATUS   VOLUME     CAPACITY   ACCESS MODES
mysql-data-mysql-statefulset-0         Bound    pvc-xxx    1Gi        RWO
```
<img width="576" height="61" alt="image" src="https://github.com/user-attachments/assets/e1fd38fc-6aab-4495-86e1-2773be325641" /> </br>

<img width="582" height="61" alt="image" src="https://github.com/user-attachments/assets/12734272-6da6-426f-87fa-e048846082f1" /> </br>

<img width="1254" height="100" alt="image" src="https://github.com/user-attachments/assets/b4894455-be29-4918-9d70-62b479033a18" />

> 💡 The PVC name follows the pattern: `<volumeClaimTemplate-name>-<pod-name>` → `mysql-data-mysql-statefulset-0`. This naming is automatic and deterministic.

---

### Step 6 — Inspect the StatefulSet

```bash
# Full details — events, volume claim templates, update strategy
kubectl describe statefulset mysql-statefulset -n mysql

# See which node the pod landed on
kubectl get pods -n mysql -o wide
```
<img width="1120" height="674" alt="image" src="https://github.com/user-attachments/assets/0d2914d1-b51f-48d0-a09d-532e4d412157" /> </br>

<img width="1267" height="83" alt="image" src="https://github.com/user-attachments/assets/a5256722-377b-41b8-89cb-9c9a9a15344a" /> </br>

---

### Step 7 — Test Stable Identity (Self-Healing)

Delete the pod and watch it come back with the **same name and same PVC**:

```bash
# Delete the pod
kubectl delete pod mysql-statefulset-0 -n mysql

# Watch it get recreated with the same name
kubectl get pods -n mysql -w
```
<img width="814" height="43" alt="image" src="https://github.com/user-attachments/assets/23e49c1b-a6da-4f00-9a81-67bf03148928" />
<img width="1248" height="62" alt="image" src="https://github.com/user-attachments/assets/79a5634c-b1e8-41fa-89a1-72e651cfe77b" />

After recreation:
```bash
# The PVC is still there — data is preserved
kubectl get pvc -n mysql
```
<img width="1238" height="96" alt="image" src="https://github.com/user-attachments/assets/bb664a32-5aa5-425c-9b62-cd375d417fbf" />

> 💡 This is the core value of a StatefulSet. The pod comes back as `mysql-statefulset-0` (not a random name) and reattaches to `mysql-data-mysql-statefulset-0` (its original PVC). Data survives the pod restart.

---

### Step 8 — Test Scale Up

```bash
# Scale to 3 replicas — watch them start in order: -0 (exists), -1, then -2
kubectl scale statefulset mysql-statefulset -n mysql --replicas=3

# Watch the ordered startup
kubectl get pods -n mysql -w
```

Check that each pod got its own PVC:
```bash
kubectl get pvc -n mysql
```

Expected:
```
mysql-data-mysql-statefulset-0   Bound   1Gi
mysql-data-mysql-statefulset-1   Bound   1Gi
mysql-data-mysql-statefulset-2   Bound   1Gi
```

---

### Step 9 — Verify Per-Pod DNS (Stable Network Identity)

```bash
# Exec into the pod and test DNS resolution
kubectl exec -it mysql-statefulset-0 -n mysql -- bash

# Inside the pod — resolve its own stable DNS name
getent hosts mysql-statefulset-0.mysql-service.mysql.svc.cluster.local

```
<img width="860" height="51" alt="image" src="https://github.com/user-attachments/assets/d7d2d614-9c8b-4810-8638-d2a7740d1e8c" />

<img width="940" height="39" alt="image" src="https://github.com/user-attachments/assets/257fa46f-d6cc-4935-bf62-fda13244f396" />

> 💡 This DNS name is stable. Even if the pod is rescheduled to a different node with a different IP, this DNS name will resolve to the new IP. Other services can always reach this pod by name.

---

### Step 10 — Clean Up

```bash
# Delete the StatefulSet and Service
kubectl delete statefulset mysql-statefulset -n mysql
kubectl delete service mysql-service -n mysql

# ⚠️ PVCs are NOT deleted automatically — you must delete them manually
kubectl delete pvc -n mysql --all

# Delete the namespace
kubectl delete namespace mysql
```

> ⚠️ This is intentional behavior. Kubernetes protects your data by not auto-deleting PVCs when a StatefulSet is deleted. Always clean up PVCs explicitly.

---

## volumeClaimTemplates — Per-Pod Persistent Storage

This is the most important StatefulSet-specific feature. It's what separates StatefulSets from every other workload type.

```
volumeClaimTemplates creates one PVC per pod, automatically:

  StatefulSet (replicas: 3)
  ├── mysql-statefulset-0  ──→  PVC: mysql-data-mysql-statefulset-0  (1Gi)
  ├── mysql-statefulset-1  ──→  PVC: mysql-data-mysql-statefulset-1  (1Gi)
  └── mysql-statefulset-2  ──→  PVC: mysql-data-mysql-statefulset-2  (1Gi)

  Each pod reads/writes only its own PVC. No sharing.
```

**Key behaviors:**

1. **PVCs are created automatically** when a pod is created — you don't create them manually
2. **PVCs survive pod deletion** — if `mysql-statefulset-1` is deleted and recreated, it reattaches to `mysql-data-mysql-statefulset-1`
3. **PVCs are NOT deleted when the StatefulSet is deleted** — you must delete them manually
4. **PVCs are NOT deleted when scaling down** — if you scale from 3 to 1, the PVCs for `-1` and `-2` remain. If you scale back to 3, those pods reattach to their original PVCs

This design ensures data is never accidentally lost due to a pod restart or a scaling event.

---

## StatefulSets in Production — What Changes

The `statefulset.yml` in this lab is intentionally minimal. In production, you would add:

```yaml
kind: StatefulSet
apiVersion: apps/v1
metadata:
  name: mysql-statefulset
  namespace: mysql
spec:
  serviceName: mysql-service
  replicas: 3
  podManagementPolicy: OrderedReady   # Default — ensures ordered startup
  updateStrategy:
    type: RollingUpdate               # Updates one pod at a time, in reverse order
    rollingUpdate:
      partition: 0                    # Update all pods (set > 0 for canary rollouts)
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0              # Pin to a specific version — never :latest
        ports:
        - containerPort: 3306
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: MYSQL_ROOT_PASSWORD
        - name: MYSQL_DATABASE
          valueFrom:
            configMapKeyRef:
              name: mysql-config-map
              key: MYSQL_DATABASE
        resources:
          requests:
            cpu: "250m"
            memory: "512Mi"
          limits:
            cpu: "1"
            memory: "1Gi"
        readinessProbe:
          exec:
            command: ["mysqladmin", "ping", "-h", "localhost"]
          initialDelaySeconds: 10
          periodSeconds: 5
        livenessProbe:
          exec:
            command: ["mysqladmin", "ping", "-h", "localhost"]
          initialDelaySeconds: 30
          periodSeconds: 10
        volumeMounts:
        - name: mysql-data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: mysql-data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: "gp2"         # Explicitly set StorageClass in production
      resources:
        requests:
          storage: 20Gi
```

---

## Interview Q&A — Straight to the Point

**Q: What is a StatefulSet and when do you use it?**
A StatefulSet is a Kubernetes workload controller for stateful applications. Use it when your application needs stable pod names, stable per-pod persistent storage, or ordered startup/shutdown. Common use cases: databases (MySQL, PostgreSQL, MongoDB), message queues (Kafka, RabbitMQ), and distributed coordination systems (ZooKeeper, etcd).

---

**Q: What are the three guarantees a StatefulSet provides?**
1. Stable, unique network identity — pods are named `<name>-0`, `<name>-1`, etc., and keep that name across restarts
2. Stable, persistent storage — each pod gets its own PVC via `volumeClaimTemplates` that survives pod restarts
3. Ordered, graceful deployment and scaling — pods start sequentially (0 → 1 → 2) and terminate in reverse (2 → 1 → 0)

---

**Q: What is a Headless Service and why does a StatefulSet require one?**
A Headless Service has `clusterIP: None`. It doesn't create a virtual IP or load balance traffic. Instead, DNS returns individual pod IPs, and each pod gets its own DNS A record (`pod-0.service.namespace.svc.cluster.local`). StatefulSets require it because stateful applications need to address individual pods directly — not a random pod behind a load balancer.

---

**Q: What happens to PVCs when you delete a StatefulSet?**
PVCs are NOT deleted. Kubernetes intentionally protects data by leaving PVCs behind. You must delete them manually with `kubectl delete pvc`. This is a common gotcha — if you forget, you'll accumulate orphaned PVCs and storage costs.

---

**Q: What happens to PVCs when you scale a StatefulSet down?**
PVCs are NOT deleted. If you scale from 3 to 1, the PVCs for pods `-1` and `-2` remain. If you scale back to 3, those pods reattach to their original PVCs and recover their data.

---

**Q: What is `podManagementPolicy` and what are the options?**
It controls how pods are created and deleted during scaling. `OrderedReady` (default) creates/deletes pods one at a time in order. `Parallel` creates/deletes all pods simultaneously — useful when ordering doesn't matter but you still need stable identities and storage.

---

**Q: What is the difference between a StatefulSet's rolling update and a Deployment's?**
A Deployment updates pods in parallel (configurable). A StatefulSet updates pods one at a time in reverse ordinal order (`-2` → `-1` → `-0`), waiting for each pod to become Ready before proceeding. StatefulSets also support `partition` for canary-style rollouts — only pods with ordinal ≥ partition value are updated.

---

**Q: Can you change `spec.selector` on an existing StatefulSet?**
No. Like ReplicaSets, the selector is immutable after creation. Delete and recreate the StatefulSet to change it.

---

**Q: What is the naming pattern for PVCs created by `volumeClaimTemplates`?**
`<volumeClaimTemplate-name>-<statefulset-name>-<ordinal>`. For this lab: `mysql-data-mysql-statefulset-0`.

---

**Q: What's the difference between a StatefulSet and a DaemonSet?**
A StatefulSet runs a specified number of pods with stable identities and storage — you control the replica count. A DaemonSet runs exactly one pod per node (or per selected node) — the cluster topology controls the count. Use DaemonSets for node-level agents (log collectors, monitoring), not for databases.

---

## Common Mistakes & Gotchas

| Mistake | Why It's Wrong | Fix |
|---|---|---|
| Forgetting the Headless Service | StatefulSet pods won't get stable DNS entries | Always create the Headless Service before the StatefulSet |
| `serviceName` doesn't match the Service `metadata.name` | DNS registration fails silently | They must be identical |
| Using a regular ClusterIP Service instead of Headless | Pods get load-balanced, not individually addressable | Set `clusterIP: None` |
| Expecting PVCs to be deleted with the StatefulSet | PVCs are intentionally preserved — you'll have orphaned storage | Always `kubectl delete pvc --all -n <namespace>` after cleanup |
| Using `image: mysql:latest` | Image can silently change — breaks reproducibility and data compatibility | Pin to `mysql:8.0` or a specific patch version |
| No `readinessProbe` on a database | Kubernetes marks the pod Ready before MySQL is actually accepting connections | Add a `mysqladmin ping` readiness probe |
| Scaling down without understanding PVC retention | PVCs for scaled-down pods remain and incur storage costs | Monitor PVCs and clean up intentionally |
| `volumeMounts[].name` ≠ `volumeClaimTemplates[].metadata.name` | Kubernetes can't match the PVC to the mount — pod fails to start | They must be identical |
| Applying the StatefulSet before the Secret/ConfigMap exist | Pod fails with `CreateContainerConfigError` | Apply Secret and ConfigMap first |
| No `storageClassName` in production | Uses the default StorageClass — may not be appropriate for your workload | Explicitly set `storageClassName` |

---

## Files in This Lab

```
01-statefulsets/
├── mysql/
│   ├── namespace.yml     ← Creates the mysql namespace
│   ├── service.yml       ← Headless Service (clusterIP: None) — required for DNS
│   └── statefulset.yml   ← MySQL StatefulSet with volumeClaimTemplates
└── README.md             ← This file
```

**Apply order matters:**
```
namespace.yml → Secret + ConfigMap → service.yml → statefulset.yml
```

---

## ✍️ Author

**[Himanshu Kumar](https://www.linkedin.com/in/h1manshu-kumar/)** - Learning by building, documenting, and sharing 🚀

---
<div align="center">

**Built while learning Kubernetes hands-on.**
*The best way to understand StatefulSets is to delete a pod, watch it come back with the same name, and verify your data is still there.*

</div>
