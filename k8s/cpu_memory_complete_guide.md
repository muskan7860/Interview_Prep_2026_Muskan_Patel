# CPU, Memory, and Memory Leaks in Kubernetes — Complete Guide

> Level: 4 Years Experience
> Author: Muskan Patel

---

## 1. The Hotel Room Analogy

A hotel manager (Scheduler) places groups (Pods) into rooms (Nodes).

Each group tells the manager two numbers:
- "We NEED at least 2 rooms" → this is **requests**
- "We will NEVER use more than 4 rooms" → this is **limits**

The manager uses the NEED number to decide WHERE to place the group
— this is a planning/reservation decision made ONCE. The LIMIT
number is only checked AFTER the group is already inside the hotel —
this is an ongoing enforcement rule.

---

## 2. requests — Used ONLY by the Scheduler

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "128Mi"
```

### What requests means

- `250m` = 0.25 of one CPU core (m = millicores, 1000m = 1 core)
- `128Mi` = 128 Mebibytes of RAM

### How it works internally

For EVERY node, the scheduler calculates:

```
free_for_scheduling = node.Allocatable - sum(requests of pods already on this node)
```

If `free_for_scheduling >= this pod's requests` → that node becomes
a CANDIDATE for this pod.

### The Critical Trick — Reservation, NOT Real Usage

```
10 pods are running on a node
Each pod REQUESTED 100m CPU → total RESERVED = 1000m (1 full core)

But in REALITY, all 10 pods together are ACTUALLY USING only 50m CPU
(they are mostly idle)

The scheduler STILL sees "1000m reserved" on this node.
If the node has only 1 CPU total and a new pod arrives wanting
100m CPU, the scheduler says "NO ROOM" — even though 950m is
ACTUALLY free right now, just not "reserved."
```

```bash
# RESERVATION view (based on requests)
kubectl describe node <node-name> | grep -A 5 "Allocated resources"

# Output:
#   Resource           Requests      Limits
#   cpu                1000m (50%)   2000m (100%)
#   memory             2Gi (40%)     4Gi (90%)

# ACTUAL usage view (real-time)
kubectl top nodes
# Often shows MUCH lower numbers than "Allocated resources"
```

### Consequence of setting requests too HIGH

If requests are set far above what the app actually needs, nodes
will appear "full" to the scheduler much sooner than they're
actually full by real usage. New pods cannot be scheduled even
though plenty of real capacity exists — it's just not "unreserved."
This wastes cluster capacity and is a very common real-world issue.

### Consequence of setting requests too LOW

The pod might get scheduled onto an already-busy node. When the app
actually needs CPU/memory, it has to FIGHT other pods for the same
physical resources — causing slowdowns under load even though
nothing "crashed."

---

## 3. limits — Enforced by the Linux Kernel, Continuously

```yaml
resources:
  limits:
    cpu: "500m"
    memory: "256Mi"
```

`limits` has NOTHING to do with scheduling. It is a runtime
restriction enforced AFTER the pod is already running, by Linux
cgroups.

---

## 4. CPU Limit Exceeded — Throttling

### The Speed Limiter Analogy

```
A car CAN physically go 200 km/h.
A speed limiter caps it at 80 km/h.
The car still RUNS — just slower than it "wants" to go.
```

CPU is a "compressible" resource — Linux can give a process LESS CPU
time over time WITHOUT destroying it. When a container hits its CPU
`limits`, the cgroup CFS (Completely Fair Scheduler) quota mechanism
caps how much CPU time the process gets per scheduling period
(typically every 100ms). The app does NOT crash — it just runs
SLOWER (higher latency, slower response times).

```bash
# See throttling happen
kubectl run cpu-demo --image=polinux/stress \
  --requests=cpu=100m --limits=cpu=200m \
  -- stress --cpu 1

kubectl top pod cpu-demo
# CPU usage stays capped near 200m, never higher
# Pod stays Running — just throttled

kubectl delete pod cpu-demo
```

### How to detect CPU throttling in production

```bash
# Check the container's cgroup throttling stats directly
kubectl exec <pod> -- cat /sys/fs/cgroup/cpu/cpu.stat
# Look at "nr_throttled" (how many times throttled)
# and "throttled_time" (total time spent throttled)

# A non-zero, growing nr_throttled means the app is
# regularly hitting its CPU limit
```

---

## 5. Memory Limit Exceeded — OOMKilled

### Why Memory Is Different From CPU

```
Memory cannot be "throttled" the way CPU can.
Either the memory exists, or it doesn't — there's no "half-allocated"
state.

So when a container tries to use MORE memory than its limit, the
Linux kernel says: "This process is asking for memory I promised it
CANNOT have. I must KILL it immediately to protect the rest of the
system."

The kernel sends SIGKILL → process dies INSTANTLY.
Kubernetes reports this as:
  Reason: OOMKilled
  Exit Code: 137
```

```bash
# See this happen live
kubectl run memory-test --image=polinux/stress \
  --requests=memory=50Mi \
  --limits=memory=50Mi \
  -- stress --vm 1 --vm-bytes 100M --vm-hang 1

kubectl get pod memory-test -w
# Status: Running → Error/OOMKilled

kubectl describe pod memory-test | grep -A 3 "Last State"
# Reason: OOMKilled
# Exit Code: 137

kubectl delete pod memory-test
```

### Why exit code 137?

137 = 128 + 9. The "128 +" prefix means "killed by a signal." Signal
9 is SIGKILL. So 137 always means "this process was force-killed,"
and in a Kubernetes container context, the overwhelming majority of
the time this means OOMKilled.

---

## 6. QoS Classes — Direct Result of requests/limits

| Setup | QoS Class | Evicted (when NODE runs low on resources) |
|---|---|---|
| requests == limits for everything | Guaranteed | LAST |
| requests set, limits different (or partial) | Burstable | SECOND |
| nothing set | BestEffort | FIRST |

```bash
kubectl get pod <name> -o jsonpath='{.status.qosClass}'
```

**Important distinction:** QoS class eviction is a NODE-LEVEL
decision (the node itself is under memory/disk pressure and must
evict SOME pod to survive). OOMKilled is a CONTAINER-LEVEL decision
(THIS SPECIFIC container exceeded ITS OWN limit). These are two
DIFFERENT mechanisms — don't confuse them in an interview.

---

## 7. What Is a Memory Leak — Complete Explanation

### The Notebook Analogy

Imagine your app's job: every time a payment comes in, write down
some details about it in a notebook (this is like storing data in a
HashMap/cache/list in memory).

```
Payment 1 comes in → write entry in notebook
Payment 2 comes in → write entry in notebook
Payment 3 comes in → write entry in notebook
...
```

### The Bug

The CORRECT code should ERASE each entry from the notebook ONCE that
payment is fully processed — you don't need it anymore.

But with a memory leak bug, the code NEVER erases anything. Every
payment EVER processed stays written in the notebook FOREVER.

```
After 1 hour:  notebook has 1,000 entries  → uses 50 MB memory
After 2 hours: notebook has 2,000 entries  → uses 100 MB memory
After 4 hours: notebook has 4,000 entries  → uses 200 MB memory
After 8 hours: notebook has 8,000 entries  → uses 400 MB memory
                                              ↑ memory.limit is 256Mi
                                              → OOMKilled!
```

Memory usage KEEPS GROWING over time, even though the WORK being
done (processing payments) stays the same volume. This
growing-forever pattern is a **memory leak**.

### The "Quick Fix" That Doesn't Actually Fix Anything

Before finding the real bug, a common first reaction is: just give
the container MORE memory room (256Mi → 512Mi). This buys TIME — the
container takes LONGER to hit OOMKilled — but does NOT fix the leak.
Eventually even 512Mi fills up too, just later.

### The Real Fix

The team examines the CODE and finds the HashMap (or list, or cache)
that is never cleared. They add code to ERASE old entries once each
payment is done:

```
Payment 1 comes in → write entry → payment done → ERASE entry ✓
Payment 2 comes in → write entry → payment done → ERASE entry ✓
```

Now the notebook NEVER grows unboundedly — it stays small and STEADY
no matter how long the app runs.

### What "Reduced Memory Usage by 80%" Means

```
BEFORE the fix (after running a while): app uses 500 MB
  (because the notebook accumulated tons of old entries)

AFTER the fix: app uses 100 MB
  (just enough for CURRENT in-progress payments — old ones erased)

Reduction = 500 - 100 = 400 MB
Percentage reduction = 400 / 500 = 80%
```

The app now uses only 20% of the memory it used to use, because the
leak was fixed at the source — not patched around with a bigger
limit.

### Why This Story Matters in Interviews

It demonstrates you don't just "throw resources at the problem"
(increase limit and move on) — you INVESTIGATE the ROOT CAUSE, fix
the actual code issue, and MEASURE the result. A specific number (80%
reduction) proves the fix actually worked, rather than just "I think
it's fixed now."

---

## 8. Common Causes of Memory Leaks in Real Applications

| Cause | Example |
|---|---|
| Unbounded cache/HashMap | Storing every request/response without ever evicting old entries |
| Unclosed connections | Opening DB connections or file handles without closing them — each one holds memory |
| Event listener accumulation | Registering callbacks/listeners repeatedly without removing old ones |
| Growing logs/buffers in memory | Buffering log lines in an array before "eventually" flushing, but flush never happens or is too slow |
| Session data never expiring | User session objects kept in memory forever, never timed out |
| Static collections in Java | `static` fields holding references to objects that should be garbage collected but can't be, because the static reference keeps them alive |

---

## 9. How to Detect a Memory Leak vs Normal High Usage

### Normal high usage (NOT a leak)

```
Memory usage goes UP under load, comes back DOWN when load decreases
Pattern: sawtooth / wave shape that tracks traffic
```

### Memory leak

```
Memory usage goes UP under load, but does NOT come back down
when load decreases — and keeps climbing even at STEADY load
Pattern: continuously rising line (staircase or ramp), never
plateaus or drops
```

```bash
# Watch memory usage over time
watch -n 5 'kubectl top pod <pod-name>'

# Better — graph it in Grafana/Prometheus with the metric:
# container_memory_working_set_bytes{pod="<pod-name>"}
# A leak shows as a line that NEVER goes down, climbing steadily
# until OOMKilled, then resetting to near-zero on restart, then
# climbing again — a classic "sawtooth that resets via crash"
# pattern (different from the healthy sawtooth that tracks traffic)
```

---

## HANDS-ON LABS

---

## Lab 1 — Watch CPU Throttling Happen

```bash
kubectl run cpu-throttle --image=polinux/stress \
  --requests=cpu=100m --limits=cpu=200m \
  -- stress --cpu 2

# Watch usage stay capped
kubectl top pod cpu-throttle
# Run this a few times — usage stays near 200m even though
# "stress --cpu 2" is trying to use 2 full cores

# Check throttling stats
kubectl exec cpu-throttle -- cat /sys/fs/cgroup/cpu/cpu.stat | grep throttled

kubectl delete pod cpu-throttle
```

---

## Lab 2 — Watch OOMKilled Happen

```bash
kubectl apply -f - << EOF
apiVersion: v1
kind: Pod
metadata:
  name: oom-demo
spec:
  containers:
  - name: stress
    image: polinux/stress
    resources:
      requests:
        memory: "50Mi"
      limits:
        memory: "50Mi"
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "100M", "--vm-hang", "1"]
  restartPolicy: Never
EOF

kubectl get pod oom-demo -w
# Watch it transition to Error/OOMKilled

kubectl describe pod oom-demo | grep -A 3 "Last State"
# Reason: OOMKilled
# Exit Code: 137

kubectl get pod oom-demo -o jsonpath='{.status.qosClass}'
echo
# Guaranteed (because requests == limits)

kubectl delete pod oom-demo
```

---

## Lab 3 — Simulate a Memory Leak and Watch It Grow

```bash
# This pod allocates more memory every few seconds, never freeing it
kubectl apply -f - << EOF
apiVersion: v1
kind: Pod
metadata:
  name: leak-demo
spec:
  containers:
  - name: leaker
    image: polinux/stress
    resources:
      requests:
        memory: "50Mi"
      limits:
        memory: "200Mi"
    command: ["sh", "-c"]
    args:
    - |
      i=0
      while true; do
        i=$((i+1))
        stress --vm 1 --vm-bytes $((i * 20))M --vm-hang 5 &
        sleep 5
      done
  restartPolicy: Never
EOF

# Watch memory CLIMB steadily — this is the leak pattern
watch -n 2 'kubectl top pod leak-demo'

# Eventually it will hit the 200Mi limit and OOMKill
kubectl get pod leak-demo -w

kubectl describe pod leak-demo | grep -A 3 "Last State"

kubectl delete pod leak-demo
```

---

## Lab 4 — Compare Allocated (requests) vs Actual Usage

```bash
# Create 3 pods that REQUEST a lot but USE very little
kubectl apply -f - << EOF
apiVersion: v1
kind: Pod
metadata:
  name: low-usage-1
spec:
  containers:
  - name: idle
    image: busybox
    command: ["sleep", "3600"]
    resources:
      requests:
        cpu: "500m"
        memory: "256Mi"
EOF

kubectl apply -f - << EOF
apiVersion: v1
kind: Pod
metadata:
  name: low-usage-2
spec:
  containers:
  - name: idle
    image: busybox
    command: ["sleep", "3600"]
    resources:
      requests:
        cpu: "500m"
        memory: "256Mi"
EOF

# Reservation-based view — looks "busy"
kubectl describe node <node-name> | grep -A 5 "Allocated resources"

# Real usage view — almost nothing
kubectl top pods

# This gap is exactly what an interviewer means by
# "over-provisioned requests"

kubectl delete pod low-usage-1 low-usage-2
```

---

# INTERVIEW QUESTIONS

---

## THEORETICAL QUESTIONS

---

**Q1. What is the fundamental difference between requests and limits?**

`requests` is a planning/reservation value used ONLY by the scheduler
at pod-placement time — it determines WHICH NODE can host the pod,
based on guaranteed minimum need, and does NOT reflect real-time
usage. `limits` is a runtime enforcement value applied CONTINUOUSLY
by the Linux kernel via cgroups, AFTER the pod is already running —
it caps how much CPU/memory the container can actually consume.

---

**Q2. Why does exceeding a CPU limit cause throttling, but exceeding a memory limit causes the process to be killed?**

CPU is a "compressible" resource — the kernel can give a process less
CPU time over a period without destroying it; this is throttling via
the cgroup CFS quota. Memory is "incompressible" — there's no
in-between state for memory allocation; either the allocation
succeeds or it fails. When a container's memory usage would exceed
its limit, the kernel's OOM killer sends SIGKILL to terminate the
process immediately, reported by Kubernetes as OOMKilled with exit
code 137.

---

**Q3. What does exit code 137 mean, and why specifically that number?**

137 = 128 + 9. The "128 +" indicates the process was terminated by a
signal. Signal number 9 is SIGKILL. In a container context, exit code
137 almost always means the container was OOMKilled — its memory
usage exceeded its configured `limits.memory`, and the Linux kernel
forcibly killed it with SIGKILL, allowing no graceful cleanup.

---

**Q4. What are the three QoS classes, how are they determined, and what do they affect?**

- Guaranteed — requests == limits for BOTH cpu and memory on EVERY
  container in the pod. Evicted LAST under node pressure.
- Burstable — requests set, but not equal to limits (or only some
  resources/containers have both set). Evicted SECOND.
- BestEffort — no requests or limits set on any container, for any
  resource. Evicted FIRST.

This affects eviction PRIORITY when the NODE itself is under
resource pressure (e.g., low on memory) — Kubernetes decides which
pods to evict from that node to relieve pressure, starting with the
lowest QoS class.

---

**Q5. What is the difference between "node-level eviction based on QoS" and "container OOMKilled"? Are these the same thing?**

No, these are two DIFFERENT mechanisms. Container OOMKilled is a
PER-CONTAINER decision — THIS container exceeded ITS OWN
`limits.memory`, so the kernel kills just this container's process,
regardless of what else is happening on the node. Node-level eviction
based on QoS is a NODE-WIDE decision — the entire NODE is running low
on memory/disk (e.g., from the cumulative usage of ALL pods, or other
processes), and the kubelet must evict ENTIRE PODS to relieve
pressure on the node, choosing which pods to evict based on their QoS
class (BestEffort first, then Burstable, Guaranteed last).

---

**Q6. What is a memory leak? Why does increasing the memory limit not "fix" it?**

A memory leak is when an application's memory usage grows
continuously over time, even though the WORKLOAD (volume of requests/
transactions) stays constant — typically because the application
allocates memory for some data structure (cache, list, map) but never
releases/clears entries that are no longer needed. Increasing the
memory limit does NOT fix this — it only delays the inevitable. The
growth is UNBOUNDED, so given enough time, even a much larger limit
will eventually be exceeded too. The actual fix requires identifying
and correcting the code that fails to release memory.

---

**Q7. How would you DISTINGUISH "normal high memory usage under load" from "a memory leak" just by looking at a memory usage graph over time?**

Normal high usage under load shows a sawtooth or wave pattern that
TRACKS traffic — memory rises when load increases and FALLS BACK DOWN
when load decreases, settling back to a baseline. A memory leak shows
a CONTINUOUSLY RISING line that does NOT come back down even when
load decreases, and keeps climbing even at STEADY load — until it
hits the limit and the container is OOMKilled, at which point memory
resets near-zero (on restart) and the climb begins again. The
distinguishing factor is whether memory returns to baseline after
load decreases.

---

**Q8. If `requests` is not set on a container, what happens for scheduling purposes?**

If `requests` is omitted, Kubernetes effectively treats it as 0 for
that resource in most configurations — meaning the scheduler reserves
NO capacity for this pod and can place it on a node regardless of how
"full" that node already is (by requests). This is risky because the
pod could land on an already heavily-requested node and then have to
compete for real resources. Note: if `limits` IS set but `requests`
is not, some cluster configurations (via LimitRange or admission
defaults) automatically set `requests = limits`.

---

**Q9. Why might setting requests TOO HIGH be a problem even if the application never crashes?**

The scheduler treats requests as RESERVED capacity regardless of
ACTUAL usage. If requests are set far above real usage, nodes will
appear "full" to the scheduler (based on reservations) much sooner
than they're actually full (based on real usage measured by
`kubectl top`). This wastes cluster capacity — new pods cannot be
scheduled onto a node that LOOKS full even though it has plenty of
genuinely free resources, just not "unreserved" ones. The fix is
right-sizing requests based on observed real usage (often via
Vertical Pod Autoscaler recommendations or historical Prometheus
data).

---

**Q10. What is the difference between what `kubectl top pod` shows and what the "Allocated resources" section of `kubectl describe node` shows?**

`kubectl top pod` (and `kubectl top node`) shows ACTUAL real-time
resource usage, sourced from metrics-server. `kubectl describe
node`'s "Allocated resources" section shows the SUM of `requests`
(and separately `limits`) declared by all pods scheduled on that
node — these are RESERVATIONS/CEILINGS, not actual usage. A node can
show "90% allocated" by requests while `kubectl top node` shows only
"20% used" — the gap represents reserved-but-unused capacity.

---

## TROUBLESHOOTING QUESTIONS

---

**Q11. TROUBLESHOOTING: A pod is repeatedly getting OOMKilled every few hours, always at roughly the same memory threshold. Walk through your full investigation.**

```bash
# Step 1 — confirm it's OOMKilled and at what threshold
kubectl describe pod <name> | grep -A 3 "Last State"
# Reason: OOMKilled, Exit Code: 137

kubectl get pod <name> -o jsonpath='{.spec.containers[0].resources.limits.memory}'
echo

# Step 2 — observe the memory growth pattern over time
# (Grafana/Prometheus query on container_memory_working_set_bytes
#  for this pod, over the last 24 hours)

# Step 3 — check if growth correlates with traffic or is
# independent of traffic (steady growth even at low traffic
# = memory leak, not just under-provisioning)
```

If the memory graph shows a STEADY, NEVER-DECREASING climb
regardless of traffic, this points to a memory leak in the
application code, NOT simply an under-provisioned limit. Immediate
mitigation: increase the limit to buy time/reduce frequency of
crashes. Root cause fix: profile the application (e.g., heap dump
analysis for Java, memory profiler for Python/Node) to find the
unreleased data structure, and fix the code to properly release
entries.

---

**Q12. TROUBLESHOOTING: `kubectl top pod` shows a pod using 90m CPU, well under its `limits.cpu: 500m`, but the application is reporting slow response times. Could this still be a CPU throttling issue? How would you check?**

Yes, potentially — `kubectl top pod` shows AVERAGE usage over its
sampling interval, but CPU throttling happens based on usage within
very short windows (the cgroup CFS period, typically 100ms). A
container could have BURSTS that exceed its limit for brief moments
(causing throttling during those bursts, adding latency), while the
AVERAGE over a longer window looks well under the limit.

```bash
# Check cgroup throttling stats directly
kubectl exec <pod> -- cat /sys/fs/cgroup/cpu/cpu.stat
# nr_periods:        total scheduling periods
# nr_throttled:      periods where throttling occurred
# throttled_time:    total nanoseconds spent throttled
```

If `nr_throttled` is non-zero and growing, the container IS being
throttled during bursts even though average usage looks fine — the
fix is increasing `limits.cpu` to allow for burst headroom.

---

**Q13. TROUBLESHOOTING: A pod's memory usage graph shows a sawtooth pattern — climbing during business hours, dropping back to baseline overnight, repeating daily. Is this a memory leak?**

No — this is HEALTHY behavior. The pattern TRACKS the daily traffic
cycle: memory rises as load increases during business hours
(more concurrent requests, more cached data) and FALLS BACK to
baseline overnight when load decreases (caches expire, connections
close, garbage collection reclaims memory from completed requests). A
TRUE memory leak would show memory NOT returning to baseline even
during low-traffic periods — each day's "low point" would be HIGHER
than the previous day's, an upward-trending baseline over time, not a
repeating sawtooth at a CONSTANT baseline.

---

**Q14. TROUBLESHOOTING: After deploying a new version of an application, pods immediately start getting OOMKilled within minutes — they previously ran fine for days on end with the old version. What's your approach?**

```bash
# Step 1 — confirm OOMKilled and immediate timing
kubectl describe pod <name> | grep -A 3 "Last State"

# Step 2 — compare resource requests/limits between old
# and new deployment — did anything change?
kubectl get deployment <name> -o yaml | grep -A 5 resources

# Step 3 — check what changed in the new version
# (new dependency, new caching layer, new in-memory
#  data structure added in this release)

# Step 4 — if limits are UNCHANGED but the new version
# OOMKills immediately (not after hours like a slow leak),
# this suggests the new code has a STARTUP-TIME memory
# spike — e.g., loading a large dataset/model into memory
# at startup that exceeds the existing limit
```

Given the IMMEDIATE failure (minutes, not hours), this is more likely
a baseline memory requirement INCREASE in the new version (e.g., a
new library, larger startup cache, bigger thread pool) rather than a
slow leak. The fix is either increasing `limits.memory` to
accommodate the new baseline, or optimizing the new code's startup
memory footprint — I'd check the release notes/diff for the new
version for anything that loads data into memory at startup.

---

**Q15. TROUBLESHOOTING: You increase a pod's `limits.memory` from 256Mi to 1Gi to "fix" OOMKills. A week later, it's OOMKilled again, now at the 1Gi threshold, but it took 4x longer to happen. What does this tell you, and what should you do differently?**

This is a CLASSIC signature of a memory leak, NOT an
under-provisioning issue. If it were simply under-provisioned (the
app genuinely NEEDS more memory at steady-state), increasing the
limit would have made the OOMKills STOP entirely — the app would
stabilize at its new (higher) steady-state usage and stay there. The
fact that it took "4x longer" to OOMKill at "4x the limit" (256Mi →
1Gi is roughly 4x) suggests memory is growing at a roughly CONSTANT
RATE over time — quadrupling the ceiling simply quadrupled the time
to reach it. This is a STRONG signal of unbounded growth (a leak),
not a one-time higher baseline. What to do differently: stop
increasing the limit further (it will just keep delaying the
inevitable) and instead profile the application's memory usage over
time to find what's accumulating — e.g., heap dump analysis at two
points in time and diff the object counts to find what's growing.

---

## SCENARIO QUESTIONS

---

**Q16. SCENARIO: You set `requests.cpu: "2"` on a pod, but your cluster's largest node has only 1.5 allocatable CPU cores. What happens, and how is this DIFFERENT from a memory-related scheduling problem?**

The pod stays `Pending` indefinitely. `kubectl describe pod` shows an
event like `0/3 nodes are available: 3 Insufficient cpu` — no node
has 2 full cores of allocatable capacity, so the scheduler can never
place this pod, regardless of how empty the cluster otherwise is.
This is FUNDAMENTALLY THE SAME for memory — `Insufficient memory`
would show identically if `requests.memory` exceeded every node's
allocatable memory. In BOTH cases, this is a SCHEDULING-TIME problem
based on `requests`, resolved only by reducing the requested amount
or adding a larger node — it has NOTHING to do with `limits` or
runtime enforcement, since the pod never even STARTS running.

---

**Q17. SCENARIO: Pod A has `requests.memory: 256Mi, limits.memory: 256Mi` (Guaranteed). Pod B has no resources set at all (BestEffort). Both are on the same node. The node starts running low on memory due to a THIRD process (not Pod A or B) consuming memory outside Kubernetes' awareness. What happens, and to which pod first?**

The kubelet detects node-level memory pressure (via its eviction
manager, monitoring overall node memory availability) and begins
evicting PODS to relieve pressure — starting with the LOWEST QoS
class first. Pod B (BestEffort) is evicted FIRST, since it made no
resource guarantees and is considered least important to protect.
Pod A (Guaranteed) is evicted LAST — Kubernetes tries hardest to keep
Guaranteed pods running, since they represent workloads with explicit
matched resource commitments. Note this is a DIFFERENT mechanism from
container OOMKilled — neither Pod A nor Pod B individually exceeded
THEIR OWN memory limit; the NODE itself is under pressure from
external causes, and kubelet is evicting whole pods to free up node
memory.

---

**Q18. SCENARIO: You're told "our payment service pod restarts every 6 hours exactly, like clockwork." What questions would you ask, and what's your initial hypothesis?**

The exact, REGULAR interval ("every 6 hours exactly") is a strong
signal. My initial hypothesis: this smells like a memory leak with a
PREDICTABLE growth rate — if memory grows at a roughly constant rate
under roughly constant traffic, it will ALWAYS take the same amount
of time to fill up a fixed-size limit, producing this clockwork
pattern. (A genuinely RANDOM crash cause — like an occasional bad
input causing a crash — would NOT produce such a regular interval.)

Questions I'd ask: (1) "Is the restart reason OOMKilled?" —
`kubectl describe pod | grep -A 3 "Last State"` would confirm/deny
this immediately. (2) "Does the 6-hour interval change if traffic
changes (e.g., is it shorter during peak hours, longer overnight)?" —
if YES, the growth rate is traffic-dependent (leak triggered by
request processing); if NO (always exactly 6 hours regardless of
traffic), the leak might be triggered by a TIME-based process (e.g.,
something accumulating per-tick of an internal timer/cron, unrelated
to request volume).

---

**Q19. SCENARIO: A Java application pod is configured with `limits.memory: 1Gi`, but the JVM's `-Xmx` (max heap size) flag is set to `2g`. The pod gets OOMKilled almost immediately on startup. Explain what's happening.**

The JVM's `-Xmx2g` tells the Java Virtual Machine "you are ALLOWED to
grow your heap up to 2GB" — the JVM may not use all of it
immediately, but it can ATTEMPT to allocate up to that much as the
application runs. However, the Kubernetes `limits.memory: 1Gi`
caps the ENTIRE container (including JVM heap, JVM metaspace, thread
stacks, and any native memory) at 1GB — enforced by the kernel
cgroup. The moment the JVM's actual memory usage (which can approach
or exceed the configured `-Xmx` due to additional JVM overhead beyond
just the heap) crosses 1Gi, the kernel OOM-kills the container — often
EARLY in the application's life if it does any significant
initialization that allocates heap space. The fix: the JVM's `-Xmx`
(and other memory-related JVM flags) MUST be set CONSISTENTLY BELOW
the container's `limits.memory`, accounting for non-heap overhead —
typically `-Xmx` should be set to roughly 70-75% of `limits.memory` to
leave headroom for JVM overhead outside the heap.

---

**Q20. SCENARIO: You're asked to right-size the `requests` and `limits` for a Deployment that currently has NO resources section at all (BestEffort). What is your step-by-step approach, and what tools/data would you use?**

Step 1 — observe REAL usage over a representative period (at least
one full business cycle, e.g., a week, to capture peak and off-peak
patterns): `kubectl top pod` repeatedly, or better, query
`container_cpu_usage_seconds_total` and
`container_memory_working_set_bytes` from Prometheus over time for
this deployment's pods.

Step 2 — identify the BASELINE (steady-state/typical) usage and the
PEAK (highest observed) usage for both CPU and memory.

Step 3 — set `requests` close to the BASELINE/typical usage (so the
scheduler reserves roughly what the pod NORMALLY needs, without
over-reserving) — for memory specifically, requests should be set
high enough that the pod is NOT immediately under memory pressure at
startup.

Step 4 — set `limits` above the observed PEAK, with some headroom
(commonly 1.5x-2x the peak, or 1.5x-2x the request) — high enough
that normal peak traffic doesn't get throttled (CPU) or OOMKilled
(memory), but low enough to still protect the node from a truly
runaway/misbehaving instance.

Step 5 — after applying, MONITOR for: (a) any new OOMKills (limit too
low), (b) CPU throttling via `cgroup cpu.stat` (limit too low), (c)
the resulting QoS class and whether it matches intent, and (d) whether
`kubectl describe node`'s "Allocated resources" now reflects realistic
reservations rather than 0 (BestEffort) or wildly over-provisioned
numbers.

Tools: `kubectl top`, Prometheus + Grafana for historical trends, and
optionally Vertical Pod Autoscaler in "recommendation only" mode,
which automatically computes suggested requests/limits based on
observed usage history without actually changing anything until
configured to do so.

---

*Run Labs 1-4 on your MicroK8s — especially Lab 3 (simulated memory
leak) — watching memory climb in real time via `watch -n 2 'kubectl
top pod leak-demo'` makes Q6, Q7, Q15, and Q18 click immediately.*
