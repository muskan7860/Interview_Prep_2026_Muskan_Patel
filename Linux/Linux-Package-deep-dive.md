# Package Management — Deep Dive: Without Package Manager, Repositories & wget/curl

---

## Question 1 — Without a Package Manager, How Do You Install Software?

### First — Understand the Difference

```
WITH package manager:
apt install nginx
→ finds nginx online, downloads it, installs it, handles all dependencies
→ ONE command does everything

WITHOUT package manager:
→ YOU have to do all the steps manually
→ find the file online
→ download it
→ install it
→ handle dependencies yourself
```

There are 3 manual ways to install software without a package manager.

---

### Way 1 — Download a Pre-Built Package File (.rpm or .deb)

These are files the software company already built and packaged for you.
You just download and install them.

```
Official Website (nginx.org)
        ↓
Download .rpm file (for RHEL) or .deb file (for Ubuntu)
        ↓
Install using rpm or dpkg command manually
```

```bash
# ── RHEL/CentOS way (.rpm file) ──

# Step 1: Download the rpm file
wget https://nginx.org/packages/rhel/8/x86_64/RPMS/nginx-1.24.0-1.el8.ngx.x86_64.rpm

# Step 2: Install it
rpm -ivh nginx-1.24.0-1.el8.ngx.x86_64.rpm
# -i = install
# -v = verbose (show me what is happening)
# -h = hash marks (show progress bar)

# Step 3: Verify it installed
rpm -qa | grep nginx
nginx --version


# ── Ubuntu way (.deb file) ──

# Step 1: Download the deb file
wget https://nginx.org/packages/ubuntu/pool/nginx/n/nginx/nginx_1.24.0-1~focal_amd64.deb

# Step 2: Install it
dpkg -i nginx_1.24.0-1~focal_amd64.deb
# -i = install

# Step 3: Fix dependencies if dpkg complains
apt install -f
# -f = fix broken dependencies

# Step 4: Verify
dpkg -l | grep nginx
nginx --version
```

**The BIG problem with this method:**

```
You install nginx rpm
        ↓
rpm says ERROR:
"nginx needs libssl, libpcre, zlib — not found!"
        ↓
You manually download libssl rpm
        ↓
libssl says ERROR:
"libssl needs libcrypto — not found!"
        ↓
You manually download libcrypto...
        ↓
This goes on and on — called "dependency hell"
```

This is exactly why package managers were invented — to escape this loop automatically.

---

### Way 2 — Download Source Code and Build It Yourself

Source code = the raw code files written by developers.
It is NOT a working program yet — you have to compile it (turn code into a working program).

```
Download source code (raw .tar.gz file)
        ↓
Extract it
        ↓
./configure  ← checks your system is ready to build
        ↓
make         ← compiles the code (turns it into a working program)
        ↓
make install ← copies the built program to the right locations
```

```bash
# Example: installing nginx from source code

# Step 1: Install build tools first (you need these to compile anything)
# Ubuntu:
apt install build-essential libpcre3-dev libssl-dev zlib1g-dev -y
# RHEL:
yum groupinstall "Development Tools" -y
yum install pcre-devel openssl-devel zlib-devel -y

# Step 2: Download source code
wget https://nginx.org/download/nginx-1.24.0.tar.gz

# Step 3: Extract
tar -xzf nginx-1.24.0.tar.gz
ls
# You will see a new folder: nginx-1.24.0/

cd nginx-1.24.0/
ls
# You will see: configure, Makefile, src/, auto/, etc.

# Step 4: Configure
# This checks your system has everything needed and prepares the build
./configure --prefix=/etc/nginx --sbin-path=/usr/sbin/nginx
# --prefix = where config files will go
# --sbin-path = where the nginx binary will be installed

# Step 5: Compile (this can take 1-5 minutes)
make
# You will see lots of output as code is being compiled
# At the end: make[1]: Leaving directory = success

# Step 6: Install
make install
# Copies the compiled files to the locations specified in configure

# Step 7: Verify
nginx -v
which nginx
```

**When do you use source build?**
```
✅ When no .rpm or .deb package exists for your OS version
✅ When you need to enable/disable specific features during build
✅ When you need a very specific version not available anywhere else
✅ When your server has no internet — you build on one machine, copy binary to others
❌ Hard to update later
❌ Hard to track what is installed (package manager does not know about it)
❌ You must manually handle all dependencies
```

---

### Way 3 — Pre-Compiled Binary (Download and Run Directly)

Some tools come as a single file that works immediately — no installation, no compilation.
These are called **static binaries** — everything the tool needs is bundled inside one file.

```bash
# Example 1: kubectl (Kubernetes command-line tool)

# Step 1: Download the binary
curl -LO https://dl.k8s.io/release/v1.28.0/bin/linux/amd64/kubectl

# Step 2: Make it executable (by default downloaded files are not executable)
chmod +x kubectl

# Step 3: Move to /usr/local/bin so you can run it from anywhere
mv kubectl /usr/local/bin/

# Step 4: Test
kubectl version --client


# Example 2: terraform

# Step 1: Download
wget https://releases.hashicorp.com/terraform/1.6.0/terraform_1.6.0_linux_amd64.zip

# Step 2: Unzip
unzip terraform_1.6.0_linux_amd64.zip
# This gives you a single file called: terraform

# Step 3: Make executable and move to PATH
chmod +x terraform
mv terraform /usr/local/bin/

# Step 4: Test
terraform version


# Example 3: downloading a Docker image as a tar file (air-gapped server)
# On a machine WITH internet:
docker pull nginx:latest
docker save nginx:latest -o nginx-image.tar

# Copy to air-gapped server:
scp nginx-image.tar user@airgapped-server:/tmp/

# On the air-gapped server (no internet):
docker load -i /tmp/nginx-image.tar
docker images | grep nginx
```

**Why this is the easiest manual method:**
```
✅ Single file — nothing else needed
✅ No dependencies
✅ No compilation
✅ Works immediately after chmod +x
✅ Remove it by just deleting the file
```

---

### All 3 Methods — Summary Table

```
Method              Command              Use when
────────────────────────────────────────────────────────────────────
apt / yum / dnf     apt install X        ALWAYS try this first
.rpm / .deb file    rpm -ivh X.rpm       Package exists but not in repo
                    dpkg -i X.deb        (common for enterprise software)
Binary download     chmod +x + mv        Single-file tools like kubectl,
                                         terraform, helm, github-cli
Source build        ./configure          No package or binary available,
                    make                 or you need custom build options
                    make install
```

---

## Question 2 — How Does the Repository Work? How Does It Stay Updated?

### Part A — What is Inside a Repository?

A repository is simply a **web server** (just a website) that stores packages and an index.

```
Repository Server (example: archive.ubuntu.com)
│
├── dists/
│   └── focal/                    ← Ubuntu version folder (focal = Ubuntu 20.04)
│       ├── Release                ← metadata: version, date, checksums
│       ├── main/
│       │   └── binary-amd64/
│       │       └── Packages.gz    ← THE INDEX FILE — list of ALL packages
│       └── universe/
│           └── binary-amd64/
│               └── Packages.gz    ← another index for community packages
│
└── pool/                          ← actual package files stored here
    ├── main/
    │   ├── n/nginx/
    │   │   ├── nginx_1.18.0.deb   ← older version
    │   │   └── nginx_1.24.0.deb   ← newer version
    │   └── d/docker/
    │       └── docker-ce_24.0.deb
    └── universe/
        └── h/htop/
            └── htop_3.2.2.deb
```

**The Packages.gz index file contains:**
```
Package: nginx
Version: 1.24.0
Architecture: amd64
Depends: libpcre3, libssl1.1, zlib1g
Filename: pool/main/n/nginx/nginx_1.24.0.deb
Size: 587432
SHA256: abc123def456...
Description: high performance web server
```

---

### Part B — How apt/yum Knows WHERE the Repository Is

The repository URL is stored in a config file on YOUR server.

```bash
# Ubuntu — repository URLs are here:
cat /etc/apt/sources.list

# Example content:
# deb http://archive.ubuntu.com/ubuntu focal main restricted universe
#  ↑    ↑                               ↑     ↑
#  type URL of the repo server          OS    sections
#
# deb-src http://archive.ubuntu.com/ubuntu focal main
# (deb-src = source code packages)

# Additional repos go here (one file per repo):
ls /etc/apt/sources.list.d/
# Example files you might see:
# docker.list       ← Docker's own repository
# google-chrome.list
# nodesource.list   ← NodeJS repository

cat /etc/apt/sources.list.d/docker.list
# deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable
#                   ↑ Docker's OWN repository URL
```

```bash
# RHEL/CentOS — repository configs are here:
ls /etc/yum.repos.d/
# CentOS-Base.repo
# CentOS-AppStream.repo
# docker-ce.repo
# epel.repo

cat /etc/yum.repos.d/CentOS-Base.repo
# [BaseOS]
# name=CentOS Linux $releasever - BaseOS
# baseurl=http://mirror.centos.org/centos/$releasever/BaseOS/x86_64/os/
#                                          ↑ $releasever is replaced with 7 or 8
# enabled=1
# gpgcheck=1
# gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-centosofficial
```

---

### Part C — Full Flow: What Happens When You Run apt update

```
Step 1: apt reads /etc/apt/sources.list
        → finds URL: http://archive.ubuntu.com/ubuntu

Step 2: apt connects to archive.ubuntu.com
        → downloads dists/focal/main/binary-amd64/Packages.gz
        → this is just the INDEX (list of packages), not the packages themselves

Step 3: apt saves index locally at:
        /var/lib/apt/lists/archive.ubuntu.com_ubuntu_dists_focal_main_binary-amd64_Packages

Step 4: apt update is DONE
        → your server now knows what packages exist and their latest versions
        → no packages were downloaded yet

Step 5: You run: apt install nginx
        → apt checks local index: "nginx 1.24.0 is at pool/main/n/nginx/nginx_1.24.0.deb"
        → apt downloads only that file from archive.ubuntu.com
        → apt installs it
```

```bash
# See the locally stored index files:
ls /var/lib/apt/lists/
# You will see many files — one per repository section

# See how much space the index files take:
du -sh /var/lib/apt/lists/

# Clear the index (to force fresh download on next apt update):
apt clean
```

---

### Part D — How Repository Knows About RHEL 7 vs RHEL 8 (Different Versions)

The magic is in the URL — each OS version has its OWN folder on the repo server.

```
RHEL/CentOS 7 repo:
http://mirror.centos.org/centos/7/BaseOS/x86_64/os/
                                    ↑
                             "7" folder = CentOS 7 packages

RHEL/CentOS 8 repo:
http://mirror.centos.org/centos/8/BaseOS/x86_64/os/
                                    ↑
                             "8" folder = CentOS 8 packages

Ubuntu 20.04 (Focal) repo:
http://archive.ubuntu.com/ubuntu/dists/focal/
                                          ↑
                                   "focal" folder

Ubuntu 22.04 (Jammy) repo:
http://archive.ubuntu.com/ubuntu/dists/jammy/
                                          ↑
                                   "jammy" folder
```

**How the config file knows which version to use:**

```bash
# RHEL automatically detects version and puts it in the config:
cat /etc/yum.repos.d/CentOS-Base.repo | grep baseurl
# baseurl=http://mirror.centos.org/centos/$releasever/BaseOS/...
#                                          ↑
#                              $releasever is a variable

# See what $releasever equals on your system:
cat /etc/redhat-release
# CentOS Linux release 8.5.2111

# So yum replaces $releasever with 8
# URL becomes: http://mirror.centos.org/centos/8/BaseOS/...
```

---

### Part E — How Does the Repo Get New Packages When Software Updates?

```
Day 1: nginx team releases nginx 1.25
        ↓
nginx team builds .rpm and .deb files for all supported OS versions
        ↓
They EITHER:

PATH A (Software has its own repo — like Docker, nginx):
nginx team uploads nginx_1.25.rpm to their own repo server (nginx.org)
They update the Packages index on their server
Next time you run apt update / yum update
→ your server downloads the new index
→ sees nginx 1.25 is available
→ apt upgrade installs it

PATH B (Software goes through Ubuntu/RHEL official repo):
nginx team submits package to Ubuntu/RHEL team
Ubuntu/RHEL team TESTS the package (can take weeks/months)
After testing, they add it to their repo server
→ your server gets it on next apt update

Timeline difference:
nginx.org own repo  → nginx 1.25 available SAME DAY as release
Ubuntu official repo → nginx 1.25 available weeks/months later (after Ubuntu testing)
```

**This is why you add Docker's OWN repo:**
```bash
# Ubuntu official repo has old Docker version (tested but old)
apt show docker.io
# Version: 20.10 (old)

# Docker's own repo has the latest version (always up to date)
# So you add Docker's repo:
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable"
apt update
apt install docker-ce
# Version: 24.0 (latest)
```

---

### Part F — GPG Keys — Why They Exist (Security)

When apt downloads a package, how does it know the file was not modified by a hacker?
Answer: GPG signature verification.

```
Software team signs their packages with a private key (like a digital stamp)
        ↓
They publish their PUBLIC key (so anyone can verify)
        ↓
When you download a package, apt checks:
"Does this package match the signature from the trusted public key?"
        ↓
If YES → install safely
If NO  → refuse to install (someone may have tampered with it)
```

```bash
# Add Docker's GPG key (trust Docker's packages):
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -

# List all trusted keys:
apt-key list

# RHEL checks GPG automatically using:
# gpgcheck=1 in /etc/yum.repos.d/*.repo files
```

---

## Question 3 — What Actually Happens When You Use wget or curl?

### The Simple Picture First

```
Your Linux Server                         Remote Server (nginx.org)
        │                                         │
        │ Step 1: DNS — find IP of nginx.org      │
        │ ───────────────────────────────────>    │
        │                                         │
        │ Step 2: TCP connect to that IP:443      │
        │ ───────────────────────────────────>    │
        │                                         │
        │ Step 3: TLS handshake (encryption setup)│
        │ <──────────────────────────────────>    │
        │                                         │
        │ Step 4: Send HTTP GET request           │
        │ "GET /download/nginx-1.24.tar.gz"       │
        │ ───────────────────────────────────>    │
        │                                         │
        │ Step 5: Server sends file bytes back    │
        │ <───────────────────────────────────    │
        │                                         │
        │ Step 6: wget/curl saves bytes to disk   │
        └─────────────────────────────────────────┘
```

---

### Step by Step — What Happens When You Run wget

```bash
wget https://nginx.org/download/nginx-1.24.0.tar.gz
```

**Step 1 — DNS Lookup (finding the address)**
```
wget reads the URL: https://nginx.org/download/nginx-1.24.0.tar.gz
Extracts the hostname: nginx.org

wget asks Linux: "what is the IP address of nginx.org?"
Linux checks /etc/resolv.conf → finds DNS server (example: 8.8.8.8)
Linux asks DNS server: "what IP is nginx.org?"
DNS server replies: "nginx.org = 52.58.199.22"

Now wget knows where to connect
```

```bash
# You can do this DNS lookup manually:
dig nginx.org
# Shows: nginx.org. 300 IN A 52.58.199.22
#                           ↑ this is the IP address

nslookup nginx.org
# Another way to do DNS lookup
```

**Step 2 — TCP Connection (opening a channel)**
```
wget connects to 52.58.199.22 on port 443 (https = always port 443)

TCP handshake happens — 3 messages:
Your server → nginx.org: "SYN" (I want to connect)
nginx.org → your server: "SYN-ACK" (OK, I accept)
Your server → nginx.org: "ACK" (Great, connected!)

Now a reliable two-way channel is open
```

**Step 3 — TLS Handshake (encryption setup, because it is https)**
```
wget and nginx.org exchange security certificates
They agree on which encryption method to use
They generate shared secret keys for this session
All data from here on is encrypted

If it were http:// (no s) — this step is skipped, no encryption
```

**Step 4 — HTTP Request (asking for the file)**
```
wget sends this message to nginx.org:

GET /download/nginx-1.24.0.tar.gz HTTP/1.1
Host: nginx.org
User-Agent: Wget/1.21.2
Accept: */*
Connection: keep-alive
```

**Step 5 — Server Responds (sends the file)**
```
nginx.org server sends back:

HTTP/1.1 200 OK
Content-Type: application/x-gzip
Content-Length: 1045231
Last-Modified: Tue, 24 Oct 2023 15:00:00 GMT

[bytes of the actual file come here...]
[bytes...]
[more bytes...]
[bytes finish]
```

**Step 6 — wget Saves to Disk**
```
wget receives all the bytes
Writes them to: nginx-1.24.0.tar.gz in your current folder
Shows a progress bar during download
When all bytes received: "nginx-1.24.0.tar.gz saved"
```

---

### wget vs curl — Easy Comparison

```
wget                                    curl
────────────────────────────────────────────────────────────────────
Downloads files and saves them          Downloads files OR sends data
Saves to disk automatically             Shows on screen by default
Good for simple downloads               Good for APIs and scripts
Can resume interrupted downloads        More control over requests
Simpler to use                          More powerful and flexible
```

**wget examples:**
```bash
wget https://example.com/file.zip
# Downloads and saves as file.zip automatically

wget -O myfile.zip https://example.com/file.zip
# Save with a different name

wget -c https://example.com/bigfile.zip
# -c = continue/resume an interrupted download

wget -q https://example.com/file.zip
# -q = quiet mode, no output, no progress bar
# Good for use in scripts
```

**curl examples:**
```bash
curl https://example.com/file.zip -o file.zip
# -o = save to this filename (needed, curl does not save automatically)

curl -L https://example.com/file.zip -o file.zip
# -L = follow redirects (if URL redirects to another URL, follow it)

curl -I https://google.com
# -I = show only HTTP headers, do not download the page
# Useful to check if a website is up without downloading everything

curl -v https://google.com
# -v = verbose — show EVERYTHING
# Shows DNS lookup, connection, headers, response
# Great for debugging connection problems

curl -fsSL https://example.com/script.sh
# -f = fail silently if server returns error
# -s = silent (no progress bar)
# -S = but show errors if they happen
# -L = follow redirects
# This combination is very common in DevOps scripts

curl -X POST https://api.example.com/data \
  -H "Content-Type: application/json" \
  -d '{"key": "value"}'
# Send data to an API endpoint
# -X = specify HTTP method (GET, POST, PUT, DELETE)
# -H = add a header
# -d = data to send
```

---

### Real DevOps Example — Adding Docker Repository

This uses curl in a real-world way that connects all concepts together:

```bash
# Step 1: Download Docker's GPG key using curl
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
# curl downloads the key file
# | pipes it directly to apt-key add
# apt-key stores it as a trusted key

# Step 2: Add Docker's repository URL to your sources
echo "deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable" \
  > /etc/apt/sources.list.d/docker.list

# Step 3: Update package list (apt now knows about Docker's packages)
apt update

# Step 4: Install Docker using apt (which now uses Docker's repo)
apt install docker-ce -y

# What happened:
# curl fetched the security key from Docker's server
# apt now trusts packages signed by that key
# Your sources.list.d/docker.list tells apt where Docker's repo is
# apt update downloaded Docker's Packages.gz index
# apt install docker-ce found docker-ce in that index and downloaded it
```

---

### HTTP Status Codes — What Server Sends Back

When curl or wget sends a request, server replies with a status code:

```
200 OK              → file found, here it is
301 Moved           → URL has moved permanently to new address (curl -L follows this)
302 Found           → temporary redirect
400 Bad Request     → your request had an error
401 Unauthorized    → you need to log in
403 Forbidden       → you do not have permission
404 Not Found       → file does not exist at that URL
500 Server Error    → something broken on the server side
```

```bash
# Check status code without downloading the file:
curl -o /dev/null -sw "%{http_code}\n" https://google.com
# Output: 200

curl -o /dev/null -sw "%{http_code}\n" https://google.com/fakepage
# Output: 404

# This is very useful in health check scripts:
STATUS=$(curl -o /dev/null -sw "%{http_code}" https://myapp.com/health)
if [ "$STATUS" = "200" ]; then
    echo "App is UP"
else
    echo "App is DOWN — status: $STATUS"
fi
```

---

## Lab Practice

### Lab Setup
```bash
mkdir -p ~/linux-lab/download-lab
cd ~/linux-lab/download-lab
```

---

### Lab 1 — wget basics

```bash
# Download a small test file
wget https://www.google.com/robots.txt

# See what was downloaded
ls -la
cat robots.txt

# Download and save with a different name
wget -O my-robots.txt https://www.google.com/robots.txt
ls -la

# Download quietly (no output — good for scripts)
wget -q https://www.google.com/robots.txt -O quiet-download.txt
ls -la
```

---

### Lab 2 — curl basics

```bash
# Show content on screen (curl default)
curl https://www.google.com/robots.txt

# Save to a file
curl https://www.google.com/robots.txt -o my-robots-curl.txt

# Show only the HTTP headers (is the server up?)
curl -I https://www.google.com

# Verbose — see everything that happens
curl -v https://www.google.com 2>&1 | head -40
# 2>&1 combines error and normal output so you see everything
```

---

### Lab 3 — Check HTTP status codes

```bash
# Check if a website is up (get only the status code)
curl -o /dev/null -sw "%{http_code}\n" https://www.google.com
# Expected: 200

# Check a page that does not exist
curl -o /dev/null -sw "%{http_code}\n" https://www.google.com/this-does-not-exist
# Expected: 404

# Write a simple health check
URL="https://www.google.com"
STATUS=$(curl -o /dev/null -sw "%{http_code}" $URL)
echo "Status for $URL: $STATUS"
if [ "$STATUS" = "200" ]; then
    echo "RESULT: Website is UP"
else
    echo "RESULT: Website is DOWN"
fi
```

---

### Lab 4 — DNS lookup (what happens before wget connects)

```bash
# Look up IP address of a domain
dig google.com | grep "^google"
# Shows: google.com. 300 IN A 142.250.x.x

# Simple lookup
nslookup google.com

# Check your DNS server (where your server asks for lookups)
cat /etc/resolv.conf
# nameserver 8.8.8.8 ← this is the DNS server your machine uses

# Trace the whole DNS chain
dig google.com +trace | head -30
```

---

### Lab 5 — Download a binary tool manually (kubectl simulation)

```bash
# Check if curl is available
which curl

# Download kubectl binary
curl -LO https://dl.k8s.io/release/v1.28.0/bin/linux/amd64/kubectl

# See the file
ls -la kubectl
file kubectl
# Should say: ELF 64-bit LSB executable (this means it is a compiled binary)

# Make it executable
chmod +x kubectl

# Test it (without moving to PATH)
./kubectl version --client

# Move to PATH so you can run from anywhere
sudo mv kubectl /usr/local/bin/

# Now test from anywhere
kubectl version --client
```

---

### Lab 6 — Understand repository config files

```bash
# Ubuntu — see your repo list
cat /etc/apt/sources.list

# See any extra repos
ls /etc/apt/sources.list.d/

# See the locally stored index (downloaded by apt update)
ls /var/lib/apt/lists/ | head -10

# How big is the index?
du -sh /var/lib/apt/lists/
```

```bash
# RHEL/CentOS/Amazon Linux — see your repo list
ls /etc/yum.repos.d/
cat /etc/yum.repos.d/*.repo | head -30

# See the locally stored index
ls /var/cache/yum/ 2>/dev/null || ls /var/cache/dnf/
```

---

### Lab 7 — Install from .rpm or .deb file (download and manual install)

```bash
# Ubuntu — download and install a .deb file manually
# We will use the tree package as an example

# First remove it if already installed
sudo apt remove tree -y 2>/dev/null

# Find the download URL (from packages.ubuntu.com)
# Download it
wget http://archive.ubuntu.com/ubuntu/pool/universe/t/tree/tree_2.0.2-1_amd64.deb

# Install manually using dpkg (not apt)
sudo dpkg -i tree_2.0.2-1_amd64.deb

# Verify
tree --version
dpkg -l | grep tree
```

---

## Interview Questions with Easy Answers

### Q1: How do you install software on Linux without internet access (air-gapped server)?

**Say this:**
"There are a few approaches I have used in banking environments where servers have no internet. First option — copy the package file: I download the .rpm or .deb file on a machine that has internet, then scp it to the air-gapped server and install with rpm -ivh or dpkg -i. The challenge is handling dependencies — I need to download those too. Second option — set up an internal repository mirror: I create a local repo server inside the network that mirrors the official packages. I then update /etc/yum.repos.d/ on all servers to point to the internal server instead of the internet. Third option — for single-file tools like kubectl or terraform, I download the binary on an internet machine, scp it across, chmod +x, and put it in /usr/local/bin. In our banking project, we had an internal Nexus repository server that mirrored all approved packages — every server pointed to Nexus instead of the internet."

---

### Q2: What is the difference between wget and curl?

**Say this:**
"Both download files from the internet, but they work differently. wget is simpler — it downloads a file and saves it automatically. curl is more powerful and flexible — it can download files, send data to APIs, show HTTP headers, follow redirects, and is better for use in scripts. In practice: I use wget for simple file downloads like wget https://example.com/file.zip. I use curl in scripts because it gives more control — for example in a health check script: curl -o /dev/null -sw '%{http_code}' URL gives me just the HTTP status code. curl is also the standard way to add repository keys, like when setting up Docker: curl -fsSL URL | apt-key add -."

---

### Q3: What actually happens when you run wget or curl to download a file?

**Say this:**
"Four things happen in order. First — DNS lookup: the tool extracts the hostname from the URL and asks the DNS server for its IP address. Second — TCP connection: it connects to that IP on port 80 (http) or 443 (https). Third — if it is https, a TLS handshake happens to set up encryption. Fourth — an HTTP GET request is sent asking for the specific file path. The server responds with the file bytes, and wget or curl saves them to disk. I can see all of this by running curl -v which shows every step — DNS resolution, connection, SSL handshake, request and response headers. This is useful for debugging why a download is failing."

---

### Q4: What is a GPG key and why do you add it when setting up a new repository?

**Say this:**
"A GPG key is a digital signature used to verify that the packages you download are genuine and have not been modified by anyone. When a software company like Docker publishes packages, they sign each package using their private key. When you add Docker's public GPG key to your system using apt-key add, you are telling apt: trust packages signed by this key. Before installing any package from Docker's repo, apt checks: does this package match Docker's signature? If someone intercepted the download and modified the package, the signature would not match and apt would refuse to install it. This is a security layer that protects against supply chain attacks."

---

### Q5: What is the difference between installing from source code versus installing a binary?

**Say this:**
"Installing from source means downloading the raw code files, compiling them on your machine using make, then installing. This gives you full control — you can enable or disable features during the configure step, and you can compile for your exact hardware. The downsides are: it is slow, you need build tools installed, you must handle dependencies manually, and the package manager does not know about what you installed so updates are manual. Installing a pre-compiled binary means someone else already compiled it and you just download the finished program file. Much faster and simpler. I use binary downloads for tools like kubectl, terraform, and helm — they are distributed as static binaries that work without any dependencies. I only use source builds as a last resort when no package or binary is available."

---

### Scenario Q1: curl is failing to download a file. How do you debug it?

**Say this:**
"Step 1: add -v to the curl command — `curl -v URL` — this shows every step: DNS lookup, TCP connection, TLS handshake, HTTP request and response. I read the output top to bottom and find where it fails. Step 2: check the HTTP status code — if I see 404, the URL is wrong or the file does not exist. If I see 403, I do not have permission. If I see 000 or connection refused, the server is unreachable. Step 3: test basic connectivity — can I ping the server? `ping hostname`. Can I reach the port? `nc -zv hostname 443`. Step 4: check DNS — does the hostname resolve? `dig hostname`. Step 5: check if a proxy is required — in banking environments, servers often need to go through an HTTP proxy. I check if the `http_proxy` and `https_proxy` environment variables are set, and if not, set them: `export https_proxy=http://proxy.internal:8080`."
