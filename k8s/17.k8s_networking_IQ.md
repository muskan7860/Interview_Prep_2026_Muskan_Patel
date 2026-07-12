# Kubernetes Networking — Interview Questions & Answers
## ClusterIP · NodePort · LoadBalancer · Ingress · Ingress Controller · CNI · CoreDNS
> Target: 4 Years DevOps Experience | Senior-Level Interviews
> Style: Easy to memorize + Professional to say out loud

---

## 💡 HOW TO USE THIS FILE

- Read the question first
- Try answering in your own words
- Then read the answer — written to be said out loud in an interview
- Real-world banking examples are included — use them to sound experienced

---

## SECTION A — FUNDAMENTALS

---

### Q1. What are the four fundamental networking rules Kubernetes guarantees?

**Answer:**

> "Kubernetes makes four networking promises that everything else builds on.
>
> First: every pod gets its own unique IP address — no two pods share an IP, even on the same node.
>
> Second: any pod can reach any other pod directly using that pod IP, without NAT — Network Address Translation. Pod A on Node 1 can send a packet directly to Pod B on Node 2 using Pod B's IP.
>
> Third: nodes can reach all pod IPs directly.
>
> Fourth — and this is the problem Services solve — pod IPs are ephemeral. When a pod restarts it gets a new IP. So hardcoding pod IPs in your application config always breaks eventually. Services fix this by providing a stable virtual IP in front of the changing pod IPs.
>
> Kubernetes defines these rules but does NOT implement the cross-node communication itself — that's the CNI plugin's job."

---

### Q2. What is a Kubernetes Service and why is it needed?

**Answer:**

> "A Service is a stable virtual IP — called a ClusterIP — that sits in front of a group of pods and provides a permanent address that never changes regardless of how many times the pods behind it are replaced.
>
> The problem it solves: pods are mortal. A pod restarts after a crash, a rolling update replaces pods, a node fails and pods reschedule — every time this happens the pod gets a completely new IP. If your payment service is hardcoded to call database pod IP `10.244.1.5` and that pod restarts and comes back as `10.244.2.8` — your payment service breaks.
>
> A Service gives you `10.96.45.200` — a ClusterIP that is permanent. The Endpoint Controller keeps the list of current pod IPs behind this ClusterIP updated automatically. kube-proxy on every node translates traffic to the ClusterIP into actual load-balanced traffic to the current pod IPs using iptables rules. The caller never knows or cares which pod IP it ends up on."

---

### Q3. Explain the three Service types — ClusterIP, NodePort, LoadBalancer — and when you use each.

**Answer:**

> "These three types build on each other — each one includes the previous.
>
> **ClusterIP** is the default. It creates a virtual IP that is only reachable from inside the cluster. No external access. I use this for all internal service-to-service communication — payment service calling transaction service, transaction service calling the database. In our banking project every microservice communicated via ClusterIP. Pods calling each other never need to go outside the cluster.
>
> **NodePort** opens a port between 30000 and 32767 on EVERY node in the cluster. Traffic arriving at any node's IP on that port gets forwarded to the service. It gives external access without a cloud load balancer. I use this in development environments or on-premises clusters where a cloud LB isn't available. Limitation: clients need to know node IPs, which can change, and you only get one service per port number.
>
> **LoadBalancer** takes NodePort one step further — it tells the cloud provider (AWS, GCP, Azure) to create a real external load balancer with a static public IP. That LB sits in front of the nodes. I use this when I need a simple external entry point with a stable public IP. The drawback: every Service of type LoadBalancer creates its own cloud LB — ten services means ten ALBs, which is expensive. For multiple services, Ingress is the better answer."

---

### Q4. What is the difference between `port`, `targetPort`, and `nodePort` in a Service?

**Answer:**

> "These three fields describe three different ports in the traffic path and people mix them up constantly.
>
> `port` is the port that the SERVICE itself listens on — what other pods use to reach this service. If port is 80, callers connect to `my-service:80`.
>
> `targetPort` is the port INSIDE the container that the traffic is forwarded to. If your application process listens on port 8080 inside the container, targetPort is 8080. Port and targetPort can be different — port 80 facing callers, targetPort 8080 inside the container. This is common because containers often run on non-standard ports internally but the service exposes a standard port.
>
> `nodePort` only exists on NodePort and LoadBalancer type services. It's the port that gets opened on every node's network interface. Valid range is 30000-32767. External clients hit `node-IP:nodePort` and it gets forwarded to the ClusterIP and then to the targetPort inside the container.
>
> Simple memory trick: port = what callers see. targetPort = what the container sees. nodePort = what external clients see on the node."

---

### Q5. What is the Endpoints object and who creates it?

**Answer:**

> "The Endpoints object is a Kubernetes resource that holds the list of actual pod IP addresses and ports that are currently backing a Service. It's the live mapping from a Service to its pods.
>
> The Endpoint Controller — which runs inside the Controller Manager — creates and maintains this object. It continuously watches all pods in the cluster. For each Service, it finds all pods matching the Service's label selector that are in Running state AND have their readiness probe passing. It writes those pods' IPs and ports into the Endpoints object.
>
> When a pod dies — its IP is removed from Endpoints. kube-proxy on every node watches Endpoints and updates iptables rules accordingly. So traffic stops going to the dead pod almost immediately — within a second or two.
>
> The key insight: a pod that is Running but not Ready — readiness probe failing — is NOT in the Endpoints list. Traffic never reaches it. This is why readiness probes are critical for zero-downtime deployments — new pods only get traffic after they're actually ready."

---

## SECTION B — INGRESS

---

### Q6. What is the difference between an Ingress resource and an Ingress Controller?

**Answer:**

> "This is one of the most commonly confused pairs in Kubernetes.
>
> An Ingress resource is just a YAML object — a configuration file that says 'traffic to this hostname and this path should go to this Service.' By itself it does absolutely nothing. You can `kubectl apply` an Ingress YAML and nothing will happen if there's no controller running.
>
> An Ingress Controller is the actual software — a running pod — that reads Ingress resources and implements the routing they describe. NGINX Ingress Controller reads your Ingress rules and configures an NGINX process to do the actual HTTP routing. AWS ALB Ingress Controller reads your rules and configures an actual Application Load Balancer in AWS.
>
> The relationship is exactly like how a Deployment object is just a configuration that the Deployment Controller acts on. Ingress resource is the config. Ingress Controller is the actor.
>
> Common mistake: people create an Ingress resource, nothing works, and they don't realize there's no Ingress Controller installed. Always confirm an Ingress Controller is running first."

---

### Q7. Why use Ingress instead of multiple LoadBalancer Services?

**Answer:**

> "Cost and complexity. If you have 15 microservices that need external access, 15 LoadBalancer Services means 15 cloud load balancers. On AWS that's 15 ALBs — each ALB costs money per hour plus data transfer charges. It also means 15 different external IP addresses that clients need to know about, 15 separate TLS certificate configurations, and 15 separate places to update when something changes.
>
> Ingress gives you ONE load balancer for all 15 services. Traffic comes in to one public IP, the Ingress Controller — NGINX or AWS ALB Controller — reads the routing rules and directs each request to the right backend Service based on the hostname or URL path. TLS is terminated once at the Ingress Controller.
>
> In our banking project we had 12 microservices. We went from 12 potential ALBs down to 1 by using the AWS ALB Ingress Controller with path-based routing. This saved significant monthly cost and made certificate management trivial — one cert, one renewal, one place to configure."

---

### Q8. What is path-based routing vs host-based routing in Ingress?

**Answer:**

> "Both are ways to route different traffic to different backends using one Ingress.
>
> Path-based routing uses the URL path to decide where to send traffic. Same hostname, different paths go to different services. For example: `bank.example.com/payment` goes to the payment service, `bank.example.com/accounts` goes to the account service, `bank.example.com/reports` goes to the reporting service. All the same domain, split by what comes after the slash.
>
> Host-based routing uses the subdomain or hostname to decide. Different hostnames go to different services. For example: `payment.bank.com` goes to payment service, `accounts.bank.com` goes to account service. Each microservice gets its own subdomain.
>
> You can combine both in one Ingress resource — different hosts each with different path rules. In practice, internal company tools often use host-based routing (each team gets a subdomain) while public APIs often use path-based routing (one domain with versioned paths like `/api/v1/`, `/api/v2/`)."

---

### Q9. What does `pathType: Prefix` vs `pathType: Exact` mean?

**Answer:**

> "pathType controls how strictly the path in the URL must match the path in the Ingress rule.
>
> `pathType: Prefix` means match any URL that STARTS WITH the defined path. If the rule says `/payment`, then `/payment`, `/payment/checkout`, `/payment/history/march` all match. It's a prefix — anything after it also matches.
>
> `pathType: Exact` means the URL must be EXACTLY the defined path. Only `/payment` matches. `/payment/checkout` does NOT match — it has extra characters after the defined path.
>
> In practice I almost always use Prefix for microservices — a payment service handles all paths under `/payment/`. Exact is used for specific single endpoints that you want to expose without exposing everything under that path."

---

### Q10. What are Ingress annotations? Give examples.

**Answer:**

> "Annotations in Ingress are key-value pairs in the metadata section that pass extra configuration to the Ingress Controller. They're controller-specific — NGINX annotations don't work on Traefik and vice versa.
>
> Common NGINX Ingress annotations I've used:
>
> `nginx.ingress.kubernetes.io/rewrite-target: /` — strips the prefix from the URL before forwarding. If your rule matches `/payment` and a request comes in as `/payment/checkout`, without rewrite the backend pod receives `/payment/checkout`. With rewrite-target `/`, the pod receives just `/checkout`. This is needed when the backend pod doesn't know it's being served under a prefix.
>
> `nginx.ingress.kubernetes.io/ssl-redirect: 'true'` — automatically redirects HTTP requests to HTTPS. Anyone hitting port 80 gets a 301 redirect to port 443.
>
> `nginx.ingress.kubernetes.io/proxy-body-size: '50m'` — increase max upload size. By default NGINX limits request body to 1MB. For file upload services you need to increase this.
>
> `nginx.ingress.kubernetes.io/rate-limit: '100'` — rate limit requests per second. Used for API protection.
>
> In our banking project the TLS redirect annotation was on every Ingress — no HTTP allowed anywhere in production."

---

## SECTION C — CNI PLUGINS

---

### Q11. What is a CNI plugin and why does Kubernetes need one?

**Answer:**

> "CNI stands for Container Network Interface. It's a specification — a standard interface — that defines how networking should be set up for containers. Kubernetes uses this standard to delegate all the actual networking implementation to a separate plugin.
>
> Kubernetes itself only says: 'every pod must have a unique IP, pods must be able to reach each other without NAT.' It does NOT say how to implement this. That's the CNI plugin's job.
>
> When kubelet creates a pod, it calls the CNI plugin and says 'set up networking for this new container.' The CNI plugin then assigns an IP from its allocated range, configures the virtual network interfaces inside the container, and sets up whatever routing is needed so other pods can reach this IP.
>
> Different environments need different approaches — on AWS you might use VPC-native networking, on bare metal you need something like BGP or VXLAN overlays, in high-security environments you need encryption between pods. The CNI standard lets you swap plugins without changing Kubernetes itself."

---

### Q12. Compare Calico, Flannel, and Cilium. When would you use each?

**Answer:**

> "These three cover different points on the spectrum from simple to powerful.
>
> **Flannel** is the simplest. It uses VXLAN — it wraps pod-to-pod packets inside another packet addressed to the destination node, sends it, and the destination node unwraps it. This overlay approach works everywhere but adds overhead because of the encapsulation and decapsulation on every packet. Flannel does NOT support NetworkPolicy — you can't restrict pod-to-pod traffic with Flannel alone. I'd use Flannel only for simple lab environments or where ease of setup matters more than performance.
>
> **Calico** is the most widely used in production. In its default mode it uses BGP — the same routing protocol internet routers use. Each node tells others 'I own these pod IP ranges' via BGP. No encapsulation — pure IP routing, very fast. Calico fully supports NetworkPolicy and adds its own extended GlobalNetworkPolicy for cross-namespace rules. In our banking project we used Calico because we needed strict network isolation between namespaces — payment pods could only reach the database pods, not anything else.
>
> **Cilium** is the modern choice. It uses eBPF — small programs that run inside the Linux kernel — instead of iptables. This means it can completely replace kube-proxy, is significantly faster at high scale, has much better observability (you can see exactly which pods are talking to which), and supports Layer 7 policies — you can say 'allow only HTTP GET requests from payment pods to the database, block everything else.' GKE and EKS are both moving toward Cilium as the default."

---

### Q13. What is a NetworkPolicy and what are its defaults?

**Answer:**

> "A NetworkPolicy is a Kubernetes object that acts as a firewall rule for pods — it controls which pods can send traffic to which other pods, and on which ports.
>
> The critical default behavior that catches people out: by default, with NO NetworkPolicy objects, ALL traffic is allowed between ALL pods in ALL namespaces. There is no default deny. Kubernetes starts wide open.
>
> NetworkPolicy works as an opt-in deny mechanism. Once you create a NetworkPolicy that selects certain pods, those pods have DENY ALL for the specified traffic type, and the policy adds explicit allows. Pods with NO NetworkPolicy selecting them remain wide open.
>
> There are two traffic directions: `Ingress` controls incoming traffic TO selected pods. `Egress` controls outgoing traffic FROM selected pods.
>
> In production banking, we applied a default-deny NetworkPolicy to every namespace — an empty ingress rule with `podSelector: {}` which selects all pods and denies all incoming traffic. Then we added explicit allow policies: allow payment pods to reach database pods on port 5432, allow monitoring pods to reach everything on port 9090, allow ingress controller to reach app pods on port 8080. Everything else blocked.
>
> Important: NetworkPolicy only works if your CNI plugin supports it. Flannel does not. Calico, Cilium, and Weave all do."

---

### Q14. What is VXLAN and when does a CNI use it?

**Answer:**

> "VXLAN stands for Virtual Extensible LAN. It's an overlay networking technique where you wrap one network packet inside another to carry it across a network that doesn't natively support the inner packet's routing.
>
> In Kubernetes terms: Pod A on Node 1 wants to send a packet to Pod B on Node 2. The pod IPs are on a virtual network — say `10.244.0.0/16` — but the physical nodes are on a different network — say `192.168.1.0/24`. The physical network doesn't know how to route `10.244.x.x` addresses.
>
> VXLAN solves this by taking the original pod-to-pod packet and wrapping it inside a UDP packet addressed to Node 2's physical IP. Node 2 receives the outer UDP packet, unwraps it, finds the inner pod packet, and delivers it to Pod B. This is called encapsulation and decapsulation.
>
> CNI plugins like Flannel use VXLAN by default. Calico can also use VXLAN in environments where BGP isn't available — like when nodes are in different VPCs or behind NAT. The downside is overhead — every packet has extra bytes added for the VXLAN header, and there's CPU cost for the wrap and unwrap operations."

---

## SECTION D — CoreDNS

---

### Q15. What is CoreDNS and what does it do in Kubernetes?

**Answer:**

> "CoreDNS is the DNS server that runs inside a Kubernetes cluster. It runs as a Deployment with typically 2 replicas in the `kube-system` namespace for high availability.
>
> Its job is name resolution inside the cluster. Instead of remembering that the payment service's ClusterIP is `10.96.45.200`, pods just call `payment-svc` or the full name `payment-svc.banking.svc.cluster.local` and CoreDNS resolves it to the ClusterIP.
>
> CoreDNS has two modes of resolution: for names ending in `cluster.local`, it looks up the Kubernetes Service or Pod registry and returns the appropriate ClusterIP or pod IP. For everything else — like `google.com` — it forwards the query to the upstream DNS server configured in the node's `/etc/resolv.conf` (usually the cloud provider's DNS or a corporate DNS server).
>
> Every pod gets a `/etc/resolv.conf` automatically injected by kubelet that points to the CoreDNS Service IP as the nameserver. This is how pods automatically know to use CoreDNS without any configuration from developers."

---

### Q16. What is the full DNS name format for a Kubernetes Service? Why do short names work?

**Answer:**

> "The full DNS name for a Service follows this pattern: `<service-name>.<namespace>.svc.<cluster-domain>`. The cluster domain is almost always `cluster.local` by default. So a service named `payment-svc` in namespace `banking` has the full name `payment-svc.banking.svc.cluster.local`.
>
> Short names work because of the DNS search path. Every pod's `/etc/resolv.conf` has a `search` line that lists suffixes to try when a name doesn't resolve as-is. It looks like: `search banking.svc.cluster.local svc.cluster.local cluster.local`.
>
> When a pod in the `banking` namespace calls just `payment-svc`, the DNS resolver tries appending each search suffix in order. First it tries `payment-svc.banking.svc.cluster.local` — this matches — resolution succeeds. Done.
>
> This is why a pod in the `banking` namespace can reach `payment-svc` with just the short name, but a pod in the `frontend` namespace cannot — it would first try `payment-svc.frontend.svc.cluster.local` which doesn't exist. The pod in `frontend` needs to use `payment-svc.banking` or the full name `payment-svc.banking.svc.cluster.local`."

---

### Q17. What is `ndots:5` in a pod's `/etc/resolv.conf` and what problem can it cause?

**Answer:**

> "`ndots:5` is a DNS resolver option that says: if a name has fewer than 5 dots in it, try the search domain suffixes first before treating it as an absolute name.
>
> This is normally helpful — `payment-svc` has zero dots, so the resolver appends search suffixes and finds `payment-svc.banking.svc.cluster.local` quickly.
>
> The problem comes with external domain names that have fewer than 5 dots — like `api.stripe.com` which has 2 dots. With ndots:5, the resolver first tries `api.stripe.com.banking.svc.cluster.local` (fails), then `api.stripe.com.svc.cluster.local` (fails), then `api.stripe.com.cluster.local` (fails), THEN finally tries `api.stripe.com` as absolute (succeeds). That's 3 extra DNS lookups that all fail before the correct one is tried.
>
> In high-traffic applications making frequent external API calls, this creates a noticeable DNS resolution overhead — each call makes 4 DNS queries instead of 1.
>
> The fix: use a trailing dot in external names to make them absolute — `api.stripe.com.` — the dot tells the resolver this is a fully qualified name, skip the search suffixes. Or configure `dnsConfig` in the pod spec to override `ndots` to a lower value like 2 for specific workloads."

---

### Q18. How would you troubleshoot a DNS resolution failure inside a pod?

**Answer:**

> "DNS failures show up as connection timeouts or 'name resolution failed' errors. My systematic approach:
>
> First, exec into the affected pod and check `/etc/resolv.conf` — confirm the nameserver is the CoreDNS Service IP and the search domains look correct. If this file is wrong, kubelet didn't inject it properly.
>
> Second, run `nslookup payment-svc` from inside the pod. If this fails but the ClusterIP works directly, it's definitely DNS. If both fail, it's a network connectivity problem, not DNS.
>
> Third, check if CoreDNS pods are healthy — `kubectl get pods -n kube-system | grep coredns`. If CoreDNS pods are CrashLoopBackOff or Pending, that's the root cause. Check `kubectl logs <coredns-pod> -n kube-system` for errors.
>
> Fourth, test if the CoreDNS Service is reachable from inside the pod — `nc -vz 10.96.0.10 53`. If this connection fails, the pod can't reach CoreDNS at all — a network policy might be blocking DNS traffic (port 53 UDP/TCP must be allowed).
>
> Fifth, check CoreDNS ConfigMap — `kubectl get configmap coredns -n kube-system -o yaml` — make sure the Corefile is correct and the `kubernetes` plugin is configured for `cluster.local`.
>
> In our banking cluster we had a NetworkPolicy misconfiguration that blocked egress from certain pods to the CoreDNS Service IP on port 53. DNS stopped working for those pods and apps got mysterious connection failures. Adding an explicit egress allow for port 53 fixed it."

---

## SECTION E — kube-proxy AND HOW SERVICES ACTUALLY WORK

---

### Q19. What is kube-proxy and how does it implement Service routing?

**Answer:**

> "kube-proxy is a DaemonSet — one pod on every node — whose job is to implement Service routing on that node. It watches the API Server for Service and Endpoints changes, and when something changes, it programs the node's networking so that traffic to ClusterIPs gets forwarded to the correct pod IPs.
>
> In the default iptables mode: kube-proxy creates iptables rules for each Service. When a pod sends a packet to a ClusterIP, the kernel's iptables intercepts it before it leaves the node, performs DNAT — Destination NAT — changing the destination from the ClusterIP to one of the real pod IPs selected from the Endpoints list (randomly, for basic load balancing), and forwards the packet. The response comes back and iptables reverses the NAT so the caller sees the response as coming from the ClusterIP.
>
> In IPVS mode — which you enable by setting `--proxy-mode=ipvs` — kube-proxy uses the Linux kernel's IP Virtual Server instead of iptables. IPVS uses hash tables for lookups so it's O(1) regardless of the number of Services. iptables is O(n) — it has to check rules linearly, which becomes slow at thousands of Services. For large clusters with many Services, IPVS is significantly faster."

---

### Q20. What happens at the network level when Pod A calls a Service and the traffic ends up at Pod B on a different node?

**Answer:**

> "Let me trace this end to end.
>
> Pod A on Node 1 wants to call `payment-svc`. It first does a DNS lookup — CoreDNS returns the ClusterIP, say `10.96.45.200`. Pod A sends a packet destined for `10.96.45.200:8080`.
>
> The packet hits iptables on Node 1. kube-proxy has written a DNAT rule: 'traffic to `10.96.45.200:8080` should go to one of these pod IPs — `10.244.1.5`, `10.244.2.7`, `10.244.3.3`.' iptables randomly picks `10.244.2.7` — which is Pod B on Node 2. iptables rewrites the destination IP from the ClusterIP to `10.244.2.7`.
>
> Now the packet is addressed to `10.244.2.7`. The CNI plugin's routing — whether BGP routes from Calico or VXLAN tunnels from Flannel — knows that `10.244.2.x` addresses live on Node 2. The packet is sent to Node 2's physical IP.
>
> Node 2 receives it. If using VXLAN, it's decapsulated. The packet arrives at Pod B on port 8080. Pod B processes the request and sends a response back to Pod A's IP.
>
> The response goes back through Node 2 → Node 1. On Node 1, iptables performs reverse NAT — changes the source IP from `10.244.2.7` back to the ClusterIP `10.96.45.200` — so Pod A sees the response as coming from the Service, not from Pod B directly. Pod A never knows Pod B's IP."

---

## SECTION F — TROUBLESHOOTING QUESTIONS

---

### Q21. A Service has pods Running but traffic is not reaching them. How do you debug?

**Answer:**

> "I follow a four-step layered approach, isolating each layer one at a time.
>
> Step one — check Endpoints. `kubectl get endpoints <service-name>`. If this shows `<none>`, the Service has no pods backing it. The cause is almost always a label selector mismatch — the Service selector says `app: payment` but the pods have label `app: payment-service`. Fix the labels or the selector.
>
> Step two — if Endpoints has pod IPs, check if those pods are actually Ready. `kubectl get pods --show-labels`. If pods show `0/1 Ready`, their readiness probe is failing. Endpoint Controller correctly excluded them. Fix the readiness probe issue.
>
> Step three — if Endpoints has IPs and pods are Ready, test connectivity from inside another pod. `kubectl exec -it <caller-pod> -- curl http://<service-name>:<port>`. If this times out, there might be a NetworkPolicy blocking the traffic. Check `kubectl get networkpolicies -n <namespace>`.
>
> Step four — if curl to the service name fails but curl to the pod IP directly works, it's a DNS or kube-proxy problem. Test `nslookup <service-name>` from inside the pod to check DNS. If DNS returns the wrong IP or fails, check CoreDNS.
>
> In our banking cluster, we had a case where endpoints were populated, pods were Ready, but traffic still failed. The culprit was a NetworkPolicy in the database namespace that had an ingress allow for `app: payment-service` but our pods were labeled `app: payment` — one character difference, no match, all traffic dropped silently."

---

### Q22. Pods in namespace A cannot reach a Service in namespace B. What do you check?

**Answer:**

> "Cross-namespace communication requires using the full Service DNS name — `service-name.namespace-b.svc.cluster.local` or at minimum `service-name.namespace-b`. Using just the short name `service-name` from a pod in namespace A will search in namespace A's DNS scope first and won't find the service in namespace B.
>
> So first thing I check: is the caller using the correct full DNS name? `kubectl exec -it <pod-in-namespace-a> -- nslookup payment-svc.namespace-b`.
>
> Second: NetworkPolicy. If namespace B has a default-deny NetworkPolicy, it blocks ALL ingress including from namespace A. I check `kubectl get networkpolicies -n namespace-b`. If there's a deny-all policy, I need an explicit allow rule that permits ingress from pods in namespace A — either by matching a label on namespace A's pods or by matching the namespace itself using `namespaceSelector`.
>
> Third: kube-proxy rules. ClusterIPs are accessible from any namespace by default — this is rarely the issue. But if kube-proxy is having problems on the source node, the iptables rules might be stale. Check kube-proxy logs on the node: `kubectl logs -n kube-system -l k8s-app=kube-proxy`."

---

### Q23. How would you debug a pod that cannot connect to the internet (external DNS)?

**Answer:**

> "No internet connectivity from a pod can have several causes. I work through them systematically.
>
> First, test if it's DNS or connectivity. From inside the pod: `nslookup google.com`. If this fails, either CoreDNS isn't forwarding external queries correctly, or DNS traffic to CoreDNS is blocked. If nslookup succeeds but `curl https://google.com` fails, DNS works but TCP/HTTPS is blocked.
>
> Second, check the node's internet connectivity. If the node can't reach the internet, pods on it can't either. `kubectl get node -o wide` — SSH to the node and `curl https://google.com` from the node itself.
>
> Third, check Egress NetworkPolicy on the pod's namespace. If there's an egress NetworkPolicy with `policyTypes: [Egress]` and no explicit allow for external traffic — egress to the internet is blocked. The policy needs an allow rule for `0.0.0.0/0` on port 443 for HTTPS, or a rule for the specific IP ranges needed.
>
> Fourth, check cloud-level security groups. On AWS, the node's security group might not allow outbound traffic on port 443. Kubernetes doesn't control this — it's an AWS VPC level restriction.
>
> Fifth, check if a corporate HTTP proxy is required. In banking environments, direct internet access is often blocked and all external traffic must go through a proxy. The pod needs `HTTP_PROXY` and `HTTPS_PROXY` environment variables configured."

---

## SECTION G — QUICK FIRE QUESTIONS

| Question | Answer |
|----------|--------|
| Default Service type if you don't specify type? | ClusterIP |
| Is ClusterIP reachable from outside the cluster? | No — internal only |
| NodePort range? | 30000–32767 |
| How many cloud LBs does one LoadBalancer Service create? | One |
| What object holds the current pod IPs behind a Service? | Endpoints |
| Which controller updates the Endpoints object? | Endpoint Controller (inside Controller Manager) |
| What is kube-proxy's job? | Write iptables rules on every node to route Service traffic |
| What mode makes kube-proxy faster at scale? | IPVS mode |
| What is an Ingress resource? | YAML routing rules — hostname/path → Service |
| What is an Ingress Controller? | Software that reads Ingress rules and implements them |
| Does an Ingress resource do anything without a controller? | No — nothing at all |
| What is `ingressClassName`? | Tells which Ingress Controller handles this Ingress |
| What does `pathType: Prefix` mean? | Match any URL that starts with the defined path |
| What does `pathType: Exact` mean? | Match ONLY the exact URL, nothing after it |
| What is CNI? | Container Network Interface — standard for pod networking plugins |
| Does Flannel support NetworkPolicy? | No |
| Which CNI uses BGP routing? | Calico |
| Which CNI uses eBPF? | Cilium |
| What overlay protocol does Flannel use? | VXLAN |
| What is CoreDNS? | The DNS server inside Kubernetes — resolves service names to IPs |
| Full DNS name format for a Service? | `<svc>.<namespace>.svc.cluster.local` |
| Why can pods use short Service names? | DNS search path appends namespace and cluster suffixes automatically |
| What port does CoreDNS listen on? | Port 53 (standard DNS port) |
| What is `ndots:5`? | Try search suffixes first for names with fewer than 5 dots |
| What is a NetworkPolicy default? | All traffic allowed — NetworkPolicy is opt-in deny |
| Which CNI plugins support NetworkPolicy? | Calico, Cilium, Weave (not Flannel) |
| What annotation enables HTTP→HTTPS redirect in NGINX Ingress? | `nginx.ingress.kubernetes.io/ssl-redirect: "true"` |
| What is `failurePolicy` in a webhook? | What to do if the webhook server is unreachable |
| Where does kubelet write pod DNS config? | `/etc/resolv.conf` inside each pod |
| What does `kubectl get endpoints <svc>` showing `<none>` mean? | No pods match the Service selector OR no pods are Ready |

---

## SECTION H — THINGS THAT SOUND IMPRESSIVE IN AN INTERVIEW

Use these naturally — understand the idea, don't memorize word for word:

1. **"The reason Kubernetes uses a flat pod network — every pod can reach every other pod without NAT — is to make service discovery simple. If pods had to go through NAT, source IPs would be hidden and you couldn't write NetworkPolicies based on pod identity. The flat model is a conscious design choice that everything else depends on."**

2. **"In our banking project, the first thing we did in every new namespace was apply a default-deny NetworkPolicy. Wide-open pod networking in a banking environment is unacceptable — a compromised pod could reach any database in the cluster. We worked backwards from zero trust — deny everything, then add explicit allows for each communication path we actually needed."**

3. **"We moved from Calico to Cilium on our staging cluster and the observability improvement was remarkable. Cilium's Hubble component gives you a live network flow graph — you can see exactly which pods are talking to which, on which ports, right now. Debugging a connectivity issue that used to take an hour of packet capturing took five minutes with Hubble. We're planning the production migration."**

4. **"The ndots:5 issue is a real performance problem that people don't know about until they profile their application. We had a payment service making 50 external API calls per transaction. Each call was doing 4 DNS queries — 3 failed search-suffix attempts before the real lookup. Under load, this DNS overhead was adding 20-30ms per transaction. Switching external calls to use trailing-dot FQDNs dropped transaction latency by 15%."**

5. **"One thing I always explain to developers new to Kubernetes: the Ingress Controller is just an NGINX pod running inside your cluster. When you configure TLS in Ingress, you're not configuring a cloud load balancer — you're configuring NGINX inside the cluster. The cloud LB in front (if using LoadBalancer type for the controller) is just raw TCP passthrough. All the HTTP intelligence, SSL termination, and path routing happens inside your cluster in the NGINX pod."**

---

*File: K8s_Networking_Interview_Questions.md*
*Repository: Interview_Preparation_2026 → Kubernetes/*
*Companion file: K8s_Networking_Concept_and_Lab.md*
