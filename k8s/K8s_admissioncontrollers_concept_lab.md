# Kubernetes API Server Request Flow — Admission Controllers, Mutating & Validating Webhooks
## Deep Dive Concept + Hands-On Lab
> Written for: Someone with 4 years of DevOps experience preparing for senior-level interviews
> Style: First-standard student explanation → deep technical truth → hands-on lab with line-by-line explanation

---

## 🧠 SECTION 1 — WHAT IS THIS TOPIC ABOUT? (Story First)

### Forget Kubernetes for 3 minutes — start with an airport story

Imagine you are flying internationally. You walk up to the check-in counter with your ticket and passport.

Before you can get on the plane, you go through **multiple checkpoints**:

```
CHECKPOINT 1: Ticket counter
  → "Is this a real ticket? Is your name on it?"
  → They VERIFY your identity
  → If fake ticket → REJECTED immediately

CHECKPOINT 2: Passport Control
  → "Are you allowed to travel to this country?"
  → They CHECK your authorization
  → If visa expired → REJECTED

CHECKPOINT 3: Security — MODIFICATION DESK (special desk)
  → "You have a water bottle — you can keep it but we must empty it"
  → They MODIFY something about you before letting you proceed
  → You pass through, but slightly changed

CHECKPOINT 4: Security Scanner
  → "Do you have any weapons? Any prohibited items?"
  → They VALIDATE everything is safe
  → If weapon found → REJECTED

CHECKPOINT 5: Gate
  → "All checks passed — board the plane"
  → Request is ACCEPTED and written to the flight manifest
```

**This airport IS the Kubernetes API Server.**

Every time you run `kubectl apply`, `kubectl create`, `kubectl delete` — your request goes through this **exact same pipeline of checkpoints** before anything actually happens in the cluster.

The checkpoints are called **Admission Controllers**.

---

## 🏗️ SECTION 2 — THE FULL REQUEST PIPELINE

Here is the complete picture of what happens between `kubectl apply` and data being written to etcd:

```
kubectl apply -f pod.yaml
        │
        ▼
┌─────────────────────────────────────────────────────────────────┐
│                        API SERVER                               │
│                                                                 │
│  Step 1: AUTHENTICATION                                         │
│  "Who are you? Prove it."                                       │
│  → Checks your certificate / token / kubeconfig                 │
│  → FAIL → 401 Unauthorized → request DIES here                 │
│                                                                 │
│  Step 2: AUTHORIZATION (RBAC)                                   │
│  "Are you ALLOWED to do this?"                                  │
│  → Checks your Role/ClusterRole permissions                     │
│  → FAIL → 403 Forbidden → request DIES here                    │
│                                                                 │
│  Step 3: MUTATING ADMISSION WEBHOOKS                            │
│  "Can I ADD or CHANGE anything before saving?"                  │
│  → External webhook servers can MODIFY the request              │
│  → Runs ALL mutating webhooks in sequence                       │
│  → Request may come out DIFFERENT than it went in               │
│                                                                 │
│  Step 4: OBJECT SCHEMA VALIDATION                               │
│  "Is the YAML/JSON structure valid?"                            │
│  → Checks required fields, correct types, valid values          │
│  → FAIL → 422 Unprocessable Entity → request DIES here         │
│                                                                 │
│  Step 5: VALIDATING ADMISSION WEBHOOKS                          │
│  "Should we ALLOW or DENY this specific request?"               │
│  → External webhook servers can REJECT the request              │
│  → If ANY validating webhook says NO → request DIES here        │
│                                                                 │
│  Step 6: PERSISTED TO etcd                                      │
│  → Request passes all checks                                    │
│  → Object written to etcd (the cluster database)               │
│  → This is the "point of no return"                             │
│  → 201 Created response sent back to kubectl                    │
└─────────────────────────────────────────────────────────────────┘
```

### The ONE Rule to Remember:

> **Mutating webhooks run BEFORE Validating webhooks.**
> MODIFY first, then CHECK if the modified result is acceptable.

Why this order? Because the validator needs to see the FINAL version of the object — including anything the mutator added. If validation ran first, it might reject something that the mutator was about to fix.

---

## 🔍 SECTION 3 — WHAT ARE ADMISSION CONTROLLERS?

### The Simple Definition

An **Admission Controller** is a piece of code that sits inside the API Server pipeline and intercepts requests AFTER authentication and authorization but BEFORE the object is saved to etcd.

Think of them as **bouncers at a nightclub**. Authentication is checking your ID at the door. Authorization is checking if you're on the guest list. Admission Controllers are the **bouncers inside** who check: are you dressed appropriately? Are you carrying anything prohibited? Do you need a wristband?

### Two Types of Admission Controllers

```
TYPE 1: MUTATING Admission Controller
  → Can READ the request AND MODIFY it
  → Adds fields, sets defaults, injects things
  → Like a hotel concierge who checks your booking
    AND automatically upgrades your room if available

TYPE 2: VALIDATING Admission Controller  
  → Can READ the request but CANNOT MODIFY it
  → Can only say YES (allow) or NO (reject)
  → Like a safety inspector who can fail your car
    but cannot fix it — just approves or rejects

KEY POINT: Some controllers are BOTH mutating and validating
  → First they modify → then they validate their own modification
```

### Built-in vs Webhook-based

There are two ways admission controllers are implemented:

```
BUILT-IN (compiled into the API Server binary):
  → Always available, no extra setup needed
  → Examples: NamespaceLifecycle, LimitRanger, ResourceQuota
  → Enabled/disabled via --enable-admission-plugins flag on API Server

WEBHOOK-BASED (external servers):
  → You deploy a separate web server
  → API Server calls this server via HTTPS for every request
  → Server responds: allow or deny (and optionally what changes to make)
  → Examples: OPA Gatekeeper, Istio, Kyverno
  → Configured via MutatingWebhookConfiguration and ValidatingWebhookConfiguration objects
```

---

## 🔄 SECTION 4 — MUTATING ADMISSION WEBHOOKS (Deep Dive)

### What Is a Mutating Webhook?

A mutating webhook is an **external HTTPS server** that the API Server calls during Step 3 of the pipeline. The API Server sends the request to the webhook server, the webhook server can **change the object**, and sends it back. The API Server uses the modified version going forward.

### The Modification Mechanism — JSON Patch

Mutating webhooks don't send back the whole modified object. They send back a **JSON Patch** — a list of specific changes to make.

```json
[
  {"op": "add", "path": "/spec/containers/0/resources/limits/memory", "value": "256Mi"},
  {"op": "add", "path": "/metadata/labels/injected-by", "value": "mutating-webhook"}
]
```

**Breaking this down:**
- `op: "add"` → operation type: add a new field
- `path` → which field in the object to target (using JSON Pointer notation)
- `value` → what value to set

The API Server applies these patches to the original object.

### Real-World Examples of Mutating Webhooks

```
EXAMPLE 1: Istio Sidecar Injection
  You create a Pod with ONE container (your app).
  Istio's mutating webhook intercepts the request.
  It ADDS a second container (the Envoy sidecar proxy) to your pod spec.
  The pod that gets created has TWO containers — your app + Envoy.
  You never wrote the Envoy container. The webhook added it automatically.

EXAMPLE 2: LimitRanger (built-in)
  You create a Pod WITHOUT resource limits (no CPU/memory limits defined).
  LimitRanger mutating webhook fires.
  It ADDS default resource limits based on the namespace's LimitRange config.
  Your pod gets resource limits you never specified.

EXAMPLE 3: Pod Security Admission (default security profile)
  You create a Pod without a securityContext.
  A mutating webhook ADDS default security settings:
    → runAsNonRoot: true
    → readOnlyRootFilesystem: true
  Your pod is more secure without you writing the security config.

EXAMPLE 4: Annotation Injection
  Your company has a webhook that ADDS annotations to every object:
    → "created-by: ci-pipeline"
    → "environment: production"
    → "cost-center: engineering"
  Used for billing, auditing, tracking — happens automatically on every create.
```

### The AdmissionReview Object

When the API Server calls a webhook, it sends an **AdmissionReview** object. The webhook receives this, processes it, and sends back an **AdmissionReview** response.

```json
REQUEST (API Server → Webhook):
{
  "apiVersion": "admission.k8s.io/v1",
  "kind": "AdmissionReview",
  "request": {
    "uid": "abc123",
    "kind": {"group": "apps", "version": "v1", "kind": "Deployment"},
    "operation": "CREATE",
    "object": {
      ...the full object being created...
    }
  }
}

RESPONSE (Webhook → API Server):
{
  "apiVersion": "admission.k8s.io/v1",
  "kind": "AdmissionReview",
  "response": {
    "uid": "abc123",
    "allowed": true,
    "patchType": "JSONPatch",
    "patch": "W3sib3AiOiAiYWRkIiwgInBhdGgiOiAiL..."  ← base64 encoded JSON patch
  }
}
```

**Key fields in the response:**
- `uid` → must match the request UID (so the API Server knows which request this response is for)
- `allowed: true` → "I'm not blocking this request"
- `patch` → the changes to make (base64 encoded JSON Patch)
- If `allowed: false` → request is rejected (but mutating webhooks usually don't reject, that's the validator's job)

---

## ✅ SECTION 5 — VALIDATING ADMISSION WEBHOOKS (Deep Dive)

### What Is a Validating Webhook?

A validating webhook is also an **external HTTPS server**. But unlike mutating webhooks, it **cannot modify the object**. It can only look at the final object (after all mutations have been applied) and say:

```
"YES, this is acceptable" → allowed: true
"NO, I am blocking this" → allowed: false + reason message
```

### Real-World Examples of Validating Webhooks

```
EXAMPLE 1: OPA Gatekeeper — Policy Enforcement
  You try to create a Pod without resource limits.
  OPA Gatekeeper validates the request.
  Policy says: "All pods MUST have CPU and memory limits"
  Pod has no limits → REJECTED
  Error: "Resource limits are required. Add limits.cpu and limits.memory"

EXAMPLE 2: Image Registry Policy
  You try to deploy an image from Docker Hub (public registry).
  Company policy: only allow images from internal ECR registry.
  Validating webhook checks the image name.
  Image: "nginx:latest" (from Docker Hub) → REJECTED
  Error: "Only images from 123456789.dkr.ecr.us-east-1.amazonaws.com are allowed"

EXAMPLE 3: Label Enforcement
  Every resource MUST have label "team" and "cost-center".
  Validating webhook checks metadata.labels.
  Missing labels → REJECTED
  Error: "Required labels missing: team, cost-center"

EXAMPLE 4: Namespace Restrictions
  Certain namespaces are protected — no one can create Pods directly.
  Validating webhook: if namespace = "production" and user is not CI/CD SA → REJECTED
  Error: "Direct Pod creation is not allowed in production namespace. Use Deployments."

EXAMPLE 5: ResourceQuota (built-in)
  Namespace has quota: max 10 pods.
  Currently 10 pods exist. You try to create pod 11.
  ResourceQuota validating controller → REJECTED
  Error: "exceeded quota: pods, requested: 1, used: 10, limited: 10"
```

### The Response — Validating Webhook

```json
ALLOW response:
{
  "response": {
    "uid": "abc123",
    "allowed": true
  }
}

DENY response:
{
  "response": {
    "uid": "abc123",
    "allowed": false,
    "status": {
      "code": 403,
      "message": "Resource limits are required for all pods. Add limits.cpu and limits.memory to your pod spec."
    }
  }
}
```

The `message` field is what the user sees when kubectl returns an error. Write clear, helpful messages — the user will see exactly what you put here.

---

## 📋 SECTION 6 — WEBHOOK CONFIGURATION OBJECTS

### MutatingWebhookConfiguration

This is the Kubernetes object that registers a mutating webhook with the API Server. It tells the API Server: "for these kinds of requests, call THIS URL."

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: sidecar-injector
webhooks:
- name: inject-sidecar.example.com
  clientConfig:
    service:
      name: sidecar-injector-svc
      namespace: istio-system
      path: "/mutate"
    caBundle: <base64-encoded-CA-cert>
  rules:
  - apiGroups: [""]
    apiVersions: ["v1"]
    operations: ["CREATE"]
    resources: ["pods"]
  namespaceSelector:
    matchLabels:
      istio-injection: enabled
  admissionReviewVersions: ["v1"]
  sideEffects: None
  failurePolicy: Ignore
  timeoutSeconds: 5
```

**Explaining EVERY line:**

```yaml
apiVersion: admissionregistration.k8s.io/v1
```
→ This object belongs to the `admissionregistration.k8s.io` API group — the group specifically for webhook registration.

```yaml
kind: MutatingWebhookConfiguration
```
→ We are creating a Mutating webhook registration (not Validating).

```yaml
  name: sidecar-injector
```
→ Name of this webhook configuration object. Can be anything descriptive.

```yaml
webhooks:
- name: inject-sidecar.example.com
```
→ `webhooks` is a list — you can register MULTIPLE webhooks in one configuration object.
→ Each webhook needs a `name` — must be a fully qualified domain name format.

```yaml
  clientConfig:
    service:
      name: sidecar-injector-svc
      namespace: istio-system
      path: "/mutate"
```
→ `clientConfig` tells the API Server WHERE to call.
→ `service` = the webhook server is running as a Kubernetes Service (internal).
→ `name: sidecar-injector-svc` = the Service name.
→ `namespace: istio-system` = which namespace that Service is in.
→ `path: "/mutate"` = the HTTP path on that server to call (like `/mutate` or `/webhook`).
→ Alternative to `service`: use `url` for an external HTTPS server outside the cluster.

```yaml
    caBundle: <base64-encoded-CA-cert>
```
→ The API Server needs to verify the webhook server's TLS certificate.
→ `caBundle` is the CA certificate (base64 encoded) that signed the webhook server's cert.
→ Without this, the API Server cannot trust the webhook server's HTTPS connection.

```yaml
  rules:
  - apiGroups: [""]
    apiVersions: ["v1"]
    operations: ["CREATE"]
    resources: ["pods"]
```
→ `rules` = WHEN should this webhook be called?
→ `apiGroups: [""]` = core API group (empty string = core group, which includes Pods, Services).
→ `apiVersions: ["v1"]` = only v1 version.
→ `operations: ["CREATE"]` = only on CREATE operations. Could also be `["CREATE", "UPDATE", "DELETE"]`.
→ `resources: ["pods"]` = only for Pod objects. Could be `["deployments", "services"]` etc.

```yaml
  namespaceSelector:
    matchLabels:
      istio-injection: enabled
```
→ `namespaceSelector` = only call this webhook for objects in namespaces that have this label.
→ `istio-injection: enabled` = only inject sidecars into namespaces labeled with this.
→ This is how Istio works — you label a namespace and Istio automatically starts injecting sidecars there.
→ Without this, the webhook would fire for EVERY pod in EVERY namespace.

```yaml
  admissionReviewVersions: ["v1"]
```
→ Which version of the AdmissionReview API format to use when calling this webhook.
→ Must match what the webhook server supports.

```yaml
  sideEffects: None
```
→ Does this webhook have side effects outside the request? (e.g., writing to a database)
→ `None` = no side effects. The webhook only looks at and modifies the incoming request.
→ Important for dry-run requests — if sideEffects is not None, dry-run won't work properly.

```yaml
  failurePolicy: Ignore
```
→ What happens if the webhook server is DOWN or times out?
→ `Ignore` = if webhook is unreachable → proceed as if webhook said "allow". Request goes through.
→ `Fail` = if webhook is unreachable → reject the request with an error.
→ `Ignore` is safer for availability (cluster keeps working if webhook dies).
→ `Fail` is safer for security (no requests slip through if policy enforcement is down).

```yaml
  timeoutSeconds: 5
```
→ How long the API Server waits for the webhook to respond before applying `failurePolicy`.
→ 5 seconds = if webhook doesn't respond in 5 seconds → apply failurePolicy.
→ Keep this low — a slow webhook delays EVERY request that matches its rules.

---

### ValidatingWebhookConfiguration

Same structure as MutatingWebhookConfiguration but with `kind: ValidatingWebhookConfiguration`.

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: policy-enforcer
webhooks:
- name: validate-resources.company.com
  clientConfig:
    service:
      name: opa-gatekeeper-svc
      namespace: gatekeeper-system
      path: "/validate"
    caBundle: <base64-encoded-CA-cert>
  rules:
  - apiGroups: ["*"]
    apiVersions: ["*"]
    operations: ["CREATE", "UPDATE"]
    resources: ["pods", "deployments"]
  failurePolicy: Fail
  timeoutSeconds: 10
  sideEffects: None
  admissionReviewVersions: ["v1"]
```

**New things here:**
- `apiGroups: ["*"]` → `*` means ALL API groups (core, apps, batch, etc.)
- `apiVersions: ["*"]` → all versions
- `operations: ["CREATE", "UPDATE"]` → check on both create AND update
- `failurePolicy: Fail` → for policy enforcement, if the validator is down → block all requests (security first)
- `timeoutSeconds: 10` → slightly higher timeout for a complex policy engine like OPA

---

## ⚙️ SECTION 7 — BUILT-IN ADMISSION CONTROLLERS

These are compiled INTO the API Server. You enable/disable them with the `--enable-admission-plugins` flag.

### The Important Built-in Controllers:

```
NamespaceLifecycle
  → Prevents creating objects in a namespace that is being deleted
  → Prevents deletion of system namespaces (default, kube-system, kube-public)
  → Type: Validating

LimitRanger
  → Applies default CPU/memory requests and limits from LimitRange objects
  → If a pod doesn't specify limits → LimitRanger adds them automatically
  → Type: Mutating + Validating (adds defaults AND enforces min/max)

ResourceQuota
  → Enforces namespace-level resource quotas
  → Counts current usage + requested usage, rejects if over quota
  → Type: Validating

ServiceAccount
  → If a pod doesn't specify a serviceAccountName → adds "default" ServiceAccount
  → Mounts the service account token as a volume automatically
  → Type: Mutating

DefaultStorageClass
  → If a PVC doesn't specify a storageClassName → adds the default StorageClass
  → Makes storage "just work" without specifying class every time
  → Type: Mutating

PodSecurity (replaced PodSecurityPolicy in K8s 1.25+)
  → Enforces Pod Security Standards (Privileged, Baseline, Restricted)
  → Set at namespace level with labels
  → Type: Validating

MutatingAdmissionWebhook
  → The controller that CALLS external mutating webhooks
  → This is what enables webhook-based mutation
  → Must be enabled for external mutating webhooks to work

ValidatingAdmissionWebhook
  → The controller that CALLS external validating webhooks
  → Must be enabled for external validating webhooks to work
```

### How to See Which Are Enabled:

```bash
kubectl describe pod kube-apiserver-controlplane -n kube-system | grep admission
```

Look for: `--enable-admission-plugins=NodeRestriction,ResourceQuota,...`

---

## 🔐 SECTION 8 — WEBHOOK SECURITY (TLS Is Not Optional)

### Why Webhooks MUST Use HTTPS

The API Server sends all requests to webhooks over HTTPS. This is mandatory — no HTTP allowed.

Why? Because the webhook sees EVERY request including:
- Secrets being created (containing passwords, API keys)
- ServiceAccount tokens
- RBAC definitions

If webhooks could use HTTP, someone could intercept this traffic and steal all your secrets. TLS is non-negotiable.

### The Certificate Chain for Webhooks

```
Webhook Server has:
  1. A TLS private key (generated when setting up the webhook)
  2. A TLS certificate signed by a CA (could be cluster CA or self-signed CA)

The MutatingWebhookConfiguration has:
  3. caBundle → the CA certificate that signed the webhook's cert
                API Server uses this to VERIFY the webhook's identity

Flow:
  API Server → connects to webhook via HTTPS
  Webhook presents its TLS cert
  API Server verifies: "Is this cert signed by the CA in caBundle?"
  If YES → connection trusted, AdmissionReview sent
  If NO → connection rejected, failurePolicy applied
```

---

## 🔄 SECTION 9 — COMPLETE FLOW WITH REAL EXAMPLE

Let's trace EXACTLY what happens when you run:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  namespace: production
spec:
  containers:
  - name: app
    image: nginx:latest
EOF
```

```
STEP 1: AUTHENTICATION
  kubectl reads ~/.kube/config
  Finds your client certificate
  Sends HTTPS POST to API Server port 6443
  API Server: "Is this certificate signed by the cluster CA?"
  YES → proceed. NO → 401 Unauthorized.

STEP 2: AUTHORIZATION (RBAC)
  API Server checks: "Can user 'muskan' CREATE pods in namespace 'production'?"
  Checks RoleBindings/ClusterRoleBindings for user 'muskan'
  YES → proceed. NO → 403 Forbidden.

STEP 3: MUTATING ADMISSION WEBHOOKS
  API Server checks: which MutatingWebhookConfigurations match this request?
  (Pod, CREATE operation, production namespace)
  
  Found: Istio sidecar injector (namespace has label istio-injection=enabled)
    → Sends AdmissionReview to https://sidecar-injector-svc.istio-system/mutate
    → Webhook responds: here is a JSON patch
    → Patch adds: a second container 'istio-proxy' to spec.containers
    → Object now has 2 containers instead of 1
  
  Found: Default limits injector
    → Sends AdmissionReview with the ALREADY MUTATED object
    → Webhook responds: add default resource limits
    → Patch adds: requests and limits to both containers
  
  Object has now been MODIFIED by 2 mutations.

STEP 4: SCHEMA VALIDATION
  API Server validates the MUTATED object against Pod schema
  Checks: all required fields present? Correct types?
  Container names valid? Port numbers in range?
  PASS → proceed. FAIL → 422 Unprocessable Entity.

STEP 5: VALIDATING ADMISSION WEBHOOKS
  API Server checks: which ValidatingWebhookConfigurations match?
  
  Found: OPA Gatekeeper policy enforcer
    → Sends AdmissionReview with FINAL mutated object
    → Policy 1: "All pods must have resource limits" → PASS (mutator added them)
    → Policy 2: "Image must be from approved registry" 
    → Image: "nginx:latest" from Docker Hub → FAIL
    → Response: allowed: false, message: "Only ECR images allowed"
  
  REQUEST REJECTED.
  kubectl output: Error from server: "Only ECR images allowed"

--- If the image had been from ECR, we continue: ---

STEP 6: WRITTEN TO etcd
  All checks passed
  Final mutated, validated object written to etcd
  API Server responds: 201 Created
  kubectl output: pod/my-app created
  
  (Then: Scheduler assigns node, kubelet starts container — separate flow)
```

---

## 💻 SECTION 10 — HANDS-ON LAB

> Every command explained word by word. Every YAML line explained. Nothing skipped.

### Lab Prerequisites

- Kubernetes cluster running (MicroK8s or KillerCoda)
- kubectl configured and working
- For webhook labs: you need to be able to create Deployments and Services

---

### LAB 1 — See Which Admission Controllers Are Currently Enabled

```bash
kubectl describe pod kube-apiserver-controlplane -n kube-system | grep -i admission
```

**Breaking down:**
- `describe pod kube-apiserver-controlplane` → detailed info about the API Server pod
- `-n kube-system` → in kube-system namespace
- `|` → pipe
- `grep -i admission` → search for "admission" (case-insensitive with `-i`)

**Output:**
```
--enable-admission-plugins=NodeRestriction
```
Or more commonly on kubeadm:
```
--enable-admission-plugins=NodeRestriction,ResourceQuota,LimitRanger
```

**If you can also check the API server config file directly:**
```bash
sudo cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep admission
```

**Breaking down:**
- `sudo` → run as root (needed to read this system file)
- `cat` → print file contents to terminal
- `/etc/kubernetes/manifests/kube-apiserver.yaml` → the API Server static pod manifest
- `| grep admission` → filter for admission-related lines

---

### LAB 2 — See the LimitRanger Mutating Controller in Action

**Step 1: Create a LimitRange in a namespace**

```bash
kubectl create namespace lab-limits
```

**Breaking down:**
- `create namespace` → create a new Namespace object
- `lab-limits` → name of the namespace

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: lab-limits
spec:
  limits:
  - default:
      cpu: "200m"
      memory: "256Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
    type: Container
EOF
```

**Explaining every line:**

```yaml
apiVersion: v1
```
→ LimitRange is in the core API group (v1).

```yaml
kind: LimitRange
```
→ Creating a LimitRange object.

```yaml
  name: default-limits
  namespace: lab-limits
```
→ This LimitRange is named `default-limits` and applies to the `lab-limits` namespace only.

```yaml
spec:
  limits:
  - default:
      cpu: "200m"
      memory: "256Mi"
```
→ `limits` → the list of limit rules.
→ `default` → these are the DEFAULT LIMITS applied when a container doesn't specify limits.
→ `cpu: "200m"` → 200 millicores = 0.2 of one CPU core. If a container doesn't say how much CPU it can use → it gets this.
→ `memory: "256Mi"` → 256 Mebibytes. Default memory limit.

```yaml
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
```
→ `defaultRequest` → these are default REQUESTS (minimum guaranteed resources) when not specified.
→ Requests are what the scheduler uses to find a node.
→ Limits are the maximum the container can use.

```yaml
    type: Container
```
→ This LimitRange applies at the Container level (not Pod or PersistentVolumeClaim level).

**Step 2: Create a pod WITHOUT resource limits**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: no-limits-pod
  namespace: lab-limits
spec:
  containers:
  - name: app
    image: nginx
EOF
```

Notice: NO `resources:` section. No limits. No requests. Nothing.

**Step 3: Check if the LimitRanger mutating controller added them**

```bash
kubectl describe pod no-limits-pod -n lab-limits | grep -A 5 "Limits:"
```

**Breaking down:**
- `describe pod no-limits-pod` → detailed info about our pod
- `-n lab-limits` → in the lab-limits namespace
- `| grep -A 5 "Limits:"` → find the "Limits:" line and show 5 lines AFTER it

**Output:**
```
Limits:
  cpu:     200m
  memory:  256Mi
Requests:
  cpu:     100m
  memory:  128Mi
```

**You never wrote those limits. The LimitRanger mutating controller added them automatically.**
This is mutation happening right in front of you.

---

### LAB 3 — See the ResourceQuota Validating Controller in Action

**Step 1: Create a ResourceQuota**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: pod-quota
  namespace: lab-limits
spec:
  hard:
    pods: "3"
    requests.cpu: "500m"
    requests.memory: "512Mi"
EOF
```

**Explaining every line:**

```yaml
kind: ResourceQuota
```
→ Creating a ResourceQuota object.

```yaml
spec:
  hard:
    pods: "3"
```
→ `hard` → these are HARD LIMITS. Cannot be exceeded.
→ `pods: "3"` → maximum 3 pods in this namespace at any time.

```yaml
    requests.cpu: "500m"
```
→ Total CPU requests across ALL pods in this namespace cannot exceed 500 millicores.

```yaml
    requests.memory: "512Mi"
```
→ Total memory requests across ALL pods cannot exceed 512Mi.

**Step 2: Create pods until quota is hit**

```bash
# Create pod 1
kubectl run quota-test-1 --image=nginx -n lab-limits

# Create pod 2
kubectl run quota-test-2 --image=nginx -n lab-limits

# Create pod 3
kubectl run quota-test-3 --image=nginx -n lab-limits

# Try to create pod 4 — this should FAIL
kubectl run quota-test-4 --image=nginx -n lab-limits
```

**Breaking down `kubectl run`:**
- `run` → quick way to create a Pod (without writing YAML)
- `quota-test-1` → pod name
- `--image=nginx` → use nginx image
- `-n lab-limits` → in the lab-limits namespace

**Output on pod 4:**
```
Error from server (Forbidden): pods "quota-test-4" is forbidden: 
exceeded quota: pod-quota, requested: pods=1, used: pods=3, limited: pods=3
```

**This is the ResourceQuota validating controller rejecting your request.** The error message tells you exactly what happened: you requested 1 more pod, used=3, limited=3. Quota exceeded.

**Step 3: Check quota status**

```bash
kubectl describe resourcequota pod-quota -n lab-limits
```

**Output:**
```
Name:            pod-quota
Namespace:       lab-limits
Resource         Used   Hard
--------         ----   ----
pods             3      3
requests.cpu     300m   500m
requests.memory  384Mi  512Mi
```

- `Used` → current consumption
- `Hard` → the limit you set
- You can see exactly how much is used vs available

---

### LAB 4 — See a Webhook Configuration Object

```bash
# List all mutating webhook configurations in the cluster
kubectl get mutatingwebhookconfigurations
```

**Breaking down:**
- `get mutatingwebhookconfigurations` → list all MutatingWebhookConfiguration objects in the cluster
- These are cluster-scoped (not namespaced) — no `-n` needed

**Output (on a cluster with Istio):**
```
NAME                                      WEBHOOKS   AGE
istio-sidecar-injector                    1          5d
pod-mutation-webhook-cfg                  1          2d
```

```bash
# List all validating webhook configurations
kubectl get validatingwebhookconfigurations
```

**Output (on a cluster with OPA Gatekeeper):**
```
NAME                                            WEBHOOKS   AGE
gatekeeper-validating-webhook-configuration     1          5d
```

```bash
# See the full details of a mutating webhook
kubectl describe mutatingwebhookconfiguration istio-sidecar-injector
```

**Output:** (abridged)
```
Name:         istio-sidecar-injector
Webhooks:
  Name:      rev.namespace.sidecar-injector.istio.io
  Client Config:
    CA Bundle:  <ca cert data>
    Service:
      Name:       istiod
      Namespace:  istio-system
      Path:       /inject
      Port:       443
  Rules:
    Operations:  CREATE
    Resources:   pods
  Failure Policy:  Fail
  Timeout Seconds: 10
  Namespace Selector:
    Match Labels:
      istio-injection=enabled
```

---

### LAB 5 — Build a Simple Validating Webhook (Conceptual Walkthrough)

> This lab walks through the architecture of building a webhook. Understanding this will help you answer "how would you implement a custom policy?"

**What we want to build:**
A webhook that REJECTS any Pod that uses the `latest` tag for its image. (Using `latest` is bad practice — you can't tell what version you're running.)

**The webhook server code logic (Python pseudocode):**

```python
from flask import Flask, request, jsonify
import base64
import json

app = Flask(__name__)

@app.route('/validate', methods=['POST'])
def validate():
    # Step 1: Parse the AdmissionReview request
    admission_review = request.get_json()
    pod = admission_review['request']['object']
    
    # Step 2: Check all containers in the pod
    for container in pod['spec']['containers']:
        image = container['image']
        
        # Step 3: If image ends with :latest or has no tag → reject
        if ':latest' in image or ':' not in image:
            return jsonify({
                "apiVersion": "admission.k8s.io/v1",
                "kind": "AdmissionReview",
                "response": {
                    "uid": admission_review['request']['uid'],
                    "allowed": False,
                    "status": {
                        "code": 403,
                        "message": f"Image '{image}' uses 'latest' tag. Specify an explicit version tag."
                    }
                }
            })
    
    # Step 4: All images are pinned versions → allow
    return jsonify({
        "apiVersion": "admission.k8s.io/v1",
        "kind": "AdmissionReview",
        "response": {
            "uid": admission_review['request']['uid'],
            "allowed": True
        }
    })
```

**The ValidatingWebhookConfiguration for this:**

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: no-latest-tag
webhooks:
- name: no-latest-tag.company.com
  clientConfig:
    service:
      name: image-validator-svc
      namespace: webhook-system
      path: "/validate"
    caBundle: <base64-CA>
  rules:
  - apiGroups: [""]
    apiVersions: ["v1"]
    operations: ["CREATE", "UPDATE"]
    resources: ["pods"]
  failurePolicy: Fail
  admissionReviewVersions: ["v1"]
  sideEffects: None
```

**Test: what happens when you try `image: nginx:latest`?**

```bash
kubectl run test-latest --image=nginx:latest
# Error: Image 'nginx:latest' uses 'latest' tag. Specify an explicit version tag.

kubectl run test-pinned --image=nginx:1.25.3
# pod/test-pinned created  ← ALLOWED because it has a specific version
```

---

### LAB 6 — See the Complete Mutation Trail on a Pod

```bash
# Create a pod and inspect ALL the fields that were auto-added
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: mutation-trail
  namespace: lab-limits
spec:
  containers:
  - name: app
    image: nginx:1.25
EOF
```

**Now look at the ACTUAL pod object stored in Kubernetes — much bigger than what you wrote:**

```bash
kubectl get pod mutation-trail -n lab-limits -o yaml
```

**Breaking down:**
- `get pod mutation-trail` → get this specific pod
- `-o yaml` → show the FULL YAML (including all auto-added fields)

**Compare what you WROTE vs what's ACTUALLY stored:**

```
YOU WROTE:                    KUBERNETES ADDED (via mutation):
-----------                   --------------------------------
metadata:                     metadata:
  name: mutation-trail          name: mutation-trail
                                namespace: lab-limits
                                resourceVersion: "12345"    ← auto
                                uid: "abc-def-123"          ← auto
                                creationTimestamp: "..."    ← auto

spec:                         spec:
  containers:                   containers:
  - name: app                   - name: app
    image: nginx:1.25             image: nginx:1.25
                                  resources:               ← ADDED by LimitRanger
                                    limits:
                                      cpu: "200m"
                                      memory: "256Mi"
                                    requests:
                                      cpu: "100m"
                                      memory: "128Mi"
                                  terminationMessagePath: /dev/termination-log  ← auto
                                  terminationMessagePolicy: File                ← auto
                                  imagePullPolicy: IfNotPresent                 ← auto
                                
                                dnsPolicy: ClusterFirst          ← auto
                                restartPolicy: Always            ← auto
                                schedulerName: default-scheduler ← auto
                                serviceAccountName: default      ← auto (ServiceAccount controller)
                                tolerations:                     ← auto
                                - effect: NoExecute
                                  key: node.kubernetes.io/not-ready
                                  ...
```

**Everything under "KUBERNETES ADDED" was injected by admission controllers and defaulting logic.** You wrote 9 lines. Kubernetes stored 50+ lines.

---

### LAB 7 — Dry Run to Test Without Actually Creating

```bash
# Test if your pod would be accepted WITHOUT creating it
kubectl apply -f pod.yaml --dry-run=server
```

**Breaking down:**
- `--dry-run=server` → send the request to the API Server and run it through ALL steps (authentication, authorization, ALL admission webhooks, validation) — but DON'T write to etcd at the end.
- This is different from `--dry-run=client` which only validates YAML syntax locally without talking to the server.
- `server` dry-run is what you want — it tests the full pipeline including webhooks.

**Why is this useful?**
- Test if your YAML will be accepted by OPA policies BEFORE applying
- Check if resource quota allows the new object
- Verify webhook mutations would apply

```bash
# Server dry run — full pipeline test
kubectl apply -f mypod.yaml --dry-run=server

# If accepted:
pod/mypod configured (server dry run)

# If rejected:
Error from server (Forbidden): pods "mypod" is forbidden: 
exceeded quota: pod-quota, requested: pods=1, used: 3, limited: 3
```

---

## 📊 SECTION 11 — COMPARISON TABLE

| Feature | Mutating Webhook | Validating Webhook | Built-in Controller |
|---------|-----------------|-------------------|---------------------|
| Can modify the object | ✅ YES | ❌ NO | Depends |
| Can reject the object | Usually no | ✅ YES | ✅ YES |
| Runs when | Step 3 (before validation) | Step 5 (after mutation) | Steps 3 or 5 |
| Implementation | External HTTPS server | External HTTPS server | Inside API Server binary |
| Examples | Istio sidecar injection, LimitRanger defaults | OPA Gatekeeper, ResourceQuota | LimitRanger, ResourceQuota, NamespaceLifecycle |
| Config object | MutatingWebhookConfiguration | ValidatingWebhookConfiguration | `--enable-admission-plugins` flag |
| failurePolicy options | Ignore / Fail | Ignore / Fail | N/A |
| JSON Patch response | ✅ YES | ❌ NO | N/A |

---

## 🔑 SECTION 12 — KEY TERMS TO REMEMBER

| Term | Simple Meaning |
|------|---------------|
| **Admission Controller** | A gatekeeper between API request and etcd write |
| **Mutating Webhook** | External server that can MODIFY the object before saving |
| **Validating Webhook** | External server that can ALLOW or DENY but not modify |
| **AdmissionReview** | The JSON object API Server sends to webhooks (request + response) |
| **JSON Patch** | The format mutating webhooks use to specify their changes |
| **MutatingWebhookConfiguration** | K8s object that registers a mutating webhook |
| **ValidatingWebhookConfiguration** | K8s object that registers a validating webhook |
| **failurePolicy: Ignore** | If webhook is down → allow the request through |
| **failurePolicy: Fail** | If webhook is down → block the request |
| **caBundle** | CA cert in webhook config that API Server uses to verify webhook identity |
| **namespaceSelector** | Filter in webhook config — only call this webhook for certain namespaces |
| **LimitRanger** | Built-in mutating controller that adds default resource limits |
| **ResourceQuota** | Built-in validating controller that enforces namespace resource limits |
| **OPA Gatekeeper** | Popular external policy engine that works as validating webhook |
| **Kyverno** | Alternative to OPA — Kubernetes-native policy engine via webhooks |
| **dry-run=server** | Test the full admission pipeline without writing to etcd |
| **sideEffects: None** | Webhook has no external side effects (required for dry-run to work) |

---

*File: K8s_AdmissionControllers_Concept_and_Lab.md*
*Repository: Interview_Preparation_2026 → Kubernetes/*
*Next: K8s_AdmissionControllers_Interview_Questions.md*
