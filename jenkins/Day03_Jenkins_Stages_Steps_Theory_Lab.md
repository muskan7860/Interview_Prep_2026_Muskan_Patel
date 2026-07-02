# Jenkins Day 3 — Stages & Steps
## Theory + Hands-On Lab

---

## Before We Start — What Are Stages and Steps?

### Simple Analogy First

Think of making a cup of tea:
1. Boil water
2. Add tea bag
3. Add sugar
4. Pour into cup
5. Serve

Each of these is one **stage**. Inside each stage, you do specific actions — that's a **step**.

If you forget to boil water (stage 1 fails), you don't proceed to add tea bag. Same in Jenkins — if one stage fails, the pipeline stops.

---

### In Jenkins Language

```
Pipeline = The entire tea-making process
Stage    = One step in the process (Boil water, Add tea bag...)
Step     = The actual action inside that stage (turn on kettle, wait 3 minutes...)
```

---

## The Full Pipeline We Are Building Today

We will build a real pipeline that does this in order:

```
Checkout → Build → Test → Code Analysis → Docker Build → Docker Push → Deploy → Post Actions
```

This is exactly how a real company pipeline looks.

---

## Stage 1 — Checkout

### What is it?

This is always the FIRST stage in every pipeline. It tells Jenkins: "Go to GitHub and download the latest code into the workspace."

### Why do we need it?

Jenkins workspace starts empty. Before building anything, you need the code. Checkout fetches it.

### Code:

```groovy
stage('Checkout') {
    steps {
        checkout scm
    }
}
```

### Explanation word by word:

**`checkout`** → Jenkins built-in command that fetches code from source control

**`scm`** → Stands for "Source Control Manager" — means "use whatever Git repo this job is configured with." Jenkins already knows the repo URL from the job config, so you don't repeat it here.

---

## Stage 2 — Build

### What is it?

After getting the code, compile it and package it into a runnable file (JAR for Java, or just validate for other languages).

### Why do we need it?

Raw code is just text files. A computer cannot run text files. Build converts code into an executable artifact.

### Code:

```groovy
stage('Build') {
    steps {
        sh 'mvn clean package -DskipTests'
    }
}
```

### Explanation word by word:

**`sh`** → "Run this as a shell command on the agent machine"

**`mvn`** → Maven — the Java build tool

**`clean`** → Delete any previous build output first (the `target/` folder). Fresh start every time.

**`package`** → Compile the code and package it into a JAR/WAR file

**`-DskipTests`** → Skip running tests during build. We have a separate Test stage for that — no need to run tests twice.

---

## Stage 3 — Test

### What is it?

Run automated unit tests to verify the code works correctly.

### Why do we need it?

Tests catch bugs before code reaches production. If tests fail here, the pipeline stops — broken code never moves forward.

### Code:

```groovy
stage('Test') {
    steps {
        sh 'mvn test'
    }
    post {
        always {
            junit 'target/surefire-reports/*.xml'
        }
    }
}
```

### Explanation word by word:

**`mvn test`** → Maven command that runs all unit tests

**`post { always { } }`** → After this stage finishes, ALWAYS run this — even if tests failed

**`junit`** → Jenkins built-in step that reads test result files and shows a pass/fail graph in the UI

**`'target/surefire-reports/*.xml'`** → The path where Maven saves test results. `*` means "all XML files in this folder"

### Why `always` here and not `success`?

Because even when tests fail, you want to see WHICH tests failed. If you used `success`, you'd never see test reports on failure — that defeats the purpose.

---

## Stage 4 — Code Analysis (SonarQube)

### What is SonarQube?

SonarQube is a code quality tool. It scans your code and reports:
- Bugs
- Security vulnerabilities
- Code smells (bad practices)
- How much of your code is covered by tests

Think of it as a **grammar checker for code** — like Grammarly, but for Java/Python/etc.

### What is a Quality Gate?

After scanning, SonarQube decides: "Is this code good enough?" If yes → Quality Gate PASSED → pipeline continues. If no → Quality Gate FAILED → pipeline stops. You don't deploy bad code.

### Code:

```groovy
stage('Code Analysis') {
    steps {
        withSonarQubeEnv('SonarQube-Server') {
            sh 'mvn sonar:sonar -Dsonar.projectKey=my-app'
        }
    }
}

stage('Quality Gate') {
    steps {
        timeout(time: 5, unit: 'MINUTES') {
            waitForQualityGate abortPipeline: true
        }
    }
}
```

### Explanation word by word:

**`withSonarQubeEnv('SonarQube-Server')`** → Jenkins step (from SonarQube plugin) that automatically injects the SonarQube server URL and authentication token as environment variables. The name `'SonarQube-Server'` must match the name configured in Jenkins → Manage Jenkins → System → SonarQube servers.

**`mvn sonar:sonar`** → Maven command that runs the SonarQube scanner plugin, sends code analysis to SonarQube server

**`-Dsonar.projectKey=my-app`** → Tells SonarQube which project this belongs to in its dashboard

**`timeout(time: 5, unit: 'MINUTES')`** → SonarQube needs time to process results. This says "wait maximum 5 minutes for the result. If no answer in 5 minutes, fail."

**`waitForQualityGate`** → Pauses the pipeline and waits for SonarQube to send back pass/fail

**`abortPipeline: true`** → If Quality Gate fails (too many bugs/vulnerabilities), STOP the entire pipeline. Do not proceed to Docker build or deploy.

---

## Stage 5 — Docker Build

### What is it?

Package your application and all its dependencies into a Docker image.

### Why?

A JAR file alone needs Java installed on the server to run. A Docker image contains everything — Java, your JAR, all dependencies — in one portable package. Run it anywhere.

### Code:

```groovy
stage('Docker Build') {
    steps {
        sh "docker build -t myrepo/my-app:${env.BUILD_NUMBER} ."
    }
}
```

### Explanation word by word:

**`docker build`** → Command to build a Docker image from a Dockerfile

**`-t`** → "tag" — give this image a name

**`myrepo/my-app`** → The image name. `myrepo` = your Docker Hub username. `my-app` = image name.

**`:${env.BUILD_NUMBER}`** → Tag the image with the Jenkins build number (1, 2, 3...). Each build creates a uniquely tagged image. This lets you roll back to any previous version.

**`.`** → The dot means "look for the Dockerfile in the current directory (workspace)"

---

## Stage 6 — Docker Push

### What is it?

Upload the Docker image to Docker Hub (or any registry) so servers can download and run it.

### Why?

The image built in Stage 5 only exists on the Jenkins agent machine right now. To deploy it anywhere, it must be stored in a central registry — like a library where everyone can borrow books.

### Code:

```groovy
stage('Docker Push') {
    steps {
        withCredentials([usernamePassword(
            credentialsId: 'dockerhub-creds',
            usernameVariable: 'DOCKER_USER',
            passwordVariable: 'DOCKER_PASS'
        )]) {
            sh "echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin"
            sh "docker push myrepo/my-app:${env.BUILD_NUMBER}"
        }
    }
}
```

### Explanation word by word:

**`withCredentials([...])`** → Jenkins step that safely retrieves stored credentials (username/password) and injects them as temporary environment variables. Credentials are NEVER visible in logs.

**`credentialsId: 'dockerhub-creds'`** → The ID of the credential you saved in Jenkins (Manage Jenkins → Credentials). `'dockerhub-creds'` is just the name you give it — you set this when saving.

**`usernameVariable: 'DOCKER_USER'`** → Jenkins will put the username into a variable called `DOCKER_USER`

**`passwordVariable: 'DOCKER_PASS'`** → Jenkins will put the password into a variable called `DOCKER_PASS`

**`echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin`** → Logs into Docker Hub. `--password-stdin` means password is passed through input pipe — safer than typing it as a flag (which would show in process list)

**`docker push`** → Uploads the image to Docker Hub

### Why NOT hardcode credentials?

```groovy
// WRONG — never do this
sh "docker login -u muskan -p MyPassword123"
```

If someone reads your Jenkinsfile in GitHub, they see your password. Always use `withCredentials`.

---

## Stage 7 — Deploy

### What is it?

Take the Docker image and run it on the target server.

### Code:

```groovy
stage('Deploy') {
    when {
        branch 'main'
    }
    steps {
        sh './deploy.sh'
    }
}
```

### Explanation word by word:

**`when { branch 'main' }`** → Only run this stage if the current branch is `main`. Feature branches should NOT auto-deploy to production.

**`sh './deploy.sh'`** → Run a shell script called `deploy.sh` that lives in your repo. This script contains the actual deployment commands — like SSH into a server and run `docker pull` + `docker run`.

### Example deploy.sh:

```bash
#!/bin/bash
# Pull the latest image
docker pull myrepo/my-app:${BUILD_NUMBER}

# Stop old container if running
docker stop my-app-container || true

# Remove old container
docker rm my-app-container || true

# Run new container
docker run -d \
  --name my-app-container \
  -p 8080:8080 \
  myrepo/my-app:${BUILD_NUMBER}

echo "Deployment complete!"
```

---

## Stage 8 — Post Actions

### What is it?

Actions that run AFTER all stages finish — regardless of pass or fail.

### The three main conditions:

| Condition | When it runs |
|-----------|-------------|
| `always` | Every time — pass or fail |
| `success` | Only when pipeline passed |
| `failure` | Only when pipeline failed |

### Code:

```groovy
post {
    success {
        echo "✅ Pipeline PASSED! Image: myrepo/my-app:${env.BUILD_NUMBER}"
        // In real projects: send Slack message, trigger next pipeline, etc.
    }
    failure {
        echo "❌ Pipeline FAILED! Check the red stage above."
        // In real projects: send email alert, create incident ticket, etc.
    }
    always {
        // Clean up Docker image from agent to save disk space
        sh "docker rmi myrepo/my-app:${env.BUILD_NUMBER} || true"
        // Clean up workspace
        cleanWs()
    }
}
```

### Explanation word by word:

**`cleanWs()`** → Jenkins built-in step that deletes the workspace folder after the build. Keeps your agent disk clean. Without this, every build adds more files and disk fills up over time.

**`|| true`** → Even if `docker rmi` fails (image wasn't created because an earlier stage failed), don't fail the pipeline. `|| true` makes the shell command always exit with success.

---

## The Complete Jenkinsfile for Today

This is the full pipeline combining all stages above:

```groovy
pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'myrepo/my-app'
        IMAGE_TAG    = "${env.BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }

        stage('Code Analysis') {
            steps {
                withSonarQubeEnv('SonarQube-Server') {
                    sh 'mvn sonar:sonar -Dsonar.projectKey=my-app'
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh "docker build -t ${DOCKER_IMAGE}:${IMAGE_TAG} ."
            }
        }

        stage('Docker Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh "echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin"
                    sh "docker push ${DOCKER_IMAGE}:${IMAGE_TAG}"
                }
            }
        }

        stage('Deploy') {
            when {
                branch 'main'
            }
            steps {
                sh './deploy.sh'
            }
        }

    }

    post {
        success {
            echo "✅ Pipeline PASSED! Image: ${DOCKER_IMAGE}:${IMAGE_TAG}"
        }
        failure {
            echo "❌ Pipeline FAILED! Check the red stage above."
        }
        always {
            sh "docker rmi ${DOCKER_IMAGE}:${IMAGE_TAG} || true"
            cleanWs()
        }
    }
}
```

---

## HANDS-ON LAB

> Read each step fully before typing. Every command is explained.
> Do steps in order — do not skip.

---

### What We Are Building in This Lab

We will create a **simplified version** of the pipeline above (without SonarQube and Maven since those need extra installation). Instead we will simulate each stage with echo commands so you can see the full pipeline flow working in Jenkins.

After this lab you will see all 6 stages light up green in Jenkins UI.

---

### Where to Do This Lab

Everything happens in TWO places:
1. Your Ubuntu terminal — to create and push files
2. Jenkins browser UI — to run the pipeline

---

### Step 1 — Open Terminal and Go to Your Repo

```bash
cd ~/Devops-lab/Practice-Lab/jenkins
```

This takes you into your existing practice repo folder.

Confirm you are in the right place:

```bash
ls
```

You should see your `Jenkinsfile` listed. If yes, continue.

---

### Step 2 — Switch to Main Branch

```bash
git checkout main
```

You will see:
```
Switched to branch 'main'
```

We always work on `main` for today's lab.

---

### Step 3 — Open Your Jenkinsfile for Editing

```bash
nano Jenkinsfile
```

This opens the file in a text editor inside your terminal.

---

### Step 4 — Select All and Delete Everything

Inside nano:
- Press `Ctrl + K` repeatedly until the file is completely empty
- You should see a blank screen

---

### Step 5 — Type the New Jenkinsfile

Copy and paste this EXACTLY into nano:

```groovy
pipeline {
    agent any

    environment {
        APP_NAME  = 'my-practice-app'
        IMAGE_TAG = "${env.BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                echo "✅ Code checked out successfully"
            }
        }

        stage('Build') {
            steps {
                echo "🔨 Building application: ${APP_NAME}"
                echo "Simulating: mvn clean package -DskipTests"
                sh 'echo BUILD SUCCESS'
            }
        }

        stage('Test') {
            steps {
                echo "🧪 Running unit tests..."
                echo "Simulating: mvn test"
                sh 'echo ALL TESTS PASSED'
            }
        }

        stage('Code Analysis') {
            steps {
                echo "🔍 Running SonarQube code analysis..."
                echo "Simulating: mvn sonar:sonar"
                sh 'echo SONARQUBE SCAN COMPLETE - Quality Gate PASSED'
            }
        }

        stage('Docker Build') {
            steps {
                echo "🐳 Building Docker image: ${APP_NAME}:${IMAGE_TAG}"
                echo "Simulating: docker build -t ${APP_NAME}:${IMAGE_TAG} ."
                sh "echo DOCKER IMAGE BUILT SUCCESSFULLY"
            }
        }

        stage('Docker Push') {
            steps {
                echo "📤 Pushing Docker image to registry..."
                echo "Simulating: docker push ${APP_NAME}:${IMAGE_TAG}"
                sh "echo DOCKER IMAGE PUSHED SUCCESSFULLY"
            }
        }

        stage('Deploy') {
            when {
                branch 'main'
            }
            steps {
                echo "🚀 Deploying ${APP_NAME}:${IMAGE_TAG} to server..."
                sh 'echo DEPLOYMENT COMPLETE'
            }
        }

    }

    post {
        success {
            echo "✅ PIPELINE PASSED! App: ${APP_NAME}, Build: ${IMAGE_TAG}"
        }
        failure {
            echo "❌ PIPELINE FAILED! Check the red stage above."
        }
        always {
            echo "🧹 Cleanup complete. Workspace will be cleared."
            cleanWs()
        }
    }
}
```

---

### Step 6 — Save the File

Press `Ctrl + O` → Press `Enter` → Press `Ctrl + X`

You are back to the terminal.

---

### Step 7 — Verify the File Looks Correct

```bash
cat Jenkinsfile
```

You should see your full pipeline printed on screen. Quickly count — the file should end with exactly:

```groovy
    }

}
```

Two closing braces at the end — one for `post { }` and one for `pipeline { }`.

---

### Step 8 — Push to GitHub

Run these three commands one by one:

```bash
git add Jenkinsfile
```
(Stages the file for commit — tells Git "I want to save this change")

```bash
git commit -m "Day 3 lab: full pipeline with all stages"
```
(Saves the change with a message)

```bash
git push origin main
```
(Uploads to GitHub)

You should see output ending with something like:
```
main -> main
```
That means GitHub received your file.

---

### Step 9 — Go to Jenkins in Your Browser

Open: `http://localhost:8080`

---

### Step 10 — Go to Your Day 1 Job

Click on **`Day1_Declarative_Pipeline`** job.

We are using this existing job because it already points to your GitHub repo and `main` branch. No need to create a new job.

---

### Step 11 — Build Now

Click **Build Now** on the left sidebar.

A new build number appears at the bottom left (e.g. `#3`). Click on it.

---

### Step 12 — Click Console Output

Click **Console Output** on the left sidebar.

---

### Step 13 — What You Should See

You should see each stage run one by one:

```
[Pipeline] stage
[Pipeline] { (Checkout)
✅ Code checked out successfully

[Pipeline] stage
[Pipeline] { (Build)
🔨 Building application: my-practice-app
Simulating: mvn clean package -DskipTests
BUILD SUCCESS

[Pipeline] stage
[Pipeline] { (Test)
🧪 Running unit tests...
ALL TESTS PASSED

[Pipeline] stage
[Pipeline] { (Code Analysis)
🔍 Running SonarQube code analysis...
SONARQUBE SCAN COMPLETE - Quality Gate PASSED

[Pipeline] stage
[Pipeline] { (Docker Build)
🐳 Building Docker image: my-practice-app:3
DOCKER IMAGE BUILT SUCCESSFULLY

[Pipeline] stage
[Pipeline] { (Docker Push)
📤 Pushing Docker image to registry...
DOCKER IMAGE PUSHED SUCCESSFULLY

[Pipeline] stage
[Pipeline] { (Deploy)
🚀 Deploying my-practice-app:3 to server...
DEPLOYMENT COMPLETE

[Pipeline] { (Declarative: Post Actions)
✅ PIPELINE PASSED! App: my-practice-app, Build: 3
🧹 Cleanup complete. Workspace will be cleared.

Finished: SUCCESS
```

---

### Step 14 — Check the Stage View in Jenkins UI

Go back to the job page (click `Day1_Declarative_Pipeline` in the breadcrumb at top).

You will see a visual block diagram showing all stages — each one should be **green**.

This is what Jenkins looks like in real companies — every stage is visible and colour-coded.

---

### Step 15 — Verify the `when` Condition Worked

Look in your console output for the Deploy stage. You should see it ran because you are on `main` branch.

Now let's verify it gets SKIPPED on a non-main branch.

In terminal:

```bash
git checkout feature/test-multibranch
```

Now go to Jenkins → `Day2_Multibranch_lab` → click `feature/test-multibranch` → **Build Now**.

Check console output — you will see:

```
Stage 'Deploy' skipped due to when condition
```

**This proves the `when { branch 'main' }` condition works correctly.**

Go back to main:

```bash
git checkout main
```

---

## ✅ Lab Checklist — Confirm All of These Before Moving to Day 4

- [ ] I replaced my Jenkinsfile with the new 7-stage pipeline
- [ ] I pushed it to GitHub with `git push origin main`
- [ ] I ran the build in Jenkins and all stages showed green
- [ ] I saw each stage's echo message in Console Output
- [ ] I saw `Finished: SUCCESS` at the bottom
- [ ] I saw `Deploy` stage was SKIPPED on the feature branch
- [ ] I saw `Deploy` stage RAN on the main branch

---

## Troubleshooting — If Something Goes Wrong

### Problem: "unexpected token" error
**Cause:** A bracket `{` or `}` is missing or extra.
**Fix:** Run `cat -n Jenkinsfile` and paste output here. I will find the exact line.

### Problem: "cleanWs() not found" error
**Cause:** Workspace Cleanup plugin not installed in Jenkins.
**Fix:** Go to Manage Jenkins → Plugins → Available → search `Workspace Cleanup` → install → restart Jenkins → run again.

### Problem: Deploy stage did not skip on feature branch
**Cause:** Your feature branch Jenkinsfile is the OLD one (without the `when` block).
**Fix:**
```bash
git checkout feature/test-multibranch
nano Jenkinsfile
```
Add the same pipeline content, save, push. Then rebuild.

### Problem: `git push` asks for username/password
**Fix:**
```bash
git config --global credential.helper store
git push origin main
```
Enter your GitHub username and your **Personal Access Token** (not your GitHub password) when asked. It will be saved for future pushes.

---

## Key Takeaways from Day 3

| Stage | What it Does | Key Command/Step |
|-------|-------------|-----------------|
| Checkout | Fetch code from Git | `checkout scm` |
| Build | Compile code into artifact | `mvn clean package` |
| Test | Run unit tests | `mvn test` |
| Code Analysis | Check code quality | `mvn sonar:sonar` |
| Quality Gate | Pass/fail based on scan | `waitForQualityGate` |
| Docker Build | Create Docker image | `docker build -t name:tag .` |
| Docker Push | Upload image to registry | `docker push name:tag` |
| Deploy | Run image on server | `./deploy.sh` |
| post success | Runs if all passed | Notification/confirmation |
| post failure | Runs if any stage failed | Alert/ticket creation |
| post always | Runs no matter what | Cleanup workspace/images |

---

*Next: Day 4 — Agents & Nodes (Master-Agent Architecture, Docker Agent, Kubernetes Agent)*
