# Users, Groups, and Permissions — Full Study Guide with Labs

---

## Why Permissions Exist — easy picture

Think of an apartment building:
- **You (owner)** have full key access to your own flat
- **Your flatmates (group)** can access shared spaces — kitchen, living room
- **Random strangers (others)** have NO access to anything

Linux permissions work exactly the same way — for every single file and folder on the system.

---

## The 3 Powers — read, write, execute

```
r = read     → "I can open it and look inside"
w = write    → "I can change it"
x = execute  → "I can run it as a program" (or enter a folder)
```

**Numbers:**
```
r = 4
w = 2
x = 1

Add them up for what you want:
read + write         = 4 + 2 = 6
read + write + run   = 4 + 2 + 1 = 7
read only            = 4
nothing              = 0
```

> **One line:** r=4, w=2, x=1. Add them up.

---

## The 3 Groups of People

```
Owner   → the one person who created or owns the file
Group   → a small team sharing access to this file
Others  → literally everyone else on the system
```

Every file has ONE set of permissions for each of these three. So every file has 3 numbers.

---

## Reading the Permission Letters — easy version

```bash
ls -la script.sh
-rwxr-xr-- 1 ubuntu devops 4096 May 27 script.sh

Break it down:
- rwx r-x r--
↑  ↑   ↑   ↑
│  │   │   └── OTHERS:  r-- = can only READ (4)
│  │   └────── GROUP:   r-x = can READ and RUN (5)
│  └────────── OWNER:   rwx = can READ, WRITE, RUN (7)
└──────────── type: - = file, d = folder, l = link
```

**Reading it as numbers:** rwxr-xr-- = 7-5-4 = 754

---

## chmod — changing permissions

```bash
chmod 755 script.sh
# Owner: 7 = rwx (full power)
# Group: 5 = r-x (read + run, no write)
# Others: 5 = r-x (read + run, no write)

chmod 644 config.conf
# Owner: 6 = rw- (read + write, no run)
# Group: 4 = r-- (read only)
# Others: 4 = r-- (read only)

chmod 600 private.key
# Owner: 6 = rw- (read + write)
# Group: 0 = --- (nothing)
# Others: 0 = --- (nothing)

chmod 400 secret.pem
# Owner: 4 = r-- (read only)
# Group: 0 = --- (nothing)
# Others: 0 = --- (nothing)

chmod 777 public.txt
# Everyone gets full power — AVOID this in production

chmod -R 755 /opt/app/
# -R means recursive = apply to ALL files and folders inside
```

**Symbolic way (changing just one thing):**
```bash
chmod u+x script.sh    # add execute for the owner (u = user/owner)
chmod g-w file.txt     # remove write from group
chmod o= file.txt      # remove ALL permissions from others
chmod a+r file.txt     # add read for everyone (a = all)
```

---

## Common Permission Recipes — what you set in production

```bash
# Web server files (nginx serving website):
chown -R www-data:www-data /var/www/html/
chmod -R 755 /var/www/html/        # folders: enter + read
chmod -R 644 /var/www/html/*.html  # files: owner writes, others read

# SSH key files (EXACT values required — SSH refuses if wrong):
chmod 700 ~/.ssh/                  # only YOU can enter this folder
chmod 600 ~/.ssh/authorized_keys   # only YOU can read/write
chmod 600 ~/.ssh/id_rsa            # private key — yours only
chmod 644 ~/.ssh/id_rsa.pub        # public key — others can read, that is fine

# App deployment files:
chown -R appuser:appgroup /opt/myapp/
chmod 750 /opt/myapp/              # owner full, group read+enter, others nothing
chmod 640 /opt/myapp/config.env    # owner read+write, group read, others nothing
```

---

## chown — changing who owns a file

```bash
chown ubuntu file.txt
# Make ubuntu the new owner

chown ubuntu:devops file.txt
# Make ubuntu the owner AND devops the group

chgrp devops file.txt
# Change ONLY the group

chown -R www-data:www-data /var/www/html/
# Recursive — change owner AND group for EVERYTHING inside that folder
```

**Real story:**
You copy website files using sudo — now they are owned by root. Nginx runs as www-data. www-data cannot read root's files. Website shows 403 Forbidden. Fix:
```bash
chown -R www-data:www-data /var/www/html/
```
Always check ownership first when you see permission errors — not just the numbers.

---

## SUID — easy version (go slow, read twice)

Normally, when YOU run a program, it acts as YOU.

SUID is a special switch. When ON, the program acts as the FILE'S OWNER instead — not as you.

**Real example:**
- The `passwd` command is owned by root
- SUID is switched ON for passwd
- So when a normal user runs passwd to change their password — it temporarily acts as root
- This is needed because only root can write to /etc/shadow (where passwords are stored)

```bash
ls -la /usr/bin/passwd
-rwsr-xr-x  root  passwd
    ↑
    's' here instead of 'x' = SUID is ON

# Setting SUID:
chmod u+s binary           # set SUID symbolically
chmod 4755 /usr/bin/myapp  # set SUID with number (4 prefix = SUID)
```

> **One line:** SUID = program runs as the file's owner, not the person running it.

**Security risk:** If a SUID program owned by root has a bug, an attacker can exploit that bug to get root access — even if they started as a normal user.

---

## SGID — easy version

For FOLDERS (not single files).

Normally, when you create a file inside a folder, the file takes YOUR group.

With SGID ON, new files automatically take the FOLDER'S group instead — no matter who creates them.

**Why useful:** shared team folder — everyone's files automatically end up in the same group.

```bash
chmod g+s /var/shared/        # set SGID symbolically
chmod 2755 /var/shared/       # set SGID with number (2 prefix = SGID)

ls -la /var/shared
drwxr-sr-x  root  devteam  /var/shared
        ↑
        's' in group position = SGID is ON
```

> **One line:** SGID on a folder = new files inside get the folder's group automatically.

---

## Sticky Bit — easy version

/tmp is shared — everyone can put files there.

Without sticky bit, anyone could delete anyone else's files in /tmp.

Sticky bit fixes that: you can only delete YOUR OWN files, even in a shared folder.

```bash
chmod +t /tmp                # set sticky bit symbolically
chmod 1777 /tmp              # set sticky bit with number (1 prefix = sticky bit)

ls -la /
drwxrwxrwt  root  /tmp
         ↑
         't' at the very end = sticky bit is ON
```

> **One line:** sticky bit = shared folder, but each person can only delete their own files.

---

## Memorize this table (comes up in every interview)

```
4 = SUID       → program runs as file's owner
2 = SGID       → new files in folder inherit folder's group
1 = sticky bit → only owner can delete their own files in shared folder

4755 = SUID + rwxr-xr-x
2755 = SGID + rwxr-xr-x
1777 = sticky bit + rwxrwxrwx  (this is /tmp)
3770 = SGID + sticky bit + rwxrws--- (used for secure shared team folders)
```

---

## User Management — creating and managing users

```bash
useradd username
# Create a user — bare minimum, may not create home folder on all distros

useradd -m -s /bin/bash username
# Create user WITH home folder (-m) and bash as their shell (-s /bin/bash)

useradd -m -G sudo,docker username
# Create user AND add them to sudo and docker groups right away

usermod -aG docker ubuntu
# ADD ubuntu to the docker group — keep all their existing groups
# WARNING: always use -aG, never just -G alone (see danger note below)

usermod -s /bin/bash username
# Change their shell

usermod -L username
# LOCK the account — they cannot log in anymore

usermod -U username
# UNLOCK the account

userdel username
# Delete the user — keeps their home folder

userdel -r username
# Delete the user AND their home folder

passwd username
# Set or change their password

passwd -e username
# Force them to change their password next time they log in
```

**Checking users:**
```bash
id username          # show their UID, GID, and all groups
whoami               # who am I right now?
cat /etc/passwd      # list all users: username:x:UID:GID:comment:home:shell
cat /etc/shadow      # hashed passwords — only root can read this
```

> **DANGER:** `usermod -G docker ubuntu` (without -a) REMOVES all of ubuntu's existing groups and only leaves docker. Always use `-aG`. The `a` means APPEND — add to existing, don't replace.

---

## Group Management

```bash
groupadd developers         # create a new group
groupdel developers         # delete a group
groups username             # show which groups a user belongs to
cat /etc/group              # list all groups: groupname:x:GID:member1,member2
```

**Setting up a shared team folder (practical pattern):**
```bash
groupadd devteam
usermod -aG devteam alice
usermod -aG devteam bob
mkdir /opt/shared-project
chgrp -R devteam /opt/shared-project/
chmod -R 770 /opt/shared-project/
# 770 = owner full, group full, others nothing
```

---

## sudo — temporary root power

```bash
sudo command
# Run ONE command as root — just this once

sudo -u postgres psql
# Run as a DIFFERENT user (not root)

sudo -i
# Switch to a full root shell

sudo -l
# List what commands YOU are allowed to sudo

visudo
# The SAFE way to edit who can sudo
# Validates the file before saving — prevents mistakes that lock you out
```

**How sudoers file works:**
```bash
# Format: WHO  WHERE=(AS_WHO)  COMMAND
myuser  ALL=(ALL) NOPASSWD:ALL
# myuser can run anything as root, no password needed

# Safer — give access to ONE command only:
deploy  ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart myapp

# Put this in /etc/sudoers.d/ (cleaner than editing main file):
echo "deploy ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart myapp" > /etc/sudoers.d/deploy
chmod 440 /etc/sudoers.d/deploy
```

---

## Security Auditing — important for banking environments

```bash
find / -perm /4000 -type f 2>/dev/null
# Find ALL SUID files on the system
# In banking environments, this is run regularly as a compliance check

find / -perm /2000 -type f 2>/dev/null
# Find ALL SGID files

find / -perm -002 -type f 2>/dev/null
# Find world-writable files — security risk

awk -F: '$3 == 0' /etc/passwd
# Find any user account with UID 0 (same power as root — dangerous if unexpected)

grep "Failed password" /var/log/auth.log
# Check how many failed SSH login attempts happened
```

---

## Dummy Files for Lab Practice

```bash
# These get created during lab setup below
# permissions-lab/
#   secret.key       (should be 600 — owner only)
#   website.html     (should be 644 — owner write, all read)
#   deploy.sh        (should be 755 — executable script)
#   shared-folder/   (should be 770 with SGID)
```

---

## Lab 3 — Users, Groups, Permissions Practice

### Setup
```bash
mkdir -p ~/linux-lab/permissions-lab
cd ~/linux-lab/permissions-lab

# Create test files
touch secret.key website.html deploy.sh
mkdir -p shared-folder

# Write some content into them
echo "PRIVATE_KEY=abc123xyz" > secret.key
echo "<html><body>Hello World</body></html>" > website.html
echo "#!/bin/bash
echo 'Deploying application...'
echo 'Done'" > deploy.sh

echo "Lab files ready"
ls -la
```

---

### Lab 3.1 — Reading permissions
```bash
cd ~/linux-lab/permissions-lab
ls -la

# Look at the first column of each line
# What type is each? (-, d, l)
# What are the current permissions?
# Who is the owner?
```

---

### Lab 3.2 — Setting correct permissions
```bash
cd ~/linux-lab/permissions-lab

# secret.key should only be readable by owner
chmod 600 secret.key
ls -la secret.key
# Should show: -rw------- (only owner can read and write, nobody else)

# website.html should be readable by everyone but only owner can write
chmod 644 website.html
ls -la website.html
# Should show: -rw-r--r--

# deploy.sh needs to be executable
chmod 755 deploy.sh
ls -la deploy.sh
# Should show: -rwxr-xr-x

# Now actually run it
./deploy.sh
```

---

### Lab 3.3 — chmod symbolic way
```bash
cd ~/linux-lab/permissions-lab

# Make a test file
touch testfile.txt
ls -la testfile.txt

# Add execute permission ONLY for the owner
chmod u+x testfile.txt
ls -la testfile.txt

# Remove write from group
chmod g-w testfile.txt
ls -la testfile.txt

# Remove ALL permissions from others
chmod o= testfile.txt
ls -la testfile.txt

# Add read for EVERYONE
chmod a+r testfile.txt
ls -la testfile.txt
```

---

### Lab 3.4 — chown practice
```bash
cd ~/linux-lab/permissions-lab

# See current owner
ls -la secret.key

# If you have sudo access, try changing the owner
# (replace "youruser" with your actual username)
sudo chown root secret.key
ls -la secret.key
# Now root owns it

# Change it back to yourself
sudo chown $USER secret.key
ls -la secret.key
```

---

### Lab 3.5 — SUID in the real world (view only, do not change system files)
```bash
# Look at the passwd command — see the SUID bit
ls -la /usr/bin/passwd
# You should see 's' in the owner execute position: -rwsr-xr-x

# Find all SUID files on the system (this is what you do in a security audit)
find /usr -perm /4000 -type f 2>/dev/null
# This shows you every SUID binary under /usr
```

---

### Lab 3.6 — Sticky bit (look at /tmp)
```bash
# Look at /tmp
ls -la /
# Find /tmp in the list
# You should see the 't' at the very end: drwxrwxrwt

# Create a file in /tmp as yourself
touch /tmp/myfile-$USER.txt
ls -la /tmp/myfile-$USER.txt

# Try to delete another user's file in /tmp (if one exists)
# It should say "Operation not permitted" — that is the sticky bit working
```

---

### Lab 3.7 — sudo practice
```bash
# See what sudo commands you are allowed to run
sudo -l

# Run a command as root using sudo
sudo whoami
# Should print: root

# View a file that only root can read
sudo cat /etc/shadow | head -5

# Try to read /etc/shadow WITHOUT sudo — it should fail
cat /etc/shadow
# Should say: Permission denied — this proves permissions are working
```

---

### Lab 3.8 — User info
```bash
# Who am I?
whoami

# See my UID, GID, and all my groups
id

# See all users on the system
cat /etc/passwd | awk -F: '{print $1, $3, $7}' | column -t
# Shows: username, UID, shell

# Check which users can log in (have a real shell, not /sbin/nologin)
grep -v "nologin\|false" /etc/passwd | awk -F: '{print $1}'
```

---

### Lab 3.9 — Shared folder with SGID (team folder simulation)
```bash
cd ~/linux-lab/permissions-lab

# Set SGID on shared-folder
chmod g+s shared-folder
ls -la
# Should show 's' in group execute position for shared-folder

# Set permissions so group has full access
chmod 770 shared-folder

# Create a file inside and check its group
touch shared-folder/teamfile.txt
ls -la shared-folder/
# The file should inherit the group of shared-folder because of SGID
```

---

### Lab 3.10 — Security audit simulation
```bash
# Find all SUID files (like a security audit)
find /usr/bin -perm /4000 -type f 2>/dev/null

# Find world-writable files in /tmp (these are normal in /tmp)
find /tmp -perm -002 -type f 2>/dev/null

# Check your own permissions file to understand structure
cat /etc/group | grep $USER
# Shows which groups you belong to
```

---

## Interview Questions with Easy Answers

### Q1: What does chmod 755 mean?
**Say this:**
"7 for the owner means full power — read, write, and run. 5 for the group means read and run only, no writing. 5 for others means the same — read and run, no writing. I use 755 for scripts and folders that need to be accessible but should only be written to by the owner."

---

### Q2: What is SUID? Give a real example.
**Say this:**
"SUID makes a program run as the file's owner, not as the person running it. Real example: the passwd command is owned by root and has SUID on. When a normal user runs passwd to change their password, it temporarily acts as root — because only root is allowed to write to /etc/shadow where passwords are stored. The security risk is: if that SUID program has a bug, an attacker can exploit the bug and get root access."

---

### Q3: What is the sticky bit?
**Say this:**
"Sticky bit means in a shared folder where everyone can write, you can only delete your OWN files — not anyone else's. The classic example is /tmp — everyone can create files there, but you cannot delete another user's file. You can see the sticky bit as a 't' at the end of the permissions when you run ls -la on /tmp."

---

### Q4: What is the difference between chown and chmod?
**Say this:**
"chown changes WHO owns the file. chmod changes WHAT that person is allowed to do with it. They solve different problems. chown is about identity — whose file is it. chmod is about capability — what can they do to it."

---

### Q5: How do you add a user to a group without losing their existing groups?
**Say this:**
"I use `usermod -aG groupname username`. The important part is the `-a` which means APPEND — it adds the new group while keeping all the groups they already had. If I forget the -a and just use -G, it replaces all their existing groups with only the new one — which can break their access to other things like sudo."

---

### Q6: A user added themselves to the docker group but still gets permission denied. Why?
**Say this:**
"Group changes don't take effect in an already-running session. The shell caches the group list at login time. The user needs to log out and log back in — or run `newgrp docker` to start a new shell with updated groups, without fully logging out. I would also verify the group was actually added correctly by running `groups username`."

---

### Q7: What is the difference between chmod 750 and chmod 755?
**Say this:**
"750 means others — people outside the owner and group — have zero access. They cannot even look inside the folder. 755 means others can read and enter the folder. I use 750 for deployment folders that might have config files with secrets — I don't want anyone outside the app's group to even see what is inside. I use 755 for public web content folders where the web server user needs to read the files."

---

### Scenario Q1: You need to set up a shared folder where 5 developers can all create and edit files, but no one can delete someone else's files.
**Say this:**
"Step 1: create a group for the team — `groupadd devteam`. Step 2: add all 5 developers to it — `usermod -aG devteam alice` (repeat for each person). Step 3: create the folder and set the group — `mkdir /opt/shared && chgrp devteam /opt/shared`. Step 4: set SGID so all new files automatically get the devteam group — `chmod g+s /opt/shared`. Step 5: set permissions 2770 — `chmod 2770 /opt/shared` — owner and group have full access, others have nothing. Step 6: add sticky bit to prevent deleting each other's files — `chmod 3770 /opt/shared`. Now everyone in devteam can create and edit, but each person can only delete their own files."

---

### Scenario Q2: During a security audit you need to find unauthorized SUID binaries.
**Say this:**
"First I run `find / -perm /4000 -type f 2>/dev/null` and save the output to a file. Then I compare it against the list from the last known good audit — or against what the package manager says should be there. Any SUID file that is NOT from a trusted package, especially in unusual locations like /tmp or hidden folders, is suspicious. I would immediately remove the SUID bit from it with `chmod u-s` and escalate to the security team. In our banking environment, this is a compliance obligation — unauthorized SUID files must be reported, not just quietly removed."
