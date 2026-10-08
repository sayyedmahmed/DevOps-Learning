# 05 - Hosting Multiple Sites 
> ### Virtual Hosts + SSL + HTTP Redirect

> **Series:** 5 of 5 | Previous: [04 - Security Hardening](04-security-hardening.md) | End of series

## Overview

This guide hosts multiple websites on a single VM using **Apache Virtual Hosts**.

Apache chooses the correct site based on the requested hostname:

- `site1.local`
- `site2.local`
- `wp.local`
- `test.local` as the default/fallback site

Each site has:

1. An HTTP `*:80` block that redirects to HTTPS.
2. An HTTPS `*:443` block that serves the site securely.

### Sites hosted in this lab

| Site | Hostname | DocumentRoot | Purpose |
|---|---|---|---|
| Site 1 | `site1.local` | `/var/www/html/site1` | Static test page |
| Site 2 | `site2.local` | `/var/www/html/site2` | Static test page |
| WordPress | `wp.local` | `/var/www/html/wordpress` | WordPress site |
| Default | `test.local` | `/var/www/html` | Fallback/index page |

**Prerequisites:**

- Apache + SSL working from Parts 3–4.
- WordPress installed in `/var/www/html/wordpress`.
- SSL certificate and key available at:
  - `/etc/pki/tls/certs/cert.pem`
  - `/etc/pki/tls/private/key.pem`

---

## Step 1 - Create the Site Directories

Create folders for the static test sites.

```bash
mkdir -p /var/www/html/site1 /var/www/html/site2
```

WordPress is assumed to already exist here:

```bash
/var/www/html/wordpress
```

The default site uses the main web root:

```bash
/var/www/html
```

### Why are we doing this?

Each virtual host needs its own `DocumentRoot`. This keeps sites separated so one site does not serve another site’s files.

---

## Step 2 - Add Test Content

Create a simple homepage for Site 1:

```bash
cat > /var/www/html/site1/index.html <<'EOF'
<!DOCTYPE html>
<html>
<head><title>Site 1</title></head>
<body>
    <h1>This is Webserver 1</h1>
</body>
</html>
EOF
```

Create a simple homepage for Site 2:

```bash
cat > /var/www/html/site2/index.html <<'EOF'
<!DOCTYPE html>
<html>
<head><title>Site 2</title></head>
<body>
    <h1>This is Webserver 2</h1>
</body>
</html>
EOF
```

Optionally create a default index page for `test.local`:

```bash
cat > /var/www/html/index.html <<'EOF'
<!DOCTYPE html>
<html>
<head><title>Default Site</title></head>
<body>
    <h1>This is Index Page</h1>
</body>
</html>
EOF
```

### Why are we doing this?

These files help verify that Apache is serving the correct site for each hostname.

---

## Step 3 - Fix Ownership and SELinux Labels

Give Apache ownership of the web files:

```bash
chown -R apache:apache /var/www/html
```

Restore correct SELinux labels:

```bash
restorecon -Rv /var/www/html
```

### Why are we doing this?

Apache must be allowed to read these files.

On AlmaLinux/RHEL systems, SELinux can block Apache even if normal Linux permissions look correct. `restorecon` fixes the SELinux file labels and helps prevent silent `403 Forbidden` errors.

---

## Step 4 - Create the Virtual Host File

Create the main virtual host configuration:

```bash
vi /etc/httpd/conf.d/vhosts.conf
```

Paste the following:

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

Save and exit.

### Why are we doing this?

Each site has two blocks:

| Block | Purpose |
|---|---|
| `<VirtualHost *:80>` | Catches normal HTTP requests and redirects them to HTTPS |
| `<VirtualHost *:443>` | Serves the actual site over HTTPS |

The important line is:

```apache
ServerName site1.local
```

Apache uses `ServerName` to decide which site should answer the request.

---

## Step 5 - Confirm SSL Certificate Files Exist

Your virtual hosts point to:

```text
/etc/pki/tls/certs/cert.pem
/etc/pki/tls/private/key.pem
```

Check that they exist:

```bash
ls -l /etc/pki/tls/certs/cert.pem
ls -l /etc/pki/tls/private/key.pem
```

If they are missing, copy them from `/opt/ssl`:

```bash
cp /opt/ssl/cert.pem /etc/pki/tls/certs/cert.pem
cp /opt/ssl/key.pem  /etc/pki/tls/private/key.pem
```

Secure the private key:

```bash
chmod 600 /etc/pki/tls/private/key.pem
chown root:root /etc/pki/tls/private/key.pem
```

Restore SELinux labels:

```bash
restorecon -Rv /etc/pki/tls
```

### Why are we doing this?

Apache needs the certificate and private key to serve HTTPS.

The private key must be protected because anyone who can read it can impersonate your server.

---

## Step 6 - Test and Restart Apache

Always test Apache configuration before restarting:

```bash
httpd -t
```

Expected output:

```text
Syntax OK
```

If syntax is OK, restart Apache:

```bash
systemctl restart httpd
```

### Why are we doing this?

`httpd -t` checks for configuration mistakes without stopping the live website.

If you restart Apache with a broken config, the site may go down.

---

## Step 7 - Map Hostnames on the Client Machine

On Windows, edit the hosts file as Administrator.

Open Notepad as Administrator, then open:

```text
C:\Windows\System32\drivers\etc\hosts
```

Add:

```text
10.10.10.101  site1.local
10.10.10.101  site2.local
10.10.10.101  wp.local
10.10.10.101  test.local
```

Save the file.

Flush the Windows DNS cache:

```cmd
ipconfig /flushdns
```

### Why are we doing this?

These `.local` hostnames do not exist on the public internet.

The Windows hosts file tells your computer:

```text
When I type site1.local, send the request to 10.10.10.101
```

---

## Step 8 - Verify Normally First

Try the normal setup first.

If the sites load correctly, **you do not need to modify the default SSL vhost**.

### Test from the server

Bypass DNS and force the hostname to the VM IP:

```bash
curl -I --resolve site1.local:80:10.10.10.101  http://site1.local/
curl -I --resolve site2.local:80:10.10.10.101  http://site2.local/
curl -I --resolve wp.local:80:10.10.10.101     http://wp.local/
```

Expected result:

```text
HTTP/1.1 301 Moved Permanently
Location: https://site1.local/
```

or similar for the other sites.

Now test HTTPS:

```bash
curl -k --resolve site1.local:443:10.10.10.101 https://site1.local/
curl -k --resolve site2.local:443:10.10.10.101 https://site2.local/
curl -k --resolve wp.local:443:10.10.10.101    https://wp.local/
curl -k --resolve test.local:443:10.10.10.101  https://test.local/
```

Expected results:

| URL | Expected Result |
|---|---|
| `http://site1.local` | Redirects to `https://site1.local/` |
| `http://site2.local` | Redirects to `https://site2.local/` |
| `http://wp.local` | Redirects to `https://wp.local/` |
| `https://site1.local` | Shows `This is Webserver 1` |
| `https://site2.local` | Shows `This is Webserver 2` |
| `https://wp.local` | Shows WordPress |
| `https://test.local` | Shows default index page |

### Test from the browser

On Windows, open:

```text
site1.local
site2.local
wp.local
test.local
```

They should upgrade to HTTPS and load the correct site.

> Because this lab uses a self-signed certificate, the browser will show a security warning. That is expected.

### If everything works

Stop here.

You do **not** need to name the default SSL vhost.

---

## Step 9 - Optional Troubleshooting: If the Default SSL Vhost Steals Requests

Only do this step if you see one of these symptoms:

- `https://site1.local` shows the default index page instead of Site 1.
- `https://site2.local` shows the default index page instead of Site 2.
- Only one HTTPS site works, while others fall back to the default page.
- Apache seems to ignore your named `*:443` virtual hosts.

### Why this may happen

Apache may have a default SSL virtual host defined in:

```text
/etc/httpd/conf.d/ssl.conf
```

It often looks like this:

```apache
<VirtualHost _default_:443>
```

If that block has no clear `ServerName`, it can become an ambiguous fallback for HTTPS requests.

In some setups this causes the default site to “steal” requests that should go to `site1.local`, `site2.local`, or `wp.local`.

### Fix: Name the default SSL vhost

Edit the SSL configuration:

```bash
vi /etc/httpd/conf.d/ssl.conf
```

Find:

```apache
<VirtualHost _default_:443>
```

Add a `ServerName` inside that block:

```apache
<VirtualHost _default_:443>
    ServerName test.local
```

It should look conceptually like this:

```apache
<VirtualHost _default_:443>
    ServerName test.local

    # existing SSL settings...
</VirtualHost>
```

Save and exit.

Test Apache:

```bash
httpd -t
```

Restart Apache:

```bash
systemctl restart httpd
```

Verify again:

```bash
curl -k --resolve site1.local:443:10.10.10.101 https://site1.local/
curl -k --resolve site2.local:443:10.10.10.101 https://site2.local/
curl -k --resolve wp.local:443:10.10.10.101    https://wp.local/
curl -k --resolve test.local:443:10.10.10.101  https://test.local/
```

### Why this is optional

If Apache is already matching the correct named virtual hosts, this step is not required.

The normal rule is:

```text
Correct named VirtualHost wins.
Default VirtualHost only handles unmatched requests.
```

This fix is only needed when the default SSL vhost is interfering.

---

## Useful Diagnostic Command

If virtual hosts are not behaving correctly, run:

```bash
httpd -S
```

This lists all loaded virtual hosts.

Check that each site appears with the correct `DocumentRoot`.

You should see entries similar to:

```text
port 80  site1.local  /var/www/html/site1
port 443 site1.local  /var/www/html/site1

port 80  site2.local  /var/www/html/site2
port 443 site2.local  /var/www/html/site2

port 80  wp.local     /var/www/html/wordpress
port 443 wp.local     /var/www/html/wordpress
```

If a site is missing, duplicated, or pointing to the wrong folder, check:

```bash
/etc/httpd/conf.d/vhosts.conf
/etc/httpd/conf.d/ssl.conf
```

---

## How Apache Chooses the Site

For HTTP:

```text
Browser requests http://site1.local
Apache checks Host header: site1.local
Apache matches <VirtualHost *:80> with ServerName site1.local
Apache redirects to https://site1.local/
```

For HTTPS:

```text
Browser requests https://site1.local
TLS handshake sends SNI: site1.local
HTTP request sends Host header: site1.local
Apache matches <VirtualHost *:443> with ServerName site1.local
Apache serves /var/www/html/site1
```

If nothing matches, Apache uses the default virtual host for that port.

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---|---|---|
| Typing `site1.local` shows default index page | No `*:80` virtual host or no redirect | Add `<VirtualHost *:80>` with `Redirect permanent` |
| Only one HTTPS site works, others show default page | Default SSL vhost may be interfering | Use Step 9 and add `ServerName test.local` to `_default_:443` |
| `421 Misdirected Request` | SNI and `Host:` header disagree | Test using `curl --resolve`, not manual `Host:` header |
| `403 Forbidden` | SELinux label or permission issue | Run `chown -R apache:apache /var/www/html` and `restorecon -Rv /var/www/html` |
| `httpd -t` fails with missing certificate | Cert/key not copied to `/etc/pki/tls` | Run Step 5 |
| Apache will not restart | Wrong `SSLCertificateFile` or `SSLCertificateKeyFile` path | Verify paths in `vhosts.conf` |
| Hostname does not resolve on Windows | Hosts file not saved correctly | Edit as Administrator and run `ipconfig /flushdns` |
| WordPress redirects to wrong URL | WordPress still uses old site URL | Go to `wp-admin → Settings → General` and set URL to `https://wp.local` |
| Browser shows certificate warning | Self-signed certificate | Expected in lab; accept warning |
| Browser says certificate name mismatch | Certificate was created for IP, not `.local` names | For lab, accept warning. For cleaner HTTPS, regenerate certificate with DNS SANs |

---

## WordPress URL Check

If `wp.local` loads but redirects incorrectly, log into WordPress and check:

```text
wp-admin → Settings → General
```

Set both fields to:

```text
https://wp.local
```

Example:

| Field | Value |
|---|---|
| WordPress Address (URL) | `https://wp.local` |
| Site Address (URL) | `https://wp.local` |

Save changes and log in again.

### Why are we doing this?

WordPress stores its own URLs in the database.

If these still point to the old IP or HTTP URL, WordPress can create redirect loops or mixed-content warnings.

---

## Final Checklist

- [ ] Site folders created:
  - `/var/www/html/site1`
  - `/var/www/html/site2`
  - `/var/www/html/wordpress`
- [ ] Test `index.html` files created for Site 1 and Site 2
- [ ] Default `/var/www/html/index.html` exists if you want `test.local` to show a fallback page
- [ ] Ownership set:

```bash
chown -R apache:apache /var/www/html
```

- [ ] SELinux labels restored:

```bash
restorecon -Rv /var/www/html
```

- [ ] `/etc/httpd/conf.d/vhosts.conf` created
- [ ] Each site has a `*:80` redirect block
- [ ] Each site has a `*:443` SSL block
- [ ] Each block has a unique `ServerName`
- [ ] Certificate and key exist in `/etc/pki/tls`
- [ ] Private key is protected with `600`
- [ ] `httpd -t` returns `Syntax OK`
- [ ] Apache restarted
- [ ] Windows hosts file updated
- [ ] `ipconfig /flushdns` run on Windows
- [ ] `site1.local`, `site2.local`, `wp.local`, and `test.local` load correctly
- [ ] Step 9 only used if the default SSL vhost is actually stealing requests

---
