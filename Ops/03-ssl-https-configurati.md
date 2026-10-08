# 03 - SSL Certificate & HTTPS Configuration

> **Series:** 3 of 4 | Previous: [02 - LAMP & WordPress](02-lamp-wordpress-installation.md) | Next: [04 - Security Hardening](04-security-hardening.md)

## Overview

This guide secures the site with **HTTPS** using a **self-signed certificate**. We will generate the key and certificate, configure Apache to use them, and redirect all HTTP traffic to HTTPS.

### Why Self-Signed?

A self-signed certificate encrypts traffic (just like a commercial one), but it is not verified by a trusted Certificate Authority (CA). Browsers will show a warning ("Not Secure") until you manually trust the certificate. This is perfect for a local lab or internal testing environment. For public production sites, you would use Let's Encrypt or a commercial CA.

### Why SAN (Subject Alternative Name)?

Modern browsers ignore the old `Common Name` field for validation. They only check the `Subject Alternative Name` (SAN). If we don't explicitly add the IP address (`10.10.10.101`) to the SAN, the browser will throw a name-mismatch error even if you try to proceed.

**Prerequisites:**
- WordPress working over HTTP (from Part 2).
- `mod_ssl` installed. Check with `rpm -q mod_ssl`. If missing, run: `dnf install mod_ssl -y`.

---

## Step 1 - Prepare OpenSSL Config (AlmaLinux Specific)

Minimal AlmaLinux installations sometimes lack the default `/etc/pki/tls/openssl.cnf` file required by certain OpenSSL commands. We create a minimal config that defers to the system-wide crypto policy.

```bash
vi /etc/pki/tls/openssl.cnf
```

Paste the following content:
```ini
openssl_conf = default_modules

[default_modules]
ssl_conf = ssl_module

[ssl_module]
system_default = crypto_policy

[crypto_policy]
.include = /etc/crypto-policies/back-ends/opensslcnf.config
```

Save and exit (`:wq`).

### Why are we doing this?
OpenSSL needs a configuration file to know which encryption standards to use. By pointing it to the system crypto policy, we ensure our certificates follow the same security rules as the rest of the OS, avoiding compatibility issues.

---

## Step 2 - Create Working Directory

```bash
mkdir -p /opt/ssl
cd /opt/ssl
```

### Why are we doing this?
We keep temporary generation files in a neutral directory (`/opt/ssl`) before moving them to their final secure locations. This prevents cluttering system directories during creation.

---

## Step 3 - Generate Self-Signed Certificate with SAN

Run this single command to generate both the private key and the certificate:

```bash
openssl req -x509 -nodes -newkey rsa:2048 \
  -keyout key.pem \
  -out cert.pem \
  -days 365 \
  -subj "/C=PK/ST=Punjab/L=Lahore/O=MyCompany/OU=IT/CN=10.10.10.101" \
  -addext "subjectAltName = IP:10.10.10.101" \
  -config /dev/null
```

| Option | Meaning |
|---|---|
| `-x509` | Outputs a self-signed certificate instead of a CSR. |
| `-nodes` | No passphrase on the private key (allows Apache to start automatically). |
| `-newkey rsa:2048` | Generates a new 2048-bit RSA key pair. |
| `-days 365` | Certificate validity period. |
| `-subj` | Defines the identity fields (Country, State, Org, etc.). |
| `-addext "subjectAltName..."` | **Crucial:** Adds the IP address so browsers accept it. |
| `-config /dev/null` | Ignores external config files to prevent errors. |

**Verify the SAN was added correctly:**
```bash
openssl x509 -in cert.pem -noout -text | grep -A1 "Subject Alternative Name"
```
You should see:
```text
X509v3 Subject Alternative Name: 
    IP Address:10.10.10.101
```

### Why are we doing this?
This creates the cryptographic keys needed for HTTPS. The `-nodes` flag is important because if the key had a password, Apache would hang waiting for input every time it restarted. The SAN ensures modern browsers (Chrome, Edge, Firefox) recognize the IP address as valid.

---

## Step 4 - Install Certificate and Key

Move the generated files to standard RHEL/AlmaLinux locations and secure them.

```bash
cp /opt/ssl/cert.pem /etc/pki/tls/certs/cert.pem
cp /opt/ssl/key.pem /etc/pki/tls/private/key.pem

# Secure the private key (only root can read/write)
chmod 600 /etc/pki/tls/private/key.pem
chown root:root /etc/pki/tls/private/key.pem
```

### Why are we doing this?
- **Locations:** `/etc/pki/tls/certs/` and `/etc/pki/tls/private/` are the standard paths expected by Apache and SELinux policies on AlmaLinux.
- **Permissions:** The private key (`key.pem`) is the most sensitive file. If an attacker steals it, they can decrypt your traffic or impersonate your server. `600` ensures only the `root` user can access it.

---

## Step 5 - Configure Apache for SSL

Edit the SSL configuration file provided by `mod_ssl`.

```bash
vi /etc/httpd/conf.d/ssl.conf
```

Find and update these lines to point to your new files:
```apache
SSLCertificateFile      /etc/pki/tls/certs/cert.pem
SSLCertificateKeyFile   /etc/pki/tls/private/key.pem
```

Test the configuration syntax before restarting:
```bash
httpd -t
```
If it says `Syntax OK`, restart Apache:
```bash
systemctl restart httpd
```

### Why are we doing this?
Apache needs to be told exactly where the certificate and key are located. Running `httpd -t` checks for typos without crashing the live server.

---

## Step 6 - Verify HTTPS Connection

Check if Apache is actually serving the new certificate on port 443.

```bash
openssl s_client -connect 10.10.10.101:443 </dev/null 2>/dev/null | openssl x509 -noout -subject -ext subjectAltName
```

Expected Output:
```text
subject=C = PK, ST = Punjab, L = Lahore, O = MyCompany, OU = IT, CN = 10.10.10.101
X509v3 Subject Alternative Name: 
    IP Address:10.10.10.101
```

### Why are we doing this?
This confirms that the web server is successfully loading the certificate from disk and presenting it to clients. If this fails, the browser will not connect securely.

---

## Step 7 - Redirect HTTP to HTTPS

Force all visitors to use the secure connection.

Create a new configuration file:
```bash
vi /etc/httpd/conf.d/redirect.conf
```

Add this VirtualHost block:
```apache
<VirtualHost *:80>
    ServerName 10.10.10.101
    Redirect permanent / https://10.10.10.101/
</VirtualHost>
```

Disable the default welcome page to prevent conflicts:
```bash
mv /etc/httpd/conf.d/welcome.conf /etc/httpd/conf.d/welcome.conf.disabled
```

Test and restart:
```bash
httpd -t
systemctl restart httpd
```

**Verify the redirect:**
```bash
curl -I http://10.10.10.101
```
Look for:
```text
HTTP/1.1 301 Moved Permanently
Location: https://10.10.10.101/
```

### Why are we doing this?
Users often type `http://` or click old links. This ensures they are automatically upgraded to the encrypted `https://` version. The `301` status code tells browsers and search engines that this move is permanent.

---

## Step 8 - Update WordPress URLs

WordPress stores its own URL settings in the database. If these remain as `http://`, WordPress will generate insecure links, causing mixed-content warnings or redirect loops.

1. Log in to `https://10.10.10.101/wp-admin`.
2. Go to **Settings → General**.
3. Change both fields to use `https`:
   - **WordPress Address (URL):** `https://10.10.10.101`
   - **Site Address (URL):** `https://10.10.10.101`
4. Click **Save Changes**.
5. You will be logged out. Log back in.

### Why are we doing this?
Without this step, your site might load securely initially, but clicking any menu item could drop you back to an insecure HTTP connection, triggering browser security warnings.

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---|---|---|
| Browser shows "NET::ERR_CERT_COMMON_NAME_INVALID" | Missing SAN | Re-run Step 3 ensuring `-addext "subjectAltName = IP:..."` is included. |
| `httpd -t` fails with "Invalid command 'SSLCertificateFile'" | `mod_ssl` not installed | Run `dnf install mod_ssl -y`. |
| Connection Refused on Port 443 | Firewall blocking | Ensure `firewall-cmd --list-services` includes `https`. |
| Infinite Redirect Loop | WordPress URL mismatch | Ensure Step 8 is completed; check `wp_options` table in DB if locked out. |
| Permission Denied on Key | Wrong permissions | Re-run Step 4: `chmod 600 /etc/pki/tls/private/key.pem`. |

## Checklist

- [ ] OpenSSL config created in `/etc/pki/tls/openssl.cnf`
- [ ] Certificate generated with correct SAN (IP Address)
- [ ] Private key secured with `600` permissions
- [ ] Apache configured to use new cert/key paths
- [ ] `httpd -t` passes syntax check
- [ ] HTTPS redirects work via `curl`
- [ ] WordPress Admin settings updated to `https://`

---

## References

For more documentation on creating self-signed certificates, I followed this guide:

- Linuxize: [Creating a Self-Signed SSL Certificate](https://linuxize.com/post/creating-a-self-signed-ssl-certificate/)

> **Note:** The linked guide uses basic `openssl` commands. Our implementation adds specific flags (`-addext`) and AlmaLinux-specific path configurations to ensure compatibility with modern browsers and SELinux.

**Next:** [04 - Security Hardening](04-security-hardening.md)
