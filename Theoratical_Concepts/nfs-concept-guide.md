# NFS Beginner Guide (Linux)

## 1. What is NFS?

**NFS (Network File System)** lets one machine (the **server**) share a folder over the network, and other machines (the **clients**) mount it and use it like a local folder.

```
   NFS SERVER                          NFS CLIENT
 +----------------+                  +------------------+
 | /srv/shared    |  <--- network -> | /mnt/data        |
 | (real files)   |     port 2049    | (mount point)    |
 +----------------+                  +------------------+
   "exports" folder                    "mounts" folder
```

- Server **exports** a folder (listed in `/etc/exports`).
- Client **mounts** it onto any empty local folder.

---

## 2. Do folder names have to match?

**No.** The server folder and the client mount point can have completely different names.

```
Server:  /srv/shared      <-- real folder
Client:  /mnt/anything    <-- any empty folder you create
```

Mount command links them:
```bash
sudo mount 192.168.1.10:/srv/shared /mnt/anything
```

---

## 3. Permissions: the most important part

### 3.1 Linux permission basics

```
 -rwxr-x---  1  alice  devs  4096  file.txt
 |\_/\_/\_/     |      |
 | |  |  |      |      +-- GROUP owner
 | |  |  |      +--------- USER (owner)
 | |  |  +---------------- OTHERS  (r-x?  here: ---)
 | |  +------------------- GROUP   (r-x)
 | +---------------------- OWNER   (rwx)
 +------------------------ file type (- file, d directory)

 r = read (4)   w = write (2)   x = execute/enter (1)

 rwx r-x ---  =  7 5 0  =  chmod 750
```

### 3.2 UID and GID: do they have to match?

**YES. This is the #1 thing to understand.**

NFS (default `sec=sys`) does **not** send usernames. It sends **numbers**: UID (user ID) and GID (group ID). The server checks permissions using those numbers only.

> Names don't matter. **Numbers must match** on server and client.

```
 CLIENT                         SERVER
 user: bob  UID=1001   ----->   sees UID 1001
                                 |
                                 +--> on server UID 1001 = "alice"
                                      so bob's files appear owned by alice!
```

**Good (matching):**
```
 Server: alice  UID=1001 GID=1001
 Client: alice  UID=1001 GID=1001   --> works, same person
```

**Bad (mismatch):**
```
 Server: alice  UID=1001
 Client: alice  UID=1002   --> server sees 1002, a different user
                              --> "Permission denied" or wrong owner
```

Check your numbers on **both** machines:
```bash
id alice
# uid=1001(alice) gid=1001(alice) groups=1001(alice),1005(devs)
```

---

## 4. Sharing with a group (recommended approach)

Goal: several users on different machines share one folder.

### Step 1: Create the SAME group (same GID) on server and client
```bash
# run on BOTH machines
sudo groupadd -g 2000 nfsgroup
```

### Step 2: Create/adjust users with SAME UID, and add to group
```bash
# new user, run on BOTH machines
sudo useradd -m -u 1500 -G nfsgroup alice

# existing user: add to group
sudo usermod -aG nfsgroup alice
```

> Changing an existing user's UID: `sudo usermod -u 1500 alice` (then fix their file ownership with `chown`). Do this carefully.

### Step 3: Set up the shared folder on the SERVER
```bash
sudo mkdir -p /srv/shared
sudo chown root:nfsgroup /srv/shared
sudo chmod 2770 /srv/shared
```

`2770` explained:
```
 2   7   7   0
 |   |   |   +-- others: no access
 |   |   +------ group:  rwx
 |   +---------- owner:  rwx
 +-------------- setgid: new files inherit group "nfsgroup"
```

### Final picture
```
        SERVER                              CLIENT
  group nfsgroup  GID 2000            group nfsgroup  GID 2000
  alice  UID 1500 (in nfsgroup)       alice  UID 1500 (in nfsgroup)
        |                                    |
   /srv/shared  root:nfsgroup 2770  <--> /mnt/data
```

---

## 5. Setup steps (Ubuntu/Debian)

### On the SERVER
```bash
sudo apt update
sudo apt install nfs-kernel-server

# edit exports
sudo nano /etc/exports
```
Add a line:
```
/srv/shared  192.168.1.0/24(rw,sync,no_subtree_check)
```
Apply:
```bash
sudo exportfs -ra        # reload exports
sudo exportfs -v         # verify
sudo systemctl enable --now nfs-kernel-server
```
Open firewall (if used):
```bash
sudo ufw allow from 192.168.1.0/24 to any port 2049
```

### On the CLIENT
```bash
sudo apt install nfs-common
sudo mkdir -p /mnt/data
sudo mount 192.168.1.10:/srv/shared /mnt/data
df -h                    # confirm it's mounted
```

### Make it permanent (survive reboot)
Add to `/etc/fstab` on the client:
```
192.168.1.10:/srv/shared  /mnt/data  nfs  defaults,_netdev  0  0
```
Test: `sudo mount -a`

---

## 6. Two layers of permissions

Access must pass **both**. The stricter one wins.

```
 Client request
      |
      v
 [Layer 1] /etc/exports   -> allowed IP? rw or ro? squash rules?
      |
      v
 [Layer 2] File permissions on server (owner/group/others + UID/GID)
      |
      v
   ACCESS GRANTED / DENIED
```

---

## 7. Common export options

| Option | Meaning |
|---|---|
| `rw` / `ro` | Read-write / read-only |
| `sync` | Write to disk before replying (safer) |
| `no_subtree_check` | Avoids subtle issues, recommended |
| `root_squash` | **Default.** Client root becomes `nobody` on server (safe) |
| `no_root_squash` | Client root stays root on server (risky) |
| `all_squash` | Everyone is mapped to `nobody` (or `anonuid`/`anongid`) |
| `anonuid=` `anongid=` | UID/GID used for squashed users |

### Shortcut when you don't want to match UIDs
Map every client user to one server user:
```
/srv/shared  192.168.1.0/24(rw,sync,all_squash,anonuid=1500,anongid=2000)
```
```
 Any client user  -->  appears as UID 1500 / GID 2000 on server
```
Simple, but you lose per-user ownership.

---

## 8. Troubleshooting

| Problem | Likely cause / fix |
|---|---|
| `Permission denied` | UID/GID mismatch, or group/permissions too strict on server folder |
| Files show owner as a number (e.g. `1002`) | That UID has no name on this machine; make UIDs match |
| Files show `nobody:nogroup` | Squashing (`root_squash`/`all_squash`) or idmap issue |
| Can't mount / timeout | Firewall, server not running, or wrong IP/path |
| `access denied by server` | Client IP not allowed in `/etc/exports` |
| `Stale file handle` | Folder was deleted/re-exported; unmount and remount |
| Root can't write | `root_squash` is working as intended; use a normal user |

Useful commands:
```bash
showmount -e SERVER_IP      # list exports (from client)
exportfs -v                 # show active exports (on server)
id username                 # check UID/GID
mount | grep nfs            # see mounted NFS shares
```

---

## 9. Quick summary

- Folder **names** do **not** need to match.
- **UID and GID numbers must match** on server and client (usernames/group names don't matter).
- Use a **shared group with the same GID** and `chmod 2770` on the folder for team sharing.
- Permissions are checked in **two layers**: `/etc/exports` and file permissions.
- Keep `root_squash` on (default) for safety.
- For many machines, use central user management (LDAP/FreeIPA) so UIDs stay consistent automatically.
- Default `sec=sys` trusts the client's claimed UID, so only export to trusted networks. Kerberos (`sec=krb5`) is the secure option.
