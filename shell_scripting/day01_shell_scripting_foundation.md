# Day 01 — Shell Scripting: Complete Foundation
### 15-Day DevOps Shell Scripting Interview Preparation | Muskan | 2026

---

## 📌 TABLE OF CONTENTS
1. [What is a Shell? — Start from Zero](#1-what-is-a-shell)
2. [What is the Linux Architecture?](#2-what-is-the-linux-architecture)
3. [What is Shell Scripting?](#3-what-is-shell-scripting)
4. [Types of Shells](#4-types-of-shells)
5. [What is an Interpreter?](#5-what-is-an-interpreter)
6. [The Shebang Line — #!/bin/bash](#6-the-shebang-line)
7. [How Linux Executes Your Script — Full Flow](#7-how-linux-executes-your-script)
8. [Variables — The Labeled Box Concept](#8-variables)
9. [echo and read Commands](#9-echo-and-read-commands)
10. [Comments in Shell](#10-comments-in-shell)
11. [Special Variables — Most Asked in Interviews](#11-special-variables)
12. [Day 1 Resume Script — system_info.sh](#12-day-1-resume-script)
13. [Lab Instructions — Run on Your Machine](#13-lab-instructions)
14. [Interview Questions with Exact Answers](#14-interview-questions)
15. [Resume Talking Points](#15-resume-talking-points)

---

## 1. What is a Shell?

### 🧠 Understand Like a Child First

Imagine your computer is a **big restaurant kitchen**.

- The **Kitchen Chef** = **Linux Kernel** — he is the real boss. He controls everything: memory, CPU, disk, network, hardware. But you CANNOT talk to him directly.
- The **Waiter** = **Shell** — he stands between you and the chef. You give your order to the waiter. The waiter passes it to the chef. The chef cooks it and the result comes back to you.
- **You** = the person typing commands.

So when you type `ls` in the terminal:
```
YOU type "ls"
    → SHELL receives it, understands it, passes to KERNEL
        → KERNEL talks to HARDWARE (disk)
            → Disk sends back file list
        → KERNEL returns result to SHELL
    → SHELL prints it on your screen
```

### 📖 Professional Definition (Say this in interview)
> "A Shell is a command-line interpreter that acts as an interface between the user and the Linux kernel. It accepts human-readable commands, interprets them, and communicates with the kernel to execute them."

---

## 2. What is the Linux Architecture?

This is important. Many interviewers ask: **"Explain Linux architecture."**

```
┌─────────────────────────────────────────┐
│             YOU (User)                  │  ← Types commands
├─────────────────────────────────────────┤
│         SHELL (bash/sh/zsh)             │  ← Interprets commands
│    This is where your scripts run       │
├─────────────────────────────────────────┤
│      SYSTEM LIBRARIES (glibc)           │  ← Helper functions
├─────────────────────────────────────────┤
│         LINUX KERNEL                    │  ← Core of the OS
│  (Memory Mgmt, Process Mgmt, I/O)       │
├─────────────────────────────────────────┤
│           HARDWARE                      │  ← CPU, RAM, Disk, NIC
└─────────────────────────────────────────┘
```

**Key point for interview:** Shell sits between the user and the kernel. It is NOT part of the kernel. You can change your shell without changing the kernel.

---

## 3. What is Shell Scripting?

### 🧠 Understand Like a Child First

Every morning you do the same things manually:
1. Open terminal
2. Check disk space
3. Check RAM
4. Check if nginx is running
5. Write results in a log file

This takes 10 minutes every day. That is **70 minutes per week wasted**.

A **Shell Script** is a file where you write all these steps ONCE. Then you just run one command and all 5 steps happen automatically in seconds.

Think of it like a **recipe card**. You write the recipe once. Anyone can follow it anytime. The computer follows your recipe card = shell script.

### 📖 Professional Definition
> "A shell script is a plain text file containing a sequence of shell commands. When executed, the shell reads and runs each command in order, enabling automation of repetitive tasks, system administration, CI/CD pipeline steps, and infrastructure management."

### Why Shell Scripting is CRITICAL for DevOps (memorize this list):

| Use Case | Real Script Name |
|----------|-----------------|
| Server provisioning on EC2 | `bootstrap.sh` (User Data script) |
| Application deployment | `deploy.sh` |
| Health checks & monitoring | `health_check.sh` |
| Log rotation & cleanup | `log_rotate.sh` |
| Database backup to S3 | `backup.sh` |
| Jenkins build/test steps | `build.sh`, `test.sh` |
| Bulk user creation | `create_users.sh` |
| Scheduled reporting | `daily_report.sh` (via cron) |

---

## 4. Types of Shells

Think of different shells like different languages. French waiter vs English waiter — both take your order, but speak differently.

```bash
# See all shells installed on your machine:
cat /etc/shells

# See which shell you are currently using:
echo $SHELL

# See which shell is running right now:
echo $0
```

| Shell | Full Name | Path | Use |
|-------|-----------|------|-----|
| `sh` | Bourne Shell | `/bin/sh` | Original Unix shell. Very basic. |
| `bash` | Bourne Again Shell | `/bin/bash` | **Default on Linux. Use this always.** |
| `zsh` | Z Shell | `/bin/zsh` | Default on macOS. More features. |
| `ksh` | Korn Shell | `/bin/ksh` | Old enterprise Unix systems |
| `csh` | C Shell | `/bin/csh` | C-like syntax. Rarely used now. |
| `dash` | Debian Almquist Shell | `/bin/dash` | Lightweight. Ubuntu uses for `/bin/sh` |

### Interview Answer: "Why do we always use bash in scripts?"
> "Bash is the default shell on almost all Linux distributions — Ubuntu, RHEL, Amazon Linux 2 and 2023, CentOS. It supports arrays, functions, error handling with `set -e`, and advanced string manipulation that basic `sh` does not have. So for production DevOps scripts, bash gives us the most features and widest compatibility."

---

## 5. What is an Interpreter?

### 🧠 Understand Like a Child First

Imagine you speak only Hindi. Your computer speaks only binary (0s and 1s). 

An **interpreter** is like a **live translator** sitting between you. You speak one Hindi sentence. The translator immediately converts it to binary and the computer executes it. Then you speak the next sentence. The translator converts that. And so on — **one line at a time**.

This is different from a **compiler** which reads your ENTIRE speech first, translates everything to binary, then lets the computer run it.

### Interpreter vs Compiler:

| | Interpreter (bash, python) | Compiler (C, Java) |
|--|--------------------------|-------------------|
| How it works | Reads and runs **line by line** | Translates **entire program** first, then runs |
| Error behavior | **Stops at the line with error** | Shows all errors before running |
| Speed | Slower | Faster |
| Examples | bash, python, ruby | gcc (C), javac (Java) |

### Why this matters for DevOps (say in interview):
> "Because bash is an interpreter that runs line by line, if line 50 has an error, lines 1 to 49 have already executed. This is why we use `set -e` at the top of production scripts — it tells bash to stop immediately when any command fails, preventing partial execution which can leave the system in a broken state."

---

## 6. The Shebang Line

### 🧠 Understand Like a Child First

Imagine you receive a letter. Before reading it, you see a label on the envelope: **"Read this in French"**. So you know to apply French grammar rules when reading.

The shebang `#!/bin/bash` is that label on your script. It tells Linux: **"Read and run this file using the bash program."**

### Breaking it down — character by character:

```
#!/bin/bash
│││└──────── Path to the interpreter program
││└───────── The separator
│└────────── ! (exclamation mark) — tells kernel this is a special directive
└─────────── # (hash) — together with ! forms the "shebang" or "hashbang"
```

| Part | Name | Meaning |
|------|------|---------|
| `#` | Hash | Start of the shebang marker |
| `!` | Bang | Together = "shebang" or "hashbang" |
| `/bin/bash` | Interpreter path | Full path to the bash program |

### What happens WITHOUT shebang?

```bash
# Without shebang:
# → Linux uses your current default shell (could be zsh, dash, sh)
# → Your script may behave differently on different machines
# → Arrays and bash-specific features may BREAK silently
# → ALWAYS include shebang for predictable results
```

### Common Shebang Lines:

```bash
#!/bin/bash              # Standard. Use this in ALL your scripts.
#!/bin/sh                # POSIX portable but fewer features
#!/usr/bin/env bash      # Finds bash wherever it is installed (portable across systems)
#!/usr/bin/env python3   # Python script
#!/usr/bin/env node      # Node.js script
```

### Interview Answer for "What is a shebang?":
> "The shebang is the very first line of a shell script, written as `#!/bin/bash`. The `#!` characters are a special marker that the Linux kernel reads before anything else. It tells the kernel which interpreter program to use to execute the rest of the file. Without the shebang, the OS falls back to the user's current default shell, which can cause inconsistent behavior when the script runs on different servers with different default shells."

---

## 7. How Linux Executes Your Script — Full Flow

This is the **most important concept**. Most candidates cannot explain this. If you explain this, the interviewer will be impressed.

### Step-by-step what happens when you type `./system_info.sh`:

```
STEP 1: You type → ./system_info.sh and press Enter
         │
         ▼
STEP 2: Shell passes this to the KERNEL
         │
         ▼
STEP 3: Kernel checks FILE PERMISSIONS
         → Does this file have execute (x) permission?
         → If NO  → "Permission denied" error. STOP.
         → If YES → continue to next step
         │
         ▼
STEP 4: Kernel reads LINE 1 of the file
         → Sees: #!/bin/bash
         → Understands: "Use /bin/bash as interpreter"
         │
         ▼
STEP 5: Kernel launches /bin/bash and hands it the script file
         │
         ▼
STEP 6: bash reads the script LINE BY LINE (interpreter behavior)
         → Line 2: # comment → skip
         → Line 3: HOSTNAME=$(hostname) → run hostname, store result
         → Line 4: DATE=$(date) → run date, store result
         → Line 5: echo "$HOSTNAME" → print to screen
         → ...continues until end of file
         │
         ▼
STEP 7: Script finishes → bash exits → control returns to your terminal
```

### Visual Diagram:

```
You: ./system_info.sh
         │
    [Permission Check: chmod +x]
         │
    [Read Line 1: #!/bin/bash]
         │
    [Launch /bin/bash interpreter]
         │
    [bash reads line by line]
    ┌────────────────────────┐
    │ hostname → "web-01"   │
    │ date → "2026-06-22"   │
    │ free -h → "7.5G/16G"  │
    │ echo → print screen   │
    └────────────────────────┘
         │
    [Script done. Exit code: 0]
```

---

## 8. Variables — The Labeled Box Concept

### 🧠 Understand Like a Child First

Imagine you have a kitchen shelf with labeled boxes:
- Box labeled **"SUGAR"** contains: *100 grams*
- Box labeled **"SERVER_NAME"** contains: *web-server-01*

Every time you need sugar, you just say "get me SUGAR" — you don't repeat "100 grams" every time. Same in shell scripts. Store once, use everywhere.

### Syntax Rules — THE MOST COMMON SOURCE OF ERRORS:

```bash
# ✅ CORRECT — No spaces around = sign
SERVER_NAME="web-server-01"
PORT=8080
IS_RUNNING=true

# ❌ WRONG — Spaces break everything
SERVER_NAME = "web-server-01"   # bash thinks SERVER_NAME is a COMMAND!
PORT = 8080                      # Error!
```

**Why does space cause error?** Because bash reads `SERVER_NAME` as a command name, `=` as first argument, `"web-server-01"` as second argument. There is no command called `SERVER_NAME`, so it fails.

### Three Ways to Set Variables:

```bash
# METHOD 1: Direct assignment
NAME="Muskan"
AGE=25
IS_PROD=true

# METHOD 2: Command substitution — capture a command's OUTPUT into a variable
# Modern syntax: $() — USE THIS ALWAYS
HOSTNAME=$(hostname)
TODAY=$(date '+%Y-%m-%d')
DISK_USAGE=$(df -h / | awk 'NR==2 {print $5}')
FILE_COUNT=$(ls /var/log | wc -l)

# METHOD 3: Backtick — OLD way, AVOID THIS
HOSTNAME=`hostname`    # works but hard to read, hard to nest
```

### How to USE a Variable — Always use $ prefix:

```bash
NAME="Muskan"

echo $NAME              # ✅ prints: Muskan
echo "$NAME"            # ✅ SAFER — always quote variables
echo "${NAME}"          # ✅ CLEAREST — use when joining with other text
echo "${NAME}_backup"   # ✅ prints: Muskan_backup (need {} here)
echo "$NAME_backup"     # ❌ WRONG — bash looks for variable $NAME_backup, not $NAME
```

**Rule: Always use double quotes around variables → `"$VAR"` or `"${VAR}"`**

### Variable Types:

```bash
# 1. Local variable — only in current script/session
MY_VAR="hello"

# 2. Environment variable — shared with child processes
export DB_HOST="prod-db.internal"
# Now any script launched FROM this script can also read $DB_HOST

# 3. Read-only variable — cannot be changed
readonly MAX_RETRIES=3
MAX_RETRIES=5    # This will throw an error

# 4. Unset (delete) a variable
unset MY_VAR
echo $MY_VAR    # prints nothing now
```

### Built-in System Variables — Already exist, just use them:

```bash
echo $HOME      # /home/muskan
echo $USER      # muskan
echo $PWD       # /home/muskan/scripts (current directory)
echo $PATH      # Where Linux looks for programs
echo $SHELL     # /bin/bash
echo $HOSTNAME  # machine name
echo $RANDOM    # random number 0-32767
echo $LINENO    # current line number in script (for debugging)
echo $$         # PID of current script
echo $?         # Exit status of LAST command (0=success)
```

### Arithmetic in Shell:

```bash
NUM1=10
NUM2=3

# Integer math using $(( ))
SUM=$((NUM1 + NUM2))          # 13
DIFF=$((NUM1 - NUM2))         # 7
PRODUCT=$((NUM1 * NUM2))      # 30
QUOTIENT=$((NUM1 / NUM2))     # 3 (integer division, no decimal)
REMAINDER=$((NUM1 % NUM2))    # 1 (modulo = remainder)
POWER=$((NUM1 ** 2))          # 100

echo "Sum: $SUM"

# Decimal math — bash cannot do it natively, use bc
DECIMAL=$(echo "scale=2; 10/3" | bc)   # 3.33
# scale=2 means 2 decimal places
```

---

## 9. echo and read Commands

### `echo` — Print output to screen

```bash
# Basic
echo "Hello World"

# With variable
NAME="Muskan"
echo "Hello $NAME"

# -n flag: no newline at end (cursor stays on same line)
echo -n "Enter your name: "

# -e flag: enable special characters
echo -e "Line 1\nLine 2"     # \n = newline
echo -e "Col1\tCol2"         # \t = tab

# Colored output using escape codes
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'    # NC = No Color (reset)

echo -e "${GREEN}✅ Success${NC}"
echo -e "${RED}❌ Error${NC}"
echo -e "${YELLOW}⚠️  Warning${NC}"

# Write to a file instead of screen
echo "Deployment started" >> /var/log/deploy.log    # >> = APPEND (add to end)
echo "Fresh log" > /var/log/deploy.log              # > = OVERWRITE (careful!)
```

### `read` — Accept input FROM the user

```bash
# Basic read — stores what user types into variable NAME
read NAME
echo "You entered: $NAME"

# -p flag: show a prompt message
read -p "Enter server name: " SERVER
echo "Server chosen: $SERVER"

# -s flag: silent — hides what user types (for passwords)
read -s -p "Enter password: " PASSWORD
echo ""    # need this newline since -s suppresses it

# -t flag: timeout — wait only N seconds
read -t 10 -p "Confirm deployment? (y/n): " CONFIRM
if [ $? -ne 0 ]; then
    echo "Timeout! No confirmation received."
fi

# Read into multiple variables at once
read -p "Enter first and last name: " FIRST LAST
echo "First: $FIRST, Last: $LAST"
```

---

## 10. Comments in Shell

```bash
#!/bin/bash

# This is a single-line comment
# Everything after # on a line is ignored by bash

# ALWAYS write a proper header at top of every script:
# ============================================================
# Script Name : system_info.sh
# Description : Displays server health information
# Author      : Muskan Patel
# Created On  : 2026-06-22
# Version     : 1.0
# Usage       : ./system_info.sh
# ============================================================

# Multi-line comment trick (not official syntax but widely used):
: '
This entire block is treated as a comment.
Bash sees : (colon) as a no-op command,
and the single-quoted string as its argument.
Nothing in here gets executed.
'

# Best practice: comment WHAT and WHY, not HOW
# BAD comment (obvious, adds no value):
i=$((i+1))    # add 1 to i

# GOOD comment (explains WHY):
RETRY_COUNT=$((RETRY_COUNT+1))    # increment retry counter before sleeping
```

---

## 11. Special Variables — Most Asked in Interviews

These come in EVERY interview. Know them perfectly.

```bash
#!/bin/bash

# $0 — Name of the script itself
echo "Script name: $0"           # Output: ./system_info.sh

# $1, $2, $3 — Arguments passed to the script
# Example run: ./deploy.sh production web-server-01 v2.1.0
echo "Environment : $1"          # production
echo "Server      : $2"          # web-server-01
echo "Version     : $3"          # v2.1.0

# $# — Total COUNT of arguments
echo "Total args  : $#"          # 3

# $@ — All arguments as SEPARATE items (PREFERRED)
echo "All args    : $@"          # production web-server-01 v2.1.0

# $* — All arguments as ONE single string
echo "All args    : $*"          # production web-server-01 v2.1.0

# $$ — Process ID (PID) of the current script
echo "Script PID  : $$"          # e.g., 12345

# $! — PID of the last background process
sleep 100 &
echo "Background PID: $!"        # PID of that sleep process

# $? — Exit status of the LAST command ← MOST IMPORTANT
ls /etc/passwd
echo "Exit status : $?"          # 0 (success, file exists)

ls /this_does_not_exist
echo "Exit status : $?"          # 2 (failure, file not found)

# RULE: 0 = SUCCESS, anything else (1, 2, 127, 255...) = FAILURE
```

### $@ vs $* — The Difference (asked in senior interviews):

```bash
# Create a test script:
#!/bin/bash
print_args() {
    echo "Using \$@:"
    for arg in "$@"; do
        echo "  → $arg"
    done

    echo "Using \$*:"
    for arg in "$*"; do
        echo "  → $arg"
    done
}

print_args "hello world" "foo bar"

# Output with "$@" (each quoted argument stays separate):
#   → hello world
#   → foo bar

# Output with "$*" (all joined into one string):
#   → hello world foo bar

# RULE: Always use "$@" when passing arguments
```

---

## 12. Day 1 Resume Script — system_info.sh

This is your **first interview-showable, resume-worthy script**. Every single line is explained.

```bash
#!/bin/bash
# ============================================================
# Script Name : system_info.sh
# Description : Collects and displays complete server health
#               information. Used for quick server audits
#               after provisioning or during incidents.
# Author      : Muskan Patel
# GitHub      : github.com/muskan7860
# Created On  : 2026-06-22
# Version     : 1.0
# Usage       : ./system_info.sh
# ============================================================

# --- set -euo pipefail explanation ---
# set -e  → exit script immediately if ANY command fails
# set -u  → exit if you use a variable that was never set (catches typos)
# set -o pipefail → if a pipe like "cmd1 | cmd2" fails, catch that failure
# Together these make your script SAFE for production
set -euo pipefail

# --- COLLECT DATA SECTION ---
# We collect all data first, then display it.
# $(command) = command substitution: run command, store output in variable

# hostname → prints the machine's hostname
# On AWS EC2 it looks like: ip-10-0-1-25
HOSTNAME=$(hostname)

# date '+FORMAT' → formats current date and time
# %Y = 4-digit year, %m = month, %d = day, %H = hour, %M = minute, %S = second
CURRENT_DATE=$(date '+%Y-%m-%d %H:%M:%S')

# /etc/os-release → a file that Linux uses to identify the OS
# grep "^PRETTY_NAME" → find the line that starts with PRETTY_NAME
# cut -d'"' -f2 → split by " character, take 2nd piece (the actual OS name)
# Example output: Ubuntu 22.04.3 LTS
OS_INFO=$(grep "^PRETTY_NAME" /etc/os-release | cut -d'"' -f2)

# uname -r → prints the Linux kernel version
# Example: 5.15.0-1034-aws
KERNEL_VERSION=$(uname -r)

# uptime -p → prints how long system has been ON in readable English
# Example: up 3 days, 4 hours, 22 minutes
UPTIME=$(uptime -p)

# /proc/cpuinfo → virtual file (not real disk file) containing CPU details
# The Linux kernel writes live data here
# grep "model name" → find all lines with CPU model info
# head -1 → take only the first line (multi-core shows same info multiple times)
# cut -d':' -f2 → everything after the colon
# xargs → removes leading/trailing whitespace automatically
CPU_MODEL=$(grep "model name" /proc/cpuinfo | head -1 | cut -d':' -f2 | xargs)

# nproc → prints number of CPU cores available
# Example: 4
CPU_CORES=$(nproc)

# uptime → shows load average
# awk prints field NF-2 (3rd from last), tr removes comma
# Load average: 3 numbers = system load over last 1 min, 5 min, 15 min
# If load > number of CPU cores, system is OVERLOADED
LOAD_AVG=$(uptime | awk '{print $(NF-2), $(NF-1), $NF}' | tr -d ',')

# free -h → shows RAM in human-readable format (GB/MB)
# awk '/^Mem:/ {print $2}' → find line starting with Mem:, print column 2
# Columns: total, used, free, shared, buff/cache, available
TOTAL_RAM=$(free -h | awk '/^Mem:/ {print $2}')
USED_RAM=$(free -h | awk '/^Mem:/ {print $3}')
FREE_RAM=$(free -h | awk '/^Mem:/ {print $7}')

# df -h / → disk usage of root filesystem / in human-readable format
# awk 'NR==2 {...}' → NR=row number. Row 1 is header, row 2 is data.
# $2=total, $3=used, $4=free, $5=percentage
DISK_TOTAL=$(df -h / | awk 'NR==2 {print $2}')
DISK_USED=$(df -h / | awk 'NR==2 {print $3}')
DISK_FREE=$(df -h / | awk 'NR==2 {print $4}')
DISK_PCT=$(df -h / | awk 'NR==2 {print $5}')

# hostname -I → prints all IP addresses of this machine
# awk '{print $1}' → take the first IP only
IP_ADDRESS=$(hostname -I | awk '{print $1}')

# who | wc -l → who prints each logged-in user on one line
# wc -l counts lines = number of users currently logged in
LOGGED_USERS=$(who | wc -l)

# ss -tuln → shows all listening ports
# grep ':80 ' and ':443 ' → check if web ports are open
# We use 2>/dev/null to suppress errors if ss is not available
PORT_80=$(ss -tuln 2>/dev/null | grep ':80 ' | wc -l)
PORT_443=$(ss -tuln 2>/dev/null | grep ':443 ' | wc -l)

# --- COLOR CODES for nice output ---
GREEN='\033[0;32m'
RED='\033[0;31m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m'    # No Color — resets color back to default

# --- DISPLAY SECTION ---
# We now print everything we collected above

echo -e "${BLUE}============================================================${NC}"
echo -e "${BLUE}           SERVER HEALTH REPORT                            ${NC}"
echo -e "${BLUE}============================================================${NC}"
echo -e " Report Time       : ${YELLOW}$CURRENT_DATE${NC}"
echo    "------------------------------------------------------------"
echo -e "${GREEN} SYSTEM INFORMATION${NC}"
echo    "------------------------------------------------------------"
echo    " Hostname          : $HOSTNAME"
echo    " IP Address        : $IP_ADDRESS"
echo    " Operating System  : $OS_INFO"
echo    " Kernel Version    : $KERNEL_VERSION"
echo    " System Uptime     : $UPTIME"
echo    "------------------------------------------------------------"
echo -e "${GREEN} CPU INFORMATION${NC}"
echo    "------------------------------------------------------------"
echo    " CPU Model         : $CPU_MODEL"
echo    " CPU Cores         : $CPU_CORES"
echo    " Load Average      : $LOAD_AVG  (1min, 5min, 15min)"
echo    "------------------------------------------------------------"
echo -e "${GREEN} MEMORY INFORMATION${NC}"
echo    "------------------------------------------------------------"
echo    " Total RAM         : $TOTAL_RAM"
echo    " Used RAM          : $USED_RAM"
echo    " Available RAM     : $FREE_RAM"
echo    "------------------------------------------------------------"
echo -e "${GREEN} DISK INFORMATION  (Root Partition /)"
echo    "------------------------------------------------------------"
echo    " Total Disk        : $DISK_TOTAL"
echo    " Used Disk         : $DISK_USED"
echo    " Free Disk         : $DISK_FREE"
echo    " Disk Usage %      : $DISK_PCT"
echo    "------------------------------------------------------------"
echo -e "${GREEN} NETWORK & USERS${NC}"
echo    "------------------------------------------------------------"
echo    " Logged-in Users   : $LOGGED_USERS"

# Show port status with colored OK/CLOSED
if [ "$PORT_80" -gt 0 ]; then
    echo -e " Port 80 (HTTP)    : ${GREEN}LISTENING${NC}"
else
    echo -e " Port 80 (HTTP)    : ${RED}NOT LISTENING${NC}"
fi

if [ "$PORT_443" -gt 0 ]; then
    echo -e " Port 443 (HTTPS)  : ${GREEN}LISTENING${NC}"
else
    echo -e " Port 443 (HTTPS)  : ${YELLOW}NOT LISTENING${NC}"
fi

echo    "------------------------------------------------------------"

# $? here will be 0 because the last command (echo) succeeded
echo -e "${BLUE}============================================================${NC}"
echo    " Script completed. Exit Status: $?"
echo -e "${BLUE}============================================================${NC}"
```

---

## 13. Lab Instructions — Run on Your Machine

```bash
# STEP 1: Create your Day 1 directory
mkdir -p ~/Interview_Preparation_2026/shell_scripting/day01
cd ~/Interview_Preparation_2026/shell_scripting/day01

# STEP 2: Create the script
nano system_info.sh
# Paste the entire script from section 12 above
# Save: Ctrl+O → Enter → Ctrl+X

# STEP 3: Give execute permission
chmod +x system_info.sh

# Verify — look for 'x' in -rwxr-xr-x
ls -la system_info.sh

# STEP 4: Run the script
./system_info.sh

# STEP 5: Also try running it with bash directly (no chmod needed)
bash system_info.sh

# STEP 6: Debug mode — see every command as it runs (very useful for troubleshooting)
bash -x system_info.sh

# STEP 7: Save output to a file AND see it on screen at the same time
./system_info.sh | tee server_report.txt
# tee = T-shaped pipe. Water flows both down AND to the side.
# Output goes to SCREEN and to FILE simultaneously.

# STEP 8: Syntax check only (does not run the script)
bash -n system_info.sh

# STEP 9: Add to GitHub
git add system_info.sh
git commit -m "Day01: system_info.sh - server health report script"
git push origin main
```

---

## 14. Interview Questions — With Exact Answers

### 🟢 Beginner Level (2–3 years exp)

---

**Q1: What is a shell? How is it different from the kernel?**

> "The shell is a command-line interpreter — it is the interface between the user and the Linux kernel. When I type a command, the shell interprets it and passes the instruction to the kernel. The kernel is the core of the operating system — it manages CPU, memory, processes, and hardware. The shell does not touch hardware directly — it asks the kernel to do that. Think of the kernel as the engine and the shell as the steering wheel and pedals."

---

**Q2: What is the shebang line? Why is it needed?**

> "The shebang is the first line of a script: `#!/bin/bash`. The `#!` characters are a special signal to the Linux kernel — they tell the kernel which interpreter to use to run the rest of the file. Without the shebang, the OS will use whatever the user's current default shell is, which may be zsh or dash on different machines, causing different behavior. With `#!/bin/bash`, we guarantee the script always runs under bash regardless of which shell the user is logged into."

---

**Q3: What does `chmod +x script.sh` do? What happens without it?**

> "`chmod` stands for change mode. `+x` adds execute permission to the file. Without it, when you run `./script.sh`, Linux checks the permission bits and since execute is missing, it returns `Permission denied`. You can still run the script as `bash script.sh` without execute permission — because then you are directly calling bash and giving the file as input. But the best practice is always `chmod +x` and `./script.sh`."

---

**Q4: What is `$?` and why is it important in DevOps scripts?**

> "`$?` stores the exit status of the last executed command. `0` means success. Any non-zero number means failure — different numbers mean different types of failures. For example, `127` means command not found, `1` is a general error, `2` is misuse of shell command. In DevOps scripts, we check `$?` after critical steps — after starting a service, after a deployment command, after a health check curl. If `$?` is non-zero, we rollback or send an alert instead of continuing blindly."

---

**Q5: What is the difference between `>` and `>>`?**

> "`>` redirects output to a file and OVERWRITES it. If the file had content, it is gone. `>>` APPENDS to the file — it adds to the end without deleting existing content. In production scripts I always use `>>` for log files so we don't lose history. I use `>` only when I intentionally want a fresh file — for example, writing a fresh config file during deployment."

---

### 🟡 Mid Level (3–5 years exp — they will ask you these)

---

**Q6: What is `set -euo pipefail` and why should every production script have it?**

> "`set -e` makes the script exit immediately if any command returns a non-zero exit code. Without it, a failed command is silently ignored and the script keeps running — which in a deployment can mean we continue deploying broken code. `set -u` makes the script exit if we reference a variable that was never set — this catches typos in variable names. `set -o pipefail` makes the script catch failures in piped commands — because normally in `cmd1 | cmd2`, if `cmd1` fails, bash only looks at `cmd2`'s exit code and misses the failure. Together, these three make scripts safe for production automation."

---

**Q7: What is the difference between `$@` and `$*`?**

> "Both hold all arguments passed to the script. The difference appears when they are quoted. `"$@"` expands to individual quoted arguments — `"$1" "$2" "$3"` — which preserves spaces inside each argument. `"$*"` joins all arguments into one single string separated by the first character of `$IFS` — usually a space. For example, if I pass `"hello world"` as one argument, `"$@"` keeps it as one item but `"$*"` may split it. In production, I always use `"$@"` when passing arguments to another command."

---

**Q8: What is command substitution? What are the two syntaxes?**

> "Command substitution runs a command and captures its output as a value — either into a variable or inline in another command. The modern syntax is `$(command)` and the old syntax is backtick — `` `command` ``. I always use `$()` because it is more readable, supports nesting like `$(command1 $(command2))`, and is easier to spot in complex scripts. For example: `TODAY=$(date '+%Y-%m-%d')` runs the date command and stores its output in the TODAY variable."

---

**Q9: Explain the difference between a local variable and an environment variable.**

> "A local variable exists only in the current shell session or script. If I set `NAME="Muskan"` in a script and that script calls another script, the child script cannot see `NAME`. An environment variable is marked with `export` — like `export DB_HOST="prod-db.internal"`. Exported variables are inherited by all child processes and sub-scripts. In DevOps we use environment variables for things like database credentials, API keys, and environment names that multiple scripts or processes need to access."

---

**Q10: What happens if you have a space in variable assignment like `NAME = "Muskan"`?**

> "This is a syntax error in bash. Bash reads `NAME` as a command, `=` as its first argument, and `"Muskan"` as its second argument. It then tries to execute a command called `NAME` which does not exist, so you get a `command not found` error. The correct syntax is `NAME="Muskan"` with no spaces around the `=` sign. This is one of the most common mistakes beginners make — and I've seen it cause CI/CD pipeline failures where people assume the variable was set but it was silently not."

---

## 15. Resume Talking Points

### How to write this on your resume:

```
Shell Scripting & Automation:
• Developed system_info.sh for automated server health auditing — collects
  CPU model, core count, load averages, RAM usage, disk usage, and port
  status using bash built-ins and Linux /proc filesystem
• Reduced manual server audit time from ~15 minutes to under 5 seconds
  per server during post-provisioning checks on EC2 instances
```

### How to explain system_info.sh in the interview (practice this out loud):

> "I wrote a `system_info.sh` script that gives a complete server health snapshot in one command. It collects the hostname, OS version, kernel, CPU model and core count, load average from uptime, RAM breakdown using `free`, disk usage from `df`, current users with `who`, and whether HTTP and HTTPS ports are listening using `ss`. I use command substitution with `$(...)` throughout — for example `CPU_MODEL=$(grep "model name" /proc/cpuinfo | head -1 | cut -d':' -f2 | xargs)` — that reads from the `/proc` virtual filesystem, which gives real-time kernel data. I added color output with ANSI codes so it is easy to read at a glance. On our Atos banking platform, the ops team ran this right after provisioning new EC2 instances to confirm they were healthy before adding them to the ALB target group."

---

## 📅 Day 2 Preview

**Topic:** Conditions — `if`, `elif`, `else`, file tests, integer tests, `[[ ]]` vs `[ ]`, case statement

**Script you will build:** `disk_alert.sh` — checks disk usage on all partitions and prints colored CRITICAL/WARNING/OK status. Exactly the type of script interviewers ask you to write on the spot.

**Why it matters:** Every DevOps automation script — deployments, health checks, backups — uses conditions. This is the most tested practical topic in shell scripting interviews.

---

> 📁 Save this file to: `Interview_Preparation_2026/shell_scripting/day01/Day01_Shell_Scripting_Foundations.md`
>
> ✅ Day 01 Complete | Next: Day 02 — Conditions & disk_alert.sh
