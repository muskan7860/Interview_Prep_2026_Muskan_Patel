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
