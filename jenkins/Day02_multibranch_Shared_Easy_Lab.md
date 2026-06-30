# Jenkins Day 2 — PRACTICAL LAB ONLY
## Step-by-Step Hands-On Guide (Easy Mode)

> This file has ONLY hands-on steps. No theory. Just do exactly what is written, in order.
> Two labs: Lab A = Multibranch Pipeline, Lab B = Shared Library.
> Do Lab A first. Take a break. Then do Lab B.

---

# LAB A — Multibranch Pipeline

## What we are going to do in plain words

Right now, your Jenkins job (`Day1_Declarative_Pipeline`) only watches the `main` branch.

In this lab, we will:
1. Create a NEW branch in your GitHub repo
2. Create a NEW Jenkins job (Multibranch type)
3. Watch Jenkins automatically find BOTH branches without you creating two jobs manually

---

## Step A1 — Open Terminal on Your Ubuntu Machine

```bash
cd ~/Devops-Practice-lab
```

(This goes into your existing repo folder — the same one used in Day 1)

---

## Step A2 — Check What Branch You Are On

```bash
git branch
```

You should see:
```
* main
```

The `*` means you are currently on `main`.

---

## Step A3 — Create a New Branch

```bash
git checkout -b feature/test-multibranch
```

**What this command does, word by word:**
- `git checkout` → switch to a different branch
- `-b` → create this branch if it doesn't already exist
- `feature/test-multibranch` → the name of the new branch you're creating

After running this, type `git branch` again — you'll now see:
```
  main
* feature/test-multibranch
```

The `*` moved — you are now on the new branch.

---

## Step A4 — Make a Small Visible Change

```bash
nano Jenkinsfile
```

Find this line inside your `Jenkinsfile`:
```groovy
stage('Hello') {
    steps {
        echo 'Stage 1: Hello from Jenkins!'
```

Change the text inside the echo to:
```groovy
stage('Hello') {
    steps {
        echo 'Stage 1: Hello from FEATURE BRANCH!'
```

Save and exit nano: Press `Ctrl + O`, then `Enter`, then `Ctrl + X`.

---

## Step A5 — Commit and Push This New Branch

```bash
git add Jenkinsfile
git commit -m "Test branch for multibranch pipeline"
git push origin feature/test-multibranch
```

**What this does:**
- `git add Jenkinsfile` → stage the changed file
- `git commit -m "..."` → save the change with a message
- `git push origin feature/test-multibranch` → upload this NEW branch to GitHub

Go check your GitHub repo in browser — you should now see TWO branches: `main` and `feature/test-multibranch`.

---

## Step A6 — Go to Jenkins UI

Open Jenkins in your browser (usually `http://localhost:8080`).

---

## Step A7 — Create New Item

1. Click **New Item** (top left)
2. Type name: `Day2_Multibranch_Lab`
3. Scroll down, click **Multibranch Pipeline**
4. Click **OK**

---

## Step A8 — Add Your Git Repo

You'll land on a configuration page.

1. Find section: **Branch Sources**
2. Click **Add source** → select **Git**
3. In **Repository URL**, paste:
   ```
   https://github.com/muskan7860/Devops-Practice-lab.git
   ```
4. Leave **Credentials** as `- none -` (since it's a public repo)

---

## Step A9 — Leave Everything Else Default

Scroll down — you'll see **Build Configuration** already set to:
- Mode: **by Jenkinsfile**
- Script Path: **Jenkinsfile**

Don't change this. This is correct already.

---

## Step A10 — Save

Click **Save** button at the bottom.

---

## Step A11 — Watch What Happens (This is the Magic Moment)

Jenkins will automatically start scanning. Wait 10-20 seconds.

You will see a screen showing:
```
Day2_Multibranch_Lab
├── main
└── feature/test-multibranch
```

**Both branches appeared automatically.** You did NOT create two separate jobs. Jenkins found them by scanning your repo.

---

## Step A12 — Click on Each Branch

1. Click on `main` → it will run automatically (or click **Build Now**) → check console output
2. Click on `feature/test-multibranch` → click **Build Now** → check console output

You will see in the `feature/test-multibranch` build log:
```
Stage 1: Hello from FEATURE BRANCH!
```

And in the `main` build log:
```
Stage 1: Hello from Jenkins!
```

**Each branch ran its OWN version of the Jenkinsfile, completely separately.**

---

## ✅ Lab A Checklist — Confirm Before Moving to Lab B

- [ ] I created a new branch called `feature/test-multibranch`
- [ ] I pushed it to GitHub
- [ ] I created a Multibranch Pipeline job in Jenkins
- [ ] Jenkins showed BOTH `main` and `feature/test-multibranch` automatically
- [ ] I ran both and saw different output for each

If all boxes are checked — Lab A is done. Take a 5 minute break before Lab B.

---

---

# LAB B — Shared Library

## What we are going to do in plain words

Right now, all your pipeline code lives inside ONE Jenkinsfile.

In this lab, we will:
1. Create a SEPARATE GitHub repo (just for reusable code)
2. Write ONE simple reusable function in it
3. Tell Jenkins about this new repo
4. Call that function from your Day 1 Jenkinsfile

---

## Step B1 — Go Back to Home Directory

```bash
cd ~
```

---

## Step B2 — Create a New Folder for the Shared Library

```bash
mkdir jenkins-shared-library
cd jenkins-shared-library
```

---

## Step B3 — Create the Required Folder Structure

```bash
mkdir vars
```

**Why this exact folder name `vars`?** Jenkins specifically looks for a folder named `vars` to find reusable functions. This name cannot be changed.

---

## Step B4 — Create Your First Reusable Function File

```bash
nano vars/sayHello.groovy
```

Type this exact content inside:

```groovy
def call(String name) {
    echo "Hello, ${name}! This message came from the Shared Library."
    echo "Current build number: ${env.BUILD_NUMBER}"
}
```

Save and exit: `Ctrl + O`, `Enter`, `Ctrl + X`

**What this file means in plain words:**
- The filename `sayHello.groovy` becomes the function name `sayHello()` that you can call later
- `def call(String name)` — this is the function. It expects ONE input: a name (text)
- Whatever name you pass in, it prints a hello message with it
- `env.BUILD_NUMBER` — Jenkins automatically fills this in with the current build number

---

## Step B5 — Turn This Folder into a Git Repository

```bash
git init
git add vars/sayHello.groovy
git commit -m "Add sayHello shared library function"
git branch -M main
```

**What each command does:**
- `git init` → turns this folder into a Git repository
- `git add` → stages your new file
- `git commit -m "..."` → saves it with a message
- `git branch -M main` → names the default branch `main`

---

## Step B6 — Create a New Repo on GitHub (in Browser)

1. Go to https://github.com/new
2. Repository name: `jenkins-shared-library`
3. Keep it **Public**
4. Do NOT check "Add a README" (leave everything unchecked)
5. Click **Create repository**

GitHub will show you a page with commands — ignore those, use the ones below instead.

---

## Step B7 — Connect Your Local Folder to GitHub and Push

```bash
git remote add origin https://github.com/muskan7860/jenkins-shared-library.git
git push -u origin main
```

**What this does:**
- `git remote add origin <url>` → tells Git "this is where to upload to"
- `git push -u origin main` → uploads your code to GitHub

Go check GitHub in your browser — you should see your `vars/sayHello.groovy` file there now.

---

## Step B8 — Register This Library in Jenkins (One-Time Setup)

1. Go to Jenkins UI
2. Click **Manage Jenkins** (left sidebar)
3. Click **System** (under System Configuration)
4. Scroll down until you find **Global Pipeline Libraries** section
5. Click **Add**

Fill in:
- **Name**: `practice-library`
- **Default version**: `main`
- Under **Retrieval method**, select: **Modern SCM**
- Click **Add Source** → choose **Git**
- **Project Repository**: `https://github.com/muskan7860/jenkins-shared-library.git`

---

## Step B9 — Save Jenkins Configuration

Scroll to bottom of page → Click **Save**.

---

## Step B10 — Go Back to Your Day 1 Pipeline Job

In Jenkins, click on `Day1_Declarative_Pipeline` job.

Click **Configure** (left side).

---

## Step B11 — Edit Pipeline Script Path / Or Edit Your Jenkinsfile Directly

Since your Day 1 job uses "Pipeline script from SCM" (reads from GitHub), you need to edit the actual `Jenkinsfile` file in your repo.

On your Ubuntu terminal:

```bash
cd ~/Devops-Practice-lab
git checkout main
nano Jenkinsfile
```

---

## Step B12 — Add the Library Import Line at the VERY TOP

Your Jenkinsfile currently starts with:
```groovy
pipeline {
```

Change it to add ONE line before that:
```groovy
@Library('practice-library') _

pipeline {
```

**Important:** Notice the underscore `_` after the closing bracket `)`. This is required — don't skip it.

---

## Step B13 — Add a New Stage That Calls Your Function

Find your `stages { }` block. Add a NEW stage right after `stage('Hello')`:

```groovy
stage('Shared Library Test') {
    steps {
        sayHello('Muskan')
    }
}
```

So your stages block should now look like:

```groovy
stages {
    stage('Hello') {
        steps {
            echo 'Stage 1: Hello from Jenkins!'
        }
    }

    stage('Shared Library Test') {
        steps {
            sayHello('Muskan')
        }
    }

    stage('Who Am I') {
        ...
    }
    ...
}
```

Save and exit: `Ctrl + O`, `Enter`, `Ctrl + X`

---

## Step B14 — Push This Change

```bash
git add Jenkinsfile
git commit -m "Use shared library sayHello function"
git push origin main
```

---

## Step B15 — Run the Build in Jenkins

1. Go to Jenkins UI → `Day1_Declarative_Pipeline` job
2. Click **Build Now**
3. Click on the new build number → **Console Output**

---

## Step B16 — What You Should See in Console Output

```
[Pipeline] stage
[Pipeline] { (Shared Library Test)
[Pipeline] echo
Hello, Muskan! This message came from the Shared Library.
[Pipeline] echo
Current build number: 3
[Pipeline] }
[Pipeline] // stage
```

**If you see "Hello, Muskan! This message came from the Shared Library." — Lab B is successful.**

This proves: your Jenkinsfile called a function that physically lives in a DIFFERENT GitHub repository, and Jenkins fetched and executed it.

---

## ✅ Lab B Checklist

- [ ] I created a new repo `jenkins-shared-library` with a `vars/sayHello.groovy` file
- [ ] I pushed it to GitHub
- [ ] I registered it in Jenkins under Manage Jenkins → System → Global Pipeline Libraries
- [ ] I added `@Library('practice-library') _` to the top of my Day 1 Jenkinsfile
- [ ] I added a stage that calls `sayHello('Muskan')`
- [ ] I saw the message printed in console output from the shared library

---

## If Something Goes Wrong — Quick Fixes

### Error: "Library practice-library not found"
→ Go back to Step B8-B9. The Name you typed in Jenkins config must EXACTLY match what you wrote in `@Library('practice-library')` — spelling and case matter.

### Error: "No such DSL method 'sayHello'"
→ Check your file is named exactly `vars/sayHello.groovy` (lowercase first letter, no typos) and the method inside is named exactly `call`.

### Error: "expecting '}', found ''"
→ You likely have a missing closing bracket `}` somewhere in your Jenkinsfile after pasting the new stage. Recheck the file with:
```bash
cat Jenkinsfile
```
Count opening `{` and closing `}` — they must match.

### Nothing happens / build doesn't trigger
→ Make sure you pushed your changes: run `git status` — it should say "nothing to commit, working tree clean". If not, run `git push origin main` again.

---

## Final Recap — What You Just Built (In Plain Words)

1. **Lab A**: You made Jenkins automatically detect and build TWO different Git branches without manually creating two jobs.
2. **Lab B**: You moved reusable code OUT of your main Jenkinsfile into a separate repository, then called it like a function — exactly how real companies organize their pipeline code at scale.

You now have hands-on proof of both concepts. When an interviewer asks about these, you can say "I built this myself" — not just "I read about it."
