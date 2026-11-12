# 07 - Simple NFS Server Setup

**Prerequisite:** [01 - Hyper-V VM Setup](01-hyperv-vm-setup.md). Both VMs need `eth0` internet and `eth1` on `vSwitch-Internal`.
**Status:** Updated with live lab implementation details.

## Overview
NFS (Network File System) lets one machine share a folder over the network. Other machines mount it and use it like a local folder.

You will build:
*   An NFS server that shares a folder from a dedicated 10GB disk.
*   An NFS client that mounts it and reads/writes files.

```text
       Client                         Server
     10.10.10.101  ---- NFS ---->  10.10.10.200
      /mnt/shared                   /srv/nfs-dbstorage/shared
    (looks local)                 (files really live here)
```

| VM | Role | eth1 IP |
| --- | --- | --- |
| nfs1 | NFS server | 10.10.10.200 |
| node1 | NFS client | 10.10.10.101 |

If you only have one VM, use it as the client and build a second one by cloning.

## How to read this guide
*   Run commands one at a time. Press Enter, read the result, then continue.
*   Every step has a `Run on:` label. Check which VM you are on.
*   Run everything as `root`.
*   Lines after `#` are notes. Do not type them.

---

## Part A - Server Setup (`nfs1`)

### Step 1 - Check the Server
**Run on: nfs1**

Set the hostname:
```bash
hostnamectl set-hostname nfs1
```

Check the addresses:
```bash
ip -br a
```
*Expected:* `eth1` has `10.10.10.200/24`.

Check the internet:
```bash
ping -c 4 8.8.8.8
```

### Step 2 - Install the NFS Tools
**Run on: nfs1**
```bash
dnf install -y nfs-utils
```

### Step 3 - Use a Separate Data Disk
For a real project, a separate disk is better than using the OS disk.

1.  In **Hyper-V Manager**: `nfs1` → Settings → SCSI Controller → Hard Drive → Add → New → VHDX, dynamically expanding, **10 GB**.
2.  Start the VM.

**Run on: nfs1**

Find the new disk (about `10G`, no partitions, normally `sdb`):
```bash
lsblk
```
*Careful:* The next command erases the disk. Do not use `sda` (the OS disk).

Format it:
```bash
mkfs.xfs /dev/sdb
```

Create the mount folder:
```bash
mkdir -p /srv/nfs-dbstorage
```

Get the disk's UUID:
```bash
blkid /dev/sdb
```

Add it to `/etc/fstab`. Replace `PASTE-UUID-HERE` with your UUID (no quotes):
```bash
echo "UUID=PASTE-UUID-HERE /srv/nfs-dbstorage xfs defaults 0 0" >> /etc/fstab
```

Reload and mount:
```bash
systemctl daemon-reload
mount -a
```

Check:
```bash
df -h /srv/nfs-dbstorage
```
*Expected:* `/dev/sdb` mounted on `/srv/nfs-dbstorage` with ~10G space.

### Step 4 - Create the Folder to Share
**Run on: nfs1**

```bash
mkdir -p /srv/nfs-dbstorage/shared
```

Give it permissions. For a simple lab, allow everyone to read and write:
```bash
chmod 777 /srv/nfs-dbstorage/shared
```

Create a test file:
```bash
echo "Hello from the NFS server" > /srv/nfs-dbstorage/shared/hello.txt
```
*Note:* `777` means anyone can read, write and delete. It is fine for learning. Step 12 shows safer permissions.

### Step 5 - Export the Folder
The exports file says which folder is shared, with whom, and how.

**Run on: nfs1**

```bash
echo "/srv/nfs-dbstorage/shared 10.10.10.0/24(rw,sync,no_subtree_check)" > /etc/exports
```

| Part | Meaning |
| --- | --- |
| `/srv/nfs-dbstorage/shared` | The folder to share |
| `10.10.10.0/24` | Only machines on the lab network may connect |
| `rw` | Read and write (use `ro` for read-only) |
| `sync` | Write to disk before replying (safer) |
| `no_subtree_check` | Avoids a common NFS problem |

Check the file:
```bash
cat /etc/exports
```

Apply it:
```bash
exportfs -rav
```
*Expected:* `exporting 10.10.10.0/24:/srv/nfs-dbstorage/shared`.

### Step 6 - Allow SELinux to Share It
**Run on: nfs1**

```bash
setsebool -P nfs_export_all_rw on
```
If the command is not found: `dnf install -y policycoreutils-python-utils`, then try again.

### Step 7 - Start the NFS Service
**Run on: nfs1**

```bash
systemctl enable --now nfs-server
```

Check:
```bash
systemctl status nfs-server
```
Look for `active (exited)` or `active (running)`. Press `q`.

### Step 8 - Open the Firewall
**Run on: nfs1**

```bash
firewall-cmd --permanent --add-service=nfs
firewall-cmd --reload
```

Check the share is active:
```bash
exportfs -v
```
*Expected:* Your folder with `10.10.10.0/24`.

---

## Part B - Client Setup (`node1`)

### Step 9 - Install the NFS Tools and Test the Connection
**Run on: node1**

```bash
dnf install -y nfs-utils
```

Check the server is reachable:
```bash
ping -c 3 10.10.10.200
```

### Step 10 - Mount the Share
**Run on: node1**

Create a mount point (an empty folder):
```bash
mkdir -p /mnt/shared
```

Mount the share:
```bash
mount -t nfs4 10.10.10.200:/srv/nfs-dbstorage/shared /mnt/shared
```

Check:
```bash
df -h /mnt/shared
```
*Expected:* `10.10.10.200:/srv/nfs-dbstorage/shared` mounted on `/mnt/shared`.

Read the server's file:
```bash
cat /mnt/shared/hello.txt
```
*Expected:* `Hello from the NFS server`.

Write a file from the client:
```bash
echo "Hello from the client" > /mnt/shared/from-client.txt
```

**Run on: nfs1**
```bash
ls /srv/nfs-dbstorage/shared
```
*Expected:* Both `hello.txt` and `from-client.txt`. The file written on the client really lives on the server.

Unmount when you are done: `umount /mnt/shared`

---

## Part C - Make It Permanent and Safer

### Step 11 - Mount Automatically at Boot (Client)
**Run on: node1**

```bash
echo "10.10.10.200:/srv/nfs-dbstorage/shared /mnt/shared nfs4 defaults,_netdev 0 0" >> /etc/fstab
```
`_netdev` tells Linux to wait for the network before mounting.

**Important:** Ensure there is a new line between the last entry in `/etc/fstab` and this new line. If they are merged, `mount -a` will fail with a "parse error".

Test the entry without rebooting. Unmount first:
```bash
umount /mnt/shared
```

Then mount everything in `fstab`:
```bash
mount -a
```

Check:
```bash
df -h /mnt/shared
```

### Step 12 - Safer Permissions (Optional)
`chmod 777` is convenient but unsafe. A better way is to give the folder to one user or group, using numeric IDs that match on both machines.

**Run on: nfs1**
Make a group with a fixed ID:
```bash
groupadd -g 5000 nfsusers
```
Give the folder to that group:
```bash
chown root:nfsusers /srv/nfs-dbstorage/shared
```
Allow group members only:
```bash
chmod 2770 /srv/nfs-dbstorage/shared
```

**Run on: node1**
Create the same group with the same ID:
```bash
groupadd -g 5000 nfsusers
```
Add a user to it (replace `alice` with a real user):
```bash
usermod -aG nfsusers alice
```
The numeric ID (`5000`) must be identical on server and client. NFS matches people by number, not by name.

### Step 13 - Understand Root Squash
By default NFS maps `root` on the client to a harmless user ("root squash"). So `root` on the client cannot do everything on the share. This is a security feature.

Only if you really need it (not recommended), add `no_root_squash` to the export options in `/etc/exports`:
```text
/srv/nfs-dbstorage/shared 10.10.10.0/24(rw,sync,no_subtree_check,no_root_squash)
```
Then run `exportfs -rav`.

---

## Useful Commands

| Task | Command | Run on |
| --- | --- | --- |
| List shares being exported | `exportfs -v` | server |
| Reload `/etc/exports` | `exportfs -rav` | server |
| Is NFS running? | `systemctl status nfs-server` | server |
| Show mounted NFS shares | `mount | grep nfs` | client |
| Unmount | `umount /mnt/shared` | client |
| Mount everything in fstab | `mount -a` | client |
| NFS statistics | `nfsstat` | either |

## Troubleshooting

| Problem | Likely cause | Fix |
| --- | --- | --- |
| `mount.nfs4: Connection timed out` | Cannot reach server or firewall blocking | `ping 10.10.10.200`; redo Step 8 |
| `mount.nfs4: access denied by server` | Client not allowed in `/etc/exports` | Check the subnet in Step 5, run `exportfs -rav` |
| `No such file or directory` | Wrong path or folder missing | Compare the path with `exportfs -v` on the server |
| `Permission denied` when writing | Folder permissions | Check `ls -ld /srv/nfs-dbstorage/shared`; see Steps 4 and 12 |
| Files show owner `nobody` | Numeric ID mismatch or root squash | See Steps 12 and 13 |
| Mount hangs at boot | `_netdev` missing | Add `_netdev` in fstab (Step 11) |
| After editing `/etc/exports`, nothing changes | Export not reloaded | `exportfs -rav` |
| `Stale file handle` | Server folder replaced or re-exported | `umount -f /mnt/shared`, then mount again |
| `parse error at line X` in fstab | Two lines merged together | Open `/etc/fstab` and ensure each entry is on its own line |

## Checklist
- [ ] `nfs-utils` installed on server and client
- [ ] `/srv/nfs-dbstorage/shared` created on the server
- [ ] `exportfs -v` shows the share for `10.10.10.0/24`
- [ ] `nfs-server` is running and the `nfs` firewall service is open
- [ ] Client can mount the share and read `hello.txt`
- [ ] A file created on the client appears on the server
- [ ] (Optional) The `fstab` entry mounts at boot
- [ ] (Optional) Permissions use a group with the same numeric ID on both machines

**Next:** [P1 - MySQL HA Lab](P1-mysql-ha-nfs-pacemaker.md)
```
