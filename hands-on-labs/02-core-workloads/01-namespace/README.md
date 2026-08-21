# Kubernetes Namespaces

Namespaces let you split one Kubernetes cluster into multiple virtual clusters. Each namespace is its own isolated space where you can run workloads, set access controls, and apply resource limits independently.

A simple real-world example: your team runs `dev`, `staging`, and `prod` on the same cluster. Namespaces keep them separate without needing three different clusters.

## Default Namespaces in Every Cluster

| Namespace | What it's for |
|---|---|
| `default` | Where resources land if you don't specify a namespace |
| `kube-system` | Core Kubernetes components like DNS, scheduler, controller-manager |
| `kube-public` | Readable by everyone, rarely used in practice |
| `kube-node-lease` | Tracks node heartbeats to detect failures faster |

## Why Namespaces Matter

- Separate environments (dev, staging, prod) on a single cluster
- Prevent naming conflicts — two teams can both have a `backend` deployment
- Apply `ResourceQuota` to limit CPU/memory per team or app
- Scope RBAC permissions — a developer can have access to `dev` but not `prod`

## Manifest

```yaml
kind: Namespace
apiVersion: v1
metadata:
  name: nginx
```

This creates a namespace called `nginx`. All nginx-related workloads (pods, services, deployments) will be deployed here.

## Commands

```bash
# Create from manifest
kubectl apply -f namespace.yml

# Create directly without a manifest
kubectl create namespace nginx

# List all namespaces
kubectl get namespaces

# Deploy a resource into the nginx namespace
kubectl apply -f deployment.yml -n nginx

# View pods inside the namespace
kubectl get pods -n nginx

# Switch your default namespace so you don't have to type -n every time
kubectl config set-context --current --namespace=nginx

# Delete the namespace and everything inside it
kubectl delete namespace nginx
```

## Namespace-scoped vs Cluster-scoped Resources

Not everything in Kubernetes lives inside a namespace.

**Namespace-scoped** (affected by -n flag): `Pod`, `Deployment`, `Service`, `ConfigMap`, `Secret`, `ServiceAccount`

**Cluster-scoped** (global, no namespace): `Node`, `PersistentVolume`, `ClusterRole`, `Namespace` itself

You can check if a resource is namespaced by running:
```bash
kubectl api-resources --namespaced=true
kubectl api-resources --namespaced=false
```

## Things Worth Knowing for Interviews

- Namespaces do not provide network isolation on their own. Two pods in different namespaces can still talk to each other. Use `NetworkPolicy` to restrict that.
- DNS works across namespaces. A service in `nginx` namespace is reachable at `service-name.nginx.svc.cluster.local` from any other namespace.
- Deleting a namespace deletes everything inside it. There is no undo.
- `ResourceQuota` and `LimitRange` are applied at the namespace level, making them a good way to enforce guardrails per team.

## Folder Structure

```
01-namespace/
  namespace.yml   # Manifest to create the nginx namespace
  README.md       # This file
```
