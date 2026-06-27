# Linux Boot Process & systemd-analyze — Complete Study Guide

---

## First, Understand This — What Happens When You Press Power Button?

Most people think: "I press power, Linux starts."

Reality is: **4 things happen one after another** before you see the login screen.
Each one hands over control to the next — like a relay race.

```
You press Power Button
        ↓
  BIOS/UEFI runs      ← firmware on the motherboard chip
        ↓
  GRUB runs           ← boot loader stored on disk
        ↓
  Linux Kernel runs   ← the actual Linux OS starts
        ↓
  systemd runs        ← starts all your services (SSH, Docker, etc.)
        ↓
  Login Screen        ← you can now log in
```

> **One line to remember:** Power → BIOS → GRUB → Kernel → systemd → Login.

---

## Step 1: BIOS / UEFI — "Check everything is working"

### What is BIOS/UEFI?

BIOS and UEFI are **firmware** — a tiny program permanently stored on a chip on your motherboard.
It is NOT part of Linux. It runs BEFORE Linux even loads.

```
Motherboard chip (permanent)
        ↓
BIOS/UEFI wakes up immediately when power is turned on
```

### What does BIOS/UEFI actually do?

```
✅ Checks if CPU is working
✅ Checks how much RAM you have
✅ Checks if hard disk, keyboard, mouse are connected
✅ Finds which disk has an operating system on it
✅ Hands control to GRUB (the bootloader on that disk)
```

### Easy example you already know

When you turn on your Lenovo ThinkPad and you see the **Lenovo logo** appear first — that is BIOS/UEFI running its checks. Linux has not started yet at that point.

### BIOS vs UEFI — what is the difference?

```
BIOS (old)                      UEFI (modern — what your ThinkPad uses)
────────────────────────────────────────────────────────────
Old technology                  New, modern technology
16-bit, slow                    64-bit, faster
No mouse support                Mouse support in settings screen
Max disk size: 2TB              Supports disks bigger than 2TB
Uses MBR partitioning           Uses GPT partitioning (better)
No Secure Boot                  Has Secure Boot (security feature)
```

> Most servers and laptops today use UEFI, not old BIOS. But everyone still says "BIOS" casually.

---

## Step 2: GRUB — "Find Linux and load it"

### What is GRUB?

GRUB stands for **GRand Unified Bootloader**.
It is a small program stored on your hard disk (not in the motherboard chip like BIOS).
Its only job: find the Linux kernel file on disk and load it into RAM.

### What does GRUB actually do?

```
✅ Shows a boot menu (if you have multiple OS installed)
✅ Loads the Linux kernel file (called vmlinuz) from disk into RAM
✅ Loads a temporary mini filesystem (called initramfs) into RAM
✅ Passes control to the kernel
```

### Easy example you already know

If you ever had both Ubuntu and Windows installed, the screen where you see:

```
Ubuntu Linux
Advanced options for Ubuntu
Windows Boot Manager
```

That selection screen is **GRUB**. After you pick, GRUB loads your choice.

### Where is GRUB stored?

```bash
ls /boot/
# You will see:
# vmlinuz-5.15.0-91-generic   ← the kernel file GRUB loads
# initrd.img-5.15.0-91-generic ← the temporary mini filesystem
# grub/                        ← GRUB configuration folder

cat /boot/grub/grub.cfg | head -30
# This is the GRUB config — it tells GRUB where to find the kernel
```

---

## Step 3: Linux Kernel — "Start Linux itself"

### What does the kernel do during boot?

```
✅ Detects your CPU (how many cores, what type)
✅ Detects your RAM (how much, what speed)
✅ Detects your hard disk and network card
✅ Loads drivers for all detected hardware
✅ Mounts the root filesystem (/) from disk
✅ Starts PID 1 — which is systemd
```

The kernel boot phase is usually very fast — 2 to 3 seconds on modern hardware.

---

## Step 4: systemd — "Start all your services"

### What is systemd?

systemd is the very **first real program** the kernel starts.
It gets **PID 1** — Process ID number 1 — the first process on the entire system.
All other processes are started by systemd (directly or indirectly).

### What does systemd do during boot?

```
✅ Reads service files in /etc/systemd/system/
✅ Starts NetworkManager (so networking works)
✅ Starts SSH daemon (so you can connect remotely)
✅ Starts Docker (if installed and enabled)
✅ Starts your application services
✅ Starts the display manager (if desktop is needed)
```

---

## systemd-analyze — measuring boot time

### The main command

```bash
systemd-analyze
```

This shows you **how long each phase of boot took**.

### Real output (from your system)

```
Startup finished in
  7.614s (firmware)      ← BIOS/UEFI phase
+ 9.678s (loader)        ← GRUB phase
+ 2.285s (kernel)        ← Linux kernel phase
+13.709s (userspace)     ← systemd starting services
= 33.288s total          ← total from power-on to desktop ready
```

### Reading this output — easy explanation

```
firmware  = 7.6 seconds   → BIOS spent 7.6s checking hardware
                            This is SLOW for BIOS. Normal is 1-3 seconds.
                            Could mean: many USB devices, memory check enabled

loader    = 9.6 seconds   → GRUB spent 9.6s loading the kernel
                            This is VERY SLOW for GRUB. Normal is under 2s.
                            Could mean: GRUB timeout is set high (boot menu wait time)

kernel    = 2.2 seconds   → Kernel took 2.2s to initialize
                            This is NORMAL.

userspace = 13.7 seconds  → systemd took 13.7s starting services
                            This is SLIGHTLY SLOW. Normal is 5-10s.
                            Some service is slow — find it with blame command
```

---

## systemd-analyze blame — find the slow service

```bash
systemd-analyze blame
```

This shows **every service that started during boot**, sorted from slowest to fastest.

### Example output

```
12.345s docker.service
 8.901s NetworkManager-wait-online.service
 5.432s apt-daily-upgrade.service
 3.210s accounts-daemon.service
 2.100s udisks2.service
 1.500s bluetooth.service
 0.800s ssh.service
 0.500s cron.service
```

### How to read this

```
The service at the TOP took the longest.
docker.service took 12 seconds — that is why userspace was 13.7s total.

NetworkManager-wait-online.service taking 8 seconds is also common —
this service waits for the network to be fully ready before continuing.
It often makes boot slow unnecessarily.
```

### How to fix a slow service

```bash
# Option 1: Disable a service you don't need at boot
systemctl disable bluetooth.service

# Option 2: Mask a service completely (even stronger than disable)
systemctl mask NetworkManager-wait-online.service

# Option 3: Check why a specific service is slow
journalctl -u docker.service -b
# Shows that service's logs from the current boot
```

---

## systemd-analyze critical-chain — the dependency chain

```bash
systemd-analyze critical-chain
```

This shows **why** boot took long — specifically which services were waiting for other services.

### Example output

```
The time when unit became active or started is printed after the "@" character.
The time the unit took to start is printed after the "+" character.

graphical.target @33.1s
└─multi-user.target @33.1s
  └─docker.service @20.2s +12.3s
    └─network.target @20.1s
      └─NetworkManager.service @5.2s +3.1s
        └─dbus.service @5.0s
          └─basic.target @4.9s
```

### How to read this

```
Read it bottom to top.
Each line depends on (waits for) the line above it.

docker.service waited for network.target
network.target waited for NetworkManager.service
NetworkManager.service waited for dbus.service

So if NetworkManager was slow → everything above it was delayed.
```

---

## All Three Commands — when to use which

```
systemd-analyze
→ Use this FIRST. Get the big picture. How long was each phase?

systemd-analyze blame
→ Use this SECOND. Find which service was the slowest.

systemd-analyze critical-chain
→ Use this THIRD. Understand WHY — which service was blocked waiting for another.
```

---

## Extra Useful Commands (important additions)

```bash
# Check boot logs from current boot
journalctl -b
# Shows all logs from the moment power was pressed until now

# Check boot logs but only errors
journalctl -b -p err
# Shows only ERROR level messages from current boot

# Check boot logs from the PREVIOUS boot (very useful when server crashed)
journalctl -b -1
# -1 means one boot ago. -2 means two boots ago.

# Check which target (mode) the system booted into
systemctl get-default
# Usually shows: multi-user.target (server without GUI)
# Or: graphical.target (desktop with GUI)

# Check how long the system has been running (uptime)
uptime
# Shows: up 2 days, 4 hours, 30 min, load average: 0.5, 0.3, 0.2
```

---

## GRUB Config — useful to know

```bash
# GRUB main config (do NOT edit this directly)
cat /boot/grub/grub.cfg | head -50

# Edit GRUB settings here (safe to edit)
sudo nano /etc/default/grub

# Important settings in /etc/default/grub:
GRUB_TIMEOUT=10
# This is why GRUB took 9.6 seconds! Timeout was set to 10 seconds.
# Change to GRUB_TIMEOUT=2 to make it faster.

GRUB_DEFAULT=0
# Which OS to boot by default (0 = first option)

# After editing, always run this to apply changes:
sudo update-grub
```

---

## Lab Practice

### Lab Setup

```bash
mkdir -p ~/linux-lab/boot-lab
cd ~/linux-lab/boot-lab
```

### Lab 1 — Run systemd-analyze and save the output

```bash
# Check your boot time
systemd-analyze

# Save it to a file for your records
systemd-analyze > boot-summary.txt
cat boot-summary.txt
```

### Lab 2 — Find the slowest services

```bash
# Show all services sorted slowest to fastest
systemd-analyze blame

# Show only the top 10 slowest
systemd-analyze blame | head -10

# Save to file
systemd-analyze blame > slow-services.txt
```

### Lab 3 — Check the critical chain

```bash
systemd-analyze critical-chain
# Read it and identify: which service is at the TOP of the chain?
# That is the main reason boot is slow.
```

### Lab 4 — Check boot logs

```bash
# All logs from current boot
journalctl -b | head -50

# Only errors from current boot
journalctl -b -p err

# Logs for one specific service from boot
journalctl -u ssh.service -b

# How many error messages happened during boot?
journalctl -b -p err | wc -l
```

### Lab 5 — Check GRUB timeout (why loader was slow)

```bash
# Check current GRUB timeout setting
grep GRUB_TIMEOUT /etc/default/grub
# If it says GRUB_TIMEOUT=10 — that explains 9.6s loader time

# Check what OS options GRUB shows
grep menuentry /boot/grub/grub.cfg | head -5
```

### Lab 6 — Check systemd target

```bash
# What mode does this system boot into?
systemctl get-default

# List all services and their current status
systemctl list-units --type=service --state=running | head -20

# Check if a specific service is enabled at boot
systemctl is-enabled docker
systemctl is-enabled ssh
```

### Lab 7 — Simulate what an interviewer might ask

```bash
# Question: "Boot is slow, find the cause"
# Step 1: get overview
systemd-analyze

# Step 2: find the slow service
systemd-analyze blame | head -5

# Step 3: check its logs
# (replace docker with whatever service was slowest on your machine)
journalctl -u docker.service -b | tail -20

# Step 4: check if it can be disabled or delayed
systemctl is-enabled docker.service
```

---

## Interview Questions with Easy Answers

### Q1: Which command shows Linux boot time?

**Say this:**
"`systemd-analyze` shows total boot time broken into 4 phases — firmware (BIOS/UEFI), loader (GRUB), kernel, and userspace (systemd starting services). In my system it showed 33 seconds total, with GRUB taking 9.6 seconds — which I found was because GRUB_TIMEOUT was set to 10 seconds in the config."

---

### Q2: Which command finds the slowest service during boot?

**Say this:**
"`systemd-analyze blame` lists every service that started during boot, sorted from slowest to fastest. The service at the top took the longest. I use this first to identify the problem service, then `journalctl -u servicename -b` to see that service's logs and understand why it was slow."

---

### Q3: What is BIOS/UEFI and what does it do?

**Say this:**
"BIOS or UEFI is firmware — a small program stored permanently on a chip on the motherboard. It is not part of Linux. It runs first when you press the power button. Its job is to check hardware is working — CPU, RAM, disk — and then find the bootloader on disk and hand control to it. UEFI is the modern version of BIOS — it supports larger disks, boots faster, and has Secure Boot."

---

### Q4: What is GRUB and what does it do?

**Say this:**
"GRUB is the bootloader — a small program stored on the hard disk. After BIOS/UEFI finishes its hardware checks, it passes control to GRUB. GRUB finds the Linux kernel file on disk, loads it into RAM, and then passes control to the kernel. If you have Ubuntu and Windows both installed, GRUB is the menu that lets you choose which one to boot."

---

### Q5: What is systemd? What is PID 1?

**Say this:**
"systemd is the first real program the Linux kernel starts after it loads. It gets Process ID number 1 — PID 1 — the very first process on the system. Every other service and process is started by systemd directly or indirectly. systemd reads service files from /etc/systemd/system/ and starts services like SSH, Docker, NetworkManager in the correct order based on their dependencies."

---

### Q6: What is the difference between systemd-analyze blame and systemd-analyze critical-chain?

**Say this:**
"blame shows me a sorted list of every service and how long it took — useful to find THE slowest service. critical-chain shows me the dependency chain — it shows WHY a service was slow. For example, docker might be slow because it was waiting for network to be ready, and network was slow because NetworkManager was slow. blame finds the symptom, critical-chain finds the cause."

---

### Q7: How do you check logs from a previous boot? (Server crashed last night)

**Say this:**
"`journalctl -b -1` shows logs from the previous boot. `-b` means boot, `-1` means one boot ago, `-2` means two boots ago. I combine it with `-p err` to show only errors: `journalctl -b -1 -p err`. This is the first command I run when a server crashed overnight — it shows exactly what errors occurred before the crash."

---

### Scenario Q1: A server is slow to start after reboot. Your team asks you to investigate. What do you do?

**Say this:**
"Step 1: `systemd-analyze` — I get the total picture. Which phase was slow — firmware, GRUB, kernel, or userspace? Step 2: If userspace was slow — `systemd-analyze blame | head -10` — find the slowest service. Step 3: `systemd-analyze critical-chain` — understand if one service was blocking others. Step 4: `journalctl -u slow-service-name -b` — look at that service's boot logs to see WHY it was slow. Step 5: Fix — either disable the service if not needed, or investigate why it takes long (for example, NetworkManager-wait-online.service often waits unnecessarily and can be masked safely on servers)."

---

### Scenario Q2: You press power on a server and nothing boots. How do you think about the problem?

**Say this:**
"I think about it in order — which phase failed? If the Lenovo or Dell logo never appears — BIOS/UEFI is the problem, likely a hardware failure. If I see the logo but then a blank screen or GRUB error — GRUB is the problem, maybe a corrupt bootloader or wrong config. If I see 'Kernel panic' messages — the kernel loaded but crashed, likely a driver issue or corrupt filesystem. If the kernel loaded but services fail — I check `journalctl -b -p err` for failed services. Each phase has its own symptoms. I narrow it down by what I see on screen."
