# Jenkins Day 5 — Credentials
## Theory + Hands-On Lab

---

## Before We Start — Why Credentials Matter

### The Problem First (Story)

Imagine you write this in your Jenkinsfile:

```groovy
sh "docker login -u muskan -p MyPassword123"
sh "docker push myrepo/myapp"
```

This Jenkinsfile is stored in GitHub — a public repository. Now ANYONE who visits your GitHub can see:
- Your Docker Hub username
- Your Docker Hub password

They can now push fake images to your registry, delete your images, or use your account for anything.

This is called **hardcoding credentials** — one of the most dangerous mistakes in DevOps.

---

### The Solution — Jenkins Credentials Store

Jenkins has a secure vault built inside it. You store your passwords, tokens, and SSH keys ONCE in this vault. Jenkins encrypts them on disk. Your Jenkinsfile never contains the actual secret — it only contains a reference ID.

Think of it like a bank locker:
- You deposit your valuables (password) in the locker
- You get a locker key ID (credential ID)
- In your Jenkinsfile, you say "open locker ID `dockerhub-creds`"
- Jenkins opens it and injects the value — never shows it to anyone

---

## Topic 1 — Types of Credentials in Jenkins

Jenkins supports these credential types:

| Type | When to Use | Example |
|------|-------------|---------|
| **Secret Text** | API tokens, single string secrets | SonarQube token, Slack token |
| **Username/Password** | Docker Hub, Git, any login | Docker registry login |
| **SSH Private Key** | SSH into servers | Deploy to EC2 via SSH |
| **Secret File** | Kubeconfig, certificate files | kubectl config file |
| **Certificate** | SSL/TLS certificates | HTTPS client certs |

---

## Topic 2 — How to Add Credentials in Jenkins UI

### Adding a Secret Text

1. Go to `https://devopsbymuskan07.com/jenkins/`
2. Click **Manage Jenkins** → **Credentials**
3. Click **System** → **Global credentials (unrestricted)**
4. Click **Add Credentials** (top right)
5. Fill in:
   - Kind: **Secret text**
   - Secret: `your-actual-token-value`
   - ID: `sonar-token` ← this is what you use in Jenkinsfile
   - Description: `SonarQube authentication token`
6. Click **Create**

### Adding Username/Password

Same steps, but:
- Kind: **Username with password**
- Username: `muskanpatel71198` (your Docker Hub username)
- Password: `your-docker-hub-password`
- ID: `dockerhub-creds`
- Description: `Docker Hub login credentials`

### Adding SSH Private Key

Same steps, but:
- Kind: **SSH Username with private key**
- ID: `ec2-ssh-key`
- Username: `ubuntu` (the SSH user on the server)
- Private Key: Enter directly → paste your private key content

---

## Topic 3 — withCredentials Block

### What is it?

`withCredentials` is a Jenkins pipeline step that:
1. Retrieves the credential from Jenkins vault
2. Makes it available as an environment variable ONLY inside the block
3. Masks it in console output (shows `****` instead of actual value)
4. Destroys the variable after the block ends

### Syntax for Secret Text

```groovy
withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
    sh 'echo Using token: $SONAR_TOKEN'
    sh 'mvn sonar:sonar -Dsonar.login=$SONAR_TOKEN'
}
```

### Explanation word by word:

**`withCredentials([...])`** → The `[...]` is a list — you can inject multiple credentials at once

**`string(`** → This is the Secret Text credential type

**`credentialsId: 'sonar-token'`** → The ID you gave when saving the credential in Jenkins UI

**`variable: 'SONAR_TOKEN'`** → The environment variable name Jenkins will create inside the block. You choose this name.

**`sh 'echo Using token: $SONAR_TOKEN'`** → Jenkins will print `Using token: ****` in console output — the actual token value is masked automatically

---

### Syntax for Username/Password

```groovy
withCredentials([usernamePassword(
    credentialsId: 'dockerhub-creds',
    usernameVariable: 'DOCKER_USER',
    passwordVariable: 'DOCKER_PASS'
)]) {
    sh "echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin"
    sh "docker push myrepo/myapp:${env.BUILD_NUMBER}"
}
```

### Explanation word by word:

**`usernamePassword(`** → This credential type splits into TWO variables — one for username, one for password

**`usernameVariable: 'DOCKER_USER'`** → Username goes into this variable

**`passwordVariable: 'DOCKER_PASS'`** → Password goes into this variable

**`echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin`** → Passes password through stdin (pipe) — safer than using `-p` flag which shows in process list

---

### Syntax for SSH Key

```groovy
withCredentials([sshUserPrivateKey(
    credentialsId: 'ec2-ssh-key',
    keyFileVariable: 'SSH_KEY_FILE',
    usernameVariable: 'SSH_USER'
)]) {
    sh "ssh -i $SSH_KEY_FILE -o StrictHostKeyChecking=no $SSH_USER@10.0.1.10 'docker pull myapp && docker restart myapp'"
}
```

### Explanation word by word:

**`sshUserPrivateKey(`** → SSH key credential type

**`keyFileVariable: 'SSH_KEY_FILE'`** → Jenkins writes the private key to a temporary file and puts the file PATH into this variable. The key content itself is never exposed.

**`-i $SSH_KEY_FILE`** → SSH uses the key file for authentication instead of password

**`-o StrictHostKeyChecking=no`** → Do not prompt "Are you sure you want to connect?" — important for automated pipelines

---

## Topic 4 — environment Block with credentials()

Another way to use credentials — inject them as global environment variables for the whole pipeline:

```groovy
pipeline {
    agent { label 'k8s-agent' }

    environment {
        SONAR_TOKEN  = credentials('sonar-token')
        DOCKER_CREDS = credentials('dockerhub-creds')
    }

    stages {
        stage('Sonar Scan') {
            steps {
                sh 'mvn sonar:sonar -Dsonar.login=$SONAR_TOKEN'
            }
        }
        stage('Docker Push') {
            steps {
                sh "echo $DOCKER_CREDS_PSW | docker login -u $DOCKER_CREDS_USR --password-stdin"
            }
        }
    }
}
```

### Important — What happens with Username/Password in environment block:

When you use `credentials('dockerhub-creds')` for a username/password credential, Jenkins automatically creates THREE variables:

| Variable | Contains |
|----------|---------|
| `DOCKER_CREDS` | `username:password` combined |
| `DOCKER_CREDS_USR` | Just the username |
| `DOCKER_CREDS_PSW` | Just the password |

The `_USR` and `_PSW` suffix is automatic — Jenkins adds it.

---

## Topic 5 — Never Hardcode — Why and What Happens

### What Jenkins does to protect secrets

1. **Masks in logs** — if the secret appears anywhere in console output, Jenkins replaces it with `****`
2. **Not in environment** — credentials are only available inside the `withCredentials` block, not the whole pipeline
3. **Encrypted on disk** — Jenkins stores credentials encrypted in `/var/jenkins_home/secrets/`
4. **Audit trail** — Jenkins logs who accessed which credential and when

### What gets masked automatically

```
+ echo $DOCKER_PASS | docker login -u muskanpatel71198 --password-stdin
```

Jenkins console shows:
```
+ echo **** | docker login -u muskanpatel71198 --password-stdin
```

The password becomes `****` — even if it accidentally ends up in a log, it is safe.

---

## The Complete Credentials Pipeline

```groovy
pipeline {
    agent { label 'k8s-agent' }

    environment {
        DOCKER_IMAGE = 'muskanpatel71198/my-app'
        IMAGE_TAG    = "${env.BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                echo "Code checked out on agent: ${env.NODE_NAME}"
            }
        }

        stage('Use Secret Text') {
            steps {
                withCredentials([string(
                    credentialsId: 'my-api-token',
                    variable: 'API_TOKEN'
                )]) {
                    echo "Using API token (masked): $API_TOKEN"
                    sh 'curl -H "Authorization: Bearer $API_TOKEN" https://httpbin.org/get || true'
                }
            }
        }

        stage('Use Username Password') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    echo "Logging in as: $DOCKER_USER"
                    sh "echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin || true"
                    echo "Docker login done"
                }
            }
        }

        stage('Verify Credential is Masked') {
            steps {
                withCredentials([string(
                    credentialsId: 'my-api-token',
                    variable: 'SECRET_VAR'
                )]) {
                    sh 'echo The secret value is: $SECRET_VAR'
                    sh 'echo This proves Jenkins masks it automatically'
                }
            }
        }

    }

    post {
        success {
            echo "All credential stages passed!"
        }
        failure {
            echo "Something failed - check which stage turned red"
        }
        always {
            deleteDir()
        }
    }
}
```

---

## HANDS-ON LAB

> Read each step fully before typing.
> Everything happens in two places: Ubuntu terminal and Jenkins browser.
> Do steps in order. Do not skip.

---

### What We Are Building

We will:
1. Add a real credential in Jenkins UI (Secret Text)
2. Add a Username/Password credential
3. Write a Jenkinsfile that uses both
4. Run it and see Jenkins mask the secrets in console output
5. Prove that hardcoded values are dangerous by comparison

---

### PART A — Add Credentials in Jenkins UI

#### Step A1 — Go to Jenkins Credentials Page

Open browser and go to:
```
https://devopsbymuskan07.com/jenkins/manage/credentials/store/system/domain/_/
```

You will see the credentials list page.

---

#### Step A2 — Add Secret Text Credential

1. Click **Add Credentials** (top right button)
2. Fill in exactly:
   - **Kind**: Secret text
   - **Secret**: `my-practice-api-token-12345`
   - **ID**: `my-api-token`
   - **Description**: `Practice API token for Day 5 lab`
3. Click **Create**

You should see it appear in the list.

---

#### Step A3 — Add Username/Password Credential

1. Click **Add Credentials** again
2. Fill in exactly:
   - **Kind**: Username with password
   - **Username**: `muskanpatel71198`
   - **Password**: `practice-password-123`
   - **ID**: `dockerhub-creds`
   - **Description**: `Docker Hub credentials for Day 5 lab`
3. Click **Create**

You should now see TWO credentials in the list.

---

### PART B — Write the Jenkinsfile

#### Step B1 — Open Terminal and Go to Your Repo

```bash
cd ~/Devops-lab/Practice-Lab/jenkins
```

Check you are on main branch:

```bash
git branch
```

Should show `* main`.

---

#### Step B2 — Open Jenkinsfile

```bash
nano Jenkinsfile
```

---

#### Step B3 — Delete Everything and Paste This

Press `Ctrl + K` repeatedly until empty. Then paste:

```groovy
pipeline {
    agent { label 'k8s-agent' }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                echo "Running on agent: ${env.NODE_NAME}"
            }
        }

        stage('Secret Text Demo') {
            steps {
                withCredentials([string(
                    credentialsId: 'my-api-token',
                    variable: 'API_TOKEN'
                )]) {
                    echo "✅ Secret Text credential loaded successfully"
                    sh 'echo The token value is: $API_TOKEN'
                    sh 'echo Length of token: ${#API_TOKEN}'
                }
            }
        }

        stage('Username Password Demo') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    echo "✅ Username/Password credential loaded"
                    sh 'echo Username is: $DOCKER_USER'
                    sh 'echo Password is: $DOCKER_PASS'
                    sh 'echo Both above lines will show **** for password'
                }
            }
        }

        stage('Credentials Outside Block') {
            steps {
                echo "Trying to access credential outside withCredentials block..."
                sh 'echo API_TOKEN value outside block: ${API_TOKEN:-NOT_SET}'
                sh 'echo This proves credentials only exist INSIDE the block'
            }
        }

        stage('Multiple Credentials Together') {
            steps {
                withCredentials([
                    string(credentialsId: 'my-api-token', variable: 'API_TOKEN'),
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    echo "✅ Both credentials loaded in same block"
                    sh 'echo Using API token and Docker creds simultaneously'
                    sh 'echo Docker user: $DOCKER_USER'
                    sh 'echo API Token masked: $API_TOKEN'
                }
            }
        }

    }

    post {
        success {
            echo "✅ All credential stages passed! Secrets were masked correctly."
        }
        failure {
            echo "❌ Check which stage failed - likely credential ID mismatch"
        }
        always {
            deleteDir()
        }
    }
}
```

---

#### Step B4 — Save the File

Press `Ctrl + O` → Press `Enter` → Press `Ctrl + X`

---

#### Step B5 — Verify File Ends Correctly

```bash
tail -10 Jenkinsfile
```

Should end with:
```groovy
    }

}
```

---

#### Step B6 — Push to GitHub

```bash
git add Jenkinsfile
git commit -m "Day 5 lab: credentials withCredentials demo"
git push origin main
```

You should see `main -> main`.

---

### PART C — Run and Observe

#### Step C1 — Go to Jenkins

Open: `https://devopsbymuskan07.com/jenkins/`

---

#### Step C2 — Build Now

1. Click **`Day1_Declarative_Pipeline`**
2. Click **Build Now**
3. Click the new build number
4. Click **Console Output**

---

#### Step C3 — What You Should See

**Secret Text Stage:**
```
✅ Secret Text credential loaded successfully
+ echo The token value is: ****
The token value is: ****
```

The actual value `my-practice-api-token-12345` is replaced with `****`.

**Username Password Stage:**
```
✅ Username/Password credential loaded
+ echo Username is: muskanpatel71198
Username is: muskanpatel71198
+ echo Password is: ****
Password is: ****
```

Username is visible. Password is masked. This is correct behaviour.

**Outside Block Stage:**
```
API_TOKEN value outside block: NOT_SET
This proves credentials only exist INSIDE the block
```

This proves credentials do NOT leak outside the `withCredentials` block.

**Multiple Credentials Stage:**
```
✅ Both credentials loaded in same block
Docker user: muskanpatel71198
API Token masked: ****
```

---

### PART D — Verify Credentials Are Stored Safely on Disk

Let's check how Jenkins stores credentials on disk — INSIDE the Jenkins master pod.

#### Step D1 — Go to Jenkins Master Pod

```bash
kubectl exec -it -n monitoring $(kubectl get pod -n monitoring -l app=jenkins -o jsonpath='{.items[0].metadata.name}') -- bash
```

---

#### Step D2 — Look at Credentials Directory

```bash
ls /var/jenkins_home/credentials.xml
cat /var/jenkins_home/credentials.xml
```

You will see your credentials stored as encrypted XML. The actual password is NOT visible — it is encrypted. It looks something like:

```xml
<secret>{AQAAABAAAAAwF+encrypted+content+here}</secret>
```

This proves Jenkins does NOT store plain text passwords.

---

#### Step D3 — Exit the Pod

```bash
exit
```

---

## ✅ Lab Checklist — Confirm All Before Moving to Day 6

- [ ] I added `my-api-token` as a Secret Text credential in Jenkins UI
- [ ] I added `dockerhub-creds` as Username/Password credential in Jenkins UI
- [ ] I updated the Jenkinsfile with withCredentials blocks
- [ ] I pushed to GitHub and ran the build
- [ ] I saw `****` in console output where password should be
- [ ] I saw `NOT_SET` for credential accessed outside withCredentials block
- [ ] I saw both credentials work in the Multiple Credentials stage
- [ ] I went inside Jenkins master pod and saw credentials.xml is encrypted
- [ ] Build finished: SUCCESS

---

## Troubleshooting

### Problem: "CredentialsNotFoundException: No credentials found with id my-api-token"
**Cause:** The credential ID in Jenkinsfile does not match what you saved in Jenkins UI.
**Fix:** Go to Jenkins → Manage Jenkins → Credentials → check the exact ID shown. Copy it exactly into your Jenkinsfile. IDs are case-sensitive.

### Problem: "Handshake error" on agent
**Cause:** Jenkins restarted and generated new agent secret.
**Fix:**
```bash
kubectl delete pod jenkins-agent -n monitoring --force
kubectl apply -f ~/jenkins-agent-WORKING.yaml
sleep 20 && kubectl logs jenkins-agent -n monitoring | tail -5
```
Wait for `INFO: Connected` then rebuild.

### Problem: Password is NOT masked — shows in plain text
**Cause:** You used the variable with double quotes in a Groovy string instead of single-quoted shell string.
- Wrong: `sh "echo $DOCKER_PASS"` ← Groovy interpolates before masking
- Correct: `sh 'echo $DOCKER_PASS'` ← Shell handles it, Jenkins can mask it

Always use **single quotes** in `sh` steps when referencing credential variables.

---

## Key Takeaways — Day 5

| Concept | What it means | Example |
|---------|--------------|---------|
| Jenkins Credentials Store | Secure vault for secrets | Manage Jenkins → Credentials |
| Secret Text | Single string secret | API token, webhook secret |
| Username/Password | Login credentials | Docker Hub, Git, Nexus |
| SSH Private Key | SSH authentication | Deploy to EC2 |
| `withCredentials` | Injects credentials into a block | `withCredentials([...]) { }` |
| `credentialsId` | The ID you gave when saving | `'dockerhub-creds'` |
| Masking | Jenkins replaces secret with `****` | Automatic, always |
| Scope | Credentials only exist inside block | Outside = NOT_SET |
| `_USR` / `_PSW` suffix | Auto-created for username/password | `CREDS_USR`, `CREDS_PSW` |
| Never hardcode | Secrets in code = security breach | Use credentialsId instead |

---

*Next: Day 6 — Triggers (Webhook, pollSCM, Cron, Upstream, Parameterized Builds)*
