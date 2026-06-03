# Kubernetes — Pods, Deployments and ReplicaSets

> Level: 4 Years Experience
> Cluster: MicroK8s (local) / EKS (production)

---

## 1. Pod — The Smallest Unit

### What It Is

1. A Pod is the smallest deployable unit in Kubernetes
2. A Pod wraps one or more containers that share the same network namespace and storage
3. Every container in a Pod shares the same IP address — they communicate via localhost
4. Pods are ephemeral — they are never repaired, only replaced

### Pod Lifecycle States

| State | Meaning |
|---|---|
| Pending | Accepted but not yet scheduled or image still pulling |
| Running | At least one container is running |
| Succeeded | All containers completed successfully (Jobs) |
| Failed | All containers terminated, at least one failed |
| Unknown | Node lost communication with API server |

### Pod YAML — Understand Every Field

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

### Liveness vs Readiness vs Startup Probes

1. **Liveness Probe** — is the container alive?
   - If it fails — container is restarted
   - Use for: detecting deadlocks, infinite loops
   
2. **Readiness Probe** — is the container ready to serve traffic?
   - If it fails — pod is removed from Service endpoints, no traffic sent
   - Use for: waiting for app to warm up, DB connections to establish

3. **Startup Probe** — did the container start successfully?
   - Disables liveness and readiness until it passes
   - Use for: slow starting applications

### Commands — Pods

```bash
# Create a pod
kubectl run nginx --image=nginx -n banking

# Get pods with all details
kubectl get pods -n banking -o wide

# Describe pod — most important troubleshooting command
kubectl describe pod payment-pod -n banking

# Get pod logs
kubectl logs payment-pod -n banking

# Get previous container logs (after crash)
kubectl logs payment-pod -n banking --previous

# Stream live logs
kubectl logs -f payment-pod -n banking

# Execute into pod
kubectl exec -it payment-pod -n banking -- bash

# Delete pod
kubectl delete pod payment-pod -n banking

# Get pod YAML
kubectl get pod payment-pod -n banking -o yaml

# Watch pod status changes live
kubectl get pods -n banking -w
```

---

## 2. ReplicaSet — Keeping Pods Alive

### What It Is

1. ReplicaSet ensures a specified number of pod replicas are running at all times
2. If a pod dies — ReplicaSet controller creates a replacement immediately
3. If extra pods exist — ReplicaSet deletes the excess
4. You rarely create ReplicaSets directly — Deployments manage them for you

### ReplicaSet YAML

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: payment-rs
  namespace: banking
spec:
  replicas: 3
  selector:
    matchLabels:
      app: payment        # must match pod template labels exactly
  template:
    metadata:
      labels:
        app: payment      # must match selector above
    spec:
      containers:
      - name: payment-app
        image: payment:v1.2
```

### Commands — ReplicaSets

```bash
# Get replicasets
kubectl get rs -n banking

# Describe replicaset
kubectl describe rs payment-rs -n banking

# Scale replicaset manually
kubectl scale rs payment-rs --replicas=5 -n banking

# Delete replicaset and its pods
kubectl delete rs payment-rs -n banking
```

---

## 3. Deployment — Managing ReplicaSets

### What It Is

1. Deployment manages ReplicaSets and provides declarative updates
2. When you update a Deployment — it creates a NEW ReplicaSet and gradually shifts pods to it
3. Old ReplicaSet is kept for rollback purposes
4. This is what you use in production — never standalone pods or raw ReplicaSets

### Deployment YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-deployment
  namespace: banking
spec:
  replicas: 3
  selector:
    matchLabels:
      app: payment
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1           # max pods above desired count during update
      maxUnavailable: 0     # max pods below desired count during update
  template:
    metadata:
      labels:
        app: payment
    spec:
      containers:
      - name: payment-app
        image: payment:v1.2
        resources:
          requests:
            cpu: "250m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "256Mi"
```

### Deployment Update Strategies

1. **RollingUpdate** (default)
   - Replaces pods gradually — zero downtime
   - `maxSurge: 1` — one extra pod created during update
   - `maxUnavailable: 0` — no pod goes down until replacement is ready
   - Best for: production services that need zero downtime

2. **Recreate**
   - Kills ALL pods first, then creates new ones
   - Has downtime
   - Best for: apps that cannot run two versions simultaneously

### Commands — Deployments

```bash
# Create deployment
kubectl create deployment payment --image=payment:v1.2 -n banking

# Apply from file
kubectl apply -f deployment.yaml

# Get deployments
kubectl get deploy -n banking

# Describe deployment
kubectl describe deploy payment-deployment -n banking

# Scale deployment
kubectl scale deploy payment-deployment --replicas=5 -n banking

# Update image (triggers rolling update)
kubectl set image deploy/payment-deployment payment-app=payment:v1.3 -n banking

# Watch rolling update progress
kubectl rollout status deploy/payment-deployment -n banking

# View rollout history
kubectl rollout history deploy/payment-deployment -n banking

# Rollback to previous version
kubectl rollout undo deploy/payment-deployment -n banking

# Rollback to specific version
kubectl rollout undo deploy/payment-deployment --to-revision=2 -n banking

# Pause rolling update
kubectl rollout pause deploy/payment-deployment -n banking

# Resume rolling update
kubectl rollout resume deploy/payment-deployment -n banking

# Delete deployment
kubectl delete deploy payment-deployment -n banking
```

---

## 4. Troubleshooting Scenarios — Pods and Deployments

### Scenario 1 — Pod in CrashLoopBackOff

**What it means:** Container keeps crashing and Kubernetes keeps restarting it.

**Step by step investigation:**

```bash
# Step 1 — check pod status
kubectl get pods -n banking

# Step 2 — get crash logs from previous container
kubectl logs payment-pod -n banking --previous

# Step 3 — describe pod and read Events section
kubectl describe pod payment-pod -n banking

# Step 4 — check if OOMKilled (memory limit too low)
kubectl describe pod payment-pod -n banking | grep -i oom

# Step 5 — exec into pod to debug manually
kubectl exec -it payment-pod -n banking -- bash
```

**Root causes and fixes:**

| Cause | How to Identify | Fix |
|---|---|---|
| App crash on startup | Logs show exception/error | Fix application bug |
| OOMKilled | describe shows OOMKilled | Increase memory limit |
| Wrong command/args | describe shows wrong entrypoint | Fix command in YAML |
| Missing env variable | Logs show config error | Add env var or secret |
| Image does not exist | ImagePullBackOff before crash | Fix image name/tag |

---

### Scenario 2 — Pod Stuck in Pending

**What it means:** Pod is created but no node has been assigned.

```bash
# Step 1 — describe pod and read Events
kubectl describe pod payment-pod -n banking
# Look for "Warning FailedScheduling" in Events

# Step 2 — check node resources
kubectl describe nodes | grep -A 5 "Allocated resources"

# Step 3 — check node taints
kubectl describe nodes | grep Taints

# Step 4 — check if PVC is bound (if pod uses storage)
kubectl get pvc -n banking

# Step 5 — check resource quota in namespace
kubectl describe resourcequota -n banking
```

**Root causes and fixes:**

| Cause | Event Message | Fix |
|---|---|---|
| No nodes with enough CPU/memory | Insufficient CPU | Reduce resource requests or add nodes |
| Node taint mismatch | No nodes match tolerations | Add toleration to pod |
| Node affinity mismatch | No nodes match affinity | Fix affinity rules or label nodes |
| PVC not bound | Unbound PVC | Fix StorageClass or PV |
| ResourceQuota exceeded | Exceeded quota | Increase quota or reduce replicas |

---

### Scenario 3 — Deployment Rollout Stuck

**What it means:** New pods are not coming up, old pods not going down.

```bash
# Step 1 — check rollout status
kubectl rollout status deploy/payment-deployment -n banking

# Step 2 — check new pods status
kubectl get pods -n banking | grep payment

# Step 3 — check new pod logs
kubectl logs -l app=payment -n banking --previous

# Step 4 — check deployment events
kubectl describe deploy payment-deployment -n banking

# Step 5 — rollback immediately if production is affected
kubectl rollout undo deploy/payment-deployment -n banking
```

---

### Scenario 4 — Pod Running But App Not Responding

**What it means:** Pod is Running but requests fail.

```bash
# Step 1 — check if readiness probe is passing
kubectl describe pod payment-pod -n banking | grep -A 10 Readiness

# Step 2 — check endpoints — is pod in service?
kubectl get endpoints payment-service -n banking

# Step 3 — exec into pod and test locally
kubectl exec -it payment-pod -n banking -- curl localhost:8080/health

# Step 4 — check pod logs for errors
kubectl logs payment-pod -n banking -f

# Step 5 — check resource usage
kubectl top pod payment-pod -n banking
```

---

## 5. Interview Questions — Pods and Deployments

### Questions You Will Be Asked

1. **What is the difference between a Pod and a container?**
   - Container is a running process. Pod is the Kubernetes wrapper around one or more containers. Pod gives containers a shared network namespace, shared storage, and a single IP address. You deploy Pods in Kubernetes, not containers directly.

2. **Why do we use Deployments instead of creating Pods directly?**
   - Standalone pods are never replaced if they crash. Deployments manage ReplicaSets which ensure the desired number of pods always run. Deployments also give you rolling updates, rollback history, and scaling — none of which exist with standalone pods.

3. **What is the difference between liveness and readiness probes?**
   - Liveness probe restarts the container if it fails — used to detect deadlocks. Readiness probe removes the pod from service endpoints if it fails — used to stop traffic going to pods that are not ready. A pod can be alive but not ready.

4. **What happens during a rolling update?**
   - Deployment creates a new ReplicaSet with the updated pod template. It gradually scales up new ReplicaSet and scales down old ReplicaSet based on maxSurge and maxUnavailable settings. At no point does traffic drop to zero if configured correctly.

5. **How do you roll back a deployment?**
   - `kubectl rollout undo deploy/<name>` rolls back to the previous ReplicaSet. `--to-revision=N` rolls back to a specific version from rollout history.

6. **What is maxSurge and maxUnavailable?**
   - maxSurge — how many extra pods can exist above desired count during update. maxUnavailable — how many pods can be unavailable during update. Setting maxUnavailable to 0 ensures zero downtime rolling updates.

7. **Pod is in Running state but not serving traffic. Why?**
   - Readiness probe is failing — pod is Running but not Ready, so it is removed from Service endpoints. Check probe configuration and test the health endpoint from inside the pod.

8. **What is a restartPolicy in a Pod?**
   - Always — always restart on failure (default for Deployments). OnFailure — restart only if exit code is non-zero (for Jobs). Never — never restart (for one-time tasks).

---

## 6. MicroK8s — Useful Commands for Practice

```bash
# Check microk8s status
microk8s status

# Enable addons needed for practice
microk8s enable dns
microk8s enable metrics-server
microk8s enable dashboard

# Use kubectl via microk8s
microk8s kubectl get pods -n kube-system

# Or set alias (add to ~/.bashrc)
alias kubectl='microk8s kubectl'
alias k='microk8s kubectl'

# Check control plane processes (not pods in microk8s)
ps aux | grep kube-apiserver
ps aux | grep kube-scheduler
ps aux | grep kube-controller
ps aux | grep etcd

# Check microk8s logs
microk8s inspect
journalctl -u snap.microk8s.daemon-apiserver -f
journalctl -u snap.microk8s.daemon-scheduler -f
journalctl -u snap.microk8s.daemon-controller-manager -f
```

---

*Next: `03_Services_Networking.md`*
