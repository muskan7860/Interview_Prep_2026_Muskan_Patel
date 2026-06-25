# Kubernetes etcd — Deep Dive Concept + Hands-On Lab
> Written for: Someone with 4 years of DevOps experience preparing for senior-level interviews
> Style: First-standard student explanation → deep technical truth → hands-on lab with line-by-line explanation

---

## 🧠 SECTION 1 — WHAT IS etcd? (Story First)

### Forget Kubernetes for 2 minutes — start with a story

Imagine you are running a large hospital with 500 patients, 200 doctors, 50 departments, and 1000 rooms.

Now answer this: **how does everyone know what's happening?**

The hospital has one central **filing room**. Every single piece of information lives there:
- Which patient is in which room
- Which doctor is assigned to which patient
- Which departments are open or closed
- Which medications are prescribed to whom

Every doctor, every nurse, every manager — before doing anything — **checks the filing room** first. And after doing anything — they **update the filing room** immediately.

If that filing room **burns down** → complete chaos. Nobody knows what's happening. No new admissions. No medication orders. The hospital doesn't immediately stop (patients in rooms are still there, nurses still helping them) — but **nothing new can happen** and **nothing can be looked up**.

**That filing room IS etcd in Kubernetes.**

etcd is the **single source of truth** for the entire cluster. Every object — every pod, every deployment, every secret, every configmap, every node status — is stored in etcd.

---

## 📖 SECTION 2 — WHAT EXACTLY IS etcd?

### The Technical Definition (Now That You Have the Picture)

etcd is a **distributed, reliable key-value store**. Let's break that phrase down:

```
distributed  → runs on multiple machines, not just one
reliable     → designed to never lose your data, even if machines crash
key-value    → stores data as pairs: name (key) → information (value)
store        → a database
```

### Key-Value — What Does That Mean?

Think of it like a dictionary. In a real dictionary:
```
Key: "apple"    →    Value: "a round fruit that grows on trees"
Key: "hospital" →    Value: "a place where sick people receive treatment"
```

In etcd, Kubernetes stores things like:
```
Key: /registry/pods/default/nginx-pod    →    Value: {all the JSON info about that pod}
Key: /registry/deployments/banking/payment-app  →  Value: {full deployment spec}
Key: /registry/secrets/banking/db-password  →  Value: {base64 encoded secret data}
```

Every single Kubernetes object is stored this way. The key is a **path** (like a file path) and the value is the **full JSON specification** of that object.

---

## 🏗️ SECTION 3 — WHERE DOES etcd FIT IN THE CLUSTER?

```
┌─────────────────────────────────────────────────────────┐
│                CONTROL PLANE (Master Node)               │
│                                                         │
│   You (kubectl) ──► API Server ──► etcd (THE DATABASE)  │
│                         │                               │
│                         ▼                               │
│              Controller Manager                         │
│              Scheduler                                  │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                  WORKER NODES                           │
│   kubelet ──► reads from API Server (not etcd directly) │
└─────────────────────────────────────────────────────────┘
```

### The Most Important Rule in Kubernetes:

> **ONLY the API Server reads from and writes to etcd.**
> No other component — not kubelet, not the scheduler, not the controller manager — touches etcd directly.
> Everything goes through the API Server.

Why this rule? Because the API Server is the **gatekeeper**:
- It runs authentication (who are you?)
- It runs authorization (are you allowed?)
- It validates the data (is this valid YAML/JSON?)
- THEN it writes to etcd

If components could write to etcd directly, they could bypass all these checks. Chaos.

---

## 🔑 SECTION 4 — WHAT DOES etcd STORE?

Everything. Let me be specific:

```
WHAT etcd stores:
├── All Pod objects (spec + current status)
├── All Deployment objects
├── All ReplicaSet objects
├── All Service objects
├── All ConfigMap data
├── All Secret data (base64 encoded, not encrypted by default)
├── All Node objects (registration info + conditions)
├── All Namespace objects
├── All RBAC objects (Roles, RoleBindings, ClusterRoles)
├── All PersistentVolume and PersistentVolumeClaim objects
├── ServiceAccount objects
├── Lease objects (for leader election)
├── Events
└── ... everything else

WHAT etcd does NOT store:
├── The actual running container (that's on the worker node)
├── Container logs (those are on the worker node's disk)
├── Container images (those are in a registry)
└── Actual metrics/CPU/memory data (that's in metrics-server)
```

### Desired State vs Actual State — The Critical Concept

etcd stores **DESIRED STATE** — what you WANT to exist.

```
You run: kubectl create deployment nginx --replicas=3

What goes into etcd:
  "I want a Deployment called nginx with 3 replicas of nginx:latest"

What etcd does NOT know:
  Whether those 3 pods are actually running right now
  (That's the job of kubelet to report back via API Server)
```

Controllers read the desired state from etcd (via API Server), look at actual state (also updated in etcd by kubelet), and take action to close the gap.

---

## 🌐 SECTION 5 — WHAT DOES "DISTRIBUTED" MEAN? (etcd Cluster)

### Single etcd = Single Point of Failure

If etcd is just one machine and that machine dies → your entire cluster's brain is gone.

### etcd Cluster = Multiple machines, one "brain"

In production, etcd runs as a **cluster of 3 or 5 nodes**. They work together as if they are one database.

```
Production etcd cluster (3 nodes):

  etcd-1  ←──────────────→  etcd-2
     ↑                          ↑
     └─────────→ etcd-3 ←───────┘

All three talk to each other constantly.
All three have the SAME data (they stay in sync).
If one dies → the other two keep working.
```

**Why is this powerful?**
- Even if one machine crashes → zero downtime, data is still available on the other two
- All writes are agreed upon by the majority before being confirmed → no data loss
- If all three go down and two come back → they re-sync and continue from where they left off

---

## ⚙️ SECTION 6 — RAFT CONSENSUS (How etcd Stays in Sync)

### The Problem etcd Needs to Solve

Imagine 3 etcd nodes. You write "I want 3 replicas" to node 1. Node 2 and 3 need to also have this data immediately — perfectly in sync. But what if node 2 is slow? What if node 3 is momentarily unreachable?

This is the **distributed systems consistency problem**. etcd solves it using an algorithm called **Raft**.

### Raft in Simple Terms — The Village Chief Analogy

Think of 3 etcd nodes as 3 village elders who must always agree on new laws:

```
Raft has three roles:
  LEADER   → one elder who receives all new laws and proposes them
  FOLLOWER → other elders who vote on proposals
  CANDIDATE → an elder trying to become the new leader (during elections)
```

**How a write works with Raft:**

```
Step 1: Client (API Server) sends write to the LEADER
        "Add: pods/nginx-pod = {spec...}"

Step 2: LEADER writes to its own log (but doesn't commit yet)
        Leader sends the entry to ALL FOLLOWERS

Step 3: FOLLOWERS write to their own log
        Each FOLLOWER sends back "I got it" (vote)

Step 4: LEADER counts votes
        If MAJORITY confirmed (2 out of 3) → COMMIT
        Leader tells followers: "commit this"

Step 5: LEADER replies to client: "Write successful"
        Now ALL nodes have the data
```

### The Magic of Majority (Quorum)

**Quorum** = the minimum number of nodes that must agree for a write to succeed.

```
Formula: Quorum = (Total nodes / 2) + 1, rounded down

3 nodes → quorum = 2   → can lose 1 node and still work
5 nodes → quorum = 3   → can lose 2 nodes and still work
7 nodes → quorum = 4   → can lose 3 nodes and still work

2 nodes → quorum = 2   → losing 1 node = cluster stops (WORSE than 1 node!)
4 nodes → quorum = 3   → same fault tolerance as 3 nodes but more cost
```

### Why Always ODD Numbers?

```
3 nodes: lose 1 → 2 remaining ≥ quorum(2) ✅ WORKS
4 nodes: lose 1 → 3 remaining ≥ quorum(3) ✅ WORKS
         lose 2 → 2 remaining < quorum(3) ❌ STOPS

3 nodes: fault tolerance = 1   (same as 4 nodes but cheaper)
5 nodes: fault tolerance = 2   (same as 6 nodes but cheaper)

Even numbers give you NO EXTRA fault tolerance vs the odd number below them.
Odd numbers maximize fault tolerance per node added.
```

### Leader Election in Raft

If the current leader goes down:

```
Followers stop receiving heartbeats from leader
After election timeout (150-300ms): followers become CANDIDATES
Each candidate votes for itself and asks others to vote for it
First candidate to get majority votes becomes new LEADER
New leader starts sending heartbeats → others become followers again

Total time for new election: usually under 1 second
```

---

## 🔒 SECTION 7 — etcd SECURITY

### TLS Encryption in Transit

All communication with etcd is encrypted using TLS certificates. There are three types of certs:

```
1. etcd CA certificate     → the root certificate that signs all others
2. etcd server certificate → etcd presents this to prove "I am the real etcd"
3. etcd client certificate → API Server presents this to prove "I am authorized to talk to etcd"

Certificate files location (kubeadm clusters):
  /etc/kubernetes/pki/etcd/ca.crt          ← CA cert
  /etc/kubernetes/pki/etcd/server.crt      ← etcd server cert
  /etc/kubernetes/pki/etcd/server.key      ← etcd server private key
  /etc/kubernetes/pki/etcd/peer.crt        ← cert for etcd-to-etcd communication
```

### Encryption at Rest — The Big Nuance

**By default, etcd data is NOT encrypted at rest.**

This means:
- Secrets are stored in etcd as **base64 encoded** — not encrypted
- If someone gets access to the etcd data directory or a snapshot file → they can decode all secrets
- `echo "c2VjcmV0cGFzc3dvcmQ=" | base64 -d` → `secretpassword`

**Encryption at rest can be enabled** by configuring the API Server with an `EncryptionConfiguration` object. This encrypts the data before writing to etcd using AES-CBC or AES-GCM.

```yaml
# EncryptionConfiguration example
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  providers:
  - aescbc:
      keys:
      - name: key1
        secret: <base64-encoded-32-byte-key>
  - identity: {}
```

**In production banking environments:**
etcd encryption at rest + RBAC restricting who can `kubectl get secrets` + AWS Secrets Manager (External Secrets Operator) = the full security stack.

---

## 💾 SECTION 8 — etcd BACKUP AND RESTORE (Most Important Ops Topic)

### Why Backup is Critical

If etcd is lost with no backup:
- All cluster configuration is permanently gone
- Running pods continue (kubelet maintains them locally)
- But you have NO record of what was supposed to be running
- You cannot recover the cluster configuration
- Every deployment, every secret, every RBAC rule — gone forever

### etcd Snapshot Backup

etcd has a built-in snapshot mechanism. A snapshot is a **point-in-time copy of the entire etcd database**.

```bash
# Taking a snapshot:
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

**Breaking down every part:**
- `ETCDCTL_API=3` → environment variable. etcdctl supports API version 2 and 3. Always use version 3 for modern clusters.
- `etcdctl` → the command-line tool for interacting with etcd (like kubectl for Kubernetes, but for etcd)
- `snapshot save` → take a snapshot and save it to a file
- `/backup/etcd-snapshot.db` → the file path where the snapshot is saved. `.db` = database file
- `--endpoints=https://127.0.0.1:2379` → where etcd is listening. Port 2379 is etcd's client port (always). `127.0.0.1` = localhost (running on the same machine as etcd)
- `--cacert=` → the CA certificate to verify etcd's identity (TLS)
- `--cert=` → our client certificate to prove WE are authorized to talk to etcd
- `--key=` → our private key that pairs with the cert

### Verifying the Snapshot

```bash
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-snapshot.db --write-out=table
```
- `snapshot status` → inspect a snapshot file and show info about it
- `--write-out=table` → display output as a formatted table (easier to read)

Output:
```
+----------+----------+------------+------------+
|   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
+----------+----------+------------+------------+
| a9e5b8c2 |    12345 |       1234 |    4.2 MB  |
+----------+----------+------------+------------+
```
- `REVISION` → etcd's internal version number. Every write increments this.
- `TOTAL KEYS` → how many key-value pairs are stored (= all Kubernetes objects)
- `TOTAL SIZE` → size of the database

### etcd Restore

```bash
# Step 1: Stop the API Server (so nothing writes to etcd during restore)
# On kubeadm: move the static pod manifest out of /etc/kubernetes/manifests/

# Step 2: Restore the snapshot to a new directory
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd-restored \
  --name=master \
  --initial-cluster=master=https://127.0.0.1:2380 \
  --initial-cluster-token=etcd-cluster-1 \
  --initial-advertise-peer-urls=https://127.0.0.1:2380

# Step 3: Update etcd to use the new data directory
# Edit /etc/kubernetes/manifests/etcd.yaml
# Change --data-dir=/var/lib/etcd to --data-dir=/var/lib/etcd-restored

# Step 4: Restart etcd and API Server
```

**Breaking down restore flags:**
- `--data-dir=/var/lib/etcd-restored` → restore to THIS directory (new, separate from current data to be safe)
- `--name=master` → the name of THIS etcd member in the cluster
- `--initial-cluster=` → tells the restored etcd who all the members are
- `--initial-cluster-token=` → a unique token to identify this cluster (prevents accidental cross-cluster communication)
- `--initial-advertise-peer-urls=` → the URL other etcd members use to talk to this member (port 2380 is etcd's peer port)

---

## 📊 SECTION 9 — etcd PERFORMANCE

### Why etcd Performance Matters

etcd is on the **critical path** of EVERY Kubernetes operation. Every `kubectl get`, every controller reconciliation loop, every scheduler decision — all of them read from or write to etcd via the API Server.

```
Slow etcd = slow API Server = slow kubectl = slow everything
```

### The Number One Factor: Disk I/O Latency

etcd writes to disk on **every single transaction** using a Write-Ahead Log (WAL). It MUST flush to disk before confirming a write. This means:

```
HDD disk: average write latency = 5-10ms per operation
SSD disk: average write latency = 0.1-0.5ms per operation
NVMe disk: average write latency = 0.02-0.1ms per operation

etcd recommendation: < 10ms disk write latency
etcd ideal: < 1ms disk write latency
```

**If your etcd is slow → first thing to check is disk type.**

### etcd Database Size

Over time, your etcd database grows because:
- Old ReplicaSets accumulate (from Deployment updates)
- Completed Jobs aren't cleaned up
- Events pile up (Events are stored in etcd, expire after 1 hour by default)
- Orphaned ConfigMaps and Secrets from deleted apps
- Every write creates a new "revision" — old revisions pile up

```
etcd keeps a history of all writes (revisions).
This history is used for:
  - Watch notifications (tell me what changed since revision X)
  - Rollback capability

But old revisions take up space.
Solution: COMPACTION — delete revisions older than N
```

### Compaction and Defragmentation

```
COMPACTION: remove old revisions (free logical space)
  → Database still takes same disk space after compaction
  → Space is just marked as "reusable" internally

DEFRAGMENTATION: actually reclaim disk space
  → Rewrites the database file, removing all the gaps
  → Disk file actually shrinks after this

Kubernetes does auto-compaction every 5 minutes by default.
Defragmentation must be triggered manually (or on a schedule).
```

---

## 🔍 SECTION 10 — etcd PORTS

```
Port 2379 → CLIENT port
            Used by: API Server → etcd communication
            Used by: etcdctl commands from humans

Port 2380 → PEER port
            Used by: etcd → etcd communication (between cluster members)
            Used for: Raft consensus, leader election, data replication
```

Remember this because interviewers ask: **"What port does etcd listen on?"**
Answer: **2379 for clients, 2380 for peers.**

---

## 💻 SECTION 11 — HANDS-ON LAB

> Every command explained word by word. Every flag explained. Nothing assumed.

### Lab Prerequisites

- MicroK8s running on your ThinkPad Ubuntu OR KillerCoda
- Access to the control plane node (SSH or direct)
- `etcdctl` installed OR available inside the etcd pod

---

### LAB 1 — Find etcd Running in Your Cluster

```bash
kubectl get pods -n kube-system | grep etcd
```

**Breaking down:**
- `kubectl get pods` → list all pods
- `-n kube-system` → in the kube-system namespace (where all Kubernetes system components run)
- `|` → pipe: send left side's output as input to right side
- `grep etcd` → filter: only show lines containing the word "etcd"

**Output:**
```
etcd-controlplane   1/1   Running   0   10d
```

```bash
# Get detailed info about the etcd pod
kubectl describe pod etcd-controlplane -n kube-system
```

**Look for the Command section:**
```
Command:
  etcd
  --advertise-client-urls=https://192.168.1.10:2379
  --cert-file=/etc/kubernetes/pki/etcd/server.crt
  --client-cert-auth=true
  --data-dir=/var/lib/etcd
  --initial-cluster=controlplane=https://192.168.1.10:2380
  --key-file=/etc/kubernetes/pki/etcd/server.key
  --listen-client-urls=https://127.0.0.1:2379,https://192.168.1.10:2379
  --listen-peer-urls=https://192.168.1.10:2380
  --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
```

**What each flag means:**
- `--advertise-client-urls` → "Tell clients to connect to me at this address" (port 2379 = client port)
- `--cert-file` → etcd's own certificate (proves its identity to clients)
- `--client-cert-auth=true` → "Require clients to present a certificate too" (mutual TLS)
- `--data-dir=/var/lib/etcd` → "Store all my data files in this directory"
- `--initial-cluster` → "These are all the etcd members in the cluster" (for peering)
- `--listen-client-urls` → "Accept client connections on these addresses"
- `--listen-peer-urls` → "Accept peer connections from other etcd nodes on this address" (port 2380 = peer port)
- `--trusted-ca-file` → "Trust certificates signed by this CA"

---

### LAB 2 — Look Inside etcd Using etcdctl

First, get into the etcd pod's shell:

```bash
kubectl exec -it etcd-controlplane -n kube-system -- sh
```

**Breaking down:**
- `kubectl exec` → run a command inside a running container
- `-it` → `-i` = interactive (keep stdin open), `-t` = allocate a terminal (so it looks like a normal shell)
- `etcd-controlplane` → the pod name
- `-n kube-system` → in kube-system namespace
- `--` → separator. Everything after this is the command to run inside the container
- `sh` → start a shell inside the container

**Now set the environment variables so etcdctl works:**

```bash
export ETCDCTL_API=3
export ETCDCTL_CACERT=/etc/kubernetes/pki/etcd/ca.crt
export ETCDCTL_CERT=/etc/kubernetes/pki/etcd/server.crt
export ETCDCTL_KEY=/etc/kubernetes/pki/etcd/server.key
export ETCDCTL_ENDPOINTS=https://127.0.0.1:2379
```

**Breaking down:**
- `export` → set an environment variable that will be inherited by commands we run
- `ETCDCTL_API=3` → tell etcdctl to use API version 3
- `ETCDCTL_CACERT` → path to the CA certificate for verifying etcd's identity
- `ETCDCTL_CERT` → our client certificate (proves we're authorized)
- `ETCDCTL_KEY` → our private key matching the client certificate
- `ETCDCTL_ENDPOINTS` → where etcd is running (localhost:2379)

**Check etcd cluster health:**

```bash
etcdctl endpoint health
```

**Output:**
```
https://127.0.0.1:2379 is healthy: successfully committed proposal: took = 1.2ms
```

This tells us: etcd is healthy and can commit proposals (writes) in 1.2ms.

---

### LAB 3 — See All Kubernetes Objects Stored in etcd

```bash
# List all keys in etcd (the root directory)
etcdctl get / --prefix --keys-only | head -30
```

**Breaking down:**
- `etcdctl get` → retrieve key-value pairs from etcd
- `/` → start from the root path
- `--prefix` → return ALL keys that START WITH `/` (which is everything)
- `--keys-only` → only show the key names, not the values (values are huge JSON)
- `|` → pipe
- `head -30` → `head` shows the first N lines. `-30` = show first 30 lines only

**Output (you'll see something like):**
```
/registry/apiregistration.k8s.io/apiservices/v1.
/registry/clusterrolebindings/cluster-admin
/registry/clusterroles/admin
/registry/configmaps/kube-system/coredns
/registry/deployments/kube-system/coredns
/registry/endpoints/default/kubernetes
/registry/namespaces/default
/registry/namespaces/kube-system
/registry/pods/kube-system/etcd-controlplane
/registry/pods/kube-system/kube-apiserver-controlplane
/registry/secrets/kube-system/bootstrap-token-abc123
/registry/serviceaccounts/default/default
```

**Read this carefully** — THIS is how Kubernetes actually stores everything. Notice the pattern:
```
/registry/<object-type>/<namespace>/<name>
```

For example:
- `/registry/pods/kube-system/etcd-controlplane` = the pod object for etcd itself
- `/registry/deployments/kube-system/coredns` = the coredns deployment
- `/registry/secrets/kube-system/bootstrap-token-abc123` = a secret

---

### LAB 4 — Read an Actual Kubernetes Object from etcd

```bash
# Read the data stored for the default namespace
etcdctl get /registry/namespaces/default
```

**Output:** (it will be binary/encoded data — don't panic)
```
/registry/namespaces/default
k8s
...{bunch of binary characters}...
```

The values in etcd are stored as **protobuf** format (a binary format, more efficient than JSON). That's why it looks like gibberish. The API Server decodes this automatically when you run `kubectl get`.

```bash
# Better: search for all pods in the default namespace
etcdctl get /registry/pods/default --prefix --keys-only
```

This shows all pod keys under the `default` namespace. Each key = one pod object stored in etcd.

---

### LAB 5 — Check etcd Cluster Status and Database Size

```bash
etcdctl endpoint status --write-out=table
```

**Breaking down:**
- `endpoint status` → show status information about the etcd endpoint(s) we're connected to
- `--write-out=table` → format the output as a table

**Output:**
```
+---------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|         ENDPOINT          |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+---------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| https://127.0.0.1:2379    | 8e9e05c52164694d | 3.5.0   | 4.5 MB  |      true |      false |         4 |       1234 |               1234 |        |
+---------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
```

**Understanding each column:**
- `ENDPOINT` → which etcd server this info is about
- `ID` → unique ID of this etcd member (assigned when cluster was created)
- `VERSION` → etcd version running
- `DB SIZE` → how big the database file is on disk. Watch this number — if it grows over 2-3GB, performance degrades
- `IS LEADER` → is this node currently the Raft leader? (`true` = yes, this is the active one)
- `IS LEARNER` → is this a learner node (observer, non-voting)? Usually `false`
- `RAFT TERM` → how many leader elections have happened. Increases by 1 each election.
- `RAFT INDEX` → total number of writes ever made to etcd (revision number)
- `RAFT APPLIED INDEX` → how many writes have been applied (committed). Should match RAFT INDEX.
- `ERRORS` → any current errors. Should be empty.

---

### LAB 6 — Take an etcd Backup (Snapshot)

```bash
# Exit the etcd pod first (or open a new terminal on the control plane)
exit

# On the control plane node, take a snapshot
ETCDCTL_API=3 etcdctl snapshot save /tmp/etcd-backup-$(date +%Y%m%d-%H%M%S).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

**Breaking down the new part:**
- `/tmp/etcd-backup-$(date +%Y%m%d-%H%M%S).db` → the backup file path
  - `/tmp/` → temporary directory (use a proper backup location in production — S3, NFS, etc.)
  - `etcd-backup-` → prefix of the file name
  - `$(date +%Y%m%d-%H%M%S)` → runs the `date` command and embeds the result in the filename
    - `date` = show current date/time
    - `+%Y%m%d-%H%M%S` = format: `20240115-143022` (year, month, day, hour, minute, second)
  - `.db` → file extension for etcd database files
  - End result: a file like `etcd-backup-20240115-143022.db`

**Output:**
```
{"level":"info","msg":"snapshot saved at /tmp/etcd-backup-20240115-143022.db"}
Snapshot saved at /tmp/etcd-backup-20240115-143022.db
```

**Now verify the backup:**
```bash
ETCDCTL_API=3 etcdctl snapshot status /tmp/etcd-backup-20240115-143022.db \
  --write-out=table
```

**Output:**
```
+----------+----------+------------+------------+
|   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
+----------+----------+------------+------------+
| a9e5b8c2 |    12345 |       1876 |    4.5 MB  |
+----------+----------+------------+------------+
```

- `HASH` → a checksum. Used to verify the file isn't corrupted.
- `REVISION` → the etcd version at the time of the snapshot (every write increments this)
- `TOTAL KEYS` → 1876 key-value pairs = 1876 Kubernetes objects stored at this moment
- `TOTAL SIZE` → size of the snapshot file

---

### LAB 7 — Check etcd Database Growth and Compaction

```bash
# Check current revision number
kubectl exec -it etcd-controlplane -n kube-system -- sh -c \
  "ETCDCTL_API=3 etcdctl \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint status --write-out=json" | python3 -m json.tool | grep -i revision
```

**Breaking down the new parts:**
- `sh -c "..."` → run this string as a shell command inside the pod
- `| python3 -m json.tool` → pipe JSON output through Python's JSON pretty-printer (makes it readable)
- `| grep -i revision` → filter for lines containing "revision" (case-insensitive with `-i`)

**Now create some objects to see revision increase:**

```bash
# Create 5 configmaps
for i in 1 2 3 4 5; do
  kubectl create configmap test-cm-$i --from-literal=key=value$i
done
```

**Breaking down:**
- `for i in 1 2 3 4 5; do` → loop: `i` takes values 1, 2, 3, 4, 5 in turn
- `kubectl create configmap test-cm-$i` → create a configmap named `test-cm-1`, `test-cm-2`, etc.
- `--from-literal=key=value$i` → store a key-value pair directly (no file needed)
  - `key` = the config key name
  - `value$i` = the value (value1, value2, etc.)
- `done` → end the loop

**Check revision again** — it will have increased by 5 (one increment per write).

**Trigger manual compaction:**
```bash
# Get current revision
REV=$(kubectl exec etcd-controlplane -n kube-system -- sh -c \
  "ETCDCTL_API=3 etcdctl \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint status --write-out=json" | python3 -m json.tool | grep '"revision"' | tail -1 | grep -o '[0-9]*')

echo "Current revision: $REV"
```

**Breaking down:**
- `REV=$(...)` → run the command inside `$(...)` and store the output in variable `REV`
- `grep '"revision"'` → find the line with revision
- `tail -1` → if there are multiple matches, take the last one
- `grep -o '[0-9]*'` → `-o` = only print the matching part. `[0-9]*` = only digits (the number itself)
- `echo "Current revision: $REV"` → print the variable value

**Clean up test configmaps:**
```bash
for i in 1 2 3 4 5; do
  kubectl delete configmap test-cm-$i
done
```

---

### LAB 8 — See etcd Leader Election in Action

```bash
# See the lease object for etcd leader election
kubectl get lease -n kube-system
```

**Output:**
```
NAME                      HOLDER                                 AGE
etcd-controlplane         controlplane                           10d
kube-controller-manager   controlplane_abc123                    10d
kube-scheduler            controlplane_xyz456                    10d
```

```bash
# Get detailed info about the etcd lease
kubectl get lease etcd-controlplane -n kube-system -o yaml
```

**Breaking down:**
- `get lease` → Lease objects are the distributed locks used for leader election
- `etcd-controlplane` → the lease name
- `-o yaml` → show full YAML output

**Output:**
```yaml
apiVersion: coordination.k8s.io/v1
kind: Lease
metadata:
  name: etcd-controlplane
  namespace: kube-system
spec:
  acquireTime: "2024-01-10T10:00:00Z"
  holderIdentity: controlplane
  leaseDurationSeconds: 10
  renewTime: "2024-01-15T14:30:00Z"
```

**What each field means:**
- `acquireTime` → when the current leader first acquired this lease
- `holderIdentity` → which node is currently the leader (`controlplane` = the control plane node)
- `leaseDurationSeconds: 10` → if the lease isn't renewed within 10 seconds, it expires and a new election happens
- `renewTime` → when the current leader last renewed the lease (updates every few seconds)

---

### LAB 9 — Simulate What Happens When etcd Becomes Unavailable

> **WARNING: Only do this on a lab/test cluster. NOT on production.**

```bash
# Move the etcd static pod manifest to stop etcd
sudo mv /etc/kubernetes/manifests/etcd.yaml /tmp/etcd.yaml
```

**Breaking down:**
- `sudo` → run as superuser (root) — needed to write to system directories
- `mv` → move a file
- `/etc/kubernetes/manifests/etcd.yaml` → this is the static pod manifest. kubelet watches this directory. When a YAML file is here → kubelet runs that pod. When the file is gone → kubelet stops the pod.
- `/tmp/etcd.yaml` → moving it here (temporary storage, won't be auto-run)

**Wait 30 seconds then try:**
```bash
kubectl get pods
```

**Output:**
```
Error from server: etcdserver: request timed out
```

This is what happens when etcd is down — the API Server cannot respond.

**Restore etcd:**
```bash
sudo mv /tmp/etcd.yaml /etc/kubernetes/manifests/etcd.yaml
```

Wait 30-60 seconds for etcd to come back up, then try kubectl again.

---

## 📊 SECTION 12 — etcd SUMMARY TABLE

| Topic | Key Fact |
|-------|----------|
| What is etcd | Distributed key-value store = cluster database |
| Who talks to etcd | ONLY the API Server (no other component directly) |
| What etcd stores | All Kubernetes objects (desired state) |
| What etcd doesn't store | Running containers, logs, images, metrics |
| Client port | 2379 |
| Peer port | 2380 |
| Consensus algorithm | Raft |
| Quorum for 3 nodes | 2 nodes must agree |
| Quorum for 5 nodes | 3 nodes must agree |
| Fault tolerance (3 nodes) | Can lose 1 node |
| Fault tolerance (5 nodes) | Can lose 2 nodes |
| Why odd numbers | Maximize fault tolerance per node |
| Backup tool | `etcdctl snapshot save` |
| Backup verify | `etcdctl snapshot status` |
| Default encryption at rest | NOT encrypted (base64 only) |
| Performance bottleneck | Disk I/O latency (needs SSD) |
| Recommended disk latency | < 10ms (ideally < 1ms) |
| Database cleanup | Compaction (logical) + Defragmentation (physical) |

---

## 🔑 SECTION 13 — KEY TERMS TO REMEMBER

| Term | Simple Meaning |
|------|---------------|
| **Key-Value Store** | Database that stores data as name → information pairs |
| **Distributed** | Runs on multiple machines that work as one |
| **Raft** | The algorithm that keeps all etcd nodes in sync |
| **Quorum** | Minimum number of nodes that must agree for a write to succeed |
| **Leader** | The one etcd node that receives all writes and coordinates the cluster |
| **Follower** | Standby etcd nodes that vote on the leader's proposals |
| **WAL** | Write-Ahead Log — etcd writes here before committing (ensures no data loss) |
| **Snapshot** | A point-in-time backup of the entire etcd database |
| **Compaction** | Removing old revision history to free logical space |
| **Defragmentation** | Rewriting the database file to actually reclaim disk space |
| **Revision** | A version number — increments by 1 on every write to etcd |
| **Protobuf** | Binary format used to encode data stored in etcd (more efficient than JSON) |
| **Encryption at rest** | Encrypting data on disk so snapshots can't be read without a key |
| **TLS** | Transport Layer Security — encrypts data in transit between components |
| **2379** | etcd client port (API Server talks to etcd here) |
| **2380** | etcd peer port (etcd nodes talk to each other here) |

---

*File: K8s_etcd_Concept_and_Lab.md*
*Repository: Interview_Preparation_2026 → Kubernetes/*
*Next: K8s_etcd_Interview_Questions.md*
