# P1 - MySQL High Availability Lab (NFS + Pacemaker) on Hyper-V

> **Project 1.** Beginner edition. Builds on [01 - Hyper-V VM Setup](01-hyperv-vm-setup.md) and [06 - Pacemaker Cluster](06-pacemaker-cluster.md).
>
> **Status:** written from documentation and **not tested on AlmaLinux 10**. Package names, service names and some output may differ slightly. Where a step is most likely to differ, a note says so.

## Goal

Build a MySQL database that **keeps running and keeps its data** even if one database server dies.

You will build it in phases, learning one thing at a time:

| Phase | What you build | What you learn |
|---|---|---|
| 0 | Base VM and an NFS server VM | Hyper-V VMs and networking |
| 1 | NFS server with a data disk | How shared storage works |
| 2 | One MySQL server using NFS | How MySQL stores data |
| 3 | A second DB node and a **manual** failover | What the cluster will automate |
| 4 | Pacemaker + Corosync cluster | How nodes work together |
| 5 | Cluster resources and a Virtual IP | How clients find the active DB |
| 6 | Failure tests | Automatic failover in action |
| 7 | Backups and next steps | What real systems add |

## Final Design

```
                     Clients (Windows / apps)
                              |
                      VIP 10.10.10.200      <-- moves to the active node
                              |
               +--------------+--------------+
               |                             |
             db1                           db2
         10.10.10.101                  10.10.10.102
        (MySQL runs on                (MySQL runs on
         ONE node only)                ONE node only)
               \                             /
                \____________  ____________/
                             \/
                            nfs1
                        10.10.10.50
                  data disk (MySQL files live here)
```

| VM | Role | `eth0` (Default Switch, internet) | `eth1` (vSwitch-Internal) |
|---|---|---|---|
| `nfs1` | NFS storage server | DHCP | `10.10.10.50` |
| `db1` | Database node 1 | DHCP | `10.10.10.101` |
| `db2` | Database node 2 | DHCP | `10.10.10.102` |
| VIP | Floating client address | - | `10.10.10.200` |

## Key Words (Read Once)

| Term | Simple meaning |
|---|---|
| **NFS** | A way to share a folder over the network. Servers see it like a local folder |
| **Shared storage** | One storage place that several servers can use (not at the same time here) |
| **Active node** | The node currently running MySQL |
| **Failover** | Moving MySQL to the other node when the active one fails |
| **VIP** | Virtual IP. A floating address clients use. It follows MySQL |
| **Corosync** | Keeps the nodes talking and knowing who is alive |
| **Pacemaker** | The manager that decides where MySQL runs |
| **Resource** | Something the cluster manages (mount, MySQL, IP) |

## The Golden Rules (Read These)

1. **Only ONE node may run MySQL on the shared folder at any time.** Two MySQL servers writing to the same files will **corrupt the data**. Pacemaker enforces this for you later. Until then, *you* must enforce it.
2. **MySQL files stay on the NFS share.** The nodes only borrow them.
3. **In the cluster, Pacemaker starts and stops MySQL.** Never enable `mysqld` at boot, and never add the NFS mount to `/etc/fstab` on db1/db2.

## How to Read This Guide

- Run commands **one at a time**. Press **Enter**, read the result, then continue.
- Every step has a **Run on:** label. Always check which VM you are on.
- Run everything as `root`.
- Lines after `#` are notes for you. Do not type them.
- Passwords in this guide (`RootPass#123`, `LabPass#123`) are **lab only**. Never use them for real.

## Prerequisites

- Windows 10/11 Pro/Enterprise with Hyper-V, and the `vSwitch-Internal` switch from guide 01 (Windows adapter `10.10.10.1`)
- AlmaLinux 10 ISO
- About 50 GB free disk space on the Windows host
- 4 GB RAM free per VM is ideal (2 GB each is the minimum)

---

# Phase 0 - Build the Base VM

## Step 0.1 - Build `db1` from Guide 01

Follow **guide 01** to create one VM with two network adapters:

- First adapter: `Default Switch` → `eth0` (automatic IP, internet)
- Second adapter: `vSwitch-Internal` → `eth1` with the fixed IP `10.10.10.101`

**Run on: db1**

Set the hostname:

```bash
hostnamectl set-hostname db1
```

Check that the internet works:

```bash
ping -c 4 8.8.8.8
```

## Step 0.2 - Update and Install the NFS Tools

**Run on: db1**

```bash
dnf update -y
```

```bash
dnf install -y nfs-utils
```

Every VM in this project needs `nfs-utils`. Installing it now means the clone gets it too.

## Step 0.3 - Clone `db1` into `nfs1`

1. **Run on: db1**, shut it down:

```bash
poweroff
```

2. Follow **Appendix A (Clone a VM)** at the end of this guide.
   - New VM name: `nfs1`
   - New hostname: `nfs1`
   - New `eth1` IP: `10.10.10.50`
3. Start `db1` again afterwards.

---

# Phase 1 - Build the NFS Server

## Step 1.1 - Add a Data Disk to `nfs1`

MySQL's files should live on a **separate disk**, not on the OS disk.

1. In **Hyper-V Manager**, right-click `nfs1` → **Settings**.
2. Click **SCSI Controller** → **Hard Drive** → **Add**.
3. Click **New** → **Next** → choose **VHDX** → **Next** → **Dynamically expanding** → **Next**.
4. Name: `nfs1-data.vhdx`, Size: **10 GB** → **Next** → **Finish** → **OK**.

> This is a lab size. Real databases need much more.

## Step 1.2 - Find the New Disk

**Run on: nfs1**

```bash
lsblk
```

Expected: a disk, normally `sdb`, with `10G` and **no partitions** under it. The OS disk is `sda`.

> **Careful:** the next step erases the disk. Be sure you pick the empty 10 GB disk, not `sda`.

## Step 1.3 - Format the Disk

**Run on: nfs1**

```bash
mkfs.xfs /dev/sdb
```

## Step 1.4 - Mount the Disk at `/srv/nfs`

**Run on: nfs1**

Create the mount folder:

```bash
mkdir -p /srv/nfs
```

Get the disk's unique ID:

```bash
blkid /dev/sdb
```

Copy the value after `UUID=` (without the quotes).

Add it to `/etc/fstab` so it mounts at every boot. **Replace `PASTE-UUID-HERE`** with your UUID:

```bash
echo "UUID=PASTE-UUID-HERE /srv/nfs xfs defaults 0 0" >> /etc/fstab
```

Reload and mount:

```bash
systemctl daemon-reload
```

```bash
mount -a
```

Check:

```bash
df -h /srv/nfs
```

Expected: a line showing about `10G` mounted on `/srv/nfs`.

## Step 1.5 - Create the Database Folder

**Run on: nfs1**

```bash
mkdir -p /srv/nfs/mysql
```

MySQL runs as the user `mysql`, whose ID number is **27** on AlmaLinux. `nfs1` does not have that user, so we use the number:

```bash
chown 27:27 /srv/nfs/mysql
```

```bash
chmod 750 /srv/nfs/mysql
```

Check:

```bash
ls -ld /srv/nfs/mysql
```

Expected: owner and group show as `27`.

## Step 1.6 - Share the Folder (Export)

**Run on: nfs1**

```bash
echo "/srv/nfs/mysql 10.10.10.0/24(rw,sync,no_subtree_check)" > /etc/exports
```

| Part | Meaning |
|---|---|
| `/srv/nfs/mysql` | The folder to share |
| `10.10.10.0/24` | Only machines on our lab network may connect |
| `rw` | Read and write |
| `sync` | Write to disk **before** replying. Safer for a database |
| `no_subtree_check` | Avoids a common NFS problem |

Check the file:

```bash
cat /etc/exports
```

Apply it:

```bash
exportfs -rav
```

Expected: `exporting 10.10.10.0/24:/srv/nfs/mysql`.

## Step 1.7 - Allow SELinux to Share It

**Run on: nfs1**

```bash
setsebool -P nfs_export_all_rw on
```

> If this command is not found, run `dnf install -y policycoreutils-python-utils` first, then try again.

## Step 1.8 - Start the NFS Service

**Run on: nfs1**

```bash
systemctl enable --now nfs-server
```

```bash
systemctl status nfs-server
```

Look for `active (exited)` or `active (running)`. Press `q` to leave.

## Step 1.9 - Open the Firewall

**Run on: nfs1**

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

Expected: your folder with the options and `10.10.10.0/24`.

---

# Phase 2 - One MySQL Server Using NFS

Goal: a single working database whose files live on `nfs1`.

## Step 2.1 - Install MySQL

**Run on: db1**

```bash
dnf install -y mysql-server
```

Check the version:

```bash
mysql --version
```

> The package name should be `mysql-server` and the service `mysqld`. If `dnf` says "no match", run `dnf search mysql-server` and use the name it shows.

## Step 2.2 - Open the MySQL Port

**Run on: db1**

```bash
firewall-cmd --permanent --add-service=mysql
```

```bash
firewall-cmd --reload
```

## Step 2.3 - Check SELinux

**Run on: db1**

```bash
getenforce
```

If it says `Permissive` (as set in guide 01), continue. If it says `Enforcing`, SELinux may block MySQL from using NFS files. For this lab, set it to permissive:

```bash
setenforce 0
```

To make it permanent, use the `sed` command shown in the SELinux section of guide 01.

## Step 2.4 - Test the NFS Share

**Run on: db1**

Ping the NFS server:

```bash
ping -c 3 10.10.10.50
```

Mount the share temporarily:

```bash
mount -t nfs4 10.10.10.50:/srv/nfs/mysql /mnt
```

Check that it is mounted:

```bash
df -h /mnt
```

Test writing **as the mysql user** (the folder belongs to user 27):

```bash
runuser -u mysql -- touch /mnt/testfile
```

```bash
runuser -u mysql -- ls -l /mnt
```

Expected: `testfile` is listed.

Clean up:

```bash
runuser -u mysql -- rm /mnt/testfile
```

```bash
umount /mnt
```

> **Why use `runuser`?** NFS maps the `root` user to a harmless user ("root squash"), so even `root` on `db1` cannot read the MySQL folder. This is normal and good for security. Only the `mysql` user (ID 27) can.

## Step 2.5 - Mount the Share as MySQL's Data Folder

MySQL stores its data in `/var/lib/mysql`. We mount the NFS share **on top of** that folder.

**Run on: db1**

```bash
mount -t nfs4 -o hard,vers=4.2 10.10.10.50:/srv/nfs/mysql /var/lib/mysql
```

| Option | Meaning |
|---|---|
| `hard` | If NFS is unreachable, **wait** instead of returning errors. Prevents corrupted writes |
| `vers=4.2` | Use NFS version 4.2 |

Check:

```bash
mount | grep mysql
```

Expected: a line showing `10.10.10.50:/srv/nfs/mysql on /var/lib/mysql`.

## Step 2.6 - Start MySQL

**Run on: db1**

```bash
systemctl start mysqld
```

The first start builds the database files, so wait up to a minute. Check:

```bash
systemctl status mysqld
```

Look for `active (running)`. Press `q`.

> **Do not run `systemctl enable mysqld`.** In the cluster, Pacemaker starts MySQL.

## Step 2.7 - Secure MySQL and Create a Test Database

**Run on: db1**

Open the MySQL shell. On a fresh install the root user usually has no password:

```bash
mysql -u root
```

You will see a `mysql>` prompt. Type these **one at a time**, pressing Enter after each:

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

Expected: one row with `Hello from db1`. Leave the shell:

```sql
EXIT;
```

| What we created | Purpose |
|---|---|
| `root` password | Protects the admin account |
| `labdb` | A test database |
| `labuser` | A user that may connect from any `10.10.10.x` machine |
| `notes` table | A small table so we can see data survive failures |

> **Asked for a root password already?** Then a temporary one was generated. Find it with `grep -i "temporary password" /var/log/mysqld.log`, log in with it, and change it with the `ALTER USER` line above.

## Step 2.8 - See Where the Data Really Lives

**Run on: db1**

```bash
runuser -u mysql -- ls /var/lib/mysql
```

You will see MySQL files (`mysql`, `labdb`, `ibdata1` and others).

**Run on: nfs1**

```bash
ls /srv/nfs/mysql
```

You see **the same files**. They live on `nfs1`. `db1` only borrows them.

## Step 2.9 - Connect from Another Machine

**Run on: nfs1** (we use it as a client)

```bash
dnf install -y mysql
```

```bash
mysql -h 10.10.10.101 -u labuser -p labdb
```

Enter `LabPass#123`. At the `mysql>` prompt:

```sql
SELECT * FROM notes;
```

```sql
EXIT;
```

If it works, a remote client can use your database.

## Step 2.10 - Prove the Data Survives a Restart

**Run on: db1**

Stop MySQL and unmount the share:

```bash
systemctl stop mysqld
```

```bash
umount /var/lib/mysql
```

Check the folder is empty locally (the data is not on db1):

```bash
ls /var/lib/mysql
```

Mount again and start:

```bash
mount -t nfs4 -o hard,vers=4.2 10.10.10.50:/srv/nfs/mysql /var/lib/mysql
```

```bash
systemctl start mysqld
```

```bash
mysql -u root -p -e "SELECT * FROM labdb.notes;"
```

Enter `RootPass#123`. Your row is still there. **This is the idea behind the whole project:** the data is separate from the server that runs MySQL.

---

# Phase 3 - Add a Second Node and Fail Over by Hand

First you will move MySQL between nodes **manually**. This shows exactly what Pacemaker will later do automatically.

## Step 3.1 - Prepare `db1`

**Run on: db1**

Stop MySQL and release the share:

```bash
systemctl stop mysqld
```

```bash
umount /var/lib/mysql
```

Make sure it never starts by itself:

```bash
systemctl disable mysqld
```

## Step 3.2 - Install the Cluster Software on `db1`

We install it **before cloning**, so `db2` gets it too.

**Run on: db1**

```bash
dnf install -y dnf-plugins-core
```

```bash
dnf config-manager --set-enabled highavailability
```

```bash
dnf install -y pacemaker pcs
```

Open the cluster firewall:

```bash
firewall-cmd --permanent --add-service=high-availability
```

```bash
firewall-cmd --reload
```

Start the pcs helper service:

```bash
systemctl enable --now pcsd
```

## Step 3.3 - Clone `db1` into `db2`

1. **Run on: db1**, shut down:

```bash
poweroff
```

2. Follow **Appendix A (Clone a VM)**.
   - New VM name: `db2`
   - New hostname: `db2`
   - New `eth1` IP: `10.10.10.102`
3. Start `db1` again.

`db2` already has MySQL, the NFS tools, the cluster software and the firewall rules, because it is a copy of `db1`.

## Step 3.4 - Let All VMs Find Each Other by Name

**Run on: db1, db2 and nfs1** (all three commands on each VM)

```bash
echo "10.10.10.50 nfs1" >> /etc/hosts
```

```bash
echo "10.10.10.101 db1" >> /etc/hosts
```

```bash
echo "10.10.10.102 db2" >> /etc/hosts
```

> Run each `echo` only **once per VM**. Duplicates are harmless but untidy.

## Step 3.5 - Test the Connections

**Run on: db1**

```bash
ping -c 2 db2
```

```bash
ping -c 2 nfs1
```

**Run on: db2**

```bash
ping -c 2 db1
```

```bash
ping -c 2 nfs1
```

All must reply.

## Step 3.6 - Manual Failover: Run MySQL on `db1`

**Run on: db1**

```bash
mount -t nfs4 -o hard,vers=4.2 10.10.10.50:/srv/nfs/mysql /var/lib/mysql
```

```bash
systemctl start mysqld
```

Add a row:

```bash
mysql -u root -p -e "INSERT INTO labdb.notes (msg) VALUES ('Written on db1');"
```

## Step 3.7 - Manual Failover: Stop `db1`, Start `db2`

**Run on: db1** (stop MySQL **first**)

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

Now stop `db2`:

```bash
systemctl stop mysqld
```

```bash
umount /var/lib/mysql
```

**What you just did by hand:** stop MySQL → unmount → mount on the other node → start MySQL. In Phase 5, Pacemaker does exactly this automatically and also moves the VIP.

> **Never** start MySQL on both nodes at the same time. If you do, stop one immediately.

---

# Phase 4 - Build the Pacemaker Cluster

Both nodes must have `mysqld` **stopped** and `/var/lib/mysql` **unmounted** before you continue. Check on both:

```bash
systemctl is-active mysqld
```

Expected: `inactive`.

```bash
mount | grep mysql
```

Expected: no output.

## Step 4.1 - Set the Cluster User Password

**Run on: db1 and db2**

```bash
passwd hacluster
```

Use the **same password on both nodes**. Write it down.

## Step 4.2 - Make the Nodes Trust Each Other

**Run on: db1 only**

```bash
pcs host auth db1 db2 -u hacluster
```

Enter the password. Expected: `db1: Authorized` and `db2: Authorized`.

## Step 4.3 - Create and Start the Cluster

**Run on: db1 only**

```bash
pcs cluster setup dbcluster db1 db2
```

Expected last line: `Cluster has been successfully set up.`

Start it:

```bash
pcs cluster start --all
```

Start it automatically at boot:

```bash
pcs cluster enable --all
```

> **If `pcs cluster start --all` shows `HTTP error: 400`:** make sure `hostname` on each VM really is `db1` / `db2`, that `getent hosts db2` shows `10.10.10.102`, then run `systemctl restart pcsd` on both and try again. You can also run `pcs cluster start` on the failing node itself.

## Step 4.4 - Verify the Cluster

**Run on: db1**

```bash
pcs status
```

Expected: **Online: [ db1 db2 ]**.

```bash
corosync-cfgtool -s
```

Expected: each node shows `connected`, and the addresses are `10.10.10.x`.

```bash
pcs status corosync
```

Expected: both nodes listed under membership.

## Step 4.5 - Turn Off Fencing (Lab Only)

**Run on: db1 only**

```bash
pcs property set stonith-enabled=false
```

> **Important:** fencing protects your data from two nodes writing at once. This project relies on you not breaking the rules, because fencing is off. Real clusters **must** use fencing. See guide 06, Part G, to learn it.

---

# Phase 5 - Add the Resources and the Virtual IP

We create three resources and group them so they start in order and always run on the **same node**:

1. **DBFS** - mounts the NFS share at `/var/lib/mysql`
2. **MySQLService** - starts MySQL
3. **DBVIP** - the floating IP clients use (last, so clients connect only after MySQL is up)

**Run on: db1 only**

## Step 5.1 - The NFS Mount Resource

```bash
pcs resource create DBFS Filesystem device="10.10.10.50:/srv/nfs/mysql" directory="/var/lib/mysql" fstype="nfs4" options="hard,vers=4.2" op monitor interval=20s timeout=40s op start timeout=120s op stop timeout=120s
```

| Part | Meaning |
|---|---|
| `DBFS` | Name of the resource |
| `Filesystem` | The agent that mounts and unmounts things |
| `device`, `directory`, `fstype` | The same share and folder you mounted by hand |
| `op start / stop timeout` | Allow up to 2 minutes (NFS can be slow) |

## Step 5.2 - The MySQL Resource

```bash
pcs resource create MySQLService systemd:mysqld op monitor interval=30s timeout=60s op start timeout=180s op stop timeout=180s
```

`systemd:mysqld` means "manage the normal `mysqld` service".

## Step 5.3 - The Virtual IP Resource

```bash
pcs resource create DBVIP ocf:heartbeat:IPaddr2 ip=10.10.10.200 cidr_netmask=24 nic=eth1 op monitor interval=10s
```

## Step 5.4 - Group Them in the Right Order

```bash
pcs resource group add DBGroup DBFS MySQLService DBVIP
```

The order in the command is the start order. Stopping happens in reverse.

## Step 5.5 - Check

```bash
pcs status
```

Expected after about a minute:

```
Resource Group: DBGroup
  * DBFS          (ocf:heartbeat:Filesystem):  Started db1
  * MySQLService  (systemd:mysqld):            Started db1
  * DBVIP         (ocf:heartbeat:IPaddr2):     Started db1
```

All three must be **Started** on the **same node**.

On the active node verify the pieces:

```bash
mount | grep mysql
```

```bash
ip -br a
```

Expected: the NFS mount is present, and `eth1` shows **both** its own IP and `10.10.10.200`.

## Step 5.6 - Connect Using the VIP

**Run on: nfs1** (as a client)

```bash
mysql -h 10.10.10.200 -u labuser -p labdb -e "SELECT * FROM notes;"
```

Enter `LabPass#123`. You see all rows. Clients only need to know the VIP, never which node is active.

> **From Windows:** you can use a tool like MySQL Workbench or HeidiSQL with host `10.10.10.200`, user `labuser`.

---

# Phase 6 - Break Things and Watch It Recover

Keep a connection ready:

```bash
mysql -h 10.10.10.200 -u labuser -p labdb
```

Find the active node first with `pcs status`.

## Test 1 - Planned Move (Standby)

**Run on: db1**

```bash
pcs node standby db1
```

```bash
pcs status
```

Expected: the whole `DBGroup` moves to `db2`. Reconnect through the VIP and run:

```sql
SELECT * FROM labdb.notes;
```

Add a row to prove it is writable:

```sql
INSERT INTO labdb.notes (msg) VALUES ('Written after standby');
```

Bring `db1` back:

```bash
pcs node unstandby db1
```

## Test 2 - Crash a Node

In **Hyper-V Manager**, right-click the VM that currently runs `DBGroup` → **Turn Off** (not Shut Down).

On the surviving node:

```bash
pcs status
```

Expected after roughly 30 to 120 seconds:

- The failed node is **OFFLINE**.
- `DBGroup` is **Started** on the survivor.
- Your data is still there through the VIP.

> **Why can it take a while?** NFS keeps a lock "lease" (about 90 seconds) for a crashed client. MySQL may fail to start at first, with a message like `Unable to lock ./ibdata1`. Pacemaker retries. If it stays failed, run `pcs resource cleanup` after waiting two minutes.

Start the powered-off VM again. It rejoins on its own because of `pcs cluster enable --all`.

## Test 3 - Crash Only MySQL

**Run on: the active node**

```bash
systemctl stop mysqld
```

Watch:

```bash
pcs status
```

Within about 30 seconds Pacemaker notices and restarts it. Clear old error messages:

```bash
pcs resource cleanup
```

## Test 4 - Lose the NFS Server (See the Weak Point)

**Run on: nfs1**

```bash
systemctl stop nfs-server
```

Now try a query through the VIP. It **hangs**. That is the `hard` mount protecting your data by waiting.

Bring it back:

```bash
systemctl start nfs-server
```

The query completes. If the cluster marked resources failed, run `pcs resource cleanup`.

**The lesson:** `nfs1` is now a **single point of failure**. If it stays down, both database nodes are useless. The project's weakest link is the storage.

## Test 5 - Confirm No Data Was Lost

```bash
mysql -h 10.10.10.200 -u labuser -p labdb -e "SELECT * FROM notes;"
```

Expected: every row you ever inserted.

---

# Phase 7 - Backups and Next Steps

## Step 7.1 - Take a Backup

HA is **not** a backup. A deleted table is deleted on both nodes. Run on the active node:

```bash
mysqldump -u root -p --all-databases > /root/all-databases.sql
```

Copy the file somewhere **off** these VMs, for example to your Windows PC with MobaXterm.

## What to Learn Next

1. **Fencing (STONITH):** turn `stonith-enabled` back on with a real device. See guide 06, Part G.
2. **Highly available storage:** make `nfs1` redundant (DRBD plus a second storage node), or use replicated storage.
3. **Replication instead of shared storage:** MySQL's own replication or InnoDB Cluster, which does not share files at all and is how many production systems work.
4. **A third node or quorum device** so quorum works properly.
5. **Monitoring and alerts:** know about a failover when it happens.

---

# Appendix A - Clone a VM (Hyper-V)

Use this whenever the guide says "clone". You need the source VM shut down first.

## A.1 - Export and Import in Hyper-V

1. Right-click the source VM → **Export…** → choose a folder → **Export**.
2. In the right-hand menu click **Import Virtual Machine…** → **Next**.
3. Choose the exported folder → **Next** → select the VM → **Next**.
4. Choose **Copy the virtual machine (create a new unique ID)** → **Next** → **Next** → **Finish**.
5. Right-click the imported VM → **Rename** to the new name.
6. Check Settings → Network Adapter. Order must be: first = `Default Switch`, second = `vSwitch-Internal`.

> **MAC address:** if you set a fixed MAC address on the source adapters, change them on the clone. Two VMs must never share one.

Start the clone, and log in on the **Hyper-V console** (not SSH).

## A.2 - Give the Clone Its Own Identity

**Run on: the clone**

Set its hostname (use `nfs1` or `db2`):

```bash
hostnamectl set-hostname NEW_NAME
```

Change its internal IP (use `10.10.10.50` for `nfs1`, `10.10.10.102` for `db2`):

```bash
nmcli connection modify eth1 ipv4.addresses NEW_IP/24
```

```bash
nmcli connection down eth1
```

```bash
nmcli connection up eth1
```

Create a new unique machine ID (the clone has the source's):

```bash
rm -f /etc/machine-id
```

```bash
systemd-machine-id-setup
```

```bash
reboot
```

After reboot, verify:

```bash
hostname
```

```bash
ip -br a
```

Expected: the new hostname, `eth1` with the new IP, and `eth0` with a `172.x.x.x` address.

---

# Command Cheat Sheet

| Task | Command |
|---|---|
| Cluster overview | `pcs status` |
| Move work off a node / back | `pcs node standby db1` / `pcs node unstandby db1` |
| Clear failures | `pcs resource cleanup` |
| Move the group by hand | `pcs resource move DBGroup db2` |
| Remove the move preference | `pcs resource clear DBGroup` |
| Show resource setup | `pcs resource config` |
| Is MySQL running? | `systemctl is-active mysqld` |
| Is the NFS share mounted? | `mount | grep mysql` |
| What does the NFS server share? | `exportfs -v` (on nfs1) |
| Is the VIP here? | `ip -br a` |
| MySQL error log | `tail -n 50 /var/log/mysqld.log` |
| Cluster log | `journalctl -u pacemaker -n 50` |

---

# Troubleshooting

| Problem | Likely cause | Fix |
|---|---|---|
| `mount: access denied` or `No such file` | Export or firewall wrong | On nfs1: `exportfs -v`, check Step 1.6 and 1.9 |
| `mount.nfs4: Connection timed out` | Cannot reach nfs1 | `ping nfs1`; check the firewall allows `nfs` |
| `touch: Permission denied` on the share | Wrong owner, or tried as root | Redo Step 1.5; test with `runuser -u mysql` |
| `ls /var/lib/mysql` says Permission denied (as root) | Root squash | Normal. Use `runuser -u mysql -- ls ...` |
| `mysqld` fails to start | SELinux, permissions or lock | `tail -n 50 /var/log/mysqld.log`; check `getenforce` |
| `Unable to lock ./ibdata1` after a crash | NFS lock lease not expired yet | Wait 2 minutes, then `pcs resource cleanup` |
| Cannot log in as root | Password not set, or temporary password | See the note in Step 2.7 |
| Remote client: `Host is not allowed to connect` | User host does not match the client | The user was created for `10.10.10.%`. Connect from a `10.10.10.x` machine |
| Remote client: connection refused | Firewall or MySQL not running | Check `firewall-cmd --list-services` on the active node |
| `pcs cluster start --all` gives `HTTP error: 400` | Hostname or name resolution wrong | See the note in Step 4.3 |
| Resources show `Stopped` or `Failed` | Earlier failure, or mount problem | `pcs status`, then `pcs resource cleanup` |
| `DBFS` fails to start | NFS unreachable or already mounted | `mount | grep mysql` and `umount /var/lib/mysql` on the failing node, then `pcs resource cleanup` |
| MySQL running on both nodes | Started manually | Stop one **immediately**: `systemctl stop mysqld`, then `pcs resource cleanup` |
| VIP not reachable | VIP on another node, or wrong NIC | `pcs status`; `ip -br a` on the active node |

---

# Checklist

**Phases 0-2**
- [ ] `db1` has internet on `eth0` and `10.10.10.101` on `eth1`
- [ ] `nfs1` has a separate 10 GB data disk mounted at `/srv/nfs`
- [ ] `/srv/nfs/mysql` is owned by user `27` and exported to `10.10.10.0/24`
- [ ] `db1` can mount the share and run MySQL on it
- [ ] `labdb.notes` data survives `stop`, `umount`, `mount`, `start`

**Phase 3**
- [ ] `db2` cloned with its own hostname, IP and machine ID
- [ ] Manual failover moved MySQL from `db1` to `db2` with data intact
- [ ] `mysqld` is **disabled** at boot on both DB nodes

**Phases 4-5**
- [ ] `pcs status` shows both nodes **Online**
- [ ] `stonith-enabled` is `false` (lab only)
- [ ] `DBGroup` is **Started** on one node
- [ ] The VIP `10.10.10.200` appears on that node
- [ ] You can query the database through the VIP

**Phase 6**
- [ ] Standby test moved everything and the data was still there
- [ ] Power-off test failed over automatically
- [ ] Stopping `mysqld` was repaired by Pacemaker
- [ ] You saw what happens when `nfs1` goes down

**Phase 7**
- [ ] A backup file exists and is stored **off** the VMs
