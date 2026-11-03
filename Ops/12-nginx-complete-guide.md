# Nginx Complete Beginner Guide

Everything in one place: install, first page, virtual hosts, HTTPS, reverse proxy, load balancing, forward proxy. All labs use **simple static web pages**, nothing else to install.

**Contents**

1. What is Nginx?
2. Proxy concepts in plain words
3. Install, start and first page
4. Important files and commands
5. Virtual hosts (multiple sites)
6. HTTPS
7. Lab backends (three tiny sites)
8. Reverse proxy
9. Load balancing
10. Forward proxy
11. Troubleshooting
12. Cheat sheet

> Commands are for **AlmaLinux / RHEL / Rocky**. For Ubuntu/Debian, install with `apt install nginx` and use `ufw` instead of `firewall-cmd`. SELinux steps apply to the RHEL family only.
> **Golden rule:** after changing a config, run `sudo nginx -t`, then `sudo systemctl reload nginx`.

---

# 1. What is Nginx?

**Nginx** (say "engine-x") is a **web server**. It waits for browsers to ask for pages and sends them back. It can also act as a reverse proxy and load balancer.

```
  Browser                              SERVER (nginx)
 +--------------+   "give me the     +----------------------+
 |  Chrome      |    page" (HTTP)    |  nginx               |
 |              | -----------------> |    |                 |
 |              | <----------------- |    v                 |
 +--------------+   sends index.html |  /usr/share/nginx/   |
                                     |  html/index.html     |
                                     +----------------------+
```

| Word | Meaning |
|---|---|
| Web server | Program that sends web pages |
| Document root | Folder where the site's files live |
| `index.html` | The default page shown |
| Port 80 / 443 | Doors for HTTP / HTTPS |
| Config file | Text file that controls nginx |

---

# 2. Proxy Concepts in Plain Words

A **proxy** is a **middleman**.

## Forward proxy: middleman for the USERS

```
  Users                  FORWARD PROXY               Internet
 +------+                +-----------+             +-----------+
 | PC 1 |--+             |           |------------>| websites  |
 | PC 2 |--+------------>|   proxy   |             +-----------+
 | PC 3 |--+             |           |
 +------+                +-----------+
 Users KNOW about it. Websites only see the proxy.
```
Use: filter sites, cache, hide user IPs.

## Reverse proxy: middleman for the SERVERS

```
  Users                 REVERSE PROXY              Hidden servers
 +------+               +-----------+              +-----------+
 | User |-------------->|   proxy   |------------->|  backend  |
 +------+               +-----------+              +-----------+
 Users do NOT know the backend exists.
```
Use: hide servers, HTTPS in one place, load balancing.

## Load balancer: reverse proxy with MANY servers behind

```
                          +--> Backend 1
  Users --> LOAD BALANCER |--> Backend 2
                          +--> Backend 3
```
Use: share the work, and keep running if one server dies.

## Virtual host (server block)

One nginx serving many websites, chosen by the **domain name** in the request.

```
                        +--> site1.local --> /var/www/site1
  Browser --> nginx :80 |
                        +--> site2.local --> /var/www/site2
```

## HTTPS

HTTP + encryption (TLS). A certificate proves who the site is.
```
 HTTP  (80):   Browser --- readable text ---> Server
 HTTPS (443):  Browser === encrypted ========> Server
```

---

# 3. Install, Start and First Page

## Step 3.1: Install
```bash
sudo dnf install nginx -y
nginx -v
```

## Step 3.2: Start
```bash
sudo systemctl enable --now nginx
sudo systemctl status nginx
```

| Command | What it does |
|---|---|
| `systemctl start nginx` | Start |
| `systemctl stop nginx` | Stop |
| `systemctl restart nginx` | Stop and start |
| `systemctl reload nginx` | Apply config changes gently |
| `systemctl enable nginx` | Start on boot |

## Step 3.3: Open the firewall
```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload
```

## Step 3.4: Test
```bash
hostname -I
curl -I http://localhost
```
Open `http://YOUR_SERVER_IP` in a browser. You should see the **"Welcome to nginx"** page.

## Step 3.5: Put your own page
```bash
sudo nano /usr/share/nginx/html/index.html
```
Paste:
```html
<!DOCTYPE html>
<html>
<head><title>My Nginx Page</title></head>
<body style="font-family:Arial;text-align:center;margin-top:80px">
  <h1>Hello from Nginx!</h1>
</body>
</html>
```
Save (`Ctrl+O`, `Enter`, `Ctrl+X`) and refresh the browser. No restart needed.

---

# 4. Important Files and Commands

```
 /etc/nginx/
   |-- nginx.conf          <- main config
   |-- conf.d/*.conf       <- your configs (auto-included, only *.conf)
 /usr/share/nginx/html/    <- default web folder (RHEL)
 /var/log/nginx/
   |-- access.log          <- who visited
   |-- error.log           <- what went wrong
```

| Task | Command |
|---|---|
| Test config | `sudo nginx -t` |
| Show full loaded config | `sudo nginx -T` |
| Reload | `sudo systemctl reload nginx` |
| See listening ports | `sudo ss -tlnp \| grep nginx` |
| Live access log | `sudo tail -f /var/log/nginx/access.log` |

> Files in `conf.d` load **only if they end in `.conf`**. Renaming one to `.off` or `.txt` switches it off.

---

# 5. Virtual Hosts

**Goal:** `site1.local` and `site2.local` from one nginx.

## Step 5.1: Create folders and pages
```bash
sudo mkdir -p /var/www/site1 /var/www/site2
echo '<h1>This is SITE 1</h1>' | sudo tee /var/www/site1/index.html
echo '<h1>This is SITE 2</h1>' | sudo tee /var/www/site2/index.html
sudo chmod -R 755 /var/www
sudo restorecon -Rv /var/www          # SELinux label (RHEL family)
```

## Step 5.2: One config file per site
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

| Line | Meaning |
|---|---|
| `server { }` | One website |
| `listen 80;` | Port to open |
| `server_name` | Domain this site answers to |
| `root` | Folder with the files |
| `access_log` / `error_log` | Separate logs per site |

## Step 5.3: Test and reload
```bash
sudo nginx -t
sudo systemctl reload nginx
```

## Step 5.4: Make the names work

From the server:
```bash
curl -H "Host: site1.local" http://localhost
curl -H "Host: site2.local" http://localhost
```
From your PC's browser, add this line to its hosts file (`/etc/hosts` on Linux/Mac; on Windows `C:\Windows\System32\drivers\etc\hosts` opened as Administrator):
```
SERVER_IP   site1.local site2.local
```

## Step 5.5: How nginx picks a site
```
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

# 6. HTTPS

| Route | Use when |
|---|---|
| **6A Let's Encrypt** | Real public domain pointing to a public server |
| **6B Self-signed** | Lab / private network (browser shows a warning) |

## 6A. Let's Encrypt (certbot)

**Requirements, all of them:**
- You own a domain (e.g. `example.com`) with an A record pointing to your server's public IP.
- Ports 80 and 443 reachable from the internet.
- A server block with `server_name example.com;`.

> Let's Encrypt cannot issue certificates for `.local` names or private IPs. Use 6B for those.

```bash
sudo dnf install epel-release -y
sudo dnf install certbot python3-certbot-nginx -y
sudo certbot --nginx -d example.com -d www.example.com
```
Answer the prompts (email, terms) and choose to **redirect HTTP to HTTPS**. Certbot edits nginx's config and reloads it.

Renewal (certificates last 90 days and renew automatically):
```bash
sudo systemctl list-timers | grep certbot
sudo certbot renew --dry-run
sudo certbot certificates
```

## 6B. Self-signed certificate (lab)

### Step 1: Create the certificate
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
|---|---|
| `-x509` | Make a certificate directly |
| `-nodes` | No password on the key |
| `-days 365` | Valid one year |
| `-keyout` / `-out` | Private key (secret) / public certificate |
| `-subj "/CN=..."` | The site name |

### Step 2: Update `/etc/nginx/conf.d/site1.conf`
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

| Line | Meaning |
|---|---|
| `return 301 https://...` | Permanent redirect to HTTPS |
| `listen 443 ssl;` | HTTPS port with TLS |
| `ssl_certificate` | Public certificate |
| `ssl_certificate_key` | Private key |

### Step 3: Test
```bash
sudo nginx -t
sudo systemctl reload nginx
curl -k https://site1.local --resolve site1.local:443:127.0.0.1
```
`-k` accepts the self-signed certificate. A browser will warn; click Advanced, then Proceed.

---

# 7. Lab Backends (Three Tiny Sites)

For the proxy labs we need "hidden" servers. We just create **three small static sites** that only listen on the server's own loopback address.

```
                                  (hidden, reachable only from this server)
                              +--> Backend 1  127.0.0.1:8081
   FRONT DOOR :80  ---proxy---|--> Backend 2  127.0.0.1:8082
   (nginx)                    +--> Backend 3  127.0.0.1:8083
```

## Step 7.1: Prepare SELinux (RHEL family, once)
```bash
sudo setsebool -P httpd_can_network_connect 1     # lets nginx talk to backends
sudo dnf install policycoreutils-python-utils -y
sudo semanage port -a -t http_port_t -p tcp 8081-8083
```
If `semanage` says "already defined", use `-m` instead of `-a`:
```bash
sudo semanage port -m -t http_port_t -p tcp 8081-8083
```
No firewall change is needed for 8081-8083 because they listen only on `127.0.0.1`.

## Step 7.2: Create the three pages
```bash
for i in 1 2 3; do
  sudo mkdir -p /var/www/b$i
  echo "BACKEND $i" | sudo tee /var/www/b$i/index.html
done
sudo chmod -R 755 /var/www
sudo restorecon -Rv /var/www
```

## Step 7.3: One config file per backend
```bash
sudo nano /etc/nginx/conf.d/backend1.conf
```
```nginx
server {
    listen 127.0.0.1:8081;
    root /var/www/b1;
}
```
```bash
sudo nano /etc/nginx/conf.d/backend2.conf
```
```nginx
server {
    listen 127.0.0.1:8082;
    root /var/www/b2;
}
```
```bash
sudo nano /etc/nginx/conf.d/backend3.conf
```
```nginx
server {
    listen 127.0.0.1:8083;
    root /var/www/b3;
}
```
`listen 127.0.0.1:8081;` means "only accept connections from this same server".

## Step 7.4: Test
```bash
sudo nginx -t
sudo systemctl reload nginx
curl http://127.0.0.1:8081
curl http://127.0.0.1:8082
curl http://127.0.0.1:8083
```
Expected: `BACKEND 1`, `BACKEND 2`, `BACKEND 3`.

---

# 8. Reverse Proxy

**Goal:** port 80 shows a page that really comes from backend 1 on port 8081.

```
 curl --> nginx :80 --proxy_pass--> 127.0.0.1:8081 (BACKEND 1)
```

## Step 8.1: Create the front door
```bash
sudo nano /etc/nginx/conf.d/front.conf
```
```nginx
server {
    listen 80;
    server_name proxy.local;

    location / {
        proxy_pass http://127.0.0.1:8081;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

| Line | Meaning |
|---|---|
| `location /` | Applies to every URL |
| `proxy_pass` | Where to forward the request |
| `Host` | Passes the original domain name |
| `X-Real-IP` | Passes the real visitor IP (otherwise the backend sees only 127.0.0.1) |
| `X-Forwarded-For` | List of IPs the request passed through |
| `X-Forwarded-Proto` | Whether the user used http or https |

## Step 8.2: Test
```bash
sudo nginx -t
sudo systemctl reload nginx
curl -H "Host: proxy.local" http://localhost
```
Expected: `BACKEND 1`. You asked port 80 but the content came from 8081. **That is a reverse proxy.**

## Step 8.3: Different paths to different backends
```nginx
server {
    listen 80;
    server_name proxy.local;

    location /one/ { proxy_pass http://127.0.0.1:8081/; }
    location /two/ { proxy_pass http://127.0.0.1:8082/; }
}
```
```
 /one/  --> BACKEND 1
 /two/  --> BACKEND 2
```
The trailing `/` on `proxy_pass` strips the `/one/` part before forwarding.

## Step 8.4: HTTPS in front of a backend (TLS termination)
```nginx
server {
    listen 443 ssl;
    server_name proxy.local;
    ssl_certificate     /etc/nginx/ssl/site1.crt;
    ssl_certificate_key /etc/nginx/ssl/site1.key;

    location / {
        proxy_pass http://127.0.0.1:8081;
        proxy_set_header Host              $host;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```
```
 Browser ==== HTTPS ====> nginx ---- plain HTTP ----> Backend
```
The backend needs no HTTPS code. Nginx handles it.

---

# 9. Load Balancing

**Goal:** the front door spreads requests over all three backends.

```
                          +--> BACKEND 1
 Browser --> nginx :80 ---|--> BACKEND 2
                          +--> BACKEND 3
```

## Step 9.1: Change the front door
```bash
sudo nano /etc/nginx/conf.d/front.conf
```
Replace the content with:
```nginx
upstream mygroup {
    server 127.0.0.1:8081;
    server 127.0.0.1:8082;
    server 127.0.0.1:8083;
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
`upstream mygroup { }` is a named list of backends. `proxy_pass http://mygroup;` sends traffic to the list.

```bash
sudo nginx -t
sudo systemctl reload nginx
```

## Step 9.2: Watch it balance
```bash
for i in 1 2 3 4 5 6; do curl -s -H "Host: proxy.local" http://localhost; done
```
Expected (taking turns, called **round robin**):
```
BACKEND 1
BACKEND 2
BACKEND 3
BACKEND 1
BACKEND 2
BACKEND 3
```

## Step 9.3: Break a backend
```bash
sudo mv /etc/nginx/conf.d/backend2.conf /etc/nginx/conf.d/backend2.off
sudo systemctl reload nginx
for i in 1 2 3 4 5 6; do curl -s -H "Host: proxy.local" http://localhost; done
```
Backend 2 never answers, and users see no error:
```
BACKEND 1
BACKEND 3
BACKEND 1
BACKEND 3
...
```
(The first request or two may be slower while nginx notices backend 2 is down.)

Bring it back:
```bash
sudo mv /etc/nginx/conf.d/backend2.off /etc/nginx/conf.d/backend2.conf
sudo systemctl reload nginx
```

## Step 9.4: Balancing methods

| Method | How to enable | Meaning | Use when |
|---|---|---|---|
| Round robin | (default) | Takes turns | Servers are equal |
| Weighted | `server ... weight=3;` | Stronger server gets more | Servers differ in power |
| Least connections | `least_conn;` | Goes to the least busy | Requests take different times |
| IP hash | `ip_hash;` | Same visitor, same backend | Sessions stored on the server |

```nginx
# Weighted
upstream mygroup {
    server 127.0.0.1:8081 weight=3;
    server 127.0.0.1:8082;
}

# Least connections
upstream mygroup {
    least_conn;
    server 127.0.0.1:8081;
    server 127.0.0.1:8082;
}

# IP hash
upstream mygroup {
    ip_hash;
    server 127.0.0.1:8081;
    server 127.0.0.1:8082;
}
```

## Step 9.5: Failure handling options
```nginx
upstream mygroup {
    server 127.0.0.1:8081 max_fails=3 fail_timeout=30s;
    server 127.0.0.1:8082 max_fails=3 fail_timeout=30s;
    server 127.0.0.1:8083 backup;
}
```

| Option | Meaning |
|---|---|
| `max_fails=3` | After 3 failures, mark the server down |
| `fail_timeout=30s` | Keep it out for 30 seconds, then retry |
| `backup` | Spare, used only when the others are down |
| `down` | Manually remove from rotation |

---

# 10. Forward Proxy

**Nginx cannot do a proper forward proxy.** It does not support the `CONNECT` method needed for HTTPS tunnelling. The standard tool is **Squid**.

```
 Client --> Squid :3128 --> Internet
```

## Step 10.1: Install and start Squid
```bash
sudo dnf install squid -y
sudo systemctl enable --now squid
```

The default config already allows the server itself and private networks (`localhost`, `localnet`).

## Step 10.2: Test from the server
```bash
curl -x http://127.0.0.1:3128 http://example.com
curl -x http://127.0.0.1:3128 -I https://example.com
sudo tail -n 5 /var/log/squid/access.log
```

## Step 10.3: Allow other machines (optional)
```bash
sudo nano /etc/squid/squid.conf
```
Key lines (use your own network):
```
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

> Never leave a forward proxy open to the whole internet. Restrict it to your own network.

## Forward vs reverse in one picture
```
 FORWARD:  curl -x http://PROXY:3128 http://example.com   (you know about the proxy)
 REVERSE:  curl -H "Host: proxy.local" http://localhost   (you know nothing)
```

---

# 11. Troubleshooting

| Problem | Likely cause and fix |
|---|---|
| `nginx -t` shows an error | Typo; read the file name and line number |
| **Connection refused** | Nothing is listening on that port. Check `ss -tlnp \| grep nginx` and the `listen` lines |
| Browser cannot connect | Firewall closed, or nginx not running |
| `403 Forbidden` | Permissions (`chmod -R 755 /var/www`) or SELinux label (`restorecon -Rv /var/www`) |
| `404 Not Found` | Wrong `root` path or file missing |
| **502 Bad Gateway** | Backend down, wrong port, or SELinux: `setsebool -P httpd_can_network_connect 1` |
| **504 Gateway Timeout** | Backend too slow |
| Cannot listen on 8081-8083 | SELinux port label: `semanage port -a -t http_port_t -p tcp 8081-8083` (or `-m`) |
| Shows default page, not BACKEND | You forgot `-H "Host: proxy.local"` on `curl` |
| Wrong site appears | `server_name` mismatch; check your hosts file |
| `Address already in use` | Another program (or httpd) is on that port: `sudo ss -tlnp \| grep :80` |
| Certbot fails | Domain not pointing at the server, or port 80 not reachable from the internet |
| Config change has no effect | File does not end in `.conf`, or you forgot to reload |

**Remember:** "connection refused" means nothing is listening on that port. It is not a permission problem. In nginx, the `listen` lines inside your `server` blocks decide which ports are open. If every block drops port 80, nothing listens on 80.

Where to look first:
```bash
sudo nginx -t
sudo systemctl status nginx
sudo tail -n 30 /var/log/nginx/error.log
sudo ss -tlnp | grep nginx
getenforce
sudo ausearch -m avc -ts recent
```

---

# 12. Cheat Sheet

```
 Install      dnf install nginx
 Start        systemctl enable --now nginx
 Web folder   /usr/share/nginx/html
 Config       /etc/nginx/nginx.conf + /etc/nginx/conf.d/*.conf
 Test         nginx -t           (ALWAYS before reload)
 Reload       systemctl reload nginx
 Logs         /var/log/nginx/error.log
```

| Directive | Purpose |
|---|---|
| `server { }` | One website |
| `listen 80;` | Open a port |
| `server_name` | Domain it answers to |
| `root` / `index` | Files and default page |
| `listen 443 ssl;` + cert lines | HTTPS |
| `return 301 https://...` | Force HTTPS |
| `proxy_pass` | Reverse proxy |
| `upstream { }` | Group of backends |
| `least_conn` / `ip_hash` / `weight=` | Balancing methods |
| `max_fails` / `fail_timeout` / `backup` | Failure handling |

```
 Workflow:
 1. Edit /etc/nginx/conf.d/xxx.conf
 2. sudo nginx -t
 3. sudo systemctl reload nginx
 4. Test with curl / browser
 5. If broken: error.log -> SELinux -> firewall -> ss -tlnp
```

## Practice ideas
1. Add a third virtual host with its own logs.
2. Switch the load balancer to `ip_hash` and notice you always hit the same backend.
3. Make backend 3 a `backup` and break backends 1 and 2.
4. Add HTTPS to a virtual host with a self-signed certificate.
5. Use `curl -I` to compare HTTP and HTTPS response headers.
