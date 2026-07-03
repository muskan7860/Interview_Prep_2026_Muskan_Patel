# Jenkins Day 4 — Real Kubernetes Agent Lab
## Run Your Pipeline on Your Actual k8s-agent Pod

> This is a REAL lab — not simulated.
> Your pipeline will run inside the `jenkins-agent` Kubernetes pod.
> This is exactly how real companies run Jenkins builds.

---

## Before You Start — Confirm Agent is Connected

Open browser → go to:
```
https://devopsbymuskan07.com/jenkins/manage/computer/
```

You should see `k8s-agent` with a **green circle**.

If it shows red/offline — run this first:
```bash
kubectl delete pod jenkins-agent -n monitoring --force
kubectl apply -f ~/jenkins-agent-WORKING.yaml
sleep 20 && kubectl logs jenkins-agent -n monitoring | tail -5
```
Wait until you see `INFO: Connected` in logs. Then continue.

---

## What We Are Building in This Lab

We will run a pipeline that:
1. Runs on your REAL Kubernetes agent pod
2. Prints proof it is running inside Kubernetes
3. Checks what tools are available on the agent
4. Creates a file inside the agent workspace
5. Shows the difference between master and agent

---

## PART A — First Pipeline on k8s-agent

### Step A1 — Open Terminal and Go to Your Repo

```bash
cd ~/Devops-lab/Practice-Lab/jenkins
```

Confirm you are on main branch:
```bash
git branch
```
You should see `* main`

---

### Step A2 — Open Your Jenkinsfile

```bash
nano Jenkinsfile
```

---

### Step A3 — Delete Everything and Paste This

Press `Ctrl + K` repeatedly until file is empty. Then paste:

```groovy
pipeline {
    agent { label 'k8s-agent' }

    stages {

        stage('Prove We Are on k8s-agent') {
            steps {
                echo "============================================"
                echo "Agent Name: ${env.NODE_NAME}"
                echo "Workspace:  ${env.WORKSPACE}"
                echo "Build No:   ${env.BUILD_NUMBER}"
                echo "============================================"
                sh 'whoami'
                sh 'hostname'
                sh 'echo My IP is: $(hostname -i)'
            }
        }

        stage('Check Kubernetes Identity') {
            steps {
                echo "Checking if we are inside a Kubernetes pod..."
                sh 'cat /etc/hostname'
                sh 'echo Namespace: $(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace)'
                sh 'ls /var/run/secrets/kubernetes.io/serviceaccount/'
            }
        }

        stage('Check Available Tools') {
            steps {
                echo "What tools does this agent have?"
                sh 'java -version'
                sh 'git --version'
                sh 'curl --version | head -1'
                sh 'which wget || echo wget not installed'
                sh 'which docker || echo docker not installed'
                sh 'which kubectl || echo kubectl not installed'
            }
        }

        stage('Create File in Workspace') {
            steps {
                echo "Creating a file in the agent workspace..."
                sh '''
                    echo "This file was created by Jenkins build ${BUILD_NUMBER}" > myfile.txt
                    echo "Agent: ${NODE_NAME}" >> myfile.txt
                    echo "Date: $(date)" >> myfile.txt
                    cat myfile.txt
                '''
                echo "File created successfully inside the k8s pod workspace!"
            }
        }

        stage('Show Workspace Contents') {
            steps {
                sh 'ls -la'
                sh 'pwd'
                sh 'df -h | head -5'
            }
        }

    }

    post {
        success {
            echo "Pipeline ran successfully on agent: ${env.NODE_NAME}"
        }
        failure {
            echo "Pipeline failed on agent: ${env.NODE_NAME}"
        }
        always {
            echo "Cleaning up workspace..."
            cleanWs()
        }
    }
}
```

---

### Step A4 — Save the File

Press `Ctrl + O` → Press `Enter` → Press `Ctrl + X`

---

### Step A5 — Verify File Ends Correctly

```bash
tail -10 Jenkinsfile
```

You should see the closing `}` braces at the bottom. The file ends with:
```groovy
    }

}
```

---

### Step A6 — Push to GitHub

```bash
git add Jenkinsfile
git commit -m "Day 4 lab: run pipeline on real k8s-agent"
git push origin main
```

You should see `main -> main` at the end.

---

### Step A7 — Go to Jenkins in Browser

Open: `https://devopsbymuskan07.com/jenkins/`

---

### Step A8 — Run the Build

1. Click **`Day1_Declarative_Pipeline`**
2. Click **Build Now**
3. Click the new build number
4. Click **Console Output**

---

### Step A9 — What You Should See

Look for these specific things in the console output:

**Proof it ran on k8s-agent:**
```
Agent Name: k8s-agent
Workspace:  /home/jenkins/agent/workspace/Day1_Declarative_Pipeline
Build No:   5
```

**Proof it is inside a Kubernetes pod:**
```
jenkins-agent        ← hostname = your pod name
Namespace: monitoring
```

**Tools available:**
```
openjdk version "17..."    ← Java is installed
git version 2.x.x          ← Git is installed
curl 7.x.x                 ← curl is installed
docker not installed        ← normal, no docker in this agent
kubectl not installed       ← normal, this is a simple agent
```

**File created in workspace:**
```
This file was created by Jenkins build 5
Agent: k8s-agent
Date: Thu Jul  3 07:xx:xx UTC 2026
```

**Most important line at the top:**
```
Running on k8s-agent in /home/jenkins/agent/workspace/Day1_Declarative_Pipeline
```

This proves your pipeline ran inside the real Kubernetes pod — not on the Jenkins master.

---

## PART B — Verify From Kubernetes Side

While the build is running (or after), run this on your Ubuntu terminal to see the agent pod is actually working:

### Step B1 — Watch the Pod While Build Runs

```bash
kubectl get pod jenkins-agent -n monitoring -w
```

You will see the pod status. Press `Ctrl+C` to stop watching.

### Step B2 — Check Pod Resource Usage During Build

```bash
kubectl top pod jenkins-agent -n monitoring
```

This shows CPU and Memory the agent is using while your pipeline runs.

### Step B3 — See the Workspace Inside the Pod

After the build finishes, run:

```bash
kubectl exec -n monitoring jenkins-agent -- ls /home/jenkins/agent/workspace/ 2>/dev/null || echo "Workspace was cleaned by cleanWs()"
```

You will see either the workspace folder OR the message that cleanWs() deleted it — which proves our `post { always { cleanWs() } }` block worked.

---

## PART C — Run a Multi-Stage Pipeline With agent none

This shows the REAL power — different stages on different agents.

### Step C1 — Update Jenkinsfile

```bash
nano Jenkinsfile
```

Delete everything (`Ctrl+K` repeatedly) and paste this:

```groovy
pipeline {
    agent none

    stages {

        stage('Stage on Master') {
            agent { label 'built-in' }
            steps {
                echo "This stage runs on: ${env.NODE_NAME}"
                sh 'hostname'
                echo "Master workspace: ${env.WORKSPACE}"
            }
        }

        stage('Stage on k8s-agent') {
            agent { label 'k8s-agent' }
            steps {
                echo "This stage runs on: ${env.NODE_NAME}"
                sh 'hostname'
                echo "Agent workspace: ${env.WORKSPACE}"
                sh 'echo I am running inside a Kubernetes Pod!'
            }
        }

        stage('Back to k8s-agent') {
            agent { label 'k8s-agent' }
            steps {
                echo "Still on: ${env.NODE_NAME}"
                sh 'echo Same agent, different stage'
            }
        }

    }

    post {
        always {
            echo "Pipeline finished. Stages ran on different agents!"
        }
    }
}
```

Save: `Ctrl+O` → `Enter` → `Ctrl+X`

---

### Step C2 — Push to GitHub

```bash
git add Jenkinsfile
git commit -m "Day 4 lab: agent none with multi-agent stages"
git push origin main
```

---

### Step C3 — Run in Jenkins

Go to Jenkins → `Day1_Declarative_Pipeline` → **Build Now** → Console Output

You will see:

```
Stage on Master:
  Running on Jenkins in /var/jenkins_home/...
  This stage runs on: Jenkins
  
Stage on k8s-agent:
  Running on k8s-agent in /home/jenkins/agent/workspace/...
  This stage runs on: k8s-agent
  I am running inside a Kubernetes Pod!

Back to k8s-agent:
  Running on k8s-agent in /home/jenkins/agent/workspace/...
  Still on: k8s-agent
```

**This proves:** Different stages ran on different machines — the master and your Kubernetes pod — in the SAME pipeline. This is exactly what real companies do.

---

## ✅ Lab Checklist

- [ ] k8s-agent showed green circle in Jenkins UI before starting
- [ ] Part A — Pipeline ran and showed `Running on k8s-agent` in console
- [ ] I saw `Agent Name: k8s-agent` in the output
- [ ] I saw `hostname` showing `jenkins-agent` (the pod name)
- [ ] I saw `Namespace: monitoring` proving we are inside Kubernetes
- [ ] I saw the file created inside the pod workspace
- [ ] Part B — I ran `kubectl exec` and saw workspace (or cleanWs confirmation)
- [ ] Part C — I saw different stages running on different agents in same pipeline

---

## Troubleshooting This Lab

### Problem: "There are no nodes with the label k8s-agent"
**Fix:**
Go to Jenkins → Manage Jenkins → Nodes → k8s-agent → Configure → check Labels field shows `k8s-agent linux`. If empty, add `k8s-agent` and Save.

Then check agent is online:
```bash
kubectl logs jenkins-agent -n monitoring | tail -5
```
Should show `INFO: Connected`

### Problem: Build stays in queue with "Waiting for next available executor"
**Cause:** Agent is offline.
**Fix:**
```bash
kubectl delete pod jenkins-agent -n monitoring --force
kubectl apply -f ~/jenkins-agent-WORKING.yaml
sleep 20 && kubectl logs jenkins-agent -n monitoring | tail -5
```
Wait for `INFO: Connected` then click **Build Now** again.

### Problem: "cleanWs() step not found"
**Fix:** Go to Manage Jenkins → Plugins → Available → search `Workspace Cleanup` → install → restart Jenkins.

### Problem: Stage C1 fails on `agent { label 'built-in' }`
**Fix:** The built-in node label might be different. Go to Manage Jenkins → Nodes → Built-In Node → Configure → check what label is set. Change `built-in` in the Jenkinsfile to match.

---

## What This Lab Proves for Interviews

When interviewer asks: **"Have you worked with Jenkins agents?"**

You can say:

*"Yes — I have a Jenkins setup running on Kubernetes (MicroK8s) where Jenkins master runs as a Deployment and the agent runs as a separate pod using the `jenkins/inbound-agent` image. The agent connects to the master using WebSocket protocol through the internal Kubernetes service. I have run real pipelines using `agent { label 'k8s-agent' }` and verified from the console output that builds execute inside the Kubernetes pod — not on the master. I also demonstrated multi-agent pipelines where different stages run on different nodes using `agent none` at the pipeline level with per-stage agent definitions."*

That answer proves hands-on experience — not just reading.

---

*Lab completed: July 3, 2026*
