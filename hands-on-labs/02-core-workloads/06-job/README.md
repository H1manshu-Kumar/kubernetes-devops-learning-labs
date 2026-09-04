# ⚙️ Kubernetes Jobs — Deep Dive

> **Lab Series:** Core Workloads → 06 Job    
> **Difficulty:** Beginner → Intermediate    
> **Estimated Time:** 20–30 minutes    
> **Focus:** Understand what Jobs are, how they differ from long-running workloads, and how completion and failure handling works    

---

## 📌 Table of Contents

1. [Why Jobs Exist](#why-jobs-exist)
2. [Job vs ReplicaSet vs Deployment — Know the Difference](#job-vs-replicaset-vs-deployment--know-the-difference)
3. [How a Job Works — The Completion Loop](#how-a-job-works--the-completion-loop)
4. [Job Manifest — Field by Field Breakdown](#job-manifest--field-by-field-breakdown)
5. [Hands-On Lab](#hands-on-lab)
6. [Completion & Parallelism — The Core Mechanism](#completion--parallelism--the-core-mechanism)
7. [Jobs in Production — What Changes](#jobs-in-production--what-changes)
8. [Interview Q&A — Straight to the Point](#interview-qa--straight-to-the-point)
9. [Common Mistakes & Gotchas](#common-mistakes--gotchas)
10. [What's Next](#whats-next)

---

## Why Jobs Exist

ReplicaSets and Deployments are designed for **long-running workloads** — web servers, APIs, databases. They ensure pods keep running forever. But not every workload is long-running.

Some tasks need to **run once and finish** — a database migration, a batch report, a one-time data transformation. For these, you don't want a pod that restarts forever. You want a pod that runs, completes successfully, and stops.

A Job solves exactly this: **run one or more pods to successful completion, then stop.**

| Scenario | Bare Pod | ReplicaSet/Deployment | Job |
|---|---|---|---|
| Web server that must always run | ❌ No self-healing | ✅ | ❌ Overkill |
| Run a script once and exit | ✅ But no retry on failure | ❌ Restarts forever | ✅ |
| Run 10 batch tasks in parallel | ❌ | ❌ | ✅ |
| Retry on failure until success | ❌ | ❌ (restarts, not retries) | ✅ |
| Track successful completions | ❌ | ❌ | ✅ |

> 💡 **The Rule:** Use a Job for any task that has a defined end — something that runs, finishes, and should not be restarted once successful.

---

## Job vs ReplicaSet vs Deployment — Know the Difference

| Feature | ReplicaSet | Deployment | Job |
|---|---|---|---|
| Purpose | Keep N pods running forever | Long-running + rolling updates | Run to completion |
| Self-healing (pod crash) | ✅ Replaces pod | ✅ Replaces pod | ✅ Retries until success |
| Stops when done | ❌ Never | ❌ Never | ✅ Yes |
| Tracks completions | ❌ | ❌ | ✅ |
| Parallel execution | ❌ | ❌ | ✅ |
| `restartPolicy` allowed | `Always` only | `Always` only | `Never` or `OnFailure` |
| API group | `apps/v1` | `apps/v1` | `batch/v1` |

```
┌──────────────────────────────────────────────────────────────┐
│                         Job                                  │
│  (Tracks completions, retries failures, stops when done)     │
│                                                              │
│   ┌──────────┐   ┌──────────┐   ┌──────────┐                 │
│   │  Pod 1   │   │  Pod 2   │   │  Pod 3   │                 │
│   │ ✅ Done  │   │ ✅ Done  │   │ 🔄 Running│                │
│   └──────────┘   └──────────┘   └──────────┘                 │
│                                                              │
│   completions: 3 | parallelism: 2 | succeeded: 2             │
└──────────────────────────────────────────────────────────────┘
```

---

## How a Job Works — The Completion Loop

Unlike a ReplicaSet which reconciles to keep pods *running*, a Job reconciles to reach a *completion count*.

```
┌──────────────────────────────────────────────────────┐
│                  Job Controller Loop                 │
│                                                      │
│   Desired completions: 1                             │
│   Successful completions so far: ?                   │
│                                                      │
│   succeeded < completions  →  Create/retry pods      │
│   pod failed               →  Retry (if backoff ok)  │
│   succeeded == completions →  Job is Done ✅         │
└──────────────────────────────────────────────────────┘
```

```
kubectl apply -f job.yml
        │
        ▼
┌───────────────┐
│  API Server   │  ← Stores desired state in etcd
└───────┬───────┘
        │
        ▼
┌────────────────────┐
│  Job Controller    │  ← Watches for completion count
│                    │    creates pods, tracks success/failure
└───────┬────────────┘
        │
        ▼
┌────────────────────┐
│   Scheduler        │  ← Assigns pod to a node
└───────┬────────────┘
        │
        ▼
┌────────────────────┐
│  Kubelet (node)    │  ← Runs container, reports exit code
└────────────────────┘
        │
        ▼
   exit code 0?
   ✅ Yes → Job marks pod as Succeeded, increments completion count
   ❌ No  → Job retries (up to backoffLimit)
```

> 💡 A Job tracks success by **exit code**. If the container exits with code `0`, it's a success. Any non-zero exit code is a failure and triggers a retry.

---

## Job Manifest — Field by Field Breakdown

This is the `job.yml` used in this lab:

```yaml
kind: Job               # Resource type
apiVersion: batch/v1    # Jobs live in the batch API group
metadata:
  name: demo-nginx-job  # Name of the Job object
  namespace: nginx      # Namespace for isolation

spec:
  completions: 1        # How many successful pod completions are required
  parallelism: 1        # How many pods can run simultaneously

  template:             # Pod template — same structure as any other workload
    metadata:
      name: demo-job-pod
      labels:
        app: batch-task

    spec:
      containers:
      - name: batch-container
        image: busybox:latest
        command: ["sh", "-c", "echo Hello DevOps World && sleep 10"]
      restartPolicy: Never  # Never restart — let the Job controller handle retries
```

### Every Field Explained

| Field | Required | Purpose |
|---|---|---|
| `kind: Job` | ✅ | Resource type |
| `apiVersion: batch/v1` | ✅ | Jobs are in the `batch` group, not `apps` or core `v1` |
| `metadata.name` | ✅ | Unique name for the Job within the namespace |
| `metadata.namespace` | ❌ | Defaults to `default` if omitted |
| `spec.completions` | ❌ | Number of successful completions required. Defaults to `1` |
| `spec.parallelism` | ❌ | Max pods running at once. Defaults to `1` |
| `spec.backoffLimit` | ❌ | Max retries before Job is marked Failed. Defaults to `6` |
| `spec.template` | ✅ | Pod blueprint — same as any other workload |
| `spec.template.spec.restartPolicy` | ✅ | Must be `Never` or `OnFailure`. `Always` is not allowed in Jobs |

### `restartPolicy: Never` vs `restartPolicy: OnFailure`

| Policy | Behavior on failure |
|---|---|
| `Never` | Pod is marked Failed, Job creates a **new pod** for the retry |
| `OnFailure` | The **same pod** is restarted in-place by the kubelet |

> ⚠️ `restartPolicy: Always` is **not allowed** in a Job. Kubernetes will reject the manifest. This is the #1 beginner mistake with Jobs.

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

---

### Step 2 — Apply the Job

```bash
kubectl apply -f job.yml
```

---

### Step 3 — Verify the Job Was Created

```bash
# Check the Job status
kubectl get job -n nginx

# Check the Pod it created
kubectl get pods -n nginx
```

Expected output:
```
NAME             COMPLETIONS   DURATION   AGE
demo-nginx-job   0/1           5s         5s
```

After ~10 seconds (the `sleep 10` in the command):
```
NAME             COMPLETIONS   DURATION   AGE
demo-nginx-job   1/1           12s        20s
```

- `COMPLETIONS: 1/1` → 1 successful completion out of 1 required — Job is done ✅
- The pod status will show `Completed`, not `Running`

---

### Step 4 — Inspect the Job and Pod

```bash
# Full details — events, conditions, pod template
kubectl describe job demo-nginx-job -n nginx

# Check pod status — it should be Completed, not Running
kubectl get pods -n nginx
```

Notice the pod status is `Completed` — this is unique to Jobs. A completed pod is not deleted automatically (by default) so you can inspect its logs.

---

### Step 5 — Read the Pod Logs

```bash
# Get the pod name
kubectl get pods -n nginx

# Read the output of the job
kubectl logs <pod-name> -n nginx
```

Expected output:
```
Hello DevOps World
```

> 💡 This is the key difference from a Deployment. The pod is done and gone from `Running`, but its logs are still accessible. This is how you verify a batch job ran correctly.

---

### Step 6 — Observe Failure and Retry Behavior

Edit the command to force a failure and watch the Job retry:

```bash
# Apply a failing job
kubectl apply -f - <<EOF
apiVersion: batch/v1
kind: Job
metadata:
  name: failing-job
  namespace: nginx
spec:
  backoffLimit: 3
  template:
    spec:
      containers:
      - name: fail
        image: busybox:latest
        command: ["sh", "-c", "exit 1"]
      restartPolicy: Never
EOF

# Watch the Job create new pods on each retry
kubectl get pods -n nginx -w
```

You'll see multiple pods created — one per retry attempt. After `backoffLimit: 3` retries, the Job is marked `Failed`.

```bash
kubectl get job failing-job -n nginx
# STATUS: Failed
```

---

### Step 7 — Test Parallelism (Advanced)

```bash
kubectl apply -f - <<EOF
apiVersion: batch/v1
kind: Job
metadata:
  name: parallel-job
  namespace: nginx
spec:
  completions: 6
  parallelism: 3
  template:
    spec:
      containers:
      - name: worker
        image: busybox:latest
        command: ["sh", "-c", "echo Working && sleep 5"]
      restartPolicy: Never
EOF

# Watch 3 pods run simultaneously, then the next 3
kubectl get pods -n nginx -w
```

> 💡 With `completions: 6` and `parallelism: 3`, the Job runs 3 pods at a time, waits for them to complete, then runs the next 3 — until 6 total successes are recorded.

---

### Step 8 — Clean Up

```bash
# Delete individual jobs
kubectl delete job demo-nginx-job failing-job parallel-job -n nginx

# Or delete the namespace entirely
kubectl delete namespace nginx
```

---

## Completion & Parallelism — The Core Mechanism

These two fields together define the Job's execution model.

```
completions: 6   →  "I need 6 successful pod completions total"
parallelism: 2   →  "Run at most 2 pods at the same time"

Timeline:
  t=0s   → Pod-1 starts, Pod-2 starts
  t=5s   → Pod-1 ✅, Pod-2 ✅  (succeeded: 2)
  t=5s   → Pod-3 starts, Pod-4 starts
  t=10s  → Pod-3 ✅, Pod-4 ✅  (succeeded: 4)
  t=10s  → Pod-5 starts, Pod-6 starts
  t=15s  → Pod-5 ✅, Pod-6 ✅  (succeeded: 6) → Job Complete ✅
```

**Common patterns:**

| `completions` | `parallelism` | Use Case |
|---|---|---|
| `1` | `1` | Run a single task once (this lab) |
| `N` | `1` | Run N tasks sequentially |
| `N` | `N` | Run N tasks all at once |
| `N` | `M` (M < N) | Run N tasks with M workers in parallel |
| unset | `1` | Run until first success (work queue pattern) |

---

## Jobs in Production — What Changes

The `job.yml` in this lab is intentionally minimal. In production, you'd add:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
  namespace: myapp
spec:
  completions: 1
  parallelism: 1
  backoffLimit: 4                    # Retry up to 4 times before giving up
  activeDeadlineSeconds: 300         # Kill the job if it runs longer than 5 minutes
  ttlSecondsAfterFinished: 3600      # Auto-delete job + pods 1 hour after completion
  template:
    metadata:
      labels:
        app: db-migration
    spec:
      restartPolicy: OnFailure
      containers:
      - name: migrator
        image: myapp/migrator:1.2.3  # Never use :latest in production
        command: ["python", "migrate.py"]
        resources:
          requests:
            cpu: "200m"
            memory: "256Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
        env:
        - name: DB_HOST
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: host
```

Key additions for production:
- `backoffLimit` — prevent infinite retries
- `activeDeadlineSeconds` — prevent runaway jobs from consuming resources forever
- `ttlSecondsAfterFinished` — auto-cleanup so completed jobs don't clutter the cluster
- Pinned image tag — never `:latest`
- Resource requests/limits — prevent the job from starving other workloads
- Secrets via `secretKeyRef` — never hardcode credentials

---

## Interview Q&A — Straight to the Point

**Q: What is a Kubernetes Job?**
A Job is a controller that runs one or more pods to successful completion. Unlike a Deployment or ReplicaSet, it doesn't keep pods running forever — it tracks how many pods have exited with code `0` and stops once the desired completion count is reached.

---

**Q: What is the difference between a Job and a Deployment?**
A Deployment is for long-running workloads that should never stop (web servers, APIs). A Job is for finite tasks that run once and exit. A Deployment restarts pods that exit; a Job marks them as succeeded and counts toward completion.

---

**Q: What `restartPolicy` values are allowed in a Job?**
Only `Never` and `OnFailure`. `Always` is not permitted — Kubernetes will reject the manifest. With `Never`, the Job creates a new pod on failure. With `OnFailure`, the kubelet restarts the same pod in-place.

---

**Q: What is `backoffLimit`?**
The maximum number of times a Job will retry a failed pod before marking the entire Job as `Failed`. Defaults to `6`. Each retry creates a new pod (with `restartPolicy: Never`) with an exponential backoff delay.

---

**Q: What happens to pods after a Job completes?**
By default, completed pods are **not deleted**. They remain in `Completed` state so you can inspect their logs. The Job object also remains. You can set `ttlSecondsAfterFinished` on the Job to auto-delete both after a specified duration.

---

**Q: What is `activeDeadlineSeconds`?**
A hard timeout for the entire Job. If the Job hasn't completed within this many seconds, Kubernetes terminates all its pods and marks the Job as `Failed`. This is different from `backoffLimit` — it's a wall-clock deadline, not a retry count.

---

**Q: What is the difference between `completions` and `parallelism`?**
`completions` is the total number of successful pod completions required for the Job to be considered done. `parallelism` is how many pods can run simultaneously at any given time. For example, `completions: 10, parallelism: 5` runs 5 pods at a time until 10 total successes are recorded.

---

**Q: What API group do Jobs belong to?**
`batch/v1` — not `apps/v1` like Deployments and ReplicaSets, and not core `v1` like Pods. This is a common mistake in manifests.

---

**Q: What is a CronJob and how does it relate to a Job?**
A CronJob is a higher-level controller that creates Jobs on a schedule (like a cron expression). The relationship is: CronJob → creates → Job → creates → Pods. A CronJob is to a Job what a Deployment is to a ReplicaSet.

---

## Common Mistakes & Gotchas

| Mistake | Why It's Wrong | Fix |
|---|---|---|
| `restartPolicy: Always` in a Job | Kubernetes rejects the manifest — not allowed in Jobs | Use `Never` or `OnFailure` |
| Using `apiVersion: apps/v1` | Jobs are in `batch/v1` — wrong API group | Use `apiVersion: batch/v1` |
| Using `image: busybox:latest` in production | Image can silently change — breaks reproducibility | Pin to a specific version |
| Not setting `backoffLimit` | Defaults to `6` — may retry more than expected | Set explicitly based on your tolerance |
| Not setting `activeDeadlineSeconds` | A stuck job runs forever, consuming resources | Always set a deadline for production jobs |
| Not setting `ttlSecondsAfterFinished` | Completed jobs and pods accumulate and clutter the cluster | Set to auto-clean after a reasonable window |
| Expecting pods to be deleted after completion | Completed pods persist by default | Use `ttlSecondsAfterFinished` or clean up manually |
| Forgetting `-n <namespace>` in kubectl commands | Commands silently operate on `default` namespace | Always pass `-n <namespace>` |
| Confusing `parallelism` with `completions` | `parallelism` is concurrency, `completions` is total target | Understand both before setting them |

---

## Files in This Lab

```
06-job/
├── job.yml     ← Minimal busybox Job manifest (intentionally simple)
└── README.md   ← This file
```

---

## ✍️ Author

**[Himanshu Kumar](https://www.linkedin.com/in/h1manshu-kumar/)** - Learning by building, documenting, and sharing 🚀

---
<div align="center">

**Built while learning Kubernetes hands-on.**
*The best way to understand Jobs is to run one, break one, and watch the retry mechanism kick in.*

</div>
