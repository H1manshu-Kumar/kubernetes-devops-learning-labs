# Day 2: K8s Architecture Deep Dive & kubectl Basics

---

## Quick Recap from Day 1

- K8s = container orchestration platform (automates deploy, scale, manage).
- Follows **Master-Worker** architecture — Control Plane (brain) + Worker Nodes (muscle).
- Smallest unit = **Pod** (wraps one or more containers).

> If Day 1 is fuzzy, re-read `01-fundamentals.md` before continuing.

---

## Control Plane — Deep Dive

The Control Plane is the brain of the cluster. It makes all the decisions.

| Component | Role | Simple Analogy |
|---|---|---|
| **API Server** | Single entry point for all operations. Every `kubectl` command hits this first. | Reception desk |
| **etcd** | Distributed key-value store. Holds the entire cluster state. | Cluster's database |
| **Scheduler** | Assigns pods to worker nodes based on available resources. | HR assigning tasks to employees |
| **Controller Manager** | Watches cluster state. If actual ≠ desired, it fixes it (e.g., restarts a dead pod). | Supervisor |

> 💡 **etcd is critical** — if it goes down, the cluster loses its state. Always backed up in production.

### How a `kubectl` command flows

```
kubectl apply -f pod.yaml
       ↓
  API Server        ← validates & authenticates the request
       ↓
    etcd            ← stores the desired state
       ↓
  Scheduler         ← picks a node for the pod
       ↓
  kubelet           ← on the chosen node, pulls image & starts container
```

---

## Worker Node — Deep Dive

Every worker node runs these three components:

| Component | Role |
|---|---|
| **kubelet** | Agent on every node. Talks to API Server. Ensures containers are running as expected. |
| **kube-proxy** | Manages network rules on the node. Enables pod-to-pod and pod-to-service communication. |
| **Container Runtime** | Actually runs the containers. e.g., `containerd`, `CRI-O`. (Docker was deprecated in K8s v1.24) |

> 💡 Interview tip: *"What is kubelet?"*
> Answer: kubelet is an agent that runs on every worker node. It receives pod specs from the API Server and ensures the described containers are running and healthy.

---

## Cluster Setup Options (Overview)

You'll set these up hands-on in Day 3 & 4. For now, just know what each is:

| Tool | Use Case |
|---|---|
| **KIND** (K8s in Docker) | Runs K8s nodes as Docker containers. Best for local learning & CI. |
| **Minikube** | Single-node cluster locally or on EC2. Beginner friendly. |
| **Kubeadm** | Bootstraps a real multi-node cluster. Production-like setup. |
| **EKS / AKS / GKE** | Managed K8s on cloud (AWS / Azure / GCP). Used in production. |

---

## kubectl — The K8s CLI

`kubectl` is how you talk to your cluster. Every command goes through the **API Server**.

### Install kubectl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -sL https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/

# Verify
kubectl version --client
```

### kubeconfig — How kubectl knows which cluster to talk to

`kubectl` reads `~/.kube/config` to find the cluster endpoint and credentials.

```bash
kubectl config view              # see full kubeconfig
kubectl config get-contexts      # list all clusters/contexts
kubectl config current-context   # which cluster you're on
kubectl config use-context <name> # switch cluster
```

> A **context** = cluster + user + namespace. Lets you switch between environments quickly.

---

## Day 2 Hands-on

```bash
# Check kubectl is installed
kubectl version --client

# Explore every resource type K8s supports
kubectl api-resources

# Namespace-scoped resources only
kubectl api-resources --namespaced=true

# Get docs for any resource
kubectl explain pod
kubectl explain pod.spec
```

> 💡 `kubectl api-resources` shows every object K8s knows — pods, services, deployments, configmaps, secrets, etc. Great for discovery.

---

## Interview Quick-Fire

**Q: What is the API Server?**   
A: The single entry point for all K8s operations. Every `kubectl` command, every internal component communication goes through it. It validates requests and updates etcd.

**Q: What is etcd?**   
A: A distributed key-value store that holds the entire cluster state. It's K8s's source of truth.

**Q: What does the Controller Manager do?**   
A: It continuously watches the cluster. If the actual state drifts from the desired state (e.g., a pod crashes), it takes corrective action.

**Q: What is the difference between kubelet and kube-proxy?**   
A: kubelet manages the pod lifecycle on a node (starts/stops containers). kube-proxy manages networking rules on the node (routes traffic to the right pods).

**Q: Why was Docker deprecated as a container runtime in K8s?**   
A: K8s introduced the Container Runtime Interface (CRI) standard. Docker didn't implement CRI natively, so K8s dropped it in v1.24 in favour of `containerd` and `CRI-O` which do.

---

## Day 2 Checklist

- [ ] Can explain all 4 Control Plane components
- [ ] Can explain all 3 Worker Node components
- [ ] `kubectl` installed and `kubectl version --client` works
- [ ] Ran `kubectl api-resources` and explored the output
- [ ] Know what kubeconfig is and how contexts work

---

> ➡️ Next up — Day 3: KIND Cluster Setup (hands-on)
