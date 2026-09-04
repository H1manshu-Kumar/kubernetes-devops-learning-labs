# ⏰ Kubernetes CronJobs — Deep Dive

> **Lab Series:** Core Workloads → 07 CronJob    
> **Difficulty:** Beginner → Intermediate    
> **Estimated Time:** 25–35 minutes        
> **Focus:** Understand what CronJobs are, how they schedule Jobs, and how to manage recurring workloads reliably    

---

## 📌 Table of Contents

1. [Why CronJobs Exist](#why-cronjobs-exist)
2. [CronJob vs Job vs Deployment — Know the Difference](#cronjob-vs-job-vs-deployment--know-the-difference)
3. [How a CronJob Works — The Scheduling Loop](#how-a-cronjob-works--the-scheduling-loop)
4. [CronJob Manifest — Field by Field Breakdown](#cronjob-manifest--field-by-field-breakdown)
5. [Cron Schedule Syntax](#cron-schedule-syntax)
6. [Hands-On Lab](#hands-on-lab)
7. [Concurrency & History — The Core Controls](#concurrency--history--the-core-controls)
8. [CronJobs in Production — What Changes](#cronjobs-in-production--what-changes)
9. [Interview Q&A — Straight to the Point](#interview-qa--straight-to-the-point)
10. [Common Mistakes & Gotchas](#common-mistakes--gotchas)

---

## Why CronJobs Exist

In the previous lab, you used a Job to run a task once and exit. But many real-world tasks need to run **repeatedly on a schedule** — nightly database backups, hourly report generation, periodic cache flushes, certificate renewals.

Outside Kubernetes, you'd use a Linux `cron` daemon for this. Inside Kubernetes, a CronJob is the native equivalent.

A CronJob solves exactly this: **create a new Job on a defined schedule, automatically, forever.**

| Scenario | Bare Pod | Job | CronJob |
|---|---|---|---|
| Web server that must always run | ❌ | ❌ | ❌ |
| Run a script once and exit | ❌ No retry | ✅ | ❌ Overkill |
| Run a backup every night at midnight | ❌ | ❌ Manual trigger | ✅ |
| Run a report every hour | ❌ | ❌ | ✅ |
| Retry on failure | ❌ | ✅ | ✅ (via the Job it creates) |

> 💡 **The Rule:** A CronJob is not a workload itself — it's a **scheduler**. It creates a Job at each scheduled interval, and that Job creates the Pod. The hierarchy is: `CronJob → Job → Pod`.

---

## CronJob vs Job vs Deployment — Know the Difference

| Feature | Deployment | Job | CronJob |
|---|---|---|---|
| Purpose | Long-running workload | Run once to completion | Run on a schedule |
| Self-healing (pod crash) | ✅ Replaces pod | ✅ Retries until success | ✅ (via Job) |
| Stops when done | ❌ Never | ✅ Yes | ✅ Each run stops |
| Runs on a schedule | ❌ | ❌ | ✅ |
| Creates Jobs | ❌ | N/A | ✅ |
| Tracks completions | ❌ | ✅ | ✅ (per Job) |
| `restartPolicy` allowed | `Always` only | `Never` or `OnFailure` | `Never` or `OnFailure` |
| API group | `apps/v1` | `batch/v1` | `batch/v1` |

```
┌──────────────────────────────────────────────────────────────┐
│                        CronJob                               │
│  (Triggers on schedule — e.g., every minute)                 │
│                                                              │
│   ┌──────────────────────────────────────────────────────┐   │
│   │  Job (created at t=00:01)                            │   │
│   │   └── Pod → runs backup → exits 0 → ✅ Completed     │   │
│   └──────────────────────────────────────────────────────┘   │
│   ┌──────────────────────────────────────────────────────┐   │
│   │  Job (created at t=00:02)                            │   │
│   │   └── Pod → runs backup → exits 0 → ✅ Completed     │   │
│   └──────────────────────────────────────────────────────┘   │
│   ...                                                        │
└──────────────────────────────────────────────────────────────┘
```

---

## How a CronJob Works — The Scheduling Loop

Unlike a ReplicaSet (which reconciles pod count) or a Job (which reconciles completion count), a CronJob reconciles against **time**.

```
┌──────────────────────────────────────────────────────┐
│               CronJob Controller Loop                │
│                                                      │
│   Schedule: "* * * * *" (every minute)               │
│   Current time: matches schedule?                    │
│                                                      │
│   Yes → Create a new Job object                      │
│   No  → Wait and check again                         │
│   Job already running + concurrencyPolicy=Forbid     │
│       → Skip this run                                │
└──────────────────────────────────────────────────────┘
```

```
kubectl apply -f cron-job.yml
        │
        ▼
┌───────────────┐
│  API Server   │  ← Stores CronJob desired state in etcd
└───────┬───────┘
        │
        ▼
┌────────────────────┐
│  CronJob           │  ← Watches the clock, fires at each
│  Controller        │    scheduled interval
└───────┬────────────┘
        │  (creates a Job at each tick)
        ▼
┌────────────────────┐
│  Job Controller    │  ← Manages pod creation and completion
└───────┬────────────┘
        │
        ▼
┌────────────────────┐
│  Kubelet (node)    │  ← Runs the container, reports exit code
└────────────────────┘
```

> 💡 The CronJob controller checks the schedule approximately every 10 seconds. If the controller was down and missed a scheduled run, it will catch up — but only if the missed window is within `startingDeadlineSeconds`.

---

## CronJob Manifest — Field by Field Breakdown

This is the `cron-job.yml` used in this lab:

```yaml
kind: CronJob
apiVersion: batch/v1
metadata:
  name: minute-backup
  namespace: nginx

spec:
  schedule: "* * * * *"        # Run every minute
  jobTemplate:                 # Blueprint for the Job created at each tick
    spec:
      template:
        metadata:
          name: minute-backup
          labels:
            app: minute-backup

        spec:
          containers:
          - name: backup-container
            image: busybox:1.28
            command:
              - sh
              - -c
              - >
                echo "====Backup Started====" ;
                mkdir -p /backups &&
                mkdir -p /demo-data &&
                cp -r /demo-data /backups &&
                echo "====Backup Completed!====" ;
            volumeMounts:
                - name: data-volume
                  mountPath: /demo-data
                - name: backup-volume
                  mountPath: /backups
          restartPolicy: OnFailure
          volumes:
            - name: data-volume
              hostPath:
                path: /demo-data
                type: DirectoryOrCreate
            - name: backup-volume
              hostPath:
                path: /backups
                type: DirectoryOrCreate
```

### Every Field Explained

| Field | Required | Purpose |
|---|---|---|
| `kind: CronJob` | ✅ | Resource type |
| `apiVersion: batch/v1` | ✅ | CronJobs are in the `batch` group, same as Jobs |
| `metadata.name` | ✅ | Unique name for the CronJob within the namespace |
| `metadata.namespace` | ❌ | Defaults to `default` if omitted |
| `spec.schedule` | ✅ | Cron expression defining when to trigger a Job |
| `spec.jobTemplate` | ✅ | The Job blueprint — same structure as a standalone Job spec |
| `spec.concurrencyPolicy` | ❌ | What to do if a previous Job is still running. Defaults to `Allow` |
| `spec.successfulJobsHistoryLimit` | ❌ | How many completed Jobs to keep. Defaults to `3` |
| `spec.failedJobsHistoryLimit` | ❌ | How many failed Jobs to keep. Defaults to `1` |
| `spec.startingDeadlineSeconds` | ❌ | How late a missed run can start before being skipped |
| `spec.suspend` | ❌ | Set to `true` to pause the CronJob without deleting it |
| `restartPolicy` | ✅ | Must be `Never` or `OnFailure` — `Always` is not allowed |
| `hostPath` volumes | ⚠️ | Mounts a directory from the node's filesystem — fine for labs, avoid in production |

### Key Design Decisions in This Manifest

- `schedule: "* * * * *"` — fires every minute, making it easy to observe in a lab
- `busybox:1.28` — pinned image tag (good practice, unlike `:latest`)
- `hostPath` volumes — the backup reads from `/demo-data` and writes to `/backups` on the node. This is intentionally simple for a lab; production would use PersistentVolumeClaims or object storage
- `restartPolicy: OnFailure` — if the backup script fails, the kubelet restarts the same pod rather than creating a new one

---

## Cron Schedule Syntax

The `schedule` field uses standard Unix cron syntax with 5 fields:

```
┌─────────────── minute        (0–59)
│ ┌───────────── hour          (0–23)
│ │ ┌─────────── day of month  (1–31)
│ │ │ ┌───────── month         (1–12)
│ │ │ │ ┌─────── day of week   (0–6, Sunday=0)
│ │ │ │ │
* * * * *
```

| Expression | Meaning |
|---|---|
| `* * * * *` | Every minute |
| `0 * * * *` | Every hour (at minute 0) |
| `0 2 * * *` | Every day at 2:00 AM |
| `0 2 * * 0` | Every Sunday at 2:00 AM |
| `*/5 * * * *` | Every 5 minutes |
| `0 0 1 * *` | First day of every month at midnight |
| `0 9-17 * * 1-5` | Every hour from 9 AM to 5 PM, Monday–Friday |

> 💡 Use [crontab.guru](https://crontab.guru) to validate and translate cron expressions into plain English before applying them.

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
<img width="345" height="197" alt="image" src="https://github.com/user-attachments/assets/3c4dc55b-b2ff-458f-bfe1-5e5a41cc0c7a" />

---

### Step 2 — Apply the CronJob

```bash
kubectl apply -f cron-job.yml
```
<img width="388" height="45" alt="image" src="https://github.com/user-attachments/assets/410c037d-f361-45d2-9a1b-6520538fe44b" />

---

### Step 3 — Verify the CronJob Was Created

```bash
kubectl get cronjob -n nginx
```
<img width="784" height="63" alt="image" src="https://github.com/user-attachments/assets/3c859bd1-c601-43cf-98d2-c8a1190df519" />

Expected output:
```
NAME            SCHEDULE    SUSPEND   ACTIVE   LAST SCHEDULE   AGE
minute-backup   * * * * *   False     0        <none>          5s
```

- `SUSPEND: False` → the CronJob is active and will fire on schedule
- `ACTIVE: 0` → no Job is currently running (we just created it)
- `LAST SCHEDULE: <none>` → no run has triggered yet

---

### Step 4 — Watch the First Job Trigger

Wait up to 60 seconds and watch a Job get created automatically:

```bash
# Watch Jobs appear as the CronJob fires
kubectl get jobs -n nginx -w

# In a separate terminal, watch the pods
kubectl get pods -n nginx -w
```

Expected output after the first trigger:
```
NAME                       COMPLETIONS   DURATION   AGE
minute-backup-<timestamp>  0/1           3s         3s
minute-backup-<timestamp>  1/1           8s         8s
```

The pod will go through: `Pending → ContainerCreating → Running → Completed`

<img width="784" height="100" alt="image" src="https://github.com/user-attachments/assets/b1edba7a-c4ed-46b4-807d-289d725463bb" />

---

### Step 5 — Inspect the CronJob and Job

```bash
# Full CronJob details — schedule, last run, active jobs
kubectl describe cronjob minute-backup -n nginx

# Inspect the Job that was created
kubectl get jobs -n nginx

# Full Job details
kubectl describe job <job-name> -n nginx
```

Notice in the CronJob description:
- `Last Schedule Time` — when the last Job was triggered
- `Active Jobs` — Jobs currently running
- `Events` — shows each time a Job was created
<img width="1169" height="687" alt="image" src="https://github.com/user-attachments/assets/06ea98ac-47c0-409c-87f9-6838e900a06e" />

<img width="1169" height="552" alt="image" src="https://github.com/user-attachments/assets/2ad6c82b-3b3a-4bc2-a6b5-fa652e4eaf76" />

---

### Step 6 — Read the Pod Logs

```bash
# Get the pod name (it will have a generated suffix)
kubectl get pods -n nginx

# Read the backup output
kubectl logs <pod-name> -n nginx
```

Expected output:
```
====Backup Started====
====Backup Completed!====
```
<img width="239" height="55" alt="image" src="https://github.com/user-attachments/assets/28f59724-72cf-426a-96b3-ea73dca8baa8" />

> 💡 Even after the pod reaches `Completed` status, its logs remain accessible. This is how you verify each scheduled run executed correctly.

---

### Step 7 — Observe Multiple Runs Accumulate

Wait 3–4 minutes and check how Jobs accumulate:

```bash
kubectl get jobs -n nginx
```

You'll see multiple Jobs — one per minute. By default, the CronJob keeps the last 3 successful Jobs and 1 failed Job. Older ones are automatically deleted.

```
NAME                            COMPLETIONS   DURATION   AGE
minute-backup-<timestamp-1>     1/1           6s         3m
minute-backup-<timestamp-2>     1/1           5s         2m
minute-backup-<timestamp-3>     1/1           7s         1m
```
<img width="613" height="55" alt="image" src="https://github.com/user-attachments/assets/090e55d9-99e1-4eee-9dd2-7c482e34c219" />   

---

### Step 8 — Suspend and Resume the CronJob

```bash
# Suspend — stops new Jobs from being created (existing ones keep running)
kubectl patch cronjob minute-backup -n nginx -p '{"spec":{"suspend":true}}'

# Verify it's suspended
kubectl get cronjob -n nginx
# SUSPEND column should show: True

# Resume
kubectl patch cronjob minute-backup -n nginx -p '{"spec":{"suspend":false}}'
```
<img width="408" height="40" alt="image" src="https://github.com/user-attachments/assets/9c8cad89-670a-4c93-855a-4be72f70e9ee" />    

<img width="631" height="38" alt="image" src="https://github.com/user-attachments/assets/53fae328-fd44-4555-98c2-14998c13e366" />   

<img width="631" height="38" alt="image" src="https://github.com/user-attachments/assets/7202198b-56a8-48b8-b66f-946101d3bfac" />    

> 💡 Suspending a CronJob is the clean way to pause scheduled runs without deleting the CronJob. Useful for maintenance windows or debugging.

---

### Step 9 — Trigger a Manual Run (Advanced)

You can create a Job from a CronJob on demand, without waiting for the schedule:

```bash
kubectl create job manual-backup --from=cronjob/minute-backup -n nginx

# Watch it run
kubectl get pods -n nginx -w
```

This is useful for testing your CronJob logic or running an out-of-schedule backup.

---

### Step 10 — Clean Up

```bash
# Delete the CronJob (also deletes all Jobs and Pods it created)
kubectl delete cronjob minute-backup -n nginx

# Or delete the namespace entirely
kubectl delete namespace nginx
```
<img width="487" height="36" alt="image" src="https://github.com/user-attachments/assets/da05b7f0-151d-41c0-a321-e5296cc48a3c" />

---

## Concurrency & History — The Core Controls

### `concurrencyPolicy`

What happens if a scheduled Job is still running when the next trigger fires?

| Policy | Behavior |
|---|---|
| `Allow` (default) | Start the new Job even if the previous one is still running — both run concurrently |
| `Forbid` | Skip the new run entirely if the previous Job hasn't finished |
| `Replace` | Cancel the still-running Job and start a fresh one |

```
Schedule: every minute
Job duration: 90 seconds (longer than the interval)

Allow:   t=1m → Job-1 starts
         t=2m → Job-2 starts (Job-1 still running) ← two jobs running simultaneously
         t=2m30s → Job-1 completes
         t=3m → Job-3 starts (Job-2 still running)

Forbid:  t=1m → Job-1 starts
         t=2m → SKIP (Job-1 still running)
         t=2m30s → Job-1 completes
         t=3m → Job-3 starts ← clean, no overlap

Replace: t=1m → Job-1 starts
         t=2m → Job-1 CANCELLED, Job-2 starts fresh
```

> ⚠️ For backup jobs, `Forbid` is almost always the right choice. You don't want two backup processes writing to the same destination simultaneously.

### History Limits

```yaml
spec:
  successfulJobsHistoryLimit: 3   # Keep last 3 successful Jobs (default)
  failedJobsHistoryLimit: 1       # Keep last 1 failed Job (default)
```

Setting these to `0` means Jobs are deleted immediately after completion — you lose log access. Setting them too high clutters the cluster with old Job objects. `3` successful and `1` failed is a sensible default for most use cases.

---

## CronJobs in Production — What Changes

The `cron-job.yml` in this lab is intentionally simple. In production, you'd add:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-backup
  namespace: myapp
spec:
  schedule: "0 2 * * *"                  # Every day at 2:00 AM
  concurrencyPolicy: Forbid              # Never run two backups at once
  successfulJobsHistoryLimit: 5
  failedJobsHistoryLimit: 3
  startingDeadlineSeconds: 300           # Skip if more than 5 minutes late
  jobTemplate:
    spec:
      backoffLimit: 2                    # Retry up to 2 times on failure
      activeDeadlineSeconds: 3600        # Kill the job if it runs longer than 1 hour
      ttlSecondsAfterFinished: 86400     # Auto-delete job + pods after 24 hours
      template:
        metadata:
          labels:
            app: nightly-backup
        spec:
          restartPolicy: OnFailure
          containers:
          - name: backup
            image: myapp/backup-tool:2.1.0   # Never use :latest in production
            command: ["python", "backup.py"]
            resources:
              requests:
                cpu: "200m"
                memory: "256Mi"
              limits:
                cpu: "500m"
                memory: "512Mi"
            env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: password
```

Key additions for production:
- `concurrencyPolicy: Forbid` — prevent overlapping backup runs
- `startingDeadlineSeconds` — skip a run if the cluster was down and the window has passed
- `backoffLimit` on the Job — control retry behaviour
- `activeDeadlineSeconds` — kill runaway jobs
- `ttlSecondsAfterFinished` — auto-cleanup
- Pinned image tag — never `:latest`
- Resource requests/limits — prevent the job from starving other workloads
- Secrets via `secretKeyRef` — never hardcode credentials
- PersistentVolumeClaims instead of `hostPath` — portable across nodes

---

## Interview Q&A — Straight to the Point

**Q: What is a Kubernetes CronJob?**
A CronJob is a controller that creates a new Job on a defined schedule using standard cron syntax. It's the Kubernetes equivalent of a Linux cron daemon. The hierarchy is CronJob → Job → Pod.

---

**Q: What is the relationship between a CronJob and a Job?**
A CronJob is a scheduler — it doesn't run workloads directly. At each scheduled interval, it creates a Job object. That Job then creates and manages the Pod. The CronJob is to a Job what a Deployment is to a ReplicaSet.

---

**Q: What `restartPolicy` values are allowed in a CronJob?**
Only `Never` and `OnFailure`, because the pod template inside a CronJob is ultimately a Job pod template. `Always` is not permitted — Kubernetes will reject the manifest.

---

**Q: What is `concurrencyPolicy` and which value should you use for backups?**
`concurrencyPolicy` controls what happens when a new scheduled run fires while the previous Job is still running. For backups, use `Forbid` — it skips the new run rather than running two backup processes simultaneously, which could corrupt data or cause conflicts.

---

**Q: What is `startingDeadlineSeconds`?**
A window of time after the scheduled trigger during which the Job can still start. If the CronJob controller was down (e.g., during a cluster upgrade) and missed a scheduled run, it will only start the Job if the current time is within `startingDeadlineSeconds` of the missed schedule. If the deadline has passed, the run is skipped and counted as a missed run.

---

**Q: What happens if a CronJob misses 100 or more scheduled runs?**
Kubernetes stops trying to schedule new runs and marks the CronJob as unable to schedule. This is a known edge case when `startingDeadlineSeconds` is not set and the controller is down for a long time. Setting `startingDeadlineSeconds` prevents this by bounding the missed-run window.

---

**Q: How do you pause a CronJob without deleting it?**
Set `spec.suspend: true` via `kubectl patch` or by editing the manifest. This stops new Jobs from being created while leaving the CronJob object and its history intact. Set it back to `false` to resume.

---

**Q: What API group does CronJob belong to?**
`batch/v1` — same as Jobs. Not `apps/v1` like Deployments and ReplicaSets.

---

**Q: How do you trigger a CronJob run immediately without waiting for the schedule?**
Use `kubectl create job <name> --from=cronjob/<cronjob-name> -n <namespace>`. This creates a one-off Job using the CronJob's `jobTemplate`, independent of the schedule.

---

**Q: What is `successfulJobsHistoryLimit` and why does it matter?**
It controls how many completed (successful) Job objects the CronJob retains. Defaults to `3`. If set to `0`, Jobs are deleted immediately after completion and you lose access to their logs. Setting it too high clutters the cluster with old Job objects.

---

## Common Mistakes & Gotchas

| Mistake | Why It's Wrong | Fix |
|---|---|---|
| `restartPolicy: Always` in the pod template | Kubernetes rejects the manifest — not allowed in Jobs or CronJobs | Use `Never` or `OnFailure` |
| Using `apiVersion: apps/v1` | CronJobs are in `batch/v1` — wrong API group | Use `apiVersion: batch/v1` |
| `schedule: "* * * * *"` in production | Fires every minute — likely too frequent and resource-intensive | Use an appropriate interval like `0 2 * * *` |
| Not setting `concurrencyPolicy` for long-running jobs | Defaults to `Allow` — multiple instances can run simultaneously | Set `Forbid` or `Replace` based on your use case |
| Not setting `startingDeadlineSeconds` | If the controller misses 100+ runs, the CronJob stops scheduling | Set a reasonable deadline (e.g., `300`) |
| Using `hostPath` volumes in production | Pods are tied to a specific node — breaks if rescheduled | Use PersistentVolumeClaims or object storage (S3, GCS) |
| Using `image: busybox:latest` | Image can silently change — breaks reproducibility | Pin to a specific version like `busybox:1.28` |
| Not setting `successfulJobsHistoryLimit` | Old Job objects accumulate and clutter the cluster | Set to a small number like `3`–`5` |
| Not setting `activeDeadlineSeconds` on the Job | A stuck job runs forever, consuming resources | Always set a deadline for production jobs |
| Forgetting `-n <namespace>` in kubectl commands | Commands silently operate on `default` namespace | Always pass `-n <namespace>` |
| Expecting the CronJob to fire exactly on time | The controller checks approximately every 10 seconds — slight drift is normal | Design jobs to be idempotent, not time-exact |

---

## Files in This Lab

```
07-cron-job/
├── cron-job.yml  ← Minimal backup CronJob manifest (intentionally simple)
└── README.md     ← This file
```

---

## ✍️ Author

**[Himanshu Kumar](https://www.linkedin.com/in/h1manshu-kumar/)** - Learning by building, documenting, and sharing 🚀

---
<div align="center">

**Built while learning Kubernetes hands-on.**
*The best way to understand CronJobs is to set a 1-minute schedule, watch Jobs appear, suspend it, trigger one manually, and break the concurrency policy.*

</div>
