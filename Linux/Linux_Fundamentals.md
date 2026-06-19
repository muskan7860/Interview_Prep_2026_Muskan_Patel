# Linux Fundamentals — Full Study Guide with Labs

---

## What is Linux?

Think of your computer like a building.
- The building has electricity, water, lifts → that is the **hardware** (CPU, RAM, disk)
- One person controls everything in the building → that is the **kernel**
- You talk to the front desk to get things done → that is the **shell**

**Linux = the kernel.** The kernel is the one program that controls all the hardware.

When people say "I use Linux" they usually mean a **distribution** — like Ubuntu or RHEL.
A distribution = Linux kernel + extra tools bundled together.

---

## Kernel vs Shell — easy picture

```
YOU (type a command)
        ↓
   SHELL (bash)       ← the box where you type
        ↓
   KERNEL             ← does the real work, talks to hardware
        ↓
   HARDWARE           ← CPU, RAM, Disk, Network card
```

- **Kernel** = controls hardware. You never talk to it directly.
- **Shell** = takes your typed command, passes it to the kernel, shows you the result.
- Think: shell is the waiter, kernel is the kitchen.

---

## The 5-layer Architecture — one line per layer

```
┌──────────────────────────────────────┐
│  Apps (nginx, docker, mysql)          │  ← things you use
├──────────────────────────────────────┤
│  Shell (bash, zsh)                    │  ← where you type commands
├──────────────────────────────────────┤
│  System Libraries (glibc)             │  ← helper code between apps and kernel
├──────────────────────────────────────┤
│  Kernel                               │  ← controls everything below
├──────────────────────────────────────┤
│  Hardware (CPU, RAM, Disk, Network)   │  ← physical machine
└──────────────────────────────────────┘
```

**Easy reading:** top = what you see and use. Bottom = physical machine.
Each layer just passes the work down to the one below it.

---

## What does the Kernel do? — 4 simple jobs

### Job 1: Manages CPU (decides who runs)
Many programs want to run at the same time. Kernel gives each one a tiny turn — so fast it feels like they all run together.

### Job 2: Manages Memory (RAM)
Each program gets its own space in RAM. Kernel makes sure one program cannot touch another program's memory.

### Job 3: Manages Files
Whether your file is on a hard disk, USB, or network — kernel makes it all look the same to you. You just say "read this file," kernel handles where it actually is.

### Job 4: Manages Network
When your app wants to connect to the internet — kernel does the actual sending and receiving of data packets.

> **One line to remember:** Kernel = CPU manager + memory manager + file handler + network handler.

---

## Boot Process — what happens when server powers on

Think of this as a relay race — each step passes control to the next:

```
Step 1: Power ON
        ↓
Step 2: BIOS/UEFI runs — checks hardware is working (takes 2-3 seconds)
        ↓
Step 3: GRUB bootloader — finds the kernel file on disk and loads it into RAM
        ↓
Step 4: Kernel starts — detects CPU, disks, network card, loads drivers
        ↓
Step 5: systemd starts (PID 1) — starts all services: SSH, networking, your app
        ↓
Step 6: Login screen appears — server is ready
```

**Commands to check boot:**
```bash
journalctl -b              # show all logs from current boot
journalctl -b -1           # show logs from PREVIOUS boot (useful when server crashed)
systemd-analyze            # show how long boot took in total
systemd-analyze blame      # show which service took the most time
```

---

## Linux Distributions — easy table

| Distribution | Package Manager | Where you see it |
|---|---|---|
| Ubuntu / Debian | apt | Cloud servers, containers |
| RHEL / CentOS / Rocky | yum or dnf | Banking, enterprise, on-prem |
| Amazon Linux 2 / 2023 | yum or dnf | AWS EC2 servers |
| Alpine | apk | Docker base images (very small, ~5MB) |

**Easy rule:** Ubuntu → apt. Everything else → yum or dnf.

```bash
# Ubuntu way to install software:
apt install nginx

# RHEL / CentOS / Amazon Linux way:
yum install nginx
```

---

## systemd Targets (old name: runlevels)

When Linux boots, it needs to know WHAT MODE to start in. These are called targets.

```
poweroff.target   → shutdown. Machine turns off.
rescue.target     → emergency mode. Only root, no network. Used when something is broken.
multi-user.target → normal server. Network is on. All services running. NO desktop.
graphical.target  → normal server + desktop screen. Servers usually don't use this.
reboot.target     → restarts the machine.
```

```bash
systemctl get-default                    # what mode does this server boot into?
systemctl set-default multi-user.target  # set normal server mode as default
systemctl isolate rescue.target          # switch to emergency mode RIGHT NOW
```

---

## First Commands — type these, don't just read

```bash
pwd          # "where am I right now?"
ls           # "what is inside this folder?"
ls -la       # "show me everything including hidden files, with all details"
cd /var/log  # "take me to this exact folder"
cd ~         # "take me to my home folder"
cd ..        # "go one step back/up"
cd -         # "take me back to where I just was"
clear        # "clean the screen"
whoami       # "who am I logged in as?"
man ls       # "show me the full manual for the ls command"
history      # "show me all commands I typed recently"
```

---

## Folder Map — where things live

```
/              → very top. Everything is inside here.
/etc           → ALL settings/config files live here
/home          → personal folders for each user (/home/ubuntu, /home/devops)
/root          → personal folder for the root (admin) user
/var/log       → ALL log files — first place you check when debugging
/var/lib       → app data files (mysql data, docker images)
/tmp           → temporary junk, gets cleaned automatically on reboot
/bin, /usr/bin → where the actual commands live (ls, cp, grep, etc.)
/proc          → VIRTUAL — live info about what's happening right now in the kernel
/dev           → device files (/dev/sda = first disk, /dev/null = black hole)
```

> **Easy rule:** settings broken → check /etc. Something happened → check /var/log.

---

## Lab 1 — Linux Fundamentals Practice

### Setup (run these to create lab files)
```bash
mkdir -p ~/linux-lab/fundamentals
cd ~/linux-lab/fundamentals
```

### Lab 1.1 — Navigation practice
```bash
# Step 1: see where you are
pwd
# Expected output: /home/yourname/linux-lab/fundamentals

# Step 2: list what's here
ls -la

# Step 3: go up one level
cd ..
pwd
# Expected output: /home/yourname/linux-lab

# Step 4: go back to where you were
cd -
pwd
# Expected output: /home/yourname/linux-lab/fundamentals

# Step 5: go home
cd ~
pwd
# Expected output: /home/yourname
```

### Lab 1.2 — Check boot info
```bash
# How long did this server take to boot?
systemd-analyze

# Which service was the slowest to start?
systemd-analyze blame | head -10

# Show logs from current boot, only errors
journalctl -b -p err | tail -20
```

### Lab 1.3 — Explore /proc (the live kernel window)
```bash
# See your CPU info
cat /proc/cpuinfo | grep "model name" | head -3

# See memory breakdown
cat /proc/meminfo | head -10

# See current load average (live)
cat /proc/loadavg
```

### Lab 1.4 — Distributions and packages
```bash
# What distro am I on?
cat /etc/os-release

# If Ubuntu:
apt list --installed 2>/dev/null | head -10

# If RHEL/CentOS/Amazon Linux:
yum list installed | head -10
```

---

## Interview Questions with Easy Answers

### Q1: What is Linux?
**Say this:**
"Linux is the kernel — the program that controls the computer hardware like CPU and memory.
Ubuntu and RHEL are Linux distributions — they bundle the kernel with extra tools.
I've worked on RHEL/CentOS in our banking project and Ubuntu on cloud servers."

---

### Q2: What is the difference between kernel and shell?
**Say this:**
"Kernel controls the hardware directly — CPU, memory, disk, network. Shell is the box where I type commands. I type in the shell, shell passes it to the kernel, kernel does the work and shows me the result."

---

### Q3: What does the kernel do?
**Say this:**
"Four things. One — decides which program gets CPU time. Two — gives each program its own safe memory space. Three — handles reading and writing files no matter what storage device. Four — handles network connections."

---

### Q4: What happens when a Linux server boots up?
**Say this:**
"Power turns on. BIOS checks the hardware. GRUB bootloader loads the kernel from disk into memory. Kernel starts up, detects hardware. Then kernel starts systemd which is PID 1. systemd starts all services like SSH and networking. Then the server is ready for login."

---

### Q5: Which distributions have you worked with?
**Say this:**
"I've worked mainly on RHEL/CentOS in our banking environment — they use yum for package management. On AWS I've worked with Amazon Linux 2 which also uses yum. I know Ubuntu and apt as well. In Docker containers we often use Alpine as the base image because it's very small."

---

### Scenario Q1: Server boots very slowly — how do you find the cause?
**Say this:**
"First I run `systemd-analyze` — it shows total boot time split into how long each phase took.
Then I run `systemd-analyze blame` — it shows each service and how many seconds it took to start, sorted from slowest to fastest. The service at the top is usually the problem. I also run `journalctl -b -p err` to check if any errors happened during boot that might explain the delay."

---

### Scenario Q2: You SSH into an unknown server during an incident. First 5 things you do?
**Say this:**
"First, `hostname` — confirm I'm on the right server.
Second, `cat /etc/os-release` — see what distro it is.
Third, `uptime` — check load average, is it overloaded?
Fourth, `df -h` — is disk full?
Fifth, `free -h` — is memory full?
These five checks take under a minute and immediately tell me if there's an obvious resource problem before I start digging into logs."
