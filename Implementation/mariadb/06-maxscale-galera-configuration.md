# MaxScale with a Galera Cluster

A complete record of how MaxScale was reconfigured to sit in front of a 3 node
Galera cluster. This document builds on the primary/replica setup described in
`02-maxscale-single-node.md` and focuses on what changes, why each line
exists, and what breaks if the old configuration is kept.

---

## 1. Environment

| Role | Hostname | IP | Notes |
|---|---|---|---|
| MaxScale | maxscale | 10.10.10.190 | Proxy, client entry point on port 4006 |
| Node 1 | mariadb-1 | 10.10.10.201 | Galera node |
| Node 2 | mariadb-2 | 10.10.10.202 | Galera node |
| Node 3 | mariadb-3 | 10.10.10.203 | Galera node |

- OS: AlmaLinux 10.2 (Lavender Lion)
- MaxScale: 24.02.9
- MariaDB: 10.11 series with Galera provider (`galera` package)
- Cluster state: 3 nodes, `wsrep_cluster_size = 3`, all `Synced`

Before this guide, MaxScale was configured for a **primary/replica** topology
(see `02-maxscale-single-node.md`). It used `mariadbmon` and
`readwritesplit`. Those settings no longer apply to a Galera cluster.

---

## 2. Why the old configuration cannot be reused

The previous MaxScale configuration assumed:

- **One primary** that accepts all writes
- **One or more replicas** that receive reads and follow the primary via
  asynchronous binary log replication
- **MaxScale decides** when to promote a replica (via `auto_failover`)

A Galera cluster is different:

- **Every node is a writer.** All nodes are in sync by design.
- **The cluster decides membership itself.** Nodes join and leave through
  quorum; there is no "promotion" to perform.
- **MaxScale should follow the cluster, not manage it.**

Mixing the two models causes incorrect role detection and routing. MaxScale
must use a Galera-aware monitor, and the primary/replica-only options must be
removed.

---

## 3. What changed, in one table

| Item | Old (primary/replica) | New (Galera) | Reason |
|---|---|---|---|
| Monitor module | `mariadbmon` | `galeramon` | Only `galeramon` understands Galera's multi-primary state |
| Monitor: `failcount` | `5` | removed | Galera decides node failure; `galeramon` has no failover logic |
| Monitor: `replication_user` | `repl` | removed | No `CHANGE MASTER TO` is performed |
| Monitor: `replication_password` | `"123"` | removed | Same |
| Monitor: `auto_failover` | `true` | removed | Galera handles failure internally; MaxScale must not attempt promotion |
| Monitor: `auto_rejoin` | `true` | removed | Same |
| Service: server list | `cluster=MariaDB-Monitor` | `servers=mariadb-1,mariadb-2,mariadb-3` | Explicit list is clearer and matches Galera reference configs |
| Service: `master_reconnection` | not set | `true` | Reconnect sessions to the new primary if one drops |
| Service: `transaction_replay` | not set | `true` | Replay interrupted transactions on the new primary |

---

## 4. Full `/etc/maxscale.cnf` for Galera

Replace the entire file with:

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

[Galera-Monitor]
type=monitor
module=galeramon
servers=mariadb-1,mariadb-2,mariadb-3
user=max
password="123"
monitor_interval=2000ms

[Read-Write-Service]
type=service
router=readwritesplit
servers=mariadb-1,mariadb-2,mariadb-3
user=max
password="123"
master_reconnection=true
transaction_replay=true

[Read-Write-Listener]
type=listener
service=Read-Write-Service
protocol=MariaDBClient
port=4006
```

---

## 5. Read it line by line

### `[maxscale]`
- `threads=auto`: MaxScale spawns one worker per CPU core. No change from the
  primary/replica setup.

### `[mariadb-1]`, `[mariadb-2]`, `[mariadb-3]`
- `type=server`, each section describes one database node.
- `address` / `port`, where that node listens.
- **No change** from the primary/replica setup. These sections are reused as-is.

### `[Galera-Monitor]`
- `type=monitor`, this section is a watcher.
- `module=galeramon`: **the key change**. `galeramon` is the monitor for Galera
  clusters. It detects cluster membership, sync status, and designates one node
  as the primary for writes (based on the cluster's internal ordering, not
  MaxScale's own promotion logic).
- `servers=mariadb-1,mariadb-2,mariadb-3`, all nodes must be listed. The
  monitor watches them all.
- `user=max` / `password="123"`, the account the monitor uses. The existing
  `max` user already has `REPLICA MONITOR` and `SELECT ON mysql.*`, which is
  exactly what `galeramon` needs.
- `monitor_interval=2000ms`, how often MaxScale checks each node. `2000ms`
  matches the earlier setup's `2s`, just expressed differently.

**Removed from this section:**

| Removed | What it did | Why it cannot stay |
|---|---|---|
| `failcount=5` | How many failed checks before `mariadbmon` declared a primary dead | `galeramon` has no failover logic; the field is meaningless |
| `replication_user=repl` | Account `mariadbmon` used to run `CHANGE MASTER TO` during failover | `galeramon` never issues `CHANGE MASTER TO` |
| `replication_password="123"` | Password for `replication_user` | Same |
| `auto_failover=true` | Told `mariadbmon` to promote a replica when the primary died | In Galera, promotion is not a MaxScale responsibility; Galera keeps every node writable and MaxScale just follows the cluster's primary designation |
| `auto_rejoin=true` | Told `mariadbmon` to bring a recovered primary back as a replica | Galera rejoins nodes automatically via IST/SST |

### `[Read-Write-Service]`
- `type=service`, this is the routing brain.
- `router=readwritesplit`, routes writes to the current primary and reads to
  the others. Correct for Galera: sending all writes to one node reduces
  deadlocks across the cluster.
- `servers=mariadb-1,mariadb-2,mariadb-3`, explicit list. **Changed from
  `cluster=MariaDB-Monitor`.** Both work, but `servers=` is clearer, matches the
  Galera reference configuration, and does not depend on the monitor object's
  name.
- `user=max` / `password="123"`, the service uses this account to read the
  user list so MaxScale can authenticate clients itself.
- `master_reconnection=true`, if the primary node changes, existing client
  sessions are transparently redirected to the new primary.
- `transaction_replay=true`, if a transaction was interrupted because the
  primary dropped, MaxScale tries to replay it on the new primary. This is
  optional but recommended for Galera because node changes happen on the
  cluster's terms, not MaxScale's.

### `[Read-Write-Listener]`
- `type=listener`, the door clients connect to.
- `service=Read-Write-Service`, which service this door belongs to.
- `protocol=MariaDBClient`, the client protocol.
- `port=4006`, the client-facing port. **No change** from the primary/replica
  setup; applications keep using the same address.

---

## 6. What happens if the old lines are kept

This is the most common mistake: leaving `mariadbmon` in place, or leaving the
`auto_failover` lines in a `galeramon` block.

**If you keep `module=mariadbmon` with the Galera cluster:**
`mariadbmon` expects a traditional primary/replica layout. It will read Galera's
state incorrectly, may report nodes as `Running` without a role, and may
attempt to promote a node when one drops. That conflicts with Galera's own
membership protocol. The result is unpredictable role assignments, wrong
routing, or MaxScale trying to run `CHANGE MASTER TO` on a node that is not
doing traditional replication.

**If you keep `auto_failover=true` but switch to `module=galeramon`:**
MaxScale will reject the configuration at startup, or ignore the field. Either
way, the intended behavior (MaxScale managing failover) does not occur, and the
operator may believe failover is enabled when it is not.

**If you keep `cluster=MariaDB-Monitor` instead of `servers=...`:**
It will still work, provided the monitor object is named `MariaDB-Monitor` and
contains the right servers. But the moment the monitor is renamed or replaced,
the service silently loses its servers. `servers=` is more explicit and
easier to audit.

---

## 7. Apply and verify

### 7.1 Validate the file

```bash
sudo -u maxscale maxscale --config-check
```

Expected output ends with:

```
Configuration was successfully verified.
```

If it fails, the error names the exact section and field.

### 7.2 Restart MaxScale

```bash
sudo systemctl restart maxscale
sleep 5
```

### 7.3 Check servers

```bash
maxctrl list servers
```

Expected:

```
│ mariadb-1 │ 10.10.10.201 │ 3306 │ 0 │ Master, Synced, Running │ ... │
│ mariadb-2 │ 10.10.10.202 │ 3306 │ 0 │ Slave, Synced, Running  │ ... │
│ mariadb-3 │ 10.10.10.203 │ 3306 │ 0 │ Slave, Synced, Running  │ ... │
```

- **One node is `Master`**, the designated primary for writes.
- **The others are `Slave`**, used for reads.
- **All show `Synced`**, meaning Galera reports them in sync.

If a node is `Down`, `DONOR`, or `Desynced`, see Section 9.

### 7.4 Check the monitor

```bash
maxctrl show monitor Galera-Monitor | grep -iE 'module|servers|user|monitor_interval'
```

Confirm:

```
"module": "galeramon"
"servers": ["mariadb-1","mariadb-2","mariadb-3"]
"user": "max"
```

### 7.5 Confirm routing works

Open a new client session through MaxScale:

```bash
mariadb -h 10.10.10.190 -P 4006 -u appuser -p
```

```sql
SELECT @@hostname;               -- returns a Galera node (replica)
START TRANSACTION;
SELECT @@hostname;               -- returns the Master node
COMMIT;
INSERT INTO test.notes (txt) VALUES ('through maxscale to galera');
SELECT * FROM test.notes;        -- read from a replica
```

The write goes to the node MaxScale designates as `Master`. Reads are spread
across the others.

---

## 8. Behaviour you should expect

- **Writes go to one designated node** (the one shown as `Master`). All Galera
  nodes could accept writes directly, but routing them through one node via
  MaxScale prevents conflicting transactions and deadlocks.
- **Reads go to the other nodes**, distributing read load.
- **If the primary node drops**, `galeramon` designates another node as
  `Master` at the next monitor interval. New writes go there. Sessions using
  `master_reconnection=true` are moved; sessions with
  `transaction_replay=true` have their interrupted transaction replayed.
- **If a non-primary node drops**, it is simply removed from the read pool. No
  role change occurs.
- **If a node rejoins after being repaired**, `galeramon` sees it become
  `Synced` again and adds it back to the pool automatically. No MaxScale action
  is required.

---

## 9. Troubleshooting

### `mariadb-N` shows `Down`

MaxScale cannot reach or authenticate to the node.

1. Test the login from MaxScale:
   ```bash
   mariadb --skip-ssl -h 10.10.10.20N -u max -p123 -e "SELECT 1"
   ```
2. If it fails, check the log:
   ```bash
   sudo tail -n 30 /var/log/maxscale/maxscale.log | grep -i mariadb-N
   ```
3. If the log says `Access denied ... (using password: YES)`, the password or
   grants on that node are wrong. Compare:
   ```sql
   SHOW GRANTS FOR 'max'@'10.10.10.%';
   ```
4. If the log says `Can't connect ... (115)`, check the node's firewall:
   ```bash
   sudo firewall-cmd --list-ports
   ```
   Port 3306 must be open.

### `mariadb-N` shows `DONOR`

The node is performing an SST (State Snapshot Transfer) for a joining node and
is temporarily unavailable for normal traffic. This is normal. `galeramon`
marks it `DONOR` and excludes it from the read pool until the transfer finishes.
No action is required.

### `mariadb-N` shows `Desynced`

The node is part of the cluster but is behind on applying transactions. Check
on the node:

```sql
SHOW STATUS LIKE 'wsrep_local_state_comment';
```

If it is `Desynced` and not recovering, the node may need an IST or SST:

```bash
sudo systemctl restart mariadb
```

Watch `/var/log/mariadb/mariadb.log` for the recovery process.

### MaxScale starts but `maxctrl list servers` shows all nodes `Slave` (no `Master`)

`galeramon` designates a primary based on the cluster's internal state. If no
node has been chosen, the cluster itself may be in a bad state. On any node:

```sql
SHOW STATUS LIKE 'wsrep_cluster_status';
```

It must be `Primary`. If it is `non-Primary`, the cluster has lost quorum and
needs to be recovered before MaxScale can route to it.

### `mariadb --skip-ssl -h 10.10.10.190 -P 4006 -u appuser -p` fails

This is a client login failure through MaxScale, not a Galera problem.

1. Check the `appuser` grants on a Galera node:
   ```sql
   SHOW GRANTS FOR 'appuser'@'%';
   ```
2. Reload the user list on MaxScale:
   ```bash
   maxctrl reload service Read-Write-Service
   ```
3. Check the MaxScale log:
   ```bash
   sudo tail -n 30 /var/log/maxscale/maxscale.log | grep -i appuser
   ```

---

## 10. Summary of the switch

| Before | After |
|---|---|
| `mariadbmon` watched the primary/replica pair | `galeramon` watches the whole Galera cluster |
| `auto_failover` promoted a replica when the primary died | Galera handles failure; `galeramon` designates the next primary |
| `replication_user` / `replication_password` were needed to repoint replicas | Removed; no repointing occurs |
| Service used `cluster=MariaDB-Monitor` | Service lists servers explicitly |
| No session resilience options | `master_reconnection` and `transaction_replay` enabled |
| Application address `10.10.10.190:4006` | **Unchanged** |

The application never sees the change. It keeps connecting to the same MaxScale
address and port. What changed is how MaxScale understands the cluster behind
it: from "one writer and N readers" to "a multi-primary group where one node
is currently designated for writes."

---

## 11. Documentation references

- MaxScale Galera Monitor:
  https://mariadb.com/kb/en/mariadb-maxscale-25-galera-monitor/
- MaxScale readwritesplit router:
  https://mariadb.com/kb/en/mariadb-maxscale-25-readwritesplit/
- Galera cluster implementation on this environment:
  `mariadb-galera-cluster-implementation.md`
- Primary/replica MaxScale build (previous configuration):
  `02-maxscale-single-node.md`
