# MySQL Master-Slave Replication

## 1. What you are building

- **Master (source):** accepts reads and writes. Records every change in the *binary log*.
- **Slave (replica):** copies those changes automatically. Use it for backups and read queries.

Replication is **one-way**: write on the master, never on the slave.

| Role   | Hostname | IP (example)  |
|--------|----------|---------------|
| Master | db-master | 10.10.10.11  |
| Slave  | db-slave  | 10.10.10.12  |

Replace the IPs and hostnames with your own. This guide uses **MySQL 8.0** and **GTID** replication (simplest to set up and fix).

> Note: this is separate from a Pacemaker shared-storage cluster. Replication uses **two independent datadirs**.

---

## 2. Prerequisites (both servers)

```bash
# Set hostnames (run the matching one on each server)
hostnamectl set-hostname db-master
hostnamectl set-hostname db-slave

# Make the names resolvable
cat >> /etc/hosts << 'EOT'
10.10.10.11 db-master
10.10.10.12 db-slave
EOT

# Update the system
dnf update -y
```

### Install MySQL

```bash
dnf install -y mysql-server
systemctl enable --now mysqld
systemctl status mysqld
```

### Secure it and set the root password

```bash
mysql_secure_installation
```

### Open the firewall (both servers)

```bash
firewall-cmd --permanent --add-port=3306/tcp
firewall-cmd --reload
```

> If you use a non-default datadir, SELinux may block MySQL. Check with `ausearch -m avc -ts recent`.

---

## 3. Configure the master (db-master)

Edit the config file:

```bash
vi /etc/my.cnf.d/mysql-server.cnf
```

Add under `[mysqld]`:

```ini
[mysqld]
server-id = 1
log_bin = mysql-bin
binlog_format = ROW
gtid_mode = ON
enforce_gtid_consistency = ON
bind-address = 0.0.0.0
```

Restart:

```bash
systemctl restart mysqld
```

### Create the replication user

```bash
mysql -uroot -p
```

```sql
CREATE USER 'repl'@'10.10.10.12' IDENTIFIED BY 'StrongPass#123';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'10.10.10.12';
FLUSH PRIVILEGES;
```

`10.10.10.12` is the **slave's IP**. This user can only replicate, nothing else.

### Check the master status

```sql
SHOW BINARY LOG STATUS\G
```

(On older 8.0 versions use `SHOW MASTER STATUS\G`.) You should see a binlog file name and position.

---

## 4. Configure the slave (db-slave)

```bash
vi /etc/my.cnf.d/mysql-server.cnf
```

```ini
[mysqld]
server-id = 2
relay_log = relay-bin
gtid_mode = ON
enforce_gtid_consistency = ON
read_only = ON
super_read_only = ON
```

> `server-id` **must be different** on every server.

```bash
systemctl restart mysqld
```

---

## 5. Copy existing data (only if the master already has data)

If both servers are brand new and empty, skip to step 6.

On the **master**:

```bash
mysqldump -uroot -p --all-databases --single-transaction --routines --triggers --events > /root/full.sql
scp /root/full.sql root@10.10.10.12:/root/
```

On the **slave**:

```bash
mysql -uroot -p -e "RESET MASTER;"
mysql -uroot -p < /root/full.sql
```

`RESET MASTER` clears the slave's GTID history so the dump can load correctly.

---

## 6. Connect the slave to the master

On the **slave**:

```bash
mysql -uroot -p
```

```sql
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST='10.10.10.11',
  SOURCE_USER='repl',
  SOURCE_PASSWORD='StrongPass#123',
  SOURCE_AUTO_POSITION=1,
  GET_SOURCE_PUBLIC_KEY=1;

START REPLICA;
```

`GET_SOURCE_PUBLIC_KEY=1` is needed because MySQL 8 uses `caching_sha2_password` by default.

(On MySQL older than 8.0.23 use `CHANGE MASTER TO ... MASTER_HOST` and `START SLAVE`.)

---

## 7. Verify replication

On the **slave**:

```sql
SHOW REPLICA STATUS\G
```

Both must say **Yes**:

```
Replica_IO_Running: Yes
Replica_SQL_Running: Yes
```

Also check `Last_IO_Error` and `Last_SQL_Error` are empty.

### Test it

On the **master**:

```sql
CREATE DATABASE testrepl;
USE testrepl;
CREATE TABLE t1 (id INT PRIMARY KEY, name VARCHAR(50));
INSERT INTO t1 VALUES (1, 'hello');
```

On the **slave**:

```sql
SELECT * FROM testrepl.t1;
```

If you see the row, replication works.

---

## 8. Troubleshooting

| Problem | Likely cause | Fix |
|---|---|---|
| `Replica_IO_Running: Connecting` | Network, firewall, wrong user/password | `telnet 10.10.10.11 3306`, check firewall, retest the `repl` login from the slave |
| `ERROR 1130` host not allowed | No `repl` user for the slave's IP/hostname | Create `'repl'@'<slave-ip>'` on the master |
| `Authentication requires secure connection` | `caching_sha2_password` | Add `GET_SOURCE_PUBLIC_KEY=1` |
| `Replica_SQL_Running: No` | A statement failed on the slave | Read `Last_SQL_Error`; fix the cause (often data written directly on the slave) |
| Same `server-id` error | Both servers have the same ID | Set unique IDs and restart |

Useful commands:

```sql
STOP REPLICA;
START REPLICA;
RESET REPLICA ALL;   -- wipe replication config and start over
```

---

## 9. Good practices

- Never write directly to the slave (`read_only` protects you; `super_read_only` also blocks root).
- Use a dedicated `repl` user, not `root`.
- Use strong passwords; avoid `-p<password>` on the command line.
- Back up regularly. Replication is **not** a backup: a `DROP TABLE` on the master replicates too.
- Monitor lag with `Seconds_Behind_Source` in `SHOW REPLICA STATUS`.
- Master-slave does **not** fail over automatically. For that, use Orchestrator, MySQL InnoDB Cluster, or Pacemaker.

---

## 10. Quick checklist

- [ ] Hostnames and `/etc/hosts` set on both servers
- [ ] MySQL installed and running on both
- [ ] Port 3306 open on both
- [ ] Unique `server-id` on each
- [ ] Binary log and GTID enabled on the master
- [ ] `repl` user created on the master for the slave's IP
- [ ] Existing data copied (if any)
- [ ] `CHANGE REPLICATION SOURCE TO` and `START REPLICA` run on the slave
- [ ] Both `Replica_IO_Running` and `Replica_SQL_Running` are `Yes`
- [ ] Test database replicated
