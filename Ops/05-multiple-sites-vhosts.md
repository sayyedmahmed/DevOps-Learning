# 05 - Hosting Multiple Sites (Virtual Hosts + SSL + HTTP Redirect)

> **Series:** 5 of 5 | Previous: [04 - Security Hardening](04-security-hardening.md)

## Overview

This guide hosts **multiple websites on a single VM** using Apache **virtual hosts**, each with its **own HTTPS (SSL)** vhost and an **HTTP → HTTPS redirect**. Apache picks the right site based on the requested **hostname** (SNI + `Host:` header).

Sites hosted in this lab:

| Site | Hostname | DocumentRoot | Purpose |
|---|---|---|---|
| Site 1 | `site1.local` | `/var/www/html/site1` | static page |
| Site 2 | `site2.local` | `/var/www/html/site2` | static page |
| WordPress | `wp.local` | `/var/www/html/wordpress` | WordPress |
| Default | `test.local` | `/var/www/html` | fallback / index page |

**Prerequisites:** Apache + SSL working (Parts 3–4), WordPress installed in `/var/www/html/wordpress` (Part 2).

---

## Step 1 - Create the Site Directories

```bash
mkdir -p /var/www/html/site1 /var/www/html/site2
```

WordPress already lives in `/var/www/html/wordpress`, and the default static page in `/var/www/html`.

## Step 2 - Add Test Content

`/var/www/html/site1/index.html`
```html
<!DOCTYPE html>
<html>
<head><title>Site 1</title></head>
<body>
    <h1>This is Webserver 1</h1>
</body>
</html>
```

`/var/www/html/site2/index.html`
```html
<!DOCTYPE html>
<html>
<head><title>Site 2</title></head>
<body>
    <h1>This is Webserver 2</h1>
</body>
</html>
```

## Step 3 - Fix Ownership and SELinux Labels

```bash
chown -R apache:apache /var/www/html
restorecon -Rv /var/www/html
```

`restorecon` resets SELinux labels so Apache is allowed to read the files (prevents silent `403 Forbidden`).

## Step 4 - Create the Virtual Host File

`/etc/httpd/conf.d/vhosts.conf`
```apache
# ---------- site1.local ----------
<VirtualHost *:80>
    ServerName site1.local
    Redirect permanent / https://site1.local/
</VirtualHost>

<VirtualHost *:443>
    ServerName site1.local
    DocumentRoot /var/www/html/site1
    SSLEngine on
    SSLCertificateFile /etc/pki/tls/certs/cert.pem
    SSLCertificateKeyFile /etc/pki/tls/private/key.pem
</VirtualHost>

# ---------- site2.local ----------
<VirtualHost *:80>
    ServerName site2.local
    Redirect permanent / https://site2.local/
</VirtualHost>

<VirtualHost *:443>
    ServerName site2.local
    DocumentRoot /var/www/html/site2
    SSLEngine on
    SSLCertificateFile /etc/pki/tls/certs/cert.pem
    SSLCertificateKeyFile /etc/pki/tls/private/key.pem
</VirtualHost>

# ---------- wp.local ----------
<VirtualHost *:80>
    ServerName wp.local
    Redirect permanent / https://wp.local/
</VirtualHost>

<VirtualHost *:443>
    ServerName wp.local
    DocumentRoot /var/www/html/wordpress
    SSLEngine on
    SSLCertificateFile /etc/pki/tls/certs/cert.pem
    SSLCertificateKeyFile /etc/pki/tls/private/key.pem
</VirtualHost>
```

- Each site has **two blocks**: a `*:80` that redirects, and a `*:443` that serves over TLS.
- Each block matches a **unique `ServerName`** — that's what makes multiple sites on one IP work.

## Step 5 - Name the Default SSL Vhost

`/etc/httpd/conf.d/ssl.conf` contains a `<VirtualHost _default_:443>` block. Without a `ServerName`, it becomes the fallback for **all** `:443` requests — including your named sites — and steals them (serving `/var/www/html`).

Give it a throwaway name so it only answers for that name:

```bash
vim /etc/httpd/conf.d/ssl.conf
```

Find:
```apache
<VirtualHost _default_:443>
```

Change to:
```apache
<VirtualHost _default_:443>
    ServerName test.local
```

Now `test.local` is the default SSL site; `site1.local`, `site2.local`, and `wp.local` match their own vhosts.

## Step 6 - Verify the Certificate Files Exist

The vhosts point to:

```
/etc/pki/tls/certs/cert.pem
/etc/pki/tls/private/key.pem
```

If they're missing (still only in `/opt/ssl`), copy them:

```bash
cp /opt/ssl/cert.pem /etc/pki/tls/certs/cert.pem
cp /opt/ssl/key.pem  /etc/pki/tls/private/key.pem
chmod 600 /etc/pki/tls/private/key.pem
chown root:root /etc/pki/tls/private/key.pem
restorecon -Rv /etc/pki/tls
```

## Step 7 - Test and Restart Apache

```bash
httpd -t
systemctl restart httpd
```

`httpd -t` should print `Syntax OK`.

## Step 8 - Map Hostnames on the Client (Windows)

Edit the hosts file as **Administrator**:

1. Start → type **Notepad** → right-click → **Run as administrator**
2. **File → Open** → paste:
   ```
   C:\Windows\System32\drivers\etc\hosts
   ```
3. Change the file-type dropdown to **All files**, then open it
4. Add:
   ```
   10.10.10.101  site1.local
   10.10.10.101  site2.local
   10.10.10.101  wp.local
   10.10.10.101  test.local
   ```
5. Save (`Ctrl+S`)

Flush DNS cache after editing:

```cmd
ipconfig /flushdns
```

## Step 9 - Verify

### From the server (bypass DNS)

```bash
curl -I --resolve site1.local:80:10.10.10.101  http://site1.local/
curl -I --resolve site2.local:80:10.10.10.101  http://site2.local/
curl -I --resolve wp.local:80:10.10.10.101     http://wp.local/

curl -k --resolve site1.local:443:10.10.10.101 https://site1.local/
curl -k --resolve site2.local:443:10.10.10.101 https://site2.local/
curl -k --resolve wp.local:443:10.10.10.101    https://wp.local/
curl -k --resolve test.local:443:10.10.10.101  https://test.local/
```

Expected:

| URL | Result |
|---|---|
| `http://site1.local` | `301 Moved Permanently` → `https://site1.local/` |
| `http://site2.local` | `301 Moved Permanently` → `https://site2.local/` |
| `http://wp.local`    | `301 Moved Permanently` → `https://wp.local/` |
| `https://site1.local` | This is Webserver 1 |
| `https://site2.local` | This is Webserver 2 |
| `https://wp.local`    | WordPress |
| `https://test.local`  | This is Index Page (default) |

### From the browser

- `site1.local` → auto-upgrades to HTTPS → **This is Webserver 1**
- `site2.local` → auto-upgrades to HTTPS → **This is Webserver 2**
- `wp.local`    → auto-upgrades to HTTPS → **WordPress**
- `test.local`  → default index page

Accept the self-signed certificate warning (expected for a lab cert).

---

## The Problem We Hit (and How We Fixed It)

### Symptom

Typing `site1.local` (no scheme) in the browser showed the **default index page** instead of "This is Webserver 1". Only `https://site1.local` worked. Later, after adding more vhosts, only **site2** worked over HTTPS while site1 and wp fell back to the default page.

### Root cause

Two separate issues:

1. **No `*:80` vhosts for the named sites** — the vhosts only existed on `:443`. Typing `site1.local` makes the browser request `http://site1.local` on **port 80**, which had no matching vhost, so Apache served the **default `DocumentRoot`** (`/var/www/html/index.html`).

2. **The `_default_:443` block in `ssl.conf` had no `ServerName`** — it was acting as the fallback SSL vhost for anything that didn't match a name, serving `/var/www/html` (the index page) instead of the named site. Combined with a duplicate/ambiguous `ServerName`, only one named vhost (site2) ended up winning on `:443`.

### Diagnostic commands we used

```bash
httpd -S
```
Lists every vhost Apache has loaded, grouped by port. This is the single most useful command for vhost debugging — if a name appears once with the right `DocumentRoot`, it's loaded correctly; if it's missing or duplicated, the config isn't merged the way you expect.

```bash
grep -n "ServerName" /etc/httpd/conf.d/ssl.conf
```
Confirmed the default block had no active `ServerName`.

```bash
curl -k -H "Host: site2.local" https://10.10.10.101/
```
Returned `421 Misdirected Request` — because SNI (`10.10.10.101`) and `Host:` (`site2.local`) disagreed. This is expected and proves Apache is checking SNI/Host consistency.

```bash
curl -k --resolve site2.local:443:10.10.10.101 https://site2.local/
```
The **correct** way to test — forces SNI and `Host:` to match, like a real browser.

### The fix

1. **Added `*:80` redirect vhosts** for `site1.local`, `site2.local`, and `wp.local` so HTTP requests upgrade to HTTPS.
2. **Named the `_default_:443` block** in `ssl.conf` as `ServerName test.local`, so it no longer acts as an unnamed fallback for the named sites.

### Why it works now

- Typing `site1.local` → port 80 → `*:80` vhost matches → `301` → `https://site1.local`.
- `https://site1.local` → port 443 → `ServerName site1.local` vhost matches → serves `/var/www/html/site1`.
- Unmatched hosts fall through to `test.local` → default index page.

---

## How It Works

Apache reads the **`Host:` header** (and, on HTTPS, the **SNI** from the TLS handshake) and picks the vhost whose `ServerName` matches. If nothing matches, it uses the **default vhost** for that port — the first one defined.

```
Browser → http://site1.local
        |
        v
Port 80 vhost matches ServerName site1.local
        |
        v
301 → https://site1.local
        |
        v
TLS handshake: SNI = site1.local
HTTP request:  Host: site1.local
        |
        v
Port 443 vhost matches ServerName site1.local
        |
        v
Serves /var/www/html/site1/index.html over TLS
```

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Typing name shows default index page | No `*:80` vhost / no redirect | Add `*:80` block with `Redirect permanent` |
| Only one HTTPS name works, others show default | Unnamed `_default_:443` block winning, or duplicate `ServerName` | Name the default `test.local`; check `httpd -S` for duplicates |
| `421 Misdirected Request` | SNI and `Host:` header disagree | Test with `--resolve`, not `-H Host:` |
| `403 Forbidden` | Wrong SELinux label | `restorecon -Rv /var/www/html` |
| `httpd -t` fails | Missing cert file | Copy cert/key into `/etc/pki/tls/` |
| Apache won't start | `SSLCertificateFile` path wrong | Verify paths and `restorecon` |
| Name doesn't resolve on Windows | hosts file not saved as admin | Re-edit as Administrator, run `ipconfig /flushdns` |
| WordPress redirects to wrong URL | Site URL still old | wp-admin → Settings → General → `https://wp.local` |
| Browser shows cert warning | Self-signed cert with IP SAN | Expected in lab; accept it |

## Checklist

- [ ] Each site has its own folder with `index.html`/`index.php`
- [ ] `chown -R apache:apache /var/www/html`
- [ ] `restorecon -Rv /var/www/html`
- [ ] `vhosts.conf` has a `*:80` redirect and `*:443` SSL block per site
- [ ] Each block has a **unique `ServerName`**
- [ ] `_default_:443` in `ssl.conf` has `ServerName test.local`
- [ ] Cert/key exist in `/etc/pki/tls/certs/` and `/etc/pki/tls/private/`
- [ ] `httpd -t` returns `Syntax OK`
- [ ] `httpd -S` shows each site exactly once per port
- [ ] Apache restarted
- [ ] Windows hosts file updated as Administrator + `ipconfig /flushdns`
- [ ] Typing `site1.local` / `site2.local` / `wp.local` upgrades to HTTPS and loads the correct site
- [ ] `test.local` serves the default index page
```
