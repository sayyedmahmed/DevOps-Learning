# 01 - Hyper-V VM Setup (AlmaLinux 10)

> **Series:** 1 of 4 | Next: [02 - LAMP Stack & WordPress](02-lamp-wordpress-installation.md)

## Overview

By the end of this guide you will have an **AlmaLinux 10 virtual machine** on **Windows Hyper-V** that:

- Always has the same address (`10.10.10.101`), so you can connect by SSH.
- Can reach the internet (to install software and updates).
- Accepts SSH logins from MobaXterm on your Windows PC.

### The two network adapters (NICs)

| Name in this guide | Hyper-V switch | Job | IP address |
|---|---|---|---|
| `eth0` | `Default Switch` | **Internet** access | `172.x.x.x` (automatic) |
| `eth1` | `vSwitch-Internal` | **Fixed address** for SSH from Windows | `10.10.10.101` (you set it) |

**Why two?** The `Default Switch` gives internet but its IP changes. The `Internal` switch lets us pick a fixed IP but has no internet. Using both gives us everything.

### How to read this guide

- Run commands **one at a time**. Type one command, press **Enter**, read the result, then continue.
- Text after a `#` is a note for you. Do not type it.

## Prerequisites

- Windows 10/11 Pro/Enterprise with Hyper-V enabled
- AlmaLinux 10 ISO
- 2 GB RAM minimum (4 GB recommended), 20 GB disk

---

# Part A - Hyper-V and Windows Setup

## Step 1 - Create the VM

1. Open **Hyper-V Manager → New → Virtual Machine**.
2. Name the VM and choose **Generation 2**.
3. Memory: **4 GB** recommended.
4. Virtual hard disk: **20 GB** minimum.
5. Attach the AlmaLinux 10 ISO.
6. Install with **Minimal Install** and set a **root password** (remember it).

> If the VM will not boot the ISO: VM Settings → Security → set the Secure Boot template to *Microsoft UEFI Certificate Authority*.

## Step 2 - Create the Internal Virtual Switch

1. Hyper-V Manager → **Virtual Switch Manager → New virtual network switch**.
2. Choose **Internal** → **Create Virtual Switch**.
3. Name it `vSwitch-Internal` → **Apply → OK**.

## Step 3 - Set the Windows Adapter IP

1. Control Panel → Network and Sharing Center → **Change adapter settings**.
2. Right-click `vEthernet (vSwitch-Internal)` → **Properties**.
3. Double-click **Internet Protocol Version 4 (TCP/IPv4)**.
4. Choose "Use the following IP address" and enter:

| Field | Value |
|---|---|
| IP address | `10.10.10.1` |
| Subnet mask | `255.255.255.0` |
| Default gateway | *(leave blank)* |

Leave the gateway blank. Windows already has its own route to the internet.

## Step 4 - Attach the Adapters to the VM

The VM needs **two** adapters, in this order:

| Adapter in Hyper-V | Virtual switch | Becomes |
|---|---|---|
| Network Adapter (first) | `Default Switch` | `eth0` |
| Network Adapter (second) | `vSwitch-Internal` | `eth1` |

1. Right-click the VM → **Settings**.
2. Click the first **Network Adapter** → Virtual switch = `Default Switch` → **Apply**.
3. Click **Add Hardware → Network Adapter → Add**.
4. On the new adapter, Virtual switch = `vSwitch-Internal` → **OK**.

> **Already added both adapters?** Do not add them again. You will check which is which in Step 5.

---

# Part B - Network Setup Inside the VM

Start the VM and log in as `root` on the **Hyper-V console** (not SSH).

## Step 5 - Check Which Adapter Is Which

Show all adapters and their addresses:

```bash
ip -br a
```

You should see something like:

```
eth0    UP    172.x.x.x/20
eth1    UP
```

| Result | Meaning |
|---|---|
| `eth0` has `172.x.x.x`, `eth1` has no IP | Correct. Go to Step 6. |
| `eth1` has `172.x.x.x`, `eth0` has no IP | Reversed. Fix below. |

**If reversed:** in Hyper-V, shut down the VM, open Settings, and swap the switches: first adapter = `Default Switch`, second = `vSwitch-Internal`. Start the VM and run `ip -br a` again.

Now show the connection names:

```bash
nmcli connection show
```

Look at the **NAME** and **DEVICE** columns. Ideally the NAME matches the DEVICE (`eth0` and `eth1`). If NAME is something else, such as `Wired connection 1`, rename it so it matches (see Step 6).

## Step 6 - Make Connection Names Match Device Names

Skip any command here if the NAME already equals the DEVICE.

Rename the connection on `eth0` (replace `Wired connection 1` with the NAME shown next to `eth0`):

```bash
nmcli connection modify "Wired connection 1" connection.id eth0
```

Rename the connection on `eth1` (replace `Wired connection 2` with the NAME shown next to `eth1`):

```bash
nmcli connection modify "Wired connection 2" connection.id eth1
```

Check again:

```bash
nmcli connection show
```

Both NAME and DEVICE should now show `eth0` and `eth1`.

> **If `eth1` has no connection at all**, create it:
>
> ```bash
> nmcli connection add type ethernet ifname eth1 con-name eth1
> ```

## Step 7 - Check the Default Connection (eth0) First

> **IMPORTANT - check before changing anything**
>
> `eth0` usually already works on its own (automatic IP + internet). Test it first:
>
> ```bash
> ping -c 4 8.8.8.8
> ```
>
> - **Replies received** → internet works. **Do NOT touch `eth0`.** Skip Step 9 and go to Step 8.
> - **No replies / "Network is unreachable"** → continue to Step 8 anyway, then do Step 9 only if it is still not working.

## Step 8 - eth1 (Internal): Fixed IP (Do This First)

> **Do this on the Hyper-V console.** If you are connected by SSH, the `down` command below will disconnect you.

Turn off automatic IP:

```bash
nmcli connection modify eth1 ipv4.method manual
```

Set the fixed IP:

```bash
nmcli connection modify eth1 ipv4.addresses 10.10.10.101/24
```

Stop this adapter from being the internet route:

```bash
nmcli connection modify eth1 ipv4.never-default yes
```

Start it automatically at boot:

```bash
nmcli connection modify eth1 connection.autoconnect yes
```

Apply the changes:

```bash
nmcli connection down eth1
```

```bash
nmcli connection up eth1
```

**Why no gateway or DNS on eth1?** It only talks to your Windows PC (`10.10.10.1`). `eth0` handles the internet.

> **Now test the internet again:**
>
> ```bash
> ping -c 4 8.8.8.8
> ```
>
> - **Replies** → done, go to Step 10. Skip Step 9.
> - **No replies** → do Step 9 to fix `eth0`.

## Step 9 - Fix eth0 (Only If Internet Is Not Working)

> **Skip this step if `ping -c 4 8.8.8.8` already works.** Only do it after Step 8 is finished and internet still fails.

Use automatic IP (DHCP):

```bash
nmcli connection modify eth0 ipv4.method auto
```

Start it automatically at boot:

```bash
nmcli connection modify eth0 connection.autoconnect yes
```

Bring it up:

```bash
nmcli connection up eth0
```

## Step 10 - Check Everything

Show the addresses:

```bash
ip -br a
```

Expected: `eth0` has `172.x.x.x`, `eth1` has `10.10.10.101/24`.

Show the routes:

```bash
ip route
```

Expected: exactly **one** line starting with `default via`, and it ends with `dev eth0`.

Test the internet by IP address:

```bash
ping -c 4 8.8.8.8
```

Test internet names (DNS):

```bash
ping -c 2 google.com
```

Both should show replies. Press `Ctrl+C` if a ping does not stop by itself.

On **Windows**, open Command Prompt and run:

```
ping 10.10.10.101
```

You should get replies too.

---

# Part C - SSH Access

## Step 11 - Allow Root Login with a Password

Create the file with both settings:

```bash
printf "PermitRootLogin yes\nPasswordAuthentication yes\n" > /etc/ssh/sshd_config.d/01-custom.conf
```

Check the file:

```bash
cat /etc/ssh/sshd_config.d/01-custom.conf
```

Expected output:

```
PermitRootLogin yes
PasswordAuthentication yes
```

Check the SSH settings for mistakes (no output means OK):

```bash
sshd -t
```

Restart SSH:

```bash
systemctl restart sshd
```

Confirm the settings are active:

```bash
sshd -T | grep -i permitrootlogin
```

```bash
sshd -T | grep -i passwordauthentication
```

Both must say `yes`.

> **Side note - also try this if password login does not work properly**
>
> Add the keyboard-interactive option as well. SSH still asks for your normal root password (through PAM), but some setups accept it when plain password authentication fails.
>
> ```bash
> echo "KbdInteractiveAuthentication yes" >> /etc/ssh/sshd_config.d/01-custom.conf
> ```
>
> ```bash
> sshd -t
> ```
>
> ```bash
> systemctl restart sshd
> ```
>
> ```bash
> sshd -T | grep -i kbdinteractiveauthentication
> ```
>
> It must say `yes`. Keep both options in the file, so the file now has three lines. Then try MobaXterm again.

> **Common typo:** `sshd -T` (with a **d**) checks the SSH server. `ssh -T` is a different command and only prints help text.

> **Security warning:** Root password login is fine for a learning lab, but not for a real server. Guide 04 covers the safe replacement.

### If `sshd -t` reports a bad configuration option

Re-check the spelling in the file:

```bash
cat /etc/ssh/sshd_config.d/01-custom.conf
```

Check the OpenSSH version:

```bash
ssh -V
```

To see which options are valid on your build:

```bash
man sshd_config | grep -iE "password|kbdinteractive"
```

### If `PermitRootLogin` still says `prohibit-password`

Another file is overriding yours. Find it:

```bash
grep -rn "PermitRootLogin" /etc/ssh/sshd_config /etc/ssh/sshd_config.d/
```

If a file other than yours has an active `PermitRootLogin` line (no `#` at the start), edit that file or rename yours so it sorts first.

If nothing helps, edit the main file. First make a backup:

```bash
cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
```

Open it in the `vi` editor:

```bash
vi /etc/ssh/sshd_config
```

| Key | What it does |
|---|---|
| `/PermitRootLogin` then Enter | Search for the line |
| `i` | Start typing (insert mode) |
| `Esc` | Stop typing |
| `:wq` then Enter | Save and quit |
| `:q!` then Enter | Quit without saving |

Change the line to `PermitRootLogin yes` (remove the `#` at the start). Do the same for `PasswordAuthentication yes` (and `KbdInteractiveAuthentication yes` if you use the side note). Save, then repeat the `sshd -t`, `systemctl restart sshd` and `sshd -T` checks above, one at a time.

## Step 12 - Connect with MobaXterm

1. Click **Session → SSH**.
2. Remote host: `10.10.10.101`
3. Tick **Specify username** and enter `root`.
4. Port: `22`
5. Click **OK** and enter your root password.

## Step 13 - Reboot Test

```bash
reboot
```

When the VM is back, repeat **Step 10**. The IPs, the single default route and the internet should all still work.

---

# Troubleshooting

| Problem | Likely cause | Fix |
|---|---|---|
| Windows cannot ping `10.10.10.101` | Wrong IP on the VM, or wrong Windows IP/switch | Check `ip -br a` in the VM; re-check Steps 3 and 4 |
| VM has no internet | `eth1` is the default route, or wrong gateway | `nmcli connection modify eth1 ipv4.never-default yes`; check `ip route` |
| `eth0` has no IP | Connection not up | `nmcli connection up eth0` |
| `eth0` and `eth1` are the wrong way round | Adapter order in Hyper-V | Redo Step 5 |
| SSH says "permission denied" | Setting not applied, or wrong password | Check Step 11 with `sshd -T` |
| `sshd -t` says `Bad configuration option` | Typo or stray character in the file | Re-create the file with the `printf` command in Step 11 |
| Password login fails even though `sshd -T` shows `yes` | Plain password auth not accepted | Try the side note in Step 11 (`KbdInteractiveAuthentication yes`) |
| Log shows `Could not get shadow information` | SELinux blocking SSH | See below |

## SELinux blocking SSH login

**Symptom:** the connection opens but your correct password is always rejected. Check the log:

```bash
tail -n 20 /var/log/secure
```

If you see `Could not get shadow information`, SELinux (a security feature) is blocking SSH from reading the password file.

Check the SELinux mode:

```bash
getenforce
```

If it says `Enforcing`, that may be the cause.

**Temporary fix** (lasts until reboot):

```bash
setenforce 0
```

Try SSH again. If it works, make it permanent (lab use):

```bash
sed -i 's/^SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config
```

```bash
reboot
```

After reboot, check:

```bash
getenforce
```

Expected: `Permissive`. Do **not** disable SELinux completely, because guide 04 uses it.

**If `getenforce` already says `Permissive`**, check whether the root account is locked:

```bash
passwd -S root
```

If the second word is `L`, unlock it:

```bash
passwd -u root
```

---

# Checklist

- [ ] VM installs and boots
- [ ] Windows adapter is `10.10.10.1`
- [ ] `eth0` (Default Switch) has a `172.x.x.x` address
- [ ] `eth1` (vSwitch-Internal) is `10.10.10.101`
- [ ] `ip route` shows only one `default via` line
- [ ] `ping 8.8.8.8` and `ping google.com` work
- [ ] Windows can `ping 10.10.10.101`
- [ ] `sshd -T` shows `permitrootlogin yes` and `passwordauthentication yes` (plus `kbdinteractiveauthentication yes` if you used the side note)
- [ ] MobaXterm connects as `root`
- [ ] Everything still works after `reboot`

**Next:** [02 - LAMP Stack & WordPress](02-lamp-wordpress-installation.md)
