# Process Management — Complete Study Guide with Labs

---

## What is a Process? — Easy Version

When you install an app like Chrome — that is just a file sitting on disk.
When you OPEN Chrome and it starts running — that becomes a **process**.

```
Program (sitting on disk) = just a file, doing nothing
Process (running in memory) = that program actually working, using CPU and RAM
```

Think of it like this:
- A **recipe book** sitting on your shelf = program (doing nothing)
- Someone **actually cooking** that recipe in the kitchen = process (doing real work)

100 people can cook the same recipe at the same time — that is 100 processes of the same program.

> **One line:** Process = a running program. Every running thing on Linux is a process.

---

## PID — Process ID

Every process gets a unique number called **PID** (Process ID).
Think of it like a token number at a bank — every customer gets a different number.

```
PID 1    = systemd      ← very first process, started by kernel at boot
PID 2    = kthreadd     ← kernel helper process
PID 1234 = nginx        ← your web server
PID 5678 = your script  ← something you started
```

- **PID 1 is always systemd** — the parent of everything
- Every process has a **parent** (the process that created it)
- This is called **PPID** — Parent Process ID

```
Kernel
  ↓ starts
systemd (PID 1)
  ↓ starts
sshd (PID 500)      ← SSH service
  ↓ starts
bash (PID 800)      ← your shell when you SSH in
  ↓ starts
ps (PID 801)        ← the command you just ran
```

---

## Process Lifecycle — Birth to Death

```
Step 1: Parent calls fork()
        → Creates an exact copy of itself (child process)

Step 2: Child calls exec()
        → Replaces itself with the new program

Step 3: Program runs and does its work

Step 4: Program finishes and exits with a code
        → 0 = success
        → 1 or higher = something went wrong

Step 5: Parent calls wait()
        → Collects the exit code from the child
        → Child is fully cleaned up
```

Easy example — you type `ls` in terminal:
```
bash (your shell) does fork()     → creates a copy of itself
copy does exec("/usr/bin/ls")     → becomes the ls program
ls runs and prints files          → does its work
ls exits with code 0              → finished successfully
bash does wait()                  → collects the exit, shows next prompt
```

---

## Process States — What a Process Can Be Doing

```
R = Running
    Process is on the CPU right now, or waiting its turn for CPU.
    Many R processes = CPU is very busy.

S = Sleeping (interruptible)
    Process is waiting for something — user input, network data, a timer.
    This is NORMAL. Most processes spend most time here.

D = Uninterruptible Sleep
    Process is waiting for disk I/O (reading/writing to disk).
    CANNOT be killed — even kill -9 will not work while in D state.
    Many D processes = disk is very slow or stuck.

Z = Zombie
    Process finished but parent did not collect its exit code yet.
    Shows as "defunct" in ps output.
    Takes no CPU or memory — just occupies a PID slot.

T = Stopped
    Process is paused.
    Created by: Ctrl+Z (you paused it), or kill -STOP command.
```

> **In interviews:** if you see many Z (zombie) processes — parent program has a bug.
> If you see many D processes — disk problem.

---

## Viewing Processes — Commands

### ps — snapshot of processes

```bash
ps aux
# Shows ALL processes on the system right now
# a = all users, u = show user/CPU/memory, x = include background processes

# Output columns:
# USER   PID  %CPU  %MEM   VSZ    RSS   TTY  STAT  START  TIME  COMMAND
# root     1   0.0   0.1  168MB  9.8MB   ?    Ss   Jan01  0:05  systemd
#          ↑         ↑                        ↑
#         PID       memory                  state

ps aux | grep nginx
# Find only nginx processes

ps -ef
# Different format — also shows PPID (parent process ID)

ps -ef --forest
# Shows processes as a tree — who started what
# Very useful to see parent-child relationships
```

### top — live view of processes

```bash
top
# Opens a live updating screen

# Keys to use inside top:
# P = sort by CPU usage (highest first)
# M = sort by Memory usage (highest first)
# k = kill a process (it will ask for PID)
# 1 = show each CPU core separately
# q = quit top
```

### Other useful commands

```bash
pgrep nginx
# Find PID of process named nginx (cleaner than ps aux | grep)

pidof nginx
# Same — find PID by name

pstree -p
# Show ALL processes as a visual tree with PIDs
# Great for understanding parent-child relationships
```

---

## Foreground vs Background Process

### Foreground process
- Runs in your terminal
- Your terminal is BLOCKED — you cannot type other commands
- Press Ctrl+C to stop it
- Press Ctrl+Z to PAUSE it (does not stop, just pauses)

### Background process
- Runs behind the scenes
- Your terminal is FREE — you can type other commands
- Add `&` at the end of a command to run it in background

```bash
./backup.sh
# Runs in FOREGROUND — terminal is blocked until it finishes

./backup.sh &
# Runs in BACKGROUND — terminal is free immediately
# Shows: [1] 12345   ← job number 1, PID 12345

jobs
# Shows all background jobs in your current terminal session
# [1]+ Running    ./backup.sh &

fg %1
# Bring job number 1 back to FOREGROUND

bg %1
# If a job is paused (Ctrl+Z), this resumes it in BACKGROUND

Ctrl+Z
# PAUSE the current foreground job (does not kill it, just pauses)
```

### nohup — keep running after logout

```bash
nohup ./backup.sh &
# Run in background AND keep running even after you log out
# Normal & jobs stop when you close the terminal
# nohup ignores the logout signal
# Output goes to nohup.out file by default

nohup ./backup.sh > /var/log/backup.log 2>&1 &
# Same but save output to a specific log file
```

---

## Signals — Talking to a Process

You cannot tap a process on the shoulder — you send it a **signal**.
A signal is a pre-defined short message sent to a process.

```
Signal    Number    Easy meaning
─────────────────────────────────────────────────────────
SIGTERM     15      "Please stop. Clean up first, then exit."
                    Process CAN choose to ignore this.
                    Always try this FIRST.

SIGKILL      9      "Stop RIGHT NOW. No choice."
                    Kernel forces it. Process CANNOT ignore.
                    Use only if SIGTERM did not work.

SIGHUP       1      "Reload your config file."
                    nginx, sshd use this to reload without restart.

SIGINT       2      Same as pressing Ctrl+C on keyboard.

SIGSTOP     19      "Pause immediately." Cannot be ignored.
SIGCONT     18      "Resume from pause."
```

```bash
kill -15 1234        # Send SIGTERM to PID 1234 — ask nicely to stop
kill -9 1234         # Send SIGKILL to PID 1234 — force stop NOW
kill -1 1234         # Send SIGHUP — reload config

kill 1234            # Default is -15 (SIGTERM)

killall nginx        # Kill ALL processes named nginx
pkill -f "python"    # Kill process whose command matches "python"

# Reload nginx config without restarting:
kill -1 $(cat /var/run/nginx.pid)
# or
nginx -s reload
```

> **Important interview point:**
> Always try `kill -15` first. Wait 20-30 seconds.
> Only use `kill -9` if -15 did not work.
> Why? kill -9 gives no time to clean up — open files may get corrupted,
> database transactions may not complete, temp files are left behind.

---

## Process Priority — nice and renice

Every process has a **priority number** called **nice value**.
Range is from -20 to 19.

```
-20 = highest priority (gets MORE CPU time) — greedy
  0 = normal priority (default)
 19 = lowest priority (gets LESS CPU time) — polite
```

Think of it like a queue:
- Nice value -20 = VIP, jumps the queue, always gets served first
- Nice value 19 = very polite, always lets others go first

```bash
nice -n 10 ./heavy-script.sh
# Start a script with LOW priority (10)
# Useful for backup scripts — you don't want backup to slow down your server

nice -n -5 ./important-script.sh
# Start with HIGH priority (needs root for negative values)

renice -n 5 -p 1234
# CHANGE priority of a process that is ALREADY running
# Change PID 1234 to nice value 5

ps aux -o pid,ni,comm
# Show processes with their nice value (ni column)
```

---

## Zombie Process — Easy Explanation

```
Step 1: Child process finishes its work and exits.
Step 2: Child waits for parent to collect its exit code.
Step 3: Parent is busy or has a bug — never calls wait().
Step 4: Child is stuck in Z (zombie) state.
        It is DEAD but cannot be fully removed.
```

Zombie uses NO CPU, NO memory — just takes up a PID slot.

**How to find zombies:**
```bash
ps aux | grep Z
# Look for Z in the STAT column

ps aux | awk '$8 == "Z"'
# Cleaner way to find zombies
```

**How to fix zombies:**
```bash
# You CANNOT kill a zombie directly — it is already dead.
# Kill the PARENT process.
# When parent dies, systemd (PID 1) adopts the zombie and cleans it up.

ps -ef | grep zombie-pid
# Find the PPID (parent PID) of the zombie

kill -15 PPID
# Kill the parent — this cleans up all its zombie children
```

---

## Orphan Process — Easy Explanation

```
Parent process dies BEFORE the child finishes.
Child is now an orphan — no parent.
Linux automatically gives the orphan to systemd (PID 1) as new parent.
systemd properly cleans up when orphan finishes.
Orphans are NOT a problem — systemd handles them automatically.
```

---

## lsof — What Files Does a Process Have Open?

```bash
lsof -p 1234
# Show all files open by process with PID 1234

lsof -i :80
# Show which process is using port 80

lsof | grep deleted
# Show files that were DELETED but still held open by a process
# This is important — deleted files still take disk space until process releases them
```

---

## Dummy Data for Labs

```
We will create fake processes using sleep command.
sleep 100 = a process that does nothing for 100 seconds (easy to practice with)
```

---

## Lab Practice

### Lab Setup
```bash
mkdir -p ~/linux-lab/process-lab
cd ~/linux-lab/process-lab
```

---

### Lab 1 — View all processes

```bash
# See all processes
ps aux

# Count how many processes are running
ps aux | wc -l

# Find systemd — PID 1
ps aux | grep systemd | head -3

# Show process tree
ps -ef --forest | head -30
```

---

### Lab 2 — Create dummy processes to practice with

```bash
# Start 3 background sleep processes (these are your dummy processes)
sleep 300 &
sleep 300 &
sleep 300 &

# Now find them
ps aux | grep sleep

# Note down their PIDs from the output
# You will use these PIDs in next labs
jobs
# Shows: [1] Running sleep 300 &
#        [2] Running sleep 300 &
#        [3] Running sleep 300 &
```

---

### Lab 3 — Practice kill signals

```bash
# Start a new dummy process
sleep 200 &
echo "PID is: $!"
# $! gives you the PID of the last background command

# Store the PID in a variable
MYPID=$!
echo "I will practice kill on PID: $MYPID"

# Send SIGTERM first (ask nicely)
kill -15 $MYPID

# Check if it stopped
ps aux | grep $MYPID | grep -v grep
# If no output = process stopped

# Start another one
sleep 200 &
MYPID2=$!

# Send SIGKILL (force stop)
kill -9 $MYPID2
ps aux | grep $MYPID2 | grep -v grep
```

---

### Lab 4 — Foreground and background

```bash
# Run in foreground (your terminal will be blocked for 10 seconds)
sleep 10
# You cannot type anything. Wait for it to finish.

# Now run in background
sleep 60 &
# Terminal is free immediately

# See background jobs
jobs

# Bring it to foreground
fg %1
# Now terminal is blocked again

# Pause it with Ctrl+Z
# Then resume in background
bg %1
jobs
```

---

### Lab 5 — pgrep and pidof

```bash
# Start some sleep processes
sleep 500 &
sleep 500 &

# Find their PIDs using pgrep
pgrep sleep

# Same using pidof
pidof sleep

# Kill ALL sleep processes at once
killall sleep

# Verify they are gone
pgrep sleep
# Should show nothing
```

---

### Lab 6 — Process priority (nice)

```bash
# Start a process with LOW priority (nice value 15)
nice -n 15 sleep 200 &
NICEPID=$!

# Check its nice value
ps -o pid,ni,comm -p $NICEPID
# ni column shows the nice value

# Change priority of running process
renice -n 5 -p $NICEPID

# Check again
ps -o pid,ni,comm -p $NICEPID
# ni should now show 5

# Clean up
kill $NICEPID
```

---

### Lab 7 — top practice

```bash
# Start some dummy load to watch in top
stress --cpu 2 --timeout 30 &
# (if stress not installed: apt install stress OR yum install stress)

# Open top
top
# Inside top:
# Press P → sort by CPU (you should see stress at top)
# Press M → sort by memory
# Press 1 → see individual CPU cores
# Press q → quit

# If stress is not available, just open top and explore what you see
top
```

---

### Lab 8 — Find zombie processes (simulation)

```bash
# Check for zombies on your system right now
ps aux | awk '$8 == "Z"'

# Or
ps aux | grep " Z "

# Count how many processes are in each state
ps aux | awk '{print $8}' | sort | uniq -c | sort -rn
# This shows: how many in R, S, Z, D state
```

---

### Lab 9 — nohup practice

```bash
# Start a process that survives logout
nohup sleep 300 > /tmp/nohup-test.log 2>&1 &
echo "Started with PID: $!"

# Check it is running
ps aux | grep "sleep 300"

# Check the output file
cat /tmp/nohup-test.log

# Even if you close this terminal and open a new one,
# this process will still be running (try it!)
```

---

### Lab 10 — lsof practice

```bash
# See what files your shell has open
lsof -p $$
# $$ = PID of your current shell

# See what is using port 22 (SSH)
lsof -i :22

# See what is using port 80 (if nginx is running)
lsof -i :80

# Count how many files are open system-wide
lsof | wc -l
```

---

## Interview Questions with Easy Answers

### Q1: What is a process?

**Say this:**
"A process is a running program. When a program file is sitting on disk it is just a file — doing nothing. When it is opened and running, it becomes a process — using CPU and memory. Every process gets a unique number called PID. The very first process is systemd with PID 1, and every other process is started by it directly or indirectly."

---

### Q2: What is the difference between kill -9 and kill -15?

**Say this:**
"kill -15 sends SIGTERM — it politely asks the process to stop. The process gets a chance to clean up — close open files, finish writing to disk, close database connections — then exit. kill -9 sends SIGKILL — the kernel forces the process to stop immediately with no chance to clean up. I always try kill -15 first and wait 20-30 seconds. I only use kill -9 if the process does not respond to -15. The risk of kill -9 is: open database transactions may not complete, temp files are left behind, and data can get corrupted."

---

### Q3: What is a zombie process and how do you fix it?

**Say this:**
"A zombie process is one that has finished running but its parent process never collected its exit code. It shows as Z in the STAT column of ps. It uses no CPU and no memory — it just occupies a PID slot. You cannot kill a zombie directly because it is already dead. The fix is to kill the parent process — when the parent dies, systemd adopts the zombie and immediately cleans it up. If zombies keep coming back, it means the parent program has a bug — it is creating child processes but not properly waiting on them."

---

### Q4: What is the difference between foreground and background process?

**Say this:**
"A foreground process runs in your terminal and blocks it — you cannot type other commands until it finishes. A background process runs behind the scenes — you add & at the end of the command and your terminal is free immediately. I use background for long-running tasks like backups. I use nohup with & when I want the process to keep running even after I log out — because a normal background job stops when you close the terminal."

---

### Q5: What is nice value?

**Say this:**
"Nice value is the priority of a process — how much CPU time it gets compared to others. Range is -20 to 19. -20 is highest priority, gets the most CPU. 19 is lowest priority, very polite, always lets other processes go first. I use nice -n 15 for backup scripts so they run with low priority and do not slow down the production application. I use renice to change the priority of a process that is already running."

---

### Q6: How do you find which process is using the most CPU?

**Say this:**
"I use `top` and press P to sort by CPU — the highest CPU process appears at the top. Or I use `ps aux --sort=-%cpu | head -10` for a quick one-time snapshot. If I need to watch one specific process over time, I use `pidstat -p PID 1` which samples it every second."

---

### Q7: What is the difference between kill, killall, and pkill?

**Say this:**
"kill sends a signal to ONE specific process using its PID — like kill -15 1234. killall sends a signal to ALL processes with a specific NAME — like killall nginx kills every nginx process. pkill is like killall but matches by pattern — pkill -f 'python app.py' kills any process whose full command matches that pattern. I use kill when I know the exact PID, killall when I want to stop all instances of a service, and pkill when I need to match by part of the command."

---

### Scenario Q1: Production server load is very high. Application is slow. How do you find the cause?

**Say this:**
"Step 1: `top` — open it and immediately look at the load average at the top and the CPU% and MEM% columns. Press P to sort by CPU. Is one process eating 99% CPU? Step 2: look at the 'wa' percentage in the CPU line — if wa (I/O wait) is high, the problem is disk, not CPU. Step 3: if it is a specific process eating CPU — note its PID, then run `ps -ef | grep PID` to understand what it is and who started it. Step 4: check its logs — `journalctl -u servicename` or check the app log file in /var/log. Step 5: if it is a runaway process that should not be there — kill -15 first, then kill -9 if needed. Always tell your team before killing production processes."

---

### Scenario Q2: A process is not stopping. kill -15 is not working. What do you do?

**Say this:**
"First I confirm the process is still running: `ps aux | grep PID`. If it is still there and not responding to kill -15, I check its state: `ps aux | grep PID` — look at the STAT column. If it is in D state (uninterruptible sleep), it is waiting for disk I/O and kill -9 will also not work — I need to fix the underlying disk issue. If it is in S or R state and just ignoring SIGTERM, then I use kill -9 PID to force stop it. After that I check: did it leave any temp files? Did it have open database connections? I clean up manually if needed and check the logs to understand why it was not responding to SIGTERM in the first place."

---

### Scenario Q3: You see 50 zombie processes on the server. What do you do?

**Say this:**
"50 zombies means one parent process is creating children but not collecting their exit codes — it has a bug. First I find the zombies: `ps aux | awk '$8 == \"Z\"'`. Then I find the parent PID using `ps -ef | grep zombie-PID` — look at the PPID column. Then I check what that parent process is — `ps aux | grep PPID`. If I can safely restart it — `systemctl restart servicename`. If not, I kill -15 the parent, which makes systemd adopt and clean up all 50 zombies instantly. The real fix is longer term — the development team needs to fix the parent application to properly call wait() on its children."
