# Docker Networking — Interview Q&A

> **How to use this file:**
> Read each question out loud.
> Cover the answer.
> Try to answer in your OWN words naturally.
> Then check. Practice until it flows — not memorised word for word.
> Say the answer like you are explaining to a colleague.

---

## 📌 Table of Contents

- [Basic Level Questions](#basic-level-questions)
- [Intermediate Level Questions](#intermediate-level-questions)
- [Advanced / SRE Level Questions](#advanced--sre-level-questions)
- [Scenario-Based Troubleshooting Questions](#scenario-based-troubleshooting-questions)
- [Rapid Fire Round](#rapid-fire-round)
- [Phrases That Show Senior-Level Thinking](#phrases-that-show-senior-level-thinking)

---

## Basic Level Questions

---

### Q1. What is Docker networking? Why do we need it?

**How the interviewer asks it:**
> "Can you explain what Docker networking is and why it exists?"

**Your Answer:**

> "By default, containers are isolated from each other and from the outside world. Docker networking is the system that controls how containers communicate — with each other, with the host machine, and with the internet.
>
> We need it because a real application is never just one container. For example, a banking application might have a React frontend, a Node.js API, a PostgreSQL database, and a Redis cache — each in its own container. Docker networking lets the API container talk to the database, the frontend talk to the API, while keeping the database completely hidden from the internet.
>
> Docker networking lets you control exactly which containers can communicate, on which ports, and what is exposed to the outside world — giving you both connectivity and security."

---

### Q2. What is the default network in Docker? What are its limitations?

**How the interviewer asks it:**
> "If I run a container without specifying any network, what happens?"

**Your Answer:**

> "When you run a container without specifying a network, Docker automatically connects it to the default bridge network.
>
> The default bridge network creates a private subnet — by default `172.17.0.0/16` — and assigns each container an IP address in that range. Containers can reach each other by IP address, and they can reach the internet through the docker0 gateway using NAT.
>
> However, the default bridge has one important limitation: it does not support DNS resolution by container name. So if Container A wants to talk to Container B, it must use Container B's IP address — like `172.17.0.3` — not its name. This is a problem because IP addresses change every time a container restarts.
>
> The solution is to create a custom bridge network. On a custom bridge network, Docker runs an embedded DNS server that lets containers find each other by name. So instead of hard-coding `172.17.0.3:5432`, I can use `database:5432` and Docker DNS resolves the name to the current IP automatically. This is why I always create a custom network for multi-container applications."

---

### Q3. What is the difference between bridge, host, and none network modes?

**How the interviewer asks it:**
> "Explain the different Docker network types. When would you use each?"

**Your Answer:**

> "Docker has three main network modes for single-host use:
>
> **Bridge** is the default. The container gets its own isolated network with its own IP address. It can communicate with other containers on the same bridge network and reach the internet through NAT. This is what I use for most applications.
>
> **Host** removes all network isolation. The container shares the host machine's network stack directly. It uses the host's IP address and any port it opens is immediately available on the host. There is no NAT overhead, which means maximum performance. I would use this for monitoring tools that need to see all host network traffic, or when an application needs to bind to many ports dynamically.
>
> **None** gives the container no network at all. Only a loopback interface exists. The container cannot talk to anything and nothing can reach it. I would use this for security-sensitive batch processing jobs where the application should have zero network access — for example, a container that processes sensitive financial documents.
>
> In production, I always use custom bridge networks for application containers because it gives isolation plus automatic DNS by container name."

---

### Q4. What is port publishing in Docker? How does `-p` work?

**How the interviewer asks it:**
> "How do I make a container's service accessible from outside? What does `-p 8080:80` mean?"

**Your Answer:**

> "`-p 8080:80` tells Docker: any traffic arriving on port 8080 of my host machine should be forwarded to port 80 inside the container.
>
> The format is always `host_port:container_port`.
>
> Under the hood, Docker creates an iptables NAT rule that rewrites the destination of incoming packets. So when a browser connects to `your-server:8080`, the Linux kernel's iptables intercepts the packet, changes the destination to the container's internal IP at port 80, and forwards it. The container responds, iptables rewrites the source back, and the browser gets its response.
>
> Without port publishing, a container's ports are completely private — only accessible from other containers on the same network or from the host machine itself. Publishing is what opens a specific port to the outside world.
>
> Important security practice: I only publish the ports that absolutely need external access. For example, in a three-tier application, I only publish the frontend's port. The backend API and database ports stay private — reachable only by other containers on the same internal Docker network."

---

### Q5. What is the docker0 interface? What role does it play?

**How the interviewer asks it:**
> "I see a `docker0` interface when I run `ip addr show`. What is that?"

**Your Answer:**

> "`docker0` is a virtual network bridge that Docker creates on the host machine when you install Docker. It is the central connection point for all containers on the default bridge network.
>
> Think of it like a virtual network switch inside your machine. Each container gets a virtual ethernet cable — called a veth pair — with one end inside the container as `eth0` and the other end connected to docker0 on the host.
>
> docker0 acts as the gateway for containers. Its IP address is `172.17.0.1` and containers use it to route traffic to the internet. When a container wants to reach google.com, the packet goes: container → docker0 → iptables NAT → your machine's real network card → internet.
>
> You can verify this with `ip addr show docker0` — you will see it has the IP `172.17.0.1/16`, which is the gateway address for the default bridge subnet."

---

## Intermediate Level Questions

---

### Q6. How does container-to-container DNS work on a custom bridge network?

**How the interviewer asks it:**
> "How do containers find each other by name? Walk me through how DNS works inside Docker."

**Your Answer:**

> "On a custom bridge network, Docker runs an embedded DNS server. This DNS server is accessible at `127.0.0.11` from inside every container on that network — you can verify this in `/etc/resolv.conf` inside any container.
>
> When you start a container with a name — `docker run --name database --network myapp-network postgres` — Docker's DNS server automatically registers that name. So `database` maps to whatever IP the container receives.
>
> When another container on the same network wants to connect to the database, it makes a DNS query for `database`. The query goes to `127.0.0.11` — Docker's DNS server — which looks up the current IP and returns it. The container then connects to that IP.
>
> The key benefit is when a container restarts and gets a new IP address. The application code never changes — it always says `database:5432`. Docker DNS automatically has the updated IP, so the next lookup returns the new address.
>
> This is only available on custom bridge networks, not the default bridge network. On the default bridge, containers must use IP addresses directly, which is fragile. This is one of the main reasons I always create a custom network for any multi-container application."

---

### Q7. What is NAT in the context of Docker networking?

**How the interviewer asks it:**
> "You mentioned NAT when explaining port publishing. What is NAT and how does Docker use it?"

**Your Answer:**

> "NAT stands for Network Address Translation. It is a technique where a router or firewall changes the source or destination IP address of network packets as they pass through.
>
> Docker uses two types of NAT:
>
> **For outbound traffic** — containers use private IP addresses in the `172.17.0.0/16` range. These private IPs are not routable on the internet. When a container makes an outbound request — say, downloading a package from npm — Docker uses SNAT (Source NAT) to rewrite the packet's source address from `172.17.0.2` to your machine's real public IP. The npm server sees the request coming from your machine's IP, not the container's private IP. When the response comes back, Docker rewrites the destination back to `172.17.0.2` and delivers it to the container.
>
> **For inbound traffic** — this is what port publishing uses. When you do `-p 8080:80`, Docker sets up a DNAT (Destination NAT) rule. Incoming traffic to your machine's port 8080 has its destination rewritten to the container's IP at port 80.
>
> All of this happens automatically via iptables rules that Docker manages. You can see them with `sudo iptables -t nat -L DOCKER -n -v`."

---

### Q8. Can a container be connected to multiple networks? Why would you do this?

**How the interviewer asks it:**
> "Is it possible to connect a container to more than one Docker network at the same time? When is this useful?"

**Your Answer:**

> "Yes, a container can be connected to multiple networks simultaneously. You can either specify them at container creation or use `docker network connect` on a running container.
>
> This is useful for building security boundaries in multi-tier applications. Here is a real example:
>
> ```
> frontend-network:  frontend ←→ api
> backend-network:   api ←→ database ←→ cache
> ```
>
> The `api` container is connected to BOTH networks. The `frontend` container is only on `frontend-network` — it can reach the API but has absolutely no visibility to the database or cache. The `database` container is only on `backend-network` — the frontend cannot reach it at all.
>
> I used this pattern in our banking project. The React frontend needed to reach the Node.js API. The Node.js API needed to reach the PostgreSQL database. But we never wanted the frontend — which is exposed to the internet — to have any possible path to the database. Multiple networks gave us this guarantee at the network level, not just at the application level."

---

### Q9. What is the difference between macvlan and ipvlan networks?

**How the interviewer asks it:**
> "I heard about macvlan and ipvlan. Can you explain what they are and when to use them?"

**Your Answer:**

> "Both macvlan and ipvlan allow containers to connect directly to the physical network, appearing as real network devices rather than being behind a NAT layer. The difference is how they handle MAC addresses.
>
> With macvlan, each container gets its own unique MAC address. From the network's perspective, each container looks like a separate physical device plugged into the switch. Other machines on the network can reach containers directly using their own IP addresses without going through the host.
>
> With ipvlan, all containers share the host machine's MAC address. Each container still gets its own IP address, but they all appear to come from the same MAC. This makes ipvlan more compatible with cloud environments and corporate networks where network equipment may block multiple MAC addresses from one physical port — a common security measure.
>
> Use cases for both: legacy applications that need to be on the main network, applications that require specific IP addresses assigned by your network's DHCP server, or when you need containers to be directly reachable without port mapping.
>
> In practice, I rarely use these in modern cloud deployments. Most cloud setups use bridge networks with load balancers handling external traffic. macvlan and ipvlan are more common in on-premises environments or industrial applications."

---

## Advanced / SRE Level Questions

---

### Q10. You have a microservices application with 5 containers. How do you design the networking?

**How the interviewer asks it:**
> "Walk me through how you would design Docker networking for a production microservices application."

**Your Answer:**

> "I would think about this in terms of who needs to talk to whom, and who should NOT be able to talk to whom.
>
> For a typical application with frontend, API, database, cache, and a message queue:
>
> First, I would create separate custom bridge networks for different tiers:
> ```
> public-network:    reverse-proxy ←→ frontend
> app-network:       frontend ←→ api ←→ worker
> data-network:      api ←→ postgres ←→ redis ←→ rabbitmq
>                    worker ←→ postgres ←→ redis ←→ rabbitmq
> ```
>
> Only the reverse-proxy would have published ports (`-p 443:443`). Everything else communicates internally by container name through Docker DNS.
>
> The frontend would be on `public-network` only — it cannot reach the database.
> The API would be on both `app-network` and `data-network` — it can reach both the frontend and the data tier.
> The database would be on `data-network` only — completely unreachable from the public network.
>
> This defence-in-depth approach means even if the frontend container is completely compromised, the attacker has no network path to the database. The isolation is enforced at the kernel networking level, not just in application code.
>
> In production on Kubernetes, this translates to NetworkPolicy objects that define the same rules. Understanding Docker networking well makes Kubernetes networking much easier to understand."

---

### Q11. A container cannot reach the internet. How do you troubleshoot?

**How the interviewer asks it:**
> "You get a report: containers on this machine cannot download packages or reach external APIs. What do you check?"

**Your Answer:**

> "I would approach this systematically:
>
> **Step 1 — Confirm the problem:**
> ```bash
> docker run --rm alpine ping -c 3 8.8.8.8
> # Can the container reach a known IP?
>
> docker run --rm alpine ping -c 3 google.com
> # Is it an IP issue or a DNS issue?
> ```
>
> **Step 2 — Check if it is DNS-only:**
> If `ping 8.8.8.8` works but `ping google.com` fails, the problem is DNS.
> Check the container's DNS configuration:
> ```bash
> docker run --rm alpine cat /etc/resolv.conf
> ```
> If the DNS server is unreachable or wrong, DNS will fail.
>
> **Step 3 — Check the host network:**
> ```bash
> ping 8.8.8.8  # from the host, not a container
> ip route show  # check routing table
> ```
> If the host itself cannot reach the internet, containers cannot either.
>
> **Step 4 — Check Docker's NAT rules:**
> ```bash
> sudo iptables -t nat -L POSTROUTING -n -v
> # Look for MASQUERADE rule for 172.17.0.0/16
> # This rule is what allows containers to reach the internet
> ```
> If this rule is missing, containers cannot send outbound traffic.
>
> **Step 5 — Check IP forwarding:**
> ```bash
> cat /proc/sys/net/ipv4/ip_forward
> # Must be 1 for Docker networking to work
> # If it is 0, enable it:
> sudo sysctl -w net.ipv4.ip_forward=1
> ```
> IP forwarding must be enabled for packets to route between interfaces.
>
> In my experience, the most common cause is a firewall update that cleared iptables rules, or a security team policy that disabled IP forwarding. Both are solvable but you need to know what to look for."

---

### Q12. How does Docker networking work differently on Linux vs Docker Desktop (Mac/Windows)?

**How the interviewer asks it:**
> "My colleague on Mac says `localhost` in Docker behaves differently than on my Ubuntu machine. Why?"

**Your Answer:**

> "This is a fundamental architectural difference.
>
> On Linux, Docker runs natively. The `dockerd` daemon runs directly on Linux. Containers use the Linux kernel's network namespaces and the docker0 bridge connects directly to your machine's networking stack. `localhost` inside a container refers to the container itself. `host.docker.internal` or the docker0 gateway IP `172.17.0.1` refers to the host.
>
> On Mac and Windows, there is no Linux kernel natively. Docker Desktop runs a lightweight Linux virtual machine in the background using HyperKit on Mac or WSL2 on Windows. All containers run INSIDE this virtual machine, not directly on your Mac or Windows host.
>
> This creates differences:
> - `--network host` does not work on Mac/Windows because the 'host' would be the Linux VM, not your Mac/Windows machine
> - Performance is slightly lower due to the VM layer
> - File system mounts are slower because they cross VM boundaries
> - `localhost` behaviour is different — `host.docker.internal` is a special DNS name on Docker Desktop that resolves to your Mac/Windows host
>
> For professional DevOps work, I always develop and test on Linux when possible because it matches the production environment exactly. Docker Desktop is convenient for Mac/Windows developers but you should be aware of these differences when they cause issues."

---

## Scenario-Based Troubleshooting Questions

---

### Q13. "My frontend container gets Connection Refused when calling the backend API. What do you check?"

**Your Answer:**

> "Connection Refused means the connection was actively rejected — the target IP and port was reached but nothing was listening there. It is different from Network Unreachable which means the target could not be reached at all.
>
> I would check these in order:
>
> **Step 1 — Are both containers on the same network?**
> ```bash
> docker inspect frontend | jq '.[0].NetworkSettings.Networks | keys'
> docker inspect backend  | jq '.[0].NetworkSettings.Networks | keys'
> # They must share at least one network name
> ```
>
> **Step 2 — Is the backend actually running and listening?**
> ```bash
> docker ps | grep backend
> # Check STATUS is 'Up', not Exited
>
> docker logs backend | tail -20
> # Check for startup errors or crash messages
>
> docker exec backend netstat -tlnp
> # Or: docker exec backend ss -tlnp
> # See which ports the backend is actually listening on inside the container
> ```
>
> **Step 3 — Can frontend reach backend by name?**
> ```bash
> docker exec frontend ping -c 3 backend
> # If ping fails, it is a network/DNS issue, not a port issue
> ```
>
> **Step 4 — Test the actual connection**
> ```bash
> docker exec frontend wget -qO- http://backend:5000/health
> # Or:
> docker exec frontend curl http://backend:5000/health
> ```
>
> **Most common causes I have seen:**
> - Frontend and backend on different networks (most common)
> - Backend crashed on startup (check logs)
> - Frontend using wrong port number in config
> - Backend bound to `127.0.0.1` inside container instead of `0.0.0.0` — only accepts loopback connections, not connections from other containers"

---

### Q14. "Two containers can ping each other but HTTP requests fail. Why?"

**Your Answer:**

> "Ping uses ICMP protocol. HTTP uses TCP. They are different protocols, so this situation means network layer is working but something at the application or port level is wrong.
>
> **Step 1 — Check if the target port is actually open:**
> ```bash
> docker exec container1 nc -zv container2 8080
> # nc = netcat: a tool to test port connectivity
> # -z = just check if port is open, do not send data
> # -v = verbose output
> ```
> If this shows 'Connection refused' — the port is not listening.
> If it shows 'Connected' — the port is open and the problem is in the HTTP layer.
>
> **Step 2 — Check what port the service is listening on:**
> ```bash
> docker exec container2 ss -tlnp
> # ss = socket statistics
> # -t = TCP sockets only
> # -l = listening sockets only
> # -n = numeric port numbers
> # -p = show which process is using each socket
> ```
>
> **Step 3 — Check if the service bound to the right address:**
> If the output shows `127.0.0.1:8080` instead of `0.0.0.0:8080` or `:::8080`, the service only accepts connections from within the same container. Fix: change the app to bind to `0.0.0.0` instead of `127.0.0.1`.
>
> **Step 4 — Check application logs:**
> ```bash
> docker logs container2 | tail -30
> # Look for error messages, startup failures, or crash loops
> ```"

---

### Q15. "I need to access a service running on the host machine from inside a container. How?"

**Your Answer:**

> "This is a common need — for example, a container needs to talk to a database running directly on the host, not in another container.
>
> **On Linux**, the most reliable way is to use the docker0 gateway IP:
> ```bash
> # Find docker0 IP (usually 172.17.0.1)
> ip addr show docker0 | grep inet
>
> # Inside the container, use this IP to reach host services
> docker exec mycontainer curl http://172.17.0.1:5432
> ```
>
> Another option on Linux is to add `--add-host` to give a friendly name:
> ```bash
> docker run --add-host=host.docker.internal:host-gateway myimage
> # Now inside container: curl http://host.docker.internal:5432
> ```
>
> **On Docker Desktop (Mac/Windows)**, Docker provides a special DNS name:
> ```
> host.docker.internal
> ```
> This always resolves to the host machine from inside any container.
>
> **Security consideration:** I am careful about this pattern. If a container can reach host services, I make sure those services only accept connections from trusted sources. In production, it is better architecture to run everything in containers and use Docker networks rather than having containers reach back to the host."

---

## Rapid Fire Round

> Short answers. Practice until instant.

| Question | Answer |
|----------|--------|
| What is the default Docker network driver? | bridge |
| What IP range does the default bridge use? | 172.17.0.0/16 |
| What is docker0? | Virtual network bridge on the host that containers connect to |
| What port does Docker DNS listen on inside containers? | 127.0.0.11 |
| Does DNS by name work on default bridge? | No — only on custom bridge networks |
| What flag publishes a port? | `-p host_port:container_port` |
| What happens to a container with `--network none`? | No network access at all |
| What is a veth pair? | Virtual ethernet cable connecting container to docker0 |
| How do you create a custom network? | `docker network create mynetwork` |
| How do you connect a running container to a network? | `docker network connect mynetwork containername` |
| What is VXLAN used for? | Tunneling container traffic across multiple hosts in overlay networks |
| What is the difference between macvlan and ipvlan? | macvlan: each container gets own MAC. ipvlan: containers share host MAC but get own IP |
| What iptables table does Docker use for port publishing? | nat table (DNAT rules in DOCKER chain) |
| What must be enabled in the kernel for Docker networking? | IP forwarding (`net.ipv4.ip_forward = 1`) |

---

## Phrases That Show Senior-Level Thinking

Use these naturally in your answers. They signal real production experience:

- *"I always create a custom bridge network for multi-container applications because the default bridge does not support DNS by container name..."*

- *"In my banking project, we used multiple networks to enforce network-level isolation between tiers. The frontend had no network path to the database..."*

- *"Port publishing is essentially a DNAT rule in iptables. You can verify this with `sudo iptables -t nat -L DOCKER -n -v`..."*

- *"When troubleshooting container networking, I check three things in order: is the network right, is the container running, is the service bound to the right address..."*

- *"Container DNS relies on Docker's embedded resolver at `127.0.0.11`. If name resolution fails, checking `/etc/resolv.conf` inside the container is my first step..."*

- *"On Linux, host networking works exactly as you expect. On Docker Desktop for Mac, `--network host` connects to the VM, not your Mac — this catches a lot of people..."*

- *"In production Kubernetes, this translates to NetworkPolicy objects. Understanding Docker networking first makes Kubernetes networking much easier to reason about..."*

---

*File: `docker_networking_interview.md` | Topic 3 of 8 | Docker Interview Preparation 2026*
