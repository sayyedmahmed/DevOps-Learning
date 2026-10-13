# Installing Oracle MySQL 8.0.46 on AlmaLinux 10.2

## 1. Objective

Install Oracle MySQL Community Server 8.0.46 on AlmaLinux 10.2, configure it as a systemd service, and set the MySQL `root` password to `root`.

**Installation method:** Official MySQL generic Linux binary archive.

**Installation directory:** `/usr/local/mysql-8.0.46-linux-glibc2.28-x86_64`

**Database directory:** `/var/lib/mysql`

**Socket:** `/var/lib/mysql/mysql.sock`

**Port:** `3306`

## 2. Remove MariaDB

The system originally had MariaDB 10.11 installed. We wanted Oracle MySQL instead.

Stop MariaDB:

```bash
systemctl stop mariadb
```

Remove the MariaDB server packages:

```bash
dnf remove mariadb-server mariadb-server-utils
```

Verify:

```bash
rpm -qa | grep -Ei 'mariadb-server|mysql-community-server'
```

Review the output to confirm the MariaDB server packages are gone.

**Important:** Existing MariaDB database files must be preserved until you know they are no longer needed.

## 3. Install dependencies

Install the required system packages:

```bash
dnf install -y libaio numactl-libs xz tar curl
```

These packages provide supporting libraries and utilities for installation.

## 4. Download MySQL 8.0.46

Move to `/usr/local`:

```bash
cd /usr/local
```

Download the official MySQL generic binary archive:

```bash
curl -fL -o mysql-8.0.46-linux-glibc2.28-x86_64.tar.xz \
https://dev.mysql.com/get/Downloads/MySQL-8.0/mysql-8.0.46-linux-glibc2.28-x86_64.tar.xz
```

Verify that the archive exists:

```bash
ls -lh /usr/local/mysql-8.0.46-linux-glibc2.28-x86_64.tar.xz
```

The downloaded archive was approximately 852 MB.

## 5. Extract the archive

Run:

```bash
cd /usr/local

tar -xJf mysql-8.0.46-linux-glibc2.28-x86_64.tar.xz
```

Verify the extracted directory:

```bash
ls -ld /usr/local/mysql-8.0.46-linux-glibc2.28-x86_64
```

Check the MySQL version:

```bash
/usr/local/mysql-8.0.46-linux-glibc2.28-x86_64/bin/mysqld --version
```

Expected output identifies:

```text
MySQL Community Server - GPL
Version 8.0.46
```

## 6. Create the MySQL user and group

MySQL runs under a dedicated Linux user rather than the root operating-system account.

```bash
groupadd mysql
useradd -r -g mysql -s /sbin/nologin mysql
```

If the user or group already exists, check it rather than creating it again.

## 7. Prepare the database and log directories

```bash
mkdir -p /var/lib/mysql
mkdir -p /var/log/mysql
mkdir -p /run/mysqld
```

Set ownership and permissions:

```bash
chown mysql:mysql /var/lib/mysql
chmod 750 /var/lib/mysql

chown mysql:mysql /var/log/mysql
chmod 750 /var/log/mysql

chown mysql:mysql /run/mysqld
chmod 750 /run/mysqld
```

Restore the default SELinux context on the database directory:

```bash
restorecon -v /var/lib/mysql
```

### Why did initialization initially fail?

The original `/var/lib/mysql` directory contained MariaDB files, including `aria_log` files, InnoDB files, and a `wordpress` database directory. MySQL requires an empty data directory for initialization.

We preserved the old data directory by renaming it and then created a fresh directory for MySQL. During the later clean setup, the active directory was also backed up as `/var/lib/mysql-old-20261013`.

Do not delete these backups unless you have confirmed that their data is no longer needed.

## 8. Configure MySQL

Create `/etc/my.cnf`:

```bash
cat > /etc/my.cnf <<'EOF'
[client]
socket=/var/lib/mysql/mysql.sock

[mysqld]
user=mysql
basedir=/usr/local/mysql-8.0.46-linux-glibc2.28-x86_64
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
pid-file=/run/mysqld/mysqld.pid
log-error=/var/log/mysql/error.log
EOF
```

The client and server must use the same socket path. Initially, the client tried `/tmp/mysql.sock`, while the server used `/var/lib/mysql/mysql.sock`, causing connection errors.

## 9. Initialize the database

For the final clean setup, we used `--initialize-insecure` to create the MySQL system tables without assigning a temporary password.

```bash
/usr/local/mysql-8.0.46-linux-glibc2.28-x86_64/bin/mysqld \
  --initialize-insecure \
  --user=mysql \
  --basedir=/usr/local/mysql-8.0.46-linux-glibc2.28-x86_64 \
  --datadir=/var/lib/mysql
```

This command must only be used for a fresh, empty data directory. It should not be repeated on an initialized database.

Because this option creates the root account without a password, set the password immediately after starting MySQL.

## 10. Create a systemd service

Create `/etc/systemd/system/mysqld.service`:

```bash
cat > /etc/systemd/system/mysqld.service <<'EOF'
[Unit]
Description=Oracle MySQL Community Server 8.0
After=network.target

[Service]
Type=simple
User=mysql
Group=mysql
RuntimeDirectory=mysqld
RuntimeDirectoryMode=0750
ExecStart=/usr/local/mysql-8.0.46-linux-glibc2.28-x86_64/bin/mysqld --defaults-file=/etc/my.cnf
Restart=on-failure
RestartSec=5
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF
```

Reload systemd and start MySQL:

```bash
systemctl daemon-reload
systemctl enable --now mysqld
```

Check the service:

```bash
systemctl status mysqld --no-pager
```

The expected state is:

```text
Active: active (running)
```

If the service fails, inspect the error log:

```bash
tail -50 /var/log/mysql/error.log
```

## 11. Set the MySQL root password

Connect to MySQL using the installed binary:

```bash
/usr/local/mysql-8.0.46-linux-glibc2.28-x86_64/bin/mysql -u root
```

Because we used `--initialize-insecure`, no password is initially required.

At the `mysql>` prompt, run:

```sql
ALTER USER 'root'@'localhost' IDENTIFIED BY 'root';
```

Verify the installed version:

```sql
SELECT VERSION();
```

Expected result:

```text
8.0.46
```

Exit the MySQL client:

```sql
exit;
```

## 12. Test the password

Run:

```bash
/usr/local/mysql-8.0.46-linux-glibc2.28-x86_64/bin/mysql -u root -p
```

When prompted, enter:

```text
root
```

If login succeeds, the MySQL installation and root authentication are working.

## 13. Useful verification commands

Check the service:

```bash
systemctl status mysqld --no-pager
```

Check the version:

```bash
/usr/local/mysql-8.0.46-linux-glibc2.28-x86_64/bin/mysql --version
```

Check the listening port:

```bash
ss -lntp | grep ':3306'
```

Check the error log:

```bash
tail -50 /var/log/mysql/error.log
```

## 14. Important security notes

- The password `root` is acceptable only for an isolated learning lab. Use a strong password for production.
- Keep the old MariaDB and MySQL data backups until you have confirmed that you do not need their contents.
- Do not run database initialization commands again on an existing database.
- The generic binary installation does not automatically configure all operating-system integration, security policies, or future upgrades.
- MySQL 8.0 is an older release series. Check Oracle's support lifecycle before using it for a production system.

## Final result

Oracle MySQL Community Server **8.0.46** is installed under `/usr/local`, configured to use `/var/lib/mysql`, managed by systemd, and accessible through the MySQL client. The root password is `root` for the learning lab.