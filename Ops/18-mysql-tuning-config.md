# MySQL Tuning and Configuration Guide (Production, No-Restart Edition)

A beginner-friendly guide to configuring and tuning MySQL, and above all to **changing settings on a live production server without restarting it**.

**Contents**

1. The big picture
2. Where settings live (config files)
3. Three ways to change a setting
4. Changing settings in production without a restart (step by step)
5. Which settings need a restart and which don't
6. Step 0 of tuning: know your server
7. Memory tuning
8. InnoDB I/O and durability
9. Connections and threads
10. Temp tables and table cache
11. Logging (error, slow, binary)
12. Query tuning (the biggest win)
13. Sample annotated `my.cnf`
14. If your data directory is on NFS
15. Monitoring cheat sheet
16. Security and backups (short)
17. Troubleshooting
18. MariaDB differences
19. Cheat sheet

> Commands are for **AlmaLinux / RHEL / Rocky** with **MySQL 8.0+** (service name `mysqld`). Check yours first:
> ```sql
> SELECT VERSION();
> ```
> All numbers here are **starting points, not magic values**. Test in a non-production copy first, change **one thing at a time**, and measure.

---

# 1. The Big Picture

## What is tuning?

Tuning means adjusting MySQL's settings and your queries so the server uses the hardware well and answers fast. Most gains come from just three things:

```
 1. Good queries and indexes      <-- biggest win (most problems live here)
 2. Right memory settings         <-- InnoDB buffer pool
 3. Right I/O and connection settings
```

> Bad queries cannot be fixed by settings. Always check slow queries (Section 12) before blaming configuration.

## The golden rules of tuning

```
 1. Measure first        (baseline numbers before any change)
 2. Change ONE thing     (so you know what helped)
 3. Measure again        (did it improve?)
 4. Write it down        (old value, new value, date, reason)
 5. Keep a rollback      (you must be able to undo it)
```

## The layers of a MySQL setting

A setting (called a **system variable**) exists in several places at once. Understanding this is the key to everything below.

```
  +--------------------------------------------------------------+
  | 1. Compiled-in default      (what MySQL ships with)          |
  +--------------------------------------------------------------+
  | 2. Config file  /etc/my.cnf and /etc/my.cnf.d/*.cnf          |
  |        read ONLY when mysqld STARTS                          |
  +--------------------------------------------------------------+
  | 3. mysqld-auto.cnf  (written by SET PERSIST)                 |
  |        read at startup, AFTER my.cnf, so it WINS             |
  +--------------------------------------------------------------+
  | 4. Running value in memory  (SET GLOBAL changes THIS)        |
  |        takes effect immediately, lost on restart             |
  +--------------------------------------------------------------+
  | 5. Session value  (SET SESSION, one connection only)         |
  +--------------------------------------------------------------+
```

**The most important fact:** MySQL reads `my.cnf` only at startup. Editing the file does **nothing** to a running server. That is why production tuning is done from inside the database, and then written to the file so it survives the next restart.

---

# 2. Where Settings Live (Config Files)

## 2.1 File locations (RHEL family)

```
 /etc/my.cnf                  <- main file
 /etc/my.cnf.d/*.cnf          <- extra files (read after my.cnf)
 /var/lib/mysql/mysqld-auto.cnf   <- written by SET PERSIST (inside the datadir)
```

Ask MySQL which files it reads and in what order:
```bash
mysqld --verbose --help 2>/dev/null | grep -A1 "Default options"
```
Typical answer: `/etc/my.cnf /etc/mysql/my.cnf /usr/etc/my.cnf ~/.my.cnf`

See what the files currently set:
```bash
mysqld --print-defaults
```

## 2.2 File structure

```ini
[mysqld]                 # settings for the SERVER
innodb_buffer_pool_size = 8G
max_connections         = 300

[client]                 # settings for client programs (mysql, mysqldump)
default-character-set = utf8mb4

[mysql]                  # settings for the mysql command-line tool only
prompt = \u@\h [\d]>\_
```

Rules:
- Settings for the server **must** go under `[mysqld]`.
- Use `_` or `-` interchangeably in names (`max_connections` = `max-connections`).
- Size suffixes: `K`, `M`, `G` (e.g. `8G`).
- Edit with: `sudo vim /etc/my.cnf`
- If the same setting appears twice, the **last one wins**.

## 2.3 Validate before you restart (important in production)

A typo in `my.cnf` can stop MySQL from starting. Always check first:
```bash
mysqld --validate-config
```
No output and exit code 0 means the files are valid. Run this **before** every planned restart.

---

# 3. Three Ways to Change a Setting

## Way 1: Edit `my.cnf`, then restart

```bash
sudo vim /etc/my.cnf
sudo systemctl restart mysqld
```
- Survives restarts.
- **Needs a restart**, so it causes downtime. Not suitable for a busy production server unless planned.

## Way 2: `SET GLOBAL` (live change, temporary)

```sql
SET GLOBAL max_connections = 500;
```
- Applies **immediately**, no restart.
- **Lost at the next restart** (reverts to what the file says).
- Works only for **dynamic** variables (Section 5).

## Way 3: `SET PERSIST` (live change, permanent) (MySQL 8.0+)

```sql
SET PERSIST max_connections = 500;
```
- Applies **immediately** AND writes to `mysqld-auto.cnf`, so it survives restarts.
- Only for dynamic variables.

Related commands:

| Command | What it does |
|---|---|
| `SET GLOBAL x = v;` | Change now, forget at restart |
| `SET PERSIST x = v;` | Change now **and** remember across restarts |
| `SET PERSIST_ONLY x = v;` | **Do not** change now; remember for next restart. Works for non-dynamic (read-only) variables too |
| `RESET PERSIST x;` | Remove one persisted setting |
| `RESET PERSIST;` | Remove all persisted settings |
| `SET SESSION x = v;` | Change for **this connection only** (good for testing) |

Permissions needed: `SYSTEM_VARIABLES_ADMIN` (or `SUPER`), plus `PERSIST_RO_VARIABLES_ADMIN` for `PERSIST_ONLY`.

## 3.1 Comparison

| | Edit my.cnf | SET GLOBAL | SET PERSIST |
|---|---|---|---|
| Takes effect | After restart | **Immediately** | **Immediately** |
| Survives restart | Yes | **No** | Yes |
| Restart needed | **Yes** | No | No |
| Works on non-dynamic variables | Yes | No | No (use `PERSIST_ONLY`) |
| Where it is stored | `/etc/my.cnf` | memory only | `mysqld-auto.cnf` in datadir |

## 3.2 The picture

```
        my.cnf  (file)
            |
            | read once at startup
            v
   +---------------------+
   |  RUNNING mysqld     |  <---- SET GLOBAL (changes memory only)
   |  values in memory   |
   +---------------------+
            |
            | SET PERSIST also writes this
            v
     mysqld-auto.cnf  (read at next startup, overrides my.cnf)
```

## 3.3 A gotcha that surprises everyone

`mysqld-auto.cnf` is read **after** `my.cnf`, so it overrides it. If you `SET PERSIST` a value and later edit the **same** setting in `my.cnf`, MySQL will keep using the persisted value and your file edit seems ignored.

Find where a value comes from:
```sql
SELECT VARIABLE_NAME, VARIABLE_SOURCE, VARIABLE_PATH, SET_TIME
FROM performance_schema.variables_info
WHERE VARIABLE_NAME = 'max_connections';
```
`VARIABLE_SOURCE` shows `COMPILED`, `GLOBAL`/`SERVER` (from my.cnf), `PERSISTED`, `DYNAMIC` (changed with SET GLOBAL), etc.

List everything persisted:
```sql
SELECT * FROM performance_schema.persisted_variables;
```

> **Tip:** pick **one** habit and stick to it. Recommended for a team: make changes live with `SET GLOBAL`/`SET PERSIST`, and **also** record the same value in `my.cnf` so the file always documents the intended configuration. Avoid having different values for the same setting in the two places.

---

# 4. Changing Settings in Production Without a Restart

## 4.1 Step-by-step procedure

```
 1. BASELINE     record the current value and key metrics
 2. DYNAMIC?     check the variable can change without restart
 3. CHANGE       SET GLOBAL (or SET PERSIST) during a quiet period
 4. VERIFY       confirm the new value and watch metrics
 5. PERSIST      write it into my.cnf (or use SET PERSIST)
 6. DOCUMENT     write down old value, new value, reason, time
 7. ROLLBACK     if worse, set the old value back immediately
```

## 4.2 Worked example: raise max_connections

**Step 1: Baseline**
```sql
SELECT @@global.max_connections;
SHOW GLOBAL STATUS LIKE 'Max_used_connections';
SHOW GLOBAL STATUS LIKE 'Connection_errors_max_connections';
```
Say the current value is `151` and `Max_used_connections` reached `151`, with errors counting up. That means clients are being refused ("Too many connections").

**Step 2 and 3: Change it**
```sql
SET GLOBAL max_connections = 400;
```
If the variable is not dynamic you will see: `ERROR 1238: Variable 'x' is a read only variable`. That tells you a restart is needed (Section 4.4).

**Step 4: Verify**
```sql
SELECT @@global.max_connections;      -- 400
SHOW GLOBAL STATUS LIKE 'Threads_connected';
```

**Step 5: Make it permanent**

Either:
```sql
SET PERSIST max_connections = 400;
```
or edit the file so it matches:
```bash
sudo vim /etc/my.cnf
```
```ini
[mysqld]
max_connections = 400
```
(No restart. The file edit is only for the **next** time MySQL starts.)

**Step 7: Rollback if needed**
```sql
SET GLOBAL max_connections = 151;
```

## 4.3 Worked example: resize the InnoDB buffer pool online

Since MySQL 5.7 the buffer pool can be resized while running.

```sql
-- Current size (bytes and GB)
SELECT @@innodb_buffer_pool_size/1024/1024/1024 AS gb;

-- Set to 8 GB
SET GLOBAL innodb_buffer_pool_size = 8 * 1024 * 1024 * 1024;

-- Watch the resize progress
SHOW STATUS LIKE 'Innodb_buffer_pool_resize_status';
```

Things to know:
- The server rounds the size to a multiple of `innodb_buffer_pool_chunk_size × innodb_buffer_pool_instances`. The final value may differ slightly from what you typed.
- The resize runs in the background in chunks. Do it in a quiet period; performance can dip a little while it happens.
- Do not set it larger than the RAM you can spare (Section 7), or the server will swap.

## 4.4 What if the variable is NOT dynamic?

Examples: `innodb_flush_method`, `innodb_buffer_pool_instances`, `skip_name_resolve`, `log_bin`, `performance_schema`, `datadir`.

You still have options, in order of preference:

**Option A: Stage it and wait for a planned restart**
```sql
SET PERSIST_ONLY innodb_flush_method = 'O_DIRECT';
```
This writes the value to `mysqld-auto.cnf` without touching the running server. It takes effect at the next restart.

**Option B: Rolling restart with replicas (the real production method)**

```
   Primary (writes)  --->  Replica 1
                     --->  Replica 2

 1. Change my.cnf on Replica 1, restart it, check it is healthy and caught up
 2. Same for Replica 2
 3. Switch over: promote a replica to be the new primary (short, planned)
 4. Change my.cnf on the OLD primary, restart it, rejoin as a replica
```
Users see at most a brief switch-over, not a long outage. This needs replication or a cluster set up in advance (InnoDB Cluster, Group Replication, or a proxy like ProxySQL).

**Option C: A scheduled maintenance window**
Announce it, run `mysqld --validate-config`, restart, verify.

## 4.5 Production safety checklist

- [ ] Checked that the variable is dynamic (or planned a restart).
- [ ] Recorded the **old value** so I can roll back.
- [ ] Change is small (never jump a value 10x in one step).
- [ ] Done in a **quiet** period.
- [ ] Watched `SHOW PROCESSLIST` and error log for 10-15 minutes afterwards.
- [ ] Persisted (`SET PERSIST` or `my.cnf`), documented.
- [ ] Ran `mysqld --validate-config` after editing `my.cnf`.

## 4.6 Things you can do online without downtime (summary)

| Task | How |
|---|---|
| Change most memory/connection/log settings | `SET GLOBAL` / `SET PERSIST` |
| Resize buffer pool | `SET GLOBAL innodb_buffer_pool_size` |
| Resize redo log (8.0.30+) | `SET GLOBAL innodb_redo_log_capacity` |
| Turn the slow log on/off | `SET GLOBAL slow_query_log` |
| Add an index | `ALTER TABLE ... ADD INDEX ..., ALGORITHM=INPLACE, LOCK=NONE` |
| Kill a runaway query | `KILL QUERY <id>` |
| Rotate/purge binary logs | `PURGE BINARY LOGS ...` |

---

# 5. Which Settings Need a Restart and Which Don't

**Dynamic** = can change with `SET GLOBAL` (no restart).
**Static** = read-only at runtime; needs a restart (stage with `SET PERSIST_ONLY` or `my.cnf`).

> Always check your version's manual for the variable's "Dynamic" flag. A quick practical test: try `SET GLOBAL`. A read-only variable returns error 1238.

| Setting | Dynamic? | What it controls |
|---|---|---|
| `innodb_buffer_pool_size` | **Yes** | Main memory cache for data and indexes |
| `innodb_buffer_pool_instances` | No | Splits the pool to reduce contention |
| `innodb_redo_log_capacity` | **Yes** (8.0.30+) | Size of the redo (write-ahead) log |
| `innodb_log_file_size` | No (older than 8.0.30) | Redo log file size |
| `innodb_log_buffer_size` | Yes | Memory buffer for redo log writes |
| `innodb_flush_log_at_trx_commit` | **Yes** | Durability vs speed |
| `innodb_flush_method` | No | How InnoDB writes to disk (`O_DIRECT`) |
| `innodb_io_capacity` / `_max` | **Yes** | How hard InnoDB may push the disk |
| `innodb_file_per_table` | Yes | One file per table |
| `innodb_lock_wait_timeout` | Yes | How long to wait for a row lock |
| `max_connections` | **Yes** | Maximum simultaneous clients |
| `thread_cache_size` | Yes | Reuse threads for new connections |
| `table_open_cache` | Yes | Number of cached open tables |
| `tmp_table_size`, `max_heap_table_size` | Yes | In-memory temp table limit |
| `sort_buffer_size`, `join_buffer_size` | Yes (new sessions only) | Per-connection work memory |
| `max_allowed_packet` | Yes (new sessions) | Largest packet/row/blob |
| `wait_timeout`, `interactive_timeout` | Yes (new sessions) | Idle connection timeout |
| `slow_query_log`, `long_query_time` | **Yes** | Slow query logging |
| `general_log` | Yes | Logs every query (heavy, use briefly) |
| `sync_binlog` | Yes | Binary log durability |
| `binlog_expire_logs_seconds` | Yes | How long binlogs are kept |
| `log_bin` | No | Turns binary logging on/off |
| `read_only`, `super_read_only` | Yes | Make server read-only |
| `sql_mode` | Yes (new sessions) | SQL strictness |
| `character_set_server` | Yes | Default for **new** databases |
| `skip_name_resolve` | No | Skip DNS lookups for clients |
| `performance_schema` | No | Turns the performance schema on/off |
| `bind_address`, `port` | No | Where MySQL listens |
| `datadir` | No | Where data is stored |
| `open_files_limit` | No | Open file limit (also needs a systemd limit) |

## 5.1 Gotcha: "new sessions only"

Changing `sort_buffer_size` with `SET GLOBAL` does **not** change connections that are already open. They keep the old value. Only connections made **after** the change get the new one. (This also means a connection pool may keep old values for a long time.)

---

# 6. Step 0 of Tuning: Know Your Server

## 6.1 Hardware and version
```bash
nproc                      # CPU cores
free -h                    # RAM
lsblk -d -o name,rota      # ROTA 0 = SSD, 1 = spinning disk
df -h /var/lib/mysql
```
```sql
SELECT VERSION();
SELECT @@datadir, @@innodb_buffer_pool_size, @@max_connections;
```

## 6.2 Uptime matters
Most status counters count **since the last restart**. A server up for 10 minutes gives misleading numbers.
```sql
SHOW GLOBAL STATUS LIKE 'Uptime';     -- seconds
```
Wait until the server has run for a few days under normal load before judging counters. Better still, take two snapshots a few minutes apart and compare the difference.

## 6.3 Find the biggest databases and tables
```sql
SELECT table_schema AS db,
       ROUND(SUM(data_length + index_length)/1024/1024/1024, 2) AS size_gb
FROM information_schema.tables
GROUP BY table_schema
ORDER BY size_gb DESC;
```
Compare the total with your RAM. If your **working data** fits in the buffer pool, reads are mostly from memory and very fast.

## 6.4 Save a baseline
```sql
SHOW GLOBAL STATUS;        -- copy the output to a file with the date
SHOW GLOBAL VARIABLES;
```
```bash
mysql -e "SHOW GLOBAL STATUS" > /root/status_$(date +%F_%H%M).txt
mysql -e "SHOW GLOBAL VARIABLES" > /root/vars_$(date +%F_%H%M).txt
```
You can later compare two files with `diff`.

---

# 7. Memory Tuning

## 7.1 InnoDB buffer pool (the most important setting)

The buffer pool is MySQL's RAM cache for table data and indexes. Bigger cache means fewer slow disk reads.

```
 Query --> [ Buffer Pool (RAM) ] --hit--> fast answer
                  |
                miss
                  v
            [ Disk ]  --> slow
```

**Sizing rule of thumb:**

| Server type | Buffer pool size |
|---|---|
| Dedicated database server | **about 60-75% of RAM** |
| Shared with app / web server | about 30-50% of RAM |

| RAM | Dedicated starting point |
|---|---|
| 4 GB | 2 GB |
| 8 GB | 5 GB |
| 16 GB | 11 GB |
| 32 GB | 24 GB |
| 64 GB | 48 GB |

Leave room for the OS, per-connection memory, the page cache, and other programs. If the server starts swapping, the pool is too big.

**Check if it is big enough:**
```sql
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_read%';
```
- `Innodb_buffer_pool_read_requests` = reads served from memory.
- `Innodb_buffer_pool_reads` = reads that had to go to disk.

```
 hit ratio = 1 - ( Innodb_buffer_pool_reads / Innodb_buffer_pool_read_requests )
```
Aim for **99% or more** on a busy server. A lower ratio suggests the pool is too small (or queries scan too much data).

Also check:
```sql
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_wait_free';   -- should be 0
SHOW ENGINE INNODB STATUS\G                               -- look at BUFFER POOL AND MEMORY
```

**Shortcut for dedicated servers (MySQL 8.0.3+):** `innodb_dedicated_server = ON` in `my.cnf` makes MySQL pick the buffer pool, redo log and flush method automatically from the machine's RAM. Good for beginners on a server used only for MySQL. It needs a restart to enable, and you should not also set those values by hand.

**Warm cache after restart:** the buffer pool is empty after a restart, so the server is slow at first. MySQL 8.0 saves and reloads the hot page list by default:
```ini
innodb_buffer_pool_dump_at_shutdown = ON
innodb_buffer_pool_load_at_startup  = ON
```

## 7.2 Per-connection memory

Each connection can use extra memory for sorting and joining. Total worst-case memory is roughly:

```
 total = global memory (buffer pool + log buffer + other caches)
       + max_connections x per-connection memory
```

Per-connection settings (allocated only when a query needs them):

| Setting | Typical default | Advice |
|---|---|---|
| `sort_buffer_size` | 256 KB | Leave at default. Do not set to many MB |
| `join_buffer_size` | 256 KB | Leave at default |
| `read_buffer_size` | 128 KB | Leave at default |
| `read_rnd_buffer_size` | 256 KB | Leave at default |

> **Common beginner mistake:** setting `sort_buffer_size = 64M` "for speed". With 500 connections that can mean tens of GB, and it usually makes things slower. Fix the query or add an index instead.

## 7.3 Redo log

The redo log is a write-ahead log that makes commits fast and crash-safe. Too small a log forces frequent flushing and slows writes.

MySQL 8.0.30 and later:
```sql
SELECT @@innodb_redo_log_capacity/1024/1024 AS mb;     -- default is about 100 MB
SET PERSIST innodb_redo_log_capacity = 4 * 1024 * 1024 * 1024;   -- 4 GB, online
```

Older versions use `innodb_log_file_size` (needs a restart).

**How big?** Aim for about **one hour** of write traffic. Measure it:
```sql
SHOW GLOBAL STATUS LIKE 'Innodb_os_log_written';   -- note value A
-- wait 60 seconds
SHOW GLOBAL STATUS LIKE 'Innodb_os_log_written';   -- note value B
```
```
 bytes per hour = (B - A) x 60
```
Round up to a sensible size (for example 2-8 GB on a write-heavy server).

---

# 8. InnoDB I/O and Durability

## 8.1 The durability vs speed trade-off

`innodb_flush_log_at_trx_commit` decides what happens when you COMMIT:

| Value | What happens | Crash safety | Speed |
|---|---|---|---|
| **1** (default) | Write **and** flush to disk at every commit | Safest, no data loss | Slowest |
| 2 | Write at commit, flush about once per second | Can lose ~1 second if the **OS or power** crashes | Faster |
| 0 | Write and flush about once per second | Can lose ~1 second even if only **MySQL** crashes | Fastest |

Companion setting for the binary log:

| `sync_binlog` | Meaning |
|---|---|
| **1** (default) | Flush binlog at every commit (safest) |
| 0 | Let the OS decide (fast, risk of losing events on crash) |

**Production advice:** for money, orders, anything important, keep `innodb_flush_log_at_trx_commit = 1` and `sync_binlog = 1`. Only relax them for data you can afford to lose or rebuild (and with a clear decision from the business).

```sql
SET GLOBAL innodb_flush_log_at_trx_commit = 2;   -- dynamic, no restart
```

## 8.2 Other I/O settings

| Setting | Advice |
|---|---|
| `innodb_flush_method = O_DIRECT` | Avoids double caching (MySQL cache plus OS cache). Recommended on Linux. Needs restart |
| `innodb_io_capacity` | Default 200 suits old spinning disks. For SSD use about 2000 (check your disk). Dynamic |
| `innodb_io_capacity_max` | About 2x `innodb_io_capacity`. Dynamic |
| `innodb_flush_neighbors = 0` | For SSD (no need to flush neighbouring pages) |
| `innodb_file_per_table = ON` | Default. Each table in its own file, easier to manage |

```sql
SET PERSIST innodb_io_capacity     = 2000;
SET PERSIST innodb_io_capacity_max = 4000;
```

---

# 9. Connections and Threads

## 9.1 max_connections

Default is 151. Check how close you get:
```sql
SELECT @@max_connections;
SHOW GLOBAL STATUS LIKE 'Max_used_connections';
SHOW GLOBAL STATUS LIKE 'Threads_connected';
SHOW GLOBAL STATUS LIKE 'Connection_errors_max_connections';
```

Rules of thumb:
- `Max_used_connections` should stay **below about 80%** of `max_connections`.
- If `Connection_errors_max_connections` keeps rising, clients are being refused.
- Raising `max_connections` costs memory (Section 7.2) and does not fix slow queries. Many connections piling up usually means queries are slow, or the application is not using a **connection pool**.
- MySQL keeps one extra connection reserved for an administrator so you can still log in when the limit is hit.

## 9.2 Idle connections

```sql
SELECT @@wait_timeout;          -- default 28800 (8 hours)
```
Lower it (e.g. `600`) if applications leave connections open and idle. Existing sessions keep their old value (Section 5.1).

## 9.3 thread_cache_size

Creating a thread per connection is costly. The cache reuses them.
```sql
SHOW GLOBAL STATUS LIKE 'Threads_created';
SHOW GLOBAL STATUS LIKE 'Connections';
```
```
 miss rate = Threads_created / Connections     (aim below about 1-2%)
```
If it is high, raise `thread_cache_size` (e.g. `16`-`64`).

## 9.4 Skip DNS lookups

```ini
skip_name_resolve = ON
```
Stops MySQL from doing reverse DNS on every connect. This avoids slow logins when DNS is slow. **Needs a restart.** After enabling it, grants must use IP addresses, not hostnames.

---

# 10. Temp Tables and Table Cache

## 10.1 Temp tables

Queries with `GROUP BY`, `ORDER BY`, `DISTINCT` or subqueries may build temp tables, in memory or, if too big, on disk (slow).
```sql
SHOW GLOBAL STATUS LIKE 'Created_tmp%tables';
```
```
 disk ratio = Created_tmp_disk_tables / Created_tmp_tables    (aim below about 10-25%)
```
Settings (dynamic):
```sql
SET PERSIST tmp_table_size      = 64 * 1024 * 1024;
SET PERSIST max_heap_table_size = 64 * 1024 * 1024;
```
Keep these two equal. The smaller of the two is the effective limit. Also, queries that select `TEXT`/`BLOB` columns always go to disk, so select only the columns you need.

## 10.2 Table cache

```sql
SHOW GLOBAL STATUS LIKE 'Opened_tables';
SHOW GLOBAL STATUS LIKE 'Table_open_cache_misses';
SELECT @@table_open_cache;
```
If `Opened_tables` rises quickly during normal operation, raise `table_open_cache` (default 4000 in 8.0) and, if needed, `table_definition_cache`. Each cached table uses a file handle, so `open_files_limit` must be large enough (see Section 17).

---

# 11. Logging

## 11.1 The logs at a glance

| Log | Purpose | Default |
|---|---|---|
| **Error log** | Startup, crashes, warnings | On (`/var/log/mysqld.log` on RHEL) |
| **Slow query log** | Queries slower than a limit | **Off** |
| **General log** | Every single query | Off (very heavy) |
| **Binary log** | Changes for replication and point-in-time recovery | On in 8.0 |

## 11.2 Slow query log (turn it on in production)

This is the best tuning tool. Enable it live:
```sql
SET PERSIST slow_query_log  = ON;
SET PERSIST long_query_time = 1;                -- seconds; use 0.5 or lower for busy OLTP
SET PERSIST log_slow_extra  = ON;               -- extra detail (8.0.14+)
```
Optional extras:
```sql
SET PERSIST min_examined_row_limit = 1000;      -- ignore trivial queries
```
- `log_queries_not_using_indexes = ON` is **noisy**; enable it only briefly.

Find the file:
```sql
SELECT @@slow_query_log_file;
```
By default it is `<datadir>/<hostname>-slow.log`. A custom path must already exist, be owned by `mysql`, and (RHEL) have the right SELinux label.

Read it:
```bash
sudo mysqldumpslow -s t -t 10 /var/lib/mysql/HOSTNAME-slow.log     # top 10 by total time
```
Percona's `pt-query-digest` gives richer reports.

## 11.3 General log: only briefly

```sql
SET GLOBAL general_log = ON;     -- for a minute or two only
SET GLOBAL general_log = OFF;
```
It records every statement and can fill the disk and slow the server.

## 11.4 Binary log

```sql
SHOW BINARY LOGS;
SELECT @@binlog_expire_logs_seconds;         -- default 2592000 = 30 days
SET PERSIST binlog_expire_logs_seconds = 604800;   -- keep 7 days
```
Binary logs can eat the disk. Set a sensible retention, and never delete the files by hand. Use:
```sql
PURGE BINARY LOGS BEFORE NOW() - INTERVAL 7 DAY;
```
Keep enough logs to cover your oldest backup, so you can do point-in-time recovery.

## 11.5 Error log level

```sql
SET PERSIST log_error_verbosity = 2;     -- 1 errors, 2 + warnings, 3 + notes
```

---

# 12. Query Tuning (The Biggest Win)

## 12.1 Find the problem queries

**Live view:**
```sql
SHOW FULL PROCESSLIST;
SELECT * FROM sys.processlist WHERE command <> 'Sleep' ORDER BY time DESC LIMIT 10;
```
Kill a runaway query (without dropping the connection):
```sql
KILL QUERY 12345;       -- id from the processlist
```

**Worst queries overall (needs performance_schema, on by default):**
```sql
SELECT * FROM sys.statement_analysis LIMIT 10;
```
Read columns like `total_latency`, `exec_count`, `rows_examined`, `no_index_used_count`.

**Unused and redundant indexes:**
```sql
SELECT * FROM sys.schema_unused_indexes;
SELECT * FROM sys.schema_redundant_indexes;
```
(Unused-index data covers only the time since the last restart, so check after a full business cycle.)

**Who is blocking whom:**
```sql
SELECT * FROM sys.innodb_lock_waits\G
```

## 12.2 Read the execution plan with EXPLAIN

```sql
EXPLAIN SELECT * FROM orders WHERE customer_id = 42 ORDER BY created_at DESC LIMIT 20;
EXPLAIN ANALYZE SELECT ...;      -- 8.0.18+, actually runs it and shows real timings
```

Key columns:

| Column | Good | Bad |
|---|---|---|
| `type` | `const`, `eq_ref`, `ref`, `range` | **`ALL`** (full table scan) |
| `key` | An index name | `NULL` (no index used) |
| `rows` | Small | Huge |
| `Extra` | `Using index` | `Using filesort`, `Using temporary` |

## 12.3 Indexes (how they help)

```
 No index:    scan ALL 10,000,000 rows to find customer 42
 With index:  jump straight to the rows for customer 42
```

```sql
-- Add an index online (no blocking of reads/writes)
ALTER TABLE orders
  ADD INDEX idx_customer_created (customer_id, created_at),
  ALGORITHM=INPLACE, LOCK=NONE;
```

Index rules:
- **Composite index order matters:** `(customer_id, created_at)` helps queries that filter on `customer_id` (and also on `created_at` after it). It does not help a query that filters only on `created_at`. Put equality-filtered columns first.
- A **covering index** (contains every column the query needs) is the fastest, shown as `Using index`.
- Do not index everything. Each index slows down `INSERT`/`UPDATE` and uses space.
- Prefer small columns for indexes (INT over long VARCHAR).

## 12.4 Common mistakes that kill indexes

| Bad | Why | Better |
|---|---|---|
| `WHERE YEAR(created_at) = 2026` | Function on the column | `WHERE created_at >= '2026-01-01' AND created_at < '2027-01-01'` |
| `WHERE name LIKE '%smith'` | Leading wildcard | `LIKE 'smith%'` or a full-text index |
| `WHERE phone = 5551234` (phone is VARCHAR) | Type conversion | `WHERE phone = '5551234'` |
| `SELECT *` | Reads and ships unneeded columns | Select only needed columns |
| `LIMIT 100000, 20` | Deep pagination scans 100,020 rows | "Keyset" paging: `WHERE id > last_id ORDER BY id LIMIT 20` |
| Many `OR` conditions on different columns | May skip indexes | Use `UNION` or rethink the index |
| N+1 queries from the app | Thousands of tiny queries | Join or batch them |

## 12.5 Online schema changes (no downtime)

- Many `ALTER TABLE` operations in MySQL 8 are online. Always state the intent so MySQL refuses if it cannot comply:
```sql
ALTER TABLE t ADD COLUMN notes VARCHAR(255), ALGORITHM=INSTANT;   -- very fast for many cases
ALTER TABLE t ADD INDEX idx_x (x), ALGORITHM=INPLACE, LOCK=NONE;
```
If MySQL replies "ALGORITHM=... is not supported", the change would block or copy the table.
- For big, risky changes use **`pt-online-schema-change`** (Percona Toolkit) or **`gh-ost`**, which copy the table in the background and swap at the end.
- Run big changes in a quiet period and watch replication lag.

## 12.6 Transactions

- Keep transactions **short**. Long ones hold locks, bloat undo logs, and cause lock waits.
- Check for old transactions:
```sql
SELECT trx_id, trx_started, trx_mysql_thread_id, trx_query
FROM information_schema.innodb_trx
ORDER BY trx_started;
```
- `innodb_lock_wait_timeout` (default 50 s) controls how long a statement waits for a row lock.
- To log all deadlocks: `SET PERSIST innodb_print_all_deadlocks = ON;`

---

# 13. Sample Annotated `my.cnf`

For a **dedicated 16 GB RAM, SSD** MySQL 8 server. Adjust to your hardware. Each line says what it is and whether it is dynamic (D) or needs restart (R).

```ini
[mysqld]
# ---------- Basics ----------
datadir                 = /var/lib/mysql          # R  where data lives
bind_address            = 0.0.0.0                 # R  listen on all IPs (restrict with firewall!)
skip_name_resolve       = ON                      # R  no DNS lookups for clients
character_set_server    = utf8mb4                 # D  default charset for NEW databases
collation_server        = utf8mb4_0900_ai_ci      # D

# ---------- Connections ----------
max_connections         = 300                     # D  max clients (keep Max_used < 80%)
thread_cache_size       = 32                      # D  reuse threads
wait_timeout            = 600                     # D  close idle connections after 10 min (new sessions)
max_allowed_packet      = 64M                     # D  largest query/row/blob (new sessions)

# ---------- InnoDB memory ----------
innodb_buffer_pool_size = 11G                     # D  ~70% of RAM: the main cache
innodb_buffer_pool_instances = 8                  # R  split pool to reduce contention
innodb_buffer_pool_dump_at_shutdown = ON          # D  remember hot pages
innodb_buffer_pool_load_at_startup  = ON          # R  reload them at start
innodb_log_buffer_size  = 64M                     # D  redo log memory buffer

# ---------- InnoDB I/O and durability ----------
innodb_redo_log_capacity = 4G                     # D  (8.0.30+) redo log size
innodb_flush_log_at_trx_commit = 1                # D  1 = safest (no data loss)
sync_binlog             = 1                       # D  1 = safest
innodb_flush_method     = O_DIRECT                # R  avoid double caching
innodb_io_capacity      = 2000                    # D  SSD
innodb_io_capacity_max  = 4000                    # D
innodb_flush_neighbors  = 0                       # D  SSD
innodb_file_per_table   = ON                      # D

# ---------- Per-connection and temp memory (keep modest) ----------
sort_buffer_size        = 512K                    # D  leave small
join_buffer_size        = 512K                    # D
tmp_table_size          = 64M                     # D
max_heap_table_size     = 64M                     # D  keep equal to tmp_table_size

# ---------- Caches ----------
table_open_cache        = 4000                    # D
table_definition_cache  = 2000                    # D
open_files_limit        = 65535                   # R  (also set LimitNOFILE in systemd)

# ---------- Logging ----------
log_error               = /var/log/mysqld.log     # R
log_error_verbosity     = 2                       # D
slow_query_log          = ON                      # D
long_query_time         = 1                       # D
log_slow_extra          = ON                      # D
log_bin                 = binlog                  # R  turn binary logging on (default on in 8.0)
binlog_expire_logs_seconds = 604800               # D  keep 7 days
server_id               = 1                       # D  unique per server (needed for replication)

# ---------- Safety ----------
sql_mode                = ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION   # D
innodb_lock_wait_timeout = 30                     # D
```

**Scaling it to other machines:**

| RAM | `innodb_buffer_pool_size` | `max_connections` | `innodb_redo_log_capacity` |
|---|---|---|---|
| 4 GB | 2G | 100-150 | 512M-1G |
| 8 GB | 5G | 200 | 1-2G |
| 16 GB | 11G | 300 | 2-4G |
| 32 GB | 24G | 400-600 | 4-8G |
| 64 GB | 48G | 600-1000 | 8-16G |

After editing the file:
```bash
mysqld --validate-config          # must print nothing
```
Then it will apply at the next restart. For a live server, apply the dynamic ones with `SET PERSIST` first (Section 4).

---

# 14. If Your Data Directory Is on NFS

This applies to your lab, where `/var/lib/mysql_data` is an NFS mount. It works for learning, but remember:

```
 MySQL  ---- every write waits for network + remote disk ---->  NFS server
```

| Topic | Advice |
|---|---|
| Speed | Every commit (with `innodb_flush_log_at_trx_commit = 1`) waits for the network and the remote disk. Expect slower writes |
| Mount options | Use `hard` (never `soft`, which can corrupt data on a timeout). Example: `nfs4 rw,hard,noatime,_netdev` |
| Reads | A bigger `innodb_buffer_pool_size` helps a lot, because reads then avoid the network |
| Network outage | MySQL stalls (hard mount) until the network returns. A flaky network is dangerous for a database |
| `mysqld-auto.cnf` | `SET PERSIST` writes this file **inside the datadir**, so it lives on the NFS share |
| Safety | Keep regular `mysqldump` backups. Do not put a production database on NFS unless the storage is designed for it |

Check your mount:
```bash
mount | grep mysql_data
nfsstat -m
```

---

# 15. Monitoring Cheat Sheet

## 15.1 Quick health check

```sql
SHOW GLOBAL STATUS LIKE 'Uptime';
SHOW GLOBAL STATUS LIKE 'Threads_connected';
SHOW GLOBAL STATUS LIKE 'Threads_running';          -- queries running right now
SHOW GLOBAL STATUS LIKE 'Slow_queries';
SHOW GLOBAL STATUS LIKE 'Questions';                -- total queries
SHOW GLOBAL STATUS LIKE 'Aborted_connects';
SHOW FULL PROCESSLIST;
SHOW ENGINE INNODB STATUS\G
```

## 15.2 Metrics and what is healthy

| Metric | How to compute | Healthy |
|---|---|---|
| Buffer pool hit ratio | `1 - reads / read_requests` | 99% or more |
| Connection usage | `Max_used_connections / max_connections` | under 80% |
| Thread cache misses | `Threads_created / Connections` | under 1-2% |
| Disk temp tables | `Created_tmp_disk_tables / Created_tmp_tables` | under 10-25% |
| Slow queries | `Slow_queries` rate | low and stable |
| `Threads_running` | live | below the number of CPU cores, most of the time |
| `Innodb_buffer_pool_wait_free` | live | 0 |
| Replication lag | `SHOW REPLICA STATUS` (`Seconds_Behind_Source`) | near 0 |
| Disk space | `df -h` | never above about 85% |

## 15.3 Compare two moments (rate of change)

Counters only grow. To get a rate, sample twice:
```bash
mysqladmin -u root -p -r -i 10 extended-status | grep -E "Questions|Slow_queries|Threads_running"
```
`-r` shows differences and `-i 10` samples every 10 seconds.

## 15.4 Tools worth knowing

| Tool | Use |
|---|---|
| `sys` schema | Friendly views over performance_schema (built in) |
| `mysqldumpslow` | Summarise the slow log (built in) |
| Percona Toolkit (`pt-query-digest`, `pt-online-schema-change`) | Deep slow-log analysis, online ALTERs |
| Percona Monitoring and Management (PMM), Prometheus with `mysqld_exporter` + Grafana | Dashboards and alerts |
| `mysqltuner.pl` | Third-party script that suggests settings. Treat its advice as hints, not rules |

---

# 16. Security and Backups (Short)

## Security basics
- Use a **separate user per application** with only the privileges it needs, not `root`.
```sql
CREATE USER 'appuser'@'10.10.10.%' IDENTIFIED BY 'StrongPassword!';
GRANT SELECT, INSERT, UPDATE, DELETE ON shop.* TO 'appuser'@'10.10.10.%';
```
- Restrict `bind_address` or use the firewall so only the app servers reach port 3306.
- Run `mysql_secure_installation` on new installs.
- Require encrypted connections where possible: `require_secure_transport = ON`.
- Never expose port 3306 to the internet.

## Backups
```bash
# Consistent logical backup without blocking InnoDB tables
mysqldump -u root -p --single-transaction --routines --events --triggers --all-databases > /backup/all_$(date +%F).sql
```
- A backup you have not **tested restoring** is not a backup.
- For large databases use a physical hot backup tool (Percona XtraBackup or MySQL Enterprise Backup).
- Keep binary logs for **point-in-time recovery** between backups.
- Store backups on a different machine from the database.

---

# 17. Troubleshooting

| Problem | Likely cause and fix |
|---|---|
| I edited `my.cnf` but nothing changed | MySQL reads it only at startup. Apply with `SET GLOBAL`/`SET PERSIST` now, restart later |
| Edited `my.cnf`, restarted, value is still the old one | A `SET PERSIST` value in `mysqld-auto.cnf` overrides it. Check `performance_schema.variables_info`, then `RESET PERSIST variable_name;` |
| `ERROR 1238: Variable 'x' is a read only variable` | Static variable. Use `SET PERSIST_ONLY` and restart in a planned window |
| `ERROR 1227: Access denied; you need the SUPER or SYSTEM_VARIABLES_ADMIN privilege` | Your user lacks the privilege. Use an admin account |
| `Too many connections` | `max_connections` reached. Find the cause (slow queries, no pool). Raise it carefully with `SET GLOBAL` |
| MySQL will not start after editing `my.cnf` | Typo or invalid value. `sudo journalctl -u mysqld -n 50` and `mysqld --validate-config` |
| MySQL will not start after `SET PERSIST_ONLY` | A bad value in `mysqld-auto.cnf`. Remove it (`RESET PERSIST`) before restart, or edit/move that file while the server is down |
| Buffer pool resize seems stuck | Check `Innodb_buffer_pool_resize_status`. It runs in the background; avoid peak hours |
| `Too many open files` | Raise `open_files_limit` **and** the systemd limit: `sudo systemctl edit mysqld` then add `[Service]` `LimitNOFILE=65535` (applies at next restart) |
| Server very slow after restart | Cold buffer pool. Keep the dump/load options on (Section 7.1) |
| Changed `sort_buffer_size` but no effect | Only **new** connections get new values (Section 5.1) |
| Slow log file does not appear | Path must exist, be owned by `mysql`, and have the right SELinux label (`restorecon -v`) |
| Swapping / out of memory | Buffer pool plus connection memory too big. Lower `innodb_buffer_pool_size` live, reduce `max_connections` |
| Disk filling up | Binary logs (`binlog_expire_logs_seconds`), slow/general log left on, or large temp files |
| `SET PERSIST` seems to do nothing | Variable is not dynamic, or you are on MariaDB (not supported, Section 18) |

Useful checks:
```bash
sudo journalctl -u mysqld -n 50
sudo tail -n 50 /var/log/mysqld.log
mysqld --validate-config
getenforce
```

---

# 18. MariaDB Differences

If your server is **MariaDB** (check `SELECT VERSION();`):

| Topic | MySQL 8 | MariaDB |
|---|---|---|
| `SET PERSIST` / `PERSIST_ONLY` | Supported | **Not supported**. Use `SET GLOBAL`, then edit the config file by hand |
| Config files | `/etc/my.cnf`, `/etc/my.cnf.d/` | Same on RHEL (`/etc/my.cnf.d/server.cnf`) |
| Service name | `mysqld` | `mariadb` |
| Redo log size online | `innodb_redo_log_capacity` (8.0.30+) | `innodb_log_file_size` is dynamic only in recent versions (10.9+) |
| `sys` schema | Included | Included in newer versions |
| `mysqld --validate-config` | Available | Use `mariadbd --help --verbose` or check the log |

The principle is identical: `SET GLOBAL` for live changes, and keep the config file in sync.

---

# 19. Cheat Sheet

```
 SEE a value            SELECT @@global.max_connections;
                        SHOW GLOBAL VARIABLES LIKE 'max_conn%';
 WHERE did it come from SELECT * FROM performance_schema.variables_info WHERE VARIABLE_NAME='x';
 CHANGE live (temp)     SET GLOBAL x = v;
 CHANGE live (kept)     SET PERSIST x = v;
 STAGE for restart      SET PERSIST_ONLY x = v;
 UNDO persisted         RESET PERSIST x;
 TEST one session       SET SESSION x = v;
 CHECK config file      mysqld --validate-config
 SHOW file settings     mysqld --print-defaults
```

## The production change recipe
```
 1. Record old value and baseline metrics
 2. Is it dynamic?  Try SET GLOBAL (error 1238 = needs restart)
 3. Change a little, in a quiet period
 4. Verify value and watch metrics + error log
 5. Persist: SET PERSIST  (or edit my.cnf to match)
 6. Document it
 7. Roll back if worse
```

## Top 10 things to do on a new MySQL server
1. Set `innodb_buffer_pool_size` to about 60-75% of RAM (dedicated server).
2. Keep `innodb_flush_log_at_trx_commit = 1` and `sync_binlog = 1` for important data.
3. Set `innodb_flush_method = O_DIRECT`.
4. Size `innodb_redo_log_capacity` for about an hour of writes.
5. Set `innodb_io_capacity` to match your disk (SSD: about 2000).
6. Turn on the **slow query log** (`long_query_time = 1`).
7. Keep `max_connections` realistic and use a connection pool in the app.
8. Add indexes for slow queries (check `EXPLAIN`).
9. Set binary log retention and monitor disk space.
10. Take backups and **test restoring them**.

## Practice ideas (on a test server)
1. Run `SET GLOBAL max_connections = 50;` then open more than 50 connections and see the error. Set it back.
2. Turn on the slow log with `long_query_time = 0`, run some queries, then read the log with `mysqldumpslow`.
3. Create a table with 1 million rows, run a query with and without an index, and compare `EXPLAIN`.
4. Use `SET PERSIST`, restart MySQL, and confirm the value stayed. Then `RESET PERSIST`.
5. Try `SET GLOBAL innodb_flush_method = 'O_DIRECT';` and read the error, then use `SET PERSIST_ONLY`.
6. Resize the buffer pool online and watch `Innodb_buffer_pool_resize_status`.
