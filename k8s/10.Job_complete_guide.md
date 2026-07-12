# Job — Theory, Labs, and Deep Dive Explanation

> Level: 4 Years Experience
> Author: Muskan Patel

---

## PART 1 — THEORY

---

## 1. Why Job Exists — The Problem No Other Workload Solves

Every workload you have learned so far shares one assumption: the
pod should run forever. If it stops, something went wrong — recreate
it immediately.

But consider this real scenario from a banking deployment day. You
have just updated the payment service from version 1.2 to version
1.3. Version 1.3 has a database schema change — a new column needs
to be added to the transactions table before the new application
code can work.

You need to run this SQL script ONCE:

```sql
ALTER TABLE transactions ADD COLUMN currency VARCHAR(3);
```

Which workload do you use?

**Not a Pod** — no retry logic. If it fails (database temporarily
unavailable), it sits in Failed state forever. Nobody notices.

**Not a Deployment** — if the migration script exits with code 0
(success), Deployment thinks "pod died, recreate it immediately."
Your migration runs again and again in an infinite loop, trying to
add a column that already exists — causing errors every single time.

**Not a DaemonSet** — runs on every node. You only need to run this
once, not once per node.

**Not a StatefulSet** — needs stable identity and storage. This is
a one-time script.

You need something that says: "Run this task. If it succeeds, mark
it DONE and stop forever. If it fails, retry up to N times. If it
still fails, mark it FAILED so someone can investigate."

That is a **Job**.

---

## 2. The Delivery Package Analogy

Think of a Job like a delivery company trying to deliver a package.

The delivery company (Job controller) has ONE instruction:
"Deliver this package successfully."

```
Attempt 1: Driver goes to address -- nobody home -- FAILED
           (wait, try again with exponential delay)

Attempt 2: Driver goes again -- door locked -- FAILED
           (wait, try again)

Attempt 3: Driver goes again -- customer opens door -- SUCCESS

Job marks itself COMPLETE.
Driver goes home.
Nobody sends another driver.
The job is DONE -- permanently.
```

If after N attempts (backoffLimit) the package is still not
delivered, the company gives up and marks the job FAILED — someone
needs to investigate why.

The delivered package is NEVER "undelivered" again just because time
passed. Complete means complete, forever.

---

## 3. What Happens When You Apply a Job — Step by Step

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
spec:
  completions: 1
  parallelism: 1
  backoffLimit: 3
  template:
    spec:
      containers:
      - name: migration
        image: payment-app:v1.2
        command: ["python", "migrate.py"]
      restartPolicy: Never
```

1. API server stores the Job object in etcd.

2. The Job Controller (dedicated controller inside controller-manager,
   separate from ReplicaSet, Deployment, StatefulSet controllers)
   reads `completions: 1` — needs exactly 1 successful pod
   completion.

3. Counts current successful completions: 0. Not done yet.

4. Counts currently active pods: 0. Should have 1 running
   (parallelism: 1). Creates ONE pod.

5. Pod runs, script executes.

6. If script exits with code 0 (success):
   - Successful completion count = 1
   - Equals completions: 1 (desired)
   - Job marks itself Complete
   - No more pods ever created. Done.

7. If script exits with non-zero code (failure):
   - Job controller sees a failed pod
   - Has backoffLimit: 3 been reached? No (attempt 1)
   - Creates a NEW pod for retry attempt 2
   - Delay before retry: exponential backoff (10s, 20s, 40s...)
   - If still failing after backoffLimit retries: Job = Failed

---

## 4. The Critical Difference — `restartPolicy: Never` vs `OnFailure`

### `restartPolicy: Never`

```
Pod runs, container fails (exit code 1)
         |
         v
kubelet does NOT restart the container
Pod stays in Failed state on that node permanently
         |
         v
Job Controller sees a failed pod
Job Controller creates a BRAND NEW pod for the retry
         |
         v
Old failed pod stays around for inspection
kubectl logs <old-failed-pod> -- logs still readable
```

Use Never when you want to inspect each failed attempt separately
— each retry is its own pod object with its own logs. Answer
questions like "why did attempt 2 fail but attempt 3 succeed?"

### `restartPolicy: OnFailure`

```
Pod runs, container fails (exit code 1)
         |
         v
kubelet RESTARTS the container IN-PLACE on same pod
Same pod object, same node, RESTARTS counter goes up
         |
         v
Job Controller sees pod is still active (just restarting)
Job Controller does NOT create a new pod
         |
         v
Previous attempt logs are GONE -- overwritten by new restart
Only current attempt logs are visible
```

Use OnFailure when you want fewer pod objects cluttering the
namespace and do not need to inspect each individual attempt
separately.

### The One Rule That Never Changes

Job pods CANNOT use `restartPolicy: Always`. The API server rejects
it outright. "Always restart" means "restart forever even on
success" — the exact opposite of what a Job needs. If you write
`restartPolicy: Always` in a Job's pod template, you will see:

```
The Job "db-migration" is invalid:
spec.template.spec.restartPolicy: Unsupported value: "Always"
```

---

## 5. The Three Key Fields — Explained With Real Numbers

### `completions`

How many SUCCESSFUL pod completions are needed before the Job is
considered Done.

```yaml
completions: 1    # one success = job done (default)
completions: 10   # need 10 successful completions total
```

Real use case for completions > 1: you have 10,000 customer records
to process, split into 10 batches. A Job with `completions: 10`
needs 10 successful pod runs before the overall job is complete.

### `parallelism`

How many pods can run SIMULTANEOUSLY at any given moment.

```yaml
parallelism: 1    # one at a time, sequential (default)
parallelism: 3    # up to 3 pods running at the same time
```

The combination of completions and parallelism determines the
pattern:

| completions | parallelism | Pattern |
|---|---|---|
| 1 | 1 | Single task, runs once |
| 10 | 1 | 10 tasks, one at a time (sequential) |
| 10 | 5 | 10 tasks, 5 running at once (parallel batch) |
| 10 | 10 | All 10 run simultaneously |

Banking example — end-of-month statement generation for 1000
customers: `completions: 1000, parallelism: 50` — 50 pods running
simultaneously, each generating one customer statement, until all
1000 are done. Much faster than sequential.

### `backoffLimit`

How many times the Job retries after failure before giving up.

```yaml
backoffLimit: 3   # tries 4 times total (1 initial + 3 retries)
backoffLimit: 0   # no retries -- fail once = Job Failed immediately
backoffLimit: 6   # default value if not specified
```

Retry delay is EXPONENTIAL — not immediate:

```
Attempt 1 fails -> wait 10s  -> attempt 2
Attempt 2 fails -> wait 20s  -> attempt 3
Attempt 3 fails -> wait 40s  -> attempt 4
Attempt 4 fails -> backoffLimit exhausted -> Job = Failed
```

This exponential backoff exists because many failures are transient
(database temporarily unavailable, network blip). Waiting longer
between retries gives the dependency time to recover.

### `activeDeadlineSeconds`

Total time limit for the entire Job — overrides backoffLimit.

```yaml
activeDeadlineSeconds: 300   # kill job after 5 minutes total
```

Even if backoffLimit says "retry 6 times," if total elapsed time
exceeds activeDeadlineSeconds, the Job is terminated immediately.
Useful for preventing a stuck job from running indefinitely.

Failure reason will be `DeadlineExceeded` (not
`BackoffLimitExceeded`) when this triggers.

### `ttlSecondsAfterFinished`

Auto-delete the Job (and its pods) this many seconds after it
reaches Complete or Failed state.

```yaml
ttlSecondsAfterFinished: 3600   # delete 1 hour after finishing
```

Without this, finished Jobs stay in the namespace forever, cluttering
`kubectl get jobs` output and consuming etcd space. Set it long
enough that you have time to check results before cleanup happens.

---

## 6. Line by Line — Full Job YAML Explained

```yaml
apiVersion: batch/v1
# WHAT: batch API group, version 1
# WHY batch: Job and CronJob represent "run to completion" workloads
# -- a completely different category from "run forever" workloads
# (apps/v1). Kubernetes separates these into different API groups
# because their controllers, lifecycle, and guarantees are
# fundamentally different.

kind: Job
# WHAT: tells API server this is a Job object
# WHY: uses the Job Controller which tracks success/failure counts
# and retries -- NOT the ReplicaSet controller which just maintains
# N running pods forever with no completion tracking.

metadata:
  name: db-migration
  namespace: banking

spec:
  completions: 1
  # WHAT: number of SUCCESSFUL pod completions required
  # WHY: defines "done." Until this many pods have exited with
  # code 0, the Job is not Complete. After this many succeed,
  # the Job stops creating new pods permanently.

  parallelism: 1
  # WHAT: max pods running simultaneously
  # WHY: controls throughput vs resource usage tradeoff.
  # parallelism: 1 for sequential tasks (database migration --
  # only ONE migration should run at a time, ever).
  # parallelism: 50 for batch processing (statement generation
  # -- safe to run many in parallel).

  backoffLimit: 3
  # WHAT: max number of RETRIES after failure
  # WHY: transient failures should be retried automatically.
  # But infinite retries hide real bugs -- if migration fails
  # 4 times, something is genuinely wrong, alert the team.
  # backoffLimit: 0 = no retries, for tasks where retry would
  # cause data corruption or duplicate processing.

  activeDeadlineSeconds: 300
  # WHAT: total time limit for the entire Job in seconds
  # WHY: safety net. A migration taking more than 5 minutes is
  # probably stuck. Kill it and alert rather than run forever.
  # Takes priority over backoffLimit -- if deadline hits first,
  # Job terminates regardless of remaining retries.

  ttlSecondsAfterFinished: 3600
  # WHAT: auto-delete Job and pods this many seconds after finish
  # WHY: finished Jobs accumulate in namespace over time,
  # cluttering output and consuming etcd space.
  # 3600 = delete 1 hour after completion.

  template:
    spec:
      containers:
      - name: migration
        image: payment-app:v1.2
        # WHAT: image containing the task script
        # WHY this specific image: migration scripts are typically
        # packaged into the same image as the application itself --
        # they use the same database drivers, ORM models, and
        # connection config. Keeps everything consistent.

        command: ["python", "migrate.py"]
        # WHAT: overrides the container image's default CMD
        # WHY: the payment-app image probably starts the web server
        # by default. We override to run ONLY the migration script,
        # not the full application.

        env:
        - name: DB_HOST
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: host
        # WHAT: database connection details pulled from a Secret
        # WHY: migration script needs to connect to the database.
        # Pulling from Secret keeps credentials out of YAML and Git.

        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "256Mi"
        # WHY: migration scripts can be memory-intensive.
        # Limits prevent a runaway migration from consuming all
        # node resources. Requests ensure scheduler places pod
        # on a node with enough capacity.

      restartPolicy: Never
      # WHAT: tells kubelet NOT to restart this container if it exits
      # WHY Never (not OnFailure): for a database migration, you want
      # each failed attempt to be a SEPARATE pod so you can check
      # logs of each individual attempt independently.
      # With OnFailure, kubelet restarts SAME pod in-place,
      # overwriting the previous attempt's logs.
      # restartPolicy: Always is REJECTED by API server for Jobs.
```

---

## 7. Job vs Deployment — The Key Comparison

| | Job | Deployment |
|---|---|---|
| Pod exits with code 0 | SUCCESS — job is done | Pod "died" — recreate immediately |
| Pod exits with code 1 | FAILURE — retry up to backoffLimit | Pod "died" — recreate immediately |
| Tracks "did it succeed?" | YES — completions counter | NO — just keeps N pods running |
| Has a concept of "done"? | YES — Complete state | NO — runs forever by design |
| Use case | One-time tasks, batch processing | Long-running services |
| restartPolicy | Never or OnFailure only | Always only |

---

## PART 2 — HANDS-ON LABS

---

## Lab 1 — Successful Job — Watch the Complete Lifecycle

```bash
kubectl apply -f - << 'EOF'
apiVersion: batch/v1
kind: Job
metadata:
  name: job-success
spec:
  completions: 1
  backoffLimit: 2
  template:
    spec:
      containers:
      - name: task
        image: busybox
        command:
        - sh
        - -c
        - |
          echo "Starting migration task..."
          sleep 5
          echo "Migration complete"
          exit 0
      restartPolicy: Never
EOF

# Watch the job progress
kubectl get job job-success -w
# COMPLETIONS column goes from 0/1 to 1/1
# Ctrl+C once Complete

# Check the final status
kubectl get job job-success
# COMPLETIONS: 1/1    STATUS: Complete

# Check the pod that ran the task
kubectl get pods -l job-name=job-success
# STATUS: Completed (not Running, not restarted)

# Step 1 -- get the pod name
kubectl get pods -l job-name=job-success
# e.g. job-success-x7k2p

# Step 2 -- read the pod logs
kubectl logs job-success-x7k2p
# "Starting migration task..."
# "Migration complete"

# Describe the job -- see completion time
kubectl describe job job-success

kubectl delete job job-success
```

---

## Lab 2 — Failed Job — Watch backoffLimit in Action

```bash
kubectl apply -f - << 'EOF'
apiVersion: batch/v1
kind: Job
metadata:
  name: job-failure
spec:
  completions: 1
  backoffLimit: 2
  template:
    spec:
      containers:
      - name: task
        image: busybox
        command:
        - sh
        - -c
        - |
          echo "Attempting task..."
          echo "Database connection failed"
          exit 1
      restartPolicy: Never
EOF

# Watch pods appear -- one per retry attempt
kubectl get pods -l job-name=job-failure -w
# You will see MULTIPLE pods created, one at a time with delays:
# job-failure-abc12  Error
# job-failure-def34  Error   (retry 1, after ~10 second delay)
# job-failure-ghi56  Error   (retry 2, after ~20 second delay)
# After 3rd failure: backoffLimit reached, no more pods created
# Ctrl+C

# Check final job status
kubectl get job job-failure
# COMPLETIONS: 0/1    STATUS: Failed

# Describe job -- see the failure reason
kubectl describe job job-failure
# Conditions section shows:
# Type=Failed, Reason=BackoffLimitExceeded

# ALL failed pods are KEPT for inspection (this is why Never > OnFailure)
kubectl get pods -l job-name=job-failure
# Shows all 3 pods (1 initial + 2 retries) in Error state

# Step 1 -- note all pod names from the output above
# e.g. job-failure-abc12, job-failure-def34, job-failure-ghi56

# Step 2 -- check logs of each attempt independently
kubectl logs job-failure-abc12
kubectl logs job-failure-def34
kubectl logs job-failure-ghi56
# Each shows "Attempting task..." and "Database connection failed"
# With OnFailure policy you could only see the LAST attempt

kubectl delete job job-failure
```

---

## Lab 3 — Parallel Batch Processing

```bash
kubectl apply -f - << 'EOF'
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-job
spec:
  completions: 6
  parallelism: 2
  backoffLimit: 2
  template:
    spec:
      containers:
      - name: worker
        image: busybox
        command:
        - sh
        - -c
        - |
          echo "Processing batch on pod $HOSTNAME"
          sleep 5
          echo "Batch done"
          exit 0
      restartPolicy: Never
EOF

# Watch -- NEVER more than 2 pods Running at once
# but eventually 6 TOTAL Completed
kubectl get pods -l job-name=batch-job -w
# batch-job-abc12  Running   (pod 1)
# batch-job-def34  Running   (pod 2 -- started same time)
# (wait 5 seconds -- both complete)
# batch-job-ghi56  Running   (pod 3 -- started immediately after)
# batch-job-jkl78  Running   (pod 4)
# ...continues until 6 total completions
# Ctrl+C

# Check job progress
kubectl get job batch-job
# COMPLETIONS shows progress: 2/6, 4/6, 6/6

# Final state
kubectl describe job batch-job
# Succeeded: 6

kubectl delete job batch-job
```

---

## Lab 4 — `activeDeadlineSeconds` — Kill a Stuck Job

```bash
kubectl apply -f - << 'EOF'
apiVersion: batch/v1
kind: Job
metadata:
  name: stuck-job
spec:
  completions: 1
  backoffLimit: 5
  activeDeadlineSeconds: 15
  template:
    spec:
      containers:
      - name: task
        image: busybox
        command:
        - sh
        - -c
        - |
          echo "Starting... this will take forever"
          sleep 3600
      restartPolicy: Never
EOF

# Watch -- job gets killed after 15 seconds
# regardless of backoffLimit: 5 saying it could retry 5 more times
kubectl get job stuck-job -w
# After 15 seconds STATUS changes to Failed
# Ctrl+C

# Check why it failed
kubectl describe job stuck-job
# Conditions:
# Type=Failed, Reason=DeadlineExceeded
# NOT BackoffLimitExceeded -- deadline hit FIRST
# This is the key difference between the two failure reasons

kubectl delete job stuck-job
```

---

## Lab 5 — `ttlSecondsAfterFinished` — Auto Cleanup

```bash
kubectl apply -f - << 'EOF'
apiVersion: batch/v1
kind: Job
metadata:
  name: auto-cleanup
spec:
  completions: 1
  backoffLimit: 0
  ttlSecondsAfterFinished: 20
  template:
    spec:
      containers:
      - name: task
        image: busybox
        command: ["sh", "-c", "echo Task done; exit 0"]
      restartPolicy: Never
EOF

# Job completes almost immediately
kubectl get job auto-cleanup
# COMPLETIONS: 1/1

kubectl get pods -l job-name=auto-cleanup
# STATUS: Completed

# Wait 20 seconds then check again
sleep 25

kubectl get job auto-cleanup
# Error from server (NotFound)
# Job was automatically deleted after 20 seconds

kubectl get pods -l job-name=auto-cleanup
# No resources found
# Pods were also deleted along with the Job

# Important: logs are also gone after TTL deletion
# Set ttlSecondsAfterFinished long enough to check results
# before auto-cleanup happens
```

---

## Lab 6 — `restartPolicy: Never` vs `OnFailure` — See the Difference

```bash
# Option A -- restartPolicy: Never
# Each failed attempt = separate pod object
kubectl apply -f - << 'EOF'
apiVersion: batch/v1
kind: Job
metadata:
  name: policy-never
spec:
  completions: 1
  backoffLimit: 2
  template:
    spec:
      containers:
      - name: task
        image: busybox
        command: ["sh", "-c", "exit 1"]
      restartPolicy: Never
EOF

sleep 30
kubectl get pods -l job-name=policy-never
# You see MULTIPLE pod objects (one per attempt)
# policy-never-abc12  Error
# policy-never-def34  Error
# policy-never-ghi56  Error
# 3 separate pods, 3 separate log streams

kubectl delete job policy-never

# Option B -- restartPolicy: OnFailure
# All retries happen inside ONE pod object
kubectl apply -f - << 'EOF'
apiVersion: batch/v1
kind: Job
metadata:
  name: policy-onfailure
spec:
  completions: 1
  backoffLimit: 2
  template:
    spec:
      containers:
      - name: task
        image: busybox
        command: ["sh", "-c", "exit 1"]
      restartPolicy: OnFailure
EOF

sleep 30
kubectl get pods -l job-name=policy-onfailure
# You see only ONE pod object
# policy-onfailure-xyz99  Error  RESTARTS: 2
# One pod, but it restarted 2 times -- all retries in same pod
# Only the CURRENT attempt's logs are visible

kubectl delete job policy-onfailure
```

---

## Quick Reference — Job Commands

```bash
# Create
kubectl apply -f job.yaml
kubectl create job <name> --image=<image> -- <command>

# Get
kubectl get jobs
kubectl get job <name>

# Watch progress
kubectl get job <name> -w
kubectl describe job <name>

# See pods created by a job
kubectl get pods -l job-name=<name>

# Get the pod name then check logs
kubectl get pods -l job-name=<name>
kubectl logs <pod-name>
kubectl logs <pod-name> --previous

# Delete job (also deletes its pods by default)
kubectl delete job <name>

# Delete job but KEEP its pods for inspection
kubectl delete job <name> --cascade=orphan

# Manually trigger a job run immediately
kubectl create job <name>-manual --from=job/<name>
```

---

## Summary — What Makes Job Unique

1. Has a concept of SUCCESS and FAILURE — not just "running" or "crashed"
2. Stops permanently once required completions are reached
3. Retries on failure up to backoffLimit with exponential backoff delay
4. activeDeadlineSeconds provides a hard time limit overriding backoffLimit
5. restartPolicy CANNOT be Always — must be Never or OnFailure
6. With Never: each retry is a separate pod (all logs preserved)
7. With OnFailure: all retries happen in same pod (only last attempt's logs)
8. ttlSecondsAfterFinished auto-cleans finished Jobs from the namespace
9. parallelism + completions enables powerful batch processing patterns

---

*Next: CronJob — Job + a schedule*
