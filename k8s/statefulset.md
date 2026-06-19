# StatefulSet — Theory, Labs, and Interview Questions

> Level: 4 Years Experience
> Author: Muskan Patel

---

## PART 1 — THEORY

---

## 1. Why Deployment Is Not Enough For Everything

Deployment treats pods as interchangeable, identical clones. If
`payment-deployment-7d9f8b8f6c-x7k2p` dies, it's replaced by
`payment-deployment-7d9f8b8f6c-m3n9q` — completely different name,
completely different IP, and nobody cares, because for a stateless
app every replica is functionally identical.

Now imagine running PostgreSQL with replication — one PRIMARY
database (accepts writes) and two REPLICAS (read-only copies syncing
from the primary). Running this as a Deployment hits three
unsolvable problems:

**Problem 1 — Storage gets mixed up.** A single shared volume means
all 3 database pods write to the SAME disk simultaneously — instant
corruption. Without sharing, a replacement pod gets a brand-new empty
disk — all the primary's data is gone.

**Problem 2 — No stable identity.** Replicas need to know "where is
the primary?" to sync from it. But Deployment pods get random names
and IPs every restart — you cannot hardcode a connection target that
will still be valid tomorrow.

**Problem 3 — No startup order.** The primary must be up and
accepting connections BEFORE replicas try to sync from it. Deployment
creates all pods in parallel, with no ordering guarantee.

Kubernetes needed an object that gives each replica its OWN dedicated
storage that follows it forever, a PERMANENT predictable name and
address, and creates them in strict ORDER. That object is
**StatefulSet**.

---

## 2. The Mental Model — The Apartment Building

A Deployment is like a hotel — guests (pods) get ANY available room,
and if they leave and come back, they might get a totally different
room number. Nobody minds, because all rooms are identical and
nothing important is stored IN the room.

A StatefulSet is like an apartment building where EVERY apartment has
a permanent number, and EVERY tenant's belongings stay in THEIR
apartment even if the tenant temporarily leaves and comes back.

```
Apartment 0 (postgres-0) — always apartment 0, own storage unit
Apartment 1 (postgres-1) — always apartment 1, own storage unit
Apartment 2 (postgres-2) — always apartment 2, own storage unit
```

If the tenant in Apartment 0 moves out and a new tenant moves in, they
STILL move into "Apartment 0" — same address, and the previous
tenant's storage unit is still there, untouched.

---

## 3. Two Objects Working Together

Unlike Deployment (one object), StatefulSet ALWAYS needs a companion:
a **Headless Service**.

### Part A — The Headless Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
spec:
  clusterIP: None        # this is what makes it "headless"
  selector:
    app: postgres
  ports:
  - port: 5432
```

A NORMAL Service gets ONE shared virtual IP, and traffic to it gets
randomly load-balanced across matching pods — perfect for stateless
apps ("I don't care WHICH replica answers"). For a database, you need
to reach `postgres-0` SPECIFICALLY (the primary), not a random one of
three. A headless Service tells CoreDNS: "don't give one shared IP —
return the INDIVIDUAL IP of EACH matching pod, addressable by its
OWN name."

### Part B — The StatefulSet Itself

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres-headless    # links to the headless Service
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

---

## 4. What Happens When You Apply This — Step by Step

1. The StatefulSet Controller sees `replicas: 3`.
2. Unlike Deployment/ReplicaSet (which create ALL pods in parallel
   with random names), the StatefulSet Controller creates pods ONE
   AT A TIME, IN ORDER, with PREDICTABLE names based on ordinal
   index:

```
Step 1: Before creating postgres-0, controller FIRST creates a
        PersistentVolumeClaim "data-postgres-0"
        (from volumeClaimTemplates -- this pod's dedicated storage)

Step 2: Creates pod "postgres-0", mounts data-postgres-0 into it

Step 3: WAITS until postgres-0 is Running AND Ready
        (does NOT move to step 4 until this is true)

Step 4: Creates PVC "data-postgres-1"
Step 5: Creates pod "postgres-1", mounts data-postgres-1
Step 6: WAITS until postgres-1 is Running AND Ready

Step 7: Creates PVC "data-postgres-2"
Step 8: Creates pod "postgres-2", mounts data-postgres-2
Step 9: WAITS until postgres-2 is Running AND Ready
        DONE
```

3. CoreDNS automatically creates individual DNS records for each pod:

```
postgres-0.postgres-headless.default.svc.cluster.local -> IP of postgres-0
postgres-1.postgres-headless.default.svc.cluster.local -> IP of postgres-1
postgres-2.postgres-headless.default.svc.cluster.local -> IP of postgres-2
```

---

## 5. The Critical Guarantee — Delete a Pod, Identity and Storage Survive

```bash
kubectl delete pod postgres-0
```

1. The StatefulSet Controller notices `postgres-0` is gone.
2. It recreates it with the EXACT SAME NAME, `postgres-0` (not a new
   random name, unlike ReplicaSet).
3. Before starting the pod, it checks: does PVC `data-postgres-0`
   already exist? YES — deleting a POD does NOT delete its PVC.
4. It REATTACHES the EXISTING `data-postgres-0` PVC to the new
   `postgres-0` pod.
5. From the application's perspective, it's as if the process simply
   restarted — all data files are right where they were left,
   because it's literally the same underlying disk.

**Important distinction:** the pod's NAME and STORAGE are stable. The
pod's IP ADDRESS is NOT stable — nothing in Kubernetes guarantees a
fixed pod IP across recreation. This is exactly why we address
specific replicas via their DNS name (`postgres-0.postgres-headless...`)
rather than raw IPs.

---

## 6. Line by Line — Every Field Unique to StatefulSet

### `clusterIP: None`

Disables the normal "one shared virtual IP, load-balanced" behavior.
CoreDNS instead returns the INDIVIDUAL IP of each matching pod when
queried by its specific name.

### `serviceName: postgres-headless`

Links the StatefulSet to its headless Service. This is the DOMAIN
used to build each pod's full DNS name:
```
<pod-name>.<serviceName>.<namespace>.svc.cluster.local
postgres-0.postgres-headless.default.svc.cluster.local
```

### `volumeClaimTemplates`

Creates a SEPARATE PVC for EACH pod ordinal automatically, named
`<volumeClaimTemplates.name>-<pod-name>`. This is different from a
normal `volumes:` field (used in Deployment), which would point ALL
replicas at the SAME shared PVC — causing multiple processes to write
to the same disk simultaneously.

**CRITICAL GOTCHA:** `volumeClaimTemplates` alone does NOTHING for
data persistence. It only PROVISIONS the storage (creates the PVC).
You MUST ALSO add a matching `volumeMounts` entry in the container
spec, pointing a specific filesystem path AT that volume:

```yaml
spec:
  containers:
  - name: postgres
    volumeMounts:
    - name: data                          # must match volumeClaimTemplates name
      mountPath: /var/lib/postgresql/data # actual path the app writes to
volumeClaimTemplates:
- metadata:
    name: data
  spec:
    accessModes: ["ReadWriteOnce"]
    resources:
      requests:
        storage: 10Gi
```

Without `volumeMounts`, the PVC sits there `Bound` and unused, while
the container writes to its own throwaway filesystem layer — giving
a false sense of safety. Data appears to vanish on every pod restart
even though `kubectl get pvc` shows everything looks fine.

### Pod Ordinal Naming — `postgres-0`, `postgres-1`, `postgres-2`

Unlike ReplicaSet's random suffixes (`payment-rs-x7k2p`), StatefulSet
pods get SEQUENTIAL numbers starting from 0. This predictability is
the foundation of stable identity — `$HOSTNAME` inside the container
is ALWAYS the exact pod name, and never changes across restarts.

### `podManagementPolicy`

```yaml
spec:
  podManagementPolicy: OrderedReady   # default
  # OR
  podManagementPolicy: Parallel
```

`OrderedReady` is a strictly BLOCKING, sequential loop:

```
for i in 0, 1, 2, ... (up to replicas-1):
    if pod with ordinal i does not exist:
        create PVC for ordinal i
        create pod with ordinal i
        WAIT until that pod is Running AND Ready
        (do not proceed to i+1 until this is true)
```

There is no "smart" decision-making — it is genuinely this
mechanical: "is the previous one done? No? Wait. Yes? Move to the
next number." This exists because ordinal 0 is, by convention,
treated as the "seed"/first member that others bootstrap from — so
it must exist and be healthy before later ordinals try to connect to
it.

`Parallel` creates ALL pods at once, with no waiting — used only for
apps designed so any node can join/leave in any order (e.g.,
Cassandra).

### Deletion Order — The Exact Reverse of Creation

```bash
kubectl scale statefulset postgres --replicas=1
```

Same blocking logic, counting DOWN from the highest ordinal:

```
for i in (replicas-1), (replicas-2), ... down to new_replica_count:
    delete pod with ordinal i
    WAIT until fully terminated
    (do not proceed to i-1 until this is true)
```

This protects the foundational/oldest member (ordinal 0) from being
removed while newer members still exist.

---

## 7. Who Decides Primary vs Replica? (StatefulSet Does NOT Know)

This is the most commonly misunderstood part. **StatefulSet itself
has zero understanding of "primary" or "replica."** Its ONLY job is
stable name + stable storage + ordered creation. The APPLICATION
decides roles, typically using the pod's hostname (which equals its
ordinal-based name, guaranteed by StatefulSet) as a deterministic
signal:

```yaml
spec:
  template:
    spec:
      containers:
      - name: postgres
        image: postgres:15
        command:
        - sh
        - -c
        - |
          if [ "$HOSTNAME" = "postgres-0" ]; then
            echo "I am the PRIMARY"
            exec postgres -c wal_level=replica -c max_wal_senders=3
          else
            echo "I am a REPLICA -- syncing from postgres-0"
            exec postgres -c hot_standby=on
          fi
```

`$HOSTNAME` inside a StatefulSet pod is ALWAYS the exact pod name —
`postgres-0`, `postgres-1`, `postgres-2` — guaranteed and never
changes, unlike Deployment where it would be a random string.

**In real production, this logic is rarely hand-written.** Teams use
a purpose-built clustering tool:

- **Patroni** — wraps Postgres, performs LEADER ELECTION (same
  concept as `--leader-elect=true` on controller-manager from Day 1).
  Patroni decides "I am the leader" DYNAMICALLY, not just by hardcoded
  ordinal — so if `postgres-0` crashes, Patroni can PROMOTE
  `postgres-1` to primary automatically — true failover.
- **CloudNativePG / MongoDB Operator** — Kubernetes Operators that
  manage the entire lifecycle (failover, backups, role assignment)
  on top of a StatefulSet underneath.

---

## 8. Where Industry Actually Uses StatefulSet

| Scenario | What's Actually Used |
|---|---|
| Single-instance DB/cache, no HA needed (dev, small internal tool) | Deployment + 1 externally-created PVC (replicas: 1) |
| Multi-replica database needing per-pod storage + stable identity | StatefulSet + volumeClaimTemplates — the foundation |
| Production-grade database with automated failover, backups | A Kubernetes Operator (CloudNativePG, Percona Operator, MongoDB Operator) — creates and manages a StatefulSet underneath PLUS adds leader election/backup logic |
| Many companies at 3-4 years experience level | Often don't self-host the database in Kubernetes at all — use a managed cloud database (AWS RDS, Aurora) outside the cluster; Kubernetes only runs the stateless app tier via Deployment, which connects OUT to that external database |

**Why Deployment + volume mount "works" for replicas: 1 but breaks
beyond that:** with only ONE pod, there's no "multiple pods fighting
over the same disk" problem — the single pod mounts a PVC created
SEPARATELY (not via volumeClaimTemplates), and if it crashes,
Deployment recreates it pointing at that SAME pre-existing PVC. The
moment you try `replicas: 2` or more with this pattern, ALL pods
reference the SAME PVC — multiple processes writing to the same disk
simultaneously, which most storage backends won't even allow for
`ReadWriteOnce` (pods get stuck `ContainerCreating` with a volume
attachment conflict), and even where allowed, causes instant
corruption. This is exactly the problem `volumeClaimTemplates` solves
by giving each replica its own separate PVC.

---

## PART 2 — HANDS-ON LABS

---

## Lab 1 — Watch Ordered Creation and Stable Identity

```bash
kubectl apply -f - << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: web-headless
spec:
  clusterIP: None
  selector:
    app: web-sts
  ports:
  - port: 80
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: web-headless
  replicas: 3
  selector:
    matchLabels:
      app: web-sts
  template:
    metadata:
      labels:
        app: web-sts
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        volumeMounts:
        - name: data
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
EOF

# Watch pods come up IN ORDER -- one at a time
kubectl get pods -l app=web-sts -w
# web-0 -> Running first
# web-1 -> only starts AFTER web-0 is Ready
# web-2 -> only starts AFTER web-1 is Ready
# Ctrl+C once all 3 are Running

# Confirm the volume is actually mounted (not just provisioned)
kubectl describe pod web-0 | grep -A 3 Mounts
# Should show: /usr/share/nginx/html from data (rw)

# See the auto-created PVCs, one per pod
kubectl get pvc
# data-web-0
# data-web-1
# data-web-2
```

---

## Lab 2 — Test Stable DNS Names

```bash
kubectl run debug --image=busybox -it --rm -- sh

# Inside the debug pod:
nslookup web-0.web-headless.default.svc.cluster.local
nslookup web-1.web-headless.default.svc.cluster.local
nslookup web-2.web-headless.default.svc.cluster.local
# Each returns a DIFFERENT, SPECIFIC IP

# Compare -- query the base service name (no pod prefix)
nslookup web-headless.default.svc.cluster.local
# Returns MULTIPLE IPs (all 3) -- no load balancing, just a list
exit
```

---

## Lab 3 — The Critical Test: Delete and Watch Identity/Storage Persist

```bash
# Write a unique file into web-0's storage to PROVE persistence
kubectl exec web-0 -- sh -c 'echo "I am the original web-0" > /usr/share/nginx/html/proof.txt'
kubectl exec web-0 -- cat /usr/share/nginx/html/proof.txt

# Note the PVC's UID BEFORE deleting
kubectl get pvc data-web-0 -o jsonpath='{.metadata.uid}'
echo

# Note web-0's current IP
kubectl get pod web-0 -o wide

# THE KEY TEST -- delete it
kubectl delete pod web-0

# Watch it come back with the SAME NAME
kubectl get pods -l app=web-sts -w
# web-0 reappears -- Ctrl+C once Running

# Note its NEW IP -- this WILL be different (IPs are never stable)
kubectl get pod web-0 -o wide

# Compare the PVC UID -- should be IDENTICAL to before
kubectl get pvc data-web-0 -o jsonpath='{.metadata.uid}'
echo

# Now check the file
kubectl exec web-0 -- cat /usr/share/nginx/html/proof.txt
# "I am the original web-0" -- survives, because the SAME PVC
# (data-web-0) was reattached to the new pod

kubectl get pvc data-web-0
# Same PVC, same age -- never recreated
```

**Lesson:** pod NAME and STORAGE are stable across recreation. Pod
IP is NOT stable — that's exactly why we address replicas via their
DNS name rather than raw IPs.

```bash
kubectl delete statefulset web
kubectl delete svc web-headless
kubectl delete pvc -l app=web-sts
```

---

## Lab 4 — Scale Down and Watch Reverse-Order Deletion

```bash
kubectl apply -f - << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: scale-headless
spec:
  clusterIP: None
  selector:
    app: scale-sts
  ports:
  - port: 80
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: scale-test
spec:
  serviceName: scale-headless
  replicas: 4
  selector:
    matchLabels:
      app: scale-sts
  template:
    metadata:
      labels:
        app: scale-sts
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
EOF

kubectl get pods -l app=scale-sts
# scale-test-0, scale-test-1, scale-test-2, scale-test-3

# Scale down to 1 -- watch the ORDER of deletion
kubectl scale statefulset scale-test --replicas=1
kubectl get pods -l app=scale-sts -w
# scale-test-3 deleted FIRST,
# then scale-test-2,
# then scale-test-1,
# scale-test-0 survives -- deleted LAST

kubectl delete statefulset scale-test
kubectl delete svc scale-headless
```

---

## Lab 5 — Simulate Primary/Replica Role Assignment via Hostname

```bash
kubectl apply -f - << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: role-headless
spec:
  clusterIP: None
  selector:
    app: role-demo
  ports:
  - port: 80
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: role-demo
spec:
  serviceName: role-headless
  replicas: 3
  selector:
    matchLabels:
      app: role-demo
  template:
    metadata:
      labels:
        app: role-demo
    spec:
      containers:
      - name: app
        image: busybox
        command:
        - sh
        - -c
        - |
          if [ "$HOSTNAME" = "role-demo-0" ]; then
            echo "I AM THE PRIMARY ($HOSTNAME)"
          else
            echo "I AM A REPLICA ($HOSTNAME) -- would sync from role-demo-0"
          fi
          sleep 3600
EOF

sleep 5
kubectl logs role-demo-0
kubectl logs role-demo-1
kubectl logs role-demo-2
# role-demo-0 says "I AM THE PRIMARY"
# role-demo-1 and role-demo-2 say "I AM A REPLICA"
# This proves $HOSTNAME-based role assignment, the foundation
# of how real database StatefulSets decide primary vs replica

kubectl delete statefulset role-demo
kubectl delete svc role-headless
```

---

## Quick Reference — StatefulSet Commands

```bash
kubectl get statefulsets
kubectl get sts                       # short form
kubectl describe sts <name>

kubectl scale sts <name> --replicas=5

kubectl rollout status sts/<name>
kubectl rollout history sts/<name>

kubectl get pvc                        # see per-pod storage
kubectl delete sts <name>              # does NOT delete PVCs automatically
kubectl delete sts <name> --cascade=orphan   # leaves pods running too
```

---

*Next: DaemonSet*
