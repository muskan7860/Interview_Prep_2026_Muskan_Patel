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
