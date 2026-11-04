# Enterprise Nginx Infrastructure and Proxy Configuration Reference

This document outlines the standard operating procedures, architecture patterns, and configuration standards for deploying Nginx as a web server, reverse proxy, and load balancer within an enterprise infrastructure. It also covers the deployment of Squid for forward proxy requirements.

## Table of Contents

- [1. Architecture and Proxy Concepts](#1-architecture-and-proxy-concepts)
- [2. Installation and Service Management](#2-installation-and-service-management)
- [3. Core Configuration and File Management](#3-core-configuration-and-file-management)
- [4. Virtual Host Implementation](#4-virtual-host-implementation)
- [5. HTTPS and TLS Configuration](#5-https-and-tls-configuration)
- [6. Backend Infrastructure Preparation](#6-backend-infrastructure-preparation)
- [7. Reverse Proxy Configuration](#7-reverse-proxy-configuration)
- [8. Load Balancing Implementation](#8-load-balancing-implementation)
- [9. Advanced Load Balancing Strategies](#9-advanced-load-balancing-strategies)
- [10. Forward Proxy Implementation (Squid)](#10-forward-proxy-implementation-squid)
- [11. Troubleshooting and Diagnostics](#11-troubleshooting-and-diagnostics)
- [12. Quick Reference](#12-quick-reference)

---

## 1. Architecture and Proxy Concepts

### 1.1 Core Definitions
| Component | Definition |
| :--- | :--- |
| **Web Server** | Application that processes HTTP/HTTPS requests and serves static or dynamic content. |
| **Document Root** | The designated filesystem directory where web assets are stored. |
| **Virtual Host** | A configuration mechanism allowing a single Nginx instance to serve multiple domains based on the `Host` header. |
| **Forward Proxy** | An intermediary for client requests to external servers. Clients are aware of the proxy; destination servers are not. |
| **Reverse Proxy** | An intermediary for incoming client requests to backend servers. Clients are unaware of the backend topology. |
| **Load Balancer** | A specialized reverse proxy that distributes incoming traffic across multiple backend nodes to ensure high availability and scalability. |

### 1.2 Architectural Diagrams

**Standard Web Server Flow**
```text
  Browser                              SERVER (Nginx)
 +--------------+   "HTTP Request"    +----------------------+
 |  Client      | ------------------> |  Nginx               |
 |              | <------------------ |    |                 |
 +--------------+   "HTTP Response"   |    v                 |
                                      |  /usr/share/nginx/   |
                                      |  html/index.html     |
                                      +----------------------+
```

**Reverse Proxy Flow**
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

**Load Balancer Topology**
```text
                          +--> Backend Node 1 (10.10.10.210:8081)
  Clients --> Nginx :80 --|--> Backend Node 2 (10.10.10.210:8082)
                          +--> Backend Node 3 (10.10.10.210:8083)
```

---

## 2. Installation and Service Management

### 2.1 Package Installation
Execute the following commands to install Nginx on RHEL/AlmaLinux/Rocky systems:
```bash
sudo dnf install nginx -y
nginx -v
```

### 2.2 Service Control
Enable the service to start on boot and initiate the daemon:
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

### 2.3 Firewall Configuration
Permit inbound HTTP and HTTPS traffic via `firewalld`:
```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload
```

---

## 3. Core Configuration and File Management

### 3.1 Directory Structure
| Path | Purpose |
| :--- | :--- |
| `/etc/nginx/nginx.conf` | Main global configuration file. |
| `/etc/nginx/conf.d/*.conf` | Modular configuration directory. Only files ending in `.conf` are loaded. |
| `/usr/share/nginx/html/` | Default document root for the base installation. |
| `/var/log/nginx/` | Directory containing `access.log` and `error.log`. |

### 3.2 Configuration Workflow
**Golden Rule:** Always validate syntax before applying changes.
```bash
sudo nginx -t
sudo systemctl reload nginx
```

### 3.3 Enterprise Vim Operations
When modifying configuration files via `vim`, utilize the following operational shortcuts:
- **Insert Mode:** Press `i`.
- **Save and Quit:** Press `Esc`, type `:wq`, press `Enter`.
- **Discard Changes:** Press `Esc`, type `:q!`, press `Enter`.
- **Clear File Contents:** Press `Esc`, type `:%d`, press `Enter`.
- **Block Commenting (Multiple Lines):**
  1. Press `Esc` and navigate to the target block.
  2. Press `Ctrl+v` to enter Visual Block mode.
  3. Use arrow keys to highlight the lines.
  4. Press `Shift+i` (capital I) to enter Insert mode.
  5. Type `#` and press `Esc` twice. The comment character will apply to all selected lines.

---

## 4. Virtual Host Implementation

Virtual hosts (server blocks) allow a single Nginx instance to route traffic to different document roots based on the requested domain name.

### 4.1 Directory and Asset Preparation
```bash
sudo mkdir -p /var/www/site1 /var/www/site2
echo '<h1>Application: Site 1</h1>' | sudo tee /var/www/site1/index.html
echo '<h1>Application: Site 2</h1>' | sudo tee /var/www/site2/index.html
sudo chmod -R 755 /var/www
sudo restorecon -Rv /var/www  # Enforces SELinux contexts
```

### 4.2 Server Block Configuration
Create `/etc/nginx/conf.d/site1.conf`:
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
*(Repeat for `site2.conf` adjusting paths and `server_name` accordingly).*

### 4.3 DNS and Host Resolution
For internal routing without public DNS, append the following to the client machine's `/etc/hosts` (Linux/Mac) or `C:\Windows\System32\drivers\etc\hosts` (Windows):
```text
<SERVER_IP>   site1.local site2.local
```

---

## 5. HTTPS and TLS Configuration

### 5.1 Let's Encrypt (Public Domains)
Requires a valid public domain and open ports 80/443.
```bash
sudo dnf install epel-release -y
sudo dnf install certbot python3-certbot-nginx -y
sudo certbot --nginx -d example.com -d www.example.com
```

### 5.2 Self-Signed Certificates (Internal Infrastructure)
Generate the cryptographic material:
```bash
sudo mkdir -p /etc/nginx/ssl
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/nginx/ssl/site1.key \
  -out    /etc/nginx/ssl/site1.crt \
  -subj "/CN=site1.local"
sudo chmod 600 /etc/nginx/ssl/site1.key
sudo restorecon -Rv /etc/nginx/ssl
```

**HTTPS Server Block Implementation:**
```nginx
server {
    listen 80;
    server_name site1.local;
    return 301 https://$host$request_uri;
}

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

---

## 6. Backend Infrastructure Preparation

When configuring Nginx as a reverse proxy, backend application servers must be prepared to accept internal traffic.

### 6.1 SELinux and Firewall Adjustments
Permit Nginx to initiate outbound network connections to backend ports:
```bash
sudo setsebool -P httpd_can_network_connect 1
sudo dnf install policycoreutils-python-utils -y
sudo semanage port -a -t http_port_t -p tcp 8081-8083
```
*Note: If the port range is already defined, use `-m` (modify) instead of `-a` (add).*

Open backend ports in the local firewall:
```bash
sudo firewall-cmd --permanent --add-port=8081-8083/tcp
sudo firewall-cmd --reload
```

### 6.2 Backend Server Blocks
Define isolated listeners for each backend node (e.g., `/etc/nginx/conf.d/backend1.conf`):
```nginx
server {
    listen 8081;
    root /var/www/b1;
}
```

---

## 7. Reverse Proxy Configuration

### 7.1 Standard Reverse Proxy
To route all traffic for a specific domain to a single backend, create a configuration file (e.g., `/etc/nginx/conf.d/proxy.conf`):

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
*The `proxy_set_header` directives are critical for maintaining client IP transparency and protocol awareness at the backend application layer.*

### 7.2 Path-Based Routing (Best Practice)
To route different URL paths to distinct backend services while maintaining header transparency:

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
*Note: The trailing slash in `proxy_pass` strips the matched location prefix before forwarding the request to the backend.*

---

## 8. Load Balancing Implementation

### 8.1 Upstream Configuration
Define an upstream group and route traffic via the proxy module. Standard naming convention dictates using `lb.conf` for load balancer configurations.

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

### 8.2 Resolving Default Server Conflicts
If accessing the server via IP address yields the default OS welcome page instead of the application, the default server block in `/etc/nginx/nginx.conf` is intercepting the traffic.

**Resolution:**
1. Open `/etc/nginx/nginx.conf`.
2. Locate the default `server` block at the bottom of the `http` context (typically listening on `[::]:80` with `server_name _;`).
3. Comment out the entire block using the Vim Visual Block method (`Ctrl+v`, select, `Shift+i`, `#`).
4. Reload Nginx.

### 8.3 Verification and Testing
**Terminal Verification:**
Execute a loop to verify Round Robin distribution. Using the IP address with the `Host` header is the most reliable testing method.
```bash
for i in 1 2 3 4 5 6; do curl -s -H "Host: proxy.local" http://10.10.10.210; done
```

**Browser Verification and Caching:**
Web browsers aggressively cache HTTP responses. If the browser displays only one backend node despite terminal verification showing rotation:
- Open an **Incognito/Private** browsing window.
- Perform a **Hard Refresh** (`Ctrl+Shift+R` or `Cmd+Shift+R`).
- Ensure the client `/etc/hosts` file maps `proxy.local` to the server IP.

---

## 9. Advanced Load Balancing Strategies

### 9.1 Distribution Algorithms
| Algorithm | Directive | Use Case |
| :--- | :--- | :--- |
| **Round Robin** | *(Default)* | Standard distribution across homogeneous nodes. |
| **Weighted** | `weight=3;` | Directing proportional traffic to higher-capacity nodes. |
| **Least Connections** | `least_conn;` | Optimal for long-lived connections or varying request processing times. |
| **IP Hash** | `ip_hash;` | Ensures session persistence by binding client IPs to specific backends. |

### 9.2 Failure Handling and High Availability
Configure health checks and fallback mechanisms within the upstream block:
```nginx
upstream mygroup {
    server 10.10.10.210:8081 max_fails=3 fail_timeout=30s;
    server 10.10.10.210:8082 max_fails=3 fail_timeout=30s;
    server 10.10.10.210:8083 backup;
}
```
| Parameter | Function |
| :--- | :--- |
| `max_fails=3` | Marks the node as unavailable after 3 consecutive failed health checks. |
| `fail_timeout=30s` | Defines the window for `max_fails` and the duration the node remains quarantined. |
| `backup` | Designates a standby node that only receives traffic when all primary nodes fail. |
| `down` | Permanently removes a node from the rotation pool. |

---

## 10. Forward Proxy Implementation (Squid)

Nginx lacks native support for the `CONNECT` method required for HTTPS tunneling. **Squid** is the enterprise standard for forward proxy deployments.

### 10.1 Installation and Service Control
```bash
sudo dnf install squid -y
sudo systemctl enable --now squid
```

### 10.2 Access Control Lists (ACLs)
Edit `/etc/squid/squid.conf` to restrict proxy access to authorized internal subnets:
```text
acl mynet src 192.168.1.0/24
http_access allow mynet
http_access deny all
http_port 3128
```
*Security Warning: Never expose a forward proxy to the public internet without strict ACLs and authentication.*

### 10.3 Firewall and Restart
```bash
sudo firewall-cmd --permanent --add-port=3128/tcp
sudo firewall-cmd --reload
sudo systemctl restart squid
```

---

## 11. Troubleshooting and Diagnostics

| Symptom | Root Cause & Resolution |
| :--- | :--- |
| **`nginx -t` fails** | Syntax error. Review the exact file path and line number provided in the output. |
| **Connection Refused** | No process is bound to the target port. Verify `listen` directives and execute `ss -tlnp \| grep nginx`. |
| **502 Bad Gateway** | Nginx cannot reach the backend. Verify backend service status, port numbers, and ensure `setsebool -P httpd_can_network_connect 1` is active. |
| **403 Forbidden** | Filesystem permissions (`chmod 755`) or SELinux contexts (`restorecon -Rv`) are incorrectly applied to the document root. |
| **Default OS Page Displayed** | The default `server` block in `nginx.conf` is intercepting the request. Comment it out or ensure the correct `Host` header is being passed. |
| **Load Balancing Not Rotating in Browser** | Browser caching is serving the initial response. Use Incognito mode or verify via `curl`. |
| **SELinux Port Denials** | Custom backend ports (e.g., 8081-8083) are blocked by SELinux. Register them using `semanage port -a -t http_port_t -p tcp <range>`. |

**Diagnostic Command Stack:**
```bash
sudo nginx -t
sudo systemctl status nginx
sudo tail -n 50 /var/log/nginx/error.log
sudo ss -tlnp | grep nginx
getenforce
```

---

## 12. Quick Reference

### Core Directives
| Directive | Context | Purpose |
| :--- | :--- | :--- |
| `server { }` | `http` | Defines a virtual host / server block. |
| `listen 80;` | `server` | Binds the block to a specific port. |
| `server_name` | `server` | Dictates the domain names the block responds to. |
| `root` / `index` | `server`, `location` | Defines the filesystem path and default document. |
| `proxy_pass` | `location` | Forwards requests to a backend URI or upstream group. |
| `upstream { }` | `http` | Defines a pool of backend servers for load balancing. |

### Standard Operating Procedure
1. Modify configuration in `/etc/nginx/conf.d/*.conf`.
2. Validate syntax: `sudo nginx -t`.
3. Apply changes: `sudo systemctl reload nginx`.
4. Verify via `curl` and application logs.
5. If errors occur, inspect `/var/log/nginx/error.log` and SELinux audit logs (`ausearch -m avc -ts recent`).
