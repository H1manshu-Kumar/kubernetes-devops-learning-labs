# 🔁 Kubernetes ReplicaSets — Deep Dive

> **Lab Series:** Core Workloads → 04 ReplicaSet    
> **Difficulty:** Beginner → Intermediate    
> **Estimated Time:** 30–45 minutes    
> **Focus:** Understand what ReplicaSets are, how they work under the hood, and why you rarely use them directly   

---

## 📌 Table of Contents

1. [Why ReplicaSets Exist](#why-replicasets-exist)
2. [ReplicaSet vs Deployment — Know the Difference](#replicaset-vs-deployment--know-the-difference)
3. [How a ReplicaSet Works — The Reconciliation Loop](#how-a-replicaset-works--the-reconciliation-loop)
4. [ReplicaSet Manifest — Field by Field Breakdown](#replicaset-manifest--field-by-field-breakdown)
5. [Hands-On Lab](#hands-on-lab)
6. [Label Selectors — The Core Mechanism](#label-selectors--the-core-mechanism)
7. [ReplicaSets in Production — What Changes](#replicasets-in-production--what-changes)
8. [Interview Q&A — Straight to the Point](#interview-qa--straight-to-the-point)
9. [Common Mistakes & Gotchas](#common-mistakes--gotchas)
10. [What's Next](#whats-next)

---

## Why ReplicaSets Exist

In the previous lab, you created a bare Pod. The problem with bare Pods is that they are mortal — if a Pod crashes or its node goes down, it's gone. There's nothing to bring it back.

A ReplicaSet solves exactly one problem: **ensuring a specified number of identical Pod replicas are running at all times.**

| Scenario | Bare Pod | ReplicaSet |
|---|---|---|
| Pod crashes | ❌ Gone forever | ✅ New pod created immediately |
| Node goes down | ❌ Pod is lost | ✅ Pod rescheduled on a healthy node |
| You want 3 copies | ❌ Create 3 pods manually | ✅ Set `replicas: 3` |
| You want to update the image | ❌ Delete and recreate | ⚠️ Possible but clunky — use a Deployment instead |
| Rollback support | ❌ None | ❌ None — use a Deployment for this |

> 💡 **The Rule:** In practice, you almost never create a ReplicaSet directly. A Deployment creates and manages ReplicaSets for you. But understanding ReplicaSets is essential — they are the engine behind every Deployment.

---

## ReplicaSet vs Deployment — Know the Difference

This is the most common point of confusion for beginners.

| Feature | ReplicaSet | Deployment |
|---|---|---|
| Ensures N pods are running | ✅ | ✅ (via ReplicaSet) |
| Self-healing | ✅ | ✅ |
| Rolling updates | ❌ | ✅ |
| Rollback | ❌ | ✅ |
| Revision history | ❌ | ✅ |
| Do you create it directly? | Rarely | ✅ Yes |

```
┌─────────────────────────────────────────────────────────┐
│                      Deployment                         │
│  (Manages updates, rollbacks, and owns a ReplicaSet)    │
│                                                         │
│  ┌───────────────────────────────────────────────────┐  │
│  │                   ReplicaSet                      │  │
│  │  (Ensures N pods are always running)              │  │
│  │                                                   │  │
│  │   ┌─────────┐   ┌─────────┐   ┌─────────┐         │  │
│  │   │  Pod 1  │   │  Pod 2  │   │  Pod 3  │         │  │
│  │   └─────────┘   └─────────┘   └─────────┘         │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

> ⚠️ **Key Insight:** When you create a Deployment, Kubernetes automatically creates a ReplicaSet behind the scenes. You interact with the Deployment — the ReplicaSet is an implementation detail you don't touch directly.

---

## How a ReplicaSet Works — The Reconciliation Loop

A ReplicaSet controller runs a continuous reconciliation loop:

```
┌──────────────────────────────────────────────────────┐
│              ReplicaSet Controller Loop              │
│                                                      │
│   Desired replicas: 2                                │
│   Actual running pods: ?                             │
│                                                      │
│   actual < desired  →  Create new pods               │
│   actual > desired  →  Delete excess pods            │
│   actual == desired →  Do nothing                    │
└──────────────────────────────────────────────────────┘
```

```
kubectl apply -f replicasets.yml
        │
        ▼
┌───────────────┐
│  API Server   │  ← Stores desired state in etcd
└───────┬───────┘
        │
        ▼
┌────────────────────┐
│  ReplicaSet        │  ← Watches for pod count changes
│  Controller        │    and reconciles to desired state
└───────┬────────────┘
        │
        ▼
┌────────────────────┐
│   Scheduler        │  ← Assigns each new Pod to a node
└───────┬────────────┘
        │
        ▼
┌────────────────────┐
│  Kubelet (node)    │  ← Pulls image, starts container
└────────────────────┘
```

The ReplicaSet doesn't care *which* pods are running — it only cares about *how many* pods match its label selector. This is a critical detail (see [Label Selectors](#label-selectors--the-core-mechanism)).

---

## ReplicaSet Manifest — Field by Field Breakdown

This is the `replicasets.yml` used in this lab:

```yaml
kind: ReplicaSet          # Resource type
apiVersion: apps/v1       # ReplicaSets live in the apps API group
metadata:
  name: nginx-replicasets # Name of the ReplicaSet object
  namespace: nginx        # Namespace for isolation

spec:
  replicas: 2             # How many Pod copies to maintain at all times

  selector:               # How the ReplicaSet finds and owns its Pods
    matchLabels:
      app: nginx          # Must match template.metadata.labels

  template:               # Pod template — blueprint for every Pod this RS creates
    metadata:
      name: nginx-replicatset-pod
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
| `apiVersion: apps/v1` | ✅ | ReplicaSets are in the `apps` group, not core `v1` |
| `metadata.name` | ✅ | Unique name for the ReplicaSet within the namespace |
| `metadata.namespace` | ❌ | Defaults to `default` if omitted |
| `spec.replicas` | ❌ | Defaults to `1` if omitted |
| `spec.selector.matchLabels` | ✅ | Label query to identify which Pods this RS owns |
| `spec.template` | ✅ | The Pod blueprint — same structure as a standalone Pod spec |
| `spec.template.metadata.labels` | ✅ | Must match `selector.matchLabels` — this is the ownership contract |

> ⚠️ **Critical:** `selector.matchLabels` and `template.metadata.labels` **must match**. If they don't, Kubernetes will reject the manifest. This is the #1 beginner mistake with ReplicaSets (and Deployments).

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
<img width="618" height="46" alt="image" src="https://github.com/user-attachments/assets/4ae897d3-05dc-4846-b229-9b6aedab41ef" />

---

### Step 2 — Apply the ReplicaSet

```bash
kubectl apply -f replicasets.yml
```
<img width="460" height="42" alt="image" src="https://github.com/user-attachments/assets/a8b12d08-c68d-40e7-9cd4-f768218e6a2a" />

---

### Step 3 — Verify Everything Was Created

```bash
# Check the ReplicaSet
kubectl get replicaset -n nginx

# Check the Pods it created
kubectl get pods -n nginx
```

Expected output:
```
NAME               DESIRED   CURRENT   READY   AGE
nginx-replicasets  2         2         2       10s
```

- `DESIRED: 2` → you asked for 2 pods
- `CURRENT: 2` → 2 pods exist
- `READY: 2` → 2 pods are passing their readiness checks

<img width="643" height="98" alt="image" src="https://github.com/user-attachments/assets/3b6fa2ae-18aa-4784-b42c-0d2a510b73c7" />    

<img width="647" height="134" alt="image" src="https://github.com/user-attachments/assets/e416551d-5f26-42f3-9f2c-68c694438e49" />

---

### Step 4 — Inspect the ReplicaSet

```bash
# Full details — events, selector, pod template
kubectl describe replicaset nginx-replicasets -n nginx

# See which node each pod landed on
kubectl get pods -n nginx -o wide
```
<img width="638" height="403" alt="image" src="https://github.com/user-attachments/assets/1be0368f-939c-4933-8c02-02ae63b2564e" />   

<img width="1254" height="177" alt="image" src="https://github.com/user-attachments/assets/f507ffda-86d0-4b9f-963d-06c7eb313f27" />

---

### Step 5 — Test Self-Healing

Delete one Pod manually and watch the ReplicaSet bring it back:

```bash
# Get pod names
kubectl get pods -n nginx

# Delete one pod (replace with your actual pod name)
kubectl delete pod nginx-replicasets-<hash> -n nginx

# Watch it get recreated immediately
kubectl get pods -n nginx -w
```
<img width="734" height="100" alt="image" src="https://github.com/user-attachments/assets/ee60987b-d0f2-4830-8bc0-d717bcf35cfa" />    

<img width="634" height="43" alt="image" src="https://github.com/user-attachments/assets/6a98140b-15db-438b-9e86-71e0aaaf7150" />   

<img width="810" height="102" alt="image" src="https://github.com/user-attachments/assets/37c66c22-432a-4da4-a886-5a5abaecb6e1" />

> 💡 This is the core value of a ReplicaSet. The controller detects actual (1) < desired (2) and creates a replacement Pod within seconds.

---

### Step 6 — Test Scale Up and Down

```bash
# Scale up to 5 replicas
kubectl scale replicaset nginx-replicasets -n nginx --replicas=5

# Verify
kubectl get pods -n nginx

# Scale back down to 2
kubectl scale replicaset nginx-replicasets -n nginx --replicas=2

# Watch excess pods get terminated
kubectl get pods -n nginx -w
```
<img width="898" height="41" alt="image" src="https://github.com/user-attachments/assets/b0d492d4-f818-4d22-bdf6-4d20ee740103" />    

<img width="736" height="139" alt="image" src="https://github.com/user-attachments/assets/2785e1ca-4876-47f0-b607-1be1d9f37ee2" />    

<img width="888" height="42" alt="image" src="https://github.com/user-attachments/assets/cbb4ef99-a0c1-4f59-a32c-1cd4e1ebb70d" />    

<img width="768" height="82" alt="image" src="https://github.com/user-attachments/assets/ff69ad7e-4460-425c-9492-0d8b76ddc1b4" />    

---

### Step 7 — Observe the Adoption Behavior (Advanced)

Create a standalone Pod with the same label as the ReplicaSet selector, and watch what happens:

```bash
# Create a pod with matching label
kubectl run orphan-pod --image=nginx --labels="app=nginx" -n nginx

# Check pod count — the RS will immediately terminate the extra pod
kubectl get pods -n nginx
```
<img width="906" height="140" alt="image" src="https://github.com/user-attachments/assets/92f3c46f-3147-428b-9116-7c14239a0799" />

> 💡 The ReplicaSet sees 3 pods matching its selector (desired: 2) and terminates one. It doesn't care that you created the pod manually — it only counts labels. This is the label selector mechanism in action.

---

### Step 8 — Clean Up

```bash
# Delete the ReplicaSet (also deletes all its pods)
kubectl delete replicaset nginx-replicasets -n nginx

# Or delete the namespace entirely
kubectl delete namespace nginx
```

---

## Label Selectors — The Core Mechanism

This is the most important concept to understand about ReplicaSets (and all Kubernetes controllers).

A ReplicaSet does **not** track pods by name or by who created them. It tracks pods purely by **labels**.

```
ReplicaSet selector:  app=nginx
                           │
                           ▼ "Find all pods with this label"
                    ┌──────────────┐
                    │  Pod count?  │
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
           count < 2    count == 2   count > 2
           Create pods  Do nothing   Delete pods
```

**Consequences of this design:**

1. **Adoption** — If you create a bare Pod with `app: nginx` in the same namespace, the ReplicaSet will adopt it and count it toward its desired replicas. If that puts it over the desired count, it will delete a pod.

2. **Orphaning** — If you remove the `app: nginx` label from a running pod (via `kubectl label pod <name> app-`), the ReplicaSet releases it and immediately creates a new pod to fill the gap. The unlabeled pod keeps running — it's now unmanaged.

3. **Selector is immutable** — Once a ReplicaSet is created, you cannot change `spec.selector`. You must delete and recreate it.

---

## ReplicaSets in Production — What Changes

The `replicasets.yml` in this lab is intentionally minimal. In production, you would never use a bare ReplicaSet — you'd use a Deployment. But if you did use one directly, it would look like this:

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-replicasets
  namespace: nginx
  labels:
    app: nginx
    version: "1.25.3"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
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
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 3
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
```

---

## Interview Q&A — Straight to the Point

**Q: What is a ReplicaSet?**
A ReplicaSet is a Kubernetes controller that ensures a specified number of Pod replicas are running at all times. It continuously reconciles actual pod count (pods matching its label selector) with the desired count, creating or deleting pods as needed.

---

**Q: What is the difference between a ReplicaSet and a Deployment?**
A ReplicaSet only ensures pod count. A Deployment wraps a ReplicaSet and adds rolling updates, rollback, and revision history. In practice, you use Deployments — they manage ReplicaSets for you. You rarely create a ReplicaSet directly.

---

**Q: How does a ReplicaSet identify which Pods it owns?**
Through label selectors. The ReplicaSet counts all pods in its namespace that match `spec.selector.matchLabels`. It doesn't track pods by name or creation source — only by labels.

---

**Q: What happens if you manually create a Pod with the same labels as a ReplicaSet's selector?**
The ReplicaSet adopts it. If the total pod count now exceeds `replicas`, the RS will delete one of the pods (possibly the one you just created). This is expected behavior — the RS only cares about the count.

---

**Q: What happens when you delete a ReplicaSet?**
By default, all pods owned by the ReplicaSet are also deleted (cascade delete). You can use `--cascade=orphan` to delete the RS while leaving the pods running, but they become unmanaged.

---

**Q: Can you update a ReplicaSet's pod template to trigger a rolling update?**
No. If you update `spec.template` on a ReplicaSet (e.g., change the image), existing pods are **not** replaced. Only newly created pods (e.g., after a self-heal or scale-up) will use the new template. This is why Deployments exist — they handle this properly.

---

**Q: Is `spec.selector` mutable on a ReplicaSet?**
No. The selector is immutable after creation. If you need to change it, delete and recreate the ReplicaSet.

---

**Q: What is the difference between `spec.selector.matchLabels` and `spec.template.metadata.labels`?**
`matchLabels` is the query the ReplicaSet uses to find its pods. `template.metadata.labels` are the labels applied to pods the RS creates. They must match — otherwise the RS would create pods it can't find, looping forever. Kubernetes enforces this at admission.

---

**Q: What happens if you scale a ReplicaSet to 0?**
All pods are terminated. The ReplicaSet object still exists but has no running pods. This is exactly what a Deployment does to old ReplicaSets after a rolling update — scales them to 0 but keeps them for potential rollback.

---

**Q: Why would you ever use a ReplicaSet directly instead of a Deployment?**
Rarely — but valid cases include: you need a fixed pod set with no update strategy, you're building a custom controller on top of it, or you're in a learning/debugging context. In production, always prefer Deployments.

---

## Common Mistakes & Gotchas

| Mistake | Why It's Wrong | Fix |
|---|---|---|
| `selector.matchLabels` ≠ `template.metadata.labels` | Kubernetes rejects the manifest — the RS can't own its pods | They must be identical |
| Using `image: nginx:latest` | Image can silently change on pod restart — breaks reproducibility | Pin to a specific version: `nginx:1.25.3` |
| Editing a ReplicaSet's pod template expecting a rolling update | Existing pods are NOT replaced — only new pods use the new template | Use a Deployment for rolling updates |
| Creating a bare Pod with matching labels in the same namespace | The RS adopts it and may delete it if over desired count | Use different labels or a different namespace |
| Deleting a ReplicaSet thinking pods will survive | Cascade delete removes all owned pods by default | Use `--cascade=orphan` if you want pods to survive |
| No `resources` requests/limits | Pods can starve other workloads or get OOM-killed | Always set both in production |
| Forgetting `-n <namespace>` in kubectl commands | Commands silently operate on `default` namespace | Always pass `-n <namespace>` |
| Trying to change `spec.selector` on an existing RS | Immutable — Kubernetes will reject the update | Delete and recreate the ReplicaSet |

---

## Files in This Lab

```
04-replicasets/
├── replicasets.yml  ← Minimal nginx ReplicaSet manifest (intentionally simple)
└── README.md        ← This file
```

---

## What's Next

Now that you understand ReplicaSets, the natural next steps are:

- **Services** — Expose your ReplicaSet's Pods to network traffic (ClusterIP, NodePort, LoadBalancer)
- **ConfigMaps & Secrets** — Inject configuration and credentials into your Pods
- **HorizontalPodAutoscaler (HPA)** — Auto-scale based on CPU/memory metrics
- **DaemonSets** — Like a ReplicaSet, but ensures exactly one Pod runs on every node (used for log collectors, monitoring agents)

---

## ✍️ Author

**[Himanshu Kumar](https://www.linkedin.com/in/h1manshu-kumar/)** - Learning by building, documenting, and sharing 🚀

---
<div align="center">

**Built while learning Kubernetes hands-on.**
*The best way to understand ReplicaSets is to delete pods, mess with labels, and watch the controller react.*

</div>
