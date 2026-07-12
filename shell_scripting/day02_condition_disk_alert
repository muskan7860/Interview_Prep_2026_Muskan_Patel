# Day 02 — Conditions + disk_alert.sh
### 15-Day DevOps Shell Scripting Training | Muskan | 2026
### Concept → Practice → Real Script → Interview Questions

---

## 📌 TABLE OF CONTENTS
1. [What is a Condition?](#1-what-is-a-condition)
2. [if - elif - else — The Decision Maker](#2-if---elif---else)
3. [How to Test Things — The Square Brackets](#3-how-to-test-things)
4. [File Tests — Check If File or Folder Exists](#4-file-tests)
5. [Number Comparisons](#5-number-comparisons)
6. [String Comparisons](#6-string-comparisons)
7. [AND and OR — Multiple Conditions](#7-and-and-or)
8. [Single Bracket [ ] vs Double Bracket [[ ]]](#8-single-vs-double-bracket)
9. [case Statement — The Menu Selector](#9-case-statement)
10. [New Commands Used Today](#10-new-commands-used-today)
11. [Practice — Run These Yourself](#11-practice--run-these-yourself)
12. [Day 2 Resume Script — disk_alert.sh](#12-day-2-resume-script--disk_alertsh)
13. [Interview Questions + Exact Answers](#13-interview-questions--exact-answers)
14. [Resume Talking Points](#14-resume-talking-points)

---

## 1. What is a Condition?

### Plain English First

Every day you make decisions based on conditions.

```
IF it is raining THEN take an umbrella
ELSE go without umbrella

IF your phone battery is below 20% THEN charge it
ELSE keep using it

IF disk is above 80% THEN send alert
ELSE print OK
```

A **condition** in shell scripting works exactly the same way.
You check something. If it is true — do this. If it is false — do that.

### Why conditions matter in DevOps

Without conditions your script is just a list of commands that always run.
With conditions your script becomes **intelligent** — it can make decisions.

```
Without condition:  echo "Disk is full"         ← always prints, even when disk is fine
With condition:     if disk > 80% then alert    ← only alerts when actually needed
```

Every real DevOps script — health checks, deployments, backups — is built on conditions.

---

## 2. if - elif - else

### The Structure — Learn this shape first

```bash
if [ CONDITION ]; then
    # runs when condition is TRUE
elif [ ANOTHER CONDITION ]; then
    # runs when first is false but this is TRUE
else
    # runs when ALL conditions above are FALSE
fi
```

**Important rules:**
- `if` starts the block
- `then` comes after the condition (on same line after `;`)
- `fi` closes the block — it is `if` spelled backwards
- Indentation (spaces inside) is not required but makes it readable

### Simplest Example — Try this on your terminal

```bash
#!/bin/bash

DISK_USAGE=85

if [ $DISK_USAGE -gt 80 ]; then
    echo "WARNING: Disk is above 80 percent"
fi
```

Run it. Change 85 to 50. Run again. See the difference.

### With elif and else

```bash
#!/bin/bash

DISK_USAGE=65

if [ $DISK_USAGE -gt 90 ]; then
    echo "CRITICAL: Disk above 90 percent — take action NOW"
elif [ $DISK_USAGE -gt 80 ]; then
    echo "WARNING: Disk above 80 percent — watch this"
elif [ $DISK_USAGE -gt 70 ]; then
    echo "NOTICE: Disk above 70 percent — getting full"
else
    echo "OK: Disk is healthy at $DISK_USAGE percent"
fi
```

**Practice:** Change `DISK_USAGE` to 95, 85, 75, 50 one by one. Run each time. See which message appears.

### Real Example — Check if you are root user

```bash
#!/bin/bash

if [ "$USER" == "root" ]; then
    echo "You are running as root — be careful"
elif [ "$USER" == "muskan" ]; then
    echo "Hello Muskan — running as normal user"
else
    echo "Running as: $USER"
fi
```

---

## 3. How to Test Things

### The Square Brackets `[ ]` are a TEST command

When you write `[ $DISK -gt 80 ]` — you are running a test.
The test returns either:
- **0 = TRUE** (test passed)
- **1 = FALSE** (test failed)

Then `if` reads that result and decides what to do.

```bash
# You can even run tests alone on terminal to see the result:
[ 85 -gt 80 ]
echo $?        # prints 0 = TRUE (yes, 85 is greater than 80)

[ 50 -gt 80 ]
echo $?        # prints 1 = FALSE (no, 50 is not greater than 80)
```

**Try these right now on your terminal.**

---

## 4. File Tests

In DevOps scripts, before working with a file or folder, you ALWAYS check if it exists first.
Trying to read a file that doesn't exist = error = script crashes.

```bash
# Does this file exist?
[ -f /etc/passwd ]

# Does this directory exist?
[ -d /var/log ]

# Does this path exist? (file OR directory)
[ -e /etc/nginx ]
```

### Complete File Test Cheat Sheet

| Test | Meaning | Real Life Example |
|------|---------|------------------|
| `-f FILE` | File exists and is a regular file | Check if config file is present |
| `-d DIR` | Directory exists | Check if log folder exists |
| `-e PATH` | Path exists (file or dir) | Check if anything exists at this path |
| `-r FILE` | File exists and is readable | Check before reading a config |
| `-w FILE` | File exists and is writable | Check before writing to a file |
| `-x FILE` | File exists and is executable | Check before running a script |
| `-s FILE` | File exists and is NOT empty | Check if log file has content |
| `-L FILE` | File is a symbolic link | Check if it is a symlink |
| `! -f FILE` | File does NOT exist | Create file only if missing |

### Practice Examples — Run these on terminal

```bash
# Check if /etc/passwd exists
if [ -f /etc/passwd ]; then
    echo "File exists"
fi

# Check if /var/log exists as a directory
if [ -d /var/log ]; then
    echo "Log directory exists"
fi

# Check if a file does NOT exist — then create it
if [ ! -f /tmp/testfile.txt ]; then
    echo "File not found — creating it"
    touch /tmp/testfile.txt
else
    echo "File already exists"
fi

# Check before reading
if [ -r /etc/os-release ]; then
    cat /etc/os-release
else
    echo "Cannot read this file"
fi
```

---

## 5. Number Comparisons

**Important:** In shell, you cannot use `>` and `<` for numbers inside `[ ]`.
Those symbols mean file redirection in bash.
Instead we use special words:

| What you want to say | Shell syntax | Meaning |
|---------------------|-------------|---------|
| A equals B | `[ $A -eq $B ]` | eq = equal |
| A not equals B | `[ $A -ne $B ]` | ne = not equal |
| A less than B | `[ $A -lt $B ]` | lt = less than |
| A less than or equal B | `[ $A -le $B ]` | le = less than or equal |
| A greater than B | `[ $A -gt $B ]` | gt = greater than |
| A greater than or equal B | `[ $A -ge $B ]` | ge = greater than or equal |

### Memory trick
```
-eq  →  equal
-ne  →  not equal
-lt  →  less than       (L comes before G in alphabet, like less comes before greater)
-le  →  less or equal
-gt  →  greater than
-ge  →  greater or equal
```

### Practice

```bash
A=10
B=20

if [ $A -lt $B ]; then
    echo "$A is less than $B"
fi

if [ $A -ne $B ]; then
    echo "$A and $B are not equal"
fi

# Disk check pattern — most important for DevOps
DISK=75
THRESHOLD=80

if [ $DISK -ge $THRESHOLD ]; then
    echo "ALERT: Disk at $DISK percent"
else
    echo "OK: Disk at $DISK percent"
fi
```

---

## 6. String Comparisons

```bash
# Equal
[ "$A" == "$B" ]    # Are strings same?
[ "$A" = "$B" ]     # Also works (single = is POSIX style)

# Not equal
[ "$A" != "$B" ]    # Are strings different?

# Empty check
[ -z "$VAR" ]       # -z = zero length = string IS empty
[ -n "$VAR" ]       # -n = non-zero length = string is NOT empty
```

**Always quote your string variables** — `"$VAR"` not `$VAR`.
Without quotes, if the variable is empty, bash sees `[ == "hello" ]` which is an error.

### Practice

```bash
ENV="production"

if [ "$ENV" == "production" ]; then
    echo "Running in PRODUCTION — be careful"
elif [ "$ENV" == "staging" ]; then
    echo "Running in STAGING"
else
    echo "Running in: $ENV"
fi

# Check if variable is empty
NAME=""
if [ -z "$NAME" ]; then
    echo "Name is empty — please provide a name"
else
    echo "Hello $NAME"
fi

# Check if variable is NOT empty
SERVER="web-01"
if [ -n "$SERVER" ]; then
    echo "Server name provided: $SERVER"
fi
```

---

## 7. AND and OR

Sometimes you need to check TWO things at the same time.

### AND — both conditions must be true

```bash
# Method 1: && between two separate [ ] tests
if [ $DISK -gt 80 ] && [ -f /var/log/app.log ]; then
    echo "Disk high AND log file exists"
fi

# Method 2: Inside [[ ]] with &&
if [[ $DISK -gt 80 && -f /var/log/app.log ]]; then
    echo "Same result"
fi
```

### OR — at least one condition must be true

```bash
# Method 1: || between two separate [ ] tests
if [ "$ENV" == "prod" ] || [ "$ENV" == "production" ]; then
    echo "This is production environment"
fi
```

### The && and || shortcut pattern

This is used very heavily in real DevOps scripts:

```bash
# && means: if left side succeeds, run right side
mkdir /opt/app && echo "Directory created successfully"

# || means: if left side FAILS, run right side
mkdir /opt/app || echo "Failed to create directory"

# Combined: try this, if it fails, do that and exit
mkdir /opt/app || { echo "Cannot create directory"; exit 1; }
```

### Practice

```bash
# Check disk AND memory together
DISK=85
MEMORY=70

if [ $DISK -gt 80 ] && [ $MEMORY -gt 80 ]; then
    echo "CRITICAL: Both disk and memory are high"
elif [ $DISK -gt 80 ] || [ $MEMORY -gt 80 ]; then
    echo "WARNING: Either disk or memory is high"
else
    echo "OK: All metrics are healthy"
fi
```

---

## 8. Single Bracket [ ] vs Double Bracket [[ ]]

This question comes in almost every senior interview.

| Feature | Single `[ ]` | Double `[[ ]]` |
|---------|-------------|----------------|
| Standard | POSIX — works in sh and bash | bash only — does NOT work in sh |
| String comparison | Need to quote carefully | Safer — handles empty variables better |
| AND / OR inside | Use `-a` and `-o` (error-prone) | Use `&&` and `||` (clean) |
| Regex matching | Not supported | Supported with `=~` |
| Word splitting | Can cause issues with empty vars | Protected automatically |

### When to use which

```bash
# Use [ ] when you need POSIX portability (works in sh and bash)
if [ "$USER" = "root" ]; then
    echo "root"
fi

# Use [[ ]] for bash scripts — safer and more features
if [[ "$USER" == "root" ]]; then
    echo "root"
fi

# Only [[ ]] supports regex
IP="192.168.1.100"
if [[ "$IP" =~ ^[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}$ ]]; then
    echo "Valid IP"
fi
```

**Rule of thumb:** Since all our scripts use `#!/bin/bash`, always use `[[ ]]` — it is safer and cleaner.

---

## 9. case Statement — The Menu Selector

`case` is like a multi-choice menu. Instead of writing many `elif` blocks, you use `case` when you are checking one variable against many possible values.

### Plain English

```
Ask: What environment are you deploying to?
Choice 1: production → use prod database
Choice 2: staging    → use staging database
Choice 3: dev        → use localhost
Anything else        → show error
```

### Syntax

```bash
case "$VARIABLE" in
    value1)
        # commands for value1
        ;;
    value2|value3)
        # commands for value2 OR value3
        ;;
    *)
        # default — runs if nothing above matched
        ;;
esac
```

- `;;` means end of this choice — like a `break`
- `*` means "anything else" — the default case
- `esac` closes the case block — it is `case` spelled backwards

### Real DevOps Example

```bash
#!/bin/bash

ENVIRONMENT=$1    # get from command line argument

case "$ENVIRONMENT" in
    production|prod)
        DB_HOST="prod-db.internal"
        LOG_LEVEL="ERROR"
        echo "Deploying to PRODUCTION"
        ;;
    staging|stg)
        DB_HOST="staging-db.internal"
        LOG_LEVEL="WARN"
        echo "Deploying to STAGING"
        ;;
    development|dev)
        DB_HOST="localhost"
        LOG_LEVEL="DEBUG"
        echo "Deploying to DEVELOPMENT"
        ;;
    *)
        echo "ERROR: Unknown environment: $ENVIRONMENT"
        echo "Use: production, staging, or development"
        exit 1
        ;;
esac

echo "DB Host: $DB_HOST"
echo "Log Level: $LOG_LEVEL"
```

**Run it:**
```bash
chmod +x deploy_env.sh
./deploy_env.sh production
./deploy_env.sh staging
./deploy_env.sh dev
./deploy_env.sh unknown
```

---

## 10. New Commands Used Today

These commands appear in the disk_alert.sh script. Learn them now.

---

### `df -h` with number extraction

You already know `df -h /` shows disk usage. Today we extract just the number.

```bash
# Full output of df -h /
df -h /
# Filesystem      Size  Used Avail Use% Mounted on
# /dev/sda1       457G  143G  291G  33% /

# Get just the percentage number (without % sign)
df -h / | awk 'NR==2 {print $5}' | tr -d '%'
# Output: 33
```

`tr -d '%'` — tr = translate. `-d` = delete. Removes the `%` character.
We remove it because we need a plain number to do math comparisons.

**Practice:**
```bash
df -h / | awk 'NR==2 {print $5}'          # with % sign: 33%
df -h / | awk 'NR==2 {print $5}' | tr -d '%'    # just number: 33
```

---

### `df -h` for ALL partitions

```bash
# See all disk partitions
df -h

# Get usage of every partition — this is what we loop through
df -h | awk 'NR>1 {print $5, $6}'
# Output:
# 33% /
# 45% /boot
# 12% /tmp
```

`NR>1` means skip row 1 (the header row) and show all other rows.

---

### `systemctl is-active`

Checks if a Linux service is running.

```bash
# Check if nginx is running
systemctl is-active nginx

# Output if running:  active
# Output if stopped:  inactive

# The --quiet flag suppresses output — just gives exit code
# 0 = service is active (running)
# non-zero = service is not active
systemctl is-active --quiet nginx
echo $?    # 0 if running, 3 if stopped
```

This is how we check services in health check scripts.

**Practice:**
```bash
systemctl is-active ssh
systemctl is-active --quiet ssh && echo "SSH is running" || echo "SSH is stopped"
```

---

### `bc` — Calculator for decimals

Bash cannot do decimal math. `bc` is a calculator program.

```bash
# bash arithmetic — integers only
echo $((10/3))       # gives 3 (wrong, lost the decimal)

# bc — decimal math
echo "scale=1; 10/3" | bc     # gives 3.3
echo "scale=2; 10/3" | bc     # gives 3.33

# Real use — calculate memory percentage
MEM_USED=10240      # in MB
MEM_TOTAL=15360     # in MB
MEM_PCT=$(echo "scale=1; $MEM_USED * 100 / $MEM_TOTAL" | bc)
echo "Memory used: $MEM_PCT%"
```

**Practice:**
```bash
echo "scale=2; 143 * 100 / 457" | bc     # what % of 457 is 143?
```

---

### `free -m` — RAM in megabytes

```bash
free -m
#               total   used   free   shared  buff/cache  available
# Mem:          15360  10240   1024     365        4096       4500
# Swap:          2048      0   2048

# Get total RAM in MB
free -m | awk '/^Mem:/ {print $2}'    # 15360

# Get used RAM in MB
free -m | awk '/^Mem:/ {print $3}'    # 10240
```

We use `-m` (megabytes) instead of `-h` (human readable) because we need plain numbers for math.

---

### `uptime` load average extraction

```bash
uptime
# 19:37:53 up 14:22,  2 users,  load average: 1.72, 1.28, 1.15

# Get only the 1-minute load average (second to last field minus 2)
uptime | awk '{print $(NF-2)}' | tr -d ','
# Output: 1.72
```

`NF` = number of fields (last field). `NF-2` = third from last = 1-minute load average.

---

## 11. Practice — Run These Yourself

Do each exercise. Write the answer yourself first. Then check.

---

**Exercise 1:** Check if the number 75 is greater than 80. Print the result.

```bash
NUM=75
if [ $NUM -gt 80 ]; then
    echo "Greater than 80"
else
    echo "Less than or equal to 80"
fi
```

---

**Exercise 2:** Check if `/etc/hosts` file exists and is readable.

```bash
if [ -f /etc/hosts ] && [ -r /etc/hosts ]; then
    echo "File exists and is readable"
    cat /etc/hosts
else
    echo "File missing or not readable"
fi
```

---

**Exercise 3:** Check your actual disk usage. Alert if above 80.

```bash
DISK=$(df -h / | awk 'NR==2 {print $5}' | tr -d '%')
echo "Current disk usage: $DISK%"

if [ $DISK -gt 80 ]; then
    echo "ALERT: Disk is above 80 percent"
else
    echo "OK: Disk is healthy"
fi
```

---

**Exercise 4:** Check if SSH service is running on your machine.

```bash
if systemctl is-active --quiet ssh; then
    echo "SSH is running"
else
    echo "SSH is not running"
fi
```

---

**Exercise 5:** Write a case statement — ask user to input a day and print if it is a weekday or weekend.

```bash
read -p "Enter day (Monday/Tuesday/.../Sunday): " DAY

case "$DAY" in
    Monday|Tuesday|Wednesday|Thursday|Friday)
        echo "$DAY is a weekday"
        ;;
    Saturday|Sunday)
        echo "$DAY is a weekend"
        ;;
    *)
        echo "Invalid day entered"
        ;;
esac
```

---

## 12. Day 2 Resume Script — disk_alert.sh

This script is the foundation of Assignment 1.
It checks disk usage on ALL partitions and shows color-coded status.
This is exactly what interviewers ask you to write on the spot.

```bash
#!/bin/bash
# ============================================================
# Script Name : disk_alert.sh
# Description : Checks disk usage on all mounted partitions.
#               Prints CRITICAL (red) if above 90%
#               Prints WARNING  (yellow) if above 80%
#               Prints OK       (green) if healthy
#               Exits with code 1 if any partition is critical.
#               Used in CI/CD pipelines as a pre-deploy check.
# Author      : Muskan Patel
# GitHub      : github.com/muskan7860
# Created On  : 2026-07-05
# Version     : 1.0
# Usage       : ./disk_alert.sh
#               ./disk_alert.sh 75   (custom warning threshold)
# ============================================================

set -eu

# ── COLOR CODES ──────────────────────────────────────────────
RED='\033[0;31m'
YELLOW='\033[1;33m'
GREEN='\033[0;32m'
BLUE='\033[0;34m'
NC='\033[0m'

# ── THRESHOLDS ───────────────────────────────────────────────
# These are the limits we check against
# If user passes a custom threshold as argument $1, use that
# Otherwise use the default values below
# ${1:-80} means: use $1 if provided, otherwise use 80
WARN_THRESHOLD=${1:-80}
CRIT_THRESHOLD=90

# ── COUNTERS ─────────────────────────────────────────────────
# We count problems found so we can report a summary at the end
WARN_COUNT=0
CRIT_COUNT=0

# ── HEADER ───────────────────────────────────────────────────
echo -e "${BLUE}============================================================${NC}"
echo -e "${BLUE}           DISK USAGE HEALTH CHECK                         ${NC}"
echo -e "${BLUE}============================================================${NC}"
echo    " Host     : $(hostname)"
echo    " Date     : $(date '+%Y-%m-%d %H:%M:%S')"
echo    " Warning  : above ${WARN_THRESHOLD}%"
echo    " Critical : above ${CRIT_THRESHOLD}%"
echo -e "${BLUE}------------------------------------------------------------${NC}"
printf  " %-25s %-8s %-8s %-8s %-8s\n" "PARTITION" "TOTAL" "USED" "FREE" "STATUS"
echo -e "${BLUE}------------------------------------------------------------${NC}"

# ── CHECK ALL PARTITIONS ─────────────────────────────────────
# df -h → shows all disk partitions
# awk 'NR>1' → skip the first row (header row)
# We then read each line: usage% and mount point

# while read loop:
# each time through the loop, USAGE and MOUNT get the next partition's data
# IFS= means do not split on spaces inside field values
while IFS= read -r LINE; do

    # Extract each column from this line using awk
    # $5 = usage percentage (like 33%)
    # $6 = mount point (like / or /boot)
    # $2 = total size
    # $3 = used space
    # $4 = free space
    USAGE=$(echo "$LINE" | awk '{print $5}' | tr -d '%')
    MOUNT=$(echo "$LINE" | awk '{print $6}')
    TOTAL=$(echo "$LINE" | awk '{print $2}')
    USED=$(echo "$LINE"  | awk '{print $3}')
    FREE=$(echo "$LINE"  | awk '{print $4}')

    # Skip lines that don't have a proper mount point
    # -z check: if MOUNT is empty, skip this line
    if [ -z "$MOUNT" ] || [ -z "$USAGE" ]; then
        continue
    fi

    # Skip special system partitions that are not real disks
    # tmpfs = temporary filesystem in RAM
    # udev = device manager filesystem
    # These don't represent real disk space
    case "$MOUNT" in
        /proc|/sys|/dev|/run*) continue ;;
    esac

    # ── DECIDE STATUS BASED ON USAGE ─────────────────────────
    # Compare the usage number against our thresholds
    # Then print the result with appropriate color

    if [ "$USAGE" -ge "$CRIT_THRESHOLD" ]; then
        # CRITICAL: above 90%
        STATUS="${RED}CRITICAL${NC}"
        CRIT_COUNT=$((CRIT_COUNT + 1))

    elif [ "$USAGE" -ge "$WARN_THRESHOLD" ]; then
        # WARNING: above 80% but below 90%
        STATUS="${YELLOW}WARNING${NC}"
        WARN_COUNT=$((WARN_COUNT + 1))

    else
        # OK: healthy, below warning threshold
        STATUS="${GREEN}OK${NC}"
    fi

    # Print the row for this partition
    # printf %-25s means: left-align text in a 25-character wide column
    # This makes all columns line up neatly
    printf " %-25s %-8s %-8s %-8s " "$MOUNT" "$TOTAL" "$USED" "$FREE"
    echo -e "${STATUS} (${USAGE}%)"

done < <(df -h | awk 'NR>1')
# df -h | awk 'NR>1' → get all disk lines except header
# < <(...) → process substitution: treat command output as a file to read from
# This is safer than piping because piping creates a subshell

# ── SUMMARY ──────────────────────────────────────────────────
echo -e "${BLUE}------------------------------------------------------------${NC}"
echo    " SUMMARY"
echo -e "${BLUE}------------------------------------------------------------${NC}"

if [ "$CRIT_COUNT" -gt 0 ]; then
    echo -e " ${RED}CRITICAL issues : $CRIT_COUNT partition(s) above ${CRIT_THRESHOLD}%${NC}"
fi

if [ "$WARN_COUNT" -gt 0 ]; then
    echo -e " ${YELLOW}Warnings        : $WARN_COUNT partition(s) above ${WARN_THRESHOLD}%${NC}"
fi

if [ "$CRIT_COUNT" -eq 0 ] && [ "$WARN_COUNT" -eq 0 ]; then
    echo -e " ${GREEN}All partitions are healthy.${NC}"
fi

echo -e "${BLUE}============================================================${NC}"

# ── EXIT CODE ────────────────────────────────────────────────
# In DevOps, scripts must return the right exit code
# 0 = everything is fine (used by Jenkins, monitoring tools to confirm OK)
# 1 = something is wrong (used by Jenkins to fail the build/pipeline)
# This way other tools can automatically detect if there is a problem

if [ "$CRIT_COUNT" -gt 0 ]; then
    exit 1    # signal failure to calling system (Jenkins, cron, etc.)
else
    exit 0    # signal success
fi
```

### How to run it

```bash
# Create and run
cd ~/Lab_Practice/shell-scripting
nano disk_alert.sh
# paste script → Ctrl+O → Enter → Ctrl+X
chmod +x disk_alert.sh

# Normal run
./disk_alert.sh

# With custom warning threshold (alert at 75% instead of 80%)
./disk_alert.sh 75

# Debug mode — see every line execute
bash -x disk_alert.sh

# Check what exit code the script returned
./disk_alert.sh
echo "Exit code: $?"
# 0 = all OK
# 1 = critical issue found

# Add to GitHub
git add disk_alert.sh
git commit -m "Day02: disk_alert.sh — disk usage health check with thresholds"
git push origin main
```

### Expected Output

```
============================================================
           DISK USAGE HEALTH CHECK
============================================================
 Host     : muskan-ThinkPad-L14-Gen-1
 Date     : 2026-07-05 19:37:53
 Warning  : above 80%
 Critical : above 90%
------------------------------------------------------------
 PARTITION                 TOTAL    USED     FREE     STATUS
------------------------------------------------------------
 /                         457G     143G     291G     OK (33%)
 /boot                     512M     120M     392M     OK (23%)
 /tmp                      7.5G     1.2G     6.3G     OK (16%)
------------------------------------------------------------
 SUMMARY
------------------------------------------------------------
 All partitions are healthy.
============================================================
```

---

## 13. Interview Questions + Exact Answers

### 🟢 BEGINNER LEVEL

---

**Q1: What is the difference between `if [ ]` and `if [[ ]]` in bash?**

Say this out loud:
> "Both are used for conditional tests in bash. The single bracket `[ ]` is the POSIX standard test command — it works in both sh and bash. The double bracket `[[ ]]` is a bash built-in keyword that is safer and more powerful. With `[[ ]]` you don't need to quote variables to protect against word splitting, you can use `&&` and `||` directly inside instead of `-a` and `-o`, and you can use regex matching with `=~`. In my scripts I always use `[[ ]]` since I write for bash, and it prevents hard-to-find bugs with empty or unset variables."

---

**Q2: How do you compare numbers in bash? Why can't you use > and <?**

Say this out loud:
> "In bash we use `-gt`, `-lt`, `-ge`, `-le`, `-eq`, `-ne` for number comparisons inside square brackets. We cannot use `>` and `<` because bash interprets those as file redirection operators — `>` would try to create a file instead of comparing numbers. So `[ 85 > 80 ]` would create a file called `80`, not compare the numbers. The correct syntax is `[ 85 -gt 80 ]`. Inside double brackets `[[ ]]` you can use `>` and `<` for string comparison, but for numbers you still use the `-gt` style operators."

---

**Q3: What does `exit 1` mean and why is it important in DevOps?**

Say this out loud:
> "Every script exits with a code when it finishes. Exit code 0 means success. Exit code 1 means failure. In DevOps, tools like Jenkins, cron, and monitoring systems automatically read this exit code. If a health check script exits with 1, Jenkins knows the build should fail. If it exits with 0, Jenkins proceeds. Without the correct exit code, your script might detect a critical disk problem but Jenkins would think everything is fine and continue the deployment. So exit codes are how shell scripts communicate their result to the system that called them."

---

**Q4: What is a case statement and when do you use it instead of if-elif?**

Say this out loud:
> "A case statement is like a menu selector — you check one variable against multiple possible values. I use case when I have one variable that could have many different values, like an environment name that could be production, staging, or dev. It is cleaner than writing many elif blocks. Case also supports pattern matching — you can write `production|prod)` to match both spellings. The case block opens with case and closes with esac — which is case spelled backwards."

---

**Q5: What does `[ -f file ]` vs `[ -d file ]` vs `[ -e file ]` mean?**

Say this out loud:
> "`-f` checks if the path exists AND is a regular file — not a directory or symlink. `-d` checks if the path exists AND is a directory. `-e` checks if the path exists regardless of type — file, directory, symlink, anything. In production scripts I always check before acting on files. For example, before reading a config file I check `-f` to confirm it exists and is a regular file. Before creating a log directory I check `! -d` to avoid trying to create it when it already exists."

---

### 🟡 MID LEVEL

---

**Q6: What is process substitution `< <(command)` and why use it instead of a pipe?**

Say this out loud:
> "Process substitution `< <(command)` treats the output of a command as if it were a file. In my disk_alert script I use `while read LINE; done < <(df -h | awk 'NR>1')` instead of `df -h | awk 'NR>1' | while read LINE`. The reason is that when you pipe into a while loop, bash runs the loop in a subshell. Any variables you set inside the loop are lost when the loop ends because the subshell has its own copy of variables. With process substitution the while loop runs in the current shell, so variables like CRIT_COUNT and WARN_COUNT are preserved. This is a real production bug that catches many engineers off guard."

---

**Q7: How does `${1:-80}` work? What is it called?**

Say this out loud:
> "This is called a default value substitution. `$1` is the first argument passed to the script. If the user did not pass any argument, `$1` is empty. `${1:-80}` means: use `$1` if it is set and not empty, otherwise use `80` as the default. This makes scripts flexible — they work with sensible defaults but users can override them. I use this pattern in my disk_alert script so the warning threshold defaults to 80 percent but someone can run `./disk_alert.sh 75` to get alerts at 75 percent instead."

---

**Q8: Explain how you would write a pre-deployment disk check in a Jenkins pipeline.**

Say this out loud:
> "I would write a shell script called disk_alert.sh that checks all partitions using `df`, extracts the usage percentage with awk, and compares it against a threshold using an if condition. If any partition is above 90 percent, the script exits with code 1. In the Jenkins pipeline I add a stage called Pre-Deploy Checks that runs this script. Jenkins reads the exit code — if it gets 1, it marks the build as failed and stops the deployment automatically. This prevents deployments from failing mid-way because the server ran out of disk space while downloading artifacts or extracting files."

---

**Q9: What is the difference between `&&` and `||` when used between commands?**

Say this out loud:
> "`&&` means AND — run the right side only if the left side SUCCEEDED. `mkdir /opt/app && echo 'created'` — the echo only runs if mkdir worked. `||` means OR — run the right side only if the left side FAILED. `mkdir /opt/app || exit 1` — exit only runs if mkdir failed. I combine them in error handling: `mkdir /opt/app || { echo 'Failed to create directory'; exit 1; }` — the curly braces group multiple commands to run on failure. This pattern is everywhere in production scripts as a compact alternative to if-then-fi blocks."

---

**Q10: How do you handle a partition that reports 100% in df but the server is still working?**

Say this out loud:
> "Linux reserves about 5 percent of disk space for the root user — this is called the reserved blocks percentage. So when df shows 100 percent, regular users cannot write but root still can, which is why the server keeps running. In my disk check script I check at 90 percent CRITICAL so we catch the problem before hitting 100. I also check df output with the `-h` flag and extract the usage number with awk. Some pseudo-filesystems like tmpfs and devtmpfs also show high usage but they are in RAM, not real disk. I filter those out in my script using a case statement that skips /proc, /sys, /dev, and /run mount points."

---

## 14. Resume Talking Points

### Add to your resume:
```
• Built disk_alert.sh — automated disk usage monitoring script that checks
  all mounted partitions, applies configurable warning (80%) and critical
  (90%) thresholds, outputs color-coded status, and returns exit code 1
  on critical issues for integration with Jenkins CI/CD pipeline health gates
```

### Say this in interviews:
> "I built disk_alert.sh which checks all disk partitions on a server using df. It extracts the usage percentage with awk, strips the percent sign with tr, and compares the number against warning and critical thresholds using if conditions. The thresholds are configurable — the warning threshold defaults to 80 percent but can be overridden by passing an argument. I use a while loop with process substitution to read each partition line from df without losing variables in a subshell. The script outputs color-coded results — green for OK, yellow for warning, red for critical — and exits with code 1 if anything is critical. This exit code is read by Jenkins in our CI/CD pipeline as a pre-deployment gate — if disk is critical, the deployment is blocked automatically."

---

## 📋 Quick Reference — Day 2 Conditions Cheat Sheet

```bash
# ── BASIC STRUCTURE ───────────────────────────────────────────
if [ CONDITION ]; then
    commands
elif [ CONDITION ]; then
    commands
else
    commands
fi

# ── NUMBER TESTS ──────────────────────────────────────────────
[ $A -eq $B ]    # equal
[ $A -ne $B ]    # not equal
[ $A -lt $B ]    # less than
[ $A -le $B ]    # less than or equal
[ $A -gt $B ]    # greater than
[ $A -ge $B ]    # greater than or equal

# ── FILE TESTS ────────────────────────────────────────────────
[ -f FILE ]      # is a regular file
[ -d DIR  ]      # is a directory
[ -e PATH ]      # exists (anything)
[ -r FILE ]      # readable
[ -w FILE ]      # writable
[ -x FILE ]      # executable
[ -s FILE ]      # not empty
[ ! -f FILE ]    # does NOT exist

# ── STRING TESTS ──────────────────────────────────────────────
[ -z "$VAR" ]         # empty string
[ -n "$VAR" ]         # not empty
[ "$A" == "$B" ]      # equal
[ "$A" != "$B" ]      # not equal

# ── COMBINING ─────────────────────────────────────────────────
[ COND1 ] && [ COND2 ]   # AND
[ COND1 ] || [ COND2 ]   # OR
[[ COND1 && COND2 ]]     # AND inside double bracket

# ── SHORTCUTS ─────────────────────────────────────────────────
command && echo "success"          # run if command succeeds
command || echo "failed"           # run if command fails
command || { echo "fail"; exit 1; } # fail with cleanup

# ── CASE STATEMENT ────────────────────────────────────────────
case "$VAR" in
    value1)        commands ;; 
    value2|value3) commands ;;
    *)             default  ;;
esac

# ── DEFAULT VALUE ─────────────────────────────────────────────
VAR=${1:-default}      # use $1 if given, else "default"
```

---

> 📁 Save this file to:
> `Interview_Preparation_2026/shell_scripting/day02/Day02_Conditions_and_disk_alert.md`
>
> ✅ Day 02 Complete
> 📅 Next → Day 03: Loops (for, while, until) + health_check.sh (Assignment 1 complete)
