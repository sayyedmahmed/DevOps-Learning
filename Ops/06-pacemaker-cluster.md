# 06 - Simple Pacemaker Cluster (AlmaLinux 10)

## Overview

This guide builds a small 2-node High Availability (HA) cluster. The goal is to understand how Pacemaker works by creating two virtual machines that cooperate. If one VM fails, the other automatically takes over its services to maintain uptime.

### Concept: How It Works

Think of an HA cluster like a relay race. In a standard setup, if one runner falls, the team loses. In an HA cluster, if `node1` crashes, `node2` instantly picks up the baton (the service). The end-user never notices the switch.

| Term | Simple Definition |
|---|---|
| **Cluster** | A group of computers acting as a single system. |
| **Node** | One individual computer or VM in the cluster. |
| **Corosync** | The communication layer ("telephone line") that lets nodes check on each other. |
| **Pacemaker** | The resource manager that decides which node runs which service. |
| **pcs** | The command-line tool used to configure and control Pacemaker. |
| **Resource** | Any component managed by the cluster (e.g., an IP address, a web server). |
| **Failover** | The automatic movement of resources from a failed node to a healthy one. |
| **Quorum** | The minimum number of active nodes required for the cluster to operate safely. |
| **Fencing (STONITH)** | Forcibly powering off a suspected-failed node to prevent data corruption. Disabled in this lab for simplicity. |

### Prerequisites

- One working AlmaLinux VM from [Guide 01](01-hyperv-vm-setup.md). This will become `node1`.
- Internet connectivity on the VM (`eth0`).
- Approximately 20 GB free disk space on the Windows host for cloning the second VM.
- 2 GB RAM per VM (4 GB total recommended).

---

## Part A - Prepare Node 1

We install all necessary software on `node1` first. Then, we clone it to create `node2`. This approach saves time and ensures consistency.

### Step 1 - Verify Connectivity

Run on: `node1`

```bash
ping -c 4 8.8.8.8
```

Ensure you receive replies. If not, resolve network issues before proceeding.

### Step 2 - Update System Packages

Run on: `node1`

```bash
dnf update -y
```

### Step 3 - Enable High Availability Repository

Pacemaker is not included in the default repositories. It resides in the specialized `highavailability` repo.

Run on: `node1`

```bash
# Install the repository management plugin
dnf install -y dnf-plugins-core

# Enable the HA repository
dnf config-manager --set-enabled highavailability

# Verify the repo is enabled
dnf repolist
```

Look for `highavailability` in the output. If missing, find the exact name using `dnf repolist --all | grep -i high` and use that identifier in the `--set-enabled` command.

### Step 4 - Install Pacemaker and Tools

Run on: `node1`

```bash
dnf install -y pacemaker pcs
```

This command automatically installs `corosync`, the underlying communication daemon.

### Step 5 - Install Web Server

We will use Apache (`httpd`) as the service protected by the cluster.

Run on: `node1`

```bash
dnf install -y httpd
```

**Critical:** Do **not** start Apache manually. Pacemaker must exclusively control the service lifecycle. Starting it manually causes conflicts. Ensure it does not auto-start at boot:

```bash
systemctl disable httpd
```

### Step 6 - Configure Firewall

Nodes must communicate with each other, and the web server requires port 80 access.

Run on: `node1`

```bash
# Allow cluster communication traffic
firewall-cmd --permanent --add-service=high-availability

# Allow HTTP web traffic
firewall-cmd --permanent --add-service=http

# Apply changes immediately
firewall-cmd --reload

# Verify allowed services
firewall-cmd --list-services
```

Confirm that `high-availability` and `http` appear in the list.

### Step 7 - Start PCS Daemon

`pcsd` is the helper service enabling remote cluster management.

Run on: `node1`

```bash
systemctl enable --now pcsd
systemctl status pcsd
```

The status should indicate `active (running)`. Press `q` to exit the status view.

---

## Part B - Create Node 2

### Step 8 - Clone Node 1

Run on: `node1`

Shut down the VM:
```bash
poweroff
```

In Hyper-V Manager on Windows:
1. Right-click `node1` → **Export...** → Select a destination folder.
2. Wait for the export to complete.
3. Click **Import Virtual Machine...** → Next.
4. Select the exported folder → Next → Choose VM → Next.
5. Select **Copy the virtual machine (create a new unique ID)** → Next.
6. Finish the import. Rename the new VM to `node2`.

**Network Adapter Check:**
Ensure `node2`'s adapters match `node1`:
- Adapter 1: `Default Switch` (maps to `eth0`)
- Adapter 2: `vSwitch-Internal` (maps to `eth1`)

*Note:* If static MAC addresses were configured on `node1`, change them on `node2` to avoid network conflicts.

Start both VMs.

### Step 9 - Configure Node 2 Identity

The clone retains `node1`'s IP and hostname. We must differentiate `node2`.

Run on: `node2` (via Hyper-V console, not SSH)

```bash
# Set unique hostname
hostnamectl set-hostname node2

# Assign unique internal IP
nmcli connection modify eth1 ipv4.addresses 10.10.10.102/24
nmcli connection down eth1
nmcli connection up eth1

# Generate a new Machine ID (critical for clones)
rm -f /etc/machine-id
systemd-machine-id-setup

# Reboot to apply changes
reboot
```

After reboot, verify:
```bash
hostname      # Should return 'node2'
ip -br a      # eth1 should show 10.10.10.102
```

### Step 10 - Name Node 1

Run on: `node1`

```bash
hostnamectl set-hostname node1
```

### Step 11 - Map Hostnames

Clusters rely on names, not IPs. We define these mappings in `/etc/hosts`.

Run on: **both nodes**

```bash
echo "10.10.10.101 node1" >> /etc/hosts
echo "10.10.10.102 node2" >> /etc/hosts
```

Verify:
```bash
cat /etc/hosts
```

*Warning:* Execute these commands only once per node. Duplicate entries can cause confusion.

### Step 12 - Test Inter-Node Connectivity

Run on: `node1`
```bash
ping -c 3 node2
```

Run on: `node2`
```bash
ping -c 3 node1
```

Both must succeed. If they fail, recheck firewall rules (Step 6) and network adapter settings (Step 8).

---

## Part C - Build the Cluster

### Step 13 - Set Cluster User Password

Pacemaker uses the `hacluster` user for inter-node authentication.

Run on: **both nodes**

```bash
passwd hacluster
```

Use the **same strong password** on both nodes. Record it securely.

### Step 14 - Authenticate Nodes

Establish trust between `node1` and `node2`.

Run on: `node1`

```bash
pcs host auth node1 node2 -u hacluster
```

Enter the `hacluster` password when prompted. Expected output:
```text
node1: Authorized
node2: Authorized
```

> **Common Pitfall: HTTP 400 Error**
> If you encounter `HTTP error: 400` or "Unable to connect," it usually means `pcsd` is not running on one of the nodes, or the firewall is blocking cluster traffic.
> **Fix:** Ensure `systemctl status pcsd` shows `active (running)` on **both** nodes, and verify `firewall-cmd --list-services` includes `high-availability` on **both** nodes.

### Step 15 - Create Cluster

Initialize the cluster configuration.

Run on: `node1`

```bash
pcs cluster setup webcluster node1 node2
```

Expected success message: `Cluster has been successfully set up.`

> **Common Pitfall: Leftover Configuration Files**
> If this step fails stating that configuration already exists, previous attempts left residual files.
> **Fix:** Run `pcs cluster destroy --all` on `node1` to wipe configurations from both nodes completely, then retry Step 15.

### Step 16 - Start Cluster

Activate the cluster services.

Run on: `node1`

```bash
# Start services on all nodes
pcs cluster start --all

# Enable auto-start on boot
pcs cluster enable --all
```

### Step 17 - Check Status

Verify the cluster health.

Run on: `node1`

```bash
pcs status
```

Look for:
```text
Node List:
  * Online: [ node1 node2 ]
```

It may take ~20 seconds for both nodes to register as Online. Retry if necessary.

### Step 18 - Disable Fencing (Lab Environment Only)

Production clusters require fencing (STONITH) to prevent split-brain scenarios. Our lab VMs lack hardware fencing capabilities, so we disable this requirement.

Run on: `node1`

```bash
pcs property set stonith-enabled=false
```

Verify:
```bash
pcs property config
```

*Note:* Never disable fencing in production environments.

> **Common Pitfall: Resources Won't Start Due to STONITH Warnings**
> If resources remain stopped with messages about "STONITH required," you skipped this step.
> **Fix:** Run `pcs property set stonith-enabled=false` to allow resources to start without a fence device.

---

## Part D - Add Resources

Resources are the components the cluster manages. Here, we deploy a Floating IP and a Web Server.

### Step 19 - Create Floating IP

This IP (`10.10.10.200`) is not bound to a specific NIC. Pacemaker assigns it to the active node.

Run on: `node1`

```bash
pcs resource create VirtualIP ocf:heartbeat:IPaddr2 \
  ip=10.10.10.200 \
  cidr_netmask=24 \
  nic=eth1 \
  op monitor interval=30s
```

| Parameter | Description |
|---|---|
| `VirtualIP` | Resource name. |
| `ocf:heartbeat:IPaddr2` | Script handling IP assignment/removal. |
| `ip=...` | The floating IP address. |
| `nic=eth1` | Interface where the IP is attached. |
| `op monitor interval=30s` | Health check frequency. |

### Step 20 - Create Web Server Resource

Run on: `node1`

```bash
pcs resource create WebServer systemd:httpd \
  op monitor interval=30s
```

This instructs Pacemaker to manage the standard `httpd` service.

### Step 21 - Group Resources

A group ensures resources run together on the same node and start in a specific order.

Run on: `node1`

```bash
pcs resource group add WebGroup VirtualIP WebServer
```

Check status:
```bash
pcs status
```

You should see:
```text
Resource Group: WebGroup
  * VirtualIP ... Started node1
  * WebServer ... Started node1
```

### Step 22 - Create Test Content

Differentiate the nodes visually to observe failover.

Run on: `node1`
```bash
echo "Hello from node1" > /var/www/html/index.html
```

Run on: `node2`
```bash
echo "Hello from node2" > /var/www/html/index.html
```

### Step 23 - Verify Access

Open a browser on the Windows host and navigate to:
```text
http://10.10.10.200
```

You should see the message from the active node (e.g., `Hello from node1`).

---

## Part E - Testing Failover

Keep the browser tab open. Refresh after each test to observe changes.

### Test 1: Standby Mode

Standby places a node in maintenance mode; it stops accepting new resources but keeps existing ones until moved.

Run on: `node1` (assuming it holds resources)

```bash
# Place node1 in standby
pcs node standby node1

# Check status
pcs status
```

Refresh the browser. You should now see `Hello from node2`. The resources migrated automatically.

To restore `node1`:
```bash
pcs node unstandby node1
```
*Note:* Resources typically stay on `node2` unless forced back. This prevents unnecessary disruption.

### Test 2: Simulated Crash

Force a hard power failure.

1. Identify the active node via `pcs status` (e.g., `node2`).
2. In Hyper-V Manager, right-click `node2` → **Turn Off**.
3. On `node1`, wait 10–30 seconds and run:
   ```bash
   pcs status
   ```
4. Confirm `node2` is OFFLINE and resources started on `node1`.
5. Refresh browser: `Hello from node1`.

Restart `node2` in Hyper-V. It rejoins the cluster automatically due to the earlier `enable --all` setting.

### Test 3: Service Failure

Pacemaker monitors services and restarts them if they fail.

Run on the active node:
```bash
systemctl stop httpd
```

Wait 30–60 seconds. Run `pcs status`. Pacemaker detects the outage and restarts the service. Clear any historical error logs:
```bash
pcs resource cleanup
```

---

## Part F - Understanding Constraints (Advanced)

Groups simplify configuration, but constraints are the underlying logic. Let's deconstruct the group to see how Pacemaker thinks.

| Constraint Type | Purpose | Example Rule |
|---|---|---|
| **Colocation** | Forces resources to run on the same node. | WebServer runs WITH VirtualIP |
| **Ordering** | Dictates startup sequence. | Start VirtualIP THEN WebServer |
| **Location** | Preferences for specific nodes. | Prefer node1 |

### Step 24 - Remove Group

Run on: `node1`

```bash
pcs resource ungroup WebGroup
```

Now `VirtualIP` and `WebServer` are independent. They might drift to different nodes.

### Step 25 - Add Colocation

Force them to stay together.

```bash
pcs constraint colocation add WebServer with VirtualIP INFINITY
```

`INFINITY` implies "strictly always."

### Step 26 - Add Ordering

Ensure the IP exists before the web server binds to it.

```bash
pcs constraint order VirtualIP then WebServer
```

### Step 27 - Add Location Preference

Prefer running on `node1`.

```bash
pcs constraint location VirtualIP prefers node1=100
```

Score `100` indicates a preference. Higher scores override lower ones.

### Step 28 - View Constraints

```bash
pcs constraint config
```

### Step 29 - Test Behavior

1. Check status: Both should be on `node1`.
2. Standby `node1`: `pcs node standby node1`.
3. Check status: Both move to `node2` (colocation enforced).
4. Unstandby `node1`: `pcs node unstandby node1`.
5. Check status: They move **back** to `node1` (location preference honored).

*Optional Stickiness:*
To prevent resources from moving back automatically, increase stickiness:
```bash
pcs resource defaults update resource-stickiness=200
```
(Stickiness 200 > Preference 100 = Stay put).

---

## Part G - Fencing (Conceptual)

*Note: This section explains the necessity of fencing in production. We disabled it earlier for the lab.*

### Why Fencing?

If `node1` loses network connectivity but remains powered on, it believes it still owns the IP. If `node2` takes over the IP simultaneously, both nodes write to the same resource, causing **Split-Brain** and data corruption.

**Fencing** resolves this by forcibly powering off `node1` before `node2` assumes control.

Methods include:
- **Hardware:** IPMI/iLO/iDRAC (Physical servers).
- **Hypervisor:** VMware/KVM APIs (Virtual machines).
- **Storage:** SBD (Shared Block Device messaging).

In our lab, we set `stonith-enabled=false` because we lack hardware fencing devices. **Do not skip fencing in production.**

---

## Command Cheat Sheet

| Task | Command |
|---|---|
| Overall Status | `pcs status` |
| Start/Stop Cluster | `pcs cluster start --all` / `stop --all` |
| Node Maintenance | `pcs node standby <name>` / `unstandby <name>` |
| Move Resource | `pcs resource move <resource> <node>` |
| Clear Move Pref | `pcs resource clear <resource>` |
| Reset Errors | `pcs resource cleanup` |
| Show Constraints | `pcs constraint config` |
| Delete Constraint | `pcs constraint delete <ID>` |
| Toggle Fencing | `pcs property set stonith-enabled=true/false` |

---

## Troubleshooting

| Symptom | Likely Cause | Solution |
|---|---|---|
| `dnf install pacemaker` fails | HA Repo not enabled | Re-execute Step 3. Verify `dnf repolist`. |
| `pcs cluster setup` fails | Leftover config files | Run `pcs cluster destroy --all` on `node1`, then retry Step 15. |
| `pcs host auth` returns HTTP 400 | `pcsd` not running or firewall blocked | Ensure `systemctl status pcsd` is active on **both** nodes. Verify `high-availability` is allowed in firewall on **both** nodes. |
| Nodes cannot ping by name | Missing `/etc/hosts` entries | Re-execute Step 11. Ensure no duplicate lines exist. |
| Resources stuck in "Stopped" | STONITH/Fencing enabled | Run `pcs property set stonith-enabled=false` (Step 18). |
| WebServer shows "Failed" | Manual start conflict | Ensure `systemctl disable httpd` was run (Step 5). Run `pcs resource cleanup`. |
| Browser hangs on Floating IP | Firewall blocking HTTP | Verify `http` is allowed in firewall (Step 6). |
| Duplicate IPs/MACs | Cloning artifact | Regenerate Machine ID on `node2` (Step 9). Check Hyper-V MAC addresses. |

---

## Checklist

- [ ] HA Repo enabled, Pacemaker installed.
- [ ] `httpd` installed but **disabled** from auto-start.
- [ ] Firewall allows `high-availability` and `http`.
- [ ] `node2` cloned with unique IP, Hostname, and Machine-ID.
- [ ] Nodes can ping each other by name.
- [ ] `pcs host auth` successful (no HTTP 400 errors).
- [ ] Cluster created and started (no leftover config errors).
- [ ] Fencing disabled (`stonith-enabled=false`).
- [ ] Resources grouped and running.
- [ ] `http://10.10.10.200` accessible from Windows.
- [ ] Failover tested (Standby and Power-off).
- [ ] Constraints understood (Colocation, Order, Location).