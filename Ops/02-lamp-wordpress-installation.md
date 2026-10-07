# 02 - LAMP Stack & WordPress Installation

> **Series:** 2 of 4 | Previous: [01 - Hyper-V VM Setup](01-hyperv-vm-setup.md) | Next: [03 - SSL & HTTPS](03-ssl-https-configuration.md)

## Overview

This guide installs the **LAMP stack** and deploys **WordPress** on the VM from Part 1.

| Component | Package | Role |
|---|---|---|
| **L**inux | AlmaLinux 10 | Operating system |
| **A**pache | `httpd` | Web server |
| **M**ariaDB | `mariadb-server` | Database for WordPress content |
| **P**HP | `php` + extensions | Runs WordPress code |

**Prerequisite:** VM reachable at `10.10.10.101` with working internet (see Part 1). All commands run as `root`.

---

## Step 1 - Install Base Tools

```bash
dnf install tar gzip wget rsync openssl net-tools vim -y
```

A minimal install omits common utilities:

| Tool | Used for |
|---|---|
| `tar`, `gzip` | Extracting the WordPress archive |
| `wget` | Downloading files |
| `rsync` | Copying files with progress |
| `openssl` | Certificates (Part 3) |
| `net-tools` | `netstat`/`ifconfig` for diagnostics |
| `vim` | Text editor (`vi`) |

## Step 2 - Install Apache, MariaDB and PHP

```bash
dnf install httpd mariadb-server php php-mysqlnd php-gd php-xml php-mbstring php-json php-zip php-curl -y
```

| PHP extension | Why WordPress needs it |
|---|---|
| `php-mysqlnd` | Connects PHP to MariaDB |
| `php-gd` | Image processing (thumbnails, resizing) |
| `php-xml` | XML parsing (feeds, sitemaps, plugins) |
| `php-mbstring` | Multibyte/UTF-8 text handling |
| `php-json` | JSON handling (REST API, block editor) |
| `php-zip` | Plugin/theme installs and updates |
| `php-curl` | Outbound HTTP requests (updates, APIs) |

> **Note:** On newer PHP versions `php-json` is built into the core; if `dnf` reports it as missing or already included, that is fine.

## Step 3 - Enable and Start Services

```bash
systemctl enable --now httpd
systemctl enable --now mariadb
```

`enable --now` does two things at once: **starts** the service immediately and **enables** it to start on every boot.

**Verify:**
```bash
systemctl status httpd mariadb --no-pager
```

## Step 4 - Secure MariaDB

```bash
mysql_secure_installation
```

Recommended answers:

| Prompt | Answer | Reason |
|---|---|---|
| Switch to unix_socket authentication | `n` | Keep standard password login for the root DB user |
| Change root password | `n` | Only if none was set; otherwise set one |
| Remove anonymous users | `y` | Anonymous accounts allow unauthenticated access |
| Disallow root login remotely | `y` | Root DB access should be local only (`n` acceptable for a local-only lab VM) |
| Remove test database | `y` | Test DB is accessible by anyone by default |
| Reload privilege tables | `y` | Applies all changes immediately |

## Step 5 - Create the WordPress Database

```bash
mysql -u root -p
```
```sql
CREATE DATABASE wordpress;
CREATE USER 'wpuser'@'localhost' IDENTIFIED BY 'StrongPass123!';
GRANT ALL PRIVILEGES ON wordpress.* TO 'wpuser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

| Statement | Purpose |
|---|---|
| `CREATE DATABASE` | Creates the empty database for WordPress |
| `CREATE USER ... @'localhost'` | Dedicated user, usable only from this machine |
| `GRANT ... ON wordpress.*` | Privileges limited to **this database only** (least privilege) |
| `FLUSH PRIVILEGES` | Reloads grant tables |

> **Security:** `StrongPass123!` is an **example**. Use a unique, randomly generated password and never commit real credentials to GitHub.

## Step 6 - Download WordPress

```bash
cd /var/www/html
wget https://wordpress.org/latest.tar.gz
tar xzvf latest.tar.gz
rsync -avP wordpress/ /var/www/html/
rm -rf wordpress latest.tar.gz
```

1. Download the latest release into Apache's web root.
2. Extract it (creates a `wordpress/` subfolder).
3. Move the contents up so the site loads at `/` instead of `/wordpress/`.
4. Remove the leftover folder and archive.

## Step 7 - Configure `wp-config.php`

```bash
cp /var/www/html/wp-config-sample.php /var/www/html/wp-config.php
vi /var/www/html/wp-config.php
```

Set the database connection:
```php
define( 'DB_NAME', 'wordpress' );
define( 'DB_USER', 'wpuser' );
define( 'DB_PASSWORD', 'StrongPass123!' );
define( 'DB_HOST', 'localhost' );
```

Generate authentication keys and salts at `https://api.wordpress.org/secret-key/1.1/salt/` and paste them over the placeholder `define(...)` lines.

**Why the keys matter:** they encrypt login cookies and sessions. Default placeholder values make sessions easier to forge.

## Step 8 - Set Permissions

```bash
chown -R apache:apache /var/www/html
chmod -R 755 /var/www/html
chmod 640 /var/www/html/wp-config.php
```

| Command | Effect |
|---|---|
| `chown -R apache:apache` | Apache can read/write files (needed for uploads and updates) |
| `chmod -R 755` | Owner full access; others read/execute |
| `chmod 640 wp-config.php` | Contains DB credentials, so other users get no access |

## Step 9 - SELinux: Allow Apache to Reach MariaDB

```bash
setsebool -P httpd_can_network_connect_db on
```

SELinux blocks Apache from making database connections by default. This sets **one specific boolean** instead of disabling SELinux. `-P` makes it persistent across reboots.

## Step 10 - Open Firewall Ports

```bash
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https
firewall-cmd --reload
```

**Verify:**
```bash
firewall-cmd --list-services
```

## Step 11 - Complete the WordPress Installation

1. Open `http://10.10.10.101` in a browser on the Windows host.
2. Follow the setup wizard: site title, admin username, password, email.

> **Tip:** Avoid the username `admin`. Non-default usernames reduce brute-force exposure.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| "Error establishing a database connection" | Wrong credentials or SELinux | Re-check `wp-config.php`; confirm Step 9 |
| Apache default page appears | WordPress files not in web root | Check `ls /var/www/html` |
| Page won't load from host | Firewall | Step 10; `firewall-cmd --list-services` |
| Can't upload media or install plugins | Ownership/permissions | Re-run Step 8 |
| Check Apache errors | - | `tail -f /var/log/httpd/error_log` |

## Checklist

- [ ] `httpd` and `mariadb` running and enabled
- [ ] MariaDB secured
- [ ] `wordpress` database and `wpuser` created
- [ ] `wp-config.php` configured with real salts
- [ ] Permissions and SELinux boolean set
- [ ] Ports 80/443 open
- [ ] WordPress wizard completed

**Next:** [03 - SSL & HTTPS](03-ssl-https-configuration.md)
