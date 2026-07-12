# ReplicaSet — Theory, Labs, and Interview Questions

> Level: 4 Years Experience
> Author: Muskan Patel

---

## PART 1 — THEORY

---

## 1. Why ReplicaSet Exists

A standalone Pod, once deleted or crashed, is gone forever — nothing
recreates it. In production, this means a 3 AM crash from a network
blip would leave a service down until a human notices and manually
recreates the pod.

Kubernetes needed something that WATCHES pods continuously and says:
"If this disappears, create a replacement automatically, within
seconds, with zero human involvement." That something is the
**ReplicaSet**.

---

## 2. The Mental Model — The Headcount Manager

Think of a ReplicaSet as a strict headcount manager at a factory
gate, with ONE instruction on a clipboard: "There must ALWAYS be
exactly N workers wearing a badge that says `app: payment` inside
this factory."

The manager does NOT care WHO put a worker there. The manager does
NOT track workers by name. The manager ONLY does ONE thing, in a
loop, forever:

```
Every few seconds:
  Count workers with the matching badge right now

  If count < desired -> create more until count = desired
  If count > desired -> remove some until count = desired
  If count = desired -> do nothing, check again later
```

This counting-and-correcting loop is called the **reconciliation
loop** — the same pattern used by EVERY controller in Kubernetes
(Deployment, Node controller, Job controller — all of them compare
actual-vs-desired and correct the gap).

---

## 3. What Happens When You Apply a ReplicaSet — Step by Step

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: payment-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: payment
  template:
    metadata:
      labels:
        app: payment
    spec:
      containers:
      - name: payment-app
        image: payment:v1.2
```

1. API server validates and stores this object in etcd. At this
   point, ZERO pods exist — only the "instruction" exists.
2. The ReplicaSet Controller (code running inside controller-manager,
   continuously watching etcd) notices this new ReplicaSet wants
   `replicas: 3` matching label `app: payment`.
3. It counts existing matching pods: 0 found.
4. Desired (3) minus Actual (0) = 3 missing. The controller copies
   `template` 3 times and asks the API server to create 3 Pod
   objects — each gets the label `app: payment` and a random name
   suffix like `payment-rs-x7k2p`.
5. From here it's the normal Pod story — Scheduler assigns nodes,
   kubelet pulls images and starts containers.
6. The controller now sleeps, periodically re-checking. As long as
   exactly 3 matching pods exist, it does nothing.

---

## 4. The Self-Healing Moment

```
Pod count drops from 3 to 2 (one crashed)
         |
         v
Controller's next check cycle notices this
         |
         v
Desired (3) - Actual (2) = 1 missing
         |
         v
Controller creates ONE new pod from the SAME template
(new random name, new IP, same image/labels)
         |
         v
Count is back to 3. Controller sleeps again.
```

This happens automatically, in seconds, with zero human involvement
— this is THE feature that makes Kubernetes self-healing, and why
production workloads are never deployed as standalone pods.

---

## 5. Critical Concept — replicas Is Only Read ONCE, at Apply Time

Your YAML file is just text on your disk. Kubernetes does NOT keep
"watching the file." It only watches what's stored in etcd.

```
kubectl apply -f rs.yaml   (replicas: 2)
         |
         v
This WRITES replicas: 2 into etcd as current desired state
         |
         v
Your file's job is DONE. Kubernetes forgot it ever read a file —
it only remembers what's in etcd now.
```

`kubectl scale rs <name> --replicas=5` does NOT touch your YAML
file — it sends a direct command to update etcd's desired state to
5, overwriting whatever was there.

**Whichever command runs LAST wins.** If you `kubectl scale` to 5,
then later re-run `kubectl apply -f rs.yaml` (the OLD file still
saying `replicas: 2`), the cluster goes BACK DOWN to 2 — because
`apply` re-pushes the file's value into etcd, overwriting the scale
command.

This is exactly why GitOps tools (ArgoCD, Flux) continuously
re-apply YAML from Git — to FORCE the cluster to match the
declared source of truth and prevent manual `kubectl scale` drifts
from silently persisting.

---

## 6. Line by Line — Fields Unique to ReplicaSet

### `apiVersion: apps/v1`

Pod uses `v1` (core API group). ReplicaSet uses `apps/v1` because it
is a WORKLOAD CONTROLLER — something that manages OTHER objects. All
workload controllers (Deployment, StatefulSet, DaemonSet) live in
the `apps` group, separate from core objects like Pod, Service,
ConfigMap.

### `spec.replicas: 3`

The ONLY number the controller cares about — the "headcount"
instruction. This is a TARGET, not a guarantee of the current
instant — during a crash, the actual count might briefly be lower
while a replacement is being created.

```bash
kubectl get rs payment-rs
# NAME         DESIRED   CURRENT   READY   AGE
# payment-rs   3         3         3       5m
```
- DESIRED = `spec.replicas`
- CURRENT = pod objects that exist right now (may include
  still-starting ones)
- READY = of those, how many ALSO pass readiness checks

### `spec.selector.matchLabels`

This is the "factory badge" check — a QUERY, not a creation
instruction. It tells the controller: count and manage ONLY pods
carrying this EXACT label, REGARDLESS of who created them.

**The trap:** if you manually create another pod with the SAME
label, the controller counts it as one of its own, and may delete
ANY pod (including yours) to bring the total back to desired.

### `spec.template`

A COMPLETE Pod specification nested inside the ReplicaSet — the
"stamp." Every time the controller needs a new pod, it copies
EVERYTHING under `template`, generates a unique name, and submits a
brand-new Pod object.

**Mandatory rule:** `template.metadata.labels` MUST be a superset
of `selector.matchLabels`. If they don't match, the API server
REJECTS the object outright at creation time — otherwise the
controller would create pods it then can't count as its own,
leading to infinite pod creation attempts.

### Pod Naming Pattern

```
<ReplicaSet-name>-<5-character-random-suffix>

payment-rs-x7k2p
payment-rs-m3n9q
payment-rs-b8j4r
```

Each pod from the SAME ReplicaSet shares the name prefix but gets a
unique random suffix. (This differs from StatefulSet, which uses
sequential ordinals like `postgres-0`, `postgres-1` instead.)

---

## 7. The Biggest Limitation of ReplicaSet — No Rolling Update

Updating `spec.template` (e.g., changing the image) does NOT touch
ANY existing, already-running pods. ONLY pods created AFTER the
template change will use the new template. Existing pods keep
running their OLD image until something else removes them.

```
Update template image 1.24 -> 1.25
         |
         v
Existing pods: STILL on 1.24 (untouched)
Template inside RS object: NOW says 1.25
         |
         v
If you delete ONE existing pod -> replacement created with 1.25
   Result: ONE pod on 1.24 (survivor) + ONE pod on 1.25 (new)

If you delete ALL existing pods at once -> ALL replacements use 1.25
   Result: ALL pods on 1.25, BUT there was a moment of ZERO
   running pods in between (since all were deleted before
   any replacement existed) -> this is a service OUTAGE
```

This exact gap — no gradual, zero-downtime way to roll out a new
version — is WHY **Deployment** exists. Deployment automates
creating a NEW ReplicaSet and gradually shifting pods from old to
new, never dropping below a minimum available count.

---

## PART 2 — HANDS-ON LABS

---

## Lab 1 — Watch the Reconciliation Loop Live

```bash
kubectl apply -f - << 'EOF'
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-rs-demo
  template:
    metadata:
      labels:
        app: nginx-rs-demo
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
EOF

kubectl get pods -l app=nginx-rs-demo

# Delete ONE pod manually
kubectl delete pod $(kubectl get pods -l app=nginx-rs-demo \
  -o jsonpath='{.items[0].metadata.name}')

# Check immediately
kubectl get pods -l app=nginx-rs-demo
# A NEW pod is already appearing, different random suffix

kubectl get events --sort-by='.lastTimestamp' | tail -5
# Source: replicaset-controller

kubectl delete rs nginx-rs
```

---

## Lab 2 — Manual Scaling vs YAML File (Whichever Runs Last Wins)

```bash
kubectl apply -f - << 'EOF'
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: scale-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: scale-demo
  template:
    metadata:
      labels:
        app: scale-demo
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
EOF

kubectl get pods -l app=scale-demo
# 2 pods

# Scale UP via direct command (does NOT touch the YAML file)
kubectl scale rs scale-demo --replicas=5
kubectl get rs scale-demo
# DESIRED: 5

# Now re-apply the ORIGINAL file (still says replicas: 2)
kubectl apply -f - << 'EOF'
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: scale-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: scale-demo
  template:
    metadata:
      labels:
        app: scale-demo
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
EOF

kubectl get rs scale-demo
# DESIRED: 2 again -- the apply OVERWROTE the scale command,
# because apply ran LAST

kubectl delete rs scale-demo
```

---

## Lab 3 — The Label Selector Trap

```bash
kubectl apply -f - << 'EOF'
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: trap-rs
spec:
  replicas: 2
  selector:
    matchLabels:
      app: trap-demo
  template:
    metadata:
      labels:
        app: trap-demo
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
EOF

kubectl get pods -l app=trap-demo
# 2 pods, both owned by trap-rs

# Manually create a THIRD pod with the SAME label
kubectl run manual-pod --image=nginx:1.25 --labels="app=trap-demo"

sleep 3
kubectl get pods -l app=trap-demo
# STILL only 2 pods -- your manual pod was deleted

kubectl get events --sort-by='.lastTimestamp' | tail -3
# Shows a "Deleted pod" event from replicaset-controller

kubectl delete rs trap-rs
```

---

## Lab 4 — Template Update Does NOT Touch Existing Pods (One Old, One New)

```bash
kubectl apply -f - << 'EOF'
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: update-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: update-demo
  template:
    metadata:
      labels:
        app: update-demo
    spec:
      containers:
      - name: nginx
        image: nginx:1.24
EOF

kubectl get pods -l app=update-demo \
  -o jsonpath='{range .items[*]}{.metadata.name}{"  "}{.spec.containers[0].image}{"\n"}{end}'
# Both pods show nginx:1.24

# Update the template
kubectl set image rs/update-demo nginx=nginx:1.25

# Existing pods are UNTOUCHED -- check
kubectl get pods -l app=update-demo \
  -o jsonpath='{range .items[*]}{.metadata.name}{"  "}{.spec.containers[0].image}{"\n"}{end}'
# STILL shows nginx:1.24 for both!

# But the TEMPLATE inside the object IS updated
kubectl get rs update-demo -o jsonpath='{.spec.template.spec.containers[0].image}'
echo
# Shows nginx:1.25

# Now delete ONLY ONE pod by its EXACT name (not all of them)
kubectl get pods -l app=update-demo
# note one exact pod name, e.g. update-demo-abc12
kubectl delete pod <paste-one-exact-pod-name-here>

sleep 3
kubectl get pods -l app=update-demo \
  -o jsonpath='{range .items[*]}{.metadata.name}{"  "}{.spec.containers[0].image}{"\n"}{end}'
# NOW you see:
# the SURVIVOR pod  -> nginx:1.24 (untouched original)
# the NEW pod       -> nginx:1.25 (picked up the new template)

kubectl delete rs update-demo
```

**Important variant — what happens if you delete ALL pods at once
instead of just one:**

```bash
# If you run this instead (deletes BOTH pods simultaneously):
kubectl delete pods -l app=update-demo

# Result: ALL replacement pods will show nginx:1.25 (since there
# were zero old survivors left to compare against) -- but there
# was a brief moment of ZERO running pods in between, since ALL
# old pods were deleted BEFORE any replacement was created.
# This is a real OUTAGE, even though the end result "looks like"
# a clean rolling update. This exact problem is what Deployment's
# maxSurge/maxUnavailable strategy is built to prevent.
```

---

## Quick Reference — ReplicaSet Commands

```bash
# Get replicasets
kubectl get rs
kubectl get rs -o wide

# Describe (events, selector, template)
kubectl describe rs <name>

# Scale directly (does not touch YAML file)
kubectl scale rs <name> --replicas=5

# Update image in template (existing pods NOT affected -- see Lab 4)
kubectl set image rs/<name> <container-name>=<new-image>

# Delete ReplicaSet AND its pods
kubectl delete rs <name>

# Delete ReplicaSet but KEEP its pods (orphan them)
kubectl delete rs <name> --cascade=orphan
```

---

## PART 3 — INTERVIEW QUESTIONS

---

### THEORY QUESTIONS

---

**Q1. What is a ReplicaSet and why does it exist?**

A ReplicaSet is a controller whose only job is to ensure a specified
number of pod replicas, matching a label selector, are running at
all times. It exists because standalone Pods, once deleted or
crashed, are gone forever — nothing recreates them. ReplicaSet
solves this by continuously running a reconciliation loop: count
actual matching pods, compare to desired count, and create or delete
pods to correct any difference. This is the mechanism that makes
Kubernetes workloads self-healing.

---

**Q2. Explain the reconciliation loop in the context of ReplicaSet.**

The ReplicaSet controller runs an infinite loop: observe the current
number of pods matching its label selector, compare that to
`spec.replicas` (the desired count), and take action to close any
gap — creating pods if actual is less than desired, deleting pods if
actual is more than desired. This loop runs continuously for as long
as the ReplicaSet object exists, and is the same fundamental pattern
used by every controller in Kubernetes (Deployment controller, Node
controller, Job controller, etc.).

---

**Q3. What is the relationship between `selector.matchLabels` and `template.metadata.labels`? What happens if they don't match?**

`selector.matchLabels` is the QUERY the controller uses to find and
count pods it should manage. `template.metadata.labels` is what gets
COPIED onto every pod the controller creates. These two MUST be
compatible — specifically, `template.metadata.labels` must be a
superset of `selector.matchLabels`. If they don't match, the API
server REJECTS the ReplicaSet object at creation time with a
validation error, because otherwise the controller would create pods
that don't match its own selector, causing it to perpetually think
it has zero pods and create more, infinitely.

---

**Q4. Why does updating a ReplicaSet's `spec.template` not affect already-running pods?**

ReplicaSet has no built-in update/rollout mechanism. The `template`
is only used at the MOMENT a NEW pod is being created — it is not
continuously re-applied to existing pods. Changing the template only
affects FUTURE pod creations (e.g., when an existing pod crashes and
is replaced). Existing, already-running pods keep running their
original configuration indefinitely unless something else removes
them. This is precisely the gap that Deployment was built to close —
Deployment automates creating a new ReplicaSet on template changes
and gradually shifting pod count from the old ReplicaSet to the new
one.

---

**Q5. How does Kubernetes decide whether `kubectl scale` or the YAML file's `replicas` value is the "current" desired state?**

There is no concept of "current authority" beyond "whichever
command/API call ran most recently." `kubectl scale` and
`kubectl apply` both write to the SAME location — the `replicas`
field of the object stored in etcd. Whichever one executes LAST
overwrites whatever value was there before. The YAML file on disk
has no ongoing enforcement power; it is just a one-time instruction
set used when you explicitly apply it. This is exactly why GitOps
tools like ArgoCD continuously re-apply manifests from a Git
repository — to force the cluster's actual state back to match the
declared source of truth, preventing manual `kubectl scale` changes
from silently and permanently drifting away from what's defined in
version control.

---

**Q6. What is the naming convention for pods created by a ReplicaSet?**

`<ReplicaSet-name>-<5-character-random-alphanumeric-suffix>`, for
example `payment-rs-x7k2p`. Every pod created by the SAME ReplicaSet
shares the name prefix but gets a unique random suffix. This differs
from StatefulSet, where pods get predictable, SEQUENTIAL ordinal
names like `postgres-0`, `postgres-1` instead of random suffixes.

---

### SCENARIO QUESTIONS

---

**Q7. SCENARIO: A ReplicaSet has `replicas: 3`. You manually create a fourth pod with a label that happens to match the ReplicaSet's selector. What happens, and why might it NOT be your manually-created pod that gets deleted?**

The ReplicaSet controller does not track which pods it personally
created — it only counts pods matching its label selector. Seeing 4
matching pods but desiring only 3, it deletes ONE pod to restore the
count to 3. Which specific pod gets deleted is not something you
should rely on or assume — the controller's selection logic (in
different Kubernetes versions, it may prefer deleting pods that are
NotReady, newest, or on overloaded nodes first) is an implementation
detail, not a guarantee. It could delete your manually-created pod,
or it could delete one of its own original pods instead — the only
guarantee is that the FINAL count will be 3.

---

**Q8. SCENARIO: You change a ReplicaSet's `spec.template.spec.containers[0].image` from `v1.0` to `v2.0` using `kubectl set image`. Five minutes later, none of the running pods show `v2.0`. Is this a bug?**

No, this is expected, documented behavior. `kubectl set image rs/...`
updates the `template` field STORED inside the ReplicaSet object —
but ReplicaSet does NOT propagate template changes to already-running
pods. The new template only takes effect for pods CREATED AFTER the
change — typically only when an existing pod is deleted/crashes and
the controller creates a replacement. If no pods have been deleted or
crashed in those five minutes, all pods remain on `v1.0`, exactly as
expected.

---

**Q9. SCENARIO: You delete ALL pods belonging to a ReplicaSet simultaneously (e.g., via `kubectl delete pods -l app=myapp`), right after updating the template to a new image. What is the end result, and is there any downside compared to deleting pods one at a time?**

End result: all replacement pods will be on the NEW image, since
there are no old-template survivors left for comparison. However,
the downside is a brief period where ZERO pods are running — because
all old pods were deleted BEFORE any replacement pod was created by
the reconciliation loop. This causes a real service outage, even
though the final state (all pods on the new image) might look
identical to the result of a clean rolling update. Deleting pods ONE
AT A TIME would have kept at least one old pod serving traffic at
all times during the transition — though ReplicaSet still provides
no AUTOMATIC mechanism to do this gradually; you'd have to do it
manually, pod by pod. This exact problem (avoiding the all-at-once
gap) is what Deployment's `maxSurge`/`maxUnavailable` rolling update
strategy automates and guarantees.

---

**Q10. SCENARIO: Two different teams each create a ReplicaSet in the same namespace. Team A's ReplicaSet has selector `app: api`. Team B's ReplicaSet ALSO has selector `app: api` (an accidental naming collision), with `replicas: 2` each. What happens?**

Both ReplicaSets will be watching and counting the SAME set of pods
(any pod labeled `app: api`), but each thinks it should control the
count independently, both wanting 2. This typically results in
unstable, conflicting behavior — for example, if Team A's
ReplicaSet creates 2 pods first, Team B's ReplicaSet will see 2
EXISTING pods matching its selector (even though it didn't create
them) and think its job is already done, OR if both create pods
around the same time, the combined count could exceed 2+2=4, causing
ONE of the controllers to start deleting pods (potentially ones
created by the OTHER team) to bring the count down to its own desired
value of 2. This is why label uniqueness across different
applications/teams within a namespace is critical, and why most
organizations enforce naming conventions (such as including a unique
app name or team prefix in every label) to prevent this kind of
collision.

---

### TROUBLESHOOTING QUESTIONS

---

**Q11. TROUBLESHOOTING: `kubectl get rs myapp-rs` shows `DESIRED: 3, CURRENT: 3, READY: 1`. What does this tell you, and what would you check next?**

`DESIRED` and `CURRENT` matching (both 3) means the ReplicaSet
controller has successfully created the right NUMBER of pod objects
— from the controller's perspective, its job (maintaining pod count)
is done. However, `READY: 1` means only 1 of those 3 pods is
actually passing its READINESS PROBE — the other 2 likely exist (so
they count toward CURRENT) but are stuck in a state like
ContainerCreating, CrashLoopBackOff, or Running-but-not-Ready.

```bash
# Check individual pod statuses
kubectl get pods -l app=myapp -o wide

# Investigate the non-Ready ones specifically
kubectl describe pod <problematic-pod-name>
kubectl logs <problematic-pod-name>
```

This distinction matters because ReplicaSet's job is only to
maintain POD COUNT — it has no awareness of or responsibility for
whether those pods are actually healthy/serving traffic; that's a
separate concern handled by readiness probes and Service endpoints.

---

**Q12. TROUBLESHOOTING: You run `kubectl delete rs myapp-rs` expecting to remove the ReplicaSet but keep troubleshooting its pods independently. After the command, all the pods are ALSO gone. How do you avoid this in the future?**

By default, `kubectl delete rs <name>` CASCADES the deletion — it
deletes the ReplicaSet AND all pods it owns (identified via owner
references). To delete ONLY the ReplicaSet object while leaving its
pods running (orphaned, no longer managed by any controller), use:

```bash
kubectl delete rs <name> --cascade=orphan
```

After this, the pods continue running but are no longer
self-healing — if one crashes, nothing will replace it, since there's
no longer a controller watching them.

---

**Q13. TROUBLESHOOTING: A ReplicaSet shows `DESIRED: 5` but `CURRENT: 3`, and has stayed that way for over 10 minutes. What's your investigation process?**

This means the controller WANTS to create 2 more pods to reach 5,
but something is preventing it.

```bash
# Step 1 -- check the ReplicaSet's own events
kubectl describe rs <name>
# Look at the Events section at the bottom for messages like
# "Error creating: pods... is forbidden" or quota-related errors

# Step 2 -- check if a ResourceQuota in the namespace is blocking
# pod creation
kubectl describe resourcequota -n <namespace>

# Step 3 -- check for any pods stuck in Pending (these WOULD
# count toward CURRENT once created, but might be stuck even
# earlier, never even reaching pod-object creation)
kubectl get pods -l <selector> -n <namespace>

# Step 4 -- check node capacity, in case insufficient resources
# are preventing scheduling for pods that DID get created
kubectl describe nodes | grep -A 5 "Allocated resources"
```

Common root causes: a ResourceQuota in the namespace has been
reached (preventing new pod object creation entirely), or pods ARE
being created but stuck Pending due to insufficient cluster
resources (in which case CURRENT might actually show 5 with some in
Pending state, rather than CURRENT showing 3 — so seeing CURRENT
stuck at 3 specifically points more toward a quota or RBAC permission
issue blocking the CREATE operation itself, which I'd confirm via the
Events in step 1).

---

**Q14. TROUBLESHOOTING: You need to update the image used by a ReplicaSet's pods, with ZERO downtime, and the ReplicaSet has no built-in rolling update capability. What are your options, ranked from worst to best practice?**

Worst — delete all pods at once and let the controller recreate them
with the new template: causes a guaranteed brief outage (zero pods
running momentarily), as demonstrated by simultaneously deleting all
pods.

Better but manual and risky — delete pods ONE AT A TIME, waiting for
each replacement to become Ready before deleting the next: avoids
the all-at-once outage, but requires manual scripting/timing, has no
automatic rollback if the new image is broken, and you must do this
every single time you want to update.

Best practice — don't manage this with a bare ReplicaSet at all; use
a Deployment instead, which manages ReplicaSets FOR you and provides
this exact gradual-update capability natively via
`spec.strategy.rollingUpdate` (`maxSurge`/`maxUnavailable`), plus
automatic rollback via `kubectl rollout undo` if the new version
turns out to be broken. In real production environments, you almost
never create a ReplicaSet directly for this reason — you create a
Deployment, and Kubernetes creates and manages the ReplicaSet(s) on
your behalf.

---

*Next: Deployment — builds directly on top of everything in this
file.*
