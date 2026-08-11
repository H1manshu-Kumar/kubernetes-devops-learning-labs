# Kubernetes Fundamentals
---

## History

- Kubernetes started as an internal project at Google called **Borg** — Google used it to manage billions of containers per week internally.
- Google open-sourced it in **2014** and donated it to the **CNCF (Cloud Native Computing Foundation)**.
- The name "Kubernetes" comes from Greek, meaning **helmsman** or **pilot** (that's why the logo is a ship's wheel 🚢).
- It is often shortened to **K8s** — the "8" replaces the 8 letters between "K" and "s".
- K8s **graduated** from CNCF in 2018, meaning it became production-ready and widely trusted.
- Today it is the **industry standard** for container orchestration.

> 💡 Interview tip: If asked "What is Kubernetes?", say — *"Kubernetes is an open-source container orchestration platform, originally developed by Google, that automates deployment, scaling, and management of containerized applications."*

---

## Why Kubernetes?

Before Kubernetes, running containers in production was painful. Here's why K8s was needed:

| Problem without K8s | How K8s solves it |
|---|---|
| Containers crash and no one restarts them | K8s auto-restarts failed containers (self-healing) |
| Traffic spikes — app can't handle load | K8s auto-scales pods up/down |
| Deploying new version causes downtime | K8s does rolling updates with zero downtime |
| Running containers on multiple servers manually | K8s schedules containers across nodes automatically |
| Hard to manage configs and secrets | K8s has built-in ConfigMaps and Secrets |
| No built-in load balancing | K8s has Services for internal load balancing |

**In short:** Kubernetes lets you focus on your application, not on managing infrastructure manually.

---

## Monolith vs Microservices

Understanding this is key to understanding *why* K8s exists.

### Monolith Architecture
- The entire application (UI, business logic, database layer) is **one big deployable unit**.
- Easy to develop initially, but becomes a problem as the app grows.

**Problems with Monolith:**
- One bug can bring down the entire app.
- Scaling means scaling the whole app, even if only one feature is under load.
- Deployments are risky and slow — you redeploy everything for a small change.
- Hard for large teams to work on the same codebase.

### Microservices Architecture
- The application is split into **small, independent services** (e.g., user-service, payment-service, notification-service).
- Each service runs in its own container, can be deployed independently, and communicates over APIs.

**Benefits of Microservices:**
- Scale only the service that needs it (e.g., scale payment-service during a sale).
- One service failing doesn't crash the whole app.
- Teams can work on different services independently.
- Faster deployments.

**The catch:** Managing hundreds of containers across multiple services and servers is complex — that's exactly where **Kubernetes comes in**.

> 💡 Think of it this way: Microservices created the *need* for Kubernetes.

---

## Architecture Basics

Kubernetes follows a **Master-Worker** architecture. There are two types of nodes:

1. **Control Plane (Master Node)** — The brain. Makes all decisions.
2. **Worker Nodes** — The muscle. Actually runs your application containers.

```
                    ┌─────────────────────────────┐
                    │       Control Plane          │
                    │  ┌──────────┐ ┌───────────┐  │
                    │  │   API    │ │ Scheduler │  │
                    │  │  Server  │ └───────────┘  │
                    │  └──────────┘ ┌───────────┐  │
                    │  ┌──────────┐ │  etcd     │  │
                    │  │Controller│ │(key-value │  │
                    │  │ Manager  │ │   store)  │  │
                    │  └──────────┘ └───────────┘  │
                    └────────────┬────────────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              │                  │                  │
   ┌──────────▼──────┐  ┌────────▼────────┐  ┌─────▼───────────┐
   │   Worker Node 1  │  │  Worker Node 2  │  │  Worker Node 3  │
   │  ┌────────────┐  │  │  ┌──────────┐  │  │  ┌──────────┐   │
   │  │  kubelet   │  │  │  │ kubelet  │  │  │  │ kubelet  │   │
   │  ├────────────┤  │  │  ├──────────┤  │  │  ├──────────┤   │
   │  │ kube-proxy │  │  │  │kube-proxy│  │  │  │kube-proxy│   │
   │  ├────────────┤  │  │  ├──────────┤  │  │  ├──────────┤   │
   │  │  Pod(s)    │  │  │  │  Pod(s)  │  │  │  │  Pod(s)  │   │
   │  └────────────┘  │  │  └──────────┘  │  │  └──────────┘   │
   └──────────────────┘  └───────────────┘  └─────────────────┘
```

### Control Plane Components

| Component | What it does |
|---|---|
| **API Server** | The front door of K8s. Every command (`kubectl`) goes through it. |
| **etcd** | A key-value store that holds the entire cluster state/config. Think of it as K8s's database. |
| **Scheduler** | Decides *which worker node* a new pod should run on based on available resources. |
| **Controller Manager** | Watches the cluster state and makes sure the *actual state* matches the *desired state* (e.g., if a pod dies, it creates a new one). |

### Worker Node Components

| Component | What it does |
|---|---|
| **kubelet** | An agent running on every worker node. It talks to the API server and makes sure containers are running as expected. |
| **kube-proxy** | Handles networking on the node — manages network rules so pods can communicate with each other and the outside world. |
| **Container Runtime** | The software that actually runs containers (e.g., `containerd`, `CRI-O`). Docker was used earlier but K8s moved away from it. |

### Key Concepts to Know

- **Pod** — The smallest deployable unit in K8s. A pod wraps one or more containers.
- **Node** — A physical or virtual machine in the cluster.
- **Cluster** — A set of nodes managed by K8s (1 control plane + multiple worker nodes).
- **Namespace** — A way to logically separate resources within a cluster (e.g., dev, staging, prod namespaces).

> 💡 Interview tip: A common question is *"What is the difference between a Pod and a Container?"*
> Answer: A container is a running process. A Pod is a K8s wrapper around one or more containers that share the same network and storage. K8s doesn't manage containers directly — it manages Pods.

---
