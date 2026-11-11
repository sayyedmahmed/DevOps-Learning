# Galera Cluster Recovery and Restart Procedure

**Cluster:** `my_galera_cluster`
**Nodes:** `mariadb-1` (10.10.10.201), `mariadb-2` (10.10.10.202), `mariadb-3` (10.10.10.203)
**Software:** MariaDB 10.11 / Galera 26.4.27

---

## 1. Purpose

This document defines the standard procedure for restarting a MariaDB Galera cluster when all nodes are offline. It covers how to select the correct node to bootstrap, how to bring up the remaining nodes, how to verify cluster health, and how to resolve common error conditions.

**Applies to:** Any situation in which the entire `my_galera_cluster` is down, whether due to planned maintenance, crash, power loss, or a failed restart attempt.

---

## 2. Background

### 2.1 Cluster ports

| Port | Protocol | Purpose |
|---|---|---|
| 4567 | TCP and UDP | Galera replication |
| 4568 | TCP | Incremental State Transfer (IST) |
| 4444 | TCP | State Snapshot Transfer (SST) |
| 3306 | TCP | MySQL client access |

All ports must be reachable between all three nodes.

### 2.2 Key concepts

| Term | Definition |
|---|---|
| **Primary component** | The set of nodes permitted to accept writes. A cluster must have exactly one primary component. |
| **Bootstrap** | Starting the first node of a cluster to create the primary component. Uses `galera_new_cluster`, not `systemctl start`. |
| **Join** | A node connecting to an existing primary component. Uses `systemctl start mariadb`. |
| **IST** | Incremental State Transfer. Ships only missing transactions. Fast. |
| **SST** | State Snapshot Transfer. Full dataset copy. Slow. |
| **`seqno`** | The last transaction number applied by a node. Higher means newer data. |
| **`safe_to_bootstrap`** | Flag indicating a node was the last to shut down cleanly and is safe to bootstrap. |

### 2.3 State file

Location: `/var/lib/mysql/grastate.dat`

```ini
version: 2.1
uuid:    99abc27e-dcce-11f1-a7ba-eb92f6b0750a
seqno:   50
safe_to_bootstrap: 1
```

| Field | Meaning |
|---|---|
| `uuid` | Cluster transaction history identifier. Identical on all nodes. |
| `seqno` | Last applied transaction. `-1` means unclean shutdown, `0` means no writes yet, `>0` means normal. |
| `safe_to_bootstrap` | `1` means last clean member and safe to bootstrap. `0` means not confirmed. |

### 2.4 Fundamental rule

When the entire cluster is offline, exactly one node must be bootstrapped, and that node must be the one with the highest `seqno`. All remaining nodes must then be started normally. Starting all nodes with `systemctl start mariadb` on an offline cluster will always fail with `failed to reach primary view`.

---

## 3. Procedure for Restarting All Nodes

### Step 1: Inspect cluster state

On each node (10.10.10.201, 10.10.10.202, 10.10.10.203), execute:

```bash
cat /var/lib/mysql/grastate.dat
```

Record the `seqno` and `safe_to_bootstrap` values from all three nodes.

**Example output:**

| Node | seqno | safe_to_bootstrap |
|---|---|---|
| mariadb-1 | 50 | 1 |
| mariadb-2 | 48 | 0 |
| mariadb-3 | 49 | 0 |

### Step 2: Select the bootstrap node

Apply the following rules in order:

| Condition | Bootstrap node |
|---|---|
| A single node has the highest `seqno` | That node |
| Two nodes tie at highest `seqno` | The one with `safe_to_bootstrap: 1` |
| Highest `seqno` node has `safe_to_bootstrap: 0` | That node, after setting the flag manually (Step 2a) |
| All nodes have `seqno: -1` | See Section 4, Unclean Shutdown |

In the example above, **mariadb-1** (seqno 50, safe_to_bootstrap 1) is the bootstrap node.

### Step 2a: Highest `seqno` node has `safe_to_bootstrap: 0`

Only if no node has `safe_to_bootstrap: 1`, and the node has the highest `seqno`, force the flag:

```bash
sed -i 's/^safe_to_bootstrap:.*/safe_to_bootstrap: 1/' /var/lib/mysql/grastate.dat
grep safe_to_bootstrap /var/lib/mysql/grastate.dat
```

Expected output: `safe_to_bootstrap: 1`

**Warning:** This must be performed only on the highest `seqno` node. Forcing a lower `seqno` node to bootstrap will cause higher `seqno` nodes to roll back and lose transactions.

### Step 3: Bootstrap the selected node

On the bootstrap node only:

```bash
galera_new_cluster
```

Confirm the primary component was created:

```bash
mysql -e "SHOW STATUS LIKE 'wsrep_cluster_status';"
```

Expected output: `Primary`

If the command fails, proceed to Section 7, Error Reference.

### Step 4: Start remaining nodes

On each remaining node, in order of descending `seqno` (this order maximizes IST efficiency but is not strictly required):

```bash
systemctl start mariadb
```

The joiners will perform an IST from the bootstrap node. Monitor progress:

```bash
tail -f /var/log/mariadb/mariadb.log
```

### Step 5: Verify cluster health

From any node:

```bash
mysql -e "SHOW STATUS LIKE 'wsrep_cluster_size'; \
          SHOW STATUS LIKE 'wsrep_cluster_status'; \
          SHOW STATUS LIKE 'wsrep_local_state_comment'; \
          SHOW STATUS LIKE 'wsrep_connected'; \
          SHOW STATUS LIKE 'wsrep_last_committed';"
```

Expected values on all three nodes:

| Variable | Expected |
|---|---|
| `wsrep_cluster_size` | 3 |
| `wsrep_cluster_status` | Primary |
| `wsrep_local_state_comment` | Synced |
| `wsrep_connected` | ON |
| `wsrep_last_committed` | Identical across all nodes |

Confirm `seqno` convergence:

```bash
grep seqno /var/lib/mysql/grastate.dat
```

All three nodes must report the same `seqno`.

---

## 4. Procedure for Unclean Shutdown (All Nodes `seqno: -1`)

Use this procedure when the cluster was terminated abnormally (power loss, forced shutdown, crash) and all nodes report `seqno: -1`.

### Step 1: Recover each node's actual position

On each node:

```bash
/usr/bin/galera_recovery
```

The command outputs the last recoverable position, for example:

```
99abc27e-dcce-11f1-a7ba-eb92f6b0750a:38
```

The number after the colon is the recovered `seqno`. Record this value for all three nodes.

### Step 2: Select the bootstrap node

Identify the node with the highest recovered `seqno`. This node contains the most recent data.

### Step 3: Force bootstrap on the selected node

On that node only:

```bash
sed -i 's/^safe_to_bootstrap:.*/safe_to_bootstrap: 1/' /var/lib/mysql/grastate.dat
galera_new_cluster
```

### Step 4: Start the remaining nodes

```bash
systemctl start mariadb
```

### Step 5: Post recovery cleanup

Once all nodes are `Synced`, revert the flag on all joiner nodes:

```bash
sed -i 's/^safe_to_bootstrap:.*/safe_to_bootstrap: 0/' /var/lib/mysql/grastate.dat
```

Verify per Section 3, Step 5.

**Data loss risk:** Transactions committed after the last successful flush on the selected node are lost. Always select the node with the highest recovered `seqno`.

---

## 5. Procedure for Planned Shutdown

To ensure a straightforward restart, always shut down the cluster so that one node is clearly marked as the last clean member.

### Stop order

```bash
# On mariadb-1 and mariadb-2:
systemctl stop mariadb

# On mariadb-3 (LAST):
systemctl stop mariadb
```

The node stopped last receives `safe_to_bootstrap: 1` automatically.

### Autostart

Disable the service on all nodes to prevent the cluster from attempting an invalid automatic restart after a reboot:

```bash
systemctl disable mariadb
```

---

## 6. Verification Reference

### 6.1 Key status variables

| Variable | Healthy value | Meaning |
|---|---|---|
| `wsrep_cluster_size` | 3 | Members in the cluster |
| `wsrep_cluster_status` | Primary | Primary component exists |
| `wsrep_ready` | ON | Node can accept SQL |
| `wsrep_local_state_comment` | Synced | Node is in sync |
| `wsrep_connected` | ON | Provider connected |
| `wsrep_last_committed` | Increasing and identical on all nodes | Current cluster `seqno` |
| `wsrep_local_cached_downto` | Less than or equal to joiner `seqno` | Oldest transaction in gcache. IST is possible when joiner `seqno` is greater than or equal to this value. |
| `wsrep_flow_control_paused` | Near 0 | Node lag |

### 6.2 Diagnostic commands

| Purpose | Command |
|---|---|
| State file | `cat /var/lib/mysql/grastate.dat` |
| Listening ports | `ss -tlnp \| grep -E '4567\|4568\|4444\|3306'` |
| Service status | `systemctl status mariadb --no-pager -l` |
| Service journal | `journalctl -xeu mariadb.service --no-pager \| tail -50` |
| MariaDB log | `tail -f /var/log/mariadb/mariadb.log` |
| Recovery position | `/usr/bin/galera_recovery` |
| Configuration | `grep -rE 'wsrep_\|bind-address' /etc/my.cnf /etc/my.cnf.d/` |
| Port reachability | `nc -zv 10.10.10.202 4567` |
| Firewall rules | `firewall-cmd --list-all` |

---

## 7. Error Reference

### 7.1 `WSREP: Failed to establish connection: Connection refused`

**Cause:** The target node's Galera port (4567) is not listening.
**Resolution:** Start the peer node. If all nodes are offline, follow Section 3.

### 7.2 `WSREP: failed to reach primary view` or `-110 (Connection timed out)`

**Cause:** No primary component exists.
**Resolution:** Bootstrap the highest `seqno` node (Section 3, Step 3).

### 7.3 `It may not be safe to bootstrap the cluster from this node...`

**Cause:** `safe_to_bootstrap` is `0`.
**Resolution:** Confirm this is the highest `seqno` node, then set the flag to `1` (Section 3, Step 2a).

### 7.4 `WSREP: wsrep::connect(gcomm://...) failed: 7`

**Cause:** Unable to reach any configured peer.
**Resolution:** Verify peers are running, `wsrep_node_address` is correct on each node, and firewall rules permit ports 4567, 4568, and 4444.

### 7.5 `Aborting` (service exits with code 1)

**Cause:** Galera abandoned the attempt to join. The specific reason is logged immediately above.
**Resolution:** Inspect the preceding lines of `/var/log/mariadb/mariadb.log`.

### 7.6 `SST failed` or joiner aborts during startup

**Cause:** SST port (4444) or IST port (4568) blocked, incorrect datadir ownership, or insufficient disk space on the joiner.
**Resolution:**

```bash
firewall-cmd --add-port=4444/tcp --add-port=4568/tcp --add-port=4567/tcp --add-port=4567/udp --permanent
firewall-cmd --reload
```

Verify that `/var/lib/mysql` is owned by `mysql:mysql` and has adequate free space on both donor and joiner. If the joiner's data is inconsistent beyond gcache range, recreate its datadir:

```bash
systemctl stop mariadb
mv /var/lib/mysql /var/lib/mysql.bak.$(date +%F)
mkdir -p /var/lib/mysql
chown mysql:mysql /var/lib/mysql
systemctl start mariadb
```

### 7.7 `mariadb.service: Referenced but unset environment variable ... MYSQLD_OPTS`

**Cause:** Benign systemd warning.
**Resolution:** None required.

### 7.8 `Can't connect to local server through socket '/var/lib/mysql/mysql.sock' (2)`

**Cause:** The local `mariadb` service is not running.
**Resolution:** Start the service, or run the command on a node where the service is active.

---

## 8. Preflight Checklist

Before any restart attempt:

- [ ] All nodes reachable via ICMP (`ping 10.10.10.20x`)
- [ ] `grastate.dat` inspected on all three nodes
- [ ] Bootstrap node identified (highest `seqno`, tie broken by `safe_to_bootstrap: 1`)
- [ ] `wsrep_node_address` set to the node's 10.10.10.x address, not `172.17.119.46` and not `127.0.0.1`
- [ ] `bind-address` is not `127.0.0.1`
- [ ] Firewall permits 4567/tcp, 4567/udp, 4568/tcp, 4444/tcp, and 3306/tcp on all nodes
- [ ] `mariadb.service` disabled on all nodes
- [ ] `safe_to_bootstrap` set to `0` on all joiner nodes
- [ ] Sufficient disk space on the bootstrap node for gcache and binary logs
- [ ] Application writes paused during recovery

---

## 9. Summary

| Situation | Action |
|---|---|
| One node has clearly highest `seqno` | Bootstrap that node, then start the others normally |
| Tie on highest `seqno` | Bootstrap the node with `safe_to_bootstrap: 1` |
| Highest `seqno` node has flag `0` | Set the flag to `1`, then bootstrap |
| All nodes `seqno: -1` | Use `/usr/bin/galera_recovery` on each node and bootstrap the highest recovered position |
| One node down, others running | Start the failed node normally. Do not bootstrap. |

**Standard command sequence:**

```bash
# All nodes:
cat /var/lib/mysql/grastate.dat

# Bootstrap node:
galera_new_cluster

# Remaining nodes:
systemctl start mariadb

# Verification (any node):
mysql -e "SHOW STATUS LIKE 'wsrep_cluster_size'; SHOW STATUS LIKE 'wsrep_local_state_comment';"
```

**Rule:** The highest `seqno` node bootstraps. All other nodes are started normally.
