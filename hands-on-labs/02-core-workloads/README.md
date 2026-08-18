# Lab 02 — Core Kubernetes Workloads

> **Week 2** | Namespaces, Pods, Deployments, Services, ConfigMaps, Secrets and more — the building blocks every Kubernetes engineer works with daily

---

## What This Lab Covers

| # | Topic | What You'll Learn |
|---|---|---|
| 01 | **Namespaces** | Isolate workloads, scope resources, organise environments |
| 02 | **Pods** | The smallest deployable unit — containers, ports, lifecycle |
| 03 | **Deployments** *(coming soon)* | Declarative rollouts, scaling, self-healing |
| 04 | **Services** *(coming soon)* | Stable networking — ClusterIP, NodePort, LoadBalancer |
| 05 | **ConfigMaps** *(coming soon)* | Externalise configuration from container images |
| 06 | **Secrets** *(coming soon)* | Manage sensitive data the Kubernetes way |

---

## Why This Lab Matters

> Every production Kubernetes cluster — whether on AWS EKS, Google GKE, or Azure AKS — is built from exactly these primitives.

Mastering this lab means you can:
- Read and write any real-world Kubernetes manifest
- Understand what happens when a pod crashes, scales, or gets updated
- Speak confidently in interviews about how Kubernetes actually works

---

## Lab Structure

```
02-core-workloads/
├── README.md                  ← you are here
├── 01-namespace/
│   └── namespace.yml          ← creates the "nginx" namespace
└── 02-pods/
    └── pod.yml                ← deploys an nginx pod into that namespace
```

---

## 01 — Namespaces

**File:** `01-namespace/namespace.yml`

```yaml
kind: Namespace
apiVersion: v1
metadata:
  name: nginx
```

A Namespace is a **virtual cluster inside your cluster**. It lets you:
- Isolate teams, apps, or environments (`dev`, `staging`, `prod`) on the same cluster
- Apply resource quotas and RBAC policies per namespace
- Avoid name collisions — two teams can both have a pod named `api` in different namespaces

```bash
# Apply the namespace
kubectl apply -f 01-namespace/namespace.yml

# Verify it exists
kubectl get namespaces
```

**Expected output:**
```
NAME              STATUS   AGE
default           Active   ...
kube-system       Active   ...
nginx             Active   5s     ← your namespace
```

---

## 02 — Pods

**File:** `02-pods/pod.yml`

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: nginx-pod
  namespace: nginx
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
```

A Pod is the **smallest deployable unit in Kubernetes**. It wraps one or more containers that share:
- The same network namespace (same IP address)
- The same storage volumes
- The same lifecycle

**Key decisions explained:**
- `namespace: nginx` — scopes this pod to the namespace we created above, not `default`
- `image: nginx:latest` — pulls the official nginx image from Docker Hub
- `containerPort: 80` — documents the port the container listens on (informational, does not expose it externally)

```bash
# Apply the pod
kubectl apply -f 02-pods/pod.yml

# Verify the pod is Running
kubectl get pods -n nginx

# Inspect the pod in detail
kubectl describe pod nginx-pod -n nginx

# Quick connectivity test
kubectl port-forward pod/nginx-pod 8080:80 -n nginx
# then open http://localhost:8080 in your browser
```

**Expected output:**
```
NAME        READY   STATUS    RESTARTS   AGE
nginx-pod   1/1     Running   0          10s
```

---

## Namespace + Pod — How They Connect

```
Cluster
└── Namespace: nginx
    └── Pod: nginx-pod
        └── Container: nginx (port 80)
```

The `namespace: nginx` field in `pod.yml` is what ties the two manifests together. Without the namespace existing first, the pod apply would fail.

---

## Key Concepts for Interviews

| Concept | One-liner |
|---|---|
| Pod vs Container | A pod is a wrapper around containers — it adds networking and storage context |
| Why not deploy bare pods? | Pods don't self-heal. If a node dies, the pod is gone. Deployments fix this |
| Namespace isolation | Resources in different namespaces can't reference each other by short name |
| `containerPort` | Purely informational — it does NOT expose the port. A Service does that |

---

## Useful Commands

```bash
# Namespace operations
kubectl get namespaces
kubectl describe namespace nginx
kubectl delete namespace nginx          # also deletes everything inside it

# Pod operations
kubectl get pods -n nginx
kubectl describe pod nginx-pod -n nginx
kubectl logs nginx-pod -n nginx
kubectl exec -it nginx-pod -n nginx -- /bin/bash
kubectl delete pod nginx-pod -n nginx
```

---

## What I Learned

- A Namespace is not just a folder — it's a security and resource boundary
- Pods are ephemeral by design; production workloads always use a controller (Deployment, StatefulSet, etc.) on top
- `kubectl describe` is your best debugging tool — it shows events, resource limits, and scheduling decisions
- Applying manifests in order matters: namespace first, then the resources that live inside it

---

## What's Next

The next sections will build on this foundation:
- **Deployments** — wrap pods with self-healing, rolling updates, and replica management
- **Services** — give pods a stable network identity so other workloads can reach them
- **ConfigMaps & Secrets** — decouple configuration and credentials from your container images

---

## Author
**[Himanshu Kumar](https://www.linkedin.com/in/h1manshu-kumar/)** - Learning by building, documenting, and sharing 🚀

---
