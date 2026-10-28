# 🔐 Kubernetes ConfigMaps & Secrets — Deep Dive

> **Lab Series:** Advanced Features & Configs → 02 ConfigMaps & Secrets
> **Difficulty:** Beginner → Intermediate
> **Estimated Time:** 30–45 minutes
> **Focus:** Understand how to decouple configuration from container images using ConfigMaps and Secrets

---

## 📌 Table of Contents

1. [Why ConfigMaps & Secrets Exist](#why-configmaps--secrets-exist)
2. [ConfigMap vs Secret — Know the Difference](#configmap-vs-secret--know-the-difference)
3. [How ConfigMaps & Secrets Work](#how-configmaps--secrets-work)
4. [Manifest Breakdown — Field by Field](#manifest-breakdown--field-by-field)
5. [Consuming ConfigMaps & Secrets in Pods](#consuming-configmaps--secrets-in-pods)
6. [Hands-On Lab](#hands-on-lab)
7. [ConfigMaps & Secrets in Production — What Changes](#configmaps--secrets-in-production--what-changes)
8. [Interview Q&A — Straight to the Point](#interview-qa--straight-to-the-point)
9. [Common Mistakes & Gotchas](#common-mistakes--gotchas)

---

## Why ConfigMaps & Secrets Exist

Hardcoding configuration (database names, URLs, passwords) directly into container images is a bad practice — it means rebuilding the image every time a config value changes, and it leaks sensitive data into version control.

Kubernetes solves this with two dedicated objects:

| Problem | Without ConfigMap/Secret | With ConfigMap/Secret |
|---|---|---|
| DB name changes across environments | Rebuild image per environment | Change the ConfigMap, pods pick it up |
| Password in image or source code | ❌ Security risk | ✅ Stored separately, injected at runtime |
| Same image, different configs | ❌ Not possible cleanly | ✅ Swap ConfigMap per namespace/env |
| Audit who changed a credential | ❌ Buried in Dockerfile | ✅ Kubernetes RBAC + audit logs |

> 💡 **The Rule:** Your container image should be environment-agnostic. All environment-specific values — database names, feature flags, passwords, API keys — belong in ConfigMaps or Secrets, not in the image.

---

## ConfigMap vs Secret — Know the Difference

Both decouple config from images, but they serve different purposes:

| Feature | ConfigMap | Secret |
|---|---|---|
| Purpose | Non-sensitive configuration | Sensitive data (passwords, tokens, keys) |
| Storage in etcd | Plaintext | Base64-encoded (not encrypted by default) |
| Shown in `kubectl get` | ✅ Values visible | ❌ Values hidden |
| Use case | DB name, env flags, config files | DB password, API keys, TLS certs |
| Encryption at rest | ❌ Not by default | ⚠️ Only with etcd encryption enabled |

> ⚠️ **Critical:** Base64 is **encoding**, not **encryption**. A Secret's value can be decoded by anyone with access to the cluster. For true security, enable etcd encryption at rest and use RBAC to restrict Secret access.

```
┌──────────────────────────────────────────────────────────────┐
│                        Your Pod                              │
│                                                              │
│   env:                                                       │
│     MYSQL_DATABASE  ◄──── ConfigMap (mysql-config-map)       │
│     MYSQL_ROOT_PASSWORD ◄─ Secret   (mysql-secret)          │
└──────────────────────────────────────────────────────────────┘
```

---

## How ConfigMaps & Secrets Work

Both objects are stored in etcd. When a Pod references them, the kubelet injects the values at container startup — either as environment variables or as mounted files.

```
kubectl apply -f configMap.yml / secrets.yml
        │
        ▼
┌───────────────┐
│  API Server   │  ← Stores ConfigMap/Secret in etcd
└───────┬───────┘
        │
        ▼
┌────────────────────┐
│  Scheduler         │  ← Assigns Pod to a node
└───────┬────────────┘
        │
        ▼
┌────────────────────┐
│  Kubelet (node)    │  ← Reads ConfigMap/Secret from API Server
│                    │    Injects values into the container
└────────────────────┘
```

**Two ways to consume them in a Pod:**

1. **Environment Variables** — values injected as `env` vars at container start (most common)
2. **Volume Mounts** — values projected as files inside the container (useful for config files, TLS certs)

---

## Manifest Breakdown — Field by Field

### ConfigMap — `configMap.yml`

```yaml
kind: ConfigMap       # Resource type
apiVersion: v1        # ConfigMaps are in the core v1 API group
metadata:
  name: mysql-config-map   # Name referenced by Pods
  namespace: mysql         # Must be in the same namespace as the consuming Pod
data:
  MYSQL_DATABASE: devops   # Key-value pairs — plain text, no encoding needed
```

### Secret — `secrets.yml`

```yaml
apiVersion: v1
kind: Secret          # Resource type
metadata:
  name: mysql-secret       # Name referenced by Pods
  namespace: mysql         # Must be in the same namespace as the consuming Pod
data:
  MYSQL_ROOT_PASSWORD: cm9vdAo=   # Value MUST be base64-encoded
  # echo -n 'root' | base64  →  cm9vdA==
```

### Every Field Explained

| Field | ConfigMap | Secret | Purpose |
|---|---|---|---|
| `kind` | `ConfigMap` | `Secret` | Resource type |
| `apiVersion` | `v1` | `v1` | Both are in the core API group |
| `metadata.name` | ✅ | ✅ | Referenced by Pods via `configMapRef` / `secretKeyRef` |
| `metadata.namespace` | ✅ | ✅ | Must match the consuming Pod's namespace |
| `data` | Plain text values | Base64-encoded values | Key-value pairs |
| `stringData` | N/A | Plain text (auto-encoded) | Convenience field — Kubernetes encodes it for you |

> 💡 **Tip:** Use `stringData` in Secrets during development so you don't have to manually base64-encode values. Kubernetes converts it to `data` (base64) automatically on apply.

---

## Consuming ConfigMaps & Secrets in Pods

### Method 1 — Environment Variables (most common)

```yaml
spec:
  containers:
  - name: mysql
    image: mysql:8.0
    env:
    - name: MYSQL_DATABASE           # Env var name inside the container
      valueFrom:
        configMapKeyRef:
          name: mysql-config-map     # ConfigMap name
          key: MYSQL_DATABASE        # Key inside the ConfigMap

    - name: MYSQL_ROOT_PASSWORD      # Env var name inside the container
      valueFrom:
        secretKeyRef:
          name: mysql-secret         # Secret name
          key: MYSQL_ROOT_PASSWORD   # Key inside the Secret
```

### Method 2 — Load All Keys at Once (`envFrom`)

```yaml
spec:
  containers:
  - name: mysql
    image: mysql:8.0
    envFrom:
    - configMapRef:
        name: mysql-config-map    # Injects ALL keys as env vars
    - secretRef:
        name: mysql-secret        # Injects ALL keys as env vars
```

### Method 3 — Volume Mount (for config files / TLS certs)

```yaml
spec:
  volumes:
  - name: config-vol
    configMap:
      name: mysql-config-map      # Each key becomes a file in the mount path

  containers:
  - name: mysql
    image: mysql:8.0
    volumeMounts:
    - name: config-vol
      mountPath: /etc/mysql/conf.d  # Keys appear as files here
```

---

## Hands-On Lab

### Prerequisites
- A running Kubernetes cluster (Minikube, Kind, EKS, GKE, or AKS)
- `kubectl` configured (`kubectl cluster-info` to verify)

---

### Step 1 — Create the Namespace

```bash
kubectl create namespace mysql
```

---

### Step 2 — Apply the ConfigMap and Secret

```bash
kubectl apply -f configMap.yml
kubectl apply -f secrets.yml
```

---

### Step 3 — Verify They Were Created

```bash
# List ConfigMaps
kubectl get configmap -n mysql

# List Secrets
kubectl get secret -n mysql
```

Expected output:
```
NAME               DATA   AGE
mysql-config-map   1      5s

NAME           TYPE     DATA   AGE
mysql-secret   Opaque   1      5s
```

---

### Step 4 — Inspect the ConfigMap and Secret

```bash
# View ConfigMap data (values are visible)
kubectl describe configmap mysql-config-map -n mysql

# View Secret metadata (values are hidden)
kubectl describe secret mysql-secret -n mysql

# Decode a Secret value
kubectl get secret mysql-secret -n mysql -o jsonpath='{.data.MYSQL_ROOT_PASSWORD}' | base64 --decode
```

> 💡 `kubectl describe secret` hides values intentionally. Use `jsonpath` + `base64 --decode` to retrieve the actual value when needed.

---

### Step 5 — Create a Pod That Consumes Both

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: mysql-test
  namespace: mysql
spec:
  containers:
  - name: mysql
    image: mysql:8.0
    env:
    - name: MYSQL_DATABASE
      valueFrom:
        configMapKeyRef:
          name: mysql-config-map
          key: MYSQL_DATABASE
    - name: MYSQL_ROOT_PASSWORD
      valueFrom:
        secretKeyRef:
          name: mysql-secret
          key: MYSQL_ROOT_PASSWORD
  restartPolicy: Never
EOF
```

---

### Step 6 — Verify the Values Were Injected

```bash
# Exec into the pod and check env vars
kubectl exec -it mysql-test -n mysql -- env | grep MYSQL
```

Expected output:
```
MYSQL_DATABASE=devops
MYSQL_ROOT_PASSWORD=root
```

---

### Step 7 — Test What Happens When a Referenced Key Is Missing

```bash
# Try referencing a non-existent key
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: mysql-bad
  namespace: mysql
spec:
  containers:
  - name: mysql
    image: mysql:8.0
    env:
    - name: MISSING_VAR
      valueFrom:
        configMapKeyRef:
          name: mysql-config-map
          key: NON_EXISTENT_KEY   # This key doesn't exist
  restartPolicy: Never
EOF

kubectl get pod mysql-bad -n mysql
```

> 💡 The pod will stay in `Pending` or fail with `CreateContainerConfigError`. Kubernetes validates ConfigMap/Secret references at pod startup — a missing key blocks the container from starting.

---

### Step 8 — Clean Up

```bash
kubectl delete namespace mysql
```

---

## ConfigMaps & Secrets in Production — What Changes

The manifests in this lab are intentionally minimal. In production:

```yaml
# Production-grade Secret — use stringData for readability, let K8s encode it
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
  namespace: mysql
  annotations:
    # Document the rotation policy
    secret-rotation: "90-days"
type: Opaque
stringData:                          # Plain text — Kubernetes base64-encodes automatically
  MYSQL_ROOT_PASSWORD: "strongpassword123!"
  MYSQL_USER: "appuser"
  MYSQL_PASSWORD: "apppassword456!"
```

```yaml
# Production-grade ConfigMap — multiple keys, structured config
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-config-map
  namespace: mysql
data:
  MYSQL_DATABASE: devops
  MYSQL_MAX_CONNECTIONS: "200"
  my.cnf: |                          # Multi-line config file as a key
    [mysqld]
    max_connections=200
    character-set-server=utf8mb4
```

**Production checklist:**
- Enable etcd encryption at rest for Secrets
- Use RBAC to restrict who can `get`/`list` Secrets
- Consider external secret managers (AWS Secrets Manager, HashiCorp Vault) with the Secrets Store CSI Driver
- Never commit Secret manifests with real values to Git — use sealed-secrets or external-secrets operator
- Pin image versions (`mysql:8.0.33`, not `mysql:latest`)

---

## Interview Q&A — Straight to the Point

**Q: What is a ConfigMap?**
A ConfigMap is a Kubernetes object that stores non-sensitive configuration data as key-value pairs. It decouples environment-specific configuration from container images, allowing the same image to run in dev, staging, and production with different configs.

---

**Q: What is a Secret?**
A Secret is a Kubernetes object for storing sensitive data like passwords, tokens, and keys. Values are base64-encoded in etcd. Unlike ConfigMaps, `kubectl describe` hides Secret values, and RBAC can restrict access more granularly.

---

**Q: Is a Kubernetes Secret secure?**
Not by default. Base64 is encoding, not encryption — anyone with cluster access can decode it. True security requires: etcd encryption at rest, RBAC to restrict Secret access, and ideally an external secret manager (Vault, AWS Secrets Manager).

---

**Q: What are the two main ways to consume a ConfigMap or Secret in a Pod?**
1. Environment variables — via `env.valueFrom.configMapKeyRef` / `secretKeyRef`, or bulk-load with `envFrom`
2. Volume mounts — each key becomes a file at the specified mount path, useful for config files and TLS certs

---

**Q: What happens if a Pod references a ConfigMap or Secret that doesn't exist?**
The Pod fails to start with `CreateContainerConfigError`. Kubernetes validates all references at pod startup. The ConfigMap/Secret must exist in the same namespace before the Pod is created.

---

**Q: What is the difference between `data` and `stringData` in a Secret?**
`data` requires base64-encoded values. `stringData` accepts plain text — Kubernetes encodes it automatically on apply. `stringData` is write-only (it doesn't appear in `kubectl get secret -o yaml`); the encoded value appears under `data`.

---

**Q: Can a Pod in namespace A use a ConfigMap from namespace B?**
No. ConfigMaps and Secrets are namespace-scoped. A Pod can only reference ConfigMaps and Secrets in its own namespace.

---

**Q: What happens to a running Pod if you update a ConfigMap?**
- **Environment variables:** The pod is NOT updated. Env vars are injected at container start and are immutable for the lifetime of the container. You must restart the pod.
- **Volume mounts:** The mounted files ARE updated automatically (with a short delay, ~1 minute). This is one advantage of volume-mounted configs over env vars.

---

**Q: What is `type: Opaque` in a Secret?**
`Opaque` is the default Secret type — it means arbitrary user-defined data. Kubernetes also has built-in types like `kubernetes.io/tls` (for TLS certs), `kubernetes.io/dockerconfigjson` (for image pull secrets), and `kubernetes.io/service-account-token`.

---

## Common Mistakes & Gotchas

| Mistake | Why It's Wrong | Fix |
|---|---|---|
| Forgetting to base64-encode Secret values in `data` | Kubernetes stores the raw string, container gets garbled value | Use `echo -n 'value' \| base64` or use `stringData` |
| ConfigMap and Pod in different namespaces | Pod fails with `CreateContainerConfigError` | Both must be in the same namespace |
| Committing Secret manifests with real values to Git | Credentials exposed in version history | Use sealed-secrets, external-secrets, or environment-specific injection in CI/CD |
| Expecting env var updates after ConfigMap change | Env vars are set at container start — they don't hot-reload | Restart the pod, or use volume mounts for hot-reload |
| Using `image: mysql:latest` | Image can silently change — breaks reproducibility | Pin to a specific version: `mysql:8.0.33` |
| No RBAC on Secrets | Any pod/user in the cluster can read all Secrets | Apply least-privilege RBAC — restrict `get`/`list` on Secrets |
| Storing large data in ConfigMaps | ConfigMaps have a 1MB size limit | Use a PersistentVolume or object storage for large config files |
| Forgetting `-n <namespace>` in kubectl commands | Commands silently operate on `default` namespace | Always pass `-n <namespace>` |

---

## Files in This Lab

```
02-configmaps/
├── configMap.yml  ← MySQL ConfigMap (MYSQL_DATABASE)
├── secrets.yml    ← MySQL Secret (MYSQL_ROOT_PASSWORD, base64-encoded)
└── README.md      ← This file
```

---

## ✍️ Author

**[Himanshu Kumar](https://www.linkedin.com/in/h1manshu-kumar/)** - Learning by building, documenting, and sharing 🚀

---
<div align="center">

**Built while learning Kubernetes hands-on.**
*The best way to understand ConfigMaps and Secrets is to break a pod by deleting one, then fix it - and understand why it failed.*

</div>
