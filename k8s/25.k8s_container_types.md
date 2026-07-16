# Kubernetes Container Types — Deep Dive Concept + Hands-On Lab
## Init Containers · Sidecar Containers · Ephemeral Containers · Multi-Container Patterns
> Written for: Someone with 4 years of DevOps experience preparing for senior-level interviews
> Style: First-standard student explanation → deep technical truth → hands-on lab with line-by-line explanation

---

## 🧠 SECTION 1 — WHAT IS A POD REALLY? (Story First)

### Forget Kubernetes for 3 minutes — start with a story

Imagine you are moving into a **new apartment**.

Before you move in — there are things that MUST happen first:
```
STEP 1: Electrician comes → wires the apartment → leaves
STEP 2: Plumber comes → installs pipes → leaves
STEP 3: Painter comes → paints walls → leaves
STEP 4: YOU move in → start living there
```

These people (Step 1-3) do their job and LEAVE before you move in. They are **init containers**.

Now you are living in the apartment. You have:
```
YOU (main person living there) → this is the MAIN container
  → cook food (your actual application work)

ROOMMATE who lives with you permanently:
  → Checks your mail and summarizes it for you (log shipper)
  → Translates when foreign guests come (proxy sidecar)
  → These stay with you forever → SIDECAR containers

REPAIR PERSON who comes only when something breaks:
  → Comes in, fixes the pipe, leaves
  → This is an EPHEMERAL container (for debugging)
```

**This is exactly how Kubernetes pods work.**

### A Pod Can Have Multiple Containers

```
POD (the apartment):
  ├── Init Containers (run FIRST, one by one, then EXIT)
  │     ├── init-1: wait for database to be ready
  │     └── init-2: download config files
  │
  ├── Main Container (your actual application)
  │     └── payment-app: serves HTTP requests
  │
  ├── Sidecar Containers (run ALONGSIDE main, same lifetime)
  │     ├── log-shipper: reads app logs, ships to Elasticsearch
  │     ├── envoy-proxy: handles all network traffic (Istio)
  │     └── config-reloader: watches for config changes
  │
  └── Ephemeral Containers (temporary, injected for debugging)
        └── debug-tools: netshoot, curl, dig — injected when pod has issues
```

### Why Multiple Containers in One Pod?

```
CONTAINERS IN THE SAME POD:
  ✅ Share the same NETWORK (same IP address, same localhost)
  ✅ Share VOLUMES (can read/write same files)
  ✅ Share process namespace (can see each other's processes)
  ✅ Started and stopped together (same lifecycle)
  ✅ Scheduled on the same node (always co-located)

THIS ENABLES:
  App writes log to /var/log/app.log
  Log sidecar reads /var/log/app.log from SAME volume
  Log sidecar ships to Elasticsearch
  → No network call between containers needed (same volume)

  App listens on localhost:8080
  Envoy proxy intercepts all traffic on localhost:15001
  → Proxy can inspect traffic without any app code change
```

---

## 🔵 SECTION 2 — INIT CONTAINERS

### What is an Init Container?

An **init container** is a special container that runs **before** the main application containers start. It:
1. Runs to COMPLETION (exits with code 0)
2. Runs ONE AT A TIME in sequence (not parallel)
3. If it FAILS → pod restarts the failing init container (keeps retrying)
4. Only after ALL init containers succeed → main containers start

### Simple Story — The Restaurant Opening Checklist

```
Before a restaurant opens to customers:

INIT CONTAINER 1: Chef sets up the kitchen (exits when done)
INIT CONTAINER 2: Waiter sets tables (exits when done)
INIT CONTAINER 3: Manager checks safety compliance (exits when done)

Only when ALL prep work is done → restaurant OPENS (main container starts)
Customers can now enter and be served.

If the kitchen setup fails:
  → Chef tries again and again
  → Waiter and Manager WAIT (they haven't run yet)
  → Restaurant stays CLOSED until kitchen is ready
```

### Why Init Containers Exist — The Problems They Solve

```
PROBLEM 1: Database dependency
  Your app starts and immediately tries to connect to a database.
  Database is still starting (takes 30 seconds).
  App fails → CrashLoopBackOff.

  SOLUTION with Init Container:
    init-1: keep pinging database until it responds → exits when DB is ready
    main-app: starts ONLY after DB is confirmed ready → no crash

PROBLEM 2: Secret/Config fetching
  App needs a config file downloaded from S3 before starting.
  You don't want the app image to have AWS credentials.

  SOLUTION with Init Container:
    init-1: has AWS credentials → downloads config to shared volume → exits
    main-app: reads config from shared volume → no AWS credentials needed in app

PROBLEM 3: Database migration
  Before app starts, database schema must be updated.
  Running migration in the app itself is risky (multiple replicas = multiple migrations).

  SOLUTION with Init Container:
    init-1: runs database migration (single run, exits when done)
    main-app: starts with correct database schema guaranteed

PROBLEM 4: Permission setup
  A volume needs specific permissions (chown, chmod) before the app can use it.
  The app runs as non-root but the volume is owned by root.

  SOLUTION with Init Container:
    init-1: runs as root, fixes volume permissions → exits
    main-app: runs as non-root, can now write to volume
```

### Init Container YAML — Full Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: payment-app-pod
  namespace: banking
spec:
  initContainers:
  - name: wait-for-database
    image: busybox:1.35
    command:
    - sh
    - -c
    - |
      echo "Waiting for database to be ready..."
      until nc -z postgres-svc 5432; do
        echo "Database not ready. Sleeping 5 seconds..."
        sleep 5
      done
      echo "Database is ready. Starting main app."

  - name: run-db-migration
    image: migrate/migrate:v4.16.2
    command:
    - migrate
    - -path=/migrations
    - -database=postgres://$(DB_USER):$(DB_PASS)@postgres-svc:5432/bankdb
    - up
    env:
    - name: DB_USER
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: username
    - name: DB_PASS
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password
    volumeMounts:
    - name: migration-files
      mountPath: /migrations

  - name: fix-volume-permissions
    image: busybox:1.35
    command:
    - sh
    - -c
    - chown -R 1000:1000 /app/data && chmod 755 /app/data
    securityContext:
      runAsUser: 0              # run as root for permission fixing
    volumeMounts:
    - name: app-data
      mountPath: /app/data

  containers:
  - name: payment-app
    image: payment-service:2.0
    ports:
    - containerPort: 8080
    securityContext:
      runAsUser: 1000           # run as non-root
    volumeMounts:
    - name: app-data
      mountPath: /app/data

  volumes:
  - name: migration-files
    configMap:
      name: db-migrations
  - name: app-data
    persistentVolumeClaim:
      claimName: payment-data-pvc
```

**Every line explained:**

```yaml
  initContainers:
```
→ `initContainers` → the list of init containers. Different field from `containers:`.
→ They run in ORDER — first to last.
→ Each must EXIT with code 0 before the next starts.

```yaml
  - name: wait-for-database
    image: busybox:1.35
```
→ First init container. Name is for identification in logs and events.
→ `busybox:1.35` → tiny Linux image with basic tools (nc, sh, sleep, echo).
→ BusyBox is perfect for init containers — small image, starts fast.

```yaml
    command:
    - sh
    - -c
    - |
      echo "Waiting for database to be ready..."
      until nc -z postgres-svc 5432; do
```
→ `command` → overrides the image's ENTRYPOINT.
→ `sh` → start a shell.
→ `-c` → execute the following string as a command.
→ `|` (pipe in YAML) → multi-line string literal. Everything below is one command string.
→ `until nc -z postgres-svc 5432; do` → `nc` = netcat. `-z` = just check if port is open (zero I/O mode). `postgres-svc` = the database service name (DNS resolves to its ClusterIP). `5432` = PostgreSQL port. Loop UNTIL the port responds.
→ `sleep 5` → wait 5 seconds between checks.
→ This loop runs FOREVER until the DB is ready. When DB is ready → loop exits → init container exits with code 0.

```yaml
  - name: run-db-migration
    image: migrate/migrate:v4.16.2
    command:
    - migrate
    - -path=/migrations
    - -database=postgres://$(DB_USER):$(DB_PASS)@postgres-svc:5432/bankdb
    - up
```
→ Second init container. Runs ONLY after first one exits successfully.
→ Uses a specific migration tool image.
→ `$(DB_USER)` → environment variable substitution inside the command. The `$()` syntax in Kubernetes command fields is expanded at runtime.
→ `up` → migrate to the latest schema version.

```yaml
    env:
    - name: DB_USER
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: username
```
→ Environment variables for this init container only.
→ `secretKeyRef` → pull value from a Kubernetes Secret.
→ Init containers can have their own secrets that main containers don't have.

```yaml
  - name: fix-volume-permissions
    image: busybox:1.35
    command:
    - sh
    - -c
    - chown -R 1000:1000 /app/data && chmod 755 /app/data
    securityContext:
      runAsUser: 0
```
→ `chown -R 1000:1000 /app/data` → change ownership of everything in /app/data to UID 1000 and GID 1000.
→ `-R` → recursive (all files and subdirectories).
→ `chmod 755 /app/data` → set permissions: owner read/write/execute, group and others read/execute.
→ `runAsUser: 0` → this init container runs as ROOT (needed to chown files).
→ Main container runs as 1000 (non-root) and can now write because it owns the files.

### Init Container Lifecycle — The State Machine

```
POD STARTS
    │
    ▼
Init Container 1 STARTS
    │
    ├── Exits 0 (success) → proceed
    └── Exits non-0 (fail) → RESTART init container 1 (based on restartPolicy)
                            → if restartPolicy=Never → pod fails
                            → if restartPolicy=Always/OnFailure → retry

If Init Container 1 succeeds:
    │
    ▼
Init Container 2 STARTS
    │
    └── (same logic)

All init containers succeed:
    │
    ▼
ALL MAIN CONTAINERS START simultaneously
    │
    ▼
Pod is Running
```

### Init Container Key Behaviors

```
BEHAVIOR 1: Sequential execution
  init-1 → (exits 0) → init-2 → (exits 0) → main containers start
  NOT parallel. Always one at a time.

BEHAVIOR 2: Pod restarts on init failure
  If init-2 fails → pod goes to Init:1/3 state
  Kubernetes retries init-2
  init-1 does NOT run again (already succeeded)

BEHAVIOR 3: Separate image from main container
  init container can use a different image entirely
  init: uses busybox (has nc, wget tools)
  main: uses your app image (doesn't need those tools)
  → Smaller main image. Better security.

BEHAVIOR 4: Volume sharing with main containers
  init containers can write to volumes
  main containers can read what init containers prepared
  → Common pattern for config file setup

BEHAVIOR 5: Different security context from main
  init: runAsUser: 0 (root) to fix permissions
  main: runAsUser: 1000 (non-root) for security
  → Init can do privileged setup that main can't
```

### Pod Status During Init

```
kubectl get pods

NAME              READY   STATUS       RESTARTS   AGE
payment-pod       0/1     Init:0/3     0          5s   ← init-1 running
payment-pod       0/1     Init:1/3     0          15s  ← init-2 running
payment-pod       0/1     Init:2/3     0          25s  ← init-3 running
payment-pod       0/1     PodInitializing  0      30s  ← all inits done, main starting
payment-pod       1/1     Running      0          35s  ← main container up
```

→ `Init:0/3` → 0 out of 3 init containers completed (first one running).
→ `Init:1/3` → 1 out of 3 completed (second one running).
→ `PodInitializing` → init containers done, main containers being set up.

---

## 🟢 SECTION 3 — SIDECAR CONTAINERS

### What is a Sidecar Container?

A **sidecar container** runs ALONGSIDE the main container in the same pod. It:
- Starts at the same time as the main container
- Runs for the ENTIRE lifetime of the pod
- Shares the same network and volumes
- Provides a SUPPORTING function to the main container

### Simple Story — The Personal Assistant

```
You (the main container) are a CEO:
  → Your job: make business decisions

Your Personal Assistant (sidecar):
  → Takes your phone calls and filters them for you
  → Keeps your schedule
  → Translates documents you need to read

The assistant doesn't do CEO work.
The assistant SUPPORTS you doing CEO work.
The assistant is ALWAYS there as long as you are working.
When you leave the office (pod deleted) → assistant also goes home.

This is exactly what a sidecar does.
```

### Why Sidecars Exist — The "Separation of Concerns" Principle

```
WITHOUT SIDECAR (bad approach):
  Your app image contains:
    → Your business logic
    → Log shipping code (to Elasticsearch)
    → Metrics collection code (to Prometheus)
    → mTLS certificate rotation code
    → Config file watching code
  
  Problem: Your app becomes HUGE and complex.
  Problem: Changing log shipping = rebuild app image.
  Problem: Security team needs to audit your log shipping code mixed with business logic.
  Problem: Every app team reinvents these solutions.

WITH SIDECAR (correct approach):
  Main container: ONLY business logic
  Sidecar 1: log shipping (Fluentd/Fluent Bit) — standard image
  Sidecar 2: metrics (Prometheus exporter) — standard image
  Sidecar 3: mTLS proxy (Envoy/Istio) — standard image
  
  Benefits:
  → App team writes ONLY business logic
  → Infrastructure team manages sidecar images
  → Update log shipper → just update sidecar image, not app
  → Same sidecar pattern across ALL apps = consistency
```

### Common Sidecar Patterns

#### Pattern 1 — Log Shipping Sidecar

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: payment-pod-with-logging
spec:
  containers:
  - name: payment-app               # MAIN CONTAINER
    image: payment-service:2.0
    volumeMounts:
    - name: log-volume
      mountPath: /var/log/app       # app writes logs HERE

  - name: log-shipper               # SIDECAR CONTAINER
    image: fluent/fluent-bit:2.1
    volumeMounts:
    - name: log-volume
      mountPath: /var/log/app       # reads from SAME directory
    - name: fluent-config
      mountPath: /fluent-bit/etc
    env:
    - name: ELASTICSEARCH_HOST
      value: "elasticsearch.monitoring.svc.cluster.local"

  volumes:
  - name: log-volume
    emptyDir: {}                    # shared between both containers
  - name: fluent-config
    configMap:
      name: fluent-bit-config
```

**How it works:**
```
payment-app writes:   /var/log/app/payment.log
log-shipper reads:    /var/log/app/payment.log (SAME FILE via shared volume)
log-shipper ships to: Elasticsearch

The app doesn't know or care about Elasticsearch.
The app just writes to files like it always has.
The sidecar does the shipping transparently.
```

#### Pattern 2 — Istio Envoy Proxy Sidecar

```yaml
# This is what Istio's mutating webhook ADDS to your pod automatically
spec:
  containers:
  - name: payment-app               # YOUR container
    image: payment-service:2.0
    ports:
    - containerPort: 8080

  - name: istio-proxy               # INJECTED SIDECAR (by Istio)
    image: docker.io/istio/proxyv2:1.18.0
    ports:
    - containerPort: 15090
    - containerPort: 15021
    env:
    - name: ISTIO_META_APP_PORTS
      value: "8080"
    securityContext:
      runAsUser: 1337
```

**How it works:**
```
Without Istio:
  Client → payment-app:8080

With Istio sidecar:
  Client → Envoy proxy (intercepts ALL traffic) → payment-app:8080
  
  Envoy proxy handles:
    → mTLS (encrypts traffic between services automatically)
    → Circuit breaking (stop sending to unhealthy services)
    → Retries (retry failed requests automatically)
    → Observability (traces every request)
    → Traffic routing (canary, blue-green)
  
  Your app code changes: ZERO.
  The proxy does it all transparently.
```

#### Pattern 3 — Config Reloader Sidecar

```yaml
spec:
  containers:
  - name: nginx                     # MAIN: serves web traffic
    image: nginx:1.25
    volumeMounts:
    - name: nginx-config
      mountPath: /etc/nginx/conf.d

  - name: config-reloader           # SIDECAR: watches config changes
    image: jimmidyson/configmap-reload:v0.9.0
    args:
    - --volume-dir=/config
    - --webhook-url=http://localhost:80/-/reload
    volumeMounts:
    - name: nginx-config
      mountPath: /config            # SAME config volume

  volumes:
  - name: nginx-config
    configMap:
      name: nginx-configmap
```

**How it works:**
```
ConfigMap is mounted as files in /etc/nginx/conf.d AND /config (same volume).
config-reloader watches /config for changes.
When ConfigMap is updated → files change → reloader detects it.
Reloader calls: http://localhost:80/-/reload (nginx reload endpoint).
Nginx reloads its config WITHOUT restarting the pod.
```

#### Pattern 4 — Metrics Exporter Sidecar

```yaml
spec:
  containers:
  - name: payment-app               # MAIN: business logic
    image: payment-service:2.0
    ports:
    - containerPort: 8080           # business API port

  - name: jmx-exporter              # SIDECAR: exposes Java metrics
    image: bitnami/jmx-exporter:0.19.0
    ports:
    - containerPort: 9090           # Prometheus scrapes here
    env:
    - name: JMX_HOST
      value: "localhost"            # same pod = same localhost!
    - name: JMX_PORT
      value: "9999"                 # app exposes JMX on this port
```

**How it works:**
```
Java payment app exposes internal metrics via JMX on port 9999 (localhost).
JMX exporter sidecar reads from localhost:9999 (same pod network).
JMX exporter translates JMX → Prometheus format on port 9090.
Prometheus scrapes the sidecar's port 9090.
App doesn't know Prometheus exists.
```

### Sidecar Container in Kubernetes 1.29+ (Native Sidecar)

**Before Kubernetes 1.29:** Sidecars were just regular containers. No special handling.

**Problem:** If a sidecar needed to restart before the main container finished starting — ordering was not guaranteed. Init containers couldn't use sidecars (a sidecar started WITH main containers, not during init phase).

**Kubernetes 1.29 introduced Native Sidecar Containers:**

```yaml
spec:
  initContainers:
  - name: logshipper                # NATIVE SIDECAR — defined in initContainers
    image: fluent/fluent-bit:2.1
    restartPolicy: Always           # THIS is what makes it a native sidecar
    volumeMounts:
    - name: log-volume
      mountPath: /var/log
  
  - name: wait-for-db               # Regular init container (no restartPolicy: Always)
    image: busybox
    command: ["sh", "-c", "until nc -z db 5432; do sleep 1; done"]
  
  containers:
  - name: app
    image: myapp:1.0
    volumeMounts:
    - name: log-volume
      mountPath: /var/log
  
  volumes:
  - name: log-volume
    emptyDir: {}
```

**The key difference:**
```yaml
restartPolicy: Always
```
→ When `restartPolicy: Always` is set on an initContainer → it becomes a NATIVE SIDECAR.
→ Native sidecars START before regular init containers and BEFORE main containers.
→ Native sidecars stay RUNNING while init containers run and while main containers run.
→ Native sidecars are STOPPED LAST — after main containers exit.
→ This solves the problem: your log shipper starts BEFORE the app and keeps running until after the app exits. No logs are missed.

### Sidecar vs Init Container — Comparison

```
INIT CONTAINER:                     SIDECAR CONTAINER:
  Runs BEFORE main containers         Runs WITH main containers
  Runs to COMPLETION (exits 0)        Runs INDEFINITELY (same lifetime as pod)
  Sequential (one at a time)          Parallel (all at once)
  Does setup/preparation work         Does ongoing support work
  restartPolicy same as pod           restartPolicy: Always (native sidecar)
  Different image OK (busybox)        Often purpose-built image
  Example: DB migration               Example: log shipper, proxy, exporter
```

---

## 🔍 SECTION 4 — EPHEMERAL CONTAINERS

### What is an Ephemeral Container?

An **ephemeral container** is a temporary container you **inject into a running pod** for DEBUGGING purposes. It:
- Added to a running pod (not defined in the original pod spec)
- Does NOT have resource limits
- CANNOT be restarted
- Does NOT appear in `kubectl get pods`
- Is removed when the pod is deleted
- Cannot be added to pod spec YAML (only via `kubectl debug`)

### Why Ephemeral Containers Exist

```
PROBLEM:
  Your production pod is behaving strangely.
  You need to debug it — run curl, netstat, dig, tcpdump.
  
  BUT: Production containers are minimal — distroless images.
  They have ZERO debugging tools.
  You cannot exec into them and run curl — it doesn't exist!
  
  You also can't restart the pod to add a debug container
  because: restarting might make the bug disappear.
  You need to catch it in the act.

SOLUTION: Ephemeral Container
  kubectl debug -it <running-pod> --image=nicolaka/netshoot
  → A debug container is injected into the running pod
  → It shares the pod's network and processes
  → You have ALL network tools: curl, dig, netstat, tcpdump
  → The original containers are NOT disturbed
  → You debug, find the issue, exit
  → The ephemeral container disappears
```

### Ephemeral Container Commands

```bash
# Inject a debug container into a running pod
kubectl debug -it payment-pod \
  --image=nicolaka/netshoot \
  --target=payment-app

# Breaking down:
# debug        → kubectl debug subcommand
# -it          → interactive terminal
# payment-pod  → the running pod to inject into
# --image=nicolaka/netshoot → debug image with ALL network tools
# --target=payment-app → share the process namespace of this container

# Inside the ephemeral container — you can debug:
curl http://localhost:8080/health      # test app endpoint
dig payment-svc.banking.svc.cluster.local  # DNS check
netstat -tlnp                          # check listening ports
tcpdump -i any port 8080               # capture traffic
```

### Process Namespace Sharing

```yaml
# You can also enable this in the pod spec directly
spec:
  shareProcessNamespace: true    # all containers see each other's processes
  containers:
  - name: app
    image: myapp:1.0
  - name: debug-sidecar
    image: busybox
```

→ `shareProcessNamespace: true` → all containers in the pod share ONE process namespace.
→ The debug sidecar can run `ps aux` and see the main app's processes.
→ Can run `kill -HUP <pid>` to send signals to the main app.
→ Useful for debugging and for config reloaders.

---

## 📐 SECTION 5 — MULTI-CONTAINER PATTERNS (Named Patterns)

These are architectural patterns for how containers work together. Interviewers expect you to know these by name.

### Pattern 1 — Sidecar Pattern

```
DEFINITION: A helper container that EXTENDS or ENHANCES the main container.
Shares resources (network, volumes) but adds functionality WITHOUT changing the app.

EXAMPLES:
  → Log shipper (Fluent Bit alongside app)
  → Service mesh proxy (Envoy/Istio alongside app)
  → Config reloader (watches ConfigMap, reloads app)
  → Metrics exporter (converts app metrics to Prometheus format)

RELATIONSHIP: App ← uses ← Sidecar (sidecar serves the app)
```

### Pattern 2 — Ambassador Pattern

```
DEFINITION: A proxy container that REPRESENTS the pod to the outside world.
All OUTGOING traffic from the app goes through the ambassador.

EXAMPLE USE CASE:
  App needs to talk to a database.
  But the database is in different regions with different addresses.
  
  WITHOUT Ambassador:
    App has complex connection logic to pick the right database region.
    
  WITH Ambassador:
    Ambassador container handles all database connection logic.
    App always connects to localhost:5432 (ambassador's local port).
    Ambassador routes to the right database region.
    App is simple. All complexity in Ambassador.

RELATIONSHIP: App → Ambassador → External service
              Ambassador is the app's "representative" to the outside world.
```

```yaml
spec:
  containers:
  - name: payment-app               # MAIN: business logic
    image: payment-service:2.0
    env:
    - name: DB_HOST
      value: "localhost"            # Always talk to localhost!
    - name: DB_PORT
      value: "5432"

  - name: db-ambassador             # AMBASSADOR: handles connection routing
    image: haproxy:2.8
    ports:
    - containerPort: 5432           # listens on localhost:5432
    # Proxies to the right database based on load, region, health
```

### Pattern 3 — Adapter Pattern

```
DEFINITION: A container that TRANSFORMS the main container's output format.
Takes what the main container produces and converts it to a standard format.

EXAMPLE USE CASE:
  Your legacy app outputs metrics in its own proprietary format.
  Prometheus expects metrics in a specific format.
  
  WITHOUT Adapter:
    Either rewrite the app to output Prometheus format (expensive).
    Or Prometheus can't scrape it (blind spots).
    
  WITH Adapter:
    Adapter reads the app's proprietary metrics endpoint.
    Adapter CONVERTS to Prometheus format.
    Prometheus scrapes the Adapter's endpoint.
    App code unchanged.

RELATIONSHIP: App → [produces output] → Adapter → [transforms] → Consumer
```

```yaml
spec:
  containers:
  - name: legacy-app                # MAIN: produces custom format metrics
    image: legacy-payment:1.0
    ports:
    - containerPort: 9999           # custom metrics endpoint

  - name: metrics-adapter           # ADAPTER: transforms format
    image: custom-exporter:1.0
    ports:
    - containerPort: 9090           # Prometheus-format endpoint
    env:
    - name: SOURCE_URL
      value: "http://localhost:9999/metrics"  # reads from main container
```

---

## ❓ SECTION 6 — WHEN TO USE WHICH CONTAINER TYPE

```
USE INIT CONTAINERS WHEN:
  ✅ Waiting for a dependency to be ready (database, API, cache)
  ✅ Running database migrations before app starts
  ✅ Downloading/preparing config files
  ✅ Setting file permissions (chown/chmod)
  ✅ Registering service in a catalog before starting
  ✅ Pulling secrets from Vault before main app needs them
  ✅ Warming up caches

USE SIDECAR CONTAINERS WHEN:
  ✅ Logging: shipping logs to centralized storage
  ✅ Service mesh: mTLS, traffic management (Istio Envoy)
  ✅ Config sync: updating config without pod restart
  ✅ Monitoring: exposing metrics in standard format
  ✅ Security: secrets rotation agent
  ✅ Data sync: syncing files from git or S3 on interval

USE EPHEMERAL CONTAINERS WHEN:
  ✅ Debugging a running pod that has no debug tools
  ✅ App is in distroless/minimal image — can't exec into it
  ✅ Need to capture network traffic from inside a pod
  ✅ Need to check DNS resolution from inside a pod's context
  ✅ Production issue that disappears on restart

DO NOT USE MULTIPLE CONTAINERS WHEN:
  ❌ Containers don't need to share network or volumes
  ❌ They have independent scaling needs (use separate Deployments)
  ❌ They belong to different services or teams
  ❌ You just want containers to be "close" to each other
```

---

## 💻 SECTION 7 — HANDS-ON LAB

> Every command explained word by word. Every flag explained. Nothing skipped.

---

### LAB 1 — Init Container: Wait for Service

```bash
# Step 1: See what happens without a dependency service running
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: init-demo-pod
spec:
  initContainers:
  - name: wait-for-service
    image: busybox:1.35
    command:
    - sh
    - -c
    - |
      echo "Checking if service exists..."
      until nslookup myservice.default.svc.cluster.local; do
        echo "Service not found. Waiting 3 seconds..."
        sleep 3
      done
      echo "Service found! Proceeding."
  containers:
  - name: main-app
    image: nginx
EOF
```

**Explaining the command:**
- `nslookup myservice.default.svc.cluster.local` → DNS lookup. Fails if service doesn't exist.
- `until <command>; do ... done` → keep looping UNTIL command succeeds (exit 0).

```bash
# Watch the pod status — it will be stuck in Init state
kubectl get pod init-demo-pod -w
```
- `-w` → watch mode — updates live

Output:
```
NAME             READY   STATUS     RESTARTS   AGE
init-demo-pod    0/1     Init:0/1   0          5s
init-demo-pod    0/1     Init:0/1   0          8s
# keeps showing Init:0/1 until service exists
```

```bash
# Check init container logs to see what it's doing
kubectl logs init-demo-pod -c wait-for-service
```
- `-c wait-for-service` → logs from the init container named `wait-for-service`
- You'll see: "Service not found. Waiting 3 seconds..." repeatedly

```bash
# Now create the service to unblock the init container
kubectl create service clusterip myservice --tcp=80:80
```

```bash
# Watch the pod — it should now proceed
kubectl get pod init-demo-pod -w
```
Output:
```
init-demo-pod    0/1     Init:0/1       0          45s
init-demo-pod    0/1     PodInitializing  0         46s   ← init done!
init-demo-pod    1/1     Running          0         47s   ← main started!
```

---

### LAB 2 — Init Container: Prepare Files for Main Container

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: file-prep-pod
spec:
  initContainers:
  - name: file-preparer
    image: busybox:1.35
    command:
    - sh
    - -c
    - |
      echo "Preparing config file..."
      echo "DB_HOST=postgres-svc" > /shared/config.env
      echo "APP_PORT=8080" >> /shared/config.env
      echo "LOG_LEVEL=INFO" >> /shared/config.env
      echo "Config file created:"
      cat /shared/config.env
    volumeMounts:
    - name: shared-data
      mountPath: /shared
  containers:
  - name: main-app
    image: busybox:1.35
    command:
    - sh
    - -c
    - |
      echo "Main app starting..."
      echo "Reading config prepared by init container:"
      cat /shared/config.env
      sleep 3600
    volumeMounts:
    - name: shared-data
      mountPath: /shared        # SAME volume — reads what init wrote
  volumes:
  - name: shared-data
    emptyDir: {}
EOF
```

```bash
# Wait for pod to be running
kubectl wait pod file-prep-pod --for=condition=Ready --timeout=30s
```

```bash
# Check init container logs — it prepared the file
kubectl logs file-prep-pod -c file-preparer
```
Output:
```
Preparing config file...
Config file created:
DB_HOST=postgres-svc
APP_PORT=8080
LOG_LEVEL=INFO
```

```bash
# Check main container logs — it read the file init prepared
kubectl logs file-prep-pod -c main-app
```
Output:
```
Main app starting...
Reading config prepared by init container:
DB_HOST=postgres-svc
APP_PORT=8080
LOG_LEVEL=INFO
```

```bash
# Verify the shared volume has the file
kubectl exec file-prep-pod -c main-app -- cat /shared/config.env
```
- `-c main-app` → exec into the main-app container
- The main-app container can read the file the init container wrote

---

### LAB 3 — Sidecar Container: Log Shipping Pattern

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: sidecar-logging-pod
spec:
  containers:
  - name: main-app
    image: busybox:1.35
    command:
    - sh
    - -c
    - |
      mkdir -p /var/log/app
      while true; do
        echo "$(date): Processing payment transaction ID: $RANDOM" >> /var/log/app/app.log
        sleep 2
      done
    volumeMounts:
    - name: log-volume
      mountPath: /var/log/app     # writes logs here

  - name: log-sidecar
    image: busybox:1.35
    command:
    - sh
    - -c
    - |
      echo "Log sidecar started. Watching for logs..."
      tail -f /var/log/app/app.log  # reads same logs
    volumeMounts:
    - name: log-volume
      mountPath: /var/log/app     # reads from SAME volume

  volumes:
  - name: log-volume
    emptyDir: {}
EOF
```

```bash
# Watch what the log sidecar sees (in real time)
kubectl logs sidecar-logging-pod -c log-sidecar -f
```
- `-c log-sidecar` → logs from the sidecar container
- `-f` → follow (stream) — see new lines as they appear

Output:
```
Log sidecar started. Watching for logs...
Mon Jan 15 10:30:01 UTC 2024: Processing payment transaction ID: 18234
Mon Jan 15 10:30:03 UTC 2024: Processing payment transaction ID: 7823
Mon Jan 15 10:30:05 UTC 2024: Processing payment transaction ID: 29847
```

The sidecar is reading the SAME log file the main app is writing to — through the shared emptyDir volume.

```bash
# Also check main app is writing to the same volume
kubectl logs sidecar-logging-pod -c main-app
```

```bash
# Verify shared volume from both containers
kubectl exec sidecar-logging-pod -c main-app -- ls /var/log/app/
kubectl exec sidecar-logging-pod -c log-sidecar -- ls /var/log/app/
# Both see: app.log
```

---

### LAB 4 — Ephemeral Container: Debug a Running Pod

```bash
# Create a minimal pod with no debug tools
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: minimal-pod
spec:
  containers:
  - name: app
    image: nginx:1.25
EOF
```

```bash
kubectl wait pod minimal-pod --for=condition=Ready --timeout=30s
```

```bash
# Try to exec into it and run curl — might work for nginx but...
kubectl exec -it minimal-pod -- sh -c "which curl"
# nginx image may not have curl
```

```bash
# Inject an ephemeral debug container
kubectl debug -it minimal-pod \
  --image=busybox:1.35 \
  --target=app
```
- `debug` → kubectl debug subcommand
- `-it` → interactive terminal
- `minimal-pod` → the pod to debug
- `--image=busybox:1.35` → the debug image to inject (has sh, wget, netstat etc.)
- `--target=app` → share the process namespace of the `app` container

Inside the ephemeral container:
```bash
# Check the network
wget -qO- http://localhost:80
# Gets nginx response!

# Check what's listening
netstat -tlnp 2>/dev/null || ss -tlnp
# Shows port 80 listening

# Check DNS
nslookup kubernetes.default.svc.cluster.local

# Check environment
env | grep -v PATH | head -20

exit
```

```bash
# After exit, check the ephemeral containers (in a different terminal)
kubectl describe pod minimal-pod | grep -A5 "Ephemeral"
```
Output shows the ephemeral container in the pod's status.

---

### LAB 5 — Multiple Init Containers — Observe Sequential Execution

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: multi-init-pod
spec:
  initContainers:
  - name: init-step-1
    image: busybox:1.35
    command: ["sh", "-c", "echo 'STEP 1: Starting' && sleep 3 && echo 'STEP 1: Done'"]
  - name: init-step-2
    image: busybox:1.35
    command: ["sh", "-c", "echo 'STEP 2: Starting' && sleep 3 && echo 'STEP 2: Done'"]
  - name: init-step-3
    image: busybox:1.35
    command: ["sh", "-c", "echo 'STEP 3: Starting' && sleep 3 && echo 'STEP 3: Done'"]
  containers:
  - name: main-app
    image: busybox:1.35
    command: ["sh", "-c", "echo 'MAIN APP: All init containers completed! Running...' && sleep 3600"]
EOF
```

```bash
# Watch the status change in REAL TIME (open a separate terminal)
kubectl get pod multi-init-pod -w
```

Expected output (live):
```
NAME             READY   STATUS       RESTARTS   AGE
multi-init-pod   0/1     Init:0/3     0          0s   ← step-1 running
multi-init-pod   0/1     Init:1/3     0          4s   ← step-2 running
multi-init-pod   0/1     Init:2/3     0          8s   ← step-3 running
multi-init-pod   0/1     PodInitializing  0      12s  ← all done
multi-init-pod   1/1     Running      0          13s  ← main running
```

```bash
# Check logs of each init container in ORDER
kubectl logs multi-init-pod -c init-step-1
kubectl logs multi-init-pod -c init-step-2
kubectl logs multi-init-pod -c init-step-3
kubectl logs multi-init-pod -c main-app
```

---

### LAB 6 — Cleanup

```bash
kubectl delete pod init-demo-pod file-prep-pod sidecar-logging-pod minimal-pod multi-init-pod
kubectl delete service myservice
```

---

## 📊 SECTION 8 — COMPLETE COMPARISON TABLE

| Feature | Init Container | Sidecar Container | Ephemeral Container |
|---------|---------------|-------------------|---------------------|
| When it runs | BEFORE main containers | ALONGSIDE main containers | Injected into RUNNING pod |
| Lifetime | Until it exits (code 0) | Same as pod lifetime | Until you exit (one-time) |
| Purpose | Setup/preparation | Ongoing support | Debugging only |
| Defined in pod spec? | ✅ Yes (initContainers:) | ✅ Yes (containers:) | ❌ No (kubectl debug only) |
| Can restart? | Yes (if fails) | Yes (like any container) | ❌ No |
| Sequential execution? | ✅ Yes (one at a time) | ❌ No (parallel with main) | N/A |
| Shares volumes with main? | ✅ Yes | ✅ Yes | ✅ Yes |
| Shares network with main? | ✅ Yes | ✅ Yes | ✅ Yes |
| Can have different image? | ✅ Yes | ✅ Yes | ✅ Yes (purpose: debug images) |
| Example image | busybox, alpine | fluent-bit, envoy | netshoot, busybox |
| Native sidecar (K8s 1.29)? | N/A | `restartPolicy: Always` | N/A |

---

## 🔑 SECTION 9 — KEY TERMS TO REMEMBER

| Term | Simple Meaning |
|------|---------------|
| **Init Container** | Runs before main containers, sequentially, exits when done |
| **Sidecar Container** | Runs alongside main container for the pod's entire life |
| **Ephemeral Container** | Injected into running pod for debugging, not in pod spec |
| **initContainers:** | YAML field for defining init containers (separate from containers:) |
| **Sequential execution** | Init containers run ONE AT A TIME in order |
| **Init:0/3** | Pod status showing 0 of 3 init containers completed |
| **PodInitializing** | All init containers done, main containers starting |
| **Shared volume** | emptyDir used to pass data between containers in same pod |
| **restartPolicy: Always** | Makes an initContainer a native sidecar (K8s 1.29+) |
| **Sidecar Pattern** | Helper container extends main container without code changes |
| **Ambassador Pattern** | Proxy container handles outgoing traffic complexity |
| **Adapter Pattern** | Transformer container converts output format |
| **kubectl debug** | Command to inject ephemeral containers into running pods |
| **--target** | In kubectl debug: which container to share process namespace with |
| **shareProcessNamespace** | Pod setting: all containers see each other's processes |
| **nc -z host port** | netcat check: is this port open? (used in wait loops) |
| **until command; do** | Shell loop: repeat UNTIL command succeeds |
| **Distroless image** | Container image with ONLY the app binary, no shell or tools |
| **nicolaka/netshoot** | Popular debug image with all networking tools |
| **Fluent Bit** | Lightweight log shipper (sidecar for logs) |
| **Envoy** | High-performance proxy (used by Istio as sidecar) |
| **Process namespace** | Linux process namespace — shared in pod via shareProcessNamespace |
| **emptyDir** | Temporary shared volume — perfect for sidecar data sharing |

---

*File: K8s_ContainerTypes_Concept_and_Lab.md*
*Repository: Interview_Preparation_2026 → Kubernetes/*
*Next: K8s_ContainerTypes_Interview_Questions.md*
