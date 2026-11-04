# Apache HTTP Server — Implementation and Operations Reference

> **Platform:** AlmaLinux / RHEL / Rocky Linux (`httpd` package)
> **Last updated:** 2026-11-04
> **Maintained by:** Infrastructure Engineering

---

## Table of Contents

- [1. Overview and Architecture](#1-overview-and-architecture)
  - [1.1 Role of Apache httpd](#11-role-of-apache-httpd)
  - [1.2 Core Terminology](#12-core-terminology)
- [2. Proxy Architecture Concepts](#2-proxy-architecture-concepts)
  - [2.1 Forward Proxy](#21-forward-proxy)
  - [2.2 Reverse Proxy](#22-reverse-proxy)
  - [2.3 Load Balancer](#23-load-balancer)
  - [2.4 Virtual Hosts](#24-virtual-hosts)
  - [2.5 TLS / HTTPS](#25-tls--https)
- [3. Installation and Service Management](#3-installation-and-service-management)
  - [3.1 Package Installation](#31-package-installation)
  - [3.2 Service Enablement and Lifecycle](#32-service-enablement-and-lifecycle)
  - [3.3 Firewall Configuration](#33-firewall-configuration)
  - [3.4 Initial Verification](#34-initial-verification)
  - [3.5 Default Document Root](#35-default-document-root)
- [4. Configuration File Reference](#4-configuration-file-reference)
  - [4.1 Directory Layout](#41-directory-layout)
  - [4.2 Operational Commands](#42-operational-commands)
  - [4.3 Configuration Change Workflow](#43-configuration-change-workflow)
- [5. Virtual Host Implementation](#5-virtual-host-implementation)
  - [5.1 Document Root Preparation](#51-document-root-preparation)
  - [5.2 Virtual Host Configuration Files](#52-virtual-host-configuration-files)
  - [5.3 Directive Reference](#53-directive-reference)
  - [5.4 Validation and Activation](#54-validation-and-activation)
  - [5.5 DNS / Host Resolution for Testing](#55-dns--host-resolution-for-testing)
  - [5.6 Virtual Host Selection Logic](#56-virtual-host-selection-logic)
- [6. TLS / HTTPS Configuration](#6-tls--https-configuration)
  - [6.1 mod_ssl Installation](#61-mod_ssl-installation)
  - [6.2 Let's Encrypt (Production)](#62-lets-encrypt-production)
  - [6.3 Self-Signed Certificate (Internal / Staging)](#63-self-signed-certificate-internal--staging)
- [7. Backend Service Deployment](#7-backend-service-deployment)
  - [7.1 SELinux and Firewall Prerequisites](#71-selinux-and-firewall-prerequisites)
  - [7.2 Document Preparation](#72-document-preparation)
  - [7.3 Backend Virtual Host Configuration](#73-backend-virtual-host-configuration)
  - [7.4 Verification](#74-verification)
- [8. Reverse Proxy Configuration](#8-reverse-proxy-configuration)
  - [8.1 Module Verification](#81-module-verification)
  - [8.2 Single-Backend Proxy](#82-single-backend-proxy)
  - [8.3 Path-Based Routing](#83-path-based-routing)
  - [8.4 TLS Termination](#84-tls-termination)
- [9. Load Balancing](#9-load-balancing)
  - [9.1 Module Verification](#91-module-verification)
  - [9.2 Balancer Pool Configuration](#92-balancer-pool-configuration)
  - [9.3 Round-Robin Verification](#93-round-robin-verification)
  - [9.4 Backend Failure Handling](#94-backend-failure-handling)
  - [9.5 Balancing Algorithms and Options](#95-balancing-algorithms-and-options)
  - [9.6 Failure and Redundancy Options](#96-failure-and-redundancy-options)
- [10. Forward Proxy Configuration](#10-forward-proxy-configuration)
  - [10.1 Configuration](#101-configuration)
  - [10.2 Activation and Verification](#102-activation-and-verification)
  - [10.3 Access Control Extensions](#103-access-control-extensions)
- [11. Apache-Specific Features](#11-apache-specific-features)
  - [11.1 .htaccess Per-Directory Configuration](#111-htaccess-per-directory-configuration)
  - [11.2 Module Management](#112-module-management)
  - [11.3 Multi-Processing Modules (MPM)](#113-multi-processing-modules-mpm)
- [12. Troubleshooting Reference](#12-troubleshooting-reference)
  - [12.1 Symptom-to-Cause Matrix](#121-symptom-to-cause-matrix)
  - [12.2 Diagnostic Command Sequence](#122-diagnostic-command-sequence)
- [13. Quick Reference](#13-quick-reference)
  - [13.1 Essential Paths and Commands](#131-essential-paths-and-commands)
  - [13.2 Directive Summary](#132-directive-summary)
  - [13.3 Standard Change Workflow](#133-standard-change-workflow)

---

## 1. Overview and Architecture

### 1.1 Role of Apache httpd

Apache HTTP Server (httpd) is a production-grade web server capable of serving static and dynamic content, functioning as a reverse proxy, load balancer, and forward proxy. It is one of the most widely deployed HTTP servers in enterprise environments.

```
  Client (Browser)                       SERVER (httpd)
 +----------------+   HTTP Request     +------------------------+
 |  Chrome / curl | ---------------->  |  httpd process         |
 |                |                    |    |                   |
 |                | <----------------  |    v                   |
 +----------------+   HTTP Response    |  /var/www/html/        |
                                       |  index.html            |
                                       +------------------------+
```

### 1.2 Core Terminology

| Term | Definition |
|------|-----------|
| Web server | Process that serves HTTP/HTTPS responses |
| DocumentRoot | Filesystem directory containing site content |
| index.html | Default document served for directory requests |
| Listen | Directive that binds httpd to a TCP port |
| Module | Loadable feature extension (e.g., `mod_ssl`, `mod_proxy`) |
| VirtualHost | Configuration block mapping a domain/port to a DocumentRoot |

---

## 2. Proxy Architecture Concepts

### 2.1 Forward Proxy

A forward proxy acts as an intermediary for **clients**. Clients are explicitly configured to route traffic through the proxy. Origin servers see only the proxy's address.

```
  Clients                 FORWARD PROXY                Internet
 +--------+              +-------------+             +------------+
 | Host 1 |--+           |             |-----------> | External   |
 | Host 2 |--+---------->|   Proxy     |             | Websites   |
 | Host 3 |--+           |             |             +------------+
 +--------+              +-------------+
```

**Use cases:** content filtering, caching, client IP masking, egress control.

### 2.2 Reverse Proxy

A reverse proxy acts as an intermediary for **backend servers**. Clients are unaware of the backend topology.

```
  Clients                REVERSE PROXY               Backend Servers
 +--------+             +-------------+             +------------+
 | User   |----------->|   Proxy     |----------->|  Backend   |
 +--------+             +-------------+             +------------+
```

**Use cases:** topology abstraction, centralized TLS termination, unified access control, load distribution.

### 2.3 Load Balancer

A load balancer is a reverse proxy that distributes requests across multiple backend members.

```
                         +--> Backend 1
  Clients --> BALANCER --|--> Backend 2
                         +--> Backend 3
```

**Use cases:** horizontal scaling, high availability, rolling maintenance.

### 2.4 Virtual Hosts

A single httpd instance serves multiple websites, selected by the `Host` header in the incoming request.

```
                         +--> site1.local --> /var/www/site1
  Browser --> httpd :80 -|
                         +--> site2.local --> /var/www/site2
```

### 2.5 TLS / HTTPS

HTTP with Transport Layer Security encryption. A certificate authenticates the server identity.

```
 HTTP  (port 80):   Client --- plaintext ---------> Server
 HTTPS (port 443):  Client === TLS-encrypted ===> Server
```

---

## 3. Installation and Service Management

### 3.1 Package Installation

```bash
sudo dnf install httpd -y
httpd -v
```

> **Note:** On Debian/Ubuntu the package is `apache2`, configuration resides under `/etc/apache2/`, and site management uses `a2ensite` / `a2enmod`. This document targets the RHEL family exclusively.

### 3.2 Service Enablement and Lifecycle

```bash
sudo systemctl enable --now httpd
sudo systemctl status httpd
```

| Command | Purpose |
|---------|---------|
| `systemctl start httpd` | Start the service |
| `systemctl stop httpd` | Stop the service |
| `systemctl restart httpd` | Full stop/start — **required after adding or removing `Listen` directives** |
| `systemctl reload httpd` | Graceful configuration reload (no dropped connections) |
| `systemctl enable httpd` | Enable auto-start at boot |

### 3.3 Firewall Configuration

```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload
```

### 3.4 Initial Verification

```bash
hostname -I
curl -I http://localhost
```

Open `http://<SERVER_IP>` in a browser. The Apache test page is served by `/etc/httpd/conf.d/welcome.conf` and appears only while `/var/www/html` contains no `index.html`. Deploying a custom `index.html` removes it automatically.

### 3.5 Default Document Root

```bash
sudo vim /var/www/html/index.html
```

```html
<!DOCTYPE html>
<html>
<head><title>Apache httpd</title></head>
<body style="font-family:Arial;text-align:center;margin-top:80px">
  <h1>Apache httpd operational.</h1>
</body>
</html>
```

No service restart is required for static content changes.

---

## 4. Configuration File Reference

### 4.1 Directory Layout

```
/etc/httpd/
  ├── conf/httpd.conf              # Main configuration (contains "Listen 80")
  ├── conf.d/*.conf                # Site/feature configs (auto-included; *.conf only)
  └── conf.modules.d/*.conf        # Module loading directives

/var/www/html/                     # Default DocumentRoot

/var/log/httpd/
  ├── access_log                   # Request log
  └── error_log                    # Error/diagnostic log
```

> **Important:** Files in `conf.d/` are included only if they carry the `.conf` extension. Renaming a file to `.off` effectively disables it without deletion.

### 4.2 Operational Commands

| Task | Command |
|------|---------|
| Validate configuration syntax | `sudo httpd -t` |
| Display all virtual hosts | `sudo httpd -S` |
| List loaded modules | `httpd -M` |
| Graceful reload | `sudo systemctl reload httpd` |
| Inspect listening ports | `sudo ss -tlnp \| grep httpd` |
| Tail error log | `sudo tail -f /var/log/httpd/error_log` |

`httpd -S` outputs every configured virtual host and the file that defines it — essential for diagnosing routing issues.

### 4.3 Configuration Change Workflow

> **Golden rule:** After any configuration change, execute `sudo httpd -t` before reloading. If a `Listen` directive was added or removed, use `systemctl restart httpd` instead of `reload`.

---

## 5. Virtual Host Implementation

**Objective:** Serve `site1.local` and `site2.local` from a single httpd instance.

### 5.1 Document Root Preparation

```bash
sudo mkdir -p /var/www/site1 /var/www/site2
echo '<h1>This is SITE 1</h1>' | sudo tee /var/www/site1/index.html
echo '<h1>This is SITE 2</h1>' | sudo tee /var/www/site2/index.html
sudo chmod -R 755 /var/www
sudo restorecon -Rv /var/www          # Restore SELinux context
```

### 5.2 Virtual Host Configuration Files

**File:** `/etc/httpd/conf.d/site1.conf`

```apache
<VirtualHost *:80>
    ServerName site1.local
    DocumentRoot /var/www/site1
    <Directory /var/www/site1>
        Require all granted
        Options -Indexes
    </Directory>
    ErrorLog  /var/log/httpd/site1.error.log
    CustomLog /var/log/httpd/site1.access.log combined
</VirtualHost>
```

**File:** `/etc/httpd/conf.d/site2.conf`

```apache
<VirtualHost *:80>
    ServerName site2.local
    DocumentRoot /var/www/site2
    <Directory /var/www/site2>
        Require all granted
        Options -Indexes
    </Directory>
    ErrorLog  /var/log/httpd/site2.error.log
    CustomLog /var/log/httpd/site2.access.log combined
</VirtualHost>
```

### 5.3 Directive Reference

| Directive | Purpose |
|-----------|---------|
| `<VirtualHost *:80>` | Defines a site on any interface, port 80 |
| `ServerName` | Domain name this virtual host responds to |
| `DocumentRoot` | Filesystem path to site content |
| `<Directory>` | Access-control block for the specified path |
| `Require all granted` | Permits all clients. **Omitting this results in 403 Forbidden.** |
| `Options -Indexes` | Disables directory listing when no index file exists |
| `ErrorLog` / `CustomLog` | Per-site log files |

### 5.4 Validation and Activation

```bash
sudo httpd -t
sudo systemctl reload httpd
sudo httpd -S
```

### 5.5 DNS / Host Resolution for Testing

From the server:

```bash
curl -H "Host: site1.local" http://localhost
curl -H "Host: site2.local" http://localhost
```

From client workstations, add the following to the local hosts file (`/etc/hosts` on Linux/macOS; `C:\Windows\System32\drivers\etc\hosts` on Windows):

```
<SERVER_IP>   site1.local site2.local
```

### 5.6 Virtual Host Selection Logic

```
 Incoming request with Host: site2.local
        │
        ▼
 Compare against each ServerName / ServerAlias
        │
  match? ──yes──► Use that VirtualHost
        │
        no
        ▼
 Use the FIRST VirtualHost loaded for that port
```

"First loaded" is determined by alphabetical file order in `conf.d/`. To designate an explicit default site, name its file `00-default.conf`.

---

## 6. TLS / HTTPS Configuration

| Method | Applicability |
|--------|--------------|
| [6.2 Let's Encrypt](#62-lets-encrypt-production) | Public domains with internet-reachable servers |
| [6.3 Self-Signed](#63-self-signed-certificate-internal--staging) | Internal networks, staging environments (browser warning expected) |

### 6.1 mod_ssl Installation

Required for both methods:

```bash
sudo dnf install mod_ssl -y
```

> **Critical:** `mod_ssl` installs `/etc/httpd/conf.d/ssl.conf`, which already contains `Listen 443 https`. Do **not** add a duplicate `Listen 443` directive elsewhere, or httpd will fail with *"Address already in use"*.

### 6.2 Let's Encrypt (Production)

**Prerequisites (all must be satisfied):**

- A publicly registered domain (e.g., `example.com`) with an A record pointing to the server's public IP.
- Ports 80 and 443 reachable from the internet.
- A `<VirtualHost>` with `ServerName example.com` already configured.

> Let's Encrypt cannot issue certificates for `.local` names or RFC 1918 addresses. Use self-signed certificates for those.

```bash
sudo dnf install epel-release -y
sudo dnf install certbot python3-certbot-apache -y
sudo certbot --apache -d example.com -d www.example.com
```

Respond to the interactive prompts; select HTTP-to-HTTPS redirect when offered. Certbot modifies the Apache configuration and reloads the service automatically.

**Certificate renewal** (certificates are valid 90 days; auto-renewal timer is installed):

```bash
sudo systemctl list-timers | grep certbot
sudo certbot renew --dry-run
sudo certbot certificates
```

### 6.3 Self-Signed Certificate (Internal / Staging)

#### Step 1 — Generate the certificate and key

```bash
sudo mkdir -p /etc/httpd/ssl
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/httpd/ssl/site1.key \
  -out    /etc/httpd/ssl/site1.crt \
  -subj "/CN=site1.local"
sudo chmod 600 /etc/httpd/ssl/site1.key
sudo restorecon -Rv /etc/httpd/ssl
```

| Option | Purpose |
|--------|---------|
| `-x509` | Generate a self-signed certificate directly |
| `-nodes` | No passphrase on the private key |
| `-days 365` | Validity period: 1 year |
| `-keyout` / `-out` | Private key (secret) / public certificate |
| `-subj "/CN=..."` | Common Name (site identity) |

#### Step 2 — Virtual host configuration

**File:** `/etc/httpd/conf.d/site1.conf` (replaces the port-80-only version)

```apache
# Redirect all HTTP traffic to HTTPS
<VirtualHost *:80>
    ServerName site1.local
    Redirect permanent / https://site1.local/
</VirtualHost>

# HTTPS virtual host
<VirtualHost *:443>
    ServerName site1.local
    DocumentRoot /var/www/site1
    SSLEngine on
    SSLCertificateFile    /etc/httpd/ssl/site1.crt
    SSLCertificateKeyFile /etc/httpd/ssl/site1.key
    SSLProtocol all -SSLv3 -TLSv1 -TLSv1.1
    <Directory /var/www/site1>
        Require all granted
    </Directory>
</VirtualHost>
```

| Directive | Purpose |
|-----------|---------|
| `Redirect permanent / https://...` | 301 redirect to HTTPS |
| `<VirtualHost *:443>` | HTTPS listener |
| `SSLEngine on` | Enable TLS |
| `SSLCertificateFile` | Public certificate path |
| `SSLCertificateKeyFile` | Private key path |
| `SSLProtocol all -SSLv3 -TLSv1 -TLSv1.1` | Permit only TLS 1.2+ |

#### Step 3 — Validate and test

```bash
sudo httpd -t
sudo systemctl reload httpd
curl -k https://site1.local --resolve site1.local:443:127.0.0.1
```

`-k` suppresses the self-signed certificate warning. Browsers will display a certificate warning; proceed via **Advanced → Proceed**.

---

## 7. Backend Service Deployment

Three lightweight static backends are deployed on ports 8081, 8082, and 8083 to serve as upstream targets for reverse-proxy and load-balancing configurations.

> Replace `10.10.10.210` throughout with the actual server IP (`hostname -I`).

```
                             +--> Backend 1  10.10.10.210:8081
  FRONT DOOR :80  --proxy---|--> Backend 2  10.10.10.210:8082
  (httpd)                   +--> Backend 3  10.10.10.210:8083
```

### 7.1 SELinux and Firewall Prerequisites

```bash
# Permit httpd to initiate outbound connections to backends
sudo setsebool -P httpd_can_network_connect 1

# Label ports 8081-8083 as HTTP ports for SELinux
sudo dnf install policycoreutils-python-utils -y
sudo semanage port -a -t http_port_t -p tcp 8081-8083
# If semanage reports "already defined", use -m instead of -a:
# sudo semanage port -m -t http_port_t -p tcp 8081-8083

# Open backend ports in the firewall
sudo firewall-cmd --permanent --add-port=8081-8083/tcp
sudo firewall-cmd --reload
```

### 7.2 Document Preparation

```bash
for i in 1 2 3; do
  sudo mkdir -p /var/www/b$i
  echo "BACKEND $i" | sudo tee /var/www/b$i/index.html
done
sudo chmod -R 755 /var/www
sudo restorecon -Rv /var/www
```

### 7.3 Backend Virtual Host Configuration

Each backend requires two elements: a `Listen` directive (opens the port) and a `<VirtualHost>` block (defines what to serve).

**File:** `/etc/httpd/conf.d/backend1.conf`

```apache
Listen 8081
<VirtualHost *:8081>
    DocumentRoot /var/www/b1
    <Directory /var/www/b1>
        Require all granted
    </Directory>
</VirtualHost>
```

**File:** `/etc/httpd/conf.d/backend2.conf`

```apache
Listen 8082
<VirtualHost *:8082>
    DocumentRoot /var/www/b2
    <Directory /var/www/b2>
        Require all granted
    </Directory>
</VirtualHost>
```

**File:** `/etc/httpd/conf.d/backend3.conf`

```apache
Listen 8083
<VirtualHost *:8083>
    DocumentRoot /var/www/b3
    <Directory /var/www/b3>
        Require all granted
    </Directory>
</VirtualHost>
```

### 7.4 Verification

```bash
sudo httpd -t
sudo systemctl restart httpd          # restart required — new Listen directives added

curl http://10.10.10.210:8081
curl http://10.10.10.210:8082
curl http://10.10.10.210:8083
```

Expected output: `BACKEND 1`, `BACKEND 2`, `BACKEND 3`.

---

## 8. Reverse Proxy Configuration

**Objective:** Port 80 transparently serves content originating from Backend 1 (port 8081).

```
 curl --> httpd :80 --ProxyPass--> 10.10.10.210:8081 (BACKEND 1)
```

### 8.1 Module Verification

On RHEL-family systems, proxy modules are enabled by default via `/etc/httpd/conf.modules.d/00-proxy.conf`:

```bash
httpd -M | grep proxy
```

Required modules: `proxy_module`, `proxy_http_module`, `proxy_balancer_module`, `proxy_connect_module`.

### 8.2 Single-Backend Proxy

**File:** `/etc/httpd/conf.d/front.conf`

```apache
<VirtualHost *:80>
    ServerName proxy.local
    ProxyPreserveHost On
    ProxyPass        / http://10.10.10.210:8081/
    ProxyPassReverse / http://10.10.10.210:8081/
</VirtualHost>
```

| Directive | Purpose |
|-----------|---------|
| `ServerName proxy.local` | Responds to requests with `Host: proxy.local` |
| `ProxyPreserveHost On` | Forwards the original `Host` header to the backend |
| `ProxyPass / http://...` | Routes all requests to Backend 1 |
| `ProxyPassReverse` | Rewrites `Location` headers in backend redirects so clients never see the backend address |

Apache automatically appends `X-Forwarded-For`, `X-Forwarded-Host`, and `X-Forwarded-Server` headers.

**Validation:**

```bash
sudo httpd -t
sudo systemctl reload httpd
curl -H "Host: proxy.local" http://localhost
```

Expected output: `BACKEND 1`. The request reached port 80, but content was served from port 8081.

### 8.3 Path-Based Routing

```apache
<VirtualHost *:80>
    ServerName proxy.local
    ProxyPass        /one/ http://10.10.10.210:8081/
    ProxyPassReverse /one/ http://10.10.10.210:8081/
    ProxyPass        /two/ http://10.10.10.210:8082/
    ProxyPassReverse /two/ http://10.10.10.210:8082/
</VirtualHost>
```

```
/one/  --> BACKEND 1
/two/  --> BACKEND 2
```

> **Ordering rule:** Place more specific paths before less specific ones.

### 8.4 TLS Termination

Terminate TLS at the proxy; backends receive plain HTTP.

```apache
<VirtualHost *:443>
    ServerName proxy.local
    SSLEngine on
    SSLCertificateFile    /etc/httpd/ssl/site1.crt
    SSLCertificateKeyFile /etc/httpd/ssl/site1.key
    ProxyPreserveHost On
    RequestHeader set X-Forwarded-Proto "https"
    ProxyPass        / http://10.10.10.210:8081/
    ProxyPassReverse / http://10.10.10.210:8081/
</VirtualHost>
```

```
 Browser ==== HTTPS ====> httpd ---- plain HTTP ----> Backend
```

The backend requires no TLS configuration. The `X-Forwarded-Proto` header informs the backend that the original request was encrypted.

---

## 9. Load Balancing

**Objective:** Distribute incoming requests across all three backends.

```
                         +--> BACKEND 1
 Browser --> httpd :80 --|--> BACKEND 2
                         +--> BACKEND 3
```

### 9.1 Module Verification

```bash
httpd -M | grep -E "balancer|lbmethod"
```

Required: `proxy_balancer_module` and `lbmethod_byrequests_module` (enabled by default on RHEL).

### 9.2 Balancer Pool Configuration

**File:** `/etc/httpd/conf.d/front.conf` (replaces the single-backend proxy)

```apache
<Proxy "balancer://mygroup">
    BalancerMember http://10.10.10.210:8081
    BalancerMember http://10.10.10.210:8082
    BalancerMember http://10.10.10.210:8083
    ProxySet lbmethod=byrequests
</Proxy>

<VirtualHost *:80>
    ServerName proxy.local
    ProxyPreserveHost On
    ProxyPass        / balancer://mygroup/
    ProxyPassReverse / balancer://mygroup/
</VirtualHost>
```

- `<Proxy "balancer://mygroup">` defines a named pool of backend members.
- `ProxyPass / balancer://mygroup/` directs all traffic to the pool.

```bash
sudo httpd -t
sudo systemctl reload httpd
```

### 9.3 Round-Robin Verification

```bash
for i in 1 2 3 4 5 6; do curl -s -H "Host: proxy.local" http://localhost; done
```

Expected output (sequential rotation):

```
BACKEND 1
BACKEND 2
BACKEND 3
BACKEND 1
BACKEND 2
BACKEND 3
```

### 9.4 Backend Failure Handling

Simulate a backend outage:

```bash
sudo mv /etc/httpd/conf.d/backend2.conf /etc/httpd/conf.d/backend2.off
sudo systemctl restart httpd

for i in 1 2 3 4 5 6; do curl -s -H "Host: proxy.local" http://localhost; done
```

Backend 2 stops receiving traffic; Backends 1 and 3 continue serving. Restore:

```bash
sudo mv /etc/httpd/conf.d/backend2.off /etc/httpd/conf.d/backend2.conf
sudo systemctl restart httpd
```

> If `503 Service Unavailable` is returned temporarily, Apache is observing its retry interval. Add `retry=5` to each `BalancerMember` to reduce the retry delay to 5 seconds.

### 9.5 Balancing Algorithms and Options

| Algorithm | Directive | Behaviour |
|-----------|-----------|-----------|
| Round robin (default) | `lbmethod=byrequests` | Sequential rotation by request count |
| By traffic | `lbmethod=bytraffic` | Distributes based on bytes transferred |
| Least busy | `lbmethod=bybusyness` | Routes to the member with fewest active connections |
| Weighted | `loadfactor=N` on a member | Higher-capacity members receive proportionally more traffic |
| Sticky sessions | `stickysession=ROUTEID` | Pins a client to a specific member (requires cookie support) |

**Weighted example:**

```apache
BalancerMember http://10.10.10.210:8081 loadfactor=3
BalancerMember http://10.10.10.210:8082 loadfactor=1
```

**Least-busy example:**

```apache
ProxySet lbmethod=bybusyness
```

### 9.6 Failure and Redundancy Options

```apache
<Proxy "balancer://mygroup">
    BalancerMember http://10.10.10.210:8081 retry=5 timeout=5
    BalancerMember http://10.10.10.210:8082 retry=5 timeout=5
    BalancerMember http://10.10.10.210:8083 status=+H
    ProxySet lbmethod=byrequests
</Proxy>
```

| Option | Purpose |
|--------|---------|
| `retry=5` | Wait 5 seconds after a failure before retrying the member |
| `timeout=5` | Abort connection to a member after 5 seconds |
| `status=+H` | Hot standby — receives traffic only when all active members are unavailable |
| `status=+D` | Member administratively disabled (drained) |

---

## 10. Forward Proxy Configuration

Apache supports full forward-proxy functionality, including HTTPS CONNECT tunnelling via `mod_proxy_connect`.

```
 curl on this host --> httpd forward proxy :3128 --> external website
```

> **Security:** An unrestricted forward proxy will be exploited within hours. Always restrict access by IP and never expose the proxy port to untrusted networks.

### 10.1 Configuration

**File:** `/etc/httpd/conf.d/forward.conf`

```apache
Listen 3128

<VirtualHost *:3128>
    ProxyRequests On
    ProxyVia On
    <Proxy "*">
        Require ip 127.0.0.1
    </Proxy>
    CustomLog /var/log/httpd/forward_access.log combined
</VirtualHost>
```

| Directive | Purpose |
|-----------|---------|
| `Listen 3128` | Binds the forward proxy to port 3128 |
| `ProxyRequests On` | Enables forward-proxy mode |
| `ProxyVia On` | Inserts a `Via` header identifying the proxy hop |
| `Require ip 127.0.0.1` | Restricts proxy usage to the local host |

### 10.2 Activation and Verification

```bash
sudo httpd -t
sudo systemctl restart httpd          # new Listen directive

curl -x http://127.0.0.1:3128 http://example.com
curl -x http://127.0.0.1:3128 -I https://example.com
sudo tail -n 5 /var/log/httpd/forward_access.log
```

### 10.3 Access Control Extensions

To permit a specific subnet:

```apache
<Proxy "*">
    Require ip 192.168.1.0/24
</Proxy>
```

Open the port in the firewall:

```bash
sudo firewall-cmd --permanent --add-port=3128/tcp
sudo firewall-cmd --reload
sudo systemctl restart httpd
```

If httpd fails to bind port 3128, verify the SELinux port label:

```bash
sudo semanage port -l | grep 3128
```

**Site blocking (optional):**

```apache
ProxyBlock facebook.com tiktok.com
```

**Forward vs. Reverse proxy — operational distinction:**

```
FORWARD:  curl -x http://127.0.0.1:3128 http://example.com   (client is proxy-aware)
REVERSE:  curl -H "Host: proxy.local" http://localhost       (client is proxy-unaware)
```

---

## 11. Apache-Specific Features

### 11.1 .htaccess Per-Directory Configuration

Apache can read `.htaccess` files from content directories at request time, enabling configuration changes without editing the main configuration or reloading the service.

```
/var/www/site1/
  ├── index.html
  └── .htaccess        ← per-directory rules
```

Enable `.htaccess` processing for a directory:

```apache
<Directory /var/www/site1>
    AllowOverride All
    Require all granted
</Directory>
```

Example `.htaccess`:

```apache
Redirect permanent /old.html /new.html

<Files "secret.txt">
    Require all denied
</Files>
```

**Trade-off:** `.htaccess` is convenient in shared-hosting scenarios but incurs a per-request filesystem check in every directory along the path. Prefer main-configuration directives when full server control is available. Nginx does not support an equivalent mechanism.

### 11.2 Module Management

```bash
httpd -M                              # List all loaded modules
ls /etc/httpd/conf.modules.d/         # LoadModule directives reside here
```

### 11.3 Multi-Processing Modules (MPM)

```bash
httpd -V | grep -i mpm
```

| MPM | Model |
|-----|-------|
| `prefork` | One process per request (legacy) |
| `worker` | Multi-process, multi-threaded |
| `event` | Threaded with efficient idle-connection handling (**default**) |

---

## 12. Troubleshooting Reference

### 12.1 Symptom-to-Cause Matrix

| Symptom | Likely Cause | Resolution |
|---------|-------------|------------|
| `httpd -t` reports syntax error | Configuration typo | Read the file name and line number in the error output |
| *"Could not reliably determine the server's fully qualified domain name"* | No `ServerName` set globally | Add `ServerName localhost` to `httpd.conf` (cosmetic warning) |
| Connection refused | Nothing listening on the target port | Verify `ss -tlnp \| grep httpd`; check `Listen` directives |
| 403 Forbidden | Missing `Require all granted`, incorrect filesystem permissions, or SELinux mislabel | Add the directive; check `ls -lZ`; run `restorecon -Rv /var/www` |
| 404 Not Found | Incorrect `DocumentRoot` or missing file | Verify path and file existence |
| 503 Service Unavailable (proxy) | Backend unreachable or SELinux blocking outbound connections | Confirm backend is running; `setsebool -P httpd_can_network_connect 1` |
| Apache test page displayed instead of site content | No `index.html` in DocumentRoot, or missing `-H "Host: ..."` in curl | Deploy content; include the Host header |
| Default site served instead of expected backend | Missing or incorrect `Host` header in the request | Add `-H "Host: proxy.local"` to curl |
| *"Address already in use"* | Duplicate `Listen` directive (commonly `Listen 443` twice via `ssl.conf`) or port conflict with another process (e.g., nginx) | `sudo ss -tlnp \| grep :<PORT>` |
| Cannot bind a new port | SELinux port label missing | `sudo semanage port -a -t http_port_t -p tcp <PORT>` (or `-m` to modify) |
| New `Listen` directive not effective | `reload` used instead of `restart` | Use `systemctl restart httpd` after `Listen` changes |
| Wrong virtual host served | `ServerName` mismatch | Run `httpd -S`; verify DNS/Host header |
| Certbot failure | Domain not resolving to the server, or port 80 unreachable from the internet | Verify DNS A record and firewall |
| Forward proxy returns 403 | Client IP not permitted by `Require ip` | Adjust the allowed IP/subnet |

> **"Connection refused" always indicates that no process is listening on the target port.** It is not a permissions issue. In Apache, the global `Listen` directive opens the port. If `Listen 80` is absent or altered, nothing listens on port 80 regardless of `<VirtualHost *:80>` blocks.

### 12.2 Diagnostic Command Sequence

```bash
sudo httpd -t                          # 1. Configuration syntax
sudo httpd -S                          # 2. Virtual host map
sudo systemctl status httpd            # 3. Service state
sudo tail -n 30 /var/log/httpd/error_log   # 4. Recent errors
sudo ss -tlnp | grep httpd            # 5. Listening ports
getenforce                             # 6. SELinux mode
sudo ausearch -m avc -ts recent       # 7. Recent SELinux denials
```

---

## 13. Quick Reference

### 13.1 Essential Paths and Commands

| Item | Value |
|------|-------|
| Install | `dnf install httpd` |
| Start and enable | `systemctl enable --now httpd` |
| Default web root | `/var/www/html` |
| Main config | `/etc/httpd/conf/httpd.conf` |
| Site configs | `/etc/httpd/conf.d/*.conf` |
| Syntax check | `httpd -t` **(always run before reload)** |
| Virtual host map | `httpd -S` |
| Graceful reload | `systemctl reload httpd` |
| Full restart (after `Listen` changes) | `systemctl restart httpd` |
| Error log | `/var/log/httpd/error_log` |

### 13.2 Directive Summary

| Directive | Purpose |
|-----------|---------|
| `Listen 80` | Bind to a TCP port |
| `<VirtualHost *:80>` | Define a website |
| `ServerName` | Domain the virtual host answers to |
| `DocumentRoot` | Filesystem path to content |
| `<Directory>` + `Require all granted` | Grant access to a directory |
| `SSLEngine on` + certificate directives | Enable HTTPS |
| `Redirect permanent / https://...` | Enforce HTTPS |
| `ProxyPass` / `ProxyPassReverse` | Reverse proxy |
| `<Proxy "balancer://x">` + `BalancerMember` | Load-balancer pool |
| `lbmethod=` / `loadfactor=` / `status=+H` | Balancing algorithm and member options |
| `ProxyRequests On` | Forward proxy **(restrict access)** |

### 13.3 Standard Change Workflow

```
1. Edit /etc/httpd/conf.d/<file>.conf
2. sudo httpd -t
3. sudo systemctl reload httpd       (restart if Listen directives changed)
4. Verify with curl / browser
5. If issues arise:
     error_log → SELinux (ausearch) → firewall → ss -tlnp
```


```
