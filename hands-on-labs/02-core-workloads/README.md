# Lab 02 — Core Kubernetes Workloads

> **Week 2** | The foundational building blocks every Kubernetes engineer works with daily — from Namespaces to CronJobs

---

## What This Lab Covers

| # | Topic | What You'll Learn |
|---|---|---|
| 01 | **Namespaces** | Isolate workloads, scope resources, organise environments |
| 02 | **Pods** | The smallest deployable unit — containers, ports, lifecycle |
| 03 | **Deployments** | Declarative rollouts, scaling, rollbacks, self-healing |
| 04 | **ReplicaSets** | Reconciliation loop, label selectors, desired vs actual state |
| 05 | **DaemonSets** | Node-level workloads — one pod per node, guaranteed |
| 06 | **Jobs** | Run-to-completion tasks, retry logic, parallel execution |
| 07 | **CronJobs** | Scheduled recurring tasks, concurrency policies, history limits |

---

## Why This Lab Matters

> Every production Kubernetes cluster — whether on AWS EKS, Google GKE, or Azure AKS — is built from exactly these primitives.

Mastering this lab means you can:
- Read and write any real-world Kubernetes manifest
- Understand what happens when a pod crashes, scales, or gets updated
- Choose the right workload controller for any scenario
- Speak confidently in interviews about how Kubernetes actually works

---

## Workload Controller — Quick Decision Guide

| Scenario | Use |
|---|---|
| Web server / API that must always run | Deployment |
| Run a script once and exit | Job |
| Run a backup every night at 2 AM | CronJob |
| Run one agent on every node (logging, monitoring) | DaemonSet |
| Manage pod replicas directly (rare) | ReplicaSet |
| Scope resources and isolate teams/environments | Namespace |

---

## Lab Structure

```
02-core-workloads/
├── README.md                       ← you are here
├── 01-namespace/
│   ├── namespace.yml
│   └── README.md
├── 02-pods/
│   ├── pod.yml
│   └── README.md
├── 03-deployment/
│   ├── deployment.yml
│   └── README.md
├── 04-replicasets/
│   ├── replicasets.yml
│   └── README.md
├── 05-deamonset/
│   ├── DaemonSet.yml
│   └── README.md
├── 06-job/
│   ├── job.yml
│   └── README.md
└── 07-cron-job/
    ├── cron-job.yml
    └── README.md
```

---

## The Hierarchy — How Controllers Relate

```
Cluster
└── Namespace
    ├── Deployment → ReplicaSet → Pod(s)       ← stateless long-running apps
    ├── DaemonSet  → Pod (one per node)         ← node-level agents
    ├── Job        → Pod(s)                     ← run-to-completion tasks
    └── CronJob    → Job → Pod(s)               ← scheduled recurring tasks
```

> 💡 You almost never create bare Pods or ReplicaSets directly in production. Deployments, DaemonSets, Jobs, and CronJobs are the controllers you actually use.

---

## How to Run the Labs

Apply in order — the namespace must exist before any workload inside it:

```bash
kubectl apply -f 01-namespace/namespace.yml
kubectl apply -f 02-pods/pod.yml
kubectl apply -f 03-deployment/deployment.yml
kubectl apply -f 04-replicasets/replicasets.yml
kubectl apply -f 05-deamonset/DaemonSet.yml
kubectl apply -f 06-job/job.yml
kubectl apply -f 07-cron-job/cron-job.yml

# Verify everything
kubectl get all -n nginx
```

---

## Key Concepts for Interviews

| Concept | One-liner |
|---|---|
| Pod vs Container | A pod wraps containers and adds shared networking and storage |
| Why not bare Pods? | Pods don't self-heal — always use a controller on top |
| Deployment vs ReplicaSet | Deployment manages rolling updates and rollbacks; ReplicaSet just maintains replica count |
| DaemonSet vs ReplicaSet | DaemonSet guarantees one pod per node; ReplicaSet places N pods wherever the scheduler decides |
| Job vs Deployment | Job runs to completion and stops; Deployment runs forever |
| CronJob vs Job | CronJob creates a new Job on a schedule; Job runs once |
| `restartPolicy: Always` | Only valid in Deployments/ReplicaSets/DaemonSets — not in Jobs or CronJobs |
| `batch/v1` vs `apps/v1` | Jobs and CronJobs are in `batch/v1`; Deployments, ReplicaSets, DaemonSets are in `apps/v1` |

---

## Useful Commands

```bash
# See all workloads in the namespace at once
kubectl get all -n nginx

# Namespace
kubectl get namespaces
kubectl describe namespace nginx

# Pods
kubectl get pods -n nginx
kubectl describe pod <pod-name> -n nginx
kubectl logs <pod-name> -n nginx

# Deployment
kubectl get deployments -n nginx
kubectl rollout status deployment/<name> -n nginx
kubectl rollout undo deployment/<name> -n nginx

# ReplicaSet
kubectl get replicasets -n nginx

# DaemonSet
kubectl get daemonsets -n nginx

# Job
kubectl get jobs -n nginx
kubectl logs <job-pod-name> -n nginx

# CronJob
kubectl get cronjobs -n nginx
kubectl patch cronjob <name> -n nginx -p '{"spec":{"suspend":true}}'   # pause
kubectl create job manual-run --from=cronjob/<name> -n nginx           # trigger manually

# Cleanup
kubectl delete namespace nginx   # deletes everything inside it
```

---

## What I Learned

- A Namespace is not just a folder — it's a security and resource boundary
- Pods are ephemeral by design; production workloads always use a controller on top
- Deployments are the standard for stateless apps — they own a ReplicaSet and manage rollouts
- ReplicaSets are the engine behind Deployments, but you rarely interact with them directly
- DaemonSets are the right tool when the workload is node-scoped, not app-scoped
- Jobs are for tasks with a defined end — they track completions, not uptime
- CronJobs are schedulers, not workloads — the hierarchy is `CronJob → Job → Pod`
- `kubectl describe` is your best debugging tool across every resource type

---

## Author

**[Himanshu Kumar](https://www.linkedin.com/in/h1manshu-kumar/)** - Learning by building, documenting, and sharing 🚀

---
