# 👾 Kubernetes DaemonSets — Deep Dive

> **Lab Series:** Core Workloads → 05 DaemonSet    
> **Difficulty:** Beginner → Intermediate    
> **Estimated Time:** 30–45 minutes    
> **Focus:** Understand what DaemonSets are, how they differ from ReplicaSets, and when to use them    
---

## 📌 Table of Contents

1. [Why DaemonSets Exist](#why-daemonsets-exist)
2. [DaemonSet vs ReplicaSet — Know the Difference](#daemonset-vs-replicaset--know-the-difference)
3. [How a DaemonSet Works — The Reconciliation Loop](#how-a-daemonset-works--the-reconciliation-loop)
4. [DaemonSet Manifest — Field by Field Breakdown](#daemonset-manifest--field-by-field-breakdown)
5. [Hands-On Lab](#hands-on-lab)
6. [Node Selectors & Tolerations — Controlling Where Pods Land](#node-selectors--tolerations--controlling-where-pods-land)
7. [DaemonSets in Production — What Changes](#daemonsets-in-production--what-changes)
8. [Interview Q&A — Straight to the Point](#interview-qa--straight-to-the-point)
9. [Common Mistakes & Gotchas](#common-mistakes--gotchas)

---

## Why DaemonSets Exist

In the previous lab, you used a ReplicaSet to run N copies of a Pod across your cluster. The problem with that model is that it doesn't care *where* the pods land — the scheduler decides. You might end up with 3 pods on 2 nodes, and 1 node gets nothing.

A DaemonSet solves a completely different problem: **ensuring exactly one Pod runs on every node in the cluster (or a subset of nodes).**

| Scenario | ReplicaSet | DaemonSet |
|---|---|---|
| Run 3 copies of a pod | ✅ | ❌ Not the right tool |
| Run one pod on every node | ❌ Not guaranteed | ✅ |
| New node joins the cluster | ❌ May or may not get a pod | ✅ Pod is automatically scheduled on it |
| Node is removed | ❌ Pod is rescheduled elsewhere | ✅ Pod is simply gone (was tied to that node) |
| Log collection agent per node | ❌ Unreliable | ✅ Perfect use case |
| Node monitoring agent | ❌ Unreliable | ✅ Perfect use case |

> 💡 **The Rule:** Use a DaemonSet when your workload is node-scoped — it needs to run on every node, not just somewhere in the cluster. Classic examples: log shippers (Fluentd, Filebeat), monitoring agents (Prometheus Node Exporter, Datadog), network plugins (Calico, Weave), and storage daemons (Ceph).

---

## DaemonSet vs ReplicaSet — Know the Difference

| Feature | ReplicaSet | DaemonSet |
|---|---|---|
| Controls pod count | ✅ You set `replicas: N` | ❌ Count is determined by node count |
| Guarantees one pod per node | ❌ | ✅ |
| Responds to new nodes joining | ❌ | ✅ Automatically schedules a pod |
| Can target a subset of nodes | ❌ | ✅ Via `nodeSelector` or `tolerations` |
| Runs on control-plane nodes | ❌ (by default) | ✅ With the right toleration |
| Typical use case | Stateless app replicas | Node-level agents and daemons |

```
ReplicaSet (replicas: 3) — Scheduler decides placement:
┌──────────┐   ┌──────────┐   ┌──────────┐
│  Node 1  │   │  Node 2  │   │  Node 3  │
│  Pod ✅  │   │  Pod ✅  │   │          │  ← Node 3 gets nothing
│  Pod ✅  │   │          │   │          │
└──────────┘   └──────────┘   └──────────┘

DaemonSet — One pod per node, guaranteed:
┌──────────┐   ┌──────────┐   ┌──────────┐
│  Node 1  │   │  Node 2  │   │  Node 3  │
│  Pod ✅  │   │  Pod ✅  │   │  Pod ✅  │  ← Every node covered
└──────────┘   └──────────┘   └──────────┘
```

> ⚠️ **Key Insight:** You never set `replicas` on a DaemonSet. The number of pods is always equal to the number of eligible nodes. If you have 5 nodes, you get 5 pods. Add a 6th node — you get a 6th pod automatically.

---

## How a DaemonSet Works — The Reconciliation Loop

Like a ReplicaSet, a DaemonSet controller runs a continuous reconciliation loop — but the desired state is different:

```
┌──────────────────────────────────────────────────────────┐
│               DaemonSet Controller Loop                  │
│                                                          │
│   For each eligible node in the cluster:                 │
│     Pod running on this node? → Do nothing               │
│     Pod NOT running on this node? → Create one           │
│     Node removed from cluster? → Pod is gone (expected)  │
└──────────────────────────────────────────────────────────┘
```

```
kubectl apply -f DaemonSet.yml
        │
        ▼
┌───────────────┐
│  API Server   │  ← Stores desired state in etcd
└───────┬───────┘
        │
        ▼
┌────────────────────┐
│  DaemonSet         │  ← Watches node list and pod list
│  Controller        │    Creates one pod per eligible node
└───────┬────────────┘
        │
        ▼
┌────────────────────┐
│   Scheduler        │  ← Assigns each pod to its specific node
└───────┬────────────┘
        │
        ▼
┌────────────────────┐
│  Kubelet (node)    │  ← Pulls image, starts container
└────────────────────┘
```

> 💡 Unlike a ReplicaSet, the DaemonSet controller doesn't ask the scheduler "find me a node" — it tells the scheduler exactly which node each pod should go to. The pod spec gets a `nodeName` field set directly by the DaemonSet controller.

---

## DaemonSet Manifest — Field by Field Breakdown

This is the `DaemonSet.yml` used in this lab:

```yaml
kind: DaemonSet           # Resource type
apiVersion: apps/v1       # DaemonSets live in the apps API group
metadata:
  name: nginx-daemonset   # Name of the DaemonSet object
  namespace: nginx        # Namespace for isolation

spec:
  selector:               # How the DaemonSet finds and owns its Pods
    matchLabels:
      app: nginx          # Must match template.metadata.labels

  template:               # Pod template — blueprint for every Pod this DS creates
    metadata:
      labels:
        app: nginx        # Must match selector.matchLabels

    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
          - containerPort: 80
```

### Every Field Explained

| Field | Required | Purpose |
|---|---|---|
| `kind` | ✅ | Resource type — tells Kubernetes what object to create |
| `apiVersion: apps/v1` | ✅ | DaemonSets are in the `apps` group, not core `v1` |
| `metadata.name` | ✅ | Unique name for the DaemonSet within the namespace |
| `metadata.namespace` | ❌ | Defaults to `default` if omitted |
| `spec.replicas` | ❌ | **Does not exist on DaemonSet** — pod count = node count |
| `spec.selector.matchLabels` | ✅ | Label query to identify which Pods this DS owns |
| `spec.template` | ✅ | The Pod blueprint — same structure as a standalone Pod spec |
| `spec.template.metadata.labels` | ✅ | Must match `selector.matchLabels` — this is the ownership contract |

> ⚠️ **Critical:** Just like a ReplicaSet, `selector.matchLabels` and `template.metadata.labels` **must match**. Kubernetes will reject the manifest if they don't. This is the #1 beginner mistake.

> ⚠️ **Note:** `metadata.name` inside `spec.template.metadata` is ignored by Kubernetes for DaemonSet pods — pod names are auto-generated as `<daemonset-name>-<hash>`. You can safely remove it.

---

## Hands-On Lab

### Prerequisites
- A running Kubernetes cluster (Minikube, Kind, EKS, GKE, or AKS)
- `kubectl` configured (`kubectl cluster-info` to verify)
- Namespace `nginx` already exists (from previous labs, or run Step 1)

---

### Step 1 — Create the Namespace

```bash
kubectl create namespace nginx
```
<img width="478" height="58" alt="image" src="https://github.com/user-attachments/assets/e824e8ce-5043-490b-8d28-68ab3b2111e1" />

---

### Step 2 — Apply the DaemonSet

```bash
kubectl apply -f DaemonSet.yml
```
<img width="400" height="45" alt="image" src="https://github.com/user-attachments/assets/8d1a3bfc-6122-4f37-9736-937ac84d03ec" />

---

### Step 3 — Verify Everything Was Created

```bash
# Check the DaemonSet
kubectl get daemonset -n nginx

# Check the Pods it created
kubectl get pods -n nginx -o wide
```

Expected output:
```
NAME               DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
nginx-daemonset    3         3         3       3            3           <none>          10s
```
<img width="909" height="60" alt="image" src="https://github.com/user-attachments/assets/7a6e92c9-4524-4c28-951e-7dc07e8d8bf3" />

- `DESIRED` → number of eligible nodes in your cluster
- `CURRENT` → pods that exist
- `READY` → pods passing readiness checks
- `-o wide` shows which node each pod is running on — you'll see one pod per node

<img width="1177" height="137" alt="image" src="https://github.com/user-attachments/assets/153cb1a1-1592-4434-8472-e120e47cc56e" /> </br>

> 💡 Notice there is no `REPLICAS` column like in a ReplicaSet. The desired count is driven entirely by your node count.

---

### Step 4 — Inspect the DaemonSet

```bash
# Full details — events, selector, pod template, node selector
kubectl describe daemonset nginx-daemonset -n nginx

# See which node each pod landed on
kubectl get pods -n nginx -o wide
```

In the `describe` output, pay attention to:
- `Node-Selector` — empty means all nodes are eligible
- `Tolerations` — what taints the pods can tolerate
- `Events` — shows pod creation events per node

<img width="923" height="594" alt="image" src="https://github.com/user-attachments/assets/471212b0-861e-4cfa-a2ac-00dfca8332ea" /> </br>

---

### Step 5 — Test Self-Healing

Delete one Pod manually and watch the DaemonSet bring it back:

```bash
# Get pod names and their nodes
kubectl get pods -n nginx -o wide

# Delete one pod (replace with your actual pod name)
kubectl delete pod nginx-daemonset-<hash> -n nginx

# Watch it get recreated on the same node
kubectl get pods -n nginx -w
```
<img width="1177" height="137" alt="image" src="https://github.com/user-attachments/assets/0dc066cd-a8f2-4339-a067-2dac5e2f1997" /> </br>

<img width="603" height="44" alt="image" src="https://github.com/user-attachments/assets/81590b9b-e872-46b8-8f47-5a6db1e2751d" /> </br>

<img width="713" height="80" alt="image" src="https://github.com/user-attachments/assets/78bfbc91-917f-4292-aba7-c780c95fd30f" /> </br>

> 💡 Unlike a ReplicaSet which reschedules the pod on *any* available node, a DaemonSet recreates the pod on the **same node** the deleted pod was on. The node still needs coverage.

---

### Step 6 — Observe Node-Level Behavior (Advanced)

Check that each pod is on a different node:

```bash
# List pods with node assignment
kubectl get pods -n nginx -o wide

# Confirm pod count matches node count
kubectl get nodes
```
<img width="1173" height="136" alt="image" src="https://github.com/user-attachments/assets/b9d597dc-05db-415e-b059-dfd8b80db0c9" />

<img width="676" height="119" alt="image" src="https://github.com/user-attachments/assets/1e44922d-5842-44b1-bacc-76e761897c91" />

The number of pods in the DaemonSet should exactly equal the number of `Ready` worker nodes.

---

### Step 7 — Simulate a New Node Joining (Minikube)

If you're using Minikube, you can add a node and watch the DaemonSet respond:

```bash
# Add a new node
minikube node add

# Watch the DaemonSet automatically schedule a pod on it
kubectl get pods -n nginx -o wide -w
```

> 💡 This is the core value of a DaemonSet. You don't need to do anything — the controller detects the new node and schedules a pod on it within seconds.

---

### Step 8 — Clean Up

```bash
# Delete the DaemonSet (also deletes all its pods)
kubectl delete daemonset nginx-daemonset -n nginx

# Or delete the namespace entirely
kubectl delete namespace nginx
```

---

## Node Selectors & Tolerations — Controlling Where Pods Land

By default, a DaemonSet schedules a pod on **every node**, including control-plane nodes (if they have the right tolerations). You can restrict this.

### Target a Subset of Nodes with `nodeSelector`

```yaml
spec:
  template:
    spec:
      nodeSelector:
        node-role: worker          # Only schedule on nodes with this label
```

Label your nodes first:
```bash
kubectl label node <node-name> node-role=worker
```

### Run on Control-Plane Nodes with `tolerations`

Control-plane nodes have a taint that prevents regular pods from being scheduled on them:
```
node-role.kubernetes.io/control-plane:NoSchedule
```

To allow your DaemonSet pod to run there, add a toleration:

```yaml
spec:
  template:
    spec:
      tolerations:
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
          effect: NoSchedule
```

```
Without toleration:                With toleration:
┌──────────────┐                   ┌──────────────┐
│ Control Plane│  ← Tainted        │ Control Plane│  ← Pod ✅ (tolerated)
│   No Pod ❌  │                   │   Pod ✅     │
└──────────────┘                   └──────────────┘
┌──────────────┐                   ┌──────────────┐
│   Worker 1   │                   │   Worker 1   │
│   Pod ✅     │                   │   Pod ✅     │
└──────────────┘                   └──────────────┘
┌──────────────┐                   ┌──────────────┐
│   Worker 2   │                   │   Worker 2   │
│   Pod ✅     │                   │   Pod ✅     │
└──────────────┘                   └──────────────┘
```

> 💡 This is exactly how Kubernetes system DaemonSets like `kube-proxy` and CNI plugins run on every node including control-plane — they have tolerations for all taints.

---

## DaemonSets in Production — What Changes

The `DaemonSet.yml` in this lab is intentionally minimal. In production, a DaemonSet for a log shipper or monitoring agent would look like this:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nginx-daemonset
  namespace: nginx
spec:
  updateStrategy:
    type: RollingUpdate          # Update pods one node at a time
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      tolerations:
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
          effect: NoSchedule
      containers:
        - name: nginx
          image: nginx:1.27.5    # Never use :latest in production
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "100m"
              memory: "64Mi"
            limits:
              cpu: "200m"
              memory: "128Mi"
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 10
            periodSeconds: 15
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 10
```

---

## Interview Q&A — Straight to the Point

**Q: What is a DaemonSet?**
A DaemonSet is a Kubernetes controller that ensures exactly one Pod runs on every eligible node in the cluster. When a new node joins, the DaemonSet automatically schedules a pod on it. When a node is removed, the pod is gone.

---

**Q: What is the difference between a DaemonSet and a ReplicaSet?**
A ReplicaSet maintains a fixed number of pods and lets the scheduler decide placement. A DaemonSet ensures one pod per node — the pod count is determined by the number of nodes, not a `replicas` field. Use a ReplicaSet for stateless app replicas; use a DaemonSet for node-level agents.

---

**Q: Can you set `replicas` on a DaemonSet?**
No. `spec.replicas` does not exist on a DaemonSet. The number of pods is always equal to the number of eligible nodes.

---

**Q: What happens when a new node joins the cluster?**
The DaemonSet controller detects the new node and automatically creates a pod on it. No manual intervention needed.

---

**Q: What happens when a node is removed from the cluster?**
The pod running on that node is simply gone. The DaemonSet doesn't reschedule it elsewhere — that pod's job was tied to that specific node.

---

**Q: How does a DaemonSet run pods on control-plane nodes?**
Control-plane nodes have a `NoSchedule` taint. To run a DaemonSet pod there, add a matching toleration to the pod spec. This is how system components like `kube-proxy` run on every node.

---

**Q: What is the default update strategy for a DaemonSet?**
`RollingUpdate` — it updates pods one node at a time. The other option is `OnDelete`, where pods are only updated when you manually delete them.

---

**Q: How does a DaemonSet differ from running a static pod?**
Static pods are defined as files on a node's filesystem and managed by the kubelet directly — not by the API server. DaemonSets are managed by the Kubernetes control plane, support rolling updates, and are visible via `kubectl`. Static pods are used for bootstrapping the control plane itself (e.g., `kube-apiserver`, `etcd`).

---

**Q: What are real-world use cases for DaemonSets?**
- Log collection agents: Fluentd, Filebeat, Logstash
- Node monitoring: Prometheus Node Exporter, Datadog agent, Dynatrace
- Network plugins (CNI): Calico, Weave, Cilium
- Storage daemons: Ceph, GlusterFS
- Security agents: Falco, Sysdig

---

**Q: Can a DaemonSet target only specific nodes?**
Yes — using `nodeSelector` in the pod spec to match node labels, or using `affinity` rules for more complex targeting. Pods will only be scheduled on nodes that match.

---

**Q: What happens if you delete a DaemonSet?**
By default, all pods owned by the DaemonSet are also deleted (cascade delete). Use `--cascade=orphan` to delete the DaemonSet object while leaving the pods running as unmanaged pods.

---

## Common Mistakes & Gotchas

| Mistake | Why It's Wrong | Fix |
|---|---|---|
| Trying to set `spec.replicas` | Field doesn't exist on DaemonSet — pod count = node count | Remove `replicas` entirely |
| `selector.matchLabels` ≠ `template.metadata.labels` | Kubernetes rejects the manifest | They must be identical |
| Using `image: nginx:latest` | Image can silently change on pod restart | Pin to a specific version: `nginx:1.27.5` |
| Expecting pods on control-plane nodes without tolerations | Control-plane nodes are tainted `NoSchedule` | Add the appropriate toleration |
| No `resources` requests/limits | Pods can starve other node workloads | Always set both in production |
| Forgetting `-n <namespace>` in kubectl commands | Commands silently operate on `default` namespace | Always pass `-n <namespace>` |
| Confusing DaemonSet pod count with replicas | Pod count changes as nodes are added/removed | Understand it's node-driven, not replica-driven |
| Not setting `updateStrategy` | Defaults to `RollingUpdate` which is fine, but being explicit is better | Add `spec.updateStrategy.type: RollingUpdate` |

---

## Files in This Lab

```
05-daemonset/
├── DaemonSet.yml  ← Minimal nginx DaemonSet manifest (intentionally simple)
└── README.md      ← This file
```

---

## ✍️ Author

**[Himanshu Kumar](https://www.linkedin.com/in/h1manshu-kumar/)** - Learning by building, documenting, and sharing 🚀

---
<div align="center">

**Built while learning Kubernetes hands-on.**
*The best way to understand DaemonSets is to watch a pod appear the moment a new node joins your cluster.*

</div>
