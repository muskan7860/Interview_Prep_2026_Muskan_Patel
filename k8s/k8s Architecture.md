Kubernetes Architecture — Interview Answer Guide

Role: DevOps Engineer — 3.2 to 4 Years Experience
Purpose: Copy this to GitHub as 01_Architecture.md


How to Use This File

Read Version 1 daily — until it flows naturally without thinking
Practice Version 2 out loud — record yourself, listen back
Study Version 3 for follow-up questions
Never memorise word for word — understand the story, tell it in your own words


Rule Before You Answer
When interviewer says "Explain Kubernetes Architecture" — they are checking three things:

Do you understand the big picture or just memorised component names?
Can you explain WHY each component exists, not just WHAT it is?
Have you actually worked with it or just read about it?

Golden rule: Always follow this structure:

Start with the purpose of Kubernetes
Introduce the two sides — Control Plane and Worker Nodes
Explain each component with what breaks when it dies
End with a real flow from your banking project


Version 1 — 60 Second Answer

Use this when interviewer says "give me a quick overview"


Kubernetes is a container orchestration platform — instead of manually managing servers, you declare what you want, like three replicas of a service, and Kubernetes makes it happen and keeps it that way automatically.
The architecture has two sides — the Control Plane is the brain, and the Worker Nodes are where your applications actually run.
The Control Plane has four components:

API Server — single entry point for all requests
etcd — distributed database storing all cluster state
Scheduler — decides which node a pod runs on
Controller Manager — runs background loops to keep actual state matching desired state


Each Worker Node has three components:

kubelet — agent that starts and manages containers
kube-proxy — handles service traffic routing
Container Runtime — containerd in modern clusters, actually runs the containers


Two supporting components keep everything working:

CoreDNS — internal service discovery and DNS resolution
Metrics Server — feeds CPU and memory data to the autoscaler


In my banking project at Atos, we ran this on AWS EKS — AWS manages the control plane completely, and we managed the worker node groups.


Version 2 — 3 Minute Deep Answer

Use this in standard interviews — this is your main answer

Opening — Set the Context

Kubernetes follows a declared desired state model — you never say "go start this container on server 5."
You say "I want 3 replicas of this payment service running" — Kubernetes figures out the rest, maintains it, and self-heals when something breaks.
That mindset is the foundation of the entire architecture — everything exists to serve that promise.


Part 1 — The Control Plane

The API Server is the single entry point for everything:

Every kubectl command goes through it
Every internal component communicates through it
Every CI/CD pipeline call goes through it
It handles authentication, RBAC authorization, admission control, then persists state to etcd
If API Server dies — you lose all control, but existing running pods are unaffected


etcd is the distributed key-value store — the single source of truth:

Every deployment, pod definition, secret, and config is stored here
In production we always run etcd as a 3 or 5 node cluster for high availability — odd numbers for Raft quorum
Regular etcd backups are non-negotiable in a banking environment — if etcd is lost with no backup, entire cluster configuration is permanently gone
If etcd dies — no changes can be made to the cluster, but existing pods keep running


The Scheduler watches for pods that have no node assigned yet:

Step 1 — Filtering — removes nodes that cannot run the pod (insufficient CPU, taint mismatch, affinity rules not satisfied)
Step 2 — Scoring — ranks remaining nodes by best fit
Step 3 — Assigns pod to winning node by writing nodeName to etcd
Important — Scheduler only makes the decision, it does not start the pod
If Scheduler dies — existing pods keep running, new pods stay in Pending forever


The Controller Manager runs the reconciliation loops:

Most critical is the ReplicaSet Controller — constantly watches "you asked for 3 replicas, I see 2 running, creating one more"
This loop is why Kubernetes self-heals — if a pod dies at 3am, no one needs to wake up
Other important controllers — Deployment, Node, Endpoint, Job, CronJob, HPA
If Controller Manager dies — self-healing stops, failed pods are not replaced




Part 2 — The Worker Nodes

The kubelet is the agent on every worker node:

Receives pod assignments from the API Server
Instructs the container runtime to pull images and start containers
Runs liveness and readiness probes
Reports pod status back to API Server continuously
Critical fact — kubelet is NOT a pod, it runs as a systemd service directly on the node
Troubleshoot with systemctl status kubelet and journalctl -u kubelet -f


kube-proxy handles service traffic routing on every node:

Writes iptables rules so traffic to a Service ClusterIP reaches the correct pod IPs
Without it — services stop routing traffic, applications cannot reach each other
Does NOT handle pod-to-pod networking — that is the CNI plugin's job


The Container Runtime — containerd in modern Kubernetes:

Actually creates and runs containers when kubelet instructs it
Communicates with kubelet via the CRI (Container Runtime Interface)
Docker is no longer the default — containerd is used directly




Part 3 — Supporting Components

CoreDNS runs as a Deployment in the kube-system namespace:

Every service gets a DNS entry — service-name.namespace.svc.cluster.local
Pods reach services by name — CoreDNS resolves the name to a ClusterIP
If CoreDNS dies — pods can still talk by IP but all service-name based communication breaks, which is effectively everything in a microservices architecture


Metrics Server collects CPU and memory from all nodes and pods:

HPA depends entirely on it to make scaling decisions
Powers kubectl top nodes and kubectl top pods
If Metrics Server dies — HPA goes blind and shows <unknown> in targets




Closing — Bring It to Your Project

In practice on AWS EKS at Atos:

AWS manages the entire control plane — API Server, etcd, Scheduler, Controller Manager
We never SSH into a control plane node — it is fully abstracted
We managed worker node groups, configured IRSA for pod-level IAM permissions, and used EKS managed add-ons for CoreDNS, kube-proxy, and VPC CNI
This matches how most enterprise Kubernetes is run today




Version 3 — When Interviewer Says "Walk Me Through a Deployment"

This is the most common follow-up — know this sequence cold

When I run kubectl apply -f deployment.yaml — here is the exact sequence:

kubectl converts the YAML to a JSON HTTP request and sends it to the API Server on port 6443
API Server receives the request and runs Authentication — validates my certificate or token
API Server runs RBAC Authorization — checks if I have permission to create a Deployment in this namespace
Request passes through Mutating Admission — webhooks can modify it, for example injecting resource limits automatically or adding an Istio sidecar container
Object Schema Validation — API Server validates all required fields are present and values are correct types
Validating Admission — final policy checks run, ResourceQuota validates the request will not exceed namespace limits
Object is written to etcd — desired state is now officially stored, this is the point of no return
Deployment Controller inside Controller Manager sees the new Deployment and creates a ReplicaSet object
ReplicaSet Controller sees it needs 3 pods and creates 3 Pod objects in etcd — with no node assigned yet
Scheduler sees 3 unscheduled pods, filters and scores nodes, writes node assignments to etcd
kubelet on each assigned node sees a pod assigned to it, calls containerd to pull the image, starts the containers
kubelet reports pod status back to API Server — pods show Running
Endpoint Controller updates the Endpoints object with the new pod IPs
kube-proxy updates iptables rules on all nodes so Service traffic correctly reaches the new pods


From kubectl apply to pods running — typically under 30 seconds in a healthy cluster


What NOT to Say — Common Mistakes
❌ Avoid This✅ Say This Instead"Master node""Control Plane" — master is deprecated terminology"Docker runs the containers""containerd is the container runtime in modern Kubernetes"Listing components without connecting themAlways explain what breaks when each component dies"I read that Kubernetes...""In our banking project at Atos, we..."Stopping after listing componentsAlways end with a real flow or production scenario"I think the scheduler starts the pod""Scheduler only assigns — kubelet starts the pod"

Follow-Up Questions — Quick Reference
Interviewer AsksKey Points in Your AnswerWhat happens if etcd goes down?No changes possible, existing pods keep running, need backup restoreHow is EKS different from self-managed K8s?AWS manages control plane, you manage node groups, IRSA for IAM, no etcd accessWhat is the reconciliation loop?Desired state vs actual state, controller closes the gap, runs foreverHow does a pod get scheduled?Filter nodes, score nodes, write nodeName, kubelet picks it upWhat are admission controllers?Mutating modifies the object, validating accepts or rejects, runs after AuthZ before etcdWhat is RBAC?Role defines what actions, RoleBinding defines who gets them, ClusterRole for cluster-wideDifference between taint and node affinity?Taint repels from the node side, affinity attracts from the pod sideWhat is the difference between kubelet and kube-proxy?kubelet runs containers, kube-proxy routes service trafficWhy does etcd use odd numbers?Raft consensus needs majority quorum — 3 nodes tolerates 1 failure, 5 nodes tolerates 2
