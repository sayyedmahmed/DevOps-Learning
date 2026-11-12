# Enterprise Nginx Infrastructure and Proxy Configuration Reference

This document defines the standard operating procedures, architecture patterns, configuration standards, and troubleshooting procedures for deploying Nginx as a web server, reverse proxy, and load balancer within an enterprise infrastructure. It also covers Squid deployment for forward proxy requirements, Nginx logging and log analysis, and performance tuning.

**Target platforms:** AlmaLinux / RHEL / Rocky Linux. For Ubuntu/Debian, substitute `dnf` with `apt install nginx` and `firewall-cmd` with `ufw`. SELinux procedures apply to the RHEL family only.

**Scope note:** This document covers the Nginx and Squid infrastructure as established during implementation. HAProxy and Apache configurations were not part of the scope of this deployment and are not included. If those components are required in the future, they should be documented separately based on their actual implementation.

**Golden rule:** After changing any configuration, run `sudo nginx -t`, then `sudo systemctl reload nginx`.

---

## Table of Contents

- [1. Overview of Nginx](#1-overview-of-nginx)
- [2. Proxy Concepts](#2-proxy-concepts)
  - [2.1 Forward Proxy](#21-forward-proxy)
  - [2.2 Reverse Proxy](#22-reverse-proxy)
  - [2.3 Load Balancer](#23-load-balancer)
  - [2.4 Virtual Host (Server Block)](#24-virtual-host-server-block)
  - [2.5 HTTPS](#25-https)
- [3. Installation and Initial Configuration](#3-installation-and-initial-configuration)
  - [3.1 Package Installation](#31-package-installation)
  - [3.2 Service Control](#32-service-control)
  - [3.3 Firewall Configuration](#33-firewall-configuration)
  - [3.4 Initial Verification](#34-initial-verification)
  - [3.5 Custom Default Page](#35-custom-default-page)
- [4. Configuration Files and Command Reference](#4-configuration-files-and-command-reference)
  - [4.1 Directory Structure](#41-directory-structure)
  - [4.2 Configuration Workflow](#42-configuration-workflow)
  - [4.3 Command Reference](#43-command-reference)
  - [4.4 Configuration File Loading Behavior](#44-configuration-file-loading-behavior)
  - [4.5 Vim Operations](#45-vim-operations)
- [5. Virtual Host Implementation](#5-virtual-host-implementation)
  - [5.1 Directory and Asset Preparation](#51-directory-and-asset-preparation)
  - [5.2 Server Block Configuration site1 conf](#52-server-block-configuration-site1-conf)
  - [5.3 Server Block Configuration site2 conf](#53-server-block-configuration-site2-conf)
  - [5.4 Directive Reference](#54-directive-reference)
  - [5.5 Validation and Reload](#55-validation-and-reload)
  - [5.6 DNS and Host Resolution](#56-dns-and-host-resolution)
  - [5.7 Request Routing Logic](#57-request-routing-logic)
- [6. HTTPS and TLS Configuration](#6-https-and-tls-configuration)
  - [6.1 Let's Encrypt (Public Domains)](#61-lets-encrypt-public-domains)
  - [6.2 Self-Signed Certificates (Internal Infrastructure)](#62-self-signed-certificates-internal-infrastructure)
- [7. Backend Infrastructure Preparation](#7-backend-infrastructure-preparation)
  - [7.1 SELinux and Firewall Adjustments](#71-selinux-and-firewall-adjustments)
  - [7.2 Backend Content Creation](#72-backend-content-creation)
  - [7.3 Backend Server Blocks](#73-backend-server-blocks)
  - [7.4 Verification](#74-verification)
- [8. Reverse Proxy Configuration](#8-reverse-proxy-configuration)
  - [8.1 Standard Reverse Proxy](#81-standard-reverse-proxy)
  - [8.2 Path-Based Routing](#82-path-based-routing)
  - [8.3 TLS Termination](#83-tls-termination)
- [9. Load Balancing Implementation](#9-load-balancing-implementation)
  - [9.1 Upstream Configuration](#91-upstream-configuration)
  - [9.2 Resolving Default Server Conflicts](#92-resolving-default-server-conflicts)
  - [9.3 Verification and Testing](#93-verification-and-testing)
  - [9.4 Backend Failure Simulation](#94-backend-failure-simulation)
- [10. Advanced Load Balancing Strategies](#10-advanced-load-balancing-strategies)
  - [10.1 Distribution Algorithms](#101-distribution-algorithms)
  - [10.2 Failure Handling and High Availability](#102-failure-handling-and-high-availability)
- [11. Forward Proxy Implementation (Squid)](#11-forward-proxy-implementation-squid)
  - [11.1 Installation and Service Control](#111-installation-and-service-control)
  - [11.2 Verification](#112-verification)
  - [11.3 Access Control Lists](#113-access-control-lists)
- [12. Nginx Logging and Log Analysis](#12-nginx-logging-and-log-analysis)
  - [12.1 Log Files](#121-log-files)
  - [12.2 Access Log Format](#122-access-log-format)
  - [12.3 Per-Virtual-Host Logging](#123-per-virtual-host-logging)
  - [12.4 Error Log Levels](#124-error-log-levels)
  - [12.5 JSON Logging for Log Aggregation](#125-json-logging-for-log-aggregation)
  - [12.6 Log Rotation](#126-log-rotation)
  - [12.7 Log Analysis Commands](#127-log-analysis-commands)
- [13. Performance Tuning and Optimization](#13-performance-tuning-and-optimization)
  - [13.1 Worker Processes and Connections](#131-worker-processes-and-connections)
  - [13.2 File Transfer and Connection Handling](#132-file-transfer-and-connection-handling)
  - [13.3 Gzip Compression](#133-gzip-compression)
  - [13.4 Static Asset Caching](#134-static-asset-caching)
  - [13.5 Open File Cache](#135-open-file-cache)
  - [13.6 Upstream Keepalive Connections](#136-upstream-keepalive-connections)
  - [13.7 Proxy Buffering](#137-proxy-buffering)
  - [13.8 Client Request Limits](#138-client-request-limits)
  - [13.9 Rate Limiting and Connection Limiting](#139-rate-limiting-and-connection-limiting)
- [14. Troubleshooting and Diagnostics](#14-troubleshooting-and-diagnostics)
- [15. Quick Reference](#15-quick-reference)

---

## 1. Overview of Nginx

Nginx (pronounced "engine-x") is a web server that processes HTTP/HTTPS requests and serves responses. It also functions as a reverse proxy and load balancer.

```text
  Browser                              SERVER (nginx)
 +--------------+   "give me the     +----------------------+
 |  Chrome      |    page" (HTTP)    |  nginx               |
 |              | -----------------> |    |                 |
 |              | <----------------- |    v                 |
 +--------------+   sends index.html |  /usr/share/nginx/   |
                                     |  html/index.html     |
                                     +----------------------+
```

| Term | Definition |
| :--- | :--- |
| **Web Server** | Application that processes HTTP/HTTPS requests and serves static or dynamic content. |
| **Document Root** | The designated filesystem directory where web assets are stored. |
| **index.html** | The default page served when a directory is requested. |
| **Port 80 / 443** | Standard ports for HTTP and HTTPS traffic respectively. |
| **Configuration File** | Text file that controls Nginx behavior. |

---

## 2. Proxy Concepts

A proxy is an intermediary component that sits between clients and servers.

### 2.1 Forward Proxy

A forward proxy acts as an intermediary for outbound client requests.

```text
  Users                  FORWARD PROXY               Internet
 +------+                +-----------+             +-----------+
 | PC 1 |--+             |           |------------>| websites  |
 | PC 2 |--+------------>|   proxy   |             +-----------+
 | PC 3 |--+             |           |
 +------+                +-----------+
 Users KNOW about it. Websites only see the proxy.
```

**Use cases:** content filtering, caching, hiding client IP addresses.

### 2.2 Reverse Proxy

A reverse proxy acts as an intermediary for inbound requests directed at backend servers.

```text
  Users                 REVERSE PROXY              Hidden servers
 +------+               +-----------+              +-----------+
 | User |-------------->|   proxy   |------------->|  backend  |
 +------+               +-----------+              +-----------+
 Users do NOT know the backend exists.
```

```text
       Client Browser
            |
            |  Request to Port 80/443
            v
      +-------------+
      |   Nginx     |  <-- Reverse Proxy (front.conf / lb.conf)
      |  (Port 80)  |
      +-------------+
            |
            |  Forwards request internally
            v
      +-------------+
      |  Backend    |  <-- Application Server (e.g., Port 8081)
      | (Port 8081) |
      +-------------+
```

**Use cases:** hiding backend topology, centralized TLS termination, load balancing.

**Note:** A reverse proxy configuration is not a Virtual IP (VIP). A VIP is a network-layer address that floats between servers for high availability (handled by tools such as Keepalived). A reverse proxy is an application-layer configuration that listens on the server's actual IP address and routes traffic to backends.

### 2.3 Load Balancer

A load balancer is a reverse proxy with multiple backend servers behind it.

```text
                          +--> Backend 1
  Users --> LOAD BALANCER |--> Backend 2
                          +--> Backend 3
```

**Use cases:** distributing workload across servers, maintaining availability if a backend node fails.

### 2.4 Virtual Host (Server Block)

One Nginx instance serving multiple websites, selected by the domain name in the request.

```text
                        +--> site1.local --> /var/www/site1
  Browser --> nginx :80 |
                        +--> site2.local --> /var/www/site2
```

A `server { }` block in Nginx is a virtual host. Any configuration file containing a `server { }` block with a `server_name` directive, including reverse proxy and load balancer configurations, is technically a virtual host.

### 2.5 HTTPS

HTTP with TLS encryption. A certificate authenticates the server identity.

```text
 HTTP  (80):   Browser --- readable text ---> Server
 HTTPS (443):  Browser === encrypted ========> Server
```

---

## 3. Installation and Initial Configuration

### 3.1 Package Installation

```bash
sudo dnf install nginx -y
nginx -v
```

### 3.2 Service Control

```bash
sudo systemctl enable --now nginx
sudo systemctl status nginx
```

| Command | Function |
| :--- | :--- |
| `systemctl start nginx` | Initiates the service. |
| `systemctl stop nginx` | Halts the service. |
| `systemctl restart nginx` | Performs a hard stop and start. |
| `systemctl reload nginx` | Gracefully applies configuration changes without dropping connections. |
| `systemctl enable nginx` | Enables the service to start on boot. |

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

Open `http://YOUR_SERVER_IP` in a browser. The default Nginx welcome page confirms that:

1. Nginx is installed and running.
2. The firewall permits traffic on port 80.
3. The browser successfully connected to the server.

### 3.5 Custom Default Page

```bash
sudo vim /usr/share/nginx/html/index.html
```

```html
<!DOCTYPE html>
<html>
<head><title>My Nginx Page</title></head>
<body style="font-family:Arial;text-align:center;margin-top:80px">
  <h1>Hello from Nginx!</h1>
</body>
</html>
```

Save and quit (`Esc`, then `:wq`, then `Enter`). Refresh the browser. No restart is required.

---

## 4. Configuration Files and Command Reference

### 4.1 Directory Structure

```text
 /etc/nginx/
   |-- nginx.conf          <- main config
   |-- conf.d/*.conf       <- your configs (auto-included, only *.conf)
 /usr/share/nginx/html/    <- default web folder (RHEL)
 /var/log/nginx/
   |-- access.log          <- who visited
   |-- error.log           <- what went wrong
```

| Path | Purpose |
| :--- | :--- |
| `/etc/nginx/nginx.conf` | Main global configuration file. |
| `/etc/nginx/conf.d/*.conf` | Modular configuration directory. Only files ending in `.conf` are loaded. |
| `/usr/share/nginx/html/` | Default document root for the base installation. |
| `/var/log/nginx/` | Directory containing `access.log` and `error.log`. |

### 4.2 Configuration Workflow

**Golden Rule:** Always validate syntax before applying changes.

```bash
sudo nginx -t
sudo systemctl reload nginx
```

### 4.3 Command Reference

| Task | Command |
| :--- | :--- |
| Test config | `sudo nginx -t` |
| Show full loaded config | `sudo nginx -T` |
| Reload | `sudo systemctl reload nginx` |
| See listening ports | `sudo ss -tlnp \| grep nginx` |
| Live access log | `sudo tail -f /var/log/nginx/access.log` |

### 4.4 Configuration File Loading Behavior

Files in `/etc/nginx/conf.d/` are loaded only if they end in `.conf`. Renaming a file to `.off` or `.txt` disables it without deleting it.

Configuration files may be named according to their function. For example:

- `site1.conf`, `site2.conf` for virtual hosts.
- `front.conf` or `proxy.conf` for reverse proxy configurations.
- `lb.conf` for load balancer configurations.
- `backend1.conf`, `backend2.conf`, `backend3.conf` for backend server blocks.

Any valid `.conf` filename is accepted by Nginx.

### 4.5 Vim Operations

| Action | Command |
| :--- | :--- |
| Open a file | `sudo vim FILE` |
| Enter insert mode | Press `i` (bottom shows `-- INSERT --`) |
| Paste text | Right-click or `Ctrl+Shift+V` |
| Stop typing | Press `Esc` |
| Save and quit | `:wq` then `Enter` |
| Quit without saving | `:q!` then `Enter` |
| Delete a whole line | `Esc`, then `dd` |
| Delete entire file contents | `Esc`, then `:%d`, then `Enter` |
| Prevent indentation issues when pasting | `:set paste`, then `Enter`, then `i` |

**Block Commenting (Multiple Lines):**

1. Press `Esc` and navigate to the first line of the target block.
2. Press `Ctrl+v` to enter Visual Block mode.
3. Use arrow keys to highlight the lines.
4. Press `Shift+i` (capital I) to enter Insert mode.
5. Type `#` and press `Esc` twice. The comment character is applied to all selected lines.

---

## 5. Virtual Host Implementation

Virtual hosts (server blocks) allow a single Nginx instance to route traffic to different document roots based on the requested domain name.

### 5.1 Directory and Asset Preparation

```bash
sudo mkdir -p /var/www/site1 /var/www/site2
echo '<h1>This is SITE 1</h1>' | sudo tee /var/www/site1/index.html
echo '<h1>This is SITE 2</h1>' | sudo tee /var/www/site2/index.html
sudo chmod -R 755 /var/www
sudo restorecon -Rv /var/www          # SELinux label (RHEL family)
```

### 5.2 Server Block Configuration site1 conf

```bash
sudo vim /etc/nginx/conf.d/site1.conf
```

```nginx
server {
    listen 80;
    server_name site1.local;
    root /var/www/site1;
    index index.html;
    access_log /var/log/nginx/site1.access.log;
    error_log  /var/log/nginx/site1.error.log;
}
```

### 5.3 Server Block Configuration site2 conf

```bash
sudo vim /etc/nginx/conf.d/site2.conf
```

```nginx
server {
    listen 80;
    server_name site2.local;
    root /var/www/site2;
    index index.html;
    access_log /var/log/nginx/site2.access.log;
    error_log  /var/log/nginx/site2.error.log;
}
```

### 5.4 Directive Reference

| Directive | Meaning |
| :--- | :--- |
| `server { }` | Defines one website (virtual host). |
| `listen 80;` | Port to open. |
| `server_name` | Domain this site answers to. |
| `root` | Folder containing the site files. |
| `access_log` / `error_log` | Separate log files per site. |

### 5.5 Validation and Reload

```bash
sudo nginx -t
sudo systemctl reload nginx
```

### 5.6 DNS and Host Resolution

**Terminal verification (from the server):**

```bash
curl -H "Host: site1.local" http://localhost
curl -H "Host: site2.local" http://localhost
```

**Browser verification:**

Add the following line to the client machine's hosts file:

- **Linux/Mac:** `/etc/hosts`
- **Windows:** `C:\Windows\System32\drivers\etc\hosts` (open as Administrator)

```text
SERVER_IP   site1.local site2.local
```

Then visit `http://site1.local` and `http://site2.local` in the browser.

### 5.7 Request Routing Logic

```text
 Request with  Host: site2.local
        |
        v
 Look for a matching server_name
        |
  match? --yes--> use that server block
        |
        no
        v
 use the default server (first one, or marked default_server)
```

---

## 6. HTTPS and TLS Configuration

| Route | Use when |
| :--- | :--- |
| Let's Encrypt | Real public domain pointing to a public server. |
| Self-signed | Internal infrastructure / private network (browser shows a warning). |

### 6.1 Let's Encrypt (Public Domains)

**Requirements (all must be met):**

- A registered domain (e.g., `example.com`) with an A record pointing to the server's public IP.
- Ports 80 and 443 reachable from the internet.
- A server block with `server_name example.com;`.

Let's Encrypt cannot issue certificates for `.local` names or private IP addresses. Use self-signed certificates for those.

```bash
sudo dnf install epel-release -y
sudo dnf install certbot python3-certbot-nginx -y
sudo certbot --nginx -d example.com -d www.example.com
```

Answer the prompts (email, terms) and choose to redirect HTTP to HTTPS. Certbot edits the Nginx configuration and reloads it automatically.

**Renewal (certificates last 90 days and renew automatically):**

```bash
sudo systemctl list-timers | grep certbot
sudo certbot renew --dry-run
sudo certbot certificates
```

### 6.2 Self-Signed Certificates (Internal Infrastructure)

**Step 1: Generate the certificate.**

```bash
sudo mkdir -p /etc/nginx/ssl
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/nginx/ssl/site1.key \
  -out    /etc/nginx/ssl/site1.crt \
  -subj "/CN=site1.local"
sudo chmod 600 /etc/nginx/ssl/site1.key
sudo restorecon -Rv /etc/nginx/ssl
```

| Option | Meaning |
| :--- | :--- |
| `-x509` | Generate a certificate directly. |
| `-nodes` | No password on the private key. |
| `-days 365` | Valid for one year. |
| `-keyout` / `-out` | Private key (secret) / public certificate. |
| `-subj "/CN=..."` | The site name. |

**Step 2: Update `/etc/nginx/conf.d/site1.conf`.**

```nginx
# Redirect HTTP to HTTPS
server {
    listen 80;
    server_name site1.local;
    return 301 https://$host$request_uri;
}

# The HTTPS site
server {
    listen 443 ssl;
    server_name site1.local;
    ssl_certificate     /etc/nginx/ssl/site1.crt;
    ssl_certificate_key /etc/nginx/ssl/site1.key;
    ssl_protocols       TLSv1.2 TLSv1.3;
    root /var/www/site1;
    index index.html;
}
```

| Directive | Meaning |
| :--- | :--- |
| `return 301 https://...` | Permanent redirect to HTTPS. |
| `listen 443 ssl;` | HTTPS port with TLS. |
| `ssl_certificate` | Public certificate. |
| `ssl_certificate_key` | Private key. |

**Step 3: Validate, reload, and test.**

```bash
sudo nginx -t
sudo systemctl reload nginx
curl -k https://site1.local --resolve site1.local:443:127.0.0.1
```

`-k` accepts the self-signed certificate. A browser will display a warning; select Advanced, then Proceed.

---

## 7. Backend Infrastructure Preparation

Backend servers are required for reverse proxy and load balancing configurations. Three static sites are created on ports 8081, 8082, and 8083.

**Server IP:** This document uses `10.10.10.210`. Replace it with the actual server IP (`hostname -I`) everywhere it appears.

```text
                              +--> Backend 1  10.10.10.210:8081
   FRONT DOOR :80  ---proxy---|--> Backend 2  10.10.10.210:8082
   (nginx)                    +--> Backend 3  10.10.10.210:8083
```

### 7.1 SELinux and Firewall Adjustments

```bash
sudo setsebool -P httpd_can_network_connect 1     # lets nginx talk to backends
sudo dnf install policycoreutils-python-utils -y
sudo semanage port -a -t http_port_t -p tcp 8081-8083
```

If `semanage` reports "already defined", use `-m` instead of `-a`:

```bash
sudo semanage port -m -t http_port_t -p tcp 8081-8083
```

Open the backend ports in the firewall:

```bash
sudo firewall-cmd --permanent --add-port=8081-8083/tcp
sudo firewall-cmd --reload
```

### 7.2 Backend Content Creation

```bash
for i in 1 2 3; do
  sudo mkdir -p /var/www/b$i
  echo "BACKEND $i" | sudo tee /var/www/b$i/index.html
done
sudo chmod -R 755 /var/www
sudo restorecon -Rv /var/www
```

### 7.3 Backend Server Blocks

**`/etc/nginx/conf.d/backend1.conf`:**

```nginx
server {
    listen 8081;
    root /var/www/b1;
}
```

**`/etc/nginx/conf.d/backend2.conf`:**

```nginx
server {
    listen 8082;
    root /var/www/b2;
}
```

**`/etc/nginx/conf.d/backend3.conf`:**

```nginx
server {
    listen 8083;
    root /var/www/b3;
}
```

`listen 8081;` opens port 8081 on the server, so `http://10.10.10.210:8081` can also be accessed directly in a browser.

### 7.4 Verification

```bash
sudo nginx -t
sudo systemctl reload nginx
curl http://10.10.10.210:8081
curl http://10.10.10.210:8082
curl http://10.10.10.210:8083
```

**Expected output:** `BACKEND 1`, `BACKEND 2`, `BACKEND 3`.

---

## 8. Reverse Proxy Configuration

A reverse proxy receives requests on port 80 and forwards them to a backend server. The client interacts only with the proxy and is unaware of the backend.

```text
 curl --> nginx :80 --proxy_pass--> 10.10.10.210:8081 (BACKEND 1)
```

This is the standard industry approach for reverse proxying. In production, engineers additionally implement health checks, caching, rate limiting, and SSL/TLS termination. The core logic using `proxy_pass` and header forwarding is the same everywhere.

### 8.1 Standard Reverse Proxy

Create `/etc/nginx/conf.d/front.conf`:

```nginx
server {
    listen 80;
    server_name proxy.local;

    location / {
        proxy_pass http://10.10.10.210:8081;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

| Directive | Meaning |
| :--- | :--- |
| `location /` | Applies to every URL. |
| `proxy_pass` | Destination to which the request is forwarded. |
| `Host` | Passes the original domain name. |
| `X-Real-IP` | Passes the real client IP (otherwise the backend sees only the proxy IP). |
| `X-Forwarded-For` | List of IPs the request passed through. |
| `X-Forwarded-Proto` | Whether the client used HTTP or HTTPS. |

The `proxy_set_header` directives are critical for maintaining client IP transparency and protocol awareness at the backend application layer. These headers should always be included in production reverse proxy configurations.

**Validate, reload, and test:**

```bash
sudo nginx -t
sudo systemctl reload nginx
curl -H "Host: proxy.local" http://localhost
```

**Expected output:** `BACKEND 1`. The request was sent to port 80, but the content originated from port 8081.

### 8.2 Path-Based Routing

Path-based routing directs different URL paths to different backend servers.

Replace the content of `/etc/nginx/conf.d/front.conf` with:

```nginx
server {
    listen 80;
    server_name proxy.local;

    location /one/ {
        proxy_pass http://10.10.10.210:8081/;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /two/ {
        proxy_pass http://10.10.10.210:8082/;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

```text
 /one/  --> BACKEND 1
 /two/  --> BACKEND 2
```

The trailing `/` on `proxy_pass` strips the `/one/` or `/two/` prefix before forwarding the request to the backend.

The `proxy_set_header` directives are included in each `location` block to maintain best-practice header forwarding. Without them, the backend servers will not receive the real client IP or protocol information.

**Validate and reload:**

```bash
sudo nginx -t
sudo systemctl reload nginx
```

**Testing: Terminal (curl):**

```bash
curl -H "Host: proxy.local" http://localhost/one/
curl -H "Host: proxy.local" http://localhost/two/
```

**Expected output:** The first command returns `BACKEND 1`. The second returns `BACKEND 2`.

**Testing: Web Browser:**

Map `proxy.local` to the server IP in the client machine's hosts file:

- **Linux/Mac:** `/etc/hosts`
- **Windows:** `C:\Windows\System32\drivers\etc\hosts` (open as Administrator)

```text
10.10.10.210  proxy.local
```

Then visit:

- `http://proxy.local/one/`, displays **BACKEND 1**.
- `http://proxy.local/two/`, displays **BACKEND 2**.

### 8.3 TLS Termination

Nginx handles HTTPS at the proxy layer. The backend receives plain HTTP and requires no TLS configuration.

```text
 Browser ==== HTTPS ====> nginx ---- plain HTTP ----> Backend
```

```nginx
server {
    listen 443 ssl;
    server_name proxy.local;
    ssl_certificate     /etc/nginx/ssl/site1.crt;
    ssl_certificate_key /etc/nginx/ssl/site1.key;

    location / {
        proxy_pass http://10.10.10.210:8081;
        proxy_set_header Host              $host;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

---

## 9. Load Balancing Implementation

A load balancer distributes incoming requests across multiple backend servers using the Round Robin method by default.

```text
                          +--> BACKEND 1
 Browser --> nginx :80 ---|--> BACKEND 2
                          +--> BACKEND 3
```

### 9.1 Upstream Configuration

The standard naming convention for load balancer configurations is `lb.conf`. Any file ending in `.conf` inside `/etc/nginx/conf.d/` is loaded automatically.

**Important:** Remove or rename any previous reverse proxy configuration (e.g., `front.conf`) to avoid conflicts on port 80. Rename it to `front.conf.disable` or delete it before creating `lb.conf`.

Create `/etc/nginx/conf.d/lb.conf`:

```nginx
upstream mygroup {
    server 10.10.10.210:8081;
    server 10.10.10.210:8082;
    server 10.10.10.210:8083;
}

server {
    listen 80;
    server_name proxy.local;

    location / {
        proxy_pass http://mygroup;
        proxy_set_header Host            $host;
        proxy_set_header X-Real-IP       $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

`upstream mygroup { }` defines a named group of backend servers. `proxy_pass http://mygroup;` distributes traffic across the group.

```bash
sudo nginx -t
sudo systemctl reload nginx
```

### 9.2 Resolving Default Server Conflicts

If accessing the server via its IP address returns the default OS welcome page (e.g., the AlmaLinux test page) instead of the load-balanced application, the default `server` block in `/etc/nginx/nginx.conf` is intercepting the traffic.

This default block typically listens on `[::]:80` with `server_name _;`. The underscore `_` acts as a wildcard catch-all for requests that do not match a specific `server_name`.

**Resolution:**

1. Open `/etc/nginx/nginx.conf`:
   ```bash
   sudo vim /etc/nginx/nginx.conf
   ```
2. Locate the default `server` block at the bottom of the `http` context.
3. Comment out the entire block using the Vim Visual Block method (`Ctrl+v`, select lines, `Shift+i`, type `#`, press `Esc` twice):
   ```nginx
   # server {
   #     listen       8080;
   #     listen       [::]:80;
   #     server_name  _;
   #     root         /usr/share/nginx/html;
   #     include /etc/nginx/default.d/*.conf;
   # }
   ```
4. Validate and reload:
   ```bash
   sudo nginx -t
   sudo systemctl reload nginx
   ```

Additionally, verify that no `default.conf` file exists in `/etc/nginx/conf.d/`. If present, rename it:

```bash
sudo mv /etc/nginx/conf.d/default.conf /etc/nginx/conf.d/default.conf.disable
sudo systemctl reload nginx
```

To identify which configuration is actively listening on port 80:

```bash
sudo nginx -T | grep -A 5 "listen 80"
```

### 9.3 Verification and Testing

**Terminal verification:**

Execute a loop to verify Round Robin distribution. Using the server IP address with the `Host` header is the most reliable testing method:

```bash
for i in 1 2 3 4 5 6; do curl -s -H "Host: proxy.local" http://10.10.10.210; done
```

Testing via `localhost` is also valid when executed directly on the server:

```bash
for i in 1 2 3 4 5 6; do curl -s -H "Host: proxy.local" http://localhost; done
```

**Expected output (Round Robin):**

```text
BACKEND 1
BACKEND 2
BACKEND 3
BACKEND 1
BACKEND 2
BACKEND 3
```

**Browser verification and caching:**

Web browsers cache HTTP responses. If the browser displays only one backend node despite terminal verification showing rotation:

1. Open an **Incognito/Private** browsing window and visit `http://proxy.local`.
2. Perform a **Hard Refresh** (`Ctrl+Shift+R` or `Cmd+Shift+R`) after each page load.
3. Ensure the client `/etc/hosts` file maps `proxy.local` to the server IP.

If the terminal shows all three backends but the browser shows only one, the cause is browser caching, not a configuration error.

**Direct backend access (bypassing the load balancer):**

| Method | URL |
| :--- | :--- |
| Backend 1 directly | `http://10.10.10.210:8081/` |
| Backend 2 directly | `http://10.10.10.210:8082/` |
| Backend 3 directly | `http://10.10.10.210:8083/` |
| Via load balancer | `http://10.10.10.210/` (with `Host: proxy.local`) |

### 9.4 Backend Failure Simulation

Disable a backend by renaming its configuration file:

```bash
sudo mv /etc/nginx/conf.d/backend2.conf /etc/nginx/conf.d/backend2.off
sudo systemctl reload nginx
```

```bash
for i in 1 2 3 4 5 6; do curl -s -H "Host: proxy.local" http://localhost; done
```

Backend 2 stops receiving traffic, and clients receive no errors:

```text
BACKEND 1
BACKEND 3
BACKEND 1
BACKEND 3
...
```

The first one or two requests may be slower while Nginx detects that Backend 2 is unavailable.

**Restore the backend:**

```bash
sudo mv /etc/nginx/conf.d/backend2.off /etc/nginx/conf.d/backend2.conf
sudo systemctl reload nginx
```

---

## 10. Advanced Load Balancing Strategies

### 10.1 Distribution Algorithms

| Algorithm | Directive | Behavior | Use Case |
| :--- | :--- | :--- | :--- |
| **Round Robin** | *(default)* | Distributes requests sequentially across backends. | Homogeneous backend nodes. |
| **Weighted** | `server ... weight=3;` | Higher-capacity nodes receive proportionally more traffic. | Heterogeneous backend hardware. |
| **Least Connections** | `least_conn;` | Routes to the backend with the fewest active connections. | Requests with varying processing times. |
| **IP Hash** | `ip_hash;` | Binds a client IP to a specific backend. | Session persistence without shared session storage. |

**Weighted:**

```nginx
upstream mygroup {
    server 10.10.10.210:8081 weight=3;
    server 10.10.10.210:8082;
}
```

**Least Connections:**

```nginx
upstream mygroup {
    least_conn;
    server 10.10.10.210:8081;
    server 10.10.10.210:8082;
}
```

**IP Hash:**

```nginx
upstream mygroup {
    ip_hash;
    server 10.10.10.210:8081;
    server 10.10.10.210:8082;
}
```

### 10.2 Failure Handling and High Availability

```nginx
upstream mygroup {
    server 10.10.10.210:8081 max_fails=3 fail_timeout=30s;
    server 10.10.10.210:8082 max_fails=3 fail_timeout=30s;
    server 10.10.10.210:8083 backup;
}
```

| Parameter | Function |
| :--- | :--- |
| `max_fails=3` | Marks the node as unavailable after 3 consecutive failures. |
| `fail_timeout=30s` | Defines the failure counting window and the quarantine duration before retry. |
| `backup` | Standby node that receives traffic only when all primary nodes are unavailable. |
| `down` | Permanently removes a node from the rotation pool. |

---

## 11. Forward Proxy Implementation (Squid)

Nginx does not support the `CONNECT` method required for HTTPS tunneling and cannot function as a full forward proxy. Squid is the standard tool for forward proxy deployments.

```text
 Client --> Squid :3128 --> Internet
```

### 11.1 Installation and Service Control

```bash
sudo dnf install squid -y
sudo systemctl enable --now squid
```

The default configuration permits the server itself and private networks (`localhost`, `localnet`).

### 11.2 Verification

`127.0.0.1` refers to the local server. Test from the server itself:

```bash
curl -x http://127.0.0.1:3128 http://example.com
curl -x http://127.0.0.1:3128 -I https://example.com
sudo tail -n 5 /var/log/squid/access.log
```

**Forward vs. Reverse Proxy comparison:**

```text
 FORWARD:  curl -x http://PROXY:3128 http://example.com   (client knows about the proxy)
 REVERSE:  curl -H "Host: proxy.local" http://localhost   (client knows nothing about backends)
```

### 11.3 Access Control Lists

Restrict proxy access to authorized internal subnets.

```bash
sudo vim /etc/squid/squid.conf
```

```text
acl mynet src 192.168.1.0/24
http_access allow mynet
http_access deny all
http_port 3128
```

```bash
sudo firewall-cmd --permanent --add-port=3128/tcp
sudo firewall-cmd --reload
sudo systemctl restart squid
```

**Security requirement:** Never expose a forward proxy to the public internet without strict ACLs and authentication.

---

## 12. Nginx Logging and Log Analysis

### 12.1 Log Files

| File | Purpose |
| :--- | :--- |
| `/var/log/nginx/access.log` | Records every processed request. |
| `/var/log/nginx/error.log` | Records errors, warnings, and startup issues. |

### 12.2 Access Log Format

The log format is defined in the `http` context of `/etc/nginx/nginx.conf`:

```nginx
log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                '$status $body_bytes_sent "$http_referer" '
                '"$http_user_agent" "$http_x_forwarded_for"';

access_log /var/log/nginx/access.log main;
```

| Variable | Meaning |
| :--- | :--- |
| `$remote_addr` | Client (or proxy) IP address. |
| `$time_local` | Timestamp of the request. |
| `$request` | Full request line (method, URI, protocol). |
| `$status` | HTTP response status code. |
| `$body_bytes_sent` | Bytes returned to the client. |
| `$http_referer` | Referring page. |
| `$http_user_agent` | Client browser or tool. |
| `$http_x_forwarded_for` | Original client IP when behind a proxy. |

### 12.3 Per-Virtual-Host Logging

Define separate log files per site to isolate traffic:

```nginx
server {
    listen 80;
    server_name site1.local;
    access_log /var/log/nginx/site1.access.log;
    error_log  /var/log/nginx/site1.error.log warn;
}
```

### 12.4 Error Log Levels

Set the verbosity with the `error_log` directive. Levels from most to least verbose:

```text
debug > info > notice > warn > error > crit > alert > emerg
```

```nginx
error_log /var/log/nginx/error.log warn;
```

Use `debug` only during troubleshooting, as it generates significant output.

### 12.5 JSON Logging for Log Aggregation

For SIEM or centralized logging (ELK, Splunk, Graylog), output structured JSON:

```nginx
log_format json escape=json '{'
    '"time":"$time_iso8601",'
    '"remote_addr":"$remote_addr",'
    '"request":"$request",'
    '"status":$status,'
    '"body_bytes_sent":$body_bytes_sent,'
    '"http_user_agent":"$http_user_agent",'
    '"request_time":$request_time'
'}';

access_log /var/log/nginx/access.json json;
```

### 12.6 Log Rotation

Nginx logs are rotated automatically by `logrotate` via `/etc/logrotate.d/nginx`. To apply rotation immediately:

```bash
sudo logrotate -f /etc/logrotate.d/nginx
```

### 12.7 Log Analysis Commands

| Goal | Command |
| :--- | :--- |
| Top client IPs | `awk '{print $1}' /var/log/nginx/access.log \| sort \| uniq -c \| sort -nr \| head` |
| Status code distribution | `awk '{print $9}' /var/log/nginx/access.log \| sort \| uniq -c \| sort -nr` |
| Most requested URLs | `awk '{print $7}' /var/log/nginx/access.log \| sort \| uniq -c \| sort -nr \| head` |
| All 404 requests | `awk '$9 == 404 {print $7}' /var/log/nginx/access.log \| sort \| uniq -c` |
| All 5xx errors | `awk '$9 >= 500 {print $7}' /var/log/nginx/access.log \| sort \| uniq -c` |
| Requests in a time window | `grep '04/Nov/2026:14' /var/log/nginx/access.log` |
| Follow logs live | `sudo tail -f /var/log/nginx/access.log` |

---

## 13. Performance Tuning and Optimization

### 13.1 Worker Processes and Connections

```nginx
worker_processes auto;

events {
    worker_connections 4096;
    use epoll;
    multi_accept on;
}
```

| Directive | Purpose |
| :--- | :--- |
| `worker_processes auto;` | Spawns one worker per CPU core. |
| `worker_connections` | Maximum simultaneous connections per worker. |
| `use epoll;` | Efficient event method on Linux. |
| `multi_accept on;` | Accepts all new connections at once. |

**Total capacity** = `worker_processes × worker_connections`.

### 13.2 File Transfer and Connection Handling

```nginx
http {
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    keepalive_requests 1000;
    reset_timedout_connection on;
}
```

| Directive | Purpose |
| :--- | :--- |
| `sendfile on;` | Kernel-level file transfer, bypassing user space. |
| `tcp_nopush on;` | Sends full packets, reducing overhead. |
| `tcp_nodelay on;` | Disables Nagle's algorithm for keepalive connections. |
| `keepalive_timeout` | Duration a keepalive connection stays open. |
| `reset_timedout_connection on;` | Frees memory from timed-out connections quickly. |

### 13.3 Gzip Compression

```nginx
gzip on;
gzip_comp_level 5;
gzip_min_length 1000;
gzip_types text/plain text/css application/json application/javascript text/xml application/xml image/svg+xml;
gzip_vary on;
```

Reduces bandwidth usage. Do not compress already-compressed assets (e.g., images).

### 13.4 Static Asset Caching

```nginx
location ~* \.(css|js|jpg|jpeg|png|gif|ico|svg|woff|woff2)$ {
    expires 30d;
    add_header Cache-Control "public, immutable";
}
```

### 13.5 Open File Cache

Caches file descriptors and metadata for frequently accessed files:

```nginx
open_file_cache max=10000 inactive=60s;
open_file_cache_valid 30s;
open_file_cache_min_uses 2;
open_file_cache_errors on;
```

### 13.6 Upstream Keepalive Connections

Reuse connections to backends to reduce handshake overhead:

```nginx
upstream mygroup {
    server 10.10.10.210:8081;
    server 10.10.10.210:8082;
    keepalive 32;
}

server {
    location / {
        proxy_pass http://mygroup;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }
}
```

`proxy_http_version 1.1;` and an empty `Connection` header are required for upstream keepalive to function.

### 13.7 Proxy Buffering

```nginx
proxy_buffering on;
proxy_buffer_size 16k;
proxy_buffers 8 16k;
proxy_busy_buffers_size 32k;
```

### 13.8 Client Request Limits

```nginx
client_max_body_size 10m;
client_body_buffer_size 128k;
client_header_buffer_size 4k;
```

### 13.9 Rate Limiting and Connection Limiting

Protect backends from abuse and traffic spikes.

```nginx
http {
    limit_req_zone $binary_remote_addr zone=mylimit:10m rate=10r/s;
    limit_conn_zone $binary_remote_addr zone=addr:10m;

    server {
        location / {
            limit_req zone=mylimit burst=20 nodelay;
            limit_conn addr 10;
        }
    }
}
```

| Directive | Purpose |
| :--- | :--- |
| `limit_req_zone` | Defines a shared memory zone tracking request rates per client IP. |
| `rate=10r/s` | Allows 10 requests per second per client. |
| `burst=20 nodelay` | Permits short bursts of up to 20 excess requests without delay. |
| `limit_conn addr 10` | Limits each client IP to 10 concurrent connections. |

Rate-limited requests return HTTP `503` by default; customize with `limit_req_status 429;`.

---

## 14. Troubleshooting and Diagnostics

| Symptom | Root Cause and Resolution |
| :--- | :--- |
| `nginx -t` shows an error | Syntax error. Review the file name and line number in the output. |
| Connection refused | No process is listening on the target port. Check `ss -tlnp \| grep nginx` and the `listen` directives. |
| Browser cannot connect | Firewall is blocking traffic, or Nginx is not running. |
| 403 Forbidden | Filesystem permissions (`chmod -R 755 /var/www`) or SELinux contexts (`restorecon -Rv /var/www`). |
| 404 Not Found | Incorrect `root` path or missing file. |
| 502 Bad Gateway | Backend is down, wrong port, or SELinux blocking connections: `setsebool -P httpd_can_network_connect 1`. |
| 504 Gateway Timeout | Backend is responding too slowly. |
| Cannot listen on 8081-8083 | SELinux port label required: `semanage port -a -t http_port_t -p tcp 8081-8083` (or `-m`). |
| Default OS welcome page displayed instead of application | The default `server` block in `/etc/nginx/nginx.conf` is intercepting the request. Comment it out, or ensure the correct `Host` header is passed. Also verify that no `default.conf` exists in `/etc/nginx/conf.d/`. |
| Missing `-H "Host: proxy.local"` on curl | Without the Host header, Nginx cannot match the correct `server_name` and falls back to the default server block. |
| Wrong site appears | `server_name` mismatch. Verify the hosts file on the client. |
| Address already in use | Another process (e.g., `httpd`) occupies the port: `sudo ss -tlnp \| grep :80`. |
| Certbot fails | Domain does not resolve to the server, or port 80 is not reachable from the internet. |
| Configuration change has no effect | File does not end in `.conf`, or Nginx was not reloaded. |
| Load balancing not rotating in browser | Browser caching. Use Incognito mode or verify via `curl`. |

**Key diagnostic principle:** "Connection refused" means nothing is listening on that port. It is not a permissions issue. The `listen` directives inside `server` blocks determine which ports are open.

**Diagnostic command stack:**

```bash
sudo nginx -t
sudo systemctl status nginx
sudo tail -n 30 /var/log/nginx/error.log
sudo ss -tlnp | grep nginx
getenforce
sudo ausearch -m avc -ts recent
```

---

## 15. Quick Reference

### Core Paths and Commands

```text
 Install      dnf install nginx
 Start        systemctl enable --now nginx
 Web folder   /usr/share/nginx/html
 Config       /etc/nginx/nginx.conf + /etc/nginx/conf.d/*.conf
 Test         nginx -t           (ALWAYS before reload)
 Reload       systemctl reload nginx
 Logs         /var/log/nginx/error.log
```

### Directive Reference

| Directive | Context | Purpose |
| :--- | :--- | :--- |
| `server { }` | `http` | Defines a virtual host / server block. |
| `listen 80;` | `server` | Binds the block to a specific port. |
| `server_name` | `server` | Domain the block responds to. |
| `root` / `index` | `server`, `location` | Filesystem path and default page. |
| `listen 443 ssl;` + cert directives | `server` | HTTPS configuration. |
| `return 301 https://...` | `server`, `location` | Forces HTTPS redirection. |
| `proxy_pass` | `location` | Reverse proxy forwarding. |
| `proxy_set_header` | `location` | Passes client information to the backend. |
| `upstream { }` | `http` | Defines a pool of backend servers. |
| `least_conn` / `ip_hash` / `weight=` | `upstream` | Load balancing algorithms. |
| `max_fails` / `fail_timeout` / `backup` | `upstream` | Failure handling. |
| `worker_processes` | `main` | Number of worker processes. |
| `worker_connections` | `events` | Max connections per worker. |
| `gzip` | `http`, `server`, `location` | Compression. |
| `limit_req_zone` / `limit_req` | `http` / `server`, `location` | Rate limiting. |

### Standard Operating Procedure

```text
 1. Edit /etc/nginx/conf.d/xxx.conf
 2. sudo nginx -t
 3. sudo systemctl reload nginx
 4. Test with curl / browser
 5. If broken: error.log -> SELinux -> firewall -> ss -tlnp
```
