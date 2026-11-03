# Apache httpd Complete Beginner Guide

Everything in one place: install, first page, virtual hosts, HTTPS, reverse proxy, load balancing, forward proxy. All labs use **simple static web pages**, nothing else to install.

**Contents**

1. What is Apache httpd?
2. Proxy concepts in plain words
3. Install, start and first page
4. Important files and commands
5. Virtual hosts (multiple sites)
6. HTTPS
7. Lab backends (three tiny sites)
8. Reverse proxy
9. Load balancing
10. Forward proxy
11. Apache-only features
12. Troubleshooting
13. Cheat sheet

> Commands are for **AlmaLinux / RHEL / Rocky**, where the package is `httpd`. On Ubuntu/Debian it is `apache2` with different paths (`/etc/apache2/`, `a2ensite`, `a2enmod`). SELinux steps apply to the RHEL family only.
> **Golden rule:** after changing a config, run `sudo httpd -t`, then `sudo systemctl reload httpd`. If you **add or remove a `Listen` line**, use `restart` instead of `reload`.

---

# 1. What is Apache httpd?

**Apache HTTP Server (httpd)** is one of the oldest and most widely used web servers. It sends web pages to browsers and can also act as a reverse proxy, load balancer and forward proxy.

```
  Browser                              SERVER (httpd)
 +--------------+   "give me the     +----------------------+
 |  Chrome      |    page" (HTTP)    |  httpd               |
 |              | -----------------> |    |                 |
 |              | <----------------- |    v                 |
 +--------------+   sends index.html |  /var/www/html/      |
                                     |  index.html          |
                                     +----------------------+
```

| Word | Meaning |
|---|---|
| Web server | Program that sends web pages |
| `DocumentRoot` | Folder where the site's files live |
| `index.html` | The default page shown |
| `Listen` | Opens a port |
| Module | A feature that Apache loads (e.g. SSL, proxy) |

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

## Virtual host

One httpd serving many websites, chosen by the **domain name** in the request.

```
                        +--> site1.local --> /var/www/site1
  Browser --> httpd :80 |
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
sudo dnf install httpd -y
httpd -v
```

## Step 3.2: Start
```bash
sudo systemctl enable --now httpd
sudo systemctl status httpd
```

| Command | What it does |
|---|---|
| `systemctl start httpd` | Start |
| `systemctl stop httpd` | Stop |
| `systemctl restart httpd` | Stop and start (needed after `Listen` changes) |
| `systemctl reload httpd` | Apply config changes gently |
| `systemctl enable httpd` | Start on boot |

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
Open `http://YOUR_SERVER_IP` in a browser. You should see the **Apache test page**.

> That test page comes from `/etc/httpd/conf.d/welcome.conf`. It appears **only while `/var/www/html` has no `index.html`**. Add your own page and it disappears.

## Step 3.5: Put your own page
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
</body>
</html>
```
Save (`Ctrl+O`, `Enter`, `Ctrl+X`) and refresh the browser. No restart needed.

---

# 4. Important Files and Commands

```
 /etc/httpd/
   |-- conf/httpd.conf          <- main config (contains "Listen 80")
   |-- conf.d/*.conf            <- your configs (auto-included, only *.conf)
   |-- conf.modules.d/*.conf    <- which modules to load
 /var/www/html/                 <- default web folder
 /var/log/httpd/
   |-- access_log               <- who visited
   |-- error_log                <- what went wrong
```

| Task | Command |
|---|---|
| Test config | `sudo httpd -t` (should say **Syntax OK**) |
| Show all virtual hosts | `sudo httpd -S` |
| List loaded modules | `httpd -M` |
| Reload | `sudo systemctl reload httpd` |
| See listening ports | `sudo ss -tlnp \| grep httpd` |
| Live error log | `sudo tail -f /var/log/httpd/error_log` |

> `httpd -S` is very useful: it prints every virtual host and which file defines it.
> Files in `conf.d` load **only if they end in `.conf`**. Renaming one to `.off` switches it off.

---

# 5. Virtual Hosts

**Goal:** `site1.local` and `site2.local` from one httpd.

## Step 5.1: Create folders and pages
```bash
sudo mkdir -p /var/www/site1 /var/www/site2
echo '<h1>This is SITE 1</h1>' | sudo tee /var/www/site1/index.html
echo '<h1>This is SITE 2</h1>' | sudo tee /var/www/site2/index.html
sudo chmod -R 755 /var/www
sudo restorecon -Rv /var/www          # SELinux label
```

## Step 5.2: One config file per site
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

| Line | Meaning |
|---|---|
| `<VirtualHost *:80>` | One website, any IP, port 80 |
| `ServerName` | Domain this site answers to |
| `DocumentRoot` | Folder with the files |
| `<Directory>` | Rules for that folder |
| `Require all granted` | Let everyone read it. **Without this you get 403 Forbidden** |
| `Options -Indexes` | Do not list folder contents if there is no index file |
| `ErrorLog` / `CustomLog` | Separate logs per site |

## Step 5.3: Test and reload
```bash
sudo httpd -t
sudo systemctl reload httpd
sudo httpd -S
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

## Step 5.5: How Apache picks a site
```
 Request with  Host: site2.local
        |
        v
 Compare to each ServerName / ServerAlias
        |
  match? --yes--> use that VirtualHost
        |
        no
        v
 use the FIRST VirtualHost loaded for that port
```
"First loaded" means the first file alphabetically in `conf.d`. Name a file `00-default.conf` to make it the default.

---

# 6. HTTPS

| Route | Use when |
|---|---|
| **6A Let's Encrypt** | Real public domain pointing to a public server |
| **6B Self-signed** | Lab / private network (browser shows a warning) |

## 6.0: Install mod_ssl (both routes)
```bash
sudo dnf install mod_ssl -y
```

> **Important:** `mod_ssl` adds `/etc/httpd/conf.d/ssl.conf`, which **already contains `Listen 443 https`**. Do **not** add another `Listen 443` anywhere, or httpd fails with "Address already in use".

## 6A. Let's Encrypt (certbot)

**Requirements, all of them:**
- You own a domain (e.g. `example.com`) with an A record pointing to your server's public IP.
- Ports 80 and 443 reachable from the internet.
- A VirtualHost with `ServerName example.com`.

> Let's Encrypt cannot issue certificates for `.local` names or private IPs. Use 6B for those.

```bash
sudo dnf install epel-release -y
sudo dnf install certbot python3-certbot-apache -y
sudo certbot --apache -d example.com -d www.example.com
```
Answer the prompts and choose to **redirect HTTP to HTTPS**. Certbot edits the Apache config and reloads.

Renewal (certificates last 90 days and renew automatically):
```bash
sudo systemctl list-timers | grep certbot
sudo certbot renew --dry-run
sudo certbot certificates
```

## 6B. Self-signed certificate (lab)

### Step 1: Create the certificate
```bash
sudo mkdir -p /etc/httpd/ssl
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/httpd/ssl/site1.key \
  -out    /etc/httpd/ssl/site1.crt \
  -subj "/CN=site1.local"
sudo chmod 600 /etc/httpd/ssl/site1.key
sudo restorecon -Rv /etc/httpd/ssl
```

| Option | Meaning |
|---|---|
| `-x509` | Make a certificate directly |
| `-nodes` | No password on the key |
| `-days 365` | Valid one year |
| `-keyout` / `-out` | Private key (secret) / public certificate |
| `-subj "/CN=..."` | The site name |

### Step 2: Replace `/etc/httpd/conf.d/site1.conf`
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
</VirtualHost>
```

| Line | Meaning |
|---|---|
| `Redirect permanent / https://...` | Permanent redirect to HTTPS |
| `<VirtualHost *:443>` | The HTTPS site |
| `SSLEngine on` | Turn TLS on |
| `SSLCertificateFile` | Public certificate |
| `SSLCertificateKeyFile` | Private key |
| `SSLProtocol` | Allow only modern TLS |

### Step 3: Test
```bash
sudo httpd -t
sudo systemctl reload httpd
curl -k https://site1.local --resolve site1.local:443:127.0.0.1
```
`-k` accepts the self-signed certificate. A browser will warn; click Advanced, then Proceed.

---

# 7. Lab Backends (Three Tiny Sites)

For the proxy labs we need "hidden" servers. We create **three small static sites** that only listen on the server's own loopback address.

```
                                  (hidden, reachable only from this server)
                              +--> Backend 1  127.0.0.1:8081
   FRONT DOOR :80  ---proxy---|--> Backend 2  127.0.0.1:8082
   (httpd)                    +--> Backend 3  127.0.0.1:8083
```

## Step 7.1: Prepare SELinux (once)
```bash
sudo setsebool -P httpd_can_network_connect 1     # lets httpd talk to backends
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

In Apache, a new port needs **two things**: a `Listen` line (opens the port) and a `<VirtualHost>` (says what to show).

```bash
sudo nano /etc/httpd/conf.d/backend1.conf
```
```apache
Listen 127.0.0.1:8081
<VirtualHost 127.0.0.1:8081>
    DocumentRoot /var/www/b1
    <Directory /var/www/b1>
        Require all granted
    </Directory>
</VirtualHost>
```
```bash
sudo nano /etc/httpd/conf.d/backend2.conf
```
```apache
Listen 127.0.0.1:8082
<VirtualHost 127.0.0.1:8082>
    DocumentRoot /var/www/b2
    <Directory /var/www/b2>
        Require all granted
    </Directory>
</VirtualHost>
```
```bash
sudo nano /etc/httpd/conf.d/backend3.conf
```
```apache
Listen 127.0.0.1:8083
<VirtualHost 127.0.0.1:8083>
    DocumentRoot /var/www/b3
    <Directory /var/www/b3>
        Require all granted
    </Directory>
</VirtualHost>
```

## Step 7.4: Test
```bash
sudo httpd -t
sudo systemctl restart httpd          # restart, because we added Listen lines
curl http://127.0.0.1:8081
curl http://127.0.0.1:8082
curl http://127.0.0.1:8083
```
Expected: `BACKEND 1`, `BACKEND 2`, `BACKEND 3`.

---

# 8. Reverse Proxy

**Goal:** port 80 shows a page that really comes from backend 1 on port 8081.

```
 curl --> httpd :80 --ProxyPass--> 127.0.0.1:8081 (BACKEND 1)
```

## Step 8.1: Check the proxy modules
On RHEL they are already enabled in `/etc/httpd/conf.modules.d/00-proxy.conf`:
```bash
httpd -M | grep proxy
```
You should see `proxy_module`, `proxy_http_module`, `proxy_balancer_module`, `proxy_connect_module`.

## Step 8.2: Create the front door
```bash
sudo nano /etc/httpd/conf.d/front.conf
```
```apache
<VirtualHost *:80>
    ServerName proxy.local

    ProxyPreserveHost On
    ProxyPass        / http://127.0.0.1:8081/
    ProxyPassReverse / http://127.0.0.1:8081/
</VirtualHost>
```

| Line | Meaning |
|---|---|
| `ServerName proxy.local` | Answers when the name is `proxy.local` |
| `ProxyPreserveHost On` | Passes the original domain name to the backend |
| `ProxyPass / http://...` | Forwards requests to backend 1 |
| `ProxyPassReverse` | Fixes redirects so users never see the backend address |

Apache automatically adds `X-Forwarded-For`, `X-Forwarded-Host` and `X-Forwarded-Server` headers.

## Step 8.3: Test
```bash
sudo httpd -t
sudo systemctl reload httpd
curl -H "Host: proxy.local" http://localhost
```
Expected: `BACKEND 1`. You asked port 80 but the content came from 8081. **That is a reverse proxy.**

## Step 8.4: Different paths to different backends
```apache
<VirtualHost *:80>
    ServerName proxy.local

    ProxyPass        /one/ http://127.0.0.1:8081/
    ProxyPassReverse /one/ http://127.0.0.1:8081/
    ProxyPass        /two/ http://127.0.0.1:8082/
    ProxyPassReverse /two/ http://127.0.0.1:8082/
</VirtualHost>
```
```
 /one/  --> BACKEND 1
 /two/  --> BACKEND 2
```
Put the more specific path first.

## Step 8.5: HTTPS in front of a backend (TLS termination)
```apache
<VirtualHost *:443>
    ServerName proxy.local
    SSLEngine on
    SSLCertificateFile    /etc/httpd/ssl/site1.crt
    SSLCertificateKeyFile /etc/httpd/ssl/site1.key

    ProxyPreserveHost On
    RequestHeader set X-Forwarded-Proto "https"
    ProxyPass        / http://127.0.0.1:8081/
    ProxyPassReverse / http://127.0.0.1:8081/
</VirtualHost>
```
```
 Browser ==== HTTPS ====> httpd ---- plain HTTP ----> Backend
```
The backend needs no HTTPS code. Apache handles it.

---

# 9. Load Balancing

**Goal:** the front door spreads requests over all three backends.

```
                          +--> BACKEND 1
 Browser --> httpd :80 ---|--> BACKEND 2
                          +--> BACKEND 3
```

## Step 9.1: Check the balancer modules
```bash
httpd -M | grep -E "balancer|lbmethod"
```
You need `proxy_balancer_module` and `lbmethod_byrequests_module` (enabled by default on RHEL).

## Step 9.2: Change the front door
```bash
sudo nano /etc/httpd/conf.d/front.conf
```
Replace the content with:
```apache
<Proxy "balancer://mygroup">
    BalancerMember http://127.0.0.1:8081
    BalancerMember http://127.0.0.1:8082
    BalancerMember http://127.0.0.1:8083
    ProxySet lbmethod=byrequests
</Proxy>

<VirtualHost *:80>
    ServerName proxy.local

    ProxyPreserveHost On
    ProxyPass        / balancer://mygroup/
    ProxyPassReverse / balancer://mygroup/
</VirtualHost>
```
`<Proxy "balancer://mygroup">` is a named list of backends. `ProxyPass / balancer://mygroup/` sends traffic to the list.

```bash
sudo httpd -t
sudo systemctl reload httpd
```

## Step 9.3: Watch it balance
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

## Step 9.4: Break a backend
```bash
sudo mv /etc/httpd/conf.d/backend2.conf /etc/httpd/conf.d/backend2.off
sudo systemctl restart httpd
for i in 1 2 3 4 5 6; do curl -s -H "Host: proxy.local" http://localhost; done
```
Backend 2 stops answering and the other two keep working. Bring it back:
```bash
sudo mv /etc/httpd/conf.d/backend2.off /etc/httpd/conf.d/backend2.conf
sudo systemctl restart httpd
```

If you see `Service Unavailable`, wait a minute (Apache waits before retrying a failed backend) or add `retry=5` to each `BalancerMember` so it retries after 5 seconds.

## Step 9.5: Balancing methods

| Method | How to enable | Meaning |
|---|---|---|
| Round robin | `lbmethod=byrequests` (default) | Takes turns by request count |
| By traffic | `lbmethod=bytraffic` | Balances by bytes transferred |
| Least busy | `lbmethod=bybusyness` | Goes to the least busy member |
| Weighted | `loadfactor=3` on a member | Stronger server gets more |
| Sticky sessions | `stickysession=ROUTEID` | Same visitor, same member (needs a cookie, more setup) |

```apache
# Weighted
BalancerMember http://127.0.0.1:8081 loadfactor=3
BalancerMember http://127.0.0.1:8082 loadfactor=1

# Least busy
ProxySet lbmethod=bybusyness
```

## Step 9.6: Failure handling options
```apache
<Proxy "balancer://mygroup">
    BalancerMember http://127.0.0.1:8081 retry=5 timeout=5
    BalancerMember http://127.0.0.1:8082 retry=5 timeout=5
    BalancerMember http://127.0.0.1:8083 status=+H
    ProxySet lbmethod=byrequests
</Proxy>
```

| Option | Meaning |
|---|---|
| `retry=5` | After a failure, wait 5 seconds before trying that member again |
| `timeout=5` | Give up on a member after 5 seconds |
| `status=+H` | **Hot standby**: spare, used only when the others are down |
| `status=+D` | Member disabled (taken out manually) |

---

# 10. Forward Proxy

Unlike nginx, **Apache can be a real forward proxy**, including HTTPS tunnelling (`mod_proxy_connect`).

```
 curl on this server --> httpd forward proxy :3128 --> internet website
```

> **Security:** a forward proxy open to everyone will be abused by strangers within hours. **Always restrict who can use it** and never expose it to the whole internet.

## Step 10.1: Create the config
```bash
sudo nano /etc/httpd/conf.d/forward.conf
```
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

| Line | Meaning |
|---|---|
| `Listen 3128` | Open port 3128 |
| `ProxyRequests On` | Turn on **forward** proxy mode |
| `ProxyVia On` | Add a header saying a proxy was used |
| `Require ip 127.0.0.1` | **Only this server** may use the proxy |

## Step 10.2: Restart and test
```bash
sudo httpd -t
sudo systemctl restart httpd
curl -x http://127.0.0.1:3128 http://example.com
curl -x http://127.0.0.1:3128 -I https://example.com
sudo tail -n 5 /var/log/httpd/forward_access.log
```
You should see the page fetched **through** the proxy. (Needs internet access from the server.)

## Step 10.3: Allow other machines (optional)

Change the allowed network to your own and open the port:
```apache
<Proxy "*">
    Require ip 192.168.1.0/24
</Proxy>
```
```bash
sudo firewall-cmd --permanent --add-port=3128/tcp
sudo firewall-cmd --reload
sudo systemctl restart httpd
```
If httpd refuses to start on 3128, check the port label: `sudo semanage port -l | grep 3128`.

Block some sites (optional):
```apache
ProxyBlock facebook.com tiktok.com
```

## Forward vs reverse in one picture
```
 FORWARD:  curl -x http://127.0.0.1:3128 http://example.com   (you know about the proxy)
 REVERSE:  curl -H "Host: proxy.local" http://localhost       (you know nothing)
```

---

# 11. Apache-Only Features

## 11.1 .htaccess (per-folder config)

Apache can read a file called `.htaccess` inside a folder to change behaviour **without editing the main config or reloading**.

```
 /var/www/site1/
   |-- index.html
   |-- .htaccess      <-- rules for this folder and below
```

Allow it for a folder:
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
Trade-offs: convenient for shared hosting, but slower, because Apache checks for it in every folder on every request. Prefer the main config if you control the server. Nginx has no `.htaccess`.

## 11.2 Modules

```bash
httpd -M                          # loaded modules
ls /etc/httpd/conf.modules.d/     # LoadModule lines live here
```

## 11.3 MPM (how Apache handles requests)

```bash
httpd -V | grep -i mpm
```

| MPM | How it works |
|---|---|
| prefork | One process per request (old) |
| worker | Processes with many threads |
| **event** | Like worker but handles idle connections efficiently (**default today**) |

---

# 12. Troubleshooting

| Problem | Likely cause and fix |
|---|---|
| `httpd -t` shows an error | Typo; read the file name and line number |
| `Could not reliably determine the server's fully qualified domain name` | Harmless warning; add `ServerName localhost` to `httpd.conf` to silence it |
| **Connection refused** | Nothing listening on that port. Check `ss -tlnp \| grep httpd` and your `Listen` lines |
| `403 Forbidden` | Missing `Require all granted`, wrong permissions, or SELinux label (`restorecon -Rv /var/www`) |
| `404 Not Found` | Wrong `DocumentRoot` or missing file |
| **503 Service Unavailable** (proxy) | Backend down, or SELinux: `setsebool -P httpd_can_network_connect 1` |
| Shows the Apache test page | No `index.html` in `/var/www/html`, or you forgot `-H "Host: ..."` |
| Shows the default page, not BACKEND | Forgot `-H "Host: proxy.local"` on `curl` |
| `Address already in use` | Duplicate `Listen` (often `Listen 443` twice) or another program (nginx) on that port: `sudo ss -tlnp \| grep :80` |
| Cannot listen on a new port | SELinux port label: `semanage port -a -t http_port_t -p tcp 8081` (or `-m`) |
| New `Listen` not working | Use `restart`, not `reload` |
| Wrong site appears | `ServerName` mismatch; run `httpd -S` |
| Certbot fails | Domain not pointing at the server, or port 80 not reachable from the internet |
| Forward proxy says 403 | Your IP is not allowed by `Require ip` |

**Remember:** "connection refused" means nothing is listening on that port. It is not a permission problem. In Apache, the **global `Listen` line** opens the port. If `Listen 80` is missing or changed, nothing listens on 80, even when a `<VirtualHost *:80>` exists.

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

# 13. Cheat Sheet

```
 Install      dnf install httpd
 Start        systemctl enable --now httpd
 Web folder   /var/www/html
 Config       /etc/httpd/conf/httpd.conf + /etc/httpd/conf.d/*.conf
 Test         httpd -t           (ALWAYS before reload)
 Show vhosts  httpd -S
 Reload       systemctl reload httpd   (restart after Listen changes)
 Logs         /var/log/httpd/error_log
```

| Directive | Purpose |
|---|---|
| `Listen 80` | Open a port |
| `<VirtualHost *:80>` | One website |
| `ServerName` | Domain it answers to |
| `DocumentRoot` | Where files are |
| `<Directory>` + `Require all granted` | Allow access to a folder |
| `SSLEngine on` + cert lines | HTTPS |
| `Redirect permanent / https://...` | Force HTTPS |
| `ProxyPass` / `ProxyPassReverse` | Reverse proxy |
| `<Proxy "balancer://x">` + `BalancerMember` | Load balancer |
| `lbmethod=` / `loadfactor=` / `status=+H` | Balancing methods |
| `ProxyRequests On` | Forward proxy (restrict it!) |

```
 Workflow:
 1. Edit /etc/httpd/conf.d/xxx.conf
 2. sudo httpd -t
 3. sudo systemctl reload httpd      (restart if you changed Listen)
 4. Test with curl / browser
 5. If broken: error_log -> SELinux -> firewall -> ss -tlnp
```

## Practice ideas
1. Add a third virtual host with its own logs.
2. Switch the balancer to `bybusyness` and then to weighted with `loadfactor`.
3. Make backend 3 a hot standby (`status=+H`) and break backends 1 and 2.
4. Enable `.htaccess` for one folder and add a redirect without reloading.
5. Check `httpd -V | grep -i mpm` and read the loaded modules with `httpd -M`.
