# Dockerfile — Theory + Practical Labs

> **Who is this for?**
> A DevOps engineer with ~4 years of experience preparing for interviews in 2025-2026.
> Simple analogy first → professional explanation → exact commands you can run on Ubuntu.

---

## 📌 Table of Contents

1. [What is a Dockerfile?](#1-what-is-a-dockerfile)
2. [FROM — The Starting Point](#2-from--the-starting-point)
3. [RUN — Execute Commands at Build Time](#3-run--execute-commands-at-build-time)
4. [COPY vs ADD — Bring Files Into the Image](#4-copy-vs-add--bring-files-into-the-image)
5. [CMD vs ENTRYPOINT — What Runs at Container Start](#5-cmd-vs-entrypoint--what-runs-at-container-start)
6. [EXPOSE — Document the Port](#6-expose--document-the-port)
7. [ENV vs ARG — Variables in Dockerfile](#7-env-vs-arg--variables-in-dockerfile)
8. [WORKDIR — Set the Working Directory](#8-workdir--set-the-working-directory)
9. [USER — Don't Run as Root](#9-user--dont-run-as-root)
10. [HEALTHCHECK — Is My Container Actually Healthy?](#10-healthcheck--is-my-container-actually-healthy)
11. [Multi-Stage Builds — The Game Changer](#11-multi-stage-builds--the-game-changer)
12. [.dockerignore — Keep the Build Context Clean](#12-dockerignore--keep-the-build-context-clean)
13. [Dockerfile Best Practices Summary](#13-dockerfile-best-practices-summary)
14. [Practical Labs](#14-practical-labs)
15. [Quick Revision Cheatsheet](#15-quick-revision-cheatsheet)

---

## 1. What is a Dockerfile?

### 🧠 Simple Analogy

A Dockerfile is like a **recipe card** for a dish.

- The recipe tells the chef: start with this base ingredient, add this, cook for this long, serve this way
- The Dockerfile tells Docker: start with this base image, run these commands, copy these files, start with this command
- Every time you follow the same recipe, you get the same dish
- Every time Docker builds the same Dockerfile, you get the same image

### 📋 Technical Definition

A Dockerfile is a **plain text file** with a set of instructions.
Docker reads it top to bottom and executes each instruction to build an image.
Each instruction that modifies the filesystem creates a new **immutable layer**.

```
Dockerfile  →  docker build  →  Image  →  docker run  →  Container
(recipe)        (cooking)      (dish)       (serving)     (eaten meal)
```

### 📁 Dockerfile Naming and Location

```bash
# Default name — Docker finds it automatically
Dockerfile

# Custom name — must specify with -f
docker build -f Dockerfile.prod -t myapp:prod .

# The dot (.) at the end = build context
# Docker sends the entire current directory to the daemon
# .dockerignore controls what gets sent
```

---

## 2. FROM — The Starting Point

### 🧠 Simple Analogy

`FROM` is like choosing the **type of kitchen** you will cook in.
- A professional restaurant kitchen (ubuntu:22.04) — has everything but is large
- A small home kitchen (alpine) — minimal but enough for simple dishes
- A completely empty room (scratch) — you bring every single thing yourself

### 📋 Syntax

```dockerfile
# Most common forms
FROM ubuntu:22.04
FROM node:20-alpine
FROM python:3.12-slim

# With a platform (for multi-arch builds — Apple M1 vs Intel)
FROM --platform=linux/amd64 node:20-alpine

# Scratch — empty base, for compiled Go/Rust binaries
FROM scratch

# In multi-stage builds — give the stage a name
FROM node:20-alpine AS builder
FROM nginx:alpine AS final
```

### 🔑 Choosing the Right Base Image

| Base Image | Size | Use When |
|-----------|------|----------|
| `ubuntu:22.04` | ~77MB | Need full Ubuntu tools, apt packages |
| `debian:slim` | ~75MB | Debian ecosystem, slightly smaller |
| `alpine:3.19` | ~5MB | Minimal, security-conscious, small final image |
| `node:20-alpine` | ~55MB | Node.js apps with minimal footprint |
| `python:3.12-slim` | ~125MB | Python apps without build tools |
| `distroless/nodejs` | ~33MB | No shell, no package manager — production only |
| `scratch` | 0MB | Go/Rust static binaries |

### ⚠️ Important Points

```dockerfile
# Always pin to a specific version — NEVER use latest in production
# Bad:
FROM node:latest       # What version is this today? Next week?

# Good:
FROM node:20.11.0-alpine3.19  # Pinned — reproducible builds

# Even better — pin by digest (immutable, can't be overwritten)
FROM node:20-alpine@sha256:a1b2c3d4...
```

> **Interview insight:** *"I always pin base image versions in production Dockerfiles. `latest` is a moving target — it can pull a different version between builds and break your app or introduce a security vulnerability without you knowing."*

---

## 3. RUN — Execute Commands at Build Time

### 🧠 Simple Analogy

`RUN` is the **cooking step** — do this action while building the image.
Boil water, fry onions, mix ingredients — each is a step that changes the dish.

`RUN` runs at **build time**, not at container start time.

### 📋 Two Forms

```dockerfile
# Shell form — runs via /bin/sh -c
RUN apt-get update && apt-get install -y curl

# Exec form — runs directly, no shell, no variable expansion
RUN ["apt-get", "install", "-y", "curl"]
```

Use **shell form** for most cases — it supports `&&`, `|`, `>`, variable expansion.
Use **exec form** when you don't want a shell involved (e.g., signals go directly to process).

### 🔑 The Most Critical Rule — Chain and Clean

**Each RUN creates a new layer.**
If you install packages and clean up in separate RUN commands, the cleanup doesn't actually reduce image size — the packages are already committed to the previous layer.

```dockerfile
# ❌ WRONG — 3 layers, cleanup layer does NOT reduce size
RUN apt-get update
RUN apt-get install -y curl wget vim
RUN rm -rf /var/lib/apt/lists/*

# ✅ CORRECT — 1 layer, cleanup happens IN THE SAME LAYER
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        curl \
        wget \
        vim \
    && rm -rf /var/lib/apt/lists/*
```

### 📦 Common RUN Patterns

```dockerfile
# Install packages — Ubuntu/Debian
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        curl \
        git \
        ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# Install packages — Alpine
RUN apk add --no-cache curl git

# Install Node.js dependencies
RUN npm ci --only=production

# Install Python dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Create a directory
RUN mkdir -p /app/logs

# Run a shell script
RUN chmod +x /scripts/setup.sh && /scripts/setup.sh
```

### ⚡ BuildKit Cache Mounts — Advanced

With Docker BuildKit (enabled by default since Docker 23), you can use cache mounts to speed up builds:

```dockerfile
# syntax=docker/dockerfile:1

# Cache the apt package list between builds
RUN --mount=type=cache,target=/var/cache/apt \
    apt-get update && apt-get install -y curl

# Cache npm packages between builds
RUN --mount=type=cache,target=/root/.npm \
    npm ci --only=production

# Cache pip packages between builds
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt
```

---

## 4. COPY vs ADD — Bring Files Into the Image

### 🧠 Simple Analogy

Both are ways to **bring ingredients into your kitchen**.

- `COPY` — you walk to the pantry and bring exactly what you asked for. Simple. Predictable.
- `ADD` — a magic delivery person who can go fetch from the internet AND unpack compressed boxes. Powerful but unpredictable.

### 📋 COPY

```dockerfile
# Copy a single file
COPY app.py /app/app.py

# Copy a directory (with trailing slash on destination)
COPY src/ /app/src/

# Copy multiple files using wildcard
COPY package*.json /app/

# Copy with specific ownership (--chown)
COPY --chown=appuser:appgroup src/ /app/src/

# In multi-stage builds — copy FROM a previous stage
COPY --from=builder /app/dist /app/dist
```

### 📋 ADD

```dockerfile
# ADD can download from URL (AVOID THIS)
ADD https://example.com/file.tar.gz /app/

# ADD auto-extracts tar archives
ADD source.tar.gz /app/
# This extracts the tarball contents into /app/

# ADD with local file — same as COPY (no reason to use ADD here)
ADD app.py /app/app.py
```

### 🔑 COPY vs ADD — When to Use Which

| Feature | COPY | ADD |
|---------|------|-----|
| Copy local files | ✅ Yes | ✅ Yes |
| Copy with `--chown` | ✅ Yes | ✅ Yes |
| Copy from build stage | ✅ Yes | ❌ No |
| Auto-extract .tar.gz | ❌ No | ✅ Yes |
| Download from URL | ❌ No | ✅ Yes |
| Recommended default | ✅ **Always** | ❌ Avoid |

> **Rule of thumb:** Always use `COPY`. Use `ADD` ONLY when you need to auto-extract a local tar file.
>
> **Why avoid ADD for URLs?** It creates a layer with the file but you can't clean up in the same layer. Instead, use `RUN curl -fsSL URL -o file.tar.gz && tar -xz && rm file.tar.gz` — all in one layer.

---

## 5. CMD vs ENTRYPOINT — What Runs at Container Start

### 🧠 Simple Analogy

Think of a **restaurant**:

- `ENTRYPOINT` = the type of restaurant — it's fixed. A pizza restaurant always makes pizza. You can't change that.
- `CMD` = the default order — if you don't say what you want, you get the house special. But you CAN change your order.

Put together: "This is a pizza restaurant (ENTRYPOINT) and if you don't choose, we serve Margherita (CMD)."

### 📋 CMD — Default Command

```dockerfile
# Shell form
CMD echo "Hello World"

# Exec form (preferred — cleaner signal handling)
CMD ["nginx", "-g", "daemon off;"]

# CMD with arguments for ENTRYPOINT
CMD ["--port", "8080"]
```

CMD sets the **default command** that runs when a container starts.
You can **override** it at `docker run` time:

```bash
docker run myimage                    # runs the CMD
docker run myimage /bin/bash          # overrides CMD, runs bash instead
```

### 📋 ENTRYPOINT — Fixed Command

```dockerfile
# Shell form (avoid — doesn't handle signals properly)
ENTRYPOINT echo "Hello"

# Exec form (preferred)
ENTRYPOINT ["node", "server.js"]
```

ENTRYPOINT is the command that **always runs**. You cannot override it with arguments in `docker run`.

```bash
docker run myimage                    # runs: node server.js
docker run myimage --port 9000        # runs: node server.js --port 9000
                                      # (arguments are APPENDED to ENTRYPOINT)
```

### 🔑 CMD + ENTRYPOINT Together — The Power Combination

```dockerfile
ENTRYPOINT ["node", "server.js"]   # always runs node server.js
CMD ["--port", "8080"]             # default port is 8080

# docker run myimage               → node server.js --port 8080
# docker run myimage --port 9000   → node server.js --port 9000  (CMD overridden)
```

This is the standard pattern for tools and CLIs — the tool is fixed (ENTRYPOINT), options are flexible (CMD).

### 📊 The Full Comparison Table

| | No ENTRYPOINT | ENTRYPOINT exec | ENTRYPOINT shell |
|--|--|--|--|
| **No CMD** | Error | runs ENTRYPOINT | runs ENTRYPOINT |
| **CMD exec** | runs CMD | ENTRYPOINT + CMD as args | ENTRYPOINT only |
| **CMD shell** | runs CMD in shell | ENTRYPOINT + CMD as args | ENTRYPOINT in shell |

### ⚠️ Shell Form Problem — Signal Handling

```dockerfile
# ❌ Shell form — PID 1 is /bin/sh, your process is a child
# Signals (SIGTERM from docker stop) go to /bin/sh, NOT your app
# Your app won't shut down gracefully
ENTRYPOINT node server.js

# ✅ Exec form — your process IS PID 1
# Signals go directly to node
ENTRYPOINT ["node", "server.js"]
```

> **SRE Interview insight:** *"I always use exec form for ENTRYPOINT. Shell form makes /bin/sh PID 1 inside the container, which means SIGTERM from docker stop goes to the shell, not the application. The app doesn't get a chance to gracefully shutdown — it gets SIGKILL after the timeout. This caused data corruption in a batch job I was running once."*

---

## 6. EXPOSE — Document the Port

### 🧠 Simple Analogy

`EXPOSE` is like a **label on a door** saying "people enter here."
It doesn't actually open the door — it just documents which door to use.

### 📋 Syntax

```dockerfile
EXPOSE 8080
EXPOSE 8080/tcp
EXPOSE 53/udp
EXPOSE 8080 443     # multiple ports
```

### 🔑 What EXPOSE Does and Doesn't Do

| What EXPOSE DOES | What EXPOSE DOES NOT do |
|-----------------|------------------------|
| Documents which port the app listens on | Open the port to the host |
| Used by Docker Compose for service-to-service networking | Publish the port (-p flag does that) |
| Visible in `docker inspect` | Create any firewall rules |
| Used by `docker run -P` (capital P) to auto-publish | Affect networking behavior in any way |

```bash
# -p (lowercase) = you specify the mapping explicitly
docker run -p 8080:8080 myapp

# -P (uppercase) = Docker auto-publishes all EXPOSEd ports to random host ports
docker run -P myapp
```

> **Interview answer:** *"EXPOSE is metadata — documentation for humans and tooling. It has no effect on networking. The actual port publishing happens with `-p host:container` in docker run or the `ports:` key in docker-compose.yml."*

---

## 7. ENV vs ARG — Variables in Dockerfile

### 🧠 Simple Analogy

- `ARG` = an ingredient you buy **before you start cooking** — only used during the cooking process, gone when the dish is done
- `ENV` = a **seasoning added to the dish itself** — it becomes part of the dish and is there when you eat it (when the container runs)

### 📋 ARG — Build-Time Variable

```dockerfile
# Define an ARG with optional default
ARG NODE_VERSION=20
ARG APP_ENV

# Use it during build
FROM node:${NODE_VERSION}-alpine

# Pass value at build time
# docker build --build-arg NODE_VERSION=18 .

# ARG is NOT available in the running container
# (not in ENV, not in the process environment)
```

### 📋 ENV — Runtime Environment Variable

```dockerfile
# Set environment variables
ENV APP_PORT=8080
ENV DATABASE_URL=postgresql://localhost:5432/mydb
ENV NODE_ENV=production

# Multi-line (preferred format)
ENV APP_PORT=8080 \
    NODE_ENV=production \
    LOG_LEVEL=info

# These ARE available when the container runs
# Access in app: process.env.APP_PORT (Node.js) or os.environ['APP_PORT'] (Python)
```

### 🔑 ARG vs ENV Comparison

| Feature | ARG | ENV |
|---------|-----|-----|
| Available during build | ✅ Yes | ✅ Yes (after the ENV line) |
| Available in running container | ❌ No | ✅ Yes |
| Can be overridden | `--build-arg` at build time | `-e` at run time |
| Visible in `docker history` | ✅ Yes (security concern!) | ✅ Yes (security concern!) |
| Use for | Build customization | App configuration |

### ⚠️ Security Warning — Never Store Secrets in ARG or ENV

```dockerfile
# ❌ NEVER do this — visible in docker history and image metadata
ARG DATABASE_PASSWORD=mysecretpassword
ENV API_KEY=supersecretkey123

# ✅ Use Docker secrets (Swarm) or runtime injection
# Pass at runtime — doesn't get baked into image
# docker run -e API_KEY=$API_KEY myimage

# ✅ Or use BuildKit secrets for build-time secrets
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
    npm install
```

### 💡 ARG + ENV Together Pattern

```dockerfile
# Common pattern: use ARG to set ENV so it's configurable at build time
# but also available at runtime
ARG APP_VERSION=1.0.0
ENV APP_VERSION=${APP_VERSION}

# Now you can:
# docker build --build-arg APP_VERSION=2.0.0 .
# And inside container: echo $APP_VERSION → 2.0.0
```

---

## 8. WORKDIR — Set the Working Directory

### 🧠 Simple Analogy

`WORKDIR` is like **going to a specific room** in your house before doing work.
Instead of saying "go to /home/muskan/projects/myapp and run this, then go to /home/muskan/projects/myapp again..." — you just say "from now on, I'm working in this room" and all commands run there.

### 📋 Syntax

```dockerfile
# Sets the working directory for all subsequent instructions
WORKDIR /app

# Now all RUN, COPY, ADD, CMD, ENTRYPOINT use /app as base
COPY package.json .         # copies to /app/package.json
RUN npm install             # runs in /app
COPY . .                    # copies everything to /app

# You can change it mid-Dockerfile
WORKDIR /app/frontend
RUN npm install

WORKDIR /app/backend
RUN npm install
```

### 🔑 Rules About WORKDIR

```dockerfile
# WORKDIR creates the directory if it doesn't exist
WORKDIR /app/logs   # creates /app and /app/logs if they don't exist

# Prefer WORKDIR over RUN cd
# ❌ Bad — cd only affects that single RUN command
RUN cd /app && npm install

# ✅ Good — WORKDIR persists for all subsequent instructions
WORKDIR /app
RUN npm install

# WORKDIR can use ENV variables
ENV APP_HOME=/application
WORKDIR $APP_HOME
```

---

## 9. USER — Don't Run as Root

### 🧠 Simple Analogy

Imagine hiring a contractor to work in your house.
Would you give them a **master key** that opens every door and lets them change the locks?
No — you give them a **specific key** that only opens the rooms they need.

Running containers as root is giving the master key. `USER` gives the specific key.

### 📋 Syntax

```dockerfile
# Create a non-root user (best practice)
RUN groupadd -r appgroup && useradd -r -g appgroup appuser

# OR on Alpine
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Switch to the non-root user
USER appuser

# All subsequent RUN, CMD, ENTRYPOINT run as this user
CMD ["node", "server.js"]
```

### 🔑 Why This Matters — Security

```dockerfile
# ❌ Default — container runs as root (UID 0)
# If the app is compromised, attacker has root inside container
# With volume mounts, they may be able to write to host filesystem as root
FROM node:20-alpine
COPY . /app
CMD ["node", "server.js"]
# Runs as root!

# ✅ Best practice — run as non-root user
FROM node:20-alpine

# Create user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app
COPY --chown=appuser:appgroup . .

RUN npm ci --only=production

# Switch to non-root user before CMD
USER appuser

CMD ["node", "server.js"]
```

### 💡 File Permission with USER

```dockerfile
# If your app needs to write to a directory, set ownership before switching USER
RUN mkdir -p /app/logs && chown -R appuser:appgroup /app/logs

USER appuser

# Now the app can write to /app/logs but nothing else
```

> **Security interview insight:** *"I always run containers as non-root. In our banking project at Atos, it was a hard security requirement. Even if a vulnerability is exploited inside the container, the attacker only has the privileges of the appuser — not root. Combined with read-only filesystem and no privilege escalation, it reduces blast radius significantly."*

---

## 10. HEALTHCHECK — Is My Container Actually Healthy?

### 🧠 Simple Analogy

Your container is **running** — but is it actually **working**?

A container can be running (process is alive) but the web server inside crashed and is returning 500 errors. Without a HEALTHCHECK, Docker and Kubernetes think everything is fine.

HEALTHCHECK is like a **nurse checking on a patient** every few minutes:
- "Can you hear me?" (ping the endpoint)
- If no response 3 times in a row → the patient is not healthy → alert/replace

### 📋 Syntax

```dockerfile
HEALTHCHECK [OPTIONS] CMD <command>

# Common options:
# --interval=30s    → check every 30 seconds (default: 30s)
# --timeout=10s     → if check takes longer than 10s, it fails (default: 30s)
# --start-period=5s → grace period after container starts (default: 0s)
# --retries=3       → consecutive failures needed to mark unhealthy (default: 3)

# Example — HTTP health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1

# Example — TCP port check (no curl available)
HEALTHCHECK --interval=15s --timeout=5s \
    CMD nc -z localhost 5432 || exit 1

# Example — check a process is running
HEALTHCHECK CMD pgrep nginx || exit 1

# Disable healthcheck inherited from base image
HEALTHCHECK NONE
```

### 🔑 Health States

| State | Meaning |
|-------|---------|
| `starting` | Container just started, within start-period |
| `healthy` | Last check passed |
| `unhealthy` | Consecutive failures reached retries limit |

```bash
# Check health status
docker inspect mycontainer | jq '.[0].State.Health'

# See health check log
docker inspect mycontainer | jq '.[0].State.Health.Log'
```

### 💡 HEALTHCHECK and Kubernetes

> **Important:** In Kubernetes, `HEALTHCHECK` in the Dockerfile is **ignored**. Kubernetes uses `livenessProbe` and `readinessProbe` in the Pod spec instead. But HEALTHCHECK is still useful for standalone Docker, Docker Compose, and Docker Swarm.

---

## 11. Multi-Stage Builds — The Game Changer

### 🧠 Simple Analogy

Imagine building a car:
- You need a **huge factory** with welding machines, paint booths, assembly robots (build tools)
- But the customer only needs the **finished car** (the artifact)
- You don't ship the factory with the car

Multi-stage builds let you:
- **Stage 1:** Use the full factory (Node.js + build tools + dev dependencies) to build the app
- **Stage 2:** Put only the finished car (compiled output + production dependencies) into a tiny final image

### 📋 Basic Multi-Stage Build — Node.js

```dockerfile
# ── Stage 1: Builder ──────────────────────────────────────────────────
FROM node:20-alpine AS builder

WORKDIR /app

# Install ALL dependencies (including devDependencies)
COPY package*.json ./
RUN npm ci

# Copy source and build
COPY . .
RUN npm run build
# Output: /app/dist/

# ── Stage 2: Production ───────────────────────────────────────────────
FROM node:20-alpine AS production

WORKDIR /app

# Only install production dependencies
COPY package*.json ./
RUN npm ci --only=production

# Copy ONLY the built artifact from the builder stage
COPY --from=builder /app/dist ./dist

# Security: non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

EXPOSE 8080
CMD ["node", "dist/server.js"]
```

**Result:**
- Builder stage: ~800MB (full Node + devDeps + source)
- Final image: ~120MB (minimal Node + prodDeps + dist only)

### 📋 Multi-Stage Build — Go Binary (Extreme Reduction)

```dockerfile
# ── Stage 1: Build the Go binary ──────────────────────────────────────
FROM golang:1.22-alpine AS builder

WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o myapp .

# ── Stage 2: Minimal runtime ──────────────────────────────────────────
FROM scratch

# Copy the static binary
COPY --from=builder /app/myapp /myapp

# Copy TLS certs (needed for HTTPS calls)
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

EXPOSE 8080
ENTRYPOINT ["/myapp"]
```

**Result:**
- Builder stage: ~350MB
- Final image: ~12MB (just the binary!)

### 📋 Multi-Stage Build — Java with Maven

```dockerfile
# ── Stage 1: Build with Maven ─────────────────────────────────────────
FROM maven:3.9-eclipse-temurin-17 AS builder

WORKDIR /app
COPY pom.xml .
RUN mvn dependency:resolve   # cache dependencies separately

COPY src ./src
RUN mvn package -DskipTests

# ── Stage 2: Runtime with JRE only ────────────────────────────────────
FROM eclipse-temurin:17-jre-alpine

WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar

RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

EXPOSE 8080
ENTRYPOINT ["java", "-XX:+UseContainerSupport", "-jar", "app.jar"]
```

### 💡 Targeting a Specific Stage

```bash
# Build only up to a specific stage (useful for testing)
docker build --target builder -t myapp:builder .

# Build the full final stage (default)
docker build -t myapp:prod .
```

### 🔑 Why Multi-Stage Matters

| Without Multi-Stage | With Multi-Stage |
|--------------------|-----------------|
| Build tools in production image | Build tools never reach prod |
| Source code in image | Only compiled output in prod |
| Dev dependencies in image | Only prod dependencies |
| Large image (GBs) | Small image (MBs) |
| Larger attack surface | Minimal attack surface |

---

## 12. .dockerignore — Keep the Build Context Clean

### 🧠 Simple Analogy

When you run `docker build .`, Docker sends **everything in the current directory** to the Docker daemon before building. This is the **build context**.

`.dockerignore` is like a **packing list** that says "DON'T pack these things." It keeps your suitcase (build context) light and fast.

### 📋 Example .dockerignore

```dockerignore
# Version control
.git
.gitignore

# Node.js
node_modules
npm-debug.log

# Python
__pycache__
*.pyc
.venv
venv/

# Tests and docs
tests/
*.test.js
*.spec.js
docs/
README.md

# IDE and OS
.vscode/
.idea/
.DS_Store
*.swp

# Build outputs (we'll build inside Docker)
dist/
build/
*.jar
*.war

# Environment files (never send secrets into build context)
.env
.env.local
*.env

# Docker files themselves
Dockerfile*
docker-compose*
```

### 🔑 Why .dockerignore Matters

```bash
# Without .dockerignore:
# Sending build context to Docker daemon  900MB  ← node_modules being sent!
# Each build uploads 900MB to daemon — slow

# With .dockerignore:
# Sending build context to Docker daemon  1.2MB  ← only source files
# Fast! And COPY . . won't accidentally include node_modules
```

---

## 13. Dockerfile Best Practices Summary

```dockerfile
# ✅ COMPLETE BEST-PRACTICE DOCKERFILE — Node.js App
# syntax=docker/dockerfile:1

# ── Stage 1: Dependencies ─────────────────────────────────────────────
FROM node:20-alpine AS deps

# Install OS-level security patches
RUN apk add --no-cache dumb-init

WORKDIR /app

# Copy dependency files first (maximize cache)
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

# ── Stage 2: Builder ──────────────────────────────────────────────────
FROM node:20-alpine AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

# ── Stage 3: Production ───────────────────────────────────────────────
FROM node:20-alpine AS production

# Use dumb-init as PID 1 for proper signal handling
COPY --from=deps /usr/bin/dumb-init /usr/bin/dumb-init

# Non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

# Copy production dependencies
COPY --from=deps /app/node_modules ./node_modules

# Copy built artifact
COPY --from=builder /app/dist ./dist

# Copy package.json for version info
COPY package.json .

# Set ownership
RUN chown -R appuser:appgroup /app

USER appuser

# Document the port
EXPOSE 8080

# Environment defaults
ENV NODE_ENV=production \
    PORT=8080

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=10s --retries=3 \
    CMD wget -qO- http://localhost:8080/health || exit 1

# Run with dumb-init for proper PID 1 behavior
ENTRYPOINT ["dumb-init", "--"]
CMD ["node", "dist/server.js"]
```

---

## 14. Practical Labs

### Lab 1 — Build Your First Dockerfile (Node.js App)

```bash
mkdir -p ~/docker-labs/nodejs-app && cd ~/docker-labs/nodejs-app

# Create a simple Node.js app
cat > app.js << 'EOF'
const http = require('http');
const PORT = process.env.PORT || 3000;

const server = http.createServer((req, res) => {
  if (req.url === '/health') {
    res.writeHead(200, {'Content-Type': 'application/json'});
    res.end(JSON.stringify({ status: 'healthy', version: process.env.APP_VERSION || '1.0' }));
    return;
  }
  res.writeHead(200, {'Content-Type': 'text/plain'});
  res.end('Hello from Docker! I am running as: ' + process.getuid() + '\n');
});

server.listen(PORT, () => console.log(`Server running on port ${PORT}`));
EOF

cat > package.json << 'EOF'
{
  "name": "docker-lab",
  "version": "1.0.0",
  "main": "app.js"
}
EOF

# Create Dockerfile
cat > Dockerfile << 'EOF'
FROM node:20-alpine

RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

COPY --chown=appuser:appgroup package*.json ./
COPY --chown=appuser:appgroup app.js .

USER appuser

ENV PORT=3000
ENV APP_VERSION=1.0.0

EXPOSE 3000

HEALTHCHECK --interval=10s --timeout=5s --start-period=5s --retries=3 \
    CMD wget -qO- http://localhost:3000/health || exit 1

CMD ["node", "app.js"]
EOF

# Create .dockerignore
cat > .dockerignore << 'EOF'
.git
node_modules
*.md
EOF

# Build
docker build -t myapp:1.0 .

# Run
docker run -d --name myapp -p 3000:3000 myapp:1.0

# Test
curl http://localhost:3000/
curl http://localhost:3000/health

# Check it's running as non-root
docker exec myapp whoami

# Watch health check
docker inspect myapp | jq '.[0].State.Health'

# Cleanup
docker rm -f myapp
```

---

### Lab 2 — Observe Layer Caching

```bash
cd ~/docker-labs/nodejs-app

# Build once (all layers built fresh)
time docker build -t myapp:cache-test .

# Build again immediately (all cached)
time docker build -t myapp:cache-test .
# Notice: "Using cache" on every step — near instant!

# Now modify app.js slightly
echo "// small change" >> app.js

# Build again
time docker build -t myapp:cache-test .
# Only the COPY app.js layer and below rebuild
# Earlier layers (FROM, RUN, COPY package.json) are still cached!

# See all layers
docker history myapp:cache-test
```

---

### Lab 3 — Multi-Stage Build vs Single Stage

```bash
mkdir -p ~/docker-labs/multistage && cd ~/docker-labs/multistage

# Create a Go app (static binary — perfect for multi-stage demo)
cat > main.go << 'EOF'
package main

import (
    "fmt"
    "net/http"
)

func main() {
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "Hello from Go! Running in minimal container.\n")
    })
    http.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Content-Type", "application/json")
        fmt.Fprintf(w, `{"status":"healthy"}`)
    })
    fmt.Println("Starting server on :8080")
    http.ListenAndServe(":8080", nil)
}
EOF

cat > go.mod << 'EOF'
module goapp
go 1.22
EOF

# Single-stage Dockerfile (large)
cat > Dockerfile.single << 'EOF'
FROM golang:1.22-alpine
WORKDIR /app
COPY . .
RUN go build -o server .
EXPOSE 8080
CMD ["./server"]
EOF

# Multi-stage Dockerfile (tiny)
cat > Dockerfile.multi << 'EOF'
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod ./
COPY main.go ./
RUN CGO_ENABLED=0 GOOS=linux go build -o server .

FROM scratch
COPY --from=builder /app/server /server
EXPOSE 8080
ENTRYPOINT ["/server"]
EOF

# Build both
docker build -f Dockerfile.single -t goapp:single .
docker build -f Dockerfile.multi  -t goapp:multi  .

# Compare sizes!
docker images | grep goapp
# goapp:single  ~350MB
# goapp:multi   ~8-12MB

# Run the multi-stage version
docker run -d --name goapp -p 8080:8080 goapp:multi
curl http://localhost:8080/
curl http://localhost:8080/health

# Cleanup
docker rm -f goapp
```

---

### Lab 4 — CMD vs ENTRYPOINT Behaviour

```bash
mkdir -p ~/docker-labs/entrypoint && cd ~/docker-labs/entrypoint

# Test 1: Only CMD
cat > Dockerfile.cmd << 'EOF'
FROM alpine
CMD ["echo", "default CMD message"]
EOF
docker build -f Dockerfile.cmd -t test:cmd .
docker run test:cmd                        # prints: default CMD message
docker run test:cmd echo "overridden"      # prints: overridden  (CMD replaced)

# Test 2: Only ENTRYPOINT
cat > Dockerfile.ep << 'EOF'
FROM alpine
ENTRYPOINT ["echo"]
EOF
docker build -f Dockerfile.ep -t test:ep .
docker run test:ep                         # prints: (empty line)
docker run test:ep "hello world"           # prints: hello world
docker run test:ep hello world             # prints: hello world (args appended)

# Test 3: ENTRYPOINT + CMD together
cat > Dockerfile.both << 'EOF'
FROM alpine
ENTRYPOINT ["echo", "Fixed prefix:"]
CMD ["default argument"]
EOF
docker build -f Dockerfile.both -t test:both .
docker run test:both                        # prints: Fixed prefix: default argument
docker run test:both "custom arg"           # prints: Fixed prefix: custom arg

# Cleanup
docker rmi test:cmd test:ep test:both
```

---

### Lab 5 — ARG and ENV

```bash
mkdir -p ~/docker-labs/args && cd ~/docker-labs/args

cat > Dockerfile << 'EOF'
FROM alpine

# Build-time only
ARG BUILD_VERSION=unknown
ARG BUILD_DATE

# Runtime — set from ARG
ENV APP_VERSION=${BUILD_VERSION}
ENV APP_ENV=production

# Show what's set
RUN echo "During build — BUILD_VERSION: ${BUILD_VERSION}"
RUN echo "During build — APP_VERSION: ${APP_VERSION}"

CMD ["sh", "-c", "echo Runtime APP_VERSION=$APP_VERSION && echo Runtime APP_ENV=$APP_ENV && echo Runtime BUILD_VERSION=${BUILD_VERSION:-not available}"]
EOF

# Build with defaults
docker build -t test:args .
docker run test:args

# Build with custom ARG
docker build --build-arg BUILD_VERSION=2.5.0 --build-arg BUILD_DATE=$(date +%Y%m%d) -t test:args2 .
docker run test:args2

# Override ENV at runtime
docker run -e APP_ENV=staging test:args2

# Notice: BUILD_VERSION is NOT available at runtime (ARG only)
# APP_VERSION IS available (it was set to ENV)
```

---

### Lab 6 — HEALTHCHECK in Action

```bash
cd ~/docker-labs/nodejs-app

# Run the app (it has a HEALTHCHECK)
docker run -d --name healthtest -p 3000:3000 myapp:1.0

# Watch health status change (starting → healthy)
watch -n 2 "docker inspect healthtest | jq '.[0].State.Health.Status'"

# Or check once
docker inspect healthtest | jq '.[0].State.Health'

# See last 5 health check results
docker inspect healthtest | jq '.[0].State.Health.Log'

# Kill the health check endpoint to simulate unhealthy
# (we'd need to break the /health endpoint — skip in lab)

# Cleanup
docker rm -f healthtest
```

---

## 15. Quick Revision Cheatsheet

| Instruction | When it runs | What it does | Key rule |
|------------|-------------|--------------|----------|
| `FROM` | Build time | Sets base image | Always pin the version |
| `RUN` | Build time | Executes command, creates layer | Chain + clean in one RUN |
| `COPY` | Build time | Copies local files into image | Prefer over ADD always |
| `ADD` | Build time | Like COPY but extracts tar, fetches URL | Use only for local tar extraction |
| `CMD` | Container start | Default command (overridable) | Use exec form |
| `ENTRYPOINT` | Container start | Fixed command (not overridable) | Use exec form for signal handling |
| `EXPOSE` | Documentation | Documents which port app uses | Doesn't publish the port |
| `ENV` | Build + Runtime | Sets environment variable | Never put secrets here |
| `ARG` | Build time only | Build-time variable | Not available at runtime |
| `WORKDIR` | Build time | Sets working directory | Creates dir if not exists |
| `USER` | Build + Runtime | Sets which user runs commands | Always switch to non-root before CMD |
| `HEALTHCHECK` | Runtime | Periodic health probe | Ignored by Kubernetes |

---

*File: `dockerfile_theory_lab.md` | Topic 2 of 8 | Docker Interview Preparation 2026*
