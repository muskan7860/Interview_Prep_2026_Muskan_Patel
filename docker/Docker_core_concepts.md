# Docker Core Concepts — Theory + Practical Labs

> **Who is this for?**
> A DevOps engineer with ~4 years of experience preparing for interviews in 2025-2026.
> Written in simple English first, then professional terms — so you understand AND can speak confidently.

---

## 📌 Table of Contents

1. [Why Did Docker Come? The Problem Before Docker](#1-why-did-docker-come-the-problem-before-docker)
2. [What is Docker? Simple Explanation](#2-what-is-docker-simple-explanation)
3. [Image vs Container](#3-image-vs-container)
4. [Layers — How Docker Builds Things](#4-layers--how-docker-builds-things)
5. [Union Filesystem (OverlayFS)](#5-union-filesystem-overlayfs)
6. [Linux Namespaces — Isolation](#6-linux-namespaces--isolation)
7. [cgroups — Resource Control](#7-cgroups--resource-control)
8. [Container Runtime — containerd and runc](#8-container-runtime--containerd-and-runc)
9. [Practical Labs](#9-practical-labs)
10. [Quick Revision Cheatsheet](#10-quick-revision-cheatsheet)

---

## 1. Why Did Docker Come? The Problem Before Docker

### 🧠 Simple Story First

Imagine you build an app on your laptop. It works perfectly.
You send it to your friend (or your production server). They say:

> **"It's not working on my machine!"**

Why? Because their machine has:
- A different OS version
- Different libraries installed
- Different environment variables
- Different Python/Node/Java version

This was the **biggest problem** in software teams before Docker.

### 🏭 The Old Way — Virtual Machines (VMs)

Before Docker, teams used **Virtual Machines** to solve this.

A VM is like having a **full separate computer inside your computer**.
- It has its own OS, its own kernel, its own memory
- It solves the "works on my machine" problem
- BUT it is **very heavy** — each VM is 10–20 GB
- Takes **minutes to start**
- You can run maybe 5–10 VMs on one machine

### 🐳 Docker Came and Changed Everything

Docker introduced **containers**.

A container is like a **lightweight VM** — but it shares the host machine's OS kernel.
- Starts in **seconds**
- Uses **MBs instead of GBs**
- You can run **hundreds** of containers on one machine
- Ships your app with everything it needs — code, libraries, config

> **One-line answer for interviews:**
> *"Docker solves the 'works on my machine' problem by packaging the application and all its dependencies into a portable container that runs consistently across any environment."*

---

## 2. What is Docker? Simple Explanation

### 🎯 Analogy — Shipping Container

Think about how shipping works in the real world.

Before shipping containers existed, every ship had to load goods differently — boxes, barrels, sacks — it was a mess. Workers had to know exactly what each ship could carry.

Then someone invented the **standard shipping container**.
- Every container is the same shape and size
- Any ship, train, or truck can carry it
- You pack your goods inside — nobody cares what's inside

**Docker works the same way:**
- You pack your app + its dependencies inside a **Docker container**
- Any machine running Docker can run that container
- The machine doesn't need to know what's inside

### 🏗️ What Docker Actually Is

Docker is a **platform** (a tool + set of standards) that lets you:

| Action | What it means |
|--------|--------------|
| **Build** | Create an image from a Dockerfile |
| **Ship** | Push the image to a registry (Docker Hub, ECR) |
| **Run** | Start a container from that image anywhere |

### 🧩 Main Components of Docker

```
┌─────────────────────────────────────────────┐
│              Docker Platform                │
│                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │Dockerfile│  │  Image   │  │Container │  │
│  │(Recipe)  │→ │(Snapshot)│→ │(Running) │  │
│  └──────────┘  └──────────┘  └──────────┘  │
│                     ↑                       │
│               ┌──────────┐                  │
│               │ Registry │                  │
│               │(Storage) │                  │
│               └──────────┘                  │
└─────────────────────────────────────────────┘
```

---

## 3. Image vs Container

### 🧠 Simple Analogy

| Concept | Real World Example |
|---------|-------------------|
| **Image** | A **recipe** (or a class in programming) |
| **Container** | A **cooked dish** made from that recipe (or an object) |

You can cook the same recipe 10 times and get 10 dishes.
You can run the same image 10 times and get 10 containers.

The **recipe doesn't change** when you cook. Same with images — they are **read-only**.

### 📋 Technical Definition

**Image:**
- A read-only, immutable filesystem snapshot
- Built from a Dockerfile
- Made of multiple layers stacked on top of each other
- Stored in a registry (Docker Hub, ECR, GCR)
- Identified by a SHA256 digest (e.g., `sha256:abc123...`)
- Tagged for human use (e.g., `myapp:1.0.0`, `nginx:latest`)

**Container:**
- A running instance of an image
- Has a **thin writable layer** on top of the image layers
- Has its own **namespaces** (isolated environment)
- Has **cgroup limits** (resource control)
- Ephemeral by default — when you delete it, the writable layer is gone

### 🔑 Key Differences Table

| Feature | Image | Container |
|---------|-------|-----------|
| State | Read-only, immutable | Has a writable layer on top |
| Storage | Registry / local cache | Runs on host machine |
| Identity | SHA256 digest + tag | Container ID + name |
| Lifetime | Permanent (until deleted) | Ephemeral (temporary) |
| Multiple instances | One image | Many containers from one image |
| Data persistence | No | Only via volumes |

### 💡 Important Interview Point

> *"Multiple containers from the same image share all the read-only layers. Only the thin writable layer is unique per container. This is why Docker is so efficient — no data is duplicated."*

---

## 4. Layers — How Docker Builds Things

### 🧠 Simple Analogy — Transparent Sheets

Imagine you are making a drawing using **transparent sheets** stacked on top of each other.

- Sheet 1 (bottom): Background color
- Sheet 2: A house shape
- Sheet 3: Trees
- Sheet 4: Your name on top

When you look from above, you see everything combined.
If you remove Sheet 4, Sheets 1–3 are still there unchanged.

**Docker layers work exactly like this.**

### 📦 How Layers Are Created

Every instruction in a Dockerfile that **changes the filesystem** creates a new layer.

```dockerfile
FROM ubuntu:22.04          # Layer 1 — base OS
RUN apt-get update         # Layer 2 — update package list
RUN apt-get install -y python3  # Layer 3 — install python
COPY app.py /app/          # Layer 4 — copy your code
RUN pip install flask      # Layer 5 — install flask
```

```
┌─────────────────────────────────┐  ← Container writable layer (CoW)
├─────────────────────────────────┤
│   Layer 5: pip install flask    │
├─────────────────────────────────┤
│   Layer 4: COPY app.py          │
├─────────────────────────────────┤
│   Layer 3: install python3      │
├─────────────────────────────────┤
│   Layer 2: apt-get update       │
├─────────────────────────────────┤
│   Layer 1: ubuntu:22.04 base    │
└─────────────────────────────────┘
```

### ⚡ Layer Caching — Why Layer Order Matters

Docker **caches** each layer. If a layer hasn't changed, Docker reuses it from cache. This makes builds fast.

**But if one layer changes, ALL layers BELOW it are rebuilt.**

```dockerfile
# ❌ BAD ORDER — any code change rebuilds npm install
COPY . .               # If any file changes → cache MISS here
RUN npm install        # This runs again every time!

# ✅ GOOD ORDER — npm install cached unless package.json changes
COPY package.json .    # Only changes when dependencies change
RUN npm install        # Cached! Fast!
COPY . .               # Only app code here
```

> **Interview insight:** *"I always order Dockerfile instructions from least-frequently-changed to most-frequently-changed to maximize layer cache hits and speed up CI/CD build times."*

---

## 5. Union Filesystem (OverlayFS)

### 🧠 Simple Analogy

Imagine a **glass table** with transparent sheets.

- All the image layers are sheets placed on the table (you can see through them)
- The container gets one **opaque sheet on top** where it can write
- From above, you see everything combined as one surface
- But the sheets below are never touched

This combined view is called a **Union Filesystem**.

Docker uses **OverlayFS** (overlay2 storage driver) to do this.

### 🔧 How OverlayFS Works Technically

OverlayFS uses three key directories:

| Directory | Purpose |
|-----------|---------|
| `lowerdir` | All read-only image layers (separated by `:`) |
| `upperdir` | Container's writable layer — CoW writes go here |
| `merged` | The unified view the container process actually sees |
| `workdir` | Internal scratch space used by OverlayFS |

```
/var/lib/docker/overlay2/
├── <layer-sha>/
│   ├── diff/       ← actual files changed in this layer
│   ├── link        ← short ID
│   └── lower       ← colon-separated lower layer IDs
│
└── <container-layer>/
    ├── diff/       ← container's writable layer (CoW writes land here)
    ├── merged/     ← what the container sees (unified view)
    └── work/       ← OverlayFS internal scratch
```

### 📝 Copy-on-Write (CoW) Explained

When a container tries to **write** to a file that exists in a lower (read-only) layer:

1. Docker **copies** the file up from the lower layer to the container's `upperdir`
2. The container modifies the copy in `upperdir`
3. The original file in `lowerdir` is **never touched**

```
Container writes to /etc/nginx/nginx.conf

Step 1: File found in lower layer (read-only)
Step 2: Docker copies it UP to upperdir
Step 3: Container modifies the copy in upperdir
Step 4: On read, upperdir version is served (hides lower version)
```

> **Gotcha for interviews:** *"If a container modifies a large file — say 500MB — it copies the entire file up to the writable layer even for a 1-byte change. This is why we should never write large data inside containers — use volumes instead."*

---

## 6. Linux Namespaces — Isolation

### 🧠 Simple Analogy

Imagine you are in a **co-working space**.

Everyone shares the same building (the host machine / Linux kernel).
But each company has its own **private office** with:
- Their own door (network)
- Their own whiteboard (filesystem)
- Their own employee list (process list)
- Their own company name (hostname)

They can't see into each other's offices. That's **namespace isolation**.

### 🔑 The 6 Namespaces Docker Uses

| Namespace | What it isolates | Simple Example |
|-----------|-----------------|----------------|
| **pid** | Process IDs | Container's PID 1 is isolated from host PIDs |
| **net** | Network stack | Container has its own IP, ports, routing table |
| **mnt** | Filesystem mount points | Container sees its own `/` (root filesystem) |
| **uts** | Hostname | Container can have its own hostname |
| **ipc** | Shared memory, semaphores | Isolated inter-process communication |
| **user** | User and Group IDs | UID 0 inside container ≠ UID 0 on host (rootless) |

### ⚠️ Critical Point — Shared Kernel

```
┌─────────────────────────────────────────────┐
│              Host Machine                    │
│                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │Container1│  │Container2│  │Container3│  │
│  │(own ns)  │  │(own ns)  │  │(own ns)  │  │
│  └──────────┘  └──────────┘  └──────────┘  │
│                                             │
│  ══════════ SHARED HOST KERNEL ═══════════  │
└─────────────────────────────────────────────┘
```

> **This is the key difference between containers and VMs:**
> Containers share the host kernel. A kernel vulnerability affects ALL containers on the host.
> For untrusted or multi-tenant workloads, use gVisor or kata-containers (which give each container its own kernel).

---

## 7. cgroups — Resource Control

### 🧠 Simple Analogy

Namespaces answer: *"What can the container SEE?"*
cgroups answer: *"What can the container USE?"*

Think of a **hotel** with shared electricity.
- Each room (container) has its own key (namespace — isolated)
- But the hotel has a system that limits how much electricity each room can use (cgroups)
- Room 101 can't consume all the power and leave Room 102 in the dark

### 🔧 What cgroups Control

| Resource | Docker Flag | What It Does |
|----------|------------|--------------|
| **CPU** | `--cpus=1.5` | Limits CPU to 1.5 cores worth of time (CFS scheduler) |
| **Memory** | `--memory=512m` | Hard OOM limit — process is killed if exceeded |
| **Memory soft** | `--memory-reservation=256m` | Soft limit for scheduling hint |
| **PIDs** | `--pids-limit=100` | Max number of processes — prevents fork bombs |
| **Block I/O** | `--blkio-weight=500` | Throttle disk read/write speed |

### ⚙️ How CPU Limiting Works Internally

`--cpus=1.5` does NOT pin to 1.5 cores. It works via **CFS (Completely Fair Scheduler) quotas**:

- Every 100ms (period), the container gets 150ms of CPU time across all cores
- If all cores are idle, it can burst. If quota is consumed, it is throttled.

```bash
# What Docker sets internally:
CpuQuota  = 150000  (150ms)
CpuPeriod = 100000  (100ms)
# = 1.5 CPUs worth of time
```

> **SRE interview insight:** *"Always set both --memory and --memory-reservation in production. Without memory limits, one container can OOM-kill other containers on the same node. I've seen this happen in a banking app — an unbounded container consumed all node memory and took down 12 other microservices."*

---

## 8. Container Runtime — containerd and runc

### 🧠 Simple Analogy — Restaurant Kitchen

When you order food at a restaurant:
1. **Waiter** (Docker CLI) — takes your order, passes to kitchen
2. **Head Chef** (dockerd) — manages the kitchen, high-level decisions
3. **Kitchen Manager** (containerd) — coordinates the actual cooking, knows recipes
4. **Line Cook** (runc) — actually cooks the food using pans and gas (Linux kernel syscalls)

Each layer delegates to the next. You only talk to the waiter.

### 🏗️ The Full Runtime Stack

```
docker CLI
    │  (REST API via /var/run/docker.sock)
    ▼
dockerd  ← manages images, networks, volumes, Swarm
    │  (gRPC)
    ▼
containerd  ← CRI-compliant, manages container lifecycle
    │
    ▼
containerd-shim  ← one shim per container (keeps container alive even if containerd restarts)
    │
    ▼
runc  ← OCI runtime, calls Linux kernel (clone, cgroups, mounts)
    │
    ▼
Container Process  ← your actual app running
```

### 🔍 Role of Each Component

**dockerd (Docker Daemon):**
- The main background service
- Exposes Docker REST API
- Manages images (pull, push, build), networking, volumes
- Talks to containerd for the actual container work

**containerd:**
- Industry standard (CNCF project)
- CRI (Container Runtime Interface) compliant → Kubernetes uses this directly
- Manages container lifecycle: create, start, pause, stop, delete
- Manages image snapshots and storage

**containerd-shim:**
- One shim process per running container
- Critical for resilience: if containerd crashes and restarts, containers keep running
- Keeps STDIO (stdin/stdout/stderr) open
- Reports container exit status back to containerd

**runc:**
- OCI (Open Container Initiative) reference runtime
- The actual binary that talks to the Linux kernel
- Calls `clone()` with namespace flags
- Sets up cgroups, capabilities, seccomp profile
- Mounts the OverlayFS filesystem
- Exec's the container entrypoint
- **runc exits after the container starts** — the shim takes over

### 🚀 What Happens When You Run `docker run nginx`

```
Step 1: CLI sends POST /containers/create to dockerd via /var/run/docker.sock

Step 2: dockerd checks local image cache for nginx:latest
        → Not found? Pull from Docker Hub
        → Resolves manifest, downloads layers in parallel

Step 3: dockerd tells containerd: "create a container with this OCI bundle"
        OCI bundle = config.json (runtime spec) + rootfs (filesystem)

Step 4: containerd spawns a containerd-shim process

Step 5: shim calls runc create
        runc does:
          - clone() with CLONE_NEWPID | CLONE_NEWNET | CLONE_NEWMNT | CLONE_NEWUTS
          - Sets up cgroup limits
          - Mounts OverlayFS (lowerdir=image layers, upperdir=writable layer)
          - Drops unnecessary Linux capabilities
          - Applies seccomp profile (syscall filter)

Step 6: runc start → exec's the container process (nginx -g daemon off;)

Step 7: runc exits. Shim stays alive managing the process.

Step 8: dockerd sets up networking:
          - Creates a veth pair (virtual ethernet cable)
          - Connects one end to docker0 bridge
          - Places other end in container's net namespace
          - Assigns IP via IPAM (172.17.0.x)
          - Sets up iptables rules for port forwarding
```

### 🔑 Why This Matters for Kubernetes

> Kubernetes dropped dockershim in **version 1.24 (May 2022)**.
> It now talks directly to **containerd** (or CRI-O) via the CRI interface.
> Your existing Docker images still work perfectly — the OCI image spec is unchanged.
> Only the plumbing changed; containers didn't.

---

## 9. Practical Labs

> Run these on your Ubuntu machine with MicroK8s or standalone Docker.

---

### Lab 1 — Docker Installation Check and Info

```bash
# Check Docker version
docker version

# Full system info (storage driver, cgroup version, etc.)
docker info

# Check storage driver (should be overlay2)
docker info | grep "Storage Driver"

# Check cgroup version
docker info | grep "Cgroup"
```

**Expected output snippet:**
```
Storage Driver: overlay2
Cgroup Driver: systemd
Cgroup Version: 2
```

---

### Lab 2 — Pull an Image and Explore Layers

```bash
# Pull a small image
docker pull node:20-alpine

# See all layers and their sizes
docker history node:20-alpine

# See layer digests (SHA256 of each layer)
docker inspect node:20-alpine | jq '.[0].RootFS.Layers'

# Check total image size
docker images node:20-alpine

# Explore layers visually using dive tool
docker run --rm -it \
  -v /var/run/docker.sock:/var/run/docker.sock \
  wagoodman/dive node:20-alpine
```

---

### Lab 3 — Image vs Container in Action

```bash
# Run a container from nginx image
docker run -d --name webserver nginx

# List running containers
docker ps

# List all containers (including stopped)
docker ps -a

# See the IMAGE is unchanged, CONTAINER has writable layer
docker inspect webserver | jq '.[0].GraphDriver.Data'

# Write something to the container
docker exec webserver bash -c "echo 'Hello from container' > /tmp/test.txt"
docker exec webserver cat /tmp/test.txt

# Stop and remove the container — data is GONE
docker stop webserver && docker rm webserver

# Image still exists unchanged
docker images nginx
```

---

### Lab 4 — Observe Copy-on-Write (CoW)

```bash
# Start TWO containers from the SAME image
docker run -d --name c1 nginx
docker run -d --name c2 nginx

# Write to ONLY c1
docker exec c1 bash -c "echo 'I am container 1' > /tmp/test.txt"

# Verify isolation — c2 does NOT have the file
docker exec c1 cat /tmp/test.txt   # ✅ works
docker exec c2 cat /tmp/test.txt   # ❌ No such file

# Confirm both containers share the SAME lower layers
docker inspect c1 --format '{{.GraphDriver.Data.LowerDir}}'
docker inspect c2 --format '{{.GraphDriver.Data.LowerDir}}'
# Output should be IDENTICAL — same shared read-only layers

# But their UpperDir (writable layer) is DIFFERENT
docker inspect c1 --format '{{.GraphDriver.Data.UpperDir}}'
docker inspect c2 --format '{{.GraphDriver.Data.UpperDir}}'

# Cleanup
docker rm -f c1 c2
```

---

### Lab 5 — Observe Linux Namespaces

```bash
# Run a container
docker run -d --name nstest nginx

# Get container's PID on the HOST
CPID=$(docker inspect nstest --format '{{.State.Pid}}')
echo "Container process PID on host: $CPID"

# View the container's namespaces from the host
ls -la /proc/$CPID/ns/

# View HOST's namespace (PID 1 = systemd/init)
ls -la /proc/1/ns/

# Compare: different inode numbers = different namespaces!

# Enter the container's NETWORK namespace from the host
# (without going into the container)
sudo nsenter -t $CPID -n ip addr show
# You'll see eth0 with 172.17.x.x — the container's network

# Cleanup
docker rm -f nstest
```

---

### Lab 6 — cgroups Resource Limits

```bash
# Run container with explicit limits
docker run -d --name limited \
  --memory=128m \
  --memory-reservation=64m \
  --cpus=0.5 \
  --pids-limit=50 \
  nginx

# Verify limits are applied
docker inspect limited | jq '.[0].HostConfig | {
  Memory,
  MemoryReservation,
  CpuQuota,
  CpuPeriod,
  PidsLimit
}'

# Live resource usage
docker stats limited

# Check memory limit directly in cgroup filesystem (cgroups v2)
CID=$(docker inspect limited --format '{{.Id}}')
cat /sys/fs/cgroup/system.slice/docker-${CID}.scope/memory.max
# Output: 134217728 (= 128 * 1024 * 1024 bytes)

# Cleanup
docker rm -f limited
```

---

### Lab 7 — Understand the Runtime Stack

```bash
# See the actual runc binary
which runc
runc --version

# See containerd running
systemctl status containerd

# List containers via containerd directly (bypassing Docker)
sudo ctr containers list

# Watch Docker socket API calls in real time
# In terminal 1: watch the socket
sudo strace -e trace=network -p $(pgrep dockerd) 2>&1 | head -50

# In terminal 2: run a container
docker run --rm hello-world

# See containerd-shim processes (one per container)
docker run -d --name shimtest nginx
ps aux | grep containerd-shim

# Cleanup
docker rm -f shimtest
```

---

## 10. Quick Revision Cheatsheet

| Concept | One-Line Summary |
|---------|-----------------|
| **Image** | Read-only blueprint made of layers, stored in registry |
| **Container** | Running instance of image with thin writable layer + namespaces |
| **Layers** | Each Dockerfile instruction that changes FS = one immutable layer |
| **OverlayFS** | Merges read-only layers + writable layer into one unified view |
| **CoW** | Container copies file to writable layer before modifying it |
| **Namespaces** | Isolate what the container can SEE (pid, net, mnt, uts, ipc, user) |
| **cgroups** | Limit what the container can USE (CPU, memory, PIDs, I/O) |
| **containerd** | High-level runtime, CRI-compliant, manages lifecycle |
| **runc** | Low-level OCI runtime, calls Linux kernel to create container |
| **shim** | One per container, keeps container alive if containerd restarts |
| **docker.sock** | Unix socket Docker CLI uses to talk to dockerd |

---

## 📚 Further Reading

- [OCI Specification](https://opencontainers.org/)
- [containerd project](https://containerd.io/)
- [OverlayFS kernel docs](https://www.kernel.org/doc/html/latest/filesystems/overlayfs.html)
- [Docker official docs](https://docs.docker.com/)

---

*File: `docker_core_concepts.md` | Topic 1 of 8 | Docker Interview Preparation 2026*
