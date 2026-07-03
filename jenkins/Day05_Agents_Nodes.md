# Jenkins Day 4 — Agents & Nodes
## Theory + Hands-On Lab

---

## Before We Start — What is an Agent?

### Simple Analogy First

Think of a restaurant kitchen.

The **Head Chef** (Jenkins Master) decides:
- Which dish to make
- Which cook to assign
- What order to cook things in

The **Cooks** (Jenkins Agents) do the actual cooking — chopping, frying, plating.

The Head Chef never cooks. The Head Chef only manages and assigns work.

That is exactly how Jenkins works:
- **Jenkins Master** = decides, schedules, shows UI, manages jobs
- **Jenkins Agent** = the machine that actually runs your pipeline commands

---

### Why Not Just Run Everything on the Master?

In Day 1, when you ran `whoami` it showed `jenkins` — that means your pipeline ran ON the Jenkins master itself. This works for learning, but in real companies it is a bad idea because:

1. If a build crashes the machine → Jenkins UI goes down → everyone is affected
2. Master handles many jobs at once → builds slow each other down
3. Security risk → build code runs on the same machine as Jenkins secrets

Real companies always run builds on **separate agent machines**.

---

## Topic 1 — Master-Agent Architecture

### How it Works

```
Jenkins Master (Controller)
        │
        │ assigns work via SSH or JNLP
        │
   ┌────┴────────────────┐
   │                     │
Agent 1 (Linux)     Agent 2 (Windows)
Runs Java builds    Runs .NET builds
```

### Three Ways an Agent Connects to Master

**Method 1 — SSH**
Master SSHes into the agent machine and runs commands. Most common for permanent agents.

**Method 2 — JNLP (Java Web Start)**
Agent machine connects TO the master. Used when the agent is behind a firewall.

**Method 3 — Docker**
Jenkins spins up a Docker container as a temporary agent. Container runs the job, then gets destroyed. No permanent machine needed.

---

## Topic 2 — Agent Directive in Jenkinsfile

### `agent any`
```groovy
pipeline {
    agent any
    ...
}
```
"Run this pipeline on ANY available agent." Jenkins picks whichever is free. This is what you have been using so far.

---

### `agent none`
```groovy
pipeline {
    agent none
    stages {
        stage('Build') {
            agent any
            steps { sh 'mvn package' }
        }
        stage('Deploy') {
            agent { label 'production-server' }
            steps { sh './deploy.sh' }
        }
    }
}
```
"No global agent — each stage picks its own agent."

Why? Because different stages might need different machines. Build needs a machine with Maven. Deploy needs a machine with access to the production server.

---

### `agent { label 'linux' }`
```groovy
pipeline {
    agent { label 'linux' }
    ...
}
```
"Run ONLY on agents that have the label `linux`."

Labels are tags you assign to agents in Jenkins. You can label agents by:
- Operating system → `linux`, `windows`
- Tools installed → `docker`, `maven`, `nodejs`
- Environment → `staging`, `production`

---

## Topic 3 — Docker Agent

### What is it?

Instead of a permanent machine, Jenkins spins up a Docker container, runs your pipeline inside it, then destroys the container.

### Why is this powerful?

Your build environment is defined IN CODE. You never have to manually install Maven or Java on an agent machine. The Docker image already has everything.

```groovy
pipeline {
    agent {
        docker {
            image 'maven:3.9-eclipse-temurin-17'
        }
    }
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
    }
}
```

### Line-by-Line Explanation

**`agent { docker { } }`** → Instead of running on a permanent machine, run inside a Docker container

**`image 'maven:3.9-eclipse-temurin-17'`** → Use this Docker image. It already has Maven 3.9 and Java 17 installed. You do not need to install anything manually.

**What happens behind the scenes:**
1. Jenkins pulls the `maven:3.9-eclipse-temurin-17` image from Docker Hub
2. Starts a container from that image
3. Runs all your pipeline steps INSIDE that container
4. Destroys the container when done

---

### Docker Agent with Cache (Important Real-World Pattern)

```groovy
pipeline {
    agent {
        docker {
            image 'maven:3.9-eclipse-temurin-17'
            args '-v $HOME/.m2:/root/.m2'
        }
    }
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
    }
}
```

**`args '-v $HOME/.m2:/root/.m2'`** → Mount the Maven cache from outside the container into the container.

Why? Every time the container is destroyed, all downloaded Maven dependencies are lost. Next build downloads everything again — slow. By mounting a folder from the host machine, the dependencies are saved between builds. This can save 5-10 minutes per build.

`-v` = volume mount. `$HOME/.m2` = folder on the host machine. `/root/.m2` = folder inside the container. They are linked — same files, both places.

---

## Topic 4 — Kubernetes Agent (Pod Templates)

### What is it?

Instead of a Docker container on one machine, Jenkins creates a **Kubernetes Pod** as the agent. When the build finishes, the Pod is deleted automatically.

### Why use Kubernetes agents?

- Auto-scaling — Kubernetes can create 50 agents simultaneously if 50 builds need to run
- No wasted resources — agents exist only during the build, then disappear
- Each build gets a completely clean environment

### The Pipeline Code

```groovy
pipeline {
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: maven
    image: maven:3.9-eclipse-temurin-17
    command: ['sleep', '99d']
  - name: docker
    image: docker:dind
    securityContext:
      privileged: true
"""
        }
    }

    stages {
        stage('Build') {
            steps {
                container('maven') {
                    sh 'mvn clean package'
                }
            }
        }

        stage('Docker Build') {
            steps {
                container('docker') {
                    sh 'docker build -t my-app .'
                }
            }
        }
    }
}
```

### Line-by-Line Explanation

**`agent { kubernetes { yaml """ ... """ } }`** → Define the agent as a Kubernetes Pod using YAML specification

**`kind: Pod`** → We are creating a Kubernetes Pod (one or more containers together)

**`containers:`** → List of containers inside this Pod. Each container is a separate tool environment.

**`- name: maven`** → First container named `maven`

**`image: maven:3.9-eclipse-temurin-17`** → This container uses the Maven Docker image

**`command: ['sleep', '99d']`** → Keep the container running for 99 days. Why? By default containers run their main process and exit. Jenkins needs the container to STAY RUNNING so it can run commands inside it.

**`- name: docker`** → Second container for running Docker commands

**`image: docker:dind`** → `dind` = Docker in Docker. Allows you to run Docker commands inside a container.

**`securityContext: privileged: true`** → Docker in Docker needs special Linux permissions to run. `privileged: true` gives those permissions.

**`container('maven') { }`** → Run the steps inside this block specifically in the `maven` container, not the default one.

Why multiple containers in one Pod?
- Build stage needs Maven → use `maven` container
- Docker stage needs Docker → use `docker` container
- They share the same workspace automatically — files created in one container are visible in the other

---

## Topic 5 — Agent Labels

### What are Labels?

Labels are tags you put on agents in Jenkins so your Jenkinsfile can say "run on this TYPE of machine" instead of naming a specific machine.

### How to Add a Label to an Agent

1. Jenkins UI → **Manage Jenkins** → **Nodes**
2. Click on an agent → **Configure**
3. Find **Labels** field → type your label, e.g. `linux docker maven`
4. Save

One agent can have MULTIPLE labels separated by space.

### Using Labels in Jenkinsfile

```groovy
// Run on any agent with the 'docker' label
pipeline {
    agent { label 'docker' }
    ...
}

// Run on agent that has BOTH 'linux' AND 'maven' labels
pipeline {
    agent { label 'linux && maven' }
    ...
}

// Run on agent with 'linux' OR 'windows'
pipeline {
    agent { label 'linux || windows' }
    ...
}
```

**`&&`** = AND — agent must have both labels
**`||`** = OR — agent can have either label

### Why Labels?

Without labels, you hardcode a specific machine name. If that machine is down, the build fails. With labels, Jenkins picks ANY available machine with that label. If one is down, another takes over.

---

## The Complete Picture — When to Use Which Agent

| Situation | Use This |
|-----------|----------|
| Just learning / small projects | `agent any` |
| Need specific OS or tool version | `agent { label 'linux' }` |
| Want clean environment per build | `agent { docker { image '...' } }` |
| Large team, many parallel builds | `agent { kubernetes { ... } }` |
| Different stages need different tools | `agent none` + per-stage agents |

---

## HANDS-ON LAB

> Read each step fully before typing.
> Everything happens in TWO places: your Ubuntu terminal and Jenkins browser.
> Do steps in order. Do not skip.

---

### What We Are Building in This Lab

We will update our Jenkinsfile to explicitly show agent usage — using labels and demonstrating how different agent configurations look and behave. Since you do not have a Kubernetes cluster set up, we will practice `agent any`, `agent { label }`, and `agent none` with per-stage agents.

---

### PART A — Understand What Agent You Are Currently On

#### Step A1 — Open Terminal

```bash
cd ~/Devops-lab/Practice-Lab/jenkins
```

You are now in your jenkins practice folder.

---

#### Step A2 — Check Your Current Branch

```bash
git branch
```

You should see `* main`. Good. Stay on main.

---

#### Step A3 — Open Jenkinsfile

```bash
nano Jenkinsfile
```

---

#### Step A4 — Delete Everything and Paste This New Jenkinsfile

Inside nano, press `Ctrl + K` repeatedly until the file is empty. Then paste this:

```groovy
pipeline {
    agent none

    stages {

        stage('Stage on Any Agent') {
            agent any
            steps {
                echo "Running on agent: ${env.NODE_NAME}"
                echo "Workspace on this agent: ${env.WORKSPACE}"
                sh 'whoami'
                sh 'hostname'
            }
        }

        stage('Check Tools Available') {
            agent any
            steps {
                echo "Checking what tools are on this agent..."
                sh 'java -version || echo "Java not found"'
                sh 'docker --version || echo "Docker not found"'
                sh 'git --version || echo "Git not found"'
                sh 'echo "All checks done"'
            }
        }

        stage('Simulate Build Stage') {
            agent any
            steps {
                echo "Build would run here using Maven or Gradle"
                sh 'echo Compiling application...'
                sh 'echo Build artifact created: myapp.jar'
            }
        }

        stage('Deploy - Main Branch Only') {
            agent any
            when {
                branch 'main'
            }
            steps {
                echo "Deploying on agent: ${env.NODE_NAME}"
                sh 'echo Deployment complete!'
            }
        }

    }

    post {
        success {
            echo "All stages passed on agent: ${env.NODE_NAME}"
        }
        failure {
            echo "Pipeline failed. Check which stage turned red."
        }
        always {
            echo "Cleanup done."
        }
    }
}
```

---

#### Step A5 — Save the File

Press `Ctrl + O` → Press `Enter` → Press `Ctrl + X`

---

#### Step A6 — Verify the File is Correct

```bash
cat Jenkinsfile
```

Check the file ends with exactly:

```groovy
    }

}
```

If it looks correct, continue.

---

#### Step A7 — Push to GitHub

Run these three commands one by one:

```bash
git add Jenkinsfile
```

```bash
git commit -m "Day 4 lab: agent none with per-stage agents"
```

```bash
git push origin main
```

You should see `main -> main` at the end. That means GitHub received it.

---

#### Step A8 — Go to Jenkins in Browser

Open: `http://localhost:8080`

---

#### Step A9 — Run the Build

1. Click on **`Day1_Declarative_Pipeline`**
2. Click **Build Now**
3. Click the new build number (e.g. `#4`)
4. Click **Console Output**

---

#### Step A10 — What You Should See

Look for these specific lines in the output:

```
Running on agent: Jenkins
Workspace on this agent: /var/jenkins_home/jobs/...
jenkins                    ← from whoami command
```

And the tools check:

```
java -version: openjdk version "17..."    ← or similar
docker --version: Docker version 27...   ← or "Docker not found"
git --version: git version 2.47.3
All checks done
```

**Important observation:** The `NODE_NAME` shows `Jenkins` — this is your Jenkins master acting as its own agent. In a real company, this would show the name of the dedicated agent machine.

---

### PART B — Add a Label to Your Jenkins Node and Use It

This shows you how labels work in a real setup.

---

#### Step B1 — Go to Jenkins Node Configuration

1. Click **Manage Jenkins** (left sidebar or top right gear icon)
2. Click **Nodes**
3. You will see a node called **Built-In Node** (this is the master acting as agent)
4. Click on it → Click **Configure** (left sidebar)

---

#### Step B2 — Add a Label

Find the field called **Labels**. It might be empty.

Type this in the Labels field:
```
linux practice-agent
```

(Two labels separated by a space — `linux` and `practice-agent`)

Click **Save**.

---

#### Step B3 — Update Jenkinsfile to Use the Label

Back in your terminal:

```bash
nano Jenkinsfile
```

Find this line in the `Stage on Any Agent` stage:

```groovy
        stage('Stage on Any Agent') {
            agent any
```

Change it to:

```groovy
        stage('Stage on Any Agent') {
            agent { label 'linux' }
```

Save: `Ctrl + O` → `Enter` → `Ctrl + X`

---

#### Step B4 — Push the Change

```bash
git add Jenkinsfile
git commit -m "Day 4 lab: use label linux for first stage"
git push origin main
```

---

#### Step B5 — Run the Build Again

Go to Jenkins → `Day1_Declarative_Pipeline` → **Build Now** → click new build → **Console Output**

You will see Jenkins matched the label and ran on the correct agent:
```
Running on Jenkins in /var/jenkins_home/...
```

The label `linux` matched the Built-In Node you labeled in Step B2.

---

### PART C — See What Happens With a Label That Doesn't Exist

This teaches you how Jenkins behaves when no agent has the required label.

---

#### Step C1 — Update Jenkinsfile with a Non-Existent Label

```bash
nano Jenkinsfile
```

Change the label to something that no agent has:

```groovy
        stage('Stage on Any Agent') {
            agent { label 'this-label-does-not-exist' }
```

Save and push:

```bash
git add Jenkinsfile
git commit -m "Day 4 lab: test non-existent label"
git push origin main
```

---

#### Step C2 — Run the Build

Go to Jenkins → **Build Now** → click new build → **Console Output**

You will see the build is **stuck waiting** with a message like:

```
Waiting for next available executor on [this-label-does-not-exist]
```

Jenkins is looking for an agent with that label but cannot find one. The build just waits forever.

**This is the exact error you would see in real companies when an agent is down or misconfigured.**

---

#### Step C3 — Fix It Back

```bash
nano Jenkinsfile
```

Change the label back to `linux`:

```groovy
        stage('Stage on Any Agent') {
            agent { label 'linux' }
```

Save and push:

```bash
git add Jenkinsfile
git commit -m "Day 4 lab: restore linux label"
git push origin main
```

Go to Jenkins → the stuck build → click **Abort** (X button) if it is still running. Then click **Build Now** again. It should pass now.

---

## ✅ Lab Checklist — Confirm All Before Moving to Day 5

- [ ] I updated Jenkinsfile with `agent none` and per-stage agents
- [ ] I pushed to GitHub and ran the build successfully
- [ ] I saw `NODE_NAME` and `WORKSPACE` printed in console output
- [ ] I saw the tool check output (java, docker, git versions)
- [ ] I added labels `linux practice-agent` to the Built-In Node in Jenkins
- [ ] I changed the stage to use `agent { label 'linux' }` and it worked
- [ ] I tested a non-existent label and saw Jenkins stuck waiting
- [ ] I fixed it back and the build passed again

---

## Troubleshooting

### Problem: Build stuck on "Waiting for next available executor"
**Cause:** No agent has the label you specified, OR all agents with that label are busy.
**Fix:** Check Manage Jenkins → Nodes → confirm the label is spelled exactly the same in both the node config and the Jenkinsfile. Labels are case-sensitive. `Linux` is not the same as `linux`.

### Problem: "docker --version" shows "Docker not found"
**This is okay for this lab.** It means Docker is not installed ON the Jenkins agent. This is normal — Jenkins agent does not need Docker installed unless you use Docker agent. When you use `agent { docker { image '...' } }`, Jenkins handles Docker separately.

### Problem: git push asks for username and password
**Fix:**
```bash
git config --global credential.helper store
git push origin main
```
Enter your GitHub username and Personal Access Token when prompted. It saves for future.

---

## Key Takeaways — Day 4

| Concept | What it means | When to use |
|---------|--------------|-------------|
| Jenkins Master | Brain — schedules, UI, manages | Always running, never runs builds directly |
| Jenkins Agent | Hands — runs actual build commands | All real builds run here |
| `agent any` | Any free agent | Learning, simple projects |
| `agent none` | No global agent, per-stage agents | When stages need different tools |
| `agent { label 'x' }` | Run on agent with label x | When you need specific OS or tools |
| `agent { docker {} }` | Run inside a Docker container | Clean environment per build |
| `agent { kubernetes {} }` | Run in a Kubernetes Pod | Large scale, auto-scaling |
| `NODE_NAME` | Environment variable = agent name | Useful for debugging |
| `WORKSPACE` | Path where build files live | Useful for referencing files |
| Labels | Tags on agents | Match pipelines to right machines |

---

*Next: Day 5 — Credentials (withCredentials, Secret Text, Username/Password, SSH Key)*
