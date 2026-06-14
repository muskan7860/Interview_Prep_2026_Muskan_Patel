# Understanding `kubectl describe pod` Output — From Scratch

> Level: Zero to 4 Years Experience
> Author: Muskan Patel
> Based on a real nginx-pod created on KillerCoda

---

## The Story — What Happened From `kubectl apply` to Running Pod

---

### Chapter 1 — The Request Goes Out

You wrote:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.25
```

and ran `kubectl apply -f nginx-pod.yaml`.

`kubectl` converted this YAML to JSON and sent it over HTTPS to the
API Server (`https://<control-plane-ip>:6443`).

The API server checked: who are you (authentication via certificate),
are you allowed to create pods here (RBAC authorization), does this
YAML make sense as a Pod (schema validation). All passed. The API
server wrote this Pod object into etcd — but at this point, the Pod
has NO node assigned yet. It exists only as "desired state, location
unknown."

---

### Chapter 2 — The Scheduler Picks a Home

The Scheduler constantly watches etcd for pods with no `nodeName`
set. It saw `nginx-pod`, checked all nodes' available resources
against the pod's `requests` (100m CPU, 64Mi memory), and picked
`node01`.

```
Events:
  Normal  Scheduled  79s   default-scheduler  Successfully assigned default/nginx-pod to node01
```

This line is proof of that decision. The Scheduler wrote
`nodeName: node01` back into the Pod object in etcd. The Scheduler's
job is done now — it never touches the pod again.

---

### Chapter 3 — kubelet on node01 Wakes Up

Every node runs kubelet — an agent constantly watching etcd (via the
API server) for "any pods assigned to MY node that I don't know about
yet?"

kubelet on `node01` saw `nginx-pod` was just assigned to it. Time to
bring it to life.

---

### Chapter 4 — The Pause Container is Born First

Before touching the `nginx` container at all, kubelet asks containerd
to create the **pause container** — an empty, do-nothing container
whose ONLY job is to hold open a network namespace and an IP address.

```
IP:               192.168.1.91
```

This IP belongs to the pause container. It's the Pod's IP. Any
container in this pod will SHARE this IP.

---

### Chapter 5 — How Does the YAML Know Where to Get the Image?

This is the big question: "How does `image: nginx:1.25` know to pull
from Docker Hub, and how do we know it's an OFFICIAL image?"

#### The Story Version

Imagine you tell a courier: "Go get me a package called `nginx:1.25`."

You did NOT give a full address. But the courier has a DEFAULT
address book memorized — and the default entry for ANY name with no
address prefix is: "go to `docker.io/library/`".

So `nginx:1.25` SECRETLY EXPANDS to:

```
docker.io/library/nginx:1.25
```

- `docker.io` = the registry (Docker Hub — a website that stores
  container images, like GitHub stores code)
- `library/` = a special folder on Docker Hub reserved for OFFICIAL
  images, maintained by Docker Inc. and the software vendors
  themselves (not random users)
- `nginx` = the image name
- `1.25` = the tag (version)

This is NOT GitHub. GitHub stores source CODE. Docker Hub (and other
registries like AWS ECR, Google GCR, Quay.io) stores pre-built
container IMAGES — already-compiled, ready-to-run filesystems.

#### Proof From the describe Output

```
Image:          nginx:1.25
Image ID:       docker.io/library/nginx@sha256:a484819eb60211f5299034ac80f6a681b06f89e65866ce91f356ed7c72af059c
```

YOU wrote `nginx:1.25`. containerd EXPANDED it to
`docker.io/library/nginx@sha256:...` automatically. The `library/`
part tells you "this is an OFFICIAL image" — Docker Hub has a
verification badge for these.

#### How "Official" Is Decided

Docker Hub has a special program — companies like nginx, postgres,
redis, ubuntu submit their images to Docker's "Official Images"
program. Docker reviews and hosts them under the `library/`
namespace. When YOU push your OWN image, it goes under YOUR
username, like `muskanp/payment-app:v1.2` — NOT under `library/`.

```bash
# Official image (implicit library/)
image: nginx:1.25
→ becomes: docker.io/library/nginx:1.25

# Your custom image (your Docker Hub username)
image: muskanp/payment-app:v1.2
→ becomes: docker.io/muskanp/payment-app:v1.2

# AWS ECR (full address specified — no expansion)
image: 123456789.dkr.ecr.us-east-1.amazonaws.com/payment-app:v1.2
→ stays exactly as written
```

**The rule:** if you DON'T specify a registry (no dots, no slashes
before the image name), containerd ASSUMES `docker.io/library/<name>`.
If you specify ANYTHING with a registry hostname (has a `.` like
`ecr.amazonaws.com`, or a `/` with an org name like `muskanp/`), it
uses EXACTLY that.

#### The Actual Download

```
Normal  Pulling    76s   kubelet  spec.containers{nginx}: Pulling image "nginx:1.25"
Normal  Pulled     70s   kubelet  spec.containers{nginx}: Successfully pulled image "nginx:1.25" in 5.903s
```

1. kubelet told containerd: "pull `nginx:1.25`"
2. containerd expanded it to `docker.io/library/nginx:1.25`
3. containerd made an HTTPS request to Docker Hub's servers
4. Docker Hub sent back the image — as multiple compressed LAYERS
   (like a zip file split into pieces)
5. containerd downloaded all layers (71MB total — `Image size:
   71005258 bytes`), extracted them, stacked them into a filesystem
6. The sha256 hash (`a484819eb60211f5...`) is a FINGERPRINT of the
   EXACT image content — proof that "this is EXACTLY the bits I
   downloaded, nothing was tampered with"

---

### Chapter 6 — The Real Container Starts

```
Normal  Created    70s   kubelet  spec.containers{nginx}: Container created
Normal  Started    70s   kubelet  spec.containers{nginx}: Container started
```

containerd creates the `nginx` container USING the downloaded
filesystem, joins it to the SAME network namespace as the pause
container (so it gets IP `192.168.1.91` too), and starts the nginx
process inside.

```
State:          Running
  Started:      Sun, 14 Jun 2026 07:16:46 +0000
Ready:          True
```

---

### Chapter 7 — The ServiceAccount Token (Mounts Section)

```
Mounts:
  /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-s4n5c (ro)
```

This is automatically given to EVERY pod, even if you never asked for
it. Think of it as a "free ID card" — in case the app inside ever
needs to TALK to the Kubernetes API itself.

```
Volumes:
  kube-api-access-s4n5c:
    Type:                    Projected
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
```

This volume contains THREE things, automatically injected:

1. A JWT token (the "ID card") — proves "I am pod nginx-pod,
   ServiceAccount default, namespace default" — expires in 3607
   seconds (~1 hour), auto-renewed by kubelet
2. The cluster's CA certificate (`kube-root-ca.crt`) — so the app
   could verify the API server's identity if it wanted to call it
3. The namespace name — written to a file, so the app knows what
   namespace it's running in

Your nginx container never USES this — but it's there, "just in
case," for every pod.

---

### Chapter 8 — The Conditions Section

```
Conditions:
  Type                        Status
  PodReadyToStartContainers   True
  Initialized                 True
  Ready                       True
  ContainersReady             True
  PodScheduled                True
```

Think of these as 5 checkpoints a pod passes through, IN ORDER, like
a relay race. Each becomes `True` once that STAGE is complete. When
ALL are `True`, the pod is fully healthy and serving.

| Condition | What It Means | Becomes True When |
|---|---|---|
| PodScheduled | "I have been assigned to a node" | Scheduler picks a node (Chapter 2) |
| PodReadyToStartContainers | "My sandbox (pause container + network namespace) is ready" | The pause container is created (Chapter 4) |
| Initialized | "All my Init Containers (if any) finished successfully" | If you have NO init containers, this becomes True immediately |
| ContainersReady | "ALL my regular containers pass their readiness probes" | nginx is Running AND its readiness probe passes — since this pod has NO readiness probe, Kubernetes assumes "Running = Ready" |
| Ready | "The OVERALL pod is ready to receive traffic" | Final summary — True only when Initialized AND ContainersReady are BOTH True |

The order matters: PodScheduled → PodReadyToStartContainers →
Initialized → ContainersReady → Ready. If ANY is stuck at `False`,
that tells you EXACTLY which stage your pod is stuck at.

```bash
# See conditions in raw form
kubectl get pod nginx-pod -o jsonpath='{.status.conditions}' | jq
```

---

## The Whole Story in One Diagram

```
You run kubectl apply
       |
       v
API Server validates -> writes Pod to etcd (no node yet)
       |
       v
Scheduler picks node01 -> PodScheduled: True
       |
       v
kubelet on node01 notices the pod
       |
       v
containerd creates PAUSE container (gets IP 192.168.1.91)
       |                           -> PodReadyToStartContainers: True
       v
(No init containers)              -> Initialized: True
       |
       v
kubelet asks containerd to pull "nginx:1.25"
       |
       v
containerd expands to docker.io/library/nginx:1.25
       |
       v
Downloads from Docker Hub (71MB, verified by sha256 hash)
       |
       v
Container created, joins pause container's network namespace
       |
       v
nginx process starts                -> ContainersReady: True
       |
       v
ServiceAccount token auto-mounted (kube-api-access volume)
       |
       v
ALL conditions True                  -> Ready: True
       |
       v
Endpoint Controller would now add 192.168.1.91 to any
matching Service's endpoints (if a Service existed)
```

---

# Interview Questions Based on This Pod Output

---

**Q1. In your `describe pod` output, the Image field shows `nginx:1.25` but Image ID shows `docker.io/library/nginx@sha256:...`. Explain this discrepancy.**

`nginx:1.25` is what I wrote in the YAML — an image reference WITHOUT
a registry hostname. When no registry is specified, containerd
applies Docker's DEFAULT expansion rule: it assumes `docker.io` as
the registry and `library/` as the namespace for "official"
vendor-maintained images. So `nginx:1.25` is automatically expanded
to `docker.io/library/nginx:1.25`. The `Image ID` field shows this
FULLY-RESOLVED reference, PLUS a sha256 digest — a cryptographic
fingerprint of the EXACT image content that was downloaded,
guaranteeing the image hasn't been tampered with and is reproducible.
If I'd written `myrepo/myapp:v1`, NO expansion would happen — it would
be used exactly as written.

---

**Q2. Why does the Pod show only ONE IP address (`192.168.1.91`) even though there could theoretically be multiple containers in a pod?**

All containers in a pod SHARE a single network namespace, which is
created and "held open" by a hidden infrastructure container called
the pause container — created FIRST, before any of my declared
containers. The Pod's IP is actually the pause container's IP. Every
container I declare (just `nginx` here) JOINS this same network
namespace and therefore shares this SAME IP. If I had a sidecar
container too, it would ALSO show `192.168.1.91` — they'd communicate
with each other via `localhost` on different ports.

---

**Q3. Walk me through the exact sequence shown in the Events section, and explain what component is responsible for each step.**

1. `Scheduled` — the Scheduler evaluated all nodes' available
   resources against my pod's `requests` (100m CPU, 64Mi memory) and
   assigned the pod to `node01`. This is a ONE-TIME decision the
   scheduler makes; it never touches the pod again afterward.
2. `Pulling` — kubelet on node01 detected this pod is now assigned to
   its node, and instructed containerd (the container runtime) to
   download the `nginx:1.25` image.
3. `Pulled` — containerd successfully downloaded all image layers
   (71MB total) from Docker Hub in about 6 seconds.
4. `Created` — containerd created the container filesystem from the
   downloaded layers and prepared the container (not yet running).
5. `Started` — containerd started the nginx process inside the
   container, joining it to the pause container's network namespace
   (IP 192.168.1.91).

---

**Q4. The pod shows `QoS Class: Burstable`. Looking at the Limits and Requests sections, explain WHY this pod got this QoS class and not Guaranteed or BestEffort.**

```
Limits:    cpu: 200m,  memory: 128Mi
Requests:  cpu: 100m,  memory: 64Mi
```

For `Guaranteed`, requests MUST EQUAL limits for BOTH cpu AND memory.
Here, requests (100m/64Mi) are LESS than limits (200m/128Mi) — they
don't match. For `BestEffort`, NEITHER requests NOR limits should be
set at all — but BOTH are explicitly set here. Since requests and
limits are BOTH SET but NOT EQUAL, this pod falls into the middle
category: `Burstable`. Practically, this means the pod is GUARANTEED
at least 100m CPU/64Mi memory (for scheduling), is ALLOWED to burst up
to 200m CPU/128Mi memory (enforced at runtime), and would be evicted
SECOND (after BestEffort, before Guaranteed) if the node runs low on
resources.

---

**Q5. Explain the `Tolerations` section shown in the output. Where did these come from — did you write them in your YAML?**

```
Tolerations:  node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
              node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
```

I did NOT write these — they are AUTOMATICALLY added to EVERY pod by
default (via the `DefaultTolerationSeconds` admission controller).
They mean: "if the node this pod is running on becomes NotReady or
Unreachable (e.g., kubelet stops responding), TOLERATE that condition
for 300 seconds (5 minutes) before this pod is evicted and
rescheduled elsewhere." This is WHY, when a node goes down, pods on
it aren't IMMEDIATELY rescheduled — there's a 5-minute grace period
built in by default, to avoid mass rescheduling storms from brief
network blips.

---

**Q6. The Conditions section shows 5 entries, all `True`. Explain each one and the ORDER in which they become True.**

1. `PodScheduled: True` — first to become true. The Scheduler assigned
   this pod to node01.
2. `PodReadyToStartContainers: True` — the pod's "sandbox" (pause
   container + network namespace + IP) was successfully created by
   containerd on node01.
3. `Initialized: True` — all init containers (if any were defined —
   none here) completed successfully. With zero init containers, this
   becomes true immediately/trivially.
4. `ContainersReady: True` — all regular containers (just `nginx`
   here) are Running AND passing their readiness probes. Since I
   defined NO readiness probe, Kubernetes defaults to "Running =
   Ready" for this container.
5. `Ready: True` — the FINAL, overall summary condition — true only
   once BOTH `Initialized` and `ContainersReady` are true. This is the
   condition Services check before sending traffic to this pod.

If, say, `ContainersReady` were stuck `False` while everything before
it is `True`, that tells me the container is RUNNING but its
READINESS PROBE is failing — I'd then run `kubectl describe pod` and
check the Events for "Readiness probe failed" messages.

---

**Q7. What is the `kube-api-access-s4n5c` volume, and why is it present even though I never declared any volumes in my YAML?**

This is a `Projected` volume type, automatically mounted into EVERY
pod by Kubernetes via its default ServiceAccount mechanism. It
bundles together: (1) a short-lived JWT token (`TokenExpirationSeconds:
3607` ≈ 1 hour, auto-rotated by kubelet) identifying this pod's
ServiceAccount (`default` in this case) to the API server, (2) the
cluster's CA certificate (`kube-root-ca.crt`) so the app could verify
the API server's TLS certificate if it ever called the API, and (3)
namespace information via the Downward API. This exists so that ANY
pod COULD call the Kubernetes API (e.g., to read its own metadata, or
for tools like service mesh sidecars) — even if, like in this nginx
pod, the application never actually uses it.

---

**Q8. The Image was pulled "in 5.903s including waiting" with size 71005258 bytes. If I create ANOTHER pod with the exact same image right now on the SAME node, would it take another ~6 seconds to pull?**

No — it would be near-instant, because the image layers are now
CACHED locally on `node01`'s disk by containerd. The
`imagePullPolicy` for a SPECIFIC tag like `nginx:1.25` (not `:latest`)
defaults to `IfNotPresent` — containerd checks if
`docker.io/library/nginx:1.25` (matching that exact sha256) already
exists locally; if YES, it uses the CACHED copy with ZERO network
calls. If I created the pod on a DIFFERENT node (e.g., `controlplane`)
that has NEVER pulled this image before, THAT node would still take
~6 seconds, since the cache is PER-NODE, not cluster-wide.

---

**Q9. There is no `Annotations` shown (it says `<none>`). What's the difference between labels and annotations, and why does THIS pod only have a label (`app=nginx`)?**

Labels are key-value pairs used for SELECTION/GROUPING — Services,
ReplicaSets, and NetworkPolicies use label SELECTORS to find matching
pods. Annotations are key-value pairs for ARBITRARY METADATA NOT used
for selection — things like build version, contact info, or
configuration hints for tools (e.g., Prometheus scrape annotations,
Ingress controller config). I only declared `labels: app: nginx` in
my YAML and no annotations — so `Annotations: <none>` simply reflects
that I didn't add any. If I wanted, say, a Service to find this pod,
it would use a selector matching `app: nginx` — that's the PURPOSE
of this label.

---

**Q10. SCENARIO: You change the YAML to use `image: nginx:latest` instead of `nginx:1.25`, delete this pod, and recreate it. A teammate says this is "risky" for production. Explain why, connecting it to what you see in THIS output's Image ID field.**

With `nginx:1.25` (a SPECIFIC version tag), the `Image ID` digest
(`sha256:a484819eb...`) is DETERMINISTIC and STABLE — every time
anyone pulls `nginx:1.25`, they get the EXACT SAME bits, verified by
that hash. With `nginx:latest`, the TAG `latest` is NOT a fixed
version — Docker Hub can repoint `latest` to a NEWER underlying image
at any time, meaning the `sha256` digest behind `latest` CHANGES over
time WITHOUT the YAML changing at all. ALSO, `imagePullPolicy`
defaults to `Always` for the `:latest` tag specifically — meaning
EVERY pod creation re-checks the registry, potentially pulling a
DIFFERENT image than yesterday, even with IDENTICAL YAML. This makes
deployments NON-REPRODUCIBLE — "it worked yesterday" might fail today
purely because `:latest` now points to a different image, with no
corresponding change in YOUR configuration. Production should always
pin SPECIFIC tags (or even better, the exact `sha256` digest) for
reproducibility.

---

**Q11. The output shows `Service Account: default`. What would change in this `describe pod` output if I had created a CUSTOM ServiceAccount called `nginx-sa` and referenced it in my YAML?**

The `Service Account: default` line would instead show
`Service Account: nginx-sa`. Correspondingly, the
`kube-api-access-xxxxx` projected volume's JWT token would identify
this pod as `system:serviceaccount:default:nginx-sa` instead of
`system:serviceaccount:default:default` — meaning if this pod's RBAC
permissions were checked (`kubectl auth can-i ... --as=system:serviceaccount:default:nginx-sa`),
it would be evaluated against whatever Roles/RoleBindings are bound
to `nginx-sa` SPECIFICALLY, rather than whatever (typically minimal/no)
permissions the `default` ServiceAccount has. Everything else in the
output (image, IP, conditions, etc.) would be unchanged —
ServiceAccount only affects API-access identity, not scheduling or
networking.

---

**Q12. TROUBLESHOOTING: Suppose this pod's Events section instead showed:**

```
Normal   Scheduled  79s   default-scheduler  Successfully assigned default/nginx-pod to node01
Warning  Failed     76s   kubelet            Failed to pull image "nginx:1.25": rpc error: code = Unknown desc = failed to resolve reference "docker.io/library/nginx:1.25": failed to do request
```

**and Conditions showed `PodScheduled: True` but `Initialized: False, ContainersReady: False, Ready: False`. What's your diagnosis and first three commands?**

`PodScheduled: True` confirms the Scheduler did its job — the pod IS
assigned to node01. Everything AFTER that — image pull — failed. The
error `failed to resolve reference ... failed to do request`
indicates `node01` could NOT reach Docker Hub over the network —
likely a DNS resolution failure or outbound internet connectivity
issue from that specific node (firewall, proxy misconfiguration, or
the node has no internet access in an air-gapped environment).

```bash
# Step 1 — confirm the pod is stuck, check current status
kubectl get pod nginx-pod -o wide

# Step 2 — SSH to node01, test DNS and connectivity directly
ssh node01
nslookup docker.io
curl -v https://registry-1.docker.io/v2/

# Step 3 — check if OTHER pods on node01 (with already-cached
# images) work fine, vs pods needing NEW pulls — narrows down
# whether it's "no internet at all" vs "DNS issue only"
nslookup docker.io
ping 8.8.8.8
```

If `node01` genuinely has no internet access (common in private/
banking environments), the fix would be configuring a local image
registry mirror/cache (e.g., a pull-through cache or internal Harbor
registry) so nodes pull from an INTERNAL address instead of the
public internet.

---

## Quick Reference — Image Reference Resolution Rules

| You Write | Expands To | Registry |
|---|---|---|
| `nginx:1.25` | `docker.io/library/nginx:1.25` | Docker Hub, official image |
| `muskanp/payment-app:v1.2` | `docker.io/muskanp/payment-app:v1.2` | Docker Hub, your account |
| `123456789.dkr.ecr.us-east-1.amazonaws.com/payment-app:v1.2` | (unchanged) | AWS ECR |
| `gcr.io/my-project/myapp:v1` | (unchanged) | Google Container Registry |
| `quay.io/coreos/etcd:v3.5` | (unchanged) | Quay.io |

**Rule:** No dot/registry-hostname before the first `/` → assumed
Docker Hub. No `/` at all → assumed `docker.io/library/<name>`
(official image namespace).

---

*Next: continue to Day 2 deep dives — Deployment, ReplicaSet, etc.*
