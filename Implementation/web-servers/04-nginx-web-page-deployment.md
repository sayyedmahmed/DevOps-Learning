# Nginx: Install and Deploy a Simple Web Page

## 1. What is Nginx?

**Nginx** (say "engine-x") is a **web server**. It waits for browsers to ask for a web page, then sends the page back.

```
  YOU (browser)                         SERVER (nginx)
 +--------------+   "give me the     +----------------------+
 | Chrome /     |    page" (HTTP)    |  nginx               |
 | Firefox      | -----------------> |    |                 |
 |              |                    |    v                 |
 |              | <----------------- |  /usr/share/nginx/   |
 +--------------+   sends index.html |  html/index.html     |
                                     +----------------------+
```

Key words:

| Word | Meaning |
|---|---|
| **Web server** | Program that sends web pages to browsers |
| **Document root** | The folder where your web page files live |
| **index.html** | The default page nginx shows |
| **Port 80** | The default door for web traffic (HTTP) |
| **Config file** | Text file that tells nginx how to behave |

> Commands below use `sudo`. Pick **Part A** for RHEL / CentOS / Rocky / Fedora, or **Part B** for Ubuntu / Debian. Not sure which? Run `cat /etc/os-release`.

---

## 2. Step 1: Install Nginx

### Part A: RHEL / CentOS / Rocky / Alma
```bash
sudo dnf install nginx -y
```

### Part B: Ubuntu / Debian
```bash
sudo apt update
sudo apt install nginx -y
```

Check the version to confirm it installed:
```bash
nginx -v
```

---

## 3. Step 2: Start Nginx

```bash
sudo systemctl start nginx          # start now
sudo systemctl enable nginx         # start automatically after reboot
sudo systemctl status nginx         # check it is running (press q to exit)
```

You should see **active (running)** in green.

| Command | What it does |
|---|---|
| `systemctl start nginx` | Start |
| `systemctl stop nginx` | Stop |
| `systemctl restart nginx` | Stop and start again |
| `systemctl reload nginx` | Apply config changes without dropping connections |
| `systemctl enable nginx` | Auto-start on boot |

---

## 4. Step 3: Open the Firewall

The firewall blocks web traffic by default. Open port 80.

### RHEL family (firewalld)
```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload
```

### Ubuntu (ufw)
```bash
sudo ufw allow 'Nginx HTTP'
```
(If ufw is not active, skip this step.)

```
 Internet --> [ FIREWALL: port 80 open? ] --> nginx
                      |
              if closed: page never loads
```

---

## 5. Step 4: Test in the Browser

Find your server's IP:
```bash
hostname -I
```

Or test from the server itself:
```bash
curl http://localhost
```

In a browser, open:
```
http://YOUR_SERVER_IP
```

You should see the **"Welcome to nginx"** page. Nginx is working.

---

## 6. Step 5: Deploy Your Own Simple Web Page

### Where do the files live?

| System | Default web folder |
|---|---|
| RHEL / CentOS / Rocky | `/usr/share/nginx/html` |
| Ubuntu / Debian | `/var/www/html` |

### Edit the page

RHEL family:
```bash
sudo nano /usr/share/nginx/html/index.html
```
Ubuntu:
```bash
sudo nano /var/www/html/index.html
```

Delete everything inside and paste this:

```html
<!DOCTYPE html>
<html>
<head>
  <title>My First Page</title>
  <style>
    body { font-family: Arial, sans-serif; text-align: center; margin-top: 80px; background: #f0f4f8; }
    h1 { color: #2b6cb0; }
  </style>
</head>
<body>
  <h1>Hello from Nginx!</h1>
  <p>My first web page is live.</p>
</body>
</html>
```

Save in nano: `Ctrl + O`, `Enter`, then `Ctrl + X`.

Refresh the browser. Your page appears. No restart is needed, because nginx reads the file each time.

---

## 7. Step 6 (Optional): Use Your Own Folder

Use this when you want your site in its own folder instead of the default one. We use **port 8080** so it does not clash with the default site.

```
 Browser --> :80   --> default nginx page
 Browser --> :8080 --> your own site (/var/www/mysite)
```

### 7.1 Create the folder and page
```bash
sudo mkdir -p /var/www/mysite
sudo nano /var/www/mysite/index.html
```
Paste any HTML (for example the one above), then save.

### 7.2 Create a config file
```bash
sudo nano /etc/nginx/conf.d/mysite.conf
```
Paste:
```nginx
server {
    listen 8080;
    server_name _;

    root /var/www/mysite;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

What each line means:

| Line | Meaning |
|---|---|
| `listen 8080;` | Listen on port 8080 |
| `server_name _;` | Answer for any name or IP |
| `root /var/www/mysite;` | Folder with your files |
| `index index.html;` | Default file to show |
| `try_files ... =404;` | Show the file, or a 404 error if missing |

### 7.3 Test the config and reload
```bash
sudo nginx -t                  # must say "syntax is ok" and "test is successful"
sudo systemctl reload nginx
```

### 7.4 Open port 8080 in the firewall
```bash
# RHEL family
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --reload

# Ubuntu
sudo ufw allow 8080/tcp
```

### 7.5 SELinux (RHEL family only)

RHEL has SELinux, which blocks nginx from unlisted ports and folders. Check with `getenforce` (if it says `Enforcing`, do the following).

```bash
# allow nginx to use port 8080
sudo dnf install policycoreutils-python-utils -y
sudo semanage port -a -t http_port_t -p tcp 8080

# allow nginx to read your folder
sudo semanage fcontext -a -t httpd_sys_content_t "/var/www/mysite(/.*)?"
sudo restorecon -Rv /var/www/mysite
```

### 7.6 Fix file permissions (if needed)
```bash
sudo chmod -R 755 /var/www/mysite
```

Open `http://YOUR_SERVER_IP:8080` in your browser.

---

## 8. Important Files and Folders

```
 /etc/nginx/
   |-- nginx.conf          <- main config
   |-- conf.d/             <- your extra configs (*.conf)
 /var/log/nginx/
   |-- access.log          <- who visited
   |-- error.log           <- what went wrong
 /usr/share/nginx/html/    <- default pages (RHEL)
 /var/www/html/            <- default pages (Ubuntu)
```

---

## 9. Troubleshooting

| Problem | Likely cause and fix |
|---|---|
| Browser cannot connect | Firewall closed (Step 3) or nginx not running (`systemctl status nginx`) |
| `403 Forbidden` | Wrong file permissions (`chmod -R 755`) or SELinux label (see 7.5) |
| `404 Not Found` | File missing, wrong folder name, or wrong `root` path in config |
| `nginx -t` shows an error | Typo in config; read the line number it prints |
| `Address already in use` | Another program uses that port: `sudo ss -tlnp | grep :80` |
| Change not showing | Hard refresh with `Ctrl + Shift + R`, or `sudo systemctl reload nginx` |
| Page works on server but not from your PC | Firewall, or cloud security group not allowing the port |

Look at the logs when stuck:
```bash
sudo tail -n 20 /var/log/nginx/error.log
sudo tail -f /var/log/nginx/access.log      # live view, Ctrl + C to stop
```

---

## 10. Quick Summary

```
 1. Install      dnf install nginx   (or apt install nginx)
 2. Start        systemctl enable --now nginx
 3. Firewall     open port 80
 4. Test         http://SERVER_IP  -> "Welcome to nginx"
 5. Deploy       edit index.html in the web folder
 6. Own folder   (optional) new folder + conf.d/mysite.conf
                 -> nginx -t -> reload -> SELinux -> firewall
```

**Golden rules**
- After changing a **config file**: run `sudo nginx -t`, then `sudo systemctl reload nginx`.
- After changing an **HTML file**: just refresh the browser.
- If something fails: check `systemctl status nginx` and `/var/log/nginx/error.log`.
