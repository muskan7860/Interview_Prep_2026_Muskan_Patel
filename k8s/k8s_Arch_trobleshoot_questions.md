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
