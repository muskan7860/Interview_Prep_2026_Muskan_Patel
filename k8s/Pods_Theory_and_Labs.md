# Pods — Theoretical Explanation, Examples, and Hands-On Labs

> Level: 4 Years Experience
> Author: Muskan Patel

---

## 1. What is a Pod

A Pod is the smallest deployable unit in Kubernetes. It is a wrapper
around one or more containers that:

- Share the same network namespace (one IP address — containers
  inside talk to each other via localhost)
- Can share storage volumes
- Are always scheduled together on the SAME node
- Live and die together as one unit

You never deploy a "container" directly in Kubernetes — you always
deploy a Pod, even if that Pod has only one container.

---

## 2. Why Pods Exist — The Pause Container

When a Pod is created, Kubernetes FIRST creates a hidden container
called the **pause container**. It does nothing functionally — its
only job is to hold the network namespace open.

```
Pod "payment-pod"
│
├── pause container       (holds network namespace, IP: 10.244.1.5)
├── payment-app container (joins same namespace, uses IP 10.244.1.5)
└── log-sidecar container (joins same namespace, ALSO uses 10.244.1.5)
```

Both `payment-app` and `log-sidecar` can reach each other via
`localhost` because they share the pause container's network
namespace. This is THE mechanism behind multi-container pods.

```bash
# See this for yourself on the node
kubectl run test-pod --image=nginx
sudo crictl ps | grep test-pod
# You will see TWO containers for ONE pod:
# nginx              Running
# test-pod-pause     Running   ← the hidden pause container
```

---

## 3. Pod Lifecycle — Phases Explained

| Phase | Meaning | When You See It |
|---|---|---|
| Pending | Accepted by API server, not all containers running yet | Scheduling in progress, or image still pulling |
| Running | At least one container is running | Normal healthy state |
| Succeeded | All containers exited with code 0 | Jobs, one-time tasks |
| Failed | At least one container exited non-zero, no restart | Job failed, or restartPolicy: Never crash |
| Unknown | API server lost contact with node | Node down or network partition |

```bash
# Watch a pod move through phases live
kubectl run mypod --image=nginx
kubectl get pod mypod -w
# Ctrl+C to stop
```

---

## 4. Full Pod YAML — Every Field Explained

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: payment-pod
  namespace: banking
  labels:
    app: payment
    env: production
spec:
  containers:
  - name: payment-app
    image: payment:v1.2

    ports:
    - containerPort: 8080
    # Documentation only — does NOT open a port
    # The app inside must bind to 8080 regardless

    resources:
      requests:
        cpu: "250m"
        memory: "128Mi"
      limits:
        cpu: "500m"
        memory: "256Mi"

    livenessProbe:
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 15
      periodSeconds: 10

    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5

    env:
    - name: DB_HOST
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: host

  restartPolicy: Always
```

---

## 5. Resources (requests and limits) — Plain English

### The Hotel Room Analogy

A hotel manager (Scheduler) places groups (Pods) into rooms (Nodes).

Each group says:
- "We NEED at least 2 rooms" → this is **requests**
- "We will NEVER use more than 4 rooms" → this is **limits**

The manager uses the NEED number to decide WHERE to place the group.
The limit number is only checked AFTER they're inside.

### requests — used ONLY by the Scheduler

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "128Mi"
```

For EVERY node, the scheduler calculates:
```
free_for_scheduling = node.Allocatable - sum(requests of pods already on this node)
```

If `free_for_scheduling >= this pod's requests` → node is a candidate.

IMPORTANT: this is based on RESERVATION, not real usage. 10 pods
could each REQUEST 100m CPU (1000m reserved total) while ACTUALLY
using only 50m total. The scheduler still sees "1000m reserved."

```bash
# Reservation-based view
kubectl describe node <node-name> | grep -A 5 "Allocated resources"

# Real usage view
kubectl top nodes
# Often shows MUCH lower numbers than "Allocated resources"
```

### limits — enforced by the Linux kernel, continuously

```yaml
resources:
  limits:
    cpu: "500m"
    memory: "256Mi"
```

**CPU limit exceeded** → THROTTLED (process slows down, does NOT
crash). CPU is "compressible" — Linux just gives the process less
CPU time per scheduling period.

**Memory limit exceeded** → KILLED (SIGKILL, immediate). Memory is
"incompressible" — either the allocation succeeds or it doesn't.
Kubernetes reports this as `Reason: OOMKilled, Exit Code: 137`.

### QoS Classes — direct result of requests/limits

| Setup | QoS Class | Evicted |
|---|---|---|
| requests == limits for everything | Guaranteed | LAST |
| requests set, limits different (or partial) | Burstable | SECOND |
| nothing set | BestEffort | FIRST |

```bash
kubectl get pod <name> -o jsonpath='{.status.qosClass}'
```

---

## 6. Probes — Liveness, Readiness, Startup

### Liveness Probe — "Are you ALIVE?"

If it fails → kubelet RESTARTS the container.
Used to detect deadlocks (process running but stuck).

### Readiness Probe — "Are you READY for traffic?"

If it fails → pod removed from Service endpoints (no traffic sent).
Container is NOT restarted.
Used for: app warm-up time, waiting on DB connections.

A pod can be `Running` (liveness passes) but `READY: 0/1`
(readiness fails) — common during startup.

### Startup Probe — "Have you FINISHED starting?"

Disables liveness AND readiness checks until it succeeds.
For SLOW-starting apps — gives generous time to start without
weakening the ongoing liveness/readiness checks afterward.

### Probe Mechanisms

- `httpGet` — HTTP GET, success = response 200-399
- `tcpSocket` — TCP connection succeeds
- `exec` — runs a command inside container, success = exit code 0
- `grpc` — gRPC health check

---

## 7. Environment Variables, ConfigMaps, Secrets

### Two ways to inject config:

**1. Environment variables** — injected ONCE at container start.
If the Secret/ConfigMap changes LATER, running container does NOT
see the new value until restarted.

**2. Volume mounts** — appear as FILES inside the container.
kubelet updates the files when source changes (with delay), WITHOUT
restarting the pod — but the APP must detect file changes itself.

### Secrets are NOT encrypted by default

Stored as base64-ENCODED (not encrypted) in etcd. Trivially
reversible:
```bash
echo "cG9zdGdyZXM=" | base64 -d
```
Real protection = RBAC (who can `kubectl get secrets`) + etcd
encryption-at-rest (separate cluster config).

---

## HANDS-ON LABS

---

## Lab 1 — Create and Inspect a Basic Pod

```bash
kubectl apply -f - << EOF
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    ports:
    - containerPort: 80
    resources:
      requests:
        cpu: "100m"
        memory: "64Mi"
      limits:
        cpu: "200m"
        memory: "128Mi"
EOF

# Watch it come up
kubectl get pod nginx-pod -w
# Ctrl+C once Running

# Full details
kubectl describe pod nginx-pod

# Get pod IP
kubectl get pod nginx-pod -o wide

# Exec inside
kubectl exec -it nginx-pod -- bash
curl localhost:80
exit

# Check resource usage
kubectl top pod nginx-pod

# Delete — NOT recreated (standalone pod)
kubectl delete pod nginx-pod
kubectl get pods
# Gone forever
```

---

## Lab 2 — containerPort is Documentation Only

```bash
# Create pod WITHOUT declaring containerPort
kubectl run nginx-no-port --image=nginx

# Get its IP
kubectl get pod nginx-no-port -o wide
# Note the IP, e.g. 10.244.1.10

# From ANOTHER pod, curl it on port 80
kubectl run debug --image=busybox -it --rm -- sh
wget -O- 10.244.1.10:80
exit
# WORKS — even though containerPort was never declared

kubectl delete pod nginx-no-port
```

---

## Lab 3 — CPU Throttling (Limit Exceeded, Pod Survives)

```bash
kubectl apply -f - << EOF
apiVersion: v1
kind: Pod
metadata:
  name: cpu-throttle-demo
spec:
  containers:
  - name: stress
    image: polinux/stress
    resources:
      requests:
        cpu: "100m"
      limits:
        cpu: "200m"
    command: ["stress"]
    args: ["--cpu", "1"]
EOF

# Watch CPU usage — capped near 200m, never higher
kubectl top pod cpu-throttle-demo
# Pod stays Running — just slower than it "wants" to be

kubectl delete pod cpu-throttle-demo
```

---

## Lab 4 — OOMKilled (Memory Limit Exceeded, Pod Dies)

```bash
kubectl run memory-test --image=polinux/stress \
  --requests=memory=50Mi \
  --limits=memory=50Mi \
  -- stress --vm 1 --vm-bytes 100M --vm-hang 1

kubectl get pod memory-test -w
# Status: Running → Error/OOMKilled

kubectl describe pod memory-test | grep -A 3 "Last State"
# Reason: OOMKilled
# Exit Code: 137

kubectl get pod memory-test -o jsonpath='{.status.qosClass}'
# Guaranteed (requests == limits)

kubectl delete pod memory-test
```

---

## Lab 5 — Readiness vs Liveness Probe Behavior

```bash
kubectl apply -f - << EOF
apiVersion: v1
kind: Pod
metadata:
  name: probe-demo
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    readinessProbe:
      httpGet:
        path: /nonexistent-path
        port: 80
      periodSeconds: 5
    livenessProbe:
      httpGet:
        path: /
        port: 80
      periodSeconds: 5
EOF

kubectl get pod probe-demo -w
# Notice: STATUS stays Running (liveness on "/" passes)
# But READY stays 0/1 forever (readiness on "/nonexistent-path" fails)

kubectl describe pod probe-demo | grep -A 5 "Readiness"
# Shows repeated "Readiness probe failed: HTTP probe failed
# with statuscode: 404"

# Check it's excluded from any service endpoints
kubectl expose pod probe-demo --port=80 --name=probe-svc
kubectl get endpoints probe-svc
# ENDPOINTS column will be EMPTY — pod not Ready, excluded

kubectl delete pod probe-demo
kubectl delete svc probe-svc
```

---

## Lab 6 — Secrets are Base64, Not Encrypted

```bash
kubectl create secret generic db-secret \
  --from-literal=host=postgres.banking.svc.cluster.local

# See the raw stored value
kubectl get secret db-secret -o yaml

# Decode it manually — proves it's not encrypted
kubectl get secret db-secret -o jsonpath='{.data.host}' | base64 -d
echo

# Use it in a pod
kubectl apply -f - << EOF
apiVersion: v1
kind: Pod
metadata:
  name: secret-demo
spec:
  containers:
  - name: app
    image: busybox
    command: ["sleep", "3600"]
    env:
    - name: DB_HOST
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: host
EOF

kubectl exec secret-demo -- env | grep DB_HOST

kubectl delete pod secret-demo
kubectl delete secret db-secret
```

---

## Lab 7 — The Label Selector "Trap" (ReplicaSet deletes your manual pod)

```bash
kubectl apply -f - << EOF
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: trap-rs
spec:
  replicas: 2
  selector:
    matchLabels:
      app: trap-demo
  template:
    metadata:
      labels:
        app: trap-demo
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
EOF

kubectl get pods -l app=trap-demo
# Shows 2 pods

# Manually create a THIRD pod with the SAME label
kubectl run manual-pod --image=nginx:1.25 --labels="app=trap-demo"

# Wait a few seconds
kubectl get pods -l app=trap-demo
# Still shows 2 pods! Your manual-pod was DELETED by the ReplicaSet
# controller because it saw 3 pods matching its selector but
# desired only 2

kubectl delete rs trap-rs
```

---

## Lab 8 — Init Container (Wait for Dependency)

```bash
kubectl apply -f - << EOF
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  initContainers:
  - name: wait-for-service
    image: busybox
    command: ['sh', '-c', 'echo waiting...; sleep 10; echo done waiting']
  containers:
  - name: main-app
    image: nginx
EOF

kubectl get pod init-demo -w
# STATUS shows "Init:0/1" for ~10 seconds
# THEN main container starts, STATUS becomes Running

kubectl logs init-demo -c wait-for-service
# "waiting..." then "done waiting"

kubectl delete pod init-demo
```

---

*Next file: 11_Pods_Interview_Questions.md*
