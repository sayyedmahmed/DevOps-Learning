# 01 - Hyper-V VM Setup (AlmaLinux 10)

> **Series:** 1 of 4 | Next: [02 - LAMP Stack & WordPress](02-lamp-wordpress-installation.md)

## Overview

This guide builds the foundation for the whole project: an **AlmaLinux 10 virtual machine** on **Windows Hyper-V** with:

- A **stable internal IP** (`10.10.10.101`) so the VM is always reachable at the same address from the Windows host.
- A **second network adapter** for internet access (package installs, updates).
- **SSH access** from the host using MobaXterm.

### Why two network adapters?

Hyper-V's `Default Switch` hands out a changing IP (DHCP) on every reboot, which is unreliable for hosting a website or SSH. An `Internal` switch lets us pick a fixed IP, but it has no internet. Using both gives us the best of each:

| Adapter | Switch | Purpose | IP |
|---|---|---|---|
| `eth0` | `vSwitch-Internal` | Stable management/web address | `10.10.10.101` (static) |
| `eth1` | `Default Switch` | Internet access | `172.x.x.x` (DHCP) |

## Prerequisites

- Windows 10/11 Pro/Enterprise with Hyper-V enabled
- AlmaLinux 10 ISO
- **RAM:** 2 GB minimum, 4 GB maximum
- **Disk:** 20 GB minimum
- **CPU:** 1 vCPU minimum, 2 vCPU recommended

> ⚠️ **Do not go below 2 GB RAM or 20 GB disk.** AlmaLinux 10's installer alone needs ~1.5–2 GB, and the LAMP stack in Part 2 will not fit on 10 GB. 1 GB / 10 GB will fail partway through the install.

---

## Step 1 - Create the VM

1. Open **Hyper-V Manager → New → Virtual Machine**.
2. Name the VM and choose **Generation 2** (UEFI, better performance).
3. **Assign memory:** set **Startup memory to 2 GB (2048 MB)**.
   - Optionally enable **Dynamic Memory** with:
     - Minimum RAM: `2048 MB`
     - Maximum RAM: `4096 MB`
   - This keeps idle usage at ~2 GB and lets it burst to 4 GB during `dnf install` or MariaDB work.
4. **Configure networking:** for now, leave it on `Default Switch` (we add the internal one in Steps 2–4).
5. **Create a virtual hard disk:** **20 GB minimum**, dynamically expanding.
6. **Attach the AlmaLinux 10 ISO** → Finish.
7. **Before starting the VM, fix Secure Boot** — see Step 1a.

### ⚠️ 1a. Fix Secure Boot BEFORE first boot (do not skip)

**Generation 2 VMs have Secure Boot enabled by default.** AlmaLinux's bootloader is **not signed with a Microsoft-trusted certificate**, so the VM refuses to boot the ISO and shows:

```
Virtual Machine Boot Summary
1. SCSI DVD (0,1)
   The image's hash and certificate are not allowed (DB).
...
No operating system was loaded.
```

**This is not an ISO problem and not a Hyper-V bug — it is Secure Boot doing its job.**

You have **two fixes**. Pick one:

#### Option A — Switch the Secure Boot template (keeps Secure Boot on) ✅ recommended

1. Right-click the VM → **Settings**.
2. Left pane → **Security**.
3. Keep **Enable Secure Boot** checked.
4. Change **Template** from `Microsoft Windows` to **`Microsoft UEFI Certificate Authority`**.
5. **Apply → OK**.

This template trusts the UEFI CA that signs Linux bootloaders, so AlmaLinux boots while Secure Boot stays enabled.

#### Option B — Disable Secure Boot (simplest for a lab)

1. Right-click the VM → **Settings**.
2. Left pane → **Security**.
3. **Uncheck** `Enable Secure Boot`.
4. **Apply → OK**.

Either option works. Option A is closer to production; Option B is one click. For this lab, **either is fine**.

> **If you already tried to boot and got the error:** the VM is not broken. Apply Option A or B, then start the VM again.

### 1b. Install AlmaLinux

1. Start the VM and connect to the console.
2. Walk through the Anaconda installer.
3. Choose the **Minimal Install** profile.
4. Set a **root password** (and optionally create a user).
5. Reboot when finished.

> **Why minimal install?** Fewer packages mean a smaller attack surface and less to patch. We add only what we need in Part 2.

## Step 2 - Create an Internal Virtual Switch

1. Hyper-V Manager → **Virtual Switch Manager → New virtual network switch**.
2. Select **Internal** → **Create Virtual Switch**.
3. Name it `vSwitch-Internal` → **Apply → OK**.

An *Internal* switch connects only the VM and the Windows host. It is **not** exposed to your physical LAN or the internet.

## Step 3 - Configure the Windows Host Adapter

Windows creates a virtual adapter for the switch. Give it a static IP so it sits on the same subnet as the VM.

1. Control Panel → Network and Sharing Center → **Change adapter settings**.
2. Right-click `vEthernet (vSwitch-Internal)` → **Properties**.
3. Select **Internet Protocol Version 4 (TCP/IPv4)** → **Properties**.
4. Set:

| Field | Value |
|---|---|
| IP address | `10.10.10.1` |
| Subnet mask | `255.255.255.0` |
| Default gateway | *(leave blank)* |

**Why leave the gateway blank?** Windows already has a default route to the internet. Adding another gateway here could break the host's own connectivity.

## Step 4 - Attach the VM to the Internal Switch

1. Right-click the VM → **Settings**.
2. **Network Adapter** → Virtual switch = `vSwitch-Internal` → **OK**.

## Step 5 - Add a Second Adapter for Internet

1. VM → **Settings → Add Hardware → Network Adapter**.
2. Virtual switch = `Default Switch` → **OK**.

**Result:** the VM has two NICs — one for a stable internal address, one for internet.

---

## Step 6 - Boot the VM and Set a Static IP

Log into the VM console as `root`. We use **NetworkManager (`nmcli`)**, the default network tool on AlmaLinux.

```bash
nmcli connection modify eth0 ipv4.method manual
nmcli connection modify eth0 ipv4.addresses 10.10.10.101/24
nmcli connection modify eth0 ipv4.gateway 10.10.10.1
nmcli connection modify eth0 ipv4.dns "8.8.8.8 1.1.1.1"
nmcli connection down eth0 && nmcli connection up eth0
```

| Command | What it does |
|---|---|
| `ipv4.method manual` | Disables DHCP on this interface |
| `ipv4.addresses` | Sets the fixed IP and subnet (`/24` = `255.255.255.0`) |
| `ipv4.gateway` | Sets the default gateway (the Windows host) |
| `ipv4.dns` | Sets Google and Cloudflare DNS resolvers |
| `down && up` | Restarts the connection to apply changes |

**Verify:**
```bash
ip a show eth0
```
Expected: `inet 10.10.10.101/24`

> **Tip:** Connection names may differ on your system. Run `nmcli connection show` to see the exact names.

## Step 7 - Bring Up the Second Adapter

```bash
nmcli connection show
nmcli connection up "Wired connection 1"
```

**Verify:**
```bash
ip a show eth1
```
Expected: a `172.x.x.x` address from the Default Switch's DHCP.

## Step 8 - Fix Route Priority

**The problem:** both NICs may claim to be the default route. `eth0` has no internet, so if it wins, all outbound traffic fails.

**The fix:** forbid `eth0` from being a default route, and give `eth1` a low metric (lower = higher priority).

```bash
nmcli connection modify eth0 ipv4.never-default yes
nmcli connection modify "Wired connection 1" ipv4.route-metric 50
nmcli connection down eth0 && nmcli connection up eth0
nmcli connection up "Wired connection 1"
```

**Verify:**
```bash
ip route
```
Expected: a **single** `default via ... dev eth1` route.

## Step 9 - Test Internet

```bash
ping -c 4 8.8.8.8
```
Four replies means routing is correct. Also test DNS: `ping -c 2 google.com`.

---

## Step 10 - Enable SSH Root Login

AlmaLinux ships with `PermitRootLogin prohibit-password` (set in `/etc/ssh/sshd_config.d/50-redhat.conf`), which blocks password-based root SSH. We override it with a drop-in file.

```bash
echo -e "PermitRootLogin yes\nPasswordAuthentication yes" > /etc/ssh/sshd_config.d/99-custom.conf
systemctl restart sshd
sshd -T | grep -E "permitrootlogin|passwordauthentication"
```

Both values should return `yes`.

**Why a drop-in file?** Files in `sshd_config.d/` are read in alphabetical order, and the first value wins for most options in OpenSSH. Check with `sshd -T` to confirm the effective result rather than assuming.

> **Security warning:** Root login with a password is convenient for a lab but **not recommended for production**. See [04 - Security Hardening](04-security-hardening.md) for the secure replacement.

## Step 11 - Connect via MobaXterm

1. **Session → SSH**
2. Remote host: `10.10.10.101`
3. Port: `22`
4. Username: `root`
5. Save and connect.

---

## Troubleshooting

### Secure Boot (most common first-boot issue)

| Symptom | Cause | Fix |
|---|---|---|
| `The image's hash and certificate are not allowed (DB)` | Secure Boot rejecting AlmaLinux's unsigned bootloader | **Step 1a** — switch template to *Microsoft UEFI Certificate Authority* OR uncheck *Enable Secure Boot* |
| `No operating system was loaded` on a Gen 2 VM | Same as above | Same fix |
| VM boots straight to UEFI shell | Boot order wrong | Settings → Firmware → move the DVD/ISO above the disk |

### SSH

| Symptom | Likely cause | Fix |
|---|---|---|
| Can't ping `10.10.10.101` from Windows | Wrong host IP or wrong switch | Re-check Steps 3 and 4 |
| VM has no internet | `eth0` is the default route | Re-run Step 8, check `ip route` |
| SSH "permission denied" for root | Override not applied | Run `sshd -T` and verify Step 10 |
| `eth1` has no IP | Connection not up | `nmcli connection up "Wired connection 1"` |
| SSH rejects root with `Could not get shadow information for ROOT` in `/var/log/secure` | SELinux is blocking `sshd` from reading `/etc/shadow` | See **SELinux blocking SSH root login** below |

#### SELinux blocking SSH root login

**Symptom:** `sshd` accepts the connection but every login attempt fails. `/var/log/secure` shows:

```
error: Could not get shadow information for ROOT
Failed password for invalid user root from 10.10.10.1 port 51377 ssh2
Received disconnect from 10.10.10.1 port 51377:8: [preauth]
```

This is **not** a password or `sshd_config` problem. SELinux is preventing the SSH daemon from reading `/etc/shadow` to verify credentials.

**Diagnose:**

```bash
getenforce
```

If the output is `Enforcing`, that is the cause.

**Fix (temporary, until reboot):**

```bash
setenforce 0
```

This takes effect immediately and lets you SSH back in to fix things properly.

**Fix (permanent):**

```bash
sed -i 's/SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config
reboot
```

This keeps SELinux from interfering while still logging policy violations. Do not disable SELinux entirely — Step 04 will cover the correct contexts for the web root later.

**Verify after reboot:**

```bash
getenforce
# Expected: Permissive
```

Then retry the SSH login from MobaXterm. It should now succeed.

> **Note:** If `getenforce` already returns `Permissive` or `Disabled`, the cause is elsewhere. Check whether the root account itself is locked with `passwd -S root` — an `L` in the second field means it is locked, and `passwd -u root` will unlock it.

---

## Checklist

- [ ] VM created as **Generation 2** with **2 GB RAM** (max 4 GB via dynamic memory)
- [ ] Virtual disk is **at least 20 GB**
- [ ] **Secure Boot fixed before first boot** (template = *Microsoft UEFI Certificate Authority*, OR disabled)
- [ ] AlmaLinux installed (Minimal Install)
- [ ] `eth0` = `10.10.10.101`
- [ ] `eth1` has a `172.x.x.x` address
- [ ] `ip route` shows one default route via `eth1`
- [ ] `ping 8.8.8.8` works
- [ ] SSH works from MobaXterm
- [ ] `getenforce` returns `Permissive` (or SELinux SSH issue is otherwise resolved)

**Next:** [02 - LAMP Stack & WordPress](02-lamp-wordpress-installation.md)
