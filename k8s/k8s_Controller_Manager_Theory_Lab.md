# Kubernetes Controller Manager — Deep Dive Concept + Hands-On Lab
> Written for: Someone with 4 years of DevOps experience preparing for senior-level interviews
> Style: First-standard student explanation → deep technical truth → hands-on lab with line-by-line explanation

---

## 🧠 SECTION 1 — WHAT IS A CONTROLLER MANAGER? (Story First)

### Start with a story — forget Kubernetes for 2 minutes

Imagine you own a bakery. You tell your bakery manager:

> *"I always want exactly 3 bakers working at any time."*

Now the manager's job is simple:
- He walks around every few seconds and **counts** how many bakers are currently working.
- If he sees only 2 bakers → he **calls a new baker** to come in.
- If he sees 4 bakers → he **sends one home**.
- He never stops. He never sleeps. He keeps checking forever.

That bakery manager **is the Controller Manager in Kubernetes.**

You tell Kubernetes:
> *"I want 3 copies of my application running."*

The Controller Manager walks around (watches the cluster), counts what's actually running, and if the number doesn't match what you asked for — **it fixes it automatically.**

This concept has a name: **Reconciliation Loop** — also called the **Control Loop**.

```
You say:   "I want 3 pods"      ← DESIRED STATE
Reality:   "Only 2 pods exist"  ← ACTUAL STATE
Controller: "Gap detected → create 1 more pod"  ← RECONCILE
```

This loop runs **forever, every few seconds**, silently in the background.

---

## 🏗️ SECTION 2 — WHERE DOES CONTROLLER MANAGER LIVE?

### The Kubernetes Control Plane

Before going deeper, you need to know the big picture. Kubernetes has two types of machines:

```
┌─────────────────────────────────────────────┐
│            CONTROL PLANE (Master Node)       │
│                                             │
│  ┌─────────────────┐  ┌──────────────────┐  │
│  │   API Server    │  │  Controller      │  │
│  │  (front door)   │  │  Manager         │  │
│  └─────────────────┘  └──────────────────┘  │
│                                             │
│  ┌─────────────────┐  ┌──────────────────┐  │
│  │     etcd        │  │   Scheduler      │  │
│  │  (database)     │  │                  │  │
│  └─────────────────┘  └──────────────────┘  │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│            WORKER NODES (where apps run)    │
│                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │ kubelet  │  │kube-proxy│  │containers│  │
│  └──────────┘  └──────────┘  └──────────┘  │
└─────────────────────────────────────────────┘
```

**Controller Manager lives on the Control Plane (Master Node).**

It is a **single binary** — one process — but inside it runs **many controllers** at the same time. Think of it like a building that has one address, but inside it has many offices — each office doing a different job.

```
kube-controller-manager (one process)
        │
        ├── Deployment Controller
        ├── ReplicaSet Controller
        ├── Node Controller
        ├── Job Controller
        ├── CronJob Controller
        ├── Service Account Controller
        ├── Endpoint Controller
        ├── Namespace Controller
        └── ... (many more)
```

---

## 🔄 SECTION 3 — THE CONTROL LOOP (Most Important Concept)

Every single controller inside the Controller Manager follows the **exact same pattern**:

```
STEP 1: WATCH
    → Controller watches the API Server for changes
    → "Tell me if anything changes related to my job"

STEP 2: COMPARE
    → What does the user WANT? (Desired State — stored in etcd)
    → What actually EXISTS? (Actual/Current State — reality)

STEP 3: ACT
    → If desired ≠ actual → take action to fix the gap
    → If desired = actual → do nothing, keep watching

STEP 4: REPEAT
    → Go back to STEP 1. Forever.
```

This is called the **Reconciliation Loop** or **Control Loop**.

### Why is this powerful?

Because Kubernetes **self-heals automatically**. You don't need to write scripts saying "if a pod dies, restart it." The controller is always watching and always fixing.

```
Example:
  You create a Deployment saying: replicas: 3
  3 pods start running — all good.

  Suddenly — a node crashes. 1 pod dies.
  Now only 2 pods are running.

  Controller detects: "Desired=3, Actual=2 → gap of 1"
  Controller creates a new pod on a healthy node.
  Back to 3 pods. All without you doing anything.
```

---

## 📦 SECTION 4 — TYPES OF CONTROLLERS (Each one explained simply)

### 4.1 — Deployment Controller

**What is its job?**
When you create a Deployment, this controller is responsible for **creating and managing ReplicaSets**.

**Simple story:**
You (the user) talk to the Deployment Controller and say:
> *"I want my app running with 3 copies. And if I update my app, do it slowly — don't kill all 3 at once."*

The Deployment Controller hears this and says:
> *"Okay. I'll create a ReplicaSet to manage those 3 pods. And if you update, I'll create a NEW ReplicaSet for the new version and slowly shift traffic from old to new."*

```
You create: Deployment (nginx, replicas: 3)
                    │
                    ▼
         Deployment Controller creates:
                    │
                    ▼
              ReplicaSet (nginx, replicas: 3)
                    │
                    ▼
         ReplicaSet Controller creates:
                    │
                    ▼
         Pod 1    Pod 2    Pod 3
```

**Key responsibility:**
- Creates ReplicaSets
- Handles **rolling updates** (new ReplicaSet created, old one scaled down slowly)
- Handles **rollbacks** (scale old ReplicaSet back up)

---

### 4.2 — ReplicaSet Controller

**What is its job?**
Make sure the **exact number of pod copies** you asked for is always running.

**Simple story:**
The ReplicaSet Controller is like the bakery manager from our story. His ONLY job is counting pods and maintaining the count.

```
ReplicaSet says: "I need 3 pods with label app=nginx"

ReplicaSet Controller:
  - Counts pods matching label app=nginx
  - If count < 3 → create more pods
  - If count > 3 → delete extra pods
  - If count = 3 → do nothing, keep watching
```

**Important truth:** You almost never create a ReplicaSet directly. You create a **Deployment**, and the Deployment Controller creates the ReplicaSet for you. The Deployment gives you extra features (rolling update, rollback) that a raw ReplicaSet doesn't have.

---

### 4.3 — Node Controller

**What is its job?**
Watch all the nodes (worker machines) and react if a node stops responding.

**Simple story:**
Imagine you manage 10 employees (nodes). The Node Controller is the HR manager who keeps calling each employee every few seconds. If an employee doesn't answer for 40 seconds — they're marked as "not responding." If they don't answer for 5 minutes — their work is given to someone else (pods are rescheduled).

```
Node Controller timeline:
  Node stops responding...
  
  After 40 seconds → Node status: "NotReady" (marked in API Server)
  After 5 minutes  → All pods on that node get "Evicted"
                   → Deployment/ReplicaSet controllers create replacement pods
                   → Replacement pods scheduled on healthy nodes
```

**What does it watch?**
- Node heartbeats (kubelet on each node sends a heartbeat every few seconds)
- If heartbeat stops → Node Controller takes action

---

### 4.4 — Job Controller

**What is its job?**
Make sure a **one-time task** runs to completion and then stops.

**Simple story:**
A Deployment says "always keep 3 pods running." But sometimes you just need to run a task once — like sending a monthly report email, or running a database migration. You don't want it running forever. That's what a **Job** is.

```
Job: "Run this database backup script once. Make sure it finishes successfully."

Job Controller:
  - Creates a pod to run the script
  - Watches if the pod completed successfully (exit code 0)
  - If pod succeeded → Job is Done ✅
  - If pod failed → create another pod and try again
  - After success → pod stays in Completed state (not deleted immediately)
```

**Key difference from Deployment:**
- Deployment: "always keep X pods RUNNING forever"
- Job: "run this once, make sure it COMPLETES, then stop"

---

### 4.5 — CronJob Controller

**What is its job?**
Run a Job **on a schedule** — like a cron job in Linux.

**Simple story:**
Same as Job Controller, but with a timer. Instead of "run once now," it's "run every day at midnight" or "run every 5 minutes."

```
CronJob: "Every day at 2 AM, run the database backup job"

CronJob Controller:
  - Watches the clock
  - At 2 AM every day → creates a Job object
  - Job Controller then handles running that Job
  - Keeps history of last N successful and failed jobs
```

**CronJob schedule format** — same as Linux cron:
```
┌───────────── minute (0-59)
│ ┌─────────── hour (0-23)
│ │ ┌───────── day of month (1-31)
│ │ │ ┌─────── month (1-12)
│ │ │ │ ┌───── day of week (0-6, Sunday=0)
│ │ │ │ │
* * * * *

Examples:
  0 2 * * *     = every day at 2:00 AM
  */5 * * * *   = every 5 minutes
  0 0 1 * *     = first day of every month at midnight
```

---

### 4.6 — Endpoint Controller (EndpointSlice Controller)

**What is its job?**
When a Service needs to find which pods to send traffic to, the Endpoint Controller **keeps that list updated**.

**Simple story:**
A Service is like a phone operator. When someone calls "payment-service," the operator needs to know which phones (pod IPs) are currently active and ready to receive calls. The Endpoint Controller is the person who keeps the operator's list updated.

```
Service: selector: app=payment

Endpoint Controller:
  - Watches all pods
  - Finds pods matching label app=payment that are in "Ready" state
  - Updates the Endpoints object with their IP addresses
  - If a pod dies → removes its IP from the list
  - If a new pod becomes Ready → adds its IP to the list

Result: Service always knows which pods are alive and ready
```

**Why does Ready state matter?**
If a pod is Running but its readiness probe is failing → the Endpoint Controller does NOT add it to the endpoints list → no traffic reaches that pod. This is the correct behavior.

---

### 4.7 — Service Account Controller

**What is its job?**
When you create a new Namespace, this controller **automatically creates a default ServiceAccount** in that namespace.

**Simple story:**
When a new department (namespace) is created in a company, HR automatically creates a "guest badge" (default service account) for that department. Any new employees (pods) who join that department get this guest badge unless they're given a specific one.

```
You create: Namespace called "banking"
                    │
                    ▼
Service Account Controller automatically creates:
  ServiceAccount named "default" in namespace "banking"
  
Every pod created in "banking" namespace gets this default
service account token injected automatically.
```

---

### 4.8 — Namespace Controller

**What is its job?**
When you **delete a Namespace**, this controller deletes **everything inside it** — all pods, services, configmaps, secrets, etc.

**Simple story:**
When you close down a whole department in a company, someone has to clean up that department's office — all the desks, files, equipment. The Namespace Controller is that cleanup crew.

```
You run: kubectl delete namespace banking

Namespace Controller:
  - Marks namespace as "Terminating"
  - Deletes all objects inside: pods, services, deployments, secrets, configmaps...
  - Once everything is deleted → removes the namespace itself
```

**Why does it sometimes get stuck in "Terminating"?**
Because some resource inside the namespace is refusing to be deleted (usually due to finalizers — a protection mechanism). The namespace waits until ALL resources are cleaned up before it disappears.

---

### 4.9 — StatefulSet Controller

**What is its job?**
Manages **StatefulSet pods** — makes sure they have stable names, stable storage, and are created/deleted in ORDER.

**Simple story:**
A regular Deployment treats all pods as identical and interchangeable. A StatefulSet treats each pod as a UNIQUE individual with its own identity.

```
Deployment pods:    nginx-7d8f9-x4k2p (random)
                    nginx-7d8f9-m9pz1 (random)
                    
StatefulSet pods:   postgres-0  (always this name)
                    postgres-1  (always this name)
                    postgres-2  (always this name)

StatefulSet Controller ensures:
  - postgres-0 starts FIRST, must be Running before postgres-1 starts
  - postgres-1 starts SECOND, must be Running before postgres-2 starts
  - Each pod gets its OWN PersistentVolumeClaim (own storage)
  - If postgres-1 dies and restarts → it comes back as postgres-1 (same name)
    and reconnects to the SAME storage it had before
```

---

### 4.10 — DaemonSet Controller

**What is its job?**
Make sure **one copy of a pod runs on EVERY node** in the cluster.

**Simple story:**
Every office building needs a security guard. You don't say "I need 5 guards for my 20 buildings" — you say "every building gets 1 guard." If a new building opens → a guard is placed there automatically. If a building closes → that guard's job ends.

```
DaemonSet: "Run Fluentd log collector on every node"

DaemonSet Controller:
  - Watches how many nodes exist (e.g., 10 nodes)
  - Ensures Fluentd pod is running on all 10 nodes
  - New node added (node 11) → DaemonSet Controller creates Fluentd on it
  - Node removed → DaemonSet Controller removes Fluentd from it
  - You cannot set replicas — the count equals the node count
```

**Real-world DaemonSets you'll always see:**
- `kube-proxy` — networking on every node
- `calico-node` / `cilium` — CNI networking plugin
- `node-exporter` — Prometheus monitoring agent
- `fluent-bit` / `fluentd` — log collection

---

### 4.11 — Horizontal Pod Autoscaler (HPA) Controller

**What is its job?**
Automatically **scale the number of pods up or down** based on CPU or memory usage.

**Simple story:**
At a call center, you start the day with 3 agents. As call volume increases, you bring in more agents. When calls drop, some agents go home. The HPA Controller is the supervisor who watches call volume and adjusts agent count.

```
HPA: "Keep CPU usage around 50%. Min 2 pods, Max 10 pods."

HPA Controller:
  - Checks metrics every 15 seconds (from metrics-server)
  - Current state: 3 pods, CPU at 90% → too high
  - Action: scale up to 6 pods
  
  - Later: 6 pods, CPU at 20% → too low
  - Action: scale down to 3 pods (after cooldown period)
```

**Important:** HPA needs **metrics-server** to be installed in the cluster. Without it, HPA cannot read CPU/memory and won't work.

---

## 🔍 SECTION 5 — HOW CONTROLLERS COMMUNICATE (The Watch Mechanism)

This is the deep technical truth of how controllers actually work.

### The API Server as the Central Hub

**No controller talks to etcd directly.** All controllers only talk to the **API Server**.

```
Controller Manager
    │
    │  "Watch for changes to Deployments"
    ▼
API Server
    │
    │  "Reads from and writes to etcd"
    ▼
   etcd
```

### The Watch and Informer Pattern

When a controller starts, it does two things:

**1. List** — "Give me ALL existing Deployments right now"
**2. Watch** — "Now keep sending me ANY new changes to Deployments"

This is called the **List-Watch** pattern.

The API Server sends **events** back to the controller:
- `ADDED` — a new Deployment was created
- `MODIFIED` — an existing Deployment was changed
- `DELETED` — a Deployment was deleted

The controller reacts to each event by running its reconciliation loop.

```
Controller starts:
  → LIST all Deployments (get current state)
  → WATCH for new events

API Server sends event: "ADDED — new Deployment 'payment-app' with replicas:3"
Controller reacts:
  → "I need to create a ReplicaSet for this"
  → Calls API Server: "Please create ReplicaSet with replicas:3"
  → API Server writes to etcd
  → ReplicaSet Controller gets notified: "ADDED — new ReplicaSet"
  → ReplicaSet Controller creates 3 pods
```

### The Work Queue

Inside each controller there is a **Work Queue**. When an event comes in, it's added to the queue. The controller processes items from the queue one by one and runs the reconciliation logic.

```
Event comes in → added to Work Queue → Controller picks it up → Runs reconcile → Done
                                                                        │
                                                        If error → put back in queue → retry later
```

This is why Kubernetes is **eventually consistent** — it might take a few seconds for changes to propagate, but eventually everything converges to the desired state.

---

## 🔐 SECTION 6 — LEADER ELECTION (High Availability)

### What happens when you run multiple control planes?

In a production cluster, you run **3 control plane nodes** (for high availability). But if 3 copies of Controller Manager are running at the same time, they might all try to fix the same problem — creating duplicate pods, causing chaos.

**Solution: Leader Election**

Only ONE Controller Manager is **active (the leader)** at any time. The other two are **standby (followers)** — waiting and watching.

```
3 Controller Manager instances:

cm-1 → LEADER (actively running reconciliation loops)
cm-2 → Standby (doing nothing, watching for leader failure)
cm-3 → Standby (doing nothing, watching for leader failure)

If cm-1 crashes:
  cm-2 and cm-3 race to become the new leader
  One wins → becomes new LEADER
  Other → remains standby
```

**How do they elect a leader?**
They use a **Kubernetes Lease object** (a lock in etcd). The leader continuously renews this lease every 2 seconds. If the lease is not renewed within 10 seconds → other candidates assume the leader is dead → they compete for the lease → one wins.

```bash
# You can see the leader election lease
kubectl get lease kube-controller-manager -n kube-system

# Output:
NAME                     HOLDER              AGE
kube-controller-manager  controlplane-node1  10d
# "controlplane-node1" is the current leader
```

---

## 💻 SECTION 7 — HANDS-ON LAB

> Every single command is explained word by word. Every YAML line is explained. No assumptions.

### Lab Prerequisites

You need:
- MicroK8s running on your ThinkPad (Ubuntu) OR KillerCoda playground
- `kubectl` working
- Terminal open

---

### LAB 1 — Find the Controller Manager Running in Your Cluster

```bash
kubectl get pods -n kube-system
```

**Breaking this command down word by word:**
- `kubectl` → the command-line tool to talk to Kubernetes
- `get` → "show me" / "list"
- `pods` → the type of object we want to list
- `-n kube-system` → `-n` means "namespace". `kube-system` is the special namespace where all Kubernetes internal components run

**What you'll see:**
```
NAME                                        READY   STATUS    RESTARTS   AGE
kube-controller-manager-controlplane        1/1     Running   0          10d
kube-apiserver-controlplane                 1/1     Running   0          10d
etcd-controlplane                           1/1     Running   0          10d
kube-scheduler-controlplane                 1/1     Running   0          10d
```

**Now describe the Controller Manager pod:**

```bash
kubectl describe pod kube-controller-manager-controlplane -n kube-system
```

**Breaking down:**
- `describe` → give me DETAILED information (not just a summary like `get` does)
- `pod` → the object type
- `kube-controller-manager-controlplane` → the name of the pod
- `-n kube-system` → in the kube-system namespace

**Look at the output — specifically the `Command` section:**
```
Command:
  kube-controller-manager
  --allocate-node-cidrs=true
  --authentication-kubeconfig=/etc/kubernetes/controller-manager.conf
  --controllers=*,bootstrapsigner,tokencleaner
  --leader-elect=true
  --node-monitor-grace-period=40s
  --pod-eviction-timeout=5m0s
```

**What each flag means:**
- `--controllers=*` → the `*` means "run ALL built-in controllers". You could list specific ones here.
- `--leader-elect=true` → "yes, do leader election — only one instance should be active"
- `--node-monitor-grace-period=40s` → "wait 40 seconds before marking a node as NotReady after heartbeat stops"
- `--pod-eviction-timeout=5m0s` → "evict pods from a NotReady node after 5 minutes"

---

### LAB 2 — See the Reconciliation Loop in Action (ReplicaSet Controller)

**Step 1: Create a Deployment**

```bash
kubectl create deployment nginx-demo --image=nginx --replicas=3
```

**Breaking down:**
- `create deployment` → create a new Deployment object
- `nginx-demo` → the name we're giving this deployment
- `--image=nginx` → use the official nginx container image (pulled from Docker Hub)
- `--replicas=3` → I want 3 copies of this pod running

**Step 2: Watch pods being created in real time**

```bash
kubectl get pods -w
```

**Breaking down:**
- `get pods` → list pods
- `-w` → `--watch` — don't just show once, keep watching and print any changes as they happen (like `tail -f` for pods)

**You'll see something like:**
```
NAME                          READY   STATUS              RESTARTS   AGE
nginx-demo-7d8f9-x4k2p        0/1     Pending             0          0s
nginx-demo-7d8f9-x4k2p        0/1     ContainerCreating   0          1s
nginx-demo-7d8f9-x4k2p        1/1     Running             0          3s
nginx-demo-7d8f9-m9pz1        0/1     Pending             0          0s
...
```

**Step 3: Now MANUALLY DELETE one pod — and watch the Controller bring it back**

Open a second terminal and run:

```bash
# Get the exact pod name first
kubectl get pods

# Delete one pod (replace with your actual pod name)
kubectl delete pod nginx-demo-7d8f9-x4k2p
```

**In your first terminal (where -w is running), you'll see:**
```
nginx-demo-7d8f9-x4k2p        1/1     Terminating   0          2m
nginx-demo-7d8f9-newpod       0/1     Pending       0          0s
nginx-demo-7d8f9-newpod       1/1     Running       0          3s
```

**What just happened:**
1. You deleted 1 pod → now only 2 pods exist
2. ReplicaSet Controller detected: "Desired=3, Actual=2 → gap!"
3. ReplicaSet Controller told API Server: "Create 1 new pod"
4. Scheduler assigned the pod to a node
5. kubelet on that node started the container
6. Back to 3 pods — without you doing anything

**This IS the reconciliation loop — you just watched it happen live.**

---

### LAB 3 — See the ReplicaSet That Deployment Controller Created

```bash
kubectl get replicasets
```

**What you'll see:**
```
NAME                    DESIRED   CURRENT   READY   AGE
nginx-demo-7d8f9abc     3         3         3       5m
```

- `DESIRED` → how many pods should exist (from your Deployment)
- `CURRENT` → how many pod objects have been created
- `READY` → how many pods are actually running and healthy

```bash
# See the relationship — Deployment owns ReplicaSet owns Pods
kubectl get all
```

**Breaking down:**
- `get all` → show me ALL types of objects in the current namespace (pods, services, deployments, replicasets)

**Now look at who CREATED the ReplicaSet:**

```bash
kubectl describe replicaset nginx-demo-7d8f9abc
```

Look for the `Controlled By` line:
```
Controlled By:  Deployment/nginx-demo
```

This proves: **Deployment Controller created this ReplicaSet**.

And now check a pod:
```bash
kubectl describe pod nginx-demo-7d8f9abc-x4k2p
```

Look for `Controlled By`:
```
Controlled By:  ReplicaSet/nginx-demo-7d8f9abc
```

**The full ownership chain:**
```
Deployment → owns → ReplicaSet → owns → Pods
```

---

### LAB 4 — See the Job Controller in Action

**Create a simple one-time job:**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: hello-job
spec:
  template:
    spec:
      containers:
      - name: hello
        image: busybox
        command: ["sh", "-c", "echo Hello from the Job! && sleep 5"]
      restartPolicy: Never
  backoffLimit: 3
EOF
```

**Explaining every single line of this YAML:**

```yaml
apiVersion: batch/v1
```
→ `apiVersion` tells Kubernetes which API group and version to use.
→ `batch/v1` = the "batch" group (for Jobs and CronJobs), version 1.
→ Different object types belong to different API groups. Pods and Deployments use `apps/v1`. Jobs use `batch/v1`.

```yaml
kind: Job
```
→ `kind` = what TYPE of object are we creating. Here: a Job.

```yaml
metadata:
  name: hello-job
```
→ `metadata` = information ABOUT this object (not the actual behavior).
→ `name: hello-job` = we're calling this Job "hello-job". This is how you'll reference it with kubectl.

```yaml
spec:
```
→ `spec` = specification. This is where you describe WHAT you want.
→ Everything under `spec` is the actual desired behavior.

```yaml
  template:
    spec:
      containers:
```
→ `template` = the pod template. Jobs create pods to do the work. This describes what those pods should look like.
→ Inside template there's another `spec` — this inner spec is the POD spec.
→ `containers` = list of containers inside each pod. A pod can have multiple containers.

```yaml
      - name: hello
        image: busybox
        command: ["sh", "-c", "echo Hello from the Job! && sleep 5"]
```
→ `-` (dash) = list item. This is ONE container definition.
→ `name: hello` = name of this container (for identification inside the pod).
→ `image: busybox` = use the "busybox" Docker image. BusyBox is a tiny Linux image used for testing — has basic tools like `sh`, `echo`, `sleep`.
→ `command` = override the default command of the container image. Run this instead.
→ `["sh", "-c", "echo Hello from the Job! && sleep 5"]`
  - `sh` = start a shell
  - `-c` = "run the following string as a command"
  - `"echo Hello from the Job! && sleep 5"` = print "Hello from the Job!" then wait 5 seconds
  - `&&` = "and then" — run second command only if first succeeded

```yaml
      restartPolicy: Never
```
→ `restartPolicy` = what to do if the container fails.
→ `Never` = if this container crashes, do NOT restart it inside the same pod. Instead, the Job Controller creates a FRESH new pod.
→ For Jobs, this must be either `Never` or `OnFailure`. Regular Deployments use `Always`.

```yaml
  backoffLimit: 3
```
→ `backoffLimit` = if the job fails, how many times to retry before giving up.
→ `3` = try up to 3 times total. If all 3 fail → Job status becomes Failed.

**Now watch the Job:**

```bash
kubectl get jobs -w
```
**Breaking down:**
- `get jobs` → list Job objects
- `-w` → watch for changes

```bash
# After it completes, check the logs
kubectl logs -l job-name=hello-job
```
**Breaking down:**
- `logs` → show logs from container stdout
- `-l job-name=hello-job` → `-l` means label selector. Instead of specifying a pod name, select by label. The Job automatically adds a label `job-name=hello-job` to all its pods.

**You should see:**
```
Hello from the Job!
```

**Check job status:**
```bash
kubectl describe job hello-job
```

Look for:
```
Completions:  1/1        ← 1 succeeded out of 1 needed
Pods Statuses:  0 Running / 1 Succeeded / 0 Failed
```

---

### LAB 5 — See the CronJob Controller in Action

```bash
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: CronJob
metadata:
  name: minute-logger
spec:
  schedule: "* * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: logger
            image: busybox
            command: ["sh", "-c", "echo The time is $(date)"]
          restartPolicy: OnFailure
EOF
```

**Explaining the new/different lines:**

```yaml
kind: CronJob
```
→ Creating a CronJob object (not a Job).

```yaml
  schedule: "* * * * *"
```
→ The cron schedule. `* * * * *` = "every minute of every hour of every day" — runs every 1 minute.
→ In real use: `0 2 * * *` = 2 AM daily, `*/15 * * * *` = every 15 minutes.

```yaml
  jobTemplate:
```
→ `jobTemplate` = the template for the Job that gets created each time the schedule fires.
→ The CronJob Controller reads this and creates a Job from it on schedule.
→ Inside `jobTemplate` → same structure as a regular Job.

```yaml
          restartPolicy: OnFailure
```
→ `OnFailure` = if the container crashes, restart it IN THE SAME POD (up to a limit).
→ Different from `Never` (which creates a new pod on failure) and `Always` (Deployment behavior).

**Watch it run every minute:**
```bash
kubectl get jobs -w
```

**After 2-3 minutes, you'll see multiple jobs:**
```
NAME                       COMPLETIONS   DURATION   AGE
minute-logger-28123456     1/1           5s         2m
minute-logger-28123457     1/1           5s         1m
minute-logger-28123458     0/1           3s         3s  ← running now
```

Each line = CronJob Controller created a new Job for each minute.

**Clean up:**
```bash
kubectl delete cronjob minute-logger
kubectl delete job hello-job
kubectl delete deployment nginx-demo
```

**Breaking down delete:**
- `delete` = remove this object from the cluster
- `cronjob minute-logger` = object type + name
- Deleting a CronJob also deletes its child Jobs (and their pods)
- Deleting a Deployment also deletes its ReplicaSets and Pods

---

### LAB 6 — Watch the Node Controller React (Simulate Node Pressure)

```bash
# Check node conditions right now
kubectl describe node | grep -A 10 "Conditions:"
```

**Breaking down:**
- `describe node` → detailed info about nodes (if you have one node, it describes it; with multiple, add the node name)
- `|` = pipe. Send the output of the left command as input to the right command.
- `grep` = search for text
- `-A 10` = `-A` means "After" — show 10 lines AFTER the matching line
- `"Conditions:"` = the text to search for

**Output you'll see:**
```
Conditions:
  Type                 Status  Message
  ----                 ------  -------
  NetworkUnavailable   False   Flannel is running on this node
  MemoryPressure       False   kubelet has sufficient memory available
  DiskPressure         False   kubelet has sufficient disk space available
  PIDPressure          False   kubelet has sufficient PID available
  Ready                True    kubelet is posting ready status
```

**What each condition means:**
- `MemoryPressure: False` → node has enough memory. If this becomes True → Node Controller takes action, pods get evicted.
- `DiskPressure: False` → node has enough disk. If this becomes True → new pods won't be scheduled here.
- `PIDPressure: False` → node has enough process IDs (rarely a problem).
- `Ready: True` → kubelet is healthy and sending heartbeats. If this becomes False → Node Controller starts pod eviction after 5 minutes.

---

### LAB 7 — See the Endpoint Controller in Action

```bash
# Create a deployment and expose it as a service
kubectl create deployment ep-demo --image=nginx --replicas=2
kubectl expose deployment ep-demo --port=80 --target-port=80
```

**Breaking down `expose`:**
- `expose deployment ep-demo` → create a Service that targets the pods of this Deployment
- `--port=80` → the Service listens on port 80 (the port OTHER services use to reach this service)
- `--target-port=80` → forward traffic to port 80 inside the container (where nginx listens)

**Now check the Endpoints object:**
```bash
kubectl get endpoints ep-demo
```

**Output:**
```
NAME      ENDPOINTS                       AGE
ep-demo   10.244.1.5:80,10.244.1.6:80    30s
```

These are the **actual pod IP addresses** that the Endpoint Controller discovered and registered. kube-proxy uses this list to route traffic.

**Now scale down and watch endpoints update:**
```bash
kubectl scale deployment ep-demo --replicas=0
```
**Breaking down:**
- `scale deployment ep-demo` → change the replica count of this deployment
- `--replicas=0` → scale to 0 — no pods running

```bash
kubectl get endpoints ep-demo
```
**Output:**
```
NAME      ENDPOINTS   AGE
ep-demo   <none>      2m
```

Endpoint Controller detected all pods are gone → removed all IPs → endpoints is now empty → no traffic can reach any pod (because there are no pods).

**Scale back up:**
```bash
kubectl scale deployment ep-demo --replicas=2
kubectl get endpoints ep-demo -w
```

Watch IPs appear within seconds as pods become Ready.

**Clean up:**
```bash
kubectl delete deployment ep-demo
kubectl delete service ep-demo
```

---

### LAB 8 — See Leader Election in Action

```bash
# See the current leader of Controller Manager
kubectl get lease kube-controller-manager -n kube-system -o yaml
```

**Breaking down:**
- `get lease` → list Lease objects (special lock objects used for leader election)
- `kube-controller-manager` → the name of this specific lease
- `-n kube-system` → in kube-system namespace
- `-o yaml` → `-o` means "output format". `yaml` = show the full YAML instead of the brief table view

**Output (look for holderIdentity):**
```yaml
spec:
  acquireTime: "2024-01-15T10:00:00Z"
  holderIdentity: controlplane_abc123   ← THIS is the current leader
  leaseDurationSeconds: 15
  renewTime: "2024-01-15T10:05:00Z"
```

- `holderIdentity` = which Controller Manager instance is currently the leader
- `renewTime` = when the leader last renewed the lease (updates every few seconds)
- `leaseDurationSeconds: 15` = if the lease is not renewed within 15 seconds, another instance can take over

---

## 📊 SECTION 8 — CONTROLLER MANAGER SUMMARY TABLE

| Controller | Watches | Action It Takes | Real World Use |
|------------|---------|-----------------|----------------|
| Deployment | Deployments | Creates/manages ReplicaSets, handles rolling updates | `kubectl create deployment` |
| ReplicaSet | ReplicaSets | Creates/deletes pods to maintain replica count | `kubectl scale` |
| Node | Nodes | Marks NotReady, evicts pods after node failure | Node goes down |
| Job | Jobs | Creates pods, retries on failure, marks Complete | Database migration |
| CronJob | CronJobs | Creates Jobs on schedule | Nightly backup |
| Endpoint | Services + Pods | Updates endpoint list with ready pod IPs | Service routing |
| ServiceAccount | Namespaces | Creates default SA in new namespaces | New namespace |
| Namespace | Namespaces | Cleans up all resources when namespace deleted | `kubectl delete ns` |
| StatefulSet | StatefulSets | Ordered pod creation with stable names + storage | Databases, Kafka |
| DaemonSet | Nodes | Ensures one pod per node | kube-proxy, Fluentd |
| HPA | HPA objects | Scales deployments based on CPU/memory | Auto-scaling |

---

## 🔑 SECTION 9 — KEY TERMS TO REMEMBER

| Term | Simple Meaning |
|------|---------------|
| **Reconciliation Loop** | The forever-running check: desired vs actual, fix the gap |
| **Desired State** | What YOU told Kubernetes you want |
| **Actual State** | What is REALLY happening in the cluster right now |
| **Control Loop** | Same as Reconciliation Loop |
| **List-Watch** | How controllers first get all objects, then listen for changes |
| **Leader Election** | Only ONE controller manager is active; others are on standby |
| **Lease** | A lock object in Kubernetes used for leader election |
| **Work Queue** | Internal queue inside a controller where events wait to be processed |
| **Informer** | The code pattern that implements List-Watch inside controllers |
| **Eviction** | Removing a pod from a node (Node Controller does this after node failure) |

---

*File: K8s_ControllerManager_Concept_and_Lab.md*
*Repository: Interview_Preparation_2026 → Kubernetes/*
*Next: K8s_ControllerManager_Interview_Questions.md*
