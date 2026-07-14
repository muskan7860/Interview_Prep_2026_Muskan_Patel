# Understanding cut and awk — Word by Word
### Zero Knowledge Explanation | Muskan | 2026
### "If you cannot explain every character, you do not own the command yet"

---

## THE COMMANDS YOU ASKED ABOUT

```bash
grep "^PRETTY_NAME" /etc/os-release | cut -d'"' -f2
df -h / | awk 'NR==2 {print $5}'
```

Let us break every single character of both commands.
After reading this — you will be able to explain them in your sleep.

---

## PART 1 — Understanding `cut`

### First — What Problem Does `cut` Solve?

Run this on your terminal:
```bash
grep "^PRETTY_NAME" /etc/os-release
```

You will see:
```
PRETTY_NAME="Ubuntu 24.04.4 LTS"
```

You do NOT want the whole line.
You ONLY want: `Ubuntu 24.04.4 LTS`

The problem is — there is garbage before and after the name:
```
PRETTY_NAME="Ubuntu 24.04.4 LTS"
^^^^^^^^^^^^  ^^^^^^^^^^^^^^^^^^^
don't want    want this only
```

`cut` is a knife. It cuts the line at a character you choose and gives you a specific piece.

---

### The Cutting Concept — With a Real Example

Imagine you have a sentence:
```
PRETTY_NAME="Ubuntu 24.04.4 LTS"
```

Now imagine you take scissors and cut everywhere you see a `"` (double quote).

After cutting at every `"` you get 3 pieces:

```
Piece 1:  PRETTY_NAME=
Piece 2:  Ubuntu 24.04.4 LTS
Piece 3:  (empty — nothing after the last quote)
```

You want Piece 2. So you ask for field number 2.

That is exactly what `cut -d'"' -f2` does.

---

### Breaking `cut -d'"' -f2` — Every Character Explained

```bash
cut -d'"' -f2
```

| Part | What it means | Plain English |
|------|--------------|---------------|
| `cut` | The command name | The knife/scissors tool |
| `-d` | d stands for **delimiter** | "Cut at this character" |
| `'"'` | The character to cut at | Cut at every `"` (double quote) |
| `-f` | f stands for **field** | "Give me piece number..." |
| `2` | The piece number | "...give me piece number 2" |

So the whole thing says:
**"Cut this text at every `"` character and give me piece number 2"**

---

### Practice This On Terminal

```bash
# Step 1 — See the full line first
grep "^PRETTY_NAME" /etc/os-release
# Output: PRETTY_NAME="Ubuntu 24.04.4 LTS"

# Step 2 — Now cut it and get piece 2
grep "^PRETTY_NAME" /etc/os-release | cut -d'"' -f2
# Output: Ubuntu 24.04.4 LTS
```

Run both. See the difference. Understand what cutting did.

---

### More cut Examples — To Make It Stick

```bash
# Example 1 — Cut a simple string at :
echo "name:Muskan:DevOps" | cut -d':' -f1
# Cuts at every :
# Piece 1: name
# Piece 2: Muskan
# Piece 3: DevOps
# We asked for piece 1 → Output: name

echo "name:Muskan:DevOps" | cut -d':' -f2
# Output: Muskan

echo "name:Muskan:DevOps" | cut -d':' -f3
# Output: DevOps
```

```bash
# Example 2 — Get just the username from /etc/passwd
# /etc/passwd has lines like: root:x:0:0:root:/root:/bin/bash
# Fields are separated by :
# Field 1 = username

head -1 /etc/passwd | cut -d':' -f1
# Output: root
```

```bash
# Example 3 — Remove % from disk percentage
# df -h / gives: 33%
# We want just: 33

df -h / | awk 'NR==2 {print $5}' | cut -d'%' -f1
# Cut at % and give piece 1
# Output: 33
```

**Practice all three. Run them yourself.**

---

## PART 2 — Understanding `awk`

### First — What Problem Does `awk` Solve?

Run this on your terminal:
```bash
df -h /
```

You will see a TABLE:
```
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1       457G  143G  291G  33%  /
```

You only want `33%` — the disk usage percentage.

That is in ROW 2, COLUMN 5.

`awk` lets you pick a specific ROW and a specific COLUMN from any table.

---

### The Table Concept

Think of it exactly like an Excel sheet.

```
         Col1        Col2   Col3   Col4   Col5    Col6
Row 1:   Filesystem  Size   Used   Avail  Use%    Mounted on
Row 2:   /dev/sda1   457G   143G   291G   33%     /
```

You want Row 2, Column 5 = `33%`

In awk language:
- Row = NR (Number of Row)
- Column = $1, $2, $3, $4, $5... ($NF = last column)

---

### Breaking `awk 'NR==2 {print $5}'` — Every Character Explained

```bash
awk 'NR==2 {print $5}'
```

| Part | What it means | Plain English |
|------|--------------|---------------|
| `awk` | The command name | The table-reading tool |
| `'...'` | Single quotes wrap the awk program | Everything inside is the instruction |
| `NR` | Number of Row — built-in variable | The current row number awk is reading |
| `==` | Equals sign (comparison) | "is equal to" |
| `2` | The row number we want | Row number 2 |
| `NR==2` | The condition | "Only when we are on row 2" |
| `{...}` | Curly braces | "Do this action when condition is true" |
| `print` | Print command inside awk | Show this on screen |
| `$5` | Column number 5 | The 5th column of that row |

So the whole thing says:
**"When you reach row number 2, print column number 5"**

---

### Why NR==2 and not NR==1?

Because Row 1 is the HEADER (column titles):
```
Row 1: Filesystem   Size   Used   Avail   Use%   Mounted on
Row 2: /dev/sda1    457G   143G   291G    33%    /
```

We want the DATA, not the header. Data is in row 2.
So we say `NR==2`.

---

### What is $1, $2, $3, $4, $5 in awk?

In awk, every column has a number starting from 1.

For the `df -h /` output:
```
Filesystem    Size   Used   Avail   Use%   Mounted on
$1            $2     $3     $4      $5     $6
```

| Variable | Column | Value |
|----------|--------|-------|
| `$1` | Column 1 | `/dev/sda1` |
| `$2` | Column 2 | `457G` |
| `$3` | Column 3 | `143G` |
| `$4` | Column 4 | `291G` |
| `$5` | Column 5 | `33%` |
| `$6` | Column 6 | `/` |
| `$NF` | Last column | `/` (NF = Number of Fields = last one) |

---

### Practice awk — Run These Yourself

```bash
# Get column 5 from df output (disk usage %)
df -h / | awk 'NR==2 {print $5}'
# Output: 33%

# Get column 2 (total size)
df -h / | awk 'NR==2 {print $2}'
# Output: 457G

# Get column 4 (available space)
df -h / | awk 'NR==2 {print $4}'
# Output: 291G
```

```bash
# Now try with free -h
free -h
# Output:
#               total    used    free
# Mem:          15Gi    10Gi   1.0Gi
# Swap:         2.0Gi   0.0Gi  2.0Gi

# The Mem: row — we want column 2 (total RAM)
free -h | awk '/^Mem:/ {print $2}'
# Output: 15Gi
```

Wait — this one is different! Let me explain `/^Mem:/`

---

### Understanding `/^Mem:/` in awk

```bash
free -h | awk '/^Mem:/ {print $2}'
```

Here instead of `NR==2` we used `/^Mem:/`.

Why? Because in `free -h` output the Mem: row is not always row 2.
On some systems there is an extra line at the top.
So instead of guessing the row number, we SEARCH for the row by name.

| Part | What it means | Plain English |
|------|--------------|---------------|
| `/` | Start of pattern search | "Find a line that contains..." |
| `^` | Start of line anchor | "...that STARTS WITH..." |
| `Mem:` | The text to find | "...the word Mem:" |
| `/` | End of pattern search | End of search condition |
| `{print $2}` | Action | "Print column 2 of that line" |

So `/^Mem:/ {print $2}` means:
**"Find the line that starts with Mem: and print column 2 from it"**

---

### The Two Ways to Pick a Row in awk

```bash
# Way 1: By row NUMBER
# Use when you KNOW which row number has the data
awk 'NR==2 {print $5}'

# Way 2: By row CONTENT (search by text)
# Use when you know what the row STARTS WITH
awk '/^Mem:/ {print $2}'

# Rule: If header can change → use content search
# Rule: If you know exact row number → use NR
```

---

## PART 3 — Understanding grep "^PRETTY_NAME"

You already know grep finds lines. But what is `^PRETTY_NAME`?

```bash
grep "^PRETTY_NAME" /etc/os-release
```

| Part | What it means | Plain English |
|------|--------------|---------------|
| `grep` | The search command | Find lines that match |
| `"..."` | Double quotes wrap the pattern | The pattern to search for |
| `^` | Anchor — means start of line | "Line must START WITH..." |
| `PRETTY_NAME` | The text to find | "...the word PRETTY_NAME" |
| `/etc/os-release` | The file to search in | Search in this file |

The `^` is important. Without it, grep would match ANY line that has PRETTY_NAME anywhere — even in the middle.

With `^`, it only matches lines where PRETTY_NAME is at the very start.

```bash
# Without ^ — would match these too:
# export PRETTY_NAME="Ubuntu"   ← PRETTY_NAME in middle
# # PRETTY_NAME is the OS name  ← in a comment

# With ^ — only matches:
# PRETTY_NAME="Ubuntu 24.04.4 LTS"  ← starts with PRETTY_NAME
```

---

## PUTTING IT ALL TOGETHER — The Full Command Explained

```bash
grep "^PRETTY_NAME" /etc/os-release | cut -d'"' -f2
```

**Reading it left to right:**

```
Step 1: grep "^PRETTY_NAME" /etc/os-release
        → Search the file /etc/os-release
        → Find the line that STARTS WITH PRETTY_NAME
        → Output: PRETTY_NAME="Ubuntu 24.04.4 LTS"

Step 2: | (pipe)
        → Take that output and send it to the next command

Step 3: cut -d'"' -f2
        → Cut the text at every " character
        → Give me piece number 2
        → Output: Ubuntu 24.04.4 LTS
```

**Final result:** `Ubuntu 24.04.4 LTS`

---

## QUICK CHEAT SHEET — Paste This Somewhere Visible

```bash
# cut — splits text at a character and picks a piece
cut -d'CHARACTER' -f PIECE_NUMBER

# Examples:
cut -d'"' -f2    → cut at "  give piece 2
cut -d':' -f1    → cut at :  give piece 1
cut -d'%' -f1    → cut at %  give piece 1 (removes %)
cut -d',' -f3    → cut at ,  give piece 3

# awk — picks row and column from a table
awk 'NR==ROW_NUMBER {print $COLUMN_NUMBER}'

# Examples:
awk 'NR==2 {print $5}'    → row 2, column 5
awk 'NR==2 {print $2}'    → row 2, column 2
awk 'NR==1 {print $1}'    → row 1, column 1

# awk — find row by content
awk '/^TEXT/ {print $COLUMN}'

# Examples:
awk '/^Mem:/ {print $2}'       → find line starting Mem:, col 2
awk '/^PRETTY_NAME/ {print $1}' → find line starting PRETTY_NAME, col 1

# grep — find lines matching a pattern
grep "^TEXT" FILE    → lines STARTING WITH text
grep "TEXT" FILE     → lines CONTAINING text anywhere
grep "TEXT$" FILE    → lines ENDING WITH text
```

---

## HOW TO EXPLAIN THESE IN INTERVIEW

**If interviewer asks: "What does `awk 'NR==2 {print $5}'` do?"**

Say:
> "awk reads output like a table with rows and columns.
> NR is the row number. NR==2 means — only act on row 2.
> $5 means column 5. So this command says:
> go to row 2 and print the value in column 5.
> I use it with `df -h /` because the first row is the header
> and the actual disk data is in row 2, column 5 is the usage percentage."

**If interviewer asks: "What does `cut -d'"' -f2` do?"**

Say:
> "cut is used to split text at a specific character.
> -d means delimiter — the character to cut at.
> Here I am cutting at the double quote character.
> -f2 means give me piece number 2.
> So if I have `PRETTY_NAME="Ubuntu 24.04.4 LTS"`,
> cutting at `"` gives three pieces — before the first quote,
> between the two quotes, and after the last quote.
> Piece 2 is the actual OS name — `Ubuntu 24.04.4 LTS`."

---

> 📁 Save this file to:
> `Interview_Preparation_2026/shell_scripting/Understanding_cut_awk_Commands.md`
>
> Practice every command in this file on your terminal before moving forward.
