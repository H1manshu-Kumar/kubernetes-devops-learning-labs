# 🚀 Kubernetes Deployments — Deep Dive

> **Lab Series:** Core Workloads → 03 Deployment
> **Difficulty:** Beginner → Intermediate
> **Estimated Time:** 45–60 minutes
> **Focus:** Master the most important Kubernetes workload controller — from theory to hands-on to interview-ready

---

## 📌 Table of Contents

1. [Why Deployments Exist](#why-deployments-exist)
2. [Deployment vs ReplicaSet vs Pod — The Hierarchy](#deployment-vs-replicaset-vs-pod--the-hierarchy)
3. [Deployment Architecture — What's Actually Happening](#deployment-architecture--whats-actually-happening)
4. [Deployment Manifest — Field by Field Breakdown](#deployment-manifest--field-by-field-breakdown)
5. [Hands-On Lab](#hands-on-lab)
6. [Rolling Updates — How Zero-Downtime Works](#rolling-updates--how-zero-downtime-works)
7. [Rollback — Undoing a Bad Deployment](#rollback--undoing-a-bad-deployment)
8. [Deployment Strategies Compared](#deployment-strategies-compared)
9. [Deployments in Production — What Changes](#deployments-in-production--what-changes)
10. [Interview Q&A — Straight to the Point](#interview-qa--straight-to-the-point)
11. [Common Mistakes & Gotchas](#common-mistakes--gotchas)
12. [What's Next](#whats-next)

---

## Why Deployments Exist

In the previous lab, you created a bare Pod. Here's the problem with that:

| Scenario | Bare Pod | Deployment |
|---|---|---|
| Pod crashes | ❌ Gone forever | ✅ Automatically recreated |
| Node goes down | ❌ Pod is lost | ✅ Rescheduled on healthy node |
| You want 3 replicas | ❌ Manual — create 3 pods by hand | ✅ Set `replicas: 3` |
| You want to update the image | ❌ Delete and recreate (downtime) | ✅ Rolling update — zero downtime |
| Bad update — need to revert | ❌ No history | ✅ `kubectl rollout undo` |

> 💡 **The Rule:** You almost never create bare Pods in production. You always use a Deployment (or a higher-level abstraction like StatefulSet, DaemonSet, etc.). A Deployment is the standard way to run stateless applications in Kubernetes.

---

## Deployment vs ReplicaSet vs Pod — The Hierarchy

This is the most important mental model to internalize before going further.

```
┌─────────────────────────────────────────────────────────┐
│                      Deployment                         │
│  (Manages updates, rollbacks, and owns a ReplicaSet)    │
│                                                         │
│  ┌───────────────────────────────────────────────────┐  │
│  │                   ReplicaSet                      │  │
│  │  (Ensures N pods are always running)              │  │
│  │                                                   │  │
│  │   ┌─────────┐   ┌─────────┐   ┌─────────┐        │  │
│  │   │  Pod 1  │   │  Pod 2  │   │  Pod 3  │        │  │
│  │   └─────────┘   └─────────┘   └─────────┘        │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

| Object | Responsibility | Do you create it directly? |
|---|---|---|
| **Pod** | Runs your containers | Rarely in production |
| **ReplicaSet** | Keeps N pods alive | Almost never — Deployment manages it |
| **Deployment** | Manages ReplicaSets, enables updates & rollbacks | ✅ Yes — this is your entry point |

> ⚠️ **Key Insight:** When you do a rolling update, the Deployment creates a **new ReplicaSet** and gradually scales it up while scaling down the old one. The old ReplicaSet is kept (with 0 replicas) — that's what enables rollbacks.

---

## Deployment Architecture — What's Actually Happening

```
kubectl apply -f deployment.yml
        │
        ▼
┌───────────────┐
│  API Server   │  ← Stores desired state in etcd
└───────┬───────┘
        │
        ▼
┌────────────────────┐
│ Deployment         │  ← Deployment Controller watches for changes
│ Controller         │    and creates/updates ReplicaSets
└───────┬────────────┘
        │
        ▼
┌────────────────────┐
│ ReplicaSet         │  ← ReplicaSet Controller ensures pod count matches
│ Controller         │    replicas field at all times
└───────┬────────────┘
        │
        ▼
┌────────────────────┐
│   Scheduler        │  ← Assigns each Pod to a node
└───────┬────────────┘
        │
        ▼
┌────────────────────┐
│  Kubelet (node)    │  ← Pulls image, starts container
└────────────────────┘
```

The entire system is **reconciliation-based** — every controller constantly compares *desired state* (what you declared) with *actual state* (what's running) and acts to close the gap.

---

## Deployment Manifest — Field by Field Breakdown

This is the `deployment.yml` used in this lab:

```yaml
kind: Deployment          # Resource type
apiVersion: apps/v1       # Deployments live in the apps API group
metadata:
  name: nginx-deployment  # Name of the Deployment object
  namespace: nginx        # Namespace for isolation

spec:
  replicas: 2             # How many Pod copies to maintain at all times

  selector:               # How the Deployment finds its Pods
    matchLabels:
      app: nginx          # Must match template.metadata.labels

  template:               # Pod template — this is the Pod spec
    metadata:
      name: nginx-dep-pod
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
| `kind` | ✅ | Resource type |
| `apiVersion: apps/v1` | ✅ | Deployments are in the `apps` group, not core `v1` |
| `metadata.name` | ✅ | Unique name for the Deployment |
| `spec.replicas` | ❌ | Defaults to `1` if omitted |
| `spec.selector.matchLabels` | ✅ | Must match `template.metadata.labels` — this is how the Deployment owns its Pods |
| `spec.template` | ✅ | The Pod blueprint — same structure as a Pod spec |
| `spec.template.metadata.labels` | ✅ | Must match `selector.matchLabels` |

> ⚠️ **Critical:** The `selector.matchLabels` and `template.metadata.labels` **must match**. If they don't, Kubernetes will reject the manifest with a validation error. This is the #1 beginner mistake with Deployments.

---

## Hands-On Lab

### Prerequisites
- A running Kubernetes cluster (Minikube, Kind, EKS, GKE, or AKS)
- `kubectl` configured (`kubectl cluster-info` to verify)
- Namespace `nginx` created (from the previous lab, or run Step 1)

---

### Step 1 — Create the Namespace

```bash
kubectl create namespace nginx
```

---

### Step 2 — Apply the Deployment

```bash
kubectl apply -f deployment.yml
```

---

### Step 3 — Verify Everything Was Created

```bash
# Check the Deployment
kubectl get deployment nginx-deployment -n nginx

# Check the ReplicaSet (auto-created by the Deployment)
kubectl get replicaset -n nginx

# Check the Pods (auto-created by the ReplicaSet)
kubectl get pods -n nginx
```

Expected output:
```
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   2/2     2            2           15s
```

- `2/2` → 2 desired, 2 running
- `UP-TO-DATE: 2` → both pods are on the latest spec
- `AVAILABLE: 2` → both pods are ready to serve traffic

---

### Step 4 — Inspect the Deployment

```bash
# Full details — events, strategy, conditions
kubectl describe deployment nginx-deployment -n nginx

# Watch pods in real time
kubectl get pods -n nginx -w

# See which node each pod landed on
kubectl get pods -n nginx -o wide
```

---

### Step 5 — Test Self-Healing (The Most Important Demo)

Delete one of the Pods manually and watch Kubernetes bring it back:

```bash
# Get pod names
kubectl get pods -n nginx

# Delete one pod (replace with your actual pod name)
kubectl delete pod nginx-deployment-<hash> -n nginx

# Watch it get recreated immediately
kubectl get pods -n nginx -w
```

> 💡 This is the core value of a Deployment. The ReplicaSet controller detects that actual replicas (1) < desired replicas (2) and immediately creates a new Pod.

---

### Step 6 — Scale the Deployment

```bash
# Scale up to 5 replicas
kubectl scale deployment nginx-deployment -n nginx --replicas=5

# Verify
kubectl get pods -n nginx

# Scale back down to 2
kubectl scale deployment nginx-deployment -n nginx --replicas=2
```

---

### Step 7 — Perform a Rolling Update

```bash
# Update the nginx image to a specific version
kubectl set image deployment/nginx-deployment nginx=nginx:1.25.3 -n nginx

# Watch the rolling update happen in real time
kubectl rollout status deployment/nginx-deployment -n nginx
```

---

### Step 8 — Clean Up

```bash
# Delete just the deployment (leaves namespace intact)
kubectl delete deployment nginx-deployment -n nginx

# Or nuke the entire namespace
kubectl delete namespace nginx
```

---

## Rolling Updates — How Zero-Downtime Works

This is one of the most important things to understand about Deployments.

When you update the image, Kubernetes does **not** kill all pods and restart them. It uses a controlled rollout:

```
Before Update:   [Pod v1] [Pod v1] [Pod v1]   replicas: 3

Step 1:          [Pod v1] [Pod v1] [Pod v1]
                                   [Pod v2]   ← new pod starts

Step 2:          [Pod v1] [Pod v1]
                          [Pod v2] [Pod v2]   ← old pod removed, new added

Step 3:          [Pod v1]
                 [Pod v2] [Pod v2] [Pod v2]

After Update:    [Pod v2] [Pod v2] [Pod v2]   ← all updated, zero downtime
```

This behavior is controlled by two fields under `spec.strategy.rollingUpdate`:

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1   # Max pods that can be unavailable during update
      maxSurge: 1         # Max extra pods that can be created above desired count
```

| Field | Default | Meaning |
|---|---|---|
| `maxUnavailable` | 25% | How many pods can be down during the update |
| `maxSurge` | 25% | How many extra pods can be created above `replicas` |

> 💡 Setting `maxUnavailable: 0` and `maxSurge: 1` gives you the safest rolling update — always bring up a new pod before taking down an old one.

---

## Rollback — Undoing a Bad Deployment

Kubernetes keeps a history of your Deployment revisions. This is what makes rollbacks instant.

```bash
# View rollout history
kubectl rollout history deployment/nginx-deployment -n nginx

# Rollback to the previous revision
kubectl rollout undo deployment/nginx-deployment -n nginx

# Rollback to a specific revision
kubectl rollout undo deployment/nginx-deployment -n nginx --to-revision=2

# Check rollback status
kubectl rollout status deployment/nginx-deployment -n nginx
```

**How it works under the hood:**

```
After initial deploy:   ReplicaSet-v1 (2 pods) ← active
After image update:     ReplicaSet-v1 (0 pods)   ReplicaSet-v2 (2 pods) ← active
After rollback:         ReplicaSet-v1 (2 pods) ← active   ReplicaSet-v2 (0 pods)
```

The old ReplicaSet is never deleted — it's just scaled to 0. Rollback = scale old RS back up, scale new RS down.

> ⚠️ The number of old ReplicaSets kept is controlled by `spec.revisionHistoryLimit` (default: 10). Set it to `0` and you lose rollback capability entirely.

---

## Deployment Strategies Compared

| Strategy | How It Works | Downtime | Use Case |
|---|---|---|---|
| **RollingUpdate** (default) | Gradually replaces old pods with new ones | None | Stateless apps — standard choice |
| **Recreate** | Kills all old pods, then creates new ones | Yes | When old and new versions can't run simultaneously (DB schema changes) |
| **Blue/Green** | Run two full environments, switch traffic | None | High-risk releases, instant rollback (requires external tooling) |
| **Canary** | Route small % of traffic to new version | None | Gradual validation in production (requires Service mesh or Ingress) |

> 💡 `RollingUpdate` and `Recreate` are native Kubernetes strategies. Blue/Green and Canary require additional tooling (Argo Rollouts, Flagger, or manual Service manipulation).

---

## Deployments in Production — What Changes

The `deployment.yml` in this lab is intentionally minimal. In production:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: nginx
  labels:
    app: nginx
    version: "1.25.3"
spec:
  replicas: 3
  revisionHistoryLimit: 5          # Keep last 5 ReplicaSets for rollback
  selector:
    matchLabels:
      app: nginx
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25.3        # Never use :latest in production
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "256Mi"
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 3
          periodSeconds: 5
```

---

## Interview Q&A — Straight to the Point

**Q: What is a Deployment in Kubernetes?**
A Deployment is a higher-level controller that manages a ReplicaSet, which in turn manages Pods. It adds self-healing, declarative scaling, rolling updates, and rollback capabilities on top of what a bare Pod provides.

---

**Q: What is the relationship between a Deployment, ReplicaSet, and Pod?**
Deployment owns ReplicaSet(s). ReplicaSet owns Pods. You interact with the Deployment — it manages everything below. When you update a Deployment, it creates a new ReplicaSet and gradually migrates pods from the old RS to the new one.

---

**Q: What happens when you delete a Pod that's managed by a Deployment?**
The ReplicaSet controller detects that actual pod count dropped below desired count and immediately creates a new Pod to replace it. This is self-healing — it happens automatically within seconds.

---

**Q: How does a rolling update work?**
Kubernetes creates a new ReplicaSet with the updated pod spec. It then incrementally scales up the new RS and scales down the old RS, respecting `maxUnavailable` and `maxSurge` constraints, until all pods are on the new version.

---

**Q: How does rollback work in Kubernetes?**
Kubernetes retains old ReplicaSets (scaled to 0 replicas). A rollback simply scales the previous ReplicaSet back up and scales the current one down. `kubectl rollout undo` triggers this. The number of retained revisions is controlled by `revisionHistoryLimit`.

---

**Q: What is the difference between `maxUnavailable` and `maxSurge`?**
- `maxUnavailable` — how many pods can be unavailable (below desired count) during an update. Controls minimum availability.
- `maxSurge` — how many extra pods can exist above the desired count during an update. Controls maximum pod count.

Setting both to `0` simultaneously is invalid — the update would never progress.

---

**Q: What is the difference between `RollingUpdate` and `Recreate` strategy?**
`RollingUpdate` replaces pods gradually — no downtime. `Recreate` terminates all existing pods first, then creates new ones — causes downtime but guarantees no two versions run simultaneously. Use `Recreate` when your app can't handle two versions running at the same time (e.g., incompatible DB schema changes).

---

**Q: What does `selector.matchLabels` do and why must it match `template.metadata.labels`?**
`selector.matchLabels` is how the Deployment (and its ReplicaSet) identifies which Pods it owns. If the labels on the Pod template don't match the selector, the Deployment can't track its Pods — Kubernetes rejects this at admission. It's a required contract.

---

**Q: What is `revisionHistoryLimit`?**
It controls how many old ReplicaSets Kubernetes retains after updates. Default is 10. Each retained RS represents one rollback point. Setting it to `0` saves etcd space but eliminates rollback capability.

---

**Q: Can you update a Deployment's `selector` after creation?**
No. The `spec.selector` is **immutable** after creation. If you need to change it, you must delete and recreate the Deployment. This is by design — changing the selector would cause the Deployment to lose track of its existing Pods.

---

**Q: What is `kubectl rollout status` used for?**
It blocks and streams the progress of a rollout, reporting whether each replica was successfully updated. It exits with code `0` on success and non-zero on failure — making it useful in CI/CD pipelines to gate deployments.

---

## Common Mistakes & Gotchas

| Mistake | Why It's Wrong | Fix |
|---|---|---|
| `selector.matchLabels` ≠ `template.metadata.labels` | Kubernetes rejects the manifest — the Deployment can't own its Pods | They must be identical |
| Using `image: nginx:latest` | Image can silently change on next pod restart — breaks reproducibility | Pin to a specific version: `nginx:1.25.3` |
| No `resources` requests/limits | Pods can starve other workloads or get OOM-killed unpredictably | Always set both in production |
| No liveness/readiness probes | Kubernetes sends traffic to pods that aren't ready yet | Add probes — especially `readinessProbe` |
| Setting `revisionHistoryLimit: 0` | You lose all rollback capability | Keep at least 3–5 revisions |
| Editing a ReplicaSet directly | The Deployment controller will immediately revert your change | Always edit the Deployment, never the RS |
| `maxUnavailable: 0` AND `maxSurge: 0` | Invalid — the rollout can never make progress | At least one must be non-zero |
| Forgetting `namespace` in `kubectl` commands | Commands silently operate on the `default` namespace | Always pass `-n <namespace>` or set a context |
| Using `kubectl replace` instead of `kubectl apply` | `replace` requires the full spec and fails if the resource doesn't exist | Use `apply` for idempotent declarative management |

---

## Files in This Lab

```
03-deployment/
├── deployment.yml   ← Minimal nginx Deployment manifest (intentionally simple)
└── README.md        ← This file
```

---

## What's Next

Now that you understand Deployments, the natural next steps are:

- **Services** — Expose your Deployment's Pods to network traffic (ClusterIP, NodePort, LoadBalancer)
- **ConfigMaps & Secrets** — Inject configuration and credentials into your Pods
- **HorizontalPodAutoscaler (HPA)** — Auto-scale your Deployment based on CPU/memory metrics
- **StatefulSets** — Like Deployments, but for stateful apps (databases) that need stable network identity and persistent storage

---

## ✍️ Author

**[Himanshu Kumar](https://www.linkedin.com/in/h1manshu-kumar/)** - Learning by building, documenting, and sharing 🚀

---
<div align="center">

**Built while learning Kubernetes hands-on.**
*The best way to understand Deployments is to break them, roll them back, and break them again.*

</div>
