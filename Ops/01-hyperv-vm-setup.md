
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

> **Warning:** Do not go below 2 GB RAM or 20 GB disk. AlmaLinux 10's installer alone needs ~1.5–2 GB, and the LAMP stack in Part 2 will not fit on 10 GB. 1 GB / 10 GB will fail partway through the install.

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

### 1a. Fix Secure Boot BEFORE first boot (do not skip)

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

#### Option A — Switch the Secure Boot template (keeps Secure Boot on) - recommended

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

> **Important:** This is the adapter that will appear as `eth0` inside the VM. Note which adapter you set to which switch — you'll need to keep them straight when configuring IPs.

## Step 5 - Add a Second Adapter for Internet

1. VM → **Settings → Add Hardware → Network Adapter**.
2. Virtual switch = `Default Switch` → **OK**.

**Result:** the VM has two NICs — one for a stable internal address, one for internet.

---

## Step 6 - Boot the VM and Confirm Both NICs Are Visible

Log into the VM console as `root`.

**Check that both interfaces exist:**

```bash
ip a
```

Expected: you should see `eth0` and `eth1` (plus `lo`). Both should show `state UP`.

> **If `eth0` or `eth1` is missing from `ip a`:** the vNIC is not attached in Hyper-V. Go back to Steps 4 and 5, verify both adapters are assigned to the correct switches, and reboot the VM.

**Check NetworkManager's view:**

```bash
nmcli connection show
nmcli device status
```

You may see something like:

```
NAME    DEVICE
eth1    eth1
lo      lo
eth0    --          <-- exists but not bound to a device
```

The `--` in the DEVICE column is the exact problem you hit. It means the connection profile exists but NetworkManager hasn't attached it to the NIC. The next step fixes it.

## Step 7 - Configure the Static IP on `eth0`

> **First, confirm the connection name that maps to `eth0`.** Run:
> ```bash
> nmcli -t -f NAME,DEVICE connection show
> ```
> Use whatever name appears next to `eth0`. In this guide we assume it is literally `eth0`. If yours is `"Wired connection 1"` or similar, substitute it in every command below.

```bash
nmcli connection modify eth0 ipv4.method manual
nmcli connection modify eth0 ipv4.addresses 10.10.10.101/24
nmcli connection modify eth0 ipv4.gateway 10.10.10.1
nmcli connection modify eth0 ipv4.dns "8.8.8.8 1.1.1.1"
nmcli connection modify eth0 connection.autoconnect yes
nmcli connection up eth0
```

| Command | What it does |
|---|---|
| `ipv4.method manual` | Disables DHCP on this interface |
| `ipv4.addresses` | Sets the fixed IP and subnet (`/24` = `255.255.255.0`) |
| `ipv4.gateway` | Sets the default gateway (the Windows host) |
| `ipv4.dns` | Sets Google and Cloudflare DNS resolvers |
| `connection.autoconnect yes` | Ensures it comes up on every boot |
| `connection up` | Attaches the profile to the device and applies it |

**Verify the binding and the IP:**

```bash
nmcli connection show
ip a show eth0
```

Expected:

```
NAME    DEVICE
eth1    eth1
lo      lo
eth0    eth0       <-- now bound
```

And `ip a show eth0` should include:

```
inet 10.10.10.101/24 brd 10.10.10.255 scope global noprefixroute eth0
```

> **If `nmcli connection up eth0` fails** with an error about the device being unavailable, run:
> ```bash
> nmcli device disconnect eth0
> nmcli device connect eth0
> nmcli connection up eth0
> ```
> This forces NetworkManager to claim the NIC.

## Step 8 - Confirm `eth1` Has Its DHCP Address

`eth1` is on the Default Switch and gets its IP automatically. Check it:

```bash
nmcli device status
ip a show eth1
```

Expected: `eth1` is `connected` and has an `inet 172.x.x.x/28` address.

If `eth1` has no IP:

```bash
nmcli device connect eth1
nmcli connection up eth1
```

> **Note:** The connection name bound to `eth1` may not literally be `eth1`. Use `nmcli -t -f NAME,DEVICE connection show` to find the real name paired with `eth1`, and substitute it in the commands below.

## Step 9 - Fix Route Priority (only one default route)

**The problem:** both NICs may claim to be the default route. `eth0` has no internet, so if it wins, all outbound traffic fails.

**The fix:** forbid `eth0` from being a default route, and give `eth1` a low metric (lower = higher priority).

```bash
nmcli connection modify eth0 ipv4.never-default yes
nmcli connection modify eth1 ipv4.route-metric 50
nmcli connection down eth0 && nmcli connection up eth0
nmcli connection down eth1 && nmcli connection up eth1
```

**Verify:**

```bash
ip route
```

Expected: a **single** `default via ... dev eth1` line, plus the two subnet routes:

```
default via 172.18.40.33 dev eth1
10.10.10.0/24 dev eth0 proto kernel scope link src 10.10.10.101
172.18.40.0/28 dev eth1 proto kernel scope link src 172.18.40.x
```

> **If `ipv4.never-default` gives a "setting not found" error**, do it interactively:
> ```bash
> nmcli connection edit eth0
> nmcli> set ipv4.never-default yes
> nmcli> save
> nmcli> quit
> ```

## Step 10 - Test Connectivity

```bash
ping -c 2 10.10.10.1        # Windows host on the internal switch
ping -c 4 8.8.8.8           # internet (raw IP, bypasses DNS)
ping -c 2 google.com        # DNS resolution
```

All three should succeed. If `10.10.10.1` fails, re-check Step 3 (host adapter IP). If `8.8.8.8` fails, re-check Step 9 (`ip route`). If only `google.com` fails, re-check `ipv4.dns` in Step 7.

---

## Step 11 - Enable SSH Root Login

AlmaLinux ships with `PermitRootLogin prohibit-password` (set in `/etc/ssh/sshd_config.d/50-redhat.conf`), which blocks password-based root SSH. We override it with a drop-in file.

```bash
echo -e "PermitRootLogin yes\nPasswordAuthentication yes" > /etc/ssh/sshd_config.d/99-custom.conf
systemctl restart sshd
sshd -T | grep -E "permitrootlogin|passwordauthentication"
```

Both values should return `yes`.

**Why a drop-in file?** Files in `sshd_config.d/` are read in alphabetical order, and the first value wins for most options in OpenSSH. Check with `sshd -T` to confirm the effective result rather than assuming.

> **Security warning:** Root login with a password is convenient for a lab but **not recommended for production**. See [04 - Security Hardening](04-security-hardening.md) for the secure replacement.

## Step 12 - Connect via MobaXterm

1. Open MobaXterm on the Windows host.
2. **Session → SSH**.
3. Fill in:

| Field | Value |
|---|---|
| Remote host | `10.10.10.101` |
| Specify username | `root` |
| Port | `22` |

4. Click **OK**, then double-click the saved session.
5. Enter the root password when prompted.

You should land in a shell prompt: `[root@localhost ~]#`.

> **If SSH is refused or times out:** confirm on the VM console that `sshd` is running and the firewall allows port 22:
> ```bash
> systemctl status sshd
> firewall-cmd --list-services
> ```
> If `ssh` is not listed:
> ```bash
> firewall-cmd --permanent --add-service=ssh
> firewall-cmd --reload
> ```

---

## Step 13 - Verify Persistence Across Reboot

Reboot the VM and confirm everything comes back automatically:

```bash
reboot
```

After it comes back (log in via MobaXterm at `10.10.10.101`):

```bash
ip a show eth0        # should show 10.10.10.101/24
ip a show eth1        # should show 172.x.x.x
ip route              # ONE default route via eth1
ping -c 2 google.com  # DNS + internet still work
```

If all four pass, the setup is permanent and you're ready for Part 2.

---

## Troubleshooting

### Secure Boot (most common first-boot issue)

| Symptom | Cause | Fix |
|---|---|---|
| `The image's hash and certificate are not allowed (DB)` | Secure Boot rejecting AlmaLinux's unsigned bootloader | **Step 1a** — switch template to *Microsoft UEFI Certificate Authority* OR uncheck *Enable Secure Boot* |
| `No operating system was loaded` on a Gen 2 VM | Same as above | Same fix |
| VM boots straight to UEFI shell | Boot order wrong | Settings → Firmware → move the DVD/ISO above the disk |

### Networking

| Symptom | Likely cause | Fix |
|---|---|---|
| `eth0` shows in `ip a` but has no IP | Connection profile not bound to device | Step 7 — `nmcli connection up eth0`, then verify with `nmcli connection show` |
| `nmcli connection show` shows `--` in DEVICE column for `eth0` | Same as above | Step 7 |
| Can't ping `10.10.10.101` from Windows | Wrong host IP or wrong switch | Re-check Steps 3 and 4 |
| VM has no internet | `eth0` is the default route | Re-run Step 9, check `ip route` |
| `eth1` has no IP | Connection not up | `nmcli device connect eth1 && nmcli connection up eth1` |
| `nmcli: command not found` | NetworkManager not installed | `dnf install NetworkManager -y && systemctl enable --now NetworkManager` |
| `Error: invalid or not allowed setting 'ipv4'` | Typo or profile state issue | Use `nmcli connection edit eth0` interactively |

### SSH

| Symptom | Likely cause | Fix |
|---|---|---|
| Connection timed out | Firewall blocking port 22 | `firewall-cmd --permanent --add-service=ssh && firewall-cmd --reload` |
| SSH "permission denied" for root | Override not applied | Run `sshd -T` and verify Step 11 |
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
- [ ] `nmcli connection show` lists **both** `eth0` and `eth1` with a DEVICE bound
- [ ] `eth0` = `10.10.10.101/24` (static)
- [ ] `eth1` has a `172.x.x.x` address (DHCP)
- [ ] `ip route` shows **one** default route via `eth1`
- [ ] `ping 10.10.10.1`, `ping 8.8.8.8`, and `ping google.com` all work
- [ ] SSH works from MobaXterm at `10.10.10.101`
- [ ] All of the above survive a `reboot`
- [ ] `getenforce` returns `Permissive` (or SELinux SSH issue is otherwise resolved)

**Next:** [02 - LAMP Stack & WordPress](02-lamp-wordpress-installation.md)
