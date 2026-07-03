# Jenkins Agent Directory Structure
## Explored Inside Real Kubernetes Pod — July 3, 2026

> This file documents what I found when I went INSIDE my real jenkins-agent Kubernetes pod.
> Command used to enter the pod: `kubectl exec -it -n monitoring jenkins-agent -- bash`

---

## How I Got Inside the Pod

```bash
kubectl exec -it -n monitoring jenkins-agent -- bash
```

This opens a bash shell INSIDE the running Kubernetes pod — like SSHing into a server.

After running this, my terminal prompt changed to:
```
jenkins@jenkins-agent:~$
```

This means:
- `jenkins` = the user running inside the pod
- `jenkins-agent` = the pod hostname (same as pod name in Kubernetes)
- `~` = home directory which is `/home/jenkins`

---

## Top Level — What I Found in Home Directory

```bash
jenkins@jenkins-agent:~$ ls
agent
```

Only ONE folder called `agent`. Everything Jenkins does on this agent lives inside this single folder.

---

## The Full Directory Structure

```
/home/jenkins/
└── agent/
    ├── workspace/     ← Where your code lives and builds run
    ├── caches/        ← Speed optimization files
    └── remoting/      ← Agent ↔ Jenkins Master communication
        ├── logs/
        └── jarCache/
```

---

## 1. workspace/ — The Most Important Directory

```bash
jenkins@jenkins-agent:~/agent/workspace$ ls
Day1_Declarative_Pipeline@tmp  workspaces.txt
```

### What is workspace/?

This is where Jenkins does ALL the actual work:
- Git clone happens here
- Build commands run here (`mvn package`, `docker build`)
- Test reports are generated here
- Artifacts are created here

Every Jenkins job gets its OWN subfolder inside workspace:

```
workspace/
├── Day1_Declarative_Pipeline/          ← your Day 1 pipeline files
├── Day2_Multibranch_lab_main/          ← Day 2 main branch
├── Day2_Multibranch_lab_feature_test/  ← Day 2 feature branch
```

### Why Did I See `Day1_Declarative_Pipeline@tmp` Instead of `Day1_Declarative_Pipeline`?

```bash
workspace/
└── Day1_Declarative_Pipeline@tmp    ← @tmp means workspace was cleaned
```

The `@tmp` means the actual workspace was already cleaned — either by `deleteDir()` or `cleanWs()`. Only the temporary directory remained.

**This proves our cleanup step worked!**

### What is `@tmp`?

`Day1_Declarative_Pipeline@tmp` is a temporary directory Jenkins creates DURING a build. It stores:
- Shell script wrappers
- Pipeline metadata
- Temporary Groovy files

Normally Jenkins deletes it after the build finishes. It was left behind because our build had an error in the post block.

---

### What is workspaces.txt?

```bash
jenkins@jenkins-agent:~/agent/workspace$ cat workspaces.txt
Day2_Multibranch_lab/feature%2Ftest-multibranch
Day2_Multibranch_lab_feature_test-multibranch
Day2_Multibranch_lab/main
Day2_Multibranch_lab_main
Day2_Multibranch_lab/release%2Ftest-multibranch
Day2_Multibranch_lab_release_test-multibranch
Day2_Multibranch_lab/test
Day2_Multibranch_lab_test
```

This file is a **mapping** between Git branch names and workspace folder names.

**Why?** Because Linux folder names cannot contain `/` safely. So Jenkins converts branch names:

| Git Branch Name | Workspace Folder Name |
|---|---|
| `main` | `Day2_Multibranch_lab_main` |
| `feature/test-multibranch` | `Day2_Multibranch_lab_feature_test-multibranch` |
| `release/test-multibranch` | `Day2_Multibranch_lab_release_test-multibranch` |

The `/` in branch names becomes `_` in folder names. The `%2F` is URL encoding for `/`.

---

## 2. caches/ — Speed Optimization

```bash
jenkins@jenkins-agent:~/agent/caches$ ls
durable-task
```

### What is caches/?

This stores cached data to make builds faster.

### What is durable-task?

When you run any shell command in Jenkins:
```groovy
sh 'mvn clean package'
sh 'docker build -t myapp .'
```

Jenkins does NOT run these directly. It creates **wrapper scripts** and manages them through the Durable Task plugin.

These wrapper scripts are cached here.

### Why is This Important?

If the Jenkins agent disconnects during a build (network hiccup, pod restart), the durable task cache allows Jenkins to **reconnect and continue** without losing the running build. Without this cache, any disconnection would kill the build immediately.

---

## 3. remoting/ — Agent ↔ Master Communication

```bash
jenkins@jenkins-agent:~/agent/remoting$ ls
jarCache  logs
```

This directory handles ALL communication between the agent pod and Jenkins master.

---

### remoting/logs/ — What I Found

```bash
jenkins@jenkins-agent:~/agent/remoting/logs$ ls
remoting.log.0  remoting.log.0.lck  remoting.log.1
```

```bash
jenkins@jenkins-agent:~/agent/remoting/logs$ cat remoting.log.0
Jul 03, 2026 9:45:33 AM hudson.remoting.Launcher createEngine
INFO: Setting up agent: k8s-agent

Jul 03, 2026 9:45:33 AM hudson.remoting.Engine startEngine
INFO: Using Remoting version: 3383.vc8881d4b_0e76

Jul 03, 2026 9:45:33 AM org.jenkinsci.remoting.engine.WorkDirManager initializeWorkDir
INFO: Using /home/jenkins/agent/remoting as a remoting work directory

Jul 03, 2026 9:45:34 AM hudson.remoting.Launcher$CuiListener status
INFO: WebSocket connection open

Jul 03, 2026 9:45:34 AM hudson.remoting.Launcher$CuiListener status
INFO: Connected
```

### Reading the Log Line by Line

**`Setting up agent: k8s-agent`**
→ The agent started and named itself `k8s-agent` — matching what is registered in Jenkins UI.

**`Using Remoting version: 3383.vc8881d4b_0e76`**
→ Remoting is the software that handles Jenkins master-agent communication. Version 3383 is what this agent uses.

**`Using /home/jenkins/agent/remoting as a remoting work directory`**
→ Confirms this folder is used for communication files.

**`WebSocket connection open`**
→ **Very important!** Instead of old TCP protocol on port 50000, our agent uses WebSocket through HTTP port 8080. This is why our setup works — WebSocket goes through the internal Kubernetes service.

**`Connected`**
→ Agent successfully registered with Jenkins master. Everything is working.

---

### remoting/jarCache/ — What I Found

```bash
jenkins@jenkins-agent:~/agent/remoting/jarCache$ ls
0F  12  1C  23  33  43  57  5A  60  62  69  72  8B  93  95  97  A8  B4  C2  C4  E4  ED  FC
```

These folders with hex names (`0F`, `12`, `1C`...) are **cached Java JAR files**.

### Why are they named with hex?

Jenkins uses a hash-based caching system. Each JAR file is identified by its hash (like a fingerprint). The first 2 characters of the hash become the folder name.

### Why does this cache exist?

When Jenkins runs a build, the master sends plugin code and classes to the agent. Without cache, every single build would download the same JAR files over the network.

With cache, first build downloads everything → stores here → all future builds use the cached version.

This is especially important for Kubernetes agents because:
- If the pod restarts, the cache is gone (ephemeral storage)
- But during the pod's lifetime, subsequent builds are much faster

---

## What cleanWs() and deleteDir() Actually Delete

### Before cleanup:
```
workspace/
└── Day1_Declarative_Pipeline/
    ├── .git/
    ├── Jenkinsfile
    ├── target/
    │   └── myapp.jar
    └── myfile.txt
```

### After cleanWs() or deleteDir():
```
workspace/
└── (empty)
```

Only the **job's workspace folder contents** are deleted.

---

## What cleanWs() Does NOT Delete

| Directory | Deleted by cleanWs()? | Why? |
|-----------|----------------------|------|
| `workspace/Day1_Declarative_Pipeline/` | ✅ YES — contents deleted | This is the job workspace |
| `caches/` | ❌ NO | Build speed cache — kept |
| `remoting/` | ❌ NO | Agent communication — kept |
| `remoting/jarCache/` | ❌ NO | JAR cache — kept |
| Agent connection | ❌ NO | Agent stays connected |
| Jenkins configuration | ❌ NO | Lives on master, not agent |

**Key point:** Cleaning workspace does NOT disconnect the agent or affect Jenkins master in any way. The agent keeps running and stays connected. Only the build files are removed.

---

## Visual Summary

```
/home/jenkins/agent/
│
├── workspace/          ← YOUR BUILD FILES LIVE HERE
│   ├── Job1/           ← Git clone + build output
│   ├── Job2/
│   └── workspaces.txt  ← Branch name → folder mapping
│
├── caches/             ← SPEED FILES (kept between builds)
│   └── durable-task/   ← Shell command wrapper scripts
│
└── remoting/           ← AGENT ↔ MASTER COMMUNICATION
    ├── logs/           ← Connection logs (remoting.log.0)
    │   └── remoting.log.0  → Shows "Connected" via WebSocket
    └── jarCache/       ← Cached Java JARs (0F, 12, 1C...)
        └── (hex folders = cached plugin JARs)
```

---

## Interview Answer — "What happens inside a Jenkins agent?"

*"When a Jenkins agent starts, it creates a working directory — in my Kubernetes setup that's `/home/jenkins/agent`. Inside, there are three main directories. The `workspace` folder is where actual build work happens — git clone, compilation, testing. Each job gets its own subfolder. The `caches` folder stores durable task wrapper scripts that allow builds to survive agent disconnections. And `remoting` handles all communication between the agent and master — it stores connection logs and a JAR cache so plugin code doesn't need to be re-downloaded every build.*

*When you run `cleanWs()` or `deleteDir()`, only the job's workspace folder is deleted. The agent stays connected, the remoting files stay intact, and the JAR cache is preserved. This is by design — cleanup should not affect agent health.*

*I know this from experience — I went inside my actual `jenkins-agent` Kubernetes pod using `kubectl exec -it` and explored the directory structure firsthand."*

---

*Explored on: July 3, 2026*
*Pod: jenkins-agent in namespace: monitoring*
*Command: `kubectl exec -it -n monitoring jenkins-agent -- bash`*
