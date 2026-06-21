# CronJob — Theory, Labs, and Deep Dive Explanation

> Level: 4 Years Experience
> Author: Muskan Patel

---

## PART 1 — THEORY

---

## 1. What Is a CronJob — One Sentence

A CronJob is a Job factory with a clock built in — it creates a
brand new Job object automatically, on a schedule, repeatedly.

---

## 2. Why CronJob Exists

You already know Job: "run this task, track success/failure, stop
when done." Job answers WHAT to run and HOW MANY TIMES to retry.

But Job does not answer WHEN to run it automatically. If you need
to generate an end-of-day banking report at 2am every night, you
would have to manually `kubectl apply` a new Job every single day.
That is unacceptable.

CronJob adds exactly one thing on top of Job: automatic, recurring
scheduling. That is the entire difference.

Real use cases in banking:
- End-of-day financial report generation (2am every weekday)
- Database backup snapshots to S3 (every 6 hours)
- Log cleanup and rotation (every Sunday at midnight)
- Account statement generation (1st of every month)
- Certificate expiry checks (every day at 9am)
- Data reconciliation between systems (every 30 minutes)

---

## 3. The Office Cleaner Analogy

Think of a CronJob like an office building's cleaning schedule.

The building manager (CronJob controller) has a clipboard saying:
"Every weekday at 6pm, send a cleaning crew to Floor 3."

```
Monday 6pm    -> send crew -> they clean -> they leave -> DONE
Tuesday 6pm   -> send NEW crew -> they clean -> they leave -> DONE
Wednesday 6pm -> send NEW crew -> ...
```

Each cleaning session is independent — a new crew each time, a
fresh Job each time. The building manager only decides WHEN. The
crew (Job and its pods) handles HOW.

But what if Monday's crew is STILL working when Tuesday 6pm arrives?
The building manager needs a rule:
- Send Tuesday's crew anyway and both work simultaneously (Allow)
- Skip Tuesday entirely and wait for Wednesday (Forbid)
- Send Tuesday's crew and tell Monday's crew to stop immediately (Replace)

That rule is `concurrencyPolicy`.

---

## 4. The Three-Layer Chain

CronJob creates Jobs. Jobs create Pods. Pods run containers.

```
CronJob ("end-of-day-report")
  |
  | Every day at 2am, CronJob Controller creates:
  v
Job ("end-of-day-report-28401440")
  |
  | Job Controller creates pods, tracks completions:
  v
Pod ("end-of-day-report-28401440-x7k2p")
  |
  | kubelet starts the container:
  v
Container (runs generate_report.py)
```

This is a THREE-LAYER chain. Compare with:
- Deployment -> ReplicaSet -> Pod (three layers)
- StatefulSet -> Pod (two layers, direct)
- Job -> Pod (two layers, direct)
- CronJob -> Job -> Pod (three layers)

The CronJob controller ONLY creates Job objects. It never directly
creates or manages pods. Once a Job is created, the Job Controller
takes over completely — CronJob has no further involvement in that
specific run.

---

## 5. What Happens When You Apply a CronJob — Step by Step

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: end-of-day-report
spec:
  schedule: "0 2 * * *"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  startingDeadlineSeconds: 100
  jobTemplate:
    spec:
      backoffLimit: 2
      template:
        spec:
          containers:
          - name: report
            image: payment-app:v1.2
            command: ["python", "generate_report.py"]
          restartPolicy: OnFailure
```

1. API server stores the CronJob object in etcd.

2. The CronJob Controller (inside controller-manager, separate from
   the Job Controller) wakes up and checks the schedule every
   10 seconds.

3. Every 10 seconds it asks: "Based on the cron expression
   `0 2 * * *` and `status.lastScheduleTime`, has a new scheduled
   time passed since the last Job was created?"

4. If YES — it creates a new Job object, naming it
   `end-of-day-report-<timestamp>`, copying everything from
   `jobTemplate`.

5. That Job is picked up by the Job Controller — normal Job
   lifecycle begins (pods created, completions tracked, retries
   on failure).

6. CronJob controller records `status.lastScheduleTime` and waits
   for the next scheduled moment.

---

## 6. Cron Syntax — The Five Fields

```
"0 2 * * *"
 | | | | |
 | | | | +---- day of week  (0-7, 0 and 7 = Sunday)
 | | | +------ month        (1-12)
 | | +-------- day of month (1-31)
 | +---------- hour         (0-23)
 +------------ minute       (0-59)
```

Memory trick for the order: Minutes, Hours, Day, Month, Weekday

### Common Patterns to Memorise

```bash
"0 2 * * *"          every day at 2:00 AM
"*/5 * * * *"        every 5 minutes
"0 * * * *"          every hour on the hour
"0 0 * * *"          every day at midnight
"0 0 * * 0"          every Sunday at midnight
"0 9 * * 1-5"        9:00 AM Monday through Friday only
"0 0 1 * *"          first day of every month at midnight
"0 0 1 1 *"          every January 1st at midnight (yearly)
"*/30 9-17 * * 1-5"  every 30 minutes, 9am to 5pm, weekdays only
```

Tip: use crontab.guru online to verify any cron expression in
plain English before deploying.

---

## 7. Key Fields That Control CronJob Behaviour

### `concurrencyPolicy`

What to do if the previous Job is still running when the next
scheduled time arrives.

```yaml
concurrencyPolicy: Allow    # run both simultaneously (risky for reports)
concurrencyPolicy: Forbid   # skip the new run (safe for financial tasks)
concurrencyPolicy: Replace  # kill the old run, start the new one
```

Allow: both Jobs run simultaneously. Risk — for a financial report,
two processes writing to the same output table produce conflicting
numbers.

Forbid: new scheduled run is skipped entirely if previous is still
active. Status shows a missed schedule event. Safest for banking
report generation and database operations.

Replace: the currently-running Job is deleted immediately, new Job
starts. The old run is abandoned mid-execution. Use when data
freshness matters more than completeness — e.g., a health dashboard
that only needs the latest snapshot.

### `successfulJobsHistoryLimit` and `failedJobsHistoryLimit`

```yaml
successfulJobsHistoryLimit: 3   # keep last 3 successful Jobs
failedJobsHistoryLimit: 1       # keep last 1 failed Job
```

After each Job completes, the CronJob controller checks: how many
successful (or failed) Jobs from this CronJob currently exist?
If more than the limit, the OLDEST ones are automatically deleted.

Without these limits, a CronJob running every minute creates 1440
Job objects per day — cluttering `kubectl get jobs` and consuming
etcd space.

The asymmetry (3 successful, 1 failed) is intentional:
- 3 successful: enough history to compare whether recent runs
  completed normally
- 1 failed: failed jobs need immediate investigation — old failures
  that were not investigated are not actionable

### `startingDeadlineSeconds`

```yaml
startingDeadlineSeconds: 100
```

If the CronJob controller was unavailable (controller-manager
restarting, cluster outage) and MISSED the scheduled time, how
late is too late to still run this Job?

If the missed time was more than `startingDeadlineSeconds` seconds
ago, skip this run entirely. Do NOT run it late.

Example:
- Schedule: 2:00 AM
- Controller was down from 1:59 AM to 2:05 AM
- Controller recovers at 2:05 AM
- Time elapsed since scheduled run: 5 minutes = 300 seconds
- startingDeadlineSeconds: 100
- 300 > 100, so this run is SKIPPED
- Next run: tomorrow at 2:00 AM

Without startingDeadlineSeconds: if the cluster was down for a week,
Kubernetes would try to create 7 missed Jobs all at once when the
controller recovers. Usually not what you want.

### `suspend`

```yaml
spec:
  suspend: true
```

Pauses ALL future scheduled runs without deleting the CronJob.
Currently-running Jobs are unaffected — only FUTURE scheduling stops.
Used for maintenance windows or troubleshooting.

```bash
# Suspend
kubectl patch cronjob <name> -p '{"spec":{"suspend":true}}'

# Resume
kubectl patch cronjob <name> -p '{"spec":{"suspend":false}}'
```

---

## 8. Line by Line — Full CronJob YAML Explained

```yaml
apiVersion: batch/v1
# WHAT: batch API group -- same as Job
# WHY: CronJob is in the same "run to completion" category as Job,
# not the "run forever" category (apps/v1).

kind: CronJob
# WHAT: tells API server this is a CronJob object
# WHY: uses the CronJob Controller which watches the clock and
# creates Job objects on schedule -- completely separate from
# the Job Controller which manages individual Job runs.

metadata:
  name: end-of-day-report
  namespace: banking

spec:
  schedule: "0 2 * * *"
  # WHAT: standard Unix cron expression (5 fields)
  # WHY: defines WHEN to create a new Job. The CronJob controller
  # checks this every ~10 seconds and creates a Job whenever
  # a scheduled time has passed since the last creation.
  # Uses UTC timezone by default unless timeZone is set.

  timeZone: "Asia/Kolkata"
  # WHAT: timezone for the schedule (added in Kubernetes 1.27)
  # WHY: "0 2 * * *" without this means 2:00 AM UTC.
  # For an Indian banking team, this should be IST.
  # Before 1.27, teams adjusted the cron expression manually:
  # IST = UTC+5:30, so 2am IST = 8:30pm UTC previous day
  # which means the schedule would be "30 20 * * *"

  concurrencyPolicy: Forbid
  # WHAT: what to do if previous Job is still running at schedule time
  # WHY Forbid for banking reports: two concurrent report-generation
  # processes could write conflicting data to the same output tables,
  # producing incorrect financial numbers.
  # Allow = run both (risk: conflicts)
  # Forbid = skip new run (safe, preferred for financial tasks)
  # Replace = kill old, start new (freshness over completeness)

  successfulJobsHistoryLimit: 3
  # WHAT: how many completed successful Job objects to keep
  # WHY 3: enough to compare "did this week's reports complete
  # normally?" Without a limit, every daily run accumulates
  # as a Job object forever, cluttering kubectl get jobs.

  failedJobsHistoryLimit: 1
  # WHAT: how many failed Job objects to keep
  # WHY 1: failed jobs should be investigated immediately.
  # Old failures not yet investigated are not actionable.
  # Keeping just 1 keeps the namespace clean.

  startingDeadlineSeconds: 100
  # WHAT: max seconds late a missed scheduled run can still start
  # WHY 100: if controller-manager restarts around 2:00 AM and
  # comes back 2 minutes later, do NOT run a late report.
  # Skip and wait for tomorrow's 2:00 AM run.
  # Without this: a cluster outage could trigger many missed
  # runs to all fire at once when the controller recovers.

  jobTemplate:
  # WHAT: the blueprint for each Job this CronJob creates
  # WHY jobTemplate (not template): naming makes it explicit
  # that CronJob creates JOBS, not pods directly.
  # The nested structure reflects the three-layer chain:
  # CronJob -> Job -> Pod

    spec:
      backoffLimit: 2
      # WHAT: max retries for EACH individual Job run
      # WHY 2: if today's report generation fails, retry twice
      # before marking it failed. Handles transient DB issues.

      activeDeadlineSeconds: 3600
      # WHAT: max time any single Job run can take (1 hour)
      # WHY: report generation should never take more than 1 hour.
      # If it does, something is seriously wrong -- kill it.

      template:
        spec:
          containers:
          - name: report
            image: payment-app:v1.2

            command: ["python", "generate_report.py"]
            # WHAT: overrides the image's default CMD
            # WHY: the payment-app image starts the web server
            # by default. We override to run only the report
            # generation script.

            env:
            - name: DB_HOST
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: host
            # WHY: report script needs DB connection.
            # Pulling from Secret keeps credentials out of YAML.

            resources:
              requests:
                cpu: "200m"
                memory: "256Mi"
              limits:
                cpu: "1000m"
                memory: "512Mi"
            # WHY: report generation can be CPU and memory intensive.
            # Limits prevent one report job from starving other pods.
            # Requests ensure the scheduler places it on a node
            # with adequate capacity.

          restartPolicy: OnFailure
          # WHAT: kubelet restarts container in-place on failure
          # WHY OnFailure here (not Never): for a daily scheduled
          # report, we care whether TODAY's run succeeded overall,
          # not about inspecting each individual retry pod.
          # Fewer pod objects = cleaner namespace for a task that
          # runs every day for years.
```

---

## 9. CronJob vs Job — Key Differences

| | Job | CronJob |
|---|---|---|
| Runs | Once, manually triggered | Automatically, on a schedule |
| Creates | Pods directly | Job objects (which create pods) |
| Lifecycle | Single run, then done | Runs repeatedly forever |
| Scheduling | None | Cron expression |
| concurrencyPolicy | N/A | Allow, Forbid, Replace |
| History limits | ttlSecondsAfterFinished | successfulJobsHistoryLimit, failedJobsHistoryLimit |
| Manual trigger | kubectl apply | kubectl create job --from=cronjob |

---

## 10. Industry Usage — Banking Context

```
Banking CronJobs by schedule:

Every 5 minutes:
  Transaction health check
  Payment gateway connectivity verification

Every 30 minutes:
  Real-time fraud score recalculation
  Cache warming for frequently accessed accounts

Every 6 hours:
  Database backup snapshot to S3

Every day at 2am:
  End-of-day financial report generation
  Daily account balance reconciliation
  Log cleanup and archival

Every Sunday at midnight:
  Weekly compliance audit report
  Database index rebuild and statistics update

1st of every month:
  Monthly account statement generation
  Monthly regulatory report submission
```

---

## PART 2 — HANDS-ON LABS

---

## Lab 1 — Create a CronJob and Watch It Fire

```bash
kubectl apply -f - << 'EOF'
apiVersion: batch/v1
kind: CronJob
metadata:
  name: every-minute
spec:
  schedule: "*/1 * * * *"
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            command:
            - sh
            - -c
            - |
              echo "CronJob ran at: $(date)"
              echo "I am pod: $HOSTNAME"
          restartPolicy: OnFailure
EOF

# Watch the CronJob -- LAST SCHEDULE column updates every minute
kubectl get cronjob every-minute -w
# Ctrl+C after you see LAST SCHEDULE update at least once

# See the Jobs created -- one per scheduled run
kubectl get jobs | grep every-minute
# every-minute-28401234  1/1  Complete  2m
# every-minute-28401235  1/1  Complete  1m

# Step 1 -- get a Job name from the output above
# e.g. every-minute-28401234

# Step 2 -- see the pod that ran for that Job
kubectl get pods -l job-name=every-minute-28401234

# Step 3 -- get pod name from output above
# e.g. every-minute-28401234-x7k2p

# Step 4 -- check the pod logs
kubectl logs every-minute-28401234-x7k2p
# "CronJob ran at: Mon Jun 16 09:01:00 UTC 2026"
# "I am pod: every-minute-28401234-x7k2p"

kubectl delete cronjob every-minute
```

---

## Lab 2 — concurrencyPolicy: Forbid in Action

This lab shows a scheduled run being SKIPPED because the previous
run is still active.

```bash
kubectl apply -f - << 'EOF'
apiVersion: batch/v1
kind: CronJob
metadata:
  name: slow-job
spec:
  schedule: "*/1 * * * *"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 5
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: slow
            image: busybox
            command:
            - sh
            - -c
            - |
              echo "Starting slow task..."
              sleep 90
              echo "Slow task done"
          restartPolicy: Never
EOF

# Wait for the first minute mark -- first Job starts
kubectl get jobs | grep slow-job

# Step 1 -- get first Job name
kubectl get jobs | grep slow-job
# e.g. slow-job-28401234

# Confirm it is running (sleeping for 90 seconds)
kubectl get pods -l job-name=slow-job-28401234
# STATUS: Running

# Wait for the SECOND minute mark to pass
# (first Job will STILL be running -- 90 second sleep)
sleep 70

# Check Jobs again
kubectl get jobs | grep slow-job
# With concurrencyPolicy: Forbid:
# STILL only ONE Job (the first one running)
# The second scheduled run was COMPLETELY SKIPPED

# Check CronJob events to confirm skip happened
kubectl describe cronjob slow-job
# Events section shows: "Missed scheduled time"
# OR check status:
kubectl get cronjob slow-job
# ACTIVE column shows 1 (first job still running)

kubectl delete cronjob slow-job
```

---

## Lab 3 — History Limits in Action

```bash
kubectl apply -f - << 'EOF'
apiVersion: batch/v1
kind: CronJob
metadata:
  name: history-demo
spec:
  schedule: "*/1 * * * *"
  successfulJobsHistoryLimit: 2
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: task
            image: busybox
            command: ["sh", "-c", "echo done"]
          restartPolicy: Never
EOF

# Wait 5 minutes -- let 5 Jobs complete
sleep 300

# Check how many Jobs remain
kubectl get jobs | grep history-demo
# Should show only 2 Jobs (not 5)
# successfulJobsHistoryLimit: 2 means oldest 3 were auto-deleted

kubectl delete cronjob history-demo
```

---

## Lab 4 — Manually Trigger a CronJob Immediately

This is one of the most useful commands in production.
Schedule says 2am but you want to test RIGHT NOW.

```bash
kubectl apply -f - << 'EOF'
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-report
spec:
  schedule: "0 2 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: report
            image: busybox
            command:
            - sh
            - -c
            - |
              echo "Generating daily report..."
              sleep 3
              echo "Report generated successfully"
          restartPolicy: OnFailure
EOF

# Schedule says 2am -- trigger it RIGHT NOW for testing
kubectl create job daily-report-manual --from=cronjob/daily-report

# Watch the manually triggered Job
kubectl get job daily-report-manual -w
# Ctrl+C once COMPLETIONS shows 1/1

# Step 1 -- get pod name
kubectl get pods -l job-name=daily-report-manual

# Step 2 -- check logs
kubectl logs daily-report-manual-x7k2p
# "Generating daily report..."
# "Report generated successfully"

# Clean up the manual job (does not affect the CronJob schedule)
kubectl delete job daily-report-manual
kubectl delete cronjob daily-report
```

---

## Lab 5 — Suspend and Resume a CronJob

```bash
kubectl apply -f - << 'EOF'
apiVersion: batch/v1
kind: CronJob
metadata:
  name: suspend-demo
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: task
            image: busybox
            command: ["sh", "-c", "echo running"]
          restartPolicy: Never
EOF

# Let it run once
sleep 70
kubectl get jobs | grep suspend-demo
# One job exists

# SUSPEND the CronJob -- stops ALL future scheduled runs
# without deleting the CronJob object
kubectl patch cronjob suspend-demo -p '{"spec":{"suspend":true}}'

# Verify it is suspended
kubectl get cronjob suspend-demo
# SUSPEND column shows True

# Wait another 2 minutes -- NO new Jobs should be created
sleep 120
kubectl get jobs | grep suspend-demo
# Same number of jobs as before -- no new ones created

# RESUME the CronJob
kubectl patch cronjob suspend-demo -p '{"spec":{"suspend":false}}'

# Verify resumed
kubectl get cronjob suspend-demo
# SUSPEND column shows False

# New Jobs will be created at next scheduled time (next minute)
sleep 70
kubectl get jobs | grep suspend-demo
# New Job appears

kubectl delete cronjob suspend-demo
```

---

## Lab 6 — Full Banking CronJob Setup

This is the complete setup you would describe in an interview
when asked about your banking project.

```bash
kubectl apply -f - << 'EOF'
apiVersion: batch/v1
kind: CronJob
metadata:
  name: eod-report
  namespace: default
  labels:
    team: platform
    domain: banking
    type: reporting
spec:
  schedule: "*/2 * * * *"
  timeZone: "UTC"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  startingDeadlineSeconds: 60
  jobTemplate:
    spec:
      backoffLimit: 2
      activeDeadlineSeconds: 60
      template:
        spec:
          containers:
          - name: report-generator
            image: busybox
            command:
            - sh
            - -c
            - |
              echo "=== EOD Report Generation Started ==="
              echo "Timestamp: $(date)"
              echo "Pod: $HOSTNAME"
              echo "Connecting to database..."
              sleep 5
              echo "Generating transaction summary..."
              sleep 5
              echo "Writing report to output..."
              sleep 2
              echo "=== EOD Report Generation Complete ==="
            resources:
              requests:
                cpu: "100m"
                memory: "64Mi"
              limits:
                cpu: "500m"
                memory: "128Mi"
          restartPolicy: OnFailure
EOF

# Watch it run every 2 minutes (using 2 min for lab speed)
kubectl get cronjob eod-report -w
# Ctrl+C after first LAST SCHEDULE update

# See the Job created
kubectl get jobs | grep eod-report

# Step 1 -- get Job name
kubectl get jobs | grep eod-report
# e.g. eod-report-28401240

# Step 2 -- get pod name for that Job
kubectl get pods -l job-name=eod-report-28401240

# Step 3 -- read the full report output
kubectl logs eod-report-28401240-abc12
# "=== EOD Report Generation Started ==="
# "Timestamp: ..."
# "Connecting to database..."
# ...

# Manually trigger for testing
kubectl create job eod-report-test --from=cronjob/eod-report
kubectl get job eod-report-test -w

kubectl delete cronjob eod-report
kubectl delete job eod-report-test 2>/dev/null
```

---

## Quick Reference — CronJob Commands

```bash
# Create
kubectl apply -f cronjob.yaml

# Get
kubectl get cronjobs
kubectl get cj                           # short form

# Inspect
kubectl describe cronjob <name>
kubectl get cronjob <name>
# Shows: SCHEDULE, SUSPEND, ACTIVE, LAST SCHEDULE, AGE

# See Jobs created by a CronJob
kubectl get jobs | grep <cronjob-name>

# Manually trigger an immediate run
kubectl create job <name>-manual --from=cronjob/<name>

# Suspend (stop future scheduled runs without deleting)
kubectl patch cronjob <name> -p '{"spec":{"suspend":true}}'

# Resume
kubectl patch cronjob <name> -p '{"spec":{"suspend":false}}'

# Delete CronJob (does NOT delete Jobs it already created)
kubectl delete cronjob <name>

# Delete CronJob AND all its created Jobs
kubectl delete cronjob <name>
kubectl delete jobs -l <label-matching-cronjob-jobs>
```

---

## Summary — What Makes CronJob Unique

1. CronJob is a Job factory with a clock -- it creates Jobs,
   not pods directly
2. Three-layer chain: CronJob -> Job -> Pod
3. schedule uses standard Unix cron syntax (5 fields)
4. concurrencyPolicy controls what happens when previous run
   is still active at next scheduled time
5. successfulJobsHistoryLimit and failedJobsHistoryLimit prevent
   Job object accumulation over time
6. startingDeadlineSeconds prevents late runs after controller
   outages
7. suspend: true pauses scheduling without deleting the CronJob
8. kubectl create job --from=cronjob/<name> manually triggers
   an immediate run for testing
9. timeZone field (K8s 1.27+) sets the timezone for the schedule

---

*Next: CronJob Interview Questions*
