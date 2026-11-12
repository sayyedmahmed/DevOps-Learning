# File Operations — Redirection, Counting, Disk Usage

Here's your quick practical guide to file redirection, counting, and disk usage.

---

## File Operations — Redirection, Counting, Disk Usage

---

## 1. Redirection: `>` vs `>>`

### `>` — Overwrite (Dangerous)
```bash
echo "Hello World" > file.txt
```
**What it does:** Creates the file if it doesn't exist, or **completely replaces** the content if it does.

**Example:**
```bash
echo "Line 1" > notes.txt
echo "Line 2" > notes.txt
cat notes.txt
```
Output: `Line 2` only. Line 1 is gone.

---

### `>>` — Append (Safe)
```bash
echo "Hello World" >> file.txt
```
**What it does:** Adds content to the **end** of the file without deleting anything.

**Example:**
```bash
echo "Line 1" >> notes.txt
echo "Line 2" >> notes.txt
cat notes.txt
```
Output:
```
Line 1
Line 2
```
Both lines preserved.

---

### Quick Visual
```
>   →  Opens file, wipes it clean, writes new content
>>  →  Opens file, goes to the end, adds new content
```

**DevOps tip:** Double-check your arrows. `>` on a config file by accident = disaster. `>>` for logging = safe.

---

## 2. `wc` — Word Count (Mostly Line Count)

### Count Lines
```bash
wc -l /etc/passwd
```
Output: `35 /etc/passwd` → 35 users on the system.

### Count Words
```bash
wc -w file.txt
```

### Count Characters
```bash
wc -c file.txt
```

### DevOps Use
```bash
# How many errors in today's log?
grep ERROR /var/log/messages | wc -l

# How many files in a directory?
ls | wc -l
```

---

## 3. `du -sh` — Disk Usage

### `du -sh *` — Size of Each Item in Current Directory
```bash
du -sh *
```
Output:
```
4.0K    file.txt
12M     logs
500K    script.sh
```
Shows human-readable sizes (`K`, `M`, `G`) for everything.

---

### `du -sh /home/*` — Size of Each User's Home
```bash
du -sh /home/*
```
Output:
```
150M    /home/adminuser
45M     /home/guestuser
2.3M    /home/serviceuser
```
Instantly see who's using the most disk space.

---

### Flags Explained
| Flag | Meaning |
|------|---------|
| `-s` | Summarize (only show total per item, not every subfolder) |
| `-h` | Human-readable (K, M, G instead of bytes) |
| `*` | Everything in the current directory |

---

### Other Useful Variants
```bash
du -sh .                    # Total size of current directory
du -sh /var/log             # Size of log folder
du -sh /*                   # Size of all top-level directories (slow, use sparingly)
```

---

## Full Hands-On Practice

```bash
# 1. Overwrite a file
echo "First line" > test.txt
cat test.txt

# 2. Overwrite again (first line is lost)
echo "Second line" > test.txt
cat test.txt

# 3. Append (keeps everything)
echo "Third line" >> test.txt
cat test.txt

# 4. Count lines in a system file
wc -l /etc/passwd

# 5. Count lines in your file
wc -l test.txt

# 6. Check disk usage of home directories
du -sh /home/*

# 7. Check disk usage of current directory
du -sh *

# 8. Combine: count ERROR lines, save to file
grep ERROR /var/log/messages 2>/dev/null | wc -l >> error_count.txt
cat error_count.txt
```

---

## Cheat Sheet

| Task | Command |
|------|---------|
| Create/overwrite file | `echo "text" > file.txt` |
| Append to file | `echo "text" >> file.txt` |
| Count lines | `wc -l file.txt` |
| Count words | `wc -w file.txt` |
| Size of each item here | `du -sh *` |
| Size of user homes | `du -sh /home/*` |
| Size of current folder | `du -sh .` |

---

Done. These four commands (`>`, `>>`, `wc`, `du`) cover 90% of your daily file operation needs.
