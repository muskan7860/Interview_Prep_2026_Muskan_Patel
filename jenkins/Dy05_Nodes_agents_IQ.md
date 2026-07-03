# Jenkins Day 4 — Interview Q&A
## Agents & Nodes: Master-Agent Architecture, Docker Agent, Kubernetes Agent, Labels

> All answers written to be spoken aloud naturally.
> Senior-level phrasing throughout.

---

## Question 1
**"What is the difference between the Jenkins Master and a Jenkins Agent?"**

### How to Answer:

"The Jenkins Master — also called the Controller — is the central server that hosts the Jenkins UI, manages job scheduling, stores configuration, and coordinates all pipeline activity. It is the brain of Jenkins. In a production setup, the Master should never run build jobs directly — it only orchestrates.

Jenkins Agents — also called nodes — are the worker machines where actual build steps run. When a pipeline triggers, the Master assigns the job to an available agent, and all the `sh` commands, Maven builds, Docker operations run on that agent machine.

The reason for this separation is stability, security, and scalability. If a build crashes a machine, you want it to crash the agent, not the Master. If the Master goes down, your entire Jenkins installation becomes inaccessible. Keeping builds off the Master protects it.

In my understanding of real setups, the Master runs on a small, stable instance — maybe a t3.small — while agents are larger machines or auto-scaling Docker containers that handle the actual workload."

---

## Question 2
**"What does `agent any` mean? When would you NOT use it?"**

### How to Answer:

"`agent any` tells Jenkins to run the pipeline on any available agent — the Master picks whichever has a free executor slot.

I use it for simple pipelines or when the build doesn't depend on specific tools being installed on a particular machine.

I would NOT use `agent any` when:

First, different stages need different environments — for example, the Build stage needs Maven on a Linux machine, but the Deploy stage needs SSH access to a production server. In that case I use `agent none` at the top and define the agent per stage.

Second, when I need a guaranteed clean environment — I'd use a Docker agent instead, so each build gets a fresh container.

Third, when builds need to run on a specific OS — Windows builds need a Windows agent. Writing `agent any` might send a .NET build to a Linux agent that doesn't have the required tools."

---

## Question 3
**"What is `agent none` and why would you use it?"**

### How to Answer:

"`agent none` at the top of a declarative pipeline means there is no default agent for the entire pipeline. Instead, you must define an agent inside each individual stage.

The use case is when different stages need to run in different environments. A common pattern I'd use:

```groovy
pipeline {
    agent none
    stages {
        stage('Build') {
            agent { docker { image 'maven:3.9' } }
            steps { sh 'mvn package' }
        }
        stage('Deploy') {
            agent { label 'production-ssh-access' }
            steps { sh './deploy.sh' }
        }
    }
}
```

The Build stage runs in a Maven Docker container. The Deploy stage runs on a specific agent that has SSH access to the production server. These are completely different environments, and `agent none` lets me control that per stage.

One important thing: when you use `agent none`, the `post` block runs on whichever agent last ran a stage. So if cleanup is important, I make sure to be explicit about that."

---

## Question 4
**"What is a Docker agent in Jenkins? Why is it better than a permanent agent for builds?"**

### How to Answer:

"A Docker agent is when Jenkins uses a Docker container as the build environment instead of a permanent agent machine. You specify a Docker image in the `agent` block, Jenkins pulls that image, starts a container, runs all pipeline steps inside it, and destroys the container when done.

The key advantage is environment consistency. The Docker image defines exactly what tools are available — which version of Java, Maven, Node.js. Every developer and every pipeline run uses identical tools. You never hit issues like 'it worked on my machine because I have Java 17 but the agent has Java 11.'

Another advantage is isolation. Each build gets a completely clean container with no leftover files from previous builds.

And it simplifies agent management. Instead of maintaining permanent machines with specific tools installed, you just point to a Docker image. If you need a different tool, you change the image name.

The one consideration is build caching — by default, downloaded dependencies are lost when the container is destroyed. The solution is mounting a volume from the host machine into the container using the `args` parameter, so the cache persists between runs."

---

## Question 5
**"What is a Kubernetes agent in Jenkins? How is it different from a Docker agent?"**

### How to Answer:

"A Kubernetes agent takes the Docker agent concept further. Instead of running a container on a single machine, Jenkins creates a Kubernetes Pod in a cluster. When the build finishes, the Pod is automatically deleted.

The key difference from a Docker agent is scalability. A Docker agent is limited by the capacity of the single machine Jenkins is running on. A Kubernetes agent can spin up across an entire cluster — if 50 builds need to run simultaneously, Kubernetes creates 50 pods across multiple nodes. The infrastructure scales automatically.

Another difference is multi-container Pods. A Kubernetes Pod can have multiple containers — one with Maven for building, one with Docker for image building, one with kubectl for deployment — all sharing the same workspace. You use the `container('name')` step to run commands in a specific container.

In terms of syntax, you define the Pod specification as YAML directly in the Jenkinsfile under `agent { kubernetes { yaml """ ... """ } }`.

Kubernetes agents are the enterprise-grade solution. Companies using EKS or GKE almost always use Kubernetes agents for Jenkins because it eliminates the need to manage a fixed pool of agent machines."

---

## Question 6
**"What are Labels in Jenkins? How do you use them?"**

### How to Answer:

"Labels are tags you assign to agents in Jenkins to categorize them by their capabilities. For example, you might label some agents `linux`, others `windows`, some `docker`, some `maven`. One agent can have multiple labels.

In the Jenkinsfile, you use `agent { label 'linux' }` to tell Jenkins to run this pipeline only on agents that have the `linux` label. Jenkins then picks any available agent from that group.

Labels solve a real operational problem. Without labels, you'd hardcode a specific agent name — `agent { node 'build-server-01' }`. If that server goes down, the build fails. With labels, you write `agent { label 'linux' }` and Jenkins picks from any available linux agent. If one is down, another takes the build.

You can also combine labels with AND and OR logic. `agent { label 'linux && docker' }` runs on an agent that has both labels. `agent { label 'linux || windows' }` accepts either."

---

## Question 7
**"What is `NODE_NAME` and `WORKSPACE`? When do you use them?"**

### How to Answer:

"`NODE_NAME` and `WORKSPACE` are environment variables that Jenkins automatically sets for every build.

`NODE_NAME` contains the name of the agent that is running the current pipeline. I use this for debugging — when a build fails and I need to know which agent ran it, I check `NODE_NAME` in the logs. It's also useful in post-build scripts that need to log which machine processed a specific build.

`WORKSPACE` contains the full filesystem path where Jenkins checked out the code and is running the build on that agent. For example `/var/jenkins_home/jobs/my-pipeline/workspace`. I use it when I need to reference specific files in shell commands — instead of hardcoding a path, I write `${WORKSPACE}/target/myapp.jar`.

Both are accessed in Jenkinsfile as `${env.NODE_NAME}` and `${env.WORKSPACE}`."

---

## Question 8
**"What happens when Jenkins cannot find an agent matching the specified label?"**

### How to Answer:

"Jenkins puts the build in a waiting queue. The build doesn't fail immediately — it sits there waiting with a message like 'Waiting for next available executor on [label-name].'

This waiting behaviour is intentional. Jenkins assumes the agent might become available soon — maybe it's busy with another build and will free up.

However, if no agent with that label exists at all, the build will wait forever unless you manually abort it.

This is a common real-world issue. Causes include: a typo in the label name in the Jenkinsfile, the label on the agent was changed or removed, or all agents with that label are offline.

The fix is always to go to Manage Jenkins → Nodes, check the agent configurations, and verify the label is spelled exactly the same as in the Jenkinsfile. Labels are case-sensitive — `Linux` and `linux` are two different labels."

---

## Question 9
**"In a real company, how would you design the Jenkins agent setup for a team of 20 developers?"**

### How to Answer:

"For a team of 20 developers, I would design a multi-tier agent setup.

For the Jenkins Master — a small, stable VM — t3.small or similar — with no builds running on it. Its only job is the UI, scheduling, and storing configs.

For build agents, I'd use one of two approaches depending on the infrastructure:

If the company is on AWS or has Kubernetes, I'd use dynamic Kubernetes agents. Jenkins spins up Pods on demand, scales automatically during peak hours when many developers are pushing code, and costs nothing during off-hours when no builds are running.

If they have traditional VMs, I'd have a small permanent pool of agents — maybe 4-6 machines — labeled by capability: `linux-maven` for Java builds, `linux-docker` for Docker builds, `linux-node` for JavaScript builds. These stay always-on.

In either case, agents would have no Jenkins-specific configuration hardcoded into the Jenkinsfile. Everything is done via labels. This way, if we need to replace an agent machine, we just set up the new machine, give it the same labels, and all pipelines start using it automatically without any Jenkinsfile changes."

---

## Question 10
**"What is `docker:dind` and why does it need `privileged: true`?"**

### How to Answer:

"`dind` stands for Docker in Docker. It is a Docker image that has the Docker daemon inside it — meaning you can run Docker commands from inside a Docker container.

In the context of Jenkins Kubernetes agents, when you want to build Docker images as part of your pipeline, your build container needs access to Docker. One way to do that is `docker:dind` — a container that runs the Docker daemon itself.

The reason it needs `securityContext: privileged: true` is that running a Docker daemon requires low-level Linux kernel capabilities — it needs to create namespaces, manage cgroups, and mount filesystems. These are operations that are normally blocked for containers for security reasons. The `privileged` flag grants the container those capabilities.

It's worth noting that `privileged: true` is a security concern in production — a privileged container can potentially escape its sandbox. Many companies prefer alternatives like Kaniko or Buildah for building Docker images in Kubernetes without needing a privileged container."

---

## Quick-Fire Round

**Q: What is the Jenkins Master responsible for?**
A: UI, scheduling jobs, storing configs, managing agents — never runs build steps itself.

**Q: What is a Jenkins Agent responsible for?**
A: Running the actual build steps — all `sh` commands run on the agent.

**Q: What does `agent any` mean?**
A: Run on any available agent — Jenkins picks which one.

**Q: What does `agent none` mean?**
A: No global agent — each stage defines its own agent.

**Q: What does `agent { label 'linux' }` mean?**
A: Run only on agents that have the label `linux` assigned.

**Q: What is a Docker agent?**
A: Jenkins runs the pipeline inside a Docker container instead of a permanent machine.

**Q: What is the advantage of Docker agents?**
A: Clean environment per build, tools defined in the image, no manual setup on machines.

**Q: What is `dind`?**
A: Docker in Docker — allows running Docker commands inside a container.

**Q: What environment variable holds the agent name?**
A: `env.NODE_NAME`

**Q: What environment variable holds the build workspace path?**
A: `env.WORKSPACE`

**Q: What happens when no agent has the specified label?**
A: Build waits indefinitely in queue — it never fails, it just waits.

**Q: Why mount a volume in Docker agent?**
A: To persist Maven/NPM cache between builds — without it the cache is lost when container is destroyed.

---

## Scenario Questions

### Scenario 1
**"Your Jenkins pipeline is running builds on the Master server. Your manager says this is wrong. How do you fix it?"**

### Answer:
"Running builds on the Master is risky because it puts the Jenkins server at risk if a build crashes, and it can slow down the UI when builds are heavy.

To fix this, I would first set up a dedicated agent machine — a separate VM or EC2 instance — install Java on it, create a `jenkins` user, and connect it to the Master via SSH from Manage Jenkins → Nodes → New Node.

Then I'd label it appropriately — for example `linux docker` — and update all Jenkinsfiles to use `agent { label 'linux' }` instead of `agent any`.

Finally, I'd go to the Built-In Node on the Master and set its number of executors to zero. This prevents Jenkins from running any builds on the Master itself, even if a pipeline specifies `agent any`. All builds are forced to go to the dedicated agents."

---

### Scenario 2
**"You have 3 microservices, each needing different tools — Service A needs Java 17, Service B needs Node.js 18, Service C needs Python 3.11. How do you handle agents?"**

### Answer:
"I would use Docker agents — one different image per service's Jenkinsfile.

Service A Jenkinsfile:
```groovy
agent { docker { image 'eclipse-temurin:17-jdk' } }
```

Service B Jenkinsfile:
```groovy
agent { docker { image 'node:18-alpine' } }
```

Service C Jenkinsfile:
```groovy
agent { docker { image 'python:3.11-slim' } }
```

This way, I don't need to maintain three different physical agent machines with different tools. Each build automatically gets the right environment from the Docker image. If Service A needs to upgrade from Java 17 to Java 21, I just change the image name in that Jenkinsfile — no agent machine needs to be touched."

---

*Next: Day 5 — Credentials (Credentials Binding Plugin, withCredentials, Secret Text, Username/Password, SSH Key)*
