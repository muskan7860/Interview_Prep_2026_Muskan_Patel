# Kubernetes — Imperative Commands, dry-run, YAML Generation & Live Editing
## Create · Delete · Edit · Generate YAML — Without Writing a Single File
> Written for: Someone with 4 years of DevOps experience preparing for senior-level interviews
> Style: First-standard student explanation → deep technical truth → hands-on lab with line-by-line explanation

---

## 🧠 SECTION 1 — WHAT IS THIS TOPIC ABOUT? (Story First)

### Forget Kubernetes for 2 minutes

Imagine you are a chef in a restaurant.

You have two ways to cook a dish:

**Way 1 — Declarative (Recipe card method):**
You write the full recipe on a card:
```
Recipe: Pasta
- 200g pasta
- 3 cloves garlic
- olive oil
- salt
Cook for 10 minutes. Serve hot.
```
You hand the card to the kitchen. The kitchen reads the card and makes it EXACTLY as written. Next time — same card, same dish.

**Way 2 — Imperative (Shouting at the kitchen):**
You stand at the kitchen door and shout:
```
"Make pasta! Use garlic! Cook it now!"
```
The kitchen makes it immediately. Faster. But no written record.

**In Kubernetes:**
```
DECLARATIVE = kubectl apply -f deployment.yaml
  → You write a YAML file
  → kubectl sends it to API Server
  → Object created as defined in file
  → File = permanent record, can be committed to Git

IMPERATIVE = kubectl create deployment nginx --image=nginx
  → No YAML file needed
  → kubectl generates the request internally
  → Sends directly to API Server
  → Faster but no permanent file record
```

### Why You Need BOTH

```
IMPERATIVE is good for:
  ✅ Quick tasks in interviews / exams (CKA)
  ✅ One-off tasks (create a test pod quickly)
  ✅ Generating YAML base templates
  ✅ Debugging and troubleshooting fast

DECLARATIVE is good for:
  ✅ Production environments
  ✅ Git version control
  ✅ CI/CD pipelines
  ✅ Repeatable, auditable changes
```

### The MAGIC Combination — Best of Both Worlds

```
kubectl create deployment nginx --image=nginx --dry-run=client -o yaml

This command:
  → Uses imperative speed (no file needed to start)
  → Generates perfect YAML (for declarative use later)
  → Does NOT create anything (--dry-run=client)
  → Gives you a base template to edit and save
```

---

## 🔑 SECTION 2 — UNDERSTANDING --dry-run=client

### What Does --dry-run=client Mean?

`--dry-run=client` tells kubectl:

> "Generate the request object locally on MY machine. Show me what you WOULD send to the API Server — but DON'T actually send it."

```
WITHOUT --dry-run=client:

kubectl create deployment nginx --image=nginx
        │
        ▼
kubectl builds the Deployment object
        │
        ▼ SENDS to API Server
        │
        ▼
API Server → etcd → object CREATED ✅ (real creation)


WITH --dry-run=client:

kubectl create deployment nginx --image=nginx --dry-run=client
        │
        ▼
kubectl builds the Deployment object locally
        │
        ✖ STOPS HERE — does NOT send to API Server
        │
        ▼
Prints: "deployment.apps/nginx created (dry run)"
        ↑ Note: says "dry run" — nothing actually happened
```

### What Does -o yaml Mean?

`-o` means **output format**. By default kubectl prints a brief confirmation message. `-o yaml` tells kubectl to print the FULL YAML of the object instead.

```
WITHOUT -o yaml:
kubectl create deployment nginx --image=nginx --dry-run=client
Output: deployment.apps/nginx created (dry run)
        ← just a one-liner confirmation

WITH -o yaml:
kubectl create deployment nginx --image=nginx --dry-run=client -o yaml
Output: (full YAML printed to terminal)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  ...
spec:
  replicas: 1
  ...
```

### The Two Together = YAML Generator

```
--dry-run=client = "don't actually create"
-o yaml          = "show me the full YAML"

TOGETHER = "Show me the complete YAML for this object
            without creating anything"
```

This is the **most powerful trick** in Kubernetes for fast working. Instead of writing YAML from scratch — which takes time and has indentation mistakes — you generate it in 5 seconds and edit only what you need.

---

## 🔄 SECTION 3 — --dry-run=client vs --dry-run=server

There are TWO types of dry-run. Most people only know one. Knowing both makes you look senior.

```
--dry-run=client:
  → Simulation happens LOCALLY on your machine
  → Does NOT contact the API Server at all
  → Does NOT run admission webhooks
  → Does NOT check quota
  → Does NOT validate against actual cluster policies
  → Fast, works even if cluster is unreachable
  → Use for: YAML generation

--dry-run=server:
  → Request IS sent to the API Server
  → Runs through FULL pipeline:
    authentication → RBAC → mutating webhooks → validation → validating webhooks
  → But stops BEFORE writing to etcd
  → Checks real quota, real policies, real webhooks
  → Use for: validating your YAML will actually be accepted
  → Slower but MUCH more accurate
```

**Interview question answer:** *"What is the difference between `--dry-run=client` and `--dry-run=server`?"*

> "Client runs locally, never touches API Server — good for YAML generation. Server sends to API Server and runs the full admission pipeline but skips the final etcd write — good for validation. If I want to check my YAML won't be rejected by OPA Gatekeeper or exceed quota before deploying to production, I use `--dry-run=server`."

---

## 📦 SECTION 4 — CREATING OBJECTS WITHOUT YAML (Imperative Commands)

### 4.1 — Creating a Deployment

```bash
# Basic deployment
kubectl create deployment nginx --image=nginx

# With specific replicas
kubectl create deployment nginx --image=nginx --replicas=3

# With specific port
kubectl create deployment nginx --image=nginx --port=80

# With replicas AND port
kubectl create deployment web-app --image=nginx --replicas=3 --port=80
```

**Explaining flags:**
- `create deployment` → create a Deployment object
- `nginx` → name of the deployment (comes right after the object type)
- `--image=nginx` → which container image to use
- `--replicas=3` → how many pod copies to run
- `--port=80` → which port the container exposes (informational — doesn't actually open firewall)

### 4.2 — Creating a Pod

```bash
# Basic pod
kubectl run nginx --image=nginx

# Pod with specific label
kubectl run nginx --image=nginx --labels="app=web,env=prod"

# Pod with environment variable
kubectl run nginx --image=nginx --env="DB_HOST=mysql" --env="DB_PORT=3306"

# Pod with specific port
kubectl run nginx --image=nginx --port=80

# Temporary pod for debugging (deletes itself when you exit)
kubectl run debug --image=busybox --rm -it -- sh
```

**Explaining flags:**
- `run` → creates a Pod (not a Deployment — just a standalone pod)
- `--labels` → add labels to the pod metadata. Format: `key=value,key2=value2`
- `--env` → set environment variables inside the container
- `--rm` → remove (delete) the pod automatically after you exit
- `-it` → interactive terminal
- `-- sh` → the command to run inside the container

### 4.3 — Creating a Namespace

```bash
kubectl create namespace banking
kubectl create namespace monitoring
kubectl create namespace dev
```

### 4.4 — Creating a Service (Expose)

```bash
# Expose a deployment as ClusterIP (default)
kubectl expose deployment web-app --port=80 --target-port=80

# Expose as NodePort
kubectl expose deployment web-app --port=80 --target-port=80 --type=NodePort

# Expose as LoadBalancer
kubectl expose deployment web-app --port=80 --target-port=80 --type=LoadBalancer

# Give the service a specific name
kubectl expose deployment web-app --port=80 --name=web-service

# Expose with different external and internal ports
kubectl expose deployment web-app --port=8080 --target-port=80
```

**Explaining flags:**
- `expose deployment web-app` → create a Service targeting this Deployment's pods
- `--port=80` → the port the SERVICE listens on
- `--target-port=80` → the port INSIDE the container to forward to
- `--type=NodePort` → Service type
- `--name=web-service` → name of the Service (default is same as deployment name)

### 4.5 — Creating a ConfigMap

```bash
# From literal key-value pairs
kubectl create configmap app-config --from-literal=color=blue --from-literal=size=large

# From a file
kubectl create configmap nginx-config --from-file=nginx.conf

# From multiple files in a directory
kubectl create configmap all-configs --from-file=./configs/

# From an env file (.env format)
kubectl create configmap env-config --from-env-file=app.env
```

**Explaining flags:**
- `create configmap app-config` → create a ConfigMap named `app-config`
- `--from-literal=color=blue` → store key `color` with value `blue` directly without any file
- `--from-file=nginx.conf` → read the file `nginx.conf` and store its CONTENTS as the value. Key = filename.
- `--from-env-file=app.env` → read a file in `KEY=value` format and create one ConfigMap key per line

### 4.6 — Creating a Secret

```bash
# Generic secret from literals
kubectl create secret generic db-secret \
  --from-literal=username=admin \
  --from-literal=password=MyPassword123

# TLS secret from cert files
kubectl create secret tls bank-tls \
  --cert=server.crt \
  --key=server.key

# Docker registry secret (for pulling private images)
kubectl create secret docker-registry ecr-creds \
  --docker-server=123456789.dkr.ecr.us-east-1.amazonaws.com \
  --docker-username=AWS \
  --docker-password=$(aws ecr get-login-password)
```

**Explaining:**
- `create secret generic` → generic secret (key-value pairs)
- `create secret tls` → TLS secret type (special format for certs)
- `create secret docker-registry` → for image pull authentication
- Values are automatically base64 encoded by kubectl

### 4.7 — Creating a Job

```bash
# Basic job
kubectl create job backup-job --image=busybox -- sh -c "echo backup done"

# Job from an existing CronJob (manually trigger)
kubectl create job --from=cronjob/backup-cronjob manual-backup-now
```

**Explaining:**
- `create job backup-job` → create a Job named `backup-job`
- `--image=busybox` → container image
- `-- sh -c "echo backup done"` → the command to run (everything after `--` is the container command)
- `--from=cronjob/backup-cronjob` → create a one-time Job from an existing CronJob template

### 4.8 — Creating a CronJob

```bash
kubectl create cronjob daily-backup \
  --image=busybox \
  --schedule="0 2 * * *" \
  -- sh -c "echo running backup"
```

**Explaining:**
- `create cronjob daily-backup` → create CronJob named `daily-backup`
- `--schedule="0 2 * * *"` → cron format: at 2:00 AM every day
- `-- sh -c "..."` → the command to run in each job execution

### 4.9 — Creating a ServiceAccount

```bash
kubectl create serviceaccount monitoring-sa -n monitoring
```

### 4.10 — Creating a Role and RoleBinding (RBAC)

```bash
# Create a Role
kubectl create role pod-reader \
  --verb=get,list,watch \
  --resource=pods \
  -n banking

# Create a RoleBinding
kubectl create rolebinding bind-pod-reader \
  --role=pod-reader \
  --user=muskan \
  -n banking

# Create ClusterRole
kubectl create clusterrole node-reader \
  --verb=get,list,watch \
  --resource=nodes

# Create ClusterRoleBinding
kubectl create clusterrolebinding bind-node-reader \
  --clusterrole=node-reader \
  --user=muskan
```

---

## 🗑️ SECTION 5 — DELETING OBJECTS

### Delete by Name

```bash
# Delete a specific pod
kubectl delete pod nginx-pod

# Delete a deployment
kubectl delete deployment web-app

# Delete a service
kubectl delete service web-svc

# Delete in a specific namespace
kubectl delete deployment web-app -n banking
```

### Delete by Label (Powerful)

```bash
# Delete ALL pods with a specific label
kubectl delete pods -l app=nginx

# Delete all pods in a namespace with label
kubectl delete pods -l env=test -n banking
```

**Explaining:**
- `-l app=nginx` → label selector: find and delete all resources with this label

### Delete Multiple Objects at Once

```bash
# Delete pod AND service together
kubectl delete pod nginx-pod service nginx-svc

# Delete all pods in a namespace (careful!)
kubectl delete pods --all -n test-namespace

# Delete everything in a namespace
kubectl delete all --all -n test-namespace
```

**Explaining:**
- `--all` → select all resources of that type
- `all` as resource type → covers pods, services, deployments, replicasets, statefulsets, daemonsets, jobs

### Delete by YAML File

```bash
# Delete exactly what is defined in the file
kubectl delete -f deployment.yaml

# Delete multiple files at once
kubectl delete -f deployment.yaml -f service.yaml

# Delete everything in a folder
kubectl delete -f ./manifests/
```

### Force Delete (Emergency Use Only)

```bash
# Force delete a stuck pod
kubectl delete pod stuck-pod --force --grace-period=0
```

**Explaining:**
- `--force` → don't wait, delete immediately
- `--grace-period=0` → skip the graceful termination period (normally 30 seconds)
- **Use with caution** — skips cleanup, use only for truly stuck pods

---

## ✏️ SECTION 6 — EDITING LIVE OBJECTS WITHOUT YAML FILES

### THIS IS THE CRITICAL SECTION

> **The interview question you mentioned:** *"You deleted your YAML file. The object still exists in Kubernetes. How do you change something like replicas?"*

The answer: **you don't need the YAML file. The object is in etcd. You can edit it directly.**

### Method 1 — kubectl edit (Live Editor)

```bash
kubectl edit deployment web-app
```

**What this does:**
1. kubectl fetches the current YAML of that object from the API Server
2. Opens it in your default text editor (usually `vi` or `vim`)
3. You make changes inside the editor
4. When you save and quit → kubectl sends the updated YAML back to the API Server
5. Changes are applied IMMEDIATELY

```bash
# Edit in a specific namespace
kubectl edit deployment web-app -n banking

# Edit a service
kubectl edit service web-svc

# Edit a configmap
kubectl edit configmap app-config

# Edit a pod (limited — most pod spec fields are immutable after creation)
kubectl edit pod nginx-pod

# Use a different editor (nano instead of vi)
KUBE_EDITOR="nano" kubectl edit deployment web-app
```

**How to change replicas using kubectl edit:**
```bash
kubectl edit deployment web-app
# Editor opens. Find this section:
spec:
  replicas: 1    ← change this number
# Change to:
spec:
  replicas: 3
# Save and quit (:wq in vi)
# Output: deployment.apps/web-app edited
```

**Important about kubectl edit:**
```
✅ Works even if you deleted the YAML file
✅ Shows you the COMPLETE current state including all auto-generated fields
✅ Changes applied immediately on save
❌ NOT tracked in Git (no file changes)
❌ If you edit something immutable (like a pod's container image), it will refuse
❌ Vi/vim knowledge needed (or set KUBE_EDITOR to something else)
```

### Method 2 — kubectl scale (Change Replicas Fast)

The fastest way to change replica count — no editor needed:

```bash
# Scale a deployment
kubectl scale deployment web-app --replicas=5

# Scale in a specific namespace
kubectl scale deployment web-app --replicas=0 -n banking

# Scale a StatefulSet
kubectl scale statefulset postgres --replicas=3

# Scale multiple deployments at once
kubectl scale deployment web-app payment-app --replicas=3

# Scale based on current count (only scale if currently at 3)
kubectl scale deployment web-app --replicas=5 --current-replicas=3
```

**Explaining:**
- `scale deployment web-app` → target this specific deployment
- `--replicas=5` → set replica count to 5
- `--current-replicas=3` → safety check: only scale IF current replicas = 3. Prevents accidental scaling.

### Method 3 — kubectl patch (Change Specific Fields)

Patch lets you change one specific field without opening an editor. Like a surgical operation — you touch only what you need to change.

```bash
# Change replicas
kubectl patch deployment web-app -p '{"spec":{"replicas":5}}'

# Change image
kubectl patch deployment web-app -p \
  '{"spec":{"template":{"spec":{"containers":[{"name":"nginx","image":"nginx:1.25"}]}}}}'

# Change a label
kubectl patch deployment web-app -p \
  '{"metadata":{"labels":{"version":"v2"}}}'

# Patch in a specific namespace
kubectl patch deployment web-app -p '{"spec":{"replicas":3}}' -n banking
```

**Explaining:**
- `patch deployment web-app` → target this deployment
- `-p '{"spec":{"replicas":5}}'` → `-p` means patch. The value is a JSON object describing ONLY the fields you want to change. You don't need to provide the full YAML — just the specific field to update.
- The JSON path mirrors the YAML structure. `spec.replicas` in YAML = `{"spec":{"replicas":5}}` in JSON.

### Method 4 — kubectl set (Change Image, Resources, Env)

```bash
# Change container image
kubectl set image deployment/web-app nginx=nginx:1.25

# Change image in specific namespace
kubectl set image deployment/web-app nginx=nginx:1.25 -n banking

# Change environment variable
kubectl set env deployment/web-app DB_HOST=new-database-host

# Remove an environment variable (use -)
kubectl set env deployment/web-app DB_OLD_VAR-

# Change resource requests and limits
kubectl set resources deployment web-app \
  --requests=cpu=200m,memory=256Mi \
  --limits=cpu=500m,memory=512Mi

# Change resource on specific container (if pod has multiple)
kubectl set resources deployment web-app \
  -c=nginx \
  --limits=cpu=200m,memory=512Mi
```

**Explaining `kubectl set image`:**
- `set image deployment/web-app` → target this deployment
- `nginx=nginx:1.25` → format is `container-name=new-image:tag`
  - `nginx` before `=` is the CONTAINER NAME (as defined in the deployment spec)
  - `nginx:1.25` after `=` is the new image
- This triggers a rolling update automatically

**Explaining `kubectl set resources`:**
- `--requests=cpu=200m,memory=256Mi` → minimum guaranteed resources (for scheduling)
- `--limits=cpu=500m,memory=512Mi` → maximum allowed resources
- `-c=nginx` → apply only to container named `nginx` (in case of multi-container pods)

---

## 📋 SECTION 7 — GENERATING YAML FOR EVERY OBJECT TYPE

### 7.1 — Deployment YAML

```bash
kubectl create deployment web-app \
  --image=nginx:1.25 \
  --replicas=3 \
  --port=80 \
  --dry-run=client \
  -o yaml
```

Output:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: web-app
  name: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: web-app
    spec:
      containers:
      - image: nginx:1.25
        name: nginx
        ports:
        - containerPort: 80
        resources: {}
status: {}
```

Save and edit:
```bash
kubectl create deployment web-app --image=nginx:1.25 --replicas=3 \
  --dry-run=client -o yaml > deployment.yaml

# Now edit the file to add resources, probes, etc.
vim deployment.yaml

# Apply
kubectl apply -f deployment.yaml
```

### 7.2 — Pod YAML

```bash
kubectl run nginx-pod --image=nginx:1.25 --dry-run=client -o yaml
```

```bash
# Pod with command
kubectl run busybox-pod \
  --image=busybox \
  --command \
  --dry-run=client \
  -o yaml \
  -- sh -c "sleep 3600"
```

**Explaining `--command`:**
- `--command` → everything after `--` is the COMMAND (overrides the container's ENTRYPOINT)
- Without `--command` → everything after `--` is ARGS (appended to the existing ENTRYPOINT)

### 7.3 — Service YAML

```bash
# Generate ClusterIP service
kubectl expose deployment web-app \
  --port=80 \
  --target-port=80 \
  --dry-run=client \
  -o yaml

# Generate NodePort service
kubectl expose deployment web-app \
  --port=80 \
  --target-port=8080 \
  --type=NodePort \
  --name=web-nodeport \
  --dry-run=client \
  -o yaml
```

### 7.4 — Namespace YAML

```bash
kubectl create namespace banking --dry-run=client -o yaml
```

Output:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  creationTimestamp: null
  name: banking
spec: {}
status: {}
```

### 7.5 — ConfigMap YAML

```bash
kubectl create configmap app-config \
  --from-literal=DB_HOST=mysql \
  --from-literal=DB_PORT=3306 \
  --from-literal=APP_ENV=production \
  --dry-run=client \
  -o yaml
```

Output:
```yaml
apiVersion: v1
data:
  APP_ENV: production
  DB_HOST: mysql
  DB_PORT: "3306"
kind: ConfigMap
metadata:
  creationTimestamp: null
  name: app-config
```

### 7.6 — Secret YAML

```bash
kubectl create secret generic db-creds \
  --from-literal=username=admin \
  --from-literal=password=SecurePass123 \
  --dry-run=client \
  -o yaml
```

Output:
```yaml
apiVersion: v1
data:
  password: U2VjdXJlUGFzczEyMw==    ← base64 encoded automatically
  username: YWRtaW4=
kind: Secret
metadata:
  creationTimestamp: null
  name: db-creds
type: Opaque
```

**Important:** The values are already base64 encoded in the generated YAML. Do not encode them again when editing.

### 7.7 — Job YAML

```bash
kubectl create job db-backup \
  --image=busybox \
  --dry-run=client \
  -o yaml \
  -- sh -c "echo running database backup && sleep 10"
```

### 7.8 — CronJob YAML

```bash
kubectl create cronjob nightly-backup \
  --image=busybox \
  --schedule="0 2 * * *" \
  --dry-run=client \
  -o yaml \
  -- sh -c "echo backup started"
```

### 7.9 — ServiceAccount YAML

```bash
kubectl create serviceaccount monitoring-sa \
  -n monitoring \
  --dry-run=client \
  -o yaml
```

### 7.10 — Role YAML

```bash
kubectl create role pod-reader \
  --verb=get,list,watch \
  --resource=pods,services \
  -n banking \
  --dry-run=client \
  -o yaml
```

### 7.11 — RoleBinding YAML

```bash
kubectl create rolebinding muskan-pod-reader \
  --role=pod-reader \
  --user=muskan \
  -n banking \
  --dry-run=client \
  -o yaml
```

---

## 🔍 SECTION 8 — GETTING / INSPECTING OBJECTS (READ COMMANDS)

### Get — List Objects

```bash
# List all pods in current namespace
kubectl get pods

# List all pods in all namespaces
kubectl get pods -A
kubectl get pods --all-namespaces

# List pods in specific namespace
kubectl get pods -n banking

# List with extra info (node, IP)
kubectl get pods -o wide

# List with labels shown
kubectl get pods --show-labels

# List specific pod
kubectl get pod nginx-pod

# List all resource types at once
kubectl get all
kubectl get all -n banking

# Watch live (updates in real time)
kubectl get pods -w

# List with custom columns
kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,NODE:.spec.nodeName
```

### Get — Output Formats

```bash
# Full YAML of an existing object
kubectl get deployment web-app -o yaml

# Full JSON of an existing object
kubectl get deployment web-app -o json

# Just the name
kubectl get pods -o name

# Specific field using jsonpath
kubectl get pod nginx-pod -o jsonpath='{.status.podIP}'

# Get ALL images in all pods
kubectl get pods -o jsonpath='{.items[*].spec.containers[*].image}'

# Get node names
kubectl get nodes -o jsonpath='{.items[*].metadata.name}'
```

**Explaining jsonpath:**
- `-o jsonpath='...'` → extract a specific field from the JSON output
- `{.status.podIP}` → navigate to `.status.podIP` in the object's JSON
- `{.items[*]}` → `items` is the list when getting multiple objects, `[*]` means all items
- This is powerful for scripting — get exactly the field you need

### Describe — Detailed Info

```bash
# Describe a pod (shows events, conditions, volumes, etc.)
kubectl describe pod nginx-pod

# Describe a deployment
kubectl describe deployment web-app

# Describe a node (shows capacity, conditions, pods running on it)
kubectl describe node node-1

# Describe a service (shows endpoints, selectors)
kubectl describe service web-svc

# Describe all pods with a label
kubectl describe pods -l app=nginx
```

**When to use get vs describe:**
```
kubectl get   → quick overview, one-line per object
kubectl describe → full details, events, everything
```

---

## 💻 SECTION 9 — HANDS-ON LAB

> Every command explained word by word. Nothing skipped.

---

### LAB 1 — Generate, Edit, and Apply a Deployment

```bash
# Step 1: Generate YAML without creating
kubectl create deployment payment-app \
  --image=nginx:1.25 \
  --replicas=2 \
  --dry-run=client \
  -o yaml > payment-deployment.yaml
```
- `> payment-deployment.yaml` → redirect output to a file instead of printing to terminal

```bash
# Step 2: Open and edit the file
vim payment-deployment.yaml
```

Add resource limits — find the container section and add:
```yaml
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "200m"
            memory: "256Mi"
```

Also add a readiness probe:
```yaml
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
```

```bash
# Step 3: Validate the edited file (server dry run)
kubectl apply -f payment-deployment.yaml --dry-run=server
```
- `--dry-run=server` → test through the real API Server pipeline, don't write to etcd

```bash
# Step 4: Apply for real
kubectl apply -f payment-deployment.yaml
```

```bash
# Step 5: Verify
kubectl get deployment payment-app
kubectl describe deployment payment-app
```

---

### LAB 2 — Edit a Live Object After Deleting the YAML File

```bash
# Create a deployment (simulate having no YAML file)
kubectl create deployment live-edit-demo --image=nginx --replicas=2
```

```bash
# "Accidentally delete" the yaml file (or pretend you never had one)
# The deployment still exists in Kubernetes

# Check current state
kubectl get deployment live-edit-demo
```

```bash
# METHOD 1: kubectl edit — open in editor
kubectl edit deployment live-edit-demo
```
- Find `replicas: 2` → change to `replicas: 5`
- Save and quit: press `Esc`, type `:wq`, press Enter
- Output: `deployment.apps/live-edit-demo edited`

```bash
# Verify the change took effect
kubectl get deployment live-edit-demo
# READY column should show 5/5
```

```bash
# METHOD 2: kubectl scale — faster for replicas
kubectl scale deployment live-edit-demo --replicas=1
kubectl get deployment live-edit-demo
```

```bash
# METHOD 3: kubectl patch — surgical JSON change
kubectl patch deployment live-edit-demo -p '{"spec":{"replicas":4}}'
kubectl get deployment live-edit-demo
```

```bash
# Get the YAML back (reconstruct your "lost" file)
kubectl get deployment live-edit-demo -o yaml > live-edit-demo-recovered.yaml
cat live-edit-demo-recovered.yaml
```
- This is how you recover a YAML file from a live running object
- Note: the recovered YAML has extra auto-generated fields (resourceVersion, uid, etc.)
- These are fine to keep — kubectl apply handles them

---

### LAB 3 — Change Container Image Without a File

```bash
# Create a deployment with nginx 1.24
kubectl create deployment image-demo --image=nginx:1.24

# Check current image
kubectl get deployment image-demo -o jsonpath='{.spec.template.spec.containers[0].image}'
```
- `jsonpath` → extract a specific field
- `.spec.template.spec.containers[0].image` → path to the first container's image field
- `[0]` → first item in the containers array (index starts at 0)

```bash
# Change image using kubectl set image
kubectl set image deployment/image-demo nginx=nginx:1.25
```
- `deployment/image-demo` → target the deployment
- `nginx=nginx:1.25` → container-name=new-image. The container name is `nginx` (same as deployment name by default)

```bash
# Watch the rolling update happen
kubectl rollout status deployment/image-demo
```
- `rollout status` → watch and print progress of the rolling update

```bash
# Verify image changed
kubectl get deployment image-demo -o jsonpath='{.spec.template.spec.containers[0].image}'
```

```bash
# If the new image has problems — rollback!
kubectl rollout undo deployment/image-demo
```

---

### LAB 4 — Generate Multiple Object YAMLs and Apply Together

```bash
# Generate deployment YAML
kubectl create deployment multi-demo \
  --image=nginx \
  --replicas=3 \
  --dry-run=client \
  -o yaml > multi-demo.yaml
```

```bash
# Append a separator and the service YAML to the SAME file
echo "---" >> multi-demo.yaml
kubectl expose deployment multi-demo \
  --port=80 \
  --type=ClusterIP \
  --dry-run=client \
  -o yaml >> multi-demo.yaml
```

**Explaining:**
- `>>` → append to file (not overwrite — `>` overwrites, `>>` appends)
- `echo "---"` → prints `---` which is the YAML document separator. Multiple objects in one file are separated by `---`

```bash
# Apply the combined file (creates both objects at once)
kubectl apply -f multi-demo.yaml
```

```bash
# Verify both were created
kubectl get deployment multi-demo
kubectl get service multi-demo
```

---

### LAB 5 — Extract Specific Information with jsonpath

```bash
# Create some pods
kubectl create deployment jsonpath-demo --image=nginx --replicas=3
kubectl rollout status deployment/jsonpath-demo
```

```bash
# Get all pod IPs
kubectl get pods -o jsonpath='{.items[*].status.podIP}'
```
- `.items[*]` → all pods in the list
- `.status.podIP` → the IP field inside each pod's status

```bash
# Get pod names and their nodes (formatted with newlines)
kubectl get pods \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.nodeName}{"\n"}{end}'
```
- `{range .items[*]}...{end}` → loop through all items
- `{"\t"}` → tab character (for spacing)
- `{"\n"}` → newline character (for each item on its own line)

```bash
# Get all container images in all pods in all namespaces
kubectl get pods -A \
  -o jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.metadata.name}{"\t"}{range .spec.containers[*]}{.image}{"\n"}{end}{end}'
```

```bash
# Clean up
kubectl delete deployment jsonpath-demo multi-demo payment-app live-edit-demo image-demo
```

---

### LAB 6 — Checking What Exists and Getting Object YAML Back

```bash
# See ALL object types in a namespace
kubectl get all -n default
```

```bash
# See EVERY type of Kubernetes object (including CRDs)
kubectl get $(kubectl api-resources --verbs=list --namespaced -o name | tr '\n' ',' | sed 's/,$//') -n default 2>/dev/null
```
- `kubectl api-resources` → list all resource types
- `--verbs=list` → only resources that support listing
- `--namespaced` → only namespaced resources (not cluster-level)
- `-o name` → output just names
- `tr '\n' ','` → replace newlines with commas (to make a comma-separated list)
- `sed 's/,$//'` → remove trailing comma
- This lists EVERY single resource type in the namespace

```bash
# Recover YAML of any existing object
kubectl get deployment web-app -o yaml > recovered-deployment.yaml
kubectl get service web-svc -o yaml > recovered-service.yaml
kubectl get configmap app-config -o yaml > recovered-configmap.yaml
```

```bash
# Compare what you have vs what's running
diff original-deployment.yaml recovered-deployment.yaml
```
- `diff` → compare two files and show differences

---

## 🎯 SECTION 10 — TRICKY COMMAND PATTERNS

### Replace vs Apply vs Create

```bash
# CREATE — fails if object already exists
kubectl create -f deployment.yaml
# Use when: creating for the first time

# APPLY — creates OR updates (idempotent)
kubectl apply -f deployment.yaml
# Use when: you want to create OR update without caring which it is
# This is the PREFERRED method in production

# REPLACE — deletes and recreates the object
kubectl replace -f deployment.yaml
# Use when: field is immutable and apply won't work
# More destructive — causes brief downtime

# REPLACE with force (delete and recreate)
kubectl replace --force -f deployment.yaml
```

### Pipe YAML Directly from echo or cat

```bash
# Apply YAML without a file at all (heredoc)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: quick-pod
spec:
  containers:
  - name: nginx
    image: nginx
EOF
```

**Explaining:**
- `cat <<EOF` → start a heredoc. Everything until `EOF` is treated as a file.
- `| kubectl apply -f -` → pipe the heredoc as stdin to kubectl. The `-` means "read from stdin instead of a file."
- This lets you apply YAML without creating any file on disk.

### Combine Generate and Immediately Apply

```bash
# Generate and immediately apply without saving to file
kubectl create deployment quick-deploy \
  --image=nginx \
  --replicas=3 \
  --dry-run=client \
  -o yaml | kubectl apply -f -
```

**Explaining:**
- Generate YAML with `--dry-run=client -o yaml`
- Pipe it directly to `kubectl apply -f -`
- The `-` in `-f -` means "read the file from stdin"
- Result: deployment created without writing any file

---

## 📊 SECTION 11 — EDITING METHOD COMPARISON TABLE

| Method | Command | Best For | Requires File? | Git Trackable? |
|--------|---------|----------|---------------|----------------|
| `kubectl edit` | `kubectl edit deployment web-app` | Any field, complex changes | ❌ No | ❌ No |
| `kubectl scale` | `kubectl scale deployment web-app --replicas=5` | Replica count only | ❌ No | ❌ No |
| `kubectl patch` | `kubectl patch deployment web-app -p '{...}'` | One specific field | ❌ No | ❌ No |
| `kubectl set image` | `kubectl set image deployment/web-app c=img:tag` | Container image | ❌ No | ❌ No |
| `kubectl set resources` | `kubectl set resources deployment ...` | CPU/memory limits | ❌ No | ❌ No |
| `kubectl set env` | `kubectl set env deployment/web-app KEY=value` | Environment variables | ❌ No | ❌ No |
| `kubectl apply -f` | `kubectl apply -f deployment.yaml` | Full object management | ✅ Yes | ✅ Yes |
| `kubectl replace` | `kubectl replace -f deployment.yaml` | Immutable field changes | ✅ Yes | ✅ Yes |

---

## 🔑 SECTION 12 — KEY TERMS TO REMEMBER

| Term | Simple Meaning |
|------|----------------|
| **Imperative** | Tell Kubernetes WHAT to do directly — `kubectl create deployment...` |
| **Declarative** | Write a file describing desired state — `kubectl apply -f file.yaml` |
| **--dry-run=client** | Simulate locally, never contacts API Server, just prints result |
| **--dry-run=server** | Sends to API Server, runs full pipeline, stops before etcd write |
| **-o yaml** | Print full YAML of the object instead of a brief confirmation |
| **-o json** | Print full JSON of the object |
| **-o wide** | Print extra columns (IP, node) in the list output |
| **-o jsonpath** | Extract a specific field using a path expression |
| **kubectl edit** | Open live object in text editor, save to apply changes |
| **kubectl scale** | Change replica count directly |
| **kubectl patch** | Change one specific field using JSON |
| **kubectl set image** | Change container image (triggers rolling update) |
| **kubectl set resources** | Change CPU/memory requests and limits |
| **kubectl set env** | Change/add/remove environment variables |
| **>>** | Append to file (adds to end) |
| **>** | Overwrite file (replaces completely) |
| **stdin (-)** | `-f -` means read YAML from pipe/stdin instead of a file |
| **heredoc (EOF)** | `cat <<EOF ... EOF` writes multi-line content without a file |
| **kubectl replace** | Delete and recreate object (more destructive than apply) |
| **jsonpath** | Path notation to extract specific fields from JSON/YAML output |

---

*File: K8s_ImperativeCommands_DryRun_YAML_Concept_and_Lab.md*
*Repository: Interview_Preparation_2026 → Kubernetes/*
*Next: K8s_ImperativeCommands_DryRun_YAML_Interview_Questions.md*
