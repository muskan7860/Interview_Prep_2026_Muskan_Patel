# Jenkins Day 5 — Interview Q&A
## Credentials: withCredentials, Secret Text, Username/Password, SSH Key

> All answers written to be spoken aloud naturally.
> Senior-level phrasing throughout.

---

## Question 1
**"How do you handle secrets and credentials in Jenkins pipelines?"**

### How to Answer:

"The fundamental rule is: never hardcode credentials in a Jenkinsfile. The Jenkinsfile lives in a Git repository, which could be public or accessible to many people. Putting passwords or tokens there is a serious security risk.

Jenkins provides a built-in Credentials Store — a secure vault where you store secrets once, encrypted on disk. Each credential gets an ID. In the Jenkinsfile, you reference only that ID, never the actual secret value.

There are two main ways to consume credentials in a pipeline. The first is the `withCredentials` block — you wrap the steps that need the secret, Jenkins injects it as an environment variable only inside that block, and automatically masks it in console output so it appears as `****`. Once the block ends, the variable is gone.

The second way is declaring credentials in the `environment` block at the pipeline level using the `credentials()` helper function. This makes them available throughout the pipeline but with the same masking behaviour.

In my projects at Atos, we stored Docker Hub credentials, deployment SSH keys, and SonarQube tokens in the Jenkins store. The Jenkinsfile only ever contained the credential IDs like `dockerhub-creds` or `ec2-ssh-key` — never actual values."

---

## Question 2
**"What is the difference between Secret Text and Username/Password credential types?"**

### How to Answer:

"Both are stored securely in Jenkins, but they represent different shapes of credentials.

A Secret Text is a single string — used for API tokens, webhook secrets, or any value that is just one piece of text. For example, a SonarQube authentication token or a Slack webhook URL. In the pipeline, you inject it with `string(credentialsId: 'token-id', variable: 'MY_TOKEN')` and it becomes one environment variable.

A Username/Password credential has two parts — a username and a password. Used for anything that requires a login — Docker Hub, Nexus registry, Git repositories with basic auth. When you inject it with `usernamePassword(...)`, Jenkins creates two separate environment variables — one for username and one for password. And if you use it in the `environment` block, Jenkins automatically creates three variables: the combined value, and then `_USR` and `_PSW` suffixes for the individual parts.

The key difference in practice is that a token is usually just passed as a Bearer header or `-Dsonar.login`, while username/password requires both parts separately — like `docker login -u $USER --password-stdin`."

---

## Question 3
**"What does `withCredentials` do? Why is it preferred over environment variables?"**

### How to Answer:

"`withCredentials` is a Jenkins pipeline step that retrieves a credential from the Jenkins Credentials Store and makes it available as an environment variable inside a specific block of steps. When the block ends, the variable is destroyed — it does not persist to subsequent stages.

There are three important things it does automatically: it decrypts the secret from the Jenkins vault, it injects it as a temporary environment variable, and it masks it in console output — anywhere the secret value appears in logs, Jenkins replaces it with four asterisks.

The reason it is preferred over setting environment variables directly is scope. If you set a credential as a global pipeline environment variable, it is theoretically accessible everywhere — including places where it should not be needed. `withCredentials` limits the secret's availability to only the steps that genuinely need it. This follows the principle of least privilege — code only has access to secrets for the minimum time and scope necessary.

Another practical reason: if you accidentally print the variable outside `withCredentials`, it shows `NOT_SET` rather than the secret value. This prevents accidental leaks."

---

## Question 4
**"Show me how you would use `withCredentials` for a Docker Hub login."**

### How to Answer:

"The credential is first saved in Jenkins under Manage Jenkins → Credentials with kind 'Username with password', ID `dockerhub-creds`, containing my Docker Hub username and password.

Then in the Jenkinsfile:

```groovy
stage('Docker Push') {
    steps {
        withCredentials([usernamePassword(
            credentialsId: 'dockerhub-creds',
            usernameVariable: 'DOCKER_USER',
            passwordVariable: 'DOCKER_PASS'
        )]) {
            sh "echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin"
            sh "docker push myrepo/myapp:${env.BUILD_NUMBER}"
        }
    }
}
```

A few things I specifically do here — I use `--password-stdin` instead of `-p $DOCKER_PASS`. The reason is that command-line arguments are visible in the Linux process list — anyone running `ps aux` on the agent could see the password. Piping through stdin avoids that.

I also use single quotes in the `sh` step for `$DOCKER_PASS` so Groovy does not interpolate it before Jenkins can mask it. If I used double quotes, Groovy would resolve the variable value first and Jenkins would have no chance to mask it."

---

## Question 5
**"What is the danger of using double quotes vs single quotes with credential variables in `sh` steps?"**

### How to Answer:

"This is a subtle but important distinction.

In Jenkins pipeline, `sh` accepts both single-quoted and double-quoted strings. When you use double quotes, Groovy processes the string first — it resolves any `${variable}` expressions before passing the command to the shell. When you use single quotes, Groovy treats the string literally and the shell processes the variables.

The problem with credential masking is that Jenkins masks values at the shell output level — after the command runs. If Groovy resolves `$DOCKER_PASS` to the actual password before Jenkins can intercept it, the masking never happens and the plain text password appears in the console log.

So:
```groovy
// WRONG — Groovy resolves variable, password appears in plain text
sh "echo $DOCKER_PASS | docker login"

// CORRECT — Shell handles variable, Jenkins can mask it
sh 'echo $DOCKER_PASS | docker login'
```

For `BUILD_NUMBER` and other Jenkins variables, double quotes are fine — they are not secrets. But for anything from `withCredentials`, always use single quotes in the `sh` step."

---

## Question 6
**"What is an SSH credential in Jenkins? When do you use it?"**

### How to Answer:

"An SSH credential stores an SSH private key securely in Jenkins. It is used whenever the pipeline needs to authenticate to a remote server using SSH — the most common case being deployment: SSH into a server, pull the latest Docker image, restart the application.

When you save an SSH key credential, you store the private key content in Jenkins. Jenkins never exposes this key in logs or environment variables directly.

In the pipeline, you use `sshUserPrivateKey` inside `withCredentials`:

```groovy
withCredentials([sshUserPrivateKey(
    credentialsId: 'ec2-ssh-key',
    keyFileVariable: 'SSH_KEY_FILE',
    usernameVariable: 'SSH_USER'
)]) {
    sh 'ssh -i $SSH_KEY_FILE -o StrictHostKeyChecking=no $SSH_USER@10.0.1.10 ./deploy.sh'
}
```

Notice `keyFileVariable` — Jenkins does not expose the key as a string. Instead it writes the private key to a temporary file on the agent and puts the file path into the variable. The `ssh` command then uses `-i` to point to that file. After the `withCredentials` block ends, Jenkins deletes that temporary file."

---

## Question 7
**"Where does Jenkins store credentials? How are they secured?"**

### How to Answer:

"Jenkins stores credentials in a file called `credentials.xml` inside the Jenkins home directory — in my Kubernetes setup that is `/var/jenkins_home/credentials.xml`.

The actual secret values are not stored as plain text. Jenkins uses AES-128 encryption. The master key used for encryption is stored in `/var/jenkins_home/secrets/master.key`. The encrypted values in credentials.xml look like `{AQAAABAAAAAwF+...}` — completely unreadable without the master key.

This means if someone gets hold of `credentials.xml` alone, they cannot decrypt the secrets without also having `master.key`. And vice versa — `master.key` alone is useless without `credentials.xml`.

In production, backing up Jenkins home should include both files, and both should be protected. I verified this in my practice setup by exec-ing into the Jenkins master pod and reading credentials.xml — the passwords showed as encrypted strings, confirming the security model."

---

## Question 8
**"What happens if you try to access a credential variable outside the `withCredentials` block?"**

### How to Answer:

"The variable simply does not exist outside the block. `withCredentials` creates the environment variable when the block starts and destroys it when the block ends. If you try to access it in a subsequent step, the shell sees it as an unset variable.

In bash, an unset variable evaluates to empty string by default, or you can use the `:-` operator to provide a default: `${API_TOKEN:-NOT_SET}` which prints `NOT_SET` if the variable is not defined.

This scoping behaviour is intentional and is one of the security benefits of `withCredentials` over setting credentials globally. If a credential is only needed in the Docker Push stage, there is no reason for it to be available in the Checkout or Test stages. Limiting scope limits the blast radius if something goes wrong."

---

## Question 9
**"Can you inject multiple credentials at once in a single `withCredentials` block?"**

### How to Answer:

"Yes — the `withCredentials` step accepts a list. You can inject as many credentials as you need in one block:

```groovy
withCredentials([
    string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN'),
    usernamePassword(
        credentialsId: 'dockerhub-creds',
        usernameVariable: 'DOCKER_USER',
        passwordVariable: 'DOCKER_PASS'
    ),
    sshUserPrivateKey(
        credentialsId: 'ec2-ssh-key',
        keyFileVariable: 'SSH_KEY'
    )
]) {
    sh 'mvn sonar:sonar -Dsonar.login=$SONAR_TOKEN'
    sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'
    sh 'ssh -i $SSH_KEY ubuntu@server ./deploy.sh'
}
```

I use this when a single stage needs to do multiple things — scan, push Docker image, and deploy — all requiring different credentials. Keeping them in one block keeps the code clean rather than nesting multiple `withCredentials` blocks inside each other."

---

## Question 10
**"What is the `credentials()` helper in the environment block? How does it differ from `withCredentials`?"**

### How to Answer:

"The `credentials()` helper in the `environment` block is a shorthand for making credentials available throughout the entire pipeline without wrapping individual stages in `withCredentials`.

```groovy
environment {
    DOCKER_CREDS = credentials('dockerhub-creds')
}
```

For a username/password credential, this creates three variables automatically: `DOCKER_CREDS` containing `username:password`, `DOCKER_CREDS_USR` containing just the username, and `DOCKER_CREDS_PSW` containing just the password. The `_USR` and `_PSW` suffixes are added automatically by Jenkins.

The difference from `withCredentials` is scope. The `environment` block makes credentials available for the entire pipeline — every stage can access them. `withCredentials` scopes them to a specific block.

I use `environment` with `credentials()` when multiple stages need the same credential — say, both the Docker Build and Docker Push stages need Docker Hub credentials. I use `withCredentials` when only one specific stage needs it and I want strict scoping. Security-wise, `withCredentials` is slightly safer because of the tighter scope."

---

## Quick-Fire Round

**Q: What Jenkins feature stores secrets securely?**
A: Jenkins Credentials Store — Manage Jenkins → Credentials

**Q: Name four credential types in Jenkins.**
A: Secret Text, Username/Password, SSH Private Key, Secret File

**Q: What does `withCredentials` do to secret values in console logs?**
A: Masks them — replaces with `****`

**Q: What is `credentialsId`?**
A: The unique ID you assign when saving a credential in Jenkins — referenced in Jenkinsfile

**Q: Why use `--password-stdin` instead of `-p $PASS` for Docker login?**
A: Command line arguments are visible in `ps aux` — stdin is safer

**Q: Why use single quotes in `sh` steps with credential variables?**
A: Prevents Groovy from resolving the variable before Jenkins can mask it

**Q: What happens to credential variables outside `withCredentials` block?**
A: They do not exist — variable is unset/empty

**Q: What suffix does Jenkins auto-create for username/password in environment block?**
A: `_USR` for username and `_PSW` for password

**Q: Where does Jenkins store credentials on disk?**
A: `/var/jenkins_home/credentials.xml` — encrypted with AES-128

**Q: What is `keyFileVariable` in SSH credential?**
A: Path to a temporary file containing the private key — not the key content itself

---

## Scenario Questions

### Scenario 1
**"A developer committed a password directly into the Jenkinsfile and pushed to GitHub. What do you do?"**

### Answer:
"Three things immediately. First, rotate the credential — change the password right away because it is now compromised. Anyone who saw the commit has the old password.

Second, remove the hardcoded credential from the Jenkinsfile, add it to Jenkins Credentials Store with an appropriate ID, and use `withCredentials` to reference it. Push a new commit.

Third, if the repository is public, use `git filter-branch` or BFG Repo Cleaner to remove the secret from Git history — just deleting the file in a new commit still leaves the secret visible in older commits.

Going forward, I would add a pre-commit hook using a tool like `git-secrets` or `detect-secrets` that scans for credential patterns before allowing commits. This prevents the problem at the source rather than fixing it after the fact."

---

### Scenario 2
**"Your Jenkins pipeline fails with 'CredentialsNotFoundException: No credentials found with id dockerhub-creds'. How do you debug?"**

### Answer:
"Three things to check in order.

First, verify the credential exists — go to Manage Jenkins → Credentials and confirm `dockerhub-creds` is listed there. It might not have been created yet, or might have been accidentally deleted.

Second, check the scope — Jenkins credentials can be scoped to 'System', 'Global', or a specific folder. If the credential is scoped to System, it is not accessible from pipelines. It must be Global or scoped to the folder containing the job.

Third, check spelling exactly — credential IDs are case-sensitive. If the credential is saved as `DockerHub-Creds` and the Jenkinsfile says `dockerhub-creds`, it will not match.

In my experience, the most common cause is a typo in the credential ID or the credential being in the wrong scope."

---

*Next: Day 6 — Triggers (Webhook, pollSCM, Cron, Upstream Job, Parameterized Builds)*
