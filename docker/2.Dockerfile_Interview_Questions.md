# Dockerfile — Interview Q&A

> **How to use this file:**
> Read the question out loud. Cover the answer. Try to say it naturally.
> Then check. Practice until it flows — not memorized word for word.

---

## 📌 Table of Contents

- [Basic Level Questions](#basic-level-questions)
- [Intermediate Level Questions](#intermediate-level-questions)
- [Advanced / SRE Level Questions](#advanced--sre-level-questions)
- [Multi-Stage Build Questions](#multi-stage-build-questions)
- [Scenario-Based Questions](#scenario-based-questions)
- [Rapid Fire Round](#rapid-fire-round)

---

## Basic Level Questions

---

### Q1. What is a Dockerfile? Walk me through the basic structure.

**How interviewer asks it:**
> "Can you explain what a Dockerfile is and what the key instructions are?"

**Your Answer:**

> "A Dockerfile is a plain text file with a set of step-by-step instructions that Docker reads top to bottom to build a container image. Think of it as a recipe — it tells Docker exactly what base to start from, what to install, what files to include, and how to run the application.
>
> The basic structure is:
> - `FROM` — defines the base image to start from
> - `RUN` — executes commands during the build process, like installing packages
> - `COPY` — brings files from the host machine into the image
> - `WORKDIR` — sets the working directory for subsequent instructions
> - `ENV` — sets environment variables available at runtime
> - `USER` — sets the user that runs the container process
> - `EXPOSE` — documents which port the app listens on
> - `HEALTHCHECK` — defines how Docker checks if the container is healthy
> - `ENTRYPOINT` / `CMD` — defines what command runs when the container starts
>
> Each instruction that modifies the filesystem creates a new immutable layer. Docker caches these layers to make subsequent builds faster."

---

### Q2. What is the difference between CMD and ENTRYPOINT?

**How interviewer asks it:**
> "This is a classic Docker question — explain CMD vs ENTRYPOINT and when you'd use each."

**Your Answer:**

> "Both define what runs when a container starts, but they behave differently when you pass arguments to `docker run`.
>
> **CMD** sets the **default command**. It can be completely **overridden** by arguments passed to `docker run`:
> ```bash
> docker run myimage               # runs CMD
> docker run myimage /bin/bash     # completely replaces CMD
> ```
>
> **ENTRYPOINT** sets a **fixed command** that always runs. Arguments from `docker run` are **appended** to it, not replacing it:
> ```bash
> # ENTRYPOINT ["node", "server.js"]
> docker run myimage               # runs: node server.js
> docker run myimage --port 9000   # runs: node server.js --port 9000
> ```
>
> **The power combination** is using both together:
> ```dockerfile
> ENTRYPOINT ["node", "server.js"]   # always runs node
> CMD ["--port", "8080"]             # default port, can be overridden
> ```
>
> I always use **exec form** (array syntax) for ENTRYPOINT because shell form makes `/bin/sh` PID 1 inside the container. This means `docker stop` sends SIGTERM to the shell, not your application — so the app doesn't shut down gracefully. With exec form, your process IS PID 1 and receives signals directly.
>
> Rule of thumb: use ENTRYPOINT for the main executable, CMD for default arguments."

---

### Q3. What is the difference between COPY and ADD?

**How interviewer asks it:**
> "Should I use COPY or ADD in my Dockerfile? What's the difference?"

**Your Answer:**

> "Both copy files into the image, but ADD has two extra abilities that COPY doesn't:
> - ADD can auto-extract local tar archives
> - ADD can download files from a URL
>
> However, the official Docker best practice is to **always use COPY** and avoid ADD unless you specifically need tar extraction.
>
> Reasons to avoid ADD:
> - **URL downloads** via ADD create a layer with the file but you can't clean up in the same layer. Better to use `RUN curl -fsSL URL | tar -xz` — one layer, smaller image.
> - ADD behavior is less obvious and predictable — a new team member might not realize ADD is extracting a tarball.
> - COPY is explicit and clear about what it does.
>
> One exception where ADD is valid: `ADD source.tar.gz /app/` when you have a local archive you want extracted. Otherwise, COPY always.
>
> Also importantly: in multi-stage builds, only `COPY --from=builder` works, not ADD."

---

### Q4. What is the difference between ARG and ENV?

**How interviewer asks it:**
> "I need to pass a version number into my Dockerfile. Should I use ARG or ENV?"

**Your Answer:**

> "The key difference is **when** they are available:
>
> **ARG** is a build-time variable. It's passed with `--build-arg` during `docker build`. It's NOT available when the container runs.
>
> **ENV** is both a build-time and **runtime** variable. It's baked into the image and available as an environment variable when the container runs. It can be overridden at runtime with `docker run -e`.
>
> A common pattern I use is combining them so something is configurable at build time AND available at runtime:
> ```dockerfile
> ARG APP_VERSION=1.0.0
> ENV APP_VERSION=${APP_VERSION}
> ```
>
> This way: `docker build --build-arg APP_VERSION=2.0.0` bakes the version in, and inside the running container `$APP_VERSION` is set.
>
> **Important security point:** Never put secrets in ARG or ENV. Both are visible in `docker history` and image metadata. Pass secrets at runtime via Docker secrets or your orchestration platform's secret management."

---

### Q5. What does EXPOSE do? Does it open the port?

**How interviewer asks it:**
> "If I add EXPOSE 8080 to my Dockerfile, can I access the app on port 8080 from my browser?"

**Your Answer:**

> "No — EXPOSE does NOT open or publish the port. It's purely **documentation**.
>
> EXPOSE tells humans and tooling: 'my app listens on this port.' It's like a label on a door — it says 'entrance here' but doesn't open the door.
>
> The actual port publishing happens with:
> - `-p 8080:8080` in `docker run` — maps host port 8080 to container port 8080
> - `-P` (capital P) in `docker run` — Docker auto-maps all EXPOSEd ports to random host ports
> - `ports:` key in `docker-compose.yml`
>
> EXPOSE is still useful because tools like Docker Compose use it for service discovery on internal networks, and `-P` uses it to know which ports to map. But it has zero effect on actual network connectivity."

---

## Intermediate Level Questions

---

### Q6. Why does layer order matter in a Dockerfile? How do you optimize for caching?

**How interviewer asks it:**
> "My builds are slow. How would you optimize a Dockerfile for faster builds?"

**Your Answer:**

> "Docker caches each layer. If a layer hasn't changed since the last build, Docker reuses it from cache — it doesn't re-execute that instruction. But here's the critical rule: **if one layer changes, all layers after it are rebuilt from scratch.**
>
> So the optimization principle is: **order instructions from least-to-most frequently changing.**
>
> A classic example with Node.js:
> ```dockerfile
> # ❌ Slow — any code change rebuilds npm install
> COPY . .
> RUN npm install
>
> # ✅ Fast — npm install only re-runs when package.json changes
> COPY package*.json ./
> RUN npm install
> COPY . .
> ```
>
> In practice, I follow this order:
> 1. `FROM` — base image (rarely changes)
> 2. `RUN` for OS package installs (rarely changes)
> 3. `COPY` dependency files only (package.json, requirements.txt, pom.xml)
> 4. `RUN` install dependencies (cached unless step 3 changes)
> 5. `COPY` application source code (changes every commit)
> 6. `RUN` build (rebuild only when source changes)
>
> This pattern cut our CI build time from 8 minutes to 90 seconds in a Node.js microservice pipeline because npm install was being cached on 95% of builds."

---

### Q7. Why should you chain RUN commands? Give an example.

**How interviewer asks it:**
> "I see some Dockerfiles with many RUN commands and some with few. What's the right approach?"

**Your Answer:**

> "You should chain RUN commands when they are related and especially when one step installs something and another cleans it up.
>
> The reason: each RUN creates an immutable layer. Even if you remove files in a later RUN, those files are still stored in the earlier layer. The image size includes ALL layers — you can't 'undo' a previous layer.
>
> Example:
> ```dockerfile
> # ❌ Wrong — cleanup layer does NOT reduce image size
> # apt-get packages are in layer 1 — they stay there forever
> RUN apt-get update
> RUN apt-get install -y curl wget
> RUN rm -rf /var/lib/apt/lists/*    # too late! already committed in layer above
>
> # ✅ Correct — everything in ONE layer, cleanup actually works
> RUN apt-get update \
>     && apt-get install -y --no-install-recommends curl wget \
>     && rm -rf /var/lib/apt/lists/*
> ```
>
> However — don't chain everything blindly. Instructions that are logically separate and change at different frequencies should be in separate RUN commands to preserve caching granularity. The goal is: chain commands that belong together logically AND need to be in the same layer for correctness or size."

---

### Q8. Why use WORKDIR instead of RUN cd?

**Your Answer:**

> "WORKDIR persists for all subsequent instructions in the Dockerfile. `RUN cd` only changes directory for that single RUN command — the next instruction goes back to the previous directory.
>
> ```dockerfile
> # ❌ This doesn't work as expected
> RUN cd /app
> RUN npm install   # still running from /, not /app!
>
> # ✅ WORKDIR persists
> WORKDIR /app
> RUN npm install   # correctly runs from /app
> COPY . .          # copies to /app
> CMD ["node", "server.js"]   # runs from /app
> ```
>
> Additional benefits of WORKDIR:
> - Creates the directory automatically if it doesn't exist
> - Makes the Dockerfile more readable — explicit about intent
> - Works correctly with relative paths in subsequent COPY instructions
> - Is the standard — tools, linters, and reviewers expect it"

---

### Q9. Why should you use USER in a Dockerfile? What are the risks of running as root?

**Your Answer:**

> "By default, containers run as **root** (UID 0). This is a serious security risk for several reasons:
>
> - If the application has a vulnerability and an attacker exploits it, they get **root privileges inside the container**
> - With volume mounts or misconfigured Docker socket access, root inside a container can potentially affect the host filesystem
> - It violates the principle of least privilege — your web app doesn't need root to serve HTTP requests
>
> The fix is simple:
> ```dockerfile
> RUN addgroup -S appgroup && adduser -S appuser -G appgroup
> WORKDIR /app
> COPY --chown=appuser:appgroup . .
> USER appuser
> CMD ["node", "server.js"]
> ```
>
> In my banking project at Atos, running containers as non-root was a hard security requirement from the client. We also combined it with `--read-only` filesystem and dropped all Linux capabilities except the ones the app needed. The combination means even if someone exploits the app, the blast radius is extremely limited.
>
> One thing to watch: if your app needs to bind to port 80 or 443, that requires root on Linux (ports below 1024). Solution: run the app on port 8080 and use an ingress/load balancer to handle 443 externally."

---

## Advanced / SRE Level Questions

---

### Q10. What is a multi-stage build? Why is it important in production?

**Your Answer:**

> "A multi-stage build uses multiple FROM instructions in a single Dockerfile. Each FROM starts a new stage. You can selectively copy artifacts from one stage to another using `COPY --from=stagename`.
>
> The key use case: your build process needs tools your production app doesn't. A Node.js app needs the TypeScript compiler, devDependencies, and test frameworks to build — but none of those are needed to run the compiled output.
>
> ```dockerfile
> # Stage 1: build with full toolchain
> FROM node:20-alpine AS builder
> WORKDIR /app
> COPY package*.json ./
> RUN npm ci                # includes devDependencies
> COPY . .
> RUN npm run build          # compiles TypeScript → dist/
>
> # Stage 2: minimal runtime
> FROM node:20-alpine AS production
> WORKDIR /app
> COPY package*.json ./
> RUN npm ci --only=production    # no devDependencies
> COPY --from=builder /app/dist ./dist
> USER node
> CMD ["node", "dist/server.js"]
> ```
>
> The production image never contains: TypeScript compiler, jest, webpack, source maps, test files, or devDependencies.
>
> Real impact I've seen: Node.js app from 1.2GB → 120MB. Go binary from 350MB → 12MB using scratch base. This matters because:
> - Smaller image = faster pull time in CI/CD
> - Less attack surface = fewer vulnerabilities to scan
> - Less storage cost in ECR/Registry"

---

### Q11. How do you handle secrets in a Dockerfile? What should you never do?

**Your Answer:**

> "This is a critical security topic. There are several **wrong** ways people commonly use and one right way.
>
> **Never do these:**
> ```dockerfile
> # ❌ ARG with secret — visible in docker history
> ARG DATABASE_PASSWORD=mysecret
>
> # ❌ ENV with secret — baked into image, visible to anyone who pulls it
> ENV API_KEY=supersecretkey
>
> # ❌ COPY .env — bakes your secrets file into the image
> COPY .env .
>
> # ❌ Downloading with credentials in URL — shows in layer metadata
> RUN curl https://user:password@internal.repo/file.tar.gz
> ```
>
> **Correct approaches:**
>
> 1. **Inject at runtime** — don't bake secrets into the image at all:
> ```bash
> docker run -e DATABASE_PASSWORD=$DB_PASS myimage
> # or from a file:
> docker run --env-file .env myimage
> ```
>
> 2. **BuildKit secret mounts** — for build-time secrets that should NEVER appear in layers:
> ```dockerfile
> # syntax=docker/dockerfile:1
> RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
>     npm install
> # The .npmrc (with npm token) is available ONLY during this RUN
> # It does NOT appear in any layer
> ```
> Build with: `docker build --secret id=npmrc,src=.npmrc .`
>
> 3. **Docker Swarm secrets or Kubernetes secrets** — mounted as files inside the container at runtime
>
> I check every Dockerfile in code review specifically for accidentally committed credentials. We also run `docker history` in CI to catch anything that slipped through."

---

### Q12. What is dumb-init and why would you use it in a container?

**Your Answer:**

> "Containers need a proper PID 1 process to handle signals and reap zombie processes. By default, when you run `CMD ["node", "server.js"]`, your node process becomes PID 1. But PID 1 has special responsibilities in Linux:
>
> - It must explicitly handle signals (SIGTERM, SIGCHLD) — most apps don't do this properly
> - It must reap zombie child processes — most apps don't do this either
>
> Problems that arise:
> - `docker stop` sends SIGTERM to PID 1. If node doesn't handle it properly, Docker waits 10 seconds and then sends SIGKILL. Your app gets no chance for graceful shutdown.
> - If your app spawns child processes (like running a shell command), dead children become zombie processes because PID 1 isn't reaping them.
>
> `dumb-init` is a tiny (< 1MB) init system designed for containers. It:
> - Properly forwards signals to your app and its children
> - Reaps zombie processes
>
> Usage:
> ```dockerfile
> RUN apk add --no-cache dumb-init
> ENTRYPOINT ["dumb-init", "--"]
> CMD ["node", "server.js"]
> ```
>
> Now the signal chain is: `docker stop` → SIGTERM → dumb-init → your app → graceful shutdown.
>
> Alternatives: `tini` (used by Docker Desktop itself), or `--init` flag in `docker run` which injects tini automatically."

---

### Q13. How does HEALTHCHECK work? How is it different from Kubernetes probes?

**Your Answer:**

> "HEALTHCHECK defines a command Docker runs periodically inside the container to determine if it's healthy.
>
> ```dockerfile
> HEALTHCHECK --interval=30s --timeout=10s --start-period=10s --retries=3 \
>     CMD curl -f http://localhost:8080/health || exit 1
> ```
>
> The container goes through three health states:
> - `starting` — within the start-period grace window
> - `healthy` — last check returned exit code 0
> - `unhealthy` — returned non-zero exit code `retries` times in a row
>
> In Docker Swarm, an `unhealthy` container is automatically replaced. In standalone Docker, it's just visible in `docker ps` — Docker doesn't restart it automatically (unless restart policy is set).
>
> **The important distinction for Kubernetes interviews:** Kubernetes completely **ignores** HEALTHCHECK from the Dockerfile. Instead it uses:
> - `livenessProbe` — is the container still alive? If it fails, kill and restart it.
> - `readinessProbe` — is the container ready to receive traffic? If it fails, remove it from the service's load balancer.
> - `startupProbe` — is the app still starting? Disables liveness check until this passes.
>
> So for a Kubernetes deployment, I still add HEALTHCHECK to the Dockerfile for consistency and for running the image in dev with plain Docker, but I always also define proper probes in the Kubernetes manifest."

---

## Multi-Stage Build Questions

---

### Q14. Can you have more than 2 stages in a multi-stage build?

**Your Answer:**

> "Yes, absolutely. You can have as many stages as you need. A common pattern I use for Node.js is three stages:
>
> ```dockerfile
> # Stage 1: Install all dependencies (including devDeps)
> FROM node:20-alpine AS deps
> WORKDIR /app
> COPY package*.json ./
> RUN npm ci
>
> # Stage 2: Build — uses deps stage
> FROM deps AS builder
> COPY . .
> RUN npm run build && npm run test
>
> # Stage 3: Production — minimal, only what's needed to run
> FROM node:20-alpine AS production
> WORKDIR /app
> COPY package*.json ./
> RUN npm ci --only=production
> COPY --from=builder /app/dist ./dist
> USER node
> CMD ["node", "dist/server.js"]
> ```
>
> Stages can reference each other. You can also target a specific stage to build:
> ```bash
> docker build --target builder -t myapp:test .   # stops at builder stage, useful for CI testing
> docker build --target production -t myapp:prod . # full production build
> ```
>
> This is great in CI pipelines — run tests in the builder stage, then only push the production stage to the registry."

---

### Q15. What is the `scratch` base image? When would you use it?

**Your Answer:**

> "`scratch` is a special Docker base image — it's completely empty. Zero bytes. No OS, no shell, no binaries, nothing.
>
> You'd use it when your application is a **statically compiled binary** that doesn't need any OS libraries. Go and Rust are the most common languages for this.
>
> ```dockerfile
> FROM golang:1.22-alpine AS builder
> WORKDIR /app
> COPY . .
> # CGO_ENABLED=0 = static binary, no C library dependency
> RUN CGO_ENABLED=0 GOOS=linux go build -o myapp .
>
> FROM scratch
> COPY --from=builder /app/myapp /myapp
> COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
> ENTRYPOINT ["/myapp"]
> ```
>
> Result: 8–15MB image vs 350MB with the full Go builder.
>
> Tradeoffs of scratch:
> - ✅ Absolutely minimal attack surface — nothing to exploit
> - ✅ Tiny image
> - ❌ No shell — you cannot `docker exec` into it for debugging
> - ❌ No package manager — can't add tools after the fact
> - ❌ Need to manually include anything the binary uses (TLS certs, timezone data)
>
> Alternative to scratch: `distroless` images from Google — they have no shell or package manager but include basic OS libs and are easier to use than scratch for most apps."

---

## Scenario-Based Questions

---

### Q16. "My Docker image is 2GB. How would you reduce it?"

**Your Answer:**

> "I'd attack this systematically in order of impact:
>
> **Step 1 — Audit first:**
> ```bash
> docker history myimage:latest     # see which layer is large
> docker run wagoodman/dive myimage # visual layer explorer
> ```
>
> **Step 2 — Apply these optimizations in order of impact:**
>
> 1. **Multi-stage build** — biggest win. Build in a full image, copy only the artifact to a minimal final stage. Node.js: 1.2GB → 120MB.
>
> 2. **Minimal base image** — switch from ubuntu (77MB) to alpine (5MB) or distroless.
>
> 3. **Chain RUN and clean up in same layer:**
>    ```dockerfile
>    RUN apt-get update && apt-get install -y curl \
>        && rm -rf /var/lib/apt/lists/*
>    ```
>
> 4. **.dockerignore** — ensure node_modules, .git, test files, docs are excluded from build context. A missing .dockerignore means COPY . . pulls everything in.
>
> 5. **Separate dev and prod dependencies** — `npm ci --only=production` in final stage.
>
> 6. **Use --no-cache flags** — `apk add --no-cache`, `pip install --no-cache-dir`.
>
> I track image sizes in our CI pipeline and fail the build if the image exceeds a defined threshold. This prevents gradual size creep."

---

### Q17. "My container works locally but fails in production with a permission denied error. What do you check?"

**Your Answer:**

> "Permission errors in containers usually come down to one of these causes:
>
> **Check 1 — Is the app running as root locally but non-root in production?**
> ```bash
> docker exec <container> whoami
> docker inspect <container> | jq '.[0].Config.User'
> ```
> If prod enforces non-root (via PodSecurityPolicy/PSA in Kubernetes), and the app tries to bind port 80 or write to a root-owned directory, it fails.
>
> **Check 2 — Volume mount ownership:**
> ```bash
> docker exec <container> ls -la /data
> ```
> If a volume is mounted and the directory is owned by root but the container runs as appuser (UID 1001), it can't write. Fix: `chown -R 1001:1001 /data` on the host, or use `--user` to match UIDs.
>
> **Check 3 — Entrypoint script not executable:**
> ```bash
> # In Dockerfile
> COPY entrypoint.sh .
> RUN chmod +x entrypoint.sh   # must be executable!
> ```
> If the script was committed without execute permission, or got the permission stripped during COPY, it fails with permission denied.
>
> **Check 4 — Read-only filesystem in production:**
> Some Kubernetes policies enforce `readOnlyRootFilesystem: true`. If the app tries to write to its own directory (like logs or temp files), it fails. Fix: mount a writable volume or emptyDir for those paths."

---

### Q18. "My build is very slow in CI. Each build takes 15 minutes. How do you fix it?"

**Your Answer:**

> "15 minutes usually means the cache isn't working. I'd check these in order:
>
> **Problem 1 — Layer order is wrong, busting cache on every commit:**
> ```dockerfile
> # If this is first, every code change busts npm install
> COPY . .
> RUN npm install
>
> # Fix: copy package files first
> COPY package*.json ./
> RUN npm install
> COPY . .
> ```
>
> **Problem 2 — CI pulling a fresh image every run with no cache:**
> ```bash
> # In CI, tell docker build to reuse the registry as cache source
> docker buildx build \
>   --cache-from=type=registry,ref=myrepo/myapp:cache \
>   --cache-to=type=registry,ref=myrepo/myapp:cache,mode=max \
>   -t myrepo/myapp:latest .
> ```
>
> **Problem 3 — Large build context being sent to daemon:**
> ```bash
> # Check the first line of docker build output
> Sending build context to Docker daemon  900MB   ← problem!
>
> # Fix: add .dockerignore with node_modules, .git, etc.
> ```
>
> **Problem 4 — Use BuildKit cache mounts for package managers:**
> ```dockerfile
> # syntax=docker/dockerfile:1
> RUN --mount=type=cache,target=/root/.npm \
>     npm ci
> ```
>
> With all these fixes combined, I've seen 15-minute builds drop to under 2 minutes."

---

## Rapid Fire Round

> These are short-answer questions. Practice giving crisp, 1-2 sentence answers.

| Question | Answer |
|----------|--------|
| What does `FROM scratch` mean? | Empty base image with nothing in it — used for statically compiled Go/Rust binaries |
| Can you have multiple FROM in one Dockerfile? | Yes — each FROM starts a new stage in a multi-stage build |
| What's the default file Docker looks for when you run `docker build .`? | A file named `Dockerfile` in the current directory |
| What's the difference between `docker build -f` and `docker build .`? | `-f` specifies a custom Dockerfile name/path; `.` sets the build context |
| What does `--no-cache` do in `docker build`? | Forces Docker to ignore all cached layers and rebuild every step from scratch |
| What is the build context? | The directory (and its contents) sent to the Docker daemon before building — controlled by `.dockerignore` |
| Can ENTRYPOINT be overridden? | Yes, but only with `--entrypoint` flag in `docker run` — not with regular arguments |
| What exit code should a health check return for success? | 0 for healthy, 1 for unhealthy |
| What is `dumb-init`? | A lightweight init system for containers that properly handles signals and reaps zombies |
| Can ARG be used after FROM? | Yes, but ARGs defined before FROM are only available in FROM. Redefine after FROM to use in the rest of the build. |

---

## 🎯 Phrases That Show Senior-Level Thinking

Use these naturally in your answers:

- *"In production I always use exec form for ENTRYPOINT because shell form breaks signal handling..."*
- *"We enforce a 200MB image size limit in our CI pipeline — any image over that fails the build..."*
- *"I run `docker history` to audit layers after every build to catch secrets or bloat..."*
- *"In my banking project, running as non-root was a hard security requirement from the client..."*
- *"Multi-stage builds reduced our deployment time by X minutes because smaller images pull faster..."*
- *"BuildKit cache mounts cut our pip install time from 4 minutes to 30 seconds in CI..."*

---

*File: `dockerfile_interview.md` | Topic 2 of 8 | Docker Interview Preparation 2026*
