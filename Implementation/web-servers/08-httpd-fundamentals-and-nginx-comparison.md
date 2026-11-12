# Apache httpd Fundamentals (and How It Differs from Nginx)

Same topics as the nginx guides, done with **Apache httpd**, so you can compare side by side: install, deploy a page, virtual hosts, HTTPS, reverse proxy, load balancing, forward proxy.

**Contents**

1. What is Apache httpd?
2. Apache vs Nginx: the key differences
3. Install and first page
4. Important files, folders and commands
5. Virtual hosts
6. HTTPS
7. Reverse proxy
8. Load balancing
9. Forward proxy
10. Apache-only features (.htaccess, modules, MPM)
11. Side-by-side config comparison
12. Troubleshooting
13. Summary and cheat sheet

> Main commands are for **RHEL / AlmaLinux / Rocky**, where the package is `httpd`. Ubuntu/Debian differences are noted, because there it is called `apache2` and is laid out differently (see section 4).

> **Running nginx on the same machine?** Both want port 80. For this lab either stop nginx (`sudo systemctl disable --now nginx`) or use a separate machine. Backend test apps here use ports **9000-9003** so they do not clash with the nginx labs.

---

## 1. What is Apache httpd?

**Apache HTTP Server (httpd)** is one of the oldest and most widely used web servers. Like nginx, it receives requests from browsers and sends back pages, and it can also act as a reverse proxy, load balancer and forward proxy.

```
  Browser                               SERVER (httpd)
 +--------------+   "give me the     +--------------------+
 |              |    page" (HTTP)    |  httpd             |
 |  Chrome      | -----------------> |    |               |
 |              | <----------------- |    v               |
 +--------------+   sends index.html |  /var/www/html/    |
                                     +--------------------+
```

---

## 2. Apache vs Nginx: Key Differences

### 2.1 How they handle many visitors

```
 APACHE (process/thread based, "event" MPM by default today)
 +---------------------------------------------+
 | Parent process                              |
 |   +-- Worker process --> many threads       |
 |   +-- Worker process --> many threads       |
 |  (each request is handled by a thread)      |
 +---------------------------------------------+

 NGINX (event driven)
 +---------------------------------------------+
 | Master process                              |
 |   +-- Worker (1 per CPU) handles THOUSANDS  |
 |   +-- Worker  of connections each with an   |
 |       event loop (async, non-blocking)      |
 +---------------------------------------------+
```

Modern Apache with the **event MPM** is much closer to nginx than old Apache was, but nginx still tends to use less memory under very high connection counts.

### 2.2 Comparison table

| Topic | Apache httpd | Nginx |
|---|---|---|
| Design | Process/thread based (MPMs) | Event-driven, asynchronous |
| Static files | Good | Very fast, light on memory |
| Dynamic content (PHP, etc.) | Can run PHP **inside** Apache (`mod_php`) or via PHP-FPM | Always hands off to an outside app (PHP-FPM, etc.) |
| Config style | `<VirtualHost>`, `<Directory>`, `Listen` directives | `server { }`, `location { }` blocks |
| Per-folder config | **`.htaccess` supported** (no restart needed) | **No** `.htaccess`; edit central config and reload |
| Modules | Loaded dynamically at runtime (`LoadModule`) | Mostly built in; some dynamic modules exist |
| Where `listen` lives | Global `Listen` line in `httpd.conf`, plus `<VirtualHost *:port>` | Inside each `server { }` block |
| Reverse proxy | Yes (`mod_proxy`) | Yes (`proxy_pass`), very popular |
| Load balancing | Yes (`mod_proxy_balancer`) | Yes (`upstream`) |
| Forward proxy | **Yes** (`ProxyRequests On`, with care) | Poor (no real HTTPS `CONNECT`) |
| Default test page | "Apache HTTP Server Test Page" | "Welcome to nginx" |
| Config test command | `httpd -t` or `apachectl configtest` | `nginx -t` |
| Default user | `apache` (RHEL), `www-data` (Ubuntu) | `nginx` (RHEL), `www-data` (Ubuntu) |

### 2.3 When to choose which

| Choose Apache when | Choose Nginx when |
|---|---|
| You need `.htaccess` (shared hosting, many WordPress setups) | You want a fast reverse proxy / load balancer front end |
| You want a forward proxy | You serve lots of static files or many connections |
| You like lots of loadable modules and rich per-directory control | You want a small memory footprint |
| You run older apps that expect Apache | You are building modern apps, containers, microservices |

Many real setups use **both**: nginx in front (reverse proxy, HTTPS, caching), Apache behind it running the app.

---

## 3. Install and First Page

### Step 3.1: Install

RHEL / AlmaLinux / Rocky:
```bash
sudo dnf install httpd -y
```
Ubuntu / Debian:
```bash
sudo apt update
sudo apt install apache2 -y
```

Check the version:
```bash
httpd -v            # Ubuntu: apache2 -v
```

### Step 3.2: Start it

RHEL family:
```bash
sudo systemctl enable --now httpd
sudo systemctl status httpd
```
Ubuntu (it starts automatically after install):
```bash
sudo systemctl enable --now apache2
sudo systemctl status apache2
```

### Step 3.3: Open the firewall

```bash
# RHEL family
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload

# Ubuntu
sudo ufw allow 'Apache Full'
```

### Step 3.4: Test

```bash
hostname -I
curl -I http://localhost
```
Open `http://YOUR_SERVER_IP` in a browser. You should see the Apache test page.

> **Why that test page?** On RHEL the page is created by `/etc/httpd/conf.d/welcome.conf`. It shows **only when `/var/www/html` has no `index.html`**. As soon as you add your own `index.html`, your page replaces it.

### Step 3.5: Deploy your own page

```bash
sudo nano /var/www/html/index.html
```
Paste:
```html
<!DOCTYPE html>
<html>
<head><title>My Apache Page</title></head>
<body style="font-family:Arial;text-align:center;margin-top:80px">
  <h1>Hello from Apache httpd!</h1>
  <p>My first page is live.</p>
</body>
</html>
```
Save (`Ctrl+O`, `Enter`, `Ctrl+X`) and refresh the browser. No restart needed.

SELinux note: files you create inside `/var/www/html` get the right label automatically. If you **move** files from elsewhere, run `sudo restorecon -Rv /var/www/html`.

---

## 4. Important Files, Folders and Commands

### RHEL family (httpd)

```
 /etc/httpd/
   |-- conf/httpd.conf          <- main config (has "Listen 80")
   |-- conf.d/*.conf            <- your extra configs (auto-included)
   |-- conf.modules.d/*.conf    <- which modules to load (LoadModule)
 /var/www/html/                 <- default document root
 /var/log/httpd/
   |-- access_log
   |-- error_log
```

### Ubuntu / Debian (apache2)

```
 /etc/apache2/
   |-- apache2.conf             <- main config
   |-- ports.conf               <- "Listen 80"
   |-- sites-available/*.conf   <- site configs you write
   |-- sites-enabled/           <- symlinks to enabled sites
   |-- mods-available/ , mods-enabled/
 /var/www/html/
 /var/log/apache2/
```

Ubuntu helper commands:
```bash
sudo a2ensite mysite      # enable a site
sudo a2dissite mysite     # disable a site
sudo a2enmod ssl proxy    # enable modules
sudo a2dismod ssl
```

### Command comparison

| Task | httpd (RHEL) | apache2 (Ubuntu) | nginx |
|---|---|---|---|
| Test config | `httpd -t` | `apache2ctl configtest` | `nginx -t` |
| Reload | `systemctl reload httpd` | `systemctl reload apache2` | `systemctl reload nginx` |
| Show vhosts | `httpd -S` | `apache2ctl -S` | `nginx -T` |
| List modules | `httpd -M` | `apache2ctl -M` | `nginx -V` |
| Logs | `/var/log/httpd/` | `/var/log/apache2/` | `/var/log/nginx/` |

**`httpd -S` is very useful.** It prints every virtual host and which file defines it.

**Golden rule (same as nginx):** change a config, run `sudo httpd -t`, then `sudo systemctl reload httpd`.

---

## 5. Virtual Hosts

**Goal:** `site1.local` and `site2.local` from one httpd.

```
  site1.local ----+
                  +--> httpd (SERVER_IP:80) --+--> /var/www/site1
  site2.local ----+                           +--> /var/www/site2
```

### Step 5.1: Create folders and pages

```bash
sudo mkdir -p /var/www/site1 /var/www/site2
echo '<h1>This is SITE 1 (Apache)</h1>' | sudo tee /var/www/site1/index.html
echo '<h1>This is SITE 2 (Apache)</h1>' | sudo tee /var/www/site2/index.html
sudo chmod -R 755 /var/www/site1 /var/www/site2
```

### Step 5.2: SELinux labels (RHEL family)

```bash
sudo dnf install policycoreutils-python-utils -y
sudo semanage fcontext -a -t httpd_sys_content_t "/var/www(/.*)?"
sudo restorecon -Rv /var/www
```
(The rule may already exist; if `semanage` says it is already defined, just run the `restorecon` line.)

### Step 5.3: Create one config per site

```bash
sudo nano /etc/httpd/conf.d/site1.conf
```
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

```bash
sudo nano /etc/httpd/conf.d/site2.conf
```
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

(Ubuntu: save as `/etc/apache2/sites-available/site1.conf`, use `/var/log/apache2/...` for logs, then run `sudo a2ensite site1 site2`.)

What each line means:

| Line | Meaning |
|---|---|
| `<VirtualHost *:80>` | One website, on any IP, port 80 |
| `ServerName` | Domain name this site answers to |
| `DocumentRoot` | Folder with the files |
| `<Directory ...>` | Rules for that folder |
| `Require all granted` | Let everyone read it (without this you get **403 Forbidden**) |
| `Options -Indexes` | Do not list folder contents if there is no index file |
| `ErrorLog` / `CustomLog` | Separate logs for each site |

### Step 5.4: Test and reload

```bash
sudo httpd -t              # should say "Syntax OK"
sudo systemctl reload httpd
sudo httpd -S              # shows both vhosts
```

### Step 5.5: Make the names work

From the server:
```bash
curl -H "Host: site1.local" http://localhost
curl -H "Host: site2.local" http://localhost
```
From your PC, add this line to its hosts file (`/etc/hosts` on Linux/Mac, `C:\Windows\System32\drivers\etc\hosts` as Administrator on Windows):
```
SERVER_IP   site1.local site2.local
```

### Step 5.6: How Apache picks the site

```
 Request with  Host: site2.local
        |
        v
 Compare to each ServerName (and ServerAlias)
        |
   match? --yes--> use that VirtualHost
        |
        no
        v
 use the FIRST VirtualHost Apache loaded for that port
```

"First loaded" means alphabetical order of the files in `conf.d`. To control the default, name a file `00-default.conf` so it sorts first.

---

## 6. HTTPS

| Route | Use when |
|---|---|
| **6A: Let's Encrypt** | Real public domain pointing at a public server |
| **6B: Self-signed** | Lab or private network (browser warns) |

### 6.0: Install mod_ssl (both routes)

RHEL family:
```bash
sudo dnf install mod_ssl -y
```
Ubuntu:
```bash
sudo a2enmod ssl
```

> **Important (RHEL):** `mod_ssl` adds `/etc/httpd/conf.d/ssl.conf`, which already contains `Listen 443 https`. **Do not add another `Listen 443`** anywhere, or httpd fails with "Address already in use".

### 6A. Let's Encrypt with certbot

Requirements: a real domain with an A record pointing to the server, ports 80 and 443 reachable from the internet, and a VirtualHost whose `ServerName` is that domain.

```bash
# RHEL family
sudo dnf install epel-release -y
sudo dnf install certbot python3-certbot-apache -y

# Ubuntu
sudo apt install certbot python3-certbot-apache -y
```

Get the certificate:
```bash
sudo certbot --apache -d example.com -d www.example.com
```
Answer the prompts (email, terms) and choose to **redirect HTTP to HTTPS**. Certbot edits the Apache config for you and reloads.

Check renewal:
```bash
sudo systemctl list-timers | grep certbot
sudo certbot renew --dry-run
sudo certbot certificates
```

### 6B. Self-signed certificate (test environment)

#### Step 1: Create the certificate
```bash
sudo mkdir -p /etc/httpd/ssl
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/httpd/ssl/site1.key \
  -out    /etc/httpd/ssl/site1.crt \
  -subj "/CN=site1.local"
sudo chmod 600 /etc/httpd/ssl/site1.key
sudo restorecon -Rv /etc/httpd/ssl
```
(Ubuntu: use `/etc/apache2/ssl`.)

#### Step 2: Edit `/etc/httpd/conf.d/site1.conf`

Replace the content with:
```apache
# Redirect HTTP to HTTPS
<VirtualHost *:80>
    ServerName site1.local
    Redirect permanent / https://site1.local/
</VirtualHost>

# The HTTPS site
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

    ErrorLog  /var/log/httpd/site1.ssl.error.log
    CustomLog /var/log/httpd/site1.ssl.access.log combined
</VirtualHost>
```

| Line | Meaning |
|---|---|
| `Redirect permanent / https://...` | HTTP always goes to HTTPS (301) |
| `<VirtualHost *:443>` | HTTPS site |
| `SSLEngine on` | Turn TLS on for this site |
| `SSLCertificateFile` | Public certificate |
| `SSLCertificateKeyFile` | Private key (keep secret) |
| `SSLProtocol` | Allow only modern TLS versions |

#### Step 3: Test, reload and check
```bash
sudo httpd -t
sudo systemctl reload httpd
curl -k https://site1.local --resolve site1.local:443:127.0.0.1
```
`-k` accepts the self-signed certificate. Browsers will warn; click Advanced then Proceed.

---

## 7. Reverse Proxy

**Goal:** httpd receives the request and forwards it to an app on another port.

```
 Browser --> httpd :80 --ProxyPass--> App on 127.0.0.1:9000
```

### Step 7.1: Check proxy modules are loaded

On RHEL the proxy modules are already enabled in `/etc/httpd/conf.modules.d/00-proxy.conf`:
```bash
httpd -M | grep proxy
```
You should see `proxy_module`, `proxy_http_module`, `proxy_balancer_module`, `proxy_connect_module` and others.

Ubuntu:
```bash
sudo a2enmod proxy proxy_http headers
sudo systemctl restart apache2
```

### Step 7.2: Start a tiny backend app

```bash
sudo mkdir -p /opt/papp
echo '<h1>Hello from the BACKEND app (port 9000)</h1>' | sudo tee /opt/papp/index.html
cd /opt/papp
python3 -m http.server 9000 --bind 127.0.0.1 &
curl http://127.0.0.1:9000
```

### Step 7.3: Create the proxy config

```bash
sudo nano /etc/httpd/conf.d/app1.conf
```
```apache
<VirtualHost *:80>
    ServerName app1.local

    ProxyPreserveHost On
    ProxyPass        / http://127.0.0.1:9000/
    ProxyPassReverse / http://127.0.0.1:9000/

    ErrorLog  /var/log/httpd/app1.error.log
    CustomLog /var/log/httpd/app1.access.log combined
</VirtualHost>
```

| Line | Meaning |
|---|---|
| `ProxyPreserveHost On` | Pass the original `Host` header (the domain the user typed) to the app |
| `ProxyPass / http://...` | Forward requests to the backend |
| `ProxyPassReverse` | Rewrites redirects coming back from the backend so users never see the backend address |

Apache automatically adds `X-Forwarded-For`, `X-Forwarded-Host` and `X-Forwarded-Server` headers for you (nginx makes you add them by hand).

### Step 7.4: SELinux (RHEL family)

Without this you get **503 Service Unavailable**:
```bash
sudo setsebool -P httpd_can_network_connect 1
```

### Step 7.5: Test

```bash
sudo httpd -t
sudo systemctl reload httpd
curl -H "Host: app1.local" http://localhost
```
You should see "Hello from the BACKEND app".

### Step 7.6: Different paths to different apps

```apache
<VirtualHost *:80>
    ServerName myapp.local
    ProxyPass /api/ http://127.0.0.1:9100/
    ProxyPassReverse /api/ http://127.0.0.1:9100/
    ProxyPass / http://127.0.0.1:9000/
    ProxyPassReverse / http://127.0.0.1:9000/
</VirtualHost>
```
Put the **more specific path first** (`/api/` before `/`).

### Step 7.7: HTTPS in front of the app (TLS termination)

```apache
<VirtualHost *:443>
    ServerName app1.local
    SSLEngine on
    SSLCertificateFile    /etc/httpd/ssl/site1.crt
    SSLCertificateKeyFile /etc/httpd/ssl/site1.key

    ProxyPreserveHost On
    RequestHeader set X-Forwarded-Proto "https"
    ProxyPass        / http://127.0.0.1:9000/
    ProxyPassReverse / http://127.0.0.1:9000/
</VirtualHost>
```
```
 Browser ==== HTTPS ====> httpd ---- plain HTTP ----> App
```
`RequestHeader` needs `mod_headers` (loaded by default on RHEL; `a2enmod headers` on Ubuntu).

---

## 8. Load Balancing

**Goal:** three backends with httpd spreading requests among them.

```
                          +--> Backend 1 :9001
 Browser --> httpd :80 ---|--> Backend 2 :9002
          (balancer)      +--> Backend 3 :9003
```

### Step 8.1: Start three backends

```bash
for i in 1 2 3; do
  sudo mkdir -p /opt/lb$i
  echo "<h1>Response from BACKEND $i</h1>" | sudo tee /opt/lb$i/index.html
  (cd /opt/lb$i && python3 -m http.server 900$i --bind 127.0.0.1 &)
done
curl http://127.0.0.1:9001
curl http://127.0.0.1:9002
curl http://127.0.0.1:9003
```

### Step 8.2: Check the balancer modules

RHEL has them enabled by default:
```bash
httpd -M | grep -E "balancer|lbmethod"
```
You need `proxy_balancer_module` and `lbmethod_byrequests_module`.

Ubuntu:
```bash
sudo a2enmod proxy proxy_http proxy_balancer lbmethod_byrequests
sudo systemctl restart apache2
```

### Step 8.3: Create the load balancer config

```bash
sudo nano /etc/httpd/conf.d/lb.conf
```
```apache
<Proxy "balancer://mycluster">
    BalancerMember http://127.0.0.1:9001
    BalancerMember http://127.0.0.1:9002
    BalancerMember http://127.0.0.1:9003
    ProxySet lbmethod=byrequests
</Proxy>

<VirtualHost *:80>
    ServerName lb.local

    ProxyPreserveHost On
    ProxyPass        / balancer://mycluster/
    ProxyPassReverse / balancer://mycluster/
</VirtualHost>
```

`<Proxy "balancer://mycluster">` is Apache's equivalent of nginx's `upstream`.

### Step 8.4: Test

```bash
sudo httpd -t
sudo systemctl reload httpd
for i in 1 2 3 4 5 6; do curl -s -H "Host: lb.local" http://localhost; done
```
Expected (round robin):
```
BACKEND 1
BACKEND 2
BACKEND 3
BACKEND 1
BACKEND 2
BACKEND 3
```

### Step 8.5: Load balancing methods

| Method | How | Meaning |
|---|---|---|
| **byrequests** (default) | `lbmethod=byrequests` | Round robin by request count |
| **bytraffic** | `lbmethod=bytraffic` | Balances by bytes transferred |
| **bybusyness** | `lbmethod=bybusyness` | Sends to the least busy member (like nginx `least_conn`) |
| **Weighted** | `loadfactor=3` on a member | Stronger server gets more traffic |
| **Sticky sessions** | `stickysession=ROUTEID` | Same visitor keeps hitting the same member |

Examples:
```apache
# Weighted
<Proxy "balancer://mycluster">
    BalancerMember http://127.0.0.1:9001 loadfactor=3
    BalancerMember http://127.0.0.1:9002 loadfactor=1
    ProxySet lbmethod=byrequests
</Proxy>

# Least busy
<Proxy "balancer://mycluster">
    BalancerMember http://127.0.0.1:9001
    BalancerMember http://127.0.0.1:9002
    ProxySet lbmethod=bybusyness
</Proxy>
```

Sticky sessions need a cookie. Add `Header add Set-Cookie "ROUTEID=.%{BALANCER_WORKER_ROUTE}e; path=/" env=BALANCER_ROUTE_CHANGED` (requires `mod_headers`), give each member `route=1`, `route=2`, and set `ProxySet stickysession=ROUTEID`. This is more work than nginx's `ip_hash`.

### Step 8.4: Failure handling

```apache
<Proxy "balancer://mycluster">
    BalancerMember http://127.0.0.1:9001 retry=30 timeout=5
    BalancerMember http://127.0.0.1:9002 retry=30 timeout=5
    BalancerMember http://127.0.0.1:9003 status=+H
    ProxySet lbmethod=byrequests
</Proxy>
```

| Option | Meaning |
|---|---|
| `retry=30` | After a failure, wait 30 seconds before trying that member again |
| `timeout=5` | Give up on a member after 5 seconds |
| `status=+H` | **Hot standby**: used only when the others are down (nginx `backup`) |
| `status=+D` | Member is **disabled** (taken out manually) |

### Step 8.5: Try a failure

```bash
pkill -f "http.server 9002"
for i in 1 2 3 4 5 6; do curl -s -H "Host: lb.local" http://localhost; done
```
Only backends 1 and 3 answer. Restart it:
```bash
(cd /opt/lb2 && python3 -m http.server 9002 --bind 127.0.0.1 &)
```

---

## 9. Forward Proxy

Unlike nginx, **Apache can be a real forward proxy**, including HTTPS tunnelling (`mod_proxy_connect`).

```
 Your PC --> httpd (forward proxy :3128) --> Internet
```

> **Security warning:** a forward proxy open to everyone ("open proxy") will be abused by strangers within hours. **Always restrict who may use it** (shown below), and never expose it to the internet.

### Step 9.1: Create the config

```bash
sudo nano /etc/httpd/conf.d/forward-proxy.conf
```
```apache
Listen 3128

<VirtualHost *:3128>
    ProxyRequests On
    ProxyVia On

    <Proxy "*">
        # Only these clients may use the proxy (change to YOUR network)
        Require ip 192.168.1.0/24
    </Proxy>

    ErrorLog  /var/log/httpd/fproxy.error.log
    CustomLog /var/log/httpd/fproxy.access.log combined
</VirtualHost>
```

| Line | Meaning |
|---|---|
| `Listen 3128` | Add a new port (Apache needs this **before** a vhost can use it) |
| `ProxyRequests On` | Turns on forward proxy mode |
| `ProxyVia On` | Adds a `Via` header showing the request passed a proxy |
| `Require ip ...` | Restricts who can use it |

Optional: block sites:
```apache
<Proxy "*">
    Require ip 192.168.1.0/24
</Proxy>
ProxyBlock facebook.com tiktok.com
```

### Step 9.2: SELinux and firewall

```bash
sudo setsebool -P httpd_can_network_connect 1
sudo firewall-cmd --permanent --add-port=3128/tcp
sudo firewall-cmd --reload
```
If httpd refuses to start on 3128, check how SELinux labels that port: `sudo semanage port -l | grep 3128`.

### Step 9.3: Test from a client in the allowed network

```bash
sudo httpd -t && sudo systemctl reload httpd
curl -x http://PROXY_IP:3128 http://example.com
curl -x http://PROXY_IP:3128 -I https://example.com
```
Watch the log:
```bash
sudo tail -f /var/log/httpd/fproxy.access.log
```

---

## 10. Apache-Only Features

### 10.1 .htaccess (per-folder config)

Apache can read a file named `.htaccess` inside a folder to change behaviour **without editing the main config or reloading**.

```
 /var/www/site1/
   |-- index.html
   |-- .htaccess      <-- rules for this folder and below
```

Enable it for a folder:
```apache
<Directory /var/www/site1>
    AllowOverride All
    Require all granted
</Directory>
```

Example `.htaccess` (redirect and block a file):
```apache
Redirect permanent /old.html /new.html
<Files "secret.txt">
    Require all denied
</Files>
```

Trade-offs:
- Convenient (shared hosting, WordPress permalinks).
- **Slower**, because Apache checks for `.htaccess` in every folder on every request.
- Prefer putting rules in the main config and keep `AllowOverride None` when you control the server.
- Nginx has **no** `.htaccess`. You edit the config and reload.

### 10.2 Modules

Apache loads features as modules:
```bash
httpd -M                                   # list loaded modules
ls /etc/httpd/conf.modules.d/              # where LoadModule lines live
```
A line like `LoadModule rewrite_module modules/mod_rewrite.so` loads a module. Install extra ones with `dnf install mod_ssl`, `mod_security`, and so on.

### 10.3 MPM (how Apache handles requests)

```bash
httpd -V | grep -i mpm
```

| MPM | How it works | Notes |
|---|---|---|
| **prefork** | One process per request | Old; needed only for non-thread-safe `mod_php` |
| **worker** | Processes with many threads | Better memory use |
| **event** | Like worker, but handles idle keep-alive connections efficiently | **Default today**; closest to nginx's behaviour |

### 10.4 PHP

| Setup | Description |
|---|---|
| `mod_php` | PHP runs inside Apache (needs prefork) |
| **PHP-FPM** | PHP runs as a separate service; Apache or nginx forwards requests to it (recommended, works with event MPM) |

---

## 11. Side-by-Side Config Comparison

### Virtual host

```apache
# APACHE
<VirtualHost *:80>
    ServerName site1.local
    DocumentRoot /var/www/site1
    <Directory /var/www/site1>
        Require all granted
    </Directory>
</VirtualHost>
```
```nginx
# NGINX
server {
    listen 80;
    server_name site1.local;
    root /var/www/site1;
}
```

### HTTPS redirect

```apache
# APACHE
<VirtualHost *:80>
    ServerName site1.local
    Redirect permanent / https://site1.local/
</VirtualHost>
```
```nginx
# NGINX
server {
    listen 80;
    server_name site1.local;
    return 301 https://$host$request_uri;
}
```

### Reverse proxy

```apache
# APACHE
ProxyPreserveHost On
ProxyPass        / http://127.0.0.1:9000/
ProxyPassReverse / http://127.0.0.1:9000/
```
```nginx
# NGINX
location / {
    proxy_pass http://127.0.0.1:9000;
    proxy_set_header Host $host;
}
```

### Load balancing

```apache
# APACHE
<Proxy "balancer://mycluster">
    BalancerMember http://127.0.0.1:9001
    BalancerMember http://127.0.0.1:9002
    ProxySet lbmethod=byrequests
</Proxy>
ProxyPass / balancer://mycluster/
```
```nginx
# NGINX
upstream mycluster {
    server 127.0.0.1:9001;
    server 127.0.0.1:9002;
}
location / { proxy_pass http://mycluster; }
```

### Term mapping

| Concept | Apache | Nginx |
|---|---|---|
| A website | `<VirtualHost>` | `server { }` |
| Domain name | `ServerName` / `ServerAlias` | `server_name` |
| Web folder | `DocumentRoot` | `root` |
| Folder rules | `<Directory>` | `location { }` |
| Permission to read | `Require all granted` | (none needed, uses file permissions) |
| Port | `Listen` (global) + `<VirtualHost *:80>` | `listen` (inside server) |
| Backend group | `<Proxy "balancer://x">` | `upstream x { }` |
| Standby server | `status=+H` | `backup` |
| Weight | `loadfactor=` | `weight=` |
| Least connections | `lbmethod=bybusyness` | `least_conn;` |
| Pass to backend | `ProxyPass` | `proxy_pass` |
| Per-folder file | `.htaccess` | not available |

### The port 80 lesson (your earlier issue)

```
 APACHE:  "Listen 80" in httpd.conf opens the port.
          If that line is missing or changed, nothing listens on 80,
          even if a VirtualHost says *:80.

 NGINX:   "listen 80;" inside each server block opens the port.
          If every server block drops port 80, nothing listens on 80.
```
Either way, `ss -tlnp | grep -E "httpd|nginx"` shows what is really listening, and "connection refused" means **nothing is listening on that port**, not a permission block.

---

## 12. Troubleshooting

| Problem | Likely cause and fix |
|---|---|
| `httpd -t` shows an error | Read the file and line number it prints |
| `Could not reliably determine the server's fully qualified domain name` | Harmless warning; add `ServerName localhost` to `httpd.conf` to silence it |
| **403 Forbidden** | Missing `Require all granted` in `<Directory>`, wrong file permissions, or SELinux label (`restorecon -Rv /var/www`) |
| **404 Not Found** | Wrong `DocumentRoot` or missing file |
| **503 Service Unavailable** (proxy) | Backend down, or SELinux: `setsebool -P httpd_can_network_connect 1` |
| **502 Bad Gateway** | Backend sent a bad response or closed the connection |
| Page shows Apache test page | No `index.html` in the web folder, so `welcome.conf` page appears |
| `Address already in use` | Duplicate `Listen` line (often `Listen 443` twice) or another program (such as nginx) on that port: `sudo ss -tlnp \| grep :80` |
| Connection refused | Nothing listening on that port; check `Listen` and `ss -tlnp` |
| Wrong site appears | `ServerName` mismatch; check your hosts file or `Host` header, and run `httpd -S` |
| Cannot bind to a new port | SELinux: `sudo semanage port -a -t http_port_t -p tcp 8081` |
| Certbot fails | Domain not pointing at the server, or port 80 not reachable from the internet |

Where to look first:
```bash
sudo httpd -t
sudo httpd -S
sudo systemctl status httpd
sudo tail -n 30 /var/log/httpd/error_log
sudo ss -tlnp | grep httpd
getenforce
sudo ausearch -m avc -ts recent
```

---

## 13. Summary and Cheat Sheet

```
 Install      dnf install httpd
 Start        systemctl enable --now httpd
 Firewall     firewall-cmd --add-service=http / https
 Web folder   /var/www/html
 Config       /etc/httpd/conf/httpd.conf  +  /etc/httpd/conf.d/*.conf
 Test         httpd -t        (ALWAYS before reload)
 Reload       systemctl reload httpd
 Show vhosts  httpd -S
 Logs         /var/log/httpd/error_log
```

### Key directives

| Directive | Purpose |
|---|---|
| `Listen 80` | Open a port |
| `<VirtualHost *:80>` | Define a site |
| `ServerName` | The domain it answers to |
| `DocumentRoot` | Where files are |
| `<Directory>` + `Require all granted` | Allow access to a folder |
| `SSLEngine on` + cert lines | HTTPS |
| `Redirect permanent / https://...` | Force HTTPS |
| `ProxyPass` / `ProxyPassReverse` | Reverse proxy |
| `<Proxy "balancer://x">` + `BalancerMember` | Load balancer |
| `ProxyRequests On` | Forward proxy (restrict it!) |

### Workflow

```
 1. Edit config (/etc/httpd/conf.d/xxx.conf)
 2. sudo httpd -t              <- ALWAYS test first
 3. sudo systemctl reload httpd
 4. Test with curl / browser
 5. If broken: error_log, then SELinux, then firewall, then ss -tlnp
```

### Practice ideas

1. Run the **same** lab with nginx and with httpd on two VMs and compare the configs.
2. Enable `.htaccess` for one folder and add a redirect without reloading.
3. Check `httpd -V | grep -i mpm` and read the loaded modules with `httpd -M`.
4. Put **nginx in front** (reverse proxy, HTTPS) and **httpd behind** (app) on port 8081.
5. Stop one backend during a `curl` loop and watch the balancer skip it.
