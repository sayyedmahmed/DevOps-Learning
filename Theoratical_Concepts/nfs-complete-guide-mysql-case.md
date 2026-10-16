# NFS Beginner Guide (Linux) + MySQL Data Directory Case Study

**Contents**

1. What is NFS?
2. Do folder names have to match?
3. Permissions, UID and GID (the most important part)
4. Sharing with a group
5. Setup steps
6. The two layers of permissions
7. Common export options
8. Troubleshooting
9. Quick summary
10. **Case study: moving the MySQL data folder to NFS (the issue we faced and the fix)**
11. **Final checklist**

---

## 1. What is NFS?

**NFS (Network File System)** lets one machine (the **server**) share a folder over the network. Other machines (the **clients**) attach that folder to themselves and use it like a normal local folder.

```
   NFS SERVER                          NFS CLIENT
 +----------------+                  +------------------+
 | /srv/shared    |  <--- network -> | /mnt/data        |
 | (real files)   |     port 2049    | (mount point)    |
 +----------------+                  +------------------+
   "exports" folder                    "mounts" folder
```

Words to know:

| Word | Meaning |
|---|---|
| **Server** | The machine that owns the real files |
| **Client** | The machine that uses the files over the network |
| **Export** | A folder the server offers to share (listed in `/etc/exports`) |
| **Mount** | Attaching the server's folder to a folder on the client |
| **Mount point** | The local folder on the client where the share appears |

---

## 2. Do folder names have to match?

**No.** The server folder and the client mount point can have completely different names.

```
Server:  /srv/shared      <-- real folder
Client:  /mnt/anything    <-- any empty folder you create
```

```bash
sudo mount 192.168.1.10:/srv/shared /mnt/anything
```

---

## 3. Permissions: the most important part

### 3.1 Linux permission basics

```
 -rwxr-x---  1  alice  devs  4096  file.txt
 |\_/\_/\_/     |      |
 | |  |  |      |      +-- GROUP owner
 | |  |  |      +--------- USER (owner)
 | |  |  +---------------- OTHERS  (---)
 | |  +------------------- GROUP   (r-x)
 | +---------------------- OWNER   (rwx)
 +------------------------ file type (- file, d directory)

 r = read (4)   w = write (2)   x = execute / enter folder (1)

 rwx r-x ---  =  7 5 0  =  chmod 750
```

### 3.2 UID and GID: do they have to match?

**YES. This is the number one thing to understand about NFS.**

NFS (default security `sec=sys`) does **not** send usernames. It sends **numbers**:

- **UID** = user ID number
- **GID** = group ID number

The server checks permissions using those numbers only.

> Names do not matter. **Numbers must match** on server and client.

```
 CLIENT                         SERVER
 user: bob  UID=1001   ----->   sees UID 1001
                                 |
                                 +--> on the server, UID 1001 = "alice"
                                      so bob's files appear owned by alice!
```

**Good (matching):**
```
 Server: alice  UID=1001 GID=1001
 Client: alice  UID=1001 GID=1001   --> works, same person
```

**Bad (mismatch):**
```
 Server: alice  UID=1001
 Client: alice  UID=1002   --> server sees 1002, a different user
                              --> "Permission denied" or wrong owner
```

Check your numbers on **both** machines:
```bash
id alice
# uid=1001(alice) gid=1001(alice) groups=1001(alice),1005(devs)
```

> If `ls -l` shows a bare **number** (like `27`) instead of a name, it means that machine has no user with that number. That is a clue that the numbers do not match.

---

## 4. Sharing with a group (recommended approach)

Goal: several users on different machines share one folder.

### Step 1: Create the SAME group (same GID) on server and client
```bash
# run on BOTH machines
sudo groupadd -g 2000 nfsgroup
```

### Step 2: Create or adjust users with the SAME UID, and add them to the group
```bash
# new user, run on BOTH machines
sudo useradd -m -u 1500 -G nfsgroup alice

# existing user: add to the group
sudo usermod -aG nfsgroup alice
```

> Changing an existing user's UID: `sudo usermod -u 1500 alice`, then fix their file ownership with `chown`. Do this carefully.

### Step 3: Set up the shared folder on the SERVER
```bash
sudo mkdir -p /srv/shared
sudo chown root:nfsgroup /srv/shared
sudo chmod 2770 /srv/shared
```

`2770` explained:
```
 2   7   7   0
 |   |   |   +-- others: no access
 |   |   +------ group:  rwx
 |   +---------- owner:  rwx
 +-------------- setgid: new files inherit the folder's group (nfsgroup)
```

### Final picture
```
        SERVER                              CLIENT
  group nfsgroup  GID 2000            group nfsgroup  GID 2000
  alice  UID 1500 (in nfsgroup)       alice  UID 1500 (in nfsgroup)
        |                                    |
   /srv/shared  root:nfsgroup 2770  <--> /mnt/data
```

---

## 5. Setup steps

### On the SERVER (Ubuntu/Debian)
```bash
sudo apt update
sudo apt install nfs-kernel-server
sudo nano /etc/exports
```
Add a line:
```
/srv/shared  192.168.1.0/24(rw,sync,no_subtree_check)
```
Apply:
```bash
sudo exportfs -ra        # reload exports
sudo exportfs -v         # verify
sudo systemctl enable --now nfs-kernel-server
```
Firewall (if used):
```bash
sudo ufw allow from 192.168.1.0/24 to any port 2049
```

### On the SERVER (RHEL / CentOS / Rocky)
```bash
sudo dnf install nfs-utils
sudo systemctl enable --now nfs-server
sudo firewall-cmd --permanent --add-service=nfs
sudo firewall-cmd --reload
```

### On the CLIENT
```bash
sudo apt install nfs-common        # RHEL family: sudo dnf install nfs-utils
sudo mkdir -p /mnt/data            # the mount point MUST exist first
sudo mount 192.168.1.10:/srv/shared /mnt/data
df -h                              # confirm it is mounted
```

### Make it permanent (survives reboot)
Add to `/etc/fstab` on the client:
```
192.168.1.10:/srv/shared  /mnt/data  nfs  defaults,_netdev  0  0
```
Test it: `sudo mount -a`

---

## 6. Two layers of permissions

Access must pass **both** layers. The stricter one wins.

```
 Client request
      |
      v
 [Layer 1] /etc/exports   -> allowed IP? rw or ro? squash rules?
      |
      v
 [Layer 2] File permissions on server (owner/group/others + UID/GID)
      |
      v
   ACCESS GRANTED / DENIED
```

---

## 7. Common export options

| Option | Meaning |
|---|---|
| `rw` / `ro` | Read-write / read-only |
| `sync` | Write to disk before replying (safer) |
| `no_subtree_check` | Avoids subtle issues, recommended |
| `root_squash` | **Default.** Client root becomes `nobody` on the server (safe) |
| `no_root_squash` | Client root stays root on the server (risky) |
| `all_squash` | Everyone is mapped to `nobody` (or to `anonuid`/`anongid`) |
| `anonuid=` `anongid=` | UID/GID used for squashed users |

### Shortcut when you do not want to match UIDs
Map every client user to one server user:
```
/srv/shared  192.168.1.0/24(rw,sync,all_squash,anonuid=1500,anongid=2000)
```
```
 Any client user  -->  appears as UID 1500 / GID 2000 on the server
```
Simple, but you lose per-user ownership.

---

## 8. Troubleshooting

| Problem | Likely cause / fix |
|---|---|
| `Permission denied` | UID/GID mismatch, or permissions too strict on the server folder |
| Files show owner as a number (e.g. `27`) | That UID has no name on this machine; make UIDs match |
| Files show `nobody:nogroup` | Squashing (`root_squash` / `all_squash`) or ID mapping issue |
| Cannot mount / timeout | Firewall, server not running, or wrong IP/path |
| `access denied by server` | Client IP not allowed in `/etc/exports` |
| `mount point does not exist` | Create the local folder first with `mkdir -p` |
| `Stale file handle` | Folder was deleted or re-exported; unmount and mount again |
| Root cannot write | `root_squash` is working as intended; use a normal user |

Useful commands:
```bash
showmount -e SERVER_IP      # list exports (run on client)
exportfs -v                 # show active exports (run on server)
id username                 # check UID/GID
mount | grep nfs            # see mounted NFS shares
ls -ln /path                # show owner/group as NUMBERS
```

---

## 9. Quick summary

- Folder **names** do **not** need to match.
- **UID and GID numbers must match** on server and client (user/group names do not matter).
- Use a **shared group with the same GID** and `chmod 2770` for team sharing.
- Permissions are checked in **two layers**: `/etc/exports` and file permissions.
- Keep `root_squash` on (default) for safety.
- For many machines, central user management (LDAP/FreeIPA) keeps UIDs consistent.
- Default `sec=sys` trusts the client's claimed UID, so only export to trusted networks. Kerberos (`sec=krb5`) is the secure option.

---

# 10. Case Study: Moving the MySQL Data Folder to NFS

## 10.1 The goal

Store the MySQL database files on an **NFS server** instead of the MySQL machine's own disk.

```
 CLIENT (MySQL machine)                      NFS SERVER (10.10.10.200)
 +--------------------------+                +--------------------------+
 | MySQL (runs as "mysql")  |                |                          |
 |    |                     |                |                          |
 |    v                     |                |                          |
 | /var/lib/mysql_data      |<==== NFS ====> | /srv/nfs-dbstorage       |
 |  (just a window)         |   (network)    |  (REAL data stored here) |
 +--------------------------+                +--------------------------+
```

MySQL thinks it is writing to a local folder, but the data actually lands on the server.

## 10.2 What we did

1. Changed the MySQL data folder from `/var/lib/mysql` to `/var/lib/mysql_data`.
2. Added this line to `/etc/fstab` on the client:
   ```
   10.10.10.200:/srv/nfs-dbstorage/ /var/lib/mysql_data nfs4 defaults,_netdev 0 0
   ```
3. Ran `mount -a`.

## 10.3 What went wrong

**Before the mount:**
```
drwxr-x---.  8 mysql  mysql    4096 Oct 16 00:51 mysql
drwxr-x---.  8 mysql  mysql    4096 Oct 16 00:51 mysql_data
```

**After the mount:**
```
drwxr-x---.  8 mysql  mysql    4096 Oct 16 00:51 mysql
drwxrws---.  8     27 nfsusers 4096 Oct 16 00:57 mysql_data
```

Symptoms:
- The owner of `mysql_data` changed by itself.
- MySQL could no longer access the folder, so the database was unusable.

## 10.4 Why it happened (two separate things)

### Reason 1: Mounting covers (hides) the local folder

A mount does **not** copy or merge anything. It places a "window" over the folder. Through the window you see the **server's** folder, not your local one.

```
 BEFORE mount                      AFTER mount
 ------------                      -----------
 /var/lib/mysql_data               /var/lib/mysql_data
   [your old local data]             +----------------------+
                                     |  WINDOW to server    |  <--- you see this now
                                     |  /srv/nfs-dbstorage  |
                                     +----------------------+
                                     [your old data is still
                                      underneath, just hidden]
```

- Your old data was **not deleted**. It is only hidden.
- `umount /var/lib/mysql_data` removes the window and your old data appears again.
- That is why the ownership "changed": you were now looking at the **server's** folder, which has its own owner.

### Reason 2: UID/GID mismatch

The server's folder looked like this:
```
drwxrws---.  8     27 nfsusers  mysql_data
                   |      |
                   |      +-- group "nfsusers" (name known on client = GID matched)
                   +--------- owner UID 27 (shown as a bare number, so the
                              client has NO user with UID 27)
```

Meaning of `drwxrws---`:
```
 d | rwx | rws | ---
 |    |     |     +-- others: no access
 |    |     +-------- group (nfsusers): rwx + setgid (the "s")
 |    +-------------- owner (UID 27): rwx
 +------------------- directory
```

The client's `mysql` user has a **different UID** than 27, and is **not** in the group `nfsusers`. So the server treats it as "others", which have no access.

```
 CLIENT                                   SERVER
 MySQL runs as user "mysql"               Folder owner: UID 27
 (its UID is NOT 27)                      Group: nfsusers
        |                                        |
        |  "Hi, I'm UID 1000, let me in"         |
        +--------------------------------------->|
        |                                        |  Server thinks:
        |  "Denied!"                             |  "Not the owner (27),
        |<---------------------------------------+   not in group nfsusers.
                                                    No access."
```

## 10.5 The fix

The goal: make the numbers match so the server recognises MySQL.

```
 CLIENT                                   SERVER
 mysql  UID=27  GID=27                    folder owner UID=27  GID=27
        |                                        |
        |  "Hi, I'm UID 27"                      |
        +--------------------------------------->|
        |                                        |  "27 = owner. Come in."
        |  "Access granted"                      |
        |<---------------------------------------+
```

### Step 1: Find the real numbers on the CLIENT

```bash
sudo systemctl stop mysqld        # or mariadb
id mysql
# example output: uid=27(mysql) gid=27(mysql) groups=27(mysql)
```

> **Why 27?** 27 is only the default `mysql` UID on RHEL/CentOS/Rocky. It is not special. **Use whatever number your client shows.** What matters is that both machines use the same number.

### Step 2: Fix the SERVER (pick one way)

**Way A: Create a matching `mysql` user on the server**

First check the number is free and the group exists:
```bash
getent group mysql
getent passwd 27            # replace 27 with the client's UID; should print nothing
```

Create the user using the client's numbers:
```bash
# If the server's "mysql" group already has the same GID as the client:
sudo useradd -u 27 -g mysql -M -s /sbin/nologin mysql

# If the GID is different, fix the group first:
sudo groupmod -g 27 mysql
sudo useradd -u 27 -g mysql -M -s /sbin/nologin mysql
```

Meaning of the options:

| Option | Meaning |
|---|---|
| `-u 27` | Use this exact UID |
| `-g mysql` | Use the existing group named mysql |
| `-M` | Do not create a home folder |
| `-s /sbin/nologin` | Nobody can log in as this user |

**About the error we saw:**
```
useradd: group mysql exists - if you want to add this user to that group, use -g.
```
This is not a real failure. The group `mysql` already exists, so `useradd` wants you to say `-g mysql` explicitly. Also, do **not** use UID `1000`; that is usually the first normal human user. Use the client's UID instead.

`rpcuser` (UID 29) seen in `/etc/passwd` is only for NFS's own internal services. MySQL does not use it.

Then give the share to that user:
```bash
sudo chown -R mysql:mysql /srv/nfs-dbstorage
sudo chmod 750 /srv/nfs-dbstorage
```

**Way B: No user needed on the server (use numbers only)**

NFS only cares about numbers, so you can skip creating the user:
```bash
sudo chown -R 27:27 /srv/nfs-dbstorage      # use the client's UID:GID
sudo chmod 750 /srv/nfs-dbstorage
```
The server will show bare numbers instead of names, but it works the same.

**Way C: Add the client's `mysql` to the share's existing group**

If the folder already allows the group (like `nfsusers` with `rwx` and setgid), simply do this on the client:
```bash
sudo usermod -aG nfsusers mysql
```
This only works if the group's GID is the same on both machines.

### Step 3: Make sure the mount point exists on the CLIENT

```bash
sudo mkdir -p /var/lib/mysql_data
```

If the folder is missing, `mount` fails with `mount point does not exist`.

| State of the local folder | What happens on mount |
|---|---|
| Does not exist | Mount **fails** |
| Exists and is **empty** | Mount works, you see the server's files (cleanest) |
| Exists **with data** | Mount works, but the local data is **hidden** |

### Step 4: SELinux (the `.` at the end of `drwxr-x---.` means SELinux is on)

If MySQL still gets "permission denied" after fixing the numbers, SELinux may be blocking it. Use this fstab line on the client:
```
10.10.10.200:/srv/nfs-dbstorage/ /var/lib/mysql_data nfs4 defaults,_netdev,context="system_u:object_r:mysqld_db_t:s0" 0 0
```
Then:
```bash
sudo umount /var/lib/mysql_data
sudo mount -a
```

### Step 5: Point MySQL to the new folder

Edit `/etc/my.cnf` (or `/etc/my.cnf.d/mysql-server.cnf`):
```
[mysqld]
datadir=/var/lib/mysql_data
```

## 10.6 Getting the database onto the NFS share

**A mount alone never moves data.** Pick one option.

### Option A: Fresh start (empty or test database)

```bash
# 1. Stop MySQL
sudo systemctl stop mysqld

# 2. Make sure the client mysql user can access the share (see 10.5)

# 3. Mount and confirm the folder is empty
sudo mount -a
ls -ln /var/lib/mysql_data

# 4. Set datadir in /etc/my.cnf  (datadir=/var/lib/mysql_data)

# 5. Create a new database inside the share
sudo mysqld --initialize --user=mysql        # MariaDB: sudo mysql_install_db --user=mysql
sudo systemctl start mysqld

# 6. MySQL 8 temporary root password:
sudo grep password /var/log/mysqld.log
```

The folder must be **empty** for `--initialize` to work.

### Option B: Keep existing data (without rsync)

Use a dump and import. It is the safest way to move a database.

```bash
# BEFORE changing anything, while the OLD data folder is still active:
mysqldump -u root -p --all-databases --routines --events > /root/backup.sql

# Then do Option A above. After MySQL starts on the NFS share:
mysql -u root -p < /root/backup.sql
```

### Option C: Copy files with rsync (for reference)

```bash
sudo systemctl stop mysqld
sudo umount /var/lib/mysql_data
sudo mkdir -p /mnt/tmp
sudo mount -t nfs4 10.10.10.200:/srv/nfs-dbstorage /mnt/tmp
sudo rsync -aAXv /var/lib/mysql_data/ /mnt/tmp/
sudo umount /mnt/tmp
sudo mount -a
sudo systemctl start mysqld
```

## 10.7 If something is still wrong: how to check

```bash
# On the client
id mysql                              # client numbers
ls -ln /var/lib/mysql_data            # owner/group as NUMBERS, compare with id mysql
mount | grep mysql_data               # is the NFS share actually mounted?
journalctl -u mysqld -n 50            # MySQL's error messages
getenforce                            # Enforcing = SELinux is on

# On the server
getent passwd 27                      # who is UID 27 here?
getent group mysql                    # GID of the mysql group here
exportfs -v                           # what is exported, and to whom
```

## 10.8 Important warning

Running a **live database on NFS** can be slower than a local disk, and it can be risky if the network drops, because MySQL may corrupt or stall. It is fine for learning and labs. For production, use local or block storage, or take regular backups with `mysqldump`.

---

# 11. Final Checklist

Before mounting an NFS folder for MySQL, check each item:

- [ ] `id mysql` on the client. Note the UID and GID.
- [ ] The server's shared folder is owned by the **same UID:GID** (`ls -ln` on the server).
- [ ] Folder permissions allow the owner (e.g. `750` or `770`).
- [ ] `/etc/exports` on the server allows the client's IP with `rw`.
- [ ] Firewall allows NFS (port 2049).
- [ ] The local mount-point folder exists and is **empty** (or backed up).
- [ ] The `fstab` line is correct, with `_netdev` and (if SELinux is on) the `context=` option.
- [ ] `datadir=` in `/etc/my.cnf` points to the mount point.
- [ ] MySQL was **stopped** before changing anything, and a `mysqldump` backup exists.
- [ ] After mounting, `ls -ln` shows the owner as the **same numbers** as `id mysql`.
- [ ] Start MySQL and check `journalctl -u mysqld`.

## One-line memory aid

```
 NFS checks NUMBERS (UID/GID), not names.
 Mount HIDES the local folder, it does not move data.
 Same numbers on both machines = access works.
```
