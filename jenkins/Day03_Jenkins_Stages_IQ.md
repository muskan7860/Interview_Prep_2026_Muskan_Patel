# Jenkins Day 3 — Interview Q&A
## Stages & Steps: Checkout, Build, Test, SonarQube, Docker, Deploy, Post Actions

> All answers written to be spoken aloud naturally.
> Senior-level phrasing. Use your Atos project as the example wherever possible.

---

## Question 1
**"Walk me through a typical Jenkins pipeline. What stages would it have?"**

### How to Answer:

"A typical Jenkins pipeline in a microservices environment follows this sequence:

First, **Checkout** — Jenkins fetches the latest code from the Git repository into the workspace. This is always the first stage because nothing else can happen without the code.

Second, **Build** — the code is compiled and packaged into a deployable artifact. For Java projects, that's `mvn clean package`. This produces a JAR or WAR file.

Third, **Test** — automated unit tests run. If tests fail here, the pipeline stops. Broken code never moves forward.

Fourth, **Code Analysis** — a tool like SonarQube scans the code for bugs, vulnerabilities, and code smells. A Quality Gate then determines if the code meets the team's standards.

Fifth, **Docker Build** — the artifact is packaged into a Docker image along with all its dependencies.

Sixth, **Docker Push** — the image is pushed to a registry like Docker Hub or ECR so it's accessible to the deployment environment.

Seventh, **Deploy** — the image is pulled on the target server and the application is started. In my Atos project, this was controlled by a `deploy.sh` script that ran on the EC2 instance.

Finally, **Post Actions** — cleanup happens, workspace is cleared, and the team is notified of success or failure via Slack or email."

---

## Question 2
**"What is `checkout scm` and why is it the first stage?"**

### How to Answer:

"`checkout scm` is a Jenkins built-in step that fetches the source code from the configured version control system — in most cases, Git.

The `scm` part means 'use whatever SCM this job is already configured to use.' Jenkins knows the repo URL, branch, and credentials from the job configuration, so you don't need to repeat them inside the Jenkinsfile.

It must be the first stage because the Jenkins workspace starts completely empty. There's no code to build, test, or analyze until checkout runs. Every subsequent stage depends on the files that checkout brings in.

In practice, when you use `agent any` in a declarative pipeline, Jenkins actually runs an implicit checkout automatically before your first stage. But I prefer writing it explicitly — it makes the pipeline transparent and easier to debug if something goes wrong with the fetch."

---

## Question 3
**"What is the difference between `mvn clean package` and `mvn clean package -DskipTests`?"**

### How to Answer:

"Both commands build the application, but they differ in whether tests run during the build phase.

`mvn clean package` compiles the code, runs all unit tests, and then packages it into a JAR or WAR file. If any test fails, the build fails.

`mvn clean package -DskipTests` does the same compile and package but skips running tests entirely. The `-D` flag sets a Maven property, and `skipTests=true` tells Maven to not execute tests.

I use `-DskipTests` in the Build stage when I have a dedicated separate Test stage. Running tests twice — once in Build and once in Test — wastes time and can cause confusing double failures. Build stage should only build. Test stage should only test. Separation of concerns."

---

## Question 4
**"Why do you put `junit` inside `post { always { } }` in the Test stage?"**

### How to Answer:

"The `junit` step reads the XML test result files that Maven generates and publishes them as a visual report in Jenkins — showing which tests passed and which failed, with trend graphs over time.

The reason I put it inside `post { always { } }` rather than directly in `steps { }` is important: if tests fail, Jenkins marks the stage as failed and would normally skip anything remaining in `steps`. But by putting `junit` in `always`, it runs even when tests fail.

That's critical because when tests fail is exactly when you need to see the report. You need to know WHICH tests failed, not just that something failed. If I put it in `success`, I'd never see the report when I need it most.

`always` means: no matter what the stage result is, always run this step."

---

## Question 5
**"What is SonarQube and what is a Quality Gate?"**

### How to Answer:

"SonarQube is a static code analysis tool that scans your source code without running it. It identifies bugs — actual logic errors that could cause failures in production. It finds security vulnerabilities — code patterns that could be exploited. It detects code smells — maintainability issues that make code harder to change over time. And it measures test coverage — what percentage of your code is actually tested.

A Quality Gate is a set of pass/fail conditions defined in SonarQube — for example, 'code coverage must be above 80%' or 'zero critical vulnerabilities allowed.' After the scan completes, SonarQube evaluates these conditions and returns either PASSED or FAILED.

In Jenkins, the `waitForQualityGate` step pauses the pipeline and waits for this verdict. If FAILED with `abortPipeline: true`, Jenkins stops the pipeline immediately. This prevents deploying code that has known security vulnerabilities or falls below the team's quality standard.

In my understanding of real projects, the Quality Gate is the gatekeeper between development and deployment."

---

## Question 6
**"Why is `withCredentials` used for Docker login? Why not just write the username and password directly?"**

### How to Answer:

"Hardcoding credentials in a Jenkinsfile is a serious security risk. The Jenkinsfile lives in a Git repository — which could be public, or could be accessed by many people across the organization. If your Docker Hub password is in that file, anyone with repo access can see it.

`withCredentials` solves this by storing the credentials securely inside Jenkins itself — encrypted on the Jenkins server — and injecting them as temporary environment variables only during that specific step's execution. They're never written to disk, never appear in Git, and Jenkins actively masks them in console logs so they don't appear even in build output.

The pattern I use is:
```groovy
withCredentials([usernamePassword(
    credentialsId: 'dockerhub-creds',
    usernameVariable: 'DOCKER_USER',
    passwordVariable: 'DOCKER_PASS'
)]) {
    sh "echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin"
}
```

`credentialsId` is the ID I assigned when saving the credential in Jenkins. The username and password are mapped to variables I define. `--password-stdin` passes the password through a pipe rather than as a command-line argument — because arguments appear in the process list and can be seen by other users on the same machine."

---

## Question 7
**"What does the `when` directive do? Give a real use case."**

### How to Answer:

"The `when` directive adds a condition to a stage — that stage only runs if the condition is true. If false, Jenkins skips the stage and moves to the next one.

The most common use case is controlling deployments. In a real project, you only want to deploy to production when code is on the `main` branch. Feature branches should run tests and code analysis, but should NOT automatically deploy to production.

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

With this, when a developer pushes to `feature/payment-retry`, the Deploy stage is skipped automatically. Only when code is merged into `main` does Jenkins actually deploy.

Other useful `when` conditions I use:
- `when { environment name: 'ENV', value: 'prod' }` — deploy only to production environment
- `when { expression { return params.DEPLOY == true } }` — deploy only if a build parameter is checked
- `when { not { branch 'main' } }` — run only on non-main branches, like a 'notify feature branch author' step"

---

## Question 8
**"What is the difference between `post { always }`, `post { success }`, and `post { failure }`?"**

### How to Answer:

"All three are conditions inside the `post` block that control when certain actions run after the pipeline completes.

`always` runs regardless of outcome — whether the pipeline passed, failed, or was aborted. I use this for cleanup tasks like deleting Docker images from the agent or clearing the workspace with `cleanWs()`. Cleanup should always happen, not just on success.

`success` runs only if the pipeline completed successfully — all stages passed. I use this for positive notifications like a Slack message saying 'Deployment successful, version X is live' or triggering a downstream pipeline.

`failure` runs only if the pipeline failed — at least one stage had an error. I use this for alerts — sending an email to the team, creating an incident ticket in ServiceNow, or paging the on-call engineer.

In my Atos project, the `post` block was structured so that on failure, an automated alert was raised and the failed build's logs were attached. On success, the deployment confirmation was sent to the product team. And `always` ensured workspace cleanup so the agent disk never filled up."

---

## Question 9
**"What does `cleanWs()` do and why is it important?"**

### How to Answer:

"`cleanWs()` is a Jenkins step from the Workspace Cleanup plugin that deletes the entire workspace directory after the build finishes.

Without it, every build leaves behind files — compiled classes, downloaded dependencies, Docker layers, log files. Over time this fills up the agent's disk. When the disk fills, builds start failing with 'no space left on device' errors. In a busy project with 50 builds a day, this can happen surprisingly fast.

By putting `cleanWs()` in `post { always { } }`, the workspace is cleaned after every single build regardless of success or failure. The next build always starts with a fresh, empty workspace.

There's one tradeoff: cleaning the workspace means Maven has to re-download all dependencies on the next build, which adds time. Teams often solve this by mounting a Maven cache directory from outside the workspace so dependencies persist between builds even when the workspace is cleaned."

---

## Question 10
**"What happens if a stage fails in a Jenkins pipeline? Does the rest of the pipeline still run?"**

### How to Answer:

"By default, if any stage fails in a Jenkins pipeline, the pipeline stops immediately. Subsequent stages are skipped and marked as 'Not Built' in the UI.

The only exception is the `post` block — it always runs regardless of whether stages failed, because cleanup and notifications must happen even on failure.

However, you can override this default behavior. If you want a stage to continue even when it fails, you can use:
```groovy
stage('Optional Stage') {
    steps {
        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
            sh './optional-check.sh'
        }
    }
}
```

This marks the stage as failed in the UI but allows the pipeline to continue. I use this for non-critical steps like optional performance tests that shouldn't block a deployment."

---

## Quick-Fire Round

**Q: What does `checkout scm` do?**
A: Fetches code from the Git repository configured for this job into the workspace.

**Q: What does `-DskipTests` do in Maven?**
A: Skips running unit tests during the build phase.

**Q: What is SonarQube?**
A: A static code analysis tool that checks for bugs, vulnerabilities, and code quality issues.

**Q: What does `waitForQualityGate abortPipeline: true` do?**
A: Pauses the pipeline waiting for SonarQube's verdict; stops the pipeline if Quality Gate fails.

**Q: Why use `withCredentials` instead of hardcoding passwords?**
A: Credentials stored in Jenkins are encrypted and never appear in Git or console logs.

**Q: What does `when { branch 'main' }` do?**
A: Runs that stage only when the current branch is `main`; skips it for all other branches.

**Q: What does `post { always { } }` do?**
A: Runs the steps inside regardless of whether the pipeline passed or failed.

**Q: What does `cleanWs()` do?**
A: Deletes the workspace directory after the build to free up agent disk space.

**Q: What does `|| true` after a shell command mean?**
A: Even if the command fails, treat it as success — prevents cleanup steps from failing the pipeline.

**Q: What is the IMAGE_TAG set to in most pipelines?**
A: `env.BUILD_NUMBER` — the Jenkins build number, making each Docker image uniquely tagged.

---

## Scenario Questions

### Scenario 1
**"Your pipeline deploys to production on every push, including feature branches. How do you fix this?"**

### Answer:
"Add a `when { branch 'main' }` condition to the Deploy stage. This ensures the deployment only runs when code is on the `main` branch. All other branches — feature, bugfix, hotfix — will run all other stages like build, test, and code analysis, but the Deploy stage will be automatically skipped. One line of configuration prevents accidental production deployments from in-progress feature branches."

---

### Scenario 2
**"Your Jenkins agent disk keeps filling up and builds are failing with 'no space left on device'. What is the fix?"**

### Answer:
"Two things. First, I add `cleanWs()` inside `post { always { } }` so the workspace is deleted after every build — that handles accumulated source code, compiled files, and downloaded dependencies.

Second, I add Docker image cleanup: `sh 'docker rmi image-name:tag || true'` also in the `post { always { } }` block, because Docker images are large and accumulate fast if not removed after each build.

The `|| true` at the end of the docker rmi command is important — if the image was never built because an earlier stage failed, `docker rmi` would error, which would cascade and fail the post block. `|| true` prevents that."

---

*Next: Day 4 — Agents & Nodes (Master-Agent Architecture, Docker Agent, Kubernetes Agent, Labels)*
