# MySQL Master–Slave Replication on AlmaLinux  
## Complete Production Guide using MySQL 8.0 + GTID

---

## 1. What You Are Building

This setup creates a one-way MySQL replication topology:

| Role | Hostname | Example IP | Purpose |
|---|---|---:|---|
| Master / Source | `db-master` | `10.10.10.111` | Accepts reads and writes, records changes in binary log |
| Slave / Replica | `db-slave` | `10.10.10.112` | Copies changes automatically, used for reads/backups/failover target |

Important rules:

- Write only to the **Master**.
- Do **not** write directly to the **Slave**.
- The Slave should be read-only.
- Replication is **not a backup**. A `DROP DATABASE` on the Master will also replicate to the Slave.
- This guide uses **GTID replication**, which is the modern and recommended method for MySQL 8.0+.

---

## 2. Lab Values Used in This Document

Replace these with your own values if needed.

```text
Master IP:     10.10.10.111
Slave IP:      10.10.10.112
Master hostname: db-master
Slave hostname:  db-slave
Replication user: repl
Replication password: 123
MySQL version: 8.0.46
Operating system: AlmaLinux / RHEL-like
```

For production, use a stronger password than `123`.

---

## 3. Prerequisites

You need:

- Two servers running AlmaLinux, Rocky Linux, CentOS Stream, or RHEL-like OS.
- Root or sudo access on both servers.
- Network connectivity between Master and Slave.
- MySQL 8.0 installed on both servers.
- Port `3306/tcp` allowed from Slave to Master.
- Unique `server-id` on each server.
- GTID enabled on both servers.

---

# 4. Initial Server Setup on Both Servers

Run these steps on **both Master and Slave**.

## 4.1 Set Hostnames

On Master:

```bash
hostnamectl set-hostname db-master
```

On Slave:

```bash
hostnamectl set-hostname db-slave
```

---

## 4.2 Make Hostnames Resolvable

On both servers:

```bash
cat >> /etc/hosts << 'EOT'
10.10.10.111 db-master
10.10.10.112 db-slave
EOT
```

Verify:

```bash
hostname
cat /etc/hosts
```

---

## 4.3 Update System Packages

```bash
dnf update -y
```

---

## 4.4 Install MySQL

If using distribution packages:

```bash
dnf install -y mysql-server
```

If using Oracle MySQL binary installation, make sure your systemd service is correctly configured.

Enable and start MySQL:

```bash
systemctl enable --now mysqld
systemctl status mysqld
```

---

## 4.5 Secure MySQL Installation

```bash
mysql_secure_installation
```

Recommended answers:

- Set root password: **Yes**
- Remove anonymous users: **Yes**
- Disallow root login remotely: **Yes**
- Remove test database: **Yes**
- Reload privilege tables: **Yes**

---

## 4.6 Open Firewall Port

On both servers:

```bash
firewall-cmd --permanent --add-port=3306/tcp
firewall-cmd --reload
```

Verify:

```bash
firewall-cmd --list-ports
```

Expected output should include:

```text
3306/tcp
```

For stricter security, allow only the Slave IP on the Master:

```bash
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.10.10.112" port protocol="tcp" port="3306" accept'
firewall-cmd --reload
```

---

## 4.7 SELinux Considerations

If SELinux is enforcing, MySQL may be blocked from network connections or non-default data directories.

Allow MySQL connections:

```bash
setsebool -P mysql_connect_any 1
```

If you use a custom datadir, for example:

```text
/var/lib/mysql_data
```

Make sure SELinux context is correct:

```bash
semanage fcontext -a -t mysqld_db_t "/var/lib/mysql_data(/.*)?"
restorecon -Rv /var/lib/mysql_data
```

Check for SELinux denials:

```bash
ausearch -m avc -ts recent
```

---

# 5. MySQL Configuration Files

## Important Issue We Faced

Your system had multiple MySQL configuration locations:

```text
/etc/my.cnf
/etc/my.cnf.d/mysql-server.cnf
```

This can cause confusion because settings may be split across files.

Best practice:

- Keep main MySQL paths in `/etc/my.cnf`.
- Keep replication settings in the same effective `[mysqld]` section, or ensure included files are loaded correctly.
- Avoid typos like:

```ini
bind address
```

Correct syntax is:

```ini
bind-address
```

Also avoid trailing characters like:

```ini
bind-address = 0.0.0.0`
```

That backtick will break MySQL startup.

---

# 6. Configure the Master Server

Edit the Master configuration.

Depending on your installation, use:

```bash
vi /etc/my.cnf
```

or:

```bash
vi /etc/my.cnf.d/mysql-server.cnf
```

Make sure the final effective configuration contains the following under `[mysqld]`.

## 6.1 Master Configuration Example

```ini
[mysqld]
user=mysql
basedir=/usr/local/mysql-8.0.46-linux-glibc2.28-x86_64
datadir=/var/lib/mysql_data
socket=/var/lib/mysql/mysql.sock
pid-file=/run/mysqld/mysqld.pid
log-error=/var/log/mysql/error.log

server-id = 1
log-bin = mysql-bin
binlog-format = ROW
gtid-mode = ON
enforce-gtid-consistency = ON
bind-address = 0.0.0.0

```

Explanation:

| Setting | Purpose |
|---|---|
| `server-id = 1` | Unique ID for Master |
| `log-bin = mysql-bin` | Enables binary logging |
| `binlog-format = ROW` | Safest format for replication |
| `gtid-mode = ON` | Enables GTID replication |
| `enforce-gtid-consistency = ON` | Prevents unsafe transactions for GTID |
| `bind-address = 0.0.0.0` | Allows remote connections |
| `sync-binlog = 1` | Improves durability |
| `innodb-flush-log-at-trx-commit = 1` | Improves crash safety |
| `binlog-expire-logs-seconds` | Automatically removes old binary logs |

---

## 6.2 Restart Master MySQL

```bash
systemctl restart mysqld
systemctl status mysqld
```

If MySQL fails to start, check:

```bash
journalctl -xeu mysqld
tail -n 100 /var/log/mysql/error.log
```

---

## 6.3 Verify Master Variables

Login:

```bash
mysql -uroot -p
```

Run:

```sql
SHOW VARIABLES LIKE 'server_id';
SHOW VARIABLES LIKE 'log_bin';
SHOW VARIABLES LIKE 'gtid_mode';
SHOW VARIABLES LIKE 'enforce_gtid_consistency';
SHOW VARIABLES LIKE 'binlog_format';
```

Expected important values:

```text
server_id = 1
log_bin = ON
gtid_mode = ON
enforce_gtid_consistency = ON
binlog_format = ROW
```

---

## 6.4 Check Master Binary Log Status

On newer MySQL versions, this may work:

```sql
SHOW BINARY LOG STATUS\G
```

But on MySQL 8.0.46, you may get:

```text
ERROR 1064 (42000): You have an error in your SQL syntax...
```

Use the older compatible command:

```sql
SHOW MASTER STATUS\G
```

Example output:

```text
*************************** 1. row ***************************
             File: mysql-bin.000036
         Position: 157
     Binlog_Do_DB: 
 Binlog_Ignore_DB: 
Executed_Gtid_Set: 
```

---

## Issue: `Executed_Gtid_Set` Is Empty

This happened during setup.

### Meaning

An empty `Executed_Gtid_Set` can be normal if no transaction has been recorded since GTID was enabled.

It does not always mean GTID is broken.

### Verify GTID Is Actually Enabled

Run:

```sql
SHOW VARIABLES LIKE 'gtid_mode';
SHOW VARIABLES LIKE 'enforce_gtid_consistency';
```

Both must be:

```text
ON
```

### Force a New Binary Log

```sql
FLUSH LOGS;
SHOW MASTER STATUS\G
```

### Create a Dummy Transaction

```sql
CREATE DATABASE IF NOT EXISTS gtid_test;
DROP DATABASE gtid_test;
SHOW MASTER STATUS\G
```

After this, you should see something like:

```text
Executed_Gtid_Set: 1018d002-c8de-11f1-be5e-00155dc85224:1-2
```

If it is still empty, check the MySQL error log and confirm the correct config file is being loaded.

---

## Issue: `SHOW BINARY LOG STATUS` Gives Syntax Error

Error:

```text
ERROR 1064 (42000): You have an error in your SQL syntax;
check the manual that corresponds to your MySQL server version
for the right syntax to use near 'LOG STATUS'
```

Cause:

Your MySQL version does not support `SHOW BINARY LOG STATUS`.

Solution:

Use:

```sql
SHOW MASTER STATUS\G
```

---

## Issue: MySQL Fails to Start Because of `bind-address` Typo

Error example:

```text
Can't create IP socket: No such file or directory
Can't start server: cannot resolve hostname!
```

Cause:

The config had an invalid value such as:

```ini
bind-address = 0.0.0.0.
```

or:

```ini
bind address = 0.0.0.0
```

Solution:

Use correct syntax:

```ini
bind-address = 0.0.0.0
```

Then restart:

```bash
systemctl restart mysqld
```

---

## Issue: Backtick in Config File Breaks MySQL

Bad line:

```ini
bind-address = 0.0.0.0`
```

Solution:

Remove the backtick:

```ini
bind-address = 0.0.0.0
```

---

# 7. Create Replication User on Master

The replication user is **not the Master server**.

It is a MySQL account created on the Master that allows the Slave to connect and read binary log events.

Think of it like this:

| Concept | Meaning |
|---|---|
| Master | The database server at `10.10.10.111` |
| Replication user | A MySQL account like `repl` |
| Slave | The database server at `10.10.10.112` |
| Password | The secret used by Slave to login as `repl` |

---

## 7.1 Create User for Specific Slave IP

Login to Master MySQL:

```bash
mysql -uroot -p
```

Run:

```sql
DROP USER IF EXISTS 'repl'@'%';
DROP USER IF EXISTS 'repl'@'10.10.10.12';
DROP USER IF EXISTS 'repl'@'10.10.10.112';

CREATE USER 'repl'@'10.10.10.112' IDENTIFIED BY '123';

GRANT REPLICATION SLAVE ON *.* TO 'repl'@'10.10.10.112';

FLUSH PRIVILEGES;
```

Verify:

```sql
SELECT user, host FROM mysql.user WHERE user='repl';
```

Expected:

```text
+------+--------------+
| user | host         |
+------+--------------+
| repl | 10.10.10.112 |
+------+--------------+
```

---

## Issue: Replication User Created for Wrong IP

We initially used:

```sql
'repl'@'10.10.10.12'
```

But the actual Slave IP was:

```text
10.10.10.112
```

MySQL treats these as different hosts:

```text
10.10.10.12
10.10.10.112
```

So the Slave could not authenticate.

Solution:

Create the user for the exact Slave IP:

```sql
CREATE USER 'repl'@'10.10.10.112' IDENTIFIED BY '123';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'10.10.10.112';
FLUSH PRIVILEGES;
```

---

## Issue: Testing `repl` Login from Master Gives ERROR 1130

Command run on Master:

```bash
mysql -h 10.10.10.111 -u repl -p
```

Error:

```text
ERROR 1130 (HY000): Host '10.10.10.111' is not allowed to connect to this MySQL server
```

Cause:

The user was created only for:

```text
'repl'@'10.10.10.112'
```

When you test from the Master itself, MySQL sees the connection coming from:

```text
10.10.10.111
```

That is expected.

Solution:

Test the `repl` user from the **Slave**, not from the Master.

On Slave:

```bash
mysql -h 10.10.10.111 -u repl -p
```

Password:

```text
123
```

---

## Issue: ERROR 1045 Access Denied on Slave

Command on Slave:

```bash
mysql -h 10.10.10.111 -u repl -p
```

Error:

```text
ERROR 1045 (28000): Access denied for user 'repl'@'10.10.10.112' (using password: YES)
```

Cause:

Password mismatch or user recreated incorrectly.

Solution:

On Master, recreate user with correct password:

```sql
DROP USER IF EXISTS 'repl'@'10.10.10.112';

CREATE USER 'repl'@'10.10.10.112' IDENTIFIED BY '123';

GRANT REPLICATION SLAVE ON *.* TO 'repl'@'10.10.10.112';

FLUSH PRIVILEGES;
```

Then test again from Slave:

```bash
mysql -h 10.10.10.111 -u repl -p
```

Enter:

```text
123
```

---

# 8. Configure the Slave Server

Edit Slave configuration:

```bash
vi /etc/my.cnf
```

or:

```bash
vi /etc/my.cnf.d/mysql-server.cnf
```

---

## 8.1 Slave Configuration Example

```ini
[mysqld]
user=mysql
basedir=/usr/local/mysql-8.0.46-linux-glibc2.28-x86_64
datadir=/var/lib/mysql_data
socket=/var/lib/mysql/mysql.sock
pid-file=/run/mysqld/mysqld.pid
log-error=/var/log/mysql/error.log

server-id = 2
relay-log = relay-bin
log-bin = mysql-bin
gtid-mode = ON
enforce-gtid-consistency = ON

read-only = ON
super-read-only = ON

bind-address = 0.0.0.0
```

Explanation:

| Setting | Purpose |
|---|---|
| `server-id = 2` | Must be unique and different from Master |
| `relay-log = relay-bin` | Stores replicated events before applying them |
| `log-bin = mysql-bin` | Useful if Slave may become Master or cascade replicas |
| `gtid-mode = ON` | Required for GTID auto-positioning |
| `enforce-gtid-consistency = ON` | Required for safe GTID replication |
| `read-only = ON` | Prevents normal users from writing |
| `super-read-only = ON` | Prevents even root/SUPER users from writing |
| `bind-address = 0.0.0.0` | Allows remote administrative connections if needed |

---

## Important: During Initial Import, Read-Only Must Be Temporarily Disabled

Because `super-read-only = ON` blocks all writes, including imports, you must temporarily disable it during:

- Data import
- Initial `CHANGE REPLICATION SOURCE TO`
- `RESET MASTER`
- `RESET REPLICA ALL`

You can do this at runtime:

```sql
SET GLOBAL super_read_only = OFF;
SET GLOBAL read_only = OFF;
```

After setup is complete:

```sql
SET GLOBAL read_only = ON;
SET GLOBAL super_read_only = ON;
```

---

## 8.2 Restart Slave MySQL

```bash
systemctl restart mysqld
systemctl status mysqld
```

If it fails:

```bash
journalctl -xeu mysqld
tail -n 100 /var/log/mysql/error.log
```

---

## 8.3 Verify Slave Variables

```bash
mysql -uroot -p
```

Run:

```sql
SHOW VARIABLES LIKE 'server_id';
SHOW VARIABLES LIKE 'gtid_mode';
SHOW VARIABLES LIKE 'enforce_gtid_consistency';
SHOW VARIABLES LIKE 'read_only';
SHOW VARIABLES LIKE 'super_read_only';
SHOW VARIABLES LIKE 'relay_log';
```

Expected important values:

```text
server_id = 2
gtid_mode = ON
enforce_gtid_consistency = ON
relay_log = relay-bin
```

---

# 9. Copy Existing Data from Master to Slave

If both servers are brand new and empty, you can skip to Step 10.

If the Master already has data, you must copy it to the Slave before starting replication.

---

## 9.1 Take Dump on Master

On Master:

```bash
mysqldump -uroot -p \
  --all-databases \
  --single-transaction \
  --routines \
  --triggers \
  --events \
  --set-gtid-purged=ON \
  > /root/full.sql
```

Copy dump to Slave:

```bash
scp /root/full.sql root@10.10.10.112:/root/
```

---

## 9.2 Prepare Slave for Import

On Slave:

```bash
mysql -uroot -p
```

Run:

```sql
STOP REPLICA;

RESET REPLICA ALL;

SET GLOBAL super_read_only = OFF;
SET GLOBAL read_only = OFF;

RESET MASTER;

EXIT;
```

Explanation:

| Command | Purpose |
|---|---|
| `STOP REPLICA` | Stops replication threads |
| `RESET REPLICA ALL` | Removes old replication configuration |
| `SET GLOBAL super_read_only = OFF` | Allows root import |
| `SET GLOBAL read_only = OFF` | Allows writes during import |
| `RESET MASTER` | Clears Slave GTID history so dump can load |

---

## 9.3 Import Dump on Slave

On Slave shell:

```bash
mysql -uroot -p < /root/full.sql
```

If successful, there should be no error output.

---

## 9.4 Verify Data on Slave

```bash
mysql -uroot -p
```

Run:

```sql
SHOW DATABASES;
```

You should see the Master databases.

---

## 9.5 Re-enable Read-Only on Slave

Inside MySQL on Slave:

```sql
SET GLOBAL read_only = ON;
SET GLOBAL super_read_only = ON;
```

Also keep these in the config file for persistence:

```ini
read-only = ON
super-read-only = ON
```

---

# 10. Issues Faced During Data Import

## Issue: ERROR 3546 GTID_PURGED Overlap

Error:

```text
ERROR 3546 (HY000) at line 26:
@@GLOBAL.GTID_PURGED cannot be changed:
the added gtid set must not overlap with @@GLOBAL.GTID_EXECUTED
```

Cause:

The Slave already had GTID history, and the dump tried to set GTID purged values that overlapped.

Solution:

Before importing:

```sql
RESET MASTER;
```

Full safe sequence:

```sql
STOP REPLICA;
RESET REPLICA ALL;
SET GLOBAL super_read_only = OFF;
SET GLOBAL read_only = OFF;
RESET MASTER;
```

Then import:

```bash
mysql -uroot -p < /root/full.sql
```

---

## Issue: ERROR 1290 Super Read-Only Blocks Import

Error:

```text
ERROR 1290 (HY000) at line 32:
The MySQL server is running with the --super-read-only option
so it cannot execute this statement
```

Cause:

`super_read_only = ON` blocks all client writes, including root imports.

Solution:

Temporarily disable:

```sql
SET GLOBAL super_read_only = OFF;
SET GLOBAL read_only = OFF;
```

Import:

```bash
mysql -uroot -p < /root/full.sql
```

Re-enable:

```sql
SET GLOBAL read_only = ON;
SET GLOBAL super_read_only = ON;
```

---

## Issue: ERROR 2002 Cannot Connect to Local MySQL Server

Error:

```text
ERROR 2002 (HY000): Can't connect to local MySQL server through socket...
```

Cause:

MySQL service was stopped.

Solution:

Start MySQL:

```bash
systemctl start mysqld
systemctl status mysqld
```

Then retry import.

---

## Issue: Partial Import Left GTIDs but No Data

This happened when:

1. `RESET MASTER` was run.
2. Import started.
3. Import failed because of `super_read_only`.
4. Some GTID information was recorded.
5. Actual database objects were not fully created.

Result:

Slave believed it had executed GTIDs, but the data was missing.

Example:

Master had:

```sql
CREATE DATABASE test;
```

Slave did not have `test`.

Solution:

Reload cleanly:

On Master:

```bash
mysqldump -uroot -p \
  --all-databases \
  --single-transaction \
  --routines \
  --triggers \
  --events \
  --set-gtid-purged=ON \
  > /root/full.sql

scp /root/full.sql root@10.10.10.112:/root/
```

On Slave:

```sql
STOP REPLICA;
RESET REPLICA ALL;
SET GLOBAL super_read_only = OFF;
SET GLOBAL read_only = OFF;
RESET MASTER;
EXIT;
```

Then:

```bash
mysql -uroot -p < /root/full.sql
```

Verify:

```sql
SHOW DATABASES;
```

Then:

```sql
SET GLOBAL read_only = ON;
SET GLOBAL super_read_only = ON;
```

---

# 11. Connect Slave to Master Using GTID

On Slave:

```bash
mysql -uroot -p
```

Run:

```sql
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST='10.10.10.111',
  SOURCE_USER='repl',
  SOURCE_PASSWORD='123',
  SOURCE_AUTO_POSITION=1,
  GET_SOURCE_PUBLIC_KEY=1;

START REPLICA;

SHOW REPLICA STATUS\G
```

Important settings:

| Option | Meaning |
|---|---|
| `SOURCE_HOST` | Master IP |
| `SOURCE_USER` | Replication user |
| `SOURCE_PASSWORD` | Replication password |
| `SOURCE_AUTO_POSITION=1` | Use GTID auto-positioning |
| `GET_SOURCE_PUBLIC_KEY=1` | Required for MySQL 8 `caching_sha2_password` |

---

## Issue: Wrong Master IP Used

We initially used:

```text
10.10.10.11
```

But the actual Master IP was:

```text
10.10.10.111
```

This caused connection failures.

Solution:

Always verify:

```bash
ip addr
hostname -I
```

Then set correct value:

```sql
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST='10.10.10.111',
  SOURCE_USER='repl',
  SOURCE_PASSWORD='123',
  SOURCE_AUTO_POSITION=1,
  GET_SOURCE_PUBLIC_KEY=1;
```

---

## Issue: ERROR 113 No Route to Host

Error:

```text
Last_IO_Error: Error connecting to source 'repl@10.10.10.111:3306'.
Message: Can't connect to MySQL server on '10.10.10.111:3306' (113)
```

Meaning:

The Slave cannot reach the Master on port 3306.

This is usually a firewall, SELinux, or wrong IP issue.

---

### Solution 1: Verify Master IP

On Master:

```bash
ip addr
hostname -I
```

Make sure it is actually:

```text
10.10.10.111
```

---

### Solution 2: Open Firewall on Master

On Master:

```bash
firewall-cmd --permanent --add-port=3306/tcp
firewall-cmd --reload
firewall-cmd --list-ports
```

Expected:

```text
3306/tcp
```

---

### Solution 3: Check SELinux on Master

On Master:

```bash
setsebool -P mysql_connect_any 1
```

Check denials:

```bash
ausearch -m avc -ts recent
```

---

### Solution 4: Verify MySQL Is Listening

On Master:

```bash
ss -lntp | grep 3306
```

You should see MySQL listening on:

```text
0.0.0.0:3306
```

or:

```text
10.10.10.111:3306
```

If it is listening only on:

```text
127.0.0.1:3306
```

then fix `bind-address`:

```ini
bind-address = 0.0.0.0
```

Then restart:

```bash
systemctl restart mysqld
```

---

### Solution 5: Test from Slave Shell

On Slave:

```bash
telnet 10.10.10.111 3306
```

or:

```bash
nc -zv 10.10.10.111 3306
```

If connection succeeds, network is fine.

If it fails, the issue is still firewall/network/SELinux.

---

### Solution 6: Test MySQL Login from Slave Shell

On Slave:

```bash
mysql -h 10.10.10.111 -u repl -p
```

Password:

```text
123
```

If this works, replication should work.

---

# 12. Verify Replication Status

On Slave:

```bash
mysql -uroot -p
```

Run:

```sql
SHOW REPLICA STATUS\G
```

You must see:

```text
Replica_IO_Running: Yes
Replica_SQL_Running: Yes
```

Also check:

```text
Last_IO_Error:
Last_SQL_Error:
```

Both should be empty.

---

## Issue: Replica_IO_Running = Connecting

Example:

```text
Replica_IO_State: Connecting to source
Replica_IO_Running: Connecting
```

Meaning:

Slave is trying to connect to Master but has not succeeded yet.

Possible causes:

- Wrong Master IP
- Firewall blocking 3306
- SELinux blocking connection
- Wrong replication user host
- Wrong password
- MySQL not listening on remote interface

Solutions:

Check:

```sql
SHOW REPLICA STATUS\G
```

Look at:

```text
Last_IO_Errno
Last_IO_Error
```

Then test manually from Slave shell:

```bash
mysql -h 10.10.10.111 -u repl -p
```

---

## Issue: Replica_SQL_Running = No

Meaning:

Slave connected to Master, but failed while applying replicated statements.

Check:

```sql
SHOW REPLICA STATUS\G
```

Look at:

```text
Last_SQL_Errno
Last_SQL_Error
```

Common causes:

- Data written directly on Slave
- Table already exists with different structure
- Missing database/table
- GTID inconsistency
- Partial import

Solution:

Usually rebuild Slave cleanly from Master dump.

---

# 13. Critical Issue: Same Server UUID on Master and Slave

This was one of the major problems.

## Symptoms

Replication did not work even though:

- Firewall was open
- User existed
- Password was correct
- GTID was enabled

The Slave had the same MySQL server UUID as the Master.

Example UUID:

```text
1018d002-c8de-11f1-be5e-00155dc85224
```

If both Master and Slave have the same UUID, MySQL may treat them as the same server, especially with GTID replication.

---

## Root Cause

The Slave datadir was likely copied from the Master or created from a clone/template.

MySQL stores its server UUID in:

```text
<datadir>/auto.cnf
```

If the Slave has the same `auto.cnf` as the Master, it will have the same UUID.

Important:

This is different from `server-id`.

| Value | Where it comes from |
|---|---|
| `server-id` | Config file, example: 1 or 2 |
| `server_uuid` | Generated in `auto.cnf` inside datadir |

Both must be unique.

---

## Find the Real Datadir

On Slave:

```bash
mysql -uroot -p -e "SELECT @@datadir;"
```

Example output:

```text
/var/lib/mysql_data/
```

Find `auto.cnf`:

```bash
find / -name auto.cnf 2>/dev/null
```

Example:

```text
/var/lib/mysql_data/auto.cnf
```

---

## Fix Duplicate UUID on Slave

Only do this on the Slave.

Stop MySQL:

```bash
systemctl stop mysqld
```

Delete `auto.cnf`:

```bash
rm -f /var/lib/mysql_data/auto.cnf
```

Start MySQL:

```bash
systemctl start mysqld
```

MySQL will generate a new UUID automatically.

Verify:

```bash
mysql -uroot -p -e "SELECT @@server_uuid;"
```

The Slave UUID must now be different from the Master UUID.

Check Master UUID too:

```bash
mysql -uroot -p -e "SELECT @@server_uuid;"
```

They must not match.

---

## After UUID Fix, Reset Replication on Slave

On Slave:

```bash
mysql -uroot -p
```

Run:

```sql
STOP REPLICA;
RESET REPLICA ALL;

CHANGE REPLICATION SOURCE TO
  SOURCE_HOST='10.10.10.111',
  SOURCE_USER='repl',
  SOURCE_PASSWORD='123',
  SOURCE_AUTO_POSITION=1,
  GET_SOURCE_PUBLIC_KEY=1;

START REPLICA;

SHOW REPLICA STATUS\G
```

---

# 14. Final Test

## 14.1 Create Test Data on Master

On Master:

```bash
mysql -uroot -p
```

Run:

```sql
CREATE DATABASE final_test;
USE final_test;

CREATE TABLE t1 (
  id INT PRIMARY KEY,
  name VARCHAR(50)
);

INSERT INTO t1 VALUES (1, 'replication-working');
```

---

## 14.2 Check on Slave

On Slave:

```bash
mysql -uroot -p
```

Run:

```sql
SHOW DATABASES;
```

You should see:

```text
final_test
```

Then:

```sql
SELECT * FROM final_test.t1;
```

Expected:

```text
+----+---------------------+
| id | name                |
+----+---------------------+
|  1 | replication-working |
+----+---------------------+
```

If you see the row, replication is working.

---

# 15. Slave Read-Only Behavior

## Expected Behavior

The Slave should be read-only.

If you try to write on Slave:

```sql
INSERT INTO final_test.t1 VALUES (2, 'direct-write');
```

You may get:

```text
ERROR 1290 (HY000): The MySQL server is running with the --super-read-only option
so it cannot execute this statement
```

This is correct.

Do not disable read-only for normal operation.

Only disable it temporarily for:

- Initial import
- Maintenance
- Rebuilding replication

---

# 16. Complete Troubleshooting Table

| Error / Symptom | Likely Cause | Solution |
|---|---|---|
| `SHOW BINARY LOG STATUS` syntax error | MySQL 8.0 version does not support it | Use `SHOW MASTER STATUS\G` |
| `Executed_Gtid_Set` empty | No transaction yet or GTID not loaded | Check `gtid_mode`, restart MySQL, run dummy transaction |
| MySQL fails to start | Bad config syntax | Check `/var/log/mysql/error.log`, fix typo |
| `bind address` error | Missing hyphen | Use `bind-address` |
| `0.0.0.0.` error | Invalid IP typo | Use `0.0.0.0` |
| Backtick in config | Invalid character | Remove backtick |
| `ERROR 113 No route to host` | Firewall/network/SELinux/wrong IP | Open 3306, verify IP, test telnet |
| `ERROR 1130 Host not allowed` | Replication user host mismatch | Create `'repl'@'<slave-ip>'` |
| `ERROR 1045 Access denied` | Wrong password/user | Recreate user with correct password |
| `ERROR 1290 super-read-only` | Import blocked by read-only | Temporarily disable `super_read_only` |
| `ERROR 3546 GTID_PURGED overlap` | Slave already has GTIDs | Run `RESET MASTER` before import |
| `ERROR 2002 Cannot connect local socket` | MySQL stopped | Start `mysqld` |
| `Replica_IO_Running: Connecting` | Slave cannot reach Master | Check firewall, IP, user, password |
| `Replica_SQL_Running: No` | Apply error on Slave | Check `Last_SQL_Error`, often rebuild Slave |
| Same UUID on Master/Slave | Copied datadir/auto.cnf | Delete Slave `auto.cnf`, restart MySQL |
| Data missing on Slave | Partial import or GTID inconsistency | Reload clean dump |
| Slave accepts writes | Read-only not enabled | Set `read_only` and `super_read_only` |
| Old commands fail | Using MySQL 5.x syntax | Use `CHANGE REPLICATION SOURCE TO`, `START REPLICA` |

---

# 17. Useful Diagnostic Commands

## Check MySQL Service

```bash
systemctl status mysqld
```

## Check Error Log

```bash
tail -n 100 /var/log/mysql/error.log
```

## Check Journal

```bash
journalctl -xeu mysqld
```

## Check Listening Ports

```bash
ss -lntp | grep 3306
```

## Check Firewall

```bash
firewall-cmd --list-ports
firewall-cmd --list-all
```

## Test Network from Slave

```bash
telnet 10.10.10.111 3306
```

or:

```bash
nc -zv 10.10.10.111 3306
```

## Test Replication Login from Slave

```bash
mysql -h 10.10.10.111 -u repl -p
```

## Check Replication User on Master

```sql
SELECT user, host FROM mysql.user WHERE user='repl';
```

## Check GTID Variables

```sql
SHOW VARIABLES LIKE 'gtid_mode';
SHOW VARIABLES LIKE 'enforce_gtid_consistency';
```

## Check Server UUID

```sql
SELECT @@server_uuid;
```

## Check Datadir

```sql
SELECT @@datadir;
```

## Check Replica Status

```sql
SHOW REPLICA STATUS\G
```

---

# 18. Reset and Rebuild Replication Safely

If replication becomes badly broken, rebuild the Slave cleanly.

## On Master

Take fresh dump:

```bash
mysqldump -uroot -p \
  --all-databases \
  --single-transaction \
  --routines \
  --triggers \
  --events \
  --set-gtid-purged=ON \
  > /root/full.sql
```

Copy to Slave:

```bash
scp /root/full.sql root@10.10.10.112:/root/
```

---

## On Slave

Stop and reset:

```bash
mysql -uroot -p
```

Run:

```sql
STOP REPLICA;
RESET REPLICA ALL;

SET GLOBAL super_read_only = OFF;
SET GLOBAL read_only = OFF;

RESET MASTER;

EXIT;
```

Import:

```bash
mysql -uroot -p < /root/full.sql
```

Verify:

```bash
mysql -uroot -p -e "SHOW DATABASES;"
```

Enable read-only:

```bash
mysql -uroot -p
```

Run:

```sql
SET GLOBAL read_only = ON;
SET GLOBAL super_read_only = ON;
```

Configure replication:

```sql
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST='10.10.10.111',
  SOURCE_USER='repl',
  SOURCE_PASSWORD='123',
  SOURCE_AUTO_POSITION=1,
  GET_SOURCE_PUBLIC_KEY=1;

START REPLICA;

SHOW REPLICA STATUS\G
```

---

# 19. Modern MySQL 8.0 Syntax vs Old Syntax

Use modern MySQL 8.0 commands.

| Old / Deprecated | Modern MySQL 8.0 |
|---|---|
| `CHANGE MASTER TO` | `CHANGE REPLICATION SOURCE TO` |
| `START SLAVE` | `START REPLICA` |
| `STOP SLAVE` | `STOP REPLICA` |
| `SHOW SLAVE STATUS` | `SHOW REPLICA STATUS` |
| `MASTER_HOST` | `SOURCE_HOST` |
| `MASTER_USER` | `SOURCE_USER` |
| `MASTER_PASSWORD` | `SOURCE_PASSWORD` |
| `MASTER_LOG_FILE` | `SOURCE_LOG_FILE` |
| `MASTER_LOG_POS` | `SOURCE_LOG_POS` |

Because this guide uses GTID, you should use:

```sql
SOURCE_AUTO_POSITION=1
```

Instead of manually copying:

```sql
SOURCE_LOG_FILE
SOURCE_LOG_POS
```

---

# 20. Why GTID Is Better Than File/Position Replication

Old method:

```sql
SOURCE_LOG_FILE='mysql-bin.000036',
SOURCE_LOG_POS=157
```

Problem:

If Master restarts or binary logs rotate, you may lose the correct position.

GTID method:

```sql
SOURCE_AUTO_POSITION=1
```

Benefit:

MySQL automatically tracks transactions using unique IDs.

This makes:

- Failover easier
- Recovery easier
- Replica rebuilding easier
- Replication less fragile

---

# 21. Security Best Practices

## Use Dedicated Replication User

Do not use root for replication.

Use:

```sql
CREATE USER 'repl'@'10.10.10.112' IDENTIFIED BY '123';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'10.10.10.112';
```

For production, use a strong password.

---

## Restrict Replication User to Slave IP

Bad:

```sql
'repl'@'%'
```

Better:

```sql
'repl'@'10.10.10.112'
```

This allows only your Slave to connect using that user.

---

## Firewall Restriction

Instead of opening 3306 to everyone, allow only Slave:

```bash
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.10.10.112" port protocol="tcp" port="3306" accept'
firewall-cmd --reload
```

---

## Keep Slave Read-Only

Config:

```ini
read-only = ON
super-read-only = ON
```

Runtime:

```sql
SET GLOBAL read_only = ON;
SET GLOBAL super_read_only = ON;
```

---

# 22. Monitoring Replication

Check Slave status:

```sql
SHOW REPLICA STATUS\G
```

Important fields:

```text
Replica_IO_Running
Replica_SQL_Running
Seconds_Behind_Source
Last_IO_Error
Last_SQL_Error
Executed_Gtid_Set
Retrieved_Gtid_Set
```

Healthy output:

```text
Replica_IO_Running: Yes
Replica_SQL_Running: Yes
Seconds_Behind_Source: 0
Last_IO_Error:
Last_SQL_Error:
```

Check replication performance schema:

```sql
SELECT * FROM performance_schema.replication_connection_status\G
SELECT * FROM performance_schema.replication_applier_status\G
```

---

# 23. Backup Reminder

Replication is not backup.

If someone runs:

```sql
DROP DATABASE important_db;
```

on Master, it will replicate to Slave.

You still need:

- Scheduled `mysqldump`
- MySQL Enterprise Backup / `mysqlbackup`
- XtraBackup / MariaDB Backup depending on environment
- Offsite backups
- Point-in-time recovery using binary logs

---

# 24. Failover Notes

Basic Master–Slave replication does not fail over automatically.

If Master dies, you can manually promote Slave:

On Slave:

```sql
STOP REPLICA;
RESET REPLICA ALL;
SET GLOBAL read_only = OFF;
SET GLOBAL super_read_only = OFF;
```

Then point applications to the new Master.

For automatic failover, consider:

- MySQL InnoDB Cluster
- MySQL Group Replication
- Orchestrator
- ProxySQL
- Patroni-like tools for PostgreSQL, not MySQL
- Pacemaker + resource agents, but more complex

---

# 25. Final Quick Checklist

Use this checklist to confirm everything is correct.

```text
[ ] Master hostname is db-master
[ ] Slave hostname is db-slave
[ ] /etc/hosts resolves both names
[ ] MySQL installed and running on both
[ ] Port 3306 open on Master firewall
[ ] SELinux not blocking MySQL
[ ] Master server-id = 1
[ ] Slave server-id = 2
[ ] Master UUID and Slave UUID are different
[ ] gtid_mode = ON on both
[ ] enforce_gtid_consistency = ON on both
[ ] log_bin enabled on Master
[ ] relay_log configured on Slave
[ ] repl user created on Master for exact Slave IP
[ ] repl password matches Slave configuration
[ ] Existing data dumped from Master
[ ] Slave RESET MASTER done before import
[ ] super_read_only temporarily disabled during import
[ ] Import completed without errors
[ ] Slave data matches Master
[ ] super_read_only re-enabled on Slave
[ ] CHANGE REPLICATION SOURCE TO executed on Slave
[ ] SOURCE_AUTO_POSITION=1 used
[ ] GET_SOURCE_PUBLIC_KEY=1 used
[ ] START REPLICA executed
[ ] Replica_IO_Running = Yes
[ ] Replica_SQL_Running = Yes
[ ] Last_IO_Error empty
[ ] Last_SQL_Error empty
[ ] Test database created on Master
[ ] Test database appeared on Slave
```

---

# 26. Final Working Configuration Summary

## Master: `/etc/my.cnf` or `/etc/my.cnf.d/mysql-server.cnf`

```ini
[mysqld]
user=mysql
basedir=/usr/local/mysql-8.0.46-linux-glibc2.28-x86_64
datadir=/var/lib/mysql_data
socket=/var/lib/mysql/mysql.sock
pid-file=/run/mysqld/mysqld.pid
log-error=/var/log/mysql/error.log

server-id = 1
log-bin = mysql-bin
binlog-format = ROW
gtid-mode = ON
enforce-gtid-consistency = ON
bind-address = 0.0.0.0

sync-binlog = 1
innodb-flush-log-at-trx-commit = 1
binlog-expire-logs-seconds = 604800
max-binlog-size = 100M
```

## Slave: `/etc/my.cnf` or `/etc/my.cnf.d/mysql-server.cnf`

```ini
[mysqld]
user=mysql
basedir=/usr/local/mysql-8.0.46-linux-glibc2.28-x86_64
datadir=/var/lib/mysql_data
socket=/var/lib/mysql/mysql.sock
pid-file=/run/mysqld/mysqld.pid
log-error=/var/log/mysql/error.log

server-id = 2
relay-log = relay-bin
log-bin = mysql-bin
gtid-mode = ON
enforce-gtid-consistency = ON

read-only = ON
super-read-only = ON

bind-address = 0.0.0.0
```

## Replication User on Master

```sql
CREATE USER 'repl'@'10.10.10.112' IDENTIFIED BY '123';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'10.10.10.112';
FLUSH PRIVILEGES;
```

## Replication Connection on Slave

```sql
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST='10.10.10.111',
  SOURCE_USER='repl',
  SOURCE_PASSWORD='123',
  SOURCE_AUTO_POSITION=1,
  GET_SOURCE_PUBLIC_KEY=1;

START REPLICA;
```

## Verification

```sql
SHOW REPLICA STATUS\G
```

Must show:

```text
Replica_IO_Running: Yes
Replica_SQL_Running: Yes
```

---

# 27. Summary of the Main Issues We Faced and Fixed

1. **Used MariaDB article by mistake**  
   Solution: Ignore MariaDB guides. Use MySQL 8.0 GTID guide.

2. **Used old commands like `START SLAVE` / `CHANGE MASTER TO`**  
   Solution: Use `START REPLICA` and `CHANGE REPLICATION SOURCE TO`.

3. **`SHOW BINARY LOG STATUS` syntax error**  
   Solution: Use `SHOW MASTER STATUS\G` on MySQL 8.0.

4. **`Executed_Gtid_Set` empty**  
   Solution: Confirm GTID variables, restart MySQL, flush logs, create dummy transaction.

5. **Config typo `bind address` / `0.0.0.0.` / backtick**  
   Solution: Use valid syntax: `bind-address = 0.0.0.0`.

6. **MySQL failed to start**  
   Solution: Check `/var/log/mysql/error.log` and `journalctl -xeu mysqld`.

7. **Wrong Master IP used: `10.10.10.11` instead of `10.10.10.111`**  
   Solution: Correct `SOURCE_HOST`.

8. **Wrong Slave IP used for replication user: `10.10.10.12` instead of `10.10.10.112`**  
   Solution: Recreate user as `'repl'@'10.10.10.112'`.

9. **Firewall error 113: No route to host**  
   Solution: Open port 3306, check SELinux, verify IP, test with telnet/nc.

10. **Access denied for `repl` user**  
   Solution: Recreate user with correct password `123`.

11. **Import failed due GTID overlap**  
   Solution: Run `RESET MASTER` on Slave before import.

12. **Import failed due `super_read_only`**  
   Solution: Temporarily disable `super_read_only` and `read_only`.

13. **Partial import left Slave inconsistent**  
   Solution: Reload clean dump after resetting Slave.

14. **Same server UUID on Master and Slave**  
   Solution: Delete Slave `auto.cnf`, restart MySQL, generate new UUID.

15. **Slave read-only confusion**  
   Solution: Read-only is correct for Slave. Disable only temporarily during setup/import.

---

This document now includes:

- Original setup guide
- Modern MySQL 8.0 GTID method
- AlmaLinux/RHEL commands
- Exact working configuration
- All issues faced
- Root causes
- Proper solutions
- Final verification
- Troubleshooting reference
- Reset/rebuild procedure
