# kubectl — Complete Guide

> Level: 4 Years Experience
> Author: Muskan Patel

---

## PART 1 — WHAT IS kubectl

kubectl (pronounced "kube-control" or "kube-c-t-l") is the
command-line tool that lets YOU talk to the Kubernetes API server.

Every command you type in kubectl is converted into an HTTP request
sent to the API server. kubectl reads your `~/.kube/config` file
to know WHERE the API server is and WHO you are (your certificate).

```
You type:  kubectl get pods -n banking
                |
                v
kubectl reads ~/.kube/config
(finds: API server IP, your certificate, cluster name)
                |
                v
kubectl sends HTTP GET to API server port 6443
                |
                v
API server checks: who are you? are you allowed? fetches from etcd
                |
                v
API server returns JSON
                |
                v
kubectl formats it as a table and prints it
```

kubectl is NOT Kubernetes. It is just a client — like a TV remote
control. The TV (cluster) works without the remote, but you need
the remote to control it.

---

## PART 2 — THE STRUCTURE OF EVERY kubectl COMMAND

Every kubectl command follows the same pattern:

```
kubectl  <verb>  <resource>  <name>  <flags>
```

Example breakdown:
```
kubectl   get      pods      nginx-pod    -n banking
   |        |        |           |            |
   |        |        |           |            +-- option: which namespace
   |        |        |           +--------------- which specific pod
   |        |        +--------------------------- what type of object
   |        +------------------------------------ what action to take
   +-------------------------------------------- the CLI tool
```

---

## PART 3 — VERBS (ACTIONS)

---

### GROUP 1 — CRUD VERBS (Create, Read, Update, Delete)

---

#### `get` — READ objects from the cluster

**What it does:** Fetches one or many objects and shows them as
a table. This is the command you run FIRST in any troubleshooting
situation.

```bash
kubectl get pods
# Shows all pods in the DEFAULT namespace

kubectl get pods -n banking
# Shows all pods in the BANKING namespace

kubectl get pods --all-namespaces
kubectl get pods -A
# Shows pods in EVERY namespace across the cluster

kubectl get pod nginx-pod
# Shows ONE specific pod by name

kubectl get pods,services,deployments
# Shows MULTIPLE resource types at the same time in one output
```

---

#### `create` — CREATE a new object (fails if it already exists)

**What it does:** Creates a new Kubernetes object. If an object
with the same name already exists in that namespace, it FAILS with
an "already exists" error. Use `apply` instead for most cases.

```bash
kubectl create deployment web --image=nginx
# Creates a Deployment named "web" using the nginx image

kubectl create namespace banking
# Creates a new namespace called banking

kubectl create secret generic db-secret --from-literal=password=abc123
# Creates a Secret named db-secret with one key-value pair
```

---

#### `apply` — CREATE or UPDATE (smart — works either way)

**What it does:** The most important command in production. If the
object does NOT exist, it creates it. If it DOES exist, it updates
only the fields that changed. Never fails with "already exists."
This is what you use 95% of the time.

```bash
kubectl apply -f deployment.yaml
# Reads the YAML file and creates or updates the object

kubectl apply -f ./manifests/
# Applies ALL yaml files inside the manifests directory

kubectl apply -f https://url/to/file.yaml
# Applies a YAML file directly from a URL
```

**Why apply over create:** If you run `kubectl create` twice, it
fails the second time. If you run `kubectl apply` twice, the
second run just confirms nothing changed. Safe to run repeatedly.

---

#### `delete` — REMOVE an object permanently

**What it does:** Deletes one or more objects. For pods managed
by a Deployment, the ReplicaSet immediately creates a replacement —
so deleting a Deployment's pod does NOT remove it permanently.
To permanently remove, delete the Deployment itself.

```bash
kubectl delete pod nginx-pod
# Deletes ONE specific pod by name

kubectl delete pod nginx-pod -n banking
# Deletes the pod in a specific namespace

kubectl delete -f deployment.yaml
# Deletes whatever objects are described in the YAML file

kubectl delete pods -l app=payment
# Deletes ALL pods that have the label app=payment

kubectl delete pods --all -n banking
# Deletes ALL pods in the banking namespace (dangerous)
```

---

#### `edit` — OPEN an object's YAML in a text editor and save changes live

**What it does:** Opens the full YAML of a live object in your
default editor (usually vim). When you save and close the editor,
the change is applied to the cluster immediately. Useful for quick
one-off changes. NOT recommended for production because changes
bypass Git (not tracked in version control).

```bash
kubectl edit deployment payment-deployment
# Opens the full Deployment YAML in vim for live editing

kubectl edit pod nginx-pod
# Opens the pod YAML -- note: most pod fields are IMMUTABLE,
# so most edits will be rejected by the API server
```

---

#### `patch` — UPDATE a specific field without touching the rest

**What it does:** Changes one or more specific fields on an existing
object without needing to edit the full YAML. Useful for scripted
updates in CI/CD pipelines.

```bash
kubectl patch deployment web -p '{"spec":{"replicas":5}}'
# Changes the replica count to 5 without touching anything else
# -p means "patch" -- the content after it is JSON format
```

---

#### `replace` — REPLACE the entire object with a new definition

**What it does:** Deletes the existing object and recreates it
from the file. Different from `apply` which does an in-place update.
Rarely needed -- use `apply` in almost all cases.

```bash
kubectl replace -f deployment.yaml
# Deletes the existing deployment and recreates from file
```

---

### GROUP 2 — INSPECT VERBS

---

#### `describe` — DETAILED human-readable view including Events

**What it does:** Shows full details of an object including its
current state, configuration, and most importantly the EVENTS
section at the bottom. Events show exactly what happened --
scheduling decisions, image pulls, probe failures, errors.

This is your NUMBER ONE troubleshooting command. Always run
`describe` first when a pod is not starting, stuck in Pending,
or crashing.

```bash
kubectl describe pod nginx-pod
# Full details of the pod including Events section
# Events show: Scheduled, Pulling image, Created, Started
# OR: FailedScheduling, ImagePullBackOff, CrashLoopBackOff

kubectl describe node node01
# Full node details: capacity, allocated resources, conditions,
# taints, events, list of all pods running on this node

kubectl describe deployment payment-deployment
# Deployment details: replicas, strategy, conditions, events

kubectl describe service payment-svc -n banking
# Service details: type, ClusterIP, endpoints, events
```

**What to look for in the Events section:**
- `FailedScheduling` → pod cannot be placed on any node
- `ImagePullBackOff` → cannot download the container image
- `CrashLoopBackOff` → container keeps crashing on startup
- `Readiness probe failed` → app is running but not healthy yet
- `OOMKilled` → container exceeded memory limit

---

#### `logs` — READ container output from a pod

**What it does:** Fetches the stdout and stderr output from a
container. This is what your application PRINTS — error messages,
stack traces, startup logs, etc.

```bash
kubectl logs nginx-pod
# Logs from the single container in this pod

kubectl logs nginx-pod -c sidecar
# Logs from a SPECIFIC container when pod has multiple containers
# -c means "container"

kubectl logs nginx-pod --previous
# Logs from the PREVIOUS run of the container (before it crashed)
# Critical for debugging CrashLoopBackOff -- the container is
# restarting, so you cannot see the crash logs from the live one

kubectl logs nginx-pod -f
# FOLLOW mode -- keeps streaming new log lines as they appear
# Like tail -f on Linux. Press Ctrl+C to stop.

kubectl logs nginx-pod --tail=50
# Shows only the LAST 50 lines (not the full log from the start)

kubectl logs nginx-pod --since=1h
# Shows only logs from the LAST 1 HOUR

kubectl logs -l app=payment -n banking
# Logs from ALL pods matching the label app=payment
# Useful when you have multiple replicas and want combined output
```

---

#### `exec` — RUN a command INSIDE a running container

**What it does:** Lets you execute any command inside a container
that is already running. Used to debug applications, check files,
test network connectivity, or verify configuration from inside the
container's environment.

```bash
kubectl exec nginx-pod -- ls /etc
# Runs "ls /etc" inside the container and shows the output
# The "--" separates kubectl's own flags from the command
# you want to run inside the container

kubectl exec nginx-pod -- cat /etc/nginx/nginx.conf
# Prints the nginx config file from inside the container

kubectl exec -it nginx-pod -- bash
# Opens an INTERACTIVE shell inside the container
# -i = keep stdin open (so you can type commands)
# -t = allocate a terminal (so it looks like a normal shell)
# Type "exit" to leave

kubectl exec -it nginx-pod -c sidecar -- sh
# Interactive shell in a SPECIFIC container (sidecar)
# Use sh instead of bash if bash is not installed (alpine images)
```

---

#### `top` — SHOW real-time CPU and memory usage

**What it does:** Shows current resource usage from the metrics-server.
Requires metrics-server to be installed. Different from `describe`
which shows REQUESTED resources -- `top` shows ACTUAL usage right now.

```bash
kubectl top nodes
# CPU and memory usage for each node in the cluster
# Use this to find which nodes are under heavy load

kubectl top pods
# CPU and memory usage for each pod in the default namespace

kubectl top pods -n banking
# Usage for pods in the banking namespace

kubectl top pods --sort-by=cpu
# Sort by CPU usage -- highest consumers at the top

kubectl top pods --sort-by=memory
# Sort by memory usage -- find memory hogs
```

---

#### `events` — SEE what has happened in the cluster recently

**What it does:** Lists all events -- things that happened across
the cluster or in a namespace. Events are automatically generated
when pods are scheduled, images are pulled, containers start,
probes fail, etc. They expire after ~1 hour by default.

```bash
kubectl get events
# All recent events in the default namespace

kubectl get events -n banking
# Events in the banking namespace

kubectl get events --sort-by='.lastTimestamp'
# Sorted by time -- most recent events at the bottom
# This is the most useful form for debugging

kubectl get events --field-selector type=Warning
# Only WARNING events -- filters out normal informational events
# Use this to quickly spot problems

kubectl get events --field-selector involvedObject.name=nginx-pod
# Only events ABOUT a specific pod
```

---

### GROUP 3 — MANAGEMENT VERBS

---

#### `scale` — CHANGE the replica count

**What it does:** Increases or decreases the number of pod replicas
in a Deployment, ReplicaSet, or StatefulSet. Takes effect
immediately -- the controller adds or removes pods to match the
new count.

```bash
kubectl scale deployment payment --replicas=5
# Changes payment Deployment to run 5 pods (up from whatever it was)
# If current replicas is 3, adds 2 more pods

kubectl scale deployment payment --replicas=1
# Scales DOWN to 1 pod
# If pods are mid-request, they get a graceful shutdown signal

kubectl scale deployment payment --replicas=5 -n banking
# Scale in a specific namespace

kubectl scale statefulset postgres --replicas=3
# Scale a StatefulSet (pods are added/removed in order)
```

---

#### `rollout` — MANAGE deployment update rollouts

**What it does:** Controls the process of rolling out changes to
a Deployment, StatefulSet, or DaemonSet. A "rollout" is what
happens when you change the pod template (new image, new env var,
etc.) -- old pods are gradually replaced with new ones.

```bash
kubectl rollout status deployment/payment
# WATCHES the rollout in progress and shows live updates:
# "Waiting for deployment payment rollout to finish:
#  1 out of 3 new replicas have been updated..."
# Blocks until rollout completes OR fails
# Use this in CI/CD pipelines to wait for deployment to finish

kubectl rollout history deployment/payment
# Shows the REVISION HISTORY of this deployment
# Each time you changed the template = a new revision
# Output:
# REVISION  CHANGE-CAUSE
# 1         <none>
# 2         <none>  <- current version

kubectl rollout history deployment/payment --revision=2
# Shows the DETAILS of a specific revision
# What image was used, what changed

kubectl rollout undo deployment/payment
# ROLLBACK to the PREVIOUS revision (one step back)
# Creates a new revision that matches the old one
# Kubernetes does NOT go backwards -- it creates a NEW revision
# with the old configuration

kubectl rollout undo deployment/payment --to-revision=1
# ROLLBACK to a SPECIFIC numbered revision
# Useful when you need to go back more than one step

kubectl rollout pause deployment/payment
# PAUSES the rollout mid-way
# Currently-running pods keep running, but no more old pods
# are replaced with new ones
# Use case: released new version, 2 out of 10 pods updated,
# you want to test those 2 before continuing

kubectl rollout resume deployment/payment
# RESUMES a paused rollout
# Controller continues replacing old pods with new ones from
# wherever it left off

kubectl rollout restart deployment/payment
# RESTARTS all pods in the deployment by triggering a new rollout
# with the SAME template (image and config don't change)
# Use case: you need all pods to restart (e.g., to pick up a
# new ConfigMap value or to clear a memory issue)
# Kubernetes does a rolling restart -- not all at once
```

---

#### `set` — UPDATE specific fields without editing full YAML

**What it does:** Changes specific properties on existing objects
directly from the command line without needing to edit YAML files.
Very useful in CI/CD pipelines for updating image tags.

```bash
kubectl set image deployment/payment payment-app=payment:v1.3
# Changes the container image in the payment Deployment
# "payment-app" is the container name inside the pod template
# "payment:v1.3" is the new image
# This TRIGGERS a rolling update automatically

kubectl set resources deployment/payment \
  -c=payment-app --limits=memory=512Mi
# Changes the memory LIMIT for the payment-app container
# -c means "container" -- specify which container to update

kubectl set env deployment/payment DB_HOST=newhost.example.com
# Adds or updates an environment variable
# Also TRIGGERS a rolling update (pod template changed)
```

---

#### `label` — ADD, UPDATE, or REMOVE labels on objects

**What it does:** Labels are key-value tags used for selecting
and filtering objects. Services use labels to find their pods.
You can add labels to any Kubernetes object.

```bash
kubectl label pod nginx-pod env=production
# Adds label "env=production" to the pod

kubectl label pod nginx-pod env=staging --overwrite
# CHANGES existing label "env" from production to staging
# Without --overwrite, it would fail if label already exists

kubectl label pod nginx-pod env-
# REMOVES the "env" label
# The trailing "-" means DELETE this label

kubectl label node node01 gpu=true
# Labels a NODE -- used with nodeSelector in pod specs
```

---

#### `taint` — ADD or REMOVE taints on nodes

**What it does:** Taints prevent pods from being scheduled on a
node unless the pod has a matching toleration. Used to reserve
nodes for specific workloads or to mark nodes as special.

```bash
kubectl taint node node01 dedicated=banking:NoSchedule
# Adds taint: key=dedicated, value=banking, effect=NoSchedule
# No pod can be placed on node01 UNLESS it has a toleration
# for "dedicated=banking"

kubectl taint node node01 dedicated=banking:NoSchedule-
# REMOVES the taint (trailing "-" means delete)
```

---

#### `cordon` — MARK a node as unschedulable

**What it does:** Stops NEW pods from being scheduled on this node.
Existing pods on the node KEEP RUNNING. Use before node maintenance
when you want to gradually drain it without disrupting running pods.

```bash
kubectl cordon node01
# node01 now shows "SchedulingDisabled" in kubectl get nodes
# No new pods will be placed here
# Pods already running on node01 continue running fine
```

---

#### `uncordon` — ALLOW scheduling on a node again

**What it does:** Reverses `cordon`. The node is marked schedulable
again and can receive new pods.

```bash
kubectl uncordon node01
# node01 goes back to "Ready" status
# Scheduler can now place new pods on this node
```

---

#### `drain` — EVICT all pods from a node AND cordon it

**What it does:** First cordons the node (no new pods), then sends
eviction requests to all existing pods. Pods managed by Deployments
and StatefulSets are rescheduled on other nodes. Use before taking
a node offline for maintenance.

```bash
kubectl drain node01 --ignore-daemonsets --delete-emptydir-data
# --ignore-daemonsets: DaemonSet pods cannot be evicted
#   (they run on every node by design), this flag says "that's ok,
#   skip them, don't fail because of them"
# --delete-emptydir-data: pods using emptyDir volumes will LOSE
#   that data when evicted. This flag says "I accept that"
# Without these flags, drain often fails with a warning
```

---

### GROUP 4 — GENERATION VERBS (Preview and Generate)

---

#### `--dry-run=client -o yaml` — GENERATE YAML without creating anything

**What it does:** The most useful kubectl trick. Generates valid,
complete YAML for any object without actually creating it in the
cluster. Use this as a starting template instead of writing YAML
from scratch.

```bash
kubectl create deployment web \
  --image=nginx:1.25 \
  --replicas=3 \
  --dry-run=client \
  -o yaml
# Prints the full Deployment YAML to your screen
# --dry-run=client: do NOT send to API server
# -o yaml: format output as YAML

# SAVE it to a file for editing
kubectl create deployment web \
  --image=nginx:1.25 \
  --replicas=3 \
  --dry-run=client \
  -o yaml > web-deployment.yaml

# Generate a Pod YAML template
kubectl run nginx-pod \
  --image=nginx:1.25 \
  --dry-run=client \
  -o yaml

# Generate a Service YAML template
kubectl create service clusterip payment-svc \
  --tcp=8080:8080 \
  --dry-run=client \
  -o yaml
```

---

#### `diff` — SHOW what would change before applying

**What it does:** Compares your local YAML file against the current
state in the cluster. Shows exactly what WOULD change if you ran
`kubectl apply`. Like `git diff` but for Kubernetes.

```bash
kubectl diff -f deployment.yaml
# Shows + (additions) and - (removals) between file and cluster
# Run this BEFORE applying in production to verify changes
```

---

## PART 4 — OUTPUT FORMATS (-o FLAG)

The `-o` flag changes how kubectl displays results:

```bash
kubectl get pods
# Default TABLE format -- human readable, least information
# NAME         READY   STATUS    RESTARTS   AGE
# nginx-pod    1/1     Running   0          5m

kubectl get pods -o wide
# TABLE with EXTRA columns -- adds IP, NODE, NOMINATED NODE
# NAME        READY   IP            NODE    NOMINATED NODE
# nginx-pod   1/1     10.244.1.5    node01  <none>

kubectl get pod nginx-pod -o yaml
# FULL YAML of the object including auto-generated fields
# Shows: metadata (with uid, resourceVersion), spec (what you wrote),
# status (what Kubernetes observed -- phase, conditions, podIP)
# Use this to see EVERYTHING about an object

kubectl get pod nginx-pod -o json
# Same as yaml but in JSON format
# Useful when piping to tools like jq

kubectl get pods -o name
# Just the resource/name -- minimal output
# Useful in shell scripts
# pod/nginx-pod
# pod/payment-pod
```

---

## PART 5 — IMPORTANT FLAGS

### Namespace Flags

```bash
-n banking
--namespace=banking
# Both specify which namespace to look in
# Without -n, kubectl uses your current context's default namespace
# (usually "default" unless you configured otherwise)

--all-namespaces
-A
# Both mean: look in EVERY namespace
# Useful when you don't know which namespace something is in:
kubectl get pods -A | grep nginx
```

### Label Selector Flag

```bash
-l app=payment
# Filter objects that have this exact label
# Only shows objects where labels contain app=payment

-l app=payment,env=production
# Multiple labels -- AND condition
# Objects must have BOTH labels

kubectl get pods -l app=payment -n banking
# All pods in banking namespace with label app=payment
```

### Watch Flag

```bash
-w
# Keeps the command running and UPDATES the output as things change
# Press Ctrl+C to stop watching

kubectl get pods -w
# Stays open -- when a pod's status changes, the table updates
# Great for watching a deployment rollout in real time
```

### Force Delete (When Pod is Stuck in Terminating)

```bash
--grace-period=0 --force
# Skips the graceful shutdown period and force-removes the pod
# Use ONLY when a pod is stuck in "Terminating" state for a long time
# Normal case: pod gets SIGTERM, has 30 seconds to clean up, then dies
# Force case: skip the 30 seconds, remove immediately from etcd

kubectl delete pod stuck-pod --grace-period=0 --force
```

### Cascade Flag (Delete Without Removing Children)

```bash
--cascade=orphan
# Deletes the object but LEAVES its child objects running
# Example: delete a Deployment but keep its pods running

kubectl delete deployment web --cascade=orphan
# Deployment is gone, ReplicaSet and pods still exist
# Pods are now "orphaned" -- no controller manages them
```

---

## PART 6 — CONTEXT AND CONFIG (Multiple Clusters)

In production you have multiple clusters (dev, staging, prod).
kubectl uses `~/.kube/config` to know which cluster to talk to.

```bash
kubectl config current-context
# Shows which cluster you are CURRENTLY talking to
# ALWAYS run this before making changes to confirm you are
# on the right cluster. Running a delete command on production
# when you think you're on dev is a real incident.

kubectl config get-contexts
# Shows ALL clusters configured in your kubeconfig
# CURRENT column shows which one is active

kubectl config use-context production-cluster
# SWITCHES to a different cluster
# All kubectl commands after this target the production cluster

kubectl config view
# Shows the full kubeconfig file content
# Contains: cluster addresses, certificates, user credentials

kubectl config set-context --current --namespace=banking
# Sets the DEFAULT namespace for your current context
# After this, you don't need -n banking in every command
# kubectl get pods  now automatically means banking namespace
```

---

## PART 7 — USEFUL COMMANDS FOR DAILY WORK

```bash
# Check what permissions YOU have in the cluster
kubectl auth can-i create pods
# yes or no

kubectl auth can-i delete deployments -n banking
# yes or no

kubectl auth can-i --list -n banking
# Lists ALL actions you are allowed to do in banking namespace

# Check permissions of a SERVICE ACCOUNT (not yourself)
kubectl auth can-i get pods \
  --as=system:serviceaccount:banking:payment-sa -n banking

# Port-forward -- access a pod or service from your laptop
# without creating an external Service
kubectl port-forward pod/nginx-pod 8080:80
# localhost:8080 on your laptop now routes to port 80 in the pod
# Press Ctrl+C to stop

kubectl port-forward service/payment-svc 8080:8080
# Same but through a Service (reaches any healthy pod behind it)

# Copy files between your laptop and a pod
kubectl cp nginx-pod:/etc/nginx/nginx.conf ./nginx.conf
# FROM pod TO your laptop

kubectl cp ./config.yaml nginx-pod:/app/config.yaml
# FROM your laptop TO pod

# Get all resources in a namespace at once
kubectl get all -n banking
# Shows: pods, services, deployments, replicasets, statefulsets etc.

# See ALL resource types Kubernetes knows about (including CRDs)
kubectl api-resources
# Shows: NAME, SHORTNAMES, APIVERSION, NAMESPACED, KIND
```

---

## PART 8 — SHORTNAMES REFERENCE

```bash
# Instead of typing the full name, use shortnames:

pods                     po
services                 svc
deployments              deploy
replicasets              rs
statefulsets             sts
daemonsets               ds
configmaps               cm
namespaces               ns
nodes                    no
persistentvolumes        pv
persistentvolumeclaims   pvc
serviceaccounts          sa
ingresses                ing
cronjobs                 cj
horizontalpodautoscalers hpa
networkpolicies          netpol

# Examples:
kubectl get po          # same as kubectl get pods
kubectl get svc         # same as kubectl get services
kubectl get deploy      # same as kubectl get deployments
kubectl get ns          # same as kubectl get namespaces
```

---

## PART 9 — HANDS-ON LABS

---

## Lab 1 — Generate YAML Without Creating (Most Useful Trick)

```bash
# Generate Deployment YAML and just print it
kubectl create deployment web \
  --image=nginx:1.25 \
  --replicas=3 \
  --dry-run=client \
  -o yaml
# Study the output -- this is what a complete Deployment looks like

# Save it to a file for editing
kubectl create deployment web \
  --image=nginx:1.25 \
  --replicas=3 \
  --dry-run=client \
  -o yaml > web-deployment.yaml

cat web-deployment.yaml

# Edit the file (add labels, resources, etc.) then apply
kubectl apply -f web-deployment.yaml

# Generate a Pod YAML template
kubectl run nginx \
  --image=nginx:1.25 \
  --dry-run=client \
  -o yaml

# Generate a ClusterIP Service YAML template
kubectl create service clusterip my-svc \
  --tcp=8080:8080 \
  --dry-run=client \
  -o yaml

kubectl delete deployment web
```

---

## Lab 2 — apply vs create: See the Difference

```bash
# CREATE -- fails if already exists
kubectl create deployment test-deploy --image=nginx
kubectl create deployment test-deploy --image=nginx
# Error: deployments.apps "test-deploy" already exists

# APPLY -- works both times
kubectl apply -f - << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: apply-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: apply-demo
  template:
    metadata:
      labels:
        app: apply-demo
    spec:
      containers:
      - name: nginx
        image: nginx:1.24
EOF

# Apply AGAIN with different image -- updates, does not fail
kubectl apply -f - << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: apply-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: apply-demo
  template:
    metadata:
      labels:
        app: apply-demo
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
EOF

# Verify image was updated
kubectl get deployment apply-demo -o yaml | grep image

kubectl delete deployment test-deploy
kubectl delete deployment apply-demo
```

---

## Lab 3 — Output Formats

```bash
kubectl run inspect-pod --image=nginx:1.25

# Wait for Running
kubectl get pod inspect-pod -w
# Ctrl+C once Running

# Default table
kubectl get pod inspect-pod

# Wide table with IP and Node
kubectl get pod inspect-pod -o wide

# Full YAML -- see everything including status
kubectl get pod inspect-pod -o yaml

# Find the pod IP from YAML
kubectl get pod inspect-pod -o yaml | grep podIP

# Find which node it is on
kubectl get pod inspect-pod -o yaml | grep nodeName

# Find the container image
kubectl get pod inspect-pod -o yaml | grep image

kubectl delete pod inspect-pod
```

---

## Lab 4 — rollout Commands in Action

```bash
kubectl create deployment rollout-demo --image=nginx:1.24 --replicas=3
kubectl rollout status deployment/rollout-demo
# Waits until all 3 pods are Running

# Check initial revision history
kubectl rollout history deployment/rollout-demo
# REVISION 1 exists

# Update image -- triggers a new rollout
kubectl set image deployment/rollout-demo nginx=nginx:1.25

# Watch rollout progress
kubectl rollout status deployment/rollout-demo
# "Waiting for deployment rollout to finish: 1 out of 3 new
#  replicas have been updated..."

# See revision history now
kubectl rollout history deployment/rollout-demo
# REVISION 1 (old) and REVISION 2 (current with nginx:1.25)

# ROLLBACK to previous version
kubectl rollout undo deployment/rollout-demo

# Confirm it went back
kubectl rollout history deployment/rollout-demo
# Now shows REVISION 3 (which matches old REVISION 1)

# RESTART all pods without changing image
kubectl rollout restart deployment/rollout-demo
# All pods get replaced one by one -- same image, fresh start

kubectl delete deployment rollout-demo
```

---

## Lab 5 — Cordon, Drain, Uncordon

```bash
kubectl get nodes

# Cordon -- prevent new pods going to node01
kubectl cordon node01
kubectl get nodes
# node01 shows SchedulingDisabled

# Create pods -- they go to OTHER nodes only
kubectl create deployment cordoned-test --image=nginx --replicas=3
kubectl get pods -o wide
# All pods land on controlplane, NOT node01

# Drain -- evict existing pods AND cordon
kubectl drain node01 --ignore-daemonsets --delete-emptydir-data

# Uncordon -- allow node01 to receive pods again
kubectl uncordon node01
kubectl get nodes
# node01 shows Ready

kubectl delete deployment cordoned-test
```

---

## Lab 6 — Port Forward

```bash
kubectl run web-test --image=nginx:1.25
kubectl get pod web-test -w
# Ctrl+C once Running

# Forward localhost:8080 to port 80 in the pod
kubectl port-forward pod/web-test 8080:80 &

# Test -- should get nginx welcome page
curl localhost:8080

# Stop the port-forward
kill %1

kubectl delete pod web-test
```

---

## PART 10 — QUICK REFERENCE CHART

```
VERB          WHAT IT DOES                     EXAMPLE
─────────────────────────────────────────────────────────────────────────
get           Read objects, show as table       kubectl get pods -n banking
describe      Full details + Events section     kubectl describe pod <name>
logs          Container output/errors           kubectl logs <pod> -f
exec          Shell into container              kubectl exec -it <pod> -- bash
top           Live CPU/memory usage             kubectl top pods
events        Recent cluster events             kubectl get events --sort-by='.lastTimestamp'

apply         Create or update from file        kubectl apply -f file.yaml
create        Create only (fails if exists)     kubectl create deployment web --image=nginx
delete        Remove object                     kubectl delete pod <name>
edit          Open YAML in editor               kubectl edit deployment <name>
patch         Update specific field             kubectl patch deploy web -p '{"spec":{"replicas":5}}'

scale         Change replica count              kubectl scale deployment web --replicas=5
rollout       Manage rolling updates            kubectl rollout status/history/undo/restart
set image     Change container image            kubectl set image deploy/web nginx=nginx:1.25
label         Add/remove labels                 kubectl label pod <name> env=prod
taint         Add/remove node taints            kubectl taint node <name> key=val:NoSchedule

cordon        Stop new pods on node             kubectl cordon node01
uncordon      Allow pods on node again          kubectl uncordon node01
drain         Evict all pods + cordon           kubectl drain node01 --ignore-daemonsets

auth can-i    Check your permissions            kubectl auth can-i delete pods -n banking
port-forward  Access pod from laptop            kubectl port-forward pod/<name> 8080:80
diff          Preview changes before apply      kubectl diff -f file.yaml

dry-run       Generate YAML without creating    kubectl create deploy web --image=nginx --dry-run=client -o yaml

─────────────────────────────────────────────────────────────────────────
FLAGS
─────────────────────────────────────────────────────────────────────────
-n <ns>             Specify namespace
-A                  All namespaces
-l app=payment      Filter by label
-w                  Watch for changes (live)
-f                  Follow logs (stream)
-o wide             Extra columns (IP, node)
-o yaml             Full YAML output
-o json             Full JSON output
-o name             Just the names
--dry-run=client    Preview without creating
--force --grace-period=0    Force delete stuck pods
--cascade=orphan    Delete without removing children
--previous          Previous container logs (after crash)
--tail=50           Last 50 lines of logs
--since=1h          Logs from last 1 hour

─────────────────────────────────────────────────────────────────────────
SHORTNAMES
─────────────────────────────────────────────────────────────────────────
po  svc  deploy  rs  sts  ds  cm  ns  no  pv  pvc  sa  ing  cj  hpa
```

---

## PART 11 — INTERVIEW QUESTIONS AND ANSWERS

---

**Q1. What is kubectl and how does it communicate with Kubernetes?**

kubectl is the command-line client for Kubernetes. It reads the
`~/.kube/config` file which contains the API server address, your
client certificate, and cluster information. Every kubectl command
is converted into an HTTPS request to the API server on port 6443.
kubectl itself does not run any Kubernetes logic -- it is just a
client that talks to the API server, similar to how a web browser
talks to a web server. The API server then does authentication,
authorization, and returns a JSON response which kubectl formats
into the output you see.

---

**Q2. What is the difference between `kubectl apply` and `kubectl create`?**

`kubectl create` creates a NEW object and FAILS with "already exists"
if the object is already present in the cluster. It is a one-time
operation.

`kubectl apply` is smarter -- if the object does NOT exist, it
creates it. If it DOES exist, it updates only the fields that
changed. It never fails with "already exists." In production, we
always use `apply` because it is idempotent -- you can run it 10
times and the result is always the same. This is essential for CI/CD
pipelines where the same command runs on every deployment.

---

**Q3. What is `--dry-run=client -o yaml` and when do you use it?**

`--dry-run=client` tells kubectl to go through all the local
validation steps but NOT send the request to the API server --
nothing is created in the cluster. Combined with `-o yaml`, it
generates a complete, valid YAML template for any object and prints
it to the screen.

I use this constantly. Instead of writing a Deployment YAML from
scratch (and potentially making syntax errors), I run:

```bash
kubectl create deployment web --image=nginx:1.25 --dry-run=client -o yaml > web.yaml
```

This gives me a valid starting template in seconds. Then I edit it
to add resources, labels, probes, etc. and apply it. Every senior
engineer uses this trick -- it is much faster than memorizing every
YAML field from scratch.

---

**Q4. What is the difference between `kubectl logs` and `kubectl logs --previous`?**

`kubectl logs <pod>` shows the logs from the CURRENTLY RUNNING
container instance. If the container is alive, you see its live
output.

`kubectl logs <pod> --previous` shows the logs from the PREVIOUS
container run -- the one that crashed and was replaced. This is
critical for debugging CrashLoopBackOff. When a container crashes,
a new one starts immediately. By the time you run `kubectl logs`,
you are looking at the NEW container which may have just started
and shows no errors yet. `--previous` lets you see the logs of the
CRASHED container to find out WHY it crashed.

---

**Q5. A pod is stuck in Pending state. Walk through your kubectl debugging steps.**

```bash
# Step 1 -- describe the pod to see why it cannot be scheduled
kubectl describe pod <pod-name> -n <namespace>
# Read the Events section at the bottom
# Common messages:
# "0/3 nodes available: insufficient cpu" -> need more resources
# "0/3 nodes have taint that pod didn't tolerate" -> taint issue
# "pod has unbound PersistentVolumeClaims" -> storage issue

# Step 2 -- check node capacity
kubectl describe nodes | grep -A 5 "Allocated resources"
# See if nodes have capacity for this pod's requests

# Step 3 -- check node taints
kubectl describe nodes | grep Taints

# Step 4 -- check resource quota in the namespace
kubectl get resourcequota -n <namespace>
# Quota exceeded would prevent pod creation
```

---

**Q6. What does `kubectl rollout restart deployment/<name>` do and when would you use it?**

`kubectl rollout restart` triggers a rolling restart of all pods in
the Deployment without changing any configuration. It adds a
`kubectl.kubernetes.io/restartedAt` annotation to the pod template,
which makes the template "different" from the controller's
perspective -- triggering a new rollout where all pods are replaced
one by one, same as a normal rolling update.

Use cases:
- You updated a ConfigMap or Secret and want all pods to pick up
  the new values (environment variable injected at pod start time
  does not update in-place)
- All pods in a Deployment are experiencing a memory issue and you
  want a clean restart without changing the image
- You want to redistribute pods across nodes after a node was
  drained and uncordoned

It is much safer than `kubectl delete pods --all` which would delete
ALL pods simultaneously causing a brief outage. `rollout restart`
replaces them one at a time maintaining availability.

---

**Q7. What is the difference between `kubectl cordon` and `kubectl drain`?**

`kubectl cordon node01` marks the node as unschedulable -- no NEW
pods will be placed there, but EXISTING pods on the node keep
running untouched.

`kubectl drain node01` does two things: first it cordons the node
(same as above), then it EVICTS all existing pods from the node.
Pods managed by Deployments and StatefulSets are rescheduled on
other nodes. The drain command waits for all pods to be evicted
before completing.

Use cordon when: you want to prevent new pods from landing on a
node but you do not want to disrupt currently-running workloads yet.

Use drain when: you are about to take the node completely offline
for maintenance (OS upgrade, hardware replacement) and you want all
workloads safely moved to other nodes first.

```bash
# Typical maintenance sequence:
kubectl cordon node01            # step 1: stop new pods
kubectl drain node01 \          # step 2: move existing pods
  --ignore-daemonsets \
  --delete-emptydir-data
# ... do maintenance on node01 ...
kubectl uncordon node01          # step 3: return node to service
```

---

**Q8. SCENARIO: You updated a Deployment's image but the rollout is stuck halfway. Some pods have the new image, some still have the old. How do you investigate and fix it?**

```bash
# Step 1 -- check rollout status
kubectl rollout status deployment/<name>
# Shows "Waiting for deployment... 2 out of 5 new replicas updated"
# and it is not progressing

# Step 2 -- find the new pods and check why they are not Ready
kubectl get pods -l app=<name>
# Look for pods in CrashLoopBackOff, Error, or Running/0/1 Ready

# Step 3 -- describe a failing new pod
kubectl describe pod <new-pod-name>
# Check Events for ImagePullBackOff, readiness probe failed, etc.

# Step 4 -- check logs of the failing pod
kubectl logs <new-pod-name>
kubectl logs <new-pod-name> --previous

# Step 5 -- if the new image is broken, rollback immediately
kubectl rollout undo deployment/<name>
kubectl rollout status deployment/<name>
# Old version restored via rolling rollback
```

The rollout is stuck because `maxUnavailable: 0` (default) prevents
Kubernetes from killing any old pod until a new one is Ready.
Since new pods are failing their readiness probe, Kubernetes
protects you by NOT proceeding -- you still have all old pods
serving traffic. This is correct protective behaviour.

---

**Q9. What does `kubectl auth can-i` do and when is it useful?**

`kubectl auth can-i` checks whether the current user (or a
specified service account) has permission to perform a specific
action in Kubernetes. It queries the RBAC system and returns
"yes" or "no."

Use cases:
- Debugging "Forbidden" errors: when a pod or pipeline gets a 403
  error, run auth can-i to confirm what the relevant service account
  is and is not allowed to do
- RBAC audit: before deploying an application, verify its service
  account has exactly the permissions it needs (no more, no less)
- Troubleshooting CI/CD: when a Jenkins/GitHub Actions pipeline
  cannot create pods or update deployments, check if the pipeline's
  service account has the required permissions

```bash
kubectl auth can-i create pods
# yes (you personally can create pods in current namespace)

kubectl auth can-i delete deployments -n banking
# no (you don't have delete permission in banking namespace)

kubectl auth can-i --list -n banking
# Lists every action you CAN do in banking namespace

kubectl auth can-i get pods \
  --as=system:serviceaccount:banking:payment-sa
# Checks what the payment service account is allowed to do
```

---

**Q10. What is `kubectl port-forward` and how is it different from a Kubernetes Service?**

`kubectl port-forward` creates a temporary network tunnel from
your local laptop's port to a pod's port, going through the API
server. It is for local development and debugging ONLY -- not for
production traffic routing.

A Kubernetes Service is a permanent, cluster-level networking object
that provides a stable virtual IP (ClusterIP) and can distribute
traffic to multiple pod replicas with load balancing. It works
independently of kubectl and persists after you close your terminal.

Port-forward: temporary, single pod, no load balancing, dies when
you press Ctrl+C, uses API server as a proxy.

Service: permanent, multiple pods, load balanced, independent of
kubectl, managed by kube-proxy on every node.

When to use port-forward: quickly testing a pod's HTTP endpoint
without setting up a Service, debugging a specific pod directly,
accessing admin dashboards (Kubernetes dashboard, Prometheus) that
should not be publicly exposed.

---

*Next: Topic 2 -- Namespaces*
