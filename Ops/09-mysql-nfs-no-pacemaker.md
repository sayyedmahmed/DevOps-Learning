# 09 - MySQL on Shared NFS Storage Without Pacemaker

**Architecture Name:** Shared-Nothing Compute with Shared Storage Architecture, more commonly described as a Shared Storage High Availability (HA) Architecture.

**Platform:** AlmaLinux 10  
**MySQL:** Oracle MySQL Community Server 8.0.46  
**Architecture:** Shared NFS storage with manual MySQL failover  
**Automation:** None yet. Pacemaker will be configured in a separate guide.

> **Important:** This guide documents the lab using the paths, IP addresses, and MySQL installation you actually used. Read each step before running its commands. Do not run MySQL on both database nodes simultaneously.

## 1. Overview

The goal is to store MySQL's database files on a separate NFS server and allow either database node to use those files. MySQL runs on only one node at a time. You manually stop it on one node before starting it on the other.

```text
             Database Node 1                 Database Node 2
              10.10.10.101                    10.10.10.102
              MySQL: Active                   MySQL: Stopped
                     \                            /
                      \                          /
                       +------------------------+
                                  |
                                  v
                        NFS Server
                        10.10.10.200
                                  |
                        /srv/nfs-dbstorage/shared
                                  |
                             mysql_data/
                                  |
                          Shared database files
```

### Lab configuration

| Machine | Role | IP address |
| --- | --- | --- |
| node1 | Database node 1 | 10.10.10.101 |
| node2 | Database node 2 | 10.10.10.102 |
| NFS server | Shared storage | 10.10.10.200 |

| Setting | Value |
| --- | --- |
| NFS export | /srv/nfs-dbstorage/shared |
| NFS mountpoint on database nodes | /mnt/shared |
| Shared MySQL datadir | /mnt/shared/mysql_data |
| MySQL installation | /usr/local/mysql-8.0.46-linux-glibc2.28-x86_64 |
| MySQL socket | /var/lib/mysql/mysql.sock |
| MySQL PID file | /run/mysqld/mysqld.pid |
| MySQL error log | /var/log/mysql/error.log |
| MySQL operating-system user | mysql , UID/GID 27:27 |

### What this setup provides
- Shared database files accessible from both nodes.
- Manual switching between database nodes.
- Persistence of database files independently of either database VM.

### What this setup does not provide
- Automatic failover.
- Protection against failure of the NFS server.
- A database backup merely because the files are on NFS.
- Protection against both MySQL instances running simultaneously.
- Production-grade database storage guarantees.

> **Golden rule:** Never run two MySQL servers against the same writable data directory. If a node crashes, do not start MySQL on the other node until you have established that the old instance cannot still access the shared files. In a clustered design, this requires reliable fencing.

## Part A - Prepare the NFS Server

### Step 1 - Install NFS utilities
Run on: NFS server (`10.10.10.200`)
```bash
dnf install -y nfs-utils
```

### Step 2 - Create the shared directory
```bash
mkdir -p /srv/nfs-dbstorage/shared/mysql_data
```
MySQL runs under UID/GID `27:27` on the database nodes. Confirm this on both nodes:
```bash
id mysql
```
Expected:
```text
uid=27(mysql) gid=27(mysql)
```
On the NFS server, assign ownership using the numeric IDs:
```bash
chown 27:27 /srv/nfs-dbstorage/shared/mysql_data
chmod 750 /srv/nfs-dbstorage/shared/mysql_data
```
Numeric ownership matters because NFS permissions depend on user IDs, not merely usernames.

### Step 3 - Configure the NFS export
Create a separate export configuration file so you do not accidentally overwrite other exports:
```bash
vi /etc/exports.d/mysql-nfs.exports
```
Add:
```text
/srv/nfs-dbstorage/shared 10.10.10.101(rw,sync,no_subtree_check,root_squash) 10.10.10.102(rw,sync,no_subtree_check,root_squash)
```
Save and exit.

Apply the export:
```bash
exportfs -rav
```
Verify:
```bash
exportfs -v
```
Confirm that both database node IPs are allowed to access the shared directory.

### Step 4 - Start NFS and configure the firewall
```bash
systemctl enable --now nfs-server
```
Allow NFS through the firewall:
```bash
firewall-cmd --permanent --add-service=nfs
firewall-cmd --reload
```
Check:
```bash
systemctl status nfs-server --no-pager
firewall-cmd --list-services
```
The `nfs` service should appear in the firewall's allowed services.

**SELinux note:** Keep SELinux enforcing where possible. If NFS export access is blocked, inspect the audit log and configure the appropriate SELinux policy rather than disabling SELinux without investigation.

## Part B - Prepare Both Database Nodes

### Step 5 - Verify MySQL installation
Run on: node1 and node2
```bash
/usr/local/mysql-8.0.46-linux-glibc2.28-x86_64/bin/mysqld --version
```
Both nodes should report MySQL Community Server `8.0.46`.

Check the operating-system user:
```bash
id mysql
```
Both nodes must use compatible MySQL versions, matching user IDs, and compatible configurations.

### Step 6 - Create the local log and runtime directories
Run on: node1 and node2
```bash
mkdir -p /var/log/mysql
mkdir -p /var/lib/mysql
mkdir -p /run/mysqld
chown mysql:mysql /var/log/mysql
chmod 750 /var/log/mysql
chown mysql:mysql /run/mysqld
chmod 750 /run/mysqld
```
Create the error log if it does not exist:
```bash
touch /var/log/mysql/error.log
chown mysql:mysql /var/log/mysql/error.log
chmod 640 /var/log/mysql/error.log
```
These paths are local to each database node. The database files themselves will live on NFS.

### Step 7 - Configure MySQL
Run on: node1 and node2

Back up the existing configuration:
```bash
cp -a /etc/my.cnf /etc/my.cnf.backup
```
Edit the configuration:
```bash
vi /etc/my.cnf
```
Use:
```ini
[client]
socket=/var/lib/mysql/mysql.sock

[mysqld]
user=mysql
basedir=/usr/local/mysql-8.0.46-linux-glibc2.28-x86_64
datadir=/mnt/shared/mysql_data
socket=/var/lib/mysql/mysql.sock
pid-file=/run/mysqld/mysqld.pid
log-error=/var/log/mysql/error.log
```
**Important:**
- `datadir` points to the shared NFS directory.
- The socket, PID file, and error log remain local.
- Do not use `--initialize` against the shared directory.
- Do not create a second, separate MySQL database on node2.

Verify the effective MySQL options:
```bash
/usr/local/mysql-8.0.46-linux-glibc2.28-x86_64/bin/my_print_defaults mysqld
```
Confirm that `--datadir=/mnt/shared/mysql_data` appears and `--initialize` does not.

### Step 8 - Verify the MySQL systemd service
Run on: node1 and node2

Inspect the service:
```bash
systemctl cat mysqld
```
The service should use the installed binary and `/etc/my.cnf`. For this lab, the `ExecStart` line is:
```text
ExecStart=/usr/local/mysql-8.0.46-linux-glibc2.28-x86_64/bin/mysqld --defaults-file=/etc/my.cnf
```
If you need to create the service file, use:
```bash
vi /etc/systemd/system/mysqld.service
```
Content:
```ini
[Unit]
Description=Oracle MySQL Community Server 8.0
After=network.target

[Service]
Type=simple
User=mysql
Group=mysql
RuntimeDirectory=mysqld
RuntimeDirectoryMode=0750
ExecStart=/usr/local/mysql-8.0.46-linux-glibc2.28-x86_64/bin/mysqld --defaults-file=/etc/my.cnf
Restart=on-failure
RestartSec=5
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
```
Reload systemd after creating or editing the file:
```bash
systemctl daemon-reload
```
For manual failover, disable automatic startup on both nodes after ensuring the NFS mount is configured and MySQL is safely stopped on the node being changed:
```bash
systemctl disable mysqld
```
This prevents ordinary systemd boot-time enabling. It does not prevent a manual `systemctl start mysqld`. Later, Pacemaker must manage the service explicitly.

## Part C - Mount NFS on Both Nodes

### Step 9 - Create the mountpoint
Run on: node1 and node2
```bash
mkdir -p /mnt/shared
```
Check network connectivity to the NFS server:
```bash
ping -c 3 10.10.10.200
```
Inspect the NFS exports from the client:
```bash
showmount -e 10.10.10.200
```
Note that `showmount` output can vary with NFS server configuration; the actual mount test is the more important check.

### Step 10 - Mount the shared export
Run on: node1 and node2
```bash
mount -t nfs4 -o hard,vers=4.2 \
  10.10.10.200:/srv/nfs-dbstorage/shared /mnt/shared
```
Verify:
```bash
findmnt /mnt/shared
df -h /mnt/shared
```
Expected source:
```text
10.10.10.200:/srv/nfs-dbstorage/shared
```
Check access as the MySQL user:
```bash
runuser -u mysql -- test -r /mnt/shared/mysql_data \
  && echo "MySQL can access the shared directory"
runuser -u mysql -- test -w /mnt/shared/mysql_data \
  && echo "MySQL can write to the shared directory"
```
Both checks should succeed.

**Optional: Make the mount persistent**
After confirming the mount works, add an entry to `/etc/fstab` on each database node:
```text
10.10.10.200:/srv/nfs-dbstorage/shared /mnt/shared nfs4 rw,hard,vers=4.2,_netdev 0 0
```
Before adding it, check whether a matching entry already exists:
```bash
grep -n '/mnt/shared' /etc/fstab
```
Do not create duplicate entries. Test the configuration carefully:
```bash
systemctl daemon-reload
findmnt --verify
```
The MySQL service must not start unless the NFS mount is available. A mountpoint directory existing locally does not prove that NFS is mounted.

## Part D - Move the Existing Database from node1 to NFS

This section reflects what was done in the actual lab. Node1 already had a working MySQL installation with its database in `/var/lib/mysql`.
Do not follow this migration section on a fresh server that has no existing database to preserve.

### Step 11 - Make a logical backup
Run on: node1 while the original local MySQL database is running

Create a backup directory:
```bash
mkdir -p /root/mysql-backup
chmod 700 /root/mysql-backup
```
Create a backup:
```bash
/usr/local/mysql-8.0.46-linux-glibc2.28-x86_64/bin/mysqldump \
  --socket=/var/lib/mysql/mysql.sock \
  -uroot -p \
  --all-databases \
  --routines --events --triggers --single-transaction \
  > /root/mysql-backup/all-databases.sql
```
Verify that the backup exists and is not empty:
```bash
ls -lh /root/mysql-backup/all-databases.sql
head -n 20 /root/mysql-backup/all-databases.sql
```
Keep this backup. NFS is shared storage, not an independent backup.

### Step 12 - Stop MySQL cleanly on node1
```bash
systemctl stop mysqld
systemctl is-active mysqld
```
Expected:
```text
inactive
```
Do not copy a live database directory with ordinary `rsync` and assume the copy is consistent. Stop MySQL cleanly first.

### Step 13 - Copy the existing data to NFS
Confirm that NFS is mounted:
```bash
findmnt /mnt/shared
```
Copy the files:
```bash
runuser -u mysql -- rsync -a --no-owner --no-group \
  /var/lib/mysql/ /mnt/shared/mysql_data/
```
This copies the existing local database files to NFS. Do not delete the original `/var/lib/mysql` directory.

Verify that the main files exist:
```bash
runuser -u mysql -- ls -lah /mnt/shared/mysql_data
```
Expected examples include:
- `ibdata1`
- `mysql.ibd`
- `auto.cnf`
- `mysql/`
- `performance_schema/`
- `sys/`

Compare sizes:
```bash
du -sh /var/lib/mysql
runuser -u mysql -- du -sh /mnt/shared/mysql_data
```
The sizes may differ because filesystem allocation and reporting can differ. Check for remaining differences with a dry run:
```bash
runuser -u mysql -- rsync -avn --checksum --no-owner --no-group \
  /var/lib/mysql/ /mnt/shared/mysql_data/
```
No listed file transfers is a useful sign that the source and destination contents match.

### Step 14 - Start MySQL on node1 using NFS
Check the configuration:
```bash
grep -E '^(basedir|datadir|socket|pid-file|log-error)=' /etc/my.cnf
```
It must include:
```text
datadir=/mnt/shared/mysql_data
```
Confirm NFS is mounted and MySQL remains stopped:
```bash
findmnt /mnt/shared
systemctl is-active mysqld
```
Start MySQL:
```bash
systemctl start mysqld
```
Check its status:
```bash
systemctl status mysqld --no-pager
```
Check that MySQL responds:
```bash
/usr/local/mysql-8.0.46-linux-glibc2.28-x86_64/bin/mysqladmin \
  --socket=/var/lib/mysql/mysql.sock -uroot -p ping
```
Expected:
```text
mysqld is alive
```
Verify the actual data directory:
```bash
/usr/local/mysql-8.0.46-linux-glibc2.28-x86_64/bin/mysql \
  --socket=/var/lib/mysql/mysql.sock -uroot -p \
  -e "SELECT @@hostname, @@datadir;"
```
Expected datadir:
```text
/mnt/shared/mysql_data/
```
Check databases:
```bash
/usr/local/mysql-8.0.46-linux-glibc2.28-x86_64/bin/mysql \
  --socket=/var/lib/mysql/mysql.sock -uroot -p \
  -e "SHOW DATABASES;"
```
If all checks pass, node1 is using the shared NFS database.

## Part E - Prepare and Test node2

### Step 15 - Verify node2's configuration
Before starting MySQL on node2, verify that node1 is stopped:

Run on: node1
```bash
systemctl stop mysqld
systemctl is-active mysqld
```
Expected:
```text
inactive
```

Run on: node2
```bash
systemctl is-active mysqld
```
It should also be inactive before the switch.

Check the MySQL version:
```bash
/usr/local/mysql-8.0.46-linux-glibc2.28-x86_64/bin/mysqld --version
```
Check configuration:
```bash
cat /etc/my.cnf
```
It must point to:
```text
datadir=/mnt/shared/mysql_data
```
Verify NFS and permissions:
```bash
findmnt /mnt/shared
runuser -u mysql -- test -r /mnt/shared/mysql_data/ibdata1 \
  && echo "MySQL can read shared database files"
runuser -u mysql -- test -w /mnt/shared/mysql_data \
  && echo "MySQL can write to shared database directory"
```

### Step 16 - Fix local log permissions if necessary
The lab encountered this error on node2:
```text
Could not open file '/var/log/mysql/error.log' for error logging: Permission denied
```
If the same error occurs, first stop MySQL:
```bash
systemctl stop mysqld
```
Then repair the local log permissions:
```bash
mkdir -p /var/log/mysql
touch /var/log/mysql/error.log
chown mysql:mysql /var/log/mysql
chmod 750 /var/log/mysql
chown mysql:mysql /var/log/mysql/error.log
chmod 640 /var/log/mysql/error.log
```
If SELinux is enforcing and the error persists, investigate the audit log and file contexts instead of disabling SELinux indiscriminately.

### Step 17 - Start MySQL on node2
Only proceed when node1 is confirmed inactive.

On node2:
```bash
systemctl start mysqld
systemctl status mysqld --no-pager
```
Check the server:
```bash
/usr/local/mysql-8.0.46-linux-glibc2.28-x86_64/bin/mysqladmin \
  --socket=/var/lib/mysql/mysql.sock -uroot -p ping
```
Verify hostname and datadir:
```bash
/usr/local/mysql-8.0.46-linux-glibc2.28-x86_64/bin/mysql \
  --socket=/var/lib/mysql/mysql.sock -uroot -p \
  -e "SELECT @@hostname, @@datadir; SHOW DATABASES;"
```
Expected:
- Hostname: `node2`
- Datadir: `/mnt/shared/mysql_data/`

The same databases are visible.

Do not run `mysqld --initialize` on node2. It is not a fresh database. It must use the database that already exists on NFS.

## Part F - Manual Failover Between Nodes

### Step 18 - Understand the switching process
For a normal manual switch, you must:
1. Stop MySQL on the currently active node.
2. Verify that it is stopped.
3. Start MySQL on the target node.
4. Verify the target node's hostname, datadir, and database contents.

Each node mounts NFS separately. You normally do not need to unmount `/mnt/shared` when stopping MySQL on one node; the other node has its own NFS mount.

### Step 19 - Switch from node2 to node1
First, on node2:
```bash
systemctl stop mysqld
systemctl is-active mysqld
```
Confirm:
```text
inactive
```
Then, on node1:
```bash
systemctl is-active mysqld
```
If inactive, confirm the NFS mount:
```bash
findmnt /mnt/shared
```
Start MySQL:
```bash
systemctl start mysqld
```
Verify:
```bash
/usr/local/mysql-8.0.46-linux-glibc2.28-x86_64/bin/mysql \
  --socket=/var/lib/mysql/mysql.sock -uroot -p \
  -e "SELECT @@hostname, @@datadir; SHOW DATABASES;"
```

### Step 20 - Switch from node1 to node2
On node1:
```bash
systemctl stop mysqld
systemctl is-active mysqld
```
Wait until the result is `inactive`.

On node2:
```bash
systemctl is-active mysqld
```
Confirm that it is inactive, then check NFS:
```bash
findmnt /mnt/shared
```
Start MySQL:
```bash
systemctl start mysqld
```
Verify:
```bash
/usr/local/mysql-8.0.46-linux-glibc2.28-x86_64/bin/mysql \
  --socket=/var/lib/mysql/mysql.sock -uroot -p \
  -e "SELECT @@hostname, @@datadir; SHOW DATABASES;"
```

### Step 21 - Prove that data persists across switches
On the currently active node, create a test database and table if one does not already exist:
```bash
/usr/local/mysql-8.0.46-linux-glibc2.28-x86_64/bin/mysql \
  --socket=/var/lib/mysql/mysql.sock -uroot -p
```
At the MySQL prompt:
```sql
CREATE DATABASE IF NOT EXISTS labdb;
CREATE TABLE IF NOT EXISTS labdb.notes (
    id INT AUTO_INCREMENT PRIMARY KEY,
    msg VARCHAR(100)
);
INSERT INTO labdb.notes (msg)
VALUES ('Shared NFS test');
SELECT * FROM labdb.notes;
EXIT;
```
Now switch MySQL to the other node using Step 19 or Step 20.

On the new active node, run:
```bash
/usr/local/mysql-8.0.46-linux-glibc2.28-x86_64/bin/mysql \
  --socket=/var/lib/mysql/mysql.sock -uroot -p \
  -e "SELECT * FROM labdb.notes;"
```
The inserted row should still exist because both nodes use the same database directory.

### Step 22 - Simulate a node failure: safety warning
A normal stop/start test is not the same as a crash test.

If you turn off a VM while MySQL is active, the old server may have outstanding storage operations or NFS state. Do not assume that waiting a fixed amount of time clears the problem. Do not try to clear locks manually or start the other server blindly.

For this beginner lab, first complete the normal manual switching test. Do not perform crash failover until you have a verified way to ensure the failed node cannot access the shared storage. Pacemaker with correctly configured fencing is the next phase.

## Part G - Troubleshooting

| Problem | Likely cause | What to check |
| --- | --- | --- |
| mount: access denied | Incorrect NFS export or client access rule | On NFS server: `exportfs -v` |
| NFS connection times out | Network or firewall issue | `ping -c 3 10.10.10.200` ; check NFS firewall |
| MySQL cannot write to NFS | Ownership or directory permissions | `runuser -u mysql -- test -w /mnt/shared/mysql_data` |
| mysqladmin: command not found | MySQL binary directory is not in PATH | Use the full path under `/usr/local/mysql-8.0.46-linux-glibc2.28-x86_64/bin/` |
| Could not open file ... error.log: Permission denied | Local log permissions or SELinux context | Inspect `/var/log/mysql`, then inspect SELinux audit records if needed |
| --initialize specified but the data directory has files in it | MySQL was started with an initialization option | Remove `--initialize` from the command or service configuration; do not delete the existing data |
| MySQL cannot start on node2 | Log permissions, wrong config, missing NFS mount, or another startup error | `journalctl -u mysqld -n 60 --no-pager` |
| @@datadir shows /var/lib/mysql/ | Wrong configuration or wrong MySQL instance | Check `/etc/my.cnf` and `my_print_defaults mysqld` |
| Both nodes may be running MySQL | Unsafe switch or automatic restart | Stop and investigate; never allow both to use the shared datadir |
| NFS server fails | Shared storage is unavailable | MySQL availability may be lost on both nodes |
| MySQL starts after reboot unexpectedly | Service is enabled or another startup mechanism is active | Check `systemctl is-enabled mysqld` and service status |

**Useful diagnostic commands:**
```bash
# Service status
systemctl status mysqld --no-pager
# Recent service errors
journalctl -u mysqld -n 60 --no-pager
# MySQL error log
tail -n 50 /var/log/mysql/error.log
# NFS mount
findmnt /mnt/shared
# MySQL version
/usr/local/mysql-8.0.46-linux-glibc2.28-x86_64/bin/mysqld --version
# Effective MySQL options
/usr/local/mysql-8.0.46-linux-glibc2.28-x86_64/bin/my_print_defaults mysqld
```

## Part H - Command Cheat Sheet

| Task | Command |
| --- | --- |
| Check whether MySQL is running | `systemctl is-active mysqld` |
| Stop MySQL | `systemctl stop mysqld` |
| Start MySQL | `systemctl start mysqld` |
| Check service details | `systemctl status mysqld --no-pager` |
| Confirm NFS mount | `findmnt /mnt/shared` |
| Check MySQL version | `/usr/local/mysql-8.0.46-linux-glibc2.28-x86_64/bin/mysqld --version` |
| Check the configured datadir | MySQL query: `SELECT @@datadir;` |
| Check databases | MySQL query: `SHOW DATABASES;` |
| Check MySQL error log | `tail -n 50 /var/log/mysql/error.log` |
| Check NFS exports | `exportfs -v` on the NFS server |
| Create a logical backup | Use the `mysqldump` command in Step 11 |

## Part I - Final Checklist

- [ ] NFS server `10.10.10.200` exports `/srv/nfs-dbstorage/shared`.
- [ ] Both database nodes mount the export at `/mnt/shared`.
- [ ] MySQL UID/GID matches on both nodes.
- [ ] MySQL 8.0.46 is installed on both nodes.
- [ ] Both `/etc/my.cnf` files specify `/mnt/shared/mysql_data`.
- [ ] The original node1 database was backed up before migration.
- [ ] MySQL was stopped before the database files were copied to NFS.
- [ ] MySQL successfully started on node1 using the NFS datadir.
- [ ] MySQL successfully started on node2 using the same NFS datadir.
- [ ] Database contents remained visible after a normal manual switch.
- [ ] MySQL runs on only one node at a time.
- [ ] Independent backups are retained outside the shared NFS storage.
- [ ] Automatic failover and fencing are not assumed to work yet.

## Next project

**[P1 - MySQL HA with Pacemaker and Corosync](P1-mysql-ha-nfs-pacemaker.md)**  
This next project will add cluster management, controlled resource failover, and fencing. The shared NFS server remains a separate failure point, so it will need its own availability and backup plan.


**Current lab status:** Shared NFS storage and manual MySQL switching have been tested successfully. Automatic failover has not yet been configured.
