# Lab 01 — Kubernetes Cluster Setup with KIND

> **Week 1 · Day 3** | Setting up a local multi-node Kubernetes cluster using KIND (Kubernetes IN Docker)

---

## What This Lab Covers

- Installing Docker, KIND, and kubectl on Linux (amd64 / arm64)
- Creating a **production-like local cluster** — 1 control-plane + 3 worker nodes
- Verifying the cluster is healthy and ready for workloads
- Understanding every line of the KIND config

---

## Why KIND?

| Tool | Nodes | Speed | Best For |
|---|---|---|---|
| **KIND** | Multi-node | Fast | Local dev, CI pipelines, learning |
| Minikube | Single-node | Medium | Quick demos |
| Kubeadm | Multi-node | Slow | Production-like bare metal |

KIND runs each Kubernetes node as a **Docker container** on your machine — no VMs, no cloud costs.

---

## Prerequisites

| Requirement | Minimum Version | Check |
|---|---|---|
| Linux (Ubuntu/Debian) | — | `uname -a` |
| Docker | 20.10+ | `docker --version` |
| RAM | 8 GB free | `free -h` |
| Disk | 10 GB free | `df -h` |

---

## Lab Structure

```
01-setup/
├── README.md                  ← you are here
├── kind-cluster/
│   └── config.yml             ← KIND cluster config (1 control-plane + 3 workers)
└── ../../setup-scripts/
    └── kind_install.sh        ← installs Docker + KIND + kubectl automatically
```

---

## Step 1 — Install Docker, KIND & kubectl

A single script handles everything. It detects your CPU architecture (x86_64 / arm64) and skips tools already installed.

```bash
chmod +x ../../setup-scripts/kind_install.sh
../../setup-scripts/kind_install.sh
```

**What the script does:**
1. Installs `docker.io` if not present, adds your user to the `docker` group
2. Downloads KIND `v0.29.0` binary for your architecture
3. Downloads the latest stable `kubectl` binary for your architecture

Verify everything is ready:

```bash
docker --version
kind --version
kubectl version --client
```
<img width="384" height="40" alt="image" src="https://github.com/user-attachments/assets/7e9238ac-e9ff-4b96-ad22-18c266164479" />   

<img width="211" height="41" alt="image" src="https://github.com/user-attachments/assets/b6acbe85-b6a0-4b5b-b5fb-cb16d144ed6c" />   

<img width="1008" height="119" alt="image" src="https://github.com/user-attachments/assets/2b6b3272-53f0-434e-9f45-de0d2311acc1" />

---

## Step 2 — Understand the Cluster Config

**File:** `kind-cluster/config.yml`

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4

nodes:
- role: control-plane                          # brain of the cluster
  image: kindest/node:v1.32.0@sha256:c48c62…

- role: worker                                 # runs your application pods
  image: kindest/node:v1.32.0@sha256:c48c62…

- role: worker
  image: kindest/node:v1.32.0@sha256:c48c62…

- role: worker                                 # this worker also exposes ports to localhost
  image: kindest/node:v1.32.0@sha256:c48c62…
  extraPortMappings:
  - containerPort: 80                          # maps container port 80 → localhost:80
    hostPort: 80
    protocol: TCP
  - containerPort: 443                         # maps container port 443 → localhost:443
    hostPort: 443
    protocol: TCP
```

**Key decisions explained:**

- **Pinned image with sha256** — guarantees reproducibility. Everyone running this config gets the exact same node image, no surprises.
- **3 worker nodes** — mirrors a real cluster. Lets you practice scheduling, taints, and node affinity meaningfully.
- **extraPortMappings on one worker** — allows Ingress controllers to receive traffic on ports 80/443 from your browser without extra port-forward commands.

---

## Step 3 — Create the Cluster

```bash
kind create cluster --name learning-cluster --config kind-cluster/config.yml
```

This takes 1–2 minutes. KIND will:
1. Pull the `kindest/node:v1.32.0` image (if not cached)
2. Start 4 Docker containers (1 control-plane + 3 workers)
3. Bootstrap Kubernetes inside them
4. Write credentials to `~/.kube/config` automatically

<img width="807" height="248" alt="image" src="https://github.com/user-attachments/assets/08fa9b86-b6d7-411d-ba5a-5e75aa16ad6d" />


---

## Step 4 — Verify the Cluster

```bash
# Confirm KIND sees the cluster
kind get clusters

# Confirm kubectl is pointed at it
kubectl cluster-info --context kind-learning-cluster

# Check all nodes are Ready
kubectl get nodes
```

**Expected output:**

```
NAME                             STATUS   ROLES           AGE   VERSION
learning-cluster-control-plane   Ready    control-plane   18m   v1.32.0
learning-cluster-worker          Ready    <none>          17m   v1.32.0
learning-cluster-worker2         Ready    <none>          17m   v1.32.0
learning-cluster-worker3         Ready    <none>          17m   v1.32.0
```
All 4 nodes in `Ready` state = cluster is healthy.   

<img width="724" height="122" alt="image" src="https://github.com/user-attachments/assets/11ea8f8d-5b31-45ef-b299-8837598c39ec" />

```bash
# Check system pods are all Running
kubectl get pods -A
```

You should see CoreDNS, kube-proxy, and the CNI pods all running. If any are in `Pending`, wait 30 seconds and re-run.   

<img width="1112" height="328" alt="image" src="https://github.com/user-attachments/assets/daa65e01-fda2-4541-af8e-19779cc1df46" />

---

## Step 5 — Explore the Cluster

```bash
# See every resource type Kubernetes supports
kubectl api-resources

# Describe the control-plane node
kubectl describe node learning-cluster-control-plane

# Check how kubectl knows which cluster to talk to
kubectl config view
kubectl config current-context
```

---

## Cluster Lifecycle Commands

```bash
# Stop the cluster (keeps config, frees RAM)
docker stop $(docker ps -q --filter name=learning-cluster)

# Start it again
docker start $(docker ps -aq --filter name=learning-cluster)

# Delete the cluster entirely
kind delete cluster --name learning-cluster

# Recreate from scratch
kind create cluster --name learning-cluster --config kind-cluster/config.yml
```

---

## Troubleshooting

**Node stuck in `NotReady`**
```bash
kubectl describe node <node-name>   # look at Conditions section
kubectl get pods -n kube-system     # check if CNI pods are running
```

**`ImagePullBackOff` on system pods**
```bash
# Check Docker has internet access
docker run --rm alpine ping -c 1 8.8.8.8
```

**Port 80/443 already in use**
```bash
sudo lsof -i :80    # find what's using the port
# Either stop that process or remove extraPortMappings from config.yml
```

**kubectl pointing at wrong cluster**
```bash
kubectl config get-contexts                        # list all contexts
kubectl config use-context kind-tws-cluster        # switch to this cluster
```

---

## What I Learned

- KIND nodes are Docker containers — `docker ps` shows them as running containers
- The `sha256` digest in the image tag pins the exact image layer, making the setup fully reproducible
- `extraPortMappings` is what makes Ingress work locally without `kubectl port-forward`
- `~/.kube/config` is updated automatically by KIND — no manual kubeconfig setup needed
- A `Ready` node means kubelet is healthy and the node can accept pod scheduling

---

## Key Commands Cheatsheet

```bash
kind create cluster --name <name> --config <file>   # create cluster
kind get clusters                                    # list clusters
kind delete cluster --name <name>                    # delete cluster
kubectl get nodes                                    # check node status
kubectl get pods -A                                  # all pods, all namespaces
kubectl cluster-info                                 # cluster endpoint info
kubectl config current-context                       # which cluster am I on?
```
---
## Author
**[Himanshu Kumar](https://www.linkedin.com/in/h1manshu-kumar/)** - Learning by building, documenting, and sharing 🚀

---
