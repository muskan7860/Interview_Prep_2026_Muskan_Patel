# Start minikube
minikube start

# See control plane components running as pods
kubectl get pods -n kube-system

# You will see: api-server, etcd, scheduler, controller-manager, kube-proxy, coredns

# Check node status
kubectl get nodes

# See detailed node info including kubelet version and container runtime
kubectl describe node minikube

# Check component health
kubectl get componentstatuses

# Architecture Troubleshooting — Scenario Based

---

# Scenario 1: You run kubectl get pods and get "connection refused" error.

This means the API server is not reachable.

```bash
# Check if API server pod is running
kubectl get pods -n kube-system | grep apiserver

# Check API server logs
kubectl logs -n kube-system kube-apiserver-controlplane

# If you cannot use kubectl at all, SSH to control plane node
systemctl status kube-apiserver

# Check certificates haven't expired
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -dates
```

### Root causes

* API server pod crashed
* Certificate expired (very common after 1 year)
* Network issue to control plane node

---

# Scenario 2: New pods are staying in Pending forever but nodes have resources.

```bash
# Check pod events
kubectl describe pod <pod-name>

# Look at Events section at bottom

# Check if scheduler is running
kubectl get pods -n kube-system | grep scheduler

# Check scheduler logs
kubectl logs -n kube-system kube-scheduler-controlplane

# Check node taints
kubectl describe nodes | grep Taints

# Check if node selectors or affinity rules are too strict
kubectl get pod <pod-name> -o yaml | grep -A 10 affinity

kubectl get pod <pod-name> -o yaml | grep -A 5 nodeSelector
```

### Root causes

* scheduler down
* all nodes tainted and pod has no toleration
* node affinity rules match no nodes
* resource requests too high for available nodes

---

# Scenario 3: A pod died and is not being replaced.

```bash
# Check if controller manager is running
kubectl get pods -n kube-system | grep controller-manager

# Check controller manager logs
kubectl logs -n kube-system kube-controller-manager-controlplane

# Check if the pod is part of a deployment or standalone
kubectl get pod <pod-name> -o yaml | grep ownerReferences

# Standalone pods are NEVER replaced automatically
# Only pods owned by ReplicaSet/Deployment/StatefulSet are replaced
```

### Root causes

* controller manager is down
* pod is standalone (not managed by a Deployment)
* ReplicaSet was manually deleted

---

# Scenario 4: kubectl top nodes shows "error: metrics not available".

```bash
# Check metrics server
kubectl get pods -n kube-system | grep metrics-server

# Check metrics server logs
kubectl logs -n kube-system <metrics-server-pod>

# Common fix — metrics server needs this flag in args:
# --kubelet-insecure-tls

# Check deployment args
kubectl get deployment metrics-server -n kube-system -o yaml | grep args -A 10
```

### Root causes

* metrics-server not installed
* metrics-server pod not running
* TLS certificate issue between metrics-server and kubelets

---

# Scenario 5: Internal DNS not working — pods cannot reach services by name.

```bash
# Check CoreDNS pods
kubectl get pods -n kube-system | grep coredns

# Check CoreDNS logs
kubectl logs -n kube-system <coredns-pod>

# Test DNS from inside a pod
kubectl exec -it <any-running-pod> -- nslookup kubernetes

kubectl exec -it <any-running-pod> -- nslookup <service-name>.<namespace>

# Check CoreDNS ConfigMap for misconfig
kubectl get configmap coredns -n kube-system -o yaml

# Check if CoreDNS service exists
kubectl get svc -n kube-system kube-dns
```

### Root causes

* CoreDNS pods crashing
* CoreDNS ConfigMap misconfigured
* kube-dns Service missing or wrong ClusterIP

---

# Scenario 6: Node shows NotReady. Pods on it are not being evicted.

```bash
# Check node condition
kubectl describe node <node-name>

# Look at Conditions section — check DiskPressure, MemoryPressure, PIDPressure

# SSH to the node and check kubelet
systemctl status kubelet

journalctl -u kubelet -f --since "5 minutes ago"

# Check disk on node
df -h

# Check memory
free -m

# Pods eviction starts after 5 minutes by default

# Check if pods are being evicted
kubectl get pods --all-namespaces | grep Terminating
```

### Root causes

* kubelet died
* disk pressure (disk full)
* memory pressure (OOM)
* network issue between node and control plane
* certificate issue on node

---

# Scenario 7: etcd is showing high latency alerts.

```bash
# Check etcd pod
kubectl get pods -n kube-system | grep etcd

# Check etcd logs for slow queries
kubectl logs -n kube-system etcd-controlplane | grep "slow"

# Check etcd metrics
kubectl exec -n kube-system etcd-controlplane -- etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint status --write-out=table

# Check disk I/O on etcd node (etcd is very sensitive to disk speed)
iostat -x 2
```

### Root causes

* slow disk on etcd node (etcd needs fast SSD)
* too many objects in etcd (clean up unused resources)
* network latency between etcd nodes
* resource contention on control plane node

---

# Scenario 8: RBAC — user gets "Forbidden" error.

```bash
# Check what permissions a user has
kubectl auth can-i get pods --as=john -n banking

# Check all permissions for a user
kubectl auth can-i --list --as=john -n banking

# Find RoleBindings for the user
kubectl get rolebindings -n banking -o yaml | grep john

# Find ClusterRoleBindings
kubectl get clusterrolebindings -o yaml | grep john

# Check the Role to see what verbs are allowed
kubectl get role <role-name> -n banking -o yaml
```

### Root causes

* RoleBinding missing
* wrong namespace in RoleBinding
* Role has wrong verbs
* user name in certificate does not match name in RoleBinding

---

# Interview Tip

For troubleshooting questions, always answer using this framework:

```text
1. Identify the failing component
2. Verify component health
3. Check logs
4. Check configuration
5. Validate dependencies
6. Confirm root cause
7. Apply fix
8. Verify recovery
```

This structured approach demonstrates production-level troubleshooting skills and is exactly what interviewers expect from a DevOps Engineer with 3–5 years of experience.

