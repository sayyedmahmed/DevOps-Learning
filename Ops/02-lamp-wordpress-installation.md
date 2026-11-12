# 02 - LAMP Stack & WordPress Installation

> **Series:** 2 of 4 | Previous: [01 - Hyper-V VM Setup](01-hyperv-vm-setup.md) | Next: [03 - SSL & HTTPS](03-ssl-https-configuration.md)

## Overview

This guide installs the **LAMP stack** and deploys **WordPress** on an AlmaLinux 10 VM for a simple WordPress project.

| Component | Package | Role |
|---|---|---|
| **L**inux | AlmaLinux 10 | Operating system |
| **A**pache | `httpd` | Web server that delivers WordPress pages |
| **M**ariaDB | `mariadb-server` | Database that stores WordPress posts, users, settings, and content |
| **P**HP | `php` + required extensions | Runs WordPress code |

### What is happening in this guide?

WordPress needs three main things to work:

1. **Apache** receives visitor requests from the browser.
2. **PHP** processes WordPress code.
3. **MariaDB** stores WordPress data such as posts, pages, users, and settings.

This guide installs only the essential packages needed for a simple WordPress site.

**Prerequisite:** VM reachable at `10.10.10.101` with working internet. All commands run as `root`.

---

## Step 1 - Install Download and Extraction Tools

WordPress is downloaded as a compressed archive, so these tools are needed.

```bash
dnf install wget tar gzip rsync -y
```

| Tool | Used for |
|---|---|
| `wget` | Downloading the WordPress archive |
| `tar` | Extracting the archive |
| `gzip` | Handling `.tar.gz` compression |
| `rsync` | Moving extracted WordPress files into the web root |

### Why are we doing this?

A minimal AlmaLinux installation may not include all common utilities. These tools help us download WordPress, extract it safely, and place the files in the correct directory.

---

## Step 2 - Install Apache, MariaDB, and PHP

For a simple WordPress setup, install only the core packages needed to run WordPress and handle media.

```bash
dnf install httpd mariadb-server php php-mysqlnd php-gd -y
```

| Package | Why it is needed |
|---|---|
| `httpd` | Apache web server |
| `mariadb-server` | Database server for WordPress content |
| `php` | PHP runtime for WordPress |
| `php-mysqlnd` | Allows PHP to connect to MariaDB |
| `php-gd` | Allows WordPress to process images and uploads |

### Why these packages?

- **Apache** serves the website to visitors.
- **MariaDB** stores WordPress data.
- **PHP** runs WordPress.
- **php-mysqlnd** is required because WordPress must connect to the database.
- **php-gd** is required because WordPress needs to resize images, create thumbnails, and handle media uploads.

> **Note:** Modern PHP includes JSON support in the core, so a separate `php-json` package is not required here.

---

## Step 3 - Enable and Start Services

Start Apache and MariaDB now, and make them start automatically after reboot.

```bash
systemctl enable --now httpd
systemctl enable --now mariadb
```

Verify both services are running:

```bash
systemctl status httpd mariadb --no-pager
```

You should see `active (running)` for both services.

### Why are we doing this?

Installing a service does not automatically make it run forever.

- `--now` starts the service immediately.
- `enable` makes the service start automatically after reboot.

This is important because Apache and MariaDB must always be running for WordPress to work.

---

## Step 4 - Secure MariaDB

Run the MariaDB security script:

```bash
mysql_secure_installation
```

Use these answers for this lab setup:

| Prompt | Answer | Reason |
|---|---|---|
| Enter current password for root | Press Enter if fresh install | New MariaDB installs may not have a root password yet |
| Switch to unix_socket authentication | `Y` | Secures local root database access |
| Change the root password | `Y` | Set a strong MariaDB root password |
| Remove anonymous users | `Y` | Prevents unauthenticated database access |
| Disallow root login remotely | `Y` | Root database access should stay local |
| Remove test database | `Y` | Removes the default unsafe test database |
| Reload privilege tables | `Y` | Applies changes immediately |

### Why are we doing this?

A fresh MariaDB installation has insecure defaults, such as anonymous users and a test database. This script removes those weak settings and helps protect the database.

### What is unix_socket authentication?

Unix socket authentication allows the Linux `root` user to log into MariaDB as database `root` without sending a password over the network. It is more secure for local administration.

We still set a MariaDB root password as an extra layer of protection.

---

## Step 5 - Create the WordPress Database and User

Log in to MariaDB:

```bash
mysql -u root -p
```

If unix_socket authentication is working and you are logged in as Linux `root`, this may also work:

```bash
mysql -u root
```

Inside the MariaDB prompt, run:

```sql
CREATE DATABASE wordpress;
CREATE USER 'wpuser'@'localhost' IDENTIFIED BY 'StrongPass123!';
GRANT ALL PRIVILEGES ON wordpress.* TO 'wpuser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

> Replace `StrongPass123!` with your own strong password.

| SQL command | Purpose |
|---|---|
| `CREATE DATABASE wordpress;` | Creates the database WordPress will use |
| `CREATE USER 'wpuser'@'localhost' ...` | Creates a dedicated WordPress database user |
| `GRANT ALL PRIVILEGES ON wordpress.* ...` | Gives the user access only to the WordPress database |
| `FLUSH PRIVILEGES;` | Reloads database permissions |
| `EXIT;` | Leaves the MariaDB prompt |

### Why are we doing this?

WordPress needs a database to store its content.

We create a dedicated database user called `wpuser` instead of using MariaDB `root`. This follows the principle of least privilege:

- WordPress can access only its own database.
- If the WordPress credentials are compromised, the attacker does not get full control of the entire database server.

---

## Step 6 - Download WordPress

Move to Apache's web root:

```bash
cd /var/www/html
```

Download the latest WordPress release:

```bash
wget https://wordpress.org/latest.tar.gz
```

Extract the archive:

```bash
tar xzvf latest.tar.gz
```

Move the WordPress files into the main web root:

```bash
rsync -avP wordpress/ /var/www/html/
```

Remove the temporary folder and archive:

```bash
rm -rf wordpress latest.tar.gz
```

Verify the WordPress files are present:

```bash
ls -lh /var/www/html
```

You should see files and folders such as:

```text
index.php
wp-admin
wp-content
wp-includes
wp-config-sample.php
```

### Why are we doing this?

Apache serves files from `/var/www/html`. This directory is called the **web root**.

WordPress downloads as a folder called `wordpress/`. If we leave it there, the site would load at:

```text
http://10.10.10.101/wordpress/
```

We move the files directly into `/var/www/html` so WordPress loads from the main address:

```text
http://10.10.10.101/
```

We also remove the archive and temporary folder to keep the web root clean.

---

## Step 7 - Configure `wp-config.php`

Copy the sample configuration file:

```bash
cp /var/www/html/wp-config-sample.php /var/www/html/wp-config.php
```

Edit the file:

```bash
vi /var/www/html/wp-config.php
```

Set the database connection values:

```php
define( 'DB_NAME', 'wordpress' );
define( 'DB_USER', 'wpuser' );
define( 'DB_PASSWORD', 'StrongPass123!' );
define( 'DB_HOST', 'localhost' );
```

Use the same password created in Step 5.

Generate WordPress salts here:

```text
https://api.wordpress.org/secret-key/1.1/salt/
```

Replace the placeholder salt lines in `wp-config.php` with the generated values.

### Why are we doing this?

`wp-config.php` tells WordPress how to connect to MariaDB.

If these values are wrong, WordPress will show:

```text
Error establishing a database connection
```

### What are salts?

Salts are random secret strings used to secure WordPress login cookies and sessions.

They make it much harder for attackers to forge login sessions.

Never leave the default placeholder values.

---

## Step 8 - Set Ownership and Permissions

Apache must own the WordPress files so WordPress can upload media and update itself.

```bash
chown -R apache:apache /var/www/html
chmod -R 755 /var/www/html
chmod 640 /var/www/html/wp-config.php
```

| Command | Purpose |
|---|---|
| `chown -R apache:apache` | Gives Apache ownership of WordPress files |
| `chmod -R 755` | Sets normal web directory/file access |
| `chmod 640 wp-config.php` | Protects the database password in `wp-config.php` |

### Why are we doing this?

Linux controls file access using ownership and permissions.

Apache runs as a user called `apache`. If WordPress files are owned by `root`, Apache may not be able to write to them.

That causes problems such as:

- Cannot upload images.
- Cannot install plugins.
- Cannot update themes.
- Cannot create WordPress cache folders.

So we give ownership to `apache`.

### What do the permissions mean?

`755` means:

| User type | Permission |
|---|---|
| Owner | Read, write, execute |
| Group | Read, execute |
| Others | Read, execute |

This is normal for web files and folders.

`640` on `wp-config.php` means:

| User type | Permission |
|---|---|
| Owner | Read, write |
| Group | Read |
| Others | No access |

This protects the database password because `wp-config.php` contains sensitive credentials.

---

## Step 9 - Allow Apache to Connect to MariaDB Through SELinux

AlmaLinux uses SELinux. By default, SELinux can block Apache/PHP from connecting to the database.

```bash
setsebool -P httpd_can_network_connect_db on
```

| Part | Meaning |
|---|---|
| `setsebool` | Changes an SELinux boolean |
| `-P` | Makes the change persistent after reboot |
| `httpd_can_network_connect_db` | Allows Apache to connect to a database |

### Why are we doing this?

SELinux is an extra security layer on top of normal Linux permissions.

Even if Apache owns the files and the database password is correct, SELinux may still block Apache from connecting to MariaDB.

Instead of disabling SELinux, we enable only the specific permission WordPress needs.

This keeps the system secure while allowing WordPress to work.

---

## Step 10 - Open Firewall Ports

Allow HTTP and HTTPS traffic:

```bash
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https
firewall-cmd --reload
```

Verify the firewall services:

```bash
firewall-cmd --list-services
```

You should see:

```text
http https
```

### Why are we doing this?

The firewall blocks incoming network traffic by default.

- Port `80` is used for normal HTTP websites.
- Port `443` is used for HTTPS secure websites.

We open HTTPS now because Part 3 will configure SSL.

### Why use `--permanent` and `--reload`?

- `--permanent` saves the rule so it survives reboot.
- `--reload` applies the saved rules to the running firewall.

Without `--reload`, the permanent rule may not take effect immediately.

---

## Step 11 - Complete the WordPress Installation

Open a browser on the Windows host and go to:

```text
http://10.10.10.101
```

Complete the WordPress wizard:

| Field | Recommendation |
|---|---|
| Site Title | Your project name |
| Username | Do not use `admin` |
| Password | Use a strong password |
| Email | Your email address |

After installation, log in at:

```text
http://10.10.10.101/wp-admin
```

### Why are we doing this?

The WordPress wizard creates the admin user and installs the required WordPress tables inside the MariaDB database.

Avoid using `admin` as the username because it is the first username attackers try during brute-force attacks.

---

## Troubleshooting From This Setup

### 1. Browser Shows Raw PHP Code

Symptom:

```text
<?php phpinfo(); ?>
```

appears as plain text instead of a PHP page.

Check whether Apache loaded PHP:

```bash
httpd -M | grep php
```

If there is no `php_module` output, install PHP and restart Apache:

```bash
dnf install php -y
systemctl restart httpd
```

Then refresh the browser page.

### Why does this happen?

Apache may be running, but it does not know how to process PHP files. Restarting Apache after installing PHP loads the PHP module.

---

### 2. WordPress Says "Cannot Select Database"

This means WordPress connected to MariaDB, but the database or user permission is wrong.

Log in to MariaDB:

```bash
mysql -u root -p
```

Check that the database exists:

```sql
SHOW DATABASES;
```

You should see:

```text
wordpress
```

If it is missing, create it:

```sql
CREATE DATABASE wordpress;
```

Check the user:

```sql
SELECT user, host FROM mysql.user WHERE user = 'wpuser';
```

Grant access again:

```sql
GRANT ALL PRIVILEGES ON wordpress.* TO 'wpuser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

Then reload the WordPress installation page.

### Why does this happen?

WordPress can connect to MariaDB, but it cannot access the specific database. This usually means the database name is wrong, the user does not exist, or the user has not been granted permission to use the `wordpress` database.

---

## References

For more documentation, I followed these links:

- DigitalOcean: [How To Install Linux, Apache, MySQL, PHP LAMP Stack on CentOS 7](https://www.digitalocean.com/community/tutorials/how-to-install-linux-apache-mysql-php-lamp-stack-on-centos-7)
- DigitalOcean: [How To Install WordPress on CentOS 7](https://www.digitalocean.com/community/tutorials/how-to-install-wordpress-on-centos-7)

> These references are written for CentOS 7. On AlmaLinux 10, use `dnf`, `mariadb-server`, and keep SELinux enabled.

**Next:** [03 - SSL & HTTPS Configuration](03-ssl-https-configuration.md)
