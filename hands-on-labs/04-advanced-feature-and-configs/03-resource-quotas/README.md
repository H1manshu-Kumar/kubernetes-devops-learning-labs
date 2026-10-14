# 🔒 Kubernetes Resource Quotas — Deep Dive

> **Lab Series:** Advanced Features & Configs → 03 Resource Quotas    
> **Difficulty:** Intermediate    
> **Estimated Time:** 30–45 minutes    
> **Focus:** Understand how ResourceQuota and LimitRange enforce fair resource usage across namespaces    

---

## 📌 Table of Contents

1. [Why Resource Quotas Exist](#why-resource-quotas-exist)
2. [ResourceQuota vs LimitRange — Know the Difference](#resourcequota-vs-limitrange--know-the-difference)
3. [How Resource Quotas Work — The Admission Control Flow](#how-resource-quotas-work--the-admission-control-flow)
4. [Manifest — Field by Field Breakdown](#manifest--field-by-field-breakdown)
5. [Hands-On Lab](#hands-on-lab)
6. [What Happens When a Quota Is Exceeded](#what-happens-when-a-quota-is-exceeded)
7. [Resource Quotas in Production — What Changes](#resource-quotas-in-production--what-changes)
8. [Interview Q&A — Straight to the Point](#interview-qa--straight-to-the-point)
9. [Common Mistakes & Gotchas](#common-mistakes--gotchas)
10. [What's Next](#whats-next)

---

## Why Resource Quotas Exist

In a shared Kubernetes cluster, multiple teams or applications run in different namespaces. Without any guardrails, a single namespace can consume all cluster CPU and memory, starving every other workload.

Resource Quotas solve this by enforcing **hard limits on total resource consumption per namespace**.

| Scenario | Without Quota | With Quota |
|---|---|---|
| One team deploys 100 pods | ✅ Allowed — starves others | ❌ Blocked at pod count limit |
| A pod requests 64Gi memory | ✅ Allowed — node may OOM | ❌ Blocked if namespace limit exceeded |
| No `resources` set on a pod | ✅ Allowed — unbounded usage | ❌ Blocked (unless LimitRange provides defaults) |
| Team A uses all cluster CPU | ✅ Allowed | ❌ Blocked by `requests.cpu` quota |

> 💡 **The Rule:** ResourceQuota operates at the **namespace level** — it caps the *sum* of all resource requests/limits across every pod in that namespace. LimitRange operates at the **pod/container level** — it sets defaults and per-container min/max bounds.

---

## ResourceQuota vs LimitRange — Know the Difference

These two objects are often confused but serve distinct purposes.

| Feature | ResourceQuota | LimitRange |
|---|---|---|
| Scope | Entire namespace (aggregate) | Individual pod / container |
| What it limits | Total sum of requests/limits across all pods | Per-container min, max, and defaults |
| Blocks pod creation? | ✅ Yes, if namespace total would be exceeded | ✅ Yes, if a container violates min/max |
| Sets defaults? | ❌ No | ✅ Yes — injects default requests/limits |
| Object count limits? | ✅ Yes (pods, services, PVCs, etc.) | ❌ No |
| Required together? | Not required, but recommended together | Not required, but recommended together |

```
┌──────────────────────────────────────────────────────────────────┐
│                        Namespace: quota-demo                     │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐     │
│  │  ResourceQuota (namespace ceiling)                      │     │
│  │  requests.cpu: 500m total  |  pods: 5 max               │     │
│  └─────────────────────────────────────────────────────────┘     │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐     │
│  │  LimitRange (per-container guardrails)                  │     │
│  │  default request: 100m cpu  |  max: 500m cpu            │     │
│  └─────────────────────────────────────────────────────────┘     │
│                                                                  │
│   ┌──────────┐   ┌──────────┐   ┌──────────┐                    │
│   │  Pod 1   │   │  Pod 2   │   │  Pod 3   │  ← each must fit   │
│   │ 100m cpu │   │ 100m cpu │   │ 100m cpu │    LimitRange AND   │
│   └──────────┘   └──────────┘   └──────────┘    not exceed quota │
└──────────────────────────────────────────────────────────────────┘
```

> ⚠️ **Critical Interaction:** When a ResourceQuota exists in a namespace, **every pod must have explicit `resources.requests` and `resources.limits`** — or the pod will be rejected. This is why LimitRange is almost always deployed alongside ResourceQuota: it injects defaults so pods without explicit resources still get admitted.

---

## How Resource Quotas Work — The Admission Control Flow

ResourceQuota is enforced by the `ResourceQuota` admission controller, which runs as part of the API server's admission chain.

```
kubectl apply -f resource-quota.yml
        │
        ▼
┌───────────────────┐
│    API Server     │  ← Receives the request
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│  LimitRange       │  ← Injects default requests/limits if missing
│  Admission Plugin │    Rejects if container violates min/max
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│  ResourceQuota    │  ← Checks: would this pod push namespace total
│  Admission Plugin │    over any hard limit?
└────────┬──────────┘
         │
    ┌────┴────┐
    ▼         ▼
 ADMIT      REJECT (403 Forbidden — quota exceeded)
    │
    ▼
┌───────────────────┐
│  etcd             │  ← Desired state stored
└───────────────────┘
         │
         ▼
┌───────────────────┐
│  Scheduler        │  ← Assigns pod to a node
└───────────────────┘
         │
         ▼
┌───────────────────┐
│  Kubelet          │  ← Starts the container
└───────────────────┘
```

> 💡 Quota is checked at **admission time** — before the pod is scheduled. If the namespace is at its limit, the pod is rejected immediately with a clear error message. No pod is created, no scheduling happens.

---

## Manifest — Field by Field Breakdown

This is the `resource-quota.yml` used in this lab. It contains four objects in one file.

### 1. Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: quota-demo
```

### 2. ResourceQuota

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: demo-quota
  namespace: quota-demo   # Quota is always scoped to a namespace
spec:
  hard:                   # "hard" = enforced ceiling, no exceptions
    # Compute resource totals across all pods in the namespace
    requests.cpu: "500m"       # Sum of all container cpu requests ≤ 500m
    requests.memory: "512Mi"   # Sum of all container memory requests ≤ 512Mi
    limits.cpu: "1"            # Sum of all container cpu limits ≤ 1 core
    limits.memory: "1Gi"       # Sum of all container memory limits ≤ 1Gi
    # Object count limits
    pods: "5"                  # Max 5 pods in this namespace
    services: "3"
    configmaps: "5"
    secrets: "5"
    persistentvolumeclaims: "3"
```

### 3. LimitRange

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: demo-limitrange
  namespace: quota-demo
spec:
  limits:
    - type: Container           # Applies to every container individually
      default:                  # Injected as limits if not specified
        cpu: "200m"
        memory: "256Mi"
      defaultRequest:           # Injected as requests if not specified
        cpu: "100m"
        memory: "128Mi"
      max:                      # Container cannot exceed these
        cpu: "500m"
        memory: "512Mi"
      min:                      # Container must request at least these
        cpu: "50m"
        memory: "64Mi"
```

### 4. Compliant Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: quota-compliant-pod
  namespace: quota-demo
spec:
  containers:
    - name: nginx
      image: nginx:1.25.3
      resources:
        requests:
          cpu: "100m"
          memory: "128Mi"
        limits:
          cpu: "200m"
          memory: "256Mi"
```

### Every ResourceQuota Field Explained

| Field | Required | Purpose |
|---|---|---|
| `kind: ResourceQuota` | ✅ | Resource type |
| `apiVersion: v1` | ✅ | ResourceQuota is in the core `v1` group |
| `metadata.namespace` | ✅ | Quota is always namespace-scoped |
| `spec.hard` | ✅ | Map of resource names to hard limits |
| `requests.cpu` / `requests.memory` | ❌ | Caps the sum of all pod CPU/memory requests |
| `limits.cpu` / `limits.memory` | ❌ | Caps the sum of all pod CPU/memory limits |
| `pods` | ❌ | Max number of pods allowed in the namespace |
| `services`, `configmaps`, etc. | ❌ | Object count limits for other resource types |

### Every LimitRange Field Explained

| Field | Purpose |
|---|---|
| `type: Container` | Applies limits to each container (also supports `Pod`, `PersistentVolumeClaim`) |
| `default` | Injected as `limits` if a container doesn't specify them |
| `defaultRequest` | Injected as `requests` if a container doesn't specify them |
| `max` | Hard ceiling — container is rejected if it requests more |
| `min` | Hard floor — container is rejected if it requests less |

---

## Hands-On Lab

### Prerequisites
- A running Kubernetes cluster (Minikube, Kind, EKS, GKE, or AKS)
- `kubectl` configured (`kubectl cluster-info` to verify)

---

### Step 1 — Apply the Full Manifest

```bash
kubectl apply -f resource-quota.yml
```

Expected output:
```
namespace/quota-demo created
resourcequota/demo-quota created
limitrange/demo-limitrange created
pod/quota-compliant-pod created
```

---

### Step 2 — Inspect the ResourceQuota

```bash
kubectl get resourcequota -n quota-demo
kubectl describe resourcequota demo-quota -n quota-demo
```

The `describe` output shows current usage vs hard limits:
```
Name:                   demo-quota
Namespace:              quota-demo
Resource                Used    Hard
--------                ----    ----
configmaps              1       5
limits.cpu              200m    1
limits.memory           256Mi   1Gi
pods                    1       5
requests.cpu            100m    500m
requests.memory         128Mi   512Mi
...
```

> 💡 Watch the `Used` column — it updates in real time as you create and delete pods.

---

### Step 3 — Inspect the LimitRange

```bash
kubectl describe limitrange demo-limitrange -n quota-demo
```

```
Name:       demo-limitrange
Namespace:  quota-demo
Type        Resource  Min   Max    Default Request  Default Limit
----        --------  ---   ---    ---------------  -------------
Container   cpu       50m   500m   100m             200m
Container   memory    64Mi  512Mi  128Mi            256Mi
```

---

### Step 4 — Test: Pod Without Explicit Resources (LimitRange Injects Defaults)

```bash
kubectl run no-resources-pod --image=nginx:1.25.3 -n quota-demo
kubectl get pod no-resources-pod -n quota-demo -o jsonpath='{.spec.containers[0].resources}'
```

Even though you didn't specify `resources`, the LimitRange injected defaults. The pod was admitted because it now has valid requests/limits.

---

### Step 5 — Test: Exceed the Pod Count Quota

```bash
# Create pods until the quota is hit (quota allows 5 pods)
for i in 1 2 3 4 5; do
  kubectl run test-pod-$i --image=nginx:1.25.3 -n quota-demo
done
```

The 6th pod (you already have `quota-compliant-pod` and `no-resources-pod`) will be rejected:
```
Error from server (Forbidden): pods "test-pod-5" is forbidden:
exceeded quota: demo-quota, requested: pods=1, used: pods=5, limited: pods=5
```

> 💡 This is the core behavior of ResourceQuota — hard enforcement at admission time with a clear error message.

---

### Step 6 — Test: Exceed the CPU Request Quota

```bash
# Try to create a pod requesting more CPU than the namespace has left
kubectl run cpu-hog --image=nginx:1.25.3 -n quota-demo \
  --overrides='{"spec":{"containers":[{"name":"nginx","image":"nginx:1.25.3","resources":{"requests":{"cpu":"450m"},"limits":{"cpu":"500m"}}}]}}'
```

If the namespace has already consumed enough CPU requests, this will be rejected:
```
Error from server (Forbidden): pods "cpu-hog" is forbidden:
exceeded quota: demo-quota, requested: requests.cpu=450m, used: requests.cpu=100m, limited: requests.cpu=500m
```

---

### Step 7 — Test: Violate LimitRange Max

```bash
# Try to create a container exceeding the LimitRange max cpu of 500m
kubectl run limit-violator --image=nginx:1.25.3 -n quota-demo \
  --overrides='{"spec":{"containers":[{"name":"nginx","image":"nginx:1.25.3","resources":{"requests":{"cpu":"600m"},"limits":{"cpu":"600m"}}}]}}'
```

```
Error from server (Forbidden): pods "limit-violator" is forbidden:
[maximum cpu usage per Container is 500m, but limit is 600m]
```

> 💡 Notice this error comes from LimitRange, not ResourceQuota — it's rejected before quota is even checked.

---

### Step 8 — Check Quota Usage at Any Time

```bash
kubectl get resourcequota demo-quota -n quota-demo -o yaml
```

The `status.used` section shows real-time consumption:
```yaml
status:
  hard:
    pods: "5"
    requests.cpu: 500m
  used:
    pods: "3"
    requests.cpu: 300m
```

---

### Step 9 — Clean Up

```bash
kubectl delete namespace quota-demo
```

Deleting the namespace removes the ResourceQuota, LimitRange, and all pods inside it.

---

## What Happens When a Quota Is Exceeded

This is the most important operational detail to understand.

```
Pod creation request
        │
        ▼
ResourceQuota admission check:
  would (used + requested) > hard limit?
        │
   YES  │  NO
        │   └──→ Pod admitted ✅
        ▼
Pod REJECTED ❌
HTTP 403 Forbidden
"exceeded quota: <name>, requested: X, used: Y, limited: Z"
```

**Key behaviors:**

- The rejection is **immediate and synchronous** — the `kubectl` command returns an error right away
- The error message tells you exactly which quota was exceeded, what was requested, what is currently used, and what the limit is
- **Existing pods are never evicted** when a quota is added or tightened — quota only affects new pod creation
- Quota is tracked by `requests`, not by actual usage — a pod requesting 100m CPU counts as 100m against the quota even if it's only using 10m

---

## Resource Quotas in Production — What Changes

The manifest in this lab is intentionally focused on learning. In production:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-alpha-quota
  namespace: team-alpha
spec:
  hard:
    # Compute
    requests.cpu: "4"
    requests.memory: "8Gi"
    limits.cpu: "8"
    limits.memory: "16Gi"
    # Storage
    requests.storage: "50Gi"
    persistentvolumeclaims: "10"
    # Objects
    pods: "20"
    services: "10"
    services.loadbalancers: "2"    # LoadBalancers are expensive — limit them
    services.nodeports: "0"        # Disable NodePort entirely
    secrets: "20"
    configmaps: "20"
```

**Production considerations:**

- Set quotas per team namespace — each team gets a fair share of cluster resources
- Always pair ResourceQuota with LimitRange — without LimitRange, any pod without explicit resources will be rejected when a quota exists
- Use `requests.storage` to prevent PVC sprawl
- Set `services.loadbalancers: "0"` or a low number — cloud load balancers cost money
- Monitor quota usage with `kubectl describe resourcequota` or Prometheus `kube_resourcequota` metrics
- Quota changes take effect immediately for new pods — existing pods are unaffected

---

## Interview Q&A — Straight to the Point

**Q: What is a ResourceQuota?**
A ResourceQuota is a Kubernetes object that enforces hard limits on the total resource consumption within a namespace — covering compute resources (CPU, memory), storage, and object counts (pods, services, etc.). It is enforced at admission time by the ResourceQuota admission controller.

---

**Q: What is the difference between ResourceQuota and LimitRange?**
ResourceQuota caps the *aggregate* usage across the entire namespace. LimitRange sets *per-container* defaults, minimums, and maximums. They work together: LimitRange injects defaults so pods without explicit resources can still be admitted when a ResourceQuota is active.

---

**Q: What happens if a pod doesn't have `resources` set and a ResourceQuota exists?**
The pod is rejected with a Forbidden error. When a ResourceQuota is active, every pod must have explicit `resources.requests` and `resources.limits`. This is why LimitRange is deployed alongside ResourceQuota — it injects defaults automatically.

---

**Q: Does ResourceQuota evict existing pods when you lower the quota?**
No. Quota is only enforced at admission time (pod creation). Existing pods are never evicted when a quota is added or reduced. The quota will simply prevent new pods from being created until usage drops below the new limit.

---

**Q: Is quota tracked by actual usage or by requested resources?**
By requested resources (`requests.cpu`, `requests.memory`). A pod requesting 500m CPU counts as 500m against the quota regardless of how much CPU it actually consumes. This is intentional — it makes capacity planning predictable.

---

**Q: What is the difference between `requests.cpu` and `limits.cpu` in a ResourceQuota?**
`requests.cpu` caps the sum of all container `resources.requests.cpu` values. `limits.cpu` caps the sum of all container `resources.limits.cpu` values. You can set one, both, or neither. Setting both gives you tighter control over both guaranteed and burstable resource usage.

---

**Q: Can you apply a ResourceQuota to all namespaces at once?**
No. ResourceQuota is namespace-scoped — you must create one per namespace. For cluster-wide policy enforcement, use a tool like OPA/Gatekeeper or Kyverno, or use Hierarchical Namespace Controller (HNC) to propagate quotas.

---

**Q: What object count resources can a ResourceQuota limit?**
`pods`, `services`, `replicationcontrollers`, `resourcequotas`, `secrets`, `configmaps`, `persistentvolumeclaims`, `services.nodeports`, `services.loadbalancers`, and more. Run `kubectl explain resourcequota.spec.hard` for the full list.

---

**Q: What happens to a ResourceQuota if you delete the namespace?**
It is deleted along with everything else in the namespace. ResourceQuota is a namespaced resource — it cannot exist without its namespace.

---

**Q: How do you check current quota usage?**
`kubectl describe resourcequota <name> -n <namespace>` shows a side-by-side view of `Used` vs `Hard` for every tracked resource. You can also use `kubectl get resourcequota -n <namespace> -o yaml` to see the `status.used` field programmatically.

---

## Common Mistakes & Gotchas

| Mistake | Why It's Wrong | Fix |
|---|---|---|
| Adding ResourceQuota without LimitRange | Pods without explicit `resources` are immediately rejected | Always deploy LimitRange alongside ResourceQuota |
| Setting only `limits.cpu` without `requests.cpu` | Pods can set very low requests and very high limits, bypassing the spirit of the quota | Set both `requests.*` and `limits.*` |
| Forgetting that quota tracks requests, not actual usage | A pod using 10m CPU but requesting 500m still consumes 500m of quota | Right-size your resource requests |
| Tightening quota expecting existing pods to be evicted | Quota only affects new pod creation — existing pods are untouched | Manually delete and recreate pods if needed |
| Setting `pods: "0"` accidentally | No pods can be created in the namespace | Always verify quota values before applying |
| Not monitoring quota usage | Teams hit quota limits unexpectedly in production | Set up alerts on `kube_resourcequota` Prometheus metrics |
| Applying quota to `default` namespace | Affects all unnamespaced workloads — can break system components | Apply quotas only to team/app namespaces, not `default` or `kube-system` |
| No `services.loadbalancers` limit | Teams can provision unlimited cloud load balancers, running up costs | Set a low limit like `"2"` per namespace |

---

## Files in This Lab

```
03-resource-quotas/
├── resource-quota.yml  ← Namespace + ResourceQuota + LimitRange + compliant Pod
└── README.md           ← This file
```

---

## What's Next

- **LimitRange deep dive** — explore `type: Pod` and `type: PersistentVolumeClaim` limits
- **Priority Classes** — combine with ResourceQuota scopes (`scopeSelector`) to give different quotas to different pod priority levels
- **Namespace-level network policies** — pair quotas with NetworkPolicy for full namespace isolation
- **Monitoring** — set up Prometheus alerts on `kube_resourcequota_used / kube_resourcequota_hard > 0.8` to catch teams approaching their limits

---

## ✍️ Author

**[Himanshu Kumar](https://www.linkedin.com/in/h1manshu-kumar/)** - Learning by building, documenting, and sharing 🚀

---
<div align="center">

**Built while learning Kubernetes hands-on.**
*The best way to understand Resource Quotas is to hit the limit, read the error, and understand exactly why it was rejected.*

</div>
