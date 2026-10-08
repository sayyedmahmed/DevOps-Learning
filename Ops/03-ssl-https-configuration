# 03 - SSL Certificate & HTTPS Configuration

> **Series:** 3 of 4 | Previous: [02 - LAMP & WordPress](02-lamp-wordpress-installation.md) | Next: [04 - Security Hardening](04-security-hardening.md)

## Overview

This guide secures the site with **HTTPS** using a **self-signed certificate** that includes a **Subject Alternative Name (SAN)** for the VM's IP address.

### Why a SAN is required

Modern browsers ignore the certificate's Common Name (CN) and only trust names listed in the SAN field. A certificate with only `CN=10.10.10.101` triggers a name-mismatch error even after you trust it. Adding `IP:10.10.10.101` as a SAN fixes this.

### Self-signed vs. trusted

A self-signed certificate encrypts traffic but is not verified by a Certificate Authority, so browsers show a warning until you trust it manually. That is fine for a lab; production should use Let's Encrypt or a commercial CA.

**Prerequisites:** WordPress working over HTTP (Part 2), `mod_ssl` available. If `/etc/httpd/conf.d/ssl.conf` does not exist, install it first: `dnf install mod_ssl -y`.

---

## Step 1 - Reconstruct the Missing OpenSSL Config

Minimal AlmaLinux 10 does not ship `/etc/pki/tls/openssl.cnf`, which some OpenSSL commands expect.

```bash
vi /etc/pki/tls/openssl.cnf
```
Paste:
```
openssl_conf = default_modules

[default_modules]
ssl_conf = ssl_module

[ssl_module]
system_default = crypto_policy

[crypto_policy]
.include = /etc/crypto-policies/back-ends/opensslcnf.config
```
Save with `:wq`.

This minimal config hands cipher and protocol decisions to the **system-wide crypto policy**, so OpenSSL follows the same rules as the rest of the OS.

## Step 2 - Create a Working Directory

```bash
mkdir -p /opt/ssl
cd /opt/ssl
```
A neutral working area for generating the key and certificate before installing them.

## Step 3 - Generate the Self-Signed Certificate with SAN

```bash
openssl req -x509 -nodes -newkey rsa:2048 -keyout key.pem -out cert.pem -days 365 \
  -subj "/C=PK/ST=Punjab/L=Lahore/O=MyCompany/OU=IT/CN=10.10.10.101" \
  -addext "subjectAltName = IP:10.10.10.101" \
  -config /dev/null
```

| Option | Meaning |
|---|---|
| `req -x509` | Create a self-signed certificate directly |
| `-nodes` | Do not encrypt the private key (Apache can start unattended) |
| `-newkey rsa:2048` | Generate a new 2048-bit RSA key |
| `-keyout` / `-out` | Output paths for key and certificate |
| `-days 365` | Valid for one year |
| `-subj` | Certificate identity (country, state, city, org, unit, CN) |
| `-addext "subjectAltName = IP:..."` | Adds the SAN browsers require |
| `-config /dev/null` | Ignore config files so no defaults interfere |

**Verify:**
```bash
openssl x509 -in cert.pem -noout -subject -ext subjectAltName
```
Expected SAN:
```
IP Address:10.10.10.101
```

## Step 4 - Install the Certificate and Key

```bash
cp /opt/ssl/cert.pem /etc/pki/tls/certs/cert.pem
cp /opt/ssl/key.pem /etc/pki/tls/private/key.pem
chmod 600 /etc/pki/tls/private/key.pem
chown root:root /etc/pki/tls/private/key.pem
```

These are the standard RHEL-family locations, which also carry the correct SELinux labels. The key is made **readable by root only** (`600`) because anyone holding it can impersonate your server.

> **Never commit `key.pem` to GitHub.**

## Step 5 - Point Apache to the Certificate

```bash
vi /etc/httpd/conf.d/ssl.conf
```
Set:
```
SSLCertificateFile /etc/pki/tls/certs/cert.pem
SSLCertificateKeyFile /etc/pki/tls/private/key.pem
```

Test syntax, then restart:
```bash
httpd -t
systemctl restart httpd
```
`httpd -t` should print `Syntax OK`. Always test before restarting to avoid taking the site down on a typo.

## Step 6 - Verify the Served Certificate

```bash
openssl s_client -connect 10.10.10.101:443 -servername 10.10.10.101 </dev/null 2>/dev/null | openssl x509 -noout -subject -ext subjectAltName
```
Expected:
```
subject=CN = 10.10.10.101
X509v3 Subject Alternative Name:
    IP Address:10.10.10.101
```
This confirms Apache is serving the **new** certificate on port 443, not just that the file exists on disk.

## Step 7 - Redirect HTTP to HTTPS

```bash
vi /etc/httpd/conf.d/redirect.conf
```
```apache
<VirtualHost *:80>
    ServerName 10.10.10.101
    Redirect permanent / https://10.10.10.101/
</VirtualHost>
```

Disable the default welcome page, which can intercept port 80 requests:
```bash
mv /etc/httpd/conf.d/welcome.conf /etc/httpd/conf.d/welcome.conf.disabled
```

Test and restart:
```bash
httpd -t
systemctl restart httpd
```

**Verify:**
```bash
curl -I http://10.10.10.101
```
Expected:
```
HTTP/1.1 301 Moved Permanently
Location: https://10.10.10.101/
```
A `301` tells browsers and search engines the move is permanent.

## Step 8 - Update WordPress URLs

In **wp-admin → Settings → General** set:

- WordPress Address (URL): `https://10.10.10.101`
- Site Address (URL): `https://10.10.10.101`

Save and log in again. Without this, WordPress generates `http://` links, causing redirect loops or mixed-content warnings.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Browser says name mismatch | Missing SAN | Regenerate in Step 3, verify the SAN |
| `httpd -t` errors on SSL | `mod_ssl` missing or wrong paths | `dnf install mod_ssl`; recheck Step 5 |
| Still serves the old cert | Apache not restarted | `systemctl restart httpd`, repeat Step 6 |
| Port 443 refused | Firewall | `firewall-cmd --list-services` includes `https` |
| Redirect not working | Welcome page intercepting | Confirm `welcome.conf` is disabled |
| SELinux denial on key | Wrong file location/label | `restorecon -Rv /etc/pki/tls` |
| Browser warning remains | Expected for self-signed | Import `cert.pem` into the Windows trusted root store |

## Checklist

- [ ] Certificate has `IP Address:10.10.10.101` SAN
- [ ] Key is `600`, owned by root
- [ ] `httpd -t` returns `Syntax OK`
- [ ] Port 443 serves the new certificate
- [ ] `curl -I http://...` returns `301`
- [ ] WordPress URLs use `https://`

**Next:** [04 - Security Hardening](04-security-hardening.md)
