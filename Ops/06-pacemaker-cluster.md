# 06 - Simple Pacemaker Cluster
> **Prerequisite:** [01 - Hyper-V VM Setup](01-hyperv-vm-setup.md) completed (one working VM with `eth0` internet and `eth1` = `10.10.10.101`).

## Overview

This guide builds a small **2-node high-availability (HA) cluster** so you can see how Pacemaker works. You will:

- Create two VMs, `node1` and `node2`.
- Give the cluster a **floating IP** (`10.10.10.200`) and a **web server**.
- Break a node on purpose and watch the service **move automatically** to the other node.

```
                 Windows PC (browser)
                       |
              http://10.10.10.200   <-- floating IP (moves between nodes)
                       |
        +--------------+--------------+
        |                             |
     node1                         node2
  10.10.10.101                  10.10.10.102
   (eth1)                         (eth1)
```

### Key words (read once)

| Term | Simple meaning |
|---|---|
| **Cluster** | A group of computers (nodes) working together |
| **Node** | One computer/VM in the cluster |
| **Corosync** | Lets the nodes talk to each other and know who is alive |
| **Pacemaker** | The "manager" that decides which node runs which service |
| **pcs** | The command-line tool we use to control Pacemaker |
| **Resource** | Something the cluster manages (an IP, a web server) |
| **Resource group** | Resources that must always run together on the same node |
| **Failover** | Moving a service to another node when one fails |
| **Quorum** | The "majority vote" that decides if the cluster may run |
| **Fencing (STONITH)** | Forcibly shutting down a broken node. We turn it **off** in this lab |

### How to read this guide

- Run commands **one at a time**. Press **Enter**, read the result, then continue.
- Every step starts with a **Run on:** label. Always check which node you are on.
- Lines after `#` are notes for you. Do not type them.
- Run everything as `root`.

> **Run on: tags used in this guide**
>
> - **node1** = only the first VM
> - **node2** = only the second VM
> - **both nodes** = run the same command on each VM, one after the other

## Prerequisites

- The working VM from guide 01 (this becomes `node1`)
- Internet working on `eth0` (`ping -c 4 8.8.8.8` replies)
- About 20 GB free disk on the Windows host for the second VM
- 2 GB RAM per VM (4 GB recommended)

---

# Part A - Prepare node1 (before cloning)

We install everything on `node1` **first**, then clone it. This way you install only once.

## Step 1 - Check the Internet

**Run on: node1**

```bash
ping -c 4 8.8.8.8
```

You must get replies. If not, fix guide 01 first.

## Step 2 - Update the System

**Run on: node1**

```bash
dnf update -y
```

This may take a few minutes.

## Step 3 - Enable the High Availability Repository

Pacemaker is not in the default repositories. It lives in a separate one called **HighAvailability**.

**Run on: node1**

Install the tool that manages repositories:

```bash
dnf install -y dnf-plugins-core
```

Enable the repository:

```bash
dnf config-manager --set-enabled highavailability
```

Check that it is enabled:

```bash
dnf repolist
```

Expected: a line containing `highavailability` in the list.

> **Not listed?** Find the exact name with `dnf repolist --all | grep -i high`, then use that name in the `--set-enabled` command.

## Step 4 - Install Pacemaker

**Run on: node1**

```bash
dnf install -y pacemaker pcs
```

This also installs `corosync` automatically.

## Step 5 - Install the Web Server

We use Apache (`httpd`) as the service the cluster will protect.

**Run on: node1**

```bash
dnf install -y httpd
```

> **Important:** do **not** start or enable `httpd` yourself. Pacemaker must be the only one that starts and stops it. If you start it manually, the cluster will get confused.

Make sure it will not start at boot:

```bash
systemctl disable httpd
```

## Step 6 - Open the Firewall

The nodes need to talk to each other, and the web server needs port 80.

**Run on: node1**

Allow cluster traffic:

```bash
firewall-cmd --permanent --add-service=high-availability
```

Allow web traffic:

```bash
firewall-cmd --permanent --add-service=http
```

Apply the changes:

```bash
firewall-cmd --reload
```

Check:

```bash
firewall-cmd --list-services
```

Expected: the list includes `high-availability` and `http`.

## Step 7 - Start the pcs Service

`pcsd` is the helper service that lets the nodes be managed together.

**Run on: node1**

```bash
systemctl enable --now pcsd
```

Check that it is running:

```bash
systemctl status pcsd
```

Look for `active (running)`. Press `q` to leave the screen.

---

# Part B - Create node2 and Name the Nodes

## Step 8 - Clone node1 into node2

**Run on: node1**

Shut the VM down:

```bash
poweroff
```

Then in **Hyper-V Manager** on Windows:

1. Right-click `node1` VM → **Export…** → choose a folder (for example `C:\HyperV-Export`) → **Export**.
2. Wait until it finishes.
3. In the right-hand menu click **Import Virtual Machine…** → **Next**.
4. Select the exported folder → **Next** → choose the VM → **Next**.
5. Choose **Copy the virtual machine (create a new unique ID)** → **Next**.
6. Keep the default folders → **Next** → **Finish**.
7. Rename the imported VM to `node2` (right-click → **Rename**).
8. Check the two network adapters on `node2` (Settings → Network Adapter). Order must be the same as `node1`:
   - first adapter = `Default Switch` (becomes `eth0`)
   - second adapter = `vSwitch-Internal` (becomes `eth1`)

> **Same MAC address warning:** if you set a fixed MAC address on `node1`'s adapters earlier, change it on `node2` (Settings → Network Adapter → Advanced Features → MAC address). Two machines must never share a MAC address.

Now **start both VMs**.

## Step 9 - Give node2 Its Own Identity

The clone is an exact copy, so it has `node1`'s IP and ID. Fix that on `node2`.

> **Do this on the Hyper-V console of `node2`**, not over SSH. Do **not** do it on `node1`.

**Run on: node2**

Set the hostname:

```bash
hostnamectl set-hostname node2
```

Change the internal IP (`eth1`) to a different address:

```bash
nmcli connection modify eth1 ipv4.addresses 10.10.10.102/24
```

Apply the change:

```bash
nmcli connection down eth1
```

```bash
nmcli connection up eth1
```

Generate a new unique machine ID (the clone has the same one as `node1`):

```bash
rm -f /etc/machine-id
```

```bash
systemd-machine-id-setup
```

Restart the VM:

```bash
reboot
```

After it restarts, verify on `node2`:

```bash
hostname
```

Expected: `node2`

```bash
ip -br a
```

Expected: `eth1` shows `10.10.10.102/24`, and `eth0` shows a `172.x.x.x` address.

## Step 10 - Name node1

**Run on: node1**

```bash
hostnamectl set-hostname node1
```

Check:

```bash
hostname
```

Expected: `node1`

## Step 11 - Let the Nodes Find Each Other by Name

The cluster uses names, not IPs. We teach both VMs the names using the `/etc/hosts` file.

**Run on: both nodes** (repeat all three commands on `node1`, then on `node2`)

```bash
echo "10.10.10.101 node1" >> /etc/hosts
```

```bash
echo "10.10.10.102 node2" >> /etc/hosts
```

Check the file:

```bash
cat /etc/hosts
```

Expected: the last two lines show `node1` and `node2` with their IPs.

> **Only run the `echo` commands once per node.** Running them twice adds duplicate lines. If that happens, open the file with `vi /etc/hosts` and delete the extras.

## Step 12 - Test the Connection Between Nodes

**Run on: node1**

```bash
ping -c 3 node2
```

**Run on: node2**

```bash
ping -c 3 node1
```

Both must show replies. If not, check `ip -br a`, the firewall, and that both VMs use the same `vSwitch-Internal` switch.

---

# Part C - Build the Cluster

## Step 13 - Set the Cluster User Password

The package created a special user called `hacluster`. The nodes use it to trust each other.

**Run on: both nodes**

```bash
passwd hacluster
```

Type a password twice (you will not see it while typing). **Use the exact same password on both nodes.** Write it down.

## Step 14 - Make the Nodes Trust Each Other

**Run on: node1 only**

```bash
pcs host auth node1 node2 -u hacluster
```

Enter the `hacluster` password when asked.

Expected:

```
node1: Authorized
node2: Authorized
```

## Step 15 - Create the Cluster

**Run on: node1 only**

```bash
pcs cluster setup webcluster node1 node2
```

This creates a cluster named `webcluster`. Expected last line: `Cluster has been successfully set up.`

## Step 16 - Start the Cluster

**Run on: node1 only**

Start the cluster on both nodes:

```bash
pcs cluster start --all
```

Make it start automatically at boot:

```bash
pcs cluster enable --all
```

## Step 17 - Check the Cluster Status

**Run on: node1**

```bash
pcs status
```

Look for these lines:

```
Cluster name: webcluster
...
Node List:
  * Online: [ node1 node2 ]
```

Both nodes must say **Online**.

> It can take about 20 seconds for both nodes to appear. Wait and run `pcs status` again.

## Step 18 - Turn Off Fencing (Lab Only)

By default Pacemaker refuses to run resources without a fencing device. Real servers need fencing; our lab VMs do not have one.

**Run on: node1 only**

```bash
pcs property set stonith-enabled=false
```

Check:

```bash
pcs property config
```

Expected: `stonith-enabled: false`

> **Never do this on a real production cluster.** Fencing protects your data from two nodes writing to it at once. You will turn it back on and test it in Part G.

---

# Part D - Add the Resources

## Step 19 - Create the Floating IP

This IP is not tied to one machine. Pacemaker puts it on whichever node is healthy.

**Run on: node1 only**

```bash
pcs resource create VirtualIP ocf:heartbeat:IPaddr2 ip=10.10.10.200 cidr_netmask=24 nic=eth1 op monitor interval=30s
```

| Part | Meaning |
|---|---|
| `VirtualIP` | The name we give the resource |
| `ocf:heartbeat:IPaddr2` | The "resource agent" (script) that adds/removes an IP |
| `ip=10.10.10.200` | The floating IP |
| `cidr_netmask=24` | Same as `255.255.255.0` |
| `nic=eth1` | Put the IP on the internal adapter |
| `op monitor interval=30s` | Check every 30 seconds that it is still working |

Check:

```bash
pcs status
```

Expected: `VirtualIP (ocf:heartbeat:IPaddr2): Started node1` (or `node2`).

## Step 20 - Create the Web Server Resource

**Run on: node1 only**

```bash
pcs resource create WebServer systemd:httpd op monitor interval=30s
```

`systemd:httpd` means "manage the normal `httpd` service".

## Step 21 - Group the Two Resources

A group keeps resources together on the same node and starts them in order (IP first, then web server).

**Run on: node1 only**

```bash
pcs resource group add WebGroup VirtualIP WebServer
```

Check:

```bash
pcs status
```

Expected:

```
Resource Group: WebGroup
  * VirtualIP (ocf:heartbeat:IPaddr2): Started node1
  * WebServer (systemd:httpd): Started node1
```

Both must say **Started** on the **same node**.

## Step 22 - Create a Test Web Page

Each node shows a different message, so you can see which node is answering.

**Run on: node1**

```bash
echo "Hello from node1" > /var/www/html/index.html
```

**Run on: node2**

```bash
echo "Hello from node2" > /var/www/html/index.html
```

## Step 23 - Test from Windows

On your **Windows PC** open a browser and go to:

```
http://10.10.10.200
```

You should see the message of the node that currently runs the resources (for example `Hello from node1`).

You can also test from Windows Command Prompt:

```
curl http://10.10.10.200
```

---

# Part E - Break Things and Watch the Cluster React

Keep a browser tab open on `http://10.10.10.200` and refresh it after each test. Use `pcs status` to watch.

## Test 1 - Move Everything to the Other Node (Standby)

Standby tells a node "do not run any resources".

**Run on: node1**

Find which node is running the resources:

```bash
pcs status
```

Say it is `node1`. Put it in standby:

```bash
pcs node standby node1
```

Check:

```bash
pcs status
```

Expected: `node1` is `standby`, and `WebGroup` is now **Started node2**. Refresh the browser: it now says `Hello from node2`.

Bring `node1` back:

```bash
pcs node unstandby node1
```

> The resources stay on `node2`. Pacemaker does not move them back unless told to. This is normal and avoids needless downtime.

## Test 2 - Crash a Node

Simulate a power failure.

Find the node that runs the resources (`pcs status`). In **Hyper-V Manager**, right-click that VM → **Turn Off** (not Shut Down).

On the **other** node run:

```bash
pcs status
```

Expected after about 10-30 seconds:

- The dead node shows as **OFFLINE**.
- `WebGroup` is **Started** on the surviving node.
- The browser page still works, showing the surviving node's message.

Start the powered-off VM again. Because you ran `pcs cluster enable --all`, it rejoins the cluster by itself. Check with `pcs status`.

## Test 3 - Crash Only the Web Server

Pacemaker checks the service every 30 seconds and repairs it.

**Run on: the node that runs the resources**

```bash
systemctl stop httpd
```

Now watch:

```bash
pcs status
```

Run it again after 30-60 seconds. Pacemaker notices the service is down and restarts it. You may see a `Failed Resource Actions` section.

Clear the old failure message:

```bash
pcs resource cleanup
```

## Test 4 - Move a Resource Manually

**Run on: any node**

```bash
pcs resource move WebGroup node2
```

Check:

```bash
pcs status
```

Remove the manual preference afterwards, so Pacemaker is free to decide again:

```bash
pcs resource clear WebGroup
```

---

# Part F - Constraints (Replace the Group)

A group is a shortcut. **Constraints** are the full set of rules behind it. Here we remove the group and rebuild the same behaviour rule by rule, so you can see how Pacemaker decides.

| Rule type | Question it answers | Our rule |
|---|---|---|
| **Colocation** | Which resources must (or must not) run on the same node? | The web server runs with the IP |
| **Ordering** | Which starts first? | IP first, then the web server |
| **Location** | Which node is preferred? | Prefer `node1` |

## Step 24 - Remove the Group

Ungrouping removes the group only. Your two resources stay.

**Run on: node1**

```bash
pcs resource ungroup WebGroup
```

Check:

```bash
pcs status
```

Expected: `VirtualIP` and `WebServer` are now listed separately. They may even be on different nodes now, because nothing links them any more.

## Step 25 - Add the Colocation Rule

"Run `WebServer` on the same node as `VirtualIP`."

**Run on: node1**

```bash
pcs constraint colocation add WebServer with VirtualIP INFINITY
```

`INFINITY` means "always, no exceptions".

## Step 26 - Add the Ordering Rule

"Start `VirtualIP` first, then `WebServer`."

**Run on: node1**

```bash
pcs constraint order VirtualIP then WebServer
```

## Step 27 - Add a Location Preference

"Prefer `node1` when it is healthy."

**Run on: node1**

```bash
pcs constraint location VirtualIP prefers node1=100
```

The number is a **score**. Higher means a stronger preference. `INFINITY` would mean "must", and a negative number would mean "avoid".

## Step 28 - View the Constraints

**Run on: node1**

```bash
pcs constraint config
```

Expected: one colocation, one order and one location rule.

For the rule IDs (needed to delete a rule):

```bash
pcs constraint config --full
```

## Step 29 - Test the Rules

**Run on: node1**

Check where things run:

```bash
pcs status
```

`VirtualIP` and `WebServer` should both be on `node1` (your preference).

Put `node1` in standby:

```bash
pcs node standby node1
```

```bash
pcs status
```

Expected: both resources moved together to `node2`.

Bring `node1` back:

```bash
pcs node unstandby node1
```

```bash
pcs status
```

Expected: after a few seconds both resources **move back to `node1`**. With the group they stayed on `node2`. The difference is your location preference (score 100).

> **Optional - make resources "sticky":** to stop the move-back, give resources a stickiness score higher than your preference:
>
> ```bash
> pcs resource defaults update resource-stickiness=200
> ```
>
> Pacemaker adds the scores up. 200 (stay here) beats 100 (prefer node1), so resources stay put. To undo it, run `pcs resource defaults update resource-stickiness=0`.
>
> The `update` syntax is for newer `pcs` versions. If the command is rejected, check `pcs resource defaults --help`.

## Step 30 - Remove a Constraint and See What Changes

Take the IDs from `pcs constraint config --full`, then delete the colocation rule:

```bash
pcs constraint delete ID_OF_THE_COLOCATION_RULE
```

Now put the node running the resources in standby and back, and watch `pcs status`. Without the colocation rule, the two resources can end up on different nodes. Add the rule back afterwards (Step 25).

---

# Part G - Fencing (STONITH)

> **Read this first.** This part is a **learning exercise** with a fake fence device. It was written from documentation and has **not been tested on AlmaLinux 10**, so some command output may differ. Never use a fake fence device on a real cluster.

## Step 31 - Why Fencing Exists

If a node stops answering, Pacemaker cannot know if it **crashed** or is just **cut off from the network** but still running. If it is still running and Pacemaker starts the service on the other node too, both nodes use the same data at once (called **split-brain**) and can corrupt it.

Fencing solves this: before taking over, the cluster **forcibly powers off** the silent node and only then moves the service.

| Fence method | How it powers off the node | Where used |
|---|---|---|
| IPMI / iLO / iDRAC (`fence_ipmilan`) | Server management card | Physical servers |
| Hypervisor (`fence_vmware_soap`, `fence_virsh`) | Asks the hypervisor to stop the VM | Virtual machines |
| Smart power switch (`fence_apc`) | Cuts the power outlet | Physical servers |
| SBD (storage or watchdog based) | Node reboots itself or is told to via a shared disk | VMs and shared-storage clusters |
| Fake device (`fence_dummy`) | Does nothing real, only records "off" | **Lab learning only** |

Hyper-V has no common ready-made fence agent for this lab, so we practice with the fake one.

## Step 32 - Install and List the Fence Agents

**Run on: both nodes**

```bash
dnf install -y fence-agents-all
```

**Run on: node1**

List the available fence agents:

```bash
pcs stonith list
```

Scroll through it. These are the real fence methods from the table above.

Check for the fake test agent:

```bash
pcs stonith list | grep -i dummy
```

- **A line with `fence_dummy` appears:** continue to Step 33.
- **Nothing appears:** the test agent is not shipped on your system. You cannot do the hands-on test here. Read Steps 31 and 36, then skip to the checklist.

## Step 33 - Create the Test Fence Device

**Run on: node1 only**

```bash
pcs stonith create test-fence fence_dummy pcmk_host_list="node1 node2"
```

| Part | Meaning |
|---|---|
| `test-fence` | The name of the fence device |
| `fence_dummy` | The fake agent (records "off" in a file, powers nothing off) |
| `pcmk_host_list` | Nodes this device is allowed to fence |

## Step 34 - Turn Fencing Back On

**Run on: node1 only**

```bash
pcs property set stonith-enabled=true
```

Check:

```bash
pcs status
```

Expected: `test-fence (stonith:fence_dummy): Started node1` (or `node2`) under the resources, and the earlier resources still running.

## Step 35 - Fence a Node by Hand

**Run on: node1**

Ask the cluster to fence `node2`:

```bash
pcs stonith fence node2
```

Check the result:

```bash
pcs status
```

```bash
pcs stonith history
```

Expected: the history shows a fencing action against `node2`, and the cluster treats `node2` as down.

**What to notice:** `node2` is **still running** in Hyper-V, because the fake device powers nothing off. A real device would have cut its power. This is exactly why real fencing needs a real device.

**Recover the lab** (best effort):

**Run on: node2**

```bash
pcs cluster stop
```

```bash
pcs cluster start
```

**Run on: node1**

```bash
pcs resource cleanup
```

```bash
pcs status
```

Expected: both nodes **Online** again. If node2 stays stuck, run `pcs cluster stop --all` then `pcs cluster start --all` on node1.

## Step 36 - Go Back to the Lab Setup (Optional)

To continue practising without fencing, turn it off again:

**Run on: node1 only**

```bash
pcs property set stonith-enabled=false
```

Remember: **real clusters always keep fencing on**. Your next project is a real fence method, such as SBD with a shared disk, or a hypervisor-based agent if you move the lab to KVM/libvirt or VMware.

---

# Understanding What You Built

| What happened | Who did it |
|---|---|
| Nodes noticed each other going up or down | **Corosync** |
| Decided "run the IP and web server on node2" | **Pacemaker** |
| Added the IP and started `httpd` | **Resource agents** |
| Checked every 30 seconds that things still work | `op monitor` |
| Kept IP and web server together | **Resource group** |

Useful viewing commands:

| Command | Shows |
|---|---|
| `pcs status` | Overall cluster, nodes, resources |
| `pcs resource config` | How each resource is set up |
| `pcs cluster status` | Cluster daemons' state |
| `pcs quorum status` | Quorum (voting) information |
| `corosync-cfgtool -s` | Network links between the nodes |

## Why 2-node clusters are special

Quorum needs a **majority**. With two nodes, one failing leaves 1 of 2, which is not a majority. `pcs cluster setup` automatically turns on a special "two node" mode so the survivor can continue. Real clusters usually use 3 or more nodes, or a fencing/quorum device.

---

# Command Cheat Sheet

| Task | Command |
|---|---|
| See everything | `pcs status` |
| Start / stop cluster on all nodes | `pcs cluster start --all` / `pcs cluster stop --all` |
| Stop cluster on this node only | `pcs cluster stop` |
| Put node in standby / out | `pcs node standby node1` / `pcs node unstandby node1` |
| Move a group | `pcs resource move WebGroup node2` |
| Remove the move preference | `pcs resource clear WebGroup` |
| Clear failure messages | `pcs resource cleanup` |
| Disable / enable a resource | `pcs resource disable WebServer` / `pcs resource enable WebServer` |
| Delete a resource | `pcs resource delete WebServer` |
| Show constraints (with IDs) | `pcs constraint config --full` |
| Delete a constraint | `pcs constraint delete ID` |
| List fence agents | `pcs stonith list` |
| Fence a node manually | `pcs stonith fence node2` |
| Fencing history | `pcs stonith history` |
| Fencing on / off | `pcs property set stonith-enabled=true` / `false` |

---

# Troubleshooting

| Problem | Likely cause | Fix |
|---|---|---|
| `dnf install pacemaker pcs` says "no match" | HA repository not enabled | Redo Step 3 |
| `pcs host auth` fails | `pcsd` not running, wrong password, or firewall | `systemctl status pcsd`; check Step 6 and Step 13 |
| `node2: Unable to connect` | Names not resolving or firewall blocking | Redo Steps 11 and 12 |
| Only one node is **Online** | Other node not started | On that node: `pcs cluster start` |
| Node stays `UNCLEAN` or `pending` | Network between nodes broken | `ping node1` / `ping node2`; check `eth1` |
| Resources show `Stopped` | Fencing still on, or earlier failure | Redo Step 18, then `pcs resource cleanup` |
| `WebServer` shows `Failed` | `httpd` was started manually, or the config is wrong | `systemctl stop httpd`, run `pcs resource cleanup` |
| Browser cannot open `10.10.10.200` | Wrong node group state or firewall | `pcs status` shows resources Started; check Step 6 (`http`) |
| Both nodes show the same `172.x` on `eth0` | Duplicate machine ID/MAC from clone | Redo Step 9 (machine ID) and check MAC in Hyper-V |
| Everything stopped after reboot | Cluster not enabled at boot | `pcs cluster enable --all` |
| Resources land on different nodes after Part F | Colocation rule missing | Redo Step 25 and check `pcs constraint config` |
| Resources stop after turning fencing on | No working fence device | Check `pcs status` for `test-fence`; redo Step 33, or set `stonith-enabled=false` |
| `fence_dummy` not in `pcs stonith list` | Agent not shipped on this system | Skip the hands-on fencing test (Step 32) |

## Start Over (Delete the Cluster)

If you want to rebuild from scratch:

**Run on: node1 only**

```bash
pcs cluster destroy --all
```

Then go back to **Step 13** (the `hacluster` password stays) and continue.

---

# Checklist

- [ ] HighAvailability repository enabled, `pacemaker` and `pcs` installed
- [ ] `httpd` installed but **not** enabled or started by hand
- [ ] Firewall allows `high-availability` and `http`
- [ ] `node2` cloned, with hostname `node2` and IP `10.10.10.102`
- [ ] `node1` and `node2` can ping each other by name
- [ ] `pcs host auth` shows both nodes **Authorized**
- [ ] `pcs status` shows both nodes **Online**
- [ ] `stonith-enabled` is `false` (lab only)
- [ ] `WebGroup` is **Started** on one node
- [ ] `http://10.10.10.200` works from Windows
- [ ] Standby test moved the service to the other node
- [ ] Power-off test failed over automatically
- [ ] `httpd` stop test was repaired by Pacemaker
- [ ] Group removed and rebuilt with colocation, order and location constraints
- [ ] Resources move back to `node1` after standby (location preference works)
- [ ] Fake fence device created and `pcs stonith fence node2` recorded in history (or Part G read and understood)
- [ ] You can explain why real clusters must keep fencing enabled

## What to Learn Next - (what we can add next)

1. **Real fencing**: SBD with a shared disk, or a hypervisor fence agent (KVM/libvirt or VMware).
2. **A third node** or a quorum device.
3. **Shared storage**: DRBD or a shared disk for databases.
4. **Load balancing**: HAProxy on the floating IP with two web servers behind it, kept alive by Pacemaker.
