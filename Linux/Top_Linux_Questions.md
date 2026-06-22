# Linux — Top 20 Must-Know Interview Questions
# 10 Troubleshooting + 10 Conceptual
# Zero to Expert — Every WHY Explained

---

# PART 1 — TOP 10 TROUBLESHOOTING QUESTIONS
# These are questions every interviewer asks to test if you have REAL experience

---

## Q1: Your server disk is 100% full. Application has stopped working. What do you do?

### First — WHY does this happen?

Think of your server's disk like a bucket of water.
Every file you create, every log line written, every docker image pulled — adds water to the bucket.
When the bucket is completely full — no more water fits.

In Linux, when disk is 100% full:
- Application cannot write new log lines → application crashes or hangs
- Database cannot write transactions → data loss risk
- Even SSH login can fail because Linux cannot create temp files for your session
- Simple commands like touch or mkdir fail with "No space left on device"

So 100% disk = the server is effectively broken until you free space.

### WHY do disks fill up? — The 5 most common real reasons

```
Reason 1: Log files grew too big
→ Application writes thousands of log lines per minute
→ Nobody set up log rotation
→ /var/log fills up over days/weeks

Reason 2: Docker not cleaned up
→ Every docker pull downloads an image (500MB to 2GB each)
→ Old stopped containers and unused images pile up
→ /var/lib/docker fills up silently

Reason 3: Core dump files
→ When an application crashes, Linux can write a "core dump" file
→ This is a snapshot of the application's memory at crash time
→ Memory snapshots can be gigabytes in size

Reason 4: Large backup files not deleted
→ Old .tar.gz backup files accumulating

Reason 5: Database growing unchecked
→ MySQL/PostgreSQL data files in /var/lib/mysql growing
```

### HOW to fix it — step by step thinking

```bash
# Step 1: Confirm disk is full and find WHICH disk/partition
df -h
# This shows all mounted filesystems and their usage
# Look for the one showing 100% or "Use% = 100%"

# WHY df -h?
# df = disk free
# -h = human readable (shows GB, MB instead of raw bytes)
# Output looks like:
# Filesystem      Size  Used Avail Use%  Mounted on
# /dev/xvda1       50G   50G    0  100%  /          ← THIS is full
# /dev/xvdb       100G   20G   80G  20%  /data      ← this is fine

# Step 2: Find WHAT is using the space
du -sh /*  2>/dev/null
# du = disk usage
# -s = summary (total for each folder, not every file inside)
# -h = human readable
# /* = check every folder at the root level
# 2>/dev/null = hide permission errors

# This output might look like:
# 1.2G  /usr
# 200M  /etc
# 47G   /var    ← THIS is the problem folder
# 100M  /home

# Step 3: Go deeper into the problem folder
du -sh /var/*
# Now you see:
# 45G   /var/log     ← log files are the problem
# 500M  /var/lib

# Step 4: Find the specific large files
du -sh /var/log/*  | sort -rh | head -10
# sort -rh = sort by size, largest first
# head -10 = show only top 10

# You might see:
# 44G   /var/log/application.log   ← one single log file took 44GB!

# Step 5: Quick emergency fix — free space immediately
# Option A: If log file is safe to truncate (empty it without deleting)
truncate -s 0 /var/log/application.log
# WHY truncate and not rm?
# If the application currently has this file open and you delete it (rm),
# the application still holds the file descriptor open
# Linux will NOT free the disk space until the application releases it
# The file disappears from directory listing but space is NOT freed!
# truncate empties the file content to zero bytes
# Application keeps writing to same file, space is freed immediately

# Option B: Check if deleted files are still held open (common trap)
lsof | grep deleted
# This shows files that were DELETED but still open by a process
# Example output:
# java  1234  /var/log/old.log (deleted)
# The (deleted) file is still using 5GB of disk!
# Fix: restart the application OR run:
# truncate -s 0 /proc/1234/fd/7  (7 is the file descriptor number)

# Step 6: Check Docker if it is installed
docker system df
# Shows how much space Docker is using for images, containers, volumes

docker system prune -a
# Removes all unused images, stopped containers, unused networks
# WARNING: only run this if you know what containers are running

# Step 7: Check for core dump files
find / -name "core" -o -name "core.*" 2>/dev/null | head -10
find / -name "*.dump" 2>/dev/null | head -10
```

### Long term fix — set up log rotation

```bash
# Log rotation = automatically compress and delete old log files
# Config is in /etc/logrotate.d/

cat /etc/logrotate.d/nginx
# /var/log/nginx/*.log {
#     daily           ← rotate every day
#     rotate 14       ← keep 14 days of logs
#     compress        ← compress old files to save space
#     missingok       ← do not error if log file missing
#     notifempty      ← skip if log is empty
# }
```

### How to answer in interview

"First I run `df -h` to confirm which filesystem is 100% full and identify which partition.
Then I run `du -sh /var/* | sort -rh` to find which folder is consuming the most space.
I go deeper with `du -sh` until I find the specific large files.

The most common cause in my experience is log files that grew unchecked — I truncate them with `truncate -s 0 filename` rather than deleting, because deleting a file that a running process has open does not free disk space until that process closes it.

I also check `lsof | grep deleted` for files that are deleted but still held open by processes — this is a common hidden space consumer.

If Docker is installed, I check `docker system df` and clean unused images.

Long term fix: I set up proper logrotate configuration so logs rotate daily and old ones get compressed automatically. In our banking project, we had a Java application logging at DEBUG level in production that filled a 50GB disk in 3 days — we fixed it by changing log level to INFO and setting up logrotate."

---

## Q2: Application is running but users say it is slow. How do you find the cause?

### WHY does this happen — the 4 types of slowness

```
Type 1: CPU overload
→ Some process is eating too much CPU
→ Other processes get less CPU time
→ Everything feels slow

Type 2: Memory full, swap being used
→ When RAM is full, Linux moves some data to disk (swap space)
→ Disk is 1000x slower than RAM
→ Server feels very slow

Type 3: Disk I/O bottleneck
→ Too many reads/writes happening at the same time
→ Disk cannot keep up
→ Processes wait for disk — server hangs

Type 4: Network bottleneck
→ Network is saturated
→ Responses from database or other services are delayed
```

### Step by step investigation

```bash
# Step 1: Get the big picture in 10 seconds
uptime
# Output: 10:30:01 up 5 days, load average: 8.5, 6.2, 4.1
#                                              ↑1min ↑5min ↑15min
# Load average = how many processes are WAITING for CPU or disk
# Rule: load average should be LESS than your number of CPU cores
# Check CPU count: nproc
# If nproc says 4 but load is 8.5 → server is overloaded

# Step 2: Open top and READ it carefully
top
# First line: same as uptime
# CPU line: us sy id wa
#   us = user programs using CPU (your application)
#   sy = kernel using CPU (system calls)
#   id = idle CPU (FREE cpu time)
#   wa = waiting for I/O (DISK is the bottleneck if this is high)
#
# If wa is 60% → disk is the problem, not CPU
# If us is 95% → application is eating CPU

# Step 3: If CPU is the problem — find which process
ps aux --sort=-%cpu | head -10
# Shows top 10 CPU consuming processes
# %cpu column shows percentage

# Step 4: If memory is the problem
free -h
# Output:
#               total   used    free    available
# Mem:           16G    15.8G   200M    100M
# Swap:           4G     3.9G   100M
#
# WHY look at "available" not "free"?
# "free" = completely unused memory
# "available" = memory that can be given to new processes
#               (includes cache that can be freed)
# If "available" is very low AND swap "used" is high
# → memory pressure is causing slowness

# Step 5: If disk I/O is the problem
iostat -xz 1
# Shows disk stats every 1 second
# Key column: %util
# %util = how busy the disk is
# %util 100% = disk is completely saturated → I/O bottleneck
#
# await column = average time in milliseconds for I/O to complete
# Good: < 10ms for SSD
# Problem: > 100ms = disk is struggling

# Step 6: If network is the problem
sar -n DEV 1 3
# Shows network traffic per second
# rxkB/s = kilobytes received per second
# txkB/s = kilobytes sent per second
# If near the maximum speed of your network card → network bottleneck
```

### How to answer in interview

"When application is slow, I think about 4 possible causes: CPU, memory, disk I/O, or network.

My first command is `uptime` to see the load average compared to the number of CPU cores from `nproc`. Then I open `top` and look specifically at the 'wa' percentage in the CPU line — if wa is high, the problem is disk I/O, not CPU, and adding more CPU will not help.

For CPU problems: `ps aux --sort=-%cpu | head -10` shows the process consuming the most.
For memory: `free -h` — I look at 'available' memory and whether swap is being used heavily.
For disk: `iostat -xz 1` — if %util is near 100%, disk is the bottleneck.

In one incident in our banking project, the application was slow and everyone assumed it was CPU. I checked `top` and saw wa was 70%. `iostat` showed %util at 98% on the disk. `iotop` revealed a nightly backup script was running during business hours and saturating the disk. I rescheduled the backup to 2am and slowness disappeared immediately."

---

## Q3: You cannot SSH into a server. How do you troubleshoot?

### WHY SSH might fail — 6 different reasons

```
Reason 1: Network — server is unreachable
→ Wrong IP, server down, firewall blocking, cloud security group blocking

Reason 2: SSH service is not running
→ sshd crashed, was never started, or failed to start

Reason 3: SSH is running on a different port
→ Admin changed port from 22 to something else for security

Reason 4: Firewall on the server is blocking port 22
→ iptables or firewalld is blocking incoming SSH connections

Reason 5: Wrong SSH key or wrong username
→ Your private key does not match the public key on server
→ You are using wrong username (ubuntu vs ec2-user vs root)

Reason 6: /etc/ssh/sshd_config has an error
→ Someone edited SSH config wrongly and SSH daemon crashed
```

### Step by step investigation — from outside to inside

```bash
# Test from your local machine (before trying to get into the server):

# Step 1: Can you reach the server at all?
ping -c 4 SERVER_IP
# If ping fails → server is down or network is completely blocked
# If ping works → server is reachable, problem is something else

# Step 2: Can you reach port 22 specifically?
nc -zv SERVER_IP 22
# nc = netcat
# -z = just check if port is open, do not send data
# -v = verbose, show result
#
# If this says "Connection refused" → SSH service is not running
# If this says "Connection timed out" → firewall is blocking port 22
# If this connects → SSH is running and reachable

# Step 3: Try SSH with verbose mode (shows exactly WHERE it fails)
ssh -vvv user@SERVER_IP
# -vvv = maximum verbosity
# Output shows:
# debug: Connecting to SERVER_IP port 22
# debug: Connection established
# debug: Server host key: ...
# debug: Authenticating...
# Read this output and find the last successful line
# The line after that is where it failed

# Step 4: If you have console access (AWS EC2 console, Azure serial console):
# Get into the server through the cloud provider's web browser console
# Then check:

systemctl status sshd
# Is SSH service running?
# Active: active (running) → SSH is running, problem is network/firewall
# Active: failed → SSH crashed

systemctl start sshd
# If it was not running, start it

# Check which port SSH is listening on
ss -tulnp | grep ssh
# If shows 0.0.0.0:22 → listening on all interfaces, port 22
# If shows 0.0.0.0:2222 → someone changed the port to 2222

# Check firewall on the server
firewall-cmd --list-all          # RHEL/CentOS
ufw status verbose               # Ubuntu

# Check SSH config for errors
sshd -t
# This validates the SSH config file WITHOUT restarting SSH
# If output is blank → config is fine
# If output shows errors → fix those errors first
```

### How to answer in interview

"I troubleshoot SSH failure in layers — starting from the outside and going in.

First I check basic connectivity with `ping SERVER_IP`. If ping fails, it is a network or server issue — I check the cloud console to see if the server is running and check security groups.

If ping works, I test port 22 specifically with `nc -zv SERVER_IP 22`. Connection refused means SSH service is not running. Connection timeout means a firewall is blocking port 22.

For more detail, I run `ssh -vvv user@SERVER_IP` which shows exactly at which step authentication fails.

If I have console access, I check `systemctl status sshd` and `ss -tulnp | grep ssh` to confirm SSH is running and on which port. I also check `sshd -t` to validate the SSH config file for syntax errors.

Common mistakes I have seen: someone changed Port in /etc/ssh/sshd_config without opening the new port in firewall, or an AWS security group was accidentally modified during a Terraform apply that removed the SSH inbound rule."

---

## Q4: Application port is not accessible. How do you debug?

### WHY a port might not be accessible

```
Possibility 1: Application is NOT running
→ The port cannot be open if the app is not started

Possibility 2: Application started but bound to wrong address
→ App listening on 127.0.0.1:8080 (only accessible from same machine)
→ Should be listening on 0.0.0.0:8080 (accessible from anywhere)
→ This is a very common mistake in Spring Boot, Node.js apps

Possibility 3: Firewall on the server is blocking the port
→ iptables or firewalld has no rule allowing this port

Possibility 4: Cloud firewall (security group) is blocking
→ AWS security group or Azure NSG has no inbound rule for this port

Possibility 5: Application crashed after starting
→ App started, opened port briefly, then crashed
```

### Step by step

```bash
# Step 1: Is the application process running at all?
ps aux | grep appname
# If no output (other than the grep itself) → app is not running
# Start it: systemctl start appname

# Step 2: Is the port actually listening?
ss -tulnp | grep :8080
# Output explanation:
# tcp LISTEN 0 128 0.0.0.0:8080 → listening on ALL interfaces (good, accessible externally)
# tcp LISTEN 0 128 127.0.0.1:8080 → listening on LOCALHOST ONLY (not accessible externally)
#
# If 127.0.0.1 is shown → this is the problem
# Fix: change application config to bind to 0.0.0.0
# In Spring Boot: server.address=0.0.0.0 in application.properties
# In Node.js: app.listen(8080, '0.0.0.0')

# Step 3: If port is listening correctly — check server firewall
firewall-cmd --list-all | grep 8080    # RHEL
ufw status | grep 8080                  # Ubuntu

# If port 8080 not in the list → firewall is blocking
firewall-cmd --add-port=8080/tcp --permanent
firewall-cmd --reload

# Step 4: Test connectivity from the server itself
curl localhost:8080
# If this works → app is running, problem is external firewall

# Step 5: Check cloud security group (AWS/Azure)
# Go to AWS console → EC2 → Security Groups → check inbound rules
# Port 8080 must be allowed from the source IP

# Step 6: Check application logs
journalctl -u appname -f
# Or check log file:
tail -f /var/log/app/error.log
# Look for errors that happened at startup
```

### How to answer in interview

"I approach port accessibility issues in a specific order.

First, I verify the application is actually running with `ps aux | grep appname`.

Then I check if the port is listening with `ss -tulnp | grep :8080`. I specifically look at the address — if it shows 127.0.0.1:8080 instead of 0.0.0.0:8080, the application is bound to localhost only and is not accessible from outside. This is one of the most common mistakes I have seen — developers test locally and bind to 127.0.0.1, and that setting goes to production.

Next I test from the server itself with `curl localhost:8080` — if this works but external access fails, the problem is the firewall. I check the OS firewall with `firewall-cmd --list-all` and the cloud security group in the AWS console.

I always check application logs last to see if there were startup errors."

---

## Q5: "No space left on device" error — but df -h shows disk has free space. Why?

### WHY this confusing situation happens — the inode problem

This is the most confusing disk issue and interviewers love asking it.

To understand it, you need to know that every file on Linux has TWO parts:

```
Part 1: The CONTENT (the actual data)
→ stored in data blocks on disk
→ df -h shows you how full the DATA blocks are

Part 2: The INODE (the identity card of the file)
→ stores: who owns it, permissions, size, when created, where data is on disk
→ stored separately in an inode table
→ df -i shows you how full the INODE table is
```

When you create a file, Linux needs BOTH:
- A free data block (for the content)
- A free inode (for the identity card)

If the inode table is 100% full — you CANNOT create new files.
Even if you have 40GB of free disk space for data.
Because there is no inode to give the new file its identity.

```bash
# Check inode usage
df -i
# Output:
# Filesystem   Inodes  IUsed   IFree IUse% Mounted on
# /dev/xvda1  3276800 3276800      0  100% /    ← 100% inodes used!
# BUT df -h shows 40% disk space free

# Find which directory has millions of files
find /var -xdev -printf '%h\n' | sort | uniq -c | sort -rn | head -10
# This counts files per directory
# The directory with millions of files is the problem

# Common cause: application creating millions of tiny session files
ls /var/lib/php/sessions/ | wc -l
# Might show: 5000000 (5 million files!)

# Fix: delete old session files
find /var/lib/php/sessions/ -mtime +1 -delete
# Deletes session files older than 1 day
```

### How to answer in interview

"This is an inode exhaustion problem. Every file on Linux needs two things — disk space for its content AND an inode entry which is like an identity card that stores the file's metadata. The number of inodes is fixed when the filesystem is created.

When `df -h` shows free space but creating files fails with 'no space left on device', I immediately run `df -i` to check inode usage. If any filesystem shows 100% IUse%, that is the problem.

I then find which directory has the most files using `find` to count files per directory. The most common cause I have seen is web applications creating millions of tiny session or cache files in /var/lib/php/sessions or /tmp and never cleaning them up.

The fix is to delete old files with `find /path -mtime +1 -delete` and then configure the application to clean up old session files automatically."

---

## Q6: A process is using 100% CPU. What do you do?

### WHY a process might use 100% CPU

```
Reason 1: Infinite loop in code
→ A bug makes the code loop forever doing the same thing
→ Never stops, consumes all CPU

Reason 2: Too many requests at once
→ Application is genuinely overloaded with traffic
→ CPU is legitimately busy doing real work

Reason 3: Runaway scheduled job
→ A cron job or script started and got stuck
→ Running much longer than expected

Reason 4: Cryptomining malware
→ Attacker installed a program that uses your CPU to mine cryptocurrency
→ Shows as unknown process using 100% CPU
```

### Step by step

```bash
# Step 1: Find which process is using 100% CPU
top
# Press P to sort by CPU
# The first process in the list is the highest CPU consumer
# Note its PID

# More detailed snapshot:
ps aux --sort=-%cpu | head -5

# Step 2: Get more details about that process
# Replace 1234 with the actual PID
cat /proc/1234/cmdline | tr '\0' ' '
# Shows the EXACT command that started this process
# WHY /proc/PID/cmdline?
# Sometimes ps shows a short name, /proc shows full command with arguments

ls -la /proc/1234/exe
# Shows which binary file is running
# If it shows a path you do not recognize → could be malware

# Step 3: See what the process is actually doing right now
strace -p 1234 -c
# strace watches all system calls the process makes
# -c = count and summarize
# Shows: is it stuck in a loop? waiting for something? reading files?
# This helps understand WHY it is using CPU

# Step 4: If it is a Java process — get a thread dump
jstack 1234
# Shows all Java threads and what code each one is running
# Look for threads stuck in a loop

# Step 5: Reduce its CPU impact immediately (without killing)
renice -n 15 -p 1234
# This lowers the priority to 15 (very low)
# Process still runs but gets much less CPU time
# Other processes get more CPU → server recovers

# Step 6: Kill it if necessary
kill -15 1234
# Try graceful stop first — wait 30 seconds
kill -9 1234
# Force kill if -15 did not work
```

### How to answer in interview

"When I see 100% CPU usage, my first step is `top` with P to sort by CPU and identify the process and its PID.

Then I investigate why — I check `/proc/PID/cmdline` to see the full command, and `strace -p PID -c` to see what system calls it is making. For Java applications, `jstack PID` gives thread dumps showing if there is an infinite loop.

If the server is suffering, I immediately `renice -n 15 -p PID` to lower that process's priority without killing it — this gives other processes more CPU while I investigate.

I check if it is a legitimate process gone wrong or something unexpected. An unrecognized binary showing 100% CPU is a security concern — I check `ls -la /proc/PID/exe` to see the actual binary file location and escalate to the security team if it looks suspicious."

---

## Q7: Server memory is full. Application is crashing. What do you do?

### WHY memory fills up and what happens

```
Normal situation:
RAM has space → application gets memory → works fine

When RAM fills up:
Linux uses SWAP space (a section of disk used as emergency RAM)
BUT disk is 1000x slower than RAM
→ server becomes very slow

When RAM + SWAP both fill up:
Linux activates the OOM Killer (Out Of Memory Killer)
OOM Killer is a part of the kernel
Its job: find a process and kill it to free memory
It chooses which process to kill based on a score
Usually kills the largest memory consumer
→ your application gets killed without warning
```

```bash
# Step 1: Check current memory situation
free -h
# Output:
#               total   used    free  buff/cache   available
# Mem:           16G    15G    100M       900M         500M
# Swap:           4G     3.8G  200M
#
# "available" = 500M → very low, almost out of real memory
# Swap used = 3.8G → heavily using slow disk swap → server is slow

# Step 2: Check if OOM killer already killed something
dmesg | grep -i "oom\|killed process\|out of memory"
# If you see output like:
# "Out of memory: Kill process 1234 (java) score 800"
# → OOM killer killed java process because it used too much memory
# This explains why application crashed

# Step 3: Find which process is using most memory
ps aux --sort=-%mem | head -10
# Shows top 10 memory consuming processes
# %MEM column and RSS column (RSS = actual RAM used in KB)

# Step 4: Check memory per process in detail
cat /proc/1234/status | grep -i vmrss
# VmRSS = actual RAM this process is using right now

# Step 5: Emergency fix — free some memory
# Clear the Linux page cache (safe to do, kernel rebuilds it automatically)
sync && echo 3 > /proc/sys/vm/drop_caches
# sync = write any pending data to disk first
# echo 3 = clear pagecache, dentries, and inodes
# This can free several GB immediately

# Step 6: Long term — find the memory leak
# Run this every minute and watch the memory grow over time:
watch -n 60 'ps aux --sort=-%mem | head -5'
# If one process memory keeps growing over time → memory leak in that application
```

### How to answer in interview

"When memory is full, I first run `free -h` to understand the situation — specifically looking at 'available' memory and swap usage. If swap is heavily used, the server is already degraded because swap is much slower than RAM.

Next I check `dmesg | grep -i oom` to see if the OOM killer has already killed processes. The OOM killer is the Linux kernel's emergency mechanism — when both RAM and swap are exhausted, it kills the largest memory consuming process to free memory. This is often why applications crash without any error in the application log.

I find the top memory consumers with `ps aux --sort=-%mem | head -10` and check if one process is growing over time using `watch`. A continuously growing process has a memory leak.

Emergency relief: I can drop the page cache with `sync && echo 3 > /proc/sys/vm/drop_caches` which can free several gigabytes immediately without killing any processes — the kernel rebuilds the cache automatically."

---

## Q8: Cron job is not running. How do you debug?

### WHY cron jobs silently fail

```
Cron jobs are sneaky when they fail.
Unlike manual commands, cron:
- Runs in a minimal environment (different PATH from your login)
- Runs as a specific user (not always you)
- Output goes nowhere by default (you never see errors)
- Uses a different working directory

So a script that works perfectly when YOU run it manually
can silently fail every time cron runs it
and you never know because there is no output anywhere
```

### Step by step

```bash
# Step 1: Check if cron service is running
systemctl status crond      # RHEL/CentOS
systemctl status cron       # Ubuntu
# If not running → that is why nothing executed

# Step 2: Check the crontab syntax
crontab -l
# Shows your current cron jobs
# Common mistake — wrong syntax:
# Wrong:  * * * * /path/to/script.sh  (only 5 fields, the 6th is command)
# Correct: * * * * * /path/to/script.sh

# Use crontab.guru website to verify your cron expression
# Syntax: minute hour day month weekday command
# 0 2 * * * → runs at 2:00 AM every day

# Step 3: Check if cron actually tried to run the job
grep CRON /var/log/syslog      # Ubuntu
grep CRON /var/log/cron        # RHEL/CentOS
# This shows every cron execution attempt with timestamp

# Step 4: The PATH problem — most common cause of cron failures
# When you log in, your PATH includes:
# /usr/local/bin:/usr/bin:/bin:/home/user/.local/bin etc.
# Cron runs with minimal PATH:
# /usr/bin:/bin   ← much shorter!
#
# So if your script runs: docker ps
# And docker is in /usr/local/bin (not in cron's PATH)
# Cron cannot find docker → script fails silently
#
# Fix: use absolute paths in scripts OR add PATH at top of crontab:
crontab -e
# Add this as first line:
# PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

# Step 5: Capture output so you can see errors
# Add output capture to cron job:
0 2 * * * /opt/scripts/backup.sh >> /var/log/backup.log 2>&1
# >> /var/log/backup.log = append stdout to log file
# 2>&1 = also send stderr (error messages) to same log file
# Without this, all output disappears into nothing

# Step 6: Test the script exactly as cron would run it
# Create a minimal environment like cron and run your script:
env -i HOME=/root PATH=/usr/bin:/bin /bin/bash /opt/scripts/backup.sh
# env -i = start with empty environment (like cron)
# This reveals problems that only happen in cron's environment
```

### How to answer in interview

"Cron job failures are tricky because cron runs silently — by default all output disappears, so you never see error messages.

My first check is `systemctl status crond` to ensure cron service is running. Then `grep CRON /var/log/cron` to see if cron is even attempting to run the job.

The most common cause I have seen is the PATH difference. When you run a script manually, your terminal has a full PATH. Cron uses a minimal PATH — /usr/bin:/bin. So if your script calls `docker` or `terraform` or `python3` that lives in /usr/local/bin, cron cannot find it and fails silently.

My fix: always use absolute paths in scripts (`/usr/bin/python3` not just `python3`), or set the PATH explicitly at the top of the crontab.

The other key fix is capturing output — I add `>> /var/log/myjob.log 2>&1` to every cron job so errors are saved to a file I can read."

---

## Q9: System is showing high load average but CPU is idle. Explain.

### WHY this confusion happens

This is a question that separates people who memorize from people who understand.

Most people think: high load = CPU is busy. This is WRONG.

Load average counts TWO types of waiting processes:
```
Type 1: Processes waiting for CPU (what most people think)
Type 2: Processes waiting for DISK I/O (the surprise)
```

When a process is waiting for the disk to respond,
it goes into "D" state (Uninterruptible sleep).
Linux counts D-state processes in load average.

So load can be very high while CPU shows 90% idle —
because all 50 processes are waiting for disk, not CPU.

```bash
# Prove it — see the difference:
uptime
# load average: 12.5 → very high

top
# CPU: 85% idle (wa=70%)
# wa = iowait = processes waiting for disk
# HIGH iowait + HIGH load + IDLE cpu = DISK BOTTLENECK

# Confirm with iostat
iostat -xz 1
# Look for %util column
# If /dev/sda shows %util: 100% → disk is completely saturated
# All processes waiting for disk = high load average

# Find which process is causing the disk I/O
iotop -a
# Shows I/O usage per process
# The process at top is hammering the disk

# Another way:
pidstat -d 1
# Shows disk read/write per process every second
```

### How to answer in interview

"High load average with idle CPU is a disk I/O bottleneck — not a CPU problem.

Load average in Linux counts not just processes waiting for CPU, but also processes in 'D' state — uninterruptible sleep — which means waiting for disk I/O. So when 20 processes are all waiting for the disk to respond, load average of 20 is possible while CPU is 90% idle.

I confirm this by running `top` and looking at the 'wa' (iowait) percentage in the CPU line. If wa is high, I then run `iostat -xz 1` to find which disk is at 100% utilization, and `iotop` to find which process is causing the I/O.

In a banking project I investigated exactly this — load average was 18 on a 4-core server, but CPU was 80% idle. iostat showed the database disk at 100%. The cause was a reporting query doing a full table scan on a 50GB table with no index. Adding the index brought load average back to under 1."

---

## Q10: You deleted a file but disk space was not freed. Why?

### WHY this happens — the file descriptor mystery

When you delete a file with `rm`, Linux does two things:
```
Step 1: Remove the directory entry (the filename disappears from ls)
Step 2: Decrease the link count on the inode
```

But here is the important rule:
Linux only FREES the actual disk blocks when the inode's link count reaches ZERO.

If a running process has that file open, the inode link count stays at 1
(because the process holds a reference through its file descriptor).

So:
```
rm logfile.log
→ filename disappears from ls   (directory entry removed)
→ process still has it open     (file descriptor still valid)
→ disk space NOT freed          (data blocks still in use)
→ process keeps writing to it   (file grows in size silently)
→ you see no file but disk fills up!
```

```bash
# Find deleted files still held open by processes
lsof | grep deleted
# Output example:
# java  1234  ubuntu  7u  REG  /var/log/app.log (deleted)  5368709120
#                                                            ↑ 5GB still used!

# Solution 1: Restart the application
# When application restarts, it closes all file descriptors
# Kernel then frees the disk blocks

# Solution 2: Truncate through the file descriptor (no restart needed!)
# From the lsof output: process PID is 1234, file descriptor is 7
truncate -s 0 /proc/1234/fd/7
# This empties the file content through the open file descriptor
# Disk space freed immediately
# Application continues running and writing to it (now starting from empty)

# Verify space was freed
df -h
```

### How to answer in interview

"This happens because of how Linux file deletion works. When you delete a file, Linux removes the directory entry — the filename disappears from `ls`. But the actual disk blocks are only freed when the link count reaches zero, meaning no process has the file open.

If a running process — like a Java application — has that log file open and is actively writing to it, deleting the file does not free the disk space. The file is invisible in the directory but the data blocks are still occupied and the process keeps writing to them.

I find these invisible files with `lsof | grep deleted` — it shows every process holding a deleted file open along with how much space it is consuming.

The fix without restarting the application: `truncate -s 0 /proc/PID/fd/FD_NUMBER`. This empties the file content through the process's file descriptor, freeing disk space immediately, while the application keeps running. We used this exact technique in production when a Java service was logging at DEBUG level and filled the disk — we could not restart during business hours, so truncate through fd saved us."

---

# PART 2 — TOP 10 CONCEPTUAL QUESTIONS
# These test if you UNDERSTAND Linux, not just use it

---

## Q11: What is the difference between process and thread?

### The real explanation — not the textbook version

```
PROCESS:
→ A complete, independent running program
→ Has its OWN memory space (other processes cannot see in)
→ Has its own file descriptors, its own PID
→ If one process crashes → other processes are unaffected
→ Starting a new process is EXPENSIVE (copying memory etc.)

Think of it like: separate apartments in a building
Each apartment has its own lock, own furniture, own address
If apartment 3 catches fire → apartment 5 is safe

THREAD:
→ A lightweight execution unit that lives INSIDE a process
→ Shares the SAME memory space as other threads in same process
→ Shares file descriptors, heap memory
→ If one thread corrupts memory → can crash ALL threads in the process
→ Starting a new thread is CHEAP (no memory copying)

Think of it like: roommates inside one apartment
They share the same kitchen, same living room
If one roommate makes a mess → everyone is affected
```

```bash
# See processes and their threads
ps -eLf | grep java
# -L = show threads
# Each line = one thread
# The "LWP" column = thread ID

# Count threads for a specific process
cat /proc/1234/status | grep Threads
# Threads: 45 → this process has 45 threads

# Java web servers use threads heavily
# Each incoming HTTP request → one thread handles it
# That is why Java apps have many threads
```

### How to answer in interview

"A process is an independent program with its own isolated memory space. Processes cannot see each other's memory. A thread lives inside a process and shares the same memory with all other threads in that process.

The practical difference: if thread A corrupts a value in shared memory, thread B reads the corrupted value and the whole process can crash. But if process A crashes, process B is completely unaffected.

In production: nginx uses separate worker processes for isolation — a crash in one worker does not affect others. Java applications use threads — one thread per HTTP request — which is faster because threads share memory and do not need to copy data between them, but one bad thread can take down the entire JVM.

In Docker, each container is essentially a process with its own isolated filesystem and network — not a thread — which is why containers provide isolation."

---

## Q12: What is swap space? When is it good and when is it bad?

### The real explanation

```
RAM is your work desk.
Swap is a drawer under the desk.

When your desk is full:
You take some papers from the desk and put them in the drawer.
Now desk has space for new work.
But getting something from the drawer is much slower than from the desk.

RAM is fast:  nanoseconds (billionths of a second)
Disk is slow: milliseconds (thousandths of a second)
Ratio: disk is roughly 100,000x slower than RAM

So swap is a safety net — prevents out-of-memory crashes
But it makes the server slow when actually used
```

```bash
# Check swap usage
free -h
# Swap: total used free
# 4G    3.8G  200M ← almost all swap used → server is very slow

# Check swappiness (how aggressively kernel uses swap)
cat /proc/sys/vm/swappiness
# Default: 60
# 0 = almost never use swap (good for database servers)
# 100 = use swap very aggressively

# Change swappiness (without reboot)
sysctl vm.swappiness=10
# Reduce to 10 — kernel only uses swap when absolutely necessary

# Make permanent
echo 'vm.swappiness=10' >> /etc/sysctl.conf

# Create swap file (if no swap partition exists)
fallocate -l 4G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile swap swap defaults 0 0' >> /etc/fstab
```

### How to answer in interview

"Swap is disk space used as emergency overflow when RAM is full. When Linux runs out of RAM, it moves the least-recently-used memory pages from RAM to swap on disk, freeing RAM for more urgent work.

Swap is good as a safety net — it prevents the OOM killer from killing processes when memory gets tight. Without swap, when RAM fills up, processes immediately start getting killed.

But swap being used is a warning sign, not a solution. Disk is roughly 100,000 times slower than RAM. When an application's working data is in swap, it becomes extremely slow. A server that is actively swapping is a server that needs more RAM.

For production database servers like MySQL or PostgreSQL, I set swappiness to 10 with `sysctl vm.swappiness=10` — these applications are sensitive to latency and should use RAM as much as possible. For general application servers, the default of 60 is acceptable.

In our banking project, we had a reporting service that ran at midnight and pushed memory usage high enough to trigger swap usage. The morning team would find the server slow. Solution was to give that service its own server instead of sharing with the main application."

---

## Q13: Explain the Linux boot process in detail.

### Already covered in boot guide — interview version

### How to answer in interview

"The boot process has five stages.

Stage 1 is BIOS or UEFI — firmware stored on a chip on the motherboard. When power is pressed, BIOS runs immediately before any operating system. It checks hardware — CPU, RAM, disk are working — then finds a bootable disk and passes control to the bootloader.

Stage 2 is GRUB — the bootloader. GRUB is stored in the first sectors of the disk. It loads the Linux kernel file called vmlinuz and a temporary filesystem called initramfs from /boot into RAM, then passes control to the kernel.

Stage 3 is the kernel initializing — it detects CPU cores, RAM, disk controllers, network cards, loads the right drivers for each, and mounts the real root filesystem from disk.

Stage 4 is systemd starting — the kernel launches /sbin/init which is systemd as PID 1. systemd reads service configurations from /etc/systemd/system and starts all services in the right order based on their dependencies.

Stage 5 is ready state — login prompt appears, server is accessible.

I use `systemd-analyze blame` to find which service slowed down the boot, and `journalctl -b -1` to check logs from the previous boot when troubleshooting crashes."

---

## Q14: What is a file descriptor? Why does it matter?

### The real explanation

```
When a process opens a file, Linux gives it a NUMBER to represent that file.
That number is the file descriptor.

It is like a token number at a bank.
You give your name and get token number 5.
When called "token 5", you know that means you.

Process says: "open /var/log/app.log"
Linux opens the file and gives back: fd = 7
Process uses fd 7 every time it wants to read/write that file
Process never uses the filename again after opening — just the number

Special file descriptors every process always has:
fd 0 = stdin  (keyboard input)
fd 1 = stdout (normal output to screen)
fd 2 = stderr (error output to screen)
```

```bash
# See all file descriptors for a process
ls -la /proc/1234/fd/
# Output:
# 0 → /dev/pts/0          (stdin = terminal)
# 1 → /dev/pts/0          (stdout = terminal)
# 2 → /dev/pts/0          (stderr = terminal)
# 3 → /var/log/app.log    (log file opened by app)
# 4 → socket:[12345]      (network connection)
# 7 → /var/log/error.log (deleted)  ← this is the deleted file!

# Check file descriptor limit
ulimit -n
# Default: 1024
# This means one process can open at most 1024 files simultaneously

# Common error: "Too many open files"
# Means process hit the fd limit
# Fix for current session:
ulimit -n 65536

# Permanent fix — add to /etc/security/limits.conf:
# * soft nofile 65536
# * hard nofile 65536

# For systemd services — add to service file:
# [Service]
# LimitNOFILE=65536
```

### How to answer in interview

"A file descriptor is a number that the kernel gives to a process when it opens a file. Instead of using the filename every time, the process just uses this number — it is faster and simpler.

Every process starts with three file descriptors: 0 for stdin (keyboard), 1 for stdout (screen), and 2 for stderr (errors). This is why `command > file.txt` redirects stdout — you are redirecting file descriptor 1.

File descriptors are why deleting a file does not immediately free disk space — if a process has the file open (has a fd pointing to it), the data is not deleted until the process closes or the process dies.

The practical problem I have seen: `Too many open files` error. Each process has a limit on how many file descriptors it can have open simultaneously (default 1024). High-traffic web servers or databases that handle many concurrent connections need this limit raised. I raise it with `ulimit -n 65536` for the session or permanently in `/etc/security/limits.conf`."

---

## Q15: What is the difference between /etc/profile, ~/.bashrc, and ~/.bash_profile?

### The real explanation

```
When you log in to Linux, the shell reads configuration files to set up your environment.
But WHICH files it reads depends on HOW you logged in.

Login shell: when you SSH in, or log in at console
→ reads /etc/profile FIRST (system-wide settings for ALL users)
→ then reads ~/.bash_profile (your personal settings)
→ ~/.bash_profile usually calls ~/.bashrc at the end

Non-login shell: when you open a new terminal in a GUI, or run bash in a script
→ reads ~/.bashrc ONLY

Interactive shell: you can type commands
Non-interactive shell: script running — cannot type
```

```bash
# /etc/profile
# → runs for ALL users at LOGIN
# → set system-wide PATH, ulimits, environment variables
cat /etc/profile

# ~/.bash_profile
# → runs for YOU at LOGIN only
# → set personal environment variables
# → usually contains: source ~/.bashrc
cat ~/.bash_profile

# ~/.bashrc
# → runs for YOU every time you open a terminal
# → set aliases, personal functions, prompt color
cat ~/.bashrc
# Common contents:
# alias ll='ls -la'
# alias grep='grep --color'
# export EDITOR=vim

# /etc/environment
# → NOT a script — just key=value pairs
# → loaded very early, before any shell
# → good place for system-wide environment variables
cat /etc/environment
```

### How to answer in interview

"These files control the shell environment but they run at different times.

/etc/profile runs for all users when they do a login — SSH in or log in at a physical console. It sets system-wide settings like PATH and ulimits.

~/.bash_profile runs for your user account specifically at login. It is the personal version of /etc/profile. Most ~/.bash_profile files end with `source ~/.bashrc` to also load the interactive settings.

~/.bashrc runs every time you open any terminal, not just at login. Aliases, custom functions, and prompt customization go here.

The practical problem this causes: you add an alias or export to ~/.bashrc, but it does not work in cron jobs. That is because cron runs as a non-interactive, non-login shell and does not source these files at all. I always use full absolute paths in scripts and explicitly export any needed environment variables inside the script itself."

---

## Q16: What is SELinux and have you worked with it?

### The real explanation — easy version

```
Normal Linux permissions: you have rwx control
→ if a process can read a file, it can read it from ANYWHERE

SELinux adds a second layer of security:
→ even if a process has permissions, SELinux can STILL block it
→ based on LABELS (called contexts) attached to files and processes
→ "even if nginx has read permission, it can only read files labeled httpd_content_t"

Think of it like:
Normal permissions = having a key to a room
SELinux = having a key AND being on the approved visitor list
Both must be satisfied
```

```bash
# Check SELinux status
getenforce
# Enforcing  → SELinux is ON and blocking violations
# Permissive → SELinux is ON but only logging violations (not blocking)
# Disabled   → SELinux is completely off

sestatus
# Shows detailed SELinux status

# See SELinux context of files
ls -lZ /var/www/html/
# Output: -rw-r--r-- root root system_u:object_r:httpd_sys_content_t:s0 index.html
#                                                  ↑ this is the SELinux context label

# If nginx/httpd cannot serve a file despite correct permissions
# → check SELinux context
# Fix: restore default context for web files
restorecon -Rv /var/www/html/

# Check SELinux denial logs
ausearch -m avc -ts recent
# Shows what SELinux blocked recently and WHY

# Temporarily disable SELinux (for testing — not for production)
setenforce 0
# Sets to Permissive mode
# If this fixes the problem → SELinux context was wrong
```

### How to answer in interview

"SELinux is an additional security layer on top of standard Linux permissions. Standard permissions control who can access a file. SELinux adds a second check based on security labels — even if a process has read permission, SELinux can still block it if the file's label does not match what that process type is allowed to access.

In banking environments on RHEL/CentOS, SELinux is typically enforcing mode and I have dealt with it regularly. The most common issue: we deploy a web application, set permissions correctly, but get 403 Forbidden from nginx. The cause is often SELinux — the files have the wrong context label.

I check `ls -lZ /path` to see the current SELinux context and `ausearch -m avc -ts recent` to see what SELinux denied. The quick fix is `restorecon -Rv /path` which restores the correct default context for that location.

I never permanently disable SELinux on production banking servers — it is a compliance requirement. Instead I fix the context labels or add specific SELinux policy exceptions."

---

## Q17: How does Linux handle a command you type — full internal flow?

### The real explanation

This is the full journey from your keypress to result on screen:

```
1. You press keys on keyboard
   → keyboard generates electrical signals
   → kernel's keyboard driver converts to key codes
   → key codes sent to terminal

2. Terminal (your SSH session or console) shows characters on screen
   → this is called "echoing" — showing you what you type

3. You press Enter
   → terminal sends the complete line to bash (shell)

4. Bash receives the line: "ls -la /etc"
   → bash PARSES it: command = "ls", arguments = "-la", "/etc"
   → bash searches for "ls" in PATH directories
   → checks /usr/local/bin/ls → not found
   → checks /usr/bin/ls → FOUND

5. Bash calls fork()
   → Linux kernel creates an exact copy of bash (child process)
   → child process gets a new PID

6. Child bash calls exec("/usr/bin/ls", ["-la", "/etc"])
   → kernel loads /usr/bin/ls binary from disk into the child's memory
   → child process IS NOW ls (bash code replaced by ls code)
   → ls starts running

7. ls reads the /etc directory
   → calls opendir("/etc") → kernel returns directory entries
   → calls getdents() → gets list of files
   → formats them nicely with permissions, sizes, dates

8. ls writes output
   → writes to file descriptor 1 (stdout)
   → stdout is connected to your terminal
   → characters appear on your screen

9. ls finishes
   → calls exit(0) → 0 means success
   → kernel cleans up ls process

10. Bash wakes up (it was waiting with wait())
    → receives exit code 0 from ls
    → stores it in $? variable
    → shows next prompt: ubuntu@server:~$
```

### How to answer in interview

"When I type `ls -la /etc` and press Enter, six things happen in sequence.

First, bash parses the input into command name `ls` and arguments `-la /etc`. It searches each directory in my PATH variable looking for a binary named `ls`, finds it at /usr/bin/ls.

Second, bash calls the `fork()` system call — the kernel creates an exact copy of the bash process with a new PID.

Third, the child process calls `execve()` with the path to ls — the kernel replaces the child's memory with the ls program code and starts it running.

Fourth, ls makes system calls to the kernel to read the /etc directory contents — `opendir()` and `getdents()` — the kernel talks to the filesystem driver which reads from disk.

Fifth, ls writes its formatted output to file descriptor 1 (stdout), which flows back through the terminal to my screen.

Sixth, ls calls `exit(0)` meaning success. Bash's `wait()` returns, captures the exit code into `$?`, and shows the next prompt.

I understand this flow practically because it explains common issues — `command not found` means the binary was not in any PATH directory, exit code in `$?` tells me if the last command succeeded or failed, and understanding fork/exec explains why environment changes in a child process do not affect the parent shell."

---

## Q18: What is /proc and why is it important for DevOps?

### Already covered earlier — interview version

### How to answer in interview

"/proc is a virtual filesystem — it exists only in RAM, created by the kernel at boot time. Nothing in /proc is stored on disk. The kernel populates it dynamically with real-time information about the system.

I use /proc regularly in production. `/proc/meminfo` gives detailed memory breakdown — more detail than `free -h`. `/proc/PID/cmdline` shows the exact command that started any process — useful when `ps` shows a truncated name. `/proc/PID/fd/` shows all open file descriptors of a process — I use this to find deleted files still held open.

For DevOps and Kubernetes specifically, I write to /proc/sys/ to tune kernel parameters. For example, Kubernetes nodes need IP forwarding enabled: `echo 1 > /proc/sys/net/ipv4/ip_forward`. These are temporary — to make permanent I use sysctl.conf.

The `/proc/PID/status` file is useful for memory debugging — VmRSS shows actual RAM a process is using, and Threads shows how many threads it has."

---

## Q19: What happens when you run out of file descriptors? How do you fix it?

### The full story

```
Every open file, every network connection, every socket
= one file descriptor

Default limit per process: 1024 (very low for production!)

High-traffic application:
→ 1000 concurrent users
→ Each connection = 1 fd
→ Each open file = 1 fd  
→ Application hits 1024 limit
→ Error: "Too many open files"
→ Application cannot accept new connections
→ Users see errors
```

```bash
# Check current limits
ulimit -n
# Default: 1024

# Check what limit a running process has
cat /proc/1234/limits | grep "open files"

# Check how many fds process actually has open right now
ls /proc/1234/fd | wc -l
# If this is close to the limit → problem is imminent

# Fix 1: For current shell session only
ulimit -n 65536

# Fix 2: Permanent for all users — /etc/security/limits.conf
echo "* soft nofile 65536" >> /etc/security/limits.conf
echo "* hard nofile 65536" >> /etc/security/limits.conf
# * = applies to all users
# soft = default limit (can be raised up to hard limit by user)
# hard = maximum that user can raise to
# nofile = number of open files

# Fix 3: For systemd-managed services
# Edit the service file: /etc/systemd/system/myapp.service
# Add in [Service] section:
# LimitNOFILE=65536
systemctl daemon-reload
systemctl restart myapp

# System-wide maximum (across all processes)
cat /proc/sys/fs/file-max
# How many total fds the entire system can have

# Increase system-wide max:
sysctl -w fs.file-max=2097152
echo 'fs.file-max=2097152' >> /etc/sysctl.conf
```

### How to answer in interview

"File descriptor exhaustion causes 'Too many open files' errors and the application cannot accept new connections or open new files.

Each open file, network socket, pipe, or device = one file descriptor. The default limit of 1024 per process is fine for simple applications but too low for high-traffic web servers or databases that handle hundreds or thousands of concurrent connections.

I check current usage with `ls /proc/PID/fd | wc -l` compared to `cat /proc/PID/limits | grep 'open files'`. If these numbers are close, the process will soon hit the limit.

The fix depends on how the service is managed. For systemd services — the most common in production — I add `LimitNOFILE=65536` to the [Service] section of the unit file and reload. For system-wide limits, I update /etc/security/limits.conf. In our banking project, our Java application server needed LimitNOFILE=100000 because it maintained persistent connections to many downstream services simultaneously."

---

## Q20: Explain Linux signals and give real-world examples.

### The full story

```
Signals are messages sent to a process.
Like knocking on a door with a specific knock pattern:
- 2 knocks = "please stop what you are doing"
- 3 knocks = "EMERGENCY STOP NOW"
- 1 knock = "reload your config"

Process can:
- Handle the signal (run custom code when it receives it)
- Ignore the signal (pretend it never came)
- Let the default action happen

EXCEPT SIGKILL (9) — this is delivered by kernel directly
Process has NO CHOICE — it dies immediately
```

```bash
# Most important signals in DevOps:

kill -15 PID    # SIGTERM — "please stop gracefully"
                # Use this FIRST, always
                # Process should: finish current request, close DB connections,
                #                 flush logs, delete temp files, then exit
                # Well-written apps handle this
                # If process does not stop in 30 seconds → then use kill -9

kill -9 PID     # SIGKILL — "die NOW, no choice"
                # Kernel forces immediate termination
                # Process cannot clean up → risk of:
                #   - incomplete database transactions
                #   - corrupted log files
                #   - leftover temp files
                #   - locks not released

kill -1 PID     # SIGHUP — "reload your configuration"
                # nginx, apache, sshd use this
                # Process rereads its config file without restarting
                # Users experience zero downtime
                # Example:
nginx -s reload # → sends SIGHUP to nginx master process
                # nginx master: reads new nginx.conf
                # starts new workers with new config
                # gracefully shuts down old workers after they finish requests

kill -2 PID     # SIGINT — same as Ctrl+C
                # "interrupt what you are doing"

kill -19 PID    # SIGSTOP — pause the process
                # Process is frozen, uses no CPU
                # Cannot be ignored or caught

kill -18 PID    # SIGCONT — resume a paused process

# Trapping signals in shell scripts:
#!/bin/bash
cleanup() {
    echo "Caught SIGTERM — cleaning up..."
    rm -f /tmp/myapp.lock
    exit 0
}
trap cleanup SIGTERM SIGINT
# Now your script handles Ctrl+C and kill -15 gracefully
# Instead of being killed mid-operation
```

### How to answer in interview

"Signals are messages sent to a process to control its behavior. The process can choose to handle, ignore, or use the default action for most signals — except SIGKILL which is delivered by the kernel directly and cannot be ignored.

The most important signals in DevOps practice: SIGTERM (15) is the polite stop — I always try this first and wait 30 seconds. A well-written application receives SIGTERM and gracefully shuts down — finishes processing current requests, closes database connections, flushes logs, removes lock files, then exits cleanly. SIGKILL (9) is the last resort — process dies immediately with no cleanup, which risks incomplete transactions or corrupted files.

SIGHUP (1) is important for configuration reloads — nginx, sshd, and most daemons use it. `nginx -s reload` internally sends SIGHUP to the nginx master process, which rereads the config and gracefully replaces workers, giving users zero downtime.

In shell scripts I use `trap` to handle signals gracefully — `trap cleanup SIGTERM` means if someone kills my script, it runs a cleanup function first instead of dying mid-operation and leaving temp files or locks behind."
