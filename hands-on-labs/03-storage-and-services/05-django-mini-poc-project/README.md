# 🐍 Django Notes App — Kubernetes Mini POC

> **Lab Series:** Storage & Services → 05 Django Mini POC Project    
> **Difficulty:** Beginner → Intermediate    
> **Estimated Time:** 20–30 minutes    
> **Focus:** Deploy a real Django application on Kubernetes using Namespace, Deployment, and Service    

---

## 📌 Table of Contents

1. [What This Project Does](#what-this-project-does)
2. [Architecture Overview](#architecture-overview)
3. [Manifest Breakdown](#manifest-breakdown)
4. [Hands-On Lab](#hands-on-lab)
5. [Key Concepts Reinforced](#key-concepts-reinforced)
6. [Interview Q&A](#interview-qa)
7. [Common Mistakes & Gotchas](#common-mistakes--gotchas)

---

## What This Project Does

This is a minimal Kubernetes deployment of a real Django notes application. Instead of deploying a generic nginx or busybox container, this lab uses an actual web application to reinforce how Kubernetes objects work together in a realistic scenario.

| Object | Purpose |
|---|---|
| `Namespace` | Isolates the app from other workloads in the cluster |
| `Deployment` | Manages the Django app pod — ensures it's running and handles restarts |
| `Service` | Exposes the Django app inside the cluster on port `8000` |

> 💡 **Why this matters:** Every production Kubernetes workload follows this exact pattern — Namespace → Deployment → Service. This project makes that pattern concrete with a real app.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────┐
│               Namespace: notes-app                  │
│                                                     │
│   ┌─────────────────────────────────────────────┐   │
│   │           Deployment: notes-app-deployment  │   │
│   │                                             │   │
│   │   ┌─────────────────────────────────────┐   │   │
│   │   │  Pod: notes-app (replicas: 1)       │   │   │
│   │   │  Image: himan5hu/notes-app-k8s      │   │   │
│   │   │  Container Port: 8000               │   │   │
│   │   └─────────────────────────────────────┘   │   │
│   └─────────────────────────────────────────────┘   │
│                                                     │
│   ┌─────────────────────────────────────────────┐   │
│   │  Service: notes-app-service (ClusterIP)     │   │
│   │  Port: 8000 → TargetPort: 8000              │   │
│   │  Selector: app=notes-app                    │   │
│   └─────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

Traffic flow: `Client → Service (ClusterIP:8000) → Pod (containerPort:8000)`

---

## Manifest Breakdown

### `namespace.yml`

```yaml
kind: Namespace
apiVersion: v1
metadata:
  name: notes-app
```

| Field | Purpose |
|---|---|
| `kind: Namespace` | Creates an isolated virtual cluster within the cluster |
| `metadata.name` | All resources in this project use `namespace: notes-app` |

> 💡 Namespaces prevent resource name collisions. Two teams can both have a `Deployment` named `notes-app` as long as they're in different namespaces.

---

### `deployment.yml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: notes-app-deployment
  labels:
    app: notes-app
  namespace: notes-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: notes-app
  template:
    metadata:
      labels:
        app: notes-app
    spec:
      containers:
      - name: notes-app
        image: himan5hu/notes-app-k8s
        ports:
        - containerPort: 8000
```

| Field | Purpose |
|---|---|
| `namespace: notes-app` | Scopes this Deployment to the `notes-app` namespace |
| `replicas: 1` | Runs one pod — scale up by increasing this number |
| `selector.matchLabels` | The Deployment manages pods with label `app: notes-app` |
| `template.metadata.labels` | Labels applied to the pod — must match `selector.matchLabels` |
| `image: himan5hu/notes-app-k8s` | The Django app Docker image pulled from Docker Hub |
| `containerPort: 8000` | Django's default port — informational, doesn't publish the port |

> ⚠️ `containerPort` is documentation only — it doesn't expose the port. The Service does the actual routing.

---

### `service.yml`

```yaml
kind: Service
apiVersion: v1
metadata:
  name: notes-app-service
  namespace: notes-app
spec:
  selector:
    app: notes-app
  ports:
    - protocol: TCP
      port: 8000
      targetPort: 8000
  type: ClusterIP
```

| Field | Purpose |
|---|---|
| `namespace: notes-app` | Service must be in the same namespace as the pods it targets |
| `selector: app: notes-app` | Routes traffic to pods with this label |
| `port: 8000` | Port the Service listens on |
| `targetPort: 8000` | Port on the pod the traffic is forwarded to |
| `type: ClusterIP` | Internal-only access — reachable within the cluster, not from outside |

> 💡 `ClusterIP` is the default Service type. It gives the Service a stable internal IP. Even if the pod is replaced, the Service IP stays the same — that's the point.

---

## Hands-On Lab

### Prerequisites
- A running Kubernetes cluster (Minikube or Kind)
- `kubectl` configured (`kubectl cluster-info` to verify)

---

### Step 1 — Create the Namespace

```bash
kubectl apply -f namespace.yml
kubectl get namespace notes-app
```

Expected output:
```
NAME        STATUS   AGE
notes-app   Active   3s
```
<img width="598" height="47" alt="image" src="https://github.com/user-attachments/assets/994f4cda-d3de-4545-b132-ba4ba0551d04" />

<img width="613" height="58" alt="image" src="https://github.com/user-attachments/assets/d77f10b3-0553-4e82-8266-1272db417120" />

---

### Step 2 — Deploy the Django App

```bash
kubectl apply -f deployment.yml
kubectl get deployment -n notes-app
```

Expected output:
```
NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
notes-app-deployment     1/1     1            1           10s
```
<img width="615" height="44" alt="image" src="https://github.com/user-attachments/assets/522e57f4-99de-4f31-bedc-953a73f2a400" /> </br>

<img width="652" height="59" alt="image" src="https://github.com/user-attachments/assets/1b0e62b9-c381-4ce2-88e7-7adf167c9c1b" />

Check the pod:
```bash
kubectl get pods -n notes-app
```
<img width="779" height="60" alt="image" src="https://github.com/user-attachments/assets/dd795fc6-bfba-4668-a11d-f806bc7bac10" />

Expected output:
```
NAME                                    READY   STATUS    RESTARTS   AGE
notes-app-deployment-xxxxxxxxx-xxxxx    1/1     Running   0          15s
```

> 💡 If `STATUS` is `ContainerCreating`, the image is still being pulled. Wait a few seconds and re-run.

---

### Step 3 — Create the Service

```bash
kubectl apply -f service.yml
kubectl get svc -n notes-app
```
<img width="779" height="60" alt="image" src="https://github.com/user-attachments/assets/3feda843-8232-4de2-be61-8ebca09c5acf" />

Expected output:
```
NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
notes-app-service    ClusterIP   10.96.x.x       <none>        8000/TCP   5s
```
---

### Step 4 — Verify the App is Reachable

Since the Service is `ClusterIP` (internal only), test it from inside the cluster:

```bash
kubectl run curl-test \
  --image=curlimages/curl \
  --restart=Never \
  --rm -it \
  -n notes-app \
  -- curl http://notes-app-service:8000
```

Or use port-forward to access it from your local machine:

```bash
kubectl port-forward svc/notes-app-service 8000:8000 -n notes-app
```
<img width="1172" height="121" alt="image" src="https://github.com/user-attachments/assets/79b0c327-90be-4d92-b692-6ed34832e955" /> </br>

Then open `http://localhost:8000` in your browser.

<img width="1279" height="730" alt="image" src="https://github.com/user-attachments/assets/bbb08354-497a-4213-8fc5-06f3a2f1a663" />

---

### Step 5 — Inspect the Deployment

```bash
# Full details of the deployment
kubectl describe deployment notes-app-deployment -n notes-app

# View pod logs
kubectl logs -l app=notes-app -n notes-app

# Exec into the running pod
kubectl exec -it -n notes-app \
  $(kubectl get pod -n notes-app -l app=notes-app -o jsonpath='{.items[0].metadata.name}') \
  -- /bin/sh
```
<img width="1279" height="649" alt="image" src="https://github.com/user-attachments/assets/70fd66aa-f6ec-49ba-bec8-99446075d370" />    

---

### Step 6 — Scale the Deployment

```bash
kubectl scale deployment notes-app-deployment --replicas=3 -n notes-app
kubectl get pods -n notes-app
```
<img width="1020" height="42" alt="image" src="https://github.com/user-attachments/assets/f5341ee5-205a-4dc7-a55f-1388c41f907e" /> </br>

<img width="873" height="97" alt="image" src="https://github.com/user-attachments/assets/ff735e26-750b-43d3-9fb6-07e1b1252ed6" />

Expected output:
```
NAME                                    READY   STATUS    RESTARTS   AGE
notes-app-deployment-xxxxxxxxx-aaaaa    1/1     Running   0          2m
notes-app-deployment-xxxxxxxxx-bbbbb    1/1     Running   0          5s
notes-app-deployment-xxxxxxxxx-ccccc    1/1     Running   0          5s
```

> 💡 The Service automatically load-balances across all 3 pods — no config change needed. This is the power of label-based selection.

Scale back down:
```bash
kubectl scale deployment notes-app-deployment --replicas=1 -n notes-app
```
<img width="1013" height="45" alt="image" src="https://github.com/user-attachments/assets/54c35485-43ea-4f5d-8c8e-a065a97f000f" /> </br>

<img width="816" height="100" alt="image" src="https://github.com/user-attachments/assets/5758d379-40e8-45c3-b007-e615cb102b09" />


---

### Step 7 — Clean Up

```bash
kubectl delete namespace notes-app
```

> 💡 Deleting the namespace deletes everything inside it — the Deployment, pods, and Service — in one command.

---

## Key Concepts Reinforced

| Concept | How It Appears in This Lab |
|---|---|
| Namespace isolation | All 3 resources scoped to `notes-app` namespace |
| Label-based selection | Deployment manages pods via `app: notes-app`; Service routes to them via the same label |
| Deployment self-healing | Delete a pod manually — the Deployment recreates it automatically |
| Service stability | Pod IP changes on restart; Service IP stays constant |
| ClusterIP | App is internal-only — use port-forward or ingress for external access |

---

## Interview Q&A

**Q: Why use a Namespace for this app?**
Namespaces provide isolation. If you have multiple apps or teams in the same cluster, namespaces prevent name collisions and allow separate RBAC policies and resource quotas per team/app.

---

**Q: What happens if the Django pod crashes?**
The Deployment controller detects the pod is gone and immediately schedules a replacement. This is the core value of using a Deployment over a bare Pod.

---

**Q: Why is the Service type `ClusterIP` and not `NodePort` or `LoadBalancer`?**
`ClusterIP` is for internal service-to-service communication. For this POC, we access it via `port-forward`. In production, you'd put an Ingress in front of the Service for external HTTP traffic rather than exposing it directly as `NodePort` or `LoadBalancer`.

---

**Q: How does the Service know which pods to send traffic to?**
Via label selectors. The Service has `selector: app: notes-app`. Any pod in the same namespace with that label receives traffic. The Deployment ensures all its pods carry that label via `template.metadata.labels`.

---

**Q: What is the difference between `port` and `targetPort` in a Service?**
`port` is what the Service listens on (what clients connect to). `targetPort` is the port on the pod the traffic is forwarded to. They can be different — e.g., Service on port `80`, pod on port `8000`.

---

## Common Mistakes & Gotchas

| Mistake | Why It's Wrong | Fix |
|---|---|---| 
| Service and Deployment in different namespaces | Service can't find the pods — selector matches nothing | Ensure both have `namespace: notes-app` |
| `selector` in Service doesn't match pod labels | Service has no endpoints — traffic goes nowhere | `kubectl get endpoints -n notes-app` to verify |
| Applying manifests without creating the namespace first | Deployment and Service fail with `namespace not found` | Always apply `namespace.yml` first |
| Expecting `ClusterIP` to be accessible from your browser | ClusterIP is internal-only | Use `kubectl port-forward` for local access |
| Forgetting `-n notes-app` in kubectl commands | Commands run against the `default` namespace — resources not found | Always pass `-n notes-app` or set the namespace context |

---

## Files in This Lab

```
05-django-mini-poc-project/
├── namespace.yml     ← Creates the notes-app namespace
├── deployment.yml    ← Deploys the Django notes app (1 replica)
├── service.yml       ← ClusterIP Service on port 8000
└── README.md         ← This file
```

---

## ✍️ Author

**[Himanshu Kumar](https://www.linkedin.com/in/h1manshu-kumar/)** - Learning by building, documenting, and sharing 🚀

---

<div align="center">

**Built while learning Kubernetes hands-on.**
*The best way to understand Deployments and Services is to deploy something real.*

</div>
