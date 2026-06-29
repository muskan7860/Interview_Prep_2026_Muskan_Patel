# Why Containers Exit — PID 1, `-it` Flags, REPL vs Shell

> **Who is this for?**
> Someone completely new to Docker.
> Your exact terminal output is used throughout.
> Every concept built from zero — no assumptions.

---

## 📌 Table of Contents

1. [The One Rule That Explains Everything](#1-the-one-rule-that-explains-everything)
2. [What is a Process? What is PID 1?](#2-what-is-a-process-what-is-pid-1)
3. [Why `docker run node:20-alpine` Exited Immediately](#3-why-docker-run-node20-alpine-exited-immediately)
4. [What is `-it`? Explained From Zero](#4-what-is--it-explained-from-zero)
5. [Why `-it` Made Node Stay Alive](#5-why--it-made-node-stay-alive)
6. [Why SonarQube Is Still Running After 8 Weeks](#6-why-sonarqube-is-still-running-after-8-weeks)
7. [Your Terminal Output — Every Line Explained](#7-your-terminal-output--every-line-explained)
8. [Node REPL vs Linux Shell — Why Your `echo` Command Failed](#8-node-repl-vs-linux-shell--why-your-echo-command-failed)
9. [How Docker Decides What to Run](#9-how-docker-decides-what-to-run)
10. [Practical Labs](#10-practical-labs)
11. [Why This Matters in Interviews](#11-why-this-matters-in-interviews)
12. [Quick Revision Cheatsheet](#12-quick-revision-cheatsheet)
13. [Further Reading](#13-further-reading)

---

## 1. The One Rule That Explains Everything

Before anything else — learn this one sentence.
If you understand ONLY this one thing, everything else in this file will make sense.

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│   A container lives ONLY as long as its main process is running.   │
│                                                                     │
│   Main process stops  →  Container stops.  Immediately.            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

That is the entire secret.

Every confusing Docker behaviour you will ever see — containers exiting, containers running forever, containers crashing — is explained by this one rule.

Let us now understand what a "main process" is.

---

## 2. What is a Process? What is PID 1?

### What is a Process? (From Zero)

A **process** is simply a **program that is currently running**.

When you open Google Chrome — that is a process.
When you open a text editor — that is a process.
When your antivirus runs a scan — that is a process.

A process:
- Uses CPU to do work
- Uses RAM to store data
- Has a start time
- Has an end time (when it finishes its work)
- Gets a number to identify it — called a **PID** (Process ID)

**PID** = Process ID = a unique number assigned to every running program.

```
Example processes on your Ubuntu machine right now:
PID 1      → systemd  (Ubuntu's main system manager)
PID 1234   → dockerd  (Docker daemon)
PID 1567   → Firefox  (your browser)
PID 1890   → Terminal (your terminal window)
```

Every process gets a number. The OS uses these numbers to track them.

---

### What is PID 1? Why is It Special?

When Linux starts, the VERY FIRST process it runs gets PID number **1**.
This process is special because:

1. It is the **parent of all other processes**
   Everything else on the system eventually starts from PID 1
2. It is the **last to die**
   When PID 1 dies, Linux considers the system done

On your Ubuntu machine, PID 1 is `systemd` — Ubuntu's main manager.

**Inside a Docker container, PID 1 is whatever command you told Docker to run.**

```bash
docker run nginx
# Inside this container:
# PID 1 = nginx   ← nginx IS the container's entire world

docker run python:3.12 python server.py
# Inside this container:
# PID 1 = python  ← python IS the container's entire world
```

**The critical rule:**

```
When PID 1 inside a container exits → the container exits.

There is no exception to this rule.
```

This is because Docker watches PID 1.
The moment PID 1 stops, Docker considers the container's work done.
It shuts everything down.

---

### Simple Story to Remember This

Imagine a **small office building** that was built for ONE employee.

```
The building = the container
The employee = PID 1 (the main process)
```

Rules of this office:
- The building opens when the employee arrives (container starts)
- The building stays open as long as the employee is working
- The moment the employee leaves, the building locks and shuts down
- It does not matter if the employee finished work or quit — building closes either way

```
Employee arrives at work     →  Building opens     (Container starts)
Employee works               →  Building stays open (Container runs)
Employee finishes and leaves →  Building closes     (Container exits)
Employee quits with no work  →  Building closes     (Container exits)
```

This is exactly what Docker does with PID 1.

---

## 3. Why `docker run node:20-alpine` Exited Immediately

### Your Terminal Output

```
muskan@muskan-ThinkPad-L14-Gen-1:~/Lab_Practice/Docker$ docker ps -a

CONTAINER ID   IMAGE           COMMAND                  CREATED           STATUS
f9da31a71e3a   node:20-alpine  "docker-entrypoint.s…"  9 seconds ago     Exited (0) 8 seconds ago
```

The container ran for 1 second and then `Exited`.
`Exited (0)` means: the process finished normally. Exit code 0 = success, no error.

But why did it exit so fast?

### Let Us Think Like Node.js

When you run `docker run node:20-alpine`, Docker starts the container and runs:

```
docker-entrypoint.sh node
```

This starts the `node` program.

`node` is the Node.js runtime — a program that runs JavaScript code.

Now imagine YOU are the Node.js program.
You just started.
You look around and ask yourself these questions:

```
Question 1: Was I given a JavaScript file to run?
            For example: node server.js  or  node app.js
            → NO. Nobody gave me any file.

Question 2: Is there a keyboard connected so a human can type JavaScript to me?
            → NO. There is no keyboard attached.

Question 3: Is there a terminal screen where I can show output and wait?
            → NO. There is no terminal.

Conclusion: I have nothing to do and nobody to talk to.
            I will exit.
```

And so `node` exits.
Node exits = PID 1 exits = container exits.

### The Timeline — Second by Second

```
Time 0.000s  → Docker starts the container
Time 0.001s  → docker-entrypoint.sh starts (PID 1)
Time 0.002s  → docker-entrypoint.sh calls the node program
Time 0.003s  → node starts, looks for work
Time 0.004s  → node finds: no file, no keyboard, no terminal
Time 0.005s  → node decides: nothing to do, exit
Time 0.006s  → node exits with code 0 (success)
Time 0.007s  → docker-entrypoint.sh exits (its child finished)
Time 0.008s  → PID 1 is gone → Docker stops the container
```

Total time: less than 1 second.

**`STATUS: Exited (0) 8 seconds ago`** — this is exactly what your terminal showed.

### Was This a Crash? Was Something Wrong?

**No. Nothing was wrong.**

Exit code `(0)` = success. The program ran and finished normally.
It just had nothing to do.

Think of it this way:
- If you hire a chef and give them no food and no kitchen — they say "I cannot cook here" and leave
- That is not a failure. That is a logical decision.
- Node.js made the same logical decision.

---

## 4. What is `-it`? Explained From Zero

`-it` is two separate flags combined: `-i` and `-t`

Let us understand each one separately first.

---

### What is `-i` (Interactive)?

**`-i`** stands for **interactive**.

To understand `-i`, you need to understand **STDIN**.

**What is STDIN?**

Every program has three communication channels:

```
STDIN  (Standard Input)   → the channel through which program RECEIVES input
STDOUT (Standard Output)  → the channel through which program SENDS normal output
STDERR (Standard Error)   → the channel through which program SENDS error messages
```

Think of a telephone:
- STDIN = your ear (you receive what the other person says)
- STDOUT = your mouth (you speak to the other person)
- STDERR = a separate emergency alarm line

When you type in a terminal, your keystrokes go to STDIN of the running program.
The program reads STDIN and responds.

**The problem with containers:**

By default, when you run `docker run someimage`, Docker does NOT connect your keyboard to the container's STDIN.

It is like calling someone but keeping the line on mute.
You cannot speak to the container.
The container cannot hear your keystrokes.

**`-i` fixes this:**

`-i` tells Docker: "Keep STDIN open and connected to my keyboard."
Now your keystrokes flow into the container.
The container can hear you typing.

```
Without -i:
Your keyboard → [disconnected] → Container's STDIN
Container cannot hear you type.

With -i:
Your keyboard → connected → Container's STDIN
Container hears every key you press.
```

---

### What is `-t` (TTY / Terminal)?

**`-t`** stands for **TTY** (TeleTYpewriter).

**What is a TTY?**

Long ago, before computer screens existed, people typed on machines called teletypewriters.
These machines printed output on paper as you typed.

Today, "TTY" means a **terminal interface** — the window where you type commands and see results.

When you open your Ubuntu terminal, you are using a TTY.
It gives you:
- A cursor that blinks
- A prompt that shows (`$` or `#` or `>`)
- Coloured output
- Ability to use arrow keys to navigate
- Ctrl+C to stop a program

**The problem with containers:**

By default, a container has NO terminal attached.
Programs that need a terminal to display their interface will not work properly.
Their output looks garbled or they refuse to start interactive mode.

**`-t` fixes this:**

`-t` tells Docker: "Create a fake terminal (pseudo-TTY) and attach it to the container."
Now the container thinks it is talking to a real terminal.
Programs display their proper interface.

```
Without -t:
Container runs with no terminal.
Node.js: "I don't see a terminal → I won't start interactive REPL"
Output is broken, no cursor, no colours.

With -t:
Docker creates a fake terminal and connects it.
Node.js: "Oh! I see a terminal → I'll start interactive REPL"
You see the > prompt. Colours work. Arrow keys work.
```

---

### `-it` Together = Full Interactive Experience

When you combine both flags:

```
-i = keep keyboard (STDIN) connected
-t = give me a proper terminal screen

Together -it = "Give me a proper interactive terminal where I can type
               and see output, just like a normal terminal session."
```

**Think of it like a phone call:**

```
Without -it:
You make a call but:
- Your microphone is off (-i missing = no STDIN)
- There is no speaker (-t missing = no terminal display)
Result: You cannot communicate at all.

With -it:
- Microphone is ON (-i = STDIN open)
- Speaker is ON (-t = terminal display working)
Result: Normal two-way conversation.
```

---

### When to Use `-it`

Use `-it` whenever you want to **interact** with a running program inside a container.

```bash
docker run -it ubuntu bash          # open bash shell in Ubuntu
docker run -it python:3.12 python   # open Python interactive shell
docker run -it node:20-alpine node  # open Node.js REPL
docker run -it node:20-alpine sh    # open Linux shell inside node image
docker run -it alpine sh            # open shell in minimal Alpine Linux
```

Do NOT use `-it` when:
- Running a server in background (`docker run -d nginx`)
- Running a one-time automated job (`docker run myimage python process.py`)
- You do not need to type anything to the container

---

## 5. Why `-it` Made Node Stay Alive

### Without `-it` — The Timeline

```
docker run node:20-alpine

Docker → starts container
       → runs: docker-entrypoint.sh node
       → node starts
       → node checks: is there a terminal? NO (no -t)
       → node checks: is there keyboard input? NO (no -i)
       → node decides: I have nothing to do. EXIT.
       → node exits (PID 1 gone)
       → container exits
       → total time: < 1 second
```

### With `-it` — The Timeline

```
docker run -it node:20-alpine

Docker → starts container
       → connects your keyboard to container STDIN (because -i)
       → creates a terminal interface (because -t)
       → runs: docker-entrypoint.sh node
       → node starts
       → node checks: is there a terminal? YES! (you gave -t)
       → node thinks: "A human is here! I should start interactive mode."
       → node starts the REPL (the > prompt)
       → node WAITS for you to type something
       → node is waiting... waiting... waiting...
       → PID 1 is still running (node is alive, waiting for input)
       → container is alive
       → container stays alive as long as YOU are typing or connected
```

Node is not doing heavy work.
Node is just sitting there, **waiting** for your input.
But waiting IS a running process.
A waiting process is still alive.
An alive process means the container stays alive.

### Visual Comparison

```
WITHOUT -it:

You type:  docker run node:20-alpine

[Container starts]
[docker-entrypoint.sh launches node]
[node looks around]
[node sees: no keyboard, no terminal]
[node thinks: "I have nothing to do"]
[node exits]
[Container exits]

Total time: < 1 second
Your terminal returns to your normal prompt immediately.


WITH -it:

You type:  docker run -it node:20-alpine

[Container starts]
[Docker connects your keyboard → container STDIN]
[Docker creates terminal → container]
[docker-entrypoint.sh launches node]
[node looks around]
[node sees: keyboard connected! terminal exists!]
[node thinks: "A human wants to talk to me!"]
[node starts REPL]
[node shows: > ]
[node waits...]
[node waits...]
[YOU are in control now]
[Container stays alive until YOU exit]
```

---

## 6. Why SonarQube Is Still Running After 8 Weeks

Look at your `docker ps -a` output:

```
CONTAINER ID   IMAGE                      STATUS
f9da31a71e3a   node:20-alpine             Exited (0) 8 seconds ago
ef8c88d59588   node:20-alpine             Exited (0) 10 minutes ago
6f9774dccc7c   sonarqube:10.6-community   Up 4 days
```

Two node containers — **exited**.
SonarQube — **running for 4 days** (started 8 weeks ago).

Why is SonarQube still alive?

**Because SonarQube is a SERVER.**

A server works like this:

```
Server starts
   ↓
Server says: "I am ready. Waiting for users."
   ↓
User opens browser at localhost:9000
   ↓
Server responds: "Here is the SonarQube dashboard"
   ↓
User does code analysis
   ↓
Server responds with results
   ↓
User closes browser
   ↓
Server says: "Okay. Back to waiting."
   ↓
Waiting...
Waiting...
Waiting...  ← server NEVER finishes. It always waits for next user.
```

The server's main process (PID 1) NEVER exits because it is always waiting.
A waiting process = a running process.
A running process = a running container.

**SonarQube's PID 1 has been running for 4 days and never exited.**
Therefore the container has been alive for 4 days.

### The Pattern — Two Types of Containers

```
TYPE 1: Task containers
   Purpose: Run a specific job, then stop.
   Examples: Run tests, compile code, process a file, send an email
   Behaviour: Starts → does work → finishes → exits → container stops
   node:20-alpine without -it is this type

TYPE 2: Service containers
   Purpose: Run forever, serving requests continuously.
   Examples: Web servers, databases, API servers, monitoring tools
   Behaviour: Starts → waits for requests → responds → waits → responds → never exits
   SonarQube is this type
```

**In production, almost everything is a service container.**
Web servers (nginx), databases (PostgreSQL), apps (your Node.js API) — all run forever.
You start them with `docker run -d` (detached mode) so they run in the background.

---

## 7. Your Terminal Output — Every Line Explained

```
muskan@muskan-ThinkPad-L14-Gen-1:~/Lab_Practice/Docker$ docker ps -a
```

**`docker ps`** = process status. Lists containers (like `ps` on Linux lists processes).
**`-a`** = all. Show ALL containers, not just running ones.
Without `-a`, `docker ps` only shows currently running containers.
With `-a`, it shows running + stopped + exited containers.

---

```
CONTAINER ID   IMAGE                      COMMAND                  CREATED          STATUS
```

**Column headers:**

`CONTAINER ID` = a unique ID for each container (like PID but for containers)
`IMAGE` = which image this container was created from
`COMMAND` = what command was run when the container started
`CREATED` = how long ago you created this container
`STATUS` = current state of the container

---

```
f9da31a71e3a   node:20-alpine   "docker-entrypoint.s…"   9 seconds ago   Exited (0) 8 seconds ago
```

`f9da31a71e3a` = short container ID (first 12 characters of the full 64-character ID)

`node:20-alpine` = the image it was created from

`"docker-entrypoint.s…"` = the command that ran. The `…` means it is cut off.
Full command: `docker-entrypoint.sh node`

`9 seconds ago` = you created this container 9 seconds ago

`Exited (0) 8 seconds ago` = it stopped 8 seconds ago with exit code 0

**What is exit code `(0)`?**

When any program finishes, it gives a number called an exit code.
This tells the world: "How did I finish?"

```
Exit code 0   = success. I finished normally. No problems.
Exit code 1   = general error. Something went wrong.
Exit code 2   = misuse of command. Wrong arguments given.
Exit code 137 = killed by signal 9 (SIGKILL). Probably OOM killed.
Exit code 139 = segfault. Program crashed due to memory error.
Exit code 143 = killed by signal 15 (SIGTERM). Asked to stop gracefully.
```

`Exited (0)` = node ran and finished normally. Not a crash. Just nothing to do.

---

```
ef8c88d59588   node:20-alpine   "docker-entrypoint.s…"   10 minutes ago   Exited (0) 10 minutes ago
```

Same thing. Another container you ran earlier. Also exited cleanly with code 0.

---

```
6f9774dccc7c   sonarqube:10.6-community   "/opt/sonarqube/dock…"   8 weeks ago   Up 4 days
```

`6f9774dccc7c` = container ID

`sonarqube:10.6-community` = image name

`"/opt/sonarqube/dock…"` = the startup command for SonarQube (cut off)

`8 weeks ago` = you first created this container 8 weeks ago

`Up 4 days` = it has been running continuously for 4 days

**Why 8 weeks ago but only Up 4 days?**
You created the container 8 weeks ago.
But at some point you stopped it (maybe restarted your ThinkPad).
4 days ago you started it again.
It has been running continuously since then.

---

```
PORTS
0.0.0.0:9000->9000/tcp, [::]:9000->9000/tcp
```

**`PORTS`** = shows how the container's network ports are connected to your machine.

`0.0.0.0:9000->9000/tcp`

Breaking this down:

`0.0.0.0` = all network interfaces on your machine (localhost, your WiFi IP, etc.)
`9000` = port 9000 on YOUR machine (host port)
`->` = maps to / forwards to
`9000` = port 9000 INSIDE the container
`/tcp` = using the TCP protocol

**In plain English:**
"Any traffic arriving at port 9000 on your ThinkPad is forwarded to port 9000 inside the SonarQube container."

This is why you can open `http://localhost:9000` in your browser and see SonarQube.
Your browser → port 9000 on your ThinkPad → Docker forwards → port 9000 inside SonarQube container → SonarQube responds.

`[::]:9000->9000/tcp` = same thing but for IPv6 addresses. `[::]` is IPv6's version of `0.0.0.0`.

---

```
NAMES
test-node
node
sonarqube
```

**`NAMES`** = the human-readable name of the container.

You can name containers with `--name`:
```bash
docker run --name test-node node:20-alpine
```

If you do not give a name, Docker generates a random one like `pensive_bohr` or `happy_tesla`.

Names are useful because instead of remembering `f9da31a71e3a`, you can just type `test-node`:
```bash
docker logs test-node     # instead of docker logs f9da31a71e3a
docker stop sonarqube     # instead of docker stop 6f9774dccc7c
```

---

## 8. Node REPL vs Linux Shell — Why Your `echo` Command Failed

### What Happened

You ran:
```bash
docker run -it --name test-node2 node:20-alpine
```

You saw:
```
Welcome to Node.js v20.20.2.
Type ".help" for more information.
>
```

Then you typed:
```
echo "This is node image" > node.txt
```

And got:
```
Uncaught SyntaxError: Unexpected string
```

### Why?

You were not in a Linux shell. You were inside the **Node.js REPL**.

**What is a REPL?**

REPL = Read, Evaluate, Print, Loop.

It is an interactive environment for a programming language.

```
Read    → REPL reads what you type
Evaluate → REPL tries to run it as code in that language
Print   → REPL prints the result
Loop    → REPL goes back to waiting for more input
```

The Node.js REPL only understands **JavaScript**.
Not Linux commands. Not bash. Not shell.

---

### Two Different Languages, Two Different People

Think of it this way:

```
You are talking to TWO different people:

Person 1: Speaks JAVASCRIPT only (Node.js REPL)
Person 2: Speaks LINUX COMMANDS only (Linux Shell)

You asked Person 1:
echo "This is node image" > node.txt

Person 1 (Node.js REPL) tried to understand this as JavaScript.

In JavaScript:
echo              = not a keyword JavaScript knows
"This is node"   = a string (that's fine)
> node.txt        = huh? After a string, > is a comparison operator
                    but node.txt is not a valid right side for comparison

So Node.js said:
Uncaught SyntaxError: Unexpected string

Translation: "I tried to understand what you said as JavaScript. I couldn't. This is a syntax error."
```

**You asked the wrong person.**

---

### How to Identify Where You Are

The **prompt symbol** tells you EXACTLY which environment you are in:

```
>            ← You are in Node.js REPL. Only JavaScript works here.
$            ← You are in Linux shell (normal user). Linux commands work.
#            ← You are in Linux shell (root user). Linux commands work.
/ #          ← You are in Alpine Linux shell as root.
python>>>    ← You are in Python REPL. Only Python works.
```

**Your situation:**

When you ran `docker run -it node:20-alpine`, you got `>` — the Node.js REPL.

---

### What DOES Work in the Node.js REPL

Since the Node.js REPL only understands JavaScript, you can type JavaScript:

```javascript
> 2 + 2
4

> let name = "Muskan"
undefined

> name
'Muskan'

> Math.sqrt(25)
5

> "Hello".toUpperCase()
'HELLO'

> [1, 2, 3].map(x => x * 2)
[ 2, 4, 6 ]

> .exit           // special REPL command to quit
```

Notice: these are all JavaScript expressions. Not Linux commands.

---

### How to Get a Linux Shell Instead

If you want to run Linux commands (like `echo`, `ls`, `pwd`, `cat`), you need to start the **shell**, not Node.

```bash
docker run -it node:20-alpine sh
```

**The difference:**

`docker run -it node:20-alpine`
→ runs the default CMD which is `node`
→ you get Node.js REPL (`>` prompt)

`docker run -it node:20-alpine sh`
→ you are overriding the default CMD with `sh`
→ `sh` is the Linux shell program
→ you get Linux shell (`/ #` prompt)

Now you can run Linux commands:

```bash
/ # echo "This is node image" > node.txt
/ # cat node.txt
This is node image
/ # ls
bin   dev   etc   home  lib   ...
/ # pwd
/
/ # exit
```

---

### The Full Comparison

| | Node.js REPL | Linux Shell |
|-|--------------|-------------|
| **How to start** | `docker run -it node:20-alpine` | `docker run -it node:20-alpine sh` |
| **Prompt** | `>` | `/ #` or `$` |
| **Language** | JavaScript only | Linux commands |
| **Works** | `2+2`, `let x = 5`, `Math.sqrt(9)` | `ls`, `echo`, `cat`, `pwd`, `mkdir` |
| **Does NOT work** | `echo`, `ls`, `cat`, `mkdir` | `let x = 5`, `Math.sqrt(9)` |
| **Exit command** | `.exit` | `exit` |

---

## 9. How Docker Decides What to Run

This connects the ENTRYPOINT and CMD concepts to your confusion.

The `node:20-alpine` image has these in its Dockerfile:

```dockerfile
ENTRYPOINT ["docker-entrypoint.sh"]
CMD ["node"]
```

**ENTRYPOINT** = the program that ALWAYS runs first. Cannot be replaced by arguments.
**CMD** = the default argument passed to the ENTRYPOINT. CAN be replaced.

### Case 1: `docker run -it node:20-alpine`

```
No extra command provided.
Docker uses the default CMD = "node"

Runs: docker-entrypoint.sh node
      ─────────────────────  ────
      ENTRYPOINT             CMD (default)

node starts.
node sees a terminal (because -t) and keyboard (because -i).
node starts REPL.
You see: >
```

### Case 2: `docker run -it node:20-alpine sh`

```
You provided "sh" as extra command.
Docker REPLACES the default CMD with your "sh"

Runs: docker-entrypoint.sh sh
      ─────────────────────  ──
      ENTRYPOINT             YOUR override (replaces CMD)

sh starts.
sh sees a terminal (because -t) and keyboard (because -i).
sh starts the Linux shell.
You see: / #
```

### Case 3: `docker run node:20-alpine` (no -it)

```
No extra command. No -it.
Docker uses default CMD = "node"

Runs: docker-entrypoint.sh node

node starts.
node sees: NO terminal (no -t), NO keyboard (no -i).
node thinks: "Nothing interactive here. No file to run either."
node exits immediately.
Container exits.
```

---

## 10. Practical Labs

### Lab 1 — Prove That Without `-it`, Container Exits

```bash
# Run WITHOUT -it
docker run --name no-it-test node:20-alpine

# Check status immediately
docker ps -a | grep no-it-test
```

**What you will see:**
```
STATUS
Exited (0) 1 second ago
```

**Why:** Node started, found no terminal and no keyboard, had nothing to do, exited immediately.

```bash
# Clean up
docker rm no-it-test
```

---

### Lab 2 — Prove That With `-it`, Container Stays Alive

```bash
# Run WITH -it
docker run -it --name it-test node:20-alpine
```

**What you will see:**
```
Welcome to Node.js v20.20.2.
Type ".help" for more information.
>
```

Node is waiting for you. The container is ALIVE.

Now open a SECOND terminal window (do not close this one) and run:

```bash
# In the second terminal, check container status
docker ps | grep it-test
```

**What you will see:**
```
STATUS
Up 30 seconds
```

The container is running because Node's REPL is running, waiting for your input.

Go back to the first terminal and type `.exit` to quit the REPL.

Check the second terminal again:
```bash
docker ps -a | grep it-test
```

**What you will see now:**
```
STATUS
Exited (0) 5 seconds ago
```

The moment you exited Node (PID 1 died), the container died.

```bash
# Clean up
docker rm it-test
```

---

### Lab 3 — Understand the Difference: Node REPL vs Linux Shell

```bash
# Start Node REPL (default)
docker run -it --rm node:20-alpine
```

You get `>` — you are in Node.js REPL.

Try these JAVASCRIPT commands:
```javascript
> 10 + 5
15

> "hello".length
5

> let name = "Muskan"
undefined

> name
'Muskan'

> .exit
```

Now start the LINUX SHELL:
```bash
docker run -it --rm node:20-alpine sh
```

You get `/ #` — you are in Linux shell.

Try these LINUX commands:
```bash
/ # echo "Hello from Docker"
Hello from Docker

/ # pwd
/

/ # ls
bin   dev   etc   home  lib   lib64 media mnt   opt   proc
root  run   sbin  srv   sys   tmp   usr   var

/ # echo "This is node image" > node.txt
/ # cat node.txt
This is node image

/ # exit
```

**Notice:** The SAME image, but completely different experience based on what you asked it to run.

---

### Lab 4 — See PID 1 Inside a Running Container

```bash
# Start a container in the background
docker run -d --name pid-test nginx

# Go inside the container
docker exec -it pid-test sh

# Inside the container, run:
ps aux
```

**What you will see:**
```
PID   USER  COMMAND
1     root  nginx: master process nginx -g daemon off;
...
```

PID 1 inside the container = nginx.
This is why the container stays alive — nginx (PID 1) never exits.

```bash
# Exit the container
exit

# Check the container is still running
docker ps | grep pid-test

# Clean up
docker rm -f pid-test
```

---

### Lab 5 — Verify Exit Codes Tell the Story

```bash
# Run node without -it (will exit immediately)
docker run --name exit-test node:20-alpine

# Check the exit code
docker inspect exit-test --format '{{.State.ExitCode}}'
# Output: 0  (success — node ran and exited cleanly, just had nothing to do)

# Clean up
docker rm exit-test

# Now force a failure — run a command that doesn't exist
docker run --name fail-test node:20-alpine non-existent-command

# Check the exit code
docker inspect fail-test --format '{{.State.ExitCode}}'
# Output: non-zero (error — command not found)

# Clean up
docker rm fail-test
```

---

### Lab 6 — Run a Service Container vs a Task Container

```bash
# SERVICE CONTAINER — runs forever
# -d = detached (runs in background)
docker run -d --name service-test nginx
docker ps | grep service-test
# STATUS: Up X seconds  ← running, never exits on its own

# TASK CONTAINER — runs and exits
# --rm = auto-delete when done
docker run --rm node:20-alpine node -e "console.log('Task done!')"
# Output: Task done!
# Container exits immediately after printing

# The service container is still running
docker ps | grep service-test

# Clean up
docker rm -f service-test
```

**`node -e "..."`** = the `-e` flag tells node to run a JavaScript string directly, not from a file.
This gives node actual work to do — so it runs the code and exits (task container behaviour).

---

## 11. Why This Matters in Interviews

---

### Q: Why did your container exit immediately after running `docker run node:20-alpine`?

> "A container runs only as long as its main process — PID 1 — is alive. When I ran `docker run node:20-alpine`, Docker executed `docker-entrypoint.sh node`. The Node.js runtime started but found no JavaScript file to execute, no keyboard input connected, and no terminal attached. With nothing to do and no interactive session to serve, Node exited immediately. When PID 1 (node) exited, Docker stopped the container. The exit code was 0, meaning it finished cleanly — this was expected behaviour, not a crash."

---

### Q: What does `-it` do in `docker run -it`?

> "`-it` is two flags combined. `-i` keeps STDIN open and connects the keyboard to the container's input channel. Without `-i`, the container cannot receive any keyboard input. `-t` allocates a pseudo-TTY — a fake terminal that makes the container think it is connected to a real terminal window. Without `-t`, programs that need a terminal to display their interface either refuse to start interactive mode or display garbled output. Together, `-it` gives you a proper interactive terminal session with the container, exactly like SSH-ing into a server. I use `-it` any time I need to interact with a running process inside a container."

---

### Q: What is the difference between the Node.js REPL and the Linux shell inside a Docker container?

> "When you run `docker run -it node:20-alpine`, the container starts with the default CMD which is `node`. This opens the Node.js REPL — an interactive JavaScript interpreter. The REPL only understands JavaScript, not Linux commands. The prompt shows `>`. If you want a Linux shell instead, you override the CMD by running `docker run -it node:20-alpine sh`. This starts the shell program `sh` instead of `node`, and you get the `/ #` prompt where Linux commands like `ls`, `echo`, `cat`, and `mkdir` work. Same image, completely different behaviour based on what you tell Docker to run."

---

### Q: Why is your SonarQube container running for 4 days but your Node containers exited in seconds?

> "This comes down to what PID 1 is doing. SonarQube is a server application. Its main process starts and then enters an infinite loop waiting for HTTP requests. It never exits because it is always waiting for the next user. Docker keeps the container alive because PID 1 is still running. The Node containers, on the other hand, started the Node.js runtime with nothing to do — no file to run and no interactive terminal attached. Node had no reason to keep running, so it exited within a second. Same Docker rule applies to both: the container lives only as long as its main process lives."

---

## 12. Quick Revision Cheatsheet

| Concept | Simple Explanation |
|---------|-------------------|
| **Process** | A program that is currently running |
| **PID** | Process ID — unique number assigned to every running program |
| **PID 1** | The first and main process inside a container |
| **Container lifetime** | Container lives ONLY as long as PID 1 is alive |
| **Exit code 0** | Process finished successfully — not a crash |
| **Exit code non-zero** | Process finished with an error |
| **`-i` flag** | Keep STDIN open — container can receive keyboard input |
| **`-t` flag** | Create a fake terminal — proper display, cursor, colours |
| **`-it` combined** | Full interactive terminal session with the container |
| **Node.js REPL** | JavaScript interactive interpreter — `>` prompt |
| **Linux shell** | Linux command interpreter — `/ #` or `$` prompt |
| **Service container** | Runs forever — server always waiting for requests |
| **Task container** | Runs and exits — completes a job then stops |

| Command | What it Does |
|---------|-------------|
| `docker ps` | Show only RUNNING containers |
| `docker ps -a` | Show ALL containers (running + stopped + exited) |
| `docker run -it node:20-alpine` | Interactive Node.js REPL |
| `docker run -it node:20-alpine sh` | Linux shell inside node image |
| `docker run -d nginx` | Run nginx server in background |
| `docker run --rm node:20-alpine node -e "code"` | Run JavaScript, exit, auto-delete |
| `docker inspect <id> --format '{{.State.ExitCode}}'` | Check exit code of a container |

| Prompt | Where You Are | What Works |
|--------|--------------|------------|
| `>` | Node.js REPL | JavaScript only |
| `/ #` | Alpine Linux shell (root) | Linux commands |
| `$` | Linux shell (normal user) | Linux commands |
| `#` | Linux shell (root) | Linux commands |

| Exit Code | Meaning |
|-----------|---------|
| `0` | Success — finished normally |
| `1` | General error |
| `137` | OOM killed or force killed (SIGKILL) |
| `139` | Segfault — program crashed |
| `143` | Graceful stop requested (SIGTERM) |

---

## 13. Further Reading

- [docker run official docs](https://docs.docker.com/reference/cli/docker/container/run/) — All flags including `-i`, `-t`, `-d`, `--rm`
- [Docker container lifecycle](https://docs.docker.com/engine/reference/run/#container-lifecycle) — How containers start, run, and stop
- [PID 1 in containers](https://docs.docker.com/engine/reference/run/#pid-settings---pid) — PID namespace and why it matters
- [Node.js REPL documentation](https://nodejs.org/en/learn/command-line/how-to-use-the-nodejs-repl) — What the Node REPL is and what it can do
- [Exit codes in Docker](https://docs.docker.com/engine/reference/run/#exit-status) — What different exit codes mean

---

*File: `docker_container_lifecycle_and_flags.md` | Bonus Topic | Docker Interview Preparation 2026*
