# Files and Directories — Full Study Guide with Labs

---

## File vs Directory — easy version

A **file** = one document. Like one Word file. It holds data inside it.
A **directory** = a folder. It holds other files and folders inside it. It does NOT hold data itself.

```bash
ls -la /etc
# d rwxr-xr-x   ssh        ← "d" at the start = this is a DIRECTORY (folder)
# - rw-r--r--   hosts      ← "-" at the start = this is a FILE (document)
# l rwxrwxrwx   mtab       ← "l" at the start = this is a LINK (shortcut)
```

> **One line:** file = document. Directory = folder that holds files.

---

## Hidden Files — easy version

If a file name starts with a dot (.) → it is hidden. Linux simply does not show it unless you ask.

```bash
ls          # shows ONLY normal files
ls -a       # shows ALL files, including hidden ones (starting with .)
ls -la      # shows ALL files with full details (permissions, size, date)
```

**Common hidden files you will see:**
```
.bashrc         → your personal shell settings
.bash_history   → every command you have typed
.ssh/           → hidden folder where SSH keys are stored
.gitconfig      → your personal git settings
```

> **One line:** dot at start of name = hidden. Use `ls -a` to see them.

---

## Absolute Path vs Relative Path — easy version

**Absolute path** = full address starting from /
- Works from anywhere. Always the same.

```
/var/log/nginx/error.log    ← this always means the same place, no matter where you are
```

**Relative path** = depends on where you are standing right now
- Like saying "turn left" — only makes sense if you know where the person is standing

```
error.log     ← this ONLY works if you're already inside /var/log/nginx/
../config     ← go up one level, then into config folder
```

```bash
# Test yourself:
pwd                  # see where you currently are
cd /var/log          # absolute — works from ANYWHERE
cd nginx             # relative — only works if you're already in /var/log
```

> **One line:** starts with / = absolute, works anywhere. No / at start = relative, depends where you are.

---

## Viewing Files — every method

```bash
cat file.txt
# Dumps the WHOLE file on screen at once.
# Good for small files. Bad for huge files (floods screen).

less file.txt
# Shows file one screen at a time.
# Space = next page. b = previous page. / = search. q = quit.
# Use this for big files.

head file.txt
# Shows first 10 lines only.

head -n 20 file.txt
# Shows first 20 lines.

tail file.txt
# Shows last 10 lines only.

tail -n 50 file.txt
# Shows last 50 lines.

tail -f /var/log/app.log
# Shows last lines AND keeps watching LIVE as new lines are added.
# This is your most-used command as a DevOps engineer.

tail -F /var/log/app.log
# Same as -f, but also handles log rotation (file gets renamed and recreated).
# Always prefer -F over -f when watching logs.

wc -l file.txt
# Counts how many lines are in the file.
```

> **Easy rule:** small file → cat. Big file → less. Watch live → tail -f.

---

## Editing Files — nano and vi

### nano — the easy editor
```bash
nano file.txt
# Ctrl+O = save the file
# Ctrl+X = exit
# Ctrl+W = search for text
```

### vi / vim — the one EVERY server has (must learn this)
```bash
vi file.txt

# vi has TWO modes:
# Normal mode  → for navigation and commands (this is where you start)
# Insert mode  → for actually typing new text

# HOW TO USE:
i           # press i → now you can TYPE (enters insert mode)
Esc         # press Esc → STOP typing (back to normal mode)
:wq         # type :wq → SAVE and EXIT (write + quit)
:q!         # type :q! → EXIT without saving (use this if stuck)
dd          # delete the whole current line
/word       # search for "word" in the file
```

> **If you get stuck in vi:** press Esc, then type :q! and press Enter. This always gets you out safely.

---

## Navigation Commands — full list

```bash
pwd              # where am I right now?
ls               # what is in this folder?
ls -la           # show everything with full details
ls -lh           # show sizes in human readable format (KB, MB, GB)
ls -lt           # sort by date, newest first
cd /path         # go to this exact path (absolute)
cd folder        # go into this folder (relative — must exist where you are now)
cd ~             # go to my home folder
cd ..            # go up one folder
cd -             # go back to previous folder
```

---

## File Operations — creating, copying, moving, deleting

```bash
# Copy
cp file.txt /tmp/                 # copy file into /tmp folder
cp -r folder/ /backup/            # copy a whole folder (-r means recursive = everything inside)
cp -p file.txt /backup/           # copy AND keep original permissions and dates

# Move / Rename
mv oldname.txt newname.txt        # rename a file
mv file.txt /other/folder/        # move a file to another folder

# Create
touch newfile.txt                  # create an empty file (or update its date if it exists)
mkdir newfolder                    # create a new folder
mkdir -p /opt/app/logs/2024        # create nested folders all at once (no error if already exists)

# Delete
rm file.txt                        # delete a file
rm -rf foldername/                 # delete a whole folder and everything inside it
# WARNING: rm -rf cannot be undone! Be very careful.

# View
stat file.txt                      # show full details: permissions, size, dates, inode number
file filename                      # tell me what TYPE of file this is
```

---

## Searching for Files and Content

### find — searching for FILES
```bash
find /etc -name "*.conf"
# Find all files ending in .conf inside /etc

find /var/log -name "*.log" -mtime -1
# Find log files modified in the last 1 day

find /home -type f -size +10M
# Find files bigger than 10MB

find /tmp -mtime +7 -type f -delete
# Delete files in /tmp that are older than 7 days

find / -type f -size +500M 2>/dev/null
# Find very large files anywhere on the system
# 2>/dev/null = hide "permission denied" errors
```

### grep — searching for TEXT inside files
```bash
grep "ERROR" app.log
# Find every line containing the word ERROR

grep -i "error" app.log
# Same but case-insensitive (finds ERROR, Error, error)

grep -r "database" /etc/
# Search the word "database" inside EVERY file in /etc

grep -n "ERROR" app.log
# Show the ERROR lines AND their line numbers

grep -v "DEBUG" app.log
# Show every line EXCEPT lines containing DEBUG

grep -c "ERROR" app.log
# Count how many lines have ERROR (just shows a number)

grep -A 3 -B 3 "ERROR" app.log
# Show 3 lines AFTER and 3 lines BEFORE each ERROR line (gives context)

grep -l "ERROR" *.log
# Show only the FILENAMES that contain ERROR (not the lines themselves)
```

> **Easy rule:** find = looking for a FILE. grep = looking for TEXT inside a file.

---

## Text Processing — awk, sed, sort, uniq, cut

### awk — work with columns
```bash
awk '{print $1}' access.log
# Print only the first word/column of each line

awk '{print $1, $3}' access.log
# Print columns 1 and 3

awk -F: '{print $1}' /etc/passwd
# Use : as the separator, print the first column (usernames)

awk '/ERROR/ {print $0}' app.log
# Print full lines that contain the word ERROR
```

### sed — find and replace text
```bash
sed 's/old/new/g' file.txt
# Replace every occurrence of "old" with "new" (just shows result, doesn't save)

sed -i 's/old/new/g' file.txt
# Replace AND save the change back into the file (-i = in-place)

sed -n '10,20p' file.txt
# Show only lines 10 to 20

sed '/^#/d' config.conf
# Delete all lines that start with # (comment lines)
```

### sort, uniq, cut, wc
```bash
sort file.txt
# Sort lines A to Z

sort -n numbers.txt
# Sort by number (1, 2, 3...) not by letter

sort -rn file.txt
# Sort by number, reversed (biggest first)

uniq -c file.txt
# Count how many times each line repeats
# (NOTE: sort first, then uniq — uniq only catches lines that are next to each other)

cut -d: -f1 /etc/passwd
# Use : as separator, get only the first column (usernames)

wc -l file.txt
# Count lines

wc -w file.txt
# Count words
```

---

## Pipes and Redirection — easy version

**Pipe (|)** = take output of one command, feed it into the next command

**Redirect (>)** = take output and save it into a file

```bash
command > file.txt
# Save output into file. REPLACES old content.

command >> file.txt
# Save output into file. ADDS to end, does not replace.

command 2> error.txt
# Save only ERROR messages into the file

command 2>&1
# Send error messages to the same place as normal output

command &> all.txt
# Save BOTH normal output and error messages into file

command | tee file.txt
# Show output on screen AND save to file at the same time
```

**Pipe examples:**
```bash
cat access.log | grep "404" | wc -l
# Step 1: show the whole file
# Step 2: keep only lines with 404
# Step 3: count how many lines that is

du -sh /var/* | sort -h
# Show size of each folder inside /var, sorted by size

ps aux | grep nginx | grep -v grep
# Show all processes, keep nginx ones, remove the grep line itself
```

---

## Dummy Log File for Lab Practice

Save this as `~/linux-lab/app.log` to use in the labs below:

```
2024-01-15 09:00:01 INFO  Application started successfully
2024-01-15 09:00:05 INFO  Database connection established
2024-01-15 09:01:10 DEBUG Checking config file
2024-01-15 09:02:00 INFO  User login: alice
2024-01-15 09:02:30 ERROR Failed to read config file: permission denied
2024-01-15 09:03:00 INFO  User login: bob
2024-01-15 09:03:45 ERROR Database connection timeout after 30s
2024-01-15 09:04:00 DEBUG Memory usage: 45%
2024-01-15 09:04:30 WARN  Disk usage is at 80%
2024-01-15 09:05:00 ERROR Failed to write to /var/log/app: no space left
2024-01-15 09:05:30 INFO  User login: charlie
2024-01-15 09:06:00 ERROR Database connection timeout after 30s
2024-01-15 09:06:30 INFO  Backup started
2024-01-15 09:07:00 DEBUG Cache cleared
2024-01-15 09:07:30 ERROR Failed to read config file: permission denied
2024-01-15 09:08:00 INFO  Backup completed
2024-01-15 09:08:30 WARN  Memory usage is at 85%
2024-01-15 09:09:00 ERROR Database connection timeout after 30s
2024-01-15 09:09:30 INFO  User logout: alice
2024-01-15 09:10:00 INFO  User logout: bob
2024-01-15 09:10:30 FATAL Application crashed: out of memory
```

---

## Lab 2 — Files and Directories Practice

### Setup
```bash
mkdir -p ~/linux-lab
cd ~/linux-lab

# Create the dummy log file
cat > app.log << 'EOF'
2024-01-15 09:00:01 INFO  Application started successfully
2024-01-15 09:00:05 INFO  Database connection established
2024-01-15 09:01:10 DEBUG Checking config file
2024-01-15 09:02:00 INFO  User login: alice
2024-01-15 09:02:30 ERROR Failed to read config file: permission denied
2024-01-15 09:03:00 INFO  User login: bob
2024-01-15 09:03:45 ERROR Database connection timeout after 30s
2024-01-15 09:04:00 DEBUG Memory usage: 45%
2024-01-15 09:04:30 WARN  Disk usage is at 80%
2024-01-15 09:05:00 ERROR Failed to write to /var/log/app: no space left
2024-01-15 09:05:30 INFO  User login: charlie
2024-01-15 09:06:00 ERROR Database connection timeout after 30s
2024-01-15 09:06:30 INFO  Backup started
2024-01-15 09:07:00 DEBUG Cache cleared
2024-01-15 09:07:30 ERROR Failed to read config file: permission denied
2024-01-15 09:08:00 INFO  Backup completed
2024-01-15 09:08:30 WARN  Memory usage is at 85%
2024-01-15 09:09:00 ERROR Database connection timeout after 30s
2024-01-15 09:09:30 INFO  User logout: alice
2024-01-15 09:10:00 INFO  User logout: bob
2024-01-15 09:10:30 FATAL Application crashed: out of memory
EOF

echo "Lab files ready"
```

---

### Lab 2.1 — File viewing practice
```bash
# Look at the whole file
cat app.log

# Look at first 5 lines only
head -n 5 app.log

# Look at last 5 lines only
tail -n 5 app.log

# Count how many lines are in the file
wc -l app.log
# Expected answer: 21
```

---

### Lab 2.2 — grep practice (searching text)
```bash
# Find all ERROR lines
grep "ERROR" app.log

# Count how many ERROR lines there are
grep -c "ERROR" app.log
# Expected answer: 6

# Find all lines that are NOT DEBUG
grep -v "DEBUG" app.log

# Find ERROR lines AND show 2 lines after each one (for context)
grep -A 2 "ERROR" app.log

# Find errors about database timeout specifically
grep "Database connection timeout" app.log

# Case-insensitive search for "warn"
grep -i "warn" app.log
```

---

### Lab 2.3 — awk practice (columns)
```bash
# Show only the TIME column (column 2) of each line
awk '{print $2}' app.log

# Show the DATE and ERROR TYPE columns (columns 1 and 3)
awk '{print $1, $3}' app.log

# Show only ERROR lines using awk
awk '/ERROR/ {print $0}' app.log

# Show only FATAL lines
awk '/FATAL/ {print $0}' app.log
```

---

### Lab 2.4 — sort and count (most useful combo)
```bash
# Find all unique log levels and how many times each appears
awk '{print $3}' app.log | sort | uniq -c | sort -rn
# Expected output (something like):
# 8 INFO
# 6 ERROR
# 3 DEBUG
# 2 WARN
# 1 FATAL

# Find which error message appears the most times
grep "ERROR" app.log | awk '{print $4, $5, $6, $7, $8}' | sort | uniq -c | sort -rn
```

---

### Lab 2.5 — find practice
```bash
# Create some test files to practice find
mkdir -p ~/linux-lab/testfind
cd ~/linux-lab/testfind
touch file1.txt file2.log file3.conf bigfile.txt

# Find all .txt files in this folder
find ~/linux-lab/testfind -name "*.txt"

# Find all files (not folders) in this folder
find ~/linux-lab/testfind -type f

# Find files modified in the last 1 day
find ~/linux-lab/testfind -mtime -1 -type f
```

---

### Lab 2.6 — pipes practice
```bash
cd ~/linux-lab

# How many ERROR lines are there?
cat app.log | grep "ERROR" | wc -l

# Show all unique log levels
cat app.log | awk '{print $3}' | sort | uniq

# Save all ERROR lines into a separate file
grep "ERROR" app.log > errors-only.log
cat errors-only.log

# Save all WARN lines and APPEND them to the same file
grep "WARN" app.log >> errors-only.log
cat errors-only.log
# Now you should see both ERROR and WARN lines
```

---

### Lab 2.7 — sed practice (find and replace)
```bash
cd ~/linux-lab

# Make a copy to practice on (so we don't break the original)
cp app.log app-practice.log

# Replace all "ERROR" with "PROBLEM" (just show on screen, don't save)
sed 's/ERROR/PROBLEM/g' app-practice.log | head -10

# Replace AND save to file
sed -i 's/DEBUG/TRACE/g' app-practice.log

# Confirm it changed
grep "TRACE" app-practice.log
```

---

## Interview Questions with Easy Answers

### Q1: What is the difference between absolute and relative path?
**Say this:**
"Absolute path starts with / and works from anywhere — it always points to the same location no matter where I am. Relative path depends on where I am currently standing. For example /var/log/nginx/error.log is absolute. But if I'm already inside /var/log/nginx, I can just say error.log — that's relative."

---

### Q2: How do you watch a log file live?
**Say this:**
"I use `tail -f filename`. This shows the last few lines and keeps updating live as new lines come in. I prefer `tail -F` because that also handles log rotation — when the log file gets renamed and a new one is created, -F follows the new file automatically."

---

### Q3: How do you search for an error in a big log file?
**Say this:**
"I use grep. For example `grep 'ERROR' app.log` shows me every line containing ERROR. If I want to count them: `grep -c 'ERROR' app.log`. If I want to see context around each error: `grep -A 3 'ERROR' app.log` — that shows 3 lines after each match so I can see what happened next."

---

### Q4: What is the difference between find and grep?
**Say this:**
"find searches for FILES by name, size, or date. grep searches for TEXT inside a file's content. They do different jobs. I use find to locate a file. I use grep to look inside a file."

---

### Q5: What is a pipe and when do you use it?
**Say this:**
"A pipe takes the output of one command and feeds it as input to the next. I use it to chain commands together. For example: `grep 'ERROR' app.log | wc -l` — first grep finds error lines, then wc counts them. Pipes are how I build quick one-line analysis tools on the command line."

---

### Scenario Q1: You have a 2GB log file. Find the top 5 most frequent error messages.
**Say this:**
"I would NOT use cat — that dumps 2GB on screen and freezes everything. Instead:
`grep 'ERROR' app.log | sort | uniq -c | sort -rn | head -5`
Step by step: grep keeps only error lines. sort puts matching lines next to each other. uniq -c counts how many times each unique line appears. sort -rn puts the highest count first. head -5 shows only the top 5."

---

### Scenario Q2: A deployment script works when you run it manually but fails in Jenkins. What do you check?
**Say this:**
"My first suspicion is a path problem or environment problem. When I run a script manually, I'm in a specific folder with my own PATH set up. Jenkins runs in a different folder with a different environment. I would add `pwd` and `echo $PATH` at the start of the script and check what Jenkins shows for those — often the script uses a relative path that only works from my folder, or calls a command that is not in Jenkins' PATH. The fix is to use absolute paths everywhere in scripts that run unattended."
