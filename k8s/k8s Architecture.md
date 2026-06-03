# Kubernetes Architecture — Interview Answer Guide

**Role:** DevOps Engineer (3–4 Years Experience)
**Purpose:** Interview Preparation + GitHub Portfolio Documentation

---

# How to Use This File

### Version 1

Read daily until the explanation flows naturally without memorization.

### Version 2

Practice speaking it aloud and record yourself.

### Version 3

Study the deployment flow carefully because it is one of the most frequently asked Kubernetes interview questions.

### Important Rule

Never memorize answers word for word.

Understand:

* Why the component exists
* What it does
* What happens when it fails
* How it behaves in production

Interviewers can easily identify memorized answers.

---

# Rule Before You Answer

When an interviewer asks:

> "Explain Kubernetes Architecture"

They are usually evaluating three things:

### 1. Do you understand the big picture?

Or are you simply listing component names?

### 2. Do you know WHY each component exists?

Not just WHAT it does.

### 3. Have you worked on Kubernetes?

Or have you only studied theory?

---

## Golden Structure

Always answer in this order:

1. Explain the purpose of Kubernetes
2. Introduce Control Plane and Worker Nodes
3. Explain every component
4. Explain what happens when it fails
5. End with a real production example

---

# Version 1 — 60 Second Answer

Use this when the interviewer asks for a quick overview.

### Answer

Kubernetes is a container orchestration platform that automates deployment, scaling, networking, and self-healing of containerized applications.

Instead of manually managing servers and containers, we declare the desired state, such as running three replicas of an application, and Kubernetes continuously ensures that desired state is maintained.

The architecture consists of two major parts:

## Control Plane

The Control Plane acts as the brain of the cluster.

It contains:

### API Server

* Entry point for all cluster operations
* Receives requests from kubectl, CI/CD pipelines, and internal components

### etcd

* Distributed key-value database
* Stores complete cluster state

### Scheduler

* Decides which worker node should run a Pod

### Controller Manager

* Continuously compares desired state and actual state
* Creates, deletes, or replaces resources as needed

---

## Worker Nodes

Worker nodes run application workloads.

Each worker node contains:

### kubelet

* Node agent
* Starts and manages containers

### kube-proxy

* Handles Service networking and traffic routing

### Container Runtime

* Usually containerd
* Runs containers

---

## Supporting Components

### CoreDNS

Provides internal DNS resolution.

### Metrics Server

Provides CPU and memory metrics for autoscaling.

---

### Real Project Example

In my project, we used AWS EKS.

AWS managed the Control Plane components, while our team managed worker node groups, deployments, monitoring, networking, and application delivery.

---

# Version 2 — 3 Minute Deep Answer

Use this as your primary interview answer.

---

## Opening

Kubernetes follows a desired-state model.

Instead of telling Kubernetes:

> "Start this container on Server 5."

We declare:

> "I want three replicas of this application."

Kubernetes continuously works to make reality match the declared state.

This principle is the foundation of Kubernetes architecture.

---

# Part 1 — Control Plane

The Control Plane is responsible for managing the cluster.

---

## API Server

The API Server is the central communication hub.

Every operation goes through it:

* kubectl commands
* CI/CD deployments
* Internal Kubernetes components

### Responsibilities

* Authentication
* Authorization (RBAC)
* Admission Control
* API Validation
* Persisting data to etcd

### Failure Scenario

If the API Server becomes unavailable:

* No new deployments
* No scaling
* No updates

However:

Existing Pods continue running because they already exist on worker nodes.

---

## etcd

etcd is Kubernetes' source of truth.

Everything is stored here:

* Deployments
* Pods
* Services
* Secrets
* ConfigMaps
* Node information

### High Availability

Production environments typically use:

* 3-node cluster
* 5-node cluster

Odd numbers are required for quorum.

### Failure Scenario

If etcd fails:

* Cluster state becomes unavailable
* No configuration changes can occur
* Existing Pods continue running

### Production Best Practice

Regular etcd backups are mandatory.

Without backups, cluster recovery may be impossible.

---

## Scheduler

The Scheduler determines where Pods should run.

### Scheduling Process

### Step 1 — Filtering

Remove nodes that cannot run the Pod.

Examples:

* Insufficient CPU
* Insufficient memory
* Taints
* Affinity constraints

### Step 2 — Scoring

Rank remaining nodes.

### Step 3 — Binding

Assign Pod to the best node.

### Important Interview Point

Scheduler does not start Pods.

It only assigns a node.

### Failure Scenario

If Scheduler fails:

* Existing Pods continue running
* New Pods remain Pending

---

## Controller Manager

Controller Manager continuously runs reconciliation loops.

Its job:

Compare:

Desired State

vs

Actual State

### Example

Desired Replicas = 3

Actual Replicas = 2

Controller Manager creates one more Pod.

### Important Controllers

* Deployment Controller
* ReplicaSet Controller
* Node Controller
* Endpoint Controller
* Job Controller
* CronJob Controller

### Failure Scenario

If Controller Manager fails:

* Self-healing stops
* Failed Pods are not recreated
* Scaling activities stop

---

# Part 2 — Worker Nodes

Worker nodes execute workloads.

---

## kubelet

kubelet is the primary node agent.

Responsibilities:

* Receives Pod assignments
* Pulls container images
* Starts containers
* Executes health probes
* Reports status to API Server

### Important Interview Point

kubelet is not a Pod.

It runs as a system service.

### Troubleshooting

```bash
systemctl status kubelet

journalctl -u kubelet -f
```

### Failure Scenario

Node eventually becomes NotReady.

Controller Manager schedules replacement Pods elsewhere.

---

## kube-proxy

kube-proxy handles Service networking.

Responsibilities:

* Creates iptables/IPVS rules
* Routes Service traffic
* Enables ClusterIP communication

### Important Interview Point

kube-proxy does NOT handle Pod networking.

That is the responsibility of the CNI plugin.

### Failure Scenario

Services stop routing traffic correctly.

---

## Container Runtime

Modern Kubernetes primarily uses:

* containerd

Responsibilities:

* Pull images
* Create containers
* Run containers

Communication occurs through:

CRI (Container Runtime Interface)

### Important Interview Point

Docker is no longer the default runtime.

containerd is used directly.

---

# Part 3 — Supporting Components

---

## CoreDNS

Runs inside the kube-system namespace.

Provides internal DNS resolution.

Example:

```text
payment-service.default.svc.cluster.local
```

### Failure Scenario

Pods can communicate using IP addresses.

Service-name-based communication fails.

In microservices environments, this causes major outages.

---

## Metrics Server

Collects:

* CPU Metrics
* Memory Metrics

Used by:

* HPA
* kubectl top nodes
* kubectl top pods

### Failure Scenario

Autoscaling stops functioning correctly.

HPA displays:

```text
<unknown>
```

---

# Closing — Production Example

In AWS EKS:

AWS manages:

* API Server
* etcd
* Scheduler
* Controller Manager

Our team manages:

* Worker Nodes
* IAM Roles for Service Accounts (IRSA)
* Monitoring
* Logging
* Deployments
* Networking
* Security

This architecture is commonly used across enterprise environments.

---

# Version 3 — Walk Me Through a Deployment

One of the most common Kubernetes interview questions.

---

## Scenario

```bash
kubectl apply -f deployment.yaml
```

### Step 1

kubectl converts YAML into an HTTP request.

### Step 2

Request reaches API Server.

### Step 3

Authentication occurs.

Certificate or token validation.

### Step 4

RBAC Authorization occurs.

Permission checks are performed.

### Step 5

Mutating Admission Controllers execute.

Examples:

* Inject sidecars
* Add labels
* Apply defaults

### Step 6

Schema validation occurs.

Required fields are verified.

### Step 7

Validating Admission Controllers execute.

Examples:

* ResourceQuota
* Security Policies

### Step 8

Deployment object is stored in etcd.

Desired state now exists.

### Step 9

Deployment Controller creates a ReplicaSet.

### Step 10

ReplicaSet Controller creates Pod objects.

### Step 11

Scheduler assigns nodes.

### Step 12

kubelet detects assigned Pods.

### Step 13

containerd pulls images.

### Step 14

Containers start.

### Step 15

Pod status updates to Running.

### Step 16

Endpoint Controller updates Service endpoints.

### Step 17

kube-proxy updates networking rules.

### Result

Application becomes reachable through Kubernetes Services.

---

# Common Interview Mistakes

| Avoid Saying              | Say Instead                                     |
| ------------------------- | ----------------------------------------------- |
| Master Node               | Control Plane                                   |
| Docker runs containers    | containerd runs containers in modern Kubernetes |
| Listing components only   | Explain purpose and failure impact              |
| I read that Kubernetes... | In my project we implemented...                 |
| Scheduler starts Pods     | Scheduler only assigns nodes                    |

---

# Quick Follow-Up Questions

## What happens if etcd goes down?

* No cluster changes possible
* Existing Pods continue running
* Restore from backup required

---

## What is the reconciliation loop?

Continuous comparison between:

Desired State

and

Actual State

Controllers close the gap automatically.

---

## How does scheduling work?

1. Filter nodes
2. Score nodes
3. Assign node
4. kubelet starts Pod

---

## What are Admission Controllers?

### Mutating

Modify requests.

### Validating

Approve or reject requests.

---

## What is RBAC?

RBAC controls access to Kubernetes resources.

### Role

Defines permissions.

### RoleBinding

Assigns permissions.

### ClusterRole

Cluster-wide permissions.

---

## Difference Between Taints and Affinity

### Taints

Push Pods away from nodes.

### Affinity

Pull Pods toward nodes.

---

## Difference Between kubelet and kube-proxy

### kubelet

Runs and manages containers.

### kube-proxy

Routes Service traffic.

---

## Why does etcd use odd numbers?

Raft consensus requires majority quorum.

Examples:

* 3 nodes tolerate 1 failure
* 5 nodes tolerate 2 failures

Using even numbers provides no additional fault tolerance while increasing resource usage.
NAME                               READY   STATUS    
coredns-5d78c9869d-xkj2p          1/1     Running   
etcd-minikube                      1/1     Running   
kube-apiserver-minikube            1/1     Running   
kube-controller-manager-minikube   1/1     Running   
kube-proxy-7x9kp                   1/1     Running   
kube-scheduler-minikube            1/1     Running   
storage-provisioner                1/1     Running   
metrics-server-7746886d4f-tn2p8   1/1     Running
---------------------------------------------------------------------------------------------------------
You
  │
  ▼
kubectl ──► API Server ──► etcd (stores everything)
               │
               ├──► Scheduler (assigns pods to nodes)
               │
               ├──► Controller Manager (watches & corrects state)
               │         ├── ReplicaSet Controller
               │         ├── Deployment Controller
               │         └── Node Controller
               │
               └──► Each Worker Node
                         ├── kubelet (runs containers)
                         ├── kube-proxy (routes service traffic)
                         └── Container Runtime (containerd)

CoreDNS ──► resolves service names to IPs for all pods
Metrics Server ──► feeds CPU/memory data to HPA
---------------------------------------------------------------------------------------
# Kubernetes Core Components — Deep Dive Interview Notes

## kube-apiserver

### What it is

The front door of the entire cluster. Every single thing — kubectl commands, internal components, external tools — all communicate through the API server. Nothing bypasses it.

### What it does in production

Receives your kubectl apply, validates the YAML, checks your permissions via RBAC, then writes the desired state into etcd. It also serves as the communication hub — controller manager watches it, scheduler watches it, kubelet reports to it.

### What happens if it dies

You lose all control of the cluster. Cannot deploy, scale, delete, or view anything via kubectl. But — and this is important for interviews — existing running pods continue running because kubelet keeps them alive independently.

### Interview question

**How do you check if the API server is healthy?**

```bash
kubectl get componentstatuses

kubectl get pod kube-apiserver-minikube -n kube-system
```

---

# etcd

### What it is

A distributed key-value database. The single source of truth for everything in your cluster — every pod definition, every service, every secret, every config.

### What it does in production

When API server writes "user wants 3 replicas of app X", that gets stored in etcd. When controller manager asks "what is the desired state", it reads from etcd. Everything in Kubernetes is just reading and writing to etcd.

### Critical production fact

In production clusters, etcd runs as a 3-node or 5-node cluster (always odd numbers) for high availability. It uses the Raft consensus algorithm — majority of nodes must agree before a write is accepted. With 3 nodes, you can lose 1. With 5 nodes, you can lose 2.

### Backup is mandatory

```bash
etcdctl snapshot save backup.db
```

Run this regularly. If etcd data is lost with no backup, your cluster configuration is permanently gone.

### What happens if it dies

API server cannot read or write state. Cluster management completely stops. Existing pods still run.

### Interview question

**Why does etcd use odd numbers of nodes?**

Because of Raft consensus — you need a majority (quorum) to agree. With 3 nodes, quorum = 2. With 4 nodes, quorum is still 3 — you get no extra fault tolerance over 3 nodes but have more complexity. Odd numbers give maximum fault tolerance per node added.

---

# kube-scheduler

### What it is

The component that decides which worker node a new pod runs on.

### What it does step by step

1. Watches for pods that have no node assigned yet (nodeName is empty)
2. Runs filtering — removes nodes that cannot run the pod (not enough CPU, wrong OS, has a taint the pod cannot tolerate)
3. Runs scoring — ranks remaining nodes (prefers node with most free resources, spreads pods across nodes for availability)
4. Assigns the pod to the highest scoring node by writing the nodeName into etcd

### What it does NOT do

It does not start the pod. It only decides where it goes. The kubelet on that node does the actual starting.

### What happens if it dies

Existing pods keep running. But any new pods that need scheduling will sit in Pending state forever — because nothing is assigning them to nodes.

### Interview question

**A pod is stuck in Pending. Could the scheduler be the cause?**

Yes — if the scheduler is down, or if no node passes the filtering phase (insufficient resources, taint mismatch, node selector mismatch).

Check:

```bash
kubectl describe pod <pod-name>
```

The Events section will say:

```text
no nodes available to schedule pods
```

or similar.

---

# kube-controller-manager

### What it is

A single binary that runs multiple controllers — each controller is a background loop that watches the cluster state and corrects it when reality does not match the desired state.

### The controllers inside it that matter for interviews

#### Deployment Controller

Watches Deployment objects. When you create a Deployment, it creates a ReplicaSet.

#### ReplicaSet Controller

Watches ReplicaSet objects. Makes sure the correct number of pods are running at all times. If a pod dies, this controller creates a replacement immediately.

#### Node Controller

Watches node health. If a node stops sending heartbeats, it marks it NotReady. After a timeout (default 5 minutes), it evicts pods from that node.

#### Job Controller

Watches Job objects and creates pods to run them. Marks Job complete when pods finish successfully.

#### Endpoint Controller

Watches Services and Pods. When pod IPs change, it updates the Endpoints object so traffic routes correctly.

### What happens if it dies

Existing pods keep running. But self-healing stops — if a pod dies, no replacement is created. Node failures are not handled. HPA stops scaling.

### Interview question

**What is the reconciliation loop?**

Every controller runs an infinite loop:

```text
observe current state
        ↓
compare with desired state
        ↓
take action to fix any difference
```

This is the core pattern of Kubernetes. The system is always trying to make reality match what you declared.

---

# kubelet

### What it is

An agent that runs on every worker node. It is the only Kubernetes component that actually runs containers.

### What it does

* Watches the API server for pods assigned to its node
* Pulls the container image via the container runtime (containerd)
* Creates and starts the containers
* Runs liveness and readiness probes
* Reports pod status back to API server continuously
* Manages pod lifecycle — restarts containers on failure based on restartPolicy

### Critical fact

kubelet is the only component that does not run as a pod. It runs directly as a systemd service on the node.

This is why you check it with:

```bash
systemctl status kubelet
```

not:

```bash
kubectl get pods
```

### What happens if it dies

The node stops reporting to the control plane. Existing containers on that node keep running (because the container runtime is still alive) but Kubernetes has no visibility into them. The node eventually goes NotReady.

### Interview question

**How do you troubleshoot kubelet on a node?**

```bash
systemctl status kubelet

journalctl -u kubelet -f --since "10 minutes ago"
```

Look for:

* Certificate errors
* API server connection issues
* Disk pressure errors

---

# kube-proxy

### What it is

A network component that runs on every node. It handles traffic routing for Kubernetes Services.

### What it does

When you create a Service with ClusterIP 10.96.0.1 pointing to pods on 10.244.1.5 and 10.244.2.7, kube-proxy writes iptables rules (or IPVS rules) on every node so that traffic to 10.96.0.1 gets load-balanced to the actual pod IPs.

### Important distinction

kube-proxy does NOT handle pod-to-pod networking.

That is the job of the CNI plugin (Calico, Flannel, etc.).

kube-proxy only handles Service-to-pod routing.

### What happens if it dies

Pods can still talk to each other directly by IP. But Service DNS names stop working — traffic to any Service stops being routed correctly. Applications that use service names to communicate (which is almost all of them) will start failing.

### Interview question

**What is the difference between kube-proxy and a CNI plugin?**

**CNI plugin**

* Pod networking
* How pods get IPs
* How pod-to-pod traffic flows

**kube-proxy**

* Service networking
* How traffic to a Service ClusterIP gets routed to the right pods

Both are needed for a fully functional cluster.

---

# coredns

### What it is

The DNS server for the entire cluster. Runs as a Deployment (usually 2 replicas for HA) in kube-system namespace.

### What it does

When a pod inside the cluster does:

```bash
curl http://my-service
```

it needs to resolve my-service to an IP address.

CoreDNS handles that.

It knows about every Service in the cluster and returns the ClusterIP for that service name.

### DNS format in Kubernetes

```text
<service-name>.<namespace>.svc.cluster.local
```

Example:

```text
payment-service.banking.svc.cluster.local
```

Within the same namespace, just:

```text
payment-service
```

works.

### What happens if it dies

Internal DNS resolution breaks.

Pods cannot reach services by name — only by IP.

Almost all applications use service names, so this effectively breaks inter-service communication across the cluster.

### Interview question

**A pod cannot reach another service by name but can reach it by IP. What do you check?**

CoreDNS is the first suspect.

Check:

```bash
kubectl get pods -n kube-system | grep coredns
```

Test DNS:

```bash
kubectl exec -it <pod> -- nslookup <service-name>
```

Check logs:

```bash
kubectl logs -n kube-system <coredns-pod>
```

---

# metrics-server

### What it is

A lightweight component that collects CPU and memory usage from all nodes and pods in real time.

### What it does

Scrapes resource usage from kubelet on every node every 15 seconds.

Stores it in memory (not persistent).

Exposes it via the Kubernetes Metrics API.

### Why it matters

HPA (Horizontal Pod Autoscaler) depends entirely on metrics-server to know current CPU usage before deciding to scale.

These commands also depend on it:

```bash
kubectl top nodes

kubectl top pods
```

### What happens if it dies

kubectl top commands fail.

HPA stops scaling — it shows:

```text
<unknown>
```

in the TARGETS column.

Cluster still works but autoscaling is blind.

### Interview question

**HPA is not scaling even though CPU is high. What do you check first?**

Check:

```bash
kubectl get pods -n kube-system | grep metrics-server
```

Then:

```bash
kubectl get hpa
```

If TARGETS shows:

```text
<unknown>/50%
```

metrics-server is the problem.

---

# storage-provisioner (minikube specific)

### What it is

In minikube, this is a built-in component that automatically creates PersistentVolumes when a PersistentVolumeClaim is created.

In real AWS clusters, this role is played by the EBS CSI Driver.

### What it does

Watches for PVCs with no bound PV.

Creates a PV automatically on:

* Host filesystem (Minikube)
* Amazon EBS (AWS)

