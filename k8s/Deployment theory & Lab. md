# Deployment — Theory and Labs (Easy Commands Only)

> Level: 4 Years Experience
> Author: Muskan Patel
> No JSON patch commands — only standard kubectl commands you can
> actually remember and use confidently in an interview

---

## PART 1 — THEORY

---

## 1. Why Deployment Exists — Picking Up From ReplicaSet

You already discovered, in the ReplicaSet labs, the biggest gap: when
you change a ReplicaSet's template, NOTHING happens to existing pods.
You have to manually delete pods one by one to roll out a change, and
deleting them all at once causes a real outage.

Now imagine this is your bank's payment service in production,
running 20 replicas of version `1.2`. You need to ship `1.3`. With a
bare ReplicaSet, you would have to manually delete one pod, wait, check
it's healthy, delete the next — twenty times — and if `1.3` turns out
broken halfway through, manually figure out how to undo it.

This is not realistic for a real team shipping changes daily.
Kubernetes needed an object that says: "Give me the new desired
state, and I will handle the ENTIRE transition — gradually, safely,
with a record of every version, and the ability to instantly undo it."
That object is the **Deployment**.

---

## 2. The Mental Model — Three Layers, One Job Each

**Deployment never touches Pods directly. Ever.** It only ever talks
to ReplicaSets. ReplicaSets talk to Pods.

```
Deployment  -->  manages  -->  ReplicaSet  -->  manages  -->  Pod
(version history,             (headcount maintenance,        (actual running
 rollout strategy)              self-healing)                  containers)
```

- **Pod** — runs containers
- **ReplicaSet** — maintains a COUNT of identical pods (you already
  mastered this)
- **Deployment** — manages TRANSITIONS between different VERSIONS of
  that ReplicaSet over time

This separation is WHY rollback works at all — old ReplicaSets (with
old templates) are kept around, scaled down to 0, ready to be scaled
back up instantly if needed.

---

## 3. What Happens When You Create a Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-deployment
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

1. API server stores this Deployment object in etcd.
2. The Deployment Controller (separate code from the ReplicaSet
   controller, also inside controller-manager) notices this new
   Deployment.
3. It computes a short fingerprint (hash) of `spec.template` — called
   `pod-template-hash`. Think of it as a barcode generated from the
   container image, env vars, and everything else under `template`.
   If the template text changes even slightly, this barcode changes
   completely.
4. The Deployment Controller does NOT create pods itself. It creates
   a ReplicaSet object, automatically named
   `payment-deployment-<hash>`, with the hash also added as an extra
   label on the ReplicaSet's selector and on every pod it creates.
5. The ReplicaSet Controller (the one you already know) takes over
   from here — sees this ReplicaSet wants 3 pods, creates them,
   exactly like before.

```bash
kubectl get deployment payment-deployment
kubectl get rs -l app=payment
kubectl get pods -l app=payment --show-labels
```

Notice the pod naming pattern is now THREE parts:
`<Deployment-name>-<hash>-<random-suffix>` — one extra layer compared
to a bare ReplicaSet's `<name>-<random-suffix>`.

---

## 4. Why the Hash Gets Added to the Selector (Important Concept)

Recall the "label selector trap" from ReplicaSet — a ReplicaSet
counts ANY pod matching its selector, regardless of who created it.

If TWO ReplicaSets (old version and new version, during an update)
both had the SAME selector `app: payment`, they would FIGHT — each
would see ALL pods (old AND new mixed together) as "mine," and one
might delete the WRONG pods trying to correct its count.

By adding `pod-template-hash` to each ReplicaSet's selector, every
ReplicaSet ONLY counts pods carrying its OWN specific hash. Old
version pods are invisible to the new ReplicaSet's counting, and vice
versa. They never collide, and can each scale independently.

```
Old ReplicaSet selector: app=payment, pod-template-hash=abc123
New ReplicaSet selector: app=payment, pod-template-hash=xyz789
```

You never write this hash yourself — Kubernetes adds it
automatically, in three places: the ReplicaSet's own label, the
ReplicaSet's selector, and every pod's label.

---

## 5. The Rolling Update — What Happens Step by Step

When you change the image (you'll do this with a simple command in
the lab below), here is EXACTLY what happens internally:

```
Old ReplicaSet: currently 3 pods, image v1.2
New ReplicaSet: currently 0 pods, image v1.3 (just created)

maxSurge: 1, maxUnavailable: 0, desired total: 3
Max allowed total pods at once = 3 + 1 = 4
Min required healthy pods at all times = 3 - 0 = 3

Step 1: scale NEW up by 1   -> old=3, new=1  (total=4)
        WAIT for the new pod to pass its readiness probe
Step 2: scale OLD down by 1 -> old=2, new=1  (total=3)
Step 3: scale NEW up by 1   -> old=2, new=2  (total=4)
        WAIT for readiness
Step 4: scale OLD down by 1 -> old=1, new=2  (total=3)
Step 5: scale NEW up by 1   -> old=1, new=3  (total=4)
        WAIT for readiness
Step 6: scale OLD down by 1 -> old=0, new=3  (total=3, DONE)
```

The old ReplicaSet is NOT deleted — it stays at 0 replicas, sitting in
history, ready to be scaled back up instantly for a rollback.

**The most important sentence to remember:** "WAIT for the new pod to
pass its readiness probe" is a literal BLOCKING step. If your app
takes 60 seconds to start but your readiness probe only allows 5
seconds before checking, the probe fails repeatedly, and the ENTIRE
rollout STALLS right there — not failed, just paused — until the pod
becomes Ready or you intervene manually.

---

## 6. Rolling Update Settings — maxSurge and maxUnavailable

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
```

- `maxSurge` — how many EXTRA pods (above your desired count) are
  allowed to exist temporarily during the rollout
- `maxUnavailable` — how many FEWER pods (below your desired count)
  are allowed to exist temporarily during the rollout

Both can be a plain number (`1`) or a percentage (`25%`). With
`replicas: 20` and `maxUnavailable: 25%`, that means up to 5 pods
can be missing at once during a rollout.

**The common production setting for a zero-downtime guarantee:**
```yaml
maxSurge: 1
maxUnavailable: 0
```
Never drop below your desired healthy count (zero downtime), but
allow ONE extra pod temporarily so the new version gets a chance to
prove it's healthy before any old pod is removed.

**The other strategy type — Recreate:**
```yaml
spec:
  strategy:
    type: Recreate
```
Kills ALL old pods FIRST, then creates ALL new ones. Causes
guaranteed downtime, but guarantees the old and new version are NEVER
running at the same time — used when your app cannot tolerate two
versions running simultaneously (for example, a breaking database
migration on startup).

---

## 7. Rollback — How It Actually Works

Rollback is NOT a special magic feature. It is the EXACT SAME rolling
update mechanism, just run in reverse — Kubernetes scales the OLD
(previously working) ReplicaSet back UP and the CURRENT (broken)
ReplicaSet back DOWN, using the same surge/unavailable rules.

```yaml
spec:
  revisionHistoryLimit: 10   # default
```
This controls how many OLD, scaled-to-0 ReplicaSets are kept around.
Beyond this number, the oldest ones are deleted permanently — you can
no longer roll back to versions older than this limit.

---

## 8. Pause and Resume — Manual Control

You can FREEZE a rollout mid-way through, inspect things manually,
then either continue or back out — useful for manual canary testing
(checking ONE new pod before letting the rest roll out).

---

## PART 2 — HANDS-ON LABS (EASY COMMANDS ONLY)

---

## Lab 1 — See the Three-Layer Chain Form

**Industry context:** Every time your team runs `kubectl apply` on a
Deployment manifest in a CI/CD pipeline, this exact chain happens
automatically — you've never seen it because the tooling does it for
you in seconds. This lab slows it down so you can see each layer.

```bash
# Create a deployment -- watch what gets created underneath it
kubectl create deployment web --image=nginx:1.24 --replicas=3
```
This ONE command creates a Deployment object, which immediately
triggers the Deployment Controller to create a ReplicaSet, which
immediately creates 3 pods.

```bash
kubectl rollout status deployment/web
```
This does NOT change anything — it just WATCHES and reports live
progress, then exits once the deployment is fully up. This is the
exact command a real CI/CD pipeline runs right after deploying, to
know whether the deploy succeeded.

```bash
# See all THREE layers that now exist
kubectl get deployment web
kubectl get rs -l app=web
kubectl get pods -l app=web --show-labels
```
Look at the pod labels — you'll see `pod-template-hash=<some value>`
sitting there, even though you never typed that label yourself.
Look at the ReplicaSet name — it's `web-<hash>`, automatically
generated.

```bash
kubectl delete deployment web
```

---

## Lab 2 — Watch a Real Rolling Update Happen (Two Terminals)

**Industry context:** This simulates your team shipping a tested,
working new version of an application to production, with a strict
requirement of ZERO dropped requests during the deploy.

### Step 1 — Create the "current production" deployment

```bash
kubectl create deployment rollout-demo --image=nginx:1.24 --replicas=4
kubectl rollout status deployment/rollout-demo
```

Confirm your starting point:
```bash
kubectl get pods -l app=rollout-demo
```
You should see 4 Running pods. This is your BEFORE state.

### Step 2 — Set the rollout rule using `kubectl edit` (no JSON needed)

```bash
kubectl edit deployment rollout-demo
```

This OPENS the Deployment's full YAML in your terminal's default text
editor (usually `vi` or `nano`). Scroll down and find the `strategy:`
section. It will look something like:

```yaml
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
```

Change the two values to:

```yaml
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
    type: RollingUpdate
```

Save and exit (in `vi`: press `Esc`, then type `:wq` and press
Enter). The change is applied IMMEDIATELY — `kubectl edit` is just a
convenient way to fetch the live YAML, let you change a field by
hand, then automatically apply your edited version back. This is the
SAME end result as `kubectl patch`, but in a format you can actually
read and remember, instead of a long JSON string.

**Verify the change took effect:**
```bash
kubectl get deployment rollout-demo -o yaml | grep -A 3 rollingUpdate
```

### Step 3 — Open a SECOND terminal to watch live

In a NEW terminal window (keep the first one open too):

```bash
kubectl get rs -l app=rollout-demo -w
```

The `-w` flag means "watch" — this terminal will stay running and
print a new line EVERY TIME a ReplicaSet's replica count changes.
Leave this visible — it's your live camera for the next step. Right
now you should see ONE line showing 4/4/4.

### Step 4 — Trigger the update (back in your FIRST terminal)

```bash
kubectl set image deployment/rollout-demo nginx=nginx:1.25
```

This is a SIMPLE, memorable command — much easier than JSON. It says:
"in the Deployment named `rollout-demo`, find the container named
`nginx`, and change its image to `nginx:1.25`." This is the exact
moment your CI/CD pipeline would trigger after your code passes
tests and merges to `main`.

### Step 5 — Watch your SECOND terminal now

You should see new lines appearing over the next several seconds,
something like:

```
NAME                          DESIRED   CURRENT   READY   AGE
rollout-demo-<old-hash>       4         4         4       3m
rollout-demo-<new-hash>       0         0         0       0s
rollout-demo-<new-hash>       1         0         0       0s
rollout-demo-<new-hash>       1         1         0       1s
rollout-demo-<new-hash>       1         1         1       8s
rollout-demo-<old-hash>       3         4         4       3m
rollout-demo-<old-hash>       3         3         3       3m
... (continues, surge-then-shrink, until)
rollout-demo-<old-hash>       0         0         0       4m
rollout-demo-<new-hash>       4         4         4       40s
```

You are literally WATCHING the surge-then-shrink dance from the
theory section — bump the new one up by 1, wait for READY to catch
up (notice the delay between CURRENT and READY — that's the
readiness probe being checked), then drop the old one down by 1,
repeat. Notice: at no point does (old READY) + (new READY) drop below
4 — that is your `maxUnavailable: 0` guarantee, happening live in
front of you.

### Step 6 — Confirm it's done (back in your FIRST terminal)

```bash
kubectl rollout status deployment/rollout-demo
```
Should print immediately: `deployment "rollout-demo" successfully
rolled out`

```bash
kubectl get pods -l app=rollout-demo
```
All pods should have BRAND NEW names (different random suffixes —
these are entirely new pod objects) and all should be on `1.25`.

Stop the watch terminal (Ctrl+C), then:

```bash
kubectl get rs -l app=rollout-demo
kubectl delete deployment rollout-demo
```

---

## Lab 3 — A Bad Deploy and Rollback (Real Incident Simulation)

**Industry context:** This is the scenario every DevOps engineer
eventually lives through — you ship a new version, it's broken
(typo'd image tag, crashing app), and you need to detect it fast and
get back to the last known-good version.

### Step 1 — Set up "yesterday's good state" and "today's good deploy"

```bash
kubectl create deployment bad-demo --image=nginx:1.24 --replicas=3
kubectl rollout status deployment/bad-demo
```

```bash
kubectl set image deployment/bad-demo nginx=nginx:1.25
kubectl rollout status deployment/bad-demo
```

Both should complete successfully. You now have TWO revisions in
history: revision 1 (`1.24`, now at 0 replicas) and revision 2
(`1.25`, currently active).

```bash
kubectl rollout history deployment/bad-demo
```

This lists every revision number the Deployment has gone through.

### Step 2 — Simulate the broken deploy (the incident)

```bash
kubectl set image deployment/bad-demo nginx=nginx:doesnotexist999
```

This simulates your team accidentally deploying a typo'd image tag
or a version that was never pushed to the registry — a very common
real-world mistake.

```bash
kubectl rollout status deployment/bad-demo --timeout=20s
```

The `--timeout=20s` flag means: "if the rollout hasn't finished
within 20 seconds, STOP waiting and report an error instead of
hanging forever." This is EXACTLY how a CI/CD pipeline automatically
detects a failed deploy — the pipeline step running this command
gets a non-zero exit code on timeout, which it interprets as
"deployment failed," triggering an alert.

You'll see it eventually print an error and stop.

```bash
kubectl get pods -l app=bad-demo
```

You'll see a MIX — some OLD pods still `Running` on `1.25`
(untouched), and ONE new pod stuck in `ImagePullBackOff`. This is the
safety net: because the new pod can never become Ready, Kubernetes is
BLOCKED by `maxUnavailable` from removing any of the healthy old
pods. Your "customers" experienced zero impact — the deploy got
STUCK, not rolled out.

### Step 3 — The rollback (the fix)

```bash
kubectl rollout undo deployment/bad-demo
```

This is NOT a special "magic undo." It triggers the SAME rolling
update mechanism, except instead of you specifying a new image,
Kubernetes automatically figures out "what was the PREVIOUS working
ReplicaSet's template?" and rolls FROM the current broken state BACK
to that one — scaling the broken ReplicaSet down and the previous
working one back up.

```bash
kubectl rollout status deployment/bad-demo
```

This should complete successfully now.

```bash
kubectl get deployment bad-demo -o jsonpath='{.spec.template.spec.containers[0].image}'
echo
```

Confirm it prints `nginx:1.25` — you went back to the LAST WORKING
version, not all the way back to the very first version.

### Step 4 — Rolling back to a SPECIFIC, older revision

```bash
kubectl rollout history deployment/bad-demo
```

```bash
kubectl rollout undo deployment/bad-demo --to-revision=1
```

This is for when you need to skip back MULTIPLE versions at once —
not just "undo the last change," but "jump directly to this exact
known-good revision number," useful when the last few deploys were
ALL subtly broken in different ways.

```bash
kubectl get deployment bad-demo -o jsonpath='{.spec.template.spec.containers[0].image}'
echo
```

```bash
kubectl delete deployment bad-demo
```

---

## Lab 4 — Pause and Resume (Manual Canary Testing)

**Industry context:** Sometimes a team wants to test a new version on
a SMALL slice of real traffic before committing to a full rollout —
this is a simple manual way to do that.

```bash
kubectl create deployment canary-demo --image=nginx:1.24 --replicas=4
kubectl rollout status deployment/canary-demo
```

```bash
kubectl rollout pause deployment/canary-demo
```

This FREEZES the Deployment Controller's reconciliation loop for
this object — it stops adjusting anything, leaving the current state
exactly as-is, even after you trigger a change next.

```bash
kubectl set image deployment/canary-demo nginx=nginx:1.25
```

Even though you just triggered an update, NOTHING visibly happens
yet, because the controller is paused.

```bash
kubectl get rs -l app=canary-demo
```

You'll see a NEW ReplicaSet exists, but with 0 replicas — frozen,
waiting.

```bash
kubectl get rs -l app=canary-demo
```

Note the name of the NEWEST ReplicaSet (it'll be the one with 0
replicas), then scale just that one up manually to 1:

```bash
kubectl scale rs <paste-the-new-replicaset-name-here> --replicas=1
```

```bash
kubectl get pods -l app=canary-demo
```

You should see 4 old pods PLUS 1 new pod — your manual canary, live,
while the other 4 keep serving on the old version.

Once you're satisfied the canary looks healthy, let the Deployment
Controller take back over and finish properly:

```bash
kubectl rollout resume deployment/canary-demo
kubectl rollout status deployment/canary-demo
```

```bash
kubectl delete deployment canary-demo
```

---

## Quick Reference — Deployment Commands (All Easy to Remember)

```bash
# Create / apply
kubectl create deployment <name> --image=<image> --replicas=N
kubectl apply -f deployment.yaml

# Inspect
kubectl get deployment <name>
kubectl describe deployment <name>
kubectl get rs -l app=<label>
kubectl get pods -l app=<label> --show-labels

# Edit settings by hand (no JSON needed)
kubectl edit deployment <name>

# Scale
kubectl scale deployment <name> --replicas=5

# Update the image (the actual "ship it" command)
kubectl set image deployment/<name> <container-name>=<new-image>

# Watch / confirm rollout progress
kubectl rollout status deployment/<name>
kubectl rollout status deployment/<name> --timeout=30s

# History and rollback
kubectl rollout history deployment/<name>
kubectl rollout history deployment/<name> --revision=2
kubectl rollout undo deployment/<name>
kubectl rollout undo deployment/<name> --to-revision=2

# Manual pause/resume
kubectl rollout pause deployment/<name>
kubectl rollout resume deployment/<name>

# Cleanup
kubectl delete deployment <name>
```

---

*Run Lab 2 with the two-terminal setup before moving on — actually
watching the second terminal scroll live while you run `set image`
in the first is what makes maxSurge/maxUnavailable permanent in your
memory.*
