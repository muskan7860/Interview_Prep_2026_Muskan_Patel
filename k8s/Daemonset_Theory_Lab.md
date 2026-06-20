# DaemonSet — Theory, Labs, and Deep Dive Explanation

> Level: 4 Years Experience
> Author: Muskan Patel

---

## PART 1 — THEORY

---

## 1. Why DaemonSet Exists — The Problem None of the Others Solve

Every workload type you learned so far answers the same question:
**"How MANY copies of my app should run?"**

- Pod — just runs one thing, no self-healing
- ReplicaSet — keeps N identical pods running somewhere in the cluster
- Deployment — manages rolling updates between ReplicaSet versions
- StatefulSet — like ReplicaSet but with stable identity and storage

You give a replica count. Kubernetes places those copies wherever
there is capacity.

Now here is a completely different problem.

You are running a 10-node Kubernetes cluster. You need a log
collection agent — something that reads log files being written on
each node and ships them to your central Elasticsearch server. Where
do you run it?

**Option 1 — Use a Deployment with `replicas: 10`:**
The scheduler might place all 10 pods on the same 3 nodes (the ones
with most free capacity), leaving 7 nodes with ZERO log collection.
Those 7 nodes' logs are never collected. When an 11th node is added
tomorrow, you must manually remember to scale to `replicas: 11`.

**Option 2 — Use a DaemonSet:**
The DaemonSet controller guarantees exactly ONE pod on EVERY node —
always, automatically, with zero manual intervention. Add a node →
pod appears automatically. Remove a node → its pod is cleaned up
automatically. No replica count to manage. No human memory required.

DaemonSet was built to solve ONE specific problem:
**"I need exactly ONE instance of this thing running on every node,
permanently."**

---

## 2. The Security Guard Analogy

Think of a large office building (your cluster) with multiple floors
(nodes). Management decides: every floor must have exactly ONE
security guard on duty at all times, 24/7.

The DaemonSet is the SECURITY MANAGEMENT SYSTEM that enforces this:

- New floor added → management system automatically assigns a guard
- A guard gets sick (pod crashes) → replacement sent immediately
- A floor is closed (node removed) → that position automatically eliminated
- The rule is ALWAYS: one guard per floor

You NEVER specify "I want 7 guards" — the answer is always
"however many floors exist right now."

---

## 3. What You Have Already Been Using as DaemonSets

Before you even knew what a DaemonSet was, you were already USING
things that run as DaemonSets every single day.

```bash
kubectl get daemonsets -n kube-system
```

On KillerCoda (Cilium-based cluster):
```
NAME           DESIRED   CURRENT   READY
cilium         2         2         2      <- CNI network plugin
cilium-envoy   2         2         2      <- Cilium's Envoy proxy
```

On MicroK8s:
```
calico-node
```

These are DaemonSets. This is WHY:
- Every node has pod networking working → because `cilium` or
  `calico-node` DaemonSet puts a CNI pod on EVERY node to configure
  networking
- Every node routes Service traffic correctly → because the CNI
  DaemonSet handles this on every node

If these were Deployments instead of DaemonSets, you might have
`cilium` pods on 3 out of 10 nodes, and 7 nodes would have NO
networking — catastrophic.

---

## 4. What Happens When You Apply a DaemonSet — Step by Step

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-collector
spec:
  selector:
    matchLabels:
      app: log-collector
  template:
    metadata:
      labels:
        app: log-collector
    spec:
      containers:
      - name: fluentd
        image: fluent/fluentd:v1.16
```

1. API server stores the DaemonSet object in etcd.

2. The DaemonSet Controller (a dedicated controller inside
   controller-manager — separate from ReplicaSet and Deployment
   controllers) wakes up.

3. Unlike all other controllers, it does NOT ask "how many pods
   should I create?" It asks: "Which nodes exist in this cluster
   right now?"

4. For EACH node, it checks: "Does a pod from THIS DaemonSet already
   exist on this node?" If NO → create one. If YES → do nothing.

5. For each pod it creates, the DaemonSet controller automatically:
   - Adds a `nodeAffinity` rule to the pod spec saying "this pod
     MUST run on node X specifically"
   - Adds a set of tolerations so the pod can run even on nodes
     under pressure (disk pressure, memory pressure, etc.)
   - Sends this pod spec to the API server

6. The Scheduler processes each pod. It reads the pre-set nodeAffinity,
   sees "must go to node X," and assigns it there — the Scheduler
   technically runs but has no real choice in placement.

7. The controller now watches BOTH the pod list AND the node list.
   When EITHER changes, it reacts:
   - Pod disappears → recreate it on that same node immediately
   - New node added → create a pod on the new node automatically
   - Node removed → its pod is garbage collected automatically

---

## 5. How DaemonSet Scheduling Works — The Accurate Modern Picture

### What People Often Say (Oversimplified)

"DaemonSet bypasses the Scheduler by setting nodeName directly on
the pod."

### What Actually Happens in Modern Kubernetes (1.12+)

In older Kubernetes (before 1.12), DaemonSet DID directly set
`nodeName` and completely bypass the Scheduler. This was changed
in 1.12 for a better approach.

In modern Kubernetes, the DaemonSet controller instead:

1. Creates a pod with a `nodeAffinity` rule pre-baked into it:
```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchFields:
        - key: metadata.name
          operator: In
          values:
          - node01            # locked to THIS specific node
```

2. Automatically adds MANY tolerations to the pod (more than
   regular pods get):
```yaml
tolerations:
- key: node.kubernetes.io/not-ready
  operator: Exists
  effect: NoExecute
- key: node.kubernetes.io/unreachable
  operator: Exists
  effect: NoExecute
- key: node.kubernetes.io/disk-pressure
  operator: Exists
  effect: NoSchedule
- key: node.kubernetes.io/memory-pressure
  operator: Exists
  effect: NoSchedule
- key: node.kubernetes.io/unschedulable
  operator: Exists
  effect: NoSchedule
```

3. The Scheduler reads this pod, sees the nodeAffinity forcing it
   to node01, and assigns it there — it has no real choice.

### Regular Pod vs DaemonSet Pod — Key Differences

| | Regular Pod | DaemonSet Pod |
|---|---|---|
| nodeAffinity | None (unless you wrote it) | Pre-set to a specific node by DaemonSet controller |
| Tolerations | 2 (not-ready, unreachable for 300s) | Many — including disk-pressure, memory-pressure, unschedulable |
| Scheduler choice | Full freedom — picks best node | Rubber stamp — locked by pre-set nodeAffinity |
| Survives node under pressure? | No — evicted when node has disk/memory pressure | Yes — tolerations allow it to stay |

### Why DaemonSet Pods Get Extra Tolerations

This is important and interviewers ask about it.

A log collector or network agent MUST run even on a node that is
struggling — in fact, those are the MOST important times to have
your monitoring agent running. If a node is running out of disk space,
you NEED your log collector to still be working so you can SEE the
disk pressure alerts. Regular app pods get evicted under node pressure,
but DaemonSet pods tolerate those conditions and stay running.

### What to Say in an Interview

"In modern Kubernetes (1.12+), the DaemonSet controller does not
completely bypass the Scheduler — it pre-sets a nodeAffinity on
each pod specifying exactly which node it must go to, and
automatically adds tolerations for all node condition taints so
the pod can run even on nodes under pressure. The Scheduler still
runs but has no real placement choice because the affinity locks the
destination. DaemonSet pods also automatically get more tolerations
than regular pods — disk-pressure, memory-pressure, unschedulable —
which is intentional, because you always want your networking and
monitoring agents running even on struggling nodes."

---

## 6. Tolerations for Control Plane Nodes

Control plane nodes have a taint:
```
node-role.kubernetes.io/control-plane:NoSchedule
```

This prevents normal pods from being scheduled on control plane nodes
— keeping resources free for etcd, API server, scheduler, etc.

But some DaemonSets (like CNI plugins) NEED to run on the control
plane node too — because the control plane node ALSO needs networking.

Solution: add a toleration to the DaemonSet's pod template:

```yaml
spec:
  template:
    spec:
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        effect: NoSchedule
        operator: Exists
```

Without this: DaemonSet runs on worker nodes only.
With this: DaemonSet runs on EVERY node including control plane.

```bash
# See this on your cluster -- cilium tolerates control-plane taint
kubectl describe daemonset cilium -n kube-system | grep -A 15 Tolerations
```

---

## 7. nodeSelector — Running on SPECIFIC Nodes Only

Sometimes you do not want the DaemonSet on EVERY node — only on
nodes with specific hardware or roles.

```yaml
spec:
  template:
    spec:
      nodeSelector:
        gpu: "true"
```

This DaemonSet only creates pods on nodes labeled `gpu: true`. The
"one per node" rule becomes "one per MATCHING node."

Real use case: an NVIDIA GPU monitoring agent that only needs to run
on GPU nodes, not on regular CPU worker nodes.

```bash
kubectl label node node01 gpu=true
# DaemonSet now creates a pod ONLY on node01, ignores others
```

---

## 8. updateStrategy — How DaemonSet Updates Work

When you update the pod template (new image), how should existing
pods on each node be updated?

```yaml
spec:
  updateStrategy:
    type: RollingUpdate       # default
    rollingUpdate:
      maxUnavailable: 1
```

**RollingUpdate (default):** updates one node at a time. With
`maxUnavailable: 1`, only ONE node's pod is taken down for update
at a time. All other nodes keep their old version running. Best
for production — maintains service continuity across the cluster.

**OnDelete:** pods are ONLY updated when you MANUALLY delete them.
The controller will NOT automatically replace old pods on update.
Useful when you need full control over update timing per node
(e.g., you want to test the new version on one specific node before
rolling it out everywhere).

```bash
# With OnDelete strategy:
kubectl set image ds/<name> <container>=<new-image>
# Nothing happens automatically

# You manually delete a specific node's pod to trigger update
kubectl delete pod <daemonset-pod-on-specific-node>
# Now THAT node gets the new version -- others still old
```

---

## 9. Line by Line — Full DaemonSet YAML Explained

```yaml
apiVersion: apps/v1
# WHAT: workload controller API group
# WHY apps/v1: DaemonSet is a workload controller, same API group as
# ReplicaSet, Deployment, StatefulSet.

kind: DaemonSet
# WHAT: tells API server this is a DaemonSet
# WHY: uses the DaemonSet Controller which has "one per node" logic,
# NOT the ReplicaSet Controller (N copies anywhere) or Deployment
# Controller (version transitions). Each controller is separate
# code with separate responsibilities.

metadata:
  name: node-exporter
  namespace: monitoring
# WHAT: standard object identity
# WHY namespace: good practice to put monitoring tools in a dedicated
# namespace, not default or kube-system

spec:
  selector:
    matchLabels:
      app: node-exporter
  # WHAT: the label query used to find pods this DaemonSet manages
  # WHY: same as ReplicaSet/Deployment -- the controller uses this
  # to check "does a pod from me already exist on this node?"
  # before deciding to create one. Without this, it would create
  # duplicate pods every time the controller loop runs.

  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  # WHAT: how to handle pod template updates
  # WHY maxUnavailable: 1 -- only one node loses its monitoring pod
  # at a time during an update. All other nodes keep collecting
  # metrics. This prevents a blind spot in monitoring during updates.

  template:
    metadata:
      labels:
        app: node-exporter
    spec:

      tolerations:
      - key: node-role.kubernetes.io/control-plane
        effect: NoSchedule
        operator: Exists
      # WHAT: allows this pod to be placed on control plane nodes
      # even though they have a NoSchedule taint
      # WHY: you want to monitor ALL nodes including control plane.
      # Without this toleration, node-exporter would only run on
      # worker nodes, giving you a blind spot in metrics for the
      # control plane node.

      hostNetwork: true
      # WHAT: container uses the HOST node's network namespace
      # instead of its own isolated pod network
      # WHY: node-exporter needs to see the HOST's network
      # interfaces (eth0, lo, etc.) to report the node's actual
      # network metrics. From inside an isolated pod network namespace,
      # it would only see a virtual network interface, not the real
      # host network.
      # SECURITY NOTE: significant privilege. Container can bind
      # to host ports, see all host network traffic. Only for
      # trusted system-level agents, NEVER application pods.

      hostPID: true
      # WHAT: container can see ALL processes running on the HOST
      # (not just processes inside its own container)
      # WHY: node-exporter needs this to report per-process CPU
      # and memory statistics for ALL processes on the node.
      # Without hostPID, it can only see its own container's process.
      # SECURITY NOTE: significant privilege. A compromised container
      # with hostPID can inspect and potentially signal any process
      # on the host. Banking environments typically restrict this
      # via PodSecurity admission policy.

      containers:
      - name: node-exporter
        image: prom/node-exporter:latest

        ports:
        - containerPort: 9100
          hostPort: 9100
          # WHAT: makes port 9100 accessible on the HOST node's
          # actual IP address
          # WHY: Prometheus (running elsewhere in the cluster) can
          # scrape metrics by hitting each node's IP on port 9100
          # directly, without needing to create a Service object.
          # Each node becomes directly scrapable at its own IP:9100.
          # This is a common pattern for node-level monitoring agents.

        resources:
          requests:
            cpu: "100m"
            memory: "50Mi"
          limits:
            cpu: "200m"
            memory: "100Mi"
        # WHAT: resource reservation and cap for this container
        # WHY requests: even though DaemonSet handles node placement
        # directly, the DaemonSet controller still checks whether a
        # node has enough capacity for these requests before placing
        # a pod there. A node with ZERO free capacity will not get
        # a DaemonSet pod even if the DaemonSet wants to place one.
        # WHY limits: prevents the monitoring agent from consuming
        # too many resources on the node it's monitoring. A runaway
        # monitoring agent that starves the actual application pods
        # would defeat its own purpose.

        volumeMounts:
        - name: proc
          mountPath: /host/proc
          readOnly: true
        - name: sys
          mountPath: /host/sys
          readOnly: true
        # WHAT: mounts the host's /proc and /sys directories into
        # the container at /host/proc and /host/sys
        # WHY: node-exporter reads CPU, memory, disk, network stats
        # from these Linux virtual filesystems. From inside an
        # isolated container, you see a limited/virtualized view of
        # /proc and /sys. By mounting the HOST's actual /proc and
        # /sys into the container, node-exporter sees REAL host-level
        # statistics, not the container's sandboxed view.
        # readOnly: true -- the agent only needs to READ these
        # filesystems, never write. Mounting read-only limits the
        # blast radius if the container is compromised.

      volumes:
      - name: proc
        hostPath:
          path: /proc
      - name: sys
        hostPath:
          path: /sys
      # WHAT: defines what the volumeMounts above actually point to
      # WHY hostPath: mounts a directory from the HOST node's
      # filesystem directly into the container. This is the deliberate
      # escape hatch from container filesystem isolation -- used when
      # a container genuinely needs access to the host's files.
      # The hostPath volume type maps /proc on the HOST to whatever
      # mountPath you specified in volumeMounts.
      # SECURITY NOTE: hostPath is dangerous if pointed at sensitive
      # directories (/, /etc, /var/run/docker.sock). In production
      # banking clusters, hostPath usage is restricted via PodSecurity
      # admission to only trusted system DaemonSets, never
      # application pods.
```

---

## 10. DaemonSet vs Other Workload Types — When to Use Which

| Need | Use | Why |
|---|---|---|
| Run N copies of app somewhere in cluster | Deployment | Replica count, rolling updates, rollback |
| Run N copies with stable identity and storage | StatefulSet | Per-pod PVC, ordinal names, ordered startup |
| Run on EVERY node, one copy each | DaemonSet | No replica count, auto-adjusts as nodes join/leave |
| Run once to completion | Job | Tracks success/failure, retries |
| Run on a schedule | CronJob | Job + cron schedule |

Real DaemonSet use cases in industry:
- **Log collection** — Fluentd, Fluent Bit, Filebeat
- **Monitoring agents** — node-exporter (Prometheus), Datadog agent
- **Security scanning** — Falco (runtime security), Aqua Security agent
- **CNI plugins** — Calico, Cilium, Flannel (pod networking)
- **Storage drivers** — CSI node plugins (AWS EBS CSI, NFS provisioner)
- **Service mesh sidecars** — in DaemonSet mode for some mesh implementations

---

## PART 2 — HANDS-ON LABS

---

## Lab 1 — Create a Simple DaemonSet and Watch One-Per-Node

```bash
kubectl apply -f - << 'EOF'
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: simple-daemon
  namespace: default
spec:
  selector:
    matchLabels:
      app: simple-daemon
  template:
    metadata:
      labels:
        app: simple-daemon
    spec:
      containers:
      - name: busybox
        image: busybox:1.35
        command:
        - sh
        - -c
        - |
          echo "I am running on node: $NODE_NAME"
          sleep 3600
        env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        resources:
          requests:
            cpu: "10m"
            memory: "10Mi"
EOF

# Check how many nodes you have
kubectl get nodes

# Check pods -- EXACTLY ONE per node, no more, no less
kubectl get pods -l app=simple-daemon -o wide
# NODE column shows a different node for EACH pod
# Total pod count = total node count

# DESIRED matches number of nodes exactly
kubectl get daemonset simple-daemon
# DESIRED = number of nodes in your cluster

# See what the pod logs -- it knows its own node name
kubectl logs $(kubectl get pods -l app=simple-daemon \
  -o jsonpath='{.items[0].metadata.name}')
# "I am running on node: node01"

kubectl delete daemonset simple-daemon
```

---

## Lab 2 — Understand How DaemonSet Scheduling Actually Works

This lab shows you the REAL mechanism — nodeAffinity pre-set by
the DaemonSet controller and extra tolerations added automatically.

```bash
kubectl apply -f - << 'EOF'
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: scheduler-proof
spec:
  selector:
    matchLabels:
      app: scheduler-proof
  template:
    metadata:
      labels:
        app: scheduler-proof
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
EOF

kubectl get pods -l app=scheduler-proof

# CHECK 1 -- nodeAffinity is pre-set by DaemonSet controller
# (you never wrote this in your YAML)
kubectl get pod $(kubectl get pods -l app=scheduler-proof \
  -o jsonpath='{.items[0].metadata.name}') -o yaml | grep -A 15 affinity
# You will see a nodeAffinity locking this pod to a specific node
# A regular pod you create has NO affinity unless you explicitly add it

# CHECK 2 -- extra tolerations DaemonSet adds automatically
kubectl get pod $(kubectl get pods -l app=scheduler-proof \
  -o jsonpath='{.items[0].metadata.name}') -o yaml | grep -A 30 tolerations
# You will see MANY tolerations including:
# disk-pressure, memory-pressure, unschedulable, pid-pressure
# These are added AUTOMATICALLY -- you never wrote them

# CHECK 3 -- compare with a regular pod's tolerations
kubectl run test-pod --image=nginx:1.25
kubectl get pod test-pod -o yaml | grep -A 15 tolerations
# Only 2 tolerations (not-ready and unreachable)
# DaemonSet pod has many more -- this is the KEY difference

# CHECK 4 -- events (Scheduler DOES run but has no real choice)
kubectl get events | grep scheduler-proof
# You WILL see "Scheduled" event -- but it was forced by the
# pre-set nodeAffinity, not a free Scheduler decision

kubectl delete daemonset scheduler-proof
kubectl delete pod test-pod
```

**What this lab proves:**
The DaemonSet controller does NOT completely bypass the Scheduler in
modern Kubernetes. Instead it pre-bakes a nodeAffinity to force
placement on a specific node AND automatically adds many more
tolerations than a regular pod gets. The Scheduler technically
runs but has no real choice. The tolerations are the other key
difference -- DaemonSet pods survive on unhealthy nodes where
regular pods would be evicted.

---

## Lab 3 — Explore Real DaemonSets in Your Cluster

### On KillerCoda (Cilium cluster)

```bash
# List all DaemonSets in kube-system
kubectl get daemonsets -n kube-system
# cilium         <- CNI network plugin (replaces kube-proxy too)
# cilium-envoy   <- Cilium's Envoy-based proxy

# Describe cilium DaemonSet -- note the key fields
kubectl describe daemonset cilium -n kube-system
# Look at:
# Desired Number of Nodes Scheduled = 2 (matches your node count)
# Node-Selector: kubernetes.io/os=linux
# Tolerations: many -- including control-plane ones
# Volumes: hostPath mounts for host-level access

# See one cilium pod per node
kubectl get pods -n kube-system -l k8s-app=cilium -o wide
# One pod on controlplane, one pod on node01

# Check the nodeAffinity pre-set on a cilium pod
kubectl get pod \
  $(kubectl get pods -n kube-system -l k8s-app=cilium \
  -o jsonpath='{.items[0].metadata.name}') \
  -n kube-system -o yaml | grep -A 15 affinity

# Check all the tolerations on a cilium pod
kubectl get pod \
  $(kubectl get pods -n kube-system -l k8s-app=cilium \
  -o jsonpath='{.items[0].metadata.name}') \
  -n kube-system -o yaml | grep -A 30 tolerations
```

### On MicroK8s

```bash
# List DaemonSets
kubectl get daemonsets -n kube-system
# calico-node

# Describe it
kubectl describe daemonset calico-node -n kube-system

# See pods
kubectl get pods -n kube-system -l k8s-app=calico-node -o wide
```

---

## Lab 4 — Rolling Update of a DaemonSet

```bash
kubectl apply -f - << 'EOF'
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: update-demo
spec:
  selector:
    matchLabels:
      app: update-demo
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  template:
    metadata:
      labels:
        app: update-demo
    spec:
      containers:
      - name: nginx
        image: nginx:1.24
EOF

kubectl get pods -l app=update-demo -o wide

# Verify initial image
kubectl get pods -l app=update-demo \
  -o jsonpath='{range .items[*]}{.metadata.name}{"  "}{.spec.containers[0].image}{"\n"}{end}'
# All show nginx:1.24

# Update the image
kubectl set image daemonset/update-demo nginx=nginx:1.25

# Watch the rolling update -- one node at a time
kubectl rollout status daemonset/update-demo
# Waiting for daemon set "update-demo" rollout to finish:
# 1 out of 2 new pods have been updated...
# 2 out of 2 new pods have been updated...
# daemon set "update-demo" successfully rolled out

# Verify all pods are now on new image
kubectl get pods -l app=update-demo \
  -o jsonpath='{range .items[*]}{.metadata.name}{"  "}{.spec.containers[0].image}{"\n"}{end}'
# All show nginx:1.25

# Rollback if needed
kubectl rollout undo daemonset/update-demo
kubectl rollout status daemonset/update-demo

kubectl delete daemonset update-demo
```

---

## Lab 5 — nodeSelector: Run on Specific Nodes Only

```bash
# First see your nodes
kubectl get nodes

# Label only ONE node
kubectl label node node01 role=monitored

kubectl apply -f - << 'EOF'
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: selective-daemon
spec:
  selector:
    matchLabels:
      app: selective-daemon
  template:
    metadata:
      labels:
        app: selective-daemon
    spec:
      nodeSelector:
        role: monitored
      containers:
      - name: nginx
        image: nginx:1.25
EOF

kubectl get pods -l app=selective-daemon -o wide
# ONLY on node01 -- other nodes are completely ignored
# Even though DaemonSet "wants one per node",
# nodeSelector limits "per matching node"

kubectl get daemonset selective-daemon
# DESIRED = 1 (only 1 node matches the label)

# Now label another node -- DaemonSet reacts automatically
kubectl label node controlplane role=monitored

sleep 5
kubectl get pods -l app=selective-daemon -o wide
# NOW on both node01 AND controlplane
# DaemonSet detected the new matching node and created a pod on it
# automatically -- zero manual intervention

kubectl get daemonset selective-daemon
# DESIRED = 2 now (2 nodes match the label)

# Cleanup
kubectl delete daemonset selective-daemon
kubectl label node node01 role-
kubectl label node controlplane role-
```

---

## Quick Reference — DaemonSet Commands

```bash
# Get
kubectl get daemonsets
kubectl get ds                          # short form
kubectl get ds -n kube-system

# Inspect
kubectl describe ds <name>
kubectl describe ds <name> -n kube-system

# Update image
kubectl set image ds/<name> <container>=<new-image>

# Rollout management (same as Deployment)
kubectl rollout status ds/<name>
kubectl rollout history ds/<name>
kubectl rollout undo ds/<name>

# Scale (rarely used -- count is automatic)
# DaemonSet does not have a replicas field
# To reduce where it runs, use nodeSelector or node labels

# Delete
kubectl delete ds <name>
```

---

## Summary — What Makes DaemonSet Unique

1. NO `replicas` field -- answer is always "one per matching node"
2. Automatically reacts to nodes joining or leaving the cluster
3. Pre-sets nodeAffinity on each pod (modern K8s 1.12+) -- Scheduler
   runs but has no real placement choice
4. Automatically adds MORE tolerations than regular pods -- survives
   on nodes under disk/memory pressure where regular pods get evicted
5. Commonly uses hostPath volumes, hostNetwork, hostPID for
   node-level access -- significant privileges, only for trusted
   system agents
6. Used for: CNI plugins, log collectors, monitoring agents, security
   scanners, storage drivers -- anything that needs to run on every node

---

*Next: Job*
