# How to Read, Write and Understand Shell Scripts
### The Zero to Pro Strategy | Muskan | 2026
### "You cannot explain what you cannot build yourself first"

---

## THE REAL PROBLEM — And How We Fix It

You have been reading finished scripts.
That is like watching someone cook and thinking you can cook too.
You cannot. You need to cook it yourself.

**From today — every script we build like this:**

```
Step 1: I tell you WHAT we want the script to do in plain English
Step 2: You think — what commands do I need?
Step 3: We write ONE line at a time
Step 4: We RUN that one line on terminal first
Step 5: We understand the output
Step 6: Then we add the next line
Step 7: At the end — you have a script YOU built, not one you copied
```

When an interviewer asks "explain your script" — you can explain because YOU built it.

---

## THE STRATEGY — How to Become Strong in Shell Scripting

This is the 10-year experience strategy. Write this down.

```
Rule 1: Never copy-paste a script you don't understand
Rule 2: Run every command ALONE on terminal before putting it in a script
Rule 3: Read your script OUT LOUD like you are explaining to a junior
Rule 4: If you cannot explain a line — you do not understand it yet
Rule 5: Build small, test small, then combine
Rule 6: One script per concept — master it before moving on
Rule 7: Break a working script intentionally — learn from the error
Rule 8: Every script must have a STORY — what is the problem it solves
```

---

## THE STORY APPROACH — How Real Engineers Think

Before writing ANY script, answer these 3 questions:

```
Question 1: WHAT problem am I solving?
            "I need to check if my server is healthy"

Question 2: WHAT information do I need?
            "hostname, disk space, RAM, CPU, running services"

Question 3: HOW do I get each piece of information?
            "hostname → hostname command
             disk → df command
             RAM → free command"
```

Only THEN do you start writing.

---

## NOW — Let's Rebuild system_info.sh The Right Way

### THE STORY OF system_info.sh

**The Problem:**
You are a DevOps engineer. Your manager says:
"After every new server is set up, check if it is healthy and give me a report."

Without a script you do this manually every time:
- Type hostname
- Type df -h
- Type free -h
- Type uname -r
- Copy the output somewhere

That takes 10 minutes. If you have 20 new servers it is 200 minutes.

**The Solution:**
One script that does all of this automatically in 5 seconds.

---

### STEP 1 — The Empty Script (Start here always)

Every script starts with just 2 lines. ALWAYS.

```bash
#!/bin/bash
# This script shows server health information
```

Line 1 = `#!/bin/bash`
This tells Linux — use bash to run this file.
Think of it as: "Dear Linux, read this file in BASH language."

Line 2 = a comment starting with #
This is just a note to yourself. Linux ignores it.
Good engineers ALWAYS write what the script does at the top.

**Your first task:**
```bash
# Open terminal and type:
nano system_info.sh

# Type these 2 lines:
#!/bin/bash
# This script shows server health information

# Save: Ctrl+O → Enter → Ctrl+X
# Run it:
bash system_info.sh
# Output: nothing — that is correct. Empty script runs and exits.
```

---

### STEP 2 — Add Your First Command

Now we add the first piece of information: the hostname.

**Think first:**
What command shows the hostname?
→ `hostname`

Run it alone first:
```bash
hostname
```
You see: `muskan-ThinkPad-L14-Gen-1`

Now add it to your script:
```bash
#!/bin/bash
# This script shows server health information

echo "Hostname is:"
hostname
```

Run it:
```bash
bash system_info.sh
```

Output:
```
Hostname is:
muskan-ThinkPad-L14-Gen-1
```

**What you just learned:**
- `echo "text"` → prints text on screen
- `hostname` → prints the computer name
- You can run commands inside a script just like typing them in terminal

---

### STEP 3 — Store in a Variable (Make it look better)

Two separate lines (echo + hostname) look clunky.
We want: `Hostname: muskan-ThinkPad-L14-Gen-1` on ONE line.

To do that we need to STORE the hostname in a variable first, then print it.

**The concept:**
```bash
# A variable is a labeled box
# You store a value in the box
# You use the box wherever you need that value

MY_BOX="hello"          # store "hello" in box called MY_BOX
echo $MY_BOX            # take what is in MY_BOX and print it
# Output: hello
```

**For hostname:**
```bash
# Run the hostname command and store its OUTPUT in a variable
# $(command) = run command and capture its output

SERVER_NAME=$(hostname)
# Now SERVER_NAME contains "muskan-ThinkPad-L14-Gen-1"

echo "Hostname: $SERVER_NAME"
# Output: Hostname: muskan-ThinkPad-L14-Gen-1
```

**Run this on terminal first, before adding to script:**
```bash
SERVER_NAME=$(hostname)
echo "Hostname: $SERVER_NAME"
```

Now add to your script:
```bash
#!/bin/bash
# This script shows server health information

SERVER_NAME=$(hostname)
echo "Hostname: $SERVER_NAME"
```

Run it. See the output. Does it make sense? Good.

---

### STEP 4 — Add Date and Time

**Think first:**
What command shows the date?
→ `date`

Run it alone:
```bash
date
```
Output: `Sun Jul  5 19:37:53 IST 2026` — messy format

We want: `2026-07-05 19:37:53` — clean format

```bash
date '+%Y-%m-%d %H:%M:%S'
```
Output: `2026-07-05 19:37:53` — better

Run it alone first. Understand it. Then store it:
```bash
CURRENT_DATE=$(date '+%Y-%m-%d %H:%M:%S')
echo "Date: $CURRENT_DATE"
```

Run this on terminal. Then add to script:
```bash
#!/bin/bash
# This script shows server health information

SERVER_NAME=$(hostname)
CURRENT_DATE=$(date '+%Y-%m-%d %H:%M:%S')

echo "Hostname: $SERVER_NAME"
echo "Date    : $CURRENT_DATE"
```

---

### STEP 5 — Add OS Information

**Think first:**
Where does Linux keep OS information?
→ `/etc/os-release` — a file that has OS details

Run it alone:
```bash
cat /etc/os-release
```

Output is many lines. We only want the one that says the OS name:
```
PRETTY_NAME="Ubuntu 24.04.4 LTS"
```

We need to FIND that line → use `grep`
Then EXTRACT just the name → use `cut`

```bash
# Find line starting with PRETTY_NAME
grep "^PRETTY_NAME" /etc/os-release

# Output: PRETTY_NAME="Ubuntu 24.04.4 LTS"

# Now cut out just the name between the quotes
grep "^PRETTY_NAME" /etc/os-release | cut -d'"' -f2

# Output: Ubuntu 24.04.4 LTS
```

**Run each command alone. Understand the output. Then combine:**
```bash
OS_NAME=$(grep "^PRETTY_NAME" /etc/os-release | cut -d'"' -f2)
echo "OS: $OS_NAME"
```

Add to script:
```bash
#!/bin/bash
# This script shows server health information

SERVER_NAME=$(hostname)
CURRENT_DATE=$(date '+%Y-%m-%d %H:%M:%S')
OS_NAME=$(grep "^PRETTY_NAME" /etc/os-release | cut -d'"' -f2)

echo "Hostname: $SERVER_NAME"
echo "Date    : $CURRENT_DATE"
echo "OS      : $OS_NAME"
```

---

### STEP 6 — Add Disk Space

**Think first:**
What command shows disk space?
→ `df -h /`

Run it alone:
```bash
df -h /
```

Output:
```
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1       457G  143G  291G  33% /
```

We want: `33%` — just the percentage number.

That is column 5, row 2 (row 1 is the header).

```bash
# awk = pick specific column from a table
# NR==2 = row number 2 (skip header row 1)
# {print $5} = print column 5

df -h / | awk 'NR==2 {print $5}'
# Output: 33%
```

Run it. See it. Then store it:
```bash
DISK_USAGE=$(df -h / | awk 'NR==2 {print $5}')
echo "Disk: $DISK_USAGE"
```

---

### STEP 7 — Add RAM Information

**Think first:**
What command shows RAM?
→ `free -h`

Run it alone:
```bash
free -h
```

Output:
```
               total   used    free
Mem:           15Gi   10Gi   1.0Gi
```

We want total RAM = column 2 from the Mem: row.

```bash
free -h | awk '/^Mem:/ {print $2}'
# /^Mem:/ = find line starting with Mem:
# {print $2} = print column 2
# Output: 15Gi
```

---

### STEP 8 — THE COMPLETE PLAIN SCRIPT

Now you have learned each piece. Put them all together.
This is the PLAIN version — no colors, no decoration, just working code.

```bash
#!/bin/bash
# ----------------------------------------
# Script: system_info.sh
# What it does: Shows basic server health
# How to run: ./system_info.sh
# ----------------------------------------

# Collect all information
SERVER_NAME=$(hostname)
CURRENT_DATE=$(date '+%Y-%m-%d %H:%M:%S')
OS_NAME=$(grep "^PRETTY_NAME" /etc/os-release | cut -d'"' -f2)
KERNEL=$(uname -r)
CPU_CORES=$(nproc)
TOTAL_RAM=$(free -h | awk '/^Mem:/ {print $2}')
USED_RAM=$(free -h | awk '/^Mem:/ {print $3}')
DISK_USAGE=$(df -h / | awk 'NR==2 {print $5}')
MY_IP=$(hostname -I | awk '{print $1}')

# Print the information
echo "============================="
echo "     SERVER HEALTH REPORT   "
echo "============================="
echo "Hostname  : $SERVER_NAME"
echo "IP Address: $MY_IP"
echo "Date      : $CURRENT_DATE"
echo "OS        : $OS_NAME"
echo "Kernel    : $KERNEL"
echo "CPU Cores : $CPU_CORES"
echo "Total RAM : $TOTAL_RAM"
echo "Used RAM  : $USED_RAM"
echo "Disk Used : $DISK_USAGE"
echo "============================="
```

**That is it. No colors. No fancy patterns. Just working code.**

---

## HOW TO EXPLAIN THIS SCRIPT IN AN INTERVIEW

When an interviewer says: "Explain your system_info.sh script"

Say this:

> "This script solves a real problem — after we provision a new server,
> the operations team needs to quickly verify the server is healthy.
> Doing it manually takes 10-15 minutes per server.
>
> The script collects 9 pieces of information using standard Linux commands.
> For the hostname, I use the `hostname` command.
> For OS info, I read `/etc/os-release` and extract the name with grep and cut.
> For disk usage, I use `df -h` and extract the percentage column using awk.
> For RAM, I use `free -h` and pick the Mem: row with awk.
>
> I store each value in a variable using command substitution — `$(command)`.
> Then I print everything in one clean report.
>
> This reduced post-provisioning checks from 15 minutes to 5 seconds."

**That is a confident, clear answer. Anyone with real experience talks this way.**

---

## HOW TO BUILD EVERY SCRIPT — The Formula

Use this for disk_alert.sh, health_check.sh, backup.sh — every script.

```
Step 1: Write the STORY — what problem does this solve?
Step 2: List what you NEED — what information or actions?
Step 3: Find each command — run it ALONE on terminal first
Step 4: Store outputs in variables
Step 5: Add logic (if/loop/function) ONE at a time
Step 6: Test after EVERY new line
Step 7: Read the whole script out loud — if you can explain it, you own it
```

---

## NOW — The Same Approach for disk_alert.sh

### THE STORY
Problem: Disk can fill up and crash the server.
You want: An alert when any disk partition is above 80%.

### WHAT DO YOU NEED?
1. Get disk usage for all partitions — `df -h`
2. For each partition — check if usage is above 80
3. If yes — print WARNING
4. If no — print OK

### BUILD IT BRICK BY BRICK

**Brick 1 — Get disk usage:**
```bash
df -h
# Run this. See ALL partitions.
```

**Brick 2 — Get just the % number:**
```bash
df -h / | awk 'NR==2 {print $5}' | tr -d '%'
# tr -d '%' removes the % sign so we get a plain number: 33
# Run this. See: 33
```

**Brick 3 — Compare number to 80:**
```bash
DISK=33
if [ $DISK -gt 80 ]; then
    echo "WARNING"
else
    echo "OK"
fi
# Run this. Change DISK to 85 and run again. See WARNING.
```

**Brick 4 — Combine (the full plain script):**

```bash
#!/bin/bash
# ----------------------------------------
# Script: disk_alert.sh
# What it does: Checks disk space and alerts if above 80%
# How to run: ./disk_alert.sh
# ----------------------------------------

THRESHOLD=80

DISK=$(df -h / | awk 'NR==2 {print $5}' | tr -d '%')

echo "Current disk usage: $DISK%"

if [ $DISK -gt $THRESHOLD ]; then
    echo "WARNING: Disk is above $THRESHOLD percent"
    echo "Action needed: clean up old files or expand storage"
else
    echo "OK: Disk is healthy"
fi
```

**This is 14 lines. Plain. No decoration. Fully understandable.**

---

## THE HEALTH_CHECK.SH STORY AND PLAIN VERSION

### THE STORY
Problem: You need to check 5 things on your server every morning.
Solution: One script that checks all 5 and tells you what is OK and what is not.

**The 5 things:**
1. Is disk OK? (below 80%)
2. Is RAM OK? (below 85%)
3. Is SSH service running?
4. Is cron service running?
5. What is the CPU load?

### PLAIN SCRIPT — No Colors, No Functions, Just Logic

```bash
#!/bin/bash
# ----------------------------------------
# Script: health_check.sh
# What it does: Checks server health — disk, RAM, services
# How to run: ./health_check.sh
# ----------------------------------------

echo "============================="
echo " SERVER HEALTH CHECK"
echo " Host: $(hostname)"
echo " Date: $(date '+%Y-%m-%d %H:%M:%S')"
echo "============================="

# --- CHECK 1: DISK ---
# Get disk usage as a number (no % sign)
DISK=$(df -h / | awk 'NR==2 {print $5}' | tr -d '%')

echo ""
echo "DISK CHECK:"
if [ $DISK -gt 90 ]; then
    echo "  CRITICAL: Disk is at $DISK% — take action now"
elif [ $DISK -gt 80 ]; then
    echo "  WARNING: Disk is at $DISK% — getting full"
else
    echo "  OK: Disk is at $DISK%"
fi

# --- CHECK 2: RAM ---
# Get used RAM and total RAM as numbers (in MB)
# Then calculate percentage with bc
MEM_TOTAL=$(free -m | awk '/^Mem:/ {print $2}')
MEM_USED=$(free -m | awk '/^Mem:/ {print $3}')
MEM_PCT=$(echo "scale=0; $MEM_USED * 100 / $MEM_TOTAL" | bc)

echo ""
echo "MEMORY CHECK:"
if [ $MEM_PCT -gt 85 ]; then
    echo "  WARNING: Memory is at $MEM_PCT%"
else
    echo "  OK: Memory is at $MEM_PCT%"
fi

# --- CHECK 3: SSH SERVICE ---
echo ""
echo "SERVICE CHECK:"

if systemctl is-active --quiet ssh; then
    echo "  OK: SSH is running"
else
    echo "  CRITICAL: SSH is NOT running"
fi

# --- CHECK 4: CRON SERVICE ---
if systemctl is-active --quiet cron; then
    echo "  OK: Cron is running"
else
    echo "  WARNING: Cron is NOT running"
fi

# --- CHECK 5: CPU LOAD ---
CPU_LOAD=$(uptime | awk '{print $(NF-2)}' | tr -d ',')
CPU_CORES=$(nproc)

echo ""
echo "CPU CHECK:"
echo "  Load average: $CPU_LOAD (you have $CPU_CORES cores)"
echo "  (If load is higher than number of cores, CPU is busy)"

echo ""
echo "============================="
echo " Health check complete"
echo "============================="
```

---

## THE BACKUP.SH STORY AND PLAIN VERSION

### THE STORY
Problem: Important files can get deleted by accident or disk failure.
Solution: Every day, copy important folders into a compressed file with today's date.

### PLAIN SCRIPT

```bash
#!/bin/bash
# ----------------------------------------
# Script: backup.sh
# What it does: Creates a compressed backup of /etc folder
# How to run: ./backup.sh
# ----------------------------------------

# Where to save backups
BACKUP_FOLDER="$HOME/backups"

# What to backup
SOURCE="/etc"

# Create a filename with today's date
# This makes every backup unique so old ones are not overwritten
TODAY=$(date '+%Y-%m-%d_%H-%M-%S')
BACKUP_FILE="$BACKUP_FOLDER/backup_$TODAY.tar.gz"

# Create the backup folder if it does not exist
mkdir -p "$BACKUP_FOLDER"

echo "Starting backup..."
echo "Source  : $SOURCE"
echo "Saving to: $BACKUP_FILE"

# Create the compressed backup
# tar -czf = create, compress with gzip, filename
tar -czf "$BACKUP_FILE" "$SOURCE" 2>/dev/null

# Check if backup was created
if [ -f "$BACKUP_FILE" ]; then
    SIZE=$(du -sh "$BACKUP_FILE" | cut -f1)
    echo "Backup created successfully"
    echo "File: $BACKUP_FILE"
    echo "Size: $SIZE"
else
    echo "ERROR: Backup failed"
    exit 1
fi

echo "Done"
```

---

## TROUBLESHOOTING QUESTIONS — What Interviewers Actually Ask

These are the REAL questions at 3-4 year level.
Not theory. They give you a broken scenario and ask you to fix it.

---

**Troubleshooting Q1:**
"Your backup script ran fine yesterday but today it says 'No space left on device'. What do you check?"

**Your answer:**
> "First I check disk space with `df -h` to confirm which partition is full.
> Then I check what is taking space with `du -sh /var/log/*` to find large log files.
> If old backup files are filling the disk, I delete backups older than 7 days
> using `find /backups -name '*.tar.gz' -mtime +7 -delete`.
> Then I run the backup again."

---

**Troubleshooting Q2:**
"Your script runs fine when you type it manually but fails when cron runs it. Why?"

**Your answer:**
> "Cron runs with a minimal environment — it does not load your bashrc or profile.
> So `$PATH` is very limited in cron. Commands like `aws`, `docker`, `python3`
> may not be found because their paths are not in cron's PATH.
> The fix is to either add the full path in the script — `/usr/bin/python3`
> instead of just `python3` — or add `PATH=/usr/local/bin:/usr/bin:/bin`
> at the top of the cron script."

---

**Troubleshooting Q3:**
"Your health check script exits immediately and you see no output. How do you debug?"

**Your answer:**
> "First I run `bash -x health_check.sh` to see every command as it executes.
> The output shows a `+` before each line. I look for the last `+` line
> before the script stopped — that is where it failed.
> If the script has `set -e`, any command returning non-zero stops the script.
> A common cause is `grep` finding no match and returning exit code 1,
> which `set -e` treats as failure. The fix is adding `|| true` after that grep."

---

**Troubleshooting Q4:**
"The script creates a backup file but it is 0 bytes. What happened?"

**Your answer:**
> "A 0-byte tar file means tar ran but had nothing to compress.
> I check if the source directory exists: `ls -la /source/directory`.
> If it does not exist, tar creates an empty archive.
> I also check if I have read permission: `ls -la /source/directory`.
> And I run tar manually with the `-v` flag to see what it is including:
> `tar -czvf test.tar.gz /source/` — the v flag shows each file being added."

---

**Troubleshooting Q5:**
"Two instances of your backup script are running at the same time and corrupting files. How do you fix it?"

**Your answer:**
> "I add a lock file mechanism. At the start of the script:
> Check if a lock file exists. If yes, read the PID from it.
> Use `kill -0 PID` to check if that process is still running.
> If it is — exit immediately with a message.
> If it is not — the previous run crashed, delete the stale lock file and continue.
> Then create a new lock file with my PID.
> Register `trap 'rm -f /tmp/backup.lock' EXIT` so the lock is always deleted
> when the script finishes — even if it crashes."

---

## YOUR DAILY PRACTICE ROUTINE

Do this every day for 1 hour:

```
First 15 minutes:
  → Open terminal
  → Run every command from today's topic alone
  → See the output, understand it

Next 30 minutes:
  → Build today's plain script from scratch without looking at notes
  → If you get stuck, look at ONE hint, then write it yourself

Last 15 minutes:
  → Read your script out loud
  → Explain each line as if teaching a colleague
  → If you cannot explain a line, you do not own it yet — study that line
```

---

## HOW TO TALK ABOUT SCRIPTS IN INTERVIEWS

**Wrong way:**
"I wrote a system_info script that uses hostname, df, free commands..."

**Right way:**
"We had a problem where after provisioning new EC2 instances, the team was spending 15 minutes manually verifying each server. I wrote system_info.sh which collects hostname, OS version, kernel, CPU, RAM, and disk in one command. I use the hostname command for server name, df -h piped through awk to extract the disk percentage, and free -m with awk for memory calculations. This reduced verification time from 15 minutes to 5 seconds per server."

**The formula:**
```
Problem you solved → What the script does → Key commands you used → Impact (time saved)
```

---

> 📁 Save this file to:
> `Interview_Preparation_2026/shell_scripting/How_To_Read_And_Write_Scripts.md`
>
> Read this file before every practice session.
> This is your strategy guide — not the script, but HOW to learn the script.
