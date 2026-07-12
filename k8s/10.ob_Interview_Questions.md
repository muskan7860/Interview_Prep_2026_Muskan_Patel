# Job — Interview Questions

> Level: 4 Years Experience
> Author: Muskan Patel
> Covers: Theory, Scenario, Troubleshooting

---

## HIGH PRIORITY — Almost Guaranteed to Be Asked

---

**Q1. What is a Kubernetes Job and why does it exist?**

A Job is a Kubernetes workload controller designed for tasks that
need to RUN TO COMPLETION — not run forever like a Deployment or
DaemonSet. It runs one or more pods, tracks whether each one
succeeded or failed, retries on failure up to a configured limit,
and once the required number of successful completions is reached,
it stops permanently and marks itself Complete.

It exists because no other workload type handles "run once and stop"
correctly. A Deployment treats a pod exiting with code 0 (success)
as a crash — it immediately recreates the pod, running your
one-time script in an infinite loop. A bare Pod has no retry logic
— one failure and it's done, no one is notified. Job fills this gap
with success/failure tracking, configurable retries, and a clear
terminal state.

---

**Q2. What is the difference between a Job and a Deployment when a pod exits with exit code 0?**

Deployment: treats exit code 0 as "pod died unexpectedly" and
immediately recreates it. The Deployment has no concept of
"success" — it just wants N pods running at all times. A script
that exits successfully would be restarted forever in an infinite
loop.

Job: treats exit code 0 as SUCCESS — increments the successful
completions counter. When completions counter reaches
`spec.completions`, the Job marks itself Complete and never creates
another pod. This is the fundamental design difference — Job has a
concept of "done," Deployment does not.

---

**Q3. What restartPolicy values are valid for a Job, and why is `Always` rejected?**

Valid values for a Job's pod template: `Never` or `OnFailure`.

`Always` is rejected by the API server outright with a validation
error. The reason: `Always` means "restart this container even if
it exits with code 0 (success)" — the exact opposite of a Job's
purpose. A Job needs to STOP when the task succeeds. If you could
set `restartPolicy: Always`, a successfully completed Job would
restart its container immediately, defeating the entire concept of
"run to completion."

```
# What the API server returns if you try Always:
The Job "db-migration" is invalid:
spec.template.spec.restartPolicy: Unsupported value: "Always"
```

---

**Q4. What is the difference between `restartPolicy: Never` and `restartPolicy: OnFailure` for a Job?**

Both allow retries on failure, but the MECHANISM is different.

`Never`: when the container fails, kubelet does NOT restart it. The
pod stays in Failed state permanently on that node. The Job
Controller then creates a BRAND NEW pod object for the retry.
Result: each retry attempt is a separate pod with its own name and
its own log stream. You can inspect all attempts independently with
`kubectl logs`.

`OnFailure`: when the container fails, kubelet RESTARTS the container
in-place within the SAME pod. The pod's RESTARTS counter increments.
Result: one pod object, multiple restart cycles. Only the current
(latest) restart's logs are visible — previous attempt logs are
overwritten.

Use `Never` when: you need to debug individual failure attempts
separately, or when you want a clear audit trail of how many times
the task was attempted and why each one failed.

Use `OnFailure` when: you want fewer pod objects in the namespace
and don't need to inspect each attempt individually.

---

**Q5. Explain `completions` and `parallelism` with a real banking example.**

`completions`: how many SUCCESSFUL pod completions are needed before
the Job is considered Done.

`parallelism`: how many pods can run SIMULTANEOUSLY at any given
moment.

Banking example — end-of-month statement generation for 1000
customers:

```yaml
completions: 1000
parallelism: 50
```

The Job Controller creates 50 pods simultaneously (parallelism: 50).
As each pod finishes successfully (processes one customer's
statement), the controller creates a new pod to keep 50 running at
all times — until 1000 total successful completions are reached
(completions: 1000). Total time is approximately 1000/50 = 20 batch
cycles, much faster than running 1000 pods sequentially.

For a database migration that must run exactly once:

```yaml
completions: 1
parallelism: 1
```

One pod, one attempt needed, one success marks it done.

---

**Q6. What is `backoffLimit` and what is the retry timing pattern?**

`backoffLimit` defines the maximum number of RETRIES after failure
before the Job gives up and marks itself Failed. The default value
is 6 if not specified.

```yaml
backoffLimit: 3
# Means: 1 initial attempt + 3 retries = 4 total attempts maximum
```

The retry timing follows EXPONENTIAL BACKOFF — not immediate
retries:

```
Attempt 1 fails -> wait 10 seconds  -> attempt 2
Attempt 2 fails -> wait 20 seconds  -> attempt 3
Attempt 3 fails -> wait 40 seconds  -> attempt 4
Attempt 4 fails -> backoffLimit (3) exhausted -> Job = Failed
Condition: BackoffLimitExceeded
```

The exponential delay exists because most failures are transient
(database temporarily unavailable, network blip). Waiting longer
between retries gives the dependency time to recover before the
next attempt. Immediate retries would likely hit the same transient
failure again and exhaust the backoffLimit faster.

`backoffLimit: 0` means no retries — if the task fails once, the
Job is immediately Failed. Used for tasks where retrying would cause
data corruption or duplicate processing.

---

**Q7. What is `activeDeadlineSeconds` and how is it different from `backoffLimit`?**

`activeDeadlineSeconds`: a total elapsed TIME limit for the entire
Job, measured from when the Job starts. If this time is exceeded,
the Job and all its active pods are immediately terminated,
regardless of how many retries remain.

`backoffLimit`: a total ATTEMPT COUNT limit. Controls how many times
the task can fail before giving up, regardless of how much time
has passed.

Key difference: whichever limit is hit FIRST wins.

Example: `backoffLimit: 6` (allows 7 attempts) and
`activeDeadlineSeconds: 60` (allows 60 seconds total). If the
task fails twice in 60 seconds and is about to start its third
attempt, but 60 seconds has already elapsed — the Job is
terminated immediately. The failure reason will be
`DeadlineExceeded`, not `BackoffLimitExceeded`.

Use case for `activeDeadlineSeconds`: a database migration that
normally takes 2 minutes but should never take more than 10 minutes.
If it's still running at 10 minutes, something is wrong (long-running
lock, infinite loop in script) — kill it and alert the team rather
than letting it run indefinitely.

---

## THEORY QUESTIONS

---

**Q8. What is `ttlSecondsAfterFinished` and why is it important?**

`ttlSecondsAfterFinished` automatically deletes the Job object (and
all its pods) a specified number of seconds after the Job reaches
a terminal state (Complete or Failed).

```yaml
ttlSecondsAfterFinished: 3600   # delete 1 hour after finishing
```

Without this, every finished Job stays in the namespace forever —
`kubectl get jobs` becomes cluttered with hundreds of completed
migration jobs from every deployment, and etcd accumulates objects
that serve no ongoing purpose.

Important consideration: when the Job is deleted via TTL, its pod
logs are also deleted. Set `ttlSecondsAfterFinished` long enough
that you have time to verify results and read logs before cleanup.
For critical tasks like database migrations, set it to several hours
or even days (86400 = 24 hours) to ensure the audit trail is
available.

---

**Q9. What is the Job Controller's relationship to pods — does it use a ReplicaSet underneath like Deployment?**

No — the Job Controller creates and manages pods DIRECTLY, with
no ReplicaSet in between. This is similar to StatefulSet (which
also manages pods directly), but for different reasons.

The ReplicaSet controller's core assumption is "all pods are
identical and should run forever" — it has no concept of
success/failure tracking or completion counts. The Job Controller
needs to track HOW MANY pods succeeded (completions counter) and
HOW MANY failed (for backoffLimit) — logic that ReplicaSet simply
does not have and was not designed for. So Job has its own
dedicated controller that directly manages pods.

```bash
# Prove this -- no ReplicaSet exists for a Job
kubectl get jobs
kubectl get rs -l job-name=<name>
# No resources found -- no RS in between
kubectl get pods -l job-name=<name>
# Pods exist directly, managed by Job Controller
```

---

**Q10. How do you know whether a Job failed due to `backoffLimit` being exceeded vs `activeDeadlineSeconds` being exceeded?**

Check the Job's conditions via `kubectl describe job <name>`. The
`Reason` field in the Failed condition tells you exactly which
limit was hit:

```bash
kubectl describe job <name>
```

```
Conditions:
  Type    Status  Reason
  Failed  True    BackoffLimitExceeded
  # OR
  Failed  True    DeadlineExceeded
```

`BackoffLimitExceeded` means the task was attempted
`backoffLimit + 1` times and all attempts failed.

`DeadlineExceeded` means the total elapsed time exceeded
`activeDeadlineSeconds` before enough successful completions
were reached — could happen even if no individual attempt failed
(e.g., a single pod ran too long without completing).

---

## SCENARIO QUESTIONS

---

**Q11. SCENARIO: You use a Deployment to run a database migration script. After deployment, your database has the schema change applied correctly. But you notice the migration pod is restarting repeatedly and getting errors like "column already exists." What happened and what should you use instead?**

The Deployment treated the migration pod's successful exit
(code 0 after adding the column) as a crash — its ReplicaSet
controller sees the pod count drop to 0 and immediately creates a
new pod. The new pod runs the migration script again, tries to add
a column that already exists, gets a database error, exits with
code 1 — and the cycle repeats.

The Deployment has no concept of "this pod succeeded and should not
be restarted" — it only knows "pod is not running, create another."

The correct solution is a Job with `completions: 1`. When the
migration pod exits with code 0, the Job controller increments the
success counter to 1, sees it equals completions (1), marks itself
Complete, and never creates another pod. The migration runs exactly
once and stops.

---

**Q12. SCENARIO: Your banking Job for end-of-month report generation has `backoffLimit: 3` and `restartPolicy: Never`. It fails all 4 attempts. How many pod objects exist in the namespace, and how do you investigate which attempt failed and why?**

With `restartPolicy: Never`, each retry creates a SEPARATE, NEW pod
object. So with 1 initial attempt + 3 retries = 4 total attempts,
there are 4 pod objects in the namespace, all in Error/Failed state.

```bash
# See all 4 pods
kubectl get pods -l job-name=<job-name>
# e.g:
# report-gen-abc12   0/1   Error   0   10m
# report-gen-def34   0/1   Error   0   8m
# report-gen-ghi56   0/1   Error   0   5m
# report-gen-jkl78   0/1   Error   0   2m

# Check the Job's overall failure reason
kubectl describe job <job-name>
# Conditions: Failed, Reason=BackoffLimitExceeded

# Investigate each attempt's logs independently
kubectl logs report-gen-abc12
kubectl logs report-gen-def34
kubectl logs report-gen-ghi56
kubectl logs report-gen-jkl78
```

If the error is the same across all 4 attempts — it is likely a
configuration issue (wrong database credentials, wrong script path),
not a transient failure. Fix the root cause, then delete and
recreate the Job.

If the first 2 fail with "connection refused" and the last 2 fail
differently — the exponential backoff gave the dependency time to
recover but something else went wrong. Each pod's individual logs
tell the full story — this is EXACTLY why `restartPolicy: Never`
is preferred for debugging over `OnFailure`.

---

**Q13. SCENARIO: You need to process 500 payment reconciliation records. Each record takes about 2 seconds to process. How would you design the Job spec to complete this in under 30 seconds?**

500 records at 2 seconds each = 1000 seconds sequentially
(completions: 500, parallelism: 1).

To complete in under 30 seconds: need to process at least
500/30 ≈ 17 records per second simultaneously.
With 2 seconds per record: need at least 34 pods running in
parallel.

```yaml
spec:
  completions: 500
  parallelism: 50
  backoffLimit: 3
  activeDeadlineSeconds: 60
  template:
    spec:
      containers:
      - name: reconciler
        image: payment-reconciler:v1.2
        command: ["python", "reconcile.py"]
      restartPolicy: OnFailure
```

With `parallelism: 50` and 2 seconds per record: 500/50 = 10
batches of 50 pods each, 10 × 2 seconds = 20 seconds total.
Well under 30 seconds.

`activeDeadlineSeconds: 60` provides a safety net — if something
goes wrong and it's still running at 60 seconds, kill it and alert.
`restartPolicy: OnFailure` used here (not Never) because with 500
pods we do not need to debug individual attempts — overall failures
are tracked at the Job level.

---

**Q14. SCENARIO: A Job has `backoffLimit: 6` and `activeDeadlineSeconds: 30`. The task takes 10 seconds to run. It fails on the first attempt (at the 10-second mark). Before the second retry can happen (exponential backoff means waiting 10 seconds), the Job is marked Failed at the 30-second deadline. The team says "but we still had 5 more retries available!" Is this correct behaviour?**

Yes, this is correct and intentional behaviour. `activeDeadlineSeconds`
is a HARD wall-clock deadline for the ENTIRE Job lifecycle, and it
takes absolute priority over `backoffLimit`. The two limits work
independently — whichever is hit first terminates the Job.

Timeline in this scenario:
- 0s: Job starts, attempt 1 begins
- 10s: attempt 1 fails, exponential backoff timer starts (10s wait)
- 20s: retry attempt 2 WOULD start (backoff timer expired)
- 30s: activeDeadlineSeconds hit — Job terminated

The failure reason will be `DeadlineExceeded`, not
`BackoffLimitExceeded`. The team is correct that 5 retries were
unused, but `activeDeadlineSeconds: 30` said "this Job must be
done within 30 seconds, no matter what" — and it was not.

The fix if more retries are needed within the time limit: either
increase `activeDeadlineSeconds` to give more time, or reduce the
backoff by decreasing `backoffLimit` and relying on
`activeDeadlineSeconds` as the primary timeout. In this scenario,
`activeDeadlineSeconds: 120` would have given time for multiple
retry attempts.

---

**Q15. SCENARIO: You want a Job that runs database migration on every deployment. The migration should run once per deployment and never again. How do you trigger it from your CI/CD pipeline?**

The most reliable pattern is to create a NEW Job object for each
deployment with a unique name (e.g., using the git commit SHA or
deployment timestamp):

```bash
# In your CI/CD pipeline (Jenkins/GitHub Actions)
DEPLOY_VERSION=$(git rev-parse --short HEAD)

kubectl apply -f - << EOF
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration-${DEPLOY_VERSION}
  namespace: banking
spec:
  completions: 1
  backoffLimit: 3
  activeDeadlineSeconds: 300
  ttlSecondsAfterFinished: 86400
  template:
    spec:
      containers:
      - name: migration
        image: payment-app:${DEPLOY_VERSION}
        command: ["python", "migrate.py"]
      restartPolicy: Never
EOF

# Wait for the migration to complete before deploying the app
kubectl wait job/db-migration-${DEPLOY_VERSION} \
  --for=condition=complete \
  --timeout=360s
```

`kubectl wait --for=condition=complete` blocks the pipeline until
the Job succeeds — only then does the pipeline continue to deploy
the application. If the Job fails or times out, the pipeline stops
and the application deployment does not proceed, preventing a broken
application from being deployed against an un-migrated database
schema.

---

## TROUBLESHOOTING QUESTIONS

---

**Q16. TROUBLESHOOTING: `kubectl get job` shows STATUS as Failed and COMPLETIONS as 0/1. How do you investigate and what are the common root causes?**

```bash
# Step 1 -- check the failure reason
kubectl describe job <job-name>
# Conditions section shows:
# Type=Failed, Reason=BackoffLimitExceeded (too many failed attempts)
# OR
# Type=Failed, Reason=DeadlineExceeded (ran too long)

# Step 2 -- find the pods that ran
kubectl get pods -l job-name=<job-name>
# Shows all attempt pods and their states

# Step 3 -- check logs of each failed pod
# Get pod name from step 2, e.g. migration-abc12
kubectl logs migration-abc12
kubectl logs migration-abc12 --previous

# Step 4 -- describe a failed pod for more detail
kubectl describe pod migration-abc12
# Check Events section and Last State for exit code and reason
```

Common root causes by failure reason:

`BackoffLimitExceeded`:
- Wrong database credentials (connection refused or auth failure)
- Script bug causing consistent non-zero exit
- Missing dependency (config file, environment variable not set)
- Wrong image or wrong command specified

`DeadlineExceeded`:
- Database has a long-running lock blocking the migration
- Script is stuck in an infinite loop
- activeDeadlineSeconds set too short for the actual task duration

---

**Q17. TROUBLESHOOTING: You delete a Job that has running pods. After deletion, the pods are still running. Why, and how do you clean them up?**

By default, `kubectl delete job <name>` cascades the deletion to
all pods owned by that Job. However, if the pods are in the middle
of a long-running task, they may take a moment to terminate
(graceful shutdown period). If you are seeing pods still running
immediately after deletion, wait a few seconds — they should
terminate shortly.

If you used `--cascade=orphan` flag:

```bash
kubectl delete job <name> --cascade=orphan
```

This deletes the Job object but intentionally LEAVES all its pods
running as orphaned pods (no longer managed by any controller).
To clean them up manually:

```bash
# Find the orphaned pods (no longer have job-name label managed)
kubectl get pods -l job-name=<name>

# Delete them explicitly
kubectl delete pod <pod-name-1> <pod-name-2>
```

`--cascade=orphan` is useful when you want to delete and recreate
the Job configuration (e.g., fix a bug in the spec) without
interrupting the currently-running pod that is mid-task.

---

**Q18. TROUBLESHOOTING: A Job was running fine in staging but fails immediately in production with exit code 1. The logs show "connection refused" on the very first attempt. What do you check?**

Exit code 1 with "connection refused" on the FIRST attempt (not
after retries) in production but not staging points to an
environment-specific configuration issue, not an application bug.

```bash
# Step 1 -- check the Job's environment variables and secrets
kubectl describe job <job-name> -n production
# Look at environment variables in the container spec

# Step 2 -- verify the Secret the Job references exists
# and contains the correct values in the production namespace
kubectl get secret db-secret -n production
kubectl describe secret db-secret -n production

# Step 3 -- check if the Secret value is correct
# (remember: Secrets are base64 encoded, not encrypted)
kubectl get secret db-secret -n production -o yaml
# Decode the host value:
echo "<base64-value>" | base64 -d
# Confirm it points to the PRODUCTION database, not staging

# Step 4 -- check network connectivity
# Can pods in the production namespace reach the database?
kubectl run debug -n production --image=busybox -it --rm \
  -- sh -c "nc -zv <db-host> 5432"
# If this also shows "connection refused", the issue is network
# policy or security group blocking traffic, not the Job config

# Step 5 -- compare staging vs production namespace secrets
kubectl get secret db-secret -n staging -o yaml
kubectl get secret db-secret -n production -o yaml
# Look for differences in the host, port, or credentials
```

---

**Q19. TROUBLESHOOTING: Your Job has `ttlSecondsAfterFinished: 300` (5 minutes). An incident happens 10 minutes after the Job completed. You need to check the Job's logs to investigate but they are gone. How do you prevent this in future?**

The Job and its pods were automatically deleted 5 minutes after
completion via TTL — logs are permanently gone with them.

Immediate workarounds for this incident: check if your logging
system (EFK stack, Cloudwatch Logs, Splunk) captured the pod logs
before deletion. In production banking environments, log collectors
(Fluentd/Fluent Bit DaemonSets) ship all pod logs to a central
system in real time — pod deletion does not remove logs from the
central store.

Prevention for future:

Option 1 — increase TTL:
```yaml
ttlSecondsAfterFinished: 86400   # keep for 24 hours
```

Option 2 — remove TTL entirely and rely on periodic manual cleanup
or a dedicated cleanup CronJob:
```yaml
# Remove ttlSecondsAfterFinished from the spec entirely
# Jobs stay until manually deleted or cleaned by policy
```

Option 3 — ensure your logging infrastructure (Fluentd DaemonSet
→ Elasticsearch) captures and retains all pod logs independently
of pod/Job lifecycle. This is the correct production approach —
logs should NOT depend on the pod still existing.

---

**Q20. TROUBLESHOOTING: A Job shows `COMPLETIONS: 3/10` and has been in this state for 2 hours. No pods are currently running. `kubectl describe job` shows no active pods and no recent events. What happened?**

Three completions happened successfully, then the Job stopped
creating new pods and entered a stuck state with no activity.
The most likely cause: the remaining pods are failing and
`backoffLimit` was reached, stopping ALL further pod creation.

```bash
# Step 1 -- check the Job conditions
kubectl describe job <job-name>
# Look at Conditions section
# Type=Failed, Reason=BackoffLimitExceeded
# This confirms retries were exhausted

# Step 2 -- find all pods including failed ones
kubectl get pods -l job-name=<job-name>
# You will see:
# 3 pods in Completed state (the 3 successes)
# Multiple pods in Error state (the failed retries)

# Step 3 -- check logs of the Error pods
kubectl get pods -l job-name=<job-name>
# Note the names of Error pods
kubectl logs <error-pod-name>
# What error caused these to fail?

# Step 4 -- check if failures are consistent or intermittent
# If all Error pods show the same error message:
# --> systematic bug (wrong input data, config issue, code bug)
# If Error pods show different errors:
# --> likely transient issues that exhausted the backoffLimit
```

After fixing the root cause, delete the failed Job and create a
new one. The 3 already-completed records should NOT be reprocessed
-- ensure your task is idempotent (safe to run on already-processed
records, returns success without duplicating work) OR track
completion state in the database so the new Job skips already-done
records.

---

*Run Labs 1, 2, 4, and 6 before interviews.
Lab 2 (watching multiple retry pods appear one by one) makes Q4,
Q6, and Q12 permanently clear.
Lab 4 (activeDeadlineSeconds killing a stuck job) makes Q7 and
Q14 click instantly because you personally watched the deadline
override the remaining retries.*
