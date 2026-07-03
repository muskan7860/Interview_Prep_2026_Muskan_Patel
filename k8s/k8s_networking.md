# Kubernetes Networking — Deep Dive Concept + Hands-On Lab
## ClusterIP · NodePort · LoadBalancer · Ingress · Ingress Controller · CNI Plugins · CoreDNS
> Written for: Someone with 4 years of DevOps experience preparing for senior-level interviews
> Style: First-standard student explanation → deep technical truth → hands-on lab with line-by-line explanation

---

## 🧠 SECTION 1 — WHY DOES KUBERNETES NEED NETWORKING? (Story First)

### The Problem — Forget Kubernetes for 3 Minutes

Imagine you open a restaurant chain with 10 branches across the city.

Each branch has:
- Its own kitchen (worker node)
- Its own chefs (pods/containers)
- Its own phone number (IP address)

Now the problems start:

**Problem 1:** A chef quits (pod dies). A new chef is hired (new pod). The new chef has a **different phone number** (new IP). Customers who had the old number can no longer reach the restaurant.

**Problem 2:** You have 3 chefs making pizza. A customer calls — which chef do they speak to? You need someone to **distribute calls** evenly.

**Problem 3:** You have 10 different menu categories — pizza, pasta, desserts, drinks. You don't want 10 separate phone numbers. You want **one number** that routes based on what you ordered.

**Problem 4:** The chefs in the kitchen need to talk to each other. Chef A needs to tell Chef B the pizza is ready. They need an **internal communication system**.

**Kubernetes Networking solves ALL four of these problems.**

```
Problem 1 → Service (stable virtual IP — never changes)
Problem 2 → Load balancing (Service distributes traffic)
Problem 3 → Ingress (one entry point, routes by path/hostname)
Problem 4 → CNI + DNS (pod-to-pod communication + name resolution)
```

---

## 🌐 SECTION 2 — KUBERNETES NETWORKING FUNDAMENTALS

### The Four Networking Rules Kubernetes Guarantees

Kubernetes makes four promises about networking that everything else is built on:

```
RULE 1: Every Pod gets its OWN unique IP address
  → No two pods share an IP (even on the same node)
  → Pod IPs are routable within the cluster

RULE 2: Pods can communicate with ANY other pod
  → Without NAT (Network Address Translation)
  → Pod A on Node 1 can directly reach Pod B on Node 2
  → Just use the pod's IP — no port mappings needed

RULE 3: Nodes can communicate with ALL pods
  → The node's processes can reach any pod's IP directly

RULE 4: Pod IPs are ephemeral (this is the problem Services solve)
  → When a pod restarts → new IP
  → Solution: Services provide a STABLE virtual IP in front of pods
```

### Three Layers of Networking

```
LAYER 1: Pod-to-Pod Networking (handled by CNI plugin)
  → How pods on different nodes find and talk to each other
  → Handled by: Calico, Flannel, Cilium, Weave

LAYER 2: Service Networking (handled by kube-proxy)
  → Stable virtual IPs (ClusterIP) in front of changing pod IPs
  → Load balancing across pod replicas
  → Handled by: kube-proxy using iptables/ipvs rules

LAYER 3: External-to-Cluster Networking (handled by Ingress/LB)
  → How traffic from outside the cluster reaches your apps
  → Handled by: NodePort, LoadBalancer, Ingress
```

---

## 🔵 SECTION 3 — ClusterIP (Internal Communication)

### What is ClusterIP?

ClusterIP is the **DEFAULT** service type. It creates a **stable virtual IP** that is only reachable from INSIDE the cluster. Think of it as an **internal office phone number** — only people inside the building can use it.

### The Problem It Solves

```
WITHOUT ClusterIP:
  payment-app → calls → 10.244.1.5 (database pod IP)
  Database pod restarts → new IP: 10.244.2.8
  payment-app still calls 10.244.1.5 → CONNECTION REFUSED ❌

WITH ClusterIP:
  payment-app → calls → 10.96.45.200 (ClusterIP — NEVER changes)
  ClusterIP → forwards to → 10.244.1.5 (current pod IP)
  Database pod restarts → new IP: 10.244.2.8
  ClusterIP automatically updates → now forwards to 10.244.2.8
  payment-app still calls 10.96.45.200 → WORKS ✅
```

### How ClusterIP Actually Works (Under the Hood)

```
STEP 1: You create a Service with selector: app=payment

STEP 2: Endpoint Controller watches pods
  → Finds all Running+Ready pods with label app=payment
  → Creates Endpoints object: [10.244.1.5, 10.244.1.6, 10.244.1.7]

STEP 3: kube-proxy on EVERY node reads the Endpoints
  → Writes iptables rules: "traffic to ClusterIP:port → forward to one of these pod IPs"
  → These rules exist on EVERY node

STEP 4: When payment-app calls ClusterIP
  → iptables on that node intercepts the packet
  → Randomly (or round-robin) selects one pod IP
  → Forwards the packet to that pod
  → Pod responds directly back to payment-app
```

### ClusterIP YAML

```yaml
apiVersion: v1
kind: Service
metadata:
  name: payment-svc
  namespace: banking
spec:
  type: ClusterIP          # default — can omit this line
  selector:
    app: payment           # find pods with this label
  ports:
  - name: http
    port: 8080             # port the SERVICE listens on (what callers use)
    targetPort: 8080       # port the CONTAINER listens on (where traffic goes)
    protocol: TCP
```

**Line by line:**
- `type: ClusterIP` → create an internal-only virtual IP. This is the default — if you don't specify type, you get ClusterIP.
- `selector: app: payment` → find pods with label `app: payment` and send traffic to them.
- `port: 8080` → callers use `payment-svc:8080` to reach this service. This is the SERVICE's port.
- `targetPort: 8080` → traffic is forwarded to port 8080 INSIDE the container. Could be different from `port`.
- Different port example: `port: 80` and `targetPort: 8080` → callers use port 80, but the container receives on 8080.

---

## 🟡 SECTION 4 — NodePort (External Access, Basic)

### What is NodePort?

NodePort **opens a port on EVERY node** in the cluster. Traffic that arrives at `<any-node-IP>:<nodePort>` gets forwarded to your pods. This makes your service reachable from OUTSIDE the cluster without a cloud load balancer.

### Simple Story

NodePort is like putting a **doorbell on every building** in your office complex. Visitors from outside can ring any doorbell (any node's IP) at the same port number, and the intercom routes them to the right department (your pods).

```
EXTERNAL CLIENT
      │
      ▼  hits any node IP + NodePort
NODE 1: 192.168.1.10:30080 ──┐
NODE 2: 192.168.1.11:30080 ──┤─→ Service ClusterIP → pod IPs (load balanced)
NODE 3: 192.168.1.12:30080 ──┘
```

Even if your pod is on Node 3 — a request hitting Node 1:30080 still reaches it. kube-proxy handles the forwarding across nodes.

### NodePort YAML

```yaml
apiVersion: v1
kind: Service
metadata:
  name: payment-nodeport
spec:
  type: NodePort
  selector:
    app: payment
  ports:
  - port: 8080             # ClusterIP port (internal use)
    targetPort: 8080       # container port
    nodePort: 30080        # port opened on EVERY node (30000-32767 range)
    protocol: TCP
```

**New line:**
- `nodePort: 30080` → the port opened on every node. Must be between 30000-32767 (Kubernetes reserved range). If you don't specify this, Kubernetes picks one randomly in that range.

### NodePort Limitations

```
LIMITATION 1: Only one service per nodePort number
  → If payment-svc uses 30080, nothing else can use 30080
  → You only have ports 30000-32767 = 2768 ports total

LIMITATION 2: Exposes your node IPs directly
  → Security risk — clients need to know your node IPs
  → Node IPs can change (especially in cloud environments)

LIMITATION 3: No SSL termination, no path routing
  → Just raw TCP forwarding — no HTTP intelligence

USE WHEN:
  → Testing/development environments
  → On-premises clusters without a cloud load balancer
  → When you have a few services that need external access
```

---

## 🟢 SECTION 5 — LoadBalancer (Cloud External Access)

### What is LoadBalancer?

LoadBalancer is like NodePort **but with a cloud load balancer in front**. When you create a Service of type LoadBalancer in a cloud environment (AWS, GCP, Azure), Kubernetes calls the cloud provider's API and creates a real external load balancer with a **public IP address**.

### Simple Story

LoadBalancer is like having a **real receptionist at the front desk** of your building. External visitors don't need to know the floor (node) — they go to the main entrance (public IP), the receptionist handles directing them.

```
INTERNET
    │
    ▼  public IP
AWS ALB/NLB  ← cloud creates this automatically
    │
    ▼  forwards to any node
NODE 1: 30080 ──┐
NODE 2: 30080 ──┤─→ ClusterIP → pods
NODE 3: 30080 ──┘
```

**LoadBalancer is NodePort + Cloud Load Balancer. It creates all three:** a ClusterIP, opens NodePorts on all nodes, AND creates the cloud LB in front.

### LoadBalancer YAML

```yaml
apiVersion: v1
kind: Service
metadata:
  name: payment-lb
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"  # use NLB not CLB
spec:
  type: LoadBalancer
  selector:
    app: payment
  ports:
  - port: 80
    targetPort: 8080
```

**New things:**
- `type: LoadBalancer` → tells Kubernetes to request a cloud load balancer.
- `annotations: service.beta.kubernetes.io/aws-load-balancer-type: "nlb"` → annotations are metadata hints. This specific annotation tells the AWS cloud controller to create a Network Load Balancer (NLB) instead of the old Classic Load Balancer (CLB). Annotations like these are cloud-specific configuration.

### LoadBalancer Limitations

```
LIMITATION 1: ONE load balancer per service
  → 10 services = 10 separate load balancers
  → On AWS: 10 ALBs = significant cost (ALBs are not cheap)

LIMITATION 2: No path-based routing
  → LB type LoadBalancer just forwards traffic — it doesn't understand
    /payment vs /transactions vs /reports

SOLUTION: Use Ingress (one LB + smart routing)
```

---

## 🔶 SECTION 6 — INGRESS (Smart HTTP Routing)

### What is Ingress?

Ingress is a **set of routing RULES** that allows one external load balancer to intelligently route HTTP/HTTPS traffic to different backend services based on hostname and URL path.

### The Mall Analogy (Extended)

```
WITHOUT Ingress (3 LoadBalancer Services):
  shop.com → LB1 → payment-service     ($$$)
  shop.com → LB2 → cart-service        ($$$)
  shop.com → LB3 → product-service     ($$$)
  3 load balancers, 3 external IPs, 3x cost

WITH Ingress (1 LB + routing rules):
  External IP → Ingress Controller (NGINX)
    shop.com/payment  → payment-service
    shop.com/cart     → cart-service
    shop.com/products → product-service
  1 load balancer, 1 external IP, routing inside cluster
```

### Critical Understanding: Ingress vs Ingress Controller

```
INGRESS RESOURCE:
  → Just a YAML file with routing rules
  → Like a config file or a menu
  → Has no actual functionality on its own
  → kubectl apply -f ingress.yaml → just stores rules in etcd

INGRESS CONTROLLER:
  → The SOFTWARE that reads Ingress rules and IMPLEMENTS them
  → It is an actual running application (pod) in the cluster
  → Without an Ingress Controller, Ingress rules do NOTHING
  → Examples: NGINX Ingress Controller, Traefik, AWS ALB Controller, HAProxy

RELATIONSHIP:
  Ingress Resource (rules) ──read by──→ Ingress Controller (enforces rules)
  Config file               ──read by──→ NGINX server
```

### Ingress YAML — Full Example

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: banking-ingress
  namespace: banking
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - bank.example.com
    secretName: bank-tls-cert
  rules:
  - host: bank.example.com
    http:
      paths:
      - path: /payment
        pathType: Prefix
        backend:
          service:
            name: payment-svc
            port:
              number: 8080
      - path: /transactions
        pathType: Prefix
        backend:
          service:
            name: transaction-svc
            port:
              number: 8080
      - path: /reports
        pathType: Prefix
        backend:
          service:
            name: reporting-svc
            port:
              number: 8080
```

**Every line explained:**

```yaml
apiVersion: networking.k8s.io/v1
```
→ Ingress belongs to the `networking.k8s.io` API group, version 1.

```yaml
kind: Ingress
```
→ We are creating an Ingress routing rule object.

```yaml
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
```
→ Annotations are key-value pairs that pass extra config to the Ingress Controller.
→ `rewrite-target: /` → when traffic hits `/payment/checkout`, NGINX strips `/payment` and sends just `/checkout` to the backend pod. Without this, the pod receives `/payment/checkout` and might return 404 because it only knows `/checkout`.
→ These annotations are NGINX-specific — different controllers use different annotation keys.

```yaml
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
```
→ Automatically redirect HTTP → HTTPS. Any request on port 80 gets redirected to port 443.

```yaml
  ingressClassName: nginx
```
→ Which Ingress Controller should handle this Ingress resource.
→ `nginx` = the NGINX Ingress Controller. If you have multiple controllers (NGINX + Traefik), this selects which one picks up these rules.

```yaml
  tls:
  - hosts:
    - bank.example.com
    secretName: bank-tls-cert
```
→ `tls` = enable HTTPS for these rules.
→ `hosts` = which hostnames get HTTPS.
→ `secretName: bank-tls-cert` = the name of a Kubernetes Secret that contains the TLS certificate and private key. The Ingress Controller reads this secret and configures HTTPS termination.

```yaml
  rules:
  - host: bank.example.com
```
→ `rules` = list of routing rules.
→ `host` = only apply these rules when the request's Host header matches `bank.example.com`. You can have rules for multiple hosts in one Ingress.

```yaml
      paths:
      - path: /payment
        pathType: Prefix
```
→ `path: /payment` = match requests where the URL starts with `/payment`.
→ `pathType: Prefix` = match if the path STARTS WITH `/payment`. So `/payment`, `/payment/checkout`, `/payment/history` all match.
→ `pathType: Exact` = only match exactly `/payment` — nothing after it.

```yaml
        backend:
          service:
            name: payment-svc
            port:
              number: 8080
```
→ `backend` = where to send matching traffic.
→ `service.name: payment-svc` = send to the Kubernetes Service named `payment-svc`.
→ `port.number: 8080` = on port 8080 of that service.

### Two Types of Ingress Routing

```
1. PATH-BASED ROUTING (same host, different paths):
   bank.example.com/payment    → payment-service
   bank.example.com/accounts   → account-service
   bank.example.com/reports    → reporting-service
   One domain, multiple services based on URL path

2. HOST-BASED ROUTING (different subdomains):
   payment.bank.com   → payment-service
   accounts.bank.com  → account-service
   reports.bank.com   → reporting-service
   Different subdomains, each to a different service
```

---

## 🔌 SECTION 7 — CNI PLUGINS (Pod-to-Pod Networking)

### What is CNI?

CNI stands for **Container Network Interface**. It is a **standard specification** that defines how container networking should work. Think of CNI as a **plug socket standard** — any device that follows the standard can plug in.

### The Problem CNI Solves

Remember Kubernetes Rule 2: "Any pod can reach any other pod directly, without NAT."

But Kubernetes itself does NOT implement this. It says: "I need this to work, but I don't care HOW you implement it. Here's the standard (CNI) — write a plugin that satisfies it."

CNI plugins are the actual implementations.

### How Pod Networking Works (Step by Step)

```
SAME NODE communication:
  Pod A (10.244.0.5) → Pod B (10.244.0.6)
  Both on Node 1
  → Traffic goes through a virtual ethernet bridge (cbr0 or cni0)
  → Like two computers connected to the same switch
  → Fast, direct, no routing needed

DIFFERENT NODE communication:
  Pod A (10.244.0.5) on Node 1
  Pod B (10.244.1.5) on Node 2
  
  How does Node 1 know that 10.244.1.x is on Node 2?
  → This is what the CNI plugin figures out and configures
  → Different plugins use different methods (see below)
```

### How Different CNI Plugins Route Cross-Node Traffic

```
FLANNEL (simplest):
  → Uses VXLAN overlay networking
  → Wraps the original packet in another packet (encapsulation)
  → Sends the outer packet to the destination node's IP
  → Destination node unwraps it (decapsulation) and delivers to pod
  → Like putting a letter inside another envelope with the node's address
  → Works everywhere but adds overhead (encapsulation cost)

CALICO (most popular in production):
  → Uses BGP (Border Gateway Protocol) — same protocol internet routers use
  → Each node advertises: "I own the pod IPs in range 10.244.X.0/24"
  → Other nodes learn these routes via BGP
  → No encapsulation — packets routed at L3 (pure IP routing)
  → Faster than Flannel, supports Network Policies natively
  → Also has VXLAN mode for environments where BGP isn't available

CILIUM (modern, eBPF-based):
  → Uses eBPF (extended Berkeley Packet Filter) — programs in the Linux kernel
  → Instead of iptables rules (kube-proxy), uses eBPF programs
  → Much faster, more efficient, better observability
  → Can replace kube-proxy entirely
  → Supports L7 (application-layer) Network Policies (e.g., allow only HTTP GET)
  → Used by GKE, EKS (Cilium is becoming the default)

WEAVE:
  → Creates a virtual mesh network between nodes
  → Uses encrypted tunnels between nodes
  → Good for encrypted pod-to-pod communication out of box
  → Slower than Calico due to encryption overhead
```

### Network Policy — CNI's Extra Feature

Not all CNI plugins support **NetworkPolicy**. Flannel does NOT. Calico, Cilium, and Weave DO.

A NetworkPolicy is like a **firewall rule for pods**:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-payment-to-db
  namespace: banking
spec:
  podSelector:
    matchLabels:
      app: database           # this policy applies TO these pods
  policyTypes:
  - Ingress                   # control incoming traffic
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: payment        # only allow FROM pods with this label
    ports:
    - protocol: TCP
      port: 5432              # only on this port (PostgreSQL)
```

**What this says:** The `database` pods can ONLY receive incoming TCP connections on port 5432 FROM pods labeled `app: payment`. All other incoming traffic is DENIED.

**Default behavior without NetworkPolicy:** All traffic is allowed between all pods. NetworkPolicy is an OPT-IN deny mechanism.

---

## 🌍 SECTION 8 — CoreDNS (Kubernetes DNS)

### What is DNS in Kubernetes?

DNS (Domain Name System) converts names to IP addresses. In Kubernetes, instead of remembering that the payment service is at `10.96.45.200` — you just call `payment-svc.banking.svc.cluster.local` and DNS resolves it to the ClusterIP automatically.

### Simple Story

DNS is like a **phone book inside the cluster**. Instead of memorizing phone numbers (IP addresses), you look up names. `payment-svc` → `10.96.45.200`.

### CoreDNS — The Implementation

CoreDNS is the **DNS server** that runs inside Kubernetes (since K8s 1.13). It runs as a Deployment in the `kube-system` namespace with 2 replicas for high availability.

```bash
kubectl get pods -n kube-system | grep coredns
# coredns-7db6d8ff4d-abcd1   1/1   Running   0   10d
# coredns-7db6d8ff4d-xyz99   1/1   Running   0   10d
```

### The Full DNS Name Format

Every Kubernetes Service gets a DNS name following this pattern:

```
<service-name>.<namespace>.svc.<cluster-domain>

Example:
  payment-svc.banking.svc.cluster.local

Breaking down:
  payment-svc    → Service name
  banking        → Namespace name
  svc            → Literal "svc" — indicates this is a Service
  cluster.local  → The cluster's domain (configurable, default is cluster.local)
```

### DNS Resolution — Short Names Work Too

When a pod in the `banking` namespace calls `payment-svc` (short name without namespace), DNS search path fills in the rest automatically:

```
Pod in namespace 'banking' calls: payment-svc
DNS search order:
  1. payment-svc.banking.svc.cluster.local  ← tries this first
  2. payment-svc.svc.cluster.local
  3. payment-svc.cluster.local
  4. payment-svc (bare name)

Step 1 succeeds → returns ClusterIP → connection made
```

This is why pods in the SAME namespace can use short names. Pods in DIFFERENT namespaces need the full name: `payment-svc.banking.svc.cluster.local` (or at minimum `payment-svc.banking`).

### Pod DNS Names

Pods also get DNS names but they are less commonly used:

```
Format: <pod-IP-dashes>.<namespace>.pod.<cluster-domain>
Example: 10-244-1-5.banking.pod.cluster.local
  → Pod with IP 10.244.1.5 in banking namespace
```

### CoreDNS Configuration — The Corefile

```bash
kubectl get configmap coredns -n kube-system -o yaml
```

Inside you'll find the **Corefile** — CoreDNS configuration:

```
.:53 {
    errors
    health {
       lameduck 5s
    }
    ready
    kubernetes cluster.local in-addr.arpa ip6.arpa {
       pods insecure
       fallthrough in-addr.arpa ip6.arpa
       ttl 30
    }
    prometheus :9153
    forward . /etc/resolv.conf {
       max_concurrent 1000
    }
    cache 30
    loop
    reload
    loadbalance
}
```

**What each section means:**
- `.:53` → listen on port 53 (standard DNS port) for ALL queries (`.` = root = everything)
- `errors` → log errors
- `health` → expose health check endpoint at `:8080/health`
- `ready` → expose readiness endpoint at `:8181/ready`
- `kubernetes cluster.local` → handle DNS queries for `cluster.local` domain — this is the Kubernetes plugin that knows about all Services and Pods
- `pods insecure` → allow DNS records for pods (insecure = don't verify pod IPs against actual pods)
- `prometheus :9153` → expose metrics for Prometheus scraping
- `forward . /etc/resolv.conf` → for non-cluster queries (e.g., `google.com`), forward to the DNS server in `/etc/resolv.conf` (usually the node's DNS or cloud DNS)
- `cache 30` → cache DNS responses for 30 seconds to reduce load
- `loop` → detect and break DNS forwarding loops

### How Pods Know About CoreDNS

Every pod gets a `/etc/resolv.conf` file injected by kubelet:

```
nameserver 10.96.0.10      ← CoreDNS Service ClusterIP
search banking.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

- `nameserver 10.96.0.10` → send all DNS queries to this IP (CoreDNS Service)
- `search` → the search domains to try for short names
- `ndots:5` → if a name has fewer than 5 dots, try appending search domains first before treating it as absolute

---

## 🔧 SECTION 9 — HOW kube-proxy IMPLEMENTS SERVICE NETWORKING

### What is kube-proxy?

kube-proxy runs as a **DaemonSet** on every node. Its job is to **implement Service routing** on that node by programming the Linux kernel's networking rules.

### iptables Mode (Default)

```
kube-proxy watches API Server for Service and Endpoint changes
When a Service is created:
  kube-proxy writes iptables rules on THIS node

Traffic flow when pod calls ClusterIP:
  Packet sent to ClusterIP (10.96.45.200:8080)
       │
       ▼ iptables rule intercepts
  DNAT (Destination NAT): change destination from ClusterIP → real pod IP
  (random selection from Endpoints list)
       │
       ▼
  Packet sent to 10.244.1.5:8080 (actual pod)
       │
       ▼
  Response comes back
  iptables reverse NAT: change source back to ClusterIP
       │
       ▼
  Caller sees response from ClusterIP (unaware of pod IP)
```

### IPVS Mode (Better Performance)

```
IPVS = IP Virtual Server (Linux kernel load balancer)
Used when: kube-proxy is configured with --proxy-mode=ipvs

Advantages over iptables:
  → Better performance at scale (10,000+ services)
  → iptables: O(n) rule traversal — checking every rule linearly
  → IPVS: O(1) hash table lookup — direct jump to the rule
  → More load balancing algorithms: round-robin, least-connection, etc.

When to use IPVS:
  → Large clusters (1000+ services)
  → High-traffic environments
  → When you need weighted load balancing
```

---

## 💻 SECTION 10 — HANDS-ON LAB

> Every command explained word by word. Every flag explained. Nothing skipped.

---

### LAB 1 — ClusterIP Service — Create and Test

```bash
# Step 1: Create a deployment
kubectl create deployment web-app --image=nginx --replicas=3
```
- `create deployment` → create a Deployment object
- `web-app` → name of the deployment
- `--image=nginx` → use nginx container image
- `--replicas=3` → run 3 copies

```bash
# Step 2: Create a ClusterIP service
kubectl expose deployment web-app --port=80 --target-port=80 --name=web-svc
```
- `expose deployment web-app` → create a Service that points to this Deployment's pods
- `--port=80` → the Service listens on port 80
- `--target-port=80` → forward to port 80 inside containers
- `--name=web-svc` → call this Service "web-svc"
- Default type = ClusterIP (no `--type` flag needed)

```bash
# Step 3: See the ClusterIP assigned
kubectl get service web-svc
```

Output:
```
NAME      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
web-svc   ClusterIP   10.96.123.45    <none>        80/TCP    10s
```
- `CLUSTER-IP: 10.96.123.45` → this is the stable virtual IP
- `EXTERNAL-IP: <none>` → ClusterIP is internal only, no external access

```bash
# Step 4: Test from inside the cluster (create a debug pod)
kubectl run debug-pod --image=busybox --rm -it -- sh
```
- `run debug-pod` → create a pod named debug-pod
- `--image=busybox` → use busybox (tiny Linux with networking tools)
- `--rm` → delete this pod automatically when we exit the shell
- `-it` → interactive terminal
- `-- sh` → run `sh` inside the container

```bash
# Inside the debug pod shell:
wget -O- http://web-svc    # using short name (same namespace)
wget -O- http://10.96.123.45   # using ClusterIP directly
```
- `wget` → make an HTTP request
- `-O-` → output the response to stdout (terminal) instead of a file
- `http://web-svc` → using the Service DNS name (CoreDNS resolves this)

```bash
# Exit debug pod
exit
```

```bash
# Step 5: See the Endpoints object (pod IPs behind the service)
kubectl get endpoints web-svc
```

Output:
```
NAME      ENDPOINTS                                    AGE
web-svc   10.244.0.5:80,10.244.0.6:80,10.244.0.7:80  2m
```
These are the actual pod IPs. kube-proxy uses this list for iptables rules.

---

### LAB 2 — NodePort Service — External Access

```bash
# Create a NodePort service
kubectl expose deployment web-app --port=80 --target-port=80 \
  --name=web-nodeport --type=NodePort
```
- `--type=NodePort` → create a NodePort service

```bash
kubectl get service web-nodeport
```

Output:
```
NAME            TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
web-nodeport    NodePort   10.96.200.10   <none>        80:31234/TCP   10s
```
- `80:31234/TCP` → port 80 is the ClusterIP port, port 31234 is the NodePort (Kubernetes chose it randomly)

```bash
# Get your node IP
kubectl get nodes -o wide
```
- `-o wide` → show extra columns including the node's internal and external IP

```bash
# Test from your terminal (outside the cluster)
curl http://<NODE-IP>:31234
```
- Replace `<NODE-IP>` with the IP from the previous command
- You should see nginx's HTML page — traffic went: your machine → node IP:31234 → iptables → pod

---

### LAB 3 — Watch Service Endpoints Update Automatically

```bash
# Watch endpoints in one terminal
kubectl get endpoints web-svc -w
```
- `-w` → watch mode — print updates as they happen

```bash
# In another terminal, scale the deployment to 0
kubectl scale deployment web-app --replicas=0
```
- `scale deployment web-app` → change replica count
- `--replicas=0` → scale to zero (no pods)

Watch the first terminal — endpoints become empty:
```
NAME      ENDPOINTS                                    AGE
web-svc   10.244.0.5:80,10.244.0.6:80,10.244.0.7:80  5m
web-svc   10.244.0.5:80,10.244.0.6:80                 5m  ← pod deleting
web-svc   10.244.0.5:80                               5m
web-svc   <none>                                       5m  ← all gone
```

```bash
# Scale back up
kubectl scale deployment web-app --replicas=3
```

Watch endpoints come back as pods become Ready. This is the **Endpoint Controller** in action.

---

### LAB 4 — DNS Resolution with CoreDNS

```bash
# Create a debug pod
kubectl run dns-test --image=busybox --rm -it -- sh
```

Inside the pod:
```bash
# Check what DNS server this pod uses
cat /etc/resolv.conf
```
Output:
```
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```
- `nameserver 10.96.0.10` → CoreDNS Service IP

```bash
# Resolve the service by short name
nslookup web-svc
```
- `nslookup` → DNS lookup tool
Output:
```
Server:   10.96.0.10
Address:  10.96.0.10:53

Name:   web-svc.default.svc.cluster.local
Address: 10.96.123.45
```
- CoreDNS resolved `web-svc` → appended `default.svc.cluster.local` → found the ClusterIP

```bash
# Full name lookup
nslookup web-svc.default.svc.cluster.local
```

```bash
# Try resolving an external name (forwarded to upstream DNS)
nslookup google.com
```
This works too — CoreDNS forwards non-cluster queries to the node's DNS.

```bash
exit
```

---

### LAB 5 — Install NGINX Ingress Controller and Create Ingress Rules

```bash
# Install NGINX Ingress Controller (using kubectl)
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/cloud/deploy.yaml
```
- `apply -f <url>` → download YAML from that URL and apply it
- This creates: a Namespace, Deployment (the NGINX controller), Service (LoadBalancer type), RBAC objects, ValidatingWebhookConfiguration

```bash
# Wait for ingress controller to be ready
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
```
- `wait` → wait until a condition is met before proceeding
- `--namespace ingress-nginx` → in this namespace
- `--for=condition=ready` → wait until pods have Ready condition = true
- `--selector=app.kubernetes.io/component=controller` → for pods with this label
- `--timeout=120s` → give up after 120 seconds

```bash
# Create two different apps to route to
kubectl create deployment app-payment --image=nginx --replicas=2
kubectl create deployment app-account --image=httpd --replicas=2

# Create ClusterIP services for both
kubectl expose deployment app-payment --port=80 --name=payment-svc
kubectl expose deployment app-account --port=80 --name=account-svc
```

```bash
# Create the Ingress routing rules
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /payment
        pathType: Prefix
        backend:
          service:
            name: payment-svc
            port:
              number: 80
      - path: /account
        pathType: Prefix
        backend:
          service:
            name: account-svc
            port:
              number: 80
EOF
```

```bash
# Get the Ingress external IP
kubectl get ingress demo-ingress
```
Output:
```
NAME           CLASS   HOSTS   ADDRESS         PORTS   AGE
demo-ingress   nginx   *       192.168.1.100   80      30s
```

```bash
# Test path-based routing
curl http://192.168.1.100/payment    # → reaches nginx (app-payment)
curl http://192.168.1.100/account   # → reaches httpd (app-account)
```

---

### LAB 6 — Check CNI Plugin Running in Your Cluster

```bash
# See which CNI pods are running
kubectl get pods -n kube-system
```

Look for pods named:
- `calico-*` → Calico CNI
- `cilium-*` → Cilium CNI
- `flannel-*` / `kube-flannel-*` → Flannel CNI
- `weave-net-*` → Weave CNI

```bash
# Check what CNI config exists on the node
ls /etc/cni/net.d/
```
- `/etc/cni/net.d/` → directory where CNI configuration files live
- kubelet reads from this directory to know which CNI plugin to use

```bash
# See CNI config content
cat /etc/cni/net.d/10-calico.conflist   # (if Calico)
```

```bash
# Check pod IPs and their ranges to understand CNI subnet allocation
kubectl get pods -o wide
```
- `-o wide` → shows Pod IP column
- Notice all pod IPs follow a pattern like `10.244.X.Y` — the CNI allocates IPs from this range

---

### LAB 7 — Create and Test a NetworkPolicy

```bash
# Create two namespaces and apps
kubectl create namespace frontend
kubectl create namespace backend

kubectl create deployment backend-db --image=nginx -n backend
kubectl expose deployment backend-db --port=80 -n backend

# Try accessing backend from frontend BEFORE NetworkPolicy
kubectl run test-frontend --image=busybox -n frontend --rm -it -- \
  wget -O- http://backend-db.backend.svc.cluster.local
```
- `wget -O- http://backend-db.backend.svc.cluster.local` → full DNS name (service.namespace.svc.cluster.local)
- This SUCCEEDS — by default all traffic is allowed

```bash
# Now create a default-deny NetworkPolicy for backend namespace
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: backend
spec:
  podSelector: {}
  policyTypes:
  - Ingress
EOF
```
- `podSelector: {}` → empty selector = applies to ALL pods in this namespace
- `policyTypes: - Ingress` → control incoming traffic
- No `ingress:` rules → deny ALL incoming traffic

```bash
# Try again — should TIMEOUT now
kubectl run test-frontend --image=busybox -n frontend --rm -it -- \
  wget -O- --timeout=5 http://backend-db.backend.svc.cluster.local
```
- `--timeout=5` → give up after 5 seconds (otherwise wget waits forever)
- This now TIMES OUT — NetworkPolicy is blocking the traffic

```bash
# Clean up
kubectl delete namespace frontend backend
kubectl delete deployment web-app app-payment app-account
kubectl delete service web-svc web-nodeport payment-svc account-svc
kubectl delete ingress demo-ingress
```

---

## 📊 SECTION 11 — SERVICE TYPES COMPARISON TABLE

| Feature | ClusterIP | NodePort | LoadBalancer | Ingress |
|---------|-----------|----------|--------------|---------|
| External access | ❌ No | ✅ Yes (node IP) | ✅ Yes (LB IP) | ✅ Yes (LB IP) |
| Cloud LB created | ❌ No | ❌ No | ✅ Yes (one per svc) | ✅ Yes (one for all) |
| Path-based routing | ❌ No | ❌ No | ❌ No | ✅ Yes |
| Host-based routing | ❌ No | ❌ No | ❌ No | ✅ Yes |
| TLS termination | ❌ No | ❌ No | ❌ No | ✅ Yes |
| Port range restriction | No | 30000-32767 | No | No |
| Cost | Free | Free | $$$ per service | $$$ once |
| Use case | Internal comms | Dev/on-prem | Simple external | Production HTTP/HTTPS |

---

## 🔑 SECTION 12 — KEY TERMS TO REMEMBER

| Term | Simple Meaning |
|------|----------------|
| **ClusterIP** | Stable virtual IP — internal only, never changes |
| **NodePort** | Opens same port on every node for external access |
| **LoadBalancer** | ClusterIP + NodePort + cloud load balancer |
| **Ingress** | YAML routing rules (hostname + path → service) |
| **Ingress Controller** | The software that reads Ingress rules and implements them |
| **CNI** | Container Network Interface — standard for pod networking |
| **Calico** | CNI plugin using BGP routing, supports NetworkPolicy |
| **Cilium** | CNI plugin using eBPF, modern, replaces kube-proxy |
| **Flannel** | Simple CNI using VXLAN overlay, no NetworkPolicy support |
| **CoreDNS** | DNS server inside the cluster — resolves service names to IPs |
| **kube-proxy** | DaemonSet that writes iptables rules for Service routing |
| **Endpoints** | Object listing current pod IPs behind a Service |
| **NetworkPolicy** | Firewall rules for pods — controls which pods can talk to which |
| **VXLAN** | Tunnel protocol — wraps packets inside other packets |
| **BGP** | Border Gateway Protocol — routing protocol used by Calico |
| **eBPF** | Extended Berkeley Packet Filter — kernel programs used by Cilium |
| **IPVS** | IP Virtual Server — faster alternative to iptables for kube-proxy |
| **ingressClassName** | Tells which Ingress Controller should handle this Ingress resource |
| **pathType: Prefix** | Match URL if it STARTS WITH the given path |
| **pathType: Exact** | Match URL ONLY if it is exactly the given path |

---

*File: K8s_Networking_Concept_and_Lab.md*
*Repository: Interview_Preparation_2026 → Kubernetes/*
*Next: K8s_Networking_Interview_Questions.md*
