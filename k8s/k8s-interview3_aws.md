# Kubernetes Interview Preparation — Complete Guide
### For DevOps Engineer (4 Years Experience)

---

# TABLE OF CONTENTS
1. Architecture
2. Pods & Workloads (Deployment, ReplicaSet, StatefulSet, DaemonSet)
3. Pod Scheduling (Node Affinity, Taints/Tolerations, Topology Spread)
4. Networking (Service, Ingress, Network Policies, CoreDNS)
5. Storage (PV, PVC, StorageClass)
6. ConfigMap, Secret, Probes
7. Helm
8. RBAC & Security
9. Autoscaling (HPA, VPA, Cluster Autoscaler)
10. Troubleshooting Scenarios (Production-Down Bank)
11. YAML/Code Writing Tasks
12. Quick-Fire Round

---

# 1. ARCHITECTURE

### Q1. Explain Kubernetes architecture (Control Plane + Worker Node).
**Answer:** Control Plane has API Server (entry point for all requests, handles auth/validation), etcd (key-value store, single source of truth for cluster state), Scheduler (assigns pods to nodes based on resources/affinity/taints), and Controller Manager (runs reconciliation loops like Node, ReplicaSet, Endpoint controllers).

Worker Nodes have kubelet (talks to API server, ensures pod containers are running), kube-proxy (manages networking rules for Service routing), and Container Runtime (containerd/CRI-O — runs containers).

**Request flow:** `kubectl apply` → API Server (auth+validation) → etcd (stored) → Controller Manager creates ReplicaSet → ReplicaSet creates Pods → Scheduler assigns node → kubelet pulls image and runs container → kube-proxy updates routing.

**Interviewer Notes:** The "flow" is the differentiator — say it fluently, it shows end-to-end understanding.

---

### Q2. What is etcd? How do you back it up?
**Answer:** etcd is a distributed, consistent key-value store holding all cluster state — every object's spec and status. If etcd is lost, the API server can't serve new requests (existing pods keep running temporarily via kubelet, but the cluster can't reconcile).

Backup command:
```bash
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snap.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```
In managed clusters (EKS/GKE/AKS), the cloud provider manages etcd — important to mention since most 4-year engineers work on managed clusters.

---

### Q3. What happens step-by-step when you run `kubectl apply -f deployment.yaml`?
**Answer (9-step lifecycle):**
1. kubectl sends REST request to API Server
2. API Server authenticates (who) and authorizes via RBAC (allowed?)
3. Admission controllers run (mutating/validating webhooks)
4. Object persisted to etcd
5. Deployment controller creates/updates ReplicaSet
6. ReplicaSet controller creates Pod objects (unscheduled)
7. Scheduler assigns nodes based on resources, affinity, taints
8. kubelet on assigned node pulls image, starts container via runtime
9. kube-proxy updates iptables/IPVS so Service routes to new pod once Ready

**Interviewer Notes:** Most frequently asked architecture question for 4+ YOE. Practice until fluent.

---

### Q4. How does the Scheduler decide which node to place a pod on?
**Answer:** Two-phase process:
- **Filtering (Predicates):** Eliminates nodes that don't meet requirements — insufficient CPU/memory, node selectors don't match, taints without tolerations, port conflicts, volume zone mismatches.
- **Scoring (Priorities):** Ranks remaining nodes — least requested resources, pod affinity/anti-affinity preferences, topology spread, image locality (node already has the image cached scores higher).

The highest-scored node is selected. If no node passes filtering, pod stays `Pending`.

---

### Q5. What is the role of CoreDNS in the cluster?
**Answer:** CoreDNS is the cluster's internal DNS server — runs as a Deployment in `kube-system`. It resolves Service names to ClusterIPs (e.g., `myservice.namespace.svc.cluster.local`) so pods can communicate by name instead of IP. Every pod's `/etc/resolv.conf` points to CoreDNS's ClusterIP via kube-proxy rules.

---

# 2. PODS & WORKLOADS

### Q6. Pod vs ReplicaSet vs Deployment vs StatefulSet vs DaemonSet — when to use what?
**Answer:**
- **Pod** — smallest unit, one or more containers sharing network/storage. Rarely created directly.
- **ReplicaSet** — ensures N replicas of a pod are running. Rarely used directly.
- **Deployment** — manages ReplicaSets, gives rolling updates/rollbacks. Use for **stateless apps** (web servers, APIs).
- **StatefulSet** — for **stateful apps** (databases, message queues). Gives stable pod names (mysql-0, mysql-1), stable network identity (headless service), and per-pod PVCs that persist across restarts. Pods created/deleted in strict order.
- **DaemonSet** — runs exactly one pod per node. Use for node-level agents: log collectors (Fluentd), monitoring agents (node-exporter), CNI plugins.

---

### Q7. SCENARIO: You need to deploy Kafka with 3 brokers, each needing its own persistent storage and stable hostname. Which workload type and why?
**Answer:** **StatefulSet**. Each Kafka broker needs:
- A stable identity (kafka-0, kafka-1, kafka-2) for broker registration
- Its own PVC that survives pod restarts (Kafka data must persist)
- Ordered, predictable scaling (kafka-0 starts first, others join after)

A Deployment would give random pod names and shared/non-persistent identity — unsuitable for a stateful quorum-based system.

---

### Q8. What is a headless Service and why does StatefulSet need it?
**Answer:** A headless Service (`clusterIP: None`) doesn't load-balance — instead, DNS returns the IPs of all individual pods. StatefulSet uses this so each pod gets a stable DNS name: `<pod-name>.<service-name>.<namespace>.svc.cluster.local` (e.g., `kafka-0.kafka-headless.default.svc.cluster.local`). This lets other pods/brokers address each StatefulSet pod individually — critical for clustered apps that need peer discovery.

---

# 3. POD SCHEDULING — Taints, Tolerations, Affinity

### Q9. Explain Taints and Tolerations with a real use case.
**Answer:** Taints are applied to **nodes** to repel pods. Tolerations are applied to **pods** to allow them onto tainted nodes.

```bash
kubectl taint nodes node1 dedicated=gpu:NoSchedule
```

Effects:
- `NoSchedule` — pod won't be scheduled unless it tolerates
- `PreferNoSchedule` — scheduler tries to avoid, but not guaranteed
- `NoExecute` — existing pods without toleration get evicted

**Real use case:** Dedicate GPU nodes to ML workloads only. Taint GPU nodes; only ML pods carry the matching toleration. Regular app pods never land there, even if scheduler would otherwise pick them (e.g., low CPU usage).

---

### Q10. Node Affinity vs Pod Affinity/Anti-Affinity — explain with examples.
**Answer:**
- **Node Affinity** — pod prefers/requires nodes with certain labels. Example: `requiredDuringSchedulingIgnoredDuringExecution` with `disktype=ssd` — pod only schedules on SSD-labeled nodes (use case: database needing fast disk).
- **Pod Affinity** — pod prefers to be scheduled near pods with certain labels (same node/zone). Use case: cache pod co-located with app pod for low latency.
- **Pod Anti-Affinity** — pod prefers NOT to be near pods with certain labels. Use case: spread replicas of the same Deployment across different nodes/zones for high availability — so a single node failure doesn't take down all replicas.

```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchExpressions:
        - key: app
          operator: In
          values: ["myapp"]
      topologyKey: "kubernetes.io/hostname"
```

---

### Q11. SCENARIO: You have 3 replicas of a critical app, but during a node failure, all 3 went down because they were all on the same node. How do you prevent this?
**Answer:** Use **Pod Anti-Affinity** with `topologyKey: kubernetes.io/hostname` (spread across nodes) or `topology.kubernetes.io/zone` (spread across AZs), combined with `requiredDuringSchedulingIgnoredDuringExecution` to enforce it strictly. Additionally, configure a **PodDisruptionBudget** (`minAvailable: 2`) to ensure voluntary disruptions (node drains) don't remove too many replicas at once.

For newer clusters, **Topology Spread Constraints** are the modern preferred approach:
```yaml
topologySpreadConstraints:
- maxSkew: 1
  topologyKey: kubernetes.io/hostname
  whenUnsatisfiable: DoNotSchedule
  labelSelector:
    matchLabels:
      app: myapp
```

---

# 4. NETWORKING

### Q12. ClusterIP vs NodePort vs LoadBalancer vs ExternalName.
**Answer:**
- **ClusterIP** (default) — internal-only virtual IP, for service-to-service communication.
- **NodePort** — exposes service on a static port (30000-32767) on every node's IP; built on top of ClusterIP.
- **LoadBalancer** — provisions cloud LB (AWS ELB/ALB, GCP LB); routes to NodePort/ClusterIP underneath. Standard for external production access.
- **ExternalName** — maps a Service to an external DNS name (e.g., a managed RDS endpoint) — no proxying, just DNS CNAME.

---

### Q13. Ingress vs Ingress Controller — what's the difference?
**Answer:** Ingress is a **resource/config** defining routing rules (host/path → service). Ingress Controller (NGINX, AWS ALB Controller, Traefik) is the **actual proxy/load balancer** that reads those rules and implements them. Ingress without a controller does nothing.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: api.myapp.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-service
            port:
              number: 80
```

---

### Q14. SCENARIO: Pods in namespace A suddenly can't reach pods in namespace B — used to work fine. What changed and how do you fix it?
**Answer:** Almost certainly a **NetworkPolicy** was applied. By default K8s allows all pod-to-pod traffic. Once ANY NetworkPolicy selects a pod (via `podSelector`), that pod becomes **default-deny** for the traffic direction specified — only explicitly allowed traffic passes.

```bash
kubectl get networkpolicy -n namespaceA
kubectl describe networkpolicy <name> -n namespaceA
```

Fix — add explicit ingress rule allowing the other namespace:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-b
spec:
  podSelector: {}
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: namespaceB
```
Note: target namespace must have the `name: namespaceB` label for the selector to match (apply via `kubectl label namespace namespaceB name=namespaceB`).

---

### Q15. How does kube-proxy work? IPVS vs iptables mode?
**Answer:** kube-proxy watches Services/Endpoints and programs networking rules so traffic to a Service's ClusterIP gets load-balanced to backend pods.
- **iptables mode** (default historically) — uses iptables rules, chain traversal becomes slower with thousands of services (O(n) lookup).
- **IPVS mode** — uses Linux IPVS, hash-table based (O(1) lookup), better performance/more LB algorithms (round-robin, least connection) — preferred for large clusters.

---

# 5. STORAGE

### Q16. PV vs PVC vs StorageClass — explain the relationship.
**Answer:**
- **PersistentVolume (PV)** — actual storage resource in the cluster (cloud disk, NFS, etc.), provisioned by admin or dynamically.
- **PersistentVolumeClaim (PVC)** — a request for storage by a user/pod — "I need 10Gi, ReadWriteOnce".
- **StorageClass** — defines HOW storage is dynamically provisioned (which provisioner, e.g., `ebs.csi.aws.com`, parameters like disk type).

Flow: Pod references PVC → PVC requests storage per StorageClass → StorageClass's provisioner dynamically creates a PV → PVC binds to PV → Pod mounts the PVC.

---

### Q17. SCENARIO: A pod is stuck in `Pending` and `kubectl describe pod` shows "pod has unbound immediate PersistentVolumeClaims". How do you fix?
**Answer:**
```bash
kubectl get pvc -n <namespace>
kubectl describe pvc <pvc-name> -n <namespace>
```
Common causes:
1. **No StorageClass exists / wrong name referenced** — check `kubectl get storageclass`
2. **Provisioner failing** — check CSI driver pods in `kube-system` are running
3. **Zone mismatch** — PV provisioned in AZ-1 but pod scheduled in AZ-2 (common with EBS volumes which are zone-locked) — fix with proper topology-aware provisioning or node affinity matching the PV's zone
4. **Insufficient cloud quota** — check cloud provider's volume limits/quota

---

# 6. CONFIGMAP, SECRET, PROBES

### Q18. ConfigMap vs Secret — and a common misconception interviewers test for.
**Answer:** ConfigMap = non-sensitive config (log levels, feature flags). Secret = sensitive data (passwords, tokens, certs).

**Common misconception:** Secrets are NOT encrypted by default — only **base64-encoded** (trivially reversible). For actual encryption, enable **encryption at rest** (EncryptionConfiguration with etcd) or use external secret managers (AWS Secrets Manager, Vault) with tools like External Secrets Operator.

---

### Q19. Liveness vs Readiness vs Startup Probe — with a CrashLoopBackOff scenario.
**Answer:**
- **Liveness** — "is the app alive?" Fail → container restarted.
- **Readiness** — "is the app ready for traffic?" Fail → pod removed from Service endpoints (NOT restarted).
- **Startup** — for slow-starting apps; disables liveness/readiness checks until startup succeeds.

**Scenario:** Java/Spring Boot app takes 90s to start. Liveness probe set with `initialDelaySeconds: 30` → kubelet kills the container before it finishes starting → restarts → kills again → **CrashLoopBackOff**, even though the app would've started fine given more time.

**Fix:** Add Startup probe:
```yaml
startupProbe:
  httpGet:
    path: /health
    port: 8080
  failureThreshold: 30
  periodSeconds: 10   # allows up to 300s for startup
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  periodSeconds: 10
```

---

# 7. HELM

### Q20. What is Helm and why use it?
**Answer:** Helm is K8s's package manager. It packages YAML manifests into reusable, parameterized **charts**. Benefits: templating (one chart, multiple environments via values files), versioning/rollback of entire releases, dependency management between charts, and avoiding copy-pasted YAML across dev/staging/prod.

---

### Q21. Explain Helm chart structure.
**Answer:**
```
mychart/
├── Chart.yaml          # metadata (name, version, appVersion)
├── values.yaml         # default configuration values
├── templates/          # YAML templates with Go templating
│   ├── deployment.yaml
│   ├── service.yaml
│   └── _helpers.tpl    # reusable template snippets
└── charts/             # sub-charts (dependencies)
```

Example templating:
```yaml
# templates/deployment.yaml
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
      - image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

```bash
helm install myrelease ./mychart -f values-prod.yaml
helm upgrade myrelease ./mychart --set image.tag=v2.0
helm rollback myrelease 1
helm history myrelease
```

---

### Q22. SCENARIO: `helm upgrade` succeeded, but pods are still running the old image. What went wrong?
**Answer:** Common causes:
1. **Image tag unchanged** (e.g., `latest` tag, or same tag reused) — K8s doesn't pull a new image if the tag matches and `imagePullPolicy` is `IfNotPresent`. Fix: use unique tags per build (git SHA) or set `imagePullPolicy: Always`.
2. **values.yaml not actually updated** — verify with `helm get values myrelease`
3. **Helm release succeeded but Deployment template has no change** — if only a ConfigMap changed but pod spec didn't, no rollout is triggered. Fix: add a checksum annotation:
```yaml
annotations:
  checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
```
This forces a pod restart when the ConfigMap content changes.

---

# 8. RBAC & SECURITY

### Q23. Explain RBAC components: Role, ClusterRole, RoleBinding, ClusterRoleBinding.
**Answer:**
- **Role** — set of permissions (verbs: get/list/watch/create/delete on resources) scoped to a **namespace**.
- **ClusterRole** — same but **cluster-wide** (or for cluster-scoped resources like Nodes, PVs).
- **RoleBinding** — grants a Role's permissions to a user/group/ServiceAccount **within a namespace**.
- **ClusterRoleBinding** — grants a ClusterRole **cluster-wide**.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: dev
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: dev
subjects:
- kind: ServiceAccount
  name: ci-deployer
  namespace: dev
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

---

### Q24. SCENARIO: Your CI/CD pipeline's ServiceAccount gets "Forbidden" error when trying to deploy. How do you debug?
**Answer:**
```bash
kubectl auth can-i create deployments --as=system:serviceaccount:ci:ci-deployer -n prod
kubectl get rolebinding,clusterrolebinding -n prod -o wide | grep ci-deployer
kubectl describe rolebinding <name> -n prod
```
Check: Does the ServiceAccount have a RoleBinding/ClusterRoleBinding? Does the bound Role include the needed verb (`create`) and resource (`deployments`, also check `apiGroups: ["apps"]`)? Is it bound in the correct namespace? Fix by creating/updating the RoleBinding with correct permissions — follow least-privilege, don't just grant `cluster-admin`.

---

### Q25. What are Security Contexts and Pod Security Standards?
**Answer:** SecurityContext controls privilege/access at pod or container level:
```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]
```
Pod Security Standards (replacing deprecated PodSecurityPolicy) define three levels: **Privileged**, **Baseline**, **Restricted** — enforced via namespace labels, e.g., `pod-security.kubernetes.io/enforce: restricted`.

---

# 9. AUTOSCALING

### Q26. HPA vs VPA vs Cluster Autoscaler.
**Answer:**
- **HPA (Horizontal Pod Autoscaler)** — scales number of pod replicas based on CPU/memory/custom metrics.
- **VPA (Vertical Pod Autoscaler)** — adjusts CPU/memory **requests/limits** of existing pods (requires restart to apply).
- **Cluster Autoscaler** — adds/removes **nodes** when pods are Pending due to insufficient capacity, or removes underutilized nodes.

Typically used together: HPA scales pods → if no node has capacity → Cluster Autoscaler adds nodes.

---

### Q27. SCENARIO: HPA shows `<unknown>` for current CPU utilization and isn't scaling. Debug steps?
**Answer:**
```bash
kubectl get hpa
kubectl describe hpa <name>
kubectl top pods   # if this fails, metrics-server issue
kubectl get apiservice v1beta1.metrics.k8s.io
```
Causes:
1. **metrics-server not running/crashed** — `kubectl get pods -n kube-system | grep metrics-server`
2. **No resource `requests` set on the deployment** — HPA needs `requests.cpu` to compute %; without it, can't calculate utilization
3. **metrics-server can't reach kubelet** (TLS/cert issues in some setups) — check metrics-server logs

---

# 10. TROUBLESHOOTING SCENARIO BANK (Production-Down Focus)

### Q28. CrashLoopBackOff — full debugging walkthrough.
```bash
kubectl get pods -n <ns>
kubectl describe pod <pod> -n <ns>          # Events section - WHY
kubectl logs <pod> -n <ns> --previous       # crashed container's logs
kubectl describe pod <pod> | grep -A5 "Last State"  # OOMKilled?
kubectl get configmap,secret -n <ns>        # missing config?
```
**Talk-through:** "Describe pod tells me the reason — OOMKilled, bad probe, error exit code. `--previous` is critical because the current container already restarted and lost the crash logs. If OOMKilled, I check resource limits vs actual usage. If application error, I check env vars/ConfigMaps/Secrets for missing values. If image issue, I verify tag and registry access."

**Top root causes ranked:** (1) OOMKilled — memory limit too low, (2) Bad config/missing env var causing app to exit on startup, (3) Liveness probe killing app before it's ready, (4) Application bug in new image (exit code 1).

---

### Q29. ImagePullBackOff — all causes and fixes.
**Answer:**
1. Wrong image name/tag (typo) → fix YAML, `kubectl apply`
2. Private registry, no `imagePullSecrets` → create docker-registry secret, reference in pod spec
3. Expired registry credentials → recreate secret with fresh token
4. Network/firewall blocks registry → check node egress to registry

`kubectl describe pod` shows exact error — "manifest not found" (tag issue) vs "unauthorized" (auth issue) vs "x509" (cert/network issue).

---

### Q30. Production Down: All pods 0/1 Ready but status "Running". Priority order of actions?
**Answer:** This is a readiness probe failure — pods running but not in Service endpoints.

**Priority: STABILIZE FIRST, THEN DEBUG.**
1. `kubectl rollout status deployment/<name>` and `kubectl rollout history` — was this caused by a recent deploy?
2. If YES → **immediately** `kubectl rollout undo deployment/<name>` to restore service
3. THEN investigate: `kubectl describe pod`, `kubectl logs` — why readiness probe failing? New image bug? Downstream dependency (DB/API) unreachable?

**Interviewer Notes:** Saying "rollback first, root-cause second" unprompted is the #1 signal of production maturity for senior roles.

---

### Q31. Node at 95% CPU, pods getting evicted — immediate and long-term fixes.
**Answer:**
**Immediate:**
```bash
kubectl top nodes
kubectl top pods --all-namespaces --sort-by=cpu
kubectl cordon <node>          # stop new scheduling
```
Identify the noisy-neighbor pod, scale it down or restart it, check if it has no CPU limit set (can starve the node).

**Long-term:**
- Set proper `requests`/`limits` on all workloads
- HPA for variable-load apps
- Cluster Autoscaler for capacity
- PodDisruptionBudgets so evictions don't cascade across all replicas

---

### Q32. Rolling update stuck — `kubectl rollout status` hangs forever.
**Answer:** New pods aren't reaching Ready, so old pods aren't terminated (rollout can't proceed).
```bash
kubectl get pods                 # new pods: Pending? CrashLoop? 0/1?
kubectl describe pod <new-pod>   # Events
```
- New image has a bug, readiness never passes → `kubectl rollout undo deployment/<name>`
- Insufficient cluster resources → new pods Pending → "Insufficient cpu/memory" in describe
- `maxUnavailable`/`maxSurge` too restrictive for small replica counts

To investigate without further disruption: `kubectl rollout pause deployment/<name>`.

---

### Q33. Intermittent 502/504 errors with healthy-looking pods.
**Answer:** Check in order:
1. **Readiness flapping** — `kubectl get endpoints <svc>` over time; pod added/removed repeatedly from endpoints
2. **CPU throttling** — `kubectl top pods` vs limits; throttling causes slow responses → LB timeout
3. **Connection draining on deploy** — old pods killed mid-request. Fix: `preStop` hook with sleep + proper `terminationGracePeriodSeconds`
4. **Ingress/LB health check path mismatch** vs actual `/health` endpoint
5. **CoreDNS overloaded** — check CoreDNS pod logs/resources for intermittent DNS failures

Correlate via: `kubectl get events --sort-by=.metadata.creationTimestamp`

---

### Q34. Debugging a crashed distroless pod (no shell available).
**Answer:**
```bash
kubectl debug -it <pod-name> --image=busybox --target=<container-name>
kubectl logs <pod-name> --previous
kubectl get pod <pod-name> -o yaml   # exit codes, last state
```
`kubectl debug` attaches an ephemeral debug container sharing the process namespace — inspect processes/network/filesystem from there without modifying the original image.

---

### Q35. Pod stuck in `Pending` for 10+ minutes — full diagnostic checklist.
**Answer:** `kubectl describe pod` Events will show one of:
1. "Insufficient cpu/memory" → no node has capacity; scale cluster or reduce requests
2. nodeSelector/affinity doesn't match any node → check `kubectl get nodes --show-labels`
3. PVC not bound → `kubectl get pvc`, check StorageClass/provisioner
4. All nodes tainted, pod has no toleration
5. Admission webhook silently rejecting → `kubectl get events -n <ns>`

---

# 11. YAML / SCRIPT WRITING TASKS

### Q36. Write a Deployment YAML: 3 replicas, resource requests/limits, readiness+liveness probes, env from ConfigMap and Secret, pod anti-affinity to spread across nodes.
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: myapp
              topologyKey: "kubernetes.io/hostname"
      containers:
      - name: myapp
        image: myrepo/myapp:1.2.3
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: "250m"
            memory: "256Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 10
        envFrom:
        - configMapRef:
            name: myapp-config
        - secretRef:
            name: myapp-secret
```

---

### Q37. Write a shell script to check if any pod in a namespace has restarted more than N times in the last hour and alert.
```bash
#!/bin/bash
NAMESPACE="$1"
THRESHOLD="$2"

if [[ -z "$NAMESPACE" || -z "$THRESHOLD" ]]; then
  echo "Usage: $0 <namespace> <restart-threshold>"
  exit 1
fi

kubectl get pods -n "$NAMESPACE" -o json | \
jq -r '.items[] | "\(.metadata.name) \(.status.containerStatuses[0].restartCount)"' | \
while read -r pod restarts; do
  if [[ "$restarts" -gt "$THRESHOLD" ]]; then
    echo "ALERT: Pod $pod in namespace $NAMESPACE has restarted $restarts times!"
    # Send to Slack/webhook here
    # curl -X POST -H 'Content-type: application/json' --data "{\"text\":\"ALERT: $pod restarted $restarts times\"}" $SLACK_WEBHOOK
  fi
done
```

---

### Q38. Write a NetworkPolicy that allows traffic to `backend` pods only from `frontend` pods on port 8080, and denies everything else.
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-allow-frontend
  namespace: prod
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
```
*(Once this NetworkPolicy is applied, `backend` pods become default-deny for ingress — only frontend on 8080 is allowed; all other traffic blocked.)*

---

### Q39. Write a HorizontalPodAutoscaler for a deployment to scale between 2-10 replicas at 70% CPU.
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

---

# 12. QUICK-FIRE RECALL (Answer in under 20 seconds each)

| Q | A |
|---|---|
| Diff: `kubectl delete pod` vs natural pod death? | If managed by controller, immediately recreated to maintain replica count — not truly "gone" |
| What is a PodDisruptionBudget? | Limits voluntary disruptions (drains/upgrades) — ensures min availability during maintenance |
| `rollout restart` vs `rollout undo`? | `restart` = same spec, new pods (picks up ConfigMap changes); `undo` = revert to previous revision's spec |
| Why multi-stage Docker builds? | Smaller images → faster pod startup, less storage, smaller attack surface |
| What is `kubectl drain`? | Safely evicts pods from a node for maintenance, respecting PDBs, then cordons the node |
| StatefulSet pod naming? | Predictable ordinal names: `mysql-0`, `mysql-1`, `mysql-2` |
| Default Service type? | ClusterIP |
| What triggers Cluster Autoscaler to add a node? | Pods stuck `Pending` due to "Insufficient cpu/memory" that a new node would resolve |
| Difference: `requests` vs `limits`? | `requests` = scheduling guarantee (minimum reserved); `limits` = hard ceiling (OOMKill/throttle if exceeded) |
| What is `terminationGracePeriodSeconds`? | Time given to a pod to shut down gracefully (SIGTERM) before SIGKILL — default 30s |

---

# 13. ADVANCED / FAANG-TIER TOPICS (Amazon, Google, IBM, Flipkart)

These are the topics that separate a "passed mid-level screen" candidate from one who clears senior/FAANG loops. If you only study Sections 1-12, you WILL get caught here.

### Q40. What are CRDs (Custom Resource Definitions) and Operators? Give a real example.
**Answer:** A CRD lets you define your OWN object type that the K8s API treats like a built-in resource (e.g., `kubectl get postgresdbs`). An **Operator** is a controller that watches these custom resources and takes action to manage an application's full lifecycle — install, upgrade, backup, failover — encoding human operational knowledge into code.

**Real example:** The Prometheus Operator — you create a `ServiceMonitor` CRD, and the operator automatically configures Prometheus to scrape that target, without manually editing Prometheus config. Similarly, the Postgres Operator (Zalando/CrunchyData) handles automated failover, backups, and scaling for a Postgres cluster running on K8s.

**Interviewer Notes:** Senior/FAANG interviewers ask this to gauge if you've worked beyond "deploy app via Deployment" — if your org uses Operators (cert-manager, Prometheus Operator, ArgoCD), mention it.

---

### Q41. What is GitOps? How does ArgoCD/Flux fit into K8s deployments?
**Answer:** GitOps means the desired state of the cluster is defined declaratively in a Git repo, and a controller (ArgoCD/Flux) continuously reconciles the live cluster state to match Git — Git is the single source of truth, not `kubectl apply` from a pipeline.

**Flow:** Developer merges PR changing `replicas: 3` to `5` in Git → ArgoCD detects drift → automatically (or after approval) applies the change to the cluster → ArgoCD UI shows "Synced/OutOfSync" status.

**Benefits over traditional CI/CD push:** audit trail via git history, easy rollback (`git revert`), no CI system needs cluster credentials (pull-based, more secure), automatic drift detection (if someone manually `kubectl edit`s something, ArgoCD flags/reverts it).

---

### Q42. SCENARIO: A service passes health checks but returns 503 for ~3% of requests. Walk through your diagnosis. (Common at senior loops)
**Answer:** This is a "needle in haystack" question — health checks passing means the AGGREGATE is healthy, but a small percentage failing points to a subset issue.

**Systematic approach:**
1. **Is it always the same 3%?** Check if it correlates with a specific pod (`kubectl get endpoints`, then check logs/metrics per-pod — one pod might be unhealthy but still passing readiness probe because the probe checks something different than the actual request path)
2. **Is it a specific request type/endpoint?** 3% might equal exactly the traffic hitting one particular downstream dependency (e.g., a specific DB shard or a flaky third-party API)
3. **Timing-based?** Check if 503s correlate with deploys/rolling updates (connection draining issue — `preStop` hook missing) or with HPA scale-down events (pod terminated mid-request)
4. **Load balancer level** — is the LB's health check endpoint different from the actual serving path? A pod could be "Ready" per probe but failing on the real endpoint due to a partial dependency outage
5. Use distributed tracing (if available) to find the exact pod+timestamp+error for failing requests, then `kubectl logs` that specific pod at that timestamp

**Interviewer Notes:** They want to see a hypothesis-driven, narrowing approach — not "I'd check kubectl get pods". This question tests systems-thinking under ambiguity, the core senior/SRE skill.

---

### Q43. What is a Service Mesh (Istio/Linkerd)? Why would you add one to K8s?
**Answer:** A service mesh adds a transparent proxy (sidecar, e.g., Envoy) next to every pod, handling service-to-service communication. It provides:
- **mTLS encryption** between all services automatically (zero-trust networking)
- **Traffic management** — canary releases, traffic splitting (e.g., 90% to v1, 10% to v2), retries, circuit breaking — without changing app code
- **Observability** — automatic metrics/traces for all inter-service calls
- **Fine-grained authorization policies** between services

**Trade-off to mention:** adds latency (extra network hop through sidecar), operational complexity, and resource overhead per pod — so it's justified at a certain scale (microservices count, compliance needs for mTLS), not for small clusters.

---

### Q44. Explain the difference between `Recreate`, `RollingUpdate`, and Blue-Green/Canary deployment strategies in K8s.
**Answer:**
- **Recreate** — kills all old pods, then creates new ones. Causes downtime. Use only for apps that can't run two versions simultaneously (e.g., schema-incompatible DB migrations).
- **RollingUpdate** (default) — gradually replaces old pods with new ones, controlled by `maxUnavailable`/`maxSurge`. Zero-downtime if probes are correct.
- **Blue-Green** — run two full environments (v1 "blue", v2 "green"); switch traffic at the Service/Ingress level all at once. Instant rollback (switch back), but needs 2x resources temporarily.
- **Canary** — route a small % of traffic to the new version first (via Service Mesh or Ingress weight), monitor metrics, then gradually increase. Lowest risk, but needs traffic-splitting infra (Istio, or Argo Rollouts).

**Argo Rollouts** is the common K8s-native tool for canary/blue-green beyond what Deployment natively supports.

---

### Q45. What is the difference between `resources.requests` causing scheduling vs `limits` causing throttling/OOMKill — and how does CPU throttling actually work at the kernel level?
**Answer:** `requests` are used ONLY by the scheduler to decide node placement (sum of requests must fit on a node). `limits` are enforced by the kernel via **cgroups**:
- **CPU limit exceeded** → CFS (Completely Fair Scheduler) **throttles** the process — it doesn't get killed, just gets less CPU time per period (visible as increased latency, not crashes)
- **Memory limit exceeded** → kernel **OOMKiller** kills the process immediately — pod restarts (`OOMKilled` in `kubectl describe`)

**Interviewer Notes:** This is a CKA/senior-level distinction. Many candidates think CPU limit = pod gets killed like memory — wrong. CPU = throttle (slow), Memory = kill (crash). Knowing this explains "why is my pod slow but not crashing" scenarios.

---

### Q46. SCENARIO: You need zero-downtime during a cluster upgrade (control plane + node pool upgrade). Walk through your approach.
**Answer:**
1. **Control plane upgrade first** (in managed K8s like EKS/GKE, this is often handled by the provider with minimal app impact since control plane HA means API server stays available)
2. **Node pool upgrade** — use a rolling node replacement strategy:
   - Create a new node group with the new K8s version
   - `kubectl cordon` old nodes (stop new scheduling)
   - `kubectl drain` old nodes one at a time — this respects PodDisruptionBudgets and gracefully evicts pods, which get rescheduled on new nodes
   - Ensure PDBs are set (`minAvailable`) on critical Deployments so draining doesn't take down all replicas at once
   - Verify workloads healthy on new nodes before terminating old node group
3. **Pre-checks:** confirm app's K8s API version compatibility (deprecated APIs removed between versions — check with `kubectl deprecations` or `pluto` tool)

---

### Q47. What's the difference between `kubectl exec`, `kubectl attach`, and `kubectl port-forward`?
**Answer:**
- **`exec`** — runs a NEW command inside the container (e.g., open a shell): `kubectl exec -it pod -- /bin/sh`
- **`attach`** — attaches to the EXISTING running process's stdin/stdout (PID 1) — useful to see what an app is printing without starting a new process
- **`port-forward`** — tunnels a local port to a port inside the pod, for debugging a service without exposing it externally: `kubectl port-forward pod/mypod 8080:80`

---

### Q48. How does Kubernetes handle a node failure (NotReady)? Timeline and what happens to pods?
**Answer:**
1. kubelet stops sending heartbeats → Node Controller marks node `NotReady` after `node-monitor-grace-period` (default 40s)
2. After `pod-eviction-timeout` (default 5 minutes), pods on that node are marked for deletion
3. If pods are managed by a Deployment/ReplicaSet/StatefulSet, the controller creates replacements on healthy nodes
4. **Important:** pods aren't FORCEFULLY deleted from the dead node immediately — they stay in `Terminating`/`Unknown` state until the node comes back or is manually deleted, because the kubelet (which would normally clean up) is unreachable

**Interviewer Notes:** Mentioning the actual timeouts (40s, 5min) shows real production/CKA-level knowledge — most candidates just say "K8s reschedules it" without timing detail.

---

# 14. WHAT INTERVIEWERS ACTUALLY TEST — TOPIC WEIGHTAGE MAP

Based on current (2026) hiring patterns at IBM/Amazon/Flipkart/Google-tier DevOps loops:

| Topic | Frequency | Format Used | Why They Test It |
|---|---|---|---|
| **Troubleshooting (CrashLoop, OOMKilled, ImagePullBackOff, Pending)** | Very High — almost every loop | Scenario, live debugging | Tests if you've actually run production clusters, not just learned theory |
| **Architecture & request lifecycle (Q1-Q3)** | Very High | Conceptual, "explain to me" | Filters theory-only candidates; expected to be fluent |
| **Networking (Services, Ingress, NetworkPolicy, DNS)** | High | Scenario + conceptual | Networking issues are the #1 real-world pain point |
| **RBAC & Security** | High at Amazon/Google (security-conscious orgs) | Scenario ("CI pipeline forbidden") | Compliance/least-privilege culture |
| **Probes (liveness/readiness/startup)** | High | Scenario | Misconfigured probes = #1 cause of "works on dev, fails in prod" |
| **YAML writing live** | High | Live coding / shared editor | Tests muscle memory — can't fake it |
| **Helm** | Medium-High | Conceptual + scenario | Standard packaging tool; "why didn't my upgrade apply" is common |
| **Autoscaling (HPA/Cluster Autoscaler)** | Medium | Scenario | Cost optimization is a hot topic at all 4 companies currently |
| **StatefulSets / storage (PV/PVC)** | Medium | Conceptual | Less frequent unless role involves databases on K8s |
| **GitOps (ArgoCD/Flux)** | Medium-High at Amazon/IBM (platform teams) | Conceptual | Modern deployment standard — increasingly assumed knowledge |
| **CRDs/Operators** | Medium at Google, growing elsewhere | Conceptual | Tests exposure to extensibility/ecosystem maturity |
| **Service Mesh (Istio)** | Low-Medium, higher at Google | Conceptual | Only if role touches microservices at scale |
| **Multi-layer ambiguous scenarios (Q42-style "503 for 3%")** | Growing trend 2026, esp. senior loops | Open-ended scenario | Tests systems-thinking, not memorized fixes |
| **CKA-level kernel internals (cgroups, eviction timelines)** | Medium-High at Google/Amazon (deeper bar) | Conceptual "why" follow-ups | Differentiates senior from mid-level |

### How the interview typically flows (all 4 companies, consistent pattern):
1. **Warm-up** — 2-3 conceptual questions (architecture, Pod vs Deployment) to gauge baseline
2. **Core scenario round** — 1-2 CrashLoopBackOff/Pending/networking debugging scenarios, they push deeper with "what if that didn't work, then what?" — they're testing your DEBUGGING TREE, not just the answer
3. **Live YAML/coding** — write a Deployment/NetworkPolicy/HPA from scratch, sometimes with intentional bugs to find
4. **Open-ended systems question** — ambiguous production issue (Q42-style), evaluating structured thinking
5. **Behavioral tie-in** — "tell me about a time you debugged a production K8s issue" — STAR format, connects to your real experience

**Your highest-leverage prep:** Sections 10 (troubleshooting bank) + Section 13 (Q42, Q45, Q48) + live YAML practice from Section 11. These cover ~70% of what gets asked across all 4 target companies.

---

# COVERAGE CONFIRMATION
This file now covers, end-to-end:
✅ Architecture (control plane, etcd, scheduler, request lifecycle, scheduler internals, CoreDNS)
✅ Workloads (Pod, ReplicaSet, Deployment, StatefulSet, DaemonSet, headless services)
✅ Scheduling (Taints/Tolerations, Node/Pod Affinity & Anti-Affinity, Topology Spread)
✅ Networking (Service types, Ingress vs Controller, NetworkPolicy, kube-proxy/IPVS, CoreDNS)
✅ Storage (PV/PVC/StorageClass, zone-binding issues)
✅ ConfigMap/Secret (incl. encryption misconception), all 3 probe types
✅ Helm (structure, templating, upgrade-not-applying scenario)
✅ RBAC (Role/ClusterRole/Bindings, ServiceAccount debugging)
✅ Security Contexts & Pod Security Standards
✅ Autoscaling (HPA, VPA, Cluster Autoscaler + debugging)
✅ 8 deep troubleshooting scenarios (CrashLoop, ImagePullBackOff, prod-down, node CPU, stuck rollout, 502/504, distroless debug, Pending)
✅ YAML/script writing (Deployment, NetworkPolicy, HPA, shell monitoring script)
✅ Quick-fire recall table
✅ CRDs/Operators, GitOps/ArgoCD, Service Mesh, deployment strategies (canary/blue-green), cgroup-level CPU vs memory limit behavior, node failure timeline
✅ Topic weightage map + interview flow pattern for IBM/Amazon/Flipkart/Google

If you complete ALL 48 questions across Sections 1-13 fluently (spoken, not read), you are prepared for K8s portions of these interviews. Remaining gaps would only be company-specific (e.g., EKS-specific IAM/IRSA quirks for Amazon, GKE Autopilot specifics for Google) — ask if you want those added too.

---

# 15. AWS / EKS CLOUD QUIRKS (Real-Industry + 2026 Updates)

This is the "have you actually run EKS in production" section — generic K8s knowledge doesn't cover these, and Amazon/IBM/Flipkart interviewers specifically probe here since most prod clusters run on EKS.

### Q49. EKS vs self-managed K8s — what does AWS manage vs what you manage?
**Answer:** AWS manages the **control plane** entirely — API server, etcd, scheduler, controller manager — across multiple AZs for HA, and patches/upgrades it. You manage:
- **Worker nodes** (EC2 managed node groups, self-managed EC2, or Fargate for serverless pods)
- **VPC/networking** (subnets, security groups, VPC CNI configuration)
- **Add-ons** (CoreDNS, kube-proxy, VPC CNI — though EKS now offers these as managed add-ons)
- **IAM integration** for cluster and workload access

You don't get SSH/direct access to control plane nodes or etcd — `etcdctl snapshot` style backups aren't something you do on EKS; AWS handles that.

---

### Q50. What is IRSA (IAM Roles for Service Accounts)? How does a pod get AWS permissions?
**Answer:** IRSA lets you assign an IAM role to a Kubernetes ServiceAccount (not the whole node), so pods get only the AWS permissions they need — least privilege, instead of giving the entire EC2 node's IAM role to every pod.

**How it works:**
1. EKS cluster has an OIDC identity provider associated with it
2. Create an IAM role with a trust policy that trusts this OIDC provider, scoped to a specific namespace+ServiceAccount
3. Annotate the ServiceAccount: `eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/my-app-role`
4. The pod, via the EKS Pod Identity Webhook, gets a projected token + env vars (`AWS_ROLE_ARN`, `AWS_WEB_IDENTITY_TOKEN_FILE`) that the AWS SDK automatically uses to assume the role

**2026 update — EKS Pod Identity:** This is the newer, simpler replacement for IRSA introduced in 2024 and reached general availability in 2025. It uses a per-cluster OIDC provider and simplifies role association via annotations, with advantages over IRSA including not needing to create an OIDC provider per cluster, no trust policy changes when clusters are recreated, and better performance. If asked in an interview, mention both — IRSA is still widely deployed in existing clusters, but Pod Identity is the current recommendation for new setups.

---

### Q51. SCENARIO: A pod can't access an S3 bucket even though the IAM role attached via IRSA has `s3:GetObject` permission. Debug steps?
**Answer:**
1. **Check ServiceAccount annotation is correct:** `kubectl get sa <name> -n <ns> -o yaml` — verify `eks.amazonaws.com/role-arn` matches the actual IAM role ARN exactly
2. **Check the pod is actually USING that ServiceAccount:** `kubectl get pod <pod> -o jsonpath='{.spec.serviceAccountName}'` — a common mistake is the Deployment doesn't specify `serviceAccountName`, so it defaults to `default` SA which has no role
3. **Check the IAM role's trust policy:** must trust the EKS OIDC provider AND be scoped with a condition matching `system:serviceaccount:<namespace>:<sa-name>` — a typo in namespace/SA name in the trust policy is extremely common
4. **Check the token is mounted:** `kubectl exec` into pod, verify `AWS_WEB_IDENTITY_TOKEN_FILE` env var exists and the file is present
5. **Check the IAM policy itself** — does it have `s3:GetObject` on the correct bucket ARN/path (including `/*` for objects vs bucket-level permission)?
6. **Check S3 bucket policy** — even if IAM allows it, an explicit `Deny` in the bucket policy overrides IAM allow

**Interviewer Notes:** Step 3 (trust policy namespace/SA mismatch) is the #1 real-world cause — mention it early to signal hands-on IRSA debugging experience.

---

### Q52. SCENARIO: New nodes aren't joining the EKS cluster after scaling the node group. Debug steps?
**Answer:**
1. Check `aws-auth` ConfigMap (legacy) or **EKS Access Entries** (2026 recommended) — the node's IAM role must be mapped to `system:bootstrappers` and `system:nodes` groups. If this mapping is missing/wrong, nodes register with AWS but never appear as `Ready` in `kubectl get nodes`
2. Check node's security group allows traffic to/from the control plane security group (specific ports for kubelet-API server communication)
3. Check subnet has route to internet (NAT Gateway) or VPC endpoints for pulling EKS bootstrap files / ECR images if in private subnets
4. SSH/SSM into the node, check `/var/log/cloud-init-output.log` and kubelet logs (`journalctl -u kubelet`) for bootstrap errors — wrong cluster name/endpoint/CA cert in the bootstrap script
5. Check the node's instance type has available capacity in chosen AZ (`InsufficientInstanceCapacity` errors)

---

### Q53. What is Karpenter and how is it different from Cluster Autoscaler?
**Answer:** Karpenter is AWS's modern node-provisioning tool, replacing Cluster Autoscaler in many 2025/2026 setups.

- **Cluster Autoscaler** — scales predefined node groups (ASGs) up/down based on Pending pods; limited to instance types defined in those ASGs
- **Karpenter** — directly provisions EC2 instances (no ASG needed), picks the OPTIMAL instance type/size/AZ for pending pods in real-time (bin-packing aware), supports Spot instances natively with automatic fallback to on-demand, and scales down faster by consolidating underutilized nodes

**Real benefit:** if a pod needs 3.5 vCPU, Karpenter can launch exactly the right-sized instance instead of being constrained to whatever instance types are in a pre-configured node group — significant cost savings at scale.

---

### Q54. EKS networking quirk: What is the VPC CNI "IP exhaustion" problem and how do you fix it?
**Answer:** By default, the AWS VPC CNI assigns each pod a **real VPC IP address** from the subnet (unlike other CNIs that use overlay networks). This means:
- Each EC2 instance type has an ENI/IP limit (e.g., a `t3.medium` supports ~17 pod IPs)
- Large clusters can exhaust subnet IP space fast, especially in smaller `/24` subnets

**Fixes:**
1. Use larger CIDR subnets for pod networking
2. Enable **prefix delegation** (`ENABLE_PREFIX_DELEGATION=true`) — assigns /28 IP prefixes to ENIs instead of individual IPs, massively increasing pod density per node
3. Use a separate CIDR range for pods via **custom networking** (secondary CIDR, e.g., `100.64.0.0/16`) to avoid exhausting the primary VPC CIDR
4. Switch to Fargate for some workloads (no node-level IP constraints in the same way)

---

### Q55. SCENARIO: ALB Ingress (AWS Load Balancer Controller) isn't creating a Load Balancer for your Ingress resource. Debug steps?
**Answer:**
1. Check the AWS Load Balancer Controller pod is running: `kubectl get pods -n kube-system | grep aws-load-balancer`
2. Check the controller's IRSA permissions — it needs IAM permissions to create ELB/ALB, target groups, security groups
3. Check Ingress annotations: `kubernetes.io/ingress.class: alb` (or `spec.ingressClassName: alb`) and `alb.ingress.kubernetes.io/scheme: internet-facing` (or `internal`)
4. Check subnets are tagged correctly: `kubernetes.io/role/elb=1` for public, `kubernetes.io/role/internal-elb=1` for private — ALB controller auto-discovers subnets via these tags
5. `kubectl describe ingress <name>` — events show the exact AWS API error (often IAM permission denied or subnet discovery failure)
6. Check controller logs: `kubectl logs -n kube-system deploy/aws-load-balancer-controller`

---

### Q56. SCENARIO: Production is down, 100x traffic spike during a sale event (Flipkart/Amazon-style scenario). How do you handle it end-to-end?
**Answer:** This tests planning + real-time response.

**Pre-event (planning):**
- Pre-scale node groups/Karpenter limits and HPA `maxReplicas` BEFORE the event based on load-test projections — don't rely on reactive autoscaling alone for predictable 100x spikes, since scaling has lag (new EC2 nodes take 1-3 min to join)
- Reserve capacity (Reserved Instances/Savings Plans or pre-warmed node pools) for guaranteed baseline
- Set up read replicas/caching (ElastiCache/Redis) in front of the database — DB is usually the bottleneck, not the app pods
- Set realistic HPA thresholds with fast-reacting metrics (custom metrics via CloudWatch/Prometheus, not just CPU)

**During the event (if still degraded):**
1. Check `kubectl top nodes/pods`, HPA status — is scaling keeping up?
2. Check database connections/CPU — often the actual bottleneck (connection pool exhaustion)
3. Check Cluster Autoscaler/Karpenter logs — are new nodes failing to join (capacity issues in the AZ)?
4. If a dependent service is overwhelmed, apply circuit breakers / rate limiting at the Ingress/API Gateway level to shed load gracefully rather than total outage
5. Use feature flags to disable non-critical features (recommendations, reviews) to reduce load on core checkout path

---

### Q57. What is the difference between EKS Fargate and EC2-based node groups? When would you use Fargate?
**Answer:** Fargate runs pods on serverless compute — no EC2 nodes to manage, AWS provisions right-sized compute per pod automatically.

**Use Fargate for:** batch jobs, unpredictable/spiky workloads where you don't want idle node capacity, multi-tenant clusters needing strong isolation (each pod gets its own kernel-level isolation/microVM), or teams that don't want to manage node patching/AMIs.

**Use EC2 node groups for:** DaemonSets (Fargate doesn't support DaemonSets — no access to host-level agents), workloads needing GPU, persistent local storage, or higher pod density at lower cost per vCPU (Fargate has a per-pod pricing premium).

---

### Q58. SCENARIO: A container is OOMKilled and the developer says "this is a Kubernetes bug." How do you respond? (Real interview question, Amazon/IBM-style)
**Answer:** It's not a Kubernetes problem — the container used all the memory it was allocated. Explain calmly:

"Kubernetes enforced the memory limit you (or the default) configured — it's doing exactly what it's supposed to do, protecting the node from one container consuming all its memory. The real question is: why is the APP using more memory than expected? Possible causes: a memory leak in the app, the limit was set too low for actual workload (e.g., JVM heap settings not aligned with container memory limit — classic issue where JVM doesn't respect cgroup limits without `-XX:+UseContainerSupport`), or a traffic spike causing legitimate higher memory usage. I'd pull `kubectl top pod` history/Grafana memory graphs over time to see if it's a gradual leak (sawtooth pattern resetting on each OOMKill) vs a sudden spike, then work with the dev team on the app-level fix — increasing the limit is a band-aid, not a root-cause fix unless the limit was simply miscalibrated."

**Interviewer Notes:** This question tests communication + technical accuracy under a defensive developer pushing back — stay calm, don't blame, but don't accept the wrong framing either. Very common at Amazon (Leadership Principles: "Have Backbone, Disagree and Commit" applies here in a technical sense).

---

### Q59. EKS upgrade quirk: Why can't you skip Kubernetes versions when upgrading EKS, and what's your upgrade strategy?
**Answer:** EKS only supports upgrading **one minor version at a time** (e.g., 1.28 → 1.29, not 1.28 → 1.30 directly) because API deprecations/removals between versions can break workloads if skipped.

**Strategy:**
1. Check deprecated API usage before upgrading: `kubectl get --raw /metrics | grep apiserver_requested_deprecated_apis` or use tools like `pluto`/`kubent`
2. Upgrade control plane first (AWS handles this, brief API server unavailability is possible but pods keep running)
3. Upgrade managed node groups (or let Karpenter handle node replacement with new AMI)
4. Upgrade EKS add-ons (VPC CNI, CoreDNS, kube-proxy) to versions compatible with new K8s version
5. Test in a non-prod cluster first with the same workloads to catch breaking changes

---

### Q60. What's the real-world difference between using `aws-auth` ConfigMap vs EKS Access Entries for cluster access?
**Answer:** `aws-auth` ConfigMap was the legacy way to map IAM roles/users to Kubernetes RBAC groups — edited manually as YAML, error-prone (a bad edit could lock everyone out of the cluster).

**EKS Access Entries** (2026 recommended) — manage IAM-to-RBAC mapping via AWS API/console/Terraform directly, not by editing a ConfigMap inside the cluster. Safer (can't accidentally corrupt cluster access), auditable via CloudTrail, and supports more granular policies (e.g., `AmazonEKSViewPolicy` for read-only access scoped to specific namespaces).

**Interviewer Notes:** If you mention `aws-auth` ConfigMap as your ONLY answer without knowing Access Entries exist, it signals slightly dated knowledge — mention both, note the migration.

---

# COVERAGE CONFIRMATION (Updated)
Sections 1-15 now cover generic K8s (Sections 1-14, 48 questions) PLUS 12 AWS/EKS-specific questions (Q49-Q60) covering: EKS architecture split, IRSA + 2026 Pod Identity, IAM/S3 access debugging, node-join failures, Karpenter vs Cluster Autoscaler, VPC CNI IP exhaustion, ALB Ingress debugging, real production-scale event handling, Fargate vs EC2, OOMKilled communication scenario, version upgrade strategy, and aws-auth vs Access Entries.

This is now a complete, industry-current (2026) prep covering generic K8s + real AWS/EKS production quirks — appropriate for IBM, Amazon, Flipkart, and Google-tier DevOps loops with 4 years experience.

---

# PRACTICE INSTRUCTIONS

1. **Day 1-2:** Architecture + Pods/Workloads (Q1-Q11) — speak each answer aloud, 60-90 sec
2. **Day 3-4:** Networking + Storage + Probes (Q12-Q19) — focus on Q14, Q17, Q19
3. **Day 5-6:** Helm + RBAC + Autoscaling (Q20-Q27)
4. **Day 7-9:** Troubleshooting Bank (Q28-Q35) — MOST asked in real interviews, drill daily, multiple times
5. **Day 10:** YAML/script writing (Q36-Q39) — write from memory, no copy-paste
6. **Day 11:** Advanced/FAANG topics (Q40-Q48) — especially Q42 (503 scenario), Q45 (cgroups), Q48 (node failure)
7. **Day 12-13:** AWS/EKS Cloud Quirks (Q49-Q60) — especially Q50/Q51 (IRSA), Q53 (Karpenter), Q56 (traffic spike), Q58 (OOMKilled communication)
8. **Day 14:** Re-read Section 14 (weightage map) — mentally rehearse the 5-stage interview flow
9. **Day 15:** Full mock interviews — random question order, Quick-fire warm-up, then 2-3 troubleshooting scenarios + 1 live YAML + 1 EKS scenario + 1 open-ended

**Self-check after each answer:** Can I say this in under 90 seconds without reading? Can I explain WHY, not just WHAT? Can I write the YAML from memory? If asked "what if that doesn't fix it" — do I have a next step ready?
