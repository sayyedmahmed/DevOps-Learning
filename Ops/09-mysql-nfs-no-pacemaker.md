# 08 - MySQL on NFS Without Pacemaker (AlmaLinux 10) - Beginner Edition

> **Prerequisites:** [01 - Hyper-V VM Setup](01-hyperv-vm-setup.md). The NFS basics are explained in [07 - NFS Server Setup](07-nfs-server-setup.md).
>
> **Status:** written from documentation and not tested on AlmaLinux 10. Package and service names may differ slightly.

## Overview

You will store MySQL's data on an **NFS server**, so the data is separate from the database server. You will then add a **second database server** and move MySQL between the two **by hand**. There is no cluster software and no automatic failover.

```
     db1 (10.10.10.101)         db2 (10.10.10.102)
     MySQL runs on ONE          MySQL runs on ONE
     node at a time             node at a time
              \                    /
               \__________________/
                        |
                nfs1 (10.10.10.50)
              MySQL files live here
```

| VM | Role | `eth1` IP |
|---|---|---|
| `nfs1` | NFS storage server | `10.10.10.50` |
| `db1` | Database server 1 | `10.10.10.101` (your VM from guide 01) |
| `db2` | Database server 2 (Part D) | `10.10.10.102` |

## The Golden Rule

> **MySQL must run on only ONE node at a time.** Two MySQL servers using the same files at once **corrupt the data**. Without Pacemaker, **you** are the one who enforces this. Always stop MySQL on one node **before** starting it on the other.

## What You Get and What You Don't

| You get | You don't get |
|---|---|
| Data stored safely on a separate server | Automatic failover |
| Easy manual switch to another node | A floating IP (clients must use the right node's IP) |
| A simple setup to learn from | Protection if `nfs1` dies (it is a single point of failure) |

## How to Read This Guide

- Run commands **one at a time**. Press **Enter**, read the result, then continue.
- Every step has a **Run on:** label. Check which VM you are on.
- Run everything as `root`.
- Passwords here (`RootPass#123`, `LabPass#123`) are **lab only**.

---

# Part A - NFS Server (`nfs1`)

Create `nfs1` by cloning your VM (see Appendix A of `P1-mysql-ha-nfs-pacemaker.md`), with hostname `nfs1` and `eth1` = `10.10.10.50`.

## Step 1 - Install the NFS Tools

**Run on: nfs1**

```bash
dnf install -y nfs-utils
```

## Step 2 - (Recommended) Add a Separate Data Disk

Skip to Step 3 if you want to use the OS disk.

1. Hyper-V Manager → `nfs1` → **Settings** → **SCSI Controller** → **Hard Drive** → **Add** → **New** → VHDX, dynamically expanding, **10 GB**.

**Run on: nfs1**

Find the new empty disk (about `10G`, normally `sdb`):

```bash
lsblk
```

> **Careful:** the next command erases the disk. Never use `sda`.

```bash
mkfs.xfs /dev/sdb
```

```bash
mkdir -p /srv/nfs
```

```bash
blkid /dev/sdb
```

Copy the UUID. **Replace `PASTE-UUID-HERE`**:

```bash
echo "UUID=PASTE-UUID-HERE /srv/nfs xfs defaults 0 0" >> /etc/fstab
```

```bash
systemctl daemon-reload
```

```bash
mount -a
```

```bash
df -h /srv/nfs
```

## Step 3 - Create the Database Folder

**Run on: nfs1**

```bash
mkdir -p /srv/nfs/mysql
```

MySQL runs as user ID **27**. `nfs1` has no such user, so use the number:

```bash
chown 27:27 /srv/nfs/mysql
```

```bash
chmod 750 /srv/nfs/mysql
```

## Step 4 - Export the Folder

**Run on: nfs1**

```bash
echo "/srv/nfs/mysql 10.10.10.0/24(rw,sync,no_subtree_check)" > /etc/exports
```

```bash
exportfs -rav
```

Expected: `exporting 10.10.10.0/24:/srv/nfs/mysql`.

## Step 5 - Allow SELinux, Start NFS, Open the Firewall

**Run on: nfs1**

```bash
setsebool -P nfs_export_all_rw on
```

```bash
systemctl enable --now nfs-server
```

```bash
firewall-cmd --permanent --add-service=nfs
```

```bash
firewall-cmd --reload
```

Check:

```bash
exportfs -v
```

---

# Part B - MySQL on `db1`

## Step 6 - Install MySQL and NFS Tools

**Run on: db1**

```bash
dnf install -y mysql-server nfs-utils
```

```bash
mysql --version
```

> If `dnf` says "no match", run `dnf search mysql-server` and use the name it shows.

Open the MySQL port:

```bash
firewall-cmd --permanent --add-service=mysql
```

```bash
firewall-cmd --reload
```

## Step 7 - Check SELinux

**Run on: db1**

```bash
getenforce
```

If it says `Enforcing`, SELinux may block MySQL from using NFS. For this lab:

```bash
setenforce 0
```

(For a permanent change, see the SELinux section in guide 01.)

## Step 8 - Test the NFS Share

**Run on: db1**

```bash
ping -c 3 10.10.10.50
```

```bash
mount -t nfs4 10.10.10.50:/srv/nfs/mysql /mnt
```

Write a file **as the mysql user** (the folder belongs to user 27):

```bash
runuser -u mysql -- touch /mnt/testfile
```

```bash
runuser -u mysql -- ls -l /mnt
```

Expected: `testfile` is listed. Clean up:

```bash
runuser -u mysql -- rm /mnt/testfile
```

```bash
umount /mnt
```

> **Why `runuser`?** NFS treats `root` on the client as a harmless user ("root squash"), so even `root` cannot read the MySQL folder. Only the `mysql` user (ID 27) can.

## Step 9 - Mount the Share as MySQL's Data Folder

**Run on: db1**

```bash
mount -t nfs4 -o hard,vers=4.2 10.10.10.50:/srv/nfs/mysql /var/lib/mysql
```

| Option | Meaning |
|---|---|
| `hard` | If NFS is unreachable, wait instead of failing. Protects data |
| `vers=4.2` | Use NFS version 4.2 |

Check:

```bash
mount | grep mysql
```

## Step 10 - Start MySQL

**Run on: db1**

```bash
systemctl start mysqld
```

The first start builds the database, so wait up to a minute.

```bash
systemctl status mysqld
```

Look for `active (running)`. Press `q`.

## Step 11 - Secure MySQL and Create a Test Database

**Run on: db1**

```bash
mysql -u root
```

At the `mysql>` prompt, type these one at a time:

```sql
ALTER USER 'root'@'localhost' IDENTIFIED BY 'RootPass#123';
```

```sql
CREATE DATABASE labdb;
```

```sql
CREATE USER 'labuser'@'10.10.10.%' IDENTIFIED BY 'LabPass#123';
```

```sql
GRANT ALL PRIVILEGES ON labdb.* TO 'labuser'@'10.10.10.%';
```

```sql
CREATE TABLE labdb.notes (id INT AUTO_INCREMENT PRIMARY KEY, msg VARCHAR(100));
```

```sql
INSERT INTO labdb.notes (msg) VALUES ('Hello from db1');
```

```sql
SELECT * FROM labdb.notes;
```

```sql
EXIT;
```

> **Asked for a password at the first login?** A temporary one was generated: `grep -i "temporary password" /var/log/mysqld.log`. Log in with it, then run the `ALTER USER` line.

## Step 12 - See Where the Data Lives

**Run on: db1**

```bash
runuser -u mysql -- ls /var/lib/mysql
```

**Run on: nfs1**

```bash
ls /srv/nfs/mysql
```

Same files. They are stored on `nfs1`.

## Step 13 - Connect from Another Machine

**Run on: nfs1** (as a test client)

```bash
dnf install -y mysql
```

```bash
mysql -h 10.10.10.101 -u labuser -p labdb -e "SELECT * FROM notes;"
```

Enter `LabPass#123`. You should see your row.

## Step 14 - Prove the Data Survives a Restart

**Run on: db1**

```bash
systemctl stop mysqld
```

```bash
umount /var/lib/mysql
```

```bash
ls /var/lib/mysql
```

Expected: empty. The data is not on `db1`. Mount and start again:

```bash
mount -t nfs4 -o hard,vers=4.2 10.10.10.50:/srv/nfs/mysql /var/lib/mysql
```

```bash
systemctl start mysqld
```

```bash
mysql -u root -p -e "SELECT * FROM labdb.notes;"
```

The row is still there.

---

# Part C - Create `db2`

## Step 15 - Prepare `db1`

**Run on: db1**

```bash
systemctl stop mysqld
```

```bash
umount /var/lib/mysql
```

```bash
systemctl disable mysqld
```

Then shut it down:

```bash
poweroff
```

## Step 16 - Clone `db1` into `db2`

Follow **Appendix A** of `P1-mysql-ha-nfs-pacemaker.md` with:

- New VM name: `db2`
- New hostname: `db2`
- New `eth1` IP: `10.10.10.102`

Then start `db1` again. `db2` already has MySQL, the NFS tools and the firewall rule, because it is a copy.

## Step 17 - Let the VMs Find Each Other by Name

**Run on: db1, db2 and nfs1** (all three lines on each VM, once)

```bash
echo "10.10.10.50 nfs1" >> /etc/hosts
```

```bash
echo "10.10.10.101 db1" >> /etc/hosts
```

```bash
echo "10.10.10.102 db2" >> /etc/hosts
```

Test from `db1`:

```bash
ping -c 2 db2
```

```bash
ping -c 2 nfs1
```

---

# Part D - Move MySQL Between Nodes by Hand

Currently MySQL is stopped everywhere. We will run it on `db1`, then move it to `db2`.

## Step 18 - Run MySQL on `db1`

**Run on: db1**

```bash
mount -t nfs4 -o hard,vers=4.2 10.10.10.50:/srv/nfs/mysql /var/lib/mysql
```

```bash
systemctl start mysqld
```

```bash
mysql -u root -p -e "INSERT INTO labdb.notes (msg) VALUES ('Written on db1');"
```

## Step 19 - Switch to `db2`

**Always stop the old node first.**

**Run on: db1**

```bash
systemctl stop mysqld
```

```bash
umount /var/lib/mysql
```

**Run on: db2**

```bash
mount -t nfs4 -o hard,vers=4.2 10.10.10.50:/srv/nfs/mysql /var/lib/mysql
```

```bash
systemctl start mysqld
```

```bash
mysql -u root -p -e "SELECT * FROM labdb.notes;"
```

Expected: **all rows**, including `Written on db1`. Add one more:

```bash
mysql -u root -p -e "INSERT INTO labdb.notes (msg) VALUES ('Written on db2');"
```

## Step 20 - Switch Back to `db1`

**Run on: db2**

```bash
systemctl stop mysqld
```

```bash
umount /var/lib/mysql
```

Then repeat Step 18's mount and start on `db1`. Both rows are still there.

## Step 21 - Simulate a Crash

This is what happens without automatic failover.

1. With MySQL running on `db1`, in **Hyper-V Manager** right-click `db1` → **Turn Off**.
2. MySQL is now down. Nothing moves by itself.
3. On `db2`, mount and start MySQL as in Step 19. Skip the `db1` part, because it is already off.

**Run on: db2**

```bash
mount -t nfs4 -o hard,vers=4.2 10.10.10.50:/srv/nfs/mysql /var/lib/mysql
```

```bash
systemctl start mysqld
```

> **If MySQL fails with `Unable to lock ./ibdata1`:** NFS still thinks the crashed `db1` holds the lock. Wait about 2 minutes and run `systemctl start mysqld` again.

Your data is safe, but **you had to do the recovery yourself**. Pacemaker would do this automatically (see project P1).

---

# Part E - (Optional) Helper Scripts

Typing mount and start commands every time is error-prone. These small scripts do it and add a safety check.

## Step 22 - Create the Start Script

**Run on: db1 and db2**

```bash
vi /root/db-start.sh
```

Press `i`, paste this, press `Esc`, then type `:wq` and Enter:

```bash
#!/bin/bash
# Usage: db-start.sh OTHER_NODE_NAME   (db2 on db1, db1 on db2)
OTHER="$1"
if timeout 3 bash -c "echo > /dev/tcp/$OTHER/3306" 2>/dev/null; then
  echo "STOP: MySQL is already answering on $OTHER. Stop it there first."
  exit 1
fi
mount | grep -q " /var/lib/mysql " || mount -t nfs4 -o hard,vers=4.2 10.10.10.50:/srv/nfs/mysql /var/lib/mysql
systemctl start mysqld
echo "MySQL started on $(hostname)."
```

## Step 23 - Create the Stop Script

**Run on: db1 and db2**

```bash
vi /root/db-stop.sh
```

```bash
#!/bin/bash
systemctl stop mysqld
umount /var/lib/mysql
echo "MySQL stopped on $(hostname)."
```

Make both executable:

```bash
chmod +x /root/db-start.sh /root/db-stop.sh
```

## Step 24 - Use Them

**Run on: db1**

```bash
/root/db-start.sh db2
```

To move to `db2`:

```bash
/root/db-stop.sh
```

**Run on: db2**

```bash
/root/db-start.sh db1
```

The check refuses to start MySQL if the other node still answers on port 3306. It cannot protect you if the other node is crashed but still holds the files, or if the check itself is blocked by a firewall, so stay careful.

---

# Limits of This Setup

| Limit | What it means |
|---|---|
| No automatic failover | You must notice the problem and run the commands |
| Human error | Starting MySQL on both nodes corrupts data |
| No floating IP | Clients must be told to use `db2`'s IP after a switch |
| `nfs1` is a single point of failure | If it dies, no database node can work |
| NFS is not ideal for databases | It works for a lab but is slower and more fragile than local disks |

**Next step:** [P1 - MySQL HA Lab](P1-mysql-ha-nfs-pacemaker.md) adds Pacemaker, which automates the switch and adds a floating IP.

---

# Command Cheat Sheet

| Task | Command | Run on |
|---|---|---|
| Is MySQL running? | `systemctl is-active mysqld` | db node |
| Is the share mounted? | `mount | grep mysql` | db node |
| What does the server share? | `exportfs -v` | nfs1 |
| MySQL error log | `tail -n 50 /var/log/mysqld.log` | db node |
| Start MySQL here | `/root/db-start.sh OTHER` | db node |
| Stop MySQL here | `/root/db-stop.sh` | db node |
| Backup | `mysqldump -u root -p --all-databases > /root/backup.sql` | active node |

---

# Troubleshooting

| Problem | Likely cause | Fix |
|---|---|---|
| `mount: access denied` | Export or firewall wrong | On nfs1: `exportfs -v`; redo Steps 4 and 5 |
| `Connection timed out` mounting | Cannot reach nfs1 | `ping nfs1`; check the `nfs` firewall service |
| `Permission denied` on the share (as root) | Root squash | Normal. Use `runuser -u mysql -- ...` |
| `mysqld` will not start | SELinux, permissions or lock | `tail -n 50 /var/log/mysqld.log`; `getenforce` |
| `Unable to lock ./ibdata1` | Old node's NFS lock still active | Wait 2 minutes, start again |
| Cannot log in as root | Password not set or temporary password | See the note in Step 11 |
| `Host is not allowed to connect` | Client not in `10.10.10.%` | Connect from a `10.10.10.x` machine |
| Connection refused from a client | Firewall or MySQL stopped | `firewall-cmd --list-services`; `systemctl status mysqld` |
| MySQL running on both nodes | Forgot to stop one | Stop one **now**: `systemctl stop mysqld` |

---

# Checklist

- [ ] `nfs1` exports `/srv/nfs/mysql` (owner ID 27) to `10.10.10.0/24`
- [ ] `db1` mounts the share and runs MySQL on it
- [ ] `labdb.notes` data survives stop, unmount, mount, start
- [ ] `db2` cloned with its own hostname, IP and machine ID
- [ ] `mysqld` is **disabled** at boot on both DB nodes
- [ ] You moved MySQL from `db1` to `db2` and back with all data intact
- [ ] You simulated a crash and recovered by hand
- [ ] (Optional) Helper scripts work
- [ ] You understand why this needs Pacemaker for automation
