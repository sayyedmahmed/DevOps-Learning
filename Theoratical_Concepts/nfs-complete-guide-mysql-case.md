# NFS Complete Guide (Linux) + MySQL Data Directory Case Study

This is a combined, single-file reference for understanding NFS, configuring it safely, and fixing the MySQL data-directory-on-NFS permission problem.

## Contents

1. What is NFS?
2. Do folder names have to match?
3. Permissions, UID and GID — the most important part
4. Sharing with a group — recommended approach
5. Setup steps
6. The two layers of permissions
7. Common export options
8. Troubleshooting
9. Quick summary
10. Case study: moving the MySQL data folder to NFS
11. Final checklist
12. One-line memory aid

---

## 1. What is NFS?

NFS, or Network File System, lets one machine share a folder over the network. Other machines can attach that folder locally and use it like a normal directory.

```text
   NFS SERVER                          NFS CLIENT
 +----------------+                  +------------------+
 | /srv/shared    |  <--- network -> | /mnt/data        |
 | (real files)   |     port 2049    | (mount point)    |
 +----------------+                  +------------------+
   "exports" folder                    "mounts" folder
```

### Words to know

| Word | Meaning |
|---|---|
| Server | The machine that owns the real files |
| Client | The machine that uses the files over the network |
| Export | A folder the server offers to share, listed in `/etc/exports` |
| Mount | Attaching the server folder to a folder on the client |
| Mount point | The local folder on the client where the share appears |

The server exports a folder.  
The client mounts that exported folder onto a local directory.

---

## 2. Do folder names have to match?

No.

The server folder and the client mount point can have completely different names.

Example:

```text
Server:  /srv/shared      <-- real folder
Client:  /mnt/anything    <-- any empty folder you create
```

Mount command:

```bash
sudo mount 192.168.1.10:/srv/shared /mnt/anything
```

The names do not need to match. What matters is that the client can reach the server export and has a local mount point.

---

## 3. Permissions, UID and GID — the most important part

This is the part that causes most NFS problems.

### 3.1 Linux permission basics

Example:

```text
-rwxr-x---  1  alice  devs  4096  file.txt
 |\_/\_/\_/     |      |
 | |  |  |      |      +-- GROUP owner
 | |  |  |      |      +--------- USER owner
 | |  |  +---------------- OTHERS
 | |  +------------------- GROUP
 | +---------------------- OWNER
 +------------------------ file type
```

Meaning:

```text
r = read      = 4
w = write     = 2
x = execute   = 1
```

So:

```text
rwx r-x --- = 7 5 0 = chmod 750
```

Breakdown:

```text
owner: rwx = 7
group: r-x = 5
others: --- = 0
```

---

### 3.2 UID and GID: do they have to match?

Yes.

This is the number one thing to understand about NFS.

By default, NFS uses `sec=sys`. That means NFS does not send usernames. It sends numbers:

```text
UID = user ID number
GID = group ID number
```

The NFS server checks permissions using those numbers only.

Names do not matter.  
Numbers must match on server and client.

Example:

```text
 CLIENT                         SERVER
 user: bob  UID=1001   ----->   sees UID 1001
                                 |
                                 +--> on the server, UID 1001 = "alice"
                                      so bob's files appear owned by alice!
```

Good, matching IDs:

```text
Server: alice  UID=1001 GID=1001
Client: alice  UID=1001 GID=1001   --> works, same person
```

Bad, mismatched IDs:

```text
Server: alice  UID=1001
Client: alice  UID=1002   --> server sees 1002, a different user
                              --> "Permission denied" or wrong owner
```

Check your numbers on both machines:

```bash
id alice
```

Example output:

```text
uid=1001(alice) gid=1001(alice) groups=1001(alice),1005(devs)
```

If `ls -l` shows a bare number instead of a name, that means this machine has no user with that UID.

Example:

```text
drwxr-x--- 2 27 27 4096 Oct 16 00:57 mysql_data
```

That means UID `27` exists on the server, but this client does not have a local user named `27`.

Useful command:

```bash
ls -ln /path
```

This shows owner and group as numbers.

---

## 4. Sharing with a group — recommended approach

Goal: several users on different machines share one folder.

### Step 1: Create the same group with the same GID on server and client

Run on both machines:

```bash
sudo groupadd -g 2000 nfsgroup
```

### Step 2: Create or adjust users with the same UID and add them to the group

For a new user, run on both machines:

```bash
sudo useradd -m -u 1500 -G nfsgroup alice
```

For an existing user:

```bash
sudo usermod -aG nfsgroup alice
```

Changing an existing user UID:

```bash
sudo usermod -u 1500 alice
```

After changing a UID, fix file ownership:

```bash
sudo chown -R alice:nfsgroup /home/alice
```

Do this carefully.

### Step 3: Set up the shared folder on the server

```bash
sudo mkdir -p /srv/shared
sudo chown root:nfsgroup /srv/shared
sudo chmod 2770 /srv/shared
```

### What does `2770` mean?

```text
 2   7   7   0
 |   |   |   +-- others: no access
 |   |   +------ group:  rwx
 |   +---------- owner:  rwx
 +-------------- setgid: new files inherit the folder group
```

The setgid bit, shown as `s` in group permissions, makes new files inside the folder inherit the folder group.

Example:

```text
drwxrws--- 2 root nfsgroup 4096 Oct 16 00:57 shared
```

The `s` in `rws` means setgid is enabled.

### Final picture

```text
        SERVER                              CLIENT
  group nfsgroup  GID 2000            group nfsgroup  GID 2000
  alice  UID 1500 (in nfsgroup)       alice  UID 1500 (in nfsgroup)
        |                                    |
   /srv/shared  root:nfsgroup 2770  <--> /mnt/data
```

---

## 5. Setup steps

### 5.1 On the server — Ubuntu/Debian

Install NFS:

```bash
sudo apt update
sudo apt install nfs-kernel-server
```

Edit exports:

```bash
sudo nano /etc/exports
```

Add a line:

```text
/srv/shared  192.168.1.0/24(rw,sync,no_subtree_check)
```

Apply:

```bash
sudo exportfs -ra
sudo exportfs -v
```

Start NFS:

```bash
sudo systemctl enable --now nfs-kernel-server
```

Firewall, if used:

```bash
sudo ufw allow from 192.168.1.0/24 to any port 2049
```

---

### 5.2 On the server — RHEL / CentOS / Rocky / AlmaLinux

Install NFS:

```bash
sudo dnf install nfs-utils
```

Start and enable NFS:

```bash
sudo systemctl enable --now nfs-server
```

Open firewall:

```bash
sudo firewall-cmd --permanent --add-service=nfs
sudo firewall-cmd --reload
```

Edit exports:

```bash
sudo nano /etc/exports
```

Example:

```text
/srv/shared  192.168.1.0/24(rw,sync,no_subtree_check)
```

Apply:

```bash
sudo exportfs -ra
sudo exportfs -v
```

Allow SELinux NFS sharing if needed:

```bash
sudo setsebool -P nfs_export_all_rw on
```

If the command is missing:

```bash
sudo dnf install -y policycoreutils-python-utils
sudo setsebool -P nfs_export_all_rw on
```

---

### 5.3 On the client

Ubuntu/Debian:

```bash
sudo apt install nfs-common
```

RHEL family:

```bash
sudo dnf install nfs-utils
```

Create the mount point:

```bash
sudo mkdir -p /mnt/data
```

Mount:

```bash
sudo mount 192.168.1.10:/srv/shared /mnt/data
```

Check:

```bash
df -h
mount | grep nfs
```

---

### 5.4 Make the mount permanent

Add to `/etc/fstab` on the client:

```text
192.168.1.10:/srv/shared  /mnt/data  nfs  defaults,_netdev  0  0
```

Test:

```bash
sudo mount -a
```

`_netdev` tells the system this filesystem needs network access.

---

## 6. The two layers of permissions

Access must pass both layers. The stricter one wins.

```text
 Client request
      |
      v
 [Layer 1] /etc/exports
      |
      |  allowed IP?
      |  rw or ro?
      |  squash rules?
      v
 [Layer 2] File permissions on server
      |
      |  owner/group/others
      |  UID/GID matching
      v
   ACCESS GRANTED / DENIED
```

So even if `/etc/exports` allows `rw`, the file permissions on the server can still deny access.

And even if file permissions allow access, `/etc/exports` can still deny it.

---

## 7. Common export options

| Option | Meaning |
|---|---|
| `rw` | Read and write |
| `ro` | Read only |
| `sync` | Write to disk before replying. Safer |
| `async` | Faster but less safe. Avoid for databases |
| `no_subtree_check` | Avoids subtle NFS subtree issues. Recommended |
| `root_squash` | Default. Client root becomes `nobody` on server |
| `no_root_squash` | Client root stays root on server. Risky |
| `all_squash` | Everyone is mapped to `nobody`, or to `anonuid`/`anongid` |
| `anonuid=` | UID used for squashed users |
| `anongid=` | GID used for squashed users |

Example:

```text
/srv/shared  192.168.1.0/24(rw,sync,no_subtree_check)
```

### Shortcut when you do not want to match UIDs

Map every client user to one server user:

```text
/srv/shared  192.168.1.0/24(rw,sync,all_squash,anonuid=1500,anongid=2000)
```

Meaning:

```text
Any client user  -->  appears as UID 1500 / GID 2000 on the server
```

This is simple, but you lose per-user ownership.

---

## 8. Troubleshooting

| Problem | Likely cause / fix |
|---|---|
| Permission denied | UID/GID mismatch, or permissions too strict on the server folder |
| Files show owner as a number, for example `27` | That UID has no name on this machine. Make UIDs match or use numbers consistently |
| Files show `nobody:nogroup` | Squashing, for example `root_squash` or `all_squash`, or ID mapping issue |
| Cannot mount / timeout | Firewall, server not running, wrong IP, wrong path |
| `access denied by server` | Client IP not allowed in `/etc/exports` |
| `mount point does not exist` | Create the local folder first with `mkdir -p` |
| Stale file handle | Folder was deleted or re-exported. Unmount and mount again |
| Root cannot write | `root_squash` is working as intended. Use a normal user or service account |
| MySQL cannot access NFS folder | Usually UID/GID mismatch or SELinux |
| NFS mount hangs | Network issue or NFS server down. `hard` mount will wait |

Useful commands:

```bash
showmount -e SERVER_IP
exportfs -v
id username
mount | grep nfs
ls -ln /path
df -h
systemctl status nfs-server
journalctl -u nfs-server -n 50
```

---

## 9. Quick summary

- Folder names do not need to match.
- UID and GID numbers must match on server and client.
- Usernames and group names do not matter to NFS. Numbers matter.
- Use a shared group with the same GID and `chmod 2770` for team sharing.
- Permissions are checked in two layers: `/etc/exports` and file permissions.
- Keep `root_squash` on by default for safety.
- For many machines, central user management such as LDAP or FreeIPA keeps UIDs consistent.
- Default `sec=sys` trusts the client claimed UID, so only export to trusted networks.
- Kerberos, `sec=krb5`, is the more secure option.
- For databases, use `sync`, `hard`, and careful ownership.

---

# 10. Case Study: Moving the MySQL Data Folder to NFS

## 10.1 The goal

Store the MySQL database files on an NFS server instead of the MySQL machine own disk.

```text
 CLIENT (MySQL machine)                      NFS SERVER (10.10.10.200)
 +--------------------------+                +--------------------------+
 | MySQL (runs as "mysql")  |                |                          |
 |    |                     |                |                          |
 |    v                     |                |                          |
 | /var/lib/mysql_data      |<==== NFS ====> | /srv/nfs-dbstorage       |
 |  (just a window)         |   (network)    |  (REAL data stored here) |
 +--------------------------+                +--------------------------+
```

MySQL thinks it is writing to a local folder, but the data actually lands on the NFS server.

---

## 10.2 What was done

The MySQL data folder was changed from:

```text
/var/lib/mysql
```

to:

```text
/var/lib/mysql_data
```

An `/etc/fstab` line was added on the client:

```text
10.10.10.200:/srv/nfs-dbstorage/ /var/lib/mysql_data nfs4 defaults,_netdev 0 0
```

Then:

```bash
sudo mount -a
```

---

## 10.3 What went wrong

Before the mount:

```text
drwxr-x---.  8 mysql  mysql    4096 Oct 16 00:51 mysql
drwxr-x---.  8 mysql  mysql    4096 Oct 16 00:51 mysql_data
```

After the mount:

```text
drwxr-x---.  8 mysql  mysql    4096 Oct 16 00:51 mysql
drwxrws---.  8     27 nfsusers 4096 Oct 16 00:57 mysql_data
```

Symptoms:

- The owner of `mysql_data` appeared to change by itself.
- MySQL could no longer access the folder.
- The database became unusable.

---

## 10.4 Why it happened

There were two separate things happening.

---

### Reason 1: Mounting hides the local folder

A mount does not copy or merge anything.

It places a window over the folder.

Through that window, you see the server folder, not your local one.

```text
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

Your old data was not deleted. It was only hidden.

To remove the window:

```bash
sudo umount /var/lib/mysql_data
```

Then the old local data appears again.

That is why the ownership seemed to change. You were now looking at the NFS server folder, which has its own owner.

---

### Reason 2: UID/GID mismatch

The server folder looked like this:

```text
drwxrws---.  8     27 nfsusers mysql_data
                   |      |
                   |      +-- group "nfsusers"
                   |
                   +--------- owner UID 27
```

Meaning of:

```text
drwxrws---
```

Breakdown:

```text
 d | rwx | rws | ---
 |    |     |     +-- others: no access
 |    |     +-------- group: rwx + setgid
 |    +-------------- owner: rwx
 +------------------- directory
```

The client MySQL user had a different UID than `27`, and was not in the group `nfsusers`.

So the server treated it as “others”, and others had no access.

```text
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

---

## 10.5 The fix

The goal is to make the numbers match so the server recognises MySQL.

```text
 CLIENT                                   SERVER
 mysql  UID=27  GID=27                    folder owner UID=27  GID=27
        |                                        |
        |  "Hi, I'm UID 27"                      |
        +--------------------------------------->|
        |                                        |  "27 = owner. Come in."
        |  "Access granted"                      |
        |<---------------------------------------+
```

---

### Step 1: Find the real numbers on the client

Stop MySQL first:

```bash
sudo systemctl stop mysqld
```

Or for MariaDB:

```bash
sudo systemctl stop mariadb
```

Check the MySQL user:

```bash
id mysql
```

Example:

```text
uid=27(mysql) gid=27(mysql) groups=27(mysql)
```

Why `27`?

`27` is commonly the default `mysql` UID on RHEL, CentOS, Rocky, AlmaLinux, and similar systems.

It is not magic.

Use whatever number your client shows.

What matters is that both machines use the same number.

---

### Step 2: Fix the server

You have multiple ways.

---

## Way A: Create a matching `mysql` user on the server

First check whether the number is free and whether the group exists:

```bash
getent group mysql
getent passwd 27
```

Replace `27` with the client UID.

If `getent passwd 27` prints nothing, UID `27` is free.

Create the user using the client numbers:

```bash
sudo useradd -u 27 -g mysql -M -s /sbin/nologin mysql
```

Options:

| Option | Meaning |
|---|---|
| `-u 27` | Use this exact UID |
| `-g mysql` | Use the existing group named `mysql` |
| `-M` | Do not create a home folder |
| `-s /sbin/nologin` | Nobody can log in as this user |

If the server `mysql` group has a different GID, fix the group first:

```bash
sudo groupmod -g 27 mysql
sudo useradd -u 27 -g mysql -M -s /sbin/nologin mysql
```

You may see this error:

```text
useradd: group mysql exists - if you want to add this user to that group, use -g.
```

That is not a real failure.

It means the group already exists, so you must explicitly say:

```bash
-g mysql
```

Also, do not use UID `1000` unless that is really the client MySQL UID. UID `1000` is usually the first normal human user.

The user `rpcuser`, often UID `29`, is for NFS internal services. MySQL does not use it.

Then give the share to that user:

```bash
sudo chown -R mysql:mysql /srv/nfs-dbstorage
sudo chmod 750 /srv/nfs-dbstorage
```

---

## Way B: No user needed on the server, use numbers only

NFS only cares about numbers, so you can skip creating the user:

```bash
sudo chown -R 27:27 /srv/nfs-dbstorage
sudo chmod 750 /srv/nfs-dbstorage
```

The server will show bare numbers instead of names, but it works the same.

Example:

```bash
ls -ld /srv/nfs-dbstorage
```

May show:

```text
drwxr-x--- 2 27 27 4096 Oct 16 00:57 /srv/nfs-dbstorage
```

That is fine.

---

## Way C: Add the client MySQL user to the share existing group

If the folder already allows the group, for example `nfsusers` with `rwx` and setgid, you can do this on the client:

```bash
sudo usermod -aG nfsusers mysql
```

This only works if the group GID is the same on both machines.

---

### Step 3: Make sure the mount point exists on the client

```bash
sudo mkdir -p /var/lib/mysql_data
```

If the folder is missing, mount fails with:

```text
mount point does not exist
```

State of the local folder and what happens:

| State of the local folder | What happens on mount |
|---|---|
| Does not exist | Mount fails |
| Exists and is empty | Mount works, you see the server files. Cleanest |
| Exists with data | Mount works, but the local data is hidden |

---

### Step 4: SELinux

If MySQL still gets permission denied after fixing UID/GID, SELinux may be blocking it.

A trailing dot in permissions means SELinux context exists:

```text
drwxr-x---.
```

On the client, you can add a context option to `/etc/fstab`:

```text
10.10.10.200:/srv/nfs-dbstorage/ /var/lib/mysql_data nfs4 defaults,_netdev,context="system_u:object_r:mysqld_db_t:s0" 0 0
```

Then:

```bash
sudo umount /var/lib/mysql_data
sudo mount -a
```

Check:

```bash
ls -Zd /var/lib/mysql_data
```

---

### Step 5: Point MySQL to the new folder

Edit MySQL configuration:

```bash
sudo nano /etc/my.cnf
```

Or on some systems:

```bash
sudo nano /etc/my.cnf.d/mysql-server.cnf
```

Set:

```ini
[mysqld]
datadir=/var/lib/mysql_data
```

Then start MySQL:

```bash
sudo systemctl start mysqld
```

Check:

```bash
sudo systemctl status mysqld
journalctl -u mysqld -n 50
```

---

## 10.6 Getting the database onto the NFS share

A mount alone never moves data.

Pick one option.

---

## Option A: Fresh start, empty or new database

Use this if the database is new or you do not need old data.

### 1. Stop MySQL

```bash
sudo systemctl stop mysqld
```

### 2. Make sure the client MySQL user can access the share

See section 10.5.

### 3. Mount and confirm the folder is empty

```bash
sudo mount -a
ls -ln /var/lib/mysql_data
```

The folder should be empty.

### 4. Set `datadir` in `/etc/my.cnf`

```ini
[mysqld]
datadir=/var/lib/mysql_data
```

### 5. Create a new database inside the share

MySQL 8:

```bash
sudo mysqld --initialize --user=mysql
```

MariaDB:

```bash
sudo mysql_install_db --user=mysql
```

### 6. Start MySQL

```bash
sudo systemctl start mysqld
```

### 7. Find the temporary root password for MySQL 8

```bash
sudo grep password /var/log/mysqld.log
```

Then log in and change the password:

```bash
mysql -u root -p
```

Inside MySQL:

```sql
ALTER USER 'root'@'localhost' IDENTIFIED BY 'NewStrongPassword#123';
```

Important:

The folder must be empty for `--initialize` to work.

---

## Option B: Keep existing data using dump and import

This is the safest way to move a database.

### 1. Before changing anything, while the old data folder is still active

Take a backup:

```bash
mysqldump -u root -p --all-databases --routines --events > /root/backup.sql
```

### 2. Prepare the NFS mount and MySQL datadir

Follow Option A above.

### 3. After MySQL starts on the NFS share, import the backup

```bash
mysql -u root -p < /root/backup.sql
```

This method avoids copying raw MySQL files while the database may be inconsistent.

---

## Option C: Copy files with rsync

Use this only when MySQL is stopped.

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

Explanation:

| Flag | Meaning |
|---|---|
| `-a` | Archive mode, preserves permissions and structure |
| `-A` | Preserve ACLs |
| `-X` | Preserve extended attributes |
| `-v` | Verbose |

Be careful with trailing slashes.

```text
/source/   copies contents of source
/source    copies the source directory itself
```

---

## 10.7 If something is still wrong, how to check

### On the client

```bash
id mysql
ls -ln /var/lib/mysql_data
mount | grep mysql_data
journalctl -u mysqld -n 50
getenforce
```

What to look for:

| Command | What it tells you |
|---|---|
| `id mysql` | The UID/GID MySQL uses on the client |
| `ls -ln /var/lib/mysql_data` | The owner/group numbers of the mounted folder |
| `mount | grep mysql_data` | Whether the NFS share is actually mounted |
| `journalctl -u mysqld` | MySQL startup errors |
| `getenforce` | Whether SELinux is enforcing |

The numbers from `id mysql` must match the numbers from `ls -ln`.

Example:

```bash
id mysql
```

Output:

```text
uid=27(mysql) gid=27(mysql)
```

Then:

```bash
ls -ln /var/lib/mysql_data
```

Should show:

```text
drwxr-x--- 8 27 27 4096 Oct 16 00:57 /var/lib/mysql_data
```

If it shows:

```text
drwxrws--- 8 27 nfsusers 4096 Oct 16 00:57 /var/lib/mysql_data
```

and the client MySQL user is not UID `27` and not in the group with the matching GID, access will fail.

---

### On the server

```bash
getent passwd 27
getent group mysql
exportfs -v
```

What to look for:

| Command | What it tells you |
|---|---|
| `getent passwd 27` | Who UID 27 is on the server |
| `getent group mysql` | The GID of the MySQL group on the server |
| `exportfs -v` | What folders are exported and with what options |

---

## 10.8 Important warning

Running a live database on NFS can be slower than using a local disk.

It can also be risky if the network drops, because MySQL may stall or, in bad cases, corrupt data.

For learning and practice environments, NFS is useful.

For production, prefer:

- local storage,
- block storage,
- clustered filesystems designed for databases,
- database replication,
- or storage with proper HA and fencing.

Always take regular backups with:

```bash
mysqldump
```

or your preferred backup method.

High availability is not backup.

If someone drops a table, it is dropped everywhere.

---

# 11. Final checklist

Before mounting an NFS folder for MySQL, check each item:

- [ ] Run `id mysql` on the client.
- [ ] Note the UID and GID.
- [ ] The server shared folder is owned by the same UID and GID.
- [ ] Check with `ls -ln` on the server.
- [ ] Folder permissions allow the owner, for example `750` or `770`.
- [ ] `/etc/exports` on the server allows the client IP with `rw`.
- [ ] Firewall allows NFS, especially port 2049.
- [ ] The local mount point folder exists.
- [ ] The local mount point is empty, or its contents are backed up.
- [ ] The `/etc/fstab` line is correct.
- [ ] The fstab line includes `_netdev`.
- [ ] If SELinux is enforcing, the correct `context=` option is used.
- [ ] `datadir=` in `/etc/my.cnf` points to the mount point.
- [ ] MySQL was stopped before changing anything.
- [ ] A `mysqldump` backup exists.
- [ ] After mounting, `ls -ln` shows the same owner numbers as `id mysql`.
- [ ] Start MySQL and check:

```bash
journalctl -u mysqld -n 50
```

---

# 12. One-line memory aid

```text
NFS checks NUMBERS, not names.
Mount HIDES the local folder, it does not move data.
Same UID/GID on both machines = access works.
```
