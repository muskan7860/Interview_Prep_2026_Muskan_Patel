# Jenkins Day 4 — Real Kubernetes Agent Setup
## Complete Reference Guide + Troubleshooting Log

> This file documents exactly how your real Jenkins + Kubernetes agent setup works.
> Use this as a reference whenever the agent disconnects or needs to be recreated.

---

## Your Complete Setup Architecture

```
Your ThinkPad (Ubuntu)
│
└── Kubernetes (MicroK8s) — namespace: monitoring
    │
    ├── Jenkins Master (Deployment)
    │   └── Pod: jenkins-8484d4fff7-xxxxx
    │   └── Image: jenkins/jenkins
    │   └── Service: jenkins (ClusterIP)
    │       ├── Port 8080  → Jenkins UI
    │       └── Port 50000 → Agent TCP connection
    │   └── Context path: /jenkins/
    │
    ├── Jenkins Agent (Pod)
    │   └── Pod name: jenkins-agent
    │   └── Image: jenkins/inbound-agent
    │   └── Connection: WebSocket via http://jenkins:8080/jenkins/
    │   └── Agent name in Jenkins: k8s-agent
    │
    └── Cloudflare Tunnel (cloudflared pod)
        └── Exposes Jenkins to internet
        └── Public URL: https://devopsbymuskan07.com/jenkins/
```

---

## How Your Setup Was Originally Created

You created three Kubernetes resources manually using `kubectl apply`:

### 1. Jenkins Master — Deployment
Jenkins runs as a Kubernetes Deployment in the `monitoring` namespace. It has a Kubernetes Service named `jenkins` which exposes ports 8080 and 50000 internally within the cluster.

### 2. Jenkins Agent — Pod
You created a Pod manually using a YAML file. The pod runs the `jenkins/inbound-agent` image which automatically connects to the Jenkins master using JNLP/WebSocket protocol.

### 3. Cloudflare Tunnel
You set up `cloudflared` as a Kubernetes pod that creates a secure tunnel from your ThinkPad to Cloudflare's edge network, making Jenkins accessible publicly at `https://devopsbymuskan07.com/jenkins/` without exposing any ports on your home router.

---

## The Working Agent YAML File

Save this file at `~/jenkins-agent-WORKING.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: jenkins-agent
  namespace: monitoring
spec:
  restartPolicy: Always
  containers:
  - name: jnlp
    image: jenkins/inbound-agent
    args:
    - -url
    - http://jenkins:8080/jenkins/
    - -secret
    - c2d39d33cbeb5da622df7d9a864bca6bdda8227e704625e3b87226da2a0b23d5
    - -name
    - k8s-agent
    - -webSocket
    - -workDir
    - /home/jenkins/agent
```

### Why Each Line Matters

**`image: jenkins/inbound-agent`**
This is the official Jenkins agent image. It contains Java and the Jenkins remoting JAR. It knows how to connect back to Jenkins master automatically.

**`- -url`** followed by **`- http://jenkins:8080/jenkins/`**
This is the Jenkins master URL as seen from INSIDE the Kubernetes cluster. `jenkins` is the Kubernetes Service name — pods in the same namespace can reach each other by service name. The `/jenkins/` at the end is the context path — without it you get 404.

**`- -secret`**
This is the authentication token Jenkins generates for this specific agent. It proves to Jenkins that this pod is the legitimate `k8s-agent`.

**`- -name`** followed by **`- k8s-agent`**
The name of the agent as registered in Jenkins UI under Manage Jenkins → Nodes.

**`- -webSocket`**
Use WebSocket protocol instead of raw TCP port 50000. WebSocket goes through HTTP (port 8080) which works through the Kubernetes service. Raw TCP port 50000 did not work reliably in this setup.

**`- -workDir`** followed by **`- /home/jenkins/agent`**
The directory inside the container where Jenkins will store workspace files, logs, and cache.

---

## How to Use the Agent in Your Jenkinsfile

```groovy
pipeline {
    agent { label 'k8s-agent' }

    stages {
        stage('Check Agent') {
            steps {
                echo "Running on: ${env.NODE_NAME}"
                echo "Workspace: ${env.WORKSPACE}"
                sh 'whoami'
                sh 'hostname'
            }
        }
    }
}
```

When this runs, you will see in console output:
```
Running on k8s-agent in /home/jenkins/agent/workspace/...
jenkins        ← whoami output
jenkins-agent  ← hostname = the Kubernetes pod name
```

---

## Quick Commands Reference

### Check Agent Status
```bash
# Is the agent pod running?
kubectl get pod jenkins-agent -n monitoring

# What are the agent logs?
kubectl logs jenkins-agent -n monitoring | tail -20

# Watch logs live
kubectl logs jenkins-agent -n monitoring -f
```

### Restart Agent (When Disconnected)
```bash
kubectl delete pod jenkins-agent -n monitoring --force
kubectl apply -f ~/jenkins-agent-WORKING.yaml
sleep 20 && kubectl logs jenkins-agent -n monitoring | tail -10
```

### Check Jenkins Master Health
```bash
# Is Jenkins pod running?
kubectl get pod -n monitoring -l app=jenkins

# Jenkins master logs
kubectl logs -n monitoring $(kubectl get pod -n monitoring -l app=jenkins -o jsonpath='{.items[0].metadata.name}') | tail -20

# Test Jenkins is responding
kubectl exec -n monitoring jenkins-agent -- curl -s -o /dev/null -w "%{http_code}" http://jenkins:8080/jenkins/login
# Should return: 200
```

### Check Port 50000 is Active
```bash
kubectl exec -n monitoring jenkins-agent -- curl -s http://jenkins:50000
# Should return: Jenkins-Agent-Protocols: JNLP4-connect, Ping
```

### Get Jenkins Pod Name Dynamically
```bash
kubectl get pod -n monitoring -l app=jenkins -o jsonpath='{.items[0].metadata.name}'
```

---

## Complete Troubleshooting Log — What Went Wrong and How We Fixed It

### Problem 1 — Network is Unreachable
**Error:**
```
java.net.SocketException: Network is unreachable
Failed to connect to https://devopsbymuskan07.com/jenkins/tcpSlaveAgentListener/
```

**What happened:**
The agent pod was using `https://devopsbymuskan07.com/jenkins/` as the Jenkins URL. This sent traffic OUT through the internet, then back in through Cloudflare. Port 50000 is not exposed through Cloudflare tunnel, so the connection failed.

**Root cause:** Agent should talk to Jenkins INTERNALLY using the Kubernetes service name, not the public internet URL.

**Fix:** Change `JENKINS_URL` from `https://devopsbymuskan07.com/jenkins/` back to `http://jenkins:8080`

---

### Problem 2 — 404 on tcpSlaveAgentListener
**Error:**
```
java.io.IOException: http://jenkins:8080/tcpSlaveAgentListener/ is invalid: 404 Not Found
```

**What happened:**
Jenkins had port 50000 disabled. The agent was trying to connect using TCP protocol which requires Jenkins to have `/tcpSlaveAgentListener/` endpoint active.

**Diagnosis commands used:**
```bash
# Check if Jenkins config has port set
kubectl exec -n monitoring <jenkins-pod> -- cat /var/jenkins_home/config.xml | grep -A2 slaveAgent
# Showed: <slaveAgentPort>0</slaveAgentPort>  ← 0 means disabled

# Check if endpoint responds
kubectl exec -n monitoring jenkins-agent -- curl -s http://jenkins:8080/tcpSlaveAgentListener/
# Showed: 404
```

**Fix attempt 1:** Go to Jenkins UI → Manage Jenkins → Security → TCP port for inbound agents → set Fixed → 50000 → Save

**Result:** Config saved (`<slaveAgentPort>50000</slaveAgentPort>`) but Jenkins did not activate it until restarted.

**Fix attempt 2:** Run Groovy script in Jenkins Script Console (`/manage/script`):
```groovy
import jenkins.model.Jenkins
Jenkins.instance.setSlaveAgentPort(50000)
Jenkins.instance.save()
println "Agent port set to: " + Jenkins.instance.getSlaveAgentPort()
```
**Result:** `Agent port set to: 50000` — confirmed in console.

**Still 404?** Jenkins needed a restart to activate the listener:
```bash
kubectl rollout restart deployment/jenkins -n monitoring
kubectl rollout status deployment/jenkins -n monitoring
```

---

### Problem 3 — port 50000 Not Reachable From Outside
**Error:**
```
port:50000 is not reachable on host devopsbymuskan07.com
```

**What happened:**
Cloudflare tunnel only forwards HTTPS (port 443) to Jenkins port 8080. Port 50000 is not forwarded through Cloudflare. So using the public URL for agent connection does not work for TCP.

**Fix:** Use WebSocket protocol (`-webSocket` flag) instead of TCP. WebSocket goes through HTTP port 8080 which IS accessible internally. No need for port 50000.

---

### Problem 4 — 404 on /login (Jenkins Not Ready)
**Error:**
```
INFO: http://jenkins:8080/login is not ready: 404
```

**What happened:**
After Jenkins restarted, it was still starting up when the agent tried to connect. Also, Jenkins has a context path `/jenkins/` — connecting to `http://jenkins:8080/login` without the context path returns 404.

**How to check if Jenkins is fully started:**
```bash
kubectl exec -n monitoring <jenkins-pod> -- curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/jenkins/login
# Should return 200 when ready
```

**Check Jenkins logs for startup confirmation:**
```bash
kubectl logs -n monitoring <jenkins-pod> | grep "fully up and running"
# Should show: Jenkins is fully up and running
```

**Fix:** Use the correct URL with context path:
```
http://jenkins:8080/jenkins/
```
NOT:
```
http://jenkins:8080/
```

---

### Problem 5 — Wrong Secret
**Error:**
Agent connected to Jenkins but was rejected with authentication error.

**What happened:**
After Jenkins restarted, the secret for the agent changed. The pod was using the old secret.

**How to get current secret:**
Go to `https://devopsbymuskan07.com/jenkins/manage/computer/k8s-agent/` — the page shows the current secret in the connection command.

**Fix:** Update the YAML file with the new secret and recreate the pod.

---

### Problem 6 — YAML Parse Error
**Error:**
```
error parsing jenkins-agent-backup.yaml: yaml: line 8: mapping values are not allowed in this context
```

**What happened:**
When editing the YAML file with nano, accidentally typed `i` at the start of a line:
```yaml
i    cni.projectcalico.org/podIPs: 10.1.244.177/32
```

Also the backup file had `status:` section at the bottom which cannot be applied with `kubectl apply` — status is managed by Kubernetes, not user-defined.

**Fix:** Never edit the auto-generated backup YAML. Always create a clean minimal YAML from scratch using `cat > file.yaml << 'EOF'` pattern.

**Correct minimal agent YAML contains only:**
- `apiVersion`, `kind`, `metadata`
- `spec.containers` with image and args
- `spec.restartPolicy`
- Nothing else — no status, no resourceVersion, no uid

---

### Problem 7 — JENKINS_DIRECT_CONNECTION Needs instanceIdentity
**Error:**
```
option "-direct (-directConnection)" requires the option(s) [-instanceIdentity]
```

**What happened:**
Tried to use `JENKINS_DIRECT_CONNECTION=jenkins:50000` environment variable to skip the HTTP handshake. This requires an additional `instanceIdentity` parameter which is a base64-encoded key from Jenkins.

**Fix:** Do not use `-direct` flag. Use `-tunnel jenkins:50000` for TCP or `-webSocket` for WebSocket. WebSocket is simpler and worked in this setup.

---

## Final Working Solution Summary

After all troubleshooting, the solution that worked:

| Setting | Value | Why |
|---------|-------|-----|
| Jenkins URL | `http://jenkins:8080/jenkins/` | Internal K8s service + correct context path |
| Protocol | `-webSocket` | Works through HTTP, no need for port 50000 |
| Secret | From Jenkins UI → Nodes → k8s-agent | Generated by Jenkins, must match |
| Agent name | `k8s-agent` | Must match exactly what is registered in Jenkins UI |

---

## If Secret Changes (After Jenkins Restart)

Jenkins sometimes regenerates agent secrets. If agent stops connecting after a Jenkins restart:

### Step 1 — Get New Secret
Go to: `https://devopsbymuskan07.com/jenkins/manage/computer/k8s-agent/`

Copy the secret from the connection command shown on that page.

### Step 2 — Update YAML and Restart Agent
```bash
nano ~/jenkins-agent-WORKING.yaml
# Update the secret value

kubectl delete pod jenkins-agent -n monitoring --force
kubectl apply -f ~/jenkins-agent-WORKING.yaml
sleep 20 && kubectl logs jenkins-agent -n monitoring | tail -5
```

### Step 3 — Confirm Connected
```bash
kubectl logs jenkins-agent -n monitoring | grep "Connected"
# Should show: INFO: Connected
```

---

## Jenkins Manage Page Errors and What They Mean

### Error 1 — Plugin dependency errors
```
GitHub Branch Source Plugin — missing: jjwt-api
Matrix Project Plugin — missing: junit
```
**Fix:** Manage Jenkins → Plugins → Available → search `jjwt-api` → install. Search `junit` → install. Restart.

### Error 2 — Security vulnerabilities
```
Multiple security vulnerabilities in Jenkins 2.555.2
Script Security Plugin — Sandbox bypass vulnerability
Credentials Binding Plugin — Path traversal vulnerability
```
**Fix:** Manage Jenkins → Plugins → Updates → select all → Update → restart Jenkins.
**Why important:** In real companies, security vulnerabilities in Jenkins are critical issues that must be fixed immediately. Jenkins has access to credentials and runs code — a vulnerability here means attacker access to everything.

### Error 3 — Building on built-in node
```
Building on the built-in node can be a security issue
```
**Fix:** Manage Jenkins → Nodes → Built-In Node → Configure → Number of executors → set to `0` → Save.
**Why:** Master should never run builds. Set executors to 0 to force all builds to agents only.

---

## Interview Story — How to Explain This Setup

**"Tell me about your Jenkins setup"**

"I run Jenkins on Kubernetes — specifically MicroK8s on my local machine for practice. Jenkins master runs as a Kubernetes Deployment with a ClusterIP service exposing ports 8080 for the UI and 50000 for agent connections. I use Cloudflare Tunnel to expose it publicly without opening any router ports.

For the agent, I use the `jenkins/inbound-agent` image running as a separate Kubernetes pod in the same namespace. It connects back to the master using WebSocket protocol through the internal Kubernetes service name — so all agent traffic stays within the cluster, no external network dependency.

When setting this up, the key learning was that the agent URL must use the internal Kubernetes service name `http://jenkins:8080/jenkins/` with the correct context path, not the public Cloudflare URL. And WebSocket works better than raw TCP in a setup where the Kubernetes service does not expose port 50000 externally."

---

*This file documents Day 4 hands-on troubleshooting — July 3, 2026*
