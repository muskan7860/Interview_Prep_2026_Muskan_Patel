# `05_Exit_Codes_Complete_Guide.md`

````markdown
# Exit Codes — Complete Guide for DevOps Engineers

> Level: 4 Years Experience  
> Author: Muskan Patel  
> Domain: Banking — Atos (US and European Clients)

---

## What Is an Exit Code

1. When any process (container, script, application) stops running,
   it returns a number to the operating system
2. That number is the exit code
3. Zero means success — anything else means something went wrong
4. In Kubernetes, exit codes tell you exactly WHY a container stopped
5. Exit code is always the FIRST thing you check in any container failure

---

## How to See Exit Codes

```bash
# See exit code of a failed pod
kubectl describe pod <pod-name> -n <namespace>

# Look for this section in output:
# Last State: Terminated
#   Reason: OOMKilled
#   Exit Code: 137
#   Started: Thu, 11 Jun 2026 10:00:00
#   Finished: Thu, 11 Jun 2026 10:00:05

# Also check with
kubectl get pod <pod-name> -o yaml | grep -A 10 "lastState"

# Check all restarting pods across cluster
kubectl get pods --all-namespaces | grep -v Running | grep -v Completed
```

---

## Exit Code 0

### What
Success. Container finished its job and exited cleanly.

### Why You See It
Job or CronJob completed successfully. One-time task finished.

### When It Is a Problem
Only if you see it on a container that should keep running
like a web server. A web server should NEVER exit with code 0.

### Fix
```bash
# Check what your container actually ran
kubectl logs <pod-name>

# Check CMD and ENTRYPOINT in your Dockerfile
# Web server probably ran a script that finished
# instead of starting the server process
```

---

## Exit Code 1

### What
General application error. The application itself crashed.

### Why You See It
- Java application threw NullPointerException
- Python script hit unhandled exception
- Node.js server failed to connect to database on startup
- Missing configuration or environment variable

### Investigation
```bash
# Always check logs first for exit code 1
kubectl logs <pod-name> --previous

# You will see something like:
# Exception in thread "main" java.lang.NullPointerException
# Error: Cannot connect to database at db-host:5432
# SyntaxError: Unexpected token in config.json
```

### Fix
1. Read the error in logs carefully
2. Fix database connection strings
3. Check environment variables are set correctly
4. Check configuration files exist and are valid

---

## Exit Code 2

### What
Misuse of shell command or shell built-in.
Script was called incorrectly.

### Why You See It
- Wrong arguments passed to command in entrypoint script
- Shell script syntax error

### Example
```bash
# In Dockerfile
CMD ["./start.sh", "--config"]
# But start.sh does not accept --config flag
# Container exits with code 2
```

### Fix
```bash
# Check your CMD and ENTRYPOINT in Dockerfile
# Run the command manually to verify it works
kubectl exec -it <pod-name> -- bash
# Then run the command manually and see the error
```

---

## Exit Code 125

### What
The container runtime itself failed to start the container.
Container never actually ran.

### Why You See It
Invalid container configuration, invalid runtime options,
container runtime error at infrastructure level.

### Fix
```bash
# Check pod events
kubectl describe pod <pod-name>
# Look at Events section
# Will show container runtime error
# Usually misconfigured securityContext
```

---

## Exit Code 126

### What
Container command found but cannot be executed.
Permission denied.

### Why You See It
Startup script exists but does not have execute permission.
File is there but chmod +x was not run.

### Example
```dockerfile
# Wrong — missing execute permission
COPY start.sh /app/start.sh
CMD ["/app/start.sh"]

# Correct
COPY start.sh /app/start.sh
RUN chmod +x /app/start.sh
CMD ["/app/start.sh"]
```

### Fix
```bash
# Check permission inside container
kubectl exec -it <pod-name> -- ls -la /app/start.sh
# If shows -rw-r--r-- (no x) that is the problem

# Add to Dockerfile
RUN chmod +x /app/start.sh
```

---

## Exit Code 127

### What
Command not found. Executable specified in CMD or
ENTRYPOINT does not exist in the container.

### Why You See It
- Typo in command name
- Wrong base image that does not have the binary
- Binary installed in different path than expected

### Example
```dockerfile
# Wrong — typo
CMD ["pythn", "app.py"]

# Wrong — binary in wrong path
CMD ["/usr/local/bin/myapp"]
# But binary is actually at /usr/bin/myapp
```

### Fix
```bash
# Run container interactively to find the binary
kubectl run debug --image=<your-image> -it --rm -- /bin/sh

# Inside container:
which python
which myapp
ls /usr/local/bin/
echo $PATH
```

---

## Exit Code 128

### What
Invalid argument to the exit command in a shell script.

### Why You See It
Shell script called exit with invalid argument
like exit -1 or exit abc.

### Fix
```bash
# exit only accepts values 0 to 255
# exit -1 is invalid — use exit 1 instead
# Check all your shell scripts for invalid exit calls
grep -r "exit -" /app/scripts/
```

---

## Exit Code 130

### What
Container terminated by Ctrl+C or SIGINT signal.

### Why You See It
Someone manually interrupted the container.
Script received SIGINT signal from outside.

### Is It a Problem
Usually not in development.
In production investigate who or what sent SIGINT.

---

## Exit Code 134

### What
Container received SIGABRT.
Application called abort() due to severe internal error.

### Why You See It
- Memory corruption detected
- C or C++ application hit assertion failure
- Java JVM crash

### Fix
```bash
# Check application logs for assertion failures
kubectl logs <pod-name> --previous

# Look for:
# Assertion failed
# SIGABRT
# JVM crash

# May need to increase memory limits
# Report to application developers — this is a code bug
```

---

## Exit Code 137 — MOST IMPORTANT

### What
Container was killed by SIGKILL signal.
Most commonly this is OOMKilled — Out Of Memory killed.

### Why You See It
- Container exceeded its memory limit
- Kubernetes killed it forcefully to protect the node
- No graceful shutdown — immediate death

### How to Confirm OOMKilled
```bash
# Method 1 — describe pod
kubectl describe pod <pod-name>

# Look for:
# Last State: Terminated
#   Reason: OOMKilled
#   Exit Code: 137

# Method 2 — check yaml
kubectl get pod <pod-name> -o yaml | grep -i oom

# Method 3 — check node level OOM
kubectl describe node <node-name> | grep -i oom
```

### Why This Happens
```yaml
# Container has memory limit of 128Mi
resources:
  limits:
    memory: "128Mi"
# But Java application needs 512Mi
# Kubernetes kills it when it hits 128Mi
# Exit code 137
```

### Fix
```bash
# Step 1 — check current memory usage
kubectl top pod <pod-name> -n <namespace>

# Step 2 — increase memory limit
kubectl edit deployment <name> -n <namespace>
# Change memory limit to higher value

# Step 3 — patch directly
kubectl patch deployment <name> -n <namespace> \
  --type='json' \
  -p='[{"op":"replace",
        "path":"/spec/template/spec/containers/0/resources/limits/memory",
        "value":"512Mi"}]'

# Step 4 — check for memory leak
# OOMKilled repeatedly = memory leak in code
# Profile the application
```

### Interview Answer
"Exit code 137 in Kubernetes almost always means OOMKilled.
The container exceeded its memory limit and was force killed
by the kernel. I confirm it by checking kubectl describe pod
and looking for Reason: OOMKilled in Last State.
The immediate fix is increasing the memory limit.
The proper fix is profiling the application for memory leaks.
In our banking application we had a payment service getting
OOMKilled every few hours. We increased the limit temporarily,
then profiled and found a HashMap that was never being cleared
after each request. Fixing the leak reduced memory usage by 80%."

---

## Exit Code 139

### What
Container received SIGSEGV — Segmentation Fault.
Application tried to access memory it does not own.

### Why You See It
- Bug in C or C++ native code
- JNI code in Java
- Memory corruption
- Buffer overflow

### Fix
```bash
# Check application logs
kubectl logs <pod-name> --previous
# Look for Segmentation fault or SIGSEGV

# This is always a code bug
# Report to application developers with full logs
```

---

## Exit Code 143

### What
Container received SIGTERM and exited gracefully.
This is a NORMAL graceful shutdown.

### Why You See It
Kubernetes sent SIGTERM to terminate the pod during:
- Rolling update
- Scale down
- Node drain
- Pod deletion

### Is It a Problem
No — this is exactly what you want.
Graceful shutdown working correctly.

### When It IS a Problem
```bash
# If application did not clean up properly
# Open connections not closed
# In-flight requests dropped

# Fix — give app more time to clean up
spec:
  terminationGracePeriodSeconds: 60
  containers:
  - name: payment-app
```

---

## Exit Code 255

### What
Exit status out of range OR process killed by external means.

### Why You See It
- Node was rebooted
- Container runtime was restarted
- Underlying VM was stopped
- This is what etcd showed in KillerCoda output

### Fix
```bash
# Check node events
kubectl describe node <node-name>

# Check system logs on the node
journalctl -u containerd --since "1 hour ago"
dmesg | tail -50

# In lab environment — this is normal after reboot
# In production — investigate what caused the restart
```

---

## Complete Reference Table

| Exit Code | Signal | Name | Common Cause | Problem |
|---|---|---|---|---|
| 0 | — | Success | Job completed | Only if server type |
| 1 | — | App Error | Exception, bad config | Yes — fix app |
| 2 | — | Shell Error | Wrong command usage | Yes — fix script |
| 125 | — | Runtime Error | Container runtime failure | Yes — check runtime |
| 126 | — | Permission Denied | Script not executable | Yes — chmod +x |
| 127 | — | Not Found | Binary missing or typo | Yes — fix CMD |
| 128 | — | Invalid Exit | Bad exit in script | Yes — fix script |
| 130 | SIGINT | Interrupted | Ctrl+C | Usually no |
| 134 | SIGABRT | Abort | Assert failure | Yes — code bug |
| 137 | SIGKILL | OOMKilled | Memory limit exceeded | Yes — most common |
| 139 | SIGSEGV | Segfault | Memory access violation | Yes — code bug |
| 143 | SIGTERM | Graceful Stop | Normal K8s shutdown | No — expected |
| 255 | — | Unknown | Node reboot, runtime restart | Investigate |

---

## Interview Questions — Exit Codes

### Question 1
**Your pod keeps restarting every 2 hours. How do you debug it?**

1. Run kubectl describe pod and check Last State exit code
2. If exit code 137 and Reason OOMKilled — memory limit too low
3. Run kubectl top pod to see current memory usage
4. Increase memory limit as immediate fix
5. Profile application for memory leak as permanent fix
6. Set up alert on pod restart count in Prometheus

---

### Question 2
**During rolling updates some requests are being dropped. Why?**

1. Rolling update sends SIGTERM to old pods (exit code 143)
2. If application does not handle SIGTERM gracefully —
   in-flight requests are dropped immediately
3. Fix — implement graceful shutdown in application
4. Catch SIGTERM signal
5. Stop accepting new requests
6. Finish processing current requests
7. Then exit cleanly
8. Also increase terminationGracePeriodSeconds to give
   application enough time to drain connections

---

### Question 3
**Pod crashes immediately after new deployment. Exit code 1. What do you do?**

1. Run kubectl logs pod-name --previous to see crash logs
2. Exit code 1 means application itself crashed
3. Look for exception stack traces in logs
4. Check for connection refused errors — database not reachable
5. Check for missing environment variables
6. Check for missing or invalid config files
7. Compare working deployment config with failing one
8. kubectl diff can show what changed between deployments

---

### Question 4
**What is the difference between exit code 137 and exit code 143?**

Exit code 137 is SIGKILL — forceful kill with no cleanup.
In Kubernetes this is almost always OOMKilled — container exceeded
memory limit and kernel killed it immediately.
No graceful shutdown happens.

Exit code 143 is SIGTERM — graceful termination request.
Kubernetes sends this when it wants to stop a pod normally —
during rolling updates, scale down, or pod deletion.
Application has terminationGracePeriodSeconds to clean up
before being force killed.

137 is always a problem to investigate.
143 is expected and healthy behaviour.

---

*Next file: 06_MicroK8s_Complete_Guide.md*
````

---

# `06_MicroK8s_Complete_Guide.md`

````markdown
# MicroK8s — Zero to Hero Complete Guide

> Level: 4 Years Experience
> Author: Muskan Patel
> Lab Setup: MicroK8s on ThinkPad L14

---

## What Is MicroK8s

### Simple Explanation

1. MicroK8s is Kubernetes packaged as a single snap package for Linux
2. Created and maintained by Canonical (makers of Ubuntu)
3. Instead of installing and configuring 10 different components
   manually — one command gives you a working cluster in 5 minutes
4. Behaves identically to production Kubernetes for all concepts
   that matter in interviews
5. Used for local development, testing, and learning

### Why Use MicroK8s for Interview Preparation

1. Installs in one command — no complex setup
2. Runs on your laptop with low resource usage
3. Same kubectl commands as production
4. Same YAML syntax as production
5. Same pod lifecycle, networking, storage behaviour
6. Built-in addons for DNS, monitoring, storage, dashboard
7. Can practice real troubleshooting scenarios locally

---

## How MicroK8s Is Different From Standard Kubernetes

### Standard Kubernetes (kubeadm) — Control Plane as Pods

```bash
# Control plane runs as static pods
kubectl get pods -n kube-system
# You see:
# etcd-controlplane
# kube-apiserver-controlplane
# kube-scheduler-controlplane
# kube-controller-manager-controlplane

# Config files at:
ls /etc/kubernetes/manifests/
# etcd.yaml
# kube-apiserver.yaml
# kube-scheduler.yaml
# kube-controller-manager.yaml
```

### MicroK8s — Control Plane as Snap Services

```bash
# Control plane runs as Linux snap services
snap services microk8s
# You see:
# microk8s.daemon-apiserver
# microk8s.daemon-etcd
# microk8s.daemon-scheduler
# microk8s.daemon-controller-manager
# microk8s.daemon-kubelet
# microk8s.daemon-proxy

# Config files at:
ls /var/snap/microk8s/current/args/
# kube-apiserver
# etcd
# kube-scheduler
# kube-controller-manager
# kubelet
```

### Key Differences Table

| Feature | Standard K8s | MicroK8s |
|---|---|---|
| Control plane | Static pods in kube-system | Snap services |
| API server port | 6443 | 16443 |
| etcd port | 2379 | 12379 |
| Config location | /etc/kubernetes/ | /var/snap/microk8s/current/ |
| Cert location | /etc/kubernetes/pki/ | /var/snap/microk8s/current/certs/ |
| Cert renewal | kubeadm certs renew all | microk8s refresh-certs |
| Troubleshoot | kubectl logs pod | journalctl -u snap.microk8s.daemon-* |

---

## MicroK8s Architecture

````
Your Laptop
│
├── SNAP SERVICES (control plane)
│   ├── microk8s.daemon-apiserver
│   │   └── Listens on port 16443
│   │   └── Config: /var/snap/microk8s/current/args/kube-apiserver
│   │
│   ├── microk8s.daemon-etcd
│   │   └── Listens on port 12379
│   │   └── Data: /var/snap/microk8s/current/var/kubernetes/backend/
│   │
│   ├── microk8s.daemon-scheduler
│   │   └── Watches for unscheduled pods
│   │
│   ├── microk8s.daemon-controller-manager
│   │   └── Runs all reconciliation loops
│   │
│   ├── microk8s.daemon-kubelet
│   │   └── Manages pods on this node
│   │
│   └── microk8s.daemon-proxy
│       └── Manages iptables rules for services
│
├── PODS in kube-system (add-ons)
│   ├── calico-node (CNI — pod networking)
│   ├── calico-kube-controllers
│   ├── coredns (internal DNS)
│   └── hostpath-provisioner (storage)
│
└── YOUR APPLICATION PODS
    └── In default or custom namespaces
````

---

## Installation From Zero

```bash
# Step 1 — install MicroK8s
sudo snap install microk8s --classic --channel=1.30

# Step 2 — add user to microk8s group
# WHY: Without this you need sudo for every command
sudo usermod -a -G microk8s $USER
sudo chown -f -R $USER ~/.kube

# Step 3 — apply group change without logout
newgrp microk8s

# Step 4 — wait for ready
microk8s status --wait-ready

# Step 5 — set up kubectl alias
echo "alias kubectl='microk8s kubectl'" >> ~/.bashrc
echo "alias k='microk8s kubectl'" >> ~/.bashrc
source ~/.bashrc

# Step 6 — export kubeconfig
microk8s config > ~/.kube/config

# Step 7 — verify
kubectl get nodes
kubectl get pods --all-namespaces
```

---

## Enable Essential Addons

```bash
# DNS — REQUIRED for pod-to-service communication
microk8s enable dns

# Storage — for PersistentVolume practice
microk8s enable hostpath-storage

# Metrics server — for kubectl top and HPA
microk8s enable metrics-server

# Dashboard — web UI
microk8s enable dashboard

# Ingress — for ingress controller practice
microk8s enable ingress

# Prometheus and Grafana stack
microk8s enable prometheus

# Check what is enabled
microk8s status
```

---

## All Commands — Complete Reference

### Status and Health

```bash
# Overall status
microk8s status

# Wait until ready
microk8s status --wait-ready

# Check snap services
snap services microk8s

# Full diagnostic report
microk8s inspect

# Check version
microk8s version
```

### Start and Stop

```bash
# Stop (saves battery)
microk8s stop

# Start
microk8s start

# Restart entire MicroK8s
microk8s stop && microk8s start

# Restart specific service
sudo snap restart microk8s.daemon-apiserver
sudo snap restart microk8s.daemon-etcd
sudo snap restart microk8s.daemon-scheduler
sudo snap restart microk8s.daemon-controller-manager
sudo snap restart microk8s.daemon-kubelet
```

### Control Plane Logs

```bash
# API server logs
journalctl -u snap.microk8s.daemon-apiserver -f

# etcd logs
journalctl -u snap.microk8s.daemon-etcd -f

# Scheduler logs
journalctl -u snap.microk8s.daemon-scheduler -f

# Controller manager logs
journalctl -u snap.microk8s.daemon-controller-manager -f

# kubelet logs
journalctl -u snap.microk8s.daemon-kubelet -f

# Last 50 lines of any component
journalctl -u snap.microk8s.daemon-apiserver \
  --lines=50 --no-pager

# Logs since specific time
journalctl -u snap.microk8s.daemon-apiserver \
  --since "10 minutes ago" --no-pager

# Check for errors only
journalctl -u snap.microk8s.daemon-apiserver \
  --no-pager | grep -i error
```

### Config Files

```bash
# View all config files
ls /var/snap/microk8s/current/args/

# View API server config
cat /var/snap/microk8s/current/args/kube-apiserver

# View etcd config
cat /var/snap/microk8s/current/args/etcd

# View kubelet config
cat /var/snap/microk8s/current/args/kubelet

# View scheduler config
cat /var/snap/microk8s/current/args/kube-scheduler

# View controller manager config
cat /var/snap/microk8s/current/args/kube-controller-manager

# Edit API server config
sudo vi /var/snap/microk8s/current/args/kube-apiserver
# After editing restart the service
sudo snap restart microk8s.daemon-apiserver
```

### kubectl Commands

```bash
# Nodes
kubectl get nodes
kubectl get nodes -o wide
kubectl describe node <node-name>

# Pods
kubectl get pods --all-namespaces
kubectl get pods -n kube-system
kubectl get pods -n kube-system -o wide
kubectl describe pod <pod-name> -n kube-system
kubectl logs <pod-name> -n kube-system
kubectl logs <pod-name> -n kube-system --previous
kubectl exec -it <pod-name> -n kube-system -- bash

# Control plane process check
ps aux | grep kube-apiserver
ps aux | grep etcd
ps aux | grep kube-scheduler
ps aux | grep kube-controller

# Resource usage
kubectl top nodes
kubectl top pods --all-namespaces
```

### Multi-node MicroK8s

```bash
# Generate join command on primary node
microk8s add-node

# Join from second machine
microk8s join <primary-ip>:25000/<token>

# Remove a node
microk8s remove-node <node-name>

# Verify all nodes
kubectl get nodes
```

### Dashboard

```bash
# Enable dashboard
microk8s enable dashboard

# Get login token
token=$(kubectl -n kube-system get secret | \
  grep default-token | cut -d " " -f1)
kubectl -n kube-system describe secret $token

# Access dashboard
microk8s dashboard-proxy
```

---

## Certificate Commands

```bash
# View all certs
ls /var/snap/microk8s/current/certs/

# Check API server cert expiry
openssl x509 \
  -in /var/snap/microk8s/current/certs/server.crt \
  -noout -dates

# Check CA cert expiry
openssl x509 \
  -in /var/snap/microk8s/current/certs/ca.crt \
  -noout -dates

# Check all certs at once
for cert in /var/snap/microk8s/current/certs/*.crt; do
  echo "=== $cert ==="
  openssl x509 -in $cert -noout -dates 2>/dev/null
done

# See which IPs cert is valid for
openssl x509 \
  -in /var/snap/microk8s/current/certs/server.crt \
  -noout -text | grep -A 5 "Subject Alternative Name"

# Refresh specific cert
microk8s refresh-certs --cert server.crt

# Refresh all certs
microk8s refresh-certs

# Regenerate kubeconfig after refresh
microk8s config > ~/.kube/config
```

---

## Troubleshooting Scenarios

### Problem 1 — kubectl not working after laptop reboot

```bash
# Check if MicroK8s started
microk8s status

# If not running
microk8s start
microk8s status --wait-ready

# Check if IP changed (most common cause)
hostname -I
cat ~/.kube/config | grep server
# If IP in kubeconfig is different from hostname -I output
# Regenerate kubeconfig
microk8s config > ~/.kube/config
kubectl get nodes
```

### Problem 2 — kubectl hangs and never returns

```bash
# Check API server is running
snap services microk8s | grep apiserver

# Check API server logs for errors
journalctl -u snap.microk8s.daemon-apiserver \
  --lines=30 --no-pager

# Check if IP changed after WiFi change
hostname -I

# Regenerate kubeconfig with current IP
microk8s config > ~/.kube/config
kubectl get nodes

# If still hanging — full restart
microk8s stop
microk8s start
microk8s status --wait-ready
```

### Problem 3 — Pods stuck in Pending

```bash
# Check pod events
kubectl describe pod <pod-name>

# Check if DNS is enabled
microk8s status | grep dns
microk8s enable dns

# Check node resources
kubectl describe node | grep -A 5 "Allocated resources"

# Check if storage addon needed
microk8s enable hostpath-storage
```

### Problem 4 — CoreDNS not working

```bash
# Check CoreDNS pod
kubectl get pods -n kube-system | grep coredns

# Check CoreDNS logs
kubectl logs -n kube-system <coredns-pod>

# Restart DNS addon
microk8s disable dns
microk8s enable dns

# Test DNS from inside a pod
kubectl run test --image=busybox --rm -it --restart=Never \
  -- nslookup kubernetes
```

### Problem 5 — metrics-server not working

```bash
# Enable metrics server
microk8s enable metrics-server

# Wait for it to start
kubectl get pods -n kube-system | grep metrics-server

# Test
kubectl top nodes
kubectl top pods --all-namespaces
```

---

## Interview Questions — MicroK8s

### Question 1
**What is MicroK8s and why did you use it?**

MicroK8s is a lightweight Kubernetes distribution by Canonical
that installs as a single snap package. I chose it for my practice
lab because it installs in one command, runs on my laptop with
minimal resources, and behaves identically to production Kubernetes
for everything that matters in interviews — same kubectl commands,
same YAML syntax, same pod lifecycle, same networking model. The
only differences are how the control plane is managed and which
ports are used, both of which I understand fully.

---

### Question 2
**How is MicroK8s different from production Kubernetes?**

1. Control plane runs as snap services not static pods
2. API server uses port 16443 instead of 6443
3. etcd uses port 12379 instead of 2379
4. Certs are at /var/snap/microk8s/current/certs/
   not /etc/kubernetes/pki/
5. Config files are at /var/snap/microk8s/current/args/
   not /etc/kubernetes/manifests/
6. Cert renewal uses microk8s refresh-certs
   not kubeadm certs renew all
7. Troubleshooting uses journalctl for control plane
   not kubectl logs on pods

Everything else is identical to production.

---

### Question 3
**MicroK8s kubectl stops working after WiFi change. Why and fix?**

When MicroK8s installs, it generates a TLS certificate for the
API server including the laptop's current IP as a Subject
Alternative Name. When WiFi changes, laptop gets new IP. kubectl
tries connecting to new IP but certificate only lists old IP.
TLS verification fails with certificate valid for old-ip not new-ip.

Quick fix:
```bash
microk8s refresh-certs --cert server.crt
microk8s config > ~/.kube/config
kubectl get nodes
```

Permanent fix — add entire home network to certificate:
```bash
sudo vi /var/snap/microk8s/current/args/kube-apiserver
# Add: --tls-san=127.0.0.1,localhost,192.168.1.0/24
sudo snap restart microk8s.daemon-apiserver
microk8s refresh-certs --cert server.crt
microk8s config > ~/.kube/config
```

---

### Question 4
**How do you check control plane health in MicroK8s?**

```bash
# Overall status
microk8s status

# All services running
snap services microk8s

# Check pods (add-ons)
kubectl get pods -n kube-system

# Check nodes
kubectl get nodes

# Check component logs
journalctl -u snap.microk8s.daemon-apiserver --lines=20
journalctl -u snap.microk8s.daemon-etcd --lines=20

# Full inspection
microk8s inspect
```

---

### Question 5
**Scenario: MicroK8s was working. You restarted laptop.
Now kubectl hangs. Walk through your investigation.**

1. Check if MicroK8s started after reboot
   → microk8s status
2. If not started → microk8s start
3. If started but hanging → check API server service
   → snap services microk8s | grep apiserver
4. Check API server logs for errors
   → journalctl -u snap.microk8s.daemon-apiserver --lines=30
5. Check if IP changed
   → hostname -I vs cat ~/.kube/config | grep server
6. If IP changed → microk8s config > ~/.kube/config
7. If cert error in logs → microk8s refresh-certs
8. Last resort → microk8s stop && microk8s start

---

*Next file: 07_Certificates_Complete_Guide.md*
````

---

# `07_Certificates_Complete_Guide.md`

````markdown
# Kubernetes Certificates — Complete Guide

> Level: 4 Years Experience
> Author: Muskan Patel
> Domain: Banking — Atos (US and European Clients)

---

## Why Certificates Exist — Simple Explanation

1. Without certificates anyone who reaches port 6443 on your
   control plane can run any kubectl command
2. Delete all pods, read all secrets, destroy production —
   all possible without certificates
3. Certificates solve this by doing two things:
   - Encryption — all traffic between components is TLS encrypted
   - Authentication — every component proves identity before
     being trusted
4. Every component has its own certificate signed by the
   Cluster Certificate Authority
5. If certificate is not signed by trusted CA — connection rejected

---

## The Certificate Authority

### Simple Explanation

Think of CA like a government that issues ID cards.
Every certificate in Kubernetes is signed by the cluster CA.
If your certificate is signed by the trusted CA — you are trusted.
If not — you are rejected.

```bash
# The CA files
ls /etc/kubernetes/pki/
ca.crt    # CA certificate — public, everyone has this
ca.key    # CA private key — NEVER share this

# MicroK8s CA location
ls /var/snap/microk8s/current/certs/
ca.crt
ca.key
```

---

## Types of Certificates

### Type 1 — Server Certificates

Used to prove identity of a server to clients.
When kubectl connects to API server, API server presents
its server certificate. kubectl verifies it was signed by CA.

````
Components with server certificates:
- API Server        → apiserver.crt
- etcd              → etcd/server.crt
- kubelet           → on each node
````

### Type 2 — Client Certificates

Used by clients to prove their identity to servers.
When API server connects to etcd, it presents its
client certificate. etcd verifies it.

````
Components with client certificates:
- kubectl           → in your kubeconfig file
- API Server → etcd → apiserver-etcd-client.crt
- API Server → kubelet → apiserver-kubelet-client.crt
- Controller Manager → controller-manager.conf
- Scheduler → scheduler.conf
- kubelet → API Server → kubelet-client.crt
````

### Type 3 — CA Certificate

Root of trust. Signs all other certificates.
Everyone trusts the CA.

````
CAs in Kubernetes:
- Cluster CA (ca.crt + ca.key) — signs most certificates
- etcd CA (etcd/ca.crt) — signs etcd certificates separately
- Front Proxy CA — for extension API servers
````

---

## All Certificate Files and Their Purpose

````
/etc/kubernetes/pki/
│
├── ca.crt + ca.key
│   Purpose: Root CA — signs all cluster certificates
│   Used by: API server to verify all client certificates
│
├── apiserver.crt + apiserver.key
│   Purpose: API server identity
│   Used when: kubectl connects to API server
│              kubectl verifies this certificate
│
├── apiserver-kubelet-client.crt
│   Purpose: API server proves identity to kubelet
│   Used when: API server fetches pod logs
│              API server exec into pods
│
├── apiserver-etcd-client.crt
│   Purpose: API server proves identity to etcd
│   Used when: Every read and write to etcd
│
├── front-proxy-ca.crt
│   Purpose: For extension API servers
│   Used when: Custom API servers added to cluster
│
├── sa.pub + sa.key
│   Purpose: ServiceAccount token management
│   sa.key: Controller manager signs SA tokens
│   sa.pub: API server verifies SA tokens
│
└── etcd/
    ├── ca.crt + ca.key  → etcd own CA
    ├── server.crt       → etcd server identity
    ├── peer.crt         → etcd nodes talk to each other
    └── healthcheck-client.crt → health check requests
````

---

## How Certificates Are Created

### Automatic — kubeadm (What Industry Uses)

```bash
# kubeadm init automatically:
# 1. Creates CA (ca.crt + ca.key)
# 2. Creates all server and client certificates
# 3. Signs them with CA
# 4. Places them in /etc/kubernetes/pki/
# 5. Creates kubeconfig files

sudo kubeadm init --pod-network-cidr=192.168.0.0/16

# After this all certs exist — no manual work needed
```

### Manual — Understanding the Process

```bash
# Step 1 — Create CA private key
openssl genrsa -out ca.key 2048

# Step 2 — Create CA certificate (self-signed, 10 years)
openssl req -x509 -new -nodes \
  -key ca.key \
  -subj "/CN=Kubernetes-CA" \
  -days 3650 \
  -out ca.crt

# Step 3 — Create API server private key
openssl genrsa -out apiserver.key 2048

# Step 4 — Create Certificate Signing Request
openssl req -new \
  -key apiserver.key \
  -subj "/CN=kube-apiserver" \
  -out apiserver.csr

# Step 5 — Sign with CA (1 year validity)
openssl x509 -req \
  -in apiserver.csr \
  -CA ca.crt \
  -CAkey ca.key \
  -CAcreateserial \
  -out apiserver.crt \
  -days 365
```

---

## Check Certificate Expiry

### Standard Kubernetes

```bash
# Best way — check all at once
kubeadm certs check-expiration

# Output:
# CERTIFICATE          EXPIRES              RESIDUAL TIME
# apiserver            May 16, 2027         354d
# etcd-server          May 16, 2027         354d
# controller-manager   May 16, 2027         354d

# Check specific certificate
openssl x509 \
  -in /etc/kubernetes/pki/apiserver.crt \
  -noout -dates

# Output:
# notBefore=May 16 20:27:36 2026 GMT
# notAfter=May 16 20:27:36 2027 GMT
```

### MicroK8s

```bash
# Check API server cert
openssl x509 \
  -in /var/snap/microk8s/current/certs/server.crt \
  -noout -dates

# Check CA cert
openssl x509 \
  -in /var/snap/microk8s/current/certs/ca.crt \
  -noout -dates

# Check all certs at once
for cert in /var/snap/microk8s/current/certs/*.crt; do
  echo "=== $cert ==="
  openssl x509 -in $cert -noout -dates 2>/dev/null
done
```

---

## Renew Certificates

### Standard Kubernetes

```bash
# Renew ALL certificates
kubeadm certs renew all

# Renew specific certificate
kubeadm certs renew apiserver
kubeadm certs renew etcd-server
kubeadm certs renew controller-manager.conf

# Update kubeconfig after renewal
cp /etc/kubernetes/admin.conf ~/.kube/config

# Restart control plane pods to pick up new certs
kubectl delete pod -n kube-system kube-apiserver-controlplane
kubectl delete pod -n kube-system kube-controller-manager-controlplane
kubectl delete pod -n kube-system kube-scheduler-controlplane
# kubelet recreates them immediately

# Verify
kubectl get nodes
kubeadm certs check-expiration
```

### MicroK8s

```bash
# Refresh specific cert
microk8s refresh-certs --cert server.crt
microk8s refresh-certs --cert ca.crt

# Refresh all
microk8s refresh-certs

# Update kubeconfig
microk8s config > ~/.kube/config

# Restart MicroK8s
microk8s stop && microk8s start
microk8s status --wait-ready

# Verify
kubectl get nodes
```

---

## Certificate Errors and Fixes

### Error 1 — Certificate Has Expired

````
x509: certificate has expired or is not yet valid
````

**What happened:**
Certificate passed its notAfter date.
API server is rejecting ALL connections.
No kubectl commands work.

**Fix — Standard Kubernetes:**
```bash
# Must SSH directly to control plane node first
# kubectl will not work

ssh user@control-plane-ip

# Renew all certificates
kubeadm certs renew all

# Update kubeconfig
cp /etc/kubernetes/admin.conf ~/.kube/config

# Restart control plane components
kubectl delete pod -n kube-system kube-apiserver-controlplane
kubectl delete pod -n kube-system kube-controller-manager-controlplane
kubectl delete pod -n kube-system kube-scheduler-controlplane

# Verify
kubectl get nodes
```

**Fix — MicroK8s:**
```bash
microk8s refresh-certs
microk8s config > ~/.kube/config
microk8s stop && microk8s start
kubectl get nodes
```

---

### Error 2 — Certificate Signed by Unknown Authority

````
x509: certificate signed by unknown authority
````

**What happened:**
Your kubeconfig has wrong or missing CA certificate.
kubectl cannot verify the API server certificate.

**Fix:**
```bash
# Standard Kubernetes
cp /etc/kubernetes/admin.conf ~/.kube/config

# MicroK8s
microk8s config > ~/.kube/config

# Verify correct CA in kubeconfig
cat ~/.kube/config | grep certificate-authority-data
# Should not be empty
```

---

### Error 3 — Certificate Valid for Wrong IP

````
x509: certificate is valid for 192.168.1.100, not 192.168.1.200
````

**What happened:**
Certificate was issued for old IP address.
Most common on laptops after WiFi network change.

**Quick Fix:**
```bash
# MicroK8s
microk8s refresh-certs --cert server.crt
microk8s config > ~/.kube/config
kubectl get nodes
```

**Permanent Fix — Never Break on WiFi Change:**
```bash
# Add entire home network to certificate SANs
sudo vi /var/snap/microk8s/current/args/kube-apiserver

# Add this line:
--tls-san=127.0.0.1,localhost,192.168.1.0/24

# Save and restart
sudo snap restart microk8s.daemon-apiserver
microk8s refresh-certs --cert server.crt
microk8s config > ~/.kube/config
kubectl get nodes

# Verify SANs in new certificate
openssl x509 \
  -in /var/snap/microk8s/current/certs/server.crt \
  -noout -text | grep -A 5 "Subject Alternative Name"
```

---

### Error 4 — Certificate Not Yet Valid

````
x509: certificate is not yet valid
````

**What happened:**
System clock is wrong.
Certificate start date is in the future from server's perspective.

**Fix:**
```bash
# Check current time
date
timedatectl status

# Fix clock
sudo timedatectl set-ntp true
sudo systemctl restart systemd-timesyncd

# Verify time is correct
date

# Retry kubectl
kubectl get nodes
```

---

### Error 5 — Connection Refused on Port 16443

````
dial tcp 192.168.1.100:16443: connect: connection refused
````

**What happened:**
MicroK8s API server is not running.
Or IP in kubeconfig points to old IP.

**Fix:**
```bash
# Check API server service
snap services microk8s | grep apiserver

# If inactive — start it
microk8s start
microk8s status --wait-ready

# If running but wrong IP in kubeconfig
hostname -I
microk8s config > ~/.kube/config
kubectl get nodes
```

---

## How Industry Manages Certificates

### Level 1 — Small Teams

````
- Manual renewal once a year
- Calendar reminder set 30 days before expiry
- Run kubeadm certs renew all during maintenance window
- Someone updates kubeconfig for all team members
````

### Level 2 — Medium Companies (Banking Level)

```bash
# Daily monitoring CronJob
# Alerts Slack 30 days before expiry

0 9 * * * kubeadm certs check-expiration 2>&1 | \
  awk '{if ($NF+0 < 30) print "URGENT CERT EXPIRY: "$0}' | \
  curl -X POST -H 'Content-type: application/json' \
  --data '{"text":"'"$(cat)"'"}' \
  https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK
```

### Level 3 — Large Enterprise

```bash
# cert-manager — automatic certificate management
# Kubernetes operator that handles everything

kubectl apply -f \
  https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml

# Certificate object — auto-renews 15 days before expiry
cat << EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: payment-service-tls
  namespace: banking
spec:
  duration: 2160h       # 90 days validity
  renewBefore: 360h     # renew 15 days before expiry
  secretName: payment-tls-secret
  dnsNames:
  - payment-service
  - payment-service.banking.svc.cluster.local
  issuerRef:
    name: ca-issuer
    kind: ClusterIssuer
EOF
```

---

## Interview Questions — Certificates

### Question 1
**Why do Kubernetes components need certificates?**

1. Security — without certificates anyone reaching port 6443
   can control the entire cluster
2. Encryption — TLS ensures all inter-component traffic is
   encrypted, no data can be sniffed on network
3. Authentication — each component proves identity before
   being trusted, this is mutual TLS
4. In banking environments this is non-negotiable — regulatory
   compliance requires all internal traffic to be encrypted
5. The cluster CA is the root of trust — only certificates
   signed by the cluster CA are accepted

---

### Question 2
**Kubernetes cluster is completely inaccessible Monday morning.
No kubectl commands work. What is your first check?**

1. Try connecting to API server directly
   → curl -k https://control-plane-ip:6443/healthz
2. SSH directly to control plane node
   → kubectl will not work if API server certs expired
3. Check certificate expiry
   → kubeadm certs check-expiration
4. If expired — renew immediately
   → kubeadm certs renew all
5. Update kubeconfig
   → cp /etc/kubernetes/admin.conf ~/.kube/config
6. Restart control plane components
7. Verify cluster is back
   → kubectl get nodes

In my banking environment at Atos this exact scenario happened
in a UAT cluster. Certificate had expired over a weekend.
Identified with kubeadm certs check-expiration, renewed with
kubeadm certs renew all, updated kubeconfigs for the team,
and cluster was back in under 15 minutes.
After that we set up daily monitoring alerts at 30 days before expiry.

---

### Question 3
**What is the difference between CA certificate and
component certificates?**

1. CA certificate is the root of trust — it signs all
   other certificates and is trusted by everyone
2. CA certificate typically lasts 10 years
3. Component certificates (API server, etcd, kubelet) last
   1 year by default in kubeadm clusters
4. CA private key (ca.key) must be protected above everything
   else — if compromised, entire cluster security is broken
5. Component certificates can be renewed with
   kubeadm certs renew without touching the CA
6. Rotating the CA itself is much more complex and
   requires updating trust on every component

---

### Question 4
**What is mutual TLS and how does Kubernetes use it?**

1. Normal TLS — only server proves its identity to client
   like a website presenting SSL certificate
2. Mutual TLS — BOTH sides prove identity to each other
3. Kubernetes uses mutual TLS everywhere
4. Example — when kubelet connects to API server:
   - API server presents apiserver.crt to kubelet
   - kubelet presents its client certificate to API server
   - Both verify each other's certificates against the cluster CA
   - Only if both are valid does communication proceed
5. This prevents a compromised component from
   impersonating another component

---

### Question 5
**How do you monitor certificate expiry in production?**

1. Manual check — kubeadm certs check-expiration
   Run this and look at RESIDUAL TIME column
   Under 30 days means act immediately

2. Automated monitoring — daily CronJob or script
   that checks expiry and alerts Slack or PagerDuty
   when any certificate is within 30 days of expiry

3. Enterprise solution — cert-manager operator
   Automatically renews certificates before expiry
   Configurable renewal threshold (15 days before expiry)
   Works with Let's Encrypt, Vault, or internal CA
   Zero manual intervention required

In our banking environment we used level 2 — daily monitoring
script alerting our DevOps Slack channel.
Any certificate within 30 days triggered a P2 incident ticket.

---

### Question 6
**You changed WiFi and now kubectl gives x509 error.
Explain what happened technically and how you fix it.**

**What happened technically:**

1. When MicroK8s or kubeadm installed, it generated the API
   server TLS certificate
2. The certificate included the laptop or server IP address
   as a Subject Alternative Name (SAN)
3. When WiFi changed, the machine received a new IP address
4. kubectl tried connecting to the API server using the new IP
5. TLS handshake started — API server presented its certificate
6. kubectl checked — is the IP I am connecting to listed in
   the certificate SANs?
7. New IP was not in the certificate — TLS verification failed
8. kubectl rejected the connection with x509 error

**Fix for MicroK8s:**
```bash
microk8s refresh-certs --cert server.crt
microk8s config > ~/.kube/config
kubectl get nodes
```

**Permanent fix:**
```bash
# Add entire subnet as valid SAN
echo "--tls-san=127.0.0.1,localhost,192.168.1.0/24" | \
  sudo tee -a /var/snap/microk8s/current/args/kube-apiserver
sudo snap restart microk8s.daemon-apiserver
microk8s refresh-certs --cert server.crt
microk8s config > ~/.kube/config
```

---

### Question 7
**What happens to the cluster if the CA certificate expires?**

1. CA certificates typically last 10 years so this is rare
2. If CA expires — all component certificates become unverifiable
3. Every TLS handshake in the cluster fails
4. Complete cluster communication breakdown
5. API server, etcd, kubelet — all stop trusting each other
6. This is a catastrophic failure requiring full CA rotation
7. CA rotation requires updating trust on every component
   and every kubeconfig in the cluster
8. This is why CA private key (ca.key) must be backed up
   securely — it is needed for rotation
9. Prevention — monitor CA expiry separately from component certs
   CA is easy to miss because it rarely needs attention

---

### Question 8
**What is a ServiceAccount token and how is it different
from a user certificate?**

1. User certificates are in kubeconfig files
   Used by humans running kubectl commands
   Signed by cluster CA

2. ServiceAccount tokens are JWT tokens
   Used by pods to call the Kubernetes API
   Signed by the sa.key (service account signing key)
   Mounted automatically into pods at
   /var/run/secrets/kubernetes.io/serviceaccount/token

3. Since Kubernetes 1.24 — tokens are time-limited
   They expire after 1 hour by default
   kubelet automatically rotates them

4. IRSA on EKS — ServiceAccount tokens are exchanged
   for AWS IAM credentials
   Allows pods to call AWS APIs securely
   Without storing any AWS credentials in the cluster

In our banking project every microservice had its own
ServiceAccount with minimal RBAC permissions.
No service used the default ServiceAccount.
This prevented any compromised pod from having
cluster-wide permissions.

---

### Question 9
**Scenario: cert-manager is installed. A certificate
Secret suddenly disappears. What happens and how do you fix it?**

1. If cert-manager Certificate Secret is deleted —
   pods using it will fail on next restart or mount
2. cert-manager detects the Secret is missing
3. cert-manager automatically re-requests the certificate
   from the configured issuer
4. New certificate is issued and stored in the Secret
5. This typically resolves within 1-2 minutes automatically

Fix if cert-manager does not auto-recover:
```bash
# Check certificate object status
kubectl describe certificate <cert-name> -n banking

# Check cert-manager logs
kubectl logs -n cert-manager \
  deployment/cert-manager | tail -30

# Force renewal by deleting the certificate request
kubectl delete certificaterequest -n banking \
  <cert-request-name>
# cert-manager creates a new request automatically

# Check secret was recreated
kubectl get secret <secret-name> -n banking
```

---

### Question 10
**How do you rotate the cluster CA certificate?**

This is an advanced question for senior engineers.

1. CA rotation is complex and risky — plan carefully
2. New CA must be distributed to all components before
   old CA is removed — dual trust period
3. High level process:

```bash
# Step 1 — generate new CA
# Step 2 — add new CA alongside old CA in all components
#           (both CAs trusted simultaneously)
# Step 3 — reissue all component certificates with new CA
# Step 4 — restart all components to use new certs
# Step 5 — remove old CA from trusted CAs
# Step 6 — verify all components working

# In practice — use kubeadm for this
# kubeadm alpha certs certificate-renewal
# Or use managed Kubernetes (EKS handles this automatically)
```

In production banking — we avoided manual CA rotation by using
EKS where AWS manages all control plane certificates including
CA rotation. For self-managed clusters we would schedule a
full maintenance window and follow the kubeadm CA rotation runbook.

---

## Summary — Key Facts for Interviews

1. Certificates expire after 1 year by default in kubeadm
2. CA certificate lasts 10 years
3. kubeadm certs check-expiration — check all expiry dates
4. kubeadm certs renew all — renew everything at once
5. After renewal always update kubeconfig
6. MicroK8s uses microk8s refresh-certs
7. WiFi change breaks MicroK8s because cert has old IP as SAN
8. Permanent fix is adding subnet range as SAN in API server args
9. In production cert-manager handles rotation automatically
10. CA private key must be backed up — losing it is catastrophic
11. Mutual TLS means both sides prove identity — not just server
12. ServiceAccount tokens are different from user certificates
    They are JWT tokens signed by sa.key not by cluster CA

---

*Next file: 08_Pods_Deployments_ReplicaSets.md*
````
