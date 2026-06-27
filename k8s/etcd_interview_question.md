# Kubernetes etcd — Interview Questions & Answers
> Target: 4 Years DevOps Experience | Senior-Level Interviews
> Style: Easy to memorize + Professional to say out loud

---

## 💡 HOW TO USE THIS FILE

- Read the question first
- Try answering in your own words
- Then read the answer — notice how it's simple but sounds senior-level
- Answers are written to be **said out loud** comfortably

---

## SECTION A — BASIC CONCEPT QUESTIONS

---

### Q1. What is etcd in Kubernetes?

**Answer:**

> "etcd is the distributed key-value store that acts as the single source of truth for the entire Kubernetes cluster. Every object in the cluster — pods, deployments, services, secrets, configmaps, RBAC rules — is stored in etcd as a key-value pair.
>
> The key follows a path structure like `/registry/pods/default/nginx-pod` and the value is the full JSON specification of that object. etcd stores the desired state — what you want to exist — and controllers read from etcd to know what to do.
>
> The most important rule: only the API Server talks to etcd directly. No other component — not kubelet, not the scheduler — reads from or writes to etcd. Everything goes through the API Server, which enforces authentication, authorization, and validation before any write reaches etcd."

---

### Q2. Why does Kubernetes use etcd specifically? Why not a regular database like MySQL?

**Answer:**

> "Kubernetes chose etcd for three specific reasons that a regular relational database doesn't provide well.
>
> First, etcd has a **Watch API** — you can subscribe to changes in real time. When a controller asks 'tell me whenever a new Deployment is created,' etcd pushes the notification immediately. A regular database needs polling — you'd have to keep asking 'did anything change?' every few seconds, which is wasteful and slow.
>
> Second, etcd has **strong consistency** via Raft consensus — every read returns the latest data, and every write is confirmed by a majority of nodes before being acknowledged. In a cluster where multiple controllers are making decisions, you can't afford stale reads.
>
> Third, etcd is **designed for distributed environments** — it handles network partitions, node failures, and leader elections natively. MySQL would need complex replication setup to achieve the same guarantees."

---

### Q3. What does etcd store and what does it NOT store?

**Answer:**

> "etcd stores all Kubernetes object specifications — pods, deployments, replicasets, services, configmaps, secrets, RBAC roles and bindings, namespaces, persistent volumes, service accounts, leases for leader election, and events.
>
> What etcd does NOT store: the actual running containers (those run on worker nodes and are managed by kubelet), container logs (those are on each node's disk), container images (those live in a registry), and real-time metrics like CPU and memory usage (those come from metrics-server).
>
> A useful way to think about it: etcd stores what Kubernetes WANTS to exist and the last-known STATUS of things. The actual reality of containers running is on the worker nodes, not in etcd."

---

### Q4. What port does etcd listen on? What is each port used for?

**Answer:**

> "etcd uses two ports. Port **2379** is the client port — this is where the API Server connects to read and write data. It's also the port you use when running etcdctl commands manually.
>
> Port **2380** is the peer port — this is where etcd nodes communicate WITH EACH OTHER. In a 3-node etcd cluster, the nodes send Raft messages to each other over port 2380 to stay in sync and run leader elections.
>
> Simple way to remember: 2379 = client-to-etcd traffic. 2380 = etcd-to-etcd traffic."

---

### Q5. What is Raft and why does etcd use it?

**Answer:**

> "Raft is a consensus algorithm — it's the system that keeps all etcd nodes perfectly in sync with each other even when some nodes are slow or unavailable.
>
> The problem it solves: if you have 3 etcd nodes and a write comes in, all three need to have the same data. But what if one node is slow? Do you wait for all three? If you do, one slow node can block all writes. If you don't wait, nodes can get out of sync.
>
> Raft solves this with majority voting. One node is the leader. Every write goes to the leader. The leader sends the write to all followers. Once the majority — 2 out of 3 — confirm they received it, the leader commits it and replies 'success.' The third slow node catches up later.
>
> This means writes are guaranteed to survive even if one node crashes immediately after — because the data is already on two nodes, and two nodes is a majority. You never lose a confirmed write."

---

### Q6. What is quorum in etcd? Why do we use odd numbers of etcd nodes?

**Answer:**

> "Quorum is the minimum number of etcd nodes that must be alive and agreeing for the cluster to accept writes. The formula is: floor of (total nodes divided by 2) plus 1.
>
> For 3 nodes, quorum is 2. You can lose 1 node and still have 2 working — still above quorum. Cluster stays healthy.
> For 5 nodes, quorum is 3. You can lose 2 nodes and still have 3 working — cluster stays healthy.
>
> We use odd numbers because even numbers give you no extra fault tolerance. 4 nodes has the same quorum (3) as 3 nodes — you can still only lose 1 node with either. But 4 nodes costs more and adds complexity. So 3 gives you the same fault tolerance as 4 at lower cost. Same pattern with 5 vs 6.
>
> In production Kubernetes, 3 etcd nodes is standard for most clusters. 5 nodes is used for critical environments where losing 2 nodes simultaneously is a realistic concern."

---

## SECTION B — ARCHITECTURE AND INTERNALS

---

### Q7. Explain the Raft leader election process.

**Answer:**

> "Each etcd node is in one of three states: leader, follower, or candidate. Normally you have one leader and all others are followers. The leader sends heartbeats to followers every 100-150 milliseconds to say 'I'm still alive.'
>
> If a follower doesn't receive a heartbeat for a random timeout period — between 150 and 300 milliseconds — it assumes the leader is dead. It becomes a candidate, votes for itself, and sends vote requests to all other nodes. If it gets votes from a majority, it becomes the new leader and starts sending heartbeats.
>
> The random timeout is important — if all followers had the same timeout, they'd all become candidates simultaneously and split the vote. Random timeouts mean one node typically starts the election slightly before others and wins before they even start.
>
> In practice, a new leader is elected in under 1 second after the old leader fails. During that brief window, no writes are accepted by the cluster."

---

### Q8. What is the Write-Ahead Log in etcd?

**Answer:**

> "The Write-Ahead Log, or WAL, is how etcd ensures it never loses data even if the machine crashes mid-write.
>
> Before etcd does ANYTHING with a write, it first writes that write to the WAL file on disk and flushes it. Only after the WAL is safely on disk does etcd proceed to apply the change to its in-memory data and respond to the client.
>
> So even if the machine loses power immediately after confirming a write, when etcd restarts it reads the WAL and replays all committed entries. Nothing is lost.
>
> This is why disk I/O latency is so critical for etcd. Every single write requires a disk flush to the WAL before it can complete. If your disk is slow — like an HDD with 10ms latency — every write takes at least 10ms. Under heavy load that creates a bottleneck. This is why etcd should always run on SSD or NVMe storage."

---

### Q9. What is compaction and defragmentation in etcd? Why are they needed?

**Answer:**

> "etcd keeps a full history of every write ever made — called revisions. Every time you create, update, or delete an object, etcd keeps the old version too. This history is needed for the Watch API to tell clients 'here are all changes since revision X.'
>
> Over time this history accumulates and makes the database large and slow. Compaction removes old revision history — it tells etcd 'you can forget everything before revision N.' After compaction, the database still contains all current objects but drops the old history.
>
> Defragmentation actually reclaims the disk space after compaction. Compaction marks space as reusable internally, but the actual file on disk doesn't shrink. Defragmentation rewrites the database file, physically removing the gaps. After defrag, the file is smaller on disk.
>
> Kubernetes runs automatic compaction every 5 minutes by default. Defragmentation must be scheduled manually — typically as a maintenance job running monthly or when database size grows beyond a threshold like 2GB."

---

### Q10. What happens to the cluster if etcd goes down?

**Answer:**

> "If etcd goes down completely, the effects split into two categories: what stops immediately and what keeps working.
>
> What stops immediately: the API Server can no longer process requests because it can't read or write state. kubectl commands start timing out. No new deployments, no scaling, no config changes — nothing new can happen.
>
> What keeps working: pods that are already running continue to run. kubelet on each worker node maintains the containers locally and doesn't need etcd to keep them alive. Services keep routing traffic because kube-proxy's iptables rules are already written on each node.
>
> The danger: if a pod crashes while etcd is down, no replacement is created — the ReplicaSet Controller can't read desired state. If a node fails, no pods are rescheduled.
>
> When etcd comes back up, everything resumes. Controllers run their reconciliation loops and catch up on anything that drifted.
>
> This is why regular etcd snapshots are critical. If etcd is lost permanently with no backup, all cluster configuration is gone — the running pods survive but you have no record of what was supposed to be running."

---

### Q11. What is the difference between etcd compaction and etcd defragmentation?

**Answer:**

> "Think of etcd's database like a filing cabinet. Compaction is like pulling out old files and shredding them — the drawer now has gaps where the old files were, but the cabinet still takes up the same physical space. The space is logically free but not physically reclaimed.
>
> Defragmentation is like reorganizing the entire cabinet — taking out all the remaining files, compressing them together, and returning them to a smaller cabinet. Now the physical space is actually smaller.
>
> You must always compact before you defrag. Compaction removes the old data logically. Defrag then physically rewrites the remaining data into a compact file.
>
> In a production cluster, you'd compact frequently (Kubernetes does it automatically every 5 minutes) and defrag periodically — once a month or when the DB file is significantly larger than the actual data size, which is a sign of fragmentation."

---

### Q12. Is data in etcd encrypted by default?

**Answer:**

> "No — data in etcd is NOT encrypted at rest by default. Kubernetes Secrets are stored in etcd as base64 encoded values, not encrypted. Base64 is just an encoding scheme for safe transport — anyone who can access the etcd data directory or a snapshot file can decode all secrets in seconds with a simple base64 decode command.
>
> Encryption in transit IS enabled by default — all communication between the API Server and etcd uses TLS, so data moving between them is encrypted on the network.
>
> To enable encryption at rest, you configure the API Server with an EncryptionConfiguration file that tells it to encrypt specific resource types — typically Secrets — before writing to etcd using AES-GCM or AES-CBC.
>
> In our banking environment, we had both etcd encryption at rest enabled for Secrets AND we used AWS Secrets Manager with External Secrets Operator. So actual credentials never lived in etcd — only references to them. This gave us KMS key management, audit logging, and automatic rotation on top of the cluster-level encryption."

---

## SECTION C — BACKUP AND RESTORE

---

### Q13. How do you back up etcd? What command do you use?

**Answer:**

> "etcd backup is done using the `etcdctl snapshot save` command. The command needs four things: the endpoint to connect to, and the three TLS files — the CA certificate, the client certificate, and the client private key.
>
> The command looks like:
> `ETCDCTL_API=3 etcdctl snapshot save /backup/snapshot.db --endpoints=https://127.0.0.1:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key`
>
> The `ETCDCTL_API=3` at the front is important — it tells etcdctl to use API version 3, which is the current version. Without it, it defaults to version 2 and some commands behave differently or fail.
>
> After saving, you verify the backup with `etcdctl snapshot status` which shows the hash, revision number, total keys, and file size. In production, we scheduled this as a CronJob running every 6 hours, saving snapshots to S3 with 30-day retention."

---

### Q14. How do you restore etcd from a backup?

**Answer:**

> "Restoring etcd is a four-step process.
>
> First, stop the API Server so nothing can write to etcd during the restore. On a kubeadm cluster, you do this by temporarily moving the API Server's static pod manifest out of `/etc/kubernetes/manifests/`.
>
> Second, run `etcdctl snapshot restore` pointing to your backup file and specifying a new data directory. You also need to provide the cluster member name and initial cluster configuration — this is because the restore creates a fresh etcd member that needs to know who it is.
>
> Third, update the etcd static pod manifest to point its `--data-dir` flag to the new restored directory.
>
> Fourth, move the API Server manifest back so it starts again, and etcd starts using the restored data.
>
> The critical thing to remember: snapshot restore does NOT restore into the existing etcd data directory. It creates a fresh directory. You must update etcd's configuration to use it. Trying to restore into the live data directory while etcd is running will corrupt data."

---

### Q15. What is the `ETCDCTL_API=3` environment variable and why must you set it?

**Answer:**

> "etcdctl supports two versions of the etcd API — version 2 and version 3. They are not fully compatible. All modern Kubernetes clusters use etcd v3, which uses the v3 API.
>
> If you run etcdctl without setting `ETCDCTL_API=3`, it defaults to API version 2. In v2, commands like `snapshot save` either don't exist or behave differently. You might get an error or silently do the wrong thing.
>
> Setting `ETCDCTL_API=3` as an environment variable tells etcdctl to use the v3 API for all commands in that session. You can either set it with `export ETCDCTL_API=3` at the start of your session, or prefix it directly to each command like `ETCDCTL_API=3 etcdctl snapshot save ...`.
>
> In CKA exam scenarios — and in real incidents — forgetting this is a common mistake that makes the snapshot command fail. Always check this first if etcdctl commands are not working."

---

### Q16. In a production cluster, how would you manage etcd backups?

**Answer:**

> "In production, I would set up automated, versioned, offsite backups with verification.
>
> Mechanically: a Kubernetes CronJob runs every 6 hours. It runs a pod on the control plane node (using nodeSelector or tolerations to ensure it lands there), executes `etcdctl snapshot save` to a local path, then uploads the file to S3 with a timestamped filename using `aws s3 cp`. The S3 bucket has versioning enabled and a lifecycle policy to delete backups older than 30 days.
>
> Verification: after each backup, the CronJob runs `etcdctl snapshot status` on the file and checks that TOTAL KEYS is greater than zero and the HASH is present. If verification fails, an alert fires to PagerDuty.
>
> Restoration testing: once a quarter, we restore the latest backup to a separate test cluster and verify that the cluster configuration matches expectations. A backup that was never tested is a backup you can't trust.
>
> Certificate consideration: the etcdctl command needs the CA cert, server cert, and key. These must be accessible to the backup job — in our setup we mounted them from the host node using a hostPath volume (or used a Secret containing copies of the certs)."

---

## SECTION D — PERFORMANCE AND TROUBLESHOOTING

---

### Q17. The cluster feels slow — kubectl commands are taking 10+ seconds. What do you check first and why?

**Answer:**

> "The first thing I check is etcd health and disk I/O, because etcd is on the critical path of every single API operation. Slow etcd means slow API Server means slow kubectl means slow everything.
>
> I start with `etcdctl endpoint status --write-out=table` to see DB SIZE and whether there are any errors. If DB SIZE is over 2GB, accumulated old revisions are likely causing slowness.
>
> Then I SSH to the etcd node and run `iostat -x 2 5` to check disk I/O latency. I look at the `await` column — the average time in milliseconds for I/O operations. If it's above 10ms, the disk is the bottleneck. etcd needs SSD. If it's running on an HDD or a slow network disk (like an EBS `gp2` volume at high utilization), that explains the slowness.
>
> I also check etcd logs for specific messages: 'slow apply,' 'request timed out,' or 'took too long to execute.' These are etcd's own warnings that it's struggling.
>
> Fix order: if it's disk — migrate to faster storage. If it's database size — run compaction and defragmentation. If it's resource contention — ensure etcd is on dedicated nodes, not sharing with other workloads."

---

### Q18. How do you check if etcd is healthy?

**Answer:**

> "I use three checks.
>
> First: `etcdctl endpoint health` — this directly tests whether etcd can commit a proposal. If it returns 'is healthy: successfully committed proposal,' etcd is working. If it times out or returns an error, etcd is unhealthy.
>
> Second: `etcdctl endpoint status --write-out=table` — this shows DB size, current leader, raft term, and raft index. I check that IS LEADER is true for exactly one node (in a multi-node setup), that RAFT INDEX and RAFT APPLIED INDEX are equal (if applied is behind index, etcd is falling behind on commits), and that ERRORS column is empty.
>
> Third: check etcd pod logs with `kubectl logs etcd-controlplane -n kube-system` and look for warnings about slow applies, disk I/O issues, or leader changes. Frequent leader changes (high raft term) indicate an unstable cluster — usually a network or disk problem."

---

### Q19. What is a split-brain scenario in etcd? How does Raft prevent it?

**Answer:**

> "Split-brain is a dangerous scenario in distributed systems where a network partition splits your cluster into two groups, each of which thinks the other is dead. If both groups could accept writes independently, you'd have two different versions of truth — when the partition heals, there's no way to know which version is correct.
>
> For example: 3 etcd nodes, a network partition splits them into group A (1 node) and group B (2 nodes). If both groups accepted writes, they'd diverge.
>
> Raft prevents this with quorum. Group A has 1 node — that's below quorum (need 2 of 3). So group A refuses all writes and goes into read-only mode. Group B has 2 nodes — that's exactly quorum. Group B continues accepting writes normally. When the partition heals, group A syncs from group B. Only one version of truth ever existed.
>
> This is the mathematical guarantee behind odd-number etcd clusters. A network partition always creates one group with majority and one without — the majority side continues, the minority side stops. You can never have two groups that both have majority."

---

### Q20. etcd is showing `NOSPACE` errors. What happened and how do you fix it?

**Answer:**

> "NOSPACE means the etcd database has hit its maximum size limit. By default, etcd allows a maximum database size of 2GB (configurable with `--quota-backend-bytes`). When this limit is hit, etcd rejects ALL writes — the entire cluster stops accepting new objects or updates.
>
> This usually happens because: compaction wasn't running or wasn't running frequently enough, causing old revisions to accumulate; or someone deployed a lot of objects quickly and the database grew faster than compaction could keep up.
>
> To fix it: First, run manual compaction to remove old revisions — `etcdctl compact <current-revision>`. Second, run defragmentation to physically reclaim the space — `etcdctl defrag`. Third, if the cluster is still in NOSPACE state, you may need to raise the quota temporarily to get the defrag to work — edit the etcd manifest to increase `--quota-backend-bytes`.
>
> To prevent it: ensure Kubernetes' auto-compaction is enabled (it is by default, compacting every 5 minutes), keep `revisionHistoryLimit` on Deployments set low (3-5 instead of default 10), and clean up completed Jobs and old resources regularly."

---

### Q21. What is an etcd learner node?

**Answer:**

> "A learner node is a new type of etcd member introduced to safely add new nodes to an etcd cluster without risking quorum.
>
> The problem: when you add a new etcd member to a 3-node cluster, that member starts with no data. It needs to catch up by syncing from the leader. During this sync — which can take minutes if the database is large — the new node is technically a member but can't vote. If the leader crashes during this time, you now have a 4-member cluster where only 2 original members are healthy — below the quorum of 3 — and the cluster stops.
>
> A learner node joins the cluster but does NOT count toward quorum and does NOT vote. It just receives data and syncs silently. Once it's fully caught up, you promote it to a full voting member. This way, you never risk your quorum during the sync process.
>
> In `etcdctl endpoint status`, the `IS LEARNER` column shows whether a node is a learner. Normally it's false for all voting members."

---

## SECTION E — SCENARIO-BASED QUESTIONS

---

### Q22. Your etcd backup is 4.5MB but you have 500+ Kubernetes objects. Is that normal?

**Answer:**

> "Yes, that's completely normal and actually shows how efficient etcd is. The snapshot file contains all object specifications in protobuf format — a binary encoding that's much more compact than JSON or YAML.
>
> A typical Kubernetes object — a Deployment, a Pod spec, a ConfigMap — is maybe 1-5KB when stored in protobuf format in etcd. 500 objects × 3KB average = about 1.5MB for the objects themselves. Add metadata, revision history that hasn't been compacted yet, and etcd internal structures — 4.5MB is very reasonable.
>
> I would start being concerned when the snapshot grows above 1-2GB — that usually indicates compaction isn't happening or there's a very large accumulation of Events and old ReplicaSets. At 6GB+ you're approaching the default 8GB etcd quota limit and performance will be degraded."

---

### Q23. You run `kubectl get pods` and it hangs for 30 seconds then returns. No pod issues are visible. What do you investigate?

**Answer:**

> "A `kubectl get` command hanging is almost always an API Server or etcd problem. When kubectl sends a GET request to the API Server, the API Server queries etcd. If etcd is slow to respond, the API Server waits, and kubectl waits.
>
> My investigation order: first check if the API Server pod itself is healthy — `kubectl get pods -n kube-system` specifically for the API Server pod. If that also hangs, it confirms API Server or etcd.
>
> SSH to the control plane node and check etcd pod logs directly: `sudo crictl logs <etcd-container-id>` or check journald. Look for 'slow apply' or 'request timed out' messages.
>
> Check disk I/O on the etcd node: `iostat -x` — if await is high, disk is the bottleneck.
>
> Check etcd database size: if it's grown to several GB without regular compaction, read operations slow down because etcd has to search through a large fragmented database.
>
> In our banking cluster we had this exact symptom. etcd database had grown to 3.5GB because we'd been doing multiple deployments per day for 6 months and never cleaned old ReplicaSets. Compaction + defrag dropped it to 400MB and kubectl response time went from 30 seconds back to under 1 second."

---

### Q24. A junior engineer accidentally ran `kubectl delete namespace production`. The namespace is stuck in Terminating. What do you do and how could etcd backup have helped?

**Answer:**

> "This is a two-part answer — immediate response and post-mortem.
>
> Immediate response: the namespace is stuck Terminating because a resource inside it has a finalizer that isn't being cleaned up. I'd run `kubectl get all -n production` to see what's still there, then `kubectl api-resources --verbs=list --namespaced -o name` to check for custom resources that might be blocking.
>
> If I need to force it: `kubectl patch namespace production -p '{"spec":{"finalizers":[]}}' --type=merge` — this removes the finalizers and allows the namespace to delete. But this should only be done after confirming the resources inside are truly safe to lose.
>
> If the namespace is deleted and everything inside it is gone — etcd backup is the ONLY way to recover. I'd restore the etcd snapshot to a test cluster first, `kubectl get all -n production` from there to get all the YAML definitions, then re-apply them to the production cluster. This is exactly why we backup etcd and not just individual object YAMLs — a snapshot contains EVERYTHING including state that was never saved as a YAML file.
>
> Post-mortem: add RBAC restrictions so only senior engineers and CI/CD service accounts can delete namespaces. Add a confirmation step in the deployment pipeline for destructive operations."

---

## SECTION F — QUICK FIRE QUESTIONS

| Question | Answer |
|----------|--------|
| What type of database is etcd? | Distributed key-value store |
| What algorithm does etcd use for consistency? | Raft consensus |
| What port does etcd use for client connections? | 2379 |
| What port does etcd use for peer (node-to-node) connections? | 2380 |
| How many etcd nodes can you lose in a 3-node cluster? | 1 |
| How many etcd nodes can you lose in a 5-node cluster? | 2 |
| Why use odd numbers of etcd nodes? | Maximize fault tolerance — even numbers give same tolerance as the odd number below |
| Is data in etcd encrypted by default? | No — encrypted in transit (TLS), but NOT encrypted at rest by default |
| What is a snapshot in etcd? | A point-in-time backup of the entire database |
| What command takes an etcd backup? | `etcdctl snapshot save` |
| What command verifies an etcd backup? | `etcdctl snapshot status` |
| What environment variable must you set before using etcdctl? | `ETCDCTL_API=3` |
| What is quorum? | Minimum nodes that must agree for writes to succeed |
| What is compaction in etcd? | Removing old revision history (logical space) |
| What is defragmentation in etcd? | Physically rewriting the database file to reclaim disk space |
| What is the default max etcd database size? | 2GB (configurable with `--quota-backend-bytes`) |
| Which node in a Raft cluster accepts writes? | The leader only |
| What happens to writes when etcd has no quorum? | All writes are rejected — cluster goes read-only |
| What is a WAL? | Write-Ahead Log — etcd writes here first before confirming any write |
| Which Kubernetes component is the only one that talks to etcd? | The API Server |

---

## SECTION G — THINGS THAT SOUND IMPRESSIVE IN AN INTERVIEW

Use these naturally — understand the idea, don't memorize word for word:

1. **"etcd is the only component where data loss means cluster configuration loss. Everything else — pods, controllers, kubelet — is stateless or can be rebuilt. etcd is the one thing you absolutely cannot lose without a backup."**

2. **"We had an incident where a runaway process was creating thousands of ConfigMap objects per minute. etcd hit the 2GB NOSPACE quota within 3 hours. We caught it via a Prometheus alert on etcd database size growth rate — which is a metric we specifically added after reading the etcd operational guide."**

3. **"Raft's quorum requirement is what gives etcd its safety guarantee — you can never have a split-brain because a network partition always produces one group with majority and one without. The majority side continues; the minority side stops. When the partition heals, the minority syncs from majority."**

4. **"In our banking environment, we treated etcd backups like database backups — scheduled every 6 hours, stored in S3 with versioning, and most importantly, tested quarterly by actually restoring to a test cluster. An untested backup is just a file — you only know it works when you restore it."**

5. **"The fact that etcd stores data in protobuf format instead of JSON means the API Server has to serialize and deserialize on every read and write. This is why etcd disk I/O latency is more important than etcd node CPU — the CPU cost of protobuf encoding is minimal, but waiting for disk confirmation on every WAL write is the real bottleneck."**

---

*File: K8s_etcd_Interview_Questions.md*
*Repository: Interview_Preparation_2026 → Kubernetes/*
*Companion file: K8s_etcd_Concept_and_Lab.md*
