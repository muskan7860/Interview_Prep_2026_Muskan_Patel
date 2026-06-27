# Jenkins Day 1 — Pipeline Types
## Theory + Labs: Declarative vs Scripted Pipeline, Jenkinsfile in SCM

---

## Before We Start — What is Jenkins and Why Does It Exist?

### The Problem (Story First)

Imagine you are a chef in a restaurant. Every time you make a dish, you:
1. Chop vegetables
2. Cook them
3. Plate the dish
4. Serve it

Now imagine you have 50 dishes to make every day. If you do each step manually from memory, you will forget steps, make mistakes, and become slow.

**Solution?** Write a **recipe card**. Follow it every time. Same result. No mistakes. Fast.

That is exactly what Jenkins does for software.

---

### What is Jenkins?

Jenkins is a **CI/CD automation server**.

- **CI = Continuous Integration** → Every time a developer pushes code, Jenkins automatically builds and tests it.
- **CD = Continuous Delivery/Deployment** → After testing, Jenkins automatically delivers or deploys the code to servers.

Jenkins is the chef. Your **Jenkinsfile** is the recipe card.

Without Jenkins:
- Developer pushes code ✓
- Someone manually runs tests ✗ (slow, error-prone)
- Someone manually builds Docker image ✗
- Someone manually deploys to server ✗

With Jenkins:
- Developer pushes code ✓
- Jenkins **automatically** does all the rest ✓

---

### What is a Pipeline?

A **pipeline** in Jenkins is a series of automated steps executed in order.

Think of it like an assembly line in a car factory:
- Station 1 → Fit the engine
- Station 2 → Attach the doors
- Station 3 → Paint the car
- Station 4 → Quality check
- Station 5 → Deliver to showroom

Each station = one **Stage** in Jenkins pipeline.

If any station fails, the line stops. Jenkins works the same way.

---

## Topic 1 — Declarative Pipeline

### What is it?

Declarative pipeline is the **modern, recommended way** to write Jenkins pipelines.

You **declare** what you want Jenkins to do — like filling a structured form.

It has a fixed structure. You cannot break the structure. Jenkins enforces rules.

### Why was it created?

Before declarative pipeline, people used scripted pipeline (we'll cover that next). Scripted was too flexible — it was full Groovy code. Non-developers could not read it. Mistakes were hard to catch.

Declarative pipeline was introduced to make pipelines:
- **Readable** — anyone can understand it
- **Structured** — clear sections for each part
- **Safe** — Jenkins validates the structure before running

---

### Anatomy of a Declarative Pipeline

```groovy
pipeline {
    agent any

    environment {
        APP_NAME = 'my-app'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
```

---

### Line-by-Line Explanation (Every Word Matters)

#### `pipeline { }`
- This is the **root block**. Everything lives inside this.
- It tells Jenkins: "Hey, this entire block is one pipeline."
- Without this, Jenkins won't recognize the file as a valid declarative pipeline.

---

#### `agent any`
- **agent** = "Which machine should run this pipeline?"
- **any** = "Run on any available Jenkins agent (worker machine). I don't care which one."
- Why? Because Jenkins has one master server and multiple worker agents. The master distributes work. We need to tell it where to run.
- If you write `agent none`, you must define agent inside each stage separately.

---

#### `environment { APP_NAME = 'my-app' }`
- This block defines **environment variables** available throughout the entire pipeline.
- Think of it like declaring global constants at the top of your code.
- Why? Because if your app name changes, you change it in ONE place — not 20 different steps.
- `APP_NAME` can be used later as `${APP_NAME}` in shell commands.

---

#### `stages { }`
- This is where all your **pipeline stages** live.
- A stage = one logical group of steps.
- Why stages? Because they appear as **visual blocks** in Jenkins UI. You can see exactly which stage passed and which failed.

---

#### `stage('Checkout') { }`
- A single stage. The name `'Checkout'` is what Jenkins displays in the UI.
- You can name it anything — `'Build App'`, `'Run Tests'`, `'Deploy to Prod'`.
- Why named? So when the pipeline fails, you immediately know **which stage** failed without reading logs.

---

#### `steps { }`
- Inside each stage, `steps` contains the actual commands to run.
- Steps are the smallest unit of work.

---

#### `checkout scm`
- `checkout` = Jenkins built-in step to pull code from source control.
- `scm` = "Source Control Manager" — tells Jenkins to use whatever SCM is configured for this job (usually Git).
- Why? Because before building anything, you need the latest code. This step fetches it.

---

#### `sh 'mvn clean package'`
- `sh` = "Run this as a shell command on the agent."
- `mvn clean package` = Maven command to compile and package the Java app.
- Why `sh`? Because Jenkins runs on Linux agents. `sh` executes bash commands.
- On Windows, you would use `bat` instead.

---

#### `post { success { } failure { } }`
- `post` = "After all stages finish, run this block."
- `success` = "Run these steps ONLY if the pipeline succeeded."
- `failure` = "Run these steps ONLY if the pipeline failed."
- Why? Because you might want to:
  - On success → Send Slack message "Deployment done!"
  - On failure → Send email "Build broke! Check it!"
- Other options: `always` (runs regardless), `unstable` (runs if tests failed but build passed)

---

## Topic 2 — Scripted Pipeline

### What is it?

Scripted pipeline is the **older, more powerful** way. It is written in **full Groovy code**.

No fixed structure. You have complete freedom. But that freedom comes with complexity.

### When is it used?

- Complex custom logic that declarative cannot handle
- Dynamic stage generation (stages created based on conditions at runtime)
- Legacy pipelines that were written before declarative existed

### Important for interviews:
Most companies today write **declarative pipelines**. But interviewers ask about scripted to check if you understand the difference.

---

### Example — Scripted Pipeline

```groovy
node {
    stage('Checkout') {
        checkout scm
    }

    stage('Build') {
        sh 'mvn clean package'
    }

    stage('Test') {
        try {
            sh 'mvn test'
        } catch (Exception e) {
            currentBuild.result = 'FAILURE'
            throw e
        }
    }

    stage('Deploy') {
        if (env.BRANCH_NAME == 'main') {
            sh './deploy.sh'
        } else {
            echo 'Not main branch. Skipping deploy.'
        }
    }
}
```

---

### Line-by-Line Explanation

#### `node { }`
- The root block in scripted pipeline.
- `node` = "Run this on any available Jenkins agent node."
- Same concept as `agent any` in declarative, but written differently.
- You can also write `node('linux') { }` to run on an agent with the label 'linux'.

---

#### `stage('Checkout') { }`
- Same concept as declarative. Stages group steps and show in UI.
- But notice — no `steps { }` block needed inside. In scripted, you put commands directly.

---

#### `try { } catch (Exception e) { }`
- This is full **Groovy exception handling**.
- In declarative, you cannot write try-catch inside `steps`. In scripted, you can.
- Why? If a test fails, you want to catch the error, mark the build as FAILURE, and still run cleanup steps.

---

#### `if (env.BRANCH_NAME == 'main') { }`
- Full Groovy `if` condition.
- `env.BRANCH_NAME` = environment variable Jenkins sets automatically, containing the current Git branch name.
- Why? Deploy only when the code is on the `main` branch. Feature branches should NOT auto-deploy to production.

---

## Declarative vs Scripted — Side-by-Side Comparison

| Feature | Declarative | Scripted |
|---------|-------------|---------|
| Syntax | Structured (form-like) | Full Groovy code |
| Ease of reading | Easy — anyone can read | Hard — need Groovy knowledge |
| Flexibility | Limited | Full flexibility |
| Error validation | Jenkins validates before run | Errors found only at runtime |
| `post` block | Built-in support | Must use try-finally manually |
| Recommended today? | ✅ YES | Only for complex cases |
| `pipeline { }` block | Required | NOT used — uses `node { }` |

---

## Topic 3 — Jenkinsfile in SCM (Source Control Management)

### What is Jenkinsfile?

A **Jenkinsfile** is just a text file that contains your pipeline code. It lives in your Git repository.

The file is literally named `Jenkinsfile` — no extension.

### Why store it in SCM (Git)?

This concept is called **"Pipeline as Code"**.

Before this existed, Jenkins pipelines were stored **inside Jenkins UI** — you typed your pipeline in a text box on the Jenkins server. Problems:

1. If Jenkins server crashed → pipeline lost
2. No version history → who changed what, when?
3. No code review → anyone could change the pipeline silently
4. Pipeline not linked to the code it builds → inconsistency

With Jenkinsfile in Git:
1. ✅ Pipeline version-controlled alongside application code
2. ✅ Pull Request review for pipeline changes
3. ✅ If Jenkins crashes, restore by re-reading from Git
4. ✅ Different branches can have different Jenkinsfiles

---

### How Jenkins reads the Jenkinsfile

When you create a Jenkins job and point it to a Git repo, Jenkins:
1. Clones the repo
2. Looks for a file named `Jenkinsfile` at the root
3. Reads and executes it

```
my-app/
├── src/
│   └── main/java/...
├── pom.xml
├── Dockerfile
└── Jenkinsfile          ← Jenkins reads this
```

---

### Real Example — Jenkinsfile for a Java App

```groovy
pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'myrepo/my-app'
        DOCKER_TAG   = "${env.BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                // Jenkins checks out the code from the configured Git repo
                checkout scm
            }
        }

        stage('Build') {
            steps {
                // Compile and package the Java app using Maven
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Unit Test') {
            steps {
                // Run unit tests
                sh 'mvn test'
            }
            post {
                always {
                    // Publish test results to Jenkins UI regardless of pass/fail
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }

        stage('Docker Build') {
            steps {
                // Build Docker image. -t = tag the image with name:buildnumber
                sh "docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} ."
            }
        }

        stage('Docker Push') {
            steps {
                // withCredentials safely injects Docker Hub login — never hardcode passwords
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh "echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin"
                    sh "docker push ${DOCKER_IMAGE}:${DOCKER_TAG}"
                }
            }
        }

        stage('Deploy') {
            when {
                // Only deploy when the branch is 'main'
                branch 'main'
            }
            steps {
                sh './deploy.sh'
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline SUCCESS — Image: ${DOCKER_IMAGE}:${DOCKER_TAG}"
        }
        failure {
            echo "❌ Pipeline FAILED — Check logs above"
        }
        always {
            // Clean up Docker images from the agent to free disk space
            sh "docker rmi ${DOCKER_IMAGE}:${DOCKER_TAG} || true"
        }
    }
}
```

---

### Explanation of New Lines in This File

#### `DOCKER_TAG = "${env.BUILD_NUMBER}"`
- `env.BUILD_NUMBER` = Jenkins automatically assigns a build number to every run (1, 2, 3...).
- We use it as the Docker image tag so each build creates a uniquely tagged image.
- This way you can roll back to any previous image using its build number.

---

#### `sh 'mvn clean package -DskipTests'`
- `-DskipTests` = Skip running tests during the build stage.
- Why? We have a separate `Unit Test` stage. We don't want tests to run twice.
- `clean` = Delete previous build output (the `target/` folder) before building fresh.
- `package` = Compile + create JAR/WAR file.

---

#### `junit 'target/surefire-reports/*.xml'`
- `junit` = Jenkins built-in step that reads test result XML files.
- Maven stores test results in `target/surefire-reports/` as XML.
- After reading these, Jenkins shows a **test trend graph** in the UI — green/red over time.
- `always` means even if tests fail, still publish the results so you can see which tests failed.

---

#### `when { branch 'main' }`
- `when` = Condition block — run this stage only if condition is true.
- `branch 'main'` = Only run if current Git branch is `main`.
- Why? You don't want every feature branch to deploy to production automatically.

---

#### `sh "docker rmi ${DOCKER_IMAGE}:${DOCKER_TAG} || true"`
- `docker rmi` = Remove Docker image from the agent machine.
- `|| true` = "Even if this command fails, don't fail the pipeline."
- Why `|| true`? If the image was never built (earlier stage failed), `rmi` would error out. `|| true` prevents that from failing the post block.

---

## Lab 1 — Your First Declarative Pipeline

### Goal
Create a simple Jenkins pipeline that runs on your local Jenkins and prints messages for each stage.

### Prerequisites
- Jenkins running locally (or on a VM)
- A GitHub repository you own

### Step 1 — Create Jenkinsfile in your repo

Create a file named `Jenkinsfile` (no extension) at the root of your repo:

```groovy
pipeline {
    agent any

    stages {
        stage('Hello') {
            steps {
                echo 'Stage 1: Hello from Jenkins!'
            }
        }

        stage('Who Am I') {
            steps {
                // This runs a shell command and prints the current Linux user
                sh 'whoami'
                // This prints the current directory Jenkins is working in
                sh 'pwd'
                // This lists files in the workspace
                sh 'ls -la'
            }
        }

        stage('Environment Check') {
            steps {
                // Print Jenkins-provided environment variables
                echo "Build Number: ${env.BUILD_NUMBER}"
                echo "Job Name: ${env.JOB_NAME}"
                echo "Workspace: ${env.WORKSPACE}"
                echo "Git Branch: ${env.GIT_BRANCH}"
            }
        }
    }

    post {
        success {
            echo 'All stages passed! Great job!'
        }
        failure {
            echo 'Something went wrong. Check the stage that turned red.'
        }
        always {
            echo 'This always runs — cleanup or notification goes here.'
        }
    }
}
```

### Step 2 — Push to GitHub

```bash
git add Jenkinsfile
git commit -m "Add initial Jenkinsfile"
git push origin main
```

### Step 3 — Create Jenkins Job

1. Open Jenkins UI → Click **New Item**
2. Enter job name: `day1-pipeline-test`
3. Select **Pipeline** → Click **OK**
4. Under **Pipeline** section:
   - Definition: **Pipeline script from SCM**
   - SCM: **Git**
   - Repository URL: your GitHub repo URL
   - Branch: `*/main`
   - Script Path: `Jenkinsfile`
5. Click **Save** → Click **Build Now**

### Step 4 — Observe the Output

In Jenkins UI:
- Each stage appears as a coloured block (green = pass, red = fail)
- Click any stage block → See its console output
- You will see your echo messages and shell output

---

## Lab 2 — Scripted Pipeline Comparison

Create a second Jenkins job using scripted syntax for the same steps.

### Create file `Jenkinsfile.scripted` in your repo:

```groovy
node {
    stage('Hello') {
        echo 'Stage 1: Hello from Scripted Pipeline!'
    }

    stage('Who Am I') {
        sh 'whoami'
        sh 'pwd'
    }

    stage('Environment Check') {
        echo "Build Number: ${env.BUILD_NUMBER}"
        echo "Job Name: ${env.JOB_NAME}"
    }
}
```

### In Jenkins, create a new Pipeline job:
- Script Path: `Jenkinsfile.scripted`

**Observe:** Both do the same thing, but look different. Declarative is cleaner and has the nice stage visualization.

---

## Key Concepts Summary

| Concept | What it means | Why it matters |
|---------|--------------|----------------|
| Pipeline | Series of automated steps | Replaces manual work |
| Declarative | Structured, recommended format | Easy to read, validated |
| Scripted | Groovy code, full flexibility | For complex custom logic |
| Jenkinsfile | Pipeline file stored in Git | Version control, code review |
| Pipeline as Code | Pipeline defined in code, not UI | Reproducible, auditable |
| `agent any` | Run on any available worker | Distributes load |
| `stages/stage` | Logical grouping of steps | Visible in Jenkins UI |
| `post` | Runs after all stages | Notifications, cleanup |
| `when` | Conditional stage execution | Control what runs when |
| `checkout scm` | Fetch code from Git | Always first step |

---

## Common Mistakes to Avoid

1. **Forgetting `steps { }` in declarative pipeline** — Each stage needs a `steps` block.
2. **Hardcoding passwords in Jenkinsfile** — Always use `withCredentials`. (Day 5 covers this.)
3. **Missing `|| true` in cleanup steps** — Cleanup in `post` should never fail the pipeline.
4. **Not committing Jenkinsfile to Git** — If it lives only in Jenkins UI, you lose it when Jenkins crashes.
5. **Using `sh` on Windows agents** — Windows needs `bat`. Always know your agent OS.

---

*Next: Day 1 — Interview Q&A File for Pipeline Types*
