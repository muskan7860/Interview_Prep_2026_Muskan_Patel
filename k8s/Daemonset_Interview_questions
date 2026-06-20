# DaemonSet — Interview Questions

> Level: 4 Years Experience
> Author: Muskan Patel
> Covers: Theory, Scenario, Troubleshooting

---

## HIGH PRIORITY — Almost Guaranteed to Be Asked

---

**Q1. What is a DaemonSet and why does it exist?**

A DaemonSet is a Kubernetes workload controller that ensures exactly
ONE pod runs on EVERY node in the cluster — always, automatically,
with zero manual intervention. It exists because some workloads
genuinely need to run on every node — log collectors that read log
files from each node's filesystem, monitoring agents that report
each node's CPU and memory metrics, CNI plugins that configure
networking on each node. A Deployment cannot solve this because its
scheduler placement is not guaranteed one-per-node — all replicas
could land on the same 3 nodes, leaving others uncovered. DaemonSet
solves this by tracking the node list directly: when a new node
joins, a pod is automatically created on it; when a node is removed,
its pod is automatically cleaned up.

---

**Q2. What is the difference between a DaemonSet and a Deployment with the same number of replicas as nodes?**

They look similar in pod count but behave completely differently.

A Deployment with `replicas: 10` on a 10-node cluster: the scheduler
places those 10 pods wherever there is capacity — possibly 8 on one
node and 1 each on two others, leaving 7 nodes with zero coverage.
You must manually update the replica count when nodes are added or
removed, or you end up with either gaps in coverage or wasted pods.

A DaemonSet: guaranteed exactly one pod per node, always. When a new
node joins the cluster, a pod is automatically created on it within
seconds — no human action needed. When a node is removed, its pod
is cleaned up automatically. The DaemonSet controller tracks the
node list directly and reacts to changes in it.

---

**Q3. Does DaemonSet bypass the Kubernetes Scheduler?**

In modern Kubernetes (version 1.12 onwards), no — not completely.
The DaemonSet controller pre-sets a `nodeAffinity` rule on each pod
it creates, locking that pod to a specific node. The Scheduler still
runs and processes the pod, but it has no real placement choice
because the nodeAffinity forces the destination. The Scheduler
technically rubber-stamps the decision the DaemonSet controller
already made.

In older Kubernetes (before 1.12), the DaemonSet controller DID
directly set `nodeName` on pods, completely bypassing the Scheduler.
This was changed because going through the Scheduler pipeline allows
admission controllers, resource checks, and other scheduling logic
to still apply.

The NET RESULT is the same — one pod per matching node — but the
mechanism changed. This is why you still see a "Scheduled" event
in `kubectl get events` for DaemonSet pods, unlike the old behavior
where no Scheduled event appeared.

---

**Q4. DaemonSet pods have more tolerations than regular pods. Why, and what tolerations do they get automatically?**

DaemonSet pods automatically get tolerations for node condition
taints that would normally evict regular pods. These are added by
the DaemonSet controller without you writing them in your YAML:

```
node.kubernetes.io/not-ready:NoExecute
node.kubernetes.io/unreachable:NoExecute
node.kubernetes.io/disk-pressure:NoSchedule
node.kubernetes.io/memory-pressure:NoSchedule
node.kubernetes.io/unschedulable:NoSchedule
node.kubernetes.io/pid-pressure:NoSchedule
```

The reason is intentional: a log collector or network agent MUST
run even on a node that is struggling. If a node is running out of
disk space, that is EXACTLY when you most need your monitoring agent
running — to see the disk pressure alerts. Regular app pods get
evicted when a node is under pressure, but DaemonSet pods tolerate
those conditions and stay running.

```bash
# Compare tolerations yourself
kubectl describe pod <daemonset-pod-name> | grep -A 20 Tolerations
kubectl describe pod <regular-pod-name> | grep -A 10 Tolerations
# DaemonSet pod has many more entries
```

---

**Q5. Does a DaemonSet have a `replicas` field?**

No. DaemonSet has no `replicas` field at all — the answer to "how
many pods?" is always "however many matching nodes currently exist
in the cluster." You cannot set a fixed number. If you want to
reduce coverage, you use `nodeSelector` to limit which nodes the
DaemonSet considers, not a replica count.

---

**Q6. What real-world components run as DaemonSets?**

Almost every node-level infrastructure component runs as a DaemonSet:

- CNI network plugins — Calico, Cilium, Flannel (pod networking)
- kube-proxy — Service traffic routing via iptables (in clusters
  not using eBPF-based CNI that replaces it)
- Log collectors — Fluentd, Fluent Bit, Filebeat
- Monitoring agents — node-exporter (Prometheus), Datadog agent
- Security agents — Falco (runtime security scanning)
- Storage drivers — CSI node plugins (AWS EBS CSI driver)

In your own KillerCoda cluster, `cilium` and `cilium-envoy` are
DaemonSets. On MicroK8s, `calico-node` is a DaemonSet. These are
why every node in your cluster has working pod networking — the CNI
DaemonSet ensures every node always has its own CNI agent configured.

---

## THEORY QUESTIONS

---

**Q7. What is `updateStrategy` in a DaemonSet and what are its two options?**

`updateStrategy` controls how existing pods on each node are updated
when you change the pod template (e.g., new image version).

`RollingUpdate` (default): updates one node at a time. With
`maxUnavailable: 1`, only ONE node's pod is taken down for update
at a time. All other nodes keep their old version running throughout.
Best for production — maintains service continuity across the cluster
during the update.

```yaml
updateStrategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1
```

`OnDelete`: pods are ONLY updated when you MANUALLY delete them.
The controller will NOT automatically replace old pods when the
template changes. Useful when you need full control over which
specific nodes get updated and when — for example, testing the new
version on one specific node before rolling it everywhere.

```bash
# With OnDelete: change the template -- nothing happens
kubectl set image ds/<name> <container>=<new-image>

# Manually delete the pod on a specific node to trigger update there
kubectl delete pod <daemonset-pod-on-node01>
# Only that node gets the new version -- others stay on old version
```

---

**Q8. What is `hostNetwork: true` in a DaemonSet pod, and why is it used?**

`hostNetwork: true` makes the container use the HOST node's network
namespace instead of its own isolated pod network. Normally,
containers get their own network namespace with a virtual network
interface. With `hostNetwork: true`, the container shares the actual
node's network interfaces (eth0, lo, etc.) directly.

Used by node-level monitoring agents (like node-exporter) that need
to see the HOST's actual network interfaces and traffic statistics —
from inside an isolated pod network, they would only see a virtual
interface, not the real host network.

Security consideration: significant privilege — the container can
bind to host ports, see all host network traffic, and interact with
the host's network stack. Only for trusted system-level DaemonSets,
never application pods.

---

**Q9. What is `hostPath` volume and why do DaemonSet pods commonly use it?**

`hostPath` mounts a directory from the HOST node's filesystem
directly into the container. It is the deliberate escape hatch from
container filesystem isolation.

DaemonSet pods for monitoring and logging commonly use it because:

- A log collector needs to read log files written to `/var/log` on
  the HOST by other processes — without hostPath, it can only see
  its own container's filesystem
- A metrics agent (node-exporter) needs to read `/proc` and `/sys`
  on the HOST to get real CPU, memory, and disk statistics — without
  hostPath, it only sees the container's limited/virtualized view of
  those filesystems

```yaml
volumeMounts:
- name: proc
  mountPath: /host/proc
  readOnly: true
volumes:
- name: proc
  hostPath:
    path: /proc
```

Security consideration: dangerous if pointed at sensitive directories
(/, /etc, /var/run/docker.sock). In banking environments, hostPath
usage is restricted via PodSecurity admission to only trusted system
DaemonSets — never application pods.

---

**Q10. How do you run a DaemonSet on control plane nodes as well as worker nodes?**

Control plane nodes have the taint
`node-role.kubernetes.io/control-plane:NoSchedule` which prevents
normal pods from being scheduled there. To allow a DaemonSet pod
onto control plane nodes, add a matching toleration to the pod
template:

```yaml
spec:
  template:
    spec:
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        effect: NoSchedule
        operator: Exists
```

Without this toleration — DaemonSet runs on worker nodes only,
control plane is excluded.
With this toleration — DaemonSet runs on ALL nodes including
control plane.

Real example: CNI plugins like Cilium and Calico add this toleration
because the control plane node ALSO needs pod networking configured.

```bash
kubectl describe daemonset cilium -n kube-system | grep -A 15 Tolerations
# You will see the control-plane toleration listed
```

---

**Q11. How do you limit a DaemonSet to run only on SPECIFIC nodes, not all nodes?**

Two approaches:

**nodeSelector (simple):**
```yaml
spec:
  template:
    spec:
      nodeSelector:
        gpu: "true"
```
DaemonSet only creates pods on nodes labeled `gpu: true`. All other
nodes are ignored. DESIRED count equals number of matching nodes.

```bash
kubectl label node node01 gpu=true
kubectl get daemonset <name>
# DESIRED: 1 (only node01 matches)
```

**nodeAffinity (flexible, same result but with more options):**
```yaml
spec:
  template:
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: gpu
                operator: In
                values:
                - "true"
```

---

## SCENARIO QUESTIONS

---

**Q12. SCENARIO: You deploy a log collection DaemonSet. Your cluster has 5 nodes. `kubectl get daemonset` shows `DESIRED: 5, CURRENT: 5, READY: 3`. What does this tell you and how do you investigate?**

DESIRED and CURRENT both showing 5 means the DaemonSet controller
successfully created pod OBJECTS on all 5 nodes — from the
controller's perspective, its job (one pod per node) is done. But
READY showing only 3 means 2 of those pods exist but are not passing
their readiness checks — they are likely in CrashLoopBackOff, Pending,
or Running-but-not-Ready state.

```bash
# Step 1 -- see individual pod statuses
kubectl get pods -l app=log-collector -o wide
# Look for pods with READY 0/1 or non-Running STATUS

# Step 2 -- describe the non-ready pods
kubectl describe pod <non-ready-pod-name>
# Read the Events section -- shows exact reason

# Step 3 -- check logs of a failing pod
kubectl logs <non-ready-pod-name>
kubectl logs <non-ready-pod-name> --previous
```

Common causes: the log collector needs access to a hostPath directory
that doesn't exist on those specific nodes, a permission issue
preventing the container from accessing the mounted directory, or
a resource constraint on those nodes preventing the container from
starting properly.

---

**Q13. SCENARIO: You add a new node to your cluster. After 5 minutes, you notice the DaemonSet pod for your monitoring agent has NOT appeared on the new node. Walk through your investigation.**

```bash
# Step 1 -- confirm the new node is visible and Ready
kubectl get nodes
# Is the new node listed? Is its STATUS Ready?

# Step 2 -- check if the DaemonSet DESIRED count increased
kubectl get daemonset <name>
# DESIRED should have increased by 1 when new node joined
# If DESIRED did NOT increase, the new node may not match
# the DaemonSet's nodeSelector labels

# Step 3 -- check nodeSelector on the DaemonSet
kubectl describe daemonset <name> | grep -A 5 "Node-Selector"
# If nodeSelector is set, check if new node has matching labels
kubectl describe node <new-node-name> | grep Labels

# Step 4 -- if DESIRED increased but pod is not appearing,
# check if a pod was created but is stuck
kubectl get pods -l app=<daemonset-label> -o wide
# Look for the new node in the NODE column

# Step 5 -- if pod exists but stuck, describe it
kubectl describe pod <new-pod-name>
```

Most common causes: new node does not have the label required by
the DaemonSet's `nodeSelector` (fix: label the node), new node has
a taint that the DaemonSet pod does not tolerate (fix: add
toleration), or new node is NotReady so DaemonSet controller is
waiting.

---

**Q14. SCENARIO: Your DaemonSet uses `updateStrategy: OnDelete`. You update the image. Three days later, a teammate asks "why are some nodes still on the old version?" Explain what happened and how to finish the update.**

With `OnDelete` strategy, updating the image in the DaemonSet spec
does NOT automatically replace any existing pods. Old pods on each
node continue running their original image indefinitely — they are
ONLY replaced when someone manually deletes them.

```bash
# See which pods are still on the old image
kubectl get pods -l app=<name> -o wide
# Then for each pod name you see:
kubectl describe pod <pod-name> | grep Image
# Shows which version each node is currently running

# To update a specific node's pod, delete its pod manually
kubectl delete pod <pod-name-on-node01>
# DaemonSet controller creates a replacement with the new image

# Repeat for each node you want to update
```

If you want to update ALL nodes immediately without this manual
process, switch the strategy to RollingUpdate, which will
automatically replace pods across all nodes.

---

**Q15. SCENARIO: A developer asks "can I use a DaemonSet to run my payment processing microservice so it scales automatically as we add more nodes?" Should they use a DaemonSet? What would you recommend instead?**

No — DaemonSet is the wrong tool for this. DaemonSet exists to run
infrastructure agents that need to be present on every node for
node-level reasons (networking, logging, monitoring). Using it for
an application microservice creates several problems:

- The pod count is tied to the node count, not to the application's
  actual traffic load. On a 20-node cluster you'd have 20 payment
  service pods regardless of whether you need 3 or 50.
- DaemonSet has no rolling update strategy with health gates — it
  doesn't wait for the new version to prove itself before continuing
  the update, unlike Deployment's readiness-probe-gated rolling update.
- DaemonSet has no HPA integration — you cannot scale based on CPU
  or request count, only based on node count.

The correct tool is a Deployment with HPA (Horizontal Pod
Autoscaler). Deployment gives rolling updates with readiness gates,
rollback history, and flexible replica counts. HPA automatically
adjusts the replica count based on actual traffic metrics. Adding
more nodes increases scheduling CAPACITY, and HPA then uses that
capacity to add more replicas if load demands it — the right
relationship between node count and pod count.

---

**Q16. SCENARIO: You need to add a DaemonSet for a security scanning agent to your banking project resume and explain it in an interview. What would you say?**

Resume entry: "Deployed a Falco security DaemonSet across all
worker and control plane nodes on AWS EKS to provide runtime
security monitoring. Configured hostPID and appropriate tolerations
to ensure coverage on all node types, including control plane nodes.
Set updateStrategy to RollingUpdate with maxUnavailable: 1 to
ensure continuous security coverage during agent updates."

Interview explanation: "We used a DaemonSet for the security
scanning agent because security monitoring is a node-level
responsibility — every node must be covered, with no exceptions or
gaps. A Deployment would have left some nodes unmonitored if the
scheduler placed all replicas on a subset of nodes. The DaemonSet
guaranteed one agent per node and automatically extended coverage
when new nodes joined the cluster as part of auto-scaling events.
We added the control-plane node toleration so the agent also
monitored the control plane, which is especially important in a
banking environment where any unexpected process on a control plane
node is a security concern."

---

## TROUBLESHOOTING QUESTIONS

---

**Q17. TROUBLESHOOTING: `kubectl get daemonset` shows `DESIRED: 3` but `CURRENT: 2`. One node is not getting a pod. What do you check?**

CURRENT being less than DESIRED means the DaemonSet controller
wants to create a pod on a node but something is preventing it.

```bash
# Step 1 -- check all nodes and their status
kubectl get nodes
# Is one node in NotReady state?
# A NotReady node will not receive new pods

# Step 2 -- check if the missing node has the required labels
kubectl describe daemonset <name> | grep -A 5 "Node-Selector"
kubectl describe node <problem-node> | grep Labels
# If nodeSelector is set and node lacks the label, add it:
kubectl label node <node-name> <key>=<value>

# Step 3 -- check if the node has a taint the DaemonSet cannot tolerate
kubectl describe node <problem-node> | grep Taints
kubectl describe daemonset <name> | grep -A 10 Tolerations
# If node has a taint not tolerated, add toleration to DaemonSet spec

# Step 4 -- check if a pod was created but failed immediately
kubectl get pods -l app=<name> -o wide
# If pod exists on that node but shows Error or CrashLoopBackOff
kubectl describe pod <pod-name>
kubectl logs <pod-name>

# Step 5 -- check resource availability on that specific node
kubectl describe node <problem-node> | grep -A 8 "Allocated resources"
# If node is fully allocated, DaemonSet pod cannot be placed even
# though DESIRED count says it should be there
```

---

**Q18. TROUBLESHOOTING: After upgrading a DaemonSet's image using `kubectl set image`, you run `kubectl rollout status daemonset/<name>` and it hangs at "1 out of 3 new pods have been updated." What is happening and what do you do?**

With `RollingUpdate` and `maxUnavailable: 1`, the DaemonSet updates
ONE node at a time and waits for that node's new pod to become Ready
before moving to the next node. If it hangs, the new pod on the
first updated node is not becoming Ready — the controller is blocking
on it rather than continuing to break more nodes.

```bash
# Step 1 -- find which pod is the new one (different age)
kubectl get pods -l app=<name> -o wide

# Step 2 -- describe the new pod to see why it is not Ready
kubectl describe pod <new-pod-name>
# Check Events for: ImagePullBackOff, CrashLoopBackOff,
# readiness probe failing, resource constraints

# Step 3 -- check logs of the new pod
kubectl logs <new-pod-name>
kubectl logs <new-pod-name> --previous

# Step 4 -- if the new image is broken, rollback immediately
kubectl rollout undo daemonset/<name>
kubectl rollout status daemonset/<name>
# All nodes return to the previous version
```

The blocking behavior is actually PROTECTIVE — a broken update on
one node stops automatically rather than propagating to all nodes.
This is why `maxUnavailable: 1` is the right default for production
DaemonSets: worst case, ONE node gets the broken version and then
the rollout stops, leaving the other nodes healthy.

---

**Q19. TROUBLESHOOTING: A DaemonSet pod is in CrashLoopBackOff on one specific node but Running on all other nodes. The image and config are identical. What is your investigation approach?**

Since the issue is node-specific (same DaemonSet, same image, one
node fails) the problem is almost certainly something about THAT
specific node, not the pod configuration itself.

```bash
# Step 1 -- identify which node has the failing pod
kubectl get pods -l app=<name> -o wide
# Note the node name for the CrashLoopBackOff pod

# Step 2 -- check logs of the failing pod
kubectl describe pod <failing-pod-name>
kubectl logs <failing-pod-name>
kubectl logs <failing-pod-name> --previous
# Read the error carefully -- what is it trying to access?

# Step 3 -- check node-specific resources
# If the DaemonSet uses hostPath volumes:
kubectl describe pod <failing-pod-name> | grep -A 5 Volumes
# Does that directory exist on the failing node?
# SSH to that node and check:
# ls /var/log/containers   (example hostPath)

# Step 4 -- check node health
kubectl describe node <problem-node>
# DiskPressure? MemoryPressure? Any condition True that is not True
# on other nodes?

# Step 5 -- check node-specific kubelet config or kernel version
# Sometimes a node was added with different configuration
# (different OS version, different kernel, missing kernel module)
# that causes a specific container/agent to fail only there
```

---

**Q20. TROUBLESHOOTING: You accidentally deleted a DaemonSet. You immediately realize your cluster now has NO CNI networking (because the CNI was a DaemonSet). Pods cannot communicate. How do you recover?**

Deleting a DaemonSet cascades to all its pods by default. If the
CNI DaemonSet is deleted, all CNI pods across all nodes are removed,
breaking pod-to-pod networking cluster-wide — but running pods
themselves are NOT deleted (their containers keep running, just
network connectivity breaks for new connections).

```bash
# Step 1 -- immediate: re-apply the CNI DaemonSet YAML
# (this is why you should always have your infrastructure
# manifests stored in Git)
kubectl apply -f cilium-daemonset.yaml
# OR for Calico:
kubectl apply -f calico-daemonset.yaml

# Step 2 -- watch the DaemonSet come back
kubectl get daemonset -n kube-system -w

# Step 3 -- verify pods come up on all nodes
kubectl get pods -n kube-system -o wide | grep cilium
# Should show one pod per node returning to Running

# Step 4 -- verify networking is restored
kubectl run test --image=busybox -it --rm -- ping <pod-ip>

# Prevention:
# 1. RBAC -- restrict who can delete DaemonSets in kube-system
# 2. Store all infrastructure manifests in Git
# 3. Use kubectl delete --dry-run=client before deleting anything
#    in kube-system
# 4. Add a ResourceLock or admission webhook that blocks deletion
#    of critical namespace objects without approval
```

---

*Run Labs 1, 2, and 3 before interviews. Lab 2 especially --
the comparison of DaemonSet pod tolerations vs regular pod
tolerations is a real differentiator in interviews because most
candidates only know the "one pod per node" definition but not the
WHY behind the extra tolerations.*
