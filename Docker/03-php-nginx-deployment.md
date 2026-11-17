# PHP on AlmaLinux with Nginx

Complete setup for serving a PHP application with Nginx and PHP-FPM on a fresh AlmaLinux VM. Written so a beginner can follow it, understand each command, and recreate the setup later.

---

## 1. Before / After

### Before

| Item | State |
|---|---|
| Web server | Not installed |
| PHP runtime | Not installed |
| Port 80 | Closed in firewalld |
| SELinux | Blocking Nginx to PHP-FPM connections by default |
| Browser access | Nothing listening |

### After

| Item | State |
|---|---|
| Web server | Nginx running, serving static files and proxying PHP |
| PHP runtime | PHP-FPM running as `apache`, listening on a unix socket |
| Port 80 | Open |
| SELinux | Allowed to connect Nginx to the PHP-FPM socket |
| Browser access | Shows the PHP app at `http://<vm-ip>/` |

Request flow after setup:

```
Browser  ->  Nginx (port 80)  ->  PHP-FPM (unix socket)  ->  index.php
```

Nginx serves static files (CSS, images) directly. It forwards `.php` requests to PHP-FPM over the socket at `/run/php-fpm/www.sock`.

### File layout after

```
/usr/share/nginx/html/
├── index.php
└── style.css

/etc/nginx/conf.d/php-app.conf     <-- your site config
/etc/nginx/nginx.conf              <-- default server block removed
/etc/php-fpm.d/www.conf            <-- listen = /run/php-fpm/www.sock
```

---

## 2. Key Concepts (read this first)

### The pieces

| Term | What it is |
|---|---|
| **Nginx** | Web server. Serves static files and forwards `.php` requests to PHP-FPM |
| **PHP-FPM** | PHP FastCGI Process Manager. Runs PHP code and returns HTML |
| **Unix socket** | A file-based channel at `/run/php-fpm/www.sock`. Nginx talks to PHP-FPM through it |
| **Web root** | `/usr/share/nginx/html`. Where site files live on disk |
| **SELinux** | Security layer. Blocks Nginx from connecting to the socket unless explicitly allowed |
| **firewalld** | Blocks port 80 unless explicitly opened |

### Request flow, step by step

1. Browser requests `http://vm-ip/index.php`.
2. Nginx receives the request on port 80.
3. The URL ends in `.php`, so Nginx forwards it to PHP-FPM over the unix socket.
4. PHP-FPM runs `index.php` and returns HTML.
5. Nginx sends the HTML back to the browser.

### Commands you'll meet

| Command | What it answers |
|---|---|
| `dnf install` | Install packages |
| `systemctl` | Start, enable, restart, or check services |
| `nginx -t` | Is the Nginx config valid? |
| `curl -I` | What headers does the server return? |
| `tail` | What do the logs say? |
| `firewall-cmd` | Is port 80 open? |
| `setsebool` / `restorecon` | Is SELinux allowing this? |
| `grep` | Find a line in a config file |

### The mental model

```
install  ->  configure  ->  deploy files  ->  open ports  ->  start services  ->  test
```

Each step feeds the next. Skip the SELinux or firewall step and the browser will fail even though the services are running.

---

## 3. Warnings

- **SELinux blocks the PHP socket by default.** Without `setsebool`, you get `502 Bad Gateway`.
- **firewalld blocks port 80 by default.** Without opening it, browsers cannot connect.
- **AlmaLinux `nginx.conf` ships a default `server { ... }` block** that conflicts with yours. Remove it, or Nginx fails with `duplicate default server`.
- **Files must be owned by `apache:apache`**, or Nginx returns `403 Forbidden`.
- **Always run `nginx -t`** after editing configs and before reloading.
- **Do not edit files as the wrong user.** Use `sudo` for system paths and keep app files owned by `apache`.

---

## 4. Step-by-Step

### Step 0: Inspect the current state

```bash
nginx -v          # is Nginx installed?
php -v            # is PHP installed?
sudo systemctl status nginx php-fpm   # are services running?
```

On a fresh VM these will fail or show "not found". That confirms a clean start.

### Step 1: Install packages

```bash
sudo dnf install -y epel-release
sudo dnf install -y nginx php php-fpm php-cli
```

- `dnf install` is the package manager.
- `-y` auto confirms prompts.
- `epel-release` adds the extra repository PHP needs.

Verify:

```bash
nginx -v
php -v
php-fpm -v
```

### Step 2: Create the application files

```bash
mkdir -p ~/php-app/src
cd ~/php-app/src
```

`mkdir -p` creates the folder and any missing parents.

Create `index.php`:

```bash
vim index.php
```

Paste:

```php
<?php
$name = "My First PHP App";
$time = date("l, F j, Y - H:i:s");
$host = gethostname();
?>
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title><?= $name ?></title>
    <link rel="stylesheet" href="style.css">
</head>
<body>
    <div class="card">
        <div class="emoji">🐘</div>
        <h1>Hello, PHP!</h1>
        <p class="lead">Running natively on AlmaLinux with Nginx and PHP-FPM.</p>

        <div class="info">
            <div class="row">
                <span class="label">PHP Version</span>
                <span class="value"><?= phpversion() ?></span>
            </div>
            <div class="row">
                <span class="label">Hostname</span>
                <span class="value"><?= $host ?></span>
            </div>
            <div class="row">
                <span class="label">Server Time</span>
                <span class="value"><?= $time ?></span>
            </div>
            <div class="row">
                <span class="label">Status</span>
                <span class="value online">Running</span>
            </div>
        </div>

        <p class="footer">Built with PHP on AlmaLinux</p>
    </div>
</body>
</html>
```

Create `style.css`:

```bash
vim style.css
```

Paste:

```css
* {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
}

body {
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    min-height: 100vh;
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 20px;
    color: #333;
}

.card {
    background: #ffffff;
    border-radius: 20px;
    padding: 40px;
    max-width: 420px;
    width: 100%;
    box-shadow: 0 20px 60px rgba(0, 0, 0, 0.3);
    text-align: center;
    animation: fadeIn 0.6s ease-out;
}

@keyframes fadeIn {
    from { opacity: 0; transform: translateY(20px); }
    to   { opacity: 1; transform: translateY(0); }
}

.emoji {
    font-size: 60px;
    margin-bottom: 16px;
}

h1 {
    font-size: 28px;
    margin-bottom: 8px;
    color: #1a202c;
}

.lead {
    color: #718096;
    font-size: 15px;
    margin-bottom: 28px;
    line-height: 1.6;
}

.info {
    background: #f7fafc;
    border-radius: 12px;
    padding: 8px 20px;
    margin-bottom: 24px;
}

.row {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 12px 0;
    border-bottom: 1px solid #e2e8f0;
}

.row:last-child { border-bottom: none; }

.label {
    color: #718096;
    font-size: 14px;
}

.value {
    color: #2d3748;
    font-weight: 600;
    font-size: 14px;
}

.online { color: #38a169; }

.footer {
    color: #a0aec0;
    font-size: 13px;
}

@media (max-width: 480px) {
    .card { padding: 30px 24px; }
    h1 { font-size: 24px; }
}
```

Vim basics:

| Key | Action |
|---|---|
| `i` | Enter insert mode (type text) |
| `Esc` | Leave insert mode |
| `:wq` | Save and quit |
| `:q!` | Quit without saving |

### Step 3: Copy files to the web root

```bash
sudo cp -r ~/php-app/src/* /usr/share/nginx/html/
sudo chown -R apache:apache /usr/share/nginx/html
```

- `cp -r` copies files and folders recursively.
- `/usr/share/nginx/html/` is the default web root.
- `chown -R user:group` sets the owner recursively. Nginx and PHP-FPM both run as `apache` on AlmaLinux, so they must be able to read these files.

Verify:

```bash
ls -l /usr/share/nginx/html
```

### Step 4: Check PHP-FPM configuration

```bash
sudo grep '^listen' /etc/php-fpm.d/www.conf
sudo grep -E '^(user|group)' /etc/php-fpm.d/www.conf
```

Expected output:

```
listen = /run/php-fpm/www.sock
user = apache
group = apache
```

If those match, no changes needed. The `listen` path is the socket Nginx will use. The `user` is who PHP-FPM runs as.

- `grep 'pattern' file` searches for lines matching the pattern.
- `^` means start of line.
- `-E` enables extended regex so `(user|group)` means "user or group".

### Step 5: Create the Nginx site config

```bash
sudo vim /etc/nginx/conf.d/php-app.conf
```

Paste:

```nginx
server {
    listen 80 default_server;
    server_name _;
    root /usr/share/nginx/html;
    index index.php index.html;

    location / {
        try_files $uri $uri/ /index.php;
    }

    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_pass unix:/run/php-fpm/www.sock;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
}
```

Key directives:

| Directive | Meaning |
|---|---|
| `listen 80 default_server` | Accept port 80 as the default site |
| `server_name _` | Match any hostname |
| `root` | Where files live on disk |
| `index` | Default file for directory URLs |
| `location /` | Handles normal URLs |
| `try_files` | Try file, then folder, then fall back to `index.php` |
| `location ~ \.php$` | Match PHP requests |
| `fastcgi_pass` | Socket PHP-FPM listens on. Must match `listen` from Step 4 |
| `SCRIPT_FILENAME` | Tells PHP-FPM which file to execute |

### Step 6: Remove the default server block

```bash
sudo vim /etc/nginx/nginx.conf
```

AlmaLinux ships `nginx.conf` with a built-in `server { ... }` block that also listens on port 80. It conflicts with the one in Step 5. Remove it.

Inside vim:

1. Put the cursor on the line `server {`.
2. Press `V` to enter visual line mode.
3. Press the down arrow until the closing `}` is highlighted.
4. Press `d` to delete.
5. Press `:wq` to save and quit.

The file should keep the `http { ... }` section and the line:

```nginx
include /etc/nginx/conf.d/*.conf;
```

But no `server { ... }` block inside `http`.

### Step 7: SELinux and firewall

```bash
sudo setsebool -P httpd_can_network_connect 1
sudo restorecon -Rv /usr/share/nginx/html
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload
```

- `setsebool`: toggles a SELinux permission. `httpd_can_network_connect` lets Nginx connect to the PHP-FPM socket. `-P` makes the setting persist across reboots.
- `restorecon`: resets file security labels to defaults. `-R` recursive, `-v` verbose.
- `firewall-cmd`: opens port 80. `--permanent` keeps the change after reboot. `--reload` applies it.

Without these, requests will fail with `502 Bad Gateway` or connection refused.

### Step 8: Start services and test

```bash
sudo systemctl enable --now nginx
sudo systemctl enable --now php-fpm

sudo nginx -t
sudo systemctl reload nginx
sudo systemctl restart php-fpm

curl -I http://localhost/
```

- `systemctl enable --now`: enables the service at boot and starts it now.
- `nginx -t`: checks config syntax. Always run after editing configs.
- `reload`: applies config without dropping connections.
- `restart`: full stop and start. Use for PHP-FPM after editing its config.
- `curl -I`: fetches headers only.

Expected output:

```
HTTP/1.1 200 OK
Server: nginx/...
Content-Type: text/html; charset=UTF-8
```

Open `http://<vm-ip>/` in a browser. You should see the purple card with the app details.

### Step 9: Verify and troubleshoot

If something fails, check logs:

```bash
sudo tail -20 /var/log/nginx/error.log
sudo tail -20 /var/log/php-fpm/www-error.log
sudo ausearch -m avc -ts recent
```

- `tail -20 file` shows the last 20 lines of a file.
- `ausearch -m avc` shows SELinux denials.

See Section 6 for the common error table.

### Step 10: Daily usage

Once the server is running, adding or editing PHP files is instant. Nginx serves them immediately, no restart needed.

```bash
sudo cp ~/php-app/src/* /usr/share/nginx/html/
sudo chown -R apache:apache /usr/share/nginx/html
```

To add another site later:

1. Create `/etc/nginx/conf.d/newsite.conf` with a new `server { ... }` block.
2. Use a different `server_name` or port.
3. Run:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

## 5. Command Cheat Sheet

### Files and permissions

| Command | Purpose |
|---|---|
| `mkdir -p path` | Create folder and parents |
| `cp -r src dst` | Copy files and folders |
| `chown -R user:group path` | Change owner recursively |
| `ls -l path` | List files with details |
| `chmod 2775 dir` | Set permissions. `2` sets the SGID bit so new files inherit the group |

### Services

| Command | Purpose |
|---|---|
| `systemctl enable --now svc` | Enable at boot and start now |
| `systemctl status svc` | Show status |
| `systemctl restart svc` | Full restart |
| `systemctl reload nginx` | Apply config without dropping connections |

### Nginx

| Command | Purpose |
|---|---|
| `nginx -t` | Check config syntax |
| `nginx -v` | Show version |

### Logs

| Command | Purpose |
|---|---|
| `tail -20 file` | Show last 20 lines |
| `tail -f file` | Follow log live |

### SELinux

| Command | Purpose |
|---|---|
| `getenforce` | Check SELinux mode |
| `setsebool -P httpd_can_network_connect 1` | Allow Nginx to connect to PHP-FPM |
| `restorecon -Rv path` | Fix file labels |
| `ausearch -m avc -ts recent` | Show recent denials |

### Firewall

| Command | Purpose |
|---|---|
| `firewall-cmd --permanent --add-service=http` | Open port 80 |
| `firewall-cmd --reload` | Apply changes |
| `firewall-cmd --list-all` | Show current rules |

### Testing

| Command | Purpose |
|---|---|
| `curl -I url` | Fetch headers only |
| `curl url` | Fetch full response |

---

## 6. Troubleshooting

| Error | Cause | Fix |
|---|---|---|
| `502 Bad Gateway` | Nginx cannot reach PHP-FPM | `sudo systemctl restart php-fpm`; verify socket path matches Step 4 |
| `403 Forbidden` | Wrong owner on files | `sudo chown -R apache:apache /usr/share/nginx/html` |
| `404 Not Found` | Wrong root path | Check `root` in site config |
| PHP shown as plain text | Nginx not forwarding to PHP | Check the `location ~ \.php$` block |
| `duplicate default server` | Two `default_server` blocks | Remove the one in `nginx.conf` |
| `duplicate listen options` | Two servers on port 80 | Same fix as above |
| Connection refused | Port 80 closed | `sudo firewall-cmd --permanent --add-service=http && sudo firewall-cmd --reload` |
| SELinux denial in `ausearch` | Socket connection blocked | `sudo setsebool -P httpd_can_network_connect 1` |

---

## 7. Full recreation sequence

Copy this block to rebuild from scratch on a new AlmaLinux VM:

```bash
# Install
sudo dnf install -y epel-release
sudo dnf install -y nginx php php-fpm php-cli

# App files
mkdir -p ~/php-app/src
# create index.php and style.css inside src/ using the content above

# Deploy
sudo cp -r ~/php-app/src/* /usr/share/nginx/html/
sudo chown -R apache:apache /usr/share/nginx/html

# Site config
# create /etc/nginx/conf.d/php-app.conf with the server block from Step 5
# remove the default server block from /etc/nginx/nginx.conf

# Security
sudo setsebool -P httpd_can_network_connect 1
sudo restorecon -Rv /usr/share/nginx/html
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload

# Start
sudo systemctl enable --now nginx php-fpm
sudo nginx -t
sudo systemctl reload nginx
sudo systemctl restart php-fpm

# Test
curl -I http://localhost/
```

If `nginx -t` prints `syntax is ok` and `curl` returns `200 OK`, the setup is complete.

---

## 8. Next step: migration to Docker

Every native step maps to a Dockerfile line. Once the native setup works, the Docker version becomes obvious.

| Native step | Dockerfile equivalent |
|---|---|
| `dnf install nginx php php-fpm` | `RUN dnf install -y nginx php php-fpm` |
| Create `conf.d/php-app.conf` | `COPY nginx.conf /etc/nginx/conf.d/default.conf` |
| Set `user = apache` in php-fpm | `RUN sed -i 's/^user = .*/user = nginx/' /etc/php-fpm.d/www.conf` |
| `chown -R apache:apache` | `RUN chown -R nginx:nginx /var/www/html` |
| SELinux and firewall | Not needed. Containers isolate processes. |
| Start nginx and php-fpm | `CMD ["/usr/bin/supervisord", "-c", "/etc/supervisord.conf"]` |

The container bundles both services into one image using Supervisor. That is why the Dockerfile has a `supervisord.conf` file with two program sections.

---

## 9. Quick recap

1. Installed Nginx, PHP, and PHP-FPM with `dnf`.
2. Created `index.php` and `style.css` in `~/php-app/src`.
3. Copied them to `/usr/share/nginx/html` and set owner to `apache:apache`.
4. Confirmed PHP-FPM listens on `/run/php-fpm/www.sock` as `apache`.
5. Added a site config in `/etc/nginx/conf.d/php-app.conf`.
6. Removed the default `server { ... }` block from `nginx.conf`.
7. Opened SELinux (`httpd_can_network_connect`) and the firewall (port 80).
8. Started both services and verified with `nginx -t` and `curl -I`.
9. Browser shows the app. Editing files needs no restart.
10. The Docker migration is a direct mapping of the same steps into a Dockerfile.
