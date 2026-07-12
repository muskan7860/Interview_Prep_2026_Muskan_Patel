# Deployment — Interview Questions (Theory, Scenario, Troubleshooting)

> Level: 4 Years Experience
> Author: Muskan Patel

---

## THEORY QUESTIONS

---

**Q1. What is a Deployment, and how is it different from a ReplicaSet?**

A Deployment is a controller that manages ReplicaSets, providing
version transitions over time. A ReplicaSet's only job is maintaining
a fixed COUNT of identical pods — it has no concept of "updating" a
running set of pods gradually. A Deployment adds the missing piece:
when you change the pod template (e.g., a new image), the Deployment
Controller creates a NEW ReplicaSet and gradually shifts pod count
from the OLD ReplicaSet to the NEW one, following a configurable
strategy (RollingUpdate or Recreate), while keeping old ReplicaSets
around at 0 replicas for instant rollback. In short: ReplicaSet
maintains COUNT, Deployment manages VERSION TRANSITIONS.

---

**Q2. Explain the three-layer relationship between Deployment, ReplicaSet, and Pod, and why this layering exists.**

Deployment manages ReplicaSets; ReplicaSets manage Pods. A Deployment
NEVER creates or touches a Pod directly. Each layer has exactly one
responsibility: Pod runs containers, ReplicaSet maintains a count of
identical pods and provides self-healing, and Deployment manages
TRANSITIONS between different versions of that ReplicaSet over time.
This separation is what makes rollback possible — when you update a
Deployment, the OLD ReplicaSet is not deleted, just scaled to 0,
acting as a ready-to-restore snapshot of the previous working state.

---

**Q3. What is `pod-template-hash`? Who generates it, and what is it used for?**

`pod-template-hash` is a label automatically computed and added by
the Deployment Controller — a short fingerprint generated from the
contents of `spec.template`. It is added in three places: the
ReplicaSet's own label, the ReplicaSet's selector, and every pod
created by that ReplicaSet. Its purpose is to let each ReplicaSet
created by the same Deployment uniquely identify and count ONLY the
pods belonging to its specific template version, even while multiple
ReplicaSets (old and new versions) exist simultaneously during a
rollout. Without this, two ReplicaSets created from slightly
different templates but the same base selector would have identical
selectors and would incorrectly count and potentially delete each
other's pods.

---

**Q4. What are `maxSurge` and `maxUnavailable`? Explain with a concrete numeric example.**

`maxSurge` is the number (or percentage) of EXTRA pods allowed above
the desired replica count during a rollout. `maxUnavailable` is the
number (or percentage) of FEWER pods allowed below the desired
replica count during a rollout. With `replicas: 10, maxSurge: 2,
maxUnavailable: 1`: during the rollout, total pod count can briefly
go up to 12 (10+2), and can briefly go down to 9 (10-1). Setting
`maxUnavailable: 0` guarantees zero downtime, since the available
count never drops below the desired count, at the cost of requiring
new pods to pass readiness checks before any old pod is removed,
which can make the rollout slower.

---

**Q5. What is the difference between `RollingUpdate` and `Recreate` deployment strategies?**

`RollingUpdate` (the default) gradually replaces old pods with new
ones, controlled by `maxSurge`/`maxUnavailable`, maintaining
availability throughout the transition — used for almost all
stateless services. `Recreate` kills ALL old pods first, then creates
ALL new pods — causing guaranteed downtime, but guaranteeing that old
and new versions are NEVER running simultaneously. `Recreate` is used
when running two versions in parallel would cause a conflict, such
as an application that performs a breaking database schema migration
on startup that the old version's code cannot safely coexist with.

---

**Q6. Is `kubectl rollout undo` a special "magic" feature, or does it use the same mechanism as a normal update? Explain.**

It uses the EXACT SAME mechanism as any other update — it is not a
special feature. Old ReplicaSets are kept around at 0 replicas
specifically for this purpose. When you run `rollout undo`, the
Deployment Controller identifies the PREVIOUS (or a specifically
targeted) ReplicaSet's template, and then runs the SAME rolling
update logic in reverse — scaling the previous ReplicaSet's count UP
and the current (often broken) ReplicaSet's count DOWN, following
the same `maxSurge`/`maxUnavailable` rules as a forward update.

---

**Q7. What controls how many old ReplicaSets are retained for rollback purposes? What happens once that limit is exceeded?**

`spec.revisionHistoryLimit` (default 10) controls this. Once the
number of OLD, scaled-to-0 ReplicaSets exceeds this limit, the OLDEST
ones are garbage collected (deleted entirely) automatically. After
that point, you can no longer roll back to revisions beyond this
retained history — `kubectl rollout history` would no longer list
them, and `rollout undo --to-revision=<old-number>` would fail for a
revision that's been cleaned up.

---

**Q8. What is the difference between `kubectl rollout status` and `kubectl rollout history`?**

`kubectl rollout status` is a LIVE, blocking watcher — it connects to
the API server and prints real-time progress of an IN-PROGRESS or
just-triggered rollout, then exits once it reaches a stable state (or
times out if `--timeout` is specified and exceeded). `kubectl rollout
history` is a READ-ONLY, point-in-time listing of ALL past revisions
the Deployment has gone through, along with their `CHANGE-CAUSE` if
recorded — used to review what changed over time and to choose a
specific revision number to roll back to.

---

**Q9. Why might a real production team explicitly set `maxSurge` and `maxUnavailable` instead of relying on Kubernetes defaults?**

The Kubernetes default for both is `25%`. With small replica counts
(e.g., 4), this rounds to roughly 1, which often behaves reasonably
by coincidence. But with larger replica counts (e.g., 20), the
default `maxUnavailable: 25%` means up to 5 pods can be unavailable
simultaneously during a deploy — for a critical service like a
banking payment API, a team might explicitly require
`maxUnavailable: 0` to GUARANTEE zero capacity loss during every
deploy, accepting a somewhat slower rollout in exchange for that
hard guarantee, rather than trusting an implicit percentage-based
default that scales unpredictably with replica count.

---

**Q10. What does `kubectl rollout pause` actually do internally?**

It sets `spec.paused: true` on the Deployment object. The Deployment
Controller's reconciliation loop checks this flag — when `true`, it
stops adjusting the balance between old and new ReplicaSets entirely,
leaving the current state exactly as-is, even if you trigger further
template changes (like `kubectl set image`) while paused. This is
used for manual canary testing: pause immediately, manually scale just
the new ReplicaSet up by a small amount to test it on a slice of
traffic, then either `rollout resume` to let the controller finish
the rollout normally, or roll back if something looks wrong.

---

## SCENARIO QUESTIONS

---

**Q11. SCENARIO: You run `kubectl get rs` after several deployments and see a ReplicaSet with `DESIRED: 0, CURRENT: 0`. Is this a problem? Should you delete it?**

This is NOT a problem — it is expected. A ReplicaSet scaled to 0 is
an OLD VERSION's history, kept intentionally by the Deployment for
rollback purposes (governed by `revisionHistoryLimit`). It should
NOT be manually deleted unless you are certain you will never need to
roll back to that specific version — deleting it manually would
remove that rollback option, though the Deployment itself would
continue functioning normally for any FUTURE updates, since new
ReplicaSets are always created fresh as needed.

---

**Q12. SCENARIO: You trigger `kubectl set image` to deploy a new version. Thirty seconds later, you check `kubectl get pods` and see a MIX of pods on the old and new image, with the new ones stuck `0/1 Running` (not yet Ready). Is the rollout broken?**

Not necessarily broken — this could simply be IN PROGRESS, normal
behavior. The new pods showing `Running` but `0/1` Ready means their
containers started, but the readiness probe hasn't passed yet — this
is exactly the "WAIT for readiness" blocking step in the rolling
update process. The mix of old and new pods existing simultaneously
is also expected — that's the surge mechanism at work. To determine
if it's STUCK versus just SLOW, I'd run `kubectl rollout status
deployment/<name> --timeout=60s` — if it eventually completes, it was
just progressing normally; if it times out, I'd then investigate WHY
the new pods aren't becoming Ready (`kubectl describe pod` on one of
them, checking the Events section for readiness probe failures).

---

**Q13. SCENARIO: A team member accidentally runs `kubectl delete rs <name>` directly on the CURRENTLY ACTIVE ReplicaSet under a Deployment (not the Deployment itself). What happens?**

The Deployment Controller is STILL watching and will detect that its
expected ReplicaSet (matching the current `pod-template-hash`) no
longer exists, or now has 0 pods unexpectedly. Since the Deployment's
desired state (the current template) hasn't changed, the controller
will simply CREATE the ReplicaSet again (or a new one with the same
template, same hash) to match the still-active desired state, and
that new ReplicaSet will recreate the pods. In short: this causes a
brief, unnecessary disruption (pods get recreated, getting new
names/IPs) but is NOT catastrophic — the Deployment self-heals back
to the correct state because the SOURCE OF TRUTH (the Deployment's
template) was untouched; only the intermediate ReplicaSet object was
deleted.

---

**Q14. SCENARIO: Your team needs to deploy a change where the application CANNOT have two versions running at the same time, due to an incompatible database migration. Which Deployment field do you change, and what is the tradeoff?**

Set `spec.strategy.type: Recreate` instead of the default
`RollingUpdate`. The tradeoff: this guarantees old and new pods are
NEVER running simultaneously (solving the migration conflict), but it
causes GUARANTEED DOWNTIME — all old pods are terminated completely
before any new pod is created, meaning there is a window where ZERO
pods are serving traffic. This is an explicit, deliberate tradeoff:
sacrificing availability to guarantee version isolation, appropriate
only when running both versions together would cause actual data
corruption or conflicts, not just minor inconsistency.

---

**Q15. SCENARIO: You need to roll back NOT to the immediately previous version, but to a version from three deploys ago, because the last three deploys were each subtly broken in different ways. How do you do this, and what do you check first?**

```bash
# First, review the available history
kubectl rollout history deployment/<name>

# Confirm there's enough retained history to reach that far back
# (limited by spec.revisionHistoryLimit -- if it's been exceeded,
# that old revision may already be garbage collected)

# Then target that specific revision directly
kubectl rollout undo deployment/<name> --to-revision=<N>
```

I'd check `revisionHistoryLimit` first and confirm the target
revision still appears in `rollout history` output — if three
recent deploys happened in quick succession and the default limit
of 10 wasn't exceeded, the revision should still be available; if it
WAS exceeded or the limit was set lower, that revision may already
be gone, in which case I'd need to redeploy that known-good
configuration fresh rather than relying on stored rollback history.

---

## TROUBLESHOOTING QUESTIONS

---

**Q16. TROUBLESHOOTING: `kubectl rollout status deployment/payment-deployment` hangs indefinitely, never completing. Walk through your full investigation.**

```bash
# Step 1 -- see current pod mix
kubectl get pods -l app=payment

# Step 2 -- identify the NEW (stuck) ReplicaSet and inspect events
kubectl get rs -l app=payment
kubectl describe rs <new-replicaset-name>

# Step 3 -- describe one of the new, not-yet-ready pods directly
kubectl describe pod <new-pod-name>
# Check Events section for the exact failure reason

# Step 4 -- check logs if the container is at least starting
kubectl logs <new-pod-name>
```

Common root causes: new image doesn't exist (`ImagePullBackOff`),
new pod crashing on startup (`CrashLoopBackOff` — check logs for
the exception), or readiness probe misconfigured/failing (app
running fine but probe pointing to the wrong port/path, or app
genuinely needs more startup time than `initialDelaySeconds`
allows). Because `maxUnavailable` is blocking removal of old pods
until new ones are Ready, the rollout will hang indefinitely at
this exact point until the underlying issue is fixed or I run
`kubectl rollout undo`.

---

**Q17. TROUBLESHOOTING: After a `kubectl set image` deploy, you confirm via `kubectl rollout status` that it completed successfully, but customers report the application is still behaving like the OLD version. What would you check?**

```bash
# Step 1 -- confirm what image the Deployment OBJECT actually has
kubectl get deployment <name> -o jsonpath='{.spec.template.spec.containers[0].image}'

# Step 2 -- confirm the RUNNING pods actually match that image
kubectl get pods -l app=<label> \
  -o jsonpath='{range .items[*]}{.metadata.name}{"  "}{.spec.containers[0].image}{"\n"}{end}'

# Step 3 -- if pods DO show the new image but behavior is still old,
# check if the IMAGE TAG itself is the issue (e.g., using :latest
# where the registry's "latest" hasn't actually changed, or a CDN/
# cache serving stale static assets in front of the actual pods)

# Step 4 -- check if there's a Service/Ingress routing issue sending
# traffic somewhere unexpected, bypassing the updated pods entirely
kubectl get endpoints <service-name>
```

If `rollout status` reported success and the pods genuinely show the
new image, but behavior is unchanged, the issue is likely NOT inside
Kubernetes at all — it's more likely a caching layer (browser cache,
CDN, application-level cache) serving stale content, or the image tag
itself wasn't actually rebuilt with the intended code changes
(a CI/CD pipeline bug where the build step didn't pick up the latest
commit).

---

**Q18. TROUBLESHOOTING: You run `kubectl rollout undo deployment/<name>` expecting an instant fix, but it ALSO gets stuck, with new (rolled-back) pods not becoming Ready. What does this tell you, and what's your next step?**

This tells you the issue is likely NOT specific to the "bad" version
you were trying to escape from — if rolling BACK to a previously
working version ALSO fails to become Ready, the problem may be
EXTERNAL to the application code itself: a downstream dependency
(database, external API) that BOTH versions rely on may currently be
down, a resource constraint at the cluster/node level preventing ANY
new pod from starting properly (e.g., insufficient node capacity,
unrelated to which image is used), or a recent change to something
shared across versions, like a ConfigMap, Secret, or NetworkPolicy,
that's now blocking ALL pods regardless of their image version. Next
step: `kubectl describe pod` on one of the STUCK rollback pods to see
the exact failure reason, and cross-check whether it's
image-version-specific or something more fundamental affecting the
whole namespace/cluster.

---

**Q19. TROUBLESHOOTING: A Deployment shows `READY 3/3` in `kubectl get deployment`, but you suspect one of the underlying ReplicaSets is misbehaving. How do you inspect the ReplicaSet layer specifically, bypassing the Deployment's summarized view?**

```bash
# List ReplicaSets owned by this Deployment
kubectl get rs -l app=<label>

# Look specifically for MULTIPLE ReplicaSets with non-zero replicas
# at the same time -- this would indicate a STUCK rollout, not a
# completed one, even if the Deployment's top-level READY count
# looks satisfied by the SUM across both
kubectl get rs -l app=<label> -o wide

# Describe the ReplicaSet directly for its own Events
kubectl describe rs <specific-replicaset-name>
```

A healthy, fully-completed Deployment should show exactly ONE
ReplicaSet with non-zero replicas at any given time (all others at
0). If TWO ReplicaSets both show non-zero counts that SUM to the
Deployment's desired total, this can be misleadingly reported as
"3/3 Ready" at the Deployment level while actually representing a
RollingUpdate that's stuck halfway, with old and new versions both
permanently serving traffic side by side — worth catching explicitly
at the ReplicaSet layer, since the Deployment's summary view alone
wouldn't make this obvious.

---

**Q20. TROUBLESHOOTING: You used `kubectl edit deployment <name>` to change `maxUnavailable` from `0` to `50%` to "speed up" deploys, without telling anyone. A few weeks later, a deploy causes a noticeable customer-facing outage during the rollout window. Explain the connection, and what governance you'd put in place to prevent this.**

Setting `maxUnavailable: 50%` means during ANY future rollout, up to
HALF of the desired pod count can be unavailable simultaneously — for
a Deployment with, say, 10 replicas, that's 5 pods missing at once,
a 50% capacity drop during every deploy going forward, not just a
one-time change. This directly explains a customer-facing outage
during a later rollout — the safety guarantee that previously existed
(`maxUnavailable: 0`, zero capacity loss) was silently removed.
Governance to prevent this: store Deployment manifests in Git and use
a GitOps tool (ArgoCD/Flux) that continuously reconciles the live
cluster state back to what's declared in the repository — this means
any manual `kubectl edit` change would be automatically REVERTED on
the next sync cycle, and any INTENTIONAL change to a setting like
`maxUnavailable` would have to go through a pull request review
first, creating both an audit trail and a deliberate approval step
before such a risk-relevant setting could be changed.

---

*Run Lab 2 and Lab 3 from the Theory and Labs file again, this time
predicting OUT LOUD what each `kubectl` command will do BEFORE
running it, then verifying — this is what makes Q11-Q20 answerable
from real experience instead of memorization.*
