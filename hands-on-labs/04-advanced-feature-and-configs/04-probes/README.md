# 🩺 Kubernetes Probes — Deep Dive

> **Lab Series:** Advanced Features & Configs → 04 Probes    
> **Difficulty:** Intermediate    
> **Estimated Time:** 30–45 minutes    
> **Focus:** Understand the three probe types, when each fires, what each controls, and how to configure them correctly    

---

## 📌 Table of Contents

1. [Why Probes Exist](#why-probes-exist)
2. [The Three Probes — What Each One Does](#the-three-probes--what-each-one-does)
3. [Probe Execution Order — The Lifecycle](#probe-execution-order--the-lifecycle)
4. [Probe Mechanisms — How Kubernetes Checks Health](#probe-mechanisms--how-kubernetes-checks-health)
5. [Probe Configuration Fields — Every Field Explained](#probe-configuration-fields--every-field-explained)
6. [Manifest Breakdown — Field by Field](#manifest-breakdown--field-by-field)
7. [Hands-On Lab](#hands-on-lab)
8. [Probes in Production — What Changes](#probes-in-production--what-changes)
9. [Interview Q&A — Straight to the Point](#interview-qa--straight-to-the-point)
10. [Common Mistakes & Gotchas](#common-mistakes--gotchas)

---

## Why Probes Exist

Kubernetes knows when a container *process* crashes — the exit code is non-zero and the kubelet restarts it. But Kubernetes has no way to know if a container is *running but broken* — for example:

- The process is up but deadlocked
- The app started but isn't ready to serve traffic yet
- A slow-starting app (JVM, ML model loading) that needs 2 minutes before it's healthy

Probes solve this. They give Kubernetes a way to ask the container: **"Are you alive? Are you ready? Did you finish starting?"**

Without probes, Kubernetes sends traffic to pods the moment the container process starts — even if the app isn't ready. It also never restarts a container that's running but deadlocked.

---

## The Three Probes — What Each One Does

| Probe | Question It Answers | On Failure | On Success |
|---|---|---|---|
| `startupProbe` | "Has the app finished starting?" | Kills & restarts container | Hands off to liveness/readiness |
| `livenessProbe` | "Is the app still alive and healthy?" | Kills & restarts container | Nothing — keeps running |
| `readinessProbe` | "Is the app ready to receive traffic?" | Removes pod from Service endpoints | Adds pod back to Service endpoints |

> 💡 **The Key Distinction:** Liveness controls *restarts*. Readiness controls *traffic*. Startup gates *both* until the app is up.

```
┌─────────────────────────────────────────────────────────────────┐
│                        Pod Lifecycle                            │
│                                                                 │
│  Container Starts                                               │
│       │                                                         │
│       ▼                                                         │
│  ┌─────────────┐                                                │
│  │ startupProbe│ ← Runs until success (or failureThreshold hit) │
│  │  (if set)   │   liveness + readiness are PAUSED during this  │
│  └──────┬──────┘                                                │
│         │ success                                               │
│         ▼                                                       │
│  ┌──────────────────────────────────────┐                       │
│  │  livenessProbe  ←→  readinessProbe   │ ← Both run in parallel│
│  │  (restart on fail)  (traffic on/off) │                       │
│  └──────────────────────────────────────┘                       │
└─────────────────────────────────────────────────────────────────┘
```

---

## Probe Execution Order — The Lifecycle

```
Container starts
      │
      ├─ startupProbe defined?
      │       YES → Run startupProbe every periodSeconds
      │               │
      │               ├─ Fails failureThreshold times → Container KILLED & restarted
      │               │
      │               └─ Succeeds → startupProbe stops, liveness + readiness begin
      │
      │       NO  → liveness + readiness begin immediately (after initialDelaySeconds)
      │
      ├─ livenessProbe (runs forever, every periodSeconds)
      │       Fails failureThreshold times → Container KILLED & restarted
      │
      └─ readinessProbe (runs forever, every periodSeconds)
              Fails → Pod removed from Service endpoints (NOT restarted)
              Succeeds → Pod added back to Service endpoints
```

---

## Probe Mechanisms — How Kubernetes Checks Health

There are three ways to define *how* a probe checks health:

### 1. `httpGet` — HTTP Request
Kubelet sends an HTTP GET to the container. Any response `200–399` = success.

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
    httpHeaders:                 # Optional: custom headers
      - name: X-Health-Check
        value: "true"
```

> Best for: web servers, REST APIs, any HTTP-speaking app.

---

### 2. `exec` — Run a Command Inside the Container
Kubelet runs a command inside the container. Exit code `0` = success, anything else = failure.

```yaml
livenessProbe:
  exec:
    command:
      - cat
      - /tmp/healthy
```

> Best for: apps without an HTTP server, custom health scripts, checking file existence.

---

### 3. `tcpSocket` — TCP Connection Check
Kubelet tries to open a TCP connection to the specified port. If the connection succeeds = healthy.

```yaml
livenessProbe:
  tcpSocket:
    port: 3306
```

> Best for: databases, message brokers, any TCP service that doesn't speak HTTP.

---

### 4. `grpc` — gRPC Health Check (Kubernetes 1.24+)
Kubelet calls the [gRPC health checking protocol](https://grpc.github.io/grpc/core/md_doc_health-checking.html).

```yaml
livenessProbe:
  grpc:
    port: 50051
    service: ""                  # Optional: specific gRPC service name
```

> Best for: gRPC services.

---

## Probe Configuration Fields — Every Field Explained

These fields apply to all three probe types:

| Field | Default | Purpose |
|---|---|---|
| `initialDelaySeconds` | `0` | Seconds to wait after container starts before running the first probe |
| `periodSeconds` | `10` | How often to run the probe |
| `timeoutSeconds` | `1` | Seconds to wait for a probe response before counting it as a failure |
| `failureThreshold` | `3` | Consecutive failures before taking action (restart or remove from endpoints) |
| `successThreshold` | `1` | Consecutive successes to consider the probe passing (only meaningful for readiness) |

> ⚠️ `successThreshold` must be `1` for liveness and startup probes. Only readiness can have a value > 1.

---

## Manifest Breakdown — Field by Field

### liveness-probe.yml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-probe-demo
  namespace: probes-demo
spec:
  containers:
    - name: nginx
      image: nginx:1.25.3
      ports:
        - containerPort: 80
      livenessProbe:
        httpGet:
          path: /           # Endpoint to hit — nginx serves 200 here
          port: 80
        initialDelaySeconds: 5   # Give nginx 5s to start before first check
        periodSeconds: 10        # Check every 10s
        failureThreshold: 3      # Restart after 3 consecutive failures (30s window)
```

**What happens on failure:** Kubelet kills the container and restarts it. The pod's `restartCount` increments. If it keeps failing, the pod enters `CrashLoopBackOff`.

---

### readiness-probe.yml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: readiness-probe-demo
  namespace: probes-demo
spec:
  containers:
    - name: nginx
      image: nginx:1.25.3
      ports:
        - containerPort: 80
      readinessProbe:
        httpGet:
          path: /
          port: 80
        initialDelaySeconds: 3   # Shorter than liveness — readiness checks sooner
        periodSeconds: 5
        failureThreshold: 3      # Remove from endpoints after 3 failures
        successThreshold: 1      # Re-add to endpoints after 1 success
```

**What happens on failure:** Pod is removed from the Service's endpoint list. No traffic is routed to it. The container is NOT restarted. When the probe passes again, the pod is re-added to endpoints.

---

### startup-probe.yml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: startup-probe-demo
  namespace: probes-demo
spec:
  containers:
    - name: nginx
      image: nginx:1.25.3
      ports:
        - containerPort: 80
      startupProbe:
        httpGet:
          path: /
          port: 80
        failureThreshold: 30     # 30 * 10s = 5 minutes max startup window
        periodSeconds: 10
      livenessProbe:             # Only activates AFTER startupProbe succeeds
        httpGet:
          path: /
          port: 80
        periodSeconds: 10
        failureThreshold: 3
```

**Why this matters:** Without a startupProbe, a slow-starting app (e.g., a JVM app that takes 90s to boot) would be killed by the livenessProbe before it ever finishes starting. The startupProbe gives it a generous window to start, then hands off to the tighter livenessProbe for ongoing health checks.

---

## Hands-On Lab

### Prerequisites
- A running Kubernetes cluster (Minikube, Kind, EKS, GKE, or AKS)
- `kubectl` configured (`kubectl cluster-info` to verify)

---

### Step 1 — Create the Namespace

```bash
kubectl create namespace probes-demo
```
<img width="618" height="39" alt="image" src="https://github.com/user-attachments/assets/8da2aa10-d31e-45f9-ac7b-21534035b7b2" />

---

### Step 2 — Deploy the Liveness Probe Pod

```bash
kubectl apply -f liveness-probe.yml
```
<img width="589" height="38" alt="image" src="https://github.com/user-attachments/assets/620d16b2-f44e-45a8-8eb6-0a3e1aa622a9" />

Watch the pod come up:

```bash
kubectl get pod liveness-probe-demo -n probes-demo -w
```
<img width="710" height="57" alt="image" src="https://github.com/user-attachments/assets/ada7e8bd-1d8f-430d-966d-0cb85b240ae3" />

Expected: pod reaches `Running` state with `READY 1/1`.

---

### Step 3 — Inspect the Liveness Probe

```bash
kubectl describe pod liveness-probe-demo -n probes-demo
```

Look for the `Liveness` line in the container section:

```
Liveness:  http-get http://:80/ delay=5s timeout=1s period=10s #success=1 #failure=3
```
<img width="1168" height="535" alt="image" src="https://github.com/user-attachments/assets/3d690723-4ca8-4123-8a42-13bae963270c" />

<img width="1280" height="154" alt="image" src="https://github.com/user-attachments/assets/f0c3ad27-a505-472f-a844-5bf18a2a8b14" />

Also check the `Events` section at the bottom — this is where probe failures and restarts are logged.

---

### Step 4 — Simulate a Liveness Failure

Break nginx by removing its index file, causing it to return 404 (which is still a 2xx/3xx... actually let's use exec to kill the process):

```bash
# Kill the nginx master process inside the container
kubectl exec liveness-probe-demo -n probes-demo -- kill 1
```
Watch the pod restart:

```bash
kubectl get pod liveness-probe-demo -n probes-demo -w
```

Expected output:
```
NAME                   READY   STATUS    RESTARTS   AGE
liveness-probe-demo    1/1     Running   0          2m
liveness-probe-demo    0/1     Error     0          2m
liveness-probe-demo    1/1     Running   1          2m   ← restarted
```

Check the restart count:

```bash
kubectl get pod liveness-probe-demo -n probes-demo
# RESTARTS column will show 1
```

---

### Step 5 — Deploy the Readiness Probe Pod

```bash
kubectl apply -f readiness-probe.yml
```

```bash
kubectl describe pod readiness-probe-demo -n probes-demo
```

Look for:
```
Readiness:  http-get http://:80/ delay=3s timeout=1s period=5s #success=1 #failure=3
```
<img width="1155" height="535" alt="image" src="https://github.com/user-attachments/assets/97b25b47-953a-4347-aed2-7c6d3f30ea88" />

---

### Step 6 — Simulate a Readiness Failure

```bash
# Remove nginx's default page — nginx will return 404, readiness probe fails
kubectl exec readiness-probe-demo -n probes-demo -- rm /usr/share/nginx/html/index.html
```

Watch the pod become not-ready:

```bash
kubectl get pod readiness-probe-demo -n probes-demo -w
```

Expected:
```
NAME                    READY   STATUS    RESTARTS
readiness-probe-demo    1/1     Running   0
readiness-probe-demo    0/1     Running   0    ← not ready, removed from endpoints
```

> 💡 Notice: `RESTARTS` stays at `0`. The container is NOT restarted — it's just removed from traffic. This is the key difference between liveness and readiness.

Restore it:

```bash
kubectl exec readiness-probe-demo -n probes-demo -- \
  sh -c 'echo "healthy" > /usr/share/nginx/html/index.html'
```

Watch it become ready again within a few seconds.

---

### Step 7 — Deploy the Startup Probe Pod

```bash
kubectl apply -f startup-probe.yml
```

```bash
kubectl describe pod startup-probe-demo -n probes-demo
```

Look for both `Startup` and `Liveness` entries in the container description. Notice that liveness only activates after startup succeeds.

<img width="1160" height="556" alt="image" src="https://github.com/user-attachments/assets/093f2d0d-9c07-4076-a349-3088556062f8" />

---

### Step 8 — Deploy All Three Probes Together

```bash
kubectl apply -f all-probes.yml
```

```bash
kubectl describe pod all-probes-demo -n probes-demo
```
<img width="929" height="617" alt="image" src="https://github.com/user-attachments/assets/ec946efd-4d1d-4f03-ab12-5468e99d1118" />

You'll see all three probes listed. This is the production-ready pattern.

---

### Step 9 — Check Probe Events

```bash
# See probe failures and restarts in events
kubectl get events -n probes-demo --sort-by='.lastTimestamp'
```
<img width="1280" height="499" alt="image" src="https://github.com/user-attachments/assets/136813c4-5a4c-4f11-bb70-8359f3099ef8" />

Probe failures appear as events like:
```
Liveness probe failed: HTTP probe failed with statuscode: 404
Readiness probe failed: HTTP probe failed with statuscode: 404
```

---

### Step 10 — Clean Up

```bash
kubectl delete namespace probes-demo
```
<img width="579" height="44" alt="image" src="https://github.com/user-attachments/assets/0e9db6d3-6349-4748-8a72-886c1ca8e550" />

---

## Probes in Production — What Changes

The manifests in this lab use nginx for simplicity. In production:

1. **Use a dedicated health endpoint** — don't probe `/`. Create `/healthz` (liveness) and `/readyz` (readiness) endpoints in your app that check internal state (DB connection, cache, etc.).

2. **Tune `initialDelaySeconds` to your app's actual startup time** — too short and the probe fires before the app is ready; too long and you delay detecting real failures.

3. **Use `startupProbe` for any app that takes > 30s to start** — JVM apps, ML model loading, apps that run DB migrations on startup.

4. **Readiness ≠ Liveness thresholds** — readiness can be more sensitive (shorter `periodSeconds`, lower `failureThreshold`) because it only affects traffic, not restarts.

```yaml
# Production-grade probe pattern
startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 30       # 5 minute startup window
  periodSeconds: 10

livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 0     # startupProbe already handled the delay
  periodSeconds: 15
  failureThreshold: 3
  timeoutSeconds: 5

readinessProbe:
  httpGet:
    path: /readyz             # Different endpoint — checks dependencies too
    port: 8080
  initialDelaySeconds: 0
  periodSeconds: 5
  failureThreshold: 3
  successThreshold: 1
  timeoutSeconds: 3
```

> 💡 `/healthz` and `/readyz` should be different endpoints. `/healthz` checks if the process is alive. `/readyz` checks if it can serve traffic (DB connected, cache warm, etc.).

---

## Interview Q&A — Straight to the Point

**Q: What are the three types of probes in Kubernetes?**
Liveness, Readiness, and Startup. Liveness restarts a container if it's unhealthy. Readiness removes a pod from Service endpoints if it can't serve traffic. Startup gates liveness and readiness until a slow-starting container finishes initializing.

---

**Q: What is the difference between a liveness probe and a readiness probe?**
Liveness failure → container is killed and restarted. Readiness failure → pod is removed from Service endpoints but NOT restarted. Use liveness for "is the app broken?", use readiness for "is the app ready to handle requests right now?".

---

**Q: Why would you use a startup probe instead of just setting a high `initialDelaySeconds` on the liveness probe?**
`initialDelaySeconds` is a fixed blind wait — you're delaying liveness checks for every restart, even when the app starts fast. A startup probe actively polls and hands off to liveness the moment the app is ready, giving you a fast path for normal restarts and a generous window for slow first starts.

---

**Q: What happens if a startup probe keeps failing?**
The container is killed and restarted, same as a liveness failure. The pod's `restartCount` increments. If it keeps failing, the pod enters `CrashLoopBackOff`.

---

**Q: What happens to traffic when a readiness probe fails?**
The pod is removed from the `Endpoints` object of any Service that selects it. No new traffic is routed to it. Existing connections are not dropped. When the probe passes again, the pod is re-added to endpoints.

---

**Q: What are the four probe mechanisms?**
`httpGet` (HTTP GET request), `exec` (run a command, check exit code), `tcpSocket` (TCP connection check), and `grpc` (gRPC health protocol, Kubernetes 1.24+).

---

**Q: What does `failureThreshold` mean?**
The number of consecutive probe failures before Kubernetes takes action. For liveness/startup: restart the container. For readiness: remove from endpoints. Default is 3.

---

**Q: Can `successThreshold` be greater than 1 for a liveness probe?**
No. `successThreshold` must be 1 for liveness and startup probes. Only readiness probes can have `successThreshold > 1`, which means the pod must pass N consecutive checks before being re-added to endpoints.

---

**Q: A pod shows `READY 0/1` but `STATUS Running`. What does that mean?**
The readiness probe is failing. The container is running but Kubernetes has determined it's not ready to receive traffic. It has been removed from Service endpoints. The container has not been restarted.

---

**Q: A pod keeps restarting with `CrashLoopBackOff`. How do probes relate?**
Either the container process is crashing (exit code non-zero) or the liveness/startup probe is failing repeatedly. Check `kubectl describe pod` for probe failure events and `kubectl logs --previous` for the last container's output.

---

## Common Mistakes & Gotchas

| Mistake | Why It's Wrong | Fix |
|---|---|---|
| No probes at all | Kubernetes sends traffic to pods before they're ready; broken-but-running pods never restart | Add at minimum a readiness probe to every production pod |
| Liveness probe checks external dependencies (DB, API) | If the DB goes down, all pods restart in a cascade — the app is fine, the DB isn't | Liveness should only check the app process itself; readiness can check dependencies |
| `initialDelaySeconds` too short | Probe fires before app is ready → immediate failure → restart loop | Set it to at least your app's p95 startup time, or use a startupProbe |
| `initialDelaySeconds` too long | Delays detecting real failures after restarts | Use startupProbe for slow starts; keep liveness `initialDelaySeconds` low |
| Probing `/` instead of a dedicated health endpoint | `/` may return 200 even when the app is broken internally | Implement `/healthz` and `/readyz` in your app |
| Same endpoint for liveness and readiness | Readiness should check dependencies (DB, cache); liveness should not | Use separate endpoints with different logic |
| `timeoutSeconds` too low on a slow endpoint | Probe times out and counts as failure even when app is healthy | Set `timeoutSeconds` to match your endpoint's p99 response time |
| Using `exec` probe with a shell script that always exits 0 | Probe never fails even when app is broken | Test your probe command manually; ensure it returns non-zero on failure |
| No `startupProbe` for slow-starting apps | Liveness probe kills the app before it finishes starting → infinite restart loop | Add a startupProbe with `failureThreshold * periodSeconds` ≥ max startup time |

---

## Files in This Lab

```
04-probes/
├── 01-liveness-probe.yml   ← Pod with only a livenessProbe (httpGet)
├── 02-readiness-probe.yml  ← Pod with only a readinessProbe (httpGet)
├── 03-startup-probe.yml    ← Pod with startupProbe + livenessProbe
├── 04-all-probes.yml       ← Pod with all three probes (production pattern)
└── README.md            ← This file
```

---

## ✍️ Author

**[Himanshu Kumar](https://www.linkedin.com/in/h1manshu-kumar/)** - Learning by building, documenting, and sharing 🚀

---
<div align="center">

**Built while learning Kubernetes hands-on.**
*The best way to understand probes is to break things — kill processes, delete files, and watch Kubernetes react.*

</div>
