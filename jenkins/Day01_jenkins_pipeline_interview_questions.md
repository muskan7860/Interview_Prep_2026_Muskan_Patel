# Jenkins Day 1 — Interview Q&A
## Pipeline Types: Declarative vs Scripted, Jenkinsfile in SCM

> These questions are asked in real DevOps interviews at all levels.
> Each answer is written to be spoken aloud naturally.
> Senior-level phrasing is used so you sound experienced, not bookish.

---

## Question 1
**"What is Jenkins and why do we use it?"**

### How to Answer (Script It):

"Jenkins is an open-source automation server used for CI/CD — Continuous Integration and Continuous Deployment. In my experience, the core problem Jenkins solves is removing manual steps from the software delivery process.

Before CI/CD tools, a developer would push code, and someone would manually run the build, run tests, create an artifact, and deploy it. That process was slow and error-prone. Jenkins automates that entire chain — from the moment code is pushed to Git until it reaches the target environment.

In my project at Atos, we used Jenkins to automate our build pipeline. Every push to the main branch triggered a Jenkins job that would build our Java application using Maven, build a Docker image, push it to our registry, and deploy it to EC2 via a deploy script. Zero manual steps after the push."

---

## Question 2
**"What is a Jenkins Pipeline?"**

### How to Answer:

"A Jenkins Pipeline is a series of automated steps defined as code, which Jenkins executes in sequence to build, test, and deploy an application.

Think of it as an assembly line. Each station on the line does one job. If any station fails, the line stops and alerts the team. In Jenkins terms, each station is called a Stage — like Checkout, Build, Test, Deploy.

Before pipelines existed, Jenkins used Freestyle jobs where you configured steps through a UI. The problem was those configurations weren't version-controlled. Pipelines solve that by defining everything in a Jenkinsfile that lives in Git alongside the application code."

---

## Question 3
**"What is the difference between Declarative and Scripted pipeline?"**

### How to Answer:

"Both are ways to write Jenkins pipelines, but they differ in structure and flexibility.

Declarative pipeline is the recommended modern approach. It has a strict, predefined structure — you start with a `pipeline` block, define `agent`, `stages`, `steps`, and `post`. Jenkins validates this structure before execution, so mistakes are caught early. It is readable even by someone who doesn't know Groovy.

Scripted pipeline is the older approach, written entirely in Groovy code. It starts with a `node` block and gives you full programming freedom — loops, try-catch, dynamic stage generation. But it requires Groovy knowledge and is harder to read.

In practice, I always use declarative pipeline. I use scripted syntax only inside a `script { }` block within declarative when I need complex Groovy logic for a specific step.

The rule I follow: default to declarative, drop into scripted Groovy only when declarative cannot express what I need."

---

## Question 4
**"What is a Jenkinsfile and why should it be stored in SCM?"**

### How to Answer:

"A Jenkinsfile is a text file that contains the pipeline definition. It lives at the root of your Git repository, and Jenkins reads it automatically when a build is triggered.

The reason we store it in SCM — usually Git — comes down to four benefits:

First, version control. Every change to the pipeline is tracked with a commit. You know who changed what and when. You can revert if something breaks.

Second, code review. Pipeline changes go through the same pull request process as application code. No silent changes to production pipelines.

Third, resilience. If the Jenkins server crashes and you rebuild it, you don't lose your pipelines — they're all in Git.

Fourth, consistency. The pipeline for a feature branch can be different from the pipeline for the main branch, and that's all managed in the same repo.

This concept is called 'Pipeline as Code', and it's a fundamental DevOps best practice."

---

## Question 5
**"What is `agent any` in a Jenkins pipeline? What are the other agent options?"**

### How to Answer:

"In Jenkins, the master server manages job scheduling and the UI, but it doesn't run builds itself. Builds run on agents — worker machines. The `agent` directive tells Jenkins which agent to use for the pipeline.

`agent any` means: run this pipeline on any available agent. Jenkins picks whichever agent is free.

Other options:
- `agent none` — No global agent. You define agent inside each stage individually. Useful when different stages need different environments.
- `agent { label 'linux' }` — Run on an agent that has the label 'linux' assigned in Jenkins.
- `agent { docker { image 'maven:3.9' } }` — Run inside a Docker container using that image.
- `agent { kubernetes { } }` — Spin up a Kubernetes pod as the agent.

In my projects, I use `agent any` for simple pipelines and `agent { label 'docker-agents' }` when I want to restrict builds to agents that have Docker installed."

---

## Question 6
**"What is the `post` block in Jenkins? What are its conditions?"**

### How to Answer:

"The `post` block in a declarative pipeline runs after all stages complete. It's used for actions like notifications, cleanup, and reporting.

It supports several conditions:
- `always` — Runs regardless of whether the pipeline succeeded or failed. I use this for cleanup steps like removing Docker images or workspace cleanup.
- `success` — Runs only if the pipeline passed. I use this to send a Slack message or trigger the next job.
- `failure` — Runs only if the pipeline failed. I use this to send an alert email or page the on-call person.
- `unstable` — Runs if the build succeeded but tests had failures. Useful for sending a separate 'tests degraded' alert.
- `changed` — Runs if the result of this build is different from the previous build. For example, if the build went from pass to fail or vice versa.

In my Atos project, the `post` block was critical. On failure, we triggered a ServiceNow incident creation automatically using a REST API call."

---

## Question 7
**"What is `checkout scm` and what does `scm` mean?"**

### How to Answer:

"`checkout scm` is a Jenkins built-in step that checks out the source code from the source control management system configured for the job.

`scm` here means 'use whatever SCM this job is configured to use' — which is usually Git. Jenkins has already been told the repo URL, branch, and credentials when you set up the job. `checkout scm` uses all of that configuration automatically.

The alternative is explicitly specifying the Git repo:
```groovy
git url: 'https://github.com/org/repo.git', branch: 'main'
```

I prefer `checkout scm` because it uses the job's own configuration and works correctly in Multibranch pipelines — it automatically checks out the correct branch that triggered the build."

---

## Question 8
**"What is the `when` directive in Jenkins? Give an example."**

### How to Answer:

"The `when` directive controls whether a stage should run based on a condition. If the condition is false, Jenkins skips that stage entirely.

The most common use case is controlling deployments. You only want to deploy to production when the build is from the `main` branch. For any feature branch, you skip the deploy stage.

Example:
```groovy
stage('Deploy to Production') {
    when {
        branch 'main'
    }
    steps {
        sh './deploy.sh'
    }
}
```

Other `when` conditions I use:
- `when { environment name: 'DEPLOY_TO', value: 'prod' }` — Based on an environment variable
- `when { expression { return params.RUN_TESTS == true } }` — Based on a build parameter
- `when { not { branch 'main' } }` — Negation: run this stage on every branch EXCEPT main

`when` is powerful because it lets you have one Jenkinsfile that handles multiple scenarios without duplicating code."

---

## Question 9
**"How is `environment { }` block used in Jenkins?"**

### How to Answer:

"The `environment` block defines environment variables that are available throughout the pipeline. I use it to centralize values that are referenced in multiple places.

For example:
```groovy
environment {
    APP_NAME   = 'payment-service'
    DOCKER_REG = 'registry.mycompany.com'
    IMAGE_TAG  = "${env.BUILD_NUMBER}"
}
```

Now `${APP_NAME}` can be used in shell commands anywhere in the pipeline.

The benefit is maintainability — if the registry URL changes, I update it in one line in the `environment` block. I don't go hunting through 10 different `sh` commands.

You can also define credentials as environment variables using the `credentials()` helper:
```groovy
environment {
    SONAR_TOKEN = credentials('sonar-token-id')
}
```
This safely injects the secret value without hardcoding it."

---

## Question 10
**"What is Pipeline as Code? Why is it a DevOps best practice?"**

### How to Answer:

"Pipeline as Code means defining your CI/CD pipeline in a file that is version-controlled in your source code repository, rather than configuring it through a GUI.

In Jenkins, this file is the Jenkinsfile.

The reason it's considered a best practice comes down to the same principles we apply to application code: version control, peer review, repeatability, and auditability.

If a pipeline change breaks production, with Pipeline as Code, I can `git log` the Jenkinsfile and see exactly who changed what on which commit. I can `git revert` and restore the previous working pipeline in minutes.

Without it, if the Jenkins server goes down or the UI configuration gets corrupted, that pipeline configuration is gone. You're recreating from memory.

In my team at Atos, every pipeline change required a PR. The tech lead reviewed Jenkinsfile changes just like application code changes. That discipline prevented several incidents where someone accidentally removed the security scan stage."

---

## Question 11
**"Can you have multiple Jenkinsfiles in one repository?"**

### How to Answer:

"Yes. The default is for Jenkins to look for a file named `Jenkinsfile` at the root of the repository. But you can configure Jenkins to use a different path.

In the job configuration, under Pipeline → Script Path, you can specify any path like `jenkins/Jenkinsfile.deploy` or `ci/pipeline.groovy`.

This is useful in monorepos where you have multiple services and each needs its own pipeline:
```
monorepo/
├── service-a/
│   └── Jenkinsfile
├── service-b/
│   └── Jenkinsfile
└── Jenkinsfile          ← root pipeline for common tasks
```

You create separate Jenkins jobs, each pointing to the path of its respective Jenkinsfile."

---

## Question 12
**"What is the `script { }` block in declarative pipeline? When do you use it?"**

### How to Answer:

"In declarative pipeline, most steps must follow a predefined structure. But sometimes you need to write actual Groovy code — like a loop, a map, a try-catch, or dynamic logic.

The `script { }` block is an escape hatch inside declarative pipeline that lets you write scripted Groovy syntax.

Example — dynamically setting a variable:
```groovy
stage('Set Version') {
    steps {
        script {
            def gitCommit = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
            env.IMAGE_TAG = gitCommit
        }
    }
}
```

Without `script { }`, I cannot define a variable and assign it to `env`. With it, I can run Groovy code inline.

The rule: keep `script { }` blocks small and focused. If you find yourself writing 50 lines of Groovy inside `script`, consider moving that logic to a Shared Library function instead."

---

## Quick-Fire Round (Short Answers for Rapid Interview Questions)

**Q: What is the root block for declarative pipeline?**
A: `pipeline { }`

**Q: What is the root block for scripted pipeline?**
A: `node { }`

**Q: Where does Jenkinsfile live?**
A: Root of the Git repository (by default).

**Q: What does `sh` do in a pipeline step?**
A: Executes a shell (bash) command on the agent.

**Q: What is `env.BUILD_NUMBER`?**
A: A Jenkins environment variable automatically set to the current build number.

**Q: What does `|| true` at the end of a shell command mean?**
A: Even if the command fails, exit with success (prevents pipeline failure in cleanup steps).

**Q: Can you use `if` statements directly inside `steps { }` in declarative pipeline?**
A: No. You need a `when` block or a `script { }` block for conditional logic.

**Q: What is the purpose of `agent none` at the top level?**
A: No global agent — forces you to define an agent per stage, giving each stage a different environment.

---

*Next: Day 2 — Multibranch Pipeline + Shared Libraries (Theory + Labs)*# Jenkins Day 1 — Interview Q&A


**Q: What is the purpose of `agent none` at the top level?**
A: No global agent — forces you to define an agent per stage, giving each stage a different environment.

---

*Next: Day 2 — Multibranch Pipeline + Shared Libraries (Theory + Labs)*
