# Jenkins Plugin Installation Troubleshooting (Kubernetes)

## Objective

Enable the **GitHub Branch Source Plugin** so that a Jenkins Multibranch Pipeline automatically scans and builds when code is pushed to GitHub.

---

# Initial Problem

While installing the **GitHub Branch Source Plugin**, Jenkins showed:

```
Plugin is missing: jjwt-api
```

and

```
java.net.SocketException: Network is unreachable
```

As a result,

- GitHub Branch Source Plugin failed to load.
- Webhooks could not be configured properly.
- Multibranch Pipeline was not automatically detecting new branches.

---

# Step 1: Verify Internet Connectivity

Entered the Jenkins pod.

```bash
kubectl exec -it <jenkins-pod> -n monitoring -- sh
```

Checked connectivity:

```bash
curl https://updates.jenkins.io
```

Result:

- Successfully received HTML page.

Conclusion:

✅ Jenkins Pod has Internet connectivity.

---

# Step 2: Test Plugin Download URL

Executed:

```bash
curl -I https://updates.jenkins.io/download/plugins/jjwt-api/0.13.0-141.vd58b_a_9592b_6c/jjwt-api.hpi
```

Result:

```
HTTP/2 302
Location:
https://get.jenkins.io/plugins/...
```

Observation:

Plugin download uses HTTP Redirect.

Flow:

```
updates.jenkins.io
        │
        ▼
302 Redirect
        ▼
get.jenkins.io
```

---

# Step 3: Follow Redirect

Executed:

```bash
curl -L -I https://updates.jenkins.io/download/plugins/jjwt-api/0.13.0-141.vd58b_a_9592b_6c/jjwt-api.hpi
```

Output:

```
302
↓

get.jenkins.io

↓

302

↓

mirror.bom2.albony.in

↓

HTTP 200 OK
```

Conclusion:

The complete download chain is:

```
updates.jenkins.io
        │
        ▼
get.jenkins.io
        │
        ▼
Mirror Server
        │
        ▼
Plugin Download
```

---

# Step 4: Check DNS Resolution

Inside Jenkins Pod:

```bash
getent hosts get.jenkins.io
```

Output:

```
2603:1030:408:3::2bd
```

Observation:

Only IPv6 address was returned.

---

# Step 5: Test Direct Connection

Executed:

```bash
curl https://get.jenkins.io
```

Result:

```
Failed to connect
```

Initially this suggested a possible IPv6 connectivity issue inside the pod.

---

# Step 6: Verify from Host Machine

On Ubuntu host:

```bash
getent hosts get.jenkins.io
```

Result:

```
2603:1030:408:3::2bd
```

Then:

```bash
curl -I https://get.jenkins.io
```

Result:

```
HTTP/2 200 OK
```

Observation:

Host machine can reach `get.jenkins.io`.

---

# Step 7: Check Java Environment

Inside Jenkins Pod:

```bash
java -version
```

Result:

```
OpenJDK 21
```

---

Checked JVM properties:

```bash
java -XshowSettings:properties -version
```

No abnormal networking configuration found.

---

# Step 8: Check Proxy

Executed:

```bash
env | grep -i proxy
```

Output:

No proxy variables.

Meaning:

```
HTTP_PROXY
HTTPS_PROXY
NO_PROXY
```

were not configured.

---

# Step 9: Verify Jenkins Process

Executed:

```bash
ps -ef | grep java
```

Result:

Jenkins running normally using:

```
jenkins.war
```

No custom JVM networking options were configured.

---

# Step 10: Kubernetes DNS

Checked CoreDNS:

```bash
kubectl get pods -n kube-system
```

Output:

```
coredns
Running
```

Meaning:

Cluster DNS service is healthy.

---

# Findings

The following components were verified:

✅ Internet connectivity

✅ DNS service (CoreDNS)

✅ Java installation

✅ Jenkins process

✅ Plugin redirect chain

Yet Jenkins Plugin Manager still failed with:

```
java.net.SocketException:
Network is unreachable
```

---

# Root Cause

Jenkins Plugin Manager was unable to successfully download the dependency plugin:

```
jjwt-api
```

Since **GitHub Branch Source Plugin** depends on it, Jenkins reported:

```
Plugin is missing:
jjwt-api
```

Similarly,

```
Matrix Project Plugin
```

could not load because

```
junit
```

plugin was missing.

Dependency chain:

```
GitHub Branch Source
        │
        ▼
Requires

jjwt-api

Missing

↓

Plugin cannot load
```

---

# Practical Solution

Instead of relying on Plugin Manager to download plugins dynamically,

Install required plugins manually.

Required plugins:

- jjwt-api
- junit

Copy them into:

```
/var/jenkins_home/plugins/
```

Restart Jenkins.

This resolves dependency loading.

---

# Alternative Production Approach

If Jenkins is installed using Helm, specify plugins during deployment.

Example:

```yaml
controller:
  installPlugins:
    - github
    - github-branch-source
    - jjwt-api
    - junit
```

This avoids runtime plugin download failures.

---

# Lessons Learned

## Plugin Manager Flow

```
Install Plugin
      │
      ▼
Check Dependencies
      │
      ▼
Download Dependency
      │
      ▼
Install Dependency
      │
      ▼
Load Plugin
```

If any dependency fails,

```
Plugin remains disabled.
```

---

## Plugin Dependencies

Plugins are not standalone.

Example:

```
GitHub Branch Source

depends on

↓

jjwt-api

↓

GitHub API

↓

Git Plugin
```

If one dependency is missing,

the entire plugin fails to load.

---

## Kubernetes Debugging Commands Used

```bash
kubectl exec -it <pod> -n monitoring -- sh
```

```bash
curl https://updates.jenkins.io
```

```bash
curl -I https://updates.jenkins.io
```

```bash
curl -L -I <plugin-url>
```

```bash
getent hosts get.jenkins.io
```

```bash
env | grep -i proxy
```

```bash
java -version
```

```bash
java -XshowSettings:properties -version
```

```bash
ps -ef | grep java
```

```bash
kubectl get pods -n kube-system
```

---

# Interview Questions

### Why does a Jenkins plugin fail to load?

- Missing dependency plugins.
- Incompatible Jenkins version.
- Failed plugin download.
- Corrupted plugin file.
- Network issues during installation.

---

### What is a plugin dependency?

A plugin may require another plugin to provide classes or APIs. Jenkins checks dependencies before enabling a plugin.

---

### Why did the GitHub Branch Source Plugin fail?

Because its required dependency **jjwt-api** was not installed successfully.

---

### Why was checking with `curl` useful?

It verified:

- Internet connectivity.
- Redirect chain.
- DNS resolution.
- Mirror accessibility.

This helped isolate the issue from general network problems.

---

# Key Takeaways

- Always read the dependency error first.
- Verify network connectivity before blaming Jenkins.
- Understand the plugin download redirect chain.
- Check DNS, proxy, Java, and Kubernetes networking.
- Manual plugin installation is a valid workaround when Plugin Manager fails.
- Installing plugins during Helm deployment is the recommended production approach.
