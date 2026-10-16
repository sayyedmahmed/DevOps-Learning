# Complete Guide: Installing Oracle MySQL 8.0.46 on AlmaLinux 10.2

## 1. Objective
Install Oracle MySQL Community Server 8.0.46 using the generic binary archive. 

*   **Installation Directory:** `/usr/local/mysql-8.0.46-linux-glibc2.28-x86_64`
*   **Database Directory:** `/var/lib/mysql`
*   **Socket:** `/var/lib/mysql/mysql.sock`
*   **Port:** `3306`
*   **Root Password:** `root` (for learning lab only)

---

## 2. Remove Conflicting MariaDB Services
AlmaLinux includes MariaDB by default. MariaDB and Oracle MySQL use the same default port (3306) and similar socket paths.

Stop the MariaDB service:
```bash
systemctl stop mariadb
```
> **Why we do this:** We must stop the service to release port 3306 and unlock the data files in `/var/lib/mysql` so they don't conflict with the new MySQL installation.

Remove the MariaDB server packages:
```bash
dnf remove -y mariadb-server mariadb-server-utils
```
> **Why we do this:** This removes the MariaDB server binaries. We remove the server, but leaving client libraries behind is fine if other applications need them.

Verify the removal:
```bash
rpm -qa | grep -Ei 'mariadb-server|mysql-community-server'
```
> **Why we do this:** This queries the RPM database to ensure no conflicting server packages are left installed on the OS.

**Important:** Existing MariaDB database files in `/var/lib/mysql` are preserved during package removal. We will handle this data manually in Step 7.

---

## 3. Install System Dependencies
Install the required system packages:
```bash
dnf install -y libaio numactl-libs xz tar curl
```
> **Why we do this:** 
> *   `libaio`: The Asynchronous I/O library is strictly required by MySQL's InnoDB storage engine for efficient disk operations.
> *   `numactl-libs`: Helps MySQL allocate memory efficiently across multiple CPU nodes (NUMA architecture).
> *   `xz`, `tar`, `curl`: Utilities required to download and extract the compressed binary archive.

---

## 4. Download MySQL 8.0.46
Move to the standard directory for third-party software:
```bash
cd /usr/local
```
> **Why we do this:** `/usr/local` is the standard Linux filesystem hierarchy location for software compiled or installed manually by the system administrator, keeping it separate from OS-managed `/usr` files.

Download the official MySQL generic binary archive:
```bash
curl -fL -o mysql-8.0.46-linux-glibc2.28-x86_64.tar.xz \
https://dev.mysql.com/get/Downloads/MySQL-8.0/mysql-8.0.46-linux-glibc2.28-x86_64.tar.xz
```
> **Why we do this:** `curl -fL` ensures we fail silently on HTTP errors (`-f`) and follow redirects (`-L`). This downloads the exact, verified version directly from Oracle.

Verify that the archive exists and check its size:
```bash
ls -lh /usr/local/mysql-8.0.46-linux-glibc2.28-x86_64.tar.xz
```
> **Why we do this:** The downloaded archive should be approximately 852 MB. Checking the size ensures the download completed fully and isn't a corrupted partial file.

---

## 5. Extract the Archive
Extract the compressed tarball:
```bash
tar -xJf mysql-8.0.46-linux-glibc2.28-x86_64.tar.xz
```
> **Why we do this:** `-x` extracts, `-J` handles `.xz` compression, and `-f` specifies the file. This creates the actual directory structure containing the MySQL binaries.

Verify the extracted directory:
```bash
ls -ld /usr/local/mysql-8.0.46-linux-glibc2.28-x86_64
```
> **Why we do this:** Confirms the directory was successfully created and shows its permissions.

Check the MySQL version to verify binary integrity:
```bash
/usr/local/mysql-8.0.46-linux-glibc2.28-x86_64/bin/mysqld --version
```
> **Why we do this:** Running the binary directly ensures it is executable and not corrupted. Expected output: `MySQL Community Server - GPL Version 8.0.46`.

---

## 6. Create the MySQL User and Group
Create the dedicated group:
```bash
groupadd mysql
```
> **Why we do this:** Creates a specific group for database files, allowing us to manage permissions cleanly.

Create the dedicated system user:
```bash
useradd -r -u 27 -g 27 -s /sbin/nologin mysql
```
> **Why we do this:** MySQL should **never** run as the Linux `root` user. 
> *   `-r`: Creates a system account (no home directory, no aging).
> *   `-g mysql`: Assigns it to the mysql group.
> *   `-s /sbin/nologin`: Prevents anyone from logging into the server via SSH as the "mysql" user. If the DB is hacked, the attacker is trapped.

---

## 7. Prepare the Database and Log Directories
Create the required directories:
```bash
mkdir -p /var/lib/mysql
mkdir -p /var/log/mysql
mkdir -p /run/mysqld
```
> **Why we do this:** 
> *   `/var/lib/mysql`: Stores the actual database files (InnoDB data, logs).
> *   `/var/log/mysql`: Stores the MySQL error log for troubleshooting.
> *   `/run/mysqld`: A temporary filesystem (tmpfs) directory to store the `mysqld.pid` file, which systemd uses to track the process.

Set ownership for all directories:
```bash
chown mysql:mysql /var/lib/mysql
chown mysql:mysql /var/log/mysql
chown mysql:mysql /run/mysqld
```
> **Why we do this:** The MySQL daemon runs as the `mysql` user. If these directories are owned by `root`, the daemon will crash on startup because it won't have permission to write data or logs.

Set strict permissions:
```bash
chmod 750 /var/lib/mysql
chmod 750 /var/log/mysql
chmod 750 /run/mysqld
```
> **Why we do this:** `750` means the owner (`mysql`) has full read/write/execute, the group (`mysql`) can read/execute, and "others" have zero access. This prevents other Linux users from reading raw database files.

Restore the default SELinux context:
```bash
restorecon -v /var/lib/mysql
```
> **Why we do this:** AlmaLinux uses SELinux. If we just `chown` a directory, SELinux might still block MySQL from writing to it because the security context tag is wrong. `restorecon` fixes the SELinux tags.

### Handling Old Data (Crucial Step)
> **Why initialization fails if skipped:** MySQL requires a **completely empty** data directory to initialize. The original `/var/lib/mysql` contained MariaDB files (like `aria_log`, InnoDB files, and a `wordpress` database directory). 
> **Action taken:** We preserved the old data by renaming it (e.g., `mv /var/lib/mysql /var/lib/mysql-old-20261013`) and created a fresh, empty `/var/lib/mysql`. Do not delete the backup until you confirm you don't need the old data.

---

## 8. Configure MySQL
Create the configuration file `/etc/my.cnf`:
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
> **Why we do this:** 
> *   `[mysqld]`: Tells the server where its binaries, data, and logs are.
> *   `[client]`: Tells the command-line tool where to find the server.
> *   **The Socket Path:** By default, the server might put the socket in `/var/lib/mysql`, while the client looks in `/tmp`. Explicitly defining `socket=` in both sections ensures they match, preventing the dreaded "Can't connect to local MySQL server through socket" error.

---

## 9. Initialize the Database
Run the initialization command:
```bash
/usr/local/mysql-8.0.46-linux-glibc2.28-x86_64/bin/mysqld \
  --initialize-insecure \
  --user=mysql \
  --basedir=/usr/local/mysql-8.0.46-linux-glibc2.28-x86_64 \
  --datadir=/var/lib/mysql
```
> **Why we do this:** This creates the internal system tables (the `mysql` schema) required for the database to function. 
> *   `--initialize-insecure`: Creates the `root` account **without** a password. This allows us to log in immediately to set our own password, rather than hunting for a random temporary password in the error logs.
> *   *Warning:* This must only be run on a fresh, empty data directory. Running it on an existing database will corrupt it.

---

## 10. Create a Systemd Service
Create the service file `/etc/systemd/system/mysqld.service`:
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
> **Why we do this:** Because we used a generic binary, the OS doesn't know how to start it automatically. This file tells `systemd` how to manage it.
> *   `User=mysql`: Ensures the process drops root privileges and runs safely as the mysql user.
> *   `LimitNOFILE=65535`: Databases open thousands of files. This increases the OS limit for open file descriptors so MySQL doesn't crash under heavy load.

Reload systemd to read the new file:
```bash
systemctl daemon-reload
```
> **Why we do this:** Tells the systemd manager to re-read all unit files and detect the new `mysqld.service`.

Enable and start MySQL:
```bash
systemctl enable --now mysqld
```
> **Why we do this:** `enable` sets it to start automatically on server reboot. `--now` starts it immediately in the current session.

Check the service status:
```bash
systemctl status mysqld --no-pager
```
> **Why we do this:** Verifies the service started successfully. The expected state is `Active: active (running)`.

---

## 11. Set the MySQL Root Password & Enable Direct Access

Add the MySQL binaries to your system PATH:
```bash
echo 'export PATH="/usr/local/mysql-8.0.46-linux-glibc2.28-x86_64/bin:$PATH"' >> /root/.bashrc
source /root/.bashrc
```
> **Why we do this:** Typing `/usr/local/.../bin/mysql` every time is inefficient. This adds the directory to your environment variables so you can just type `mysql`. `source` applies it to your current terminal session immediately.

Connect to MySQL (no password required yet):
```bash
mysql -u root
```
> **Why we do this:** Because we used `--initialize-insecure` in Step 9, the root account is currently blank.

At the `mysql>` prompt, set the password:
```sql
ALTER USER 'root'@'localhost' IDENTIFIED BY 'root';
```
> **Why we do this:** Secures the database by assigning a password to the root account. (Note: 'root' is for learning only; use strong passwords in production).

Reload the privilege tables:
```sql
FLUSH PRIVILEGES;
```
> **Why we do this:** Forces the MySQL server to re-read the grant tables in memory, ensuring the new password takes effect immediately.

Verify the version inside the prompt:
```sql
SELECT VERSION();
```
> **Why we do this:** Confirms you are connected to the correct Oracle MySQL instance (Expected: `8.0.46`).

Exit the client:
```sql
exit;
```

---

## 12. Test the Password and Database Functionality

Test the password login:
```bash
mysql -u root -p
```
> **Why we do this:** When prompted, enter `root`. This verifies that the password change was successful and authentication is working.

> **Side Note: Using the Absolute Path**
> If you ever need to use the absolute path (e.g., in a bash script where the PATH isn't loaded, or if you are troubleshooting a broken PATH environment), you can bypass the environment variables and call the binary directly:
> `/usr/local/mysql-8.0.46-linux-glibc2.28-x86_64/bin/mysql -u root -p`

Test actual database operations (run these inside the `mysql>` prompt):
```sql
CREATE DATABASE test_lab_db;
SHOW DATABASES;
DROP DATABASE test_lab_db;
EXIT;
```
> **Why we do this:** Logging in only proves authentication works. Creating and dropping a database proves that the InnoDB storage engine, file permissions, and SQL parser are fully functional and can write to the disk.

---

## 13. Useful Verification and Troubleshooting Commands

Check the service status:
```bash
systemctl status mysqld --no-pager
```
> **Why:** The first step in troubleshooting is always checking if the OS thinks the service is running.

Check the MySQL version from the CLI:
```bash
mysql --version
```
> **Why:** Verifies that your PATH is configured correctly and points to the right binary.

Check the listening port:
```bash
ss -lntp | grep ':3306'
```
> **Why:** `ss` shows socket statistics. This confirms MySQL is actually listening on the network port 3306 and accepting connections.

Check the error log:
```bash
tail -50 /var/log/mysql/error.log
```
> **Why:** If `systemctl status` says the service failed, or if you can't connect, this log contains the exact reason (e.g., permission denied, out of memory, corrupt InnoDB log).

---

## 14. Important Security Notes

*   **Password Strength:** The password `root` is acceptable only for an isolated learning lab. Use a strong, complex password for production.
*   **Data Backups:** Keep the old MariaDB and MySQL data backups (like `/var/lib/mysql-old-20261013`) until you have confirmed that you do not need their contents.
*   **No Re-initialization:** Do not run database initialization commands (`--initialize`) again on an existing database. It will overwrite system tables and corrupt your data.
*   **Manual Upgrades:** The generic binary installation does not automatically configure OS integration or future upgrades. You must manually download and replace the binary to update MySQL.
*   **Support Lifecycle:** MySQL 8.0 is an older release series. Check Oracle's support lifecycle before using it for a production system.

---

## Final Result

Oracle MySQL Community Server **8.0.46** is successfully installed under `/usr/local`, configured to use `/var/lib/mysql`, managed by systemd, and accessible through the direct `mysql` client command. The root password is set to `root` for the learning lab.

---

## Install MySQL Community Server 8.0.46 Using DNF

### Objective
Install exactly MySQL Community Server **8.0.46** using DNF on AlmaLinux.

### Steps Followed

**Step 1: Find the Oracle MySQL repository setup package**

Visit the [MySQL Yum Repository Downloads](https://dev.mysql.com/downloads/repo/yum/) page and select the repository setup RPM for the appropriate operating system version.

For AlmaLinux 9, we selected:

`mysql84-community-release-el9-4.noarch.rpm`

**Step 2: Install the repository setup package**

Copy the download link from Oracle's website and install it directly using DNF:

```bash
dnf install -y https://dev.mysql.com/get/mysql84-community-release-el9-4.noarch.rpm
```

This installs the repository configuration package, not MySQL Server itself.

**Step 3: Check available MySQL repositories**

```bash
dnf repolist --all | grep -i mysql
```

**Step 4: Enable the MySQL 8.0 repository**

```bash
dnf config-manager --enable mysql80-community
```

Verify that it is enabled:

```bash
dnf repolist --enabled | grep mysql80
```

**Step 5: Check available server versions**

```bash
dnf --refresh --showduplicates list mysql-community-server
```

Look for version `8.0.46` before installing.

**Step 6: Install the exact version**

If version `8.0.46` is available, install it using DNF:

```bash
dnf install -y mysql-community-server-8.0.46
```

**Step 7: Verify the installation**

```bash
mysqld --version
rpm -q mysql-community-server
```

### Issue Encountered on AlmaLinux 10.2

The MySQL 8.0 repository returned HTTP `404` when DNF attempted to retrieve metadata from:

`https://repo.mysql.com/yum/mysql-8.0-community/el/10/x86_64/`

This indicates that the repository does not provide the requested EL10 metadata. Therefore, the MySQL 8.0.46 installation could not proceed through this repository on AlmaLinux 10.2.

### Conclusion

The repository configuration package was installed successfully, and the `mysql80-community` repository was enabled. However, DNF could not retrieve its metadata on AlmaLinux 10.2.

**For future attempts:** Use an AlmaLinux 9 VM, configure the MySQL 8.0 repository, verify that version `8.0.46` is available, and then install it through DNF. Do not install EL9 RPM packages on AlmaLinux 10. Always verify package availability before installation.
