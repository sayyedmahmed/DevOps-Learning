# MariaDB MaxScale Beginner Guide

What MaxScale is, why you would use it, and how to set it up step by step, even if you have never used or configured it.

**Contents**

1. What is MaxScale?
2. The problem it solves
3. Core concepts (the five building blocks)
4. Read/write splitting and failover explained
5. Licensing and compatibility (read this first)
6. Lab plan
7. Step 1: Build two database servers with replication
8. Step 2: Create the users MaxScale needs
9. Step 3: Install MaxScale
10. Step 4: Configure MaxScale
11. Step 5: Start it and check it
12. Step 6: Test read/write splitting
13. Managing MaxScale with `maxctrl` (no restart needed)
14. Failover and switchover
15. Useful router settings
16. Filters (optional)
17. If your database is MySQL
18. Production notes
19. Troubleshooting
20. Cheat sheet

> Commands are for **AlmaLinux / RHEL / Rocky**. Edit files with `sudo vim FILE` (press `i` to type, `Esc` then `:wq` to save and quit).
> Software changes over time. Where a version detail matters, the guide says so. Always compare with the official MaxScale documentation for **your** version.

---

# 1. What is MaxScale?

**MariaDB MaxScale** is a **database proxy**: a program that sits **between your applications and your database servers**. Applications talk to MaxScale as if it were a normal database. MaxScale then decides which real database server should handle each request.

```
  WITHOUT MaxScale                       WITH MaxScale

  App ------------> DB server            App ---> MaxScale ---+--> DB 1 (primary)
                                                              |
                                                              +--> DB 2 (replica)
```

It speaks the normal MySQL/MariaDB protocol, so most applications need **only one change**: connect to MaxScale's IP and port instead of the database's.

Think of MaxScale as a **receptionist** for your databases. Visitors (queries) arrive at one desk. The receptionist knows which doctor (server) is available, who handles what, and quietly redirects people if a doctor is off sick.

## What it can do

| Feature | Meaning |
|---|---|
| **Read/write splitting** | Sends writes to the primary, reads to replicas (spreads load) |
| **Load balancing** | Shares reads across several replicas |
| **Failover / switchover** | Promotes a replica if the primary dies (automatic), or on request (planned) |
| **Connection routing** | Hides database IPs from applications |
| **Monitoring** | Constantly checks which servers are up and what role they have |
| **Filters** | Logs, rewrites, blocks or caches queries |
| **Security** | One place to control who connects and from where |

---

# 2. The Problem It Solves

## Problem 1: One database cannot handle all the reads

Most applications **read** far more than they **write**. You can add replicas (copies) to share the reads, but the application then has to know which server to use for what.

```
 Without a proxy, the APPLICATION must decide:
   "writes -> 10.10.10.201,  reads -> 10.10.10.202"    <- messy code
```
With MaxScale the application connects to **one address**, and MaxScale does the choosing.

## Problem 2: The primary dies

If the primary fails and applications use its IP directly, they all break until someone changes every config.

```
 Before:  App --> DB1 (dies)   --> everything fails

 With MaxScale:
          App --> MaxScale --> DB1 (dies)
                       |
                       +--> promotes DB2 to primary, app keeps the same address
```

## Problem 3: Changing servers means changing applications

With MaxScale you can add, remove, or maintain servers **without touching the applications**.

---

# 3. Core Concepts (The Five Building Blocks)

MaxScale is built from five kinds of objects. Learn these and the config file becomes easy to read.

```
  CLIENT
    |
    v
 +----------+      +-----------+      +----------+      +----------+
 | LISTENER | ---> |  SERVICE  | ---> |  SERVER  |      | MONITOR  |
 | (the     |      | (router + |      | (a real  | <--- | (watches |
 |  door)   |      |  rules)   |      |  DB)     |      |  servers)|
 +----------+      +-----------+      +----------+      +----------+
   port 4006        readwritesplit      db1, db2          mariadbmon
```

| Object | What it is | Analogy |
|---|---|---|
| **Server** | One real database server (address and port) | A doctor |
| **Monitor** | Checks every server's health and role (primary or replica) | A nurse taking vitals |
| **Router (inside a Service)** | The logic that decides where each query goes | The receptionist's rules |
| **Service** | A named router plus the servers/cluster it uses | A department |
| **Listener** | A port that clients connect to, attached to a service | The front door |
| **Filter** (optional) | Inspects or changes queries on the way through | A security scanner |

## Common routers

| Router | What it does | Use when |
|---|---|---|
| `readwritesplit` | Looks at **each statement**: writes to primary, reads to replicas | Normal applications with replication |
| `readconnroute` | Picks a server **once when the client connects**, whole session stays there | Simple load balancing, read-only pools |

## Config file layout

Everything lives in **`/etc/maxscale.cnf`**, in sections:
```ini
[section-name]
type=server        # tells MaxScale what kind of object this is
setting=value
```

---

# 4. Read/Write Splitting and Failover Explained

## 4.1 How readwritesplit decides

```
 App sends:                        MaxScale routes to:
 --------------------------        ---------------------------
 INSERT / UPDATE / DELETE    --->   PRIMARY  (db1)
 SELECT (outside a transaction) -->  a REPLICA (db2)  (load balanced)
 START TRANSACTION ... COMMIT -->   PRIMARY  (whole transaction stays together)
 SET / session commands      --->   ALL servers (so every session stays consistent)
```

```
              +--> db1 (primary)   <-- writes + transactions
  App --> MaxScale
              +--> db2 (replica)   <-- plain reads
```

## 4.2 Replication lag warning

A replica can be a moment behind the primary. If the application **writes then immediately reads**, the read may hit a replica that has not caught up yet.

```
 INSERT order #55  --> primary
 SELECT order #55  --> replica  (not there yet!)  <-- "read-after-write" problem
```
Solutions: put the read inside the same transaction, use `causal_reads` (MariaDB feature, see Section 15), or set a maximum allowed lag.

## 4.3 Failover vs switchover

| | Failover | Switchover |
|---|---|---|
| When | Primary has **crashed** | **Planned** (maintenance, upgrades) |
| Triggered | Automatically (if enabled) or manually | Manually by you |
| Data risk | Possible small loss if the replica was behind | None, it waits for replicas to catch up |

---

# 5. Licensing and Compatibility (Read This First)

## 5.1 License

MaxScale 2.x and later use the **Business Source License (BSL)**, not a normal open-source license. As described in MariaDB's own material:

- **Testing is free.**
- **Production use is free for a limited number of database servers** (the published limit has been *no more than two* backend servers). More than that requires a commercial MariaDB subscription.
- Each version **converts to GPL** after its "Change Date" (several years after release).
- The exact terms are in the `LICENSE.TXT` shipped with each version. **Read the license of the version you install** before using it in production.

This lab uses **two database servers**, which stays within that limit.

## 5.2 Which databases does it work with?

| Backend | Status |
|---|---|
| **MariaDB** | The intended and fully supported backend. Use this for the main lab |
| **MySQL** | Basic routing is reported to work, but features that need MariaDB extensions do not (for example `causal_reads` and MariaDB-style GTID failover). MySQL 8's default `caching_sha2_password` login was reported as **not directly supported**, so users must use `mysql_native_password` |

> The MySQL notes come from MaxScale developers' statements in a mailing list in 2023. Newer versions may have changed this, so verify against the current MaxScale documentation before relying on it. See Section 17.

## 5.3 Alternatives

| Tool | Notes |
|---|---|
| **ProxySQL** | Popular open-source proxy for both MySQL and MariaDB (GPL) |
| **MySQL Router** | Oracle's router, designed for MySQL InnoDB Cluster |
| **HAProxy** | Simple TCP load balancer (does not understand SQL, so no read/write split) |

---

# 6. Lab Plan

Three machines (virtual machines are fine). Example IPs, **replace with yours**:

```
                         +----------------------+
   Applications  ------> |  MAXSCALE            |
   (mysql client)        |  10.10.10.203        |
                         |  port 4006           |
                         +----------+-----------+
                                    |
                      +-------------+--------------+
                      |                            |
             +--------v-------+          +---------v------+
             |  DB1 (PRIMARY) |--------->|  DB2 (REPLICA) |
             |  10.10.10.201  |replicate |  10.10.10.202  |
             |  server_id=1   |          |  server_id=2   |
             +----------------+          +----------------+
```

| Machine | IP | Role |
|---|---|---|
| db1 | 10.10.10.201 | Primary (accepts writes) |
| db2 | 10.10.10.202 | Replica (copy, reads) |
| maxscale | 10.10.10.203 | The proxy |

---

# 7. Step 1: Build Two Database Servers with Replication

MaxScale needs a working **replication** setup first. If you already have one, skip to Section 8.

This lab uses **MariaDB** with brand-new empty servers.

## 7.1 Install MariaDB (on db1 and db2)
```bash
sudo dnf install mariadb-server -y
sudo systemctl enable --now mariadb
sudo mysql_secure_installation
```

## 7.2 Configure db1 (primary)
```bash
sudo vim /etc/my.cnf.d/mariadb-server.cnf
```
Under `[mysqld]` add:
```ini
[mysqld]
bind-address      = 0.0.0.0
server_id         = 1
log_bin           = mariadb-bin
log_slave_updates = ON
binlog_format     = ROW
```

## 7.3 Configure db2 (replica)
```ini
[mysqld]
bind-address      = 0.0.0.0
server_id         = 2
log_bin           = mariadb-bin
log_slave_updates = ON
binlog_format     = ROW
read_only         = ON
```

| Setting | Why |
|---|---|
| `server_id` | Must be **different** on every server |
| `log_bin` | Records changes. The replica also needs it so it can become primary later |
| `log_slave_updates` | Lets a replica pass on changes (needed for failover) |
| `read_only = ON` | Stops accidental writes on the replica. MaxScale also manages this during failover |

Restart both (a fresh lab, so downtime does not matter):
```bash
sudo systemctl restart mariadb
```

## 7.4 Open the firewall (db1 and db2)
```bash
sudo firewall-cmd --permanent --add-port=3306/tcp
sudo firewall-cmd --reload
```

## 7.5 Create the replication user (on db1 only)
```bash
sudo mysql -u root -p
```
```sql
CREATE USER 'repl'@'10.10.10.%' IDENTIFIED BY 'ReplPass#123';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'10.10.10.%';
```

## 7.6 Point db2 at db1 (on db2 only)
```sql
CHANGE MASTER TO
  MASTER_HOST='10.10.10.201',
  MASTER_USER='repl',
  MASTER_PASSWORD='ReplPass#123',
  MASTER_USE_GTID=slave_pos;

START SLAVE;
SHOW SLAVE STATUS\G
```
In the output both of these must say **Yes**:
```
Slave_IO_Running: Yes
Slave_SQL_Running: Yes
```
(`MASTER_USE_GTID=slave_pos` uses MariaDB's GTIDs, which MaxScale needs for failover.)

## 7.7 Quick replication test
On db1:
```sql
CREATE DATABASE labtest;
```
On db2:
```sql
SHOW DATABASES;      -- labtest should appear
```

> If you already have data on db1, you must copy it to db2 first (for example with `mariadb-dump --single-transaction --gtid`) before `START SLAVE`. Replication of empty servers is the easiest way to learn.

---

# 8. Step 2: Create the Users MaxScale Needs

MaxScale logs in to your database servers as **its own users**. Create them on **db1**; replication copies them to db2.

```sql
-- 1. Monitor user: lets MaxScale check health and manage replication
CREATE USER 'maxmon'@'10.10.10.%' IDENTIFIED BY 'MonPass#123';
GRANT SUPER, RELOAD, PROCESS, SHOW DATABASES, EVENT ON *.* TO 'maxmon'@'10.10.10.%';
GRANT REPLICATION CLIENT, REPLICATION SLAVE ON *.* TO 'maxmon'@'10.10.10.%';
GRANT REPLICATION SLAVE ADMIN, READ_ONLY ADMIN ON *.* TO 'maxmon'@'10.10.10.%';

-- 2. Service user: lets MaxScale read the list of users so it can authenticate clients
CREATE USER 'maxsvc'@'10.10.10.%' IDENTIFIED BY 'SvcPass#123';
GRANT SELECT ON mysql.user         TO 'maxsvc'@'10.10.10.%';
GRANT SELECT ON mysql.db           TO 'maxsvc'@'10.10.10.%';
GRANT SELECT ON mysql.tables_priv  TO 'maxsvc'@'10.10.10.%';
GRANT SELECT ON mysql.columns_priv TO 'maxsvc'@'10.10.10.%';
GRANT SELECT ON mysql.proxies_priv TO 'maxsvc'@'10.10.10.%';
GRANT SELECT ON mysql.roles_mapping TO 'maxsvc'@'10.10.10.%';
GRANT SHOW DATABASES ON *.* TO 'maxsvc'@'10.10.10.%';

-- 3. An application user to test with
CREATE USER 'appuser'@'%' IDENTIFIED BY 'AppPass#123';
GRANT ALL PRIVILEGES ON labtest.* TO 'appuser'@'%';
```

> Privilege names differ between MariaDB versions (for example, `REPLICATION SLAVE ADMIN` and `READ_ONLY ADMIN` exist from MariaDB 10.5). If a `GRANT` fails, check the MaxScale documentation page for the **MariaDB Monitor** for the list that matches your version.

Why each user:

| User | Used by | Why |
|---|---|---|
| `maxmon` | The **monitor** | Checks server state, reads replication status, performs failover |
| `maxsvc` | The **service** (router) | Reads the user list so MaxScale can check client logins itself |
| `appuser` | Your application | A normal user, to prove everything works |

**Important detail:** MaxScale connects to the backend **on behalf of the client**, so the backend sees the connection coming from **MaxScale's IP (10.10.10.203)**, not the client's. An account such as `'appuser'@'10.10.10.50'` would be rejected by the backend. Use `'%'` or include MaxScale's address in the host pattern.

Check they exist on db2 too:
```sql
SELECT user, host FROM mysql.user WHERE user LIKE 'max%' OR user='appuser';
```

---

# 9. Step 3: Install MaxScale

On the **maxscale** machine (needs internet access):

```bash
curl -LsS https://r.mariadb.com/downloads/mariadb_repo_setup | sudo bash
sudo dnf install -y maxscale
```
The first command adds the official MariaDB repository. If it fails, the machine may be blocked from reaching the internet.

Check:
```bash
maxscale --version
```
Open the client port in the firewall:
```bash
sudo firewall-cmd --permanent --add-port=4006/tcp
sudo firewall-cmd --reload
```
(Do **not** open port 8989 to the network. That is the admin REST API, and it should stay local.)

---

# 10. Step 4: Configure MaxScale

## 10.1 Back up the default file
```bash
sudo cp /etc/maxscale.cnf /etc/maxscale.cnf.original
```

## 10.2 Write the configuration
```bash
sudo vim /etc/maxscale.cnf
```
Delete everything (in vim: `Esc`, then `:%d`, `Enter`, then `i`) and paste:

```ini
# ---------- Global ----------
[maxscale]
threads=auto

# ---------- The two database servers ----------
[db1]
type=server
address=10.10.10.201
port=3306

[db2]
type=server
address=10.10.10.202
port=3306

# ---------- The monitor (watches the servers) ----------
[MariaDB-Monitor]
type=monitor
module=mariadbmon
servers=db1,db2
user=maxmon
password=MonPass#123
monitor_interval=2s

# ---------- The service (read/write splitting) ----------
[Read-Write-Service]
type=service
router=readwritesplit
cluster=MariaDB-Monitor
user=maxsvc
password=SvcPass#123

# ---------- The listener (the front door) ----------
[Read-Write-Listener]
type=listener
service=Read-Write-Service
protocol=MariaDBClient
port=4006
```

## 10.3 Read it line by line

```
 [db1] / [db2]                 one section per real server
   type=server                 "this is a server object"
   address / port              where the database lives

 [MariaDB-Monitor]
   type=monitor                "this is a monitor"
   module=mariadbmon           the monitor type for MariaDB replication
   servers=db1,db2             which servers to watch
   user / password             the maxmon account from Section 8
   monitor_interval=2s         check every 2 seconds

 [Read-Write-Service]
   type=service                "this is a service"
   router=readwritesplit       split reads and writes
   cluster=MariaDB-Monitor     use the servers that this monitor watches
   user / password             the maxsvc account from Section 8

 [Read-Write-Listener]
   type=listener               "this is a listener"
   service=Read-Write-Service  attach to the service above
   port=4006                   clients connect here
```

> **Version note:** `cluster=` in a service works in MaxScale 22.08 and later. On older versions use `servers=db1,db2` in the service instead. In older releases the listener protocol was named `MySQLClient`.

## 10.4 Check the file before starting
```bash
sudo maxscale --config-check
```
No errors means the syntax is fine.

## 10.5 Protect the passwords (recommended)

The config currently holds plain-text passwords. Encrypt them:
```bash
sudo maxkeys                      # creates the key file /var/lib/maxscale/.secrets
sudo maxpasswd 'MonPass#123'      # prints an encrypted string
sudo maxpasswd 'SvcPass#123'
```
Put the **encrypted strings** into `password=` in `/etc/maxscale.cnf`, then lock the file:
```bash
sudo chmod 600 /etc/maxscale.cnf
sudo chown maxscale:maxscale /etc/maxscale.cnf
```
(If you encrypt, run `maxkeys` once **before** `maxpasswd`, and keep the same `.secrets` file afterwards.)

---

# 11. Step 5: Start It and Check It

```bash
sudo systemctl enable --now maxscale
sudo systemctl status maxscale
```

Look at the servers:
```bash
maxctrl list servers
```
Expected (the exact look varies by version):
```
┌────────┬───────────────┬──────┬─────────────┬─────────────────┬───────┐
│ Server │ Address       │ Port │ Connections │ State           │ GTID  │
├────────┼───────────────┼──────┼─────────────┼─────────────────┼───────┤
│ db1    │ 10.10.10.201  │ 3306 │ 0           │ Master, Running │ ...   │
├────────┼───────────────┼──────┼─────────────┼─────────────────┼───────┤
│ db2    │ 10.10.10.202  │ 3306 │ 0           │ Slave, Running  │ ...   │
└────────┴───────────────┴──────┴─────────────┴─────────────────┴───────┘
```

What the states mean:

| State | Meaning |
|---|---|
| `Master, Running` | Primary, healthy |
| `Slave, Running` | Replica, healthy |
| `Down` | Cannot connect. Check firewall, user, and that MariaDB is running |
| `Maintenance` | You took it out of rotation on purpose |
| `Running` only (no role) | Up, but MaxScale cannot tell its role. Check replication and the monitor user's grants |

Other checks:
```bash
maxctrl list services
maxctrl list listeners
maxctrl list monitors
sudo tail -n 30 /var/log/maxscale/maxscale.log
```

> `maxctrl` uses the built-in admin account (a default exists on a new install). **Change it** in production (Section 18).

---

# 12. Step 6: Test Read/Write Splitting

From **any machine** (here a client machine, with the `mysql`/`mariadb` client installed). Connect to **MaxScale**, not to the databases:

```bash
mysql -h 10.10.10.203 -P 4006 -u appuser -p
```

## 12.1 Make a table and write data
```sql
USE labtest;
CREATE TABLE notes (id INT AUTO_INCREMENT PRIMARY KEY, txt VARCHAR(50));
INSERT INTO notes (txt) VALUES ('hello via maxscale');
SELECT * FROM notes;
```

## 12.2 See which server answered

```sql
SELECT @@hostname;                 -- plain SELECT: goes to a replica (db2)

START TRANSACTION;
SELECT @@hostname;                 -- inside a transaction: goes to the primary (db1)
COMMIT;
```
(Your servers must have different hostnames for this to be obvious.)

## 12.3 Watch the load from MaxScale's side

```bash
maxctrl list servers          # the Connections column shows sessions per server
maxctrl list sessions         # every client connected through MaxScale
```

## 12.4 Prove the replica is read-only

Connect **directly** to db2 and try to write:
```bash
mysql -h 10.10.10.202 -u appuser -p -e "INSERT INTO labtest.notes (txt) VALUES ('direct')"
```
It should be refused (`--read-only`). Through MaxScale, the same insert works because MaxScale sends it to the primary. That is read/write splitting in action.

---

# 13. Managing MaxScale with `maxctrl` (No Restart Needed)

**This matches your production goal.** MaxScale lets you change most things **while running**, with no restart and no downtime.

## 13.1 Inspect
```bash
maxctrl list servers
maxctrl list services
maxctrl list listeners
maxctrl list monitors
maxctrl list sessions
maxctrl show server db1
maxctrl show service Read-Write-Service
maxctrl show monitor MariaDB-Monitor
maxctrl show maxscale
maxctrl list filters
```

## 13.2 Maintenance mode (take a server out of rotation)

```bash
maxctrl set server db2 maintenance        # no new connections go to db2
maxctrl clear server db2 maintenance      # put it back
```
Existing sessions on that server finish normally. Do this before patching or restarting a replica.

## 13.3 Add a server to a running MaxScale

```bash
maxctrl create server db3 10.10.10.204 3306
maxctrl link monitor MariaDB-Monitor db3
maxctrl link service Read-Write-Service db3
```
Remove it:
```bash
maxctrl unlink service Read-Write-Service db3
maxctrl unlink monitor MariaDB-Monitor db3
maxctrl destroy server db3
```
(Keep the license limit in mind, Section 5.)

## 13.4 Change a setting live
```bash
maxctrl alter service Read-Write-Service max_slave_replication_lag=10s
maxctrl alter monitor MariaDB-Monitor monitor_interval=1s
```

## 13.5 How runtime changes are stored

Changes made with `maxctrl` are saved by MaxScale in `/var/lib/maxscale/maxscale.cnf.d/`. They are loaded at startup **after** `/etc/maxscale.cnf` and **override** it for the same object. This is similar to MySQL's `SET PERSIST`: if you later edit the same object in `/etc/maxscale.cnf`, your edit may seem ignored. Prefer **one method** for changes and document it.

## 13.6 Reload TLS certificates without restart
```bash
maxctrl reload tls
```

## 13.7 Logs
```bash
sudo tail -f /var/log/maxscale/maxscale.log
sudo journalctl -u maxscale -n 50
```

---

# 14. Failover and Switchover

## 14.1 Planned switchover (safe, no data loss)

Promote db2 and demote db1:
```bash
maxctrl call command mariadbmon switchover MariaDB-Monitor db2 db1
maxctrl list servers
```
Format: `switchover <monitor> <new-primary> <current-primary>`. Applications keep using `10.10.10.203:4006` and notice only a brief pause.

## 14.2 Turn on automatic failover and rejoin

Add to the **monitor** section:
```ini
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
```

| Setting | Meaning |
|---|---|
| `auto_failover=true` | Promote a replica automatically if the primary stays down |
| `auto_rejoin=true` | When the old primary comes back, make it a replica of the new one |
| `failcount=5` | Number of failed checks before declaring the primary dead (with `monitor_interval=2s`, about 10 seconds) |
| `replication_user/password` | Credentials the monitor uses to set up replication during failover and rejoin |

Apply live:
```bash
maxctrl alter monitor MariaDB-Monitor auto_failover=true auto_rejoin=true failcount=5 \
        replication_user=repl replication_password=ReplPass#123
```

## 14.3 Practise a failover (test lab only)

```bash
# on db1
sudo systemctl stop mariadb
# on maxscale, watch
watch -n 1 maxctrl list servers
```
After about `failcount x monitor_interval`, db2 becomes `Master`. Start db1 again:
```bash
sudo systemctl start mariadb
```
With `auto_rejoin=true` it comes back as a replica.

## 14.4 Make applications survive a failover

On the **service**:
```ini
[Read-Write-Service]
...
transaction_replay=true
transaction_replay_max_size=1Mi
master_reconnection=true
delayed_retry=true
delayed_retry_timeout=10s
```
These let MaxScale reconnect to the new primary and, where possible, **replay** an interrupted transaction so the application sees no error. Test with your own application first.

---

# 15. Useful Router Settings (readwritesplit)

Add to `[Read-Write-Service]`, or apply live with `maxctrl alter service`.

| Setting | Meaning |
|---|---|
| `max_slave_replication_lag=10s` | Do not send reads to a replica that is more than 10 seconds behind |
| `slave_selection_criteria=LEAST_CURRENT_OPERATIONS` | How to pick a replica (default). Others: `LEAST_GLOBAL_CONNECTIONS`, `ADAPTIVE_ROUTING`, `LEAST_BEHIND_MASTER` |
| `master_accept_reads=true` | Let the primary also serve reads (good when you have few replicas) |
| `max_slave_connections=2` | Limit how many replicas each client session connects to |
| `causal_reads=local` | Guarantees a client sees its **own** writes (fixes read-after-write). Needs MariaDB and GTID; some versions use `true`/`global` instead |
| `connection_keepalive=300s` | Pings idle connections so they are not dropped |
| `max_connections=1000` (service) | Limit total client connections |
| `master_failure_mode=fail_on_write` | When there is no primary, still allow reads, fail only writes |

---

# 16. Filters (Optional)

Filters process queries on the way through. Example: **log every query** to a file with the Query Log All filter:

```ini
[QLA]
type=filter
module=qlafilter
filebase=/var/log/maxscale/qla
log_type=unified

[Read-Write-Service]
type=service
router=readwritesplit
cluster=MariaDB-Monitor
user=maxsvc
password=SvcPass#123
filters=QLA
```
This is useful for auditing and for finding what an application really sends. It adds overhead, so use it only when needed. Other filters include `regexfilter` (rewrite queries), `cache`, and `masking`.

---

# 17. If Your Database Is MySQL

You mentioned `mysqld` earlier, so you may be on **MySQL**, not MariaDB. Be careful:

| Topic | MariaDB | MySQL |
|---|---|---|
| Official support | Yes | Basic routing only (based on developer statements) |
| Failover with `mariadbmon` | Yes (uses MariaDB GTID) | Do not rely on it. Treat failover as **not supported** unless the current docs say otherwise |
| `causal_reads` | Yes | No (needs MariaDB features) |
| MySQL 8 default login (`caching_sha2_password`) | n/a | Reported as **not directly supported**. Accounts used through MaxScale should use `mysql_native_password` |

If you try it anyway with MySQL 8:
```sql
CREATE USER 'appuser'@'%' IDENTIFIED WITH mysql_native_password BY 'AppPass#123';
```
Do the same for the monitor/service users, and check the current documentation for your versions.

**Recommendation:** to learn MaxScale properly, use **MariaDB** for the backends. For a MySQL production setup, look at **ProxySQL** or **MySQL Router** instead.

---

# 18. Production Notes

- **Do not expose the REST API (port 8989).** Keep it on localhost (default) and use `maxctrl` locally.
- **Change the default admin user:**
```bash
maxctrl create user admin2 'StrongPassword!' --type=admin
maxctrl destroy user admin
```
(log in as the new user afterwards).
- **Encrypt passwords** in `maxscale.cnf` (Section 10.5) and keep `.secrets` backed up.
- **MaxScale is now a single point of failure.** Run **two** MaxScale servers and put a virtual IP (for example **Keepalived**) in front, or use a load balancer. If both run a monitor with automatic failover, make sure only one makes failover decisions (use MaxScale's cooperative monitoring; see the documentation).
- **Applications should use a connection pool** and handle reconnects.
- **Monitor lag** with `max_slave_replication_lag` and your normal monitoring.
- **Test failover regularly** in staging, and write a runbook.
- **Keep backups.** MaxScale does not back up your data.
- **Check the license** (Section 5) before using it in production, especially if you add more backend servers.
- **Plan upgrades:** upgrade replicas first, switchover, then upgrade the old primary, using maintenance mode and `switchover`.

---

# 19. Troubleshooting

| Problem | Likely cause and fix |
|---|---|
| MaxScale will not start | `sudo maxscale --config-check` and `sudo journalctl -u maxscale -n 50`. Usually a typo or wrong section name |
| Server shows `Down` | MariaDB not running, firewall closed (3306), wrong IP, or `maxmon` cannot log in from MaxScale's IP |
| Server shows `Running` but no `Master`/`Slave` | Replication not running, or the monitor user lacks privileges. Check `SHOW SLAVE STATUS\G` and the monitor grants |
| `Access denied` when connecting to port 4006 | User missing, wrong password, or the user's host pattern does not match. Remember backends see MaxScale's IP |
| `Authentication plugin 'caching_sha2_password' ...` | MySQL 8 login method. Use `mysql_native_password` (Section 17) |
| Connection to 4006 refused | MaxScale not running, firewall closed, or listener section wrong. `maxctrl list listeners`, `ss -tlnp \| grep 4006` |
| Everything goes to the primary | No healthy replica, replica lag over the limit, or statements run inside transactions. Check `maxctrl list servers` |
| Stale reads after writing | Replication lag. Use `causal_reads`, read inside the transaction, or lower `max_slave_replication_lag` |
| New user cannot connect | MaxScale caches users. Refresh: `maxctrl reload service Read-Write-Service`, or wait for the refresh interval |
| Failover did not happen | `auto_failover` off, `failcount` not reached, replica not caught up or no binlog, missing monitor grants |
| `maxctrl` says unauthorized or cannot connect | Wrong admin credentials, or MaxScale not running |
| My edit in `/etc/maxscale.cnf` is ignored | A runtime change in `/var/lib/maxscale/maxscale.cnf.d/` overrides it (Section 13.5) |

Where to look first:
```bash
maxctrl list servers
maxctrl list monitors
sudo tail -n 50 /var/log/maxscale/maxscale.log
sudo journalctl -u maxscale -n 50
ss -tlnp | grep maxscale
```
On the database side:
```sql
SHOW SLAVE STATUS\G
SELECT user, host FROM mysql.user;
SHOW GRANTS FOR 'maxmon'@'10.10.10.%';
```
SELinux does not normally block MaxScale, but if something odd happens, check `getenforce` and `sudo ausearch -m avc -ts recent`.

---

# 20. Cheat Sheet

```
 CONCEPTS
   server    = one real database
   monitor   = watches servers (health, primary/replica)
   service   = router + servers (readwritesplit, readconnroute)
   listener  = port clients connect to
   filter    = optional query processing

 FILES
   /etc/maxscale.cnf                     main config
   /var/lib/maxscale/maxscale.cnf.d/     runtime changes made with maxctrl
   /var/log/maxscale/maxscale.log        log

 BASIC COMMANDS
   systemctl enable --now maxscale
   maxscale --config-check
   maxctrl list servers | services | listeners | monitors | sessions
   maxctrl show server db1
   maxctrl set server db2 maintenance
   maxctrl clear server db2 maintenance
   maxctrl call command mariadbmon switchover MariaDB-Monitor NEWPRIMARY OLDPRIMARY
   maxctrl alter service SERVICE setting=value
   maxctrl create server NAME IP PORT
   maxctrl link service SERVICE SERVERNAME
```

## Minimal working config (copy and adapt)
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

## Order of work
```
 1. Two MariaDB servers with working replication (Slave_IO/SQL = Yes)
 2. Create users: maxmon, maxsvc, app user (host pattern allows MaxScale's IP)
 3. Open firewall: 3306 (databases), 4006 (MaxScale)
 4. Install MaxScale, write /etc/maxscale.cnf, run --config-check
 5. systemctl enable --now maxscale, then maxctrl list servers
 6. Point the application at MaxScale-IP:4006
 7. Practise: maintenance mode, switchover, then failover
```

## Practice ideas
1. Stop MariaDB on db2 and watch `maxctrl list servers`. Do reads still work?
2. Use `maxctrl set server db2 maintenance` and confirm all reads go to db1.
3. Run a switchover and check which server is `Master` afterwards.
4. Add `QLA` and look at the query log.
5. Turn on `auto_failover`, stop the primary, and time how long failover takes.
6. Add `max_slave_replication_lag=5s`, create artificial lag on the replica, and see reads move to the primary.
