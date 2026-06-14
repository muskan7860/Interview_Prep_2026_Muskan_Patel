# Pods — Interview Questions (Architectural, Scenario, Troubleshooting)

> Organized by priority — HIGH PRIORITY first (most commonly asked)
> Author: Muskan Patel

---

## HIGH PRIORITY — Almost Guaranteed to Be Asked

---

**Q1. What is a Pod and why does Kubernetes use it instead of running
containers directly?**

A Pod is the smallest deployable unit in Kubernetes — a wrapper
around one or more containers that share network namespace (one IP,
talk via localhost), can share storage volumes, and are always
scheduled together on the same node.

Kubernetes uses Pods because sometimes one container needs a helper
running beside it — like a sidecar that ships logs. These containers
need to share network and storage and start/stop together as a unit.
Docker has no native concept of "these containers belong together."
The Pod is that grouping abstraction.

---

**Q2. What is the difference between requests and limits?**

`requests` is a RESERVATION used ONLY by the scheduler to decide
WHICH NODE can host the pod — checked once at scheduling time, based
on guaranteed minimum need, NOT real-time usage.

`limits` is a HARD CEILING enforced CONTINUOUSLY by the Linux kernel
via cgroups. CPU limit exceeded → throttled (slows down, survives).
Memory limit exceeded → OOM killed (SIGKILL, exit code 137).

---

**Q3. What happens if a container exceeds its memory limit vs its CPU
limit? Why are these different?**

CPU is "compressible" — Linux can give the process less CPU time
over time without destroying it, so exceeding the CPU limit causes
THROTTLING via cgroup CFS quota — app runs slower but survives.

Memory is "incompressible" — either the allocation succeeds or it
doesn't. Exceeding the memory limit triggers the kernel's OOM killer,
which sends SIGKILL immediately. Kubernetes reports this as
`Reason: OOMKilled, Exit Code: 137`.

---

**Q4. What is the difference between a liveness probe and a readiness
probe?**

Liveness probe answers "is this container alive?" — if it fails,
kubelet RESTARTS the container. Used to detect deadlocks.

Readiness probe answers "is this container ready for traffic?" — if
it fails, the pod is REMOVED from Service endpoints (no traffic
sent), but the container is NOT restarted. Used for app warm-up or
waiting on dependencies.

A pod can be Running + alive (liveness passes) but show READY: 0/1
(readiness fails) — common during slow startup.

---

**Q5. Does `containerPort` actually open a port?**

No. It is purely documentation. It does NOT configure networking or
open any port. The application inside the container determines what
port it listens on, based on its own code/config. Even with zero
ports declared, any port the app binds to is reachable from other
pods by default (subject to NetworkPolicy). Its actual uses:
documentation in `kubectl describe`, and allowing a Service's
`targetPort` to reference it by NAME.

---

**Q6. What is the difference between a standalone Pod and a Pod
managed by a Deployment?**

A standalone Pod, if deleted or crashed, is gone forever — nothing
recreates it. A Pod managed by a Deployment (via ReplicaSet) is
continuously monitored by the ReplicaSet controller's reconciliation
loop — if it dies, a replacement is created automatically within
seconds, with a different name and IP but the same template.

---

**Q7. What is QoS class? What are the three classes and how are they
determined?**

QoS class determines eviction priority when a NODE runs low on
resources (node-level eviction).

- Guaranteed — requests == limits for BOTH cpu and memory, every
  container. Evicted LAST.
- Burstable — requests set but != limits (or partially set).
  Evicted SECOND.
- BestEffort — nothing set at all. Evicted FIRST.

```bash
kubectl get pod <name> -o jsonpath='{.status.qosClass}'
```

---

**Q8. Are Kubernetes Secrets encrypted?**

No, by default they are base64-ENCODED (not encrypted) in etcd —
trivially reversible with `base64 -d`. Real protection comes from
RBAC (controlling who can `kubectl get secrets`) and etcd
encryption-at-rest (a separate cluster config). For sensitive
environments, external secret managers (AWS Secrets Manager via
External Secrets Operator) are preferred over native Secrets alone.

---

## ARCHITECTURE QUESTIONS

---

**Q9. What is the "pause" container and why does it exist?**

When a Pod is created, Kubernetes first creates a hidden pause
container whose only job is to hold the network namespace open. All
other containers in the pod join this same network namespace — this
is WHY all containers in a pod share one IP address and can reach
each other via localhost. Even if the main app container restarts,
the pause container keeps the network identity stable.

---

**Q10. What is a startup probe and why was it introduced?**

A startup probe DISABLES liveness AND readiness checks until it
itself succeeds. It exists for SLOW-starting applications. Without
it, you'd need a very long `initialDelaySeconds` on the liveness
probe — but that means liveness checks stay "lazy" for the ENTIRE
lifetime of the container. With a startup probe, the app gets
generous startup time, and ONCE it passes, liveness/readiness switch
to tight, normal checking intervals.

---

**Q11. What is restartPolicy and what are its three values?**

Tells kubelet what to do when a container exits.

- `Always` (default) — restart regardless of exit code. Used for
  long-running apps (Deployments).
- `OnFailure` — restart only if exit code is non-zero. Used for
  Jobs.
- `Never` — never restart. Used for debug pods or Jobs where you
  want to inspect the failed pod's exact state.

Job/CronJob pods must use OnFailure or Never — API server rejects
Always for them.

---

**Q12. What are the different Pod phases?**

| Phase | Meaning |
|---|---|
| Pending | Accepted but not all containers running yet |
| Running | At least one container is running |
| Succeeded | All containers exited with code 0 |
| Failed | At least one container exited non-zero, no further restart |
| Unknown | API server lost contact with the node |

---

**Q13. Two ways to inject ConfigMap/Secret into a Pod — explain both
and the key difference.**

1. Environment variables — injected ONCE at container start. If the
source changes later, the running container does NOT see the new
value until restarted.

2. Volume mounts — appear as FILES inside the container. kubelet
updates the mounted files when source changes (with sync delay,
~1 minute) WITHOUT restarting the pod — but the application must be
coded to detect file changes and re-read them.

---

**Q14. Can a Pod have multiple containers? When would you use this?**

Yes — the multi-container pattern, used for:

- Sidecar — helper container (log shipper, service mesh proxy)
- Init containers — run and complete BEFORE main containers start
  (wait for DB, download config)
- Ambassador — proxy container simplifying how main container talks
  to external services

All containers in a pod share network namespace (one IP, localhost
communication) and can share volumes.

---

## SCENARIO-BASED QUESTIONS

---

**Q15. SCENARIO: You delete a standalone pod. Then you delete a pod
that IS part of a Deployment. Compare what happens.**

Standalone pod: deleted permanently, gone forever — nothing
recreates it.

Pod part of a Deployment: the ReplicaSet controller's reconciliation
loop detects the actual pod count dropped below `spec.replicas`, and
immediately creates a NEW pod (different name/IP, same template) to
restore the desired count. The Deployment object itself is
unaffected — only the ReplicaSet reacts.

---

**Q16. SCENARIO: You manually create a Pod with label `app: payment`
in a namespace where a ReplicaSet (selector: app=payment, replicas:
3) already has exactly 3 pods running. What happens to your manually
created pod?**

The ReplicaSet controller watches pods matching ITS label selector —
it does not check who created the pod, only whether labels match. It
now sees 4 pods matching `app: payment`, but desired is 3. To
reconcile, it DELETES one pod to bring count back to 3 — and there's
a real chance it deletes YOUR manually created pod. Label selectors
must be carefully scoped to avoid this kind of collision.

---

**Q17. SCENARIO: A Pod has restartPolicy: Never. Its container exits
with code 0. What is the Pod's phase? What if it exits with code 1?**

Exit 0 + restartPolicy Never → Pod phase = Succeeded. Container not
restarted (it succeeded, nothing to retry).

Exit 1 (non-zero) + restartPolicy Never → Pod phase = Failed.
Container not restarted (policy says Never), pod remains in Failed
state for inspection via kubectl logs/describe.

---

**Q18. SCENARIO: Your pod's container keeps crashing with exit code
137 right after a deployment that increased traffic. Five minutes ago
it was stable. What's your hypothesis and how do you confirm/fix it?**

Hypothesis: increased traffic → increased memory usage → container
hits its memory limit → OOMKilled (exit 137).

```bash
# Confirm
kubectl describe pod <name> | grep -A 3 "Last State"
# Should show Reason: OOMKilled

# Check usage trend
kubectl top pod <name>
```

Immediate fix — increase memory limit:
```bash
kubectl set resources deployment <name> --limits=memory=512Mi --requests=memory=256Mi
```

Longer term — check if memory usage scales linearly with traffic
(healthy) or accumulates without bound even at steady traffic
(memory leak) — the latter needs application profiling, not just a
bigger limit.

---

**Q19. SCENARIO: You exec into a pod and `curl localhost:8080` works.
But from ANOTHER pod, `curl <pod-ip>:8080` fails with connection
refused, even though containerPort: 8080 is declared. What's wrong?**

containerPort is documentation only — it does NOT mean the app is
listening on `0.0.0.0:8080` (all interfaces). If the app is bound to
`127.0.0.1:8080` (localhost only), `curl localhost:8080` from inside
the pod works (localhost = 127.0.0.1 within the pod's namespace), but
external pods get connection refused because the app isn't listening
on the externally-reachable interface. Fix: change app config to bind
to `0.0.0.0` instead of `127.0.0.1`/`localhost`.

---

**Q20. SCENARIO: You set requests.cpu: "2" on a pod, but your
cluster's largest node only has 1.5 CPU cores allocatable. What
happens?**

The pod stays Pending INDEFINITELY. `kubectl describe pod` shows
`0/3 nodes are available: 3 Insufficient cpu` — no node has 2 full
cores of allocatable capacity, so the scheduler can never find a fit,
regardless of how empty the cluster otherwise is. Either reduce
requests.cpu to something achievable, or add a larger node.

---

**Q21. SCENARIO: Pod A is Guaranteed, Pod B is BestEffort. The node
they're on runs out of memory (node-level pressure). Which is evicted
first?**

Pod B (BestEffort) first. Node-pressure eviction prioritizes evicting
pods with the LOWEST QoS guarantee first — BestEffort pods made no
resource promises and are least important to keep. Guaranteed pods
are evicted last.

---

**Q22. SCENARIO: requests is too HIGH compared to actual usage — what
problem does this cause, even though the pod never crashes?**

The scheduler treats requests as RESERVED capacity regardless of
actual usage. Over-high requests make nodes appear "full" to the
scheduler much sooner than they're actually full by real usage. New
pods can't be scheduled even though nodes have plenty of real free
capacity, just not "unreserved" capacity. `kubectl describe node`
("Allocated resources" — based on requests) vs `kubectl top node`
(actual usage) — a big gap indicates over-provisioned requests,
wasting cluster capacity.

---

## TROUBLESHOOTING QUESTIONS

---

**Q23. TROUBLESHOOTING: Pod shows READY: 0/1, STATUS: Running,
RESTARTS: 0, for 10 minutes. Walk through your debugging.**

STATUS Running with READY 0/1 means the container process started
(liveness passing) but READINESS is failing — pod gets NO traffic,
not restarted.

```bash
# Step 1 — Events section shows exact probe failure
kubectl describe pod <name>

# Step 2 — test the readiness endpoint directly
kubectl exec -it <pod> -- curl -v localhost:<port>/ready

# Step 3 — check app logs for what it's waiting on
kubectl logs <pod>
```

Common causes: app not finished initializing (increase
initialDelaySeconds or add startupProbe), readiness endpoint checks
a dependency (DB/cache) not yet reachable, or wrong port/path in the
probe config.

---

**Q24. TROUBLESHOOTING: A pod is repeatedly getting OOMKilled. Walk
through investigation and fix.**

```bash
# Step 1 — confirm OOMKilled
kubectl describe pod <name> | grep -A 3 "Last State"
# Reason: OOMKilled, Exit Code: 137

# Step 2 — check usage pattern
kubectl top pod <name>

# Step 3 — check configured limit
kubectl get pod <name> -o jsonpath='{.spec.containers[0].resources.limits.memory}'
```

Immediate fix — increase memory limit:
```bash
kubectl set resources deployment <name> -c=<container> --limits=memory=512Mi
```

Permanent fix — profile the application for a memory leak, since
repeated OOMKills over time often indicate unbounded growth, not just
under-provisioning. (See full example in theory file — fixing a
never-cleared HashMap reduced memory usage by 80% in a real banking
payment service.)

---

**Q25. TROUBLESHOOTING: If a liveness probe keeps failing, what
happens repeatedly, and how do you identify it from `kubectl get
pods`?**

The container is killed (SIGTERM, then SIGKILL after grace period)
and restarted by kubelet. If it keeps failing the probe after each
restart, the pod enters CrashLoopBackOff with INCREASING delay
between restart attempts (10s, 20s, 40s... capped at 5 min).
`kubectl get pods` shows STATUS: CrashLoopBackOff and a rising
RESTARTS count.

```bash
kubectl describe pod <name> | grep -A 5 Liveness
kubectl logs <pod> --previous
```

---

**Q26. TROUBLESHOOTING: kubectl top pod shows very low usage, but
kubectl describe node shows the node is "90% allocated." Explain the
discrepancy and what you'd check.**

`kubectl top pod/node` shows ACTUAL real-time usage from
metrics-server. `kubectl describe node`'s "Allocated resources"
section shows the SUM of `requests` (reservations) of all pods on
that node — NOT actual usage. A node can show 90% allocated (by
requests) but only 20% actual usage (by top) if pods requested far
more than they currently use. I'd check each pod's
`resources.requests` vs its actual `kubectl top pod` usage over time,
and consider reducing over-provisioned requests to free up scheduling
headroom.

---

## TRICKY / GOTCHA QUESTIONS

---

**Q27. If I run `kubectl delete pod <name>` on a pod managed by a
Deployment, does `spec.replicas` change?**

No. `spec.replicas` is the DESIRED count — never changes due to pod
deletions. The ACTUAL current count momentarily drops below desired,
triggering the ReplicaSet controller to create a replacement,
restoring actual count to match the unchanged desired count.

---

**Q28. A pod has NO resources section at all. What QoS class does it
get, and what does this mean practically?**

BestEffort. Practically:
1. Scheduler places it with ZERO reservation — lands on ANY node
   regardless of how "full" by requests
2. NO ceiling — could consume unlimited resources if app misbehaves,
   starving other pods on the node
3. If node runs low on resources, this pod is evicted FIRST, before
   Burstable or Guaranteed pods

---

**Q29. What's the difference between a Pod being "Terminating" and
being "Failed"?**

Terminating is a TRANSIENT display state (not an actual
status.phase) — `kubectl delete` was called, pod has a
deletionTimestamp set, gracefully shutting down (SIGTERM sent,
waiting up to terminationGracePeriodSeconds before SIGKILL). Failed
is an actual terminal status.phase — all containers terminated, at
least one with non-zero exit, and restartPolicy doesn't call for
further restarts.

---

**Q30. What is an init container? Give a real use case.**

A container that runs to COMPLETION BEFORE the main app container(s)
start. Multiple init containers run SEQUENTIALLY — each must succeed
before the next starts, and ALL must succeed before main containers
begin. Real use case: an init container running `nc -z db-host 5432`
in a loop until the database port is reachable — preventing the main
app container from starting (and crashing on DB connection) before
its dependency is up.

---

*Run all 8 labs from 10_Pods_Theory_and_Labs.md before moving to Day 3.
Especially Lab 7 (label selector trap) and Lab 4 (OOMKilled) — these
two alone answer the most commonly asked resource/scheduling
questions because you've SEEN it happen.*
