# MariaDB Galera Cluster Implementation Guide

A complete record of how the 3 node Galera cluster was built on AlmaLinux 10,
including every command used and every workaround applied. Written so the same
result can be reproduced later.

## 0. Environment

| Role | Hostname | IP | Notes |
|---|---|---|---|
| Node 1 | mariadb-1 | 10.10.10.201 | Bootstrapped (cluster creator) |
| Node 2 | mariadb-2 | 10.10.10.202 | Joins normally |
| Node 3 | mariadb-3 | 10.10.10.203 | Joins normally |
| Proxy (existing) | maxscale | 10.10.10.190 | Not part of Galera; see Section 8 |

OS: AlmaLinux 10.2 (Lavender Lion), x86_64
MariaDB: 10.11 series
Galera provider: `galera` package from AlmaLinux AppStream
SST method: `rsync`

The three nodes previously formed a primary/replica setup. That replication was
removed before Galera was configured. Galera cannot coexist with standard
replication settings.

## 1. Prerequisites (run on all three nodes)

### 1.1 Install rsync (needed for SST)

```bash
sudo dnf install rsync -y
```

### 1.2 Install the Galera provider and the bootstrap script

```bash
sudo dnf install galera mariadb-server-galera -y
```

Why two packages:

`galera` provides the wsrep provider library (`libgalera_smm.so`). This is the
actual replication engine.
`mariadb-server-galera` provides the `galera_new_cluster` script and the
systemd integration needed to bootstrap a cluster. Without it,
`galera_new_cluster` does not exist and the bootstrap step fails with
`command not found`.

The package is named `galera` on AlmaLinux 10. `dnf install galera-4` fails
with `No match for argument: galera-4`.

### 1.3 Confirm the provider library path

```bash
find /usr -name 'libgalera_smm.so' 2>/dev/null
```

On AlmaLinux 10 the output is:

```
/usr/lib64/galera/libgalera_smm.so
```

This exact path must be used in `wsrep_provider=`.

### 1.4 Stop and disable the old MariaDB service

```bash
sudo systemctl stop mariadb
sudo systemctl disable mariadb
```

The nodes were primary/replicas. Starting MariaDB with the old replication
config while Galera is also enabled causes conflicts. Stop it, configure
Galera, then start via the Galera bootstrap path.

### 1.5 Remove conflicting settings from `mariadb-server.cnf`

Edit `/etc/my.cnf.d/mariadb-server.cnf` and comment out or remove any of these
lines under `[mysqld]`, if present:

```ini
#server_id         = ...
#log_bin           = ...
#binlog_format     = ...
#log_slave_updates = ...
#read_only         = ...
```

Galera requires its own settings for these. Leaving the old replication values
in place produces warnings or errors at startup and can prevent the node from
syncing. `galera.cnf` (next section) sets `binlog_format=ROW` explicitly; the
rest are managed by Galera. Leave `bind-address` and `datadir` as they are.

### 1.6 Open the Galera ports

```bash
sudo firewall-cmd --permanent --add-port=3306/tcp
sudo firewall-cmd --permanent --add-port=4567/tcp
sudo firewall-cmd --permanent --add-port=4567/udp
sudo firewall-cmd --permanent --add-port=4568/tcp
sudo firewall-cmd --permanent --add-port=4444/tcp
sudo firewall-cmd --reload
sudo firewall-cmd --list-ports
```

Why each port:

| Port | Purpose |
|---|---|
| 3306 | Normal client connections (MaxScale, apps) |
| 4567 | Galera group communication between nodes (both TCP and UDP) |
| 4568 | IST, a node catching up from a short absence |
| 4444 | SST, a node receiving a full data copy on first join |

Without 4567 the nodes cannot see each other. Without 4444 the joining nodes
cannot receive the initial snapshot. Without 4568, restarted nodes fall back to
a full SST instead of a fast IST.

## 2. Create `galera.cnf` on all three nodes

```bash
sudo vim /etc/my.cnf.d/galera.cnf
```

Content (shown for node1; node2 and node3 change only the two marked lines):

```ini
[mysqld]
# Basic Galera settings
binlog_format=ROW
default_storage_engine=InnoDB
innodb_autoinc_lock_mode=2
bind-address=0.0.0.0

# Galera Provider
wsrep_on=ON
wsrep_provider=/usr/lib64/galera/libgalera_smm.so

# Cluster Configuration (identical on every node)
wsrep_cluster_name="my_galera_cluster"
wsrep_cluster_address="gcomm://10.10.10.201,10.10.10.202,10.10.10.203"

# Node Configuration (CHANGE THESE TWO LINES PER NODE)
wsrep_node_name="node1"
wsrep_node_address="10.10.10.201"

# SST Method
wsrep_sst_method=rsync
```

Per node values:

| Node | `wsrep_node_name` | `wsrep_node_address` |
|---|---|---|
| node1 (.201) | `node1` | `10.10.10.201` |
| node2 (.202) | `node2` | `10.10.10.202` |
| node3 (.203) | `node3` | `10.10.10.203` |

Why each setting exists:

| Setting | Purpose |
|---|---|
| `binlog_format=ROW` | Galera requires row based binary logging |
| `default_storage_engine=InnoDB` | Galera only replicates transactional storage engines |
| `innodb_autoinc_lock_mode=2` | Required for multi primary writes; prevents auto increment conflicts |
| `bind-address=0.0.0.0` | Listen on all interfaces so other nodes and MaxScale can reach it |
| `wsrep_on=ON` | Enables the Galera replication plugin |
| `wsrep_provider=` | Path to the Galera library. Must match the actual file from Section 1.3 |
| `wsrep_cluster_name` | Must be identical on every node. Nodes with a different name form separate clusters |
| `wsrep_cluster_address` | Comma separated list of all node IPs, prefixed with `gcomm://`. Identical on every node |
| `wsrep_node_name` | A label, unique per node. Not the OS hostname; just a unique string |
| `wsrep_node_address` | This node's own IP. Different per node |
| `wsrep_sst_method=rsync` | How a joining node receives the initial data copy. `rsync` is the simplest method |

Verify the file before proceeding, on each node:

```bash
grep -E 'wsrep_(on|provider|cluster_name|cluster_address|node_name|node_address|sst_method)' /etc/my.cnf.d/galera.cnf
```

Expected on node1:

```
wsrep_on=ON
wsrep_provider=/usr/lib64/galera/libgalera_smm.so
wsrep_cluster_name="my_galera_cluster"
wsrep_cluster_address="gcomm://10.10.10.201,10.10.10.202,10.10.10.203"
wsrep_node_name="node1"
wsrep_node_address="10.10.10.201"
wsrep_sst_method=rsync
```

node2 and node3 differ only in `wsrep_node_name` and `wsrep_node_address`.

## 3. Fix the `galera_new_cluster` bug (node1 only)

On AlmaLinux 10's `mariadb-server` package, the `galera_new_cluster` script
creates a flag file (`/run/mariadb/wsrep-new-cluster`) that is supposed to tell
MariaDB to bootstrap a new cluster. The systemd unit in this package does not
read that file, so `galera_new_cluster` starts MariaDB as if it were a normal
join attempt, and it fails because no cluster exists yet.

Symptom: `galera_new_cluster` runs, but MariaDB does not come up as a new
cluster. The log shows the node trying to connect to the `wsrep_cluster_address`
nodes and timing out or reporting no primary component.

Workaround (node1 only):

```bash
sudo mkdir -p /etc/systemd/system/mariadb.service.d
sudo tee /etc/systemd/system/mariadb.service.d/fix_galera_new_cluster.conf >/dev/null <<'EOF'
[Service]
EnvironmentFile=-/run/mariadb/wsrep-new-cluster
EOF
sudo systemctl daemon-reload
```

What this does:

Creates a systemd drop in that adds
`EnvironmentFile=-/run/mariadb/wsrep-new-cluster` to the MariaDB service.
The leading `-` means "if the file does not exist, ignore it silently."
When `galera_new_cluster` runs, it creates that file containing
`_WSREP_NEW_CLUSTER=--wsrep-new-cluster`. systemd now reads it and passes the
flag to MariaDB, which then bootstraps the cluster as intended.
On node2 and node3, the file never exists, so the drop in has no effect.

This is a workaround for a packaging bug. Apply it to node1 only. If a future
AlmaLinux update fixes the package, this drop in can be removed.

## 4. Bootstrap the cluster

Bootstrapping means "start the very first node of a brand new cluster."
It must be done on exactly one node, exactly once, on the initial start.

If you bootstrap two nodes at the same time, each forms its own separate one
node cluster. They never merge. This is called split brain, and the only fix is
to wipe the data directories and start over.

On node1 (.201) only:

```bash
sudo galera_new_cluster
```

This starts MariaDB with `--wsrep-new-cluster`, forming a one node primary
component.

On node2 (.202) and node3 (.203):

```bash
sudo systemctl start mariadb
```

These start normally. They read `wsrep_cluster_address`, contact node1, and
join. On first join they receive a full SST (`rsync` copy) of node1's data
directory.

Important: the SST from node1 overwrites the data directory on the joining
nodes. If the previous primary/replica setup left different data on each node,
the cluster will contain whatever node1 has. This is expected. For a production
migration, take a dump first or plan which node's data is authoritative.

If MariaDB fails to start:

```bash
sudo journalctl -u mariadb -n 30 --no-pager
```

Common causes and fixes:

| Error | Cause | Fix |
|---|---|---|
| `WSREP: Failed to load provider` or `dlopen(): ... No such file` | `wsrep_provider` path wrong | Use the actual path from Section 1.3 |
| `It may not be safe to bootstrap the cluster from this node` | `safe_to_bootstrap: 0` in `/var/lib/mysql/grastate.dat` from an unclean previous shutdown | If this node has no unique data, `sudo sed -i 's/^safe_to_bootstrap: 0/safe_to_bootstrap: 1/' /var/lib/mysql/grastate.dat` and retry `galera_new_cluster` |
| Node starts but `wsrep_cluster_size` stays 1 | Other nodes not started, or firewall blocking 4567/4444 | Check `firewall-cmd --list-ports` on all nodes, start node2/node3 |
| `Access denied` or auth errors | Old replication users or config still active | Confirm Section 1.5 was applied |

## 5. Verify the cluster

### 5.1 Cluster size

On any node:

```bash
sudo mariadb -u root -p -e "SHOW STATUS LIKE 'wsrep_cluster_size';"
```

Expected:

```
| wsrep_cluster_size | 3 |
```

### 5.2 Full health check

On any node:

```sql
SHOW STATUS LIKE 'wsrep_cluster_status';       -- Primary
SHOW STATUS LIKE 'wsrep_ready';                -- ON
SHOW STATUS LIKE 'wsrep_connected';            -- ON
SHOW STATUS LIKE 'wsrep_local_state_comment';  -- Synced
```

All four must match the comments. If `wsrep_ready` is `OFF` or the state is not
`Synced`, that node is still catching up. Wait a minute and re check.

### 5.3 Synchronous replication test

On node1:

```sql
CREATE DATABASE galera_test;
USE galera_test;
CREATE TABLE messages (id INT AUTO_INCREMENT PRIMARY KEY, text VARCHAR(255));
INSERT INTO messages (text) VALUES ('written on node1');
```

On node2:

```sql
SELECT * FROM galera_test.messages;
```

The row must appear immediately.

Then write on node2:

```sql
INSERT INTO galera_test.messages (text) VALUES ('written on node2');
```

Check on node1. The row must be there. This proves multi primary synchronous
replication works in both directions.

### 5.4 Node view

From any node:

```sql
SHOW STATUS LIKE 'wsrep_incoming_addresses';
```

Should list all three node addresses:

```
10.10.10.201:3306,10.10.10.202:3306,10.10.10.203:3306
```

## 6. What the cluster now does

Multi primary: every node accepts reads and writes.
Synchronous: a transaction commits only after it is certified across the
primary component. All nodes see the same logical order.
Automatic membership: if a node fails, the remaining two continue. When it
returns, it rejoins (via IST if it was gone briefly, or SST if it fell too far
behind).
Quorum: with three nodes, the cluster tolerates one node being down. With two
nodes up, there is still quorum. With one node up, the cluster loses quorum and
stops accepting writes. This is by design and prevents split brain.

## 7. MaxScale is not compatible with this cluster as currently configured

The existing `/etc/maxscale.cnf` uses:

```ini
[MariaDB-Monitor]
module=mariadbmon
...
[Read-Write-Service]
router=readwritesplit
```

`mariadbmon` and `readwritesplit` are designed for a primary/replica topology:
one writer, one or more readers. A Galera cluster is multi primary: every node
is a writer. Pointing `mariadbmon` at a Galera cluster produces incorrect role
detection and routing.

To put MaxScale in front of Galera, the configuration must change:

Monitor module: `mariadbmon` becomes `galeramon`
Service router: `readwritesplit` becomes `readconnroute` with
`router_options=master`

This is a separate change and is not part of the Galera build documented here.
Do not point the existing MaxScale config at the Galera cluster until that
reconfiguration is done.

## 8. Reproduction checklist

Run in this order. Every step must succeed before moving on.

| Step | Command / check | Expected |
|---|---|---|
| 1 | `sudo dnf install rsync galera mariadb-server-galera -y` (all nodes) | Packages install |
| 2 | `find /usr -name libgalera_smm.so` (all nodes) | `/usr/lib64/galera/libgalera_smm.so` |
| 3 | `sudo systemctl stop mariadb && sudo systemctl disable mariadb` (all nodes) | Service stopped |
| 4 | Remove `server_id`, `log_bin`, `log_slave_updates`, `read_only` from `mariadb-server.cnf` (all nodes) | File edited |
| 5 | Open ports 3306, 4567/tcp+udp, 4568, 4444 (all nodes) | `firewall-cmd --list-ports` shows them |
| 6 | Create `/etc/my.cnf.d/galera.cnf` with per node name/address (all nodes) | `grep wsrep_` matches Section 2 |
| 7 | Create systemd drop in `fix_galera_new_cluster.conf` (node1 only) | File exists, `daemon-reload` done |
| 8 | `sudo galera_new_cluster` (node1 only) | MariaDB starts, log shows new cluster |
| 9 | `sudo systemctl start mariadb` (node2, node3) | Both start and join |
| 10 | `SHOW STATUS LIKE 'wsrep_cluster_size'` (any node) | `3` |
| 11 | `SHOW STATUS LIKE 'wsrep_ready'` | `ON` |
| 12 | Write on node1, read on node2 | Row visible |
| 13 | Write on node2, read on node1 | Row visible |

## 9. Reference: all commands used, in order

On all three nodes:

```bash
# 1. Packages
sudo dnf install rsync -y
sudo dnf install galera mariadb-server-galera -y

# 2. Confirm provider path
find /usr -name 'libgalera_smm.so' 2>/dev/null

# 3. Stop old service
sudo systemctl stop mariadb
sudo systemctl disable mariadb

# 4. Edit /etc/my.cnf.d/mariadb-server.cnf: comment out
#    server_id, log_bin, log_slave_updates, read_only

# 5. Firewall
sudo firewall-cmd --permanent --add-port=3306/tcp
sudo firewall-cmd --permanent --add-port=4567/tcp
sudo firewall-cmd --permanent --add-port=4567/udp
sudo firewall-cmd --permanent --add-port=4568/tcp
sudo firewall-cmd --permanent --add-port=4444/tcp
sudo firewall-cmd --reload

# 6. Create /etc/my.cnf.d/galera.cnf (Section 2)
sudo vim /etc/my.cnf.d/galera.cnf

# 7. Verify
grep -E 'wsrep_(on|provider|cluster_name|cluster_address|node_name|node_address|sst_method)' /etc/my.cnf.d/galera.cnf
```

On node1 (.201) only:

```bash
# 8. Workaround for the AlmaLinux 10 galera_new_cluster bug
sudo mkdir -p /etc/systemd/system/mariadb.service.d
sudo tee /etc/systemd/system/mariadb.service.d/fix_galera_new_cluster.conf >/dev/null <<'EOF'
[Service]
EnvironmentFile=-/run/mariadb/wsrep-new-cluster
EOF
sudo systemctl daemon-reload

# 9. Bootstrap
sudo galera_new_cluster
```

On node2 (.202) and node3 (.203):

```bash
# 10. Join
sudo systemctl start mariadb
```

Verification (any node):

```bash
sudo mariadb -u root -p -e "SHOW STATUS LIKE 'wsrep_cluster_size';"
sudo mariadb -u root -p -e "SHOW STATUS LIKE 'wsrep_cluster_status';"
sudo mariadb -u root -p -e "SHOW STATUS LIKE 'wsrep_ready';"
sudo mariadb -u root -p -e "SHOW STATUS LIKE 'wsrep_local_state_comment';"
```

## 10. Notes for the future

Bootstrapping is a one time operation. After the cluster exists, start nodes
with `sudo systemctl start mariadb`, not `galera_new_cluster`. Use
`galera_new_cluster` only if you are intentionally rebuilding the whole cluster
from scratch (for example after wiping all data directories).
Never bootstrap two nodes. That creates two separate clusters (split brain).
The systemd drop in is a workaround. If a future AlmaLinux package update fixes
the `mariadb-server-galera` integration, the drop in can be removed.
SST overwrites data. When a new node joins, it receives node1's data directory.
Confirm that is what you want before adding a node.
MaxScale needs reconfiguration before it can sit in front of this cluster
(Section 7). The existing `mariadbmon` + `readwritesplit` config is for
primary/replica, not multi primary.
Quorum matters. Three nodes tolerate one failure. Two nodes up still have
quorum. One node alone has no quorum and refuses writes. This is intended, to
prevent split brain.

## 11. Documentation references

This guide was written with the help of the following MariaDB documentation:

https://mariadb.com/docs/galera-cluster/galera-cluster-quickstart-guides/mariadb-galera-cluster-guide

https://github.com/mariadb-corporation/mariadb-docs/blob/main/galera-cluster/galera-cluster-quickstart-guides/mariadb-galera-cluster-guide.md

```

**File name:** `mariadb-galera-cluster-implementation.md`
