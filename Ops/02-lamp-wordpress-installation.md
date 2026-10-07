# 02 - LAMP Stack & WordPress Installation

> **Series:** 2 of 4 | Previous: [01 - Hyper-V VM Setup](01-hyperv-vm-setup.md) | Next: [03 - SSL & HTTPS](03-ssl-https-configuration.md)

## Overview

This guide installs the **LAMP stack** and deploys **WordPress** on an AlmaLinux 10 VM for a simple WordPress project.

| Component | Package | Role |
|---|---|---|
| **L**inux | AlmaLinux 10 | Operating system |
| **A**pache | `httpd` | Web server |
| **M**ariaDB | `mariadb-server` | Database for WordPress content |
| **P**HP | `php` + required extensions | Runs WordPress code |

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

> The salts secure login cookies and sessions. Do not leave the default placeholder values.

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

---

## References

For more documentation, I followed these links:

- DigitalOcean: [How To Install Linux, Apache, MySQL, PHP LAMP Stack on CentOS 7](https://www.digitalocean.com/community/tutorials/how-to-install-linux-apache-mysql-php-lamp-stack-on-centos-7)
- DigitalOcean: [How To Install WordPress on CentOS 7](https://www.digitalocean.com/community/tutorials/how-to-install-wordpress-on-centos-7)

> These references are written for CentOS 7. On AlmaLinux 10, use `dnf`, `mariadb-server`, and keep SELinux enabled.

**Next:** [03 - SSL & HTTPS Configuration](03-ssl-https-configuration.md)
