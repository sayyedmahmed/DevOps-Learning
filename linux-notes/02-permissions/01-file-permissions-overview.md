# Linux File Permissions — Beginner to Technical

> **Topic 4.0** | chmod · chown · chgrp · umask · SUID · SGID · Sticky Bit

---

## Why File Permissions Exist

Linux is a **multi-user** system. Multiple users can be logged in at the same time. Without permissions, any user could read, edit, or delete anyone else's files — including system files.

Permissions solve this by answering three questions for every file:
- Who **owns** this file?
- What can the **owner** do with it?
- What can **everyone else** do with it?

---

## The Permission Model

Every file and directory in Linux has:

| Property | Description |
|----------|-------------|
| **Owner** | The user who owns the file |
| **Group** | A group of users associated with the file |
| **Others** | Everyone else on the system |

### Viewing Permissions

```bash
ls -l myfile.txt
```

Output:
```
-rwxr-xr-- 1 alice devs 1024 Jun 10 10:00 myfile.txt
```

Let's break this down character by character:

```
- rwx r-x r--
│ │   │   │
│ │   │   └── Others:  r-- = read only
│ │   └─────── Group:   r-x = read + execute
│ └─────────── Owner:   rwx = read + write + execute
└───────────── File type: - = regular file, d = directory
```

### The Three Permission Types

| Symbol | Name | On a File | On a Directory |
|--------|------|-----------|----------------|
| `r` | Read | View file contents | List directory contents (`ls`) |
| `w` | Write | Edit/delete file | Create/delete files inside |
| `x` | Execute | Run as a program | Enter the directory (`cd`) |

---

## chmod — Change File Mode (Permissions)

`chmod` lets you change who can read, write, or execute a file.

### Symbolic Mode (Easy to Read)

```bash
chmod [who][operator][permission] filename
```

| Who | Meaning |
|-----|---------|
| `u` | User (owner) |
| `g` | Group |
| `o` | Others |
| `a` | All (u + g + o) |

| Operator | Meaning |
|----------|---------|
| `+` | Add permission |
| `-` | Remove permission |
| `=` | Set exactly (replaces existing) |

**Examples:**

```bash
# Add execute permission for the owner
chmod u+x script.sh

# Remove write permission from others
chmod o-w file.txt

# Give group read and execute
chmod g+rx file.txt

# Set owner to rwx, group and others to nothing
chmod u=rwx,go= secret.txt
```

---

### Numeric (Octal) Mode

Each permission has a number value:

| Permission | Value |
|------------|-------|
| Read (r) | 4 |
| Write (w) | 2 |
| Execute (x) | 1 |
| None (-) | 0 |

You **add** these values together for each group (owner / group / others):

```
rwx = 4+2+1 = 7
r-x = 4+0+1 = 5
r-- = 4+0+0 = 4
--- = 0+0+0 = 0
```

So `chmod 755 script.sh` means:

```
7 = rwx  → owner can read, write, execute
5 = r-x  → group can read and execute
5 = r-x  → others can read and execute
```

**Common permission patterns:**

| Command | Result | Use Case |
|---------|--------|----------|
| `chmod 755 file` | rwxr-xr-x | Executable scripts, public programs |
| `chmod 644 file` | rw-r--r-- | Regular files, config files |
| `chmod 600 file` | rw------- | Private files (SSH keys, passwords) |
| `chmod 777 file` | rwxrwxrwx | ⚠️ Everyone can do everything — avoid |
| `chmod 400 file` | r-------- | Read-only, highly protected |

**Practical example:**

```bash
# Make a shell script executable
chmod 755 script.sh

# Verify it
ls -l script.sh
# -rwxr-xr-x 1 alice devs 512 Jun 10 script.sh
```

---

## chown — Change Owner

```bash
chown [user]:[group] filename
```

Only **root (sudo)** can change file ownership.

**Examples:**

```bash
# Change owner to 'bob'
sudo chown bob file.txt

# Change owner to 'bob' AND group to 'devs'
sudo chown bob:devs file.txt

# Change only the group (leave owner unchanged)
sudo chown :devs file.txt

# Change ownership recursively (entire directory)
sudo chown -R bob:devs /var/www/myapp
```

**Check the result:**

```bash
ls -l file.txt
# -rw-r--r-- 1 bob devs 1024 Jun 10 file.txt
#              ^^^ ^^^^
#              │   └── group
#              └── owner
```

---

## chgrp — Change Group

`chgrp` changes only the **group** of a file (shortcut instead of `chown :group`).

```bash
chgrp [group] filename
```

**Examples:**

```bash
# Change group to 'devs'
chgrp devs file.txt

# Recursively change group for all files in a directory
chgrp -R devs /var/www/project
```

> **Tip:** `chgrp devs file.txt` is equivalent to `chown :devs file.txt`

---

## umask — Default Permission Mask

When you create a new file or directory, what permissions does it get by default? That's controlled by `umask`.

### How umask Works

Think of it as a **subtraction** from the maximum permissions:

| Type | Max permissions | umask | Result |
|------|----------------|-------|--------|
| File | 666 (rw-rw-rw-) | 022 | 644 (rw-r--r--) |
| Directory | 777 (rwxrwxrwx) | 022 | 755 (rwxr-xr-x) |

The formula: `result = max - umask`

> Files max out at 666 (not 777) because a newly created file should never be automatically executable.

**Check your current umask:**

```bash
umask
# 0022
```

**Examples:**

```bash
# Set umask so new files are private (only owner can read/write)
umask 077
touch newfile.txt
ls -l newfile.txt
# -rw------- 1 alice alice 0 Jun 10 newfile.txt

# Set umask back to standard
umask 022
```

**Common umask values:**

| umask | New File | New Dir | Use Case |
|-------|----------|---------|----------|
| `022` | 644 | 755 | Standard (most systems) |
| `027` | 640 | 750 | Group has no write, others nothing |
| `077` | 600 | 700 | Private — only owner |
| `002` | 664 | 775 | Collaborative team (group can write) |

> **Note:** `umask` only affects the current shell session. To make it permanent, add it to `~/.bashrc` or `~/.profile`.

---

## Special Permission Bits

Beyond the standard rwx, Linux has three special bits: **SUID**, **SGID**, and **Sticky**. These handle advanced scenarios.

---

### SUID — Set User ID

**What it does:** When a user runs an executable with SUID set, the program runs as the **file's owner** (not the person running it).

**Real-world example:** The `passwd` command needs to write to `/etc/shadow` (which only root can do). SUID lets a regular user run `passwd` with root's privileges — but only for that one program.

```bash
# Check passwd has SUID
ls -l /usr/bin/passwd
# -rwsr-xr-x 1 root root 54256 ... /usr/bin/passwd
#    ^
#    's' here = SUID is set (x + SUID)
```

The `s` in the owner's execute position = SUID is active.

**Setting SUID:**

```bash
# Symbolic
chmod u+s myprogram

# Numeric — prepend 4 to the 3-digit mode
chmod 4755 myprogram
# 4 = SUID, 755 = rwxr-xr-x
```

**Verify:**
```bash
ls -l myprogram
# -rwsr-xr-x  ← lowercase 's' = SUID set + executable
# -rwSr-xr-x  ← uppercase 'S' = SUID set but NOT executable (misconfigured!)
```

> ⚠️ **Security warning:** SUID on scripts or arbitrary files is dangerous. Only use it when you understand exactly why it's needed.

---

### SGID — Set Group ID

**On executables:** Similar to SUID, but the program runs with the **file's group** instead of the runner's group.

**On directories (more common use):** Any new file created inside the directory **inherits the directory's group** — not the creator's primary group.

**Real-world example:** A shared project folder where all files should belong to the `devs` group, regardless of who creates them.

```bash
# Create a shared directory
mkdir /shared/project
chown :devs /shared/project
chmod g+s /shared/project

# Now any file created inside gets group 'devs' automatically
```

**Setting SGID:**

```bash
# Symbolic
chmod g+s directory/

# Numeric — prepend 2
chmod 2755 directory/
```

**Verify:**
```bash
ls -ld /shared/project
# drwxr-sr-x  ← 's' in group execute position = SGID
```

---

### Sticky Bit

**What it does:** On a directory, users can only **delete or rename their own files** — even if they have write permission on the directory.

**Real-world example:** `/tmp` is writable by everyone, but you don't want user A deleting user B's temp files. The sticky bit prevents this.

```bash
ls -ld /tmp
# drwxrwxrwt 1 root root ...
#          ^
#          't' = sticky bit is set
```

**Setting the sticky bit:**

```bash
# Symbolic
chmod +t /shared/uploads

# Numeric — prepend 1
chmod 1777 /tmp
```

**Verify:**
```bash
ls -ld /shared/uploads
# drwxrwxrwt  ← lowercase 't' = sticky + execute
# drwxrwxrwT  ← uppercase 'T' = sticky but no execute on others
```

---

## Quick Reference — All Special Bits

| Bit | Numeric Prefix | Symbol | Where | Effect |
|-----|---------------|--------|-------|--------|
| SUID | 4 | `s` in owner's x | Executables | Runs as file owner |
| SGID | 2 | `s` in group's x | Executables / Dirs | Runs as file group / new files inherit group |
| Sticky | 1 | `t` in others' x | Directories | Only owner can delete their files |

**Combined example:**

```bash
chmod 4755 file   # SUID + rwxr-xr-x
chmod 2755 dir/   # SGID + rwxr-xr-x
chmod 1777 dir/   # Sticky + rwxrwxrwx
chmod 6755 file   # SUID + SGID + rwxr-xr-x (4+2=6)
```

---

## Putting It All Together — Practical Lab

Try these commands in order on your Linux machine:

```bash
# 1. Create test files
mkdir ~/permissions_lab
cd ~/permissions_lab
touch myfile.txt
mkdir mydir

# 2. Check default permissions
ls -la

# 3. chmod practice
chmod 755 myfile.txt       # standard executable
chmod 600 myfile.txt       # private file
chmod u+x,g-w myfile.txt   # symbolic mode

# 4. Create a script and make it executable
echo '#!/bin/bash' > script.sh
echo 'echo "Hello from script"' >> script.sh
chmod 755 script.sh
./script.sh

# 5. chown practice (requires sudo)
sudo chown root:root myfile.txt
ls -l myfile.txt
sudo chown $USER:$USER myfile.txt  # give it back to yourself

# 6. umask practice
umask          # check current
umask 077      # private mode
touch private.txt
ls -l private.txt    # should be 600
umask 022      # restore default

# 7. SGID on directory
chmod g+s mydir
ls -ld mydir   # look for 's' in group position

# 8. Sticky bit
chmod +t mydir
ls -ld mydir   # look for 't' in others position
```

---

## Common Mistakes to Avoid

| Mistake | Problem | Fix |
|---------|---------|-----|
| `chmod 777` on everything | Huge security risk | Use `755` for dirs, `644` for files |
| SUID on shell scripts | Ignored by Linux kernel (by design) | SUID only works on compiled binaries |
| Forgetting `-R` with chown | Only changes top-level, not contents | Use `chown -R user:group dir/` |
| Uppercase `S` or `T` in ls output | Special bit set but execute bit missing | Add execute: `chmod +x file` |

---

## Summary Cheatsheet

```
chmod 755 file       → rwxr-xr-x  (owner full, rest read+exec)
chmod 644 file       → rw-r--r--  (owner read+write, rest read)
chmod 600 file       → rw-------  (owner only)
chown user:group file → change owner and group
chgrp group file     → change group only
umask 022            → new files get 644, new dirs get 755
chmod u+s file       → set SUID
chmod g+s dir/       → set SGID (group inheritance)
chmod +t dir/        → set sticky bit (delete protection)
```

---

*Next topic: Process Management — ps, top, kill, jobs, fg, bg*
