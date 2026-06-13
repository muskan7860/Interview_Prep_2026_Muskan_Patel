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

# PRACTICE INSTRUCTIONS

1. **Day 1-3:** Architecture + Pods/Workloads (Q1-Q11) — speak each answer aloud, 60-90 sec
2. **Day 4-6:** Networking + Storage + Probes (Q12-Q19) — focus on Q14, Q17, Q19 (highest scenario frequency)
3. **Day 7-9:** Helm + RBAC + Autoscaling (Q20-Q27)
4. **Day 10-12:** Troubleshooting Bank (Q28-Q35) — these are the MOST asked in real interviews, drill daily
5. **Day 13-14:** YAML writing (Q36-Q39) — practice writing from memory, no copy-paste
6. **Ongoing:** Quick-fire round daily as warm-up before mock interviews

**Self-check after each answer:** Can I say this in under 90 seconds without reading? Can I explain WHY, not just WHAT? Can I write the YAML from memory?
