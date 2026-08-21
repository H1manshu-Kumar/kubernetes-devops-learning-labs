# 🚀 Kubernetes Pods — Deep Dive

> **Lab Series:** Core Workloads → 02 Pods
> **Difficulty:** Beginner → Intermediate
> **Estimated Time:** 30–45 minutes
> **Focus:** Master the smallest deployable unit in Kubernetes — from theory to hands-on to interview-ready

---

## 📌 Table of Contents

1. [What is a Pod?](#what-is-a-pod)
2. [Pod vs Container — The Key Distinction](#pod-vs-container--the-key-distinction)
3. [Pod Architecture — What Lives Inside](#pod-architecture--what-lives-inside)
4. [Pod Manifest — Field by Field Breakdown](#pod-manifest--field-by-field-breakdown)
5. [Hands-On Lab](#hands-on-lab)
6. [Pod Lifecycle — Every State Explained](#pod-lifecycle--every-state-explained)
7. [Multi-Container Pod Patterns](#multi-container-pod-patterns)
8. [How Kubernetes Schedules a Pod](#how-kubernetes-schedules-a-pod)
9. [Pods in Production — What Changes](#pods-in-production--what-changes)
10. [Interview Q&A — Straight to the Point](#interview-qa--straight-to-the-point)
11. [Common Mistakes & Gotchas](#common-mistakes--gotchas)
12. [What's Next](#whats-next)

---

## What is a Pod?

A **Pod** is the **smallest deployable unit** in Kubernetes. Kubernetes does not schedule containers directly — it schedules Pods.

A Pod is a **logical wrapper** around one or more containers that guarantees:

| Shared Resource | What It Means in Practice |
|---|---|
| **Network Namespace** | All containers share the same IP and port space — they talk via `localhost` |
| **Storage (Volumes)** | Volumes are defined at the Pod level and mounted into individual containers |
| **Lifecycle** | All containers in a Pod are co-scheduled, co-located, and co-terminated |
| **IPC Namespace** | Containers can communicate via shared memory and semaphores |

> 💡 **Mental Model:** Think of a Pod as a **mini virtual machine**. The Pod is the VM, and the containers inside it are the processes running on that VM. They share the same network stack and can share disk.

---

## Pod vs Container — The Key Distinction

This is one of the most commonly misunderstood concepts for Kubernetes beginners.

```
Docker World:          Kubernetes World:
┌─────────────┐        ┌──────────────────────────────┐
│  Container  │   →    │  Pod                         │
│  (process)  │        │  ┌──────────┐ ┌──────────┐   │
└─────────────┘        │  │Container │ │Container │   │
                       │  │  (app)   │ │ (sidecar)│   │
                       │  └──────────┘ └──────────┘   │
                       │  Shared IP · Shared Volumes   │
                       └──────────────────────────────┘
```

| | Container | Pod |
|---|---|---|
| Scheduled by | Docker / container runtime | Kubernetes scheduler |
| Has its own IP | No (in K8s context) | Yes — one IP per Pod |
| Can share volumes natively | No | Yes |
| Unit of scaling | ❌ | ✅ |
| Unit of deployment | ❌ | ✅ |

---

## Pod Architecture — What Lives Inside

```
┌─────────────────────────────────────────────────────┐
│                        POD                          │
│                                                     │
│  ┌─────────────────┐    ┌─────────────────────┐     │
│  │  Init Container │    │   Main Container    │     │
│  │  (runs first,   │ →  │   (your app)        │     │
│  │   exits clean)  │    │                     │     │
│  └─────────────────┘    └─────────────────────┘     │
│                                                     │
│  ┌──────────────────────────────────────────────┐   │
│  │              Shared Volume (emptyDir)        │   │
│  └──────────────────────────────────────────────┘   │
│                                                     │
│  Pod IP: 10.244.0.5   ← one IP for the whole Pod    │
│  Node: worker-node-1                               │
└─────────────────────────────────────────────────────┘
```

**Key components:**
- **Pause container (infra container)** — Kubernetes injects a hidden `pause` container into every Pod. It holds the network namespace so that if your app container restarts, the Pod IP doesn't change.
- **Init containers** — Run sequentially before any app container starts. Used for setup tasks (DB migrations, config fetching, waiting for dependencies).
- **App containers** — Your actual workload. Run in parallel once all init containers succeed.

---

## Pod Manifest — Field by Field Breakdown

This is the `pod.yml` used in this lab:

```yaml
kind: Pod               # Kubernetes resource type
apiVersion: v1          # Pods belong to the core API group (v1)
metadata:
  name: nginx-pod       # Name must be unique within the namespace
  namespace: nginx      # Logical isolation — keeps resources organized
spec:
  containers:
  - name: nginx         # Container name — used in kubectl logs/exec
    image: nginx:latest # Image pulled from Docker Hub (or private registry)
    ports:
    - containerPort: 80 # INFORMATIONAL ONLY — does not expose the port
```

### Every Field Explained

| Field | Required | Purpose |
|---|---|---|
| `kind` | ✅ | Tells Kubernetes what type of object this is |
| `apiVersion` | ✅ | Which API group/version handles this resource |
| `metadata.name` | ✅ | Unique identifier within the namespace |
| `metadata.namespace` | ❌ | Defaults to `default` if omitted |
| `spec.containers[].name` | ✅ | Used in logs, exec, and events |
| `spec.containers[].image` | ✅ | The container image to run |
| `spec.containers[].ports[].containerPort` | ❌ | Documentation only — has no networking effect |

> ⚠️ **Common Misconception:** `containerPort: 80` does **NOT** open or expose port 80. It is purely metadata. To expose a Pod to traffic, you need a **Service**.

---

## Hands-On Lab

### Prerequisites
- A running Kubernetes cluster (Minikube, Kind, EKS, GKE, or AKS)
- `kubectl` configured and pointing to your cluster (`kubectl cluster-info` to verify)

---

### Step 1 — Create the Namespace

```bash
kubectl create namespace nginx
```

Namespaces provide logical isolation. All resources in this lab live under the `nginx` namespace.

---

### Step 2 — Apply the Pod Manifest

```bash
kubectl apply -f pod.yml
```

`apply` is declarative — Kubernetes reads your desired state and reconciles it. If the Pod already exists, it updates it. If not, it creates it.

---

### Step 3 — Verify the Pod is Running

```bash
kubectl get pod nginx-pod -n nginx
```

Expected output:
```
NAME        READY   STATUS    RESTARTS   AGE
nginx-pod   1/1     Running   0          10s
```

- `1/1` → 1 container running out of 1 total
- `RESTARTS: 0` → no crashes yet

---

### Step 4 — Inspect the Pod in Detail

```bash
# Full details — node assignment, IP, events, container state
kubectl describe pod nginx-pod -n nginx

# Stream live logs from the nginx container
kubectl logs nginx-pod -n nginx

# Follow logs in real time (like tail -f)
kubectl logs -f nginx-pod -n nginx

# Open a shell inside the running container
kubectl exec -it nginx-pod -n nginx -- /bin/bash

# Get the Pod's IP address
kubectl get pod nginx-pod -n nginx -o wide
```

---

### Step 5 — Test Connectivity from Inside the Cluster

```bash
# Spin up a temporary pod to curl nginx
kubectl run curl-test --image=curlimages/curl -it --rm --restart=Never \
  -n nginx -- curl http://<nginx-pod-ip>:80
```

Replace `<nginx-pod-ip>` with the IP from `kubectl get pod -o wide`.

---

### Step 6 — Clean Up

```bash
kubectl delete pod nginx-pod -n nginx

# Or delete the entire namespace (removes everything inside it)
kubectl delete namespace nginx
```

---

## Pod Lifecycle — Every State Explained

```
                    ┌─────────┐
                    │ Pending │  ← Scheduler found a node, pulling image
                    └────┬────┘
                         │
                    ┌────▼────┐
                    │ Running │  ← At least one container is active
                    └────┬────┘
              ┌──────────┼──────────┐
         ┌────▼────┐          ┌─────▼──────┐
         │Succeeded│          │   Failed   │
         │(exit: 0)│          │ (exit: ≠0) │
         └─────────┘          └────────────┘
```

| Phase | What's Happening | Common Cause |
|---|---|---|
| `Pending` | Accepted by API server, waiting to be scheduled or image is pulling | Node resource pressure, image pull delay |
| `ContainerCreating` | Node is pulling the image and setting up the container | First-time image pull |
| `Running` | At least one container is executing | Normal operation |
| `Succeeded` | All containers exited with code `0` | Batch jobs, one-off tasks |
| `Failed` | At least one container exited with non-zero code | App crash, OOM kill |
| `CrashLoopBackOff` | Container keeps crashing and restarting | Bad config, missing env vars, app bug |
| `ImagePullBackOff` | Kubernetes can't pull the container image | Wrong image name, private registry auth missing |
| `Unknown` | Node stopped reporting to the API server | Node failure, network partition |

> 💡 `CrashLoopBackOff` is not an official Pod phase — it's a container state. But it's one of the most important states to recognize and debug in real clusters.

---

## Multi-Container Pod Patterns

Pods can run more than one container. These are the three canonical patterns:

### 1. Sidecar Pattern
A helper container that enhances the main container without modifying it.

```
┌─────────────────────────────────┐
│  Pod                            │
│  ┌──────────────┐ ┌───────────┐ │
│  │  App         │ │  Log      │ │
│  │  (writes     │ │  Shipper  │ │
│  │   logs to    │ │  (reads   │ │
│  │   /var/log)  │ │   /var/log│ │
│  └──────────────┘ └───────────┘ │
│       └──── Shared Volume ────┘  │
└─────────────────────────────────┘
```

**Use cases:** Log shipping (Fluentd), metrics collection, service mesh proxies (Envoy/Istio)

---

### 2. Init Container Pattern
Runs to completion **before** the main container starts. Used for setup tasks.

```
Init Container          Main Container
┌──────────────┐        ┌──────────────┐
│ Wait for DB  │ ──✅──▶│  Start App   │
│ to be ready  │        │              │
└──────────────┘        └──────────────┘
```

**Use cases:** DB schema migrations, waiting for a dependency to be healthy, fetching secrets/config

---

### 3. Ambassador Pattern
A proxy container that handles external communication on behalf of the main container.

```
┌──────────────────────────────────┐
│  Pod                             │
│  ┌──────────┐    ┌─────────────┐ │
│  │  App     │───▶│  Ambassador │ │──▶ External DB / API
│  │(localhost│    │  (proxy)    │ │
│  │  :5432)  │    │             │ │
│  └──────────┘    └─────────────┘ │
└──────────────────────────────────┘
```

**Use cases:** Database connection pooling, routing to different environments

---

## How Kubernetes Schedules a Pod

Understanding this is critical for debugging `Pending` pods.

```
kubectl apply -f pod.yml
        │
        ▼
┌───────────────┐
│  API Server   │  ← Validates and stores the Pod spec in etcd
└───────┬───────┘
        │
        ▼
┌───────────────┐
│   Scheduler   │  ← Finds the best node based on:
│               │    - Available CPU/Memory
└───────┬───────┘    - Node selectors / Affinity rules
        │            - Taints and Tolerations
        ▼
┌───────────────┐
│    Kubelet    │  ← Agent on the selected node pulls the image
│  (on node)    │    and starts the container via the container runtime
└───────────────┘
```

---

## Pods in Production — What Changes

The `pod.yml` in this lab is intentionally minimal. In production, a Pod spec includes:

```yaml
spec:
  containers:
  - name: nginx
    image: nginx:1.25.3          # Pin to a specific version, never use :latest
    resources:
      requests:
        cpu: "100m"              # Minimum CPU the scheduler reserves
        memory: "128Mi"          # Minimum memory reserved
      limits:
        cpu: "500m"              # Hard cap — container throttled beyond this
        memory: "256Mi"          # Hard cap — container OOM-killed beyond this
    livenessProbe:               # Kubelet restarts container if this fails
      httpGet:
        path: /healthz
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 10
    readinessProbe:              # Pod removed from Service endpoints if this fails
      httpGet:
        path: /ready
        port: 80
      initialDelaySeconds: 3
      periodSeconds: 5
    env:
    - name: ENV
      value: "production"
  restartPolicy: Always          # Always | OnFailure | Never
```

> ⚠️ **Never use `image: nginx:latest` in production.** If the image is updated upstream, your next Pod restart will pull a different version — breaking reproducibility.

---

## Interview Q&A — Straight to the Point

**Q: What is a Pod in Kubernetes?**
The smallest deployable unit. A Pod wraps one or more containers and gives them a shared network namespace, shared storage volumes, and a unified lifecycle.

---

**Q: Can a Pod have multiple containers? When would you use that?**
Yes. The three patterns are Sidecar (helper alongside main app), Init Container (setup before app starts), and Ambassador (proxy for external communication). The most common real-world example is a service mesh proxy like Envoy running as a sidecar.

---

**Q: Do containers in a Pod share the same IP?**
Yes. All containers in a Pod share one network namespace — same IP, same port space. They communicate with each other via `localhost`. This is why two containers in the same Pod cannot both listen on port 80.

---

**Q: Are Pods ephemeral?**
Yes. Pods are mortal by design. A bare Pod that dies is gone — Kubernetes does not reschedule it. This is why you almost never create bare Pods in production. You use a `Deployment` which manages a `ReplicaSet` which ensures the desired number of Pods are always running.

---

**Q: What is the difference between a Pod and a Deployment?**
A Pod is a single instance with no self-healing. A Deployment is a higher-level controller that manages a ReplicaSet, which ensures N replicas of a Pod are always running. Deployments also enable rolling updates and rollbacks.

---

**Q: What is `CrashLoopBackOff`?**
It means a container is repeatedly crashing and Kubernetes is backing off before restarting it again (with exponential backoff: 10s → 20s → 40s → ... up to 5 minutes). Root causes: application bug, missing environment variables, misconfigured config, or the container exits immediately after starting.

---

**Q: What is the difference between `livenessProbe` and `readinessProbe`?**
- `livenessProbe` — answers "is this container alive?" If it fails, kubelet **restarts** the container.
- `readinessProbe` — answers "is this container ready to serve traffic?" If it fails, the Pod is **removed from the Service's endpoint list** but NOT restarted.

---

**Q: What is the `pause` container?**
Every Pod has a hidden infrastructure container called `pause` (or "sandbox" container). It holds the network namespace for the Pod. This means if your app container restarts, the Pod IP stays the same because the `pause` container never restarts.

---

**Q: What is `restartPolicy` and what are the options?**
It controls what kubelet does when a container exits.
- `Always` (default) — always restart, regardless of exit code. Used for long-running services.
- `OnFailure` — restart only if exit code is non-zero. Used for batch jobs.
- `Never` — never restart. Used for one-off tasks where you want to inspect the result.

---

**Q: What happens to a Pod when its node fails?**
The Pod enters `Unknown` state. If the Pod is managed by a Deployment/ReplicaSet, the controller will reschedule it on a healthy node after the node is marked `NotReady` (default: ~5 minutes). A bare Pod is lost permanently.

---

## Common Mistakes & Gotchas

| Mistake | Why It's Wrong | Fix |
|---|---|---|
| Using `image: nginx:latest` | Unpredictable — image can change on next pull | Pin to a specific version: `nginx:1.25.3` |
| Thinking `containerPort` exposes the port | It's purely informational | Create a `Service` to expose traffic |
| Creating bare Pods in production | No self-healing, no rolling updates | Use a `Deployment` |
| No resource `requests` or `limits` | Pod can starve other workloads or get OOM-killed unpredictably | Always set both |
| No liveness/readiness probes | Kubernetes can't detect a broken app | Add probes for production workloads |
| Storing secrets in env vars directly | Secrets visible in Pod spec | Use Kubernetes `Secret` objects |

---

## What's Next

| Lab | Topic | Why It Matters |
|---|---|---|
| [03 — ReplicaSets](../03-replicasets/) | Ensure N copies of a Pod always run | Foundation of self-healing |
| [04 — Deployments](../04-deployments/) | Rolling updates, rollbacks | How you run Pods in production |
| [05 — Services](../05-services/) | Expose Pods to network traffic | Pods need Services to be reachable |
| [06 — ConfigMaps & Secrets](../06-configmaps-secrets/) | Inject config and sensitive data | Decouple config from code |

---

## Files in This Lab

```
02-pods/
├── pod.yml       ← Minimal nginx Pod manifest (intentionally simple)
└── README.md     ← This file
```

---

<div align="center">

**Built while learning Kubernetes hands-on.**
*The best way to understand Pods is to break them, fix them, and break them again.*

</div>
