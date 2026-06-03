# Kubernetes Architecture – Interview Questions & Answers

## 1. What happens if etcd goes down?

### Answer

etcd is the distributed key-value store that serves as Kubernetes' source of truth. All cluster state information is stored in etcd.

If etcd becomes unavailable:

* No new deployments can be created.
* Existing deployments cannot be scaled.
* ConfigMaps and Secrets cannot be modified.
* New nodes cannot join the cluster.
* Any operation that requires updating cluster state will fail.

### What continues to work?

Existing running Pods continue to run because the kubelet on each worker node manages containers locally. Applications already running inside the cluster are typically unaffected in the short term.

### Recovery

Recovery requires restoring etcd from a valid backup or bringing the etcd cluster back online.

### Interview Tip

**No etcd = No cluster state changes. Existing workloads may continue running, but Kubernetes loses its ability to manage the cluster.**

---

## 2. What is the difference between the Scheduler and the Controller Manager?

### Answer

Although both are control plane components, they perform completely different responsibilities.

### Scheduler

The Scheduler decides **where a Pod should run**.

Responsibilities:

* Watches for newly created Pods that do not have a node assigned.
* Evaluates available worker nodes.
* Selects the most suitable node based on:

  * CPU availability
  * Memory availability
  * Taints and tolerations
  * Node affinity/anti-affinity
  * Resource requests and limits

**Example:**

A Deployment creates five Pods.

The Scheduler decides:

* Pod-1 → Worker Node A
* Pod-2 → Worker Node B
* Pod-3 → Worker Node C

### Controller Manager

The Controller Manager ensures the **actual state matches the desired state**.

Responsibilities:

* Creates Pods when replicas are missing.
* Recreates failed Pods.
* Monitors node health.
* Manages Deployments, ReplicaSets, Jobs, and StatefulSets.
* Performs rescheduling when nodes fail.

**Example:**

Desired replicas = 3

Current replicas = 2

The Controller Manager creates one additional Pod to restore the desired state.

### Quick Comparison

| Component          | Responsibility                      |
| ------------------ | ----------------------------------- |
| Scheduler          | Decides where Pods run              |
| Controller Manager | Ensures desired state is maintained |

### Interview Tip

**Scheduler chooses the node. Controller Manager maintains the desired state.**

---

## 3. Can the control plane run on worker nodes?

### Answer

### Development Environment

Yes.

Single-node Kubernetes distributions commonly run both control plane and worker components on the same machine.

Examples:

* Minikube
* Kind
* Single-node MicroK8s
* K3s development environments

### Production Environment

No.

Production environments separate control plane nodes from worker nodes.

Reasons:

* Improved security
* Better resource isolation
* Higher availability
* Easier maintenance
* Worker node failures do not impact cluster management

### Production Architecture

```text
+-----------------------+
|   Control Plane Node  |
+-----------------------+
| API Server            |
| Scheduler             |
| Controller Manager    |
| etcd                  |
+-----------------------+

          |

+-----------------------+
|     Worker Node       |
+-----------------------+
| kubelet               |
| kube-proxy            |
| Application Pods      |
+-----------------------+
```

### Interview Tip

**Control plane and worker nodes are separated in production to eliminate single points of failure.**

---

## 4. What is a Highly Available (HA) Control Plane?

### Answer

A Highly Available Control Plane ensures Kubernetes management components remain available even if one or more control plane nodes fail.

### Typical HA Architecture

#### Multiple API Servers

* API Server 1
* API Server 2
* API Server 3

#### Multiple Scheduler Instances

* One active leader
* Others remain standby through leader election

#### Multiple Controller Managers

* One active leader
* Others ready for failover

#### etcd Cluster

Recommended:

* 3-node etcd cluster (minimum production)
* 5-node etcd cluster (large environments)

#### Load Balancer

A load balancer sits in front of all API servers.

```text
                 kubectl
                     |
              Load Balancer
                     |
      --------------------------------
      |              |              |
   API-1          API-2          API-3
      |              |              |
      --------------------------------
                     |
              etcd Cluster
          (3 or 5 Members)
```

### Benefits

* No single point of failure.
* Control plane remains accessible during node failures.
* Cluster management continues uninterrupted.
* Higher uptime and reliability.

### Interview Tip

**HA Control Plane = Multiple control plane components + etcd quorum + load balancer.**

---

## 5. If kubelet on a node dies, what happens to Pods on that node?

### Answer

The kubelet is the primary agent running on every worker node.

If kubelet stops:

1. The node stops communicating with the API Server.
2. The Controller Manager detects the heartbeat loss.
3. After the node-monitor grace period (approximately 5 minutes by default), the node status changes to:

```text
NotReady
```

### For Deployment or ReplicaSet Managed Pods

The Controller Manager:

* Marks the Pods unavailable.
* Evicts them from the failed node.
* Creates replacement Pods on healthy worker nodes.

### Example

```text
Desired Replicas = 3

Node-1 Failure
    |
Pod-A Lost

Actual Replicas = 2
```

Controller Manager detects the difference and creates:

```text
New Pod-A
    |
Healthy Node
```

### For Standalone Pods

Standalone Pods are not managed by any controller.

Therefore:

* Kubernetes does not recreate them.
* They remain lost until manually recreated.

### Interview Tip

**Only controller-managed workloads (Deployment, ReplicaSet, StatefulSet, DaemonSet) are automatically recreated after node failures. Standalone Pods are not.**

---

# Senior DevOps Engineer Summary

### etcd

Stores cluster state. If it fails, Kubernetes cannot process changes.

### Scheduler

Decides where Pods run.

### Controller Manager

Maintains the desired state of the cluster.

### Control Plane

Should be isolated from worker nodes in production.

### High Availability

Requires multiple control plane nodes, etcd quorum, and a load balancer.

### kubelet Failure

Node becomes NotReady and managed workloads are rescheduled to healthy nodes.
----------------------------------------------------------------------------------------------
# Kubernetes Architecture — Troubleshooting, Scenario Based and Interview Questions

> Level: 4 Years Experience  
> Author: Muskan Patel  
> Domain: Banking — Atos (US and European Clients)

---

## How Interviewers Test Architecture Knowledge

1. They will NOT just ask "explain architecture" — that is only the opening question
2. After your answer they will immediately go into scenario based questions
3. Every scenario tests whether you have actually operated Kubernetes in production
4. For every problem — always follow this structure when answering:
   - What is the symptom
   - What commands you run to investigate
   - What you found
   - How you fixed it
   - What you did to prevent it happening again

---

## Section 1 — Control Plane Failure Scenarios

---

### Scenario 1 — kubectl is not responding, all commands hang or fail

**Interviewer asks:**  
*"You come in Monday morning. Your team says kubectl is not working. No one can deploy anything. What do you do?"*

**Your answer structure:**

1. First check if API server is reachable at all
2. Then check the API server pod or process
3. Then check certificates
4. Then check etcd health
5. Communicate to the team with status updates throughout

**Investigation commands — run in this exact order:**

```bash
# Step 1 — check if API server is reachable
curl -k https://localhost:6443/healthz

# Step 2 — check API server pod (managed k8s like EKS — check pod)
kubectl get pods -n kube-system | grep apiserver

# Step 3 — on self managed k8s or microk8s — check process
ps aux | grep kube-apiserver

# Step 4 — check API server logs
kubectl logs -n kube-system kube-apiserver-controlplane

# Step 5 — on microk8s check via journalctl
journalctl -u snap.microk8s.daemon-apiserver -f --since "10 minutes ago"

# Step 6 — check if certificates have expired (very common cause)
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -dates

# Step 7 — check etcd is healthy (API server depends on etcd)
microk8s kubectl get pods -n kube-system | grep etcd

# Step 8 — check system resources on control plane node
df -h        # disk full can kill API server
free -m      # memory pressure
top          # CPU usage
```

**Root causes and fixes:**

| Cause | How You Know | Fix |
|---|---|---|
| API server pod crashed | Pod not running in kube-system | Check logs, restart pod |
| Certificate expired | openssl shows past date | Renew certs with kubeadm certs renew |
| etcd down | API server logs show etcd connection refused | Fix etcd first |
| Disk full on control plane | df -h shows 100% | Free disk space immediately |
| Memory pressure | OOMKiller killed API server process | Add memory or reduce load |

**What to say in interview:**  
*"In our banking environment we had a certificate expiry incident. The API server started rejecting all requests because the client certificates had expired. We identified it by running openssl against the apiserver.crt file which showed the expiry date had passed. We renewed certificates using kubeadm certs renew all and restarted the control plane components. After that we set up a monitoring alert in CloudWatch to notify us 30 days before any certificate expiry."*

---

### Scenario 2 — Pods are not being scheduled, stuck in Pending

**Interviewer asks:**  
*"Developers are complaining new pods are not starting. They are all showing Pending. The cluster was fine yesterday. What happened?"*

```bash
# Step 1 — check pod status and events
kubectl describe pod <pod-name> -n <namespace>
# Read the Events section at the bottom carefully

# Step 2 — check if scheduler is running
kubectl get pods -n kube-system | grep scheduler

# On microk8s
ps aux | grep kube-scheduler
journalctl -u snap.microk8s.daemon-scheduler --since "10 minutes ago"

# Step 3 — check node status
kubectl get nodes

# Step 4 — check node resources
kubectl describe nodes | grep -A 8 "Allocated resources"

# Step 5 — check if nodes have taints blocking scheduling
kubectl describe nodes | grep Taints

# Step 6 — check resource quota in namespace
kubectl describe resourcequota -n banking

# Step 7 — check if node selector or affinity is too strict
kubectl get pod <pod-name> -o yaml | grep -A 10 affinity
kubectl get pod <pod-name> -o yaml | grep -A 5 nodeSelector
```

**Root causes and fixes:**

| Cause | Event Message in describe | Fix |
|---|---|---|
| Scheduler is down | No events at all after pod creation | Restart scheduler |
| Nodes full — no CPU | 0/3 nodes available insufficient cpu | Reduce requests or scale cluster |
| Nodes full — no memory | 0/3 nodes available insufficient memory | Reduce limits or add nodes |
| Taint blocking all pods | 0/3 nodes have taint that pod does not tolerate | Add toleration or remove taint |
| ResourceQuota exceeded | exceeded quota | Increase quota or reduce replicas |
| PVC not bound | unbound immediate PersistentVolumeClaims | Fix StorageClass |

---

### Scenario 3 — Pod crashed and was NOT replaced automatically

**Interviewer asks:**  
*"A critical payment service pod died at 2am. It was not replaced. The service was down for 2 hours until someone noticed. How does this happen and how do you prevent it?"*

```bash
# Step 1 — check if controller manager is running
kubectl get pods -n kube-system | grep controller-manager

# On microk8s
ps aux | grep kube-controller-manager
journalctl -u snap.microk8s.daemon-controller-manager --since "3 hours ago"

# Step 2 — check how the pod was created
kubectl get pod <pod-name> -o yaml | grep ownerReferences
# If ownerReferences is empty — it is a standalone pod, never replaced

# Step 3 — check if deployment exists
kubectl get deploy -n banking

# Step 4 — check replicaset
kubectl get rs -n banking

# Step 5 — check events around the time of failure
kubectl get events -n banking --sort-by='.lastTimestamp'
```

**Root causes:**

1. Pod was created as a standalone pod — not managed by a Deployment or ReplicaSet
2. Controller manager was down — no one was watching pod health
3. ReplicaSet was manually deleted — pods became orphaned

**Fix and prevention:**

1. Never run standalone pods in production — always use Deployments
2. Set up monitoring alert on pod restarts and pod count dropping below threshold
3. Use PodDisruptionBudget to ensure minimum availability
4. In banking project — we had Prometheus alert firing if any Deployment replica count dropped below desired for more than 2 minutes

```yaml
# PodDisruptionBudget — ensure minimum pods always running
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: payment-pdb
  namespace: banking
spec:
  minAvailable: 2        # always keep minimum 2 pods running
  selector:
    matchLabels:
      app: payment
```

---

### Scenario 4 — etcd is slow, entire cluster is sluggish

**Interviewer asks:**  
*"The cluster is responding very slowly. kubectl commands take 30 seconds. Deployments take forever. No node or pod issues visible. What do you investigate?"*

```bash
# Step 1 — check etcd pod health
kubectl get pods -n kube-system | grep etcd

# Step 2 — check etcd logs for slow queries
kubectl logs -n kube-system etcd-controlplane | grep -i "slow"
kubectl logs -n kube-system etcd-controlplane | grep -i "took too long"

# Step 3 — check etcd endpoint status
# Self managed cluster
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint status --write-out=table

# Step 4 — check disk I/O on etcd node (etcd is extremely sensitive to disk speed)
iostat -x 2 5
# Look for high await time on disk

# Step 5 — check how many objects are in etcd (too many = slow)
kubectl get all --all-namespaces | wc -l

# Step 6 — check API server request rate
kubectl get --raw /metrics | grep apiserver_request_total
```

**Root causes and fixes:**

| Cause | How You Know | Fix |
|---|---|---|
| Slow disk on etcd node | High iostat await | Move etcd to SSD storage |
| Too many objects in cluster | wc -l shows thousands | Clean up unused resources |
| Network latency between etcd nodes | etcd logs show high RTT | Fix network or move nodes closer |
| etcd running on shared node | CPU contention | Dedicate a node to etcd |

**What to say in interview:**  
*"etcd is extremely sensitive to disk I/O latency. In production banking environments we always run etcd on dedicated nodes with SSD disks. We also set up regular etcd compaction and defragmentation as a CronJob to keep the database size manageable."*

```bash
# Compact and defragment etcd — run periodically
ETCDCTL_API=3 etcdctl defrag \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

---

### Scenario 5 — Node showing NotReady

**Interviewer asks:**  
*"You get an alert that one of your worker nodes is NotReady. Pods on that node are not being rescheduled. Walk me through your investigation."*

```bash
# Step 1 — check node status
kubectl get nodes
# Look for NotReady status

# Step 2 — describe the node — check Conditions section
kubectl describe node <node-name>
# Look for:
# MemoryPressure, DiskPressure, PIDPressure — True means problem
# Ready — False means kubelet is not reporting

# Step 3 — SSH into the node
ssh user@<node-ip>

# Step 4 — check kubelet service
systemctl status kubelet
journalctl -u kubelet -f --since "10 minutes ago"

# Step 5 — check disk on the node
df -h
# If /var/lib/kubelet or /var/lib/docker is full — pods cannot start

# Step 6 — check memory
free -m
# Check if OOMKiller ran
dmesg | grep -i "oom"

# Step 7 — check if kubelet can reach API server
curl -k https://<api-server-ip>:6443/healthz

# Step 8 — check kubelet certificate
openssl x509 -in /var/lib/kubelet/pki/kubelet-client-current.pem -noout -dates

# Step 9 — after fixing node — check if pods reschedule
kubectl get pods -n banking -o wide
# Pods should move to healthy nodes after 5 minutes automatically
```

**Why pods were not rescheduling:**

1. Default eviction timeout is 5 minutes — pods may still be rescheduling
2. Check if pods are managed by Deployment — standalone pods never reschedule
3. Check if all other nodes are also full — nowhere to reschedule to

---

### Scenario 6 — CoreDNS failure, services not reachable by name

**Interviewer asks:**  
*"Suddenly all microservices in the banking application cannot reach each other. Pods are Running. Services exist. But inter-service calls are failing with connection refused or unknown host. What do you check?"*

```bash
# Step 1 — check CoreDNS pods
kubectl get pods -n kube-system | grep coredns

# Step 2 — check CoreDNS logs
kubectl logs -n kube-system <coredns-pod-name>
# Look for errors, panics, OOM

# Step 3 — test DNS from inside a pod
kubectl exec -it <any-running-pod> -n banking -- nslookup kubernetes
kubectl exec -it <any-running-pod> -n banking -- nslookup payment-service.banking.svc.cluster.local

# Step 4 — check if kube-dns service exists and has correct IP
kubectl get svc -n kube-system kube-dns
kubectl get endpoints -n kube-system kube-dns

# Step 5 — check CoreDNS ConfigMap for misconfig
kubectl get configmap coredns -n kube-system -o yaml

# Step 6 — restart CoreDNS pods (quick fix)
kubectl rollout restart deployment/coredns -n kube-system

# Step 7 — check if pods have correct DNS config
kubectl exec -it <pod-name> -n banking -- cat /etc/resolv.conf
# Should show nameserver pointing to kube-dns ClusterIP
```

**On MicroK8s specifically:**

```bash
# Check if DNS addon is enabled
microk8s status | grep dns

# Enable DNS if not enabled
microk8s enable dns

# Restart DNS
microk8s disable dns && microk8s enable dns
```

---

### Scenario 7 — RBAC Forbidden error in production

**Interviewer asks:**  
*"Your CI/CD pipeline was working fine. After a team member made some changes to the cluster, all deployments from the pipeline started failing with Forbidden error. How do you debug this?"*

```bash
# Step 1 — check exact error message
# Error will say something like:
# User "system:serviceaccount:banking:cicd-sa" cannot create deployments

# Step 2 — identify which service account the pipeline uses
kubectl get serviceaccount -n banking

# Step 3 — check what permissions the SA currently has
kubectl auth can-i create deployments \
  --as=system:serviceaccount:banking:cicd-sa -n banking

# Step 4 — list all permissions of the SA
kubectl auth can-i --list \
  --as=system:serviceaccount:banking:cicd-sa -n banking

# Step 5 — find role bindings for this SA
kubectl get rolebindings -n banking -o yaml | grep -A 5 cicd-sa
kubectl get clusterrolebindings -o yaml | grep -A 5 cicd-sa

# Step 6 — check the role that is bound
kubectl get role cicd-role -n banking -o yaml
kubectl describe role cicd-role -n banking

# Step 7 — fix by adding missing permission to role
kubectl edit role cicd-role -n banking
# Add missing verb to rules
```

**Fix — add deployment permission to role:**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: cicd-role
  namespace: banking
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "create", "update", "patch", "delete"]
- apiGroups: [""]
  resources: ["pods", "services"]
  verbs: ["get", "list", "watch"]
```

**What to say in interview:**  
*"We had this exact issue in our banking CI/CD pipeline at Atos. A team member had cleaned up some RBAC objects thinking they were unused. The CI/CD service account lost its deployment permissions. We identified it by running kubectl auth can-i which immediately showed the missing permission. We restored the RoleBinding and added an audit step in our pipeline to verify RBAC permissions before running any deployment steps."*

---

### Scenario 8 — Metrics Server down, HPA not scaling

**Interviewer asks:**  
*"The banking application is under heavy load. CPU is high. But HPA is not scaling up pods. kubectl top nodes shows error. What is the issue?"*

```bash
# Step 1 — check HPA status
kubectl get hpa -n banking
# If TARGETS shows <unknown>/80% — metrics not available

# Step 2 — describe HPA to see conditions
kubectl describe hpa payment-hpa -n banking
# Look for: "unable to get metrics for resource cpu"

# Step 3 — check metrics server pod
kubectl get pods -n kube-system | grep metrics-server

# Step 4 — check metrics server logs
kubectl logs -n kube-system <metrics-server-pod>

# Step 5 — test if metrics API is working
kubectl get --raw /apis/metrics.k8s.io/v1beta1/nodes

# Step 6 — on MicroK8s enable metrics server
microk8s enable metrics-server

# Step 7 — verify after fix
kubectl top nodes
kubectl top pods -n banking
kubectl get hpa -n banking
# TARGETS should now show actual values like 85%/80%
```

---

## Section 2 — Architecture Interview Questions and Answers

---

### Question 1  
**"What happens to running pods if the control plane goes down completely?"**

1. Running pods continue to run — kubelet on each node keeps them alive independently
2. No new pods can be scheduled — scheduler is gone
3. Failed pods are not replaced — controller manager is gone
4. No kubectl commands work — API server is gone
5. Existing Services continue routing traffic — kube-proxy rules are already written in iptables
6. This is by design — data plane (workloads) is intentionally decoupled from control plane

---

### Question 2  
**"Explain etcd and why it is critical"**

1. etcd is a distributed key-value store using the Raft consensus algorithm
2. It is the single source of truth for all cluster state
3. Every object in Kubernetes — pod, deployment, secret, configmap — is stored in etcd
4. API server is the only component that reads and writes to etcd directly
5. In production we run etcd as a 3 or 5 node cluster — odd numbers for Raft quorum
6. With 3 nodes — can tolerate 1 node failure. With 5 nodes — can tolerate 2 node failures
7. Regular backup using etcdctl snapshot save is mandatory — especially in banking
8. etcd requires fast SSD disk — it is extremely sensitive to disk I/O latency

---

### Question 3  
**"What is the difference between kubectl delete pod and a pod being killed by OOMKiller?"**

1. kubectl delete pod — sends SIGTERM to container, waits for graceful shutdown (terminationGracePeriodSeconds), then SIGKILL
2. OOMKiller — Linux kernel kills the process immediately when node runs out of memory, no graceful shutdown
3. Both result in pod restart if managed by Deployment
4. OOMKilled shows in kubectl describe pod under Last State as reason OOMKilled
5. Fix for OOMKilled — increase memory limit in pod spec

---

### Question 4  
**"How does Kubernetes handle a node failure in production?"**

1. Node stops sending heartbeats to API server via kubelet
2. After 40 seconds — node condition changes to Unknown
3. Node Controller marks node as NotReady
4. Node Controller adds taint node.kubernetes.io/not-ready:NoExecute to the node
5. After 5 minutes (default tolerationSeconds) — pods on that node are evicted
6. ReplicaSet Controller creates replacement pods on healthy nodes
7. kube-proxy updates routing rules to stop sending traffic to pods on failed node
8. If node comes back — it rejoins cluster, kubelet re-registers, new pods can be scheduled

---

### Question 5  
**"What is the reconciliation loop and why does it matter?"**

1. Every controller in Kubernetes runs an infinite reconciliation loop
2. The loop does three things — observe current state, compare with desired state, take action to close the gap
3. This is the reason Kubernetes self-heals — the loop runs every few seconds permanently
4. Example — you want 3 replicas, a pod crashes, loop detects 2 running, creates 1 new pod
5. This is also why manually deleting a pod from a Deployment does not help — the loop recreates it immediately
6. The pattern is called level-triggered — it does not react to events, it continuously checks state

---

### Question 6  
**"What is the difference between a liveness probe failure and a readiness probe failure?"**

1. Liveness probe failure — container is considered dead, kubelet restarts it
2. Readiness probe failure — pod is removed from Service endpoints, traffic stops going to it, but container is NOT restarted
3. A pod can be Running but not Ready — liveness passes, readiness fails
4. This is used for graceful handling — when app is temporarily overloaded, remove from load balancer without restarting
5. In banking — readiness probe on payment service checked database connection. If DB was slow, pod went NotReady, traffic routed to healthy pods, no customer impact

---

### Question 7  
**"How do you back up and restore etcd?"**

```bash
# Backup etcd
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify backup
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-snapshot.db

# Restore etcd
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd-restored

# Update etcd to use restored data directory
# Edit /etc/kubernetes/manifests/etcd.yaml
# Change --data-dir to /var/lib/etcd-restored
```

---

### Question 8  
**"What is a static pod and where is it used?"**

1. Static pods are managed directly by kubelet — not by the API server or controller manager
2. kubelet reads pod manifests from a local directory — default /etc/kubernetes/manifests
3. If a static pod crashes — kubelet restarts it automatically without API server involvement
4. This is exactly how control plane components run in kubeadm clusters — kube-apiserver, etcd, scheduler, controller-manager are all static pods
5. If you kubectl delete a static pod — it immediately comes back because kubelet recreates it from the manifest file
6. To truly stop a static pod — remove or rename its manifest file from the directory

---

## Section 3 — Quick Fire Questions

> Interviewers ask these to test depth — answer in 2 to 3 sentences maximum

| Question | Answer |
|---|---|
| What port does API server run on? | 6443 for HTTPS. All kubectl commands connect here. |
| What database does Kubernetes use? | etcd — a distributed key-value store |
| What is kubelet? | Agent on every worker node that starts and manages containers via container runtime |
| What is kube-proxy? | Manages iptables rules on each node to route Service traffic to correct pod IPs |
| What container runtime does modern K8s use? | containerd — Docker was deprecated as runtime in K8s 1.24 |
| What is a manifest file? | YAML or JSON file describing a Kubernetes object — applied with kubectl apply |
| Where are static pod manifests stored? | /etc/kubernetes/manifests on the control plane node |
| What is the default pod eviction timeout? | 5 minutes after node goes NotReady |
| What is IRSA in EKS? | IAM Roles for Service Accounts — gives pods AWS IAM permissions without storing credentials |
| What is the difference between kubectl apply and kubectl create? | apply is declarative and idempotent — create fails if resource already exists |

---

## Section 4 — Banking Project Answers

> Use these when interviewer asks "give me a real example from your work"

1. **Certificate expiry** — We had API server certificates expire in a non-production cluster. Identified with openssl, renewed with kubeadm certs renew, set up 30-day expiry alerting after.

2. **etcd backup** — In our banking environment, we ran automated etcd snapshots every 6 hours using a CronJob, stored snapshots in an encrypted S3 bucket with 30 day retention.

3. **Node failure** — A worker node went NotReady due to disk pressure from application logs filling the volume. We identified it via kubectl describe node showing DiskPressure True. Cleared old logs, added log rotation policy, added disk usage alert.

4. **RBAC issue** — CI/CD pipeline lost deployment permissions after a cleanup. Identified with kubectl auth can-i, restored RoleBinding, added permission verification step to pipeline.

5. **CoreDNS crash** — Payment service could not reach transaction service. All pods Running. Found CoreDNS OOMKilled due to memory limit too low. Increased CoreDNS memory limit and added monitoring alert on CoreDNS pod restarts.

---

*Next file: `04_Pods_Deployments_ReplicaSets.md`*
