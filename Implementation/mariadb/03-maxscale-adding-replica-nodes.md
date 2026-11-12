# MaxScale: Adding Replica Nodes (Node 2, then Node 3)

A complete, beginner-friendly guide for extending a MaxScale setup from one
MariaDB server to two, and then to three. Each node has its own section
because the problems faced when adding node 2 differ from those faced when
adding node 3. Every configuration line is explained, and every issue met
along the way is documented with its fix.

---

## Part 1: The Big Picture

### The starting point

```
App --> MaxScale (10.10.10.190:4006) --> mariadb-1 (10.10.10.201)  primary
```

### After adding node 2

```
App --> MaxScale (10.10.10.190:4006) --+--> mariadb-1 (10.10.10.201)  primary
                                       |
                                       +--> mariadb-2 (10.10.10.202)  replica
```

### After adding node 3

```
App --> MaxScale (10.10.10.190:4006) --+--> mariadb-1 (10.10.10.201)  primary
                                       |
                                       +--> mariadb-2 (10.10.10.202)  replica
                                       |
                                       +--> mariadb-3 (10.10.10.203)  replica
```

The application keeps using **one address**: MaxScale's. It never knows
how many servers are behind it. MaxScale routes:

- **Writes** (`INSERT`, `UPDATE`, `DELETE`, DDL) → primary
- **Reads** (`SELECT` outside a transaction) → a replica
- **Anything inside a transaction** → primary

### The core rule

> **Replication is set up at the MariaDB level first. MaxScale only
> discovers it and routes traffic.**

MaxScale cannot create replication. It watches it. Step order matters:
replication first, MaxScale second.

### Two replication methods, and why it matters

| Method | How it tracks position | MaxScale failover support |
|---|---|---|
| **File/position** (`MASTER_USE_GTID=no`) | "Read from file X at byte Y" | ❌ Does not work |
| **GTID** (`MASTER_USE_GTID=slave_pos`) | "Read every transaction I have not seen" | ✅ Required |

**Node 2 was initially added with file/position and later switched to
GTID** (see Part 6). **Node 3 is added with GTID from the start**, because
file/position caused repeated failures (Part 8).

GTID is simpler: no file names or byte positions to copy, no "truncated
binlog" errors, and automatic failover works.

---

## PART A: ADDING NODE 2 (mariadb-2)

---

### Part 2: Prepare mariadb-2 (10.10.10.202)

#### 2.1 Install MariaDB

```bash
sudo dnf install mariadb-server -y
sudo systemctl enable --now mariadb
```

#### 2.2 Server configuration

```bash
sudo vim /etc/my.cnf.d/mariadb-server.cnf
```

Under `[mysqld]`:

```ini
bind-address      = 0.0.0.0
server_id         = 2
log_bin           = mariadb-bin
binlog_format     = ROW
log_slave_updates = ON
read_only         = ON
report_host       = 10.10.10.202
```

**Why each setting is required:**

| Setting | Reason |
|---|---|
| `bind-address = 0.0.0.0` | Without it, MariaDB listens only on `127.0.0.1` and MaxScale cannot reach it from another machine |
| `server_id = 2` | Every server needs a **unique** ID. The primary is `1`, so the first replica must be `2`. Two servers with the same ID break replication with error 1593 |
| `log_bin = mariadb-bin` | Records this server's own changes, needed if this replica is later promoted to primary |
| `binlog_format = ROW` | Records the actual data changed rather than the SQL statement. Safest for replication |
| `log_slave_updates = ON` | Lets this replica pass on changes it receives, needed for promotion to primary |
| `read_only = ON` | Blocks normal users from writing directly to the replica. Prevents data divergence. MaxScale flips this at runtime during failover |
| `report_host = 10.10.10.202` | Shows the server's real address in replication status |

> **On `read_only = ON`:** this does **not** block replication. The
> replica's SQL thread still applies changes from the primary. It only
> blocks direct client writes. MaxScale automatically sets
> `read_only = OFF` when it promotes this node to primary during failover,
> and back to `ON` when it rejoins. You never flip it manually.

```bash
sudo systemctl restart mariadb
```

#### 2.3 Open the firewall

```bash
sudo firewall-cmd --permanent --add-port=3306/tcp
sudo firewall-cmd --reload
sudo firewall-cmd --list-ports
```

Must show `3306/tcp`. Without it, the TCP connection from MaxScale is
dropped before MariaDB sees it.

#### 2.4 Confirm the server listens on all interfaces

```bash
ss -tlnp | grep 3306
```

Must show `0.0.0.0:3306`, not `127.0.0.1:3306`. If it shows `127.0.0.1`,
`bind-address` did not take effect, restart MariaDB again.

---

### Part 3: Set Up Replication from the Primary

#### 3.1 On the primary (201), create the replication user

```sql
CREATE USER 'repl'@'10.10.10.%' IDENTIFIED BY '123';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'10.10.10.%';
```

**Why:** the replica logs in with this account to read the primary's binary
log. `REPLICATION SLAVE` is the minimum privilege for that.

#### 3.2 On the primary, note the current binlog position

```sql
SHOW MASTER STATUS;
```

Example:

```
+--------------------+----------+
| File               | Position |
+--------------------+----------+
| mariadb-bin.000005 |     1051 |
+--------------------+----------+
```

**Record the exact `File` and `Position`.** You will need them next.

#### 3.3 On the replica (202), point it at the primary

For node 2, replication was first set up with **file/position** (this was
later switched to GTID, see Part 6):

```sql
STOP SLAVE;
RESET SLAVE ALL;

CHANGE MASTER TO
  MASTER_HOST='10.10.10.201',
  MASTER_USER='repl',
  MASTER_PASSWORD='123',
  MASTER_LOG_FILE='mariadb-bin.000005',
  MASTER_LOG_POS=1051,
  MASTER_USE_GTID=no;

START SLAVE;
SHOW SLAVE STATUS\G
```

**Why each line:**

| Line | Reason |
|---|---|
| `STOP SLAVE; RESET SLAVE ALL;` | Clear any previous replication state so you start clean |
| `MASTER_HOST` | Must be **exactly** the same string MaxScale uses in its server section (`10.10.10.201`). Do not mix hostname and IP |
| `MASTER_USER` / `MASTER_PASSWORD` | The account from 3.1 |
| `MASTER_LOG_FILE` / `MASTER_LOG_POS` | Where to start reading the primary's binary log. Must be the **current** values from `SHOW MASTER STATUS` |
| `MASTER_USE_GTID=no` | Use file/position replication |

Both `Slave_IO_Running` and `Slave_SQL_Running` must say **Yes**.

> **Critical:** use the **current** `File` and `Position` from
> `SHOW MASTER STATUS` right now. If the primary has rotated its binlog to
> `mariadb-bin.000007`, pointing at `mariadb-bin.000002` fails with error
> 1236 (`binlog truncated in the middle of event`). If you are unsure,
> use GTID instead (Part 6) and skip file names entirely.

---

### Part 4: Add mariadb-2 to MaxScale

Pick **one** method.

#### Method A: Live, no restart (used for node 2)

On MaxScale (190):

```bash
maxctrl create server mariadb-2 10.10.10.202 3306
maxctrl link monitor MariaDB-Monitor mariadb-2
```

- `create server`, creates the new server object
- `link monitor`, tells the monitor to watch it

The service does not need editing. It uses `cluster=MariaDB-Monitor`,
meaning "use whatever servers the monitor watches." So adding the server
to the monitor automatically brings it into the service.

#### Method B: File edit (needs restart)

In `/etc/maxscale.cnf`:

```ini
[mariadb-2]
type=server
address=10.10.10.202
port=3306
```

In the monitor:

```ini
[MariaDB-Monitor]
servers=mariadb-1,mariadb-2
```

```bash
sudo -u maxscale maxscale --config-check
sudo systemctl restart maxscale
```

**Do not use both methods.** Using both creates the server object twice.

#### What changed in the config

```
Before (1 server)               After (2 servers)
[mariadb-1] type=server         [mariadb-1] type=server
                                [mariadb-2] type=server         <-- new
servers=mariadb-1               servers=mariadb-1,mariadb-2     <-- changed
cluster=MariaDB-Monitor         cluster=MariaDB-Monitor         <-- unchanged
port=4006                       port=4006                        <-- unchanged
```

Only the monitor's server list changed. The service and listener are
untouched. Clients keep using the same address.

---

### Part 5: Create the `max` User on mariadb-2

The monitor logs in to **every** server in `servers=`. The `max` user must
exist on the replica with identical grants and password.

With **file/position replication**, users created on the primary **before**
the replication starting position are **not** replayed. The `max` user was
created on the primary earlier, so it does **not** appear on the replica.
It must be created manually.

On mariadb-2 (202):

```sql
CREATE USER 'max'@'10.10.10.%' IDENTIFIED BY '123';
GRANT REPLICA MONITOR ON *.*             TO 'max'@'10.10.10.%';
GRANT SELECT ON mysql.user               TO 'max'@'10.10.10.%';
GRANT SELECT ON mysql.global_priv        TO 'max'@'10.10.10.%';
GRANT SELECT ON mysql.db                 TO 'max'@'10.10.10.%';
GRANT SELECT ON mysql.tables_priv        TO 'max'@'10.10.10.%';
GRANT SELECT ON mysql.columns_priv       TO 'max'@'10.10.10.%';
GRANT SELECT ON mysql.proxies_priv       TO 'max'@'10.10.10.%';
GRANT SELECT ON mysql.roles_mapping      TO 'max'@'10.10.10.%';
GRANT SHOW DATABASES ON *.*              TO 'max'@'10.10.10.%';

-- Required later for automatic failover (Part 6 onwards)
GRANT RELOAD, PROCESS, SHOW DATABASES, EVENT, SET USER, READ_ONLY ADMIN
  ON *.* TO 'max'@'10.10.10.%';
GRANT REPLICATION SLAVE ADMIN, BINLOG ADMIN, CONNECTION ADMIN
  ON *.* TO 'max'@'10.10.10.%';

FLUSH PRIVILEGES;
```

**Why each grant group:**

| Group | Purpose |
|---|---|
| `REPLICA MONITOR` | Read replication status (`SHOW SLAVE STATUS`) |
| `SELECT ON mysql.*` | Read the account list so MaxScale can authenticate clients |
| `RELOAD, PROCESS, SHOW DATABASES, EVENT, SET USER, READ_ONLY ADMIN` | Change server state during failover |
| `REPLICATION SLAVE ADMIN, BINLOG ADMIN, CONNECTION ADMIN` | Run `STOP SLAVE` / `CHANGE MASTER` during failover |

Without the last two grant groups, failover **will fail** with
`Access denied; you need (at least one of) the SUPER, REPLICATION SLAVE ADMIN privilege(s)`.

**Only one row per host pattern.** Confirm:

```sql
SELECT user, host FROM mysql.user WHERE user='max';
```

Must show exactly `max | 10.10.10.%`. A second row such as
`'max'@'10.10.10.190'` shadows the wildcard row and causes `Auth Error`.

---

### Part 6: Switch to GTID (so failover works)

File/position replication does not support MaxScale's `auto_failover`.
Switch the replica to GTID:

On mariadb-2 (202):

```sql
STOP SLAVE;
RESET SLAVE ALL;

CHANGE MASTER TO
  MASTER_HOST='10.10.10.201',
  MASTER_USER='repl',
  MASTER_PASSWORD='123',
  MASTER_USE_GTID=slave_pos;

START SLAVE;
SHOW SLAVE STATUS\G
```

**What changed:** only the position method. Instead of
`MASTER_LOG_FILE` + `MASTER_LOG_POS`, GTID uses `MASTER_USE_GTID=slave_pos`,
which means "resume from the GTID position I already have applied." No data
changes, only the bookkeeping method.

Verify:

```
Using_Gtid: Slave_Pos       <-- must be this, not No
Slave_IO_Running: Yes
Slave_SQL_Running: Yes
```

Then restart MaxScale to apply:

```bash
sudo systemctl restart maxscale
sleep 5
maxctrl list servers
```

Expected:

```
│ mariadb-1 │ 10.10.10.201 │ 3306 │ 0 │ Master, Running │ ... │
│ mariadb-2 │ 10.10.10.202 │ 3306 │ 0 │ Slave, Running  │ ... │
```

---

### Part 7: Issues Encountered While Adding Node 2

#### 7.1 `Auth Error, Down` in `maxctrl list servers`

**Symptom:** `maxctrl list servers` shows `Auth Error, Down` even though a
manual login from the shell works.

**Cause:** MaxScale's monitor runs a specific set of queries at connect
time (replication status, account list). If any required grant is missing,
MariaDB rejects the login.

**Fix:** verify grants on the actual server:
```sql
SHOW GRANTS FOR 'max'@'10.10.10.%';
```
Confirm `REPLICA MONITOR` and the `SELECT ON mysql.*` grants are present.

#### 7.2 `Access denied … (using password: NO)`

**Symptom:** MaxScale log shows `Access denied for user 'max'@'…' (using
password: NO)`.

**Cause:** the password was never sent. Several sub-causes:

| Sub-cause | Fix |
|---|---|
| `password=123` parsed as a JSON integer, not a string | Use `password="123"` (quoted) |
| `/var/lib/maxscale/.secrets` exists → encrypted-password mode → plaintext value fails to decrypt | Move `.secrets` aside, or run `maxpasswd 123` and paste the hex into `password=` |
| `maxctrl alter … password=123` rejected with `Expected a json string, but got a json integer` | Use `password='"123"'` on the command line |

**The `.secrets` issue:** when `/var/lib/maxscale/.secrets` exists, MaxScale
treats every `password=` as an **encrypted** value. A plaintext `"123"`
cannot be decrypted, so MaxScale sends no password. Fix with:

```bash
sudo systemctl stop maxscale
sudo mv /var/lib/maxscale/.secrets /var/lib/maxscale/.secrets.bak
sudo systemctl start maxscale
```

For production, keep `.secrets` and run `maxpasswd 123`, pasting the hex
output into `password=`.

#### 7.3 Duplicate `max` user

**Cause:** two rows: `'max'@'10.10.10.190'` and `'max'@'10.10.10.%'`.
MariaDB matches the **most specific host first**, so MaxScale (from
`.190`) authenticated against the `.190` row, which had different grants.

**Fix:**
```sql
DROP USER 'max'@'10.10.10.190';
FLUSH PRIVILEGES;
```

#### 7.4 `ERROR 2026 (HY000): SSL is required, but the server does not support it`

**Cause:** the MariaDB client on the MaxScale host defaults to requiring
TLS, but the server has no certificates. MaxScale's monitor uses the same
client library and hits the same wall.

**Fix (temporary):**
```bash
sudo tee /etc/my.cnf.d/zz-no-ssl.cnf >/dev/null <<'EOF'
[client]
ssl=0
EOF
```

**Fix (production):** add `ssl_cert` and `ssl_key` to `[mysqld]` on the
server so TLS is available. Never disable TLS in production.

#### 7.5 `ERROR 2002 (HY000): Can't connect to server … (115)`

**Cause:** TCP-level reachability problem. The packet never reaches the
server. Firewall, route, or interface misconfiguration.

**Diagnostic:**

```bash
# on MaxScale
ping -c 2 10.10.10.202
timeout 5 bash -c 'cat < /dev/null > /dev/tcp/10.10.10.202/3306' && echo OPEN || echo BLOCKED
ip neigh show 10.10.10.202

# on the server
ss -tlnp | grep 3306
sudo firewall-cmd --list-all
ip -4 addr show
ip route get 10.10.10.190     # src must equal the server's own address
```

**Common causes:**

- `bind-address` still set to `127.0.0.1`
- `firewall-cmd --list-ports` empty (port not open)
- The server's static IP is not on the interface expected, so ARP fails
  (`ip neigh` shows `FAILED`, `ping` returns `Destination Host Unreachable`)

#### 7.6 Missing database and table on the replica

**Symptom:** the `test` database exists on the primary but not on the
replica, even though replication looks healthy.

**Cause:** file/position replication only replays events **from the
starting position onward**. Objects created on the primary before that
position (`CREATE DATABASE test`, `CREATE USER 'max'`) are never replayed.

**Fix:** create the missing objects on the replica manually, or take a
fresh dump from the primary and start replication from the position
recorded in the dump. Switching to GTID (Part 6) avoids the issue entirely
because GTID replays the full history.

#### 7.7 `Got fatal error 1236 … binlog truncated in the middle of event`

**Cause:** the replica pointed at an old binlog file that had been
truncated or rotated away on the primary.

**Fix:** reset replication and point at the primary's **current** position
(`SHOW MASTER STATUS`), or switch to GTID:

```sql
STOP SLAVE;
RESET SLAVE ALL;
CHANGE MASTER TO
  MASTER_HOST='10.10.10.201',
  MASTER_USER='repl',
  MASTER_PASSWORD='123',
  MASTER_USE_GTID=slave_pos;
START SLAVE;
```

#### 7.8 Typos in monitor option names

**Cause:** `autofailover=ture` and `autorejoin=true`.

**Fix:** the correct names are `auto_failover=true` and `auto_rejoin=true`.
Option names use underscores; booleans are `true` / `false`.

#### 7.9 Failover fails: `Failed to stop replica connection on 'mariadb-2': Access denied; you need (at least one of) the SUPER, REPLICATION SLAVE ADMIN privilege(s)`

**Symptom:** failover starts: MaxScale selects mariadb-2 and tries to
promote it, then fails:

```
Failed to stop replica connection on 'mariadb-2': Query
'SET STATEMENT max_statement_time=3 FOR STOP SLAVE' failed:
'Access denied; you need … SUPER, REPLICATION SLAVE ADMIN privilege(s)'
(1227). Failover 'mariadb-1' -> 'mariadb-2' failed.
```

**Cause:** the `max` user had only the **monitoring** grants
(`REPLICA MONITOR`, `SELECT ON mysql.*`). To perform failover, MaxScale must
run `STOP SLAVE` and `CHANGE MASTER TO` on the replica, which require
`REPLICATION SLAVE ADMIN` (or `SUPER`) and related privileges.

**Fix:** on the replica (and the primary, for future rejoin):

```sql
GRANT RELOAD, PROCESS, SHOW DATABASES, EVENT, SET USER, READ_ONLY ADMIN
  ON *.* TO 'max'@'10.10.10.%';
GRANT REPLICATION SLAVE ADMIN, BINLOG ADMIN, CONNECTION ADMIN
  ON *.* TO 'max'@'10.10.10.%';
FLUSH PRIVILEGES;
```

Then reload the monitor so it picks up the new privileges:

```bash
maxctrl stop monitor MariaDB-Monitor
maxctrl start monitor MariaDB-Monitor
```

Failover will now proceed within `failcount × monitor_interval` seconds.

---

## PART B: ADDING NODE 3 (mariadb-3)

Node 3 is added with **GTID from the start**, which avoids the file/position
problems faced with node 2 (errors 1236, missing historical objects, manual
user creation).

---

### Part 8: Prepare mariadb-3 (10.10.10.203)

#### 8.1 Install MariaDB

```bash
sudo dnf install mariadb-server -y
sudo systemctl enable --now mariadb
```

#### 8.2 Server configuration

```bash
sudo vim /etc/my.cnf.d/mariadb-server.cnf
```

Under `[mysqld]`:

```ini
bind-address      = 0.0.0.0
server_id         = 3
log_bin           = mariadb-bin
binlog_format     = ROW
log_slave_updates = ON
read_only         = ON
report_host       = 10.10.10.203
```

**Only two values differ from mariadb-2:**

| Setting | mariadb-2 | mariadb-3 | Why it must differ |
|---|---|---|---|
| `server_id` | `2` | `3` | Every server needs a unique ID. Two servers with the same ID break replication with error 1593 |
| `report_host` | `10.10.10.202` | `10.10.10.203` | Shows the server's own address in replication status |

Everything else is identical.

```bash
sudo systemctl restart mariadb
```

#### 8.3 Firewall

```bash
sudo firewall-cmd --permanent --add-port=3306/tcp
sudo firewall-cmd --reload
sudo firewall-cmd --list-ports
```

#### 8.4 Confirm the server listens on all interfaces

```bash
ss -tlnp | grep 3306
```

Must show `0.0.0.0:3306`.

---

### Part 9: Set Up GTID Replication from the Primary

The `repl` user already exists on the primary (Part 3.1). Do **not** recreate
it.

On mariadb-3 (203):

```sql
STOP SLAVE;
RESET SLAVE ALL;

CHANGE MASTER TO
  MASTER_HOST='10.10.10.201',
  MASTER_USER='repl',
  MASTER_PASSWORD='123',
  MASTER_USE_GTID=slave_pos;

START SLAVE;
SHOW SLAVE STATUS\G
```

Expected:

```
Slave_IO_Running: Yes
Slave_SQL_Running: Yes
Using_Gtid: Slave_Pos
Seconds_Behind_Master: 0
Last_IO_Errno: 0
Last_Error:
```

**Why GTID and not file/position:** with file/position you must copy the
exact `File` and `Position` from `SHOW MASTER STATUS` at the moment you run
`CHANGE MASTER`. If the primary has rotated its binlog (e.g. to
`mariadb-bin.000007`) and you paste a stale file name (e.g.
`mariadb-bin.000002`), replication fails with error 1236. GTID asks the
primary "what do you have that I don't" and resumes automatically. No file
numbers to copy.

---

### Part 10: Create the `max` User on mariadb-3

The monitor logs in to every server in `servers=`. With GTID replication,
the `max` user created on the primary **will** replicate to the new node,
unlike file/position, where it did not.

Check first:

```sql
SELECT user, host FROM mysql.user WHERE user='max';
```

If it exists, verify failover grants are present:

```sql
SHOW GRANTS FOR 'max'@'10.10.10.%';
```

If `REPLICATION SLAVE ADMIN` or `READ_ONLY ADMIN` are missing, add them:

```sql
GRANT RELOAD, PROCESS, SHOW DATABASES, EVENT, SET USER, READ_ONLY ADMIN
  ON *.* TO 'max'@'10.10.10.%';
GRANT REPLICATION SLAVE ADMIN, BINLOG ADMIN, CONNECTION ADMIN
  ON *.* TO 'max'@'10.10.10.%';
FLUSH PRIVILEGES;
```

If the user is **missing entirely**, create it with the full grant set:

```sql
CREATE USER 'max'@'10.10.10.%' IDENTIFIED BY '123';
GRANT REPLICA MONITOR ON *.*             TO 'max'@'10.10.10.%';
GRANT SELECT ON mysql.user               TO 'max'@'10.10.10.%';
GRANT SELECT ON mysql.global_priv        TO 'max'@'10.10.10.%';
GRANT SELECT ON mysql.db                 TO 'max'@'10.10.10.%';
GRANT SELECT ON mysql.tables_priv        TO 'max'@'10.10.10.%';
GRANT SELECT ON mysql.columns_priv       TO 'max'@'10.10.10.%';
GRANT SELECT ON mysql.proxies_priv       TO 'max'@'10.10.10.%';
GRANT SELECT ON mysql.roles_mapping      TO 'max'@'10.10.10.%';
GRANT SHOW DATABASES ON *.*              TO 'max'@'10.10.10.%';
GRANT RELOAD, PROCESS, SHOW DATABASES, EVENT, SET USER, READ_ONLY ADMIN
  ON *.* TO 'max'@'10.10.10.%';
GRANT REPLICATION SLAVE ADMIN, BINLOG ADMIN, CONNECTION ADMIN
  ON *.* TO 'max'@'10.10.10.%';
FLUSH PRIVILEGES;
```

**Only one row per host pattern.** Confirm:

```sql
SELECT user, host FROM mysql.user WHERE user='max';
```

Must show exactly `max | 10.10.10.%`.

---

### Part 11: Register mariadb-3 with MaxScale

Pick **one** method. Node 3 was added via the **file edit**.

#### Method A: File edit (used for node 3)

In `/etc/maxscale.cnf`, add:

```ini
[mariadb-3]
type=server
address=10.10.10.203
port=3306
```

Add to the monitor:

```ini
[MariaDB-Monitor]
servers=mariadb-1,mariadb-2,mariadb-3
```

Validate and restart:

```bash
sudo -u maxscale maxscale --config-check
sudo systemctl restart maxscale
```

The config check output should include:

```
Password encryption key file '/var/lib/maxscale/.secrets' not found, using configured passwords as plaintext.
Configuration was successfully verified.
```

That is correct, plaintext mode is what we want for this setup.

#### Method B: Live, no restart

```bash
maxctrl create server mariadb-3 10.10.10.203 3306
maxctrl link monitor MariaDB-Monitor mariadb-3
```

**Do not use both.** Using both creates the server object twice. If a
duplicate exists, remove it:

```bash
maxctrl destroy server mariadb-3
```

then re-add via one method only.

---

### Part 12: Verify

From MaxScale (190):

```bash
mariadb --skip-ssl -h 10.10.10.203 -u max -p123 -e "SELECT 1"
```

Must return `1`.

```bash
maxctrl list servers
```

Expected:

```
│ mariadb-1 │ 10.10.10.201 │ 3306 │ 0 │ Master, Running │ 0-1-23 │ MariaDB-Monitor │
│ mariadb-2 │ 10.10.10.202 │ 3306 │ 0 │ Slave, Running  │ ...    │ MariaDB-Monitor │
│ mariadb-3 │ 10.10.10.203 │ 3306 │ 0 │ Slave, Running  │ ...    │ MariaDB-Monitor │
```

If `mariadb-3` shows `Auth Error, Down`, check the log:

```bash
sudo tail -n 30 /var/log/maxscale/maxscale.log | grep -i mariadb-3
```

| Log message | Cause | Fix |
|---|---|---|
| `Access denied … (using password: YES)` | Password mismatch or missing failover grants | Re-grant, or `ALTER USER 'max'@'10.10.10.%' IDENTIFIED BY '123';` |
| `Access denied … (using password: NO)` | Password not applied (`.secrets` present or unquoted value) | `ls /var/lib/maxscale/.secrets`; ensure `password="123"` in file |
| `Can't connect … (115)` | Network / firewall / wrong interface address | `firewall-cmd --list-ports`, `bind-address`, `ip addr show` |
| `Access denied; you need … REPLICATION SLAVE ADMIN` | Failover grants missing | Apply the failover `GRANT` block from Part 10 |

Confirm replication reaches mariadb-3:

On the primary (201):

```sql
USE test;
INSERT INTO test.notes (txt) VALUES ('check mariadb-3');
```

On mariadb-3 (203):

```sql
SELECT * FROM test.notes WHERE txt = 'check mariadb-3';
```

The row must appear.

---

### Part 13: Issues Encountered While Adding Node 3

#### 13.1 `Last_IO_Errno: 1236`: `binlog truncated in the middle of event`

**Symptom:**

```
Last_IO_Errno: 1236
Last_IO_Error: Got fatal error 1236 from master when reading data from
binary log: 'binlog truncated in the middle of event; consider out of disk
space on master; the first event 'mariadb-bin.000002' at 1203, the last byte
read from 'mariadb-bin.000002' at 1222.'
```

**Cause:** the replica was pointed at an **old** binlog file
(`mariadb-bin.000002`) that had been rotated away on the primary. The
current file was `mariadb-bin.000007`. The file number and position were
stale, probably carried over from a previous `CHANGE MASTER` attempt.

**Fix:** on the replica, confirm the current file on the primary, then
re-point:

On the primary (201):

```sql
SHOW MASTER STATUS;
```

Note the exact `File` and `Position`.

On the replica (203):

```sql
STOP SLAVE;
RESET SLAVE ALL;

CHANGE MASTER TO
  MASTER_HOST='10.10.10.201',
  MASTER_USER='repl',
  MASTER_PASSWORD='123',
  MASTER_LOG_FILE='mariadb-bin.000007',   -- from SHOW MASTER STATUS
  MASTER_LOG_POS=<position>,              -- from SHOW MASTER STATUS
  MASTER_USE_GTID=no;

START SLAVE;
```

**Better fix, use GTID:** file/position replication is prone to this
error every time the primary rotates its binlog. GTID ignores file names:

```sql
STOP SLAVE;
RESET SLAVE ALL;
CHANGE MASTER TO
  MASTER_HOST='10.10.10.201',
  MASTER_USER='repl',
  MASTER_PASSWORD='123',
  MASTER_USE_GTID=slave_pos;
START SLAVE;
```

#### 13.2 `ERROR 1200 (HY000): Misconfigured slave: MASTER_HOST was not set`

**Symptom:** `START SLAVE` fails with this error, and `SHOW SLAVE STATUS`
returns `Empty set`.

**Cause:** the `CHANGE MASTER TO` did not execute, it was either rejected
silently, interrupted, or never run. Connection info was never stored.

**Fix:** run `CHANGE MASTER TO` as a **single line**, watch for `Query OK`,
and check `SHOW SLAVE STATUS` **before** starting:

```sql
STOP SLAVE;
RESET SLAVE ALL;
CHANGE MASTER TO MASTER_HOST='10.10.10.201', MASTER_USER='repl', MASTER_PASSWORD='123', MASTER_USE_GTID=slave_pos;
SHOW SLAVE STATUS\G
```

`SHOW SLAVE STATUS` must now show `Master_Host: 10.10.10.201`. If it still
returns `Empty set`, `CHANGE MASTER` did not execute, look for an error
line.

If the state files are stale:

```bash
sudo systemctl stop mariadb
sudo rm -f /var/lib/mysql/master.info /var/lib/mysql/relay-log.info
sudo systemctl start mariadb
```

Then re-run `CHANGE MASTER` and `START SLAVE`.

#### 13.3 `Last_IO_Errno: 1593`: `master and slave have equal MariaDB server ids`

**Symptom:**

```
Last_IO_Errno: 1593
Last_IO_Error: Fatal error: The slave I/O thread stops because master and
slave have equal MariaDB server ids; these ids must be different for
replication to work
```

**Cause:** the replica had the same `server_id` as the primary (both `1`).
MariaDB uses `server_id` to distinguish its own events from the primary's.
Identical IDs break replication entirely.

**Fix:** on the replica (203), edit
`/etc/my.cnf.d/mariadb-server.cnf`:

```ini
[mysqld]
server_id = 3
```

```bash
sudo systemctl restart mariadb
```

Then restart replication:

```sql
STOP SLAVE;
START SLAVE;
SHOW SLAVE STATUS\G
```

Verify:

```sql
SHOW VARIABLES LIKE 'server_id';   -- must be 3
```

Both replication threads must return to `Yes`.

#### 13.4 `Auth Error, Down` after adding mariadb-3

**Symptom:** `maxctrl list servers` shows `mariadb-3` as `Auth Error, Down`.

**Cause:** the `max` user on `.203` is missing, has a different password, or
lacks failover grants.

**Fix:**

1. Test the login from MaxScale:
   ```bash
   mariadb --skip-ssl -h 10.10.10.203 -u max -p123 -e "SELECT 1"
   ```
2. If it fails, read the log:
   ```bash
   sudo tail -n 30 /var/log/maxscale/maxscale.log | grep -i mariadb-3
   ```
3. Create or fix the `max` user on `.203` (Part 10).
4. If the user exists but lacks failover grants, add them (Part 10).
5. Reload the monitor:
   ```bash
   maxctrl stop monitor MariaDB-Monitor
   maxctrl start monitor MariaDB-Monitor
   ```

#### 13.5 `SHOW MASTER STATUS` on the replica shows a different file than on the primary

**Symptom:** on the replica, `SHOW MASTER STATUS` returns
`mariadb-bin.000003`, but the primary's `SHOW MASTER STATUS` returns
`mariadb-bin.000007`.

**Cause:** this is **expected and harmless**. `SHOW MASTER STATUS` on the
replica shows the replica's **own** binary log (it records what it applied),
not the primary's. The replica's replication source is shown by
`Master_Log_File` in `SHOW SLAVE STATUS`, which will match the primary's
current file.

**Fix:** none required. Ignore `SHOW MASTER STATUS` on replicas. Use
`SHOW SLAVE STATUS\G` to check replication, and `SHOW MASTER STATUS` only on
the primary.

---

### Part 14: Final Configuration Reference

#### `/etc/maxscale.cnf` (complete, after adding nodes 2 and 3)

```ini
[maxscale]
threads=auto

[mariadb-1]
type=server
address=10.10.10.201
port=3306

[mariadb-2]
type=server
address=10.10.10.202
port=3306

[mariadb-3]
type=server
address=10.10.10.203
port=3306

[MariaDB-Monitor]
type=monitor
module=mariadbmon
servers=mariadb-1,mariadb-2,mariadb-3
user=max
password="123"
monitor_interval=2s
failcount=5
replication_user=repl
replication_password="123"
auto_failover=true
auto_rejoin=true

[Read-Write-Service]
type=service
router=readwritesplit
cluster=MariaDB-Monitor
user=max
password="123"

[Read-Write-Listener]
type=listener
service=Read-Write-Service
protocol=MariaDBClient
port=4006
```

#### Grants on every database server

```sql
-- Monitor and service
GRANT REPLICA MONITOR ON *.*             TO 'max'@'10.10.10.%';
GRANT SELECT ON mysql.user               TO 'max'@'10.10.10.%';
GRANT SELECT ON mysql.global_priv        TO 'max'@'10.10.10.%';
GRANT SELECT ON mysql.db                 TO 'max'@'10.10.10.%';
GRANT SELECT ON mysql.tables_priv        TO 'max'@'10.10.10.%';
GRANT SELECT ON mysql.columns_priv       TO 'max'@'10.10.10.%';
GRANT SELECT ON mysql.proxies_priv       TO 'max'@'10.10.10.%';
GRANT SELECT ON mysql.roles_mapping      TO 'max'@'10.10.10.%';
GRANT SHOW DATABASES ON *.*              TO 'max'@'10.10.10.%';

-- Failover operations
GRANT RELOAD, PROCESS, SHOW DATABASES, EVENT, SET USER, READ_ONLY ADMIN
  ON *.* TO 'max'@'10.10.10.%';
GRANT REPLICATION SLAVE ADMIN, BINLOG ADMIN, CONNECTION ADMIN
  ON *.* TO 'max'@'10.10.10.%';
```

#### Files that matter

```
/etc/maxscale.cnf                   main configuration
/var/lib/maxscale/maxscale.cnf.d/   live overrides (win over main file)
/var/lib/maxscale/.secrets          toggles encrypted-password mode
/var/log/maxscale/maxscale.log      log, first place to look
/etc/my.cnf.d/zz-no-ssl.cnf         client TLS override (temporary)
```

#### Commands

```bash
# Inspect
maxctrl list servers
maxctrl show monitor MariaDB-Monitor
maxctrl show service Read-Write-Service

# Validate and restart
sudo -u maxscale maxscale --config-check
sudo systemctl restart maxscale

# Live changes
maxctrl create server NAME IP PORT
maxctrl link monitor MariaDB-Monitor NAME
maxctrl destroy server NAME
maxctrl stop monitor MariaDB-Monitor
maxctrl start monitor MariaDB-Monitor

# Logs
sudo tail -n 50 /var/log/maxscale/maxscale.log
```

---

### Part 15: Rules to Remember

1. **Replication first, MaxScale second.** MaxScale sees replication; it does not create it.
2. **Unique `server_id` on every server.** Duplicates break replication (error 1593).
3. **Use GTID for new nodes.** `MASTER_USE_GTID=slave_pos` avoids truncated-binlog errors (1236) and replays the full history.
4. **`max` user on every server** in `servers=`, with identical grants and password.
5. **Quote passwords.** `"123"` is a string; `123` is a number and gets dropped.
6. **One `max` user per host pattern.** Duplicates cause the wrong row to be matched.
7. **`.secrets` + plaintext password = failure.** Remove `.secrets` or use `maxpasswd`.
8. **`MASTER_HOST` must match the MaxScale address exactly**, no hostname/IP mixing.
9. **Monitor grants ≠ failover grants.** `REPLICA MONITOR` reads state; `REPLICATION SLAVE ADMIN` changes it.
10. **`read_only = ON` on replicas.** MaxScale flips it at runtime during failover.
11. **`failcount × monitor_interval` is the failover delay.** `5 × 2s = 10s`.
12. **Do not use both `maxctrl create server` and a file section for the same node.** Pick one.
13. **The application never changes.** It always uses MaxScale's address and port.
