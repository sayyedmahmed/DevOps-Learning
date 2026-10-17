# File Viewing Commands

Let's go through each one — what it does, when you'll use it in DevOps, and the exact commands.

---

## File Viewing Commands — Your Eyes on the System

| Command | Purpose | DevOps Use |
|---------|---------|-------------|
| `ls -la` | List all files with details | Check permissions, owners, sizes |
| `cat` | Print entire file | Quick config check, pipe to grep |
| `head` | First few lines | Confirm file isn't empty, see top of logs |
| `tail -f` | Follow live log updates | Debug in real time |
| `less` | Scroll through large files | Read logs without locking up terminal |
| `zcat` | Read compressed `.gz` files | Check rotated/archived logs |

---

## 1. `ls -la` — List Everything

```bash
ls -la
```
**What it means:**
- `-l` → long format (permissions, owner, size, date)
- `-a` → show hidden files (files starting with `.`)

**Example output:**
```
drwx------. 2 dev1 dev1   62 Jun 10 11:47 .
drwxr-xr-x. 4 root root   28 Jun 10 11:00 ..
-rw-r--r--. 1 dev1 dev1   18 Jun 10 11:47 .bashrc
-rw-r--r--. 1 dev1 dev1  141 Jun 10 11:47 file.txt
```
Shows permissions, owner:group, size, timestamp.

---

## 2. `cat` — Dump Entire File

```bash
cat /etc/passwd
```
Prints the whole file at once. Best for **small files**.

**DevOps use:** Quick checks like:
```bash
cat /etc/hostname
cat /etc/os-release
```

---

## 3. `head` — First Few Lines

```bash
head -n5 /var/log/messages
```
- `-n5` → show first 5 lines only

**DevOps use:** Check if a log is the right file, see recent boot messages:
```bash
head -n20 /var/log/boot.log
```

---

## 4. `tail -f` — Follow Live

```bash
tail -f /var/log/messages
```
Shows last 10 lines, then **keeps watching** — new lines appear in real time.

**DevOps use:** The classic debugging command:
```bash
tail -f /var/log/nginx/access.log    # Watch web traffic live
tail -f /var/log/mysql/error.log     # Catch database errors as they happen
```
Press `Ctrl+C` to stop.

---

## 5. `less` — Scroll Through Big Files

```bash
less /var/log/messages
```
Opens the file in a pager. You control it:
- `Space` → next page
- `b` → back one page
- `/keyword` → search forward
- `q` → quit

**DevOps use:** Reading giant log files without crashing your terminal. `cat` on a 10GB file freezes everything; `less` doesn't.

---

## 6. `zcat` — Read Compressed Files

```bash
zcat /var/log/messages-20240601.gz
```
Reads a `.gz` file without unzipping it first.

**DevOps use:** Log rotation compresses old logs:
```bash
ls /var/log/messages*
# messages           ← current
# messages-20240601.gz  ← yesterday, compressed

zcat messages-20240601.gz | grep ERROR   # Search old logs instantly
```

---

## Quick Hands-On Practice

```bash
# List everything in current directory
ls -la

# See all users on the system
cat /etc/passwd

# First 5 lines of system log
head -n5 /var/log/messages

# Watch system log live (Ctrl+C to stop)
tail -f /var/log/messages

# Browse system log interactively (q to quit)
less /var/log/messages

# Read a compressed file if it exists
ls /var/log/messages*.gz
zcat /var/log/messages-*.gz | head -n10
```

---

## Cheat Sheet

| Task | Command |
|------|---------|
| See all files + hidden | `ls -la` |
| Read small file | `cat filename` |
| First 10 lines | `head filename` or `head -n10 filename` |
| Last 10 lines | `tail filename` |
| Watch live | `tail -f filename` |
| Scroll big file | `less filename` |
| Read .gz file | `zcat filename.gz` |
| Search inside output | `grep keyword` — pipe it: `cat file \| grep error` |

---

Done. These six commands will be 90% of your daily log and config work.
---

**Done.** Copy this, and you have your reference notes. Ready for the next topic whenever.
