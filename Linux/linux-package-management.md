# Package Management — Complete Study Guide with Labs

---

## What is a Package? — Easy Version

When you want to install software on Linux — like nginx, docker, or python —
you do NOT go to a website and download a .exe file like Windows.

Instead, Linux uses a **package manager** — a tool that:
- Downloads the software for you
- Installs it in the right place
- Handles all dependencies automatically
- Lets you update or remove it cleanly

A **package** = a zip file containing:
```
- The actual program files
- Where to put them on the system
- List of other software it needs (dependencies)
- Instructions for installing and removing
```

> **One line:** Package = software bundled with installation instructions.
> Package manager = the tool that handles installing, updating, removing packages.

---

## What is a Repository? — Easy Version

A **repository (repo)** = an online store/library of packages.

Think of it like an App Store on your phone:
- App Store = repository
- Downloading an app = installing a package
- App Store knows what apps exist and where to get them = repo has package list

```
Your Linux Server
      ↓ asks for nginx
Package Manager (apt or yum)
      ↓ goes to
Repository (online server with packages)
      ↓ downloads
nginx package + all its dependencies
      ↓ installs everything
nginx is ready to use
```

When you run `apt update` or `yum update` — you are refreshing the list of
available packages from the repository. Not installing anything yet, just updating the list.

---

## Two Families — apt and yum/dnf

```
Ubuntu / Debian family     → uses apt
RHEL / CentOS / Rocky      → uses yum (older) or dnf (newer)
Amazon Linux 2             → uses yum
Amazon Linux 2023          → uses dnf
```

> **Easy rule:** If you are on Ubuntu → use apt. Everything else → use yum or dnf.

---

## apt — Ubuntu/Debian Package Manager

### Most used commands

```bash
apt update
# Refresh the package list from repositories
# Does NOT install or upgrade anything
# Always run this FIRST before installing anything
# Like refreshing the App Store to see new apps

apt upgrade
# Upgrade all installed packages to latest version
# Run AFTER apt update

apt install nginx
# Install the nginx package
# apt automatically finds and installs all dependencies too

apt install nginx=1.18.0
# Install a SPECIFIC version of nginx

apt remove nginx
# Remove nginx but KEEP its config files

apt purge nginx
# Remove nginx AND delete all its config files
# Use this when you want a completely clean removal

apt autoremove
# Remove packages that were installed as dependencies
# but are no longer needed by anything

apt search nginx
# Search for packages related to nginx

apt show nginx
# Show detailed information about the nginx package
# (version, size, dependencies, description)

apt list --installed
# Show all currently installed packages

apt list --installed | grep nginx
# Check if nginx is installed

dpkg -l | grep nginx
# Another way to check if nginx is installed
# dpkg is the low-level tool under apt

dpkg -L nginx
# List ALL files that were installed by the nginx package
# Useful to find where config files or binaries are

dpkg -S /usr/sbin/nginx
# Find which package owns this specific file
```

### Repository config for apt

```bash
cat /etc/apt/sources.list
# Main repository list — where apt looks for packages

ls /etc/apt/sources.list.d/
# Additional repo files — each software can add their own repo here
# Example: Docker adds its own repo file here

# Adding a new repository (example — adding Docker repo):
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable"
apt update
apt install docker-ce
```

---

## yum — RHEL/CentOS/Amazon Linux Package Manager

### Most used commands

```bash
yum update -y
# Update all packages to latest version
# -y means "yes to everything, don't ask me to confirm"

yum install nginx -y
# Install nginx
# -y skips the "are you sure?" confirmation

yum install nginx-1.18.0 -y
# Install specific version

yum remove nginx
# Remove nginx

yum search nginx
# Search for packages related to nginx

yum info nginx
# Show detailed info about nginx package

yum list installed
# Show all installed packages

yum list installed | grep nginx
# Check if nginx is installed

rpm -qa | grep nginx
# rpm is the low-level tool under yum (like dpkg under apt)
# -q = query, -a = all packages

rpm -ql nginx
# List all files installed by nginx package

rpm -qf /usr/sbin/nginx
# Which package owns this file?

rpm -qi nginx
# Detailed package info

yum clean all
# Clear yum's cache — useful when you have download issues
# or when repo metadata seems stale
```

### yum groups — install a group of related packages

```bash
yum grouplist
# Show available package groups

yum groupinstall "Development Tools"
# Install all development tools at once (gcc, make, git, etc.)
```

---

## dnf — Modern yum (RHEL 8+, Amazon Linux 2023)

dnf is the newer, faster replacement for yum.
Commands are almost identical — just replace `yum` with `dnf`.

```bash
dnf update -y
dnf install nginx -y
dnf remove nginx
dnf search nginx
dnf info nginx
dnf list installed
dnf clean all

# dnf extra features:
dnf history
# Show history of all install/remove operations
# Very useful — you can see what was installed and when

dnf history undo 5
# Undo the 5th operation (roll back an install/update)

dnf update --security -y
# Install ONLY security updates — important for banking environments

dnf module list
# Show available modules (groups of related packages)
```

---

## Dependency Management — Easy Explanation

When you install nginx, nginx might need other packages to work.
Those other packages are called **dependencies**.

Package managers handle this automatically:

```bash
apt install nginx
# apt says:
# "nginx needs libpcre3, libssl, zlib — I will install all of them too"
# You do not need to install them manually

apt remove nginx
# Only removes nginx
# Dependencies stay (because something else might need them)

apt autoremove
# Removes dependencies that nothing needs anymore
# Safe to run after removing packages
```

What happens when you have a dependency conflict?

```bash
# Example error you might see:
# "Package X depends on libssl 1.0 but libssl 1.1 is installed"

# Fix options:
apt install -f
# -f = fix broken dependencies
# apt will try to resolve conflicts automatically

apt install packagename --fix-broken
# Another way to fix dependency issues
```

---

## Software Installation Methods — All 3 Ways

Not everything is in the official repository. Here are all the ways to install software:

### Method 1: Package manager (best — use this when possible)
```bash
apt install nginx          # Ubuntu
yum install nginx          # RHEL/CentOS
```

### Method 2: Download and install a .deb or .rpm file manually
```bash
# Ubuntu — download a .deb file and install it
wget https://example.com/software.deb
dpkg -i software.deb
apt install -f             # fix any dependency issues after

# RHEL/CentOS — download a .rpm file and install it
wget https://example.com/software.rpm
rpm -ivh software.rpm
# -i = install, -v = verbose, -h = show progress bars
```

### Method 3: Install from source (when no package exists)
```bash
# Download source code
wget https://example.com/software.tar.gz
tar -xzf software.tar.gz
cd software/

# Build and install
./configure
make
make install
```

---

## Package Updates — Security Best Practice

```bash
# Check what updates are available (without installing)
apt list --upgradable          # Ubuntu
yum check-update               # RHEL/CentOS

# Install only security updates (important for production servers)
apt upgrade --only-upgrade     # Ubuntu
dnf update --security -y       # RHEL8+/Amazon Linux 2023

# Check when packages were installed or updated
grep " install " /var/log/dpkg.log    # Ubuntu — package install history
grep "Installed" /var/log/yum.log     # RHEL — package install history

# Hold a package at current version (prevent auto-updates)
apt-mark hold nginx            # Ubuntu — never auto-update nginx
apt-mark unhold nginx          # Ubuntu — allow updates again

yum versionlock add nginx      # RHEL — lock nginx version
```

---

## Dummy Data for Labs

```bash
# We will practice installing, checking, and removing these packages:
# - tree    (shows directory structure as a tree — small, safe to install)
# - wget    (download files from internet)
# - curl    (another download tool)
# - htop    (better version of top)
# These are all safe to install and remove on any system
```

---

## Lab Practice

### Lab Setup
```bash
mkdir -p ~/linux-lab/package-lab
cd ~/linux-lab/package-lab
```

---

### Lab 1 — Check your Linux distribution first

```bash
# What distro am I on?
cat /etc/os-release

# Based on output:
# Ubuntu/Debian → use apt commands below
# RHEL/CentOS/Amazon Linux → use yum commands below
```

---

### Lab 2 — Update package list

```bash
# Ubuntu:
sudo apt update
# Watch the output — it shows which repositories it is checking
# At the end it says: "X packages can be upgraded"

# RHEL/CentOS/Amazon Linux:
sudo yum check-update
# Shows what updates are available
```

---

### Lab 3 — Search for a package before installing

```bash
# Ubuntu:
apt search tree
# Look for a package called "tree"
# Output shows package name, version, short description

apt show tree
# Show detailed info about tree package
# Shows: size, dependencies, what it does

# RHEL/CentOS:
yum search tree
yum info tree
```

---

### Lab 4 — Install a package

```bash
# Ubuntu:
sudo apt install tree -y
# -y = yes to everything

# RHEL/CentOS/Amazon Linux:
sudo yum install tree -y

# Verify it installed successfully
which tree
# Should show: /usr/bin/tree

tree --version
# Should show the version number

# Use it to see your lab folder as a tree
tree ~/linux-lab/
```

---

### Lab 5 — Check what is installed

```bash
# Ubuntu:
dpkg -l | grep tree
# Should show tree with "ii" at the start (ii = installed)

apt list --installed | grep tree

# RHEL/CentOS:
rpm -qa | grep tree

yum list installed | grep tree
```

---

### Lab 6 — Find where a package put its files

```bash
# Ubuntu:
dpkg -L tree
# Shows every file that was installed by the tree package
# You will see: /usr/bin/tree, /usr/share/man/...

# Which package owns /usr/bin/tree?
dpkg -S /usr/bin/tree

# RHEL/CentOS:
rpm -ql tree
rpm -qf /usr/bin/tree
```

---

### Lab 7 — Install multiple packages at once

```bash
# Ubuntu:
sudo apt install wget curl htop -y
# Installs all 3 in one command

# Verify all 3 installed:
which wget
which curl
which htop

# RHEL/CentOS:
sudo yum install wget curl htop -y
```

---

### Lab 8 — Remove a package

```bash
# Ubuntu — remove but keep config files:
sudo apt remove tree

# Check it is gone:
which tree
# Should say "not found"

dpkg -l | grep tree
# Should show "rc" (r=removed, c=config still there)

# Ubuntu — remove AND delete config files:
sudo apt purge tree
dpkg -l | grep tree
# Should show nothing now (completely gone)

# RHEL/CentOS:
sudo yum remove tree
rpm -qa | grep tree
# Should show nothing
```

---

### Lab 9 — autoremove (clean up unused dependencies)

```bash
# Ubuntu:
sudo apt autoremove
# Shows which packages are no longer needed
# Confirm with Y to remove them

# Check how much space you freed
df -h /
```

---

### Lab 10 — View package install history

```bash
# Ubuntu — see what was installed recently:
grep " install " /var/log/dpkg.log | tail -20
# Shows last 20 install operations with timestamps

# RHEL/CentOS:
cat /var/log/yum.log | tail -20
# or for dnf:
dnf history | head -20
```

---

### Lab 11 — Install a specific version

```bash
# Ubuntu — see what versions are available:
apt-cache policy nginx
# Shows: installed version, candidate version, all available versions

# Install specific version (example):
# sudo apt install nginx=1.18.0-0ubuntu1

# RHEL/CentOS — see available versions:
yum --showduplicates list nginx | head -10
```

---

### Lab 12 — Fix broken packages (simulation)

```bash
# Ubuntu — fix any broken dependencies:
sudo apt install -f
# This scans for broken installs and fixes them

# Check for broken packages:
dpkg --audit
# Shows packages that are not fully installed or have issues
```

---

## Interview Questions with Easy Answers

### Q1: What is a package manager and why do we use it?

**Say this:**
"A package manager is a tool that handles installing, updating, and removing software on Linux. Instead of manually downloading files from websites, the package manager connects to a repository — an online collection of packages — downloads the software, puts files in the right locations, and automatically handles any other software that is needed (dependencies). Ubuntu uses apt, RHEL and Amazon Linux use yum or dnf. I use it daily for installing tools on servers and keeping systems updated."

---

### Q2: What is the difference between apt update and apt upgrade?

**Say this:**
"apt update only refreshes the list of available packages from the repositories — it does not install or change anything. It is like refreshing the App Store to see what new versions exist. apt upgrade actually installs the newer versions of all packages that have updates available. I always run apt update first, then apt upgrade — in that order. Running upgrade without update first means you might install old versions because the list is stale."

---

### Q3: What is the difference between apt remove and apt purge?

**Say this:**
"apt remove removes the program files but keeps the configuration files on disk. So if I reinstall the package later, my old settings are still there. apt purge removes everything — the program AND all its config files. I use remove when I might reinstall later and want to keep my settings. I use purge when I want a completely clean removal — like when troubleshooting a broken install and starting fresh."

---

### Q4: What is a dependency? How does the package manager handle it?

**Say this:**
"A dependency is another package that your software needs to work. For example, nginx needs libssl to handle HTTPS connections — libssl is a dependency. The package manager handles this automatically — when I run apt install nginx, it reads the dependency list, finds all required packages, and installs them all in one step. I never need to manually track what nginx needs. After I remove a package, I run apt autoremove to clean up any dependencies that are no longer needed by anything."

---

### Q5: What is the difference between yum and dnf?

**Say this:**
"yum is the older package manager used on RHEL 7 and CentOS 7 and Amazon Linux 2. dnf is the newer, faster replacement used on RHEL 8+, Amazon Linux 2023, and newer systems. The commands are almost identical — just replace yum with dnf. dnf has some extra features: dnf history shows what was installed and when, dnf history undo can roll back an installation, and dnf update --security installs only security patches. In our banking environment I use dnf on newer servers for the security update feature specifically."

---

### Q6: How do you check if a package is installed?

**Say this:**
"On Ubuntu I use `dpkg -l | grep packagename` — if it shows 'ii' at the start it is installed. Or `apt list --installed | grep packagename`. On RHEL/CentOS I use `rpm -qa | grep packagename`. I can also just check if the command exists using `which packagename` — if it returns a path, it is installed. For more details I use `dpkg -L packagename` to see exactly which files were installed and where."

---

### Q7: How do you install security updates only without updating everything?

**Say this:**
"On RHEL/CentOS/Amazon Linux I use `dnf update --security -y` — this installs only patches marked as security updates and leaves everything else unchanged. On Ubuntu I use `apt upgrade --only-upgrade` or configure unattended-upgrades for automatic security patching. In our banking environment we are careful about this — we do not want to accidentally update a package version that might break the application, so installing only security patches is the safer approach for production servers."

---

### Scenario Q1: You run apt install nginx and get a dependency error. What do you do?

**Say this:**
"First I read the exact error message carefully — it usually tells me which dependency has a conflict. For example 'Package X requires libssl 1.0 but 1.1 is installed.' Step 1: try `apt install -f` — this tells apt to try and fix broken dependencies automatically. Step 2: if that does not work, `apt install --fix-broken`. Step 3: if there is a version conflict, I check what is installed with `dpkg -l | grep libssl` and what nginx specifically needs with `apt show nginx`. Step 4: sometimes I need to install a specific older or newer version of nginx that is compatible with what is already on the system. If all else fails, I check if there is a backports repository or a newer version of the conflicting library available."

---

### Scenario Q2: You need to install Docker on a fresh RHEL server that has no internet access (air-gapped). How do you do it?

**Say this:**
"On an air-gapped server with no internet, I cannot reach the online repository. I have a few options. Option 1 — copy the RPM file: I download the Docker RPM package and all its dependencies on a machine that HAS internet, copy the files to the air-gapped server using SCP, then install with `rpm -ivh docker-ce.rpm`. Option 2 — set up a local repository: I mirror the required packages to an internal server inside the network, then configure the air-gapped server's yum config to point to that internal server instead of the internet. I update /etc/yum.repos.d/ with the internal server URL and then `yum install docker-ce` works normally. Option 3 — use a tarball: some software like Docker also provides a static binary download with no dependencies — I download the tarball, extract it, and put the binaries in /usr/local/bin. In banking environments, air-gapped servers are common for security reasons, so knowing how to install software without internet is important."

---

### Scenario Q3: A package update broke your application. How do you roll back?

**Say this:**
"On RHEL/CentOS with dnf, the easiest way is `dnf history` to find the transaction number of the update, then `dnf history undo NUMBER` to reverse it. On Ubuntu, I first check `grep upgrade /var/log/dpkg.log` to see what was upgraded and when. Then I install the previous version explicitly: `apt install packagename=previous-version`. If I do not know the previous version, `apt-cache showpkg packagename` shows all available versions. For critical applications in production, I would also check if we have a system snapshot (LVM snapshot or cloud instance snapshot) taken before the update — that is the fastest full rollback if the damage is severe. Going forward, I use `apt-mark hold packagename` or `yum versionlock` to pin that package and prevent it from being automatically updated again."
