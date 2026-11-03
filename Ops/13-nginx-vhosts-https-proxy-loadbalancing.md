# Nginx Next Steps: Virtual Hosts, HTTPS, Proxies and Load Balancing

A step-by-step guide for beginners. It assumes nginx is already installed and running (see the earlier nginx beginner guide).

**Contents**

1. The big picture (concepts)
2. Lab setup
3. Virtual hosts (multiple sites on one server)
4. HTTPS (Let's Encrypt and self-signed)
5. Reverse proxy
6. Load balancing
7. Forward proxy
8. Putting it all together
9. Troubleshooting
10. Summary and cheat sheet

> Commands are shown for **RHEL / Rocky / CentOS** and **Ubuntu / Debian** where they differ. Always run `sudo nginx -t` before `sudo systemctl reload nginx`.

---

# 1. The Big Picture (Concepts)

## 1.1 What is a proxy?

A **proxy** is a middleman. It sits between two sides and passes requests and answers back and forth.

There are two kinds, and the difference is **whose side the proxy is on**.

## 1.2 Forward proxy (works for the CLIENT)

The proxy sits in front of **users**. Users go to the internet *through* it. The websites see the proxy, not the users.

```
  Users (inside company)             Internet
 +--------+                                         +-----------+
 | PC 1   |--+                                  +-->| google.com|
 +--------+  |    +----------------+            |   +-----------+
 | PC 2   |--+--->| FORWARD PROXY  |------------+
 +--------+  |    | (e.g. Squid)   |            |   +-----------+
 | PC 3   |--+    +----------------+            +-->| youtube   |
 +--------+                                         +-----------+
   Users KNOW about the proxy.
   Websites only see the proxy's IP.
```

Used for:
- Blocking websites (filtering)
- Caching (saves bandwidth)
- Hiding users' IP addresses
- Logging what users visit

## 1.3 Reverse proxy (works for the SERVER)

The proxy sits in front of **servers**. Users talk to the proxy and think it is the real website. The real servers stay hidden behind it.

```
 Internet                                   Your private network
 +--------+      +-----------------+      +------------------+
 | User 1 |--+   |                 |----->| App server :3000 |
 +--------+  |   | REVERSE PROXY   |      +------------------+
 | User 2 |--+-->| (nginx)         |      +------------------+
 +--------+  |   |                 |----->| API server :4000 |
 | User 3 |--+   +-----------------+      +------------------+
   Users do NOT know about the backend servers.
```

Used for:
- Hiding and protecting backend servers
- HTTPS in one place (called **TLS termination**)
- Load balancing
- Caching and compression
- One public address for many apps

## 1.4 Forward vs reverse at a glance

| | Forward proxy | Reverse proxy |
|---|---|---|
| Sits in front of | Clients (users) | Servers |
| Who knows about it | Client is configured to use it | Client has no idea |
| Hides | The client | The server |
| Typical tool | Squid | Nginx, HAProxy |
| Main goal | Control and filter users | Protect, speed up, scale servers |

## 1.5 Load balancing

A **load balancer** is a reverse proxy that spreads requests over **several identical servers**, so no single server is overloaded and, if one dies, the others keep working.

```
                         +--> Server A  (gets 1st, 4th, 7th ...)
 Users --> LOAD BALANCER |--> Server B  (gets 2nd, 5th, 8th ...)
            (nginx)      +--> Server C  (gets 3rd, 6th, 9th ...)
```

## 1.6 Virtual hosts (server blocks)

One nginx can serve **many websites** from one IP. Nginx reads the **domain name** the browser asked for (the `Host` header) and picks the matching site.

```
                         +--> site1.local --> /var/www/site1
 Browser --> nginx :80 --|
                         +--> site2.local --> /var/www/site2
```

## 1.7 HTTPS and TLS

HTTPS = HTTP + encryption (TLS). A **certificate** proves the site is who it says it is. **Let's Encrypt** gives free certificates that renew automatically.

```
 HTTP  (port 80):   Browser ---- readable text ----> Server    (anyone can read it)
 HTTPS (port 443):  Browser ==== encrypted =========> Server    (only you two can read it)
```

---

# 2. Lab Setup

We will build everything on **one server** using different ports and names, so you need no extra machines (except for the optional forward proxy test).

Check nginx is healthy:
```bash
sudo systemctl status nginx
sudo nginx -t
```

Find your server IP and note it:
```bash
hostname -I
```
We will call it `SERVER_IP` below.

Open the web ports in the firewall:

```bash
# RHEL family
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload

# Ubuntu
sudo ufw allow 'Nginx Full'
```

Check SELinux (RHEL family):
```bash
getenforce        # Enforcing = on. Keep it on, the steps below handle it.
```

---

# 3. Virtual Hosts (Multiple Sites)

**Goal:** `site1.local` and `site2.local` both served by one nginx, each with its own page.

```
  site1.local ----+
                  +--> nginx (SERVER_IP:80) --+--> /var/www/site1/index.html
  site2.local ----+                           +--> /var/www/site2/index.html
```

## Step 3.1: Create the folders and pages

```bash
sudo mkdir -p /var/www/site1 /var/www/site2

echo '<h1>This is SITE 1</h1>' | sudo tee /var/www/site1/index.html
echo '<h1>This is SITE 2</h1>' | sudo tee /var/www/site2/index.html
```

## Step 3.2: Set permissions

```bash
sudo chmod -R 755 /var/www/site1 /var/www/site2
```

## Step 3.3: SELinux labels (RHEL family only)

SELinux must know these folders hold web content:
```bash
sudo dnf install policycoreutils-python-utils -y
sudo semanage fcontext -a -t httpd_sys_content_t "/var/www(/.*)?"
sudo restorecon -Rv /var/www
```

## Step 3.4: Create one config file per site

Site 1:
```bash
sudo nano /etc/nginx/conf.d/site1.conf
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

Site 2:
```bash
sudo nano /etc/nginx/conf.d/site2.conf
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

What matters here:

| Line | Meaning |
|---|---|
| `server { }` | One "virtual host" (one website) |
| `listen 80;` | Port to listen on |
| `server_name site1.local;` | The domain name this block answers to |
| `root` | Folder holding the files |
| `access_log` / `error_log` | Separate logs per site (very handy) |

## Step 3.5: Test and reload

```bash
sudo nginx -t
sudo systemctl reload nginx
```

## Step 3.6: Make the names work

`site1.local` is not a real domain, so tell your computer where it points.

**Option A: test from the server itself (no changes needed)**
```bash
curl -H "Host: site1.local" http://localhost
curl -H "Host: site2.local" http://localhost
```
You should see SITE 1, then SITE 2.

**Option B: test from your own PC's browser** by editing its hosts file:

- Linux / Mac: `sudo nano /etc/hosts`
- Windows: open Notepad **as Administrator**, then open `C:\Windows\System32\drivers\etc\hosts`

Add (use your real IP):
```
SERVER_IP   site1.local site2.local
```
Now open `http://site1.local` and `http://site2.local` in the browser.

## Step 3.7: How nginx picks a site

```
 Request arrives with  Host: site2.local
        |
        v
 Check each server_name for a match
        |
   match found? --yes--> use that server block
        |
        no
        v
 use the default (first server block, or the one marked default_server)
```

To make a catch-all that rejects unknown names:
```nginx
server {
    listen 80 default_server;
    server_name _;
    return 444;      # closes the connection without a reply
}
```

---

# 4. HTTPS

Two routes:

| Route | Use when |
|---|---|
| **4A: Let's Encrypt** | You have a **real public domain** pointing to a **public server** |
| **4B: Self-signed** | Lab or private network (browser will show a warning) |

## 4A. Let's Encrypt with Certbot

### Requirements (all must be true)

- You own a domain, e.g. `example.com`.
- An **A record** points that domain to your server's **public IP**.
- Ports **80 and 443** are open and reachable **from the internet**.
- Nginx has a `server` block with `server_name example.com;`.

> Let's Encrypt cannot give certificates for `.local` names or private IPs. If your server is only on a private network, use 4B.

```
   Let's Encrypt                         Your server
 +---------------+  1. "prove you own   +----------------+
 |               |     example.com"     |                |
 |  Certificate  | -------------------> |  certbot +     |
 |  Authority    | 2. fetches a secret  |  nginx :80     |
 |               | <------------------  |                |
 |               | 3. issues certificate|                |
 |               | -------------------> |  now HTTPS     |
 +---------------+                      +----------------+
```

### Step 4A.1: Install certbot

RHEL / Rocky:
```bash
sudo dnf install epel-release -y
sudo dnf install certbot python3-certbot-nginx -y
```

Ubuntu / Debian:
```bash
sudo apt install certbot python3-certbot-nginx -y
```

### Step 4A.2: Make a server block for your real domain

```nginx
server {
    listen 80;
    server_name example.com www.example.com;
    root /var/www/example;
    index index.html;
}
```
Then `sudo nginx -t && sudo systemctl reload nginx`.

### Step 4A.3: Get the certificate

```bash
sudo certbot --nginx -d example.com -d www.example.com
```

Certbot asks:
1. Your email (for expiry warnings).
2. Agree to terms.
3. Whether to **redirect HTTP to HTTPS**. Choose **redirect**.

It edits your nginx config automatically and reloads nginx.

### Step 4A.4: Test

Open `https://example.com`. You should see the padlock.

### Step 4A.5: Check automatic renewal

Certificates last **90 days**. Certbot renews them by itself with a timer:
```bash
sudo systemctl list-timers | grep certbot
sudo certbot renew --dry-run        # simulates a renewal; should say "success"
```

### Useful certbot commands
```bash
sudo certbot certificates           # list your certificates and expiry dates
sudo certbot renew                  # renew now
sudo certbot delete --cert-name example.com
```

---

## 4B. Self-signed Certificate (Lab)

You create your own certificate. Traffic is encrypted, but browsers warn you because no trusted authority signed it.

### Step 4B.1: Create the certificate

```bash
sudo mkdir -p /etc/nginx/ssl

sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/nginx/ssl/site1.key \
  -out    /etc/nginx/ssl/site1.crt \
  -subj "/CN=site1.local"

sudo chmod 600 /etc/nginx/ssl/site1.key
```

| Option | Meaning |
|---|---|
| `-x509` | Make a certificate directly |
| `-nodes` | No password on the key (nginx can start unattended) |
| `-days 365` | Valid for one year |
| `-newkey rsa:2048` | Generate a new 2048-bit key |
| `-keyout` | Private key file (**keep secret**) |
| `-out` | Certificate file (public) |
| `-subj "/CN=..."` | The site name |

### Step 4B.2: Update the site config

Replace `/etc/nginx/conf.d/site1.conf` with:

```nginx
# Redirect all HTTP to HTTPS
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

| Line | Meaning |
|---|---|
| `return 301 https://...` | Permanent redirect from HTTP to HTTPS |
| `listen 443 ssl;` | HTTPS port with TLS on |
| `ssl_certificate` | Path to the public certificate |
| `ssl_certificate_key` | Path to the private key |
| `ssl_protocols` | Only allow modern TLS versions |

### Step 4B.3: SELinux (RHEL family)
```bash
sudo restorecon -Rv /etc/nginx/ssl
```

### Step 4B.4: Test and reload
```bash
sudo nginx -t
sudo systemctl reload nginx
```

### Step 4B.5: Test
```bash
curl -k https://site1.local --resolve site1.local:443:127.0.0.1
```
(`-k` means "accept the self-signed certificate".) In a browser you will see a warning; click **Advanced** then **Proceed**.

---

# 5. Reverse Proxy

**Goal:** nginx receives the request and forwards it to an app running on another port. Users only ever see nginx.

```
 Browser --> nginx :80 --proxy_pass--> App on 127.0.0.1:3000
```

## Step 5.1: Create a tiny backend app

We use Python's built-in web server as the "app":

```bash
sudo mkdir -p /opt/app1
echo '<h1>Hello from the BACKEND app (port 3000)</h1>' | sudo tee /opt/app1/index.html
cd /opt/app1
python3 -m http.server 3000 --bind 127.0.0.1 &
```

Test it directly:
```bash
curl http://127.0.0.1:3000
```

`--bind 127.0.0.1` means the app is reachable **only from the server itself**. The outside world cannot reach it directly, only through nginx. This is a main benefit of a reverse proxy.

## Step 5.2: Create the proxy config

```bash
sudo nano /etc/nginx/conf.d/app1.conf
```
```nginx
server {
    listen 80;
    server_name app1.local;

    location / {
        proxy_pass http://127.0.0.1:3000;

        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

What each line does:

| Line | Meaning |
|---|---|
| `location /` | Applies to every URL |
| `proxy_pass` | Where to forward the request |
| `Host` | Passes the original domain name to the app |
| `X-Real-IP` | Passes the real visitor IP (otherwise the app only sees 127.0.0.1) |
| `X-Forwarded-For` | List of IPs the request passed through |
| `X-Forwarded-Proto` | Tells the app if the user used http or https |

## Step 5.3: SELinux (RHEL family only)

By default SELinux stops nginx from connecting to other services. Allow it:
```bash
sudo setsebool -P httpd_can_network_connect 1
```
(`-P` makes it permanent.) If you skip this, you get **502 Bad Gateway**.

## Step 5.4: Test and reload
```bash
sudo nginx -t
sudo systemctl reload nginx
curl -H "Host: app1.local" http://localhost
```
You should see "Hello from the BACKEND app".

## Step 5.5: Proxy different paths to different apps

```nginx
server {
    listen 80;
    server_name myapp.local;

    location / {                      # website
        proxy_pass http://127.0.0.1:3000;
    }
    location /api/ {                  # API
        proxy_pass http://127.0.0.1:4000;
    }
}
```
```
 /          --> app on :3000
 /api/...   --> app on :4000
```

## Step 5.6: HTTPS in front of the app (TLS termination)

```nginx
server {
    listen 443 ssl;
    server_name app1.local;
    ssl_certificate     /etc/nginx/ssl/site1.crt;
    ssl_certificate_key /etc/nginx/ssl/site1.key;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host              $host;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```
```
 Browser ==== HTTPS ====> nginx ---- plain HTTP ----> App
                   (decrypts here)   (private, local)
```
The app needs no HTTPS code; nginx handles it.

---

# 6. Load Balancing

**Goal:** three identical backends, and nginx spreads requests across them.

```
                          +--> Backend 1 :8081
 Browser --> nginx :80 ---|--> Backend 2 :8082
          (upstream)      +--> Backend 3 :8083
```

## Step 6.1: Start three backends

Each shows its own name so you can see who answered:
```bash
for i in 1 2 3; do
  sudo mkdir -p /opt/backend$i
  echo "<h1>Response from BACKEND $i</h1>" | sudo tee /opt/backend$i/index.html
  (cd /opt/backend$i && python3 -m http.server 808$i --bind 127.0.0.1 &)
done
```
Test them:
```bash
curl http://127.0.0.1:8081
curl http://127.0.0.1:8082
curl http://127.0.0.1:8083
```

## Step 6.2: Create the load balancer config

```bash
sudo nano /etc/nginx/conf.d/lb.conf
```
```nginx
upstream my_backends {
    server 127.0.0.1:8081;
    server 127.0.0.1:8082;
    server 127.0.0.1:8083;
}

server {
    listen 80;
    server_name lb.local;

    location / {
        proxy_pass http://my_backends;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
    }
}
```

`upstream` is a named group of servers. `proxy_pass http://my_backends;` sends traffic to the group.

## Step 6.3: Test and reload
```bash
sudo nginx -t
sudo systemctl reload nginx
```

## Step 6.4: Watch it balance

```bash
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

## Step 6.5: Load balancing methods

| Method | How to enable | Meaning | Use when |
|---|---|---|---|
| **Round robin** | (default) | Takes turns in order | Servers are equal |
| **Weighted** | `server ... weight=3;` | Stronger server gets more | Servers differ in power |
| **Least connections** | `least_conn;` | Goes to the least busy server | Requests take different times |
| **IP hash** | `ip_hash;` | Same visitor always hits same server | Sessions stored on the server ("sticky") |

Examples:
```nginx
# Weighted
upstream my_backends {
    server 127.0.0.1:8081 weight=3;   # gets 3x more traffic
    server 127.0.0.1:8082;
    server 127.0.0.1:8083;
}

# Least connections
upstream my_backends {
    least_conn;
    server 127.0.0.1:8081;
    server 127.0.0.1:8082;
}

# IP hash (sticky)
upstream my_backends {
    ip_hash;
    server 127.0.0.1:8081;
    server 127.0.0.1:8082;
}
```

## Step 6.6: Failure handling (high availability)

```nginx
upstream my_backends {
    server 127.0.0.1:8081 max_fails=3 fail_timeout=30s;
    server 127.0.0.1:8082 max_fails=3 fail_timeout=30s;
    server 127.0.0.1:8083 backup;      # used only if the others are down
}
```

| Option | Meaning |
|---|---|
| `max_fails=3` | After 3 failed attempts, mark the server as down |
| `fail_timeout=30s` | Keep it out of rotation for 30 seconds, then try again |
| `backup` | Spare server, used only when the main ones fail |
| `down` | Manually take a server out of rotation |

## Step 6.7: Try a failure

Find and stop backend 2:
```bash
pkill -f "http.server 8082"
for i in 1 2 3 4 5 6; do curl -s -H "Host: lb.local" http://localhost; done
```
Only backends 1 and 3 answer, and users see no error. That is the point of a load balancer.

Restart it:
```bash
(cd /opt/backend2 && python3 -m http.server 8082 --bind 127.0.0.1 &)
```

---

# 7. Forward Proxy

> **Nginx is not a good forward proxy.** It cannot handle HTTPS tunnelling (the `CONNECT` method) without extra modules. The standard tool is **Squid**. This section shows a basic setup.

```
 Your PC --> Squid proxy :3128 --> Internet
```

You need **two machines** (or two terminals on different hosts): one for Squid, one as the client.

## Step 7.1: Install Squid (on the proxy server)

RHEL family:
```bash
sudo dnf install squid -y
```
Ubuntu:
```bash
sudo apt install squid -y
```

## Step 7.2: Configure it

```bash
sudo nano /etc/squid/squid.conf
```

Key lines (RHEL ships with similar defaults):
```
# Who may use the proxy (change to YOUR network)
acl mynet src 192.168.1.0/24

# Block some sites (optional)
acl blocked dstdomain .facebook.com .tiktok.com

http_access deny blocked
http_access allow mynet
http_access deny all

http_port 3128
```

Order matters: Squid reads `http_access` rules top to bottom and stops at the first match.

## Step 7.3: Start and open the port

```bash
sudo systemctl enable --now squid
sudo firewall-cmd --permanent --add-port=3128/tcp     # RHEL
sudo firewall-cmd --reload
# Ubuntu: sudo ufw allow 3128/tcp
```

## Step 7.4: Test from a client

```bash
curl -x http://PROXY_IP:3128 http://example.com
curl -x http://PROXY_IP:3128 -I https://example.com
```

In a browser, set the proxy in **Settings > Network > Proxy**: host `PROXY_IP`, port `3128`.

## Step 7.5: Watch the logs

```bash
sudo tail -f /var/log/squid/access.log
```

## Forward proxy vs reverse proxy: quick recap

```
 FORWARD:   Client knows --> [Proxy] --> Internet sites
 REVERSE:   Client unaware --> [Proxy] --> your backend servers
```

---

# 8. Putting It All Together

A realistic setup combining everything:

```
                         INTERNET
                            |
                  https://example.com
                            |
                            v
              +---------------------------+
              |  NGINX                    |
              |  - HTTPS (Let's Encrypt)  |  <-- TLS termination
              |  - virtual hosts          |
              |  - reverse proxy          |
              |  - load balancing         |
              +---------------------------+
                  |         |         |
        plain HTTP, private network only
                  |         |         |
                  v         v         v
             +--------+ +--------+ +--------+
             | App 1  | | App 2  | | App 3  |
             | :8081  | | :8082  | | :8083  |
             +--------+ +--------+ +--------+
                              |
                              v
                        +-----------+
                        |  MySQL    |  (can store data on NFS)
                        +-----------+
```

Final nginx config for this:
```nginx
upstream my_backends {
    least_conn;
    server 127.0.0.1:8081 max_fails=3 fail_timeout=30s;
    server 127.0.0.1:8082 max_fails=3 fail_timeout=30s;
    server 127.0.0.1:8083 backup;
}

server {
    listen 80;
    server_name example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name example.com;

    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    location / {
        proxy_pass http://my_backends;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

---

# 9. Troubleshooting

| Problem | Likely cause and fix |
|---|---|
| `nginx -t` shows an error | Typo; read the file name and line number it prints |
| **502 Bad Gateway** | Backend is not running, wrong port, or SELinux blocking: `setsebool -P httpd_can_network_connect 1` |
| **504 Gateway Timeout** | Backend is too slow or stuck; check the app |
| **403 Forbidden** | File permissions (`chmod 755`) or SELinux label (`restorecon -Rv /var/www`) |
| **404 Not Found** | Wrong `root` path or missing file |
| Wrong site shows | `server_name` does not match the name you used; check your hosts file or `Host` header |
| Browser can't connect | Firewall closed, or nginx not running |
| Certbot fails | Domain not pointing to this server, or port 80 not open to the internet |
| Browser warns about the certificate | Normal for self-signed; use Let's Encrypt for real sites |
| `Address already in use` | Port taken: `sudo ss -tlnp \| grep :80` |
| Backend sees only 127.0.0.1 as visitor | Add the `X-Real-IP` / `X-Forwarded-For` headers (Step 5.2) |

Where to look first:
```bash
sudo nginx -t
sudo systemctl status nginx
sudo tail -n 30 /var/log/nginx/error.log
sudo ss -tlnp | grep nginx          # which ports nginx listens on
getenforce                          # SELinux state
sudo ausearch -m avc -ts recent     # recent SELinux denials (RHEL)
```

---

# 10. Summary and Cheat Sheet

## Concepts

| Term | One line |
|---|---|
| Forward proxy | Middleman for **users** going out to the internet |
| Reverse proxy | Middleman for **servers**, hides them from users |
| Load balancer | Reverse proxy that spreads traffic over many servers |
| Virtual host | Many websites on one nginx, chosen by domain name |
| TLS termination | Nginx handles HTTPS, backends use plain HTTP |
| Upstream | A named group of backend servers |

## Key directives

| Directive | Purpose |
|---|---|
| `server_name` | Which domain this site answers to |
| `root` / `index` | Where files are / default file |
| `listen 443 ssl` | HTTPS |
| `return 301 https://...` | Redirect to HTTPS |
| `proxy_pass` | Forward to a backend |
| `upstream { }` | Group of backends |
| `least_conn` / `ip_hash` / `weight=` | Load balancing methods |
| `max_fails` / `fail_timeout` / `backup` | Failure handling |

## Workflow to remember

```
 1. Edit config (/etc/nginx/conf.d/xxx.conf)
 2. sudo nginx -t                 <- ALWAYS test first
 3. sudo systemctl reload nginx
 4. Test with curl / browser
 5. If broken: error.log, then SELinux, then firewall
```

## Practice ideas

1. Add a third virtual host and give each site its own log file.
2. Use `curl -I` to see response headers and compare HTTP and HTTPS.
3. Stop one backend while running a loop of `curl` and watch nginx keep working.
4. Change to `ip_hash` and observe that you always hit the same backend.
5. Get a free domain or cheap domain and run a real Let's Encrypt certificate.
