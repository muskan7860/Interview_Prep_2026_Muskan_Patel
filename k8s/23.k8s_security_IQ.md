# Kubernetes Security — Interview Questions & Answers
## RBAC · ServiceAccount · NetworkPolicy · PodSecurityContext · securityContext
> Target: 4 Years DevOps Experience | Senior-Level Interviews
> Style: Easy to memorize + Professional to say out loud
> Tricky questions marked with ⚠️

---

## 💡 HOW TO USE THIS FILE

- Read the question first
- Try answering in your own words
- Then read the answer — written to say out loud in an interview
- Banking/production examples are included — use them naturally

---

## SECTION A — RBAC BASICS

---

### Q1. What is RBAC in Kubernetes? What problem does it solve?

**Answer:**

> "RBAC stands for Role-Based Access Control. It controls who can perform which actions on which Kubernetes API objects. It answers the authorization question — after Kubernetes confirms who you are (authentication), RBAC decides whether you are ALLOWED to do what you're asking.
>
> The problem it solves: without RBAC, any authenticated user or service can do anything — list all secrets, delete deployments, modify RBAC rules themselves. In a company with 50 developers and 100 services, this is catastrophic. A junior developer could accidentally delete a production database. A compromised pod could steal all secrets across the entire cluster.
>
> RBAC gives you the principle of least privilege — each human and each pod gets only the exact permissions needed for their job, nothing more. In our banking project, the payment service could read only its own secrets in the banking namespace. The CI/CD pipeline could update Deployments but not delete them. Monitoring could read pods across all namespaces but not modify anything. Every access was explicit and auditable."

---

### Q2. What is the difference between Role and ClusterRole?

**Answer:**

> "Both define WHAT actions are allowed on WHAT resources. The difference is SCOPE.
>
> A Role is namespaced — it only applies within one specific namespace. A Role named `pod-reader` in the `banking` namespace grants permission to read pods ONLY in the banking namespace. A subject with this Role cannot see pods in any other namespace.
>
> A ClusterRole is cluster-wide — it applies across all namespaces. A ClusterRole named `pod-reader` grants permission to read pods in ALL namespaces. ClusterRoles are also the ONLY way to grant permissions on non-namespaced resources like nodes, PersistentVolumes, StorageClasses, and Namespaces themselves — these resources have no namespace, so a namespaced Role cannot reference them.
>
> There's a powerful combination: using a ClusterRole with a RoleBinding (not ClusterRoleBinding). This grants the ClusterRole's permissions but ONLY within the RoleBinding's namespace. It's useful for sharing permission templates — define once as ClusterRole, reuse across namespaces via different RoleBindings."

---

### Q3. What is the difference between RoleBinding and ClusterRoleBinding?

**Answer:**

> "Both assign permissions to a subject. The difference is scope.
>
> RoleBinding assigns permissions within ONE namespace — the namespace where the RoleBinding lives. Even if it references a ClusterRole, the permissions only apply in that namespace.
>
> ClusterRoleBinding assigns permissions across ALL namespaces and for cluster-level resources. A ClusterRoleBinding giving someone the `pod-reader` ClusterRole means they can read pods everywhere.
>
> Common mistake: thinking you can use a ClusterRoleBinding to restrict someone to one namespace. You cannot — ClusterRoleBinding is always cluster-wide. To restrict to one namespace, use a RoleBinding (not ClusterRoleBinding) even if you're referencing a ClusterRole.
>
> In our banking project: monitoring SA had a ClusterRoleBinding to read pods and nodes everywhere (needs to scrape all namespaces). Payment SA had a RoleBinding to read secrets only in the banking namespace. Two different subjects, two different binding types, correctly scoped."

---

### Q4. ⚠️ What is `apiGroups` in a Role? Why is it important and how do you know which group a resource belongs to?

**Answer:**

> "apiGroups specifies which Kubernetes API group the resources in that rule belong to. Kubernetes organizes resources into API groups — not all resources are in the same group, and you must specify the correct group or the rule won't work.
>
> The core API group — represented by an empty string `['']` — contains the most fundamental resources: pods, services, configmaps, secrets, endpoints, namespaces, nodes, PersistentVolumes, PersistentVolumeClaims, and ServiceAccounts.
>
> The `apps` group contains higher-level workload resources: Deployments, ReplicaSets, StatefulSets, DaemonSets.
>
> The `batch` group contains Jobs and CronJobs. `networking.k8s.io` contains Ingresses and NetworkPolicies. `autoscaling` contains HorizontalPodAutoscalers. `rbac.authorization.k8s.io` contains Roles, ClusterRoles, RoleBindings, ClusterRoleBindings.
>
> To find the correct group for any resource, run `kubectl api-resources`. The output shows NAME, SHORTNAMES, APIVERSION, NAMESPACED, and KIND. The APIVERSION column shows the group — if it says `apps/v1`, the group is `apps`. If it says `v1`, the group is the empty string core group.
>
> A common mistake is writing a rule for Deployments with `apiGroups: ['']`. This is wrong — Deployments are in `apps`, not the core group. The rule silently does nothing because there are no Deployments in the core group."

---

### Q5. ⚠️ You want to give a user read access to pods in namespace A, and write access to deployments in namespace B. How do you set this up?

**Answer:**

> "This requires two separate RoleBindings — one per namespace. You cannot do cross-namespace permissions in a single object.
>
> First, create a Role in namespace A for pod reading, then a RoleBinding in namespace A binding that user to the Role.
>
> Second, create a Role in namespace B for deployment write access, then a RoleBinding in namespace B binding that same user to that Role.
>
> Or more elegantly with ClusterRoles: create a ClusterRole `pod-reader` with get/list/watch on pods, and a ClusterRole `deployment-writer` with create/update/patch on deployments. Then create a RoleBinding in namespace A binding the user to `pod-reader` ClusterRole, and a RoleBinding in namespace B binding the user to `deployment-writer` ClusterRole.
>
> The ClusterRole approach is better because the permission templates are reusable — any namespace can grant these permissions to any user via its own RoleBinding, without recreating the permission rules each time."

---

## SECTION B — SERVICEACCOUNT

---

### Q6. What is a ServiceAccount and why does every pod need one?

**Answer:**

> "A ServiceAccount is the identity Kubernetes assigns to a pod when that pod communicates with the Kubernetes API Server. When a pod calls the API — to list other pods, to read a ConfigMap, to create a Job — the API Server needs to know who is calling. The ServiceAccount is that identity.
>
> Every pod in Kubernetes is assigned a ServiceAccount whether you specify one or not. If you don't specify one, Kubernetes uses the `default` ServiceAccount in the pod's namespace. Every namespace gets a `default` ServiceAccount created automatically.
>
> The problem with using the default SA: historically the default SA had broad implicit permissions, and many clusters still configure it with more access than needed. A compromised pod using the default SA can potentially do a lot of damage.
>
> Best practice: create specific ServiceAccounts per workload with only the permissions that workload needs. The payment service gets `payment-sa` which can only read payment secrets. The reporting service gets `reporting-sa` which can only read ConfigMaps. If either is compromised, the blast radius is contained to what that specific SA can do."

---

### Q7. What is `automountServiceAccountToken` and why would you set it to false?

**Answer:**

> "By default, Kubernetes automatically mounts the ServiceAccount's authentication token as a file inside every pod at `/var/run/secrets/kubernetes.io/serviceaccount/token`. The pod process can read this file and use it to authenticate to the Kubernetes API.
>
> For pods that need to call the Kubernetes API — custom controllers, operators, monitoring agents — this is necessary and correct.
>
> For most application pods — web servers, microservices, batch jobs — they have no reason to ever call the Kubernetes API. They serve HTTP requests, process data, write to databases. The SA token sitting in their filesystem is an unnecessary attack surface. If the pod is compromised, the attacker has a token they can use to call the Kubernetes API with whatever permissions the SA has.
>
> Setting `automountServiceAccountToken: false` on the ServiceAccount or the pod spec prevents the token from being mounted. The pod cannot call the Kubernetes API — which is fine because it didn't need to.
>
> In our banking project, we set `automountServiceAccountToken: false` on all ServiceAccounts by default. Only specific SAs that genuinely needed API access — our custom operators and the CI/CD agent — had it enabled with explicit opt-in in their pod specs."

---

### Q8. ⚠️ What is IRSA and why is it better than putting AWS credentials in environment variables?

**Answer:**

> "IRSA stands for IAM Roles for Service Accounts. It's AWS's mechanism for giving Kubernetes pods secure, automatic access to AWS services without storing AWS access keys anywhere.
>
> The old way: put `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` in a Kubernetes Secret. Inject them as environment variables. The pod uses them to call AWS APIs. Problems: credentials are static and long-lived, stored in etcd (even as Secrets they're just base64), if leaked you must manually rotate them, no audit trail of which pod used which credential.
>
> IRSA works differently. You create a Kubernetes ServiceAccount and annotate it with an IAM Role ARN. AWS EKS sets up an OIDC trust between the cluster and AWS IAM. When the pod starts, the SA token (a JWT) is mounted in the pod. The AWS SDK running in the pod automatically exchanges this short-lived JWT for temporary AWS credentials using AWS STS. The credentials expire in 1 hour and are automatically rotated.
>
> Benefits: no long-lived keys anywhere, credentials are temporary and auto-rotated, each pod gets exactly the IAM permissions of its SA's linked IAM role, AWS CloudTrail shows which pod called which AWS API. This is the mandatory approach for production banking on EKS."

---

## SECTION C — NETWORKPOLICY

---

### Q9. What is NetworkPolicy? What is the default network behavior without it?

**Answer:**

> "NetworkPolicy is a Kubernetes API object that acts as a Layer 3 and Layer 4 firewall for pod-to-pod traffic. It controls which pods can send traffic to which other pods, on which ports and protocols.
>
> The default behavior without NetworkPolicy is critical to understand: ALL traffic is allowed between ALL pods in ALL namespaces. There is no default deny. Any pod can reach any other pod anywhere in the cluster by default. This is a deliberate design choice — flat networking makes service discovery simple — but it means without NetworkPolicy you have zero network isolation.
>
> NetworkPolicy works as opt-in restriction. Once you create a NetworkPolicy that selects certain pods, those selected pods now have default-deny for the traffic types you specified (Ingress, Egress, or both), and only explicitly listed sources/destinations are allowed. Pods NOT selected by any NetworkPolicy remain wide open.
>
> Important: NetworkPolicy only works if your CNI plugin supports it. Flannel does NOT enforce NetworkPolicies — they're just ignored. Calico, Cilium, and Weave all support NetworkPolicies."

---

### Q10. ⚠️ What is the difference between AND and OR logic in NetworkPolicy `from` rules?

**Answer:**

> "This is the most commonly misunderstood part of NetworkPolicy and causes serious security mistakes.
>
> In a NetworkPolicy's `from` section, you can combine `podSelector` and `namespaceSelector`. How you write them determines whether they're combined with AND or OR.
>
> AND logic — same list item, indented together. `podSelector` and `namespaceSelector` at the same indentation level under one dash. This means: allow traffic from pods that BOTH match the podSelector AND are in a namespace that matches the namespaceSelector.
>
> OR logic — separate list items, each with their own dash. `podSelector` and `namespaceSelector` as separate dash items. This means: allow traffic from pods that match the podSelector (in ANY namespace) OR from any pod in a namespace that matches the namespaceSelector.
>
> Why this matters for security: OR logic with just a namespaceSelector means ANY pod in that namespace can reach you — even a freshly deployed malicious pod. AND logic means it must be a specific pod with a specific label in a specific namespace — much more restrictive. In banking, we always used AND logic for database policies: only pods with label `app=payment` AND in namespace `banking` could reach the database. Using OR would mean any pod in banking could reach the database, which is too broad."

---

### Q11. ⚠️ You applied Egress NetworkPolicy to a pod and now DNS is broken. Why?

**Answer:**

> "This is one of the most common NetworkPolicy mistakes. When you apply an Egress policy to pods and don't explicitly allow DNS traffic, those pods can no longer resolve hostnames — including Kubernetes Service names.
>
> DNS in Kubernetes works by pods sending UDP queries to port 53 to the CoreDNS Service (usually in kube-system namespace). When you add `policyTypes: - Egress` to a NetworkPolicy with no egress rules — or egress rules that don't include DNS — ALL outgoing traffic is denied, including the DNS queries.
>
> The fix: always add a DNS egress allow rule when using Egress NetworkPolicy:
>
> ```yaml
> egress:
> - to:
>   - namespaceSelector:
>       matchLabels:
>         kubernetes.io/metadata.name: kube-system
>   ports:
>   - protocol: UDP
>     port: 53
>   - protocol: TCP
>     port: 53   # TCP DNS for large responses
> ```
>
> Also add TCP 53 — DNS uses TCP for large responses (zone transfers, some DNSSEC). Without TCP 53, some DNS queries silently fail.
>
> The symptom of this mistake: pod is Running, Readiness probe fails, application logs show 'connection refused' or 'name resolution failure' — pointing to services that definitely exist. First thing to check whenever a pod starts failing after a NetworkPolicy change: can it resolve DNS? `kubectl exec <pod> -- nslookup kubernetes.default`."

---

### Q12. How do you implement a zero-trust network model in Kubernetes?

**Answer:**

> "Zero-trust networking means: deny everything by default, then explicitly allow only what is needed. In Kubernetes, this means applying a default-deny NetworkPolicy to every namespace, then adding specific allow policies for each legitimate communication path.
>
> Step one: in every namespace, create a NetworkPolicy with empty `podSelector: {}` (selects all pods) and both `Ingress` and `Egress` in policyTypes with no rules. This denies all traffic to and from all pods in the namespace.
>
> Step two: for each communication path that is legitimate, add a specific allow policy. Payment to database: allow payment pods to reach postgres pods on port 5432. Ingress controller to app: allow traffic from ingress-nginx namespace to app pods on port 8080. App to DNS: allow all pods to reach CoreDNS on port 53.
>
> Step three: allow monitoring. Prometheus needs to reach all pods to scrape metrics. Add a policy allowing ingress from monitoring namespace to all pods on port 9090.
>
> Step four: document every allow policy with its business justification. Policy-as-code in Git.
>
> In our banking project, this model meant that even if an attacker compromised one pod, they could only reach the specific services that pod was legitimately allowed to reach — nothing else. Lateral movement was impossible."

---

## SECTION D — POD SECURITY CONTEXT

---

### Q13. What is `securityContext` in a pod? What is the difference between pod-level and container-level?

**Answer:**

> "securityContext defines OS-level security settings that control what the container process can and cannot do at the Linux system level. It goes beyond Kubernetes RBAC — it restricts what the container can do on the underlying Linux host.
>
> Pod-level securityContext sits at `spec.securityContext` and applies to ALL containers in the pod. Settings here include: `runAsUser` (which Linux UID all containers run as), `runAsGroup` (primary GID), `fsGroup` (group ownership of mounted volumes), `runAsNonRoot` (policy: reject if any container would run as root), and `seccompProfile` (syscall filtering).
>
> Container-level securityContext sits at `spec.containers[].securityContext` and applies to ONE specific container. Settings here include everything in pod-level PLUS: `allowPrivilegeEscalation` (can the process gain more privileges), `readOnlyRootFilesystem` (make the container's filesystem read-only), `capabilities` (add or drop Linux capabilities), and `privileged` (give full host access — almost never use this).
>
> Container-level overrides pod-level for that specific container. Pod-level sets the baseline, container-level fine-tunes individual containers."

---

### Q14. ⚠️ What is `allowPrivilegeEscalation: false` and why is it critical?

**Answer:**

> "Privilege escalation is when a process gains MORE privileges than it started with. Classic examples: running `sudo`, executing a SUID binary (a program that runs as its owner rather than the caller), or calling `setuid()` to change to a different user.
>
> Without `allowPrivilegeEscalation: false`: a container running as user 1000 could potentially run a SUID binary that gives it root access. Even if you set `runAsUser: 1000`, an attacker who exploits your app and finds a SUID binary can escalate to root inside the container, then potentially escape to the host.
>
> With `allowPrivilegeEscalation: false`: the kernel blocks any attempt to gain additional privileges. SUID binaries are ignored. `sudo` fails. `setuid()` is blocked. A process running as UID 1000 is permanently UID 1000 — no escape path.
>
> This should be set to false on every production container unless you have a specific technical reason not to. In our banking security audit, finding any container without `allowPrivilegeEscalation: false` was a critical finding that had to be fixed before the deployment was approved. We added it to our Kyverno policy so containers missing this setting were rejected at admission."

---

### Q15. ⚠️ What does `readOnlyRootFilesystem: true` do and what problem does it cause?

**Answer:**

> "readOnlyRootFilesystem: true mounts the container's entire root filesystem as read-only. The container process cannot create, modify, or delete any files in the container's own filesystem.
>
> Security benefit: attackers cannot write malware to disk, cannot modify application binaries, cannot create reverse shell scripts, cannot write to log files used for covering tracks. Even if an attacker exploits the application, the read-only filesystem severely limits what they can do.
>
> The problem it causes: many applications write to their local filesystem legitimately. Log files, temp files, cache files, uploaded files, compiled templates, PID files. Setting readOnlyRootFilesystem: true breaks these apps immediately with 'Read-only file system' errors.
>
> The solution: use emptyDir volumes for every directory the app legitimately needs to write to. Common pattern:
>
> Mount `/tmp` as emptyDir for temporary files. Mount `/app/logs` as emptyDir for log files. Mount `/app/cache` as emptyDir for cache. Mount `/var/run` as emptyDir for PID files. The app writes to these volumes (which are writable) but the rest of the filesystem is read-only.
>
> You need to know your application's write patterns before enabling this. I always start by running the app with readOnlyRootFilesystem: false, then check which paths it writes to with `kubectl exec <pod> -- find / -newer /proc/1/exe -type f 2>/dev/null` during normal operation. Mount those specific paths as emptyDir, then enable readOnlyRootFilesystem."

---

### Q16. What are Linux capabilities and why do you `drop: ALL` in Kubernetes?

**Answer:**

> "Linux capabilities break the traditional binary root vs non-root permission model into fine-grained privileges. Instead of 'you are root and can do everything,' capabilities let you say 'this process can bind to port 80 but cannot change system time.'
>
> A container running as root has ALL capabilities — all 40+ of them. This includes dangerous ones like SYS_ADMIN (nearly full system control), NET_ADMIN (modify network), SYS_PTRACE (debug any process), and SYS_MODULE (load kernel modules). Even a container running as non-root by default has a subset of capabilities from its parent process.
>
> `capabilities: drop: - ALL` removes every single capability from the container. The process has literally zero special privileges beyond what a completely unprivileged user has. This is the most restrictive posture.
>
> After dropping ALL, you add back ONLY what's specifically needed: `add: - NET_BIND_SERVICE` for a web server that needs to listen on port 80 (ports below 1024 require this capability). Nothing else.
>
> In our banking project, payment containers had `drop: ALL` with no additions — they listened on port 8080 (above 1024, no capability needed), didn't touch system resources, didn't need any special privileges. The nginx reverse proxy in front needed `add: NET_BIND_SERVICE` for port 80. Security principle: minimize capabilities to the absolute minimum for the application's function."

---

### Q17. ⚠️ What is `fsGroup` and when do you need it?

**Answer:**

> "fsGroup specifies the supplementary group that all containers in the pod are given, and critically — all volumes mounted into the pod have their group ownership changed to this GID. This solves the problem of file permission mismatches between the container user and the volume owner.
>
> Here's the problem without fsGroup: you run a container as user 1000. You mount a PersistentVolume that was formatted with files owned by root (GID 0). Your container process (UID 1000) cannot write to the volume because it doesn't own the files and isn't root.
>
> With `fsGroup: 2000`: when the pod starts, Kubernetes runs a chown on the mounted volume, changing group ownership to GID 2000. The container process runs with supplementary group 2000. Process is in group 2000, files are owned by group 2000, with group write permission — the container can write to the volume.
>
> Important caveat: the chown on large volumes takes time. A PersistentVolume with millions of files might take minutes to finish the ownership change at pod startup. Newer Kubernetes versions have `fsGroupChangePolicy: OnRootMismatch` which only changes ownership if the root directory doesn't already match — much faster for volumes that were already correctly set up."

---

## SECTION E — PUTTING IT ALL TOGETHER

---

### Q18. ⚠️ A pod is running as root. How do you fix it without breaking the application?

**Answer:**

> "Running as root is a common legacy issue. Here's my systematic approach.
>
> Step one: understand what user the current container runs as. `kubectl exec <pod> -- id`. If it shows `uid=0(root)` — confirmed running as root.
>
> Step two: check if the application actually needs root. Most don't — it's just that no one set up the Dockerfile with a USER instruction. `kubectl exec <pod> -- cat /proc/1/status | grep -i uid` to see the running process user.
>
> Step three: if the image can be modified — add a non-root USER to the Dockerfile. Create a user, give them ownership of the app directory, set USER. Rebuild and redeploy.
>
> Step four: if you can't modify the image — set `runAsUser: 1000` in the securityContext. This overrides whatever the image defaults to. Test if the app still works — it might fail if it writes to directories owned by root.
>
> Step five: also add `runAsNonRoot: true` as a validation layer. `allowPrivilegeEscalation: false` to block escalation paths.
>
> Step six: test thoroughly. Some apps legitimately need a non-standard setup — for example, some databases need to chown their data directory at startup. Document exceptions with business justification.
>
> In our banking migration to non-root, we found that 60% of our containers just needed a `runAsUser: 1000` override — the apps worked perfectly. 30% needed the Dockerfile updated with a USER instruction AND directory permissions fixed. 10% needed specific capabilities we granted explicitly after `drop: ALL`."

---

### Q19. ⚠️ A developer says "my pod needs to access the Kubernetes API to list pods in its namespace." How do you set this up securely?

**Answer:**

> "This is a legitimate use case — many applications need to discover their peers. Here's the secure setup.
>
> Create a dedicated ServiceAccount for this pod — not the default SA. `kubectl create serviceaccount peer-discovery-sa -n myapp`.
>
> Create a Role with the minimum necessary permissions — only list/watch pods, nothing else:
>
> ```yaml
> rules:
> - apiGroups: ['']
>   resources: ['pods']
>   verbs: ['list', 'watch']
> ```
>
> Create a RoleBinding linking the SA to the Role in the same namespace. Now this SA can list pods in its namespace only — no other namespaces, no other resources.
>
> In the pod spec, set `serviceAccountName: peer-discovery-sa`. The token is automatically mounted. The application uses the token at `/var/run/secrets/kubernetes.io/serviceaccount/token` to authenticate, and the API Server allows only pod listing.
>
> Do NOT use the default SA, do NOT give ClusterRole permissions when namespaced permissions suffice, do NOT give write permissions if only read is needed. Verify with `kubectl auth can-i list pods --as=system:serviceaccount:myapp:peer-discovery-sa -n myapp` returns yes, and `kubectl auth can-i delete pods --as=...` returns no."

---

## SECTION F — QUICK FIRE QUESTIONS

| Question | Answer |
|----------|--------|
| RBAC stands for? | Role-Based Access Control |
| Role is namespaced or cluster-scoped? | Namespaced |
| ClusterRole is namespaced or cluster-scoped? | Cluster-scoped (no namespace) |
| To grant permissions across ALL namespaces? | ClusterRole + ClusterRoleBinding |
| To grant permissions in ONE namespace only? | Role + RoleBinding (or ClusterRole + RoleBinding) |
| apiGroups for pods/services/secrets? | `[""]` (empty string — core group) |
| apiGroups for deployments? | `["apps"]` |
| apiGroups for jobs/cronjobs? | `["batch"]` |
| How to check SA permissions? | `kubectl auth can-i <verb> <resource> --as=system:serviceaccount:<ns>:<name>` |
| List all permissions for an identity? | `kubectl auth can-i --list --as=...` |
| Default SA in every namespace? | `default` |
| automountServiceAccountToken: false does? | Prevents SA token being mounted into pod |
| IRSA stands for? | IAM Roles for Service Accounts |
| Default network behavior without NetworkPolicy? | All traffic allowed between all pods |
| NetworkPolicy requires which CNI? | Calico, Cilium, or Weave (NOT Flannel) |
| podSelector: {} selects? | ALL pods in the namespace |
| AND logic in NetworkPolicy from? | podSelector and namespaceSelector in SAME dash item |
| OR logic in NetworkPolicy from? | podSelector and namespaceSelector as SEPARATE dash items |
| Why does DNS break with Egress NetworkPolicy? | DNS queries to CoreDNS on port 53 are blocked |
| Fix for broken DNS in NetworkPolicy? | Add egress allow to kube-system on UDP/TCP port 53 |
| pod-level securityContext applies to? | ALL containers in the pod |
| container-level securityContext applies to? | ONE specific container |
| runAsNonRoot: true does? | REJECTS pod if container would run as root |
| runAsUser: 1000 does? | Runs container as Linux UID 1000 |
| fsGroup does? | Sets group ownership of mounted volumes |
| allowPrivilegeEscalation: false does? | Blocks SUID, sudo, setuid() — cannot gain more privileges |
| readOnlyRootFilesystem: true does? | Container cannot write to its own filesystem |
| capabilities: drop: ALL does? | Removes ALL Linux special privileges from container |
| privileged: true gives? | Nearly full root access to host — NEVER in production apps |
| seccompProfile: RuntimeDefault does? | Filters dangerous syscalls using container runtime defaults |
| What replaced PodSecurityPolicy? | Pod Security Admission (PSA) with Restricted/Baseline/Privileged standards |
| PSA Restricted requires? | runAsNonRoot, drop ALL capabilities, no privilege escalation |

---

## SECTION G — THINGS THAT SOUND IMPRESSIVE IN AN INTERVIEW

1. **"In our banking project we found a compromised node during a security incident. The attacker had exploited a dependency vulnerability in one of our pods. Because every pod ran as a non-root user with `allowPrivilegeEscalation: false`, `readOnlyRootFilesystem: true`, and `capabilities: drop: ALL` — they were completely stuck. They couldn't write malware, couldn't escalate to root, couldn't reach other services due to NetworkPolicy. The blast radius was one pod. Without these controls, they could have moved laterally to the database."**

2. **"The AND vs OR distinction in NetworkPolicy is a real security gotcha. We inherited a NetworkPolicy that used OR logic — allowing 'any pod with label app=payment OR any pod in the banking namespace.' The intent was 'payment pods in banking namespace' (AND). The OR meant any new pod someone created in the banking namespace could reach the database immediately, even before its RBAC was set up properly. We fixed it to AND logic and from that point forward tested every NetworkPolicy with curl from pods that SHOULD be blocked."**

3. **"I treat the default ServiceAccount like a shared login — something nobody should actually use. In every namespace we create, we immediately set the default SA to `automountServiceAccountToken: false` and create specific SAs for each workload with exactly the permissions they need. This was not standard practice in the team when I joined. After a security review, we found several pods using the default SA which in that cluster had accidentally been given cluster-admin via a misconfigured ClusterRoleBinding. We fixed it in one day, but it showed why least-privilege SA discipline matters."**

4. **"When implementing securityContext, `readOnlyRootFilesystem: true` is the one that breaks the most things but provides the most value. Our approach: deploy with it disabled first, use `strace` or `inotifywait` during load testing to find every path the application writes to, add those paths as emptyDir volume mounts, then enable readOnlyRootFilesystem. We've done this for 30+ services now. Average discovery time: 2 hours. Value: each service went from 'attacker can write malware' to 'attacker literally cannot persist anything on disk.'"**

5. **"NetworkPolicy is only as good as your CNI plugin. We spent two days debugging what looked like a NetworkPolicy bug — policies were applying but traffic was still flowing that should have been blocked. Turned out the customer had deployed the cluster with Flannel (which doesn't enforce NetworkPolicies) and we assumed it was Calico. The policies were silently ignored. Since then our cluster baseline check includes verifying the CNI type and that NetworkPolicy enforcement is actually working by testing a deny rule."**

---

*File: K8s_Security_Interview_Questions.md*
*Repository: Interview_Preparation_2026 → Kubernetes/*
*Companion file: K8s_Security_Concept_and_Lab.md*
