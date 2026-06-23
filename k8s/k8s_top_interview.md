# Kubernetes — Top 20 Interview Questions (Architectural + Troubleshooting)
> Complete with Teacher Explanation + Interviewer Answer for each question

---

## 📋 QUICK REFERENCE CARD

### ARCHITECTURAL:
| # | Topic |
|---|-------|
| Q1 | kubectl apply full flow (14 steps, 7 components) |
| Q2 | Deployment vs ReplicaSet vs StatefulSet |
| Q3 | etcd — what it is, what happens when it fails |
| Q4 | RBAC — Role, ClusterRole, RoleBinding, ClusterRoleBinding |
| Q5 | Taints, tolerations, node affinity — differences and when to use |
| Q6 | Services — why needed, ClusterIP vs NodePort vs LoadBalancer |
| Q7 | ConfigMap vs Secret — when to use, base64 vs encrypted |
| Q8 | Ingress vs LoadBalancer Service |
| Q9 | DaemonSet vs Deployment |
| Q10 | Liveness vs Readiness vs Startup probes |

### TROUBLESHOOTING:
| # | Topic |
|---|-------|
| T1 | CrashLoopBackOff — 5 causes, exact investigation commands |
| T2 | Pending — 5 causes, describe pod, check nodes |
| T3 | Running but READY: 0/1 — readiness probe failing, not restarted |
| T4 | ImagePullBackOff — wrong tag, private registry, rate limit, network |
| T5 | Service empty endpoints — label selector mismatch primary cause |
| T6 | Node NotReady — kubelet, disk, memory, network, certificate |
| T7 | etcd slow — disk I/O, database size, shared node, network latency |
| T8 | OOMKilled — TYPE A (under-provisioned) vs TYPE B (memory leak) |
| T9 | Rollout stuck — protective behavior, new pods failing readiness |
| T10 | Pod cannot reach service — DNS, endpoints, NetworkPolicy, port |

---

# PART 1 — TOP 10 ARCHITECTURAL QUESTIONS

---

## ARCH Q1. What happens when you run `kubectl apply -f deployment.yaml`? Walk me through every step.

### 🧑‍🏫 Teacher Explanation

This question tests whether you understand the **FULL chain of events** inside Kubernetes. Most people say "the deployment gets created" and stop. That is wrong — there are **14 distinct steps involving 7 different components**.

**Analogy:** Think of it like ordering food on an app. Your order goes to the app server → checks your account (auth) → checks you have balance (RBAC) → sends it to the restaurant (etcd) → the kitchen manager reads it (controller) → assigns a chef (scheduler) → the chef actually cooks it (kubelet + containerd). Each step is separate and sequential.

### The 14 Steps in Order:

```
Step 1:  kubectl reads ~/.kube/config
         → finds API server address and YOUR certificate

Step 2:  kubectl converts YAML to JSON
         → sends HTTPS POST request to API server port 6443

Step 3:  API Server runs AUTHENTICATION
         → "who are you?" checks your certificate against cluster CA
         → if certificate is not signed by trusted CA → 401 Unauthorized → STOPS HERE

Step 4:  API Server runs AUTHORIZATION (RBAC)
         → "are you ALLOWED to create a Deployment in THIS namespace?"
         → checks Role/RoleBinding/ClusterRole for your identity
         → if no permission → 403 Forbidden → STOPS HERE

Step 5:  MUTATING ADMISSION webhooks run
         → external webhook servers can MODIFY your request
         → example: Istio injects a sidecar container you never wrote
         → example: LimitRanger adds default resource limits

Step 6:  OBJECT SCHEMA VALIDATION
         → API server checks: are all required fields present?
         → are field types correct? (replicas: "three" fails here)
         → if invalid → 422 Unprocessable Entity → STOPS HERE

Step 7:  VALIDATING ADMISSION webhooks run
         → external webhook servers can REJECT your request
         → example: OPA Gatekeeper blocks pods without resource limits
         → example: ResourceQuota rejects if namespace quota exceeded

Step 8:  Object written to ETCD
         → this is the "point of no return"
         → desired state is now stored in the cluster's database
         → Deployment object exists but NO pods yet

Step 9:  DEPLOYMENT CONTROLLER (inside controller-manager) detects new Deployment
         → computes pod-template-hash of your template
         → creates a ReplicaSet object (also written to etcd)

Step 10: REPLICASET CONTROLLER detects new ReplicaSet
         → sees replicas: 3 but actual pods = 0
         → creates 3 Pod objects in etcd (no node assigned yet)

Step 11: SCHEDULER detects 3 unscheduled pods
         → for each pod: runs filtering (remove nodes that can't host it)
         → runs scoring (rank remaining nodes)
         → writes chosen nodeName into each pod object in etcd

Step 12: KUBELET on each assigned node detects "pod assigned to me"
         → calls containerd: "pull this image"
         → containerd downloads image from registry (Docker Hub, ECR)
         → creates the container filesystem

Step 13: Containers START
         → kubelet reports pod status back to API server
         → pod phase changes: Pending → Running
         → readiness probe starts being checked

Step 14: ENDPOINT CONTROLLER detects new Ready pods
         → updates the Service's Endpoints object with new pod IPs
         → kube-proxy on every node updates iptables rules
         → traffic can now reach the new pods
```

### 🎤 Interviewer Answer

> "When I run `kubectl apply`, kubectl reads my kubeconfig to find the API server and sends an HTTPS POST request. The API server processes it through a pipeline: first **authentication** (verifying my certificate), then **RBAC authorization** (checking I have permission to create a Deployment in that namespace), then **mutating admission webhooks** which can modify the request — for example, Istio injects sidecars here — then **schema validation**, then **validating admission webhooks** which can reject the request for policy violations, and finally the object is persisted to **etcd**.
>
> Once in etcd, the **Deployment Controller** creates a ReplicaSet, the **ReplicaSet Controller** creates Pod objects, the **Scheduler** assigns nodes to those pods by writing nodeName into etcd, and **kubelet** on each assigned node pulls the image and starts the containers via containerd. Finally the **Endpoint Controller** updates the Service's endpoints so traffic reaches the new pods. From `kubectl apply` to pods running is typically under 30 seconds in a healthy cluster."

---

## ARCH Q2. Explain the difference between Deployment, ReplicaSet, and StatefulSet. When would you use each?

### 🧑‍🏫 Teacher Explanation

Think of these three like different types of workers:

- **Deployment + ReplicaSet = factory workers.** Every worker is IDENTICAL. Worker A is identical to Worker B. If Worker A quits, you hire another identical worker — nobody cares which specific person it is, as long as someone is doing the job. They have no permanent desk, no permanent tools — just a uniform and a job to do.

- **StatefulSet = senior engineers.** Each one has their OWN desk (persistent storage), their OWN employee number (stable pod name), and they were hired IN ORDER (junior first, then senior). If the engineer in seat 0 leaves and comes back, they get seat 0 back with ALL their stuff on the desk.

### Key Technical Differences:

```
DEPLOYMENT (via ReplicaSet):
- Pods: random names (payment-rs-x7k2p)
- Storage: shared or none
- Order: all created simultaneously, in parallel
- IP: changes every restart
- Use when: stateless apps (web servers, APIs, microservices)

STATEFULSET:
- Pods: ordered names (postgres-0, postgres-1, postgres-2)
- Storage: dedicated PVC per pod (volumeClaimTemplates)
- Order: strictly sequential, one at a time
- IP: changes every restart BUT DNS name is stable
- Use when: stateful apps (databases, Kafka, Zookeeper)
```

**Why not just use Deployment for everything?** Try to run a 3-replica PostgreSQL cluster as a Deployment. All 3 pods would share the same disk (data corruption), have random names (replicas can't find the primary to sync from), and would all start simultaneously (replica starts before primary exists → connection refused). StatefulSet was built specifically because Deployment CANNOT solve these three problems.

### 🎤 Interviewer Answer

> "I use **Deployment** for stateless applications — things like web servers, REST APIs, or microservices where every pod is functionally identical and interchangeable. The Deployment manages ReplicaSets, which ensures the desired number of pods are running, and handles rolling updates and rollbacks.
>
> I use **StatefulSet** when the application needs stable identity — specifically: a stable network name per replica (postgres-0, postgres-1), dedicated per-replica persistent storage that survives pod restarts via `volumeClaimTemplates`, and ordered creation so a primary database is ready before replicas try to connect to it.
>
> In our banking project at Atos, we used Deployment for all our payment and transaction microservices — they're stateless and horizontally scalable. We used StatefulSet for PostgreSQL with Patroni for automated failover, because three postgres replicas each need their own dedicated EBS volume and need to discover each other by stable DNS name."

---

## ARCH Q3. What is etcd and what happens if it goes down?

### 🧑‍🏫 Teacher Explanation

etcd is the **DATABASE of Kubernetes**. Everything Kubernetes knows — every pod, every deployment, every secret, every config — is stored in etcd as key-value pairs.

**Analogy:** Think of Kubernetes like a company. etcd is the company's filing cabinet — every HR record, every contract, every project document lives there. If the filing cabinet catches fire, the company does not immediately stop functioning (people already at their desks keep working) but NOTHING NEW can happen and nobody can look anything up.

**Critical insight:** etcd stores **DESIRED STATE**, not current state. When you create a pod, etcd stores "I want a pod called nginx running nginx:1.25." The ACTUAL running container on the node is NOT in etcd — it's on the node. Controllers READ from etcd to know what should exist and then MAKE it happen.

### What happens when etcd goes down:

```
IMMEDIATELY:
- kubectl stops working (API server can't read/write)
- No new deployments possible
- No scaling possible
- No config changes possible

PODS ALREADY RUNNING:
- Keep running (kubelet on each node maintains them independently)
- If a pod crashes: NO replacement created (controller can't read desired state)
- If a node fails: NO pods rescheduled (scheduler can't access state)

SERVICES ALREADY ROUTING:
- Keep routing (kube-proxy rules already in iptables on nodes)

RECOVERY:
- Must restore from etcd backup
- If no backup: cluster configuration is permanently lost
- Only running pods survive — all object metadata is gone
```

### Why etcd uses odd numbers (3 or 5 nodes):

etcd uses **Raft consensus** — a majority (quorum) of nodes must agree before any write is accepted.
- With 3 nodes, quorum = 2. You can lose 1 and still operate.
- With 4 nodes, quorum is STILL 3 — no extra fault tolerance vs 3 nodes but more complexity.
- With 5 nodes, quorum = 3. You can lose 2 and still operate.

Odd numbers maximize fault tolerance per node added.

### 🎤 Interviewer Answer

> "etcd is the distributed key-value store that is the **single source of truth** for all cluster state. Every object — pods, deployments, secrets, configmaps — is stored in etcd. The API server is the only component that reads from and writes to etcd directly.
>
> If etcd goes down: the API server can no longer process requests, so `kubectl` stops working and no new changes can be made. However, **pods that are already running continue to run** because kubelet maintains them locally on each node — etcd failure does not kill running workloads. But if a pod crashes, no replacement is created because the controller manager can't reach etcd to compare desired vs actual state.
>
> This is why in production banking environments, etcd runs as a 3 or 5-node cluster using Raft consensus — 3 nodes can tolerate 1 failure, 5 nodes can tolerate 2 failures. We ran automated etcd snapshot backups every 6 hours to S3 with 30-day retention. Critical for disaster recovery: if etcd is lost without a backup, the entire cluster configuration is gone permanently."

---

## ARCH Q4. Explain RBAC — Role, ClusterRole, RoleBinding, ClusterRoleBinding

### 🧑‍🏫 Teacher Explanation

RBAC is the **permission system** of Kubernetes. Think of a hospital:

- **Role** = a job description specific to ONE floor (namespace). *"Floor 3 Nurse: can read patient records on Floor 3 only."*
- **ClusterRole** = a job description for the WHOLE hospital. *"Chief Medical Officer: can read patient records on ALL floors."*
- **RoleBinding** = the HR letter that gives a SPECIFIC PERSON a job on ONE floor. *"Dr. Smith is the Floor 3 Nurse."*
- **ClusterRoleBinding** = the HR letter that gives a SPECIFIC PERSON a hospital-wide role. *"Dr. Jones is the Chief Medical Officer."*

### The Two Key Rules:
- `Role` + `RoleBinding` → permissions in **ONE namespace** only
- `ClusterRole` + `ClusterRoleBinding` → permissions **ACROSS ALL namespaces**
- `ClusterRole` + `RoleBinding` → gives the ClusterRole's permissions but **ONLY within the RoleBinding's namespace** (useful trick for sharing permission templates)

### Example YAML:

```yaml
# Step 1: Define WHAT is allowed (the Role)
kind: Role
metadata:
  name: pod-reader
  namespace: banking      # only works in THIS namespace
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
  # apiGroups "" = core API group (pods, services, etc.)
  # resources = what objects
  # verbs = what actions (get=read one, list=read many, watch=stream)

---
# Step 2: Define WHO gets it (the RoleBinding)
kind: RoleBinding
metadata:
  name: read-pods-binding
  namespace: banking
subjects:
- kind: User
  name: muskan           # the authenticated username
roleRef:
  kind: Role
  name: pod-reader       # the Role above
```

### 🎤 Interviewer Answer

> "RBAC in Kubernetes has four objects. **Role** defines what actions are allowed on what resources within a single namespace. **ClusterRole** is the same but applies cluster-wide or for non-namespaced resources like nodes. **RoleBinding** connects a Role or ClusterRole to a subject — a user, group, or service account — within a specific namespace. **ClusterRoleBinding** connects a ClusterRole to a subject across the entire cluster.
>
> In our banking project, we used namespace-scoped Roles for most things — the payment service's ServiceAccount could only read ConfigMaps and Secrets in the banking namespace, nothing else. We used ClusterRoles for our monitoring stack because Prometheus needs to read pods and endpoints across ALL namespaces to scrape metrics. The principle of least privilege is critical in banking — every service account had only the exact permissions it needed, audited quarterly."

---

## ARCH Q5. What are taints, tolerations, and node affinity? How are they different?

### 🧑‍🏫 Teacher Explanation

These are all ways to control **WHERE pods land**. But they work from **DIFFERENT directions**:

- **Taint** = a "No Entry" sign on a **NODE**. *"This node repels all pods EXCEPT those with permission."*
- **Toleration** = the "special pass" on a **POD**. *"This pod is allowed on nodes with that specific 'No Entry' sign."*
- **Node Affinity** = a preference or requirement on a **POD**. *"This pod WANTS to (or MUST) go to nodes with certain labels."*

```
TAINT + TOLERATION: Node says "stay away" → Pod says "I can handle that"
NODE AFFINITY:      Pod says "I want to go HERE specifically"
```

### Real scenario — GPU nodes:

You have 3 GPU nodes and 7 regular nodes. You want ONLY ML training pods on GPU nodes.

```bash
# Step 1: Taint the GPU nodes
kubectl taint nodes gpu-node1 gpu=true:NoSchedule
# → All regular pods CANNOT land on GPU nodes (they'd waste GPU resources)

# Step 2: ML pods get a TOLERATION for that taint
# → ML pods CAN land on GPU nodes (have the pass)

# Step 3: ML pods also get NODE AFFINITY preferring nodes with label gpu=true
# → ML pods PREFER GPU nodes (actively seek them out)

# Result:
# Regular pods → stay off GPU nodes (taint repels)
# ML pods → land ON GPU nodes (toleration + affinity attract)
```

**The critical difference:** Taint/Toleration works from the **NODE's perspective** ("I repel"). Node Affinity works from the **POD's perspective** ("I prefer/require"). Neither alone is sufficient for the GPU example — you need both.

### 🎤 Interviewer Answer

> "**Taints** are applied to nodes and act as repellents — a tainted node rejects all pods that don't have a matching toleration. **Tolerations** are applied to pods and act as permission slips — a pod with a matching toleration can be scheduled on a tainted node. **Node affinity** is different — it's applied to pods and defines which nodes the pod prefers or requires based on node labels, from the pod's perspective rather than the node's.
>
> In our banking project, we used taints and tolerations to reserve specific high-memory nodes for our reporting and analytics workloads — we tainted those nodes with `dedicated=analytics:NoSchedule` so regular payment microservices couldn't land there and consume memory needed for large report generation. The analytics pods had matching tolerations AND node affinity rules to actively prefer those nodes. Regular pods couldn't land there even if all other nodes were full."

---

## ARCH Q6. What is a Service in Kubernetes and why is it needed?

### 🧑‍🏫 Teacher Explanation

**The problem:** every pod gets an IP address, but pod IPs are **EPHEMERAL**. When a pod restarts (OOMKilled, rolling update, node failure), the NEW pod gets a completely DIFFERENT IP. If Service A needs to call Service B, it cannot hardcode Service B's pod IP because that IP will be different tomorrow.

A **Service** is a **STABLE VIRTUAL IP (ClusterIP)** that never changes, sitting in FRONT of pods. It's like a phone number for a company — the company's employees (pods) might change, but the phone number stays the same.

```
BEFORE Service:
  payment-app --calls--> 10.244.1.5 (database pod IP)
  Database pod restarts → NEW IP: 10.244.1.8
  payment-app still calls 10.244.1.5 → CONNECTION REFUSED ❌

WITH Service:
  payment-app --calls--> 10.96.0.15 (Service ClusterIP, NEVER changes)
  Service forwards to → 10.244.1.5 (current database pod IP)
  Database pod restarts → NEW IP: 10.244.1.8
  Service automatically updates → now forwards to 10.244.1.8
  payment-app still calls 10.96.0.15 → WORKS FINE ✅
```

**How Service finds its pods:** Label selectors. The Endpoint Controller continuously watches all pods and updates the Service's Endpoints object with the current IPs of all Running pods matching that label. kube-proxy then writes iptables rules on every node.

### Three Service Types:

| Type | Access | Use Case |
|------|--------|----------|
| **ClusterIP** | Inside cluster only (default) | Service-to-service communication |
| **NodePort** | Opens port 30000-32767 on every node | External access without cloud LB |
| **LoadBalancer** | Cloud provider creates external LB | Production external traffic (AWS ALB) |

### 🎤 Interviewer Answer

> "A Service solves the fundamental problem that **pod IPs are ephemeral** — every time a pod restarts, it gets a new IP. If you hardcoded pod IPs in your application config, it would break on every restart.
>
> A Service provides a **stable virtual IP (ClusterIP)** that never changes, regardless of how many times the underlying pods are replaced. The Endpoint Controller watches pods matching the Service's label selector and keeps the Endpoints object updated with current pod IPs. kube-proxy translates traffic to the ClusterIP into load-balanced traffic to actual pod IPs using iptables rules.
>
> We used ClusterIP for all internal service-to-service communication in our banking microservices — payment service calling transaction service via ClusterIP, never hardcoding pod IPs. We used LoadBalancer type for our ingress controller to get an AWS ALB external IP, and everything behind it used ClusterIP internally."

---

## ARCH Q7. What is a ConfigMap vs a Secret? When do you use each?

### 🧑‍🏫 Teacher Explanation

Both solve the same problem: keeping configuration **OUT of container images**. Instead of hardcoding `DB_HOST=prod-db.company.com` in your Dockerfile, you inject it at runtime.

The difference is **SENSITIVITY of the data**:

```
ConfigMap  = non-sensitive config
Examples: feature flags, app settings, nginx.conf, log level,
          database hostname (not credentials)

Secret     = sensitive data
Examples: database passwords, API keys, TLS certificates,
          OAuth tokens
```

### ⚠️ Critical fact about Secrets:

They are **NOT encrypted** by default. They are **base64 ENCODED**. Base64 is just encoding for safe transport — anyone who can `kubectl get secret` can decode the value in seconds with `echo "value" | base64 -d`.

The actual protection comes from:
1. **RBAC** — who can `kubectl get secrets`
2. **etcd encryption-at-rest** (a separate cluster config)
3. In production banking: use **AWS Secrets Manager via External Secrets Operator** instead

### Two ways to inject into a pod:

```
As ENVIRONMENT VARIABLES:
  → Injected ONCE at pod startup
  → If ConfigMap/Secret changes, pod does NOT see new value
  → Must restart pod to pick up changes

As VOLUME MOUNTS (files):
  → kubelet syncs changes within ~1 minute WITHOUT restarting pod
  → App must be coded to detect file changes and re-read config
  → Most apps don't do this automatically
```

### 🎤 Interviewer Answer

> "ConfigMaps store non-sensitive configuration — things like application settings, feature flags, or config files like nginx.conf. Secrets store sensitive data like passwords, API keys, and TLS certificates.
>
> The important nuance about Secrets: they are **base64 encoded, NOT encrypted** by default. Anyone with kubectl access to get secrets can decode them trivially. In our banking environment, we did not rely on native Kubernetes Secrets for database credentials. Instead, we used **AWS Secrets Manager with the External Secrets Operator** — secrets live in AWS (with KMS encryption, audit logging, and rotation) and ESO syncs them into the cluster as Kubernetes Secret objects that pods can consume normally. This way, even if someone reads the etcd snapshot, they only see references, not actual credentials."

---

## ARCH Q8. What is an Ingress and how is it different from a LoadBalancer Service?

### 🧑‍🏫 Teacher Explanation

**Analogy:** A LoadBalancer Service is like having a **SEPARATE entrance for every single shop** in a mall — one entrance for the coffee shop, one for the bookstore, one for the restaurant. Every shop needs its own entrance (its own external IP). Expensive and complex.

An **Ingress** is like having **ONE MAIN ENTRANCE** to the entire mall, with a reception desk that reads your destination and directs you to the right shop. One external IP, one main door, intelligent routing inside.

```
WITHOUT Ingress (3 separate LoadBalancer Services):
External IP 1 → payment-service       $$$
External IP 2 → transaction-service   $$$
External IP 3 → reporting-service     $$$
Total: 3 AWS Load Balancers, 3 separate IPs, 3x cost

WITH Ingress (1 LoadBalancer + 1 Ingress Controller + routing rules):
External IP → Ingress Controller (NGINX)
    /payment/*     → payment-service
    /transaction/* → transaction-service
    /reporting/*   → reporting-service
Total: 1 AWS Load Balancer, routing done inside Kubernetes ✅
```

- **Ingress** = just a set of routing RULES (like a config file)
- **Ingress Controller** = the actual software that reads those rules and does the routing (NGINX, Traefik, AWS ALB Controller)

### Example YAML:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: banking-ingress
spec:
  rules:
  - host: bank.example.com
    http:
      paths:
      - path: /payment
        pathType: Prefix
        backend:
          service:
            name: payment-service
            port:
              number: 8080
      - path: /transaction
        pathType: Prefix
        backend:
          service:
            name: transaction-svc
            port:
              number: 8080
```

### 🎤 Interviewer Answer

> "A LoadBalancer Service creates one external load balancer per Service — if you have 10 microservices, you need 10 separate load balancers and 10 external IPs. On AWS that's 10 ALBs which is expensive and operationally complex.
>
> An **Ingress resource** defines HTTP/HTTPS routing rules — route traffic based on hostname and URL path to different backend Services. An **Ingress Controller** (like NGINX or AWS ALB Ingress Controller) reads these rules and implements the actual routing. You only need ONE external load balancer for the Ingress Controller, and all routing decisions happen inside Kubernetes.
>
> In our banking project we used AWS ALB Ingress Controller. One ALB handled all inbound traffic. The Ingress resource had path-based routing — `/payment` to the payment service, `/transactions` to the transaction service, `/reports` to reporting. This reduced our ALB count from potentially 15 to 1, significantly reducing cost and simplifying TLS certificate management since we only had one termination point."

---

## ARCH Q9. What is a DaemonSet and what makes it different from a Deployment?

### 🧑‍🏫 Teacher Explanation

- **Deployment** answers: *"How MANY copies of my app should run? Put them wherever there's capacity."*
- **DaemonSet** answers: *"Run ONE copy on EVERY node. No choice, no negotiation."*

**Analogy:** A Deployment is like hiring 5 security guards and letting them be stationed at whichever 5 of your 20 buildings you choose. A DaemonSet is like having a rule: "every building must have exactly 1 security guard, always." If you open a new building, a guard appears there automatically. If a building closes, that guard's position disappears.

### Why specific components MUST be DaemonSets:

```
kube-proxy (Service routing) = MUST be DaemonSet
  If it were a Deployment with replicas: 3, only 3 nodes would have
  iptables rules written. The other nodes would be unable to route
  traffic to Services. Networking would be broken on those nodes.

Calico/Cilium (CNI plugin) = MUST be DaemonSet
  Same reason — pod networking only works on nodes where the CNI
  agent is running. Every node needs one.

Fluentd (log collector) = MUST be DaemonSet
  Log files are LOCAL to each node. If Fluentd only ran on 3 nodes,
  7 nodes' logs would never be collected. You'd have blind spots.
```

### 🎤 Interviewer Answer

> "A **Deployment** runs N copies of a pod across the cluster wherever there's capacity — you control the count via `replicas`. A **DaemonSet** runs exactly ONE copy on EVERY node — there's no replica count, the count is always equal to the number of matching nodes. When a new node joins the cluster, the DaemonSet controller automatically creates a pod on it. When a node is removed, that pod is cleaned up.
>
> DaemonSet is used for **node-level infrastructure agents**: CNI plugins like Calico that must configure networking on every node, kube-proxy that must manage iptables rules on every node, log collectors like Fluentd that read log files local to each node, and monitoring agents like node-exporter that report each node's CPU and memory. These cannot use Deployment because missing any single node would leave that node uncovered — broken networking, missing logs, or monitoring blind spots."

---

## ARCH Q10. What is the difference between liveness, readiness, and startup probes?

### 🧑‍🏫 Teacher Explanation

These three probes answer **three DIFFERENT questions** Kubernetes asks about your container:

```
STARTUP PROBE:   "Have you FINISHED starting up?"
LIVENESS PROBE:  "Are you still ALIVE and functioning?"
READINESS PROBE: "Are you READY to handle traffic right now?"
```

### The timeline of a container's life:

```
Container starts
    │
    ▼ STARTUP PROBE runs first
    │ (gives app generous time to start)
    │ If fails after failureThreshold attempts: KILL and restart
    │ If passes: startup probe stops running forever
    │
    ▼ LIVENESS PROBE starts running (continuously)
    │ "Is the app alive?" checked every periodSeconds
    │ If fails: KILL the container and restart it
    │ Use for: detecting deadlocks (app is running but frozen)
    │
    ▼ READINESS PROBE starts running (continuously)
      "Is the app ready for traffic?" checked every periodSeconds
      If fails: REMOVE from Service endpoints (no traffic sent)
      Container is NOT killed — just taken out of rotation
      If passes again: RE-ADD to Service endpoints
      Use for: warmup time, dependency unavailable (DB down)
```

### ⚠️ The crucial distinction people get wrong:

| Probe | On Failure | Use Case |
|-------|------------|----------|
| **Liveness** | **RESTARTS** the container | Deadlocks, frozen processes |
| **Readiness** | **REMOVES from traffic** (keeps running) | DB down, still warming up |
| **Startup** | **RESTARTS** the container | Slow-starting apps |

**Why have both Liveness and Readiness?** Imagine your app is stuck in a deadlock — Liveness detects it and kills it → app restarts → deadlock gone. But during a DB connection failure, you DON'T want the container killed — you want it to stop receiving traffic (Readiness fails) but continue trying to reconnect. When DB comes back, Readiness passes again and traffic resumes.

### 🎤 Interviewer Answer

> "The **startup probe** disables the liveness and readiness probes until the application has finished its initial startup. This prevents prematurely killing a slow-starting app before it's had time to initialize. Once the startup probe passes, it never runs again.
>
> The **liveness probe** runs continuously to answer 'is this container healthy?' If it fails, kubelet kills and restarts the container. We use it to detect deadlocked states where the process is alive but not functioning.
>
> The **readiness probe** answers 'should this container receive traffic?' If it fails, the Endpoint Controller removes this pod from the Service's endpoints — traffic stops being sent to it, but the container is **NOT restarted**. We use this for warm-up time after startup, and for situations where a dependency like the database is temporarily unavailable. In our banking payment service, the readiness probe checked the database connection — during a brief DB maintenance window, all pods went NotReady and traffic stopped, but the containers kept running and reconnected automatically when the DB came back, with no pod restarts needed."

---

# PART 2 — TOP 10 TROUBLESHOOTING QUESTIONS

---

## TROUBLE Q1. Pod is in CrashLoopBackOff. What does this mean and how do you fix it?

### 🧑‍🏫 Teacher Explanation

CrashLoopBackOff is **not ONE problem** — it is a **PATTERN of problems**. It means: *"the container keeps starting and immediately crashing, over and over, with increasing delays between restarts."*

**The "BackOff" part:** Kubernetes adds a delay between restarts to prevent a broken pod from consuming excessive resources. The delay sequence: `10s → 20s → 40s → 80s → 160s → 300s (max)`. That's why the RESTARTS counter goes up but slowly — Kubernetes is intentionally waiting longer between each attempt.

The root cause is ALWAYS the same: **the container process is exiting with a non-zero exit code**. The question is WHY.

### The Five Most Common Causes:

```
Cause 1: Application BUG on startup
  → App throws an exception during initialization
  → Exit code 1
  → Fix: check logs for the stack trace

Cause 2: Missing configuration
  → App tries to read DB_HOST from environment but it's not set
  → App crashes immediately: "required env var DB_HOST not found"
  → Fix: add the missing env var or Secret reference

Cause 3: OOMKilled (memory limit too low)
  → App starts, loads data into memory, hits memory.limits
  → Kernel sends SIGKILL → exit code 137
  → Fix: increase memory limit or fix memory leak

Cause 4: Wrong command / ENTRYPOINT
  → CMD in Dockerfile is wrong, binary not found, permissions issue
  → Exit code 127 (not found) or 126 (permission denied)
  → Fix: verify the command exists and is executable in the image

Cause 5: Dependency unavailable at startup
  → App tries to connect to DB at startup before DB is ready
  → Connection refused → app crashes
  → Fix: add an init container that waits for the DB to be ready
```

### Investigation Commands:

```bash
# Step 1: Check exit code and reason
kubectl describe pod <pod-name> -n <namespace>
# Look at: Last State: Terminated, Exit Code, Reason (OOMKilled?)

# Step 2: Get logs from the PREVIOUS crash (not current restart)
kubectl logs <pod-name> --previous
# KEY command — shows what happened before the restart
# Current logs may be empty if the new container just started

# Step 3: If still unclear, get into the container
kubectl exec -it <pod-name> -- bash
# Check if the binary exists, check env vars, check file permissions
```

### 🎤 Interviewer Answer

> "CrashLoopBackOff means the container is repeatedly starting and immediately exiting with an error, with Kubernetes adding an increasing delay between restarts — starting at 10 seconds and backing off up to 5 minutes. My investigation always starts with two commands: `kubectl describe pod` to see the exit code and Last State, and `kubectl logs --previous` to see the actual error from the CRASHED container (not the current restart which might show nothing yet).
>
> The most common causes I've encountered: application exception on startup from missing config (check the log for stack traces), OOMKilled where the exit code is 137 and describe shows `Reason: OOMKilled` (increase memory limit), wrong command in the image (exit code 127 means binary not found), or a startup ordering issue where the app tries to connect to a database that's not ready yet (fix with an init container). In our banking project, we had a payment service in CrashLoopBackOff because it was trying to load a 400MB machine learning model into memory at startup but the memory limit was 256Mi — we saw exit code 137 and OOMKilled in describe, increased the limit to 1Gi, and it stabilized."

---

## TROUBLE Q2. Pod is stuck in Pending. What are the causes and how do you debug?

### 🧑‍🏫 Teacher Explanation

**Pending** means the pod OBJECT exists in etcd (it was accepted by the API server) but **NO node has been assigned to it yet**. The Scheduler has not been able (or willing) to place it.

**Analogy:** You checked into a hotel (pod was accepted), but the hotel is full and they have not assigned you a room yet. You're sitting in the lobby (Pending) with your luggage.

The scheduler runs **FILTERING** (which nodes CAN host this pod?) then **SCORING** (which of those is BEST?). If ZERO nodes pass filtering, the pod stays Pending forever.

### The Five Most Common Causes:

```
Cause 1: Insufficient resources (most common)
  → No node has enough free CPU or memory for this pod's REQUESTS
  → Event: "0/3 nodes available: 3 Insufficient cpu"
  → Fix: reduce requests, add more nodes, or check for over-provisioned requests

Cause 2: Node taint blocking placement
  → All nodes have a taint and pod has no matching toleration
  → Event: "0/3 nodes have taint {dedicated=banking} that pod didn't tolerate"
  → Fix: add toleration to pod, or remove the taint from some nodes

Cause 3: Node selector / affinity too strict
  → Pod requires a node label that no node has
  → Event: "0/3 nodes didn't match node selector"
  → Fix: label a node with the required label, or relax the selector

Cause 4: PVC not bound
  → Pod requests a PersistentVolumeClaim that doesn't exist or is not bound
  → Event: "pod has unbound PersistentVolumeClaims"
  → Fix: create the PVC, check StorageClass, check PV availability

Cause 5: ResourceQuota exceeded
  → Namespace has a CPU/memory quota and creating this pod would exceed it
  → Event: "exceeded quota"
  → Fix: increase quota, delete unused pods, or reduce pod requests
```

### Investigation Commands:

```bash
# The single most important command
kubectl describe pod <pending-pod-name> -n <namespace>
# Scroll to bottom, read Events section
# The event message tells you EXACTLY why it can't be scheduled

# Check node capacity
kubectl describe nodes | grep -A 8 "Allocated resources"
# See how much is already reserved vs total capacity

# Check namespace quota
kubectl describe resourcequota -n <namespace>
```

### 🎤 Interviewer Answer

> "Pending means the pod exists in etcd but the Scheduler hasn't been able to assign it to any node. My first command is always `kubectl describe pod` and I read the **Events section** at the bottom — it tells me exactly why scheduling failed.
>
> The most common cause I see is **insufficient resources**: the pod requests more CPU or memory than any node has free in its allocatable capacity. Note that this is based on resource **REQUESTS** (reservations), not actual usage — a node can appear full to the scheduler while actually being mostly idle if pods have over-provisioned requests. Other causes: taint on all nodes with no matching toleration in the pod, too-strict node affinity or node selector pointing to a label no node has, or a PVC that hasn't been bound to a PV yet. In our banking cluster, we had a reporting pod stuck Pending for 30 minutes because it requested 8Gi memory for processing large transaction datasets but our largest node only had 6Gi allocatable after the operating system and DaemonSet overhead."

---

## TROUBLE Q3. Pod is Running but READY: 0/1. What's happening?

### 🧑‍🏫 Teacher Explanation

This is the most **MISUNDERSTOOD pod state**. People see "Running" and think it's fine. But Running and Ready are **TWO DIFFERENT THINGS**.

```
STATUS: Running  = the container PROCESS has started (kubelet started it)
READY: 0/1       = the readiness PROBE is FAILING (app not accepting traffic)
```

**Analogy:** A restaurant that's open (Running) but has a "Temporarily Closed" sign on the door (not Ready). The building is there, staff is inside, but no customers are being served.

**Consequence:** The Endpoint Controller has **REMOVED this pod's IP** from the Service endpoints. Traffic that goes to the Service WILL NOT reach this pod.

### Common Causes:

```
Cause 1: App still initializing (slow startup)
  → App takes 60 seconds to load, readiness probe starts checking at 5 seconds
  → Solution: increase initialDelaySeconds or add a startupProbe

Cause 2: Database connection unavailable
  → Readiness probe checks /health/ready which checks DB connection
  → DB is down or unreachable → probe returns 503 → READY: 0/1
  → This is CORRECT behavior — pod should not serve traffic without DB
  → Solution: fix the DB connectivity issue, pod will recover automatically

Cause 3: Wrong port in probe config
  → App listens on 8080, probe is configured for port 8081
  → Probe gets "connection refused" → treated as failure
  → Solution: fix the port in the readiness probe config

Cause 4: Wrong health check path
  → App's health endpoint is /api/health but probe checks /health
  → Probe gets 404 → treated as failure
  → Solution: fix the path in the readiness probe config
```

### Investigation Commands:

```bash
# See the Running/0/1 state
kubectl get pods -n <namespace>

# Describe shows probe failure details
kubectl describe pod <pod-name> -n <namespace>
# Events: "Readiness probe failed: HTTP probe failed with statuscode: 503"

# Test the probe endpoint manually from inside the pod
kubectl exec -it <pod-name> -- curl localhost:8080/health/ready

# Check app logs — what is it waiting for?
kubectl logs <pod-name> -n <namespace>

# Verify the pod is actually excluded from service endpoints
kubectl get endpoints <service-name> -n <namespace>
# If pod is not Ready, its IP is not in this list
```

### 🎤 Interviewer Answer

> "Running but 0/1 Ready means the container process is running but the **readiness probe is failing** — the pod has been removed from the Service's endpoints so it receives zero traffic. This is protective behavior.
>
> I debug this by first checking `kubectl describe pod` and reading the Events section for the exact readiness probe error message. Then I use `kubectl exec` to run the health check command directly from inside the container — `curl localhost:8080/health` — to see what the app actually returns. Common causes are slow application startup where the probe fires before the app is ready (fix: increase `initialDelaySeconds`), a downstream dependency like a database that's temporarily unavailable (fix: restore the dependency and the pod recovers automatically), or a misconfigured probe pointing to the wrong port or path. The key point: unlike a liveness probe failure which kills and restarts the container, **a readiness probe failure just removes traffic** — the container keeps running, which is correct behavior when the issue is a recoverable external dependency."

---

## TROUBLE Q4. ImagePullBackOff — what does it mean and how do you fix it?

### 🧑‍🏫 Teacher Explanation

**ImagePullBackOff** means: *"I tried to pull the container image from the registry and it FAILED. I will wait and try again with exponential backoff."*

### The Four Causes and How to Tell Them Apart:

```
Cause 1: WRONG IMAGE NAME or TAG (most common)
  Error message: "manifest unknown" or "not found"
  What happened: The image name or tag literally doesn't exist in the registry
  Example: image: nginx:1.99 → this version doesn't exist
  Fix: correct the image name/tag in the deployment YAML

Cause 2: PRIVATE REGISTRY without credentials
  Error message: "unauthorized" or "authentication required"
  What happened: The image is in a private registry (ECR, private Docker Hub)
                 and the pod has no imagePullSecret to authenticate
  Fix: create a Secret with registry credentials and add imagePullSecrets
       to the pod spec

Cause 3: DOCKER HUB RATE LIMIT (very common in CI/CD)
  Error message: "429 Too Many Requests" or "toomanyrequests"
  What happened: Docker Hub limits anonymous pulls to 100/6hours per IP
                 On a shared CI/CD environment, this limit is hit quickly
  Fix: authenticate with Docker Hub to get higher limits,
       OR mirror the image to your own ECR/Artifactory

Cause 4: REGISTRY UNREACHABLE
  Error message: "dial tcp: connection refused" or timeout
  What happened: The node cannot reach the registry
                 (firewall rule, no internet access, DNS failure)
  Fix: check network connectivity from the node, check security groups
       check if registry requires VPC endpoint (AWS ECR in private subnet)
```

### Investigation Commands:

```bash
# See ImagePullBackOff status
kubectl get pods -n <namespace>

# Describe shows the exact error message
kubectl describe pod <pod-name> -n <namespace>
# Events:
# Failed to pull image "nginx:1.99": rpc error: manifest unknown
# OR: Failed to pull image: unauthorized: authentication required
# OR: Failed to pull image: toomanyrequests

# Check what image the pod is trying to pull
kubectl get pod <pod-name> -o yaml | grep image
```

### 🎤 Interviewer Answer

> "ImagePullBackOff means the container runtime (containerd) attempted to pull the image from the registry and failed, and is now backing off with increasing delays between retries. The specific error is in `kubectl describe pod` under Events.
>
> I categorize the fix based on the error message: **'manifest unknown'** means the image or tag doesn't exist — typo in the YAML or the image was never pushed. **'unauthorized'** means it's a private registry and there's no imagePullSecret configured — I create a docker-registry Secret and add it to the pod's `imagePullSecrets`. **'toomanyrequests'** is Docker Hub rate limiting — common in shared environments, fixed by authenticating with Docker Hub or mirroring to ECR. **Network errors** mean the node can't reach the registry — check security groups and VPC connectivity.
>
> In our banking environment, all images went through our internal Artifactory registry — no direct Docker Hub pulls in production. This eliminated rate limiting, gave us image scanning before deployment, and kept all images air-gapped from the internet."

---

## TROUBLE Q5. Service has no endpoints — pods are Running but traffic doesn't reach them.

### 🧑‍🏫 Teacher Explanation

This is the classic **"everything looks fine but nothing works"** scenario. Pods are Running, Service exists, but requests time out.

The root cause is **ALMOST ALWAYS a label selector mismatch** — the Service is looking for pods with label X but the pods actually have label Y.

```
Service YAML has:
  spec:
    selector:
      app: payment        ← "find pods with this label"

Pod YAML has:
  metadata:
    labels:
      app: payment-service  ← DIFFERENT from what Service wants!

The Endpoint Controller checks: does pod label CONTAIN every key-value in selector?
payment-service ≠ payment → pod NOT added to endpoints

Result:
kubectl get endpoints payment-svc
NAME           ENDPOINTS    AGE
payment-svc    <none>       5m   ← empty! no pods match ❌
```

### Other Causes:

```
Cause 2: Pod has correct label but readiness probe FAILING
  → Pod is Running but not Ready → excluded from endpoints
  → Fix: fix the readiness probe issue

Cause 3: Service targetPort doesn't match container's port
  → Service sends traffic to port 8081 but container listens on 8080
  → Connections reach the pod but get "connection refused"
  → Fix: align targetPort with the container's actual listening port

Cause 4: Network Policy blocking traffic
  → NetworkPolicy exists that denies traffic from the client namespace
  → Fix: add an allow rule to the NetworkPolicy
```

### Investigation Commands:

```bash
# Step 1: Check endpoints — if empty, selector mismatch
kubectl get endpoints <service-name> -n <namespace>
# If ENDPOINTS shows <none> → label selector problem

# Step 2: Compare Service selector with Pod labels
kubectl describe service <service-name> -n <namespace>
# Look at: Selector: app=payment

kubectl get pods --show-labels -n <namespace>
# Look at pod labels — do they match EXACTLY?

# Step 3: If endpoints exist, test from inside cluster
kubectl run debug --image=busybox -it --rm -- sh
# Inside:
wget -O- <service-name>.<namespace>:8080
# If fails: check targetPort alignment and NetworkPolicy
```

### 🎤 Interviewer Answer

> "Empty service endpoints almost always means a **label selector mismatch**. The Endpoint Controller populates the Endpoints object with pod IPs only when the pod's labels are a SUPERSET of the Service selector AND the pod is Ready. My first commands are `kubectl get endpoints <service>` to confirm it's empty, then `kubectl describe service` to see the selector, then `kubectl get pods --show-labels` to compare — even a capitalization difference (`payment` vs `Payment`) breaks the match.
>
> If endpoints are populated but traffic still fails, the issue is usually the **targetPort** — the Service might be pointing to port 8081 but the container actually listens on 8080, so connections reach the pod but get refused. Or a **NetworkPolicy** is blocking traffic between namespaces. I test network connectivity by running a debug pod with busybox and trying to reach the service DNS name directly from inside the cluster — this isolates whether it's a routing issue or an application issue."

---

## TROUBLE Q6. Node shows NotReady. What happened and what do you check?

### 🧑‍🏫 Teacher Explanation

A node goes **NotReady** when the API server stops receiving **heartbeat signals** from the kubelet on that node. kubelet sends heartbeats every few seconds by updating a Lease object in the `kube-node-lease` namespace. If heartbeats stop, the Node Controller marks the node NotReady after **40 seconds**.

### What happens automatically:

```
After 40 seconds: node condition changes to Unknown/NotReady
After 5 minutes (default): Node Controller evicts pods from the node
  → Pods managed by Deployments/StatefulSets are rescheduled on healthy nodes
  → DaemonSet pods are NOT rescheduled (they're supposed to be on every node)
```

### Common Causes:

```
Cause 1: kubelet crashed or stopped
  → kubelet is a systemd service, it might have crashed
  → Check: ssh to node, systemctl status kubelet
  → Fix: systemctl restart kubelet

Cause 2: Node has DISK PRESSURE (disk is full)
  → When disk is full, kubelet cannot write to its working directories
  → kubelet stops sending heartbeats, node goes NotReady
  → Check: ssh to node, df -h (check disk usage)
  → Fix: clear disk space, add more disk, adjust log rotation

Cause 3: Node has MEMORY PRESSURE (OOM at node level)
  → Kernel OOM killer killed the kubelet process
  → Check: ssh to node, dmesg | grep -i oom
  → Fix: add more memory, reduce pod resource limits

Cause 4: Network partition
  → Node is running fine but cannot communicate with API server
  → Pods on that node keep running (kubelet is fine locally)
  → API server just can't hear from it
  → Check: can you ping the node? Can node reach API server IP?

Cause 5: Certificate expired
  → kubelet's client certificate expired (happens after 1 year)
  → kubelet cannot authenticate to API server, stops working
  → Check: openssl x509 -in /var/lib/kubelet/pki/kubelet-client-current.pem -noout -dates
  → Fix: renew kubelet certificates
```

### Investigation Commands:

```bash
# From kubectl (if you still have API access)
kubectl get nodes
kubectl describe node <node-name>
# Look at: Conditions section (DiskPressure, MemoryPressure, PIDPressure, Ready)
# Look at: Events section

# SSH to the affected node
ssh <node-ip>

# Check kubelet service
systemctl status kubelet
journalctl -u kubelet -f --since "10 minutes ago"

# Check disk
df -h
# If /var/lib/kubelet is full → cause found

# Check memory
free -m
dmesg | grep -i oom
```

### 🎤 Interviewer Answer

> "NotReady means the API server hasn't received heartbeats from that node's kubelet within the threshold period — 40 seconds by default. My investigation splits into two paths: what I can check from kubectl if the API server is still accessible, and what I check by SSHing directly to the node.
>
> From kubectl, `kubectl describe node` shows the **Conditions section** — DiskPressure, MemoryPressure, PIDPressure tell me if a resource constraint caused the issue. On the node itself, I check `systemctl status kubelet` and `journalctl -u kubelet` for error messages, then `df -h` for disk pressure, and `dmesg | grep oom` for memory issues.
>
> In our banking environment, we had a node go NotReady because the application logs were filling `/var/log` faster than log rotation could clear them. The disk hit 100%, kubelet couldn't write to its state directories and stopped sending heartbeats. Short term fix: deleted old log files. Long term fix: implemented Fluentd DaemonSet to ship logs to Elasticsearch in real time and set aggressive log rotation policies so no log file could grow beyond 100MB."

---

## TROUBLE Q7. etcd is slow. The whole cluster feels sluggish. How do you investigate?

### 🧑‍🏫 Teacher Explanation

etcd is the most **performance-sensitive component** in Kubernetes. Every single kubectl command, every controller reconciliation loop, every scheduler decision — all involve reading from or writing to etcd. **If etcd is slow, EVERYTHING is slow.**

etcd's performance is almost entirely determined by ONE thing: **disk I/O latency**. etcd writes to disk on every transaction (write-ahead log). If the disk is slow (HDDs instead of SSDs, high I/O contention), etcd slows down.

### Signs of etcd slowness:
- `kubectl` commands take 5-30+ seconds to return
- `"etcdserver: request timed out"` errors in API server logs
- `"slow apply"` warnings in etcd logs
- etcd leader elections happening frequently

### The Four Common Causes:

```
Cause 1: SLOW DISK (most common)
  → etcd needs SSD with low latency (< 10ms disk write latency)
  → If running on HDD or a network disk with high latency: slow etcd
  → Check: iostat -x on the etcd node, look at await column
  → Fix: migrate etcd to SSD-backed storage

Cause 2: TOO MANY OBJECTS (database has grown too large)
  → Over time: old ReplicaSets, completed Jobs, old Events, orphaned ConfigMaps
  → etcd database grows, reads/writes get slower
  → Check: etcdctl endpoint status shows database size
  → Fix: cleanup old objects, run etcd compaction and defragmentation

Cause 3: etcd running on SHARED NODE with resource contention
  → etcd is competing with other processes for CPU and disk I/O
  → Fix: dedicate separate nodes to etcd, keep them isolated

Cause 4: NETWORK LATENCY between etcd nodes
  → In a 3-node etcd cluster, every write must be confirmed by majority
  → If network between etcd nodes is slow, each write takes longer
  → Check: ping latency between etcd nodes
  → Fix: ensure etcd nodes are in same datacenter/AZ, not cross-region
```

### Investigation Commands:

```bash
# Check disk I/O on etcd node (ssh to the etcd node)
iostat -x 2 5
# Look at: await column (disk write latency in ms)
# Good: < 5ms | Concerning: > 10ms | Critical: > 50ms

# Check etcd logs for slowness warnings
kubectl logs -n kube-system etcd-controlplane | grep -i "slow"
kubectl logs -n kube-system etcd-controlplane | grep -i "took too long"

# Check etcd database size
kubectl exec -n kube-system etcd-controlplane -- sh -c \
  "ETCDCTL_API=3 etcdctl endpoint status \
   --cacert=/etc/kubernetes/pki/etcd/ca.crt \
   --cert=/etc/kubernetes/pki/etcd/server.crt \
   --key=/etc/kubernetes/pki/etcd/server.key \
   --write-out=table"
# Look at: DB SIZE column
```

### 🎤 Interviewer Answer

> "etcd slowness manifests as kubectl commands taking many seconds to return and sometimes timing out. Since every API operation touches etcd, it creates a ripple effect making the entire cluster feel sluggish.
>
> My first check is **disk I/O** on the etcd node — `iostat -x` on the node. etcd is extremely sensitive to disk write latency because it uses a write-ahead log that must fsync to disk on every transaction. If `await` is over 10ms, that's likely the root cause and the fix is moving etcd to SSD or NVMe storage. My second check is **etcd database size** via `etcdctl endpoint status` — an oversized database from accumulated old objects (completed Jobs, orphaned ReplicaSets, expired Events) slows down compaction and read operations.
>
> In our banking cluster, we had etcd slowness after 8 months of running. The database had grown to 6GB because we hadn't been cleaning up completed migration Jobs and old Deployment ReplicaSets. We cleaned up old objects with `kubectl delete jobs --field-selector status.successful=1`, reduced the `revisionHistoryLimit` on all Deployments from 10 to 3, and ran `etcd defrag`. Database size dropped from 6GB to 800MB and API response times went back to normal within minutes."

---

## TROUBLE Q8. OOMKilled — pod keeps getting killed. How do you handle it?

### 🧑‍🏫 Teacher Explanation

**OOMKilled** (Out Of Memory Killed) means the **Linux kernel's OOM killer** sent SIGKILL to the container process because it tried to use MORE memory than its `limits.memory` allows. This is NOT Kubernetes killing the pod — it's the Linux kernel enforcing the cgroup memory limit.

```
Container tries to allocate memory beyond limits.memory
                │
                ▼
Linux kernel's OOM killer activates
                │
                ▼
Sends SIGKILL to the largest offending process in that cgroup
                │
                ▼
Container dies: Exit Code 137 (128 + 9, where 9 = SIGKILL signal)
                │
                ▼
kubelet reports: Last State: Terminated, Reason: OOMKilled
                │
                ▼
restartPolicy: Always → new container starts → cycle continues
```

### Two Different Types — This Distinction Is Critical:

```
TYPE A: Memory limit genuinely too low
  → App actually needs more memory than configured
  → Memory usage is STABLE but hits the limit
  → Fix: increase limits.memory to match actual needs

TYPE B: Memory LEAK
  → App's memory usage GROWS CONTINUOUSLY over hours/days
  → Hits any limit you set, eventually
  → Fix: find and fix the memory leak in application code
  → Identifying sign: OOMKill happens every few hours like clockwork,
    increasing the limit just delays the next kill
```

### How to tell TYPE A from TYPE B:

```
TYPE A: memory usage is flat or spiky but BOUNDED
  Under light load: 150Mi | Under heavy load: 230Mi
  Limit is 256Mi → occasionally hits it under peak load
  Fix: increase limit to 512Mi and it stabilizes ✅

TYPE B: memory usage is a GROWING LINE
  After 1 hour: 150Mi | After 2 hours: 200Mi | After 4 hours: 300Mi
  (OOMKilled at 256Mi limit)
  Restart → reset to 150Mi → starts growing again
  Pattern: increasing limit to 512Mi just takes twice as long to crash ❌
```

### Investigation Commands:

```bash
# Confirm it's OOMKilled
kubectl describe pod <pod-name> -n <namespace>
# Last State: Terminated
# Reason: OOMKilled
# Exit Code: 137

# Check current memory usage
kubectl top pod <pod-name> -n <namespace>

# Check the configured limit
kubectl get pod <pod-name> -o yaml | grep -A 3 limits
```

### 🎤 Interviewer Answer

> "OOMKilled means the container exceeded its `limits.memory` and the Linux kernel's OOM killer sent SIGKILL — **exit code 137**. I confirm it with `kubectl describe pod` which shows `Reason: OOMKilled` in Last State.
>
> The critical investigation step is **distinguishing a genuine under-provisioning issue from a memory leak**. If memory usage is stable and just slightly above the current limit, I increase `limits.memory` and it stabilizes — that's under-provisioning. If memory grows continuously regardless of traffic and hits whatever limit I set, it's a leak.
>
> In our banking payment service we had OOMKills every 6 hours like clockwork with a 256Mi limit. Increasing to 512Mi made it crash every 12 hours — the pattern made clear it was a leak, not under-provisioning. We profiled the JVM heap and found a cache that stored transaction IDs indefinitely without any eviction policy. Fixing the cache to evict entries older than 1 hour reduced memory usage from a growing line to a stable 80Mi under load."

---

## TROUBLE Q9. Deployment rollout is stuck. Some pods have new version, some old. What do you do?

### 🧑‍🏫 Teacher Explanation

A rolling update is **INTENTIONALLY designed to PAUSE** if new pods aren't becoming Ready. This is **PROTECTION, not a bug**. Kubernetes is saying: *"The new version is failing to become healthy. I'm not going to take down any more old working pods until I understand what's happening."*

### The Mechanism:

```
You have: maxSurge: 1, maxUnavailable: 0, replicas: 3

Rolling update starts:
  old-pod-1 running ✓
  old-pod-2 running ✓
  old-pod-3 running ✓
  new-pod-1 starting...
    → readiness probe FAILS (new version is broken)
    → Deployment controller: "I cannot proceed"
    → Cannot kill old-pod-1 yet (maxUnavailable: 0 = must keep 3 running)
    → Cannot create new-pod-2 yet (new-pod-1 is failing, not Ready)
  STUCK ⛔

Why? maxUnavailable: 0 means "never drop below desired replica count."
If the new pod NEVER becomes Ready, the rollout NEVER progresses.
```

### The Causes:

```
Cause 1: New image has a bug causing crash on startup
  → New pods are in CrashLoopBackOff
  → Old pods keep running safely
  → Fix: rollback immediately

Cause 2: Readiness probe failing on new pods
  → New pods are Running but probe fails
  → Might be: wrong health check path/port in new version,
    new DB migration not run yet, new env var not configured
  → Fix: investigate the readiness probe failure, then rollback or fix

Cause 3: ImagePullBackOff on new pods
  → Image doesn't exist or credentials wrong
  → Fix: fix the image reference, or rollback

Cause 4: Insufficient resources
  → No room for the surge pod on any node
  → New pod stays Pending, rollout stuck
  → Fix: add nodes or reduce resource requests
```

### Investigation Commands:

```bash
# Check rollout status
kubectl rollout status deployment/<name> -n <namespace>
# "Waiting for deployment to finish: 1 out of 3 new replicas updated..."

# Find which pods are the NEW ones (different age)
kubectl get pods -l app=<name> -n <namespace>

# Check why new pods are failing
kubectl describe pod <new-pod-name> -n <namespace>
kubectl logs <new-pod-name> -n <namespace>
kubectl logs <new-pod-name> --previous -n <namespace>

# ROLLBACK if new version is broken
kubectl rollout undo deployment/<name> -n <namespace>
kubectl rollout status deployment/<name> -n <namespace>
# Old version restored, all pods healthy again ✅
```

### 🎤 Interviewer Answer

> "A stuck rollout is actually **protective behavior** — Kubernetes detected the new version isn't becoming healthy and stopped rather than replacing all old pods with a broken new version. My first command is `kubectl get pods` to see which pods are new (shorter age) and what state they're in, then `kubectl describe` and `kubectl logs --previous` on the failing new pod to find the root cause.
>
> Common causes: the new image has a startup bug (CrashLoopBackOff), the readiness probe is misconfigured in the new version pointing to a wrong path or port, the new version depends on a database migration that hasn't been run yet, or the image tag doesn't exist and it's ImagePullBackOff.
>
> Once I understand whether it's fixable quickly or requires investigation, I make a decision: if production is impacted and the fix isn't immediate, I **rollback first** with `kubectl rollout undo` to restore service, then investigate and fix in staging. The key principle from our banking environment: always rollback first to restore service, then investigate — never leave a degraded service running while you debug."

---

## TROUBLE Q10. How do you debug a pod that can't communicate with another service?

### 🧑‍🏫 Teacher Explanation

Network connectivity issues between pods are among the most complex to debug because there are **MANY layers where things can break**:

```
payment-pod tries to reach transaction-service

Layer 1: DNS resolution
  Does "transaction-service.banking.svc.cluster.local" resolve?
  If not: CoreDNS issue

Layer 2: Service endpoints
  Does transaction-service have any pod IPs in its endpoints?
  If not: label selector mismatch OR all pods are not Ready

Layer 3: kube-proxy rules
  Are iptables rules correctly routing ClusterIP → pod IPs?
  If not: kube-proxy issue

Layer 4: Network Policy
  Is there a NetworkPolicy blocking traffic between namespaces?
  If yes: traffic is blocked at the CNI level

Layer 5: Application
  Can the app reach the pod's port?
  Are there TLS/authentication issues at the app level?
```

### Systematic Debugging Approach:

```bash
# Step 1: From INSIDE the calling pod, test DNS resolution
kubectl exec -it payment-pod -n banking -- sh
nslookup transaction-service.banking.svc.cluster.local
# If FAILS: CoreDNS problem
# If SUCCEEDS: DNS works, problem is elsewhere

# Step 2: Check Service endpoints (from kubectl, not inside pod)
kubectl get endpoints transaction-service -n banking
# If EMPTY: label selector mismatch or no Ready pods
# If HAS IPs: endpoints are fine, problem is further

# Step 3: From INSIDE the calling pod, test actual connectivity
kubectl exec -it payment-pod -n banking -- sh
curl -v http://transaction-service.banking:8080/health
# Connection refused: port wrong or app not listening
# Timeout: NetworkPolicy blocking or kube-proxy issue

# Step 4: Check NetworkPolicy
kubectl get networkpolicies -n banking
kubectl describe networkpolicy <name> -n banking
# Check if ingress rules block traffic from payment namespace

# Step 5: Check CoreDNS if DNS issues
kubectl get pods -n kube-system | grep coredns
kubectl logs <coredns-pod> -n kube-system
```

### 🎤 Interviewer Answer

> "Network connectivity debugging requires **systematic layer-by-layer isolation**. I always start by exec-ing into the calling pod and testing from there — this eliminates any network setup issues outside Kubernetes.
>
> **Step 1 is DNS:** `nslookup transaction-service.banking.svc.cluster.local` from inside the pod. If DNS fails, it's a CoreDNS issue — I check CoreDNS pods in kube-system. If DNS resolves but connection fails, **Step 2** is checking Service endpoints: `kubectl get endpoints transaction-service` — if empty, there's a label selector mismatch and no pods are backing the service. **Step 3** is testing direct connectivity: `curl http://transaction-service:8080/health` from inside the pod — if connection refused, the port might be wrong; if timeout, there may be a NetworkPolicy blocking traffic.
>
> **NetworkPolicy is often overlooked.** In our banking cluster, namespaces had default-deny ingress policies — the payment namespace could only receive traffic from explicitly allowed sources. When we deployed a new notification service in a new namespace, it couldn't reach the transaction service until we added an explicit allow rule in the NetworkPolicy. The debugging signal: `curl` times out (not refused, **timeout**) → immediately suspect NetworkPolicy → `kubectl get networkpolicies -A` → find the blocking rule."

---

*End of Document — Kubernetes Top 20 Interview Q&A*
*Repository: Interview_Preparation_2026*
