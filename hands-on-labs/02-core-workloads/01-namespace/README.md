# 🗂️ Kubernetes Namespaces — Deep Dive

> **Lab Series:** Core Workloads → 01 Namespaces
> **Difficulty:** Beginner → Intermediate
> **Estimated Time:** 20–30 minutes
> **Focus:** Master logical cluster isolation — from theory to hands-on to interview-ready

---

## 📌 Table of Contents

1. [What is a Namespace?](#what-is-a-namespace)
2. [The Problem Namespaces Solve](#the-problem-namespaces-solve)
3. [Default Namespaces — What Ships with Every Cluster](#default-namespaces--what-ships-with-every-cluster)
4. [Namespace Manifest — Field by Field](#namespace-manifest--field-by-field)
5. [Hands-On Lab](#hands-on-lab)
6. [Namespace-Scoped vs Cluster-Scoped Resources](#namespace-scoped-vs-cluster-scoped-resources)
7. [Cross-Namespace Communication — DNS Deep Dive](#cross-namespace-communication--dns-deep-dive)
8. [Namespaces in Production — ResourceQuota & LimitRange](#namespaces-in-production--resourcequota--limitrange)
9. [RBAC + Namespaces — Access Isolation](#rbac--namespaces--access-isolation)
10. [Interview Q&A — Straight to the Point](#interview-qa--straight-to-the-point)
11. [Common Mistakes & Gotchas](#common-mistakes--gotchas)
12. [What's Next](#whats-next)

---

## What is a Namespace?

A **Namespace** is a mechanism for **logically partitioning a single Kubernetes cluster** into multiple virtual clusters.

Every Kubernetes resource you create — Pods, Deployments, Services, ConfigMaps — lives inside a namespace. Namespaces give you:

| Capability | What It Enables |
|---|---|
| **Isolation** | Resources in different namespaces don't conflict with each other |
| **Access Control** | RBAC policies can be scoped to a specific namespace |
| **Resource Quotas** | Limit CPU, memory, and object counts per namespace |
| **Organizational clarity** | Separate teams, environments, or applications cleanly |

> 💡 **Mental Model:** Think of a Kubernetes cluster as an **office building**. Namespaces are the **floors**. Each floor has its own rooms (resources), its own access cards (RBAC), and its own utility limits (ResourceQuota). But the building's shared infrastructure — elevators, power, plumbing — is cluster-scoped and available to everyone.

---

## The Problem Namespaces Solve

Without namespaces, a single cluster shared by multiple teams or environments becomes chaotic:

```
WITHOUT Namespaces:                WITH Namespaces:
┌──────────────────────────┐       ┌──────────────────────────────────────┐
│  Cluster                 │       │  Cluster                             │
│                          │       │  ┌────────────┐  ┌────────────┐      │
│  backend (team A)        │       │  │    dev     │  │   prod     │      │
│  backend (team B) ← ❌   │  →    │  │  backend   │  │  backend   │      │
│  CONFLICT                │       │  │  frontend  │  │  frontend  │      │
│                          │       │  └────────────┘  └────────────┘      │
│  No access boundaries    │       │  ┌────────────┐                      │
│  No resource limits      │       │  │  staging   │  Isolated · Governed │
└──────────────────────────┘       │  │  backend   │  No naming conflicts │
                                   │  └────────────┘                      │
                                   └──────────────────────────────────────┘
```

**Real-world use cases:**
- `dev`, `staging`, `prod` environments on the same cluster
- Per-team isolation — `team-payments`, `team-auth`, `team-platform`
- Per-application isolation — `monitoring`, `logging`, `ingress`

---

## Default Namespaces — What Ships with Every Cluster

When you spin up any Kubernetes cluster, four namespaces exist by default:

```bash
kubectl get namespaces
```

```
NAME              STATUS   AGE
default           Active   1d
kube-system       Active   1d
kube-public       Active   1d
kube-node-lease   Active   1d
```

| Namespace | Purpose | Touch it? |
|---|---|---|
| `default` | Where resources land when no `-n` flag is specified | Only for quick experiments |
| `kube-system` | Core cluster components — DNS (CoreDNS), scheduler, controller-manager, kube-proxy | Never manually deploy here |
| `kube-public` | Readable by all users including unauthenticated ones. Rarely used | Leave it alone |
| `kube-node-lease` | Holds `Lease` objects — one per node. Kubelet updates these to signal the node is alive | Never touch |

> ⚠️ **Never deploy your workloads into `kube-system`.** It runs the cluster's own control plane components. Polluting it with application workloads is a common beginner mistake that can destabilize the cluster.

---

## Namespace Manifest — Field by Field

This is the `namespace.yml` used in this lab:

```yaml
kind: Namespace     # Kubernetes resource type
apiVersion: v1      # Namespaces belong to the core API group (v1)
metadata:
  name: nginx       # The namespace name — must be unique across the cluster
```

### Every Field Explained

| Field | Required | Purpose |
|---|---|---|
| `kind` | ✅ | Tells Kubernetes this is a Namespace object |
| `apiVersion` | ✅ | Namespaces are a core `v1` resource |
| `metadata.name` | ✅ | Unique name for the namespace — DNS-compatible (lowercase, hyphens only) |
| `metadata.labels` | ❌ | Used to select namespaces in NetworkPolicy and other policies |

> 💡 Namespace names must follow DNS label rules: lowercase alphanumeric characters or hyphens, must start and end with alphanumeric. `my-app` ✅ `My_App` ❌

---

## Hands-On Lab

### Prerequisites
- A running Kubernetes cluster (Minikube, Kind, EKS, GKE, or AKS)
- `kubectl` configured and pointing to your cluster (`kubectl cluster-info` to verify)

---

### Step 1 — Create the Namespace from Manifest

```bash
kubectl apply -f namespace.yml
```

Or imperatively (no manifest needed):

```bash
kubectl create namespace nginx
```

---

### Step 2 — Verify It Exists

```bash
kubectl get namespaces
```

You should see `nginx` in the list with `STATUS: Active`.

---

### Step 3 — Deploy Resources into the Namespace

Any resource can be scoped to a namespace using the `-n` flag or by setting `namespace` in the manifest's `metadata`:

```bash
# Deploy a pod into the nginx namespace
kubectl run nginx-pod --image=nginx:latest -n nginx

# List pods in the nginx namespace
kubectl get pods -n nginx

# List ALL pods across ALL namespaces
kubectl get pods --all-namespaces
# or shorthand:
kubectl get pods -A
```

---

### Step 4 — Switch Your Default Namespace

Tired of typing `-n nginx` on every command? Set it as your default context namespace:

```bash
kubectl config set-context --current --namespace=nginx
```

Now all `kubectl` commands target `nginx` by default. Verify:

```bash
kubectl config view --minify | grep namespace
```

Reset back to default:

```bash
kubectl config set-context --current --namespace=default
```

---

### Step 5 — Inspect the Namespace

```bash
# Full details — labels, annotations, resource quotas
kubectl describe namespace nginx
```

---

### Step 6 — Clean Up

```bash
# Delete the namespace — this deletes EVERYTHING inside it
kubectl delete namespace nginx
```

> ⚠️ `kubectl delete namespace` is **irreversible**. It cascades — every Pod, Deployment, Service, ConfigMap, and Secret inside the namespace is permanently deleted. There is no undo.

---

## Namespace-Scoped vs Cluster-Scoped Resources

Not all Kubernetes resources live inside a namespace. This distinction is critical.

```
┌─────────────────────────────────────────────────────────┐
│                      CLUSTER                            │
│                                                         │
│   Cluster-Scoped (no namespace):                        │
│   Node · PersistentVolume · ClusterRole                 │
│   ClusterRoleBinding · StorageClass · Namespace itself  │
│                                                         │
│  ┌──────────────────┐    ┌──────────────────┐           │
│  │   Namespace: dev │    │ Namespace: prod  │           │
│  │                  │    │                  │           │
│  │  Pod             │    │  Pod             │           │
│  │  Deployment      │    │  Deployment      │           │
│  │  Service         │    │  Service         │           │
│  │  ConfigMap       │    │  ConfigMap       │           │
│  │  Secret          │    │  Secret          │           │
│  │  ServiceAccount  │    │  ServiceAccount  │           │
│  └──────────────────┘    └──────────────────┘           │
└─────────────────────────────────────────────────────────┘
```

Check whether any resource is namespace-scoped:

```bash
# List all namespace-scoped resources
kubectl api-resources --namespaced=true

# List all cluster-scoped resources
kubectl api-resources --namespaced=false
```

---

## Cross-Namespace Communication — DNS Deep Dive

Namespaces do **not** provide network isolation by default. Pods in different namespaces can communicate freely.

Kubernetes DNS follows this format for Services:

```
<service-name>.<namespace>.svc.cluster.local
```

### Example

```
┌─────────────────────┐          ┌─────────────────────┐
│  Namespace: frontend│          │  Namespace: backend  │
│                     │          │                      │
│  Pod: web-app       │─────────▶│  Service: api-svc    │
│                     │          │  (port 8080)         │
└─────────────────────┘          └─────────────────────┘

web-app calls: http://api-svc.backend.svc.cluster.local:8080
```

| DNS Format | When to Use |
|---|---|
| `api-svc` | Within the same namespace |
| `api-svc.backend` | Short form across namespaces |
| `api-svc.backend.svc.cluster.local` | Fully qualified — always works, use in production configs |

> 💡 Within the same namespace, you can use just the service name. Across namespaces, you must include the namespace in the DNS name.

---

## Namespaces in Production — ResourceQuota & LimitRange

This is where namespaces go from organizational tool to **enforcement mechanism**.

### ResourceQuota — Limit What a Namespace Can Consume

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: nginx
spec:
  hard:
    pods: "10"                  # Max 10 pods in this namespace
    requests.cpu: "4"           # Total CPU requests capped at 4 cores
    requests.memory: "8Gi"      # Total memory requests capped at 8Gi
    limits.cpu: "8"             # Total CPU limits capped at 8 cores
    limits.memory: "16Gi"       # Total memory limits capped at 16Gi
    configmaps: "20"            # Max 20 ConfigMaps
    secrets: "20"               # Max 20 Secrets
    services: "10"              # Max 10 Services
```

Once a `ResourceQuota` is applied, **every Pod in that namespace must declare resource requests and limits** — otherwise the API server rejects the Pod.

---

### LimitRange — Set Defaults and Bounds Per Container

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: container-limits
  namespace: nginx
spec:
  limits:
  - type: Container
    default:                    # Applied if container doesn't specify limits
      cpu: "500m"
      memory: "256Mi"
    defaultRequest:             # Applied if container doesn't specify requests
      cpu: "100m"
      memory: "128Mi"
    max:                        # Hard ceiling — container cannot exceed this
      cpu: "2"
      memory: "1Gi"
    min:                        # Floor — container must request at least this
      cpu: "50m"
      memory: "64Mi"
```

> 💡 `LimitRange` is a safety net. It ensures no single container in the namespace can consume unbounded resources, even if the developer forgets to set limits.

---

## RBAC + Namespaces — Access Isolation

Namespaces are the boundary for RBAC scoping. A `Role` and `RoleBinding` are namespace-scoped — they only grant permissions within one namespace.

```
┌──────────────────────────────────────────────────────┐
│  Cluster                                             │
│                                                      │
│  ┌─────────────────┐      ┌─────────────────┐        │
│  │  Namespace: dev │      │ Namespace: prod │        │
│  │                 │      │                 │        │
│  │  Role: dev-rw   │      │  Role: prod-ro  │        │
│  │  (read+write)   │      │  (read only)    │        │
│  │       │         │      │       │         │        │
│  │  RoleBinding    │      │  RoleBinding    │        │
│  │  → alice        │      │  → alice        │        │
│  └─────────────────┘      └─────────────────┘        │
│                                                      │
│  alice can write to dev, but only read prod ✅        │
└──────────────────────────────────────────────────────┘
```

- `Role` + `RoleBinding` → scoped to a single namespace
- `ClusterRole` + `ClusterRoleBinding` → applies across the entire cluster

---

## Interview Q&A — Straight to the Point

**Q: What is a Kubernetes Namespace?**
A logical partition of a cluster. It provides isolation for resources, scoping for RBAC, and boundaries for ResourceQuotas. Multiple teams or environments can share one cluster without interfering with each other.

---

**Q: Do Namespaces provide network isolation?**
No. By default, Pods in different namespaces can communicate freely. Namespaces are a logical boundary, not a network boundary. To enforce network isolation between namespaces, you need `NetworkPolicy` resources.

---

**Q: What are the four default namespaces and what do they do?**
- `default` — where resources go when no namespace is specified
- `kube-system` — core cluster components (DNS, scheduler, controller-manager)
- `kube-public` — publicly readable, rarely used
- `kube-node-lease` — node heartbeat Lease objects, used for node health detection

---

**Q: What is the difference between a `Role` and a `ClusterRole`?**
A `Role` is namespace-scoped — it grants permissions only within one namespace. A `ClusterRole` is cluster-scoped — it grants permissions across all namespaces or on cluster-scoped resources like Nodes.

---

**Q: What happens when you delete a namespace?**
Everything inside it is permanently deleted — all Pods, Deployments, Services, ConfigMaps, Secrets, and other namespace-scoped resources. It cascades and is irreversible.

---

**Q: How does DNS work across namespaces?**
Kubernetes DNS resolves services using the format `<service>.<namespace>.svc.cluster.local`. Within the same namespace, just the service name works. Across namespaces, you must include the namespace in the DNS name.

---

**Q: What is a ResourceQuota?**
A ResourceQuota is a namespace-level object that caps the total amount of CPU, memory, and object counts (pods, services, secrets, etc.) that can be consumed within that namespace. Once applied, all Pods must declare resource requests and limits or they will be rejected.

---

**Q: What is a LimitRange?**
A LimitRange sets default, minimum, and maximum resource values for containers in a namespace. It acts as a safety net — if a developer doesn't specify resource limits, LimitRange injects defaults automatically.

---

**Q: Is a Namespace itself namespace-scoped or cluster-scoped?**
Cluster-scoped. A Namespace is a top-level cluster resource — it doesn't belong to any namespace. You can verify with `kubectl api-resources --namespaced=false`.

---

**Q: Can two resources of the same type have the same name in different namespaces?**
Yes. That's one of the core benefits. Two teams can both have a `Deployment` named `backend` — one in `team-a` namespace and one in `team-b` namespace — with no conflict.

---

## Common Mistakes & Gotchas

| Mistake | Why It's Wrong | Fix |
|---|---|---|
| Deploying everything into `default` | No isolation, no quotas, hard to manage at scale | Always create dedicated namespaces |
| Assuming namespaces = network isolation | Pods across namespaces can still talk freely | Use `NetworkPolicy` to restrict traffic |
| Deleting a namespace carelessly | Cascades — deletes everything inside permanently | Double-check with `kubectl get all -n <ns>` before deleting |
| Not setting ResourceQuota | One team can starve the entire cluster | Apply quotas to every non-system namespace |
| Using `kube-system` for app workloads | Risks destabilizing core cluster components | Always use dedicated namespaces |
| Forgetting `-n` flag | Commands silently target `default` namespace | Set context namespace or always use `-n` |
| Hardcoding short DNS names across namespaces | Breaks when services move namespaces | Use fully qualified DNS: `svc.ns.svc.cluster.local` |

---

## What's Next

| Lab | Topic | Why It Matters |
|---|---|---|
| [02 — Pods](../02-pods/) | The smallest deployable unit | Everything runs inside a Pod |
| [03 — ReplicaSets](../03-replicasets/) | Ensure N copies of a Pod always run | Foundation of self-healing |
| [04 — Deployments](../04-deployments/) | Rolling updates, rollbacks | How you run Pods in production |
| [05 — Services](../05-services/) | Expose Pods to network traffic | Pods need Services to be reachable |

---

## Files in This Lab

```
01-namespace/
├── namespace.yml   ← Minimal namespace manifest
└── README.md       ← This file
```

---

<div align="center">

**Built while learning Kubernetes hands-on.**
*Namespaces are the first thing you configure and the last thing you think about — until something goes wrong.*

</div>
