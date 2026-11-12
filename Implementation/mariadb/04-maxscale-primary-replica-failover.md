# MaxScale Step by Step with MariaDB: From One Database to Primary, Replica and Automatic Failover

A learning path. You start with **one MariaDB database behind MaxScale**, learn how every line of the configuration works, then **add a replica**, then see **read/write splitting, lag protection, planned switchover and automatic failover**, one small stage at a time.

```
 Stage 1   One database behind MaxScale           (learn the config)
 Stage 2   Experiments on the single server       (see how it reacts)
 Stage 3   Add a replica                          (see what changes in config)
 Stage 4   Watch read/write splitting
 Stage 5   Replication lag and "read your own writes"
 Stage 6   Maintenance mode and a read-only port
 Stage 7   Planned switchover
 Stage 8   Automatic failover
```

> Commands are for **AlmaLinux / RHEL / Rocky** (service `mariadb`). Edit files with `sudo vim FILE` (`i` to type, `Esc` then `:wq` to save).
> **MariaDB is MaxScale's home ground:** every feature in this guide works, including automatic failover and `causal_reads`. (With MySQL some of these do not. There is a separate MySQL guide.)
> **License:** MaxScale 2.x and newer use the Business Source License. It is free for testing, and free in production only for a limited number of database servers (the published limit has been two). This guide uses at most two. Read the `LICENSE.TXT` of the version you install.

---

## 0. The Idea in 60 Seconds

```
 STAGE 1 (one database)                 STAGE 3 (primary + replica)

   App --> MaxScale --> DB1               App --> MaxScale --+--> DB1 (primary)
                                                             |
                                                             +--> DB2 (replica)
```

The application connects to **MaxScale's address and port** and never needs to know how many databases are behind it. You can add, remove and maintain databases **without touching the application**.

### The four objects you will configure

```
 CLIENT
   |
   v
 +----------+     +-----------+     +----------+
 | LISTENER | --> |  SERVICE  | --> |  SERVER  |   <--- watched by --- MONITOR
 | port 4006|     | readwrite |     |   db1    |
 +----------+     |  split    |     +----------+
                  +-----------+
```

| Object | Plain meaning | Needs |
|---|---|---|
| **server** | One real database (address and port) | IP, port |
| **monitor** | A watcher that checks every server's health and role | the list of servers, a login |
| **service** | The brain: decides where each query goes | a router type, the monitor (cluster), a login |
| **listener** | The door that clients knock on | a port and the service it belongs to |

Objects **refer to each other by name**. Read any config from the top down:

```
 [Read-Write-Listener]  service=Read-Write-Service
 [Read-Write-Service]   cluster=MariaDB-Monitor
 [MariaDB-Monitor]      servers=db1
 [db1]                  address=10.10.10.201
```

---

## Stage 1: One Database Behind MaxScale

### 1.0 Environment layout

```
   Client machine ------> MAXSCALE (10.10.10.203, port 4006) ------> DB1 (10.10.10.201, port 3306)
```
Replace the IPs with yours. Two machines for Stage 1. A third (db2) is added in Stage 3.

### 1.1 Install and prepare db1 (MariaDB)

```bash
sudo dnf install mariadb-server -y
mariadb --version              # note the version, it matters for GRANT names
```

#### Plan ahead: prepare db1 for a future replica now

We will add a replica in Stage 3 and failover in Stage 8. Some settings need a **restart**, so set them **now**, before first start. Later you will not need to restart db1.

```bash
sudo vim /etc/my.cnf.d/mariadb-server.cnf
```
Under `[mysqld]` add:
```ini
[mysqld]
bind-address      = 0.0.0.0
server_id         = 1
log_bin           = mariadb-bin
binlog_format     = ROW
log_slave_updates = ON
binlog_expire_logs_seconds = 604800
report_host       = 10.10.10.201
```

| Setting | Why it is here |
|---|---|
| `bind-address = 0.0.0.0` | Accept connections from other machines (MaxScale) |
| `server_id` | Unique number per server, required for replication |
| `log_bin` | Records changes so a replica can copy them. MariaDB then tracks **GTIDs** automatically |
| `binlog_format = ROW` | Safest format for replication |
| `log_slave_updates` | Lets a server pass on changes it received, needed so a replica can become primary later |
| `binlog_expire_logs_seconds` | Keep 7 days of binary logs (MariaDB 10.6+; on older versions use `expire_logs_days = 7`) |
| `report_host` | Shows the server's real address in replication status |

Start MariaDB and open the firewall:
```bash
sudo systemctl enable --now mariadb
sudo firewall-cmd --permanent --add-port=3306/tcp
sudo firewall-cmd --reload
```

### 1.2 Create the users on db1

```bash
sudo mysql
```
Check your version first, because privilege names depend on it:
```sql
SELECT VERSION();
```

```sql
-- Monitor user: lets MaxScale look at the server and its replication status
CREATE USER 'maxmon'@'10.10.10.%' IDENTIFIED BY 'MonPass#123';
GRANT REPLICA MONITOR ON *.* TO 'maxmon'@'10.10.10.%';
GRANT SELECT ON mysql.user        TO 'maxmon'@'10.10.10.%';
GRANT SELECT ON mysql.global_priv TO 'maxmon'@'10.10.10.%';

-- Service user: lets MaxScale read the account list to log clients in
CREATE USER 'maxsvc'@'10.10.10.%' IDENTIFIED BY 'SvcPass#123';
GRANT SELECT ON mysql.user          TO 'maxsvc'@'10.10.10.%';
GRANT SELECT ON mysql.global_priv   TO 'maxsvc'@'10.10.10.%';
GRANT SELECT ON mysql.db            TO 'maxsvc'@'10.10.10.%';
GRANT SELECT ON mysql.tables_priv   TO 'maxsvc'@'10.10.10.%';
GRANT SELECT ON mysql.columns_priv  TO 'maxsvc'@'10.10.10.%';
GRANT SELECT ON mysql.proxies_priv  TO 'maxsvc'@'10.10.10.%';
GRANT SELECT ON mysql.roles_mapping TO 'maxsvc'@'10.10.10.%';
GRANT SHOW DATABASES ON *.*         TO 'maxsvc'@'10.10.10.%';

-- A normal application user, to test with
CREATE DATABASE labtest;
CREATE USER 'appuser'@'%' IDENTIFIED BY 'AppPass#123';
GRANT ALL PRIVILEGES ON labtest.* TO 'appuser'@'%';
```

**Privilege names depend on your MariaDB version:**

| MariaDB version | Replace `REPLICA MONITOR` with |
|---|---|
| 10.5.9 and newer (including 10.6, 10.11, 11.x) | `REPLICA MONITOR` (as above) |
| 10.5.0 to 10.5.8 | `REPLICATION SLAVE ADMIN` |
| 10.4 and older | `REPLICATION CLIENT` (and skip the `mysql.global_priv` grants) |

If a `GRANT` fails, check the MaxScale documentation for the **MariaDB Monitor** and your version.

Why three kinds of users?
```
 maxmon   <-- the MONITOR logs in with this to ask "are you alive? what is your role?"
 maxsvc   <-- the SERVICE logs in with this to copy the user list (so MaxScale can check client passwords itself)
 appuser  <-- your APPLICATION (this is who connects through MaxScale)
```

**Important:** MaxScale connects to the database **on behalf of the client**, so the database sees connections from **MaxScale's IP (10.10.10.203)**, not the client's IP. An account like `'appuser'@'10.10.10.50'` would be refused. Use `'%'` or include MaxScale's address in the host pattern.

### 1.3 Install MaxScale (on the MaxScale machine)

```bash
curl -LsS https://r.mariadb.com/downloads/mariadb_repo_setup | sudo bash
sudo dnf install -y maxscale
maxscale --version
sudo firewall-cmd --permanent --add-port=4006/tcp
sudo firewall-cmd --reload
```
(The first command adds MariaDB's package repository. It does not install a database server on this machine. Do **not** open port 8989; that is the admin API.)

### 1.4 Write the configuration

```bash
sudo cp /etc/maxscale.cnf /etc/maxscale.cnf.original
sudo vim /etc/maxscale.cnf
```
Delete everything (`Esc`, `:%d`, `Enter`, then `i`) and paste:

```ini
[maxscale]
threads=auto

[db1]
type=server
address=10.10.10.201
port=3306

[MariaDB-Monitor]
type=monitor
module=mariadbmon
servers=db1
user=maxmon
password=MonPass#123
monitor_interval=2s

[Read-Write-Service]
type=service
router=readwritesplit
cluster=MariaDB-Monitor
user=maxsvc
password=SvcPass#123

[Read-Write-Listener]
type=listener
service=Read-Write-Service
protocol=MariaDBClient
port=4006
```

### 1.5 Read it line by line

```
 [maxscale]                     GLOBAL settings for MaxScale itself
   threads=auto                   use as many worker threads as there are CPU cores

 [db1]                          a section NAME you choose (db1)
   type=server                    "this section describes a database server"
   address / port                 where MariaDB listens

 [MariaDB-Monitor]              a name you choose
   type=monitor                   "this is a watcher"
   module=mariadbmon              which kind of watcher (the replication monitor)
   servers=db1                    which servers to watch (names of [server] sections)
   user / password                the maxmon account
   monitor_interval=2s            check every 2 seconds

 [Read-Write-Service]           a name you choose
   type=service                   "this is the brain"
   router=readwritesplit          split writes (primary) from reads (replicas)
   cluster=MariaDB-Monitor        use the servers this monitor looks after
   user / password                the maxsvc account

 [Read-Write-Listener]          a name you choose
   type=listener                  "this is a door"
   service=Read-Write-Service     the service behind this door
   protocol=MariaDBClient         the language clients speak
   port=4006                      clients connect to this port
```

Key ideas:
- **Section names are yours to choose.** What matters is that references match exactly, including capitals.
- `type=` tells MaxScale what a section is.
- `cluster=MariaDB-Monitor` means "whatever servers that monitor watches, I use". That is why, in Stage 3, you only add the new server in **one** place.

> **Version note:** `cluster=` works in MaxScale 22.08 or later. On older versions put `servers=db1` in the service instead. Older releases call the listener protocol `MySQLClient`.

### 1.6 Check the syntax, then start

```bash
sudo maxscale --config-check
sudo systemctl enable --now maxscale
sudo systemctl status maxscale
```

### 1.7 Look at what MaxScale sees

```bash
maxctrl list servers
```
```
┌────────┬───────────────┬──────┬─────────────┬─────────────────┬───────┐
│ Server │ Address       │ Port │ Connections │ State           │ GTID  │
├────────┼───────────────┼──────┼─────────────┼─────────────────┼───────┤
│ db1    │ 10.10.10.201  │ 3306 │ 0           │ Master, Running │ ...   │
└────────┴───────────────┴──────┴─────────────┴─────────────────┴───────┘
```

| State | Meaning |
|---|---|
| `Master, Running` | Healthy and acting as the primary |
| `Down` | MaxScale cannot reach it or cannot log in |
| `Running` (no role) | Up, but no role assigned. Check the log |

A lone server normally shows `Master, Running`. If yours only shows `Running`, read `/var/log/maxscale/maxscale.log`, because role rules differ slightly between versions.

More views:
```bash
maxctrl list services
maxctrl list listeners
maxctrl list monitors
maxctrl show server db1
```

### 1.8 Connect through MaxScale

From a client machine with the `mysql` or `mariadb` client:
```bash
mysql -h 10.10.10.203 -P 4006 -u appuser -p
```
```sql
SELECT @@hostname;                 -- db1's hostname: you are talking to db1 THROUGH MaxScale
USE labtest;
CREATE TABLE notes (id INT AUTO_INCREMENT PRIMARY KEY, txt VARCHAR(50));
INSERT INTO notes (txt) VALUES ('first row, one database');
SELECT * FROM notes;
```
In another terminal on the MaxScale machine:
```bash
maxctrl list sessions
maxctrl list servers          # Connections = 1
```

**Checkpoint 1:** you connected to port 4006 but the data lives on db1. Your application would use exactly this address. `readwritesplit` is already in charge; with one server it has nowhere else to send reads, so **everything goes to db1**.

---

## Stage 2: Experiments on the Single Server

Learn by breaking things.

### Experiment 2.1: Stop the database

On db1:
```bash
sudo systemctl stop mariadb
```
On MaxScale:
```bash
maxctrl list servers          # State changes to Down within a few seconds
sudo tail -n 15 /var/log/maxscale/maxscale.log
```
Connecting through 4006 now fails (no primary). Start it again:
```bash
sudo systemctl start mariadb
maxctrl list servers          # back to Master, Running
```
**Lesson:** the monitor checks every `monitor_interval` (2s) and updates states by itself. Nothing was restarted on MaxScale.

### Experiment 2.2: Wrong monitor password

```bash
maxctrl alter monitor MariaDB-Monitor password=wrong
maxctrl list servers          # State turns Down (login fails)
sudo tail -n 10 /var/log/maxscale/maxscale.log      # read the error
maxctrl alter monitor MariaDB-Monitor password='MonPass#123'
```
**Lesson:** `maxctrl alter` changes MaxScale **live**, without a restart, and the logs explain failures.

### Experiment 2.3: See the config MaxScale is really using

```bash
maxctrl show monitor MariaDB-Monitor
maxctrl show service Read-Write-Service
maxctrl show listener Read-Write-Listener
maxctrl show maxscale
```
MaxScale shows every setting, including defaults you never wrote.

### Experiment 2.4: Where do live changes go?

```bash
ls /var/lib/maxscale/maxscale.cnf.d/
```
When you use `maxctrl alter`, MaxScale **saves** the change there. At startup it reads `/etc/maxscale.cnf` first, then these saved files, and the saved files **win**. This is like MariaDB/MySQL's `SET PERSIST`. If you edit the same setting in `/etc/maxscale.cnf` later and it "does nothing", this is why.

### Experiment 2.5: Change the listener port

```bash
sudo systemctl stop maxscale
sudo vim /etc/maxscale.cnf          # change port=4006 to port=4010
sudo systemctl start maxscale
ss -tlnp | grep maxscale            # now listening on 4010
```
Change it back to `4006` and restart. **Lesson:** the listener port is the **only** thing clients need to know.

---

## Stage 3: Add a Replica (db2)

```
 BEFORE                                  AFTER

   MaxScale --> db1                        MaxScale --+--> db1 (primary)
                                                      |
                                                      +--> db2 (replica)
                                                           copies db1
```

```
 3A. Build db2 (a MariaDB server)
 3B. Make db2 copy db1 (replication)
 3C. Tell MaxScale about db2          <-- the MaxScale part
 3D. Check the result
```

### 3A. Build db2

On the new machine (10.10.10.202):
```bash
sudo dnf install mariadb-server -y
sudo firewall-cmd --permanent --add-port=3306/tcp
sudo firewall-cmd --reload
sudo vim /etc/my.cnf.d/mariadb-server.cnf
```
```ini
[mysqld]
bind-address      = 0.0.0.0
server_id         = 2
log_bin           = mariadb-bin
binlog_format     = ROW
log_slave_updates = ON
read_only         = ON
report_host       = 10.10.10.202
```
Differences from db1:
- `server_id = 2` (**must differ**).
- `read_only = ON` blocks normal users from writing to the replica. (Accounts with `SUPER` or `READ_ONLY ADMIN`, including root, can still write, so never write directly on the replica.)

```bash
sudo systemctl enable --now mariadb
```

> **Do not** change anything locally on db2 (no passwords, no users). Replication will bring db1's accounts across. Local changes create extra transactions that confuse replication later.

### 3B. Make db2 copy db1

**On db1:** create the replication user.
```sql
CREATE USER 'repl'@'10.10.10.%' IDENTIFIED BY 'ReplPass#123';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'10.10.10.%';
```
(No restart of db1 was needed, because we prepared it in Stage 1.)

**On db2:**
```bash
sudo mysql
```
```sql
CHANGE MASTER TO
  MASTER_HOST='10.10.10.201',
  MASTER_USER='repl',
  MASTER_PASSWORD='ReplPass#123',
  MASTER_USE_GTID=slave_pos;

START SLAVE;
SHOW SLAVE STATUS\G
```
Both must say **Yes**:
```
Slave_IO_Running: Yes
Slave_SQL_Running: Yes
```
(`MASTER_USE_GTID=slave_pos` uses MariaDB's GTIDs, which failover needs. On MariaDB 10.5+ you can also write `START REPLICA` and `SHOW REPLICA STATUS`.)

Because db1 has recorded its changes since the binary log was switched on, db2 **replays everything**: the `labtest` database, the `notes` table, and the accounts `maxmon`, `maxsvc`, `appuser` and `repl`. Check on db2:
```sql
SHOW DATABASES;                                    -- labtest appears
SELECT * FROM labtest.notes;                       -- your row is there
SELECT user, host FROM mysql.user;                 -- the MaxScale accounts arrived too
```

> **Important:** `MASTER_HOST` must be exactly the address MaxScale uses for db1 (`10.10.10.201`). MaxScale matches the replica's source to its own server list. Do not use a hostname in one place and an IP in the other.
> **In real life** db1 would hold lots of data and its binary logs may have been purged. You would first copy the data (for example `mariadb-backup` or `mariadb-dump --single-transaction --master-data=2`) and then start replication. Replaying from the beginning only works on a young server.

### 3C. Tell MaxScale about db2 (the important part)

There are two ways. **Pick ONE**, do not do both.

#### Method A: live, with `maxctrl` (no restart)

```bash
maxctrl create server db2 10.10.10.202 3306
maxctrl link monitor MariaDB-Monitor db2
```

That is all. Two commands.
- `create server` makes the new server object.
- `link monitor` tells the monitor to watch it.
- You do **not** add it to the service: the service says `cluster=MariaDB-Monitor`, so it automatically uses every server that monitor looks after.

Nothing restarted. Clients stayed connected.

#### Method B: edit the file (needs a MaxScale restart)

```bash
sudo vim /etc/maxscale.cnf
```
Make **two** changes:
```diff
+[db2]
+type=server
+address=10.10.10.202
+port=3306
+
 [MariaDB-Monitor]
 type=monitor
 module=mariadbmon
-servers=db1
+servers=db1,db2
 user=maxmon
 password=MonPass#123
 monitor_interval=2s
```
(`+` = added, `-` = removed.)
```bash
sudo maxscale --config-check
sudo systemctl restart maxscale
```
This briefly disconnects clients, so on a live system prefer **Method A**.

> If you used Method A, do **not** also add `[db2]` to the file (the object would exist twice). Method A stores its change under `/var/lib/maxscale/maxscale.cnf.d/`. If you want your main file to document everything, use Method B from the start.

#### What changed? Compare

```
 STAGE 1                         STAGE 3
 [db1]   type=server             [db1]   type=server
                                 [db2]   type=server         <-- NEW server section
 servers=db1                     servers=db1,db2             <-- monitor now watches both
 cluster=MariaDB-Monitor         cluster=MariaDB-Monitor     <-- service unchanged
 port=4006                       port=4006                   <-- listener unchanged
```
**The service and listener did not change.** Applications keep using the same address.

### 3D. Check the result

```bash
maxctrl list servers
```
```
┌────────┬───────────────┬──────┬─────────────┬─────────────────┐
│ Server │ Address       │ Port │ Connections │ State           │
├────────┼───────────────┼──────┼─────────────┼─────────────────┤
│ db1    │ 10.10.10.201  │ 3306 │ 0           │ Master, Running │
├────────┼───────────────┼──────┼─────────────┼─────────────────┤
│ db2    │ 10.10.10.202  │ 3306 │ 0           │ Slave, Running  │
└────────┴───────────────┴──────┴─────────────┴─────────────────┘
```
db2 is `Slave, Running`: MaxScale saw that db2 replicates from db1 and gave each server its role by itself.

**Checkpoint 3:** two servers, two roles, same client address.

#### Checklist (if db2 has no `Slave` role)

- [ ] `SHOW SLAVE STATUS\G` on db2 shows both threads `Yes`.
- [ ] `MASTER_HOST` is exactly `10.10.10.201`, the same string as MaxScale's `address` for db1.
- [ ] `maxmon` can log in from MaxScale to db2 (`mysql -h 10.10.10.202 -u maxmon -p`).
- [ ] `maxmon` has the monitor privileges for your version (Section 1.2 table).
- [ ] Firewall allows port 3306 between all three machines.
- [ ] `sudo tail -n 30 /var/log/maxscale/maxscale.log` for the real reason.

---

## Stage 4: Watch Read/Write Splitting

```
  INSERT / UPDATE / DELETE / transactions  --->  db1 (primary)
  plain SELECT                             --->  db2 (replica)
```

### 4.1 See the split

Reconnect (new sessions learn about the new server):
```bash
mysql -h 10.10.10.203 -P 4006 -u appuser -p
```
```sql
USE labtest;
SELECT @@hostname;                 -- db2: a plain read goes to the replica

START TRANSACTION;
SELECT @@hostname;                 -- db1: anything inside a transaction goes to the primary
COMMIT;

INSERT INTO notes (txt) VALUES ('written through maxscale');   -- goes to db1
SELECT * FROM notes;                                           -- read from db2
```
(Give db1 and db2 different hostnames to make this obvious.) Sessions opened before you added db2 keep using db1 only. **New** connections get the split.

### 4.2 Prove the replica is protected

```bash
mysql -h 10.10.10.202 -u appuser -p -e "INSERT INTO labtest.notes (txt) VALUES ('direct to replica')"
```
Refused (read-only server). Through MaxScale the same insert works, because MaxScale sent it to db1.

### 4.3 Watch the distribution

```bash
watch -n 1 maxctrl list servers
```
Open several sessions and run queries. The **Connections** column shows how sessions spread.

### 4.4 What readwritesplit does

```
 Statement from app                       Where it goes
 ---------------------------------------  ------------------------------
 INSERT, UPDATE, DELETE, DDL              primary
 START TRANSACTION ... COMMIT             primary (whole transaction)
 SELECT outside a transaction             a replica
 SET variable / USE database              ALL servers (so every connection stays consistent)
```

### 4.5 The read-after-write trap

```
 INSERT order #55  --> db1
 SELECT order #55  --> db2   (replica may not have it yet!)
```
The replica is normally milliseconds behind, but not zero. Stage 5 fixes this properly with MariaDB's `causal_reads`.

---

## Stage 5: Replication Lag and "Read Your Own Writes"

### 5.1 Protect against a lagging replica

```bash
maxctrl alter service Read-Write-Service max_slave_replication_lag=10s
maxctrl show service Read-Write-Service
```
Meaning: do not send reads to a replica more than 10 seconds behind.

### 5.2 Create artificial lag and watch

On **db2**:
```sql
STOP SLAVE SQL_THREAD;          -- db2 keeps receiving but stops applying changes
```
Through MaxScale, write a row:
```sql
INSERT INTO labtest.notes (txt) VALUES ('while db2 is stuck');
```
Wait about 15 seconds, then:
```bash
maxctrl show server db2           # look for the replication lag value
```
Run `SELECT @@hostname;` a few times through MaxScale. Once lag exceeds 10 seconds, reads go to **db1**.

Fix it on db2:
```sql
START SLAVE SQL_THREAD;
```
Once db2 catches up, reads return to it. **Lesson:** MaxScale reacts to lag automatically.

### 5.3 `causal_reads`: always see your own writes (MariaDB feature)

With causal reads, MaxScale remembers the GTID of your last write and only reads from a replica that has reached it (or waits briefly, or falls back to the primary).

```bash
maxctrl alter service Read-Write-Service causal_reads=local
maxctrl show service Read-Write-Service      # confirm
```
(Accepted values vary by MaxScale version. Newer versions accept `none`, `local`, `global`, `fast` and others; older versions use `true`/`false`. Check the `readwritesplit` documentation for yours.)

Test, with db2 deliberately slow:
```sql
-- on db2
STOP SLAVE SQL_THREAD;
```
```sql
-- through MaxScale, in ONE session
INSERT INTO labtest.notes (txt) VALUES ('causal test');
SELECT * FROM labtest.notes WHERE txt = 'causal test';
```
- **Without** `causal_reads`: the read goes to db2, which has not applied the row, so you get **no rows**.
- **With** `causal_reads=local`: MaxScale waits for db2 (up to `causal_reads_timeout`, default 10s) and, if it cannot catch up, **reads from db1**, so you get the row.

Restore db2:
```sql
START SLAVE SQL_THREAD;
```
Trade-off: causal reads can add a little latency, so enable them only if your application needs them.

---

## Stage 6: Maintenance Mode and a Read-Only Port

### 6.1 Take a server out of rotation

Imagine you must patch db2:
```bash
maxctrl set server db2 maintenance
maxctrl list servers              # db2 shows Maintenance
```
New reads go to db1. Existing sessions finish normally. Patch or restart db2, then:
```bash
maxctrl clear server db2 maintenance
```

### 6.2 A port for reads only

Sometimes reporting tools should only hit the replicas. Add a second **service** and **listener**, live:
```bash
maxctrl create service Read-Only-Service readconnroute user=maxsvc password='SvcPass#123' \
        cluster=MariaDB-Monitor router_options=slave
maxctrl create listener Read-Only-Service Read-Only-Listener 4007 protocol=MariaDBClient
sudo firewall-cmd --permanent --add-port=4007/tcp && sudo firewall-cmd --reload
```
(If `maxctrl` rejects the argument format on your version, use the file method.)

**Or in the file** (add at the end of `/etc/maxscale.cnf`, then `sudo systemctl restart maxscale`):
```ini
[Read-Only-Service]
type=service
router=readconnroute
router_options=slave
cluster=MariaDB-Monitor
user=maxsvc
password=SvcPass#123

[Read-Only-Listener]
type=listener
service=Read-Only-Service
protocol=MariaDBClient
port=4007
```
```
 port 4006  ->  Read-Write-Service  (readwritesplit)  -> db1 + db2, split by statement
 port 4007  ->  Read-Only-Service   (readconnroute)   -> replicas only
```
Test:
```bash
mysql -h 10.10.10.203 -P 4007 -u appuser -p -e "SELECT @@hostname"     # always db2
```
`readconnroute` picks a server once per connection and keeps it: simple and predictable.

---

## Stage 7: Planned Switchover

A **switchover** is a planned role swap: db2 becomes the primary and db1 becomes a replica, with **no data loss**. MaxScale's monitor does it for you with one command, but it needs more privileges and the replication credentials.

### 7.1 Give the monitor failover privileges

On the primary (db1):
```sql
-- MariaDB 10.5.2 and newer:
GRANT RELOAD, PROCESS, SHOW DATABASES, EVENT, SET USER, READ_ONLY ADMIN
  ON *.* TO 'maxmon'@'10.10.10.%';
GRANT REPLICATION SLAVE ADMIN, BINLOG ADMIN, CONNECTION ADMIN
  ON *.* TO 'maxmon'@'10.10.10.%';
```
(MariaDB 10.4 and older use `GRANT SUPER, RELOAD, PROCESS, SHOW DATABASES, EVENT ON *.* ...` instead.)

Why: to switch roles, the monitor must be able to change `read_only`, stop and start replication, repoint replicas and end connections.

### 7.2 Tell the monitor which replication account to use

```bash
maxctrl alter monitor MariaDB-Monitor replication_user=repl replication_password='ReplPass#123'
```

### 7.3 Switch over

```bash
maxctrl call command mariadbmon switchover MariaDB-Monitor db2 db1
```
Format: `switchover <monitor> <new-primary> <current-primary>`.

Watch it:
```bash
maxctrl list servers
```
After a moment: **db2 is `Master`, db1 is `Slave`**. Applications kept using `10.10.10.203:4006` and noticed at most a brief pause.

Swap back the same way:
```bash
maxctrl call command mariadbmon switchover MariaDB-Monitor db1 db2
```
**Lesson:** the monitor changed `read_only` on both servers and repointed replication by itself. You did not touch MariaDB.

---

## Stage 8: Automatic Failover

A **failover** happens when the primary has **crashed**. MaxScale promotes the replica automatically.

```
 BEFORE                     DB1 dies                    AFTER
 db1 = Master               db1 = Down                  db1 = Down (later rejoins as Slave)
 db2 = Slave       ---->    db2 = Slave      ---->      db2 = Master
```

### 8.1 Turn it on (live)

```bash
maxctrl alter monitor MariaDB-Monitor \
    auto_failover=true auto_rejoin=true failcount=5 \
    replication_user=repl replication_password='ReplPass#123'
```

| Setting | Meaning |
|---|---|
| `auto_failover=true` | Promote a replica automatically when the primary stays down |
| `auto_rejoin=true` | When the old primary comes back, make it a replica of the new primary |
| `failcount=5` | Failed checks before the primary is declared dead (5 checks x 2 seconds = about 10 seconds) |
| `replication_user/password` | Used to set up replication during failover and rejoin |

In the file (inside `[MariaDB-Monitor]`):
```ini
auto_failover=true
auto_rejoin=true
failcount=5
replication_user=repl
replication_password=ReplPass#123
```

### 8.2 Test it (test environment only!)

Terminal 1 (MaxScale):
```bash
watch -n 1 maxctrl list servers
```
Terminal 2 (db1):
```bash
sudo systemctl stop mariadb
```
After about 10 seconds: **db1 `Down`, db2 `Master, Running`**. Connect through port 4006 and write data; it works and lands on db2.

Bring db1 back:
```bash
sudo systemctl start mariadb
```
With `auto_rejoin=true`, db1 returns as **`Slave, Running`** of db2 and catches up.

### 8.3 Make applications survive a failover

On the service:
```bash
maxctrl alter service Read-Write-Service \
    transaction_replay=true master_reconnection=true delayed_retry=true
```
Meaning:
- `master_reconnection`: reconnect sessions to the new primary after a switch.
- `transaction_replay`: replay an interrupted transaction on the new primary where possible, so the application sees no error.
- `delayed_retry`: hold a query briefly instead of failing while the primary is unavailable.

Test with your own application before relying on it.

### 8.4 Safety notes

- With only **two** servers, after a failover you have **one** server until the old one returns. Plan for a third.
- All clients must go **through MaxScale**. If some applications connect directly to a database, a failover could leave two writers (a "split brain").
- A failover decision is only as good as the monitor's view of the network. Use a sensible `failcount`, and do not set it to 1.
- Run **two** MaxScale nodes with a virtual IP, because MaxScale is now a single point of failure. If both run a monitor, make sure only one makes failover decisions (cooperative monitoring; see the MaxScale documentation).
- Test failover regularly in staging and write a runbook.
- If you add more servers, remember the license limit.

---

## What You Learned: The Config Journey

```
 STAGE 1               STAGE 3                      STAGE 5-6                  STAGE 8
 -------------------   --------------------------   ------------------------   ----------------------
 [db1]                 [db1]                        [db1]                      [db1]
                       [db2]            <-- new     [db2]                      [db2]
 [MariaDB-Monitor]     [MariaDB-Monitor]            [MariaDB-Monitor]          [MariaDB-Monitor]
  servers=db1           servers=db1,db2 <-- changed  (grants + repl user)       + auto_failover=true
                                                                                + auto_rejoin=true
 [Read-Write-Service]  [Read-Write-Service]         [Read-Write-Service]       [Read-Write-Service]
                                                     + max_slave_replication_lag + transaction_replay
                                                     + causal_reads
                                                    [Read-Only-Service] <-new
 [Read-Write-Listener] [Read-Write-Listener]        [Read-Write-Listener]      [Read-Write-Listener]
                                                    [Read-Only-Listener] <-new
```

| Goal | What you changed |
|---|---|
| Add a database | New `server` object + add it to the monitor's `servers` |
| Protect against lag | One service setting |
| See your own writes | `causal_reads` on the service |
| Add a read-only door | New service + new listener |
| Maintenance | `maxctrl set server ... maintenance` (no config edit) |
| Planned role swap | `maxctrl call command mariadbmon switchover ...` |
| Survive a crash | Monitor `auto_failover`, `auto_rejoin`, `failcount` |

---

## Troubleshooting

| Problem | Likely cause and fix |
|---|---|
| `Down` for a server | MariaDB stopped, firewall (3306), wrong IP, or `maxmon` cannot log in from MaxScale's IP |
| Login through 4006 fails | Wrong password, account missing, or host pattern does not match MaxScale's IP |
| `GRANT REPLICA MONITOR` fails | Older MariaDB. Use the table in Section 1.2 for your version |
| `GRANT ... mysql.global_priv` fails | MariaDB older than 10.4. Skip that grant |
| MaxScale will not start | `sudo maxscale --config-check` and `sudo journalctl -u maxscale -n 50` |
| New server not showing a role | Replication not running, or `MASTER_HOST` differs from the address in MaxScale |
| New user cannot connect | MaxScale caches users: `maxctrl reload service Read-Write-Service` |
| Edited `/etc/maxscale.cnf` but nothing changed | Restart needed, or a saved live change in `/var/lib/maxscale/maxscale.cnf.d/` overrides it |
| `maxctrl` unauthorised or cannot connect | MaxScale not running, or wrong admin credentials |
| Reads never reach the replica | Lag above the limit, no `Slave` state, `causal_reads` falling back, or queries inside transactions |
| Switchover fails | Missing monitor privileges (Stage 7.1), no `replication_user`, replica not caught up, or no binary log on the replica |
| Failover does not happen | `auto_failover` off, `failcount` not reached, replica not replicating over GTID, or missing privileges |
| Old primary does not rejoin | `auto_rejoin` off, `read_only`/binlog issues, or it has transactions the new primary lacks (rebuild it) |
| Replication breaks with GTID errors | Local changes were made on a replica. Do all changes on the primary |

First checks, always:
```bash
maxctrl list servers
maxctrl list monitors
sudo tail -n 50 /var/log/maxscale/maxscale.log
```
On a database:
```sql
SHOW SLAVE STATUS\G
SHOW GRANTS FOR 'maxmon'@'10.10.10.%';
```

---

## Cheat Sheet

```
 FILES
   /etc/maxscale.cnf                   main config
   /var/lib/maxscale/maxscale.cnf.d/   live changes made with maxctrl (they override the main file)
   /var/log/maxscale/maxscale.log      log

 INSPECT
   maxctrl list servers | services | listeners | monitors | sessions
   maxctrl show server db1 | service NAME | monitor NAME | maxscale

 CHANGE LIVE (no restart)
   maxctrl alter service Read-Write-Service max_slave_replication_lag=10s
   maxctrl alter service Read-Write-Service causal_reads=local
   maxctrl alter monitor MariaDB-Monitor monitor_interval=1s
   maxctrl create server db2 10.10.10.202 3306
   maxctrl link monitor MariaDB-Monitor db2
   maxctrl unlink monitor MariaDB-Monitor db2
   maxctrl destroy server db2
   maxctrl set server db2 maintenance
   maxctrl clear server db2 maintenance
   maxctrl reload service Read-Write-Service      (refresh the user list)

 ROLE CHANGES
   maxctrl call command mariadbmon switchover MariaDB-Monitor NEWPRIMARY OLDPRIMARY
   maxctrl alter monitor MariaDB-Monitor auto_failover=true auto_rejoin=true failcount=5

 CHECK THE FILE
   sudo maxscale --config-check
```

### Remember

```
 1. Objects refer to each other by NAME:  listener -> service -> monitor -> servers
 2. cluster=<monitor> means the service uses whatever the monitor watches
 3. Add a database = create server + link it to the monitor
 4. maxctrl changes are live and saved; they override the main file
 5. MaxScale connects to the database from ITS OWN IP: set account host patterns accordingly
 6. The application never changes: it always uses MaxScale-IP:4006
 7. Switchover = planned (no data loss); failover = crash (small risk)
```

### Final configuration you built (file form)

```ini
[maxscale]
threads=auto

[db1]
type=server
address=10.10.10.201
port=3306

[db2]
type=server
address=10.10.10.202
port=3306

[MariaDB-Monitor]
type=monitor
module=mariadbmon
servers=db1,db2
user=maxmon
password=MonPass#123
monitor_interval=2s
replication_user=repl
replication_password=ReplPass#123
auto_failover=true
auto_rejoin=true
failcount=5

[Read-Write-Service]
type=service
router=readwritesplit
cluster=MariaDB-Monitor
user=maxsvc
password=SvcPass#123
max_slave_replication_lag=10s
causal_reads=local
transaction_replay=true
master_reconnection=true
delayed_retry=true

[Read-Write-Listener]
type=listener
service=Read-Write-Service
protocol=MariaDBClient
port=4006

[Read-Only-Service]
type=service
router=readconnroute
router_options=slave
cluster=MariaDB-Monitor
user=maxsvc
password=SvcPass#123

[Read-Only-Listener]
type=listener
service=Read-Only-Service
protocol=MariaDBClient
port=4007
```
(Encrypt the passwords with `maxkeys` and `maxpasswd`, and `chmod 600 /etc/maxscale.cnf`, before using this anywhere real.)

### Production notes

- Do not expose the REST API (port 8989). Change the default admin user:
```bash
maxctrl create user admin2 'StrongPassword!' --type=admin
maxctrl destroy user admin
```
- Encrypt passwords in `maxscale.cnf` (`maxkeys`, `maxpasswd`).
- Run two MaxScale nodes with a virtual IP (for example Keepalived).
- Applications should use a connection pool and handle reconnects.
- Monitor replication lag and disk space (binary logs).
- Keep backups. MaxScale does not back up your data.
- Check the license before production use, especially if you add more backend servers.

### Practice ideas
1. Rebuild the whole lab from scratch without looking, using only the checkpoints.
2. Add a third database (db3) and see how little changes (mind the license limit).
3. Change `monitor_interval` to `500ms` and time how fast a stopped database shows as `Down`.
4. Switch over back and forth five times while a script keeps inserting rows through port 4006. Did you lose any?
5. Kill the primary with `kill -9` on the MariaDB process and time the automatic failover.
6. Write the final `/etc/maxscale.cnf` as a documented file for your team.
