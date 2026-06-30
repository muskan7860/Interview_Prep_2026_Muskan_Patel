# Jenkins Day 2 — Interview Q&A
## Multibranch Pipeline & Shared Libraries

> Answers written to be spoken aloud naturally, senior-level phrasing.

---

## Question 1
**"What is a Multibranch Pipeline and how is it different from a regular Pipeline job?"**

### How to Answer:

"A regular Pipeline job is tied to one specific branch — usually `main`. If developers push to other branches like `feature/login` or `bugfix/auth`, that job won't pick them up unless you manually reconfigure it.

A Multibranch Pipeline solves that. It scans the entire repository, finds every branch that has a Jenkinsfile, and automatically creates a separate pipeline for each branch — without anyone manually creating jobs.

In my experience, this is the standard setup for any real team because developers work on dozens of branches simultaneously. You can't realistically have someone manually creating a Jenkins job every time a developer creates a feature branch. Multibranch Pipeline automates that entirely — and it also auto-deletes the pipeline when the branch is deleted from Git."

---

## Question 2
**"How does Jenkins know which branches to build in a Multibranch Pipeline?"**

### How to Answer:

"When you create a Multibranch Pipeline job, you configure a Branch Source — typically Git — pointing to your repository. Jenkins then performs what's called a 'repository scan'.

During the scan, Jenkins checks every branch in the repo and looks for a file with the configured script path — by default, a file named `Jenkinsfile` at the root.

If a branch has that file, Jenkins creates a pipeline job for it. If it doesn't, that branch is simply skipped — no error, it's just ignored.

This scan can be triggered periodically — say every minute — or, in production, it's usually triggered by a webhook from GitHub whenever a push happens, so detection is near-instant rather than waiting for the next scheduled scan."

---

## Question 3
**"What happens when you delete a branch in Multibranch Pipeline?"**

### How to Answer:

"Jenkins has an 'orphaned item strategy' for Multibranch Pipeline. When a branch is deleted from the Git repository, on the next scan, Jenkins notices that branch no longer exists and marks its corresponding pipeline job as disabled or removes it, depending on the configured retention strategy.

You can configure how long to keep the build history of deleted branches before fully removing it — useful if you want to keep historical build logs for auditing even after a feature branch is merged and deleted."

---

## Question 4
**"What is a Jenkins Shared Library and why would you use it?"**

### How to Answer:

"A Shared Library is reusable pipeline code — written in Groovy — stored in its own separate Git repository, which any Jenkinsfile across the organization can import and use.

The core problem it solves is duplication. If you have twenty microservices, and each Jenkinsfile has the same Docker build-and-push logic copy-pasted, then any change — say, switching Docker registries or adding a security scan — means editing twenty separate files. That's not maintainable and it's error-prone.

With a Shared Library, that logic lives in one place. Each Jenkinsfile just calls a function like `buildDockerImage('service-name', buildNumber)`. If the logic needs to change, you update it once in the library, and every consuming pipeline automatically gets the updated behavior on its next run.

In my project, I'd design this so the Jenkinsfile itself stays short — maybe 15-20 lines — and all the actual implementation logic lives in the shared library, which goes through its own code review process."

---

## Question 5
**"Explain the folder structure of a Jenkins Shared Library."**

### How to Answer:

"A Shared Library repository follows a specific structure that Jenkins expects.

The `vars` folder is the one used most often. Each Groovy file inside `vars` becomes a globally callable step. For example, if I create `vars/buildDockerImage.groovy`, I can call it directly as `buildDockerImage()` from any Jenkinsfile — it behaves just like a built-in Jenkins step.

The `src` folder is for more complex, object-oriented code — full Groovy classes following standard package structure, like `src/org/mycompany/Utils.groovy`. I use this when I need actual classes with state, not just simple reusable functions.

The `resources` folder holds non-Groovy files — JSON configs, shell script templates — that the library code needs to read at runtime, accessed using the `libraryResource` step.

In practice, ninety percent of what I write lives in `vars`, because most pipeline logic is just procedural steps, not object-oriented design."

---

## Question 6
**"How do you call a Shared Library function from a Jenkinsfile? Walk me through the syntax."**

### How to Answer:

"First, the library has to be registered in Jenkins — that's a one-time admin step done in 'Manage Jenkins → System → Global Pipeline Libraries', where you give the library a name and point it to its Git repository.

Then in the Jenkinsfile, at the very top, before the `pipeline` block, you write:

```groovy
@Library('my-shared-library') _
```

The underscore at the end is required Groovy syntax — without it, the import won't register correctly.

After that, any function defined in the library's `vars` folder is directly callable. If I have `vars/buildDockerImage.groovy` with a `call` method that takes an image name and tag, I'd use it like:

```groovy
buildDockerImage('payment-service', env.BUILD_NUMBER)
```

It looks exactly like a built-in pipeline step, which is the whole point — it makes the Jenkinsfile clean and readable."

---

## Question 7
**"Why does the function inside the `vars` file need to be named `call`?"**

### How to Answer:

"Jenkins has a specific convention here — the filename becomes the step name, but the method that actually executes when that step is invoked must be literally named `call`.

So if I name my file `vars/deployToK8s.groovy`, and inside it I define `def call(String service, String env) { ... }`, then in the Jenkinsfile I write `deployToK8s('payment-service', 'staging')` — Jenkins maps that step name to the `call` method inside the matching file.

If you name the method anything other than `call`, Jenkins won't recognize it as the entry point and the step won't work."

---

## Question 8
**"What is the difference between using positional parameters versus a Map in Shared Library functions?"**

### How to Answer:

"Positional parameters work fine when a function takes one or two simple, always-required arguments — like `buildDockerImage(imageName, tag)`.

But when a function has many optional parameters, I prefer accepting a `Map`. For example:

```groovy
def call(Map config = [:]) {
    def projectKey = config.projectKey ?: 'default-project'
    def sourcePath = config.sourcePath ?: 'src'
}
```

This lets the caller pass only what's relevant, using named arguments:

```groovy
sonarQubeScan(projectKey: 'payment-service', sourcePath: 'src/main/java')
```

It's far more readable than remembering positional order, especially as the number of parameters grows. The Elvis operator `?:` provides a sensible default if a key isn't passed, so the function doesn't break on missing optional values."

---

## Question 9
**"In a real organization, how would Multibranch Pipeline and Shared Libraries work together?"**

### How to Answer:

"They complement each other directly. The Shared Library holds all the common, reusable pipeline logic — building Docker images, running SonarQube scans, deploying to Kubernetes. It lives in its own repository with its own version control and review process.

Each microservice repository then has a Multibranch Pipeline configured against it, so every branch in that service automatically gets CI/CD without manual setup.

The Jenkinsfile in each service repo imports the shared library at the top with `@Library(...)`, and then each stage simply calls a shared library function — passing the service-specific values like the image name or branch name.

This means a Jenkinsfile for any of twenty microservices ends up being short and nearly identical in structure, while the actual implementation logic is centralized, version-controlled, and consistently applied across the whole organization. If we need to add a new security scanning step to every pipeline, we update the shared library once instead of touching twenty repos."

---

## Question 10
**"What's the difference between specifying a library version like `@Library('my-lib@main')` versus `@Library('my-lib')`?"**

### How to Answer:

"By default, when you register a library in Jenkins global config, you set a default version — usually `main` or a specific tag. If you just write `@Library('my-lib')` in the Jenkinsfile, it uses that configured default version.

But you can override it per-Jenkinsfile by specifying the branch or tag explicitly: `@Library('my-lib@v2.1')` would pull a specific tagged release of the library, or `@Library('my-lib@feature-branch')` would pull from a specific branch.

This is useful when you're developing a new feature in the shared library itself and want to test it against one service's pipeline before merging it to `main` and affecting every other team using the library."

---

## Quick-Fire Round

**Q: What job type do you select in Jenkins for auto-detecting all Git branches?**
A: Multibranch Pipeline

**Q: What does Jenkins look for in each branch during a Multibranch scan?**
A: A file matching the configured script path — default is `Jenkinsfile`

**Q: What happens to a branch's pipeline if it has no Jenkinsfile?**
A: It's skipped — no pipeline is created, no error thrown

**Q: What folder in a Shared Library contains globally callable functions?**
A: `vars/`

**Q: What must the method inside a `vars/*.groovy` file be named?**
A: `call`

**Q: What line do you add at the top of a Jenkinsfile to use a Shared Library?**
A: `@Library('library-name') _`

**Q: What does the trailing underscore after `@Library(...)` do?**
A: Required Groovy syntax to complete the import statement

**Q: What folder in a Shared Library is used for full Groovy classes?**
A: `src/`

**Q: What operator provides a default value if a Map key is missing?**
A: The Elvis operator `?:`

**Q: How do you trigger an immediate re-scan of branches in Multibranch Pipeline?**
A: Click "Scan Repository Now" in the Jenkins UI

---

## Scenario-Based Questions

### Scenario 1
**"Your team has 15 microservices, each with its own repo. Every Jenkinsfile duplicates the same SonarQube scan and Docker push logic. How would you redesign this?"**

### How to Answer:

"I'd extract the common logic into a Shared Library. I'd create functions like `sonarQubeScan()` and `buildAndPushDocker()` in the library's `vars` folder, accepting parameters for things that differ per service — like project key or image name.

Then I'd update each of the 15 Jenkinsfiles to import the library with `@Library(...)` and replace the duplicated blocks with single function calls. Each Jenkinsfile becomes much shorter and consistent.

Going forward, if we need to change how scanning works — say, switching to a new SonarQube server — I update it once in the library, and it automatically applies to all 15 services on their next build, instead of opening 15 pull requests."

---

### Scenario 2
**"A developer says their feature branch isn't getting built in Jenkins even though Multibranch Pipeline is configured. How would you troubleshoot?"**

### How to Answer:

"First, I'd check if the branch actually has a Jenkinsfile at the path configured in the job — by default that's the repo root, file named exactly `Jenkinsfile`. A missing or misnamed file is the most common cause.

Second, I'd manually trigger 'Scan Repository Now' in the Multibranch job to force an immediate scan rather than waiting for the scheduled interval.

Third, I'd check the 'Scan Repository Log' in Jenkins — it shows exactly why a branch was included or excluded, which is the fastest way to diagnose this.

Fourth, I'd verify branch filtering rules — some Multibranch configurations have branch name patterns or behaviors set up that exclude certain branches, like excluding all branches that don't match `feature/*` or `main`."

---

*Next: Day 3 — Stages & Steps (Checkout, Build, Test, SonarQube, Docker, Deploy, Post Actions)*
