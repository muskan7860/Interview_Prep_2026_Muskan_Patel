# Docker Core Concepts — Interview Q&A

> **How to use this file:**
> Read the question. Close the answer. Try to answer in your own words OUT LOUD.
> Then check the answer below.
> Practice until you can say it naturally — not memorize word for word.

---

## 📌 Table of Contents

- [Basic Level Questions](#basic-level-questions)
- [Intermediate Level Questions](#intermediate-level-questions)
- [Advanced / SRE Level Questions](#advanced--sre-level-questions)
- [Scenario-Based Troubleshooting Questions](#scenario-based-troubleshooting-questions)
- [How to Answer — Pro Tips](#how-to-answer--pro-tips)

---

## Basic Level Questions

---

### Q1. What is Docker and why do we use it?

**How the interviewer asks it:**
> "Can you tell me what Docker is and why it became popular?"

**Your Answer:**

> "Before Docker, the biggest problem in software teams was the 'works on my machine' issue. An application worked perfectly on a developer's laptop but failed in production because of different OS versions, different library versions, or different environment configurations.
>
> Docker solved this by introducing **containers** — lightweight packages that bundle the application along with all its dependencies, libraries, and config into one portable unit.
>
> The key advantages are:
> - **Portability** — same container runs on laptop, CI server, and production
> - **Speed** — containers start in seconds, not minutes like VMs
> - **Efficiency** — you can run hundreds of containers on one machine vs 5–10 VMs
> - **Consistency** — eliminates environment differences between dev, staging, and prod
>
> In my experience, Docker became the foundation for CI/CD pipelines and Kubernetes deployments — almost every modern DevOps workflow uses it."

---

### Q2. What is the difference between a Docker image and a container?

**How the interviewer asks it:**
> "Explain image vs container. Use an analogy if you can."

**Your Answer:**

> "A **Docker image** is like a **recipe or a class in OOP** — it's a read-only, immutable template. It's built from a Dockerfile, stored in a registry, and made of multiple layers.
>
> A **Docker container** is like the **dish made from that recipe, or an object instantiated from the class** — it's a running instance of the image. It has the image layers (read-only) plus a thin writable layer on top.
>
> Key differences:
> - An image is immutable and permanent. A container is ephemeral — when you delete it, any data written to it is lost.
> - Multiple containers from the same image share all the read-only layers via OverlayFS. Only the writable layer is unique per container.
> - This is why Docker is so space-efficient — 10 containers from the same image don't create 10 copies of that image.
>
> In production, I always treat containers as stateless and ephemeral — I never write important data inside a container. That goes on a volume."

---

### Q3. What are Docker layers? Why do they matter?

**How the interviewer asks it:**
> "What is a layer in Docker? Why should I care about layers?"

**Your Answer:**

> "Every instruction in a Dockerfile that **modifies the filesystem** creates an immutable layer — essentially a filesystem snapshot of what changed at that step.
>
> Layers matter for two big reasons:
>
> **1. Caching — faster builds:**
> Docker caches each layer. If a layer hasn't changed, it reuses it from cache instead of rebuilding. This makes CI/CD builds much faster. But if one layer changes, all layers after it are rebuilt. So the **order of instructions matters** — put things that change rarely (like installing OS packages) at the top and things that change often (like copying your application code) at the bottom.
>
> **2. Storage efficiency:**
> Layers are shared across images. If five of your microservice images all use the same `node:20-alpine` base, that base layer is stored only once on disk and shared across all five.
>
> In my Dockerfiles, I always follow the pattern: install OS deps first, then copy package files, then run install, then copy app code. This maximizes cache hits."

---

### Q4. What is a Union Filesystem? What is OverlayFS?

**How the interviewer asks it:**
> "How does Docker's filesystem work internally?"

**Your Answer:**

> "Docker uses a **Union Filesystem** — a system that merges multiple directory trees (layers) into a single unified view without physically copying anything.
>
> Docker's default storage driver is **OverlayFS** (overlay2). It works with three key directories:
> - `lowerdir` — the read-only image layers
> - `upperdir` — the container's writable layer
> - `merged` — the unified view the container sees
>
> When the container reads a file, OverlayFS checks the writable layer first, then the lower layers. When the container **writes** to a file that exists in a lower layer, OverlayFS performs a **copy-on-write**: it copies the file up to the writable layer first, then modifies the copy there. The original in the lower layer is never touched.
>
> This is how multiple containers can share the same image layers while keeping their writes completely isolated from each other."

---

### Q5. What is the difference between Docker and a Virtual Machine?

**How the interviewer asks it:**
> "When would you use a container vs a VM?"

**Your Answer:**

> "Both solve the environment consistency problem, but at different levels:
>
> | | Container | VM |
> |--|--|--|
> | Kernel | Shared with host | Each has its own kernel |
> | Size | MBs | GBs |
> | Startup | Seconds | Minutes |
> | Isolation | Namespace isolation | Full hardware isolation |
> | Density | Hundreds per host | 5–10 per host |
>
> Containers are faster and lighter because they share the host kernel — they're just isolated Linux processes. VMs run a full OS including their own kernel on hypervisor-emulated hardware.
>
> In practice:
> - I use **containers** for microservices, CI/CD, stateless apps
> - I use **VMs** for workloads needing full OS isolation — like running untrusted code, Windows workloads, or compliance requirements that mandate kernel-level separation
>
> A common architecture is: run containers INSIDE VMs — get the security of VMs with the efficiency of containers. That's exactly what AWS ECS on EC2 and Kubernetes worker nodes do."

---

## Intermediate Level Questions

---

### Q6. Explain Linux namespaces. Which namespaces does Docker use?

**How the interviewer asks it:**
> "How does Docker achieve process isolation? Tell me about namespaces."

**Your Answer:**

> "Linux **namespaces** are a kernel feature that partitions global system resources, giving each namespace its own isolated instance of that resource.
>
> Docker uses 6 namespaces to isolate containers:
>
> - **pid namespace** — isolates process IDs. The container has its own PID 1 (usually your app or init), completely separate from host PIDs.
> - **net namespace** — isolates network stack. Each container gets its own network interfaces, IP address, routing table, and iptables rules.
> - **mnt namespace** — isolates mount points. The container has its own root filesystem view.
> - **uts namespace** — isolates hostname. A container can have its own hostname different from the host.
> - **ipc namespace** — isolates inter-process communication (shared memory, message queues).
> - **user namespace** — allows UID/GID remapping, enabling rootless containers where UID 0 inside the container maps to an unprivileged user on the host.
>
> The critical thing to understand is that namespaces provide **isolation** but not **resource limits**. And importantly, containers still share the host kernel. A kernel exploit can affect all containers on the host — this is why for multi-tenant untrusted workloads, you'd use gVisor or kata-containers."

---

### Q7. What are cgroups and what do they control in Docker?

**How the interviewer asks it:**
> "How does Docker prevent one container from eating all the CPU or memory?"

**Your Answer:**

> "Docker uses **Linux cgroups** (Control Groups) to enforce resource limits on containers.
>
> If namespaces control what a container can **see**, cgroups control what a container can **use**.
>
> Key resources cgroups control:
> - **CPU** — `--cpus=1.5` limits the container to 1.5 CPUs worth of CPU time via CFS scheduler quotas. It's not CPU pinning — the container can burst on idle cores but gets throttled once the quota for the period is consumed.
> - **Memory** — `--memory=512m` sets a hard OOM limit. If the container exceeds this, the kernel sends SIGKILL to the process. I also set `--memory-reservation` as a soft limit for the scheduler.
> - **PIDs** — `--pids-limit` prevents fork bombs where a malicious process spawns unlimited child processes.
> - **Block I/O** — throttle disk read/write rates.
>
> In production, I always set memory limits. I once saw an incident where an unbounded container consumed all node memory and caused OOM kills on 12 other containers running on the same Kubernetes node. After that, memory limits became a mandatory policy in our infrastructure."

---

### Q8. What is containerd? What is runc? How are they different from Docker?

**How the interviewer asks it:**
> "I heard Kubernetes dropped Docker. What is containerd and how does the runtime stack work?"

**Your Answer:**

> "The container runtime has a layered architecture:
>
> - **Docker CLI** — the command you type. Sends REST API calls to dockerd.
> - **dockerd** — Docker daemon. Manages images, volumes, networking, Compose, Swarm. The high-level entry point.
> - **containerd** — the actual container runtime. A CNCF project. It's CRI (Container Runtime Interface) compliant, which means Kubernetes can talk to it directly. It manages the container lifecycle: create, start, pause, stop, delete.
> - **containerd-shim** — one shim process per container. The shim is what makes containers survive a containerd restart or upgrade — it holds STDIO open and reports exit status.
> - **runc** — the OCI-compliant low-level runtime. It's the binary that actually calls the Linux kernel: sets up namespaces with `clone()`, configures cgroups, mounts OverlayFS, applies seccomp and capabilities, then exec's your process. After the container starts, runc exits — the shim takes over.
>
> About Kubernetes dropping Docker: Kubernetes removed the dockershim adapter in version 1.24. It now talks directly to containerd (or CRI-O) via CRI. This is actually cleaner — no need for the extra docker layer. Your Docker images still work perfectly because the OCI image spec is unchanged. The image format is the same; only the plumbing that runs them changed."

---

### Q9. What happens step by step when you run `docker run nginx`?

**How the interviewer asks it:**
> "Walk me through what happens internally when you run a Docker container."

**Your Answer:**

> "Sure, here's the full flow:
>
> 1. Docker CLI parses the command and sends a `POST /containers/create` request to dockerd via the Unix socket at `/var/run/docker.sock`.
>
> 2. dockerd checks the local image cache for `nginx:latest`. If it's not there, it contacts Docker Hub, resolves the image manifest, and pulls the layers in parallel, storing them in `/var/lib/docker/overlay2/`.
>
> 3. dockerd instructs containerd to create a container, passing it an OCI bundle — a `config.json` with the runtime spec and the rootfs path.
>
> 4. containerd spawns a `containerd-shim` process for this container.
>
> 5. The shim calls `runc create`, which: creates Linux namespaces via `clone()`, sets up cgroup limits, mounts the OverlayFS (lowerdir = image layers, upperdir = new writable layer), drops Linux capabilities, applies seccomp syscall filter.
>
> 6. `runc start` exec's the container's entrypoint — `nginx -g 'daemon off;'`. runc then exits.
>
> 7. The shim stays alive, managing the nginx process.
>
> 8. dockerd sets up networking: creates a virtual ethernet pair (veth), connects one end to the `docker0` bridge on the host, places the other end inside the container's network namespace, assigns an IP via IPAM (typically 172.17.0.x), and sets up iptables rules for port forwarding if `-p` was specified."

---

## Advanced / SRE Level Questions

---

### Q10. What is the difference between a container and a VM at the kernel level?

**Your Answer:**

> "At the kernel level, a container is simply a **regular Linux process** running with:
> - A different set of namespace contexts (so it sees an isolated environment)
> - cgroup limits (so it can't consume unlimited resources)
> - A different root filesystem (via OverlayFS mount)
>
> There's no hypervisor, no emulated hardware, no separate kernel. The container process makes system calls directly to the host kernel — just like any other process.
>
> A VM, by contrast, runs on a **hypervisor** (KVM, VMware, Hyper-V) that emulates hardware. The guest OS runs its own complete kernel inside that emulated hardware. System calls go through: guest app → guest kernel → hypervisor (trap + emulate) → host kernel. This is why VMs are heavier but provide stronger isolation.
>
> This is also why you **cannot** run a Linux container on Windows without a Linux kernel somewhere — Docker Desktop on Windows runs a lightweight Linux VM (using WSL2 or Hyper-V) and runs containers inside that."

---

### Q11. How would you investigate high disk usage caused by Docker?

**Your Answer:**

> "I'd start with `docker system df` which gives a breakdown of disk usage by images, containers, volumes, and build cache.
>
> Then I'd dig into each area:
>
> ```bash
> docker system df          # overall breakdown
> docker system df -v       # verbose — shows each item
>
> # Find large images
> docker images --format '{{.Size}}\t{{.Repository}}:{{.Tag}}' | sort -rh | head -20
>
> # Remove dangling images (untagged layers with no container)
> docker image prune
>
> # Remove all unused images, stopped containers, unused networks, build cache
> docker system prune -a
>
> # Remove specific old images
> docker rmi <image-id>
> ```
>
> Common causes I've seen:
> - Dangling images from frequent builds in CI (every build leaves a layer set)
> - Stopped containers piling up (run with `--rm` in CI/CD)
> - Build cache growing unbounded (run `docker builder prune` periodically)
> - Container logs filling `/var/lib/docker/containers/<id>/*-json.log` — configure log rotation via `--log-opt max-size=10m --log-opt max-file=3` or in daemon.json"

---

### Q12. A container is OOMKilled. How do you diagnose and fix it?

**Your Answer:**

> "OOMKilled means the container exceeded its memory limit and the kernel killed the process. Here's how I diagnose it:
>
> ```bash
> # Confirm OOMKilled
> docker inspect <container-id> | jq '.[0].State'
> # Look for: "OOMKilled": true
>
> # Check kernel OOM logs
> dmesg | grep -i 'oom\|killed'
> journalctl -k | grep -i oom
>
> # Watch memory usage over time (before it dies)
> docker stats <container-id>
> ```
>
> Fix strategies:
> 1. **Short-term** — increase the memory limit: `docker run --memory=1g ...`
> 2. **Investigate the leak** — use `docker stats` to watch growth, then profile the app (heap dumps, memory profiler depending on language)
> 3. **Set proper limits** — set both hard and soft: `--memory=512m --memory-reservation=256m`
> 4. **Check for memory leaks** — a container that grows over time has a leak. Restart policy is a band-aid, not a fix.
>
> In one banking project I worked on, a Java container was OOMKilled because the JVM wasn't aware of container memory limits (pre-JDK 10 issue). The JVM was sizing its heap based on total host RAM (128GB), not the container limit (2GB). Fix: add `-XX:+UseContainerSupport` or explicitly set `-Xmx1500m`."

---

## Scenario-Based Troubleshooting Questions

---

### Q13. "I built my image and it's 2GB. How do you reduce it?"

**Your Answer:**

> "I'd approach this systematically:
>
> **Step 1 — Audit layers with history:**
> ```bash
> docker history myimage:latest
> ```
> This shows which layers are large and which instruction created them.
>
> **Step 2 — Use dive for deep analysis:**
> ```bash
> docker run --rm -it -v /var/run/docker.sock:/var/run/docker.sock wagoodman/dive myimage:latest
> ```
>
> **Step 3 — Apply these optimizations:**
>
> - **Multi-stage build** — the biggest win. Build with full toolchain (node, gcc, maven) in stage 1, copy only the compiled artifact to a minimal stage 2.
> - **Chain RUN commands and clean up in same layer:**
>   ```dockerfile
>   RUN apt-get update && apt-get install -y curl \
>       && rm -rf /var/lib/apt/lists/*
>   ```
> - **Use minimal base image** — switch from `ubuntu` to `alpine` (5MB) or `distroless` (no shell at all)
> - **Add .dockerignore** — exclude node_modules, .git, test files, docs from build context
> - **Don't install dev dependencies** in the final image
>
> Real results I've achieved: Node.js app from 1.2GB → 120MB with multi-stage + alpine. Go binary from 800MB → 12MB using scratch base image."

---

### Q14. "Two developers have the same image but get different behavior in containers. Why?"

**Your Answer:**

> "Several possible causes:
>
> 1. **Different image tags** — they might both say `nginx:latest` but pulled at different times. `latest` is not pinned. Always use digest: `nginx@sha256:abc123...` in production.
>
> 2. **Environment variables** — different `-e` flags or `.env` files. Check: `docker inspect <id> | jq '.[0].Config.Env'`
>
> 3. **Volume mounts** — one developer might have a bind mount overriding container files: `docker inspect <id> | jq '.[0].Mounts'`
>
> 4. **Different Docker daemon versions** — OverlayFS behavior or networking can differ between Docker versions.
>
> 5. **State in a named volume** — if a named volume from a previous container run persists stale data.
>
> Debug process:
> ```bash
> # Compare exactly what's running
> docker inspect <container1> > c1.json
> docker inspect <container2> > c2.json
> diff c1.json c2.json
> ```
> This usually pinpoints the difference immediately."

---

### Q15. "The container starts and exits immediately. What do you do?"

**Your Answer:**

> "This is one of the most common Docker problems. Here's my systematic approach:
>
> ```bash
> # Step 1: Check exit code
> docker ps -a
> # ExitCode tells you what happened:
> # 0 = success (process completed normally)
> # 1 = app error
> # 137 = OOMKilled or killed by SIGKILL (128+9)
> # 139 = segfault (128+11)
> # 143 = SIGTERM (graceful shutdown)
>
> # Step 2: Read the logs
> docker logs <container-id>
>
> # Step 3: Override entrypoint to get a shell and investigate
> docker run -it --entrypoint /bin/sh myimage
> ```
>
> Most common causes:
> - **CMD runs a daemon that forks to background** — e.g., `CMD ["nginx"]` — nginx daemonizes and exits → container exits. Fix: `CMD ["nginx", "-g", "daemon off;"]`
> - **Application crashes on startup** — missing env var, can't connect to DB, missing config file
> - **Wrong entrypoint script** — not executable (`chmod +x`) or wrong line endings (CRLF on Windows)
> - **Missing dependency** — library not found at runtime"

---

## How to Answer — Pro Tips

### 🎯 The STAR formula for scenario questions

**S**ituation — what was the context?
**T**ask — what needed to be done?
**A**ction — what did YOU specifically do?
**R**esult — what was the outcome / what did you learn?

Example: *"In my Atos project, we had a Java container that was OOMKilled in production (Situation). I needed to find the root cause and fix it without increasing costs (Task). I used docker stats and dmesg to confirm OOMKilled, then found the JVM was sizing heap based on host RAM not container limit (Action). I added -XX:+UseContainerSupport to JVM flags and the issue was resolved — container memory stabilized at 1.8GB as expected (Result)."*

---

### 🎯 Phrases that impress interviewers at 4 YoE level

- *"In production, I always..."*
- *"The gotcha here is..."*
- *"I've seen this cause an incident when..."*
- *"The reason this matters for Kubernetes is..."*
- *"I audit this with... [tool name]"*

---

### 🎯 Questions you should ask the interviewer

- "Are you running Docker on Kubernetes or standalone in your environment?"
- "Do you use containerd directly or through Docker?"
- "What base images does your team standardize on?"
- "How do you handle image scanning in your CI/CD pipeline?"

These show you think beyond basics.

---

*File: `docker_core_concepts_interview.md` | Topic 1 of 8 | Docker Interview Preparation 2026*
