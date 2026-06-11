# Kubernetes Deep Dive — Control Plane, Worker Node, Industry Setup, Certificates

> Level: Zero to 4 Years Experience  
> Author: Muskan Patel  
> Domain: Banking — Atos

---

## Your Question 1 — Why MicroK8s Does Not Show All Components

### The Honest Answer

There are two ways Kubernetes runs its control plane components:

**Way 1 — As Static Pods (kubeadm clusters, KillerCoda)**
- Control plane components run as pods inside kube-system namespace
- You can see them with `kubectl get pods -n kube-system`
- This is what you saw on KillerCoda — etcd, kube-apiserver, kube-scheduler, kube-controller-manager all showing as pods

**Way 2 — As System Processes (MicroK8s)**
- MicroK8s runs control plane components directly as Linux snap services
- They do NOT run as pods — they run as background processes
- That is why you only see calico, coredns, hostpath-provisioner in your MicroK8s kube-system
- The control plane is there — just not visible as pods

### Prove It — Run This on Your MicroK8s

```bash
# See the actual control plane processes running
ps aux | grep kube-apiserver
ps aux | grep kube-scheduler
ps aux | grep kube-controller
ps aux | grep etcd

# Or check snap services
snap services microk8s

# Check microk8s status
microk8s status
```

You will see all control plane components listed as running snap services.

---

## Your Question 2 — How Do You Know kube-system Is the Namespace

### Answer

`kube-system` is a built-in namespace that Kubernetes creates automatically when a cluster is installed. It is hardcoded in Kubernetes — not something an admin creates.

### All Built-in Namespaces Explained

```bash
kubectl get ns
```

| Namespace | Purpose |
|---|---|
| kube-system | All Kubernetes internal components — control plane, DNS, networking |
| kube-public | Publicly readable data — cluster info accessible without authentication |
| kube-node-lease | Node heartbeat objects — kubelet updates these to report node health |
| default | Where your applications go if you do not specify a namespace |

### Why Is This Important in Interviews

Interviewer will ask — *"If CoreDNS is failing, where do you look?"*

Answer: `kubectl get pods -n kube-system | grep coredns` — because all cluster components live in kube-system.

---

## Your Question 3 — How Industry Really Sets Up Kubernetes

### Small Company Setup (Startup)

```
1 Control Plane Node    (2 CPU, 4GB RAM)
2-3 Worker Nodes        (4 CPU, 8GB RAM each)
Total cost on AWS:      ~$300-500/month
```

### Medium Company Setup (Your Atos Banking Level)

```
3 Control Plane Nodes   (HA — if one dies, two remain)
5-10 Worker Nodes       (split by team or workload type)
Separate etcd cluster   (3 nodes dedicated to etcd only)
```

### Large Enterprise Setup (Banks, Netflix, Google)

```
3-5 Control Plane Nodes (across 3 availability zones)
50-500+ Worker Nodes    (auto-scaled based on load)
Multiple clusters       (dev, staging, production — separate clusters)
Dedicated etcd cluster  (5 nodes, SSD only, dedicated hardware)
```

### On AWS — EKS Real Industry Setup

```
┌─────────────────────────────────────────────────────┐
│                    AWS Account                       │
│                                                      │
│  ┌─────────────────────────────────────────────┐    │
│  │              VPC (your network)              │    │
│  │                                              │    │
│  │  ┌──────────────┐  ┌──────────────────────┐ │    │
│  │  │  EKS Control │  │    Worker Nodes       │ │    │
│  │  │  Plane       │  │                       │ │    │
│  │  │  (AWS managed│  │  AZ-1a   AZ-1b  AZ-1c│ │    │
│  │  │  you never   │  │  Node1   Node3  Node5 │ │    │
│  │  │  touch this) │  │  Node2   Node4  Node6 │ │    │
│  │  └──────────────┘  └──────────────────────┘ │    │
│  │                                              │    │
│  └─────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────┘
```

### Key Points for Interview

1. On EKS — AWS manages control plane completely
2. You only manage worker nodes (called Node Groups in EKS)
3. Control plane runs across 3 availability zones automatically
4. You are charged $0.10/hour just for the EKS control plane
5. Worker nodes are regular EC2 instances you pay for separately
6. You never SSH into control plane on EKS — AWS does not allow it

### How Multiple Applications Are Set Up

```bash
# Each team or application gets its own namespace
kubectl create namespace payment-service
kubectl create namespace transaction-service
kubectl create namespace reporting-service
kubectl create namespace monitoring

# Each namespace has its own:
# - Deployments
# - Services
# - ConfigMaps
# - Secrets
# - ResourceQuotas (limits how much CPU/memory a team can use)
```

### Real Banking Setup at Atos Level

```
Cluster: banking-production-eks

Namespaces:
├── payment-service      (payment processing microservices)
├── transaction-service  (transaction history, statements)
├── auth-service         (user authentication, JWT)
├── reporting-service    (reports, analytics)
├── monitoring           (Prometheus, Grafana, AlertManager)
├── logging              (Fluentd, Elasticsearch, Kibana)
├── ingress-nginx        (Ingress Controller)
└── kube-system          (cluster components)

Worker Node Groups:
├── general-nodes        (4 CPU, 8GB — most services)
├── high-memory-nodes    (8 CPU, 32GB — databases, analytics)
└── spot-nodes           (cheap — non-critical batch jobs)
```

---

## Your Question 4 — Deep Dive on Your KillerCoda Output

### Reading Your etcd describe Output

```
kubernetes.io/config.source: file
```
This tells you etcd is a **static pod** — kubelet reads its definition from a file on disk at `/etc/kubernetes/manifests/etcd.yaml`. This is how kubeadm clusters work.

```
Controlled By: Node/controlplane
```
etcd is NOT controlled by a Deployment or ReplicaSet. It is controlled directly by the Node — meaning kubelet on the controlplane node manages it. If you delete this pod, kubelet immediately recreates it from the manifest file.

```
Last State: Terminated
Reason: Unknown
Exit Code: 255
```
The pod restarted 2 times. Exit code 255 usually means the process was killed externally — likely the node was restarted or the container runtime restarted. This is normal in lab environments.

```
Restart Count: 2
```
Not a concern in a lab. In production, more than 3-4 restarts would trigger an investigation.

```
Liveness: http-get http://127.0.0.1:probe-port/livez
Readiness: http-get http://127.0.0.1:probe-port/readyz
Startup: http-get http://127.0.0.1:probe-port/readyz
```
etcd has all three probes configured. Startup probe gives etcd extra time to start before liveness kicks in (failure=24 × period=10s = 4 minutes allowed for startup). This is important — etcd can be slow to start if it is recovering from a crash.

```
Requests:
  cpu: 25m
  memory: 100Mi
```
These are very low resource requests — fine for a lab. In production banking, etcd needs minimum 2 CPU and 8GB RAM dedicated.

### Reading Your kube-controller-manager Output

```
--leader-elect=true
```
This is critical. In an HA cluster with 3 control plane nodes, all 3 run a controller manager pod. But only ONE is the active leader — the others are on standby. If the leader dies, the others elect a new leader automatically. This prevents split-brain.

```
--cluster-cidr=192.168.0.0/16
```
This is the IP range for pods in your cluster. Every pod gets an IP from this range. This is set at cluster creation and cannot be easily changed later.

```
--service-cluster-ip-range=10.96.0.0/12
```
This is the IP range for Services (ClusterIPs). Every Service gets an IP from this range.

```
--controllers=*,bootstrapsigner,tokencleaner
```
The `*` means run ALL built-in controllers. bootstrapsigner and tokencleaner are extras for node bootstrap token management.

### Reading Your kube-scheduler Output

```
--leader-elect=true
```
Same as controller manager — only one scheduler is active leader in HA setup.

```
--bind-address=127.0.0.1
```
Scheduler only listens on localhost — it is not directly accessible from outside the node. All communication goes through API server.

### Reading Your Controller Manager Logs

```
"Caches are synced"
```
This is normal healthy output. Controller manager is syncing its local cache with etcd state. You see this many times because multiple controllers all sync independently.

```
"Missing timestamp for Node. Assuming now as a timestamp" node="node01"
```
When node01 joined or restarted, it did not have a heartbeat timestamp yet. Controller manager assumed current time. This is normal on startup.

```
"Cleaning CSR as it is more than approvedExpiration duration old"
```
Certificate Signing Requests older than 1 hour are being cleaned up automatically. This is the token cleaner controller doing its job — completely normal and healthy.

---

## Your Question 5 — Control Plane Packages and Installation

### What Gets Installed on a Control Plane Node

```bash
# On a fresh Ubuntu server — this is what you install for control plane

# Step 1 — container runtime (containerd)
apt-get install containerd

# Step 2 — Kubernetes tools
apt-get install -y kubelet kubeadm kubectl

# kubelet   = runs on every node (control plane AND worker)
# kubeadm   = tool to bootstrap/manage the cluster
# kubectl   = CLI to talk to the cluster (can be on any machine)

# Step 3 — initialise the control plane
kubeadm init \
  --pod-network-cidr=192.168.0.0/16 \
  --apiserver-advertise-address=<control-plane-ip>
```

### What kubeadm init Actually Does

```
1. Generates all certificates (CA, apiserver, etcd, etc.)
2. Creates static pod manifests in /etc/kubernetes/manifests/
   ├── etcd.yaml
   ├── kube-apiserver.yaml
   ├── kube-controller-manager.yaml
   └── kube-scheduler.yaml
3. kubelet picks up these manifest files and starts the pods
4. Creates kubeconfig files in /etc/kubernetes/
   ├── admin.conf         (admin access)
   ├── controller-manager.conf
   ├── scheduler.conf
   └── kubelet.conf
5. Prints a join command for worker nodes
```

### What Gets Installed on a Worker Node

```bash
# On a fresh Ubuntu server for worker node

# Step 1 — container runtime
apt-get install containerd

# Step 2 — only kubelet and kubeadm needed
apt-get install -y kubelet kubeadm

# kubectl is optional on worker nodes

# Step 3 — join the cluster using token from kubeadm init output
kubeadm join <control-plane-ip>:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

### Packages Summary

| Package | Control Plane | Worker Node | Purpose |
|---|---|---|---|
| containerd | Yes | Yes | Container runtime |
| kubelet | Yes | Yes | Runs pods on the node |
| kubeadm | Yes | Yes | Cluster setup tool |
| kubectl | Yes | Optional | CLI to manage cluster |
| etcd | Auto (static pod) | No | Database (control plane only) |

---

## Your Question 6 — The YAML Files for Control Plane Components

### Where They Live

```bash
# On control plane node — these are the static pod manifests
ls /etc/kubernetes/manifests/

# Output:
etcd.yaml
kube-apiserver.yaml
kube-controller-manager.yaml
kube-scheduler.yaml
```

### How to View Them

```bash
# View API server config
cat /etc/kubernetes/manifests/kube-apiserver.yaml

# View etcd config
cat /etc/kubernetes/manifests/etcd.yaml

# View scheduler config
cat /etc/kubernetes/manifests/kube-scheduler.yaml

# View controller manager config
cat /etc/kubernetes/manifests/kube-controller-manager.yaml
```

### What kube-apiserver.yaml Looks Like

```yaml
apiVersion: v1
kind: Pod                          # it is a static pod
metadata:
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
  - name: kube-apiserver
    image: registry.k8s.io/kube-apiserver:v1.35.1
    command:
    - kube-apiserver
    - --advertise-address=172.30.1.2        # IP of this node
    - --etcd-servers=https://127.0.0.1:2379 # where etcd is
    - --service-cluster-ip-range=10.96.0.0/12
    - --tls-cert-file=/etc/kubernetes/pki/apiserver.crt      # API server certificate
    - --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
    - --client-ca-file=/etc/kubernetes/pki/ca.crt            # CA to verify clients
    - --authorization-mode=Node,RBAC                          # RBAC is enabled here
    - --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
    - --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
    - --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key
    volumeMounts:
    - mountPath: /etc/kubernetes/pki
      name: k8s-certs
      readOnly: true
  volumes:
  - hostPath:
      path: /etc/kubernetes/pki    # certificates from host node
    name: k8s-certs
```

### Critical — How to Modify Control Plane Components

```bash
# To change any control plane setting — edit the manifest file
# kubelet watches this directory and automatically restarts the pod
vi /etc/kubernetes/manifests/kube-apiserver.yaml

# After saving — kubelet detects the change within seconds
# API server pod restarts automatically with new config
# Watch it restart
kubectl get pods -n kube-system -w
```

> Warning — if you make a syntax error in these YAML files, the component will not start and your cluster may break. Always take a backup first.

```bash
# Always backup before editing
cp /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/kube-apiserver.yaml.bak
```

---

## Certificates — Deep Dive

### Why Certificates Are Needed

Without certificates, anyone could send requests to your Kubernetes API server and do anything — delete all pods, read all secrets, destroy the entire cluster. Certificates ensure:

1. **Authentication** — only trusted clients can talk to API server
2. **Encryption** — all traffic between components is encrypted (TLS)
3. **Trust** — components verify each other's identity

### The Certificate Authority (CA)

Think of CA like a government that issues IDs. Every certificate in Kubernetes is signed by the cluster CA. If your certificate is signed by the trusted CA — you are trusted.

```bash
# The CA files are here
ls /etc/kubernetes/pki/
ca.crt   # CA certificate (public) — everyone has this
ca.key   # CA private key — NEVER share this, keep secure
```

### All Certificates in a Kubernetes Cluster

```
/etc/kubernetes/pki/
├── ca.crt                          # Cluster CA certificate
├── ca.key                          # Cluster CA private key
├── apiserver.crt                   # API server certificate
├── apiserver.key                   # API server private key
├── apiserver-kubelet-client.crt    # API server talks to kubelet
├── apiserver-etcd-client.crt       # API server talks to etcd
├── front-proxy-ca.crt              # For extension API servers
├── sa.pub                          # Service account public key
├── sa.key                          # Service account private key
└── etcd/
    ├── ca.crt                      # etcd CA
    ├── server.crt                  # etcd server certificate
    ├── peer.crt                    # etcd peer communication
    └── healthcheck-client.crt      # health check certificate
```

### How to Check Certificate Expiry

```bash
# Check API server certificate expiry
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -dates
# Output:
# notBefore=May 16 20:27:36 2026 GMT
# notAfter=May 16 20:27:36 2027 GMT  <-- expires in 1 year

# Check all certificates at once with kubeadm
kubeadm certs check-expiration

# Output looks like:
# CERTIFICATE                EXPIRES                  RESIDUAL TIME
# admin.conf                 May 16, 2027 20:27 UTC   364d
# apiserver                  May 16, 2027 20:27 UTC   364d
# apiserver-etcd-client      May 16, 2027 20:27 UTC   364d
# etcd-server                May 16, 2027 20:27 UTC   364d
```

### How to Renew Certificates

```bash
# Renew ALL certificates at once
kubeadm certs renew all

# Renew specific certificate
kubeadm certs renew apiserver

# After renewal — restart control plane components
# They need to pick up new certificates
kubectl -n kube-system rollout restart deployment/coredns

# For static pods — they restart automatically when manifest changes
# But if not, delete the pod and kubelet recreates it
kubectl delete pod -n kube-system kube-apiserver-controlplane
```

### How Industry Sets Up Certificate Monitoring

```bash
# In production banking — we set up a CronJob that checks expiry daily
# and sends alert to Slack/PagerDuty if within 30 days

# Manual check script
cat << 'EOF' > /usr/local/bin/check-certs.sh
#!/bin/bash
EXPIRY_THRESHOLD=30  # alert if less than 30 days

kubeadm certs check-expiration | while read line; do
  days=$(echo $line | grep -oP '\d+d' | head -1 | tr -d 'd')
  if [ ! -z "$days" ] && [ "$days" -lt "$EXPIRY_THRESHOLD" ]; then
    echo "ALERT: Certificate expiring in $days days: $line"
    # send to Slack, PagerDuty, email here
  fi
done
EOF

chmod +x /usr/local/bin/check-certs.sh
```

### Interview Answer for Certificate Questions

*"In our banking environment at Atos we had a certificate expiry incident once — not in production but in a UAT cluster. The API server started rejecting all requests with certificate expired error. We identified it immediately using kubeadm certs check-expiration which showed the apiserver certificate had expired. We renewed with kubeadm certs renew all and restarted the affected pods. After that incident we implemented a monitoring CronJob that runs daily, checks certificate expiry using kubeadm, and sends a Slack alert if any certificate is within 30 days of expiry. We also added it to our monthly runbook checklist."*

---

## Security — Why and How

### Why Security Is Critical in Banking Kubernetes

1. Pods can call the Kubernetes API by default — a compromised pod could delete other pods
2. Secrets in Kubernetes are base64 encoded — not encrypted by default
3. Any pod could potentially access another pod's data without network policies
4. A developer with wrong RBAC permissions could accidentally delete production resources

### Security Layers in Production Banking Setup

```
Layer 1 — Network Security
├── VPC with private subnets (worker nodes not publicly accessible)
├── Security Groups (firewall rules for node-to-node traffic)
├── NetworkPolicy (pod-to-pod traffic rules inside cluster)
└── Ingress Controller (only entry point from internet)

Layer 2 — Authentication and Authorization
├── RBAC (who can do what in the cluster)
├── ServiceAccount per application (not using default SA)
├── IRSA on EKS (pods get AWS permissions via IAM role, no stored keys)
└── Audit logging (every API call logged to CloudWatch)

Layer 3 — Pod Security
├── Non-root containers (USER 1001 in Dockerfile)
├── Read-only root filesystem
├── No privileged containers
├── Resource limits (prevents resource exhaustion attacks)
└── Image scanning (Trivy/Snyk in CI pipeline)

Layer 4 — Secret Management
├── AWS Secrets Manager (not Kubernetes Secrets)
├── Secrets injected at runtime via External Secrets Operator
├── etcd encryption at rest enabled
└── No secrets in environment variables or logs
```

---

## Errors You Will See at kube-system Level

### Error 1 — CrashLoopBackOff on a Control Plane Component

```bash
kubectl get pods -n kube-system
# NAME                          READY   STATUS             RESTARTS
# kube-apiserver-controlplane   0/1     CrashLoopBackOff   5

# Investigate
kubectl logs -n kube-system kube-apiserver-controlplane
kubectl describe pod -n kube-system kube-apiserver-controlplane

# Common causes:
# 1. Certificate file missing or wrong path
# 2. etcd not reachable
# 3. Wrong flag in manifest YAML
# 4. Port conflict (another process using 6443)
```

### Error 2 — etcd Unhealthy

```bash
# Check etcd health
kubectl get pods -n kube-system | grep etcd
kubectl logs -n kube-system etcd-controlplane | grep -i error

# Common error in logs:
# "failed to send out heartbeat on time"  → disk too slow
# "took too long to execute"              → disk I/O issue
# "connection refused"                    → etcd peer unreachable in HA setup
```

### Error 3 — Node Not Joining Cluster

```bash
# On worker node when joining fails
kubeadm join <ip>:6443 --token <token> ...

# Common errors:
# "token expired" → generate new token
kubeadm token create --print-join-command

# "certificate mismatch" → wrong hash
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | \
  openssl rsa -pubin -outform der 2>/dev/null | \
  openssl dgst -sha256 -hex | sed 's/^.* //'
# Use this hash in the join command
```

### Error 4 — API Server Certificate Error

```bash
# Users getting x509 certificate has expired or is not yet valid

# Check expiry
kubeadm certs check-expiration

# Fix
kubeadm certs renew all

# Update kubeconfig (admin.conf is also renewed)
cp /etc/kubernetes/admin.conf ~/.kube/config
```

---

## Interview Scenario Questions — Control Plane

---

### Scenario 1
**"You join a new company. They give you access to a Kubernetes cluster. How do you assess the health of the cluster on day one?"**

```bash
# Step 1 — check node health
kubectl get nodes
kubectl describe nodes | grep -E "Ready|MemoryPressure|DiskPressure"

# Step 2 — check control plane components
kubectl get pods -n kube-system
kubectl get componentstatuses

# Step 3 — check certificate expiry
kubeadm certs check-expiration

# Step 4 — check resource usage
kubectl top nodes
kubectl top pods --all-namespaces | sort -k4 -rn | head -20

# Step 5 — check for failing pods
kubectl get pods --all-namespaces | grep -v Running | grep -v Completed

# Step 6 — check recent events for warnings
kubectl get events --all-namespaces \
  --field-selector type=Warning \
  --sort-by='.lastTimestamp' | tail -20

# Step 7 — check etcd backup status (ask the team)
# Step 8 — check monitoring and alerting setup
```

---

### Scenario 2
**"Your production cluster is running fine. Suddenly at 2am certificates expire and no one can access the cluster. Walk me through recovery."**

```bash
# Step 1 — SSH directly to control plane node
# (kubectl won't work — API server rejecting all requests)
ssh user@control-plane-ip

# Step 2 — renew certificates
kubeadm certs renew all

# Step 3 — restart control plane components
# Static pods restart automatically after cert renewal
# But if not — delete the pods
kubectl delete pod -n kube-system kube-apiserver-controlplane
kubectl delete pod -n kube-system kube-controller-manager-controlplane
kubectl delete pod -n kube-system kube-scheduler-controlplane

# Step 4 — update your kubeconfig
cp /etc/kubernetes/admin.conf ~/.kube/config

# Step 5 — verify cluster is back
kubectl get nodes
kubectl get pods --all-namespaces

# Step 6 — update kubeconfig for all team members
# Distribute new admin.conf or generate user-specific kubeconfigs

# Prevention after incident:
# Set up certificate expiry monitoring alert at 30 days
```

---

### Scenario 3
**"A new developer says they accidentally ran kubectl delete namespace production. Everything is gone. What do you do?"**

```bash
# Step 1 — do not panic — check if namespace is still terminating
kubectl get ns production
# If Terminating — you may be able to stop it

# Step 2 — if already gone — go to etcd backup immediately
# Restore from latest etcd snapshot

# Step 3 — restore etcd from backup
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd-restored

# Step 4 — update etcd manifest to use restored data
vi /etc/kubernetes/manifests/etcd.yaml
# Change --data-dir to /var/lib/etcd-restored

# Step 5 — verify restoration
kubectl get ns
kubectl get pods -n production

# Prevention:
# 1. RBAC — developers should NEVER have delete namespace permission
# 2. ResourceLock or admission webhook blocking namespace deletion
# 3. Regular etcd backups with tested restoration procedure
```

*"This is why in banking we had strict RBAC — developers had read-only access to production namespace. Only the DevOps team could delete resources in production, and even then it required a change request approval."*

---

## Key Facts to Memorise for Interviews

1. Control plane components are static pods in kubeadm clusters — manifest files in `/etc/kubernetes/manifests/`
2. MicroK8s runs control plane as snap services — not pods
3. EKS — AWS manages control plane, you manage worker node groups
4. etcd is the single source of truth — losing it without backup means losing everything
5. Certificates expire after 1 year by default in kubeadm clusters
6. kubeadm certs renew all — renews all certificates
7. kubeadm certs check-expiration — shows days until expiry
8. kube-system namespace — where all cluster components live
9. Static pods are controlled by Node (kubelet), not by Deployment
10. --leader-elect=true on scheduler and controller manager — only one active leader in HA cluster
11. --authorization-mode=Node,RBAC — this flag in apiserver manifest enables RBAC
12. /etc/kubernetes/pki/ — all cluster certificates live here

---

*Next file: `05_Pods_Deployments_ReplicaSets.md`*
