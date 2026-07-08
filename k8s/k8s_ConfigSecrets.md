# Kubernetes Config & Secrets — Deep Dive Concept + Hands-On Lab
## ConfigMap · Secret · Env Injection · Volume Mount · Sealed Secrets · External Secrets
> Written for: Someone with 4 years of DevOps experience preparing for senior-level interviews
> Style: First-standard student explanation → deep technical truth → hands-on lab with line-by-line explanation

---

## 🧠 SECTION 1 — WHY DOES KUBERNETES NEED CONFIG MANAGEMENT? (Story First)

### Forget Kubernetes for 3 minutes — start with a story

Imagine you are a chef who works at three different restaurants — **Dev Kitchen**, **Staging Kitchen**, and **Production Kitchen**.

You cook the SAME recipe everywhere. The dish is the same. But some ingredients change:

```
Dev Kitchen:       use cheap olive oil, small portions, no salt (testing taste)
Staging Kitchen:   use medium olive oil, normal portions, half salt
Production Kitchen: use premium olive oil, full portions, full salt
```

Now the WRONG way to handle this:

> You write the olive oil brand, portion size, and salt amount **inside the recipe card itself**.

Problem: you need THREE DIFFERENT RECIPE CARDS now. If the recipe changes (new cooking technique), you update THREE cards. You forget one. Inconsistency. Bugs.

The RIGHT way:

> The recipe card says **"check the kitchen's configuration board"** for olive oil brand, portion size, and salt. Each kitchen has its own configuration board with different values. The recipe card is IDENTICAL everywhere. Only the board changes.

**This is exactly what ConfigMap and Secret do in Kubernetes.**

```
RECIPE CARD  = your container image (same in dev/staging/prod)
CONFIG BOARD = ConfigMap (non-sensitive configuration)
SECRET SAFE  = Secret (passwords, API keys, certificates)

Your app container reads from the config board and secret safe at runtime.
The IMAGE never changes between environments.
Only the ConfigMap and Secret change.
```

### The Core Principle

```
ANTI-PATTERN (wrong):
  Bake config into the image
  → Different image for dev vs prod
  → Config visible in Dockerfile
  → Passwords in source code
  → Must rebuild image to change config

CORRECT PATTERN:
  Separate config from code
  → ONE image for all environments
  → Config injected at runtime
  → Secrets managed separately
  → Change config without rebuilding image
```

---

## 📋 SECTION 2 — ConfigMap (Non-Sensitive Configuration)

### What is a ConfigMap?

A **ConfigMap** stores **non-sensitive configuration data** as key-value pairs. Think of it as a **settings file for your application** that lives inside Kubernetes and can be injected into pods at runtime.

### What Goes in a ConfigMap?

```
✅ USE ConfigMap for:
  Database hostname:     DB_HOST=mysql.banking.svc.cluster.local
  Application port:      APP_PORT=8080
  Log level:             LOG_LEVEL=INFO
  Feature flags:         FEATURE_DARK_MODE=true
  Environment name:      ENVIRONMENT=production
  Cache TTL:             CACHE_TTL_SECONDS=300
  Config files:          nginx.conf, application.properties, prometheus.yml

❌ DO NOT use ConfigMap for:
  Database passwords     → use Secret
  API keys               → use Secret
  TLS certificates       → use Secret
  Any sensitive value    → use Secret
```

### Three Ways to Create a ConfigMap

**Method 1: From literal key-value pairs**

```bash
kubectl create configmap app-config \
  --from-literal=DB_HOST=mysql.banking.svc \
  --from-literal=DB_PORT=3306 \
  --from-literal=LOG_LEVEL=INFO \
  --from-literal=APP_PORT=8080
```

**Method 2: From a file**

```bash
# First create a config file
cat > app.properties <<EOF
database.host=mysql.banking.svc
database.port=3306
log.level=INFO
cache.ttl=300
EOF

# Create ConfigMap from file
kubectl create configmap app-config --from-file=app.properties
# Key = filename (app.properties), Value = entire file contents
```

**Method 3: From YAML (most common in production)**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: banking
data:
  # Simple key-value pairs
  DB_HOST: "mysql.banking.svc.cluster.local"
  DB_PORT: "3306"
  LOG_LEVEL: "INFO"
  APP_PORT: "8080"
  FEATURE_PAYMENTS: "true"

  # Multi-line value (a whole config file as a value)
  nginx.conf: |
    server {
      listen 80;
      server_name bank.example.com;
      location / {
        proxy_pass http://payment-svc:8080;
      }
    }

  application.properties: |
    spring.datasource.url=jdbc:postgresql://mysql.banking.svc:5432/bankdb
    spring.jpa.hibernate.ddl-auto=validate
    logging.level.root=INFO
```

**Every line explained:**

```yaml
apiVersion: v1
```
→ ConfigMap is in the core API group (v1) — same as Pods, Services, Namespaces.

```yaml
kind: ConfigMap
```
→ We are creating a ConfigMap object.

```yaml
  name: app-config
  namespace: banking
```
→ ConfigMaps are NAMESPACED — they only exist in the namespace they're created in.
→ A pod in namespace `banking` can only use ConfigMaps in `banking`.

```yaml
data:
  DB_HOST: "mysql.banking.svc.cluster.local"
  DB_PORT: "3306"
```
→ `data` → the key-value store. Every key is a string. Every value is a string.
→ Note: even numbers must be strings — `"3306"` not `3306`. All ConfigMap values are strings.

```yaml
  nginx.conf: |
    server {
```
→ `|` (pipe symbol in YAML) → multi-line string literal. Everything indented below it is one value.
→ The key is `nginx.conf`. The value is the entire nginx configuration text.
→ This is how you store a whole config file as a ConfigMap value.

---

## 🔐 SECTION 3 — Secret (Sensitive Data)

### What is a Secret?

A **Secret** stores **sensitive data** — passwords, API keys, tokens, TLS certificates. It looks similar to ConfigMap but has:
1. Values are **base64 encoded** (not encrypted by default — important!)
2. Kubernetes treats them with more care — doesn't log them, doesn't show them in normal describe output
3. Can be encrypted at rest with additional cluster configuration

### ⚠️ THE MOST IMPORTANT TRUTH ABOUT SECRETS

```
MYTH:    "Secrets are encrypted"
REALITY: "Secrets are BASE64 ENCODED — NOT encrypted by default"

Base64 is ENCODING, not ENCRYPTION.
Encoding = change format for safe transport (easily reversible)
Encryption = scramble with a key (cannot reverse without key)

Anyone who can:
  kubectl get secret db-secret -o yaml
  → sees the base64 encoded values
  → runs: echo "YWRtaW4=" | base64 -d
  → gets: admin
  → password exposed in 2 seconds

The REAL protection comes from:
  1. RBAC (control who can kubectl get secrets)
  2. etcd encryption at rest (encrypt before storing to disk)
  3. External secret management (AWS Secrets Manager, Vault)
```

### Three Types of Secrets

```
Opaque (generic):
  → Most common type. Arbitrary key-value pairs.
  → Example: database credentials, API keys

kubernetes.io/tls:
  → TLS certificate and private key
  → Used by Ingress for HTTPS
  → Keys must be: tls.crt and tls.key

kubernetes.io/dockerconfigjson:
  → Docker registry credentials
  → Used for pulling images from private registries
  → Referenced in pod spec as imagePullSecrets
```

### Three Ways to Create a Secret

**Method 1: From literals (kubectl encodes automatically)**

```bash
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password=SuperSecretPass123 \
  --from-literal=db-name=bankingdb
```
→ kubectl automatically base64 encodes the values

**Method 2: TLS secret from cert files**

```bash
kubectl create secret tls bank-tls \
  --cert=server.crt \
  --key=server.key
```

**Method 3: Docker registry secret**

```bash
kubectl create secret docker-registry ecr-credentials \
  --docker-server=123456789.dkr.ecr.us-east-1.amazonaws.com \
  --docker-username=AWS \
  --docker-password=$(aws ecr get-login-password --region us-east-1)
```

### Secret YAML — What It Actually Looks Like

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: banking
type: Opaque
data:
  username: YWRtaW4=          # base64 of "admin"
  password: U3VwZXJTZWNyZXRQYXNzMTIz   # base64 of "SuperSecretPass123"
  db-name: YmFua2luZ2Ri       # base64 of "bankingdb"
```

**Explaining base64:**

```yaml
type: Opaque
```
→ `type` → what kind of Secret this is.
→ `Opaque` = generic/arbitrary data. This is the default.
→ Other types: `kubernetes.io/tls`, `kubernetes.io/dockerconfigjson`

```yaml
data:
  username: YWRtaW4=
```
→ `data` → values MUST be base64 encoded when writing YAML directly.
→ `YWRtaW4=` → base64 encoding of the string `admin`
→ To encode: `echo -n 'admin' | base64` → `YWRtaW4=`
→ The `-n` flag on echo is CRITICAL — without it, echo adds a newline character which gets encoded too, giving you wrong decoded value.

### Alternative: stringData (Plain Text in YAML)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: banking
type: Opaque
stringData:
  username: admin                    # plain text — Kubernetes encodes automatically
  password: SuperSecretPass123       # plain text — Kubernetes encodes automatically
  db-name: bankingdb
```

→ `stringData` → write plain text values. Kubernetes base64 encodes them before storing.
→ When you `kubectl get secret -o yaml` you'll see the values in `data` section (encoded).
→ `stringData` is write-only — it never appears in GET output, only `data` does.
→ Use `stringData` when writing YAML by hand to avoid manual base64 encoding.

---

## 💉 SECTION 4 — ENV INJECTION (Injecting Config into Containers)

### Method 1: Inject All ConfigMap Keys as Environment Variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: payment-pod
  namespace: banking
spec:
  containers:
  - name: payment-app
    image: payment-service:1.0
    envFrom:
    - configMapRef:
        name: app-config
    - secretRef:
        name: db-credentials
```

**Every line explained:**

```yaml
    envFrom:
```
→ `envFrom` → inject ALL key-value pairs from a source as environment variables.
→ Every key in the ConfigMap becomes an environment variable name.
→ Every value becomes the environment variable value.

```yaml
    - configMapRef:
        name: app-config
```
→ `configMapRef` → inject ALL keys from this ConfigMap.
→ `name: app-config` → which ConfigMap to inject from.
→ Result: `DB_HOST`, `DB_PORT`, `LOG_LEVEL`, `APP_PORT` all become env vars.

```yaml
    - secretRef:
        name: db-credentials
```
→ `secretRef` → inject ALL keys from this Secret.
→ Result: `username`, `password`, `db-name` all become env vars (decoded automatically).

**Inside the container the app sees:**
```bash
env
DB_HOST=mysql.banking.svc.cluster.local
DB_PORT=3306
LOG_LEVEL=INFO
APP_PORT=8080
username=admin
password=SuperSecretPass123
db-name=bankingdb
```

### Method 2: Inject Specific Keys as Specific Env Var Names

```yaml
spec:
  containers:
  - name: payment-app
    image: payment-service:1.0
    env:
    - name: DATABASE_HOST           # env var name inside container
      valueFrom:
        configMapKeyRef:
          name: app-config          # ConfigMap name
          key: DB_HOST              # which key from ConfigMap
    - name: DATABASE_PORT
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: DB_PORT
    - name: DB_PASSWORD             # env var name inside container
      valueFrom:
        secretKeyRef:
          name: db-credentials      # Secret name
          key: password             # which key from Secret
    - name: STATIC_VALUE            # plain literal value (no ConfigMap/Secret)
      value: "hello-world"
```

**Every line explained:**

```yaml
    - name: DATABASE_HOST
```
→ `name` → the environment variable NAME as the container will see it. You control this — it doesn't have to match the ConfigMap key.

```yaml
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: DB_HOST
```
→ `valueFrom` → value comes from an external source (not hardcoded).
→ `configMapKeyRef` → source is a ConfigMap key.
→ `name: app-config` → which ConfigMap.
→ `key: DB_HOST` → which specific key in that ConfigMap.
→ Result: env var `DATABASE_HOST` = value of ConfigMap key `DB_HOST`.

```yaml
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
```
→ `secretKeyRef` → source is a Secret key.
→ The Secret value is automatically base64 DECODED before being set as env var.
→ Container sees plain text value, not base64.

### ⚠️ Critical Limitation of Env Injection

```
PROBLEM WITH ENV INJECTION:
  Env vars are set ONCE at pod startup.
  If the ConfigMap or Secret changes AFTER the pod starts:
    → The running pod does NOT see the new values
    → The env var still has the OLD value
    → You MUST RESTART the pod to pick up changes

This is a major operational issue.
Imagine changing a feature flag in ConfigMap.
All existing pods still have the old value.
You must rolling-restart the deployment.

SOLUTION: Volume mounts (next section) — files update without restart.
```

---

## 📁 SECTION 5 — VOLUME MOUNT (Injecting Config as Files)

### Why Volume Mounts?

Instead of injecting ConfigMap/Secret values as environment variables, you can mount them as **files inside the container**. The container sees a directory with files — one file per key.

### Advantages of Volume Mounts over Env Vars

```
ENV VARS:                          VOLUME MOUNTS:
Set once at startup                Updated automatically (~60 seconds)
No restart needed to set           Restart NOT needed for update
Simple key-value only              Can be entire config files
Good for simple settings           Good for config files (nginx.conf, etc.)
                                   App must watch/reload file for changes
```

### ConfigMap as Volume Mount

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  namespace: banking
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    volumeMounts:
    - name: nginx-config-volume
      mountPath: /etc/nginx/conf.d    # directory inside container
      readOnly: true
  volumes:
  - name: nginx-config-volume
    configMap:
      name: nginx-configmap           # which ConfigMap to mount
```

**Every line explained:**

```yaml
    volumeMounts:
    - name: nginx-config-volume
      mountPath: /etc/nginx/conf.d
```
→ `volumeMounts` → list of volumes to mount into THIS container.
→ `name: nginx-config-volume` → references the volume defined in `volumes:` section.
→ `mountPath: /etc/nginx/conf.d` → WHERE inside the container to mount. Each key in the ConfigMap becomes a file in this directory.

```yaml
      readOnly: true
```
→ Container cannot write to this directory. Good practice — app should not modify its own config.

```yaml
  volumes:
  - name: nginx-config-volume
    configMap:
      name: nginx-configmap
```
→ `volumes:` → define the volumes available to this pod (referenced by containers via volumeMounts).
→ `configMap: name: nginx-configmap` → use a ConfigMap as the volume source.
→ Each key in `nginx-configmap` becomes a file. Key = filename, Value = file contents.

**What the container sees:**
```bash
ls /etc/nginx/conf.d/
nginx.conf          ← from ConfigMap key "nginx.conf"
server.conf         ← from ConfigMap key "server.conf"

cat /etc/nginx/conf.d/nginx.conf
server {
  listen 80;
  ...               ← full content of that ConfigMap value
}
```

### Mount Specific Keys (Not All Keys)

```yaml
  volumes:
  - name: config-volume
    configMap:
      name: app-config
      items:                          # select SPECIFIC keys only
      - key: nginx.conf               # which key from ConfigMap
        path: nginx.conf              # filename inside the container directory
      - key: server.conf
        path: server.conf
```

→ `items:` → instead of mounting ALL keys, select specific ones.
→ `key: nginx.conf` → which ConfigMap key to use.
→ `path: nginx.conf` → what filename to create inside the mountPath directory.
→ Use this when you only want some ConfigMap keys as files, not all.

### Mount a Single Key as a Single File

```yaml
    volumeMounts:
    - name: app-config-volume
      mountPath: /etc/app/config.properties   # a SPECIFIC FILE path (not directory)
      subPath: application.properties          # which key to mount at that path
```

```yaml
  volumes:
  - name: app-config-volume
    configMap:
      name: app-config
```

→ `subPath: application.properties` → mount ONLY this key from the ConfigMap.
→ `mountPath: /etc/app/config.properties` → mount it at this EXACT FILE PATH (not a directory).
→ Without subPath: the whole ConfigMap is mounted as a DIRECTORY. With subPath: one key is mounted as a single FILE.
→ **⚠️ Important limitation:** SubPath mounts do NOT get automatic updates when ConfigMap changes. Only directory mounts (without subPath) update automatically.

### Secret as Volume Mount

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
  namespace: banking
spec:
  containers:
  - name: app
    image: myapp:1.0
    volumeMounts:
    - name: db-secret-volume
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: db-secret-volume
    secret:
      secretName: db-credentials     # reference Secret by name
      defaultMode: 0400              # file permissions
```

**New lines explained:**

```yaml
    secret:
      secretName: db-credentials
```
→ `secret` → use a Secret as the volume source.
→ `secretName: db-credentials` → which Secret.
→ Each key in the Secret becomes a file in /etc/secrets/. Values are base64 DECODED automatically.

```yaml
      defaultMode: 0400
```
→ `defaultMode` → file permission in octal format.
→ `0400` → owner can read only. Nobody else can read, write, or execute.
→ This restricts the secret files so only the process owner can read them.
→ Good security practice for secret files.

**What the container sees:**
```bash
ls -la /etc/secrets/
-r-------- username    ← contains "admin" (decoded)
-r-------- password    ← contains "SuperSecretPass123" (decoded)
-r-------- db-name     ← contains "bankingdb" (decoded)

cat /etc/secrets/password
SuperSecretPass123
```

### How Auto-Update Works for Volume Mounts

```
ConfigMap/Secret updated (kubectl apply or kubectl edit)
         │
         ▼ (within ~60 seconds — kubelet sync interval)
kubelet detects ConfigMap/Secret changed
         │
         ▼
kubelet updates the files inside the mounted directory
         │
         ▼
New file contents are available immediately in /etc/nginx/conf.d/

BUT:
The application might still be using the OLD config in memory.
The app must WATCH the file and RELOAD when it changes.
nginx: nginx -s reload
Java Spring: @RefreshScope with actuator
Custom: inotify watch on config file
```

---

## 📊 SECTION 6 — ENV INJECTION VS VOLUME MOUNT COMPARISON

```
┌──────────────────────┬─────────────────────────┬───────────────────────────┐
│ Feature              │ Environment Variable     │ Volume Mount              │
├──────────────────────┼─────────────────────────┼───────────────────────────┤
│ How accessed         │ $DB_HOST in shell/code   │ Read file at /etc/config  │
│ Set when             │ Pod STARTUP only         │ Pod startup + updates     │
│ Updates after start  │ ❌ NO — must restart     │ ✅ YES — ~60 sec sync     │
│ Multi-line values    │ Awkward                  │ Natural (file contents)   │
│ Config files         │ ❌ Not suitable          │ ✅ Perfect                │
│ Security (Secrets)   │ Risk: process dump shows │ Slightly better (file)    │
│ App code change?     │ Read env var (built-in)  │ Read file (code needed)   │
│ Best for             │ Simple key-value config  │ Config files, auto-update │
└──────────────────────┴─────────────────────────┴───────────────────────────┘
```

---

## 🔒 SECTION 7 — SEALED SECRETS (Encrypt Secrets for Git)

### The Problem with Storing Secrets in Git

```
SCENARIO:
  You have a deployment YAML in Git. ✅ Good.
  You have a Service YAML in Git. ✅ Good.
  You have a Secret YAML in Git. ❌ DANGEROUS.

WHY?
  Secret YAML contains base64 values.
  Base64 is NOT encryption.
  Anyone with Git access → sees the "encoded" values → decodes in 2 seconds.
  Your passwords are in plain text in your Git history. Forever.

THE RULE: NEVER commit plain Kubernetes Secret YAML to Git.

BUT THEN: How do you manage secrets declaratively? How do you GitOps secrets?
ANSWER: Sealed Secrets or External Secrets
```

### What are Sealed Secrets?

**Sealed Secrets** is a tool by Bitnami that allows you to **encrypt a Kubernetes Secret so it can be safely committed to Git**. Only the Sealed Secrets controller running in the cluster can decrypt it.

```
SEALED SECRETS FLOW:

1. You have a plain Secret YAML
         │
         ▼
2. kubeseal tool encrypts it using the cluster's PUBLIC KEY
   → Creates a SealedSecret YAML (safely committable to Git)
         │
         ▼
3. You commit SealedSecret YAML to Git ✅ SAFE
         │
         ▼
4. ArgoCD/Flux sees new SealedSecret in Git → applies it to cluster
         │
         ▼
5. Sealed Secrets Controller (running in cluster) sees new SealedSecret
   → Decrypts using PRIVATE KEY (only the controller has this)
   → Creates a regular Kubernetes Secret in the namespace
         │
         ▼
6. Pod reads the regular Kubernetes Secret normally
```

### Components of Sealed Secrets

```
SERVER SIDE (runs in Kubernetes):
  sealed-secrets-controller (Deployment in kube-system)
  → Holds the private key
  → Watches for SealedSecret objects
  → Decrypts them → creates regular Secrets

CLIENT SIDE (on developer's machine):
  kubeseal (CLI tool)
  → Fetches the public key from the controller
  → Encrypts your Secret YAML into a SealedSecret YAML
```

### How to Use Sealed Secrets

**Step 1: Install Sealed Secrets controller**

```bash
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm install sealed-secrets sealed-secrets/sealed-secrets \
  --namespace kube-system \
  --set-string fullnameOverride=sealed-secrets-controller
```

**Step 2: Install kubeseal CLI**

```bash
# On Linux
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/kubeseal-0.24.0-linux-amd64.tar.gz
tar -xzf kubeseal-0.24.0-linux-amd64.tar.gz
sudo install -m 755 kubeseal /usr/local/bin/kubeseal
```

**Step 3: Create a regular Secret YAML (DO NOT commit this)**

```bash
kubectl create secret generic db-creds \
  --from-literal=username=admin \
  --from-literal=password=SuperSecretPass123 \
  --dry-run=client \
  -o yaml > /tmp/secret.yaml
# This is the plain secret — NOT for Git
```

**Step 4: Seal it (safe to commit)**

```bash
kubeseal \
  --controller-name=sealed-secrets-controller \
  --controller-namespace=kube-system \
  --format yaml \
  < /tmp/secret.yaml \
  > sealed-secret.yaml
# This is the SEALED secret — SAFE for Git ✅
```

**Breaking down kubeseal:**
- `--controller-name=sealed-secrets-controller` → name of the controller pod
- `--controller-namespace=kube-system` → where the controller runs
- `--format yaml` → output format (yaml or json)
- `< /tmp/secret.yaml` → read input from this file
- `> sealed-secret.yaml` → write encrypted output to this file

**What sealed-secret.yaml looks like:**

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: db-creds
  namespace: banking
spec:
  encryptedData:
    username: AgBy8hpCGMzUjAo3n4hJyO...  ← encrypted (not just base64!)
    password: AgBvQs8N7KzSJnFlwp3mXt...  ← encrypted with cluster public key
  template:
    metadata:
      name: db-creds
      namespace: banking
    type: Opaque
```

→ `encryptedData` → values are ACTUALLY ENCRYPTED using RSA asymmetric encryption.
→ Only the Sealed Secrets controller (which has the private key) can decrypt this.
→ Even if someone gets your Git repository → they cannot decrypt these values without cluster access.

**Step 5: Commit sealed-secret.yaml to Git → apply to cluster**

```bash
git add sealed-secret.yaml
git commit -m "Add encrypted DB credentials"
git push

# ArgoCD applies it → controller decrypts → creates regular Secret
```

### Sealed Secrets Scope (Important Security Feature)

```bash
# Cluster-scoped seal (any namespace can decrypt)
kubeseal --scope cluster-wide ...

# Namespace-scoped seal (only specified namespace can decrypt)
kubeseal --scope namespace-wide ...

# Strict scope (only specific name+namespace can decrypt — DEFAULT)
kubeseal --scope strict ...
```

→ Default is strict — the SealedSecret can only create a Secret with the SAME name in the SAME namespace.
→ This prevents someone from copying your SealedSecret YAML and deploying it in a different namespace to steal your secrets.

---

## ☁️ SECTION 8 — EXTERNAL SECRETS OPERATOR (ESO)

### The Problem with Both Plain Secrets AND Sealed Secrets

```
PLAIN SECRETS in Kubernetes:
  ❌ Not really encrypted (base64)
  ❌ Cannot safely go to Git
  ❌ No rotation — manual process to rotate
  ❌ No audit trail — who accessed the secret?
  ❌ Secrets spread across many clusters are hard to manage

SEALED SECRETS:
  ✅ Safe for Git
  ❌ Still stores encrypted secret IN the cluster etcd
  ❌ No central rotation
  ❌ If cluster is compromised, all sealed secrets are at risk
  ❌ Audit trail limited

ENTERPRISE REQUIREMENT (especially banking):
  ✅ Centralized secret management
  ✅ Automatic rotation
  ✅ Audit logging (who accessed what and when)
  ✅ Integration with compliance tools
  ✅ Secret lives in a secure vault, NOT in the cluster
```

### What is External Secrets Operator?

**External Secrets Operator (ESO)** is a Kubernetes operator that:
1. Reads secrets from an **external secret store** (AWS Secrets Manager, HashiCorp Vault, GCP Secret Manager, Azure Key Vault)
2. Creates and keeps in sync a regular Kubernetes Secret in the cluster

The actual secret value **lives in AWS/Vault/GCP** — never written to Git. ESO just tells Kubernetes "go fetch this secret from AWS Secrets Manager and create a Kubernetes Secret from it."

### ESO Architecture

```
AWS SECRETS MANAGER (the truth — where secrets actually live)
  └── secret: /banking/production/db-credentials
      └── value: {"username":"admin","password":"SuperSecret"}
           │
           │ ESO syncs every 1 hour (or on change)
           ▼
KUBERNETES CLUSTER
  └── ExternalSecret object (defines WHAT to sync from WHERE)
           │
           ▼ ESO Controller reads ExternalSecret
           │ → calls AWS Secrets Manager API
           │ → gets the value
           │ → creates/updates Kubernetes Secret
           ▼
  └── Kubernetes Secret: db-credentials
      └── username: admin (decoded, ready for pod use)
      └── password: SuperSecret
           │
           ▼
POD uses it normally via env injection or volume mount
```

### ESO Components

```
SecretStore (or ClusterSecretStore):
  → Defines HOW to connect to the external secret system
  → Contains credentials/config for AWS, Vault, GCP, etc.
  → SecretStore = namespaced (one per namespace)
  → ClusterSecretStore = cluster-wide (share across namespaces)

ExternalSecret:
  → Defines WHAT secrets to pull and WHERE to put them
  → References a SecretStore
  → Maps external secret keys to Kubernetes Secret keys
  → Defines refresh interval (how often to re-sync)
```

### AWS Secrets Manager Setup Example

**Step 1: Install ESO**

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets \
  --namespace external-secrets \
  --create-namespace
```

**Step 2: Create a SecretStore (how to connect to AWS)**

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secrets-manager
  namespace: banking
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa   # SA with IAM role via IRSA
```

**Every line explained:**

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
```
→ ESO adds its own CRDs (Custom Resource Definitions) to Kubernetes.
→ SecretStore is one of those custom objects.

```yaml
  provider:
    aws:
      service: SecretsManager
```
→ `provider` → which external system to connect to.
→ `aws.service: SecretsManager` → use AWS Secrets Manager specifically.
→ Other options: `ParameterStore` (AWS SSM), `Vault` (HashiCorp), `gcpsm` (GCP), `azurekv` (Azure).

```yaml
      region: us-east-1
```
→ Which AWS region your secrets are stored in.

```yaml
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa
```
→ `auth` → how ESO authenticates to AWS.
→ `jwt` → use IRSA (IAM Roles for Service Accounts) — the AWS-recommended way.
→ The ServiceAccount `external-secrets-sa` must have an IAM role annotation with permission to read from Secrets Manager.

**Step 3: Create an ExternalSecret (what to sync)**

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials-external
  namespace: banking
spec:
  refreshInterval: 1h              # sync every 1 hour
  secretStoreRef:
    name: aws-secrets-manager      # which SecretStore to use
    kind: SecretStore
  target:
    name: db-credentials           # name of Kubernetes Secret to create
    creationPolicy: Owner          # ESO owns this secret (deletes when ES deleted)
  data:
  - secretKey: username            # key in Kubernetes Secret
    remoteRef:
      key: /banking/production/db  # path in AWS Secrets Manager
      property: username           # which property inside the JSON secret
  - secretKey: password
    remoteRef:
      key: /banking/production/db
      property: password
```

**Every line explained:**

```yaml
  refreshInterval: 1h
```
→ How often ESO re-reads the external secret and updates the Kubernetes Secret.
→ `1h` = every 1 hour. Options: `30s`, `5m`, `1h`, `24h`.
→ If you rotate the secret in AWS Secrets Manager, the Kubernetes Secret is updated within this interval.

```yaml
  secretStoreRef:
    name: aws-secrets-manager
    kind: SecretStore
```
→ Which SecretStore to use for authentication and connection details.

```yaml
  target:
    name: db-credentials
    creationPolicy: Owner
```
→ `name: db-credentials` → name of the Kubernetes Secret to CREATE.
→ `creationPolicy: Owner` → ESO creates and OWNS this secret. If the ExternalSecret is deleted → the Kubernetes Secret is also deleted. Use `Merge` to not delete the secret if ES is removed.

```yaml
  data:
  - secretKey: username
    remoteRef:
      key: /banking/production/db
      property: username
```
→ `data` → list of secret mappings.
→ `secretKey: username` → the KEY in the Kubernetes Secret that will be created.
→ `remoteRef.key: /banking/production/db` → the PATH of the secret in AWS Secrets Manager.
→ `remoteRef.property: username` → the JSON field inside the Secrets Manager value.

If the AWS Secrets Manager secret at `/banking/production/db` contains:
```json
{"username": "admin", "password": "SuperSecret123", "host": "prod-db.banking.com"}
```
→ ESO creates a Kubernetes Secret `db-credentials` with keys `username=admin` and `password=SuperSecret123`.

**Step 4: Verify**

```bash
kubectl get externalsecret db-credentials-external -n banking
# STATUS should be: SecretSynced

kubectl get secret db-credentials -n banking
# Regular Kubernetes Secret created by ESO — pods use it normally
```

### ESO vs Sealed Secrets — When to Use Which?

```
SEALED SECRETS:
  ✅ Simple setup — no external dependency
  ✅ Works offline — no cloud account needed
  ✅ Good for small teams / simpler infrastructure
  ✅ Secrets in Git (encrypted) — true GitOps
  ❌ Secrets still stored in etcd (encrypted but in cluster)
  ❌ Manual rotation process
  ❌ No central audit trail
  USE WHEN: small clusters, on-premises, simple GitOps

EXTERNAL SECRETS:
  ✅ Secrets NEVER in Git or Kubernetes etcd — true separation
  ✅ Automatic rotation (rotate in AWS → ESO syncs within refreshInterval)
  ✅ Full audit trail (AWS CloudTrail logs every access)
  ✅ Centralized management (one place for all environments' secrets)
  ✅ Integration with compliance (SOC2, PCI-DSS, HIPAA)
  ❌ Requires external service (AWS, Vault, GCP)
  ❌ More complex setup
  ❌ ESO must be running for secrets to update
  USE WHEN: enterprise, banking, compliance requirements, multi-cluster
```

---

## 💻 SECTION 9 — HANDS-ON LAB

> Every command explained word by word. Every flag explained. Nothing skipped.

---

### LAB 1 — Create ConfigMap and Inject as Env Variables

```bash
# Step 1: Create ConfigMap
kubectl create configmap payment-config \
  --from-literal=DB_HOST=mysql.default.svc.cluster.local \
  --from-literal=DB_PORT=3306 \
  --from-literal=LOG_LEVEL=DEBUG \
  --from-literal=APP_ENV=development
```

```bash
# Step 2: Verify ConfigMap
kubectl get configmap payment-config -o yaml
```

Output:
```yaml
apiVersion: v1
data:
  APP_ENV: development
  DB_HOST: mysql.default.svc.cluster.local
  DB_PORT: "3306"
  LOG_LEVEL: DEBUG
kind: ConfigMap
```

```bash
# Step 3: Create pod that injects ALL ConfigMap keys as env vars
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: config-env-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "env | grep -E 'DB_|LOG_|APP_' && sleep 3600"]
    envFrom:
    - configMapRef:
        name: payment-config
EOF
```

**Explaining the command:**
- `env` → print all environment variables
- `| grep -E 'DB_|LOG_|APP_'` → filter for our config keys specifically
- `-E` → extended regex mode (allows `|` for OR)

```bash
# Step 4: Check logs — see all env vars injected
kubectl logs config-env-pod
```

Expected output:
```
DB_HOST=mysql.default.svc.cluster.local
DB_PORT=3306
LOG_LEVEL=DEBUG
APP_ENV=development
```

```bash
# Step 5: Exec into pod and check manually
kubectl exec -it config-env-pod -- sh
echo $DB_HOST
echo $LOG_LEVEL
exit
```

---

### LAB 2 — Create Secret and Inject as Env Variable

```bash
# Step 1: Create Secret
kubectl create secret generic db-secret \
  --from-literal=username=bankadmin \
  --from-literal=password=BankingPass2024!
```

```bash
# Step 2: Verify — values are base64 encoded in storage
kubectl get secret db-secret -o yaml
```

Output:
```yaml
data:
  password: QmFua2luZ1Bhc3MyMDI0IQ==    ← base64 encoded
  username: YmFua2FkbWlu               ← base64 encoded
```

```bash
# Decode manually to verify
echo "YmFua2FkbWlu" | base64 -d
# Output: bankadmin
```

→ `-d` means decode (reverse base64)

```bash
# Step 3: Create pod with specific env vars from both ConfigMap and Secret
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: secret-env-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "echo DB: $DATABASE_HOST && echo User: $DB_USER && sleep 3600"]
    env:
    - name: DATABASE_HOST
      valueFrom:
        configMapKeyRef:
          name: payment-config
          key: DB_HOST
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
EOF
```

```bash
kubectl logs secret-env-pod
# DB: mysql.default.svc.cluster.local
# User: bankadmin
```

```bash
# Check that password is decoded (not base64) inside container
kubectl exec -it secret-env-pod -- sh -c 'echo $DB_PASS'
# Output: BankingPass2024!   (plain text — decoded automatically)
```

---

### LAB 3 — ConfigMap as Volume Mount (Config File)

```bash
# Step 1: Create ConfigMap with a config file as value
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  default.conf: |
    server {
      listen 80;
      location / {
        return 200 'Hello from configured nginx!';
        add_header Content-Type text/plain;
      }
      location /health {
        return 200 'healthy';
        add_header Content-Type text/plain;
      }
    }
EOF
```

```bash
# Step 2: Create nginx pod with ConfigMap mounted as config file
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx-configured
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    ports:
    - containerPort: 80
    volumeMounts:
    - name: nginx-conf
      mountPath: /etc/nginx/conf.d    # nginx reads *.conf from here
      readOnly: true
  volumes:
  - name: nginx-conf
    configMap:
      name: nginx-config
EOF
```

```bash
# Step 3: Verify the file is there
kubectl exec nginx-configured -- ls /etc/nginx/conf.d/
# Output: default.conf

kubectl exec nginx-configured -- cat /etc/nginx/conf.d/default.conf
# Output: full nginx config we wrote
```

```bash
# Step 4: Test nginx serves our custom response
kubectl port-forward pod/nginx-configured 8080:80 &
curl http://localhost:8080
# Output: Hello from configured nginx!
curl http://localhost:8080/health
# Output: healthy
```

**Explaining port-forward:**
- `kubectl port-forward pod/nginx-configured 8080:80` → create a tunnel: localhost:8080 → pod:80
- `&` → run in background so terminal is free

```bash
# Step 5: UPDATE the ConfigMap and watch the file change WITHOUT pod restart
kubectl patch configmap nginx-config -p \
  '{"data":{"default.conf":"server {\n  listen 80;\n  location / {\n    return 200 '"'"'UPDATED!'"'"';\n    add_header Content-Type text/plain;\n  }\n}"}}'
```

```bash
# Wait 60 seconds for kubelet to sync
sleep 60

# Check the file was updated inside the running pod
kubectl exec nginx-configured -- cat /etc/nginx/conf.d/default.conf
# File content is UPDATED — no pod restart needed
```

---

### LAB 4 — Secret as Volume Mount

```bash
# Step 1: Create pod with Secret mounted as files
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: secret-volume-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "ls /etc/secrets && cat /etc/secrets/username && sleep 3600"]
    volumeMounts:
    - name: db-secret-vol
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: db-secret-vol
    secret:
      secretName: db-secret
      defaultMode: 0400
EOF
```

```bash
# Check logs — app reads secret from file
kubectl logs secret-volume-pod
# username    password   (filenames = Secret keys)
# bankadmin              (content of username file = decoded value)
```

```bash
# Verify file permissions
kubectl exec secret-volume-pod -- ls -la /etc/secrets/
# -r-------- 1 root root 10 Jan 15 username
# -r-------- 1 root root 16 Jan 15 password
# Permissions are 0400 (read-only by owner)
```

---

### LAB 5 — Demonstrate Env Var vs Volume Mount Update Behavior

```bash
# Create a ConfigMap
kubectl create configmap update-test --from-literal=VALUE=original

# Create two pods — one using env var, one using volume mount
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: env-update-test
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "while true; do echo ENV: $MY_VALUE; sleep 10; done"]
    env:
    - name: MY_VALUE
      valueFrom:
        configMapKeyRef:
          name: update-test
          key: VALUE
---
apiVersion: v1
kind: Pod
metadata:
  name: vol-update-test
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "while true; do echo FILE: $(cat /config/VALUE); sleep 10; done"]
    volumeMounts:
    - name: config
      mountPath: /config
  volumes:
  - name: config
    configMap:
      name: update-test
EOF
```

```bash
# See current output from both
kubectl logs env-update-test    # ENV: original
kubectl logs vol-update-test    # FILE: original

# Update the ConfigMap
kubectl patch configmap update-test -p '{"data":{"VALUE":"UPDATED"}}'

# Wait 60 seconds
sleep 60

# Check logs again
kubectl logs env-update-test    # ENV: original  ← STILL OLD VALUE (no restart)
kubectl logs vol-update-test    # FILE: UPDATED  ← NEW VALUE (auto-updated)
```

This proves the core difference: **env vars don't update, volume mounts do**.

---

## 📊 SECTION 10 — COMPLETE COMPARISON TABLE

| Feature | ConfigMap | Secret | Sealed Secret | External Secret |
|---------|-----------|--------|---------------|-----------------|
| For sensitive data | ❌ No | ⚠️ Yes (base64 only) | ✅ Yes (encrypted) | ✅ Yes (external vault) |
| Safe to commit to Git | ✅ Yes | ❌ No | ✅ Yes | ✅ Only reference |
| Automatic rotation | ❌ No | ❌ No | ❌ No | ✅ Yes (refreshInterval) |
| Audit trail | ❌ No | ❌ No | ❌ No | ✅ Yes (CloudTrail) |
| External dependency | ❌ No | ❌ No | ❌ No | ✅ Yes (AWS/Vault) |
| Encryption at rest | ❌ No (default) | ❌ No (default) | ✅ Yes | ✅ Yes (in vault) |
| Best for | App config | Simple secrets | GitOps secrets | Enterprise/Banking |

---

## 🔑 SECTION 11 — KEY TERMS TO REMEMBER

| Term | Simple Meaning |
|------|---------------|
| **ConfigMap** | Stores non-sensitive config as key-value pairs |
| **Secret** | Stores sensitive data — base64 encoded, not encrypted by default |
| **base64** | Encoding scheme — NOT encryption. Easily reversible. |
| **Opaque** | Default Secret type — arbitrary key-value pairs |
| **envFrom** | Inject ALL keys from ConfigMap/Secret as env vars |
| **env.valueFrom** | Inject ONE specific key as a specific env var name |
| **configMapKeyRef** | Reference a specific key from a ConfigMap |
| **secretKeyRef** | Reference a specific key from a Secret |
| **volumeMount** | Mount ConfigMap/Secret as files inside a container |
| **mountPath** | Directory inside container where volume is mounted |
| **subPath** | Mount a single key as a single file (not whole ConfigMap) |
| **defaultMode** | File permissions for Secret volume files (e.g., 0400) |
| **stringData** | Write plain text in Secret YAML — Kubernetes encodes it |
| **Auto-update** | Volume mounts update ~60s after ConfigMap/Secret changes |
| **Sealed Secrets** | Bitnami tool — encrypts Secrets for safe Git storage |
| **SealedSecret** | Kubernetes object — encrypted Secret, safe for Git |
| **kubeseal** | CLI tool to encrypt a Secret into a SealedSecret |
| **External Secrets** | Operator that syncs secrets from AWS/Vault into Kubernetes |
| **SecretStore** | ESO object — defines how to connect to external secret system |
| **ExternalSecret** | ESO object — defines what to sync and where to put it |
| **refreshInterval** | How often ESO re-syncs from external secret store |
| **IRSA** | IAM Roles for Service Accounts — AWS auth for ESO |

---

*File: K8s_ConfigSecrets_Concept_and_Lab.md*
*Repository: Interview_Preparation_2026 → Kubernetes/*
*Next: K8s_ConfigSecrets_Interview_Questions.md*
