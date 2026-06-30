# Jenkins Day 2 — Multibranch Pipeline & Shared Libraries
## Theory + Labs

---

## Part 1 — Multibranch Pipeline

### The Problem First (Story)

Yesterday you created a Jenkins job called `Day1_Declarative_Pipeline`. It points to ONE branch — `main`.

Now imagine this real situation:

You're working on a feature. You don't want to push directly to `main` (that's risky — could break production). So you create a new branch:

```bash
git checkout -b feature/payment-retry
```

You push this branch to GitHub. Now ask yourself — **does your existing Jenkins job test this new branch?**

**No.** Your job only watches `main`. Your `feature/payment-retry` branch has zero CI checks running on it. If it has bugs, you won't know until you merge it into `main` — too late.

### The Real-World Need

In any real company, developers create dozens of branches every day:
- `feature/login-page`
- `feature/payment-retry`
- `bugfix/null-pointer-exception`
- `hotfix/security-patch`

Every single one of these needs its own build and test run — **automatically**, without anyone manually creating a Jenkins job for each branch.

---

### What is Multibranch Pipeline?

A **Multibranch Pipeline** is a special type of Jenkins job that:

1. Scans your entire GitHub/GitLab repository
2. Automatically discovers every branch that has a `Jenkinsfile`
3. Creates a separate pipeline job for EACH branch — automatically
4. Deletes the pipeline job automatically when the branch is deleted

Think of it like a **smart receptionist** at a hotel. Every time a new guest (branch) arrives, the receptionist automatically assigns them a room (a Jenkins job) — no manual work needed. When the guest checks out (branch deleted), the room is automatically freed up.

---

### Why Not Just Use Regular Pipeline Job for Every Branch?

| Without Multibranch | With Multibranch |
|---|---|
| Create a new Jenkins job manually for every branch | Jenkins auto-creates job when branch is pushed |
| Manually delete job when branch is deleted | Jenkins auto-deletes job |
| Hard to maintain 50 branches = 50 manual jobs | Zero manual work |
| No automatic Pull Request building | Can auto-build PRs too |

---

### How Multibranch Pipeline Works — Step by Step

```
1. You create job type "Multibranch Pipeline" in Jenkins
        ↓
2. You give it your Git repo URL
        ↓
3. Jenkins scans the repo — looks at EVERY branch
        ↓
4. For each branch, Jenkins checks: "Does this branch have a Jenkinsfile?"
        ↓
5. If YES → Jenkins creates a sub-job for that branch automatically
   If NO  → Jenkins skips that branch
        ↓
6. Every time you push new commits to any branch → that branch's pipeline runs
        ↓
7. Every time a branch is deleted from Git → Jenkins removes that sub-job
```

---

### Setting Up Multibranch Pipeline — Hands-On Lab

#### Step 1 — Create the Job

1. Jenkins UI → **New Item**
2. Name: `Devops-Practice-Multibranch`
3. Select **Multibranch Pipeline** → Click **OK**

#### Step 2 — Configure Branch Source

1. Under **Branch Sources** → Click **Add Source** → **Git**
2. Repository URL: `https://github.com/muskan7860/Devops-Practice-lab.git`
3. Credentials: (leave blank for public repo, or add if private)

#### Step 3 — Build Configuration

- Mode: **by Jenkinsfile**
- Script Path: `Jenkinsfile` (default — Jenkins looks for this filename in every branch)

#### Step 4 — Scan Triggers

- Check **Periodically if not otherwise run** → Set interval, e.g., 1 minute (for testing; production usually uses webhooks)

#### Step 5 — Save

Click **Save**. Jenkins immediately scans your repo.

---

### What You'll See After Saving

Jenkins UI will show something like:

```
Devops-Practice-Multibranch
├── main                          (branch with Jenkinsfile found)
├── feature/payment-retry         (branch with Jenkinsfile found)
└── bugfix/login-issue            (branch with Jenkinsfile found)
```

Each branch name becomes its own clickable pipeline, with its own build history, completely independent of the others.

---

### Lab Exercise — Create a Second Branch and Watch Jenkins Auto-Detect It

```bash
# In your repo
git checkout -b feature/test-multibranch

# Make a small change to Jenkinsfile (e.g., add an echo step)
echo 'Edit your Jenkinsfile to add a new echo line here'

git add Jenkinsfile
git commit -m "Test multibranch auto-detection"
git push origin feature/test-multibranch
```

Now go to your Multibranch Pipeline job in Jenkins → Click **Scan Repository Now**.

**Expected result:** A new branch `feature/test-multibranch` appears automatically as a new pipeline — you did not create it manually.

---

### Why is this important for interviews?

Multibranch Pipeline is the **standard real-world setup** in almost every company. Interviewers expect you to know this — because no real team manually creates Jenkins jobs for every Git branch. It would not scale.

---

## Part 2 — Jenkins Shared Libraries

### The Problem First (Story)

Imagine you have 20 microservices, each with its own Jenkinsfile. Every Jenkinsfile has this SAME block of code to build and push a Docker image:

```groovy
stage('Docker Build') {
    steps {
        sh "docker build -t myrepo/${APP_NAME}:${BUILD_NUMBER} ."
        withCredentials([usernamePassword(
            credentialsId: 'dockerhub-creds',
            usernameVariable: 'DOCKER_USER',
            passwordVariable: 'DOCKER_PASS'
        )]) {
            sh "echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin"
            sh "docker push myrepo/${APP_NAME}:${BUILD_NUMBER}"
        }
    }
}
```

Now imagine your Docker registry URL changes, or you need to add a security scan step before pushing. You would have to **manually edit this same code in 20 different Jenkinsfiles**.

That's a maintenance nightmare. One small change = 20 repos to update = high chance of mistakes.

### What is a Shared Library?

A Shared Library is **reusable Groovy code** stored in its own separate Git repository, which any Jenkinsfile across your organization can import and call — like a function.

Think of it like a **common toolbox** shared across a construction site. Instead of every worker carrying their own hammer (duplicate code), there's one toolbox (shared library) that everyone borrows from.

---

### Why is this a Best Practice?

| Without Shared Library | With Shared Library |
|---|---|
| Same Docker build code copy-pasted in 20 Jenkinsfiles | One function, called from 20 Jenkinsfiles |
| Bug fix requires editing 20 files | Bug fix requires editing 1 file |
| Inconsistent implementations across teams | Standardized, consistent pipeline behavior |
| No code review process for pipeline logic | Shared library is its own Git repo — gets PR review |

---

### Structure of a Shared Library Repository

A Shared Library is its own Git repository with this exact folder structure:

```
my-shared-library/
├── vars/
│   ├── buildDockerImage.groovy
│   └── deployToK8s.groovy
├── src/
│   └── org/
│       └── mycompany/
│           └── Utils.groovy
└── resources/
    └── org/
        └── mycompany/
            └── config.json
```

#### Explanation of each folder:

**`vars/`**
- Contains **global functions/steps** that can be called directly from any Jenkinsfile.
- Each `.groovy` file in this folder becomes a usable step.
- Example: `vars/buildDockerImage.groovy` becomes callable as `buildDockerImage()` in any Jenkinsfile.
- This is the folder you will use 90% of the time.

**`src/`**
- Contains full Groovy classes — for complex object-oriented logic.
- Used when you need classes, not just simple functions.
- Follows standard Java/Groovy package structure.

**`resources/`**
- Contains non-Groovy files — JSON configs, shell scripts, templates — that your library code needs to read.

---

### Writing Your First Shared Library Function

#### File: `vars/buildDockerImage.groovy`

```groovy
def call(String imageName, String tag) {
    // This 'call' method is what runs when you write buildDockerImage() in Jenkinsfile
    
    echo "Building Docker image: ${imageName}:${tag}"
    
    sh "docker build -t ${imageName}:${tag} ."
    
    withCredentials([usernamePassword(
        credentialsId: 'dockerhub-creds',
        usernameVariable: 'DOCKER_USER',
        passwordVariable: 'DOCKER_PASS'
    )]) {
        sh "echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin"
        sh "docker push ${imageName}:${tag}"
    }
    
    echo "Docker image pushed successfully!"
}
```

#### Line-by-Line Explanation:

**`def call(String imageName, String tag) { }`**
- `call` is a special method name in Groovy/Jenkins. Whatever you name the FILE becomes the function name in the Jenkinsfile. The method INSIDE must be named `call`.
- `String imageName, String tag` — these are parameters. The Jenkinsfile will pass actual values when calling this function.
- Why parameters? So the SAME function works for ANY microservice — you just pass different image names.

**`echo "Building Docker image: ${imageName}:${tag}"`**
- Standard string interpolation — prints status so you know what's happening in the logs.

**Rest of the code**
- Same Docker build/push logic you saw before — but now it lives in ONE place.

---

### How to USE the Shared Library in Your Jenkinsfile

#### Step 1 — Register the Library in Jenkins (One-Time Setup by Admin)

1. Jenkins UI → **Manage Jenkins** → **System**
2. Scroll to **Global Pipeline Libraries**
3. Add:
   - Name: `my-shared-library`
   - Default version: `main`
   - Retrieval method: Modern SCM → Git
   - Repository URL: your shared library repo

#### Step 2 — Call it from any Jenkinsfile

```groovy
@Library('my-shared-library') _

pipeline {
    agent any
    
    stages {
        stage('Build and Push') {
            steps {
                buildDockerImage('myrepo/payment-service', "${env.BUILD_NUMBER}")
            }
        }
    }
}
```

#### Explanation:

**`@Library('my-shared-library') _`**
- `@Library(...)` imports the shared library by the name you registered in Jenkins config.
- The underscore `_` at the end is required Groovy syntax — it's a placeholder that tells Jenkins "import this library globally" without assigning it to a variable.
- Why is this needed? Without this line, Jenkins has no idea your shared library functions exist.

**`buildDockerImage('myrepo/payment-service', "${env.BUILD_NUMBER}")`**
- This calls the function from `vars/buildDockerImage.groovy`.
- It looks just like a normal pipeline step — that's the beauty of `vars/` functions.
- You pass two arguments matching the `call(String imageName, String tag)` signature.

---

### Real Example — A Larger Shared Library Function

#### File: `vars/sonarQubeScan.groovy`

```groovy
def call(Map config = [:]) {
    // Map allows named parameters - more readable than positional args
    def projectKey = config.projectKey ?: 'default-project'
    def sourcePath = config.sourcePath ?: 'src'
    
    echo "Running SonarQube scan for project: ${projectKey}"
    
    withSonarQubeEnv('MySonarQubeServer') {
        sh """
            sonar-scanner \
              -Dsonar.projectKey=${projectKey} \
              -Dsonar.sources=${sourcePath} \
              -Dsonar.host.url=\$SONAR_HOST_URL \
              -Dsonar.login=\$SONAR_AUTH_TOKEN
        """
    }
    
    timeout(time: 5, unit: 'MINUTES') {
        waitForQualityGate abortPipeline: true
    }
}
```

#### Explanation:

**`def call(Map config = [:])`**
- This time we accept a `Map` instead of separate string parameters.
- Why? When you have many optional parameters, a Map is cleaner.
- `[:]` means an empty map is the default if nothing is passed.

**`config.projectKey ?: 'default-project'`**
- `?:` is the Groovy "Elvis operator" — it means "if `config.projectKey` is null/empty, use 'default-project' instead."
- Why? So the function works even if the caller doesn't provide every parameter.

**`withSonarQubeEnv('MySonarQubeServer')`**
- A Jenkins step (from SonarQube plugin) that automatically injects SonarQube server URL and auth token as environment variables.
- Why? So you don't have to hardcode the SonarQube server URL.

**`waitForQualityGate abortPipeline: true`**
- After the scan, SonarQube needs time to process results and determine pass/fail (called the "Quality Gate").
- This step PAUSES the pipeline until SonarQube responds.
- `abortPipeline: true` means if the Quality Gate fails (e.g., too many bugs/vulnerabilities found), the entire pipeline stops here — it won't proceed to deploy bad code.

#### How to call it from Jenkinsfile:

```groovy
stage('Code Quality') {
    steps {
        sonarQubeScan(projectKey: 'payment-service', sourcePath: 'src/main/java')
    }
}
```

---

## Lab — Build Your Own Mini Shared Library

### Step 1 — Create a New GitHub Repo

Create a repo named `jenkins-shared-library` (separate from your practice lab repo).

### Step 2 — Create the Folder Structure

```bash
mkdir -p jenkins-shared-library/vars
cd jenkins-shared-library
```

### Step 3 — Create `vars/sayHello.groovy`

```groovy
def call(String name) {
    echo "Hello, ${name}! This message came from the Shared Library."
    echo "Build is running on: ${env.NODE_NAME}"
    echo "Current build number: ${env.BUILD_NUMBER}"
}
```

### Step 4 — Push to GitHub

```bash
git init
git add vars/sayHello.groovy
git commit -m "Add sayHello shared library function"
git branch -M main
git remote add origin https://github.com/muskan7860/jenkins-shared-library.git
git push -u origin main
```

### Step 5 — Register Library in Jenkins

Manage Jenkins → System → Global Pipeline Libraries → Add:
- Name: `practice-library`
- Default version: `main`
- Source: Git → your new repo URL

### Step 6 — Use it in Your Day 1 Jenkinsfile

Add this at the top of your `Day1_Declarative_Pipeline`'s Jenkinsfile:

```groovy
@Library('practice-library') _

pipeline {
    agent any
    stages {
        stage('Shared Library Test') {
            steps {
                sayHello('Muskan')
            }
        }
        // ... rest of your existing stages
    }
}
```

### Step 7 — Run and Verify

Build the job. You should see in the console output:
```
Hello, Muskan! This message came from the Shared Library.
Build is running on: ...
Current build number: ...
```

**If you see this — you have successfully created and consumed a Shared Library.**

---

## Multibranch Pipeline + Shared Library — How They Work Together

In real companies, this is the standard combo:

```
jenkins-shared-library (repo)        ← common reusable functions
        ↑
        | imported by
        |
service-a (repo, Multibranch)        ← has Jenkinsfile per branch
service-b (repo, Multibranch)        ← has Jenkinsfile per branch
service-c (repo, Multibranch)        ← has Jenkinsfile per branch
```

Every microservice's Jenkinsfile is SHORT because all the heavy logic lives in the shared library:

```groovy
@Library('practice-library') _

pipeline {
    agent any
    stages {
        stage('Build') { steps { buildDockerImage('service-a', env.BUILD_NUMBER) } }
        stage('Scan')  { steps { sonarQubeScan(projectKey: 'service-a') } }
        stage('Deploy') { steps { deployToK8s('service-a', env.BRANCH_NAME) } }
    }
}
```

This is exactly the kind of Jenkinsfile a senior engineer is expected to design.

---

## Summary Table

| Concept | What it Solves | Key Syntax |
|---|---|---|
| Multibranch Pipeline | Auto-detect and build every Git branch | Job type: Multibranch Pipeline |
| Shared Library | Reuse pipeline code across multiple repos | `@Library('name') _` |
| `vars/` folder | Define callable functions | `def call(...) { }` |
| `src/` folder | Define full Groovy classes | Standard package structure |
| `resources/` folder | Store config files/templates | Read via `libraryResource` |
| Elvis operator `?:` | Provide default values | `config.x ?: 'default'` |

---

## Common Mistakes to Avoid

1. **Forgetting the underscore `_` after `@Library(...)`** — Pipeline will fail to recognize the import.
2. **Naming the method inside `vars/` file something other than `call`** — Jenkins specifically looks for `call`.
3. **Not registering the library in Jenkins global config before using `@Library`** — Jenkins won't know where to find it.
4. **Using Multibranch Pipeline but forgetting every branch needs its own Jenkinsfile** — branches without one are simply skipped, not errored.
5. **Hardcoding values inside shared library functions** — defeats the purpose; always use parameters.

---

*Next: Day 2 — Interview Q&A File for Multibranch Pipeline & Shared Libraries*
