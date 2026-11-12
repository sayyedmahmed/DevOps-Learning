# CRUD Contacts App: Complete Build and Reproduction

**Apache + PHP (app VM) → MaxScale → 3-node MariaDB Galera**

## 1. Actual Architecture

```
 Browser
   │  HTTP :80
   ▼
 App VM 10.10.10.180            MaxScale 10.10.10.190            Galera cluster
 Apache + PHP  ──SQL :4006──►   MaxScale listener  ──:3306──►   10.10.10.201
 (code only)                    (readwritesplit)                10.10.10.202
                                                                10.10.10.203
```

- The **app VM** holds only code. No database server.
- **MaxScale** is a proxy. It has no SQL engine, cannot store data, and does not run `mariadb.service`.
- The **Galera nodes** are the real MariaDB servers. All data lives there, synchronously replicated.
- The app connects to **`10.10.10.190:4006`**, not `:3306`.

---

## 2. Environment

| Role | Hostname | IP | Software | Notes |
|---|---|---|---|---|
| App server | `app-server` | `10.10.10.180` | Apache `httpd`, PHP 8.3, `php-fpm`, `php-mysqlnd`, `php-mbstring`, `mariadb` **client** | Serves the app |
| Proxy | `maxscale` | `10.10.10.190` | `maxscale-24.02.9` | Listener on **4006** |
| Galera node 1 | `mariadb-1` | `10.10.10.201` | MariaDB server | |
| Galera node 2 | `mariadb-2` | `10.10.10.202` | MariaDB server | |
| Galera node 3 | `mariadb-3` | `10.10.10.203` | MariaDB server | |

| Setting | Value |
|---|---|
| Database | `app_data` |
| App DB user | `appuser` |
| App DB password | `123` (lab only, see §13) |
| MaxScale monitor user | `max` / `123` (already configured) |
| MaxScale listener port | **4006** |
| MaxScale service | `Read-Write-Service` |
| MaxScale router | `readwritesplit` |
| MaxScale monitor | `galeramon` |

---

## 3. Deviations from the Original Tutorial (read this first)

The original guide assumed **`10.10.10.190` is a MariaDB server**. It is not. Every Part 1 instruction that touched `10.10.10.190` had to be re-targeted.

| Original guide step | What we actually did | Why |
|---|---|---|
| Install `mariadb-server` on `10.10.10.190` | **Skipped.** MaxScale is installed there instead. | `10.10.10.190` is MaxScale, not MariaDB. |
| `bind-address = 10.10.10.190` on `10.10.10.190` | **Skipped.** MaxScale listens per its own `maxscale.cnf`. | MaxScale is the listener; MariaDB is behind it. |
| Firewall on `10.10.10.190` for port **3306** | Firewall on `10.10.10.190` for port **4006** | App connects to MaxScale on 4006. |
| Run `schema.sql` on `10.10.10.190` | Run it on **a Galera node** (e.g. `.201`) | MaxScale cannot execute SQL. |
| `'appuser'@'10.10.10.180'` | `'appuser'@'10.10.10.%'` | The DB sees **MaxScale's** address (`.190`), not the app's (`.180`). |
| `db_port => 3306` in `config.php` | `db_port => 4006` | MaxScale listener port. |
| Test `mysql -h 10.10.10.190 -P 3306` | Test `mariadb -h 10.10.10.190 -P 4006` | Same reason. |
| (none) | `maxctrl reload service Read-Write-Service` after user creation | MaxScale caches the user list. |

Everything else in the original guide (the app code, the Apache vhost, permissions, SELinux boolean, CSRF/prepared-statement design) is **unchanged**.

> **`mariadb` package naming gotcha:** on AlmaLinux/RHEL, `dnf install mariadb` installs the **client only**. The server package is `mariadb-server`. The app VM only needs the client, which is why `systemctl start mariadb` on `10.10.10.180` correctly fails with *"Unit mariadb.service not found"*. That is expected, not an error to fix.

---

## 4. Verify the Architecture Before You Start

Before touching anything, confirm which host is which. This avoids repeating the whole "start mariadb on the wrong box" detour.

**On `10.10.10.190`:**

```bash
hostname
rpm -q mariadb-server maxscale
ss -tlnp | grep -E '3306|4006'
cat /etc/maxscale.cnf
```

Expected on this environment:

```
maxscale
package mariadb-server is not installed
maxscale-24.02.9-1.rhel.10.x86_64
LISTEN 0 ... 0.0.0.0:4006 ... maxscale
```

If `mariadb-server` is **not** installed and `maxscale` **is**, you are in the MaxScale architecture and this document applies. If `mariadb-server` is installed and MaxScale is not, use the original single-VM guide instead.

**On `10.10.10.180`:**

```bash
hostname
rpm -q httpd php php-fpm php-mysqlnd php-mbstring
rpm -q mariadb-server || echo "no server here (correct)"
```

---

## 5. Part A: App VM Setup (`10.10.10.180`)

### 5.1 Install packages

```bash
sudo dnf install httpd php php-fpm php-mysqlnd php-mbstring mariadb -y
```

| Package | Purpose |
|---|---|
| `httpd` | Apache web server |
| `php`, `php-fpm` | PHP runtime |
| `php-mysqlnd` | PDO MySQL driver. Required for `new PDO(...)` |
| `php-mbstring` | `mb_strlen`, `mb_substr`, `mb_strtoupper` used in the app |
| `mariadb` | **Client only**. Provides the `mariadb`/`mysql` CLI for testing |

### 5.2 Free port 80 if another server owns it

The original guide assumed a clean VM. In this environment, `nginx` was already bound to `:80` and returned `502 Bad Gateway` when Apache tried to start.

```bash
sudo ss -tlnp | grep ':80'
```

If something other than `httpd` holds port 80:

```bash
# If nginx is not needed on this VM:
sudo systemctl stop nginx
sudo systemctl disable nginx

# Then start Apache:
sudo systemctl enable --now php-fpm httpd
```

If nginx **is** intentional, do not install `httpd`. Instead point nginx at PHP-FPM (see §12, *"nginx is answering instead of Apache"*).

### 5.3 Start services and verify

```bash
sudo systemctl enable --now php-fpm httpd
sudo systemctl status httpd php-fpm --no-pager
php -v
php -m | grep -E 'pdo_mysql|mbstring'      # both must appear
```

### 5.4 Firewall

```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload
sudo firewall-cmd --list-services             # http must be listed
```

### 5.5 SELinux: required for outbound DB connections

AlmaLinux blocks Apache/PHP from making outbound network connections by default. Without this boolean the app fails with `Connection refused` even when the network is fine.

```bash
getenforce                                    # Enforcing (keep it on)
getsebool httpd_can_network_connect_db        # must be "on"
sudo setsebool -P httpd_can_network_connect_db 1
```

Verify:

```bash
getsebool httpd_can_network_connect_db
# httpd_can_network_connect_db --> on
```

> The boolean is named `httpd_*` but it governs PHP-FPM's outbound connections regardless of whether Apache or nginx serves the PHP.

### 5.6 Test MaxScale reachability from the app VM

```bash
# TCP reachability
nc -zv 10.10.10.190 4006

# Protocol and credentials (user not created yet, expect Access denied, not Connection refused)
mariadb -h 10.10.10.190 -P 4006 -u appuser -p123 -e "SELECT 1;"
```

- `Connection refused` means MaxScale is not listening on 4006, or the firewall on `.190` blocks 4006. Fix on the MaxScale host (§6.1).
- `Access denied` means the network path is correct. Finish §7 (create the user), then retry.

---

## 6. Part B: MaxScale Firewall and Cache (`10.10.10.190`)

### 6.1 Firewall: allow the app VM to port 4006

The original guide's rule opened **3306** on `.190`. That port is unused on a MaxScale host. Replace it (or add) a rule for **4006**:

```bash
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.10.10.180" port port="4006" protocol="tcp" accept'
sudo firewall-cmd --reload
sudo firewall-cmd --list-rich-rules
```

Confirm MaxScale is actually listening on 4006:

```bash
ss -tlnp | grep 4006
# LISTEN 0 ... 0.0.0.0:4006 ... maxscale
```

If nothing is listening, check `maxscale.cnf` and:

```bash
sudo systemctl status maxscale --no-pager
sudo maxctrl list services
sudo maxctrl list servers        # all three Galera nodes should be Running
```

### 6.2 Reload MaxScale's user cache (after §7)

MaxScale caches the backend user list; a brand-new user is invisible until the cache refreshes. Refresh is rate-limited on auth failure, so force it:

```bash
maxctrl reload service Read-Write-Service
```

---

## 7. Part C: Database and App User (on a Galera Node)

MaxScale cannot run `CREATE USER`, `GRANT`, or `CREATE DATABASE`. It is a proxy, not a server. Run everything below **on one Galera node**, e.g. `10.10.10.201`.

### 7.1 Why the host pattern is `10.10.10.%`, not the app's IP

The Galera nodes see the connection coming from **MaxScale's address** (`10.10.10.190`), not from the app (`10.10.10.180`). A user defined as `'appuser'@'10.10.10.180'` will therefore **never match**. You get `Access denied for user 'appuser'@'10.10.10.190'`.

| Host pattern | Works? | Reason |
|---|---|---|
| `'appuser'@'10.10.10.180'` | No | DB never sees `.180` |
| `'appuser'@'10.10.10.190'` | Yes | Exact MaxScale address |
| `'appuser'@'10.10.10.%'` | Yes | Covers MaxScale and the subnet; stricter than `%` |
| `'appuser'@'%'` | Yes | Lab-only; too broad |

This build uses **`10.10.10.%`**.

### 7.2 The SQL

On `10.10.10.201` as root (`mariadb -u root -p`):

```sql
-- Database
CREATE DATABASE IF NOT EXISTS app_data
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;

USE app_data;

-- Table
CREATE TABLE IF NOT EXISTS contacts (
    id         INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
    name       VARCHAR(100) NOT NULL,
    email      VARCHAR(150) NOT NULL,
    phone      VARCHAR(20)  NULL,
    city       VARCHAR(80)  NULL,
    created_at TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY uq_contacts_email (email)
) ENGINE=InnoDB;

-- Seed data
INSERT IGNORE INTO contacts (name, email, phone, city) VALUES
  ('Ayesha Khan', 'ayesha@example.com', '+92 300 1234567', 'Islamabad'),
  ('Bilal Ahmed', 'bilal@example.com',  '+92 321 7654321', 'Lahore'),
  ('Sara Malik',  'sara@example.com',   '+92 333 1112223', 'Karachi');

-- Application user: DB sees MaxScale's address, hence 10.10.10.%
CREATE USER IF NOT EXISTS 'appuser'@'10.10.10.%' IDENTIFIED BY '123';

-- Least privilege for the CRUD app
GRANT SELECT, INSERT, UPDATE, DELETE ON app_data.* TO 'appuser'@'10.10.10.%';

FLUSH PRIVILEGES;
```

> **Galera note:** DDL (`CREATE DATABASE`, `CREATE TABLE`, `CREATE USER`, `GRANT`) replicates synchronously to all three nodes. Run it **once** on one node; do not repeat on the others. Do **not** use `INSERT INTO mysql.user`. Direct DML against `mysql.*` does not replicate.

### 7.3 Reload MaxScale

On `10.10.10.190`:

```bash
maxctrl reload service Read-Write-Service
```

### 7.4 Verify

**On the Galera node:**

```sql
SELECT user, host FROM mysql.user WHERE user='appuser';
-- appuser | 10.10.10.%

SHOW GRANTS FOR 'appuser'@'10.10.10.%';
-- GRANT USAGE ON *.* ...
-- GRANT SELECT, INSERT, UPDATE, DELETE ON `app_data`.* ...
```

On `.202` and `.203`, run `SELECT user, host FROM mysql.user WHERE user='appuser';`. The row must be present on all three. If not, the cluster is unhealthy (`SHOW STATUS LIKE 'wsrep_cluster_size';` should be `3`).

**From the MaxScale host (proves the host pattern matches):**

```bash
mariadb --skip-ssl -h 10.10.10.201 -u appuser -p123 -e "SHOW DATABASES;"
# app_data must appear
```

**From the app VM (proves the whole path):**

```bash
mariadb -h 10.10.10.190 -P 4006 -u appuser -p123 -e "SHOW DATABASES;"
# app_data must appear
```

If the direct test (§7.4, MaxScale host) works but the proxy test (app VM) fails with `Access denied`, the MaxScale cache is stale. Re-run `maxctrl reload service Read-Write-Service`.

---

## 8. Part D: Application Files (`10.10.10.180`)

### 8.1 Layout

```
/var/www/crudapp/
  ├── config.php            ← database settings (outside web root, never downloadable)
  └── public/
        └── index.php       ← the entire app (Apache serves this folder)
```

```bash
sudo mkdir -p /var/www/crudapp/public
```

### 8.2 `config.php` with port 4006

```bash
sudo vim /var/www/crudapp/config.php
```

```php
<?php
// Database settings. Lives OUTSIDE public/, so browsers can never fetch it.
return [
    'db_host' => '10.10.10.190',   // MaxScale host
    'db_port' => 4006,             // MaxScale listener port (NOT 3306)
    'db_name' => 'app_data',
    'db_user' => 'appuser',
    'db_pass' => '123',            // lab only, must match the Galera user
];
```

Lock it down:

```bash
sudo chown root:apache /var/www/crudapp/config.php
sudo chmod 640 /var/www/crudapp/config.php
```

> This is the **only functional change** to the app vs the original guide. `index.php` is copied **verbatim**. PDO does not care whether it talks to MariaDB directly or through MaxScale; the wire protocol (`MariaDBClient`) is the same.

### 8.3 `public/index.php`

Copy verbatim from the original guide (or from `crudapp.zip`):

```bash
# If you have the zip:
scp -r crudapp user@10.10.10.180:/tmp/
sudo cp /tmp/crudapp/public/index.php /var/www/crudapp/public/
```

No edits are required. The app already uses `PDO::ATTR_EMULATE_PREPARES => false`, which MaxScale's `MariaDBClient` protocol supports.

### 8.4 Permissions and SELinux labels

```bash
sudo chown -R root:apache /var/www/crudapp
sudo find /var/www/crudapp -type d -exec chmod 750 {} \;
sudo find /var/www/crudapp -type f -exec chmod 640 {} \;
sudo restorecon -Rv /var/www/crudapp
```

---

## 9. Part E: Apache Virtual Host

```bash
sudo vim /etc/httpd/conf.d/crudapp.conf
```

```apache
<VirtualHost *:80>
    ServerName 10.10.10.180
    DocumentRoot /var/www/crudapp/public
    DirectoryIndex index.php

    <Directory /var/www/crudapp/public>
        Options -Indexes
        AllowOverride None
        Require all granted
    </Directory>

    ErrorLog  /var/log/httpd/crudapp.error.log
    CustomLog /var/log/httpd/crudapp.access.log combined
</VirtualHost>
```

Validate and reload:

```bash
sudo httpd -t                     # Syntax OK
sudo systemctl reload httpd
sudo httpd -S                     # confirms *:80 -> crudapp.conf
```

---

## 10. Part F: End-to-End Test

### 10.1 CLI

```bash
# 1. Path app VM -> MaxScale -> Galera
mariadb -h 10.10.10.190 -P 4006 -u appuser -p123 app_data \
        -e "SELECT COUNT(*) FROM contacts;"
# -> 3

# 2. HTTP
curl -sI http://10.10.10.180 | head -n 1
# -> HTTP/1.1 200 OK

# 3. Headers should show Apache + PHP, not nginx
curl -sI http://10.10.10.180 | grep -Ei 'server|x-powered-by'
# Server: Apache/2.4.x (AlmaLinux)
# X-Powered-By: PHP/8.3.x
```

### 10.2 Browser: exercise every CRUD operation

Open `http://10.10.10.180`. The footer shows the **app VM hostname** and the **database hostname reported via `@@hostname`**. Through MaxScale that will be whichever Galera node served the read, proving the proxy is in the path.

| Action | SQL executed | Where it lands |
|---|---|---|
| **+ Add contact** | `INSERT` | MaxScale routes to the **primary** Galera node |
| Search box | `SELECT ... LIKE` | MaxScale routes to a **replica** |
| **Edit** then Save | `UPDATE` | Primary |
| **Delete** then confirm | `DELETE` | Primary |
| Duplicate email | `INSERT` fails `23000` | App shows *"This email is already used"* |
| Empty Name | (no SQL) | Validation message, nothing saved |

### 10.3 Prove data is on the cluster

On **another** Galera node (not the one you inserted through, though `readwritesplit` may have picked any primary):

```bash
mariadb -u root -p -e "SELECT id, name, email, city FROM app_data.contacts ORDER BY id DESC;"
```

Add a contact in the browser, re-run, see the new row. Galera has replicated it.

### 10.4 Prove read/write split

Through MaxScale:

```bash
mariadb -h 10.10.10.190 -P 4006 -u appuser -p123
```

```sql
USE app_data;
CREATE TABLE IF NOT EXISTS notes (id INT AUTO_INCREMENT PRIMARY KEY, txt VARCHAR(50));
INSERT INTO notes (txt) VALUES ('through maxscale to galera');
SELECT * FROM notes;
```

`CREATE`/`INSERT` go to the primary, `SELECT` to a replica, Galera keeps all three consistent.

---

## 11. Database Verification and Showcase

This section gives you hands-on commands to inspect `app_data`, prove that writes reach the Galera cluster, and confirm MaxScale is routing correctly.

### 11.1 Show the database structure

On any Galera node (e.g. `10.10.10.201`):

```bash
mariadb -u root -p
```

```sql
USE app_data;
SHOW TABLES;
DESCRIBE contacts;
SHOW CREATE TABLE contacts\G
SHOW INDEX FROM contacts;
```

`SHOW CREATE TABLE` confirms the engine is InnoDB (required for Galera), the collation is `utf8mb4`, and the unique key on `email` is in place.

### 11.2 Show current data

Same node:

```sql
SELECT id, name, email, phone, city, created_at FROM app_data.contacts ORDER BY id;
SELECT COUNT(*) AS total_rows FROM app_data.contacts;
```

### 11.3 Show which Galera node you are talking to

Still on the same node:

```sql
SELECT @@hostname AS node_hostname,
       @@wsrep_node_name AS galera_node,
       @@wsrep_cluster_size AS cluster_size;
```

Expected: the current node's hostname, its Galera name (`node1`, `node2`, `node3`), and `cluster_size = 3`.

### 11.4 Show data through MaxScale as the app user

From the app VM (`10.10.10.180`):

```bash
mariadb -h 10.10.10.190 -P 4006 -u appuser -p123 app_data
```

```sql
SELECT id, name, email, city FROM contacts ORDER BY id;
SELECT COUNT(*) AS total_rows FROM contacts;
SELECT @@hostname AS read_node;
```

Run `SELECT @@hostname;` three or four times. Because `readwritesplit` sends plain SELECTs to replicas, the hostname should change between the replica nodes (`mariadb-2` and `mariadb-3`).

### 11.5 Insert through MaxScale and verify on all three nodes

Through MaxScale (from the app VM):

```sql
INSERT INTO contacts (name, email, phone, city)
VALUES ('Test User', 'test.user@example.com', '+92 300 0000000', 'Multan');
```

Now check **all three** Galera nodes to prove synchronous replication.

On `10.10.10.201`:

```bash
mariadb -u root -p -e "SELECT id, name, email, city FROM app_data.contacts WHERE email='test.user@example.com';"
```

On `10.10.10.202`:

```bash
mariadb -u root -p -e "SELECT id, name, email, city FROM app_data.contacts WHERE email='test.user@example.com';"
```

On `10.10.10.203`:

```bash
mariadb -u root -p -e "SELECT id, name, email, city FROM app_data.contacts WHERE email='test.user@example.com';"
```

All three must return the row. That is synchronous replication in action.

### 11.6 Show which node is the write target

Writes through `readwritesplit` go to whichever node `galeramon` designated as `Master`. To see which one that is right now, on the MaxScale host:

```bash
maxctrl list servers
```

The node showing `Master, Synced, Running` is the write target. All `INSERT`/`UPDATE`/`DELETE` queries through MaxScale land there.

### 11.7 Manual CRUD walkthrough

Through MaxScale:

```sql
-- CREATE
INSERT INTO contacts (name, email, phone, city)
VALUES ('Manual User', 'manual.user@example.com', '+92 333 4445556', 'Quetta');

-- READ
SELECT id, name, email, city FROM contacts WHERE email = 'manual.user@example.com';

-- UPDATE
UPDATE contacts SET city = 'Peshawar' WHERE email = 'manual.user@example.com';
SELECT city FROM contacts WHERE email = 'manual.user@example.com';

-- DELETE
DELETE FROM contacts WHERE email = 'manual.user@example.com';
SELECT * FROM contacts WHERE email = 'manual.user@example.com';   -- empty result
```

After each step, re-run the same query on a **different** Galera node to see the change replicated.

### 11.8 Compare data across all three nodes

Run on each node:

```bash
mariadb -u root -p -e "SELECT COUNT(*) AS total, MAX(id) AS last_id FROM app_data.contacts;"
```

The `total` and `last_id` values must be identical on all three. If they differ, check cluster health:

```sql
SHOW STATUS LIKE 'wsrep_cluster_size';           -- must be 3
SHOW STATUS LIKE 'wsrep_local_state_comment';    -- must be Synced on all three
SHOW STATUS LIKE 'wsrep_ready';                  -- must be ON
```

### 11.9 Cluster health from each node

On each Galera node:

```sql
SHOW STATUS LIKE 'wsrep_cluster_size';
SHOW STATUS LIKE 'wsrep_cluster_status';
SHOW STATUS LIKE 'wsrep_ready';
SHOW STATUS LIKE 'wsrep_local_state_comment';
SHOW STATUS LIKE 'wsrep_incoming_addresses';
```

Expected:
- `wsrep_cluster_size = 3`
- `wsrep_cluster_status = Primary`
- `wsrep_ready = ON`
- `wsrep_local_state_comment = Synced`
- `wsrep_incoming_addresses` lists all three node IPs

### 11.10 Single end-to-end showcase script

A single block to run from the app VM through MaxScale. It inserts, reads, shows which node served the read, then cleans up.

```bash
mariadb -h 10.10.10.190 -P 4006 -u appuser -p123 app_data <<'EOF'
SELECT '--- before ---' AS step;
SELECT COUNT(*) AS rows_before FROM contacts;

SELECT '--- insert ---' AS step;
INSERT INTO contacts (name, email, city) VALUES
  ('Showcase User', 'showcase@example.com', 'Islamabad');

SELECT '--- after ---' AS step;
SELECT COUNT(*) AS rows_after FROM contacts;
SELECT id, name, email, city FROM contacts WHERE email='showcase@example.com';

SELECT '--- node that served the read ---' AS step;
SELECT @@hostname AS read_node;

SELECT '--- cleanup ---' AS step;
DELETE FROM contacts WHERE email='showcase@example.com';
SELECT COUNT(*) AS rows_after_cleanup FROM contacts;
EOF
```

Then on each Galera node, confirm the row count matches and the deleted row is gone everywhere:

```bash
mariadb -u root -p -e "SELECT COUNT(*) AS total FROM app_data.contacts;"
```

### 11.11 Verify the table engine is Galera-compatible

```sql
SELECT TABLE_NAME, ENGINE, TABLE_COLLATION
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'app_data';
```

`ENGINE` must be `InnoDB`. Galera does not replicate non-transactional engines such as MyISAM.

---

## 12. Troubleshooting: What We Actually Hit

### 12.1 `httpd` failed to start; `curl` returned `502 Bad Gateway` from **nginx**

**Symptom**

```
Job for httpd.service failed because the control process exited with error code.
HTTP/1.1 502 Bad Gateway
Server: nginx/1.26.3
```

**Cause:** nginx was already bound to `:80`; Apache could not bind; nginx's own upstream was down, so it returned 502.

**Fix:**

```bash
sudo ss -tlnp | grep ':80'          # identify the owner
sudo systemctl stop nginx && sudo systemctl disable nginx   # if not needed
sudo systemctl start httpd
sudo journalctl -xeu httpd.service --no-pager | tail -n 30 # if it still fails
```

If nginx is intentional, skip Apache entirely and proxy to PHP-FPM:

```nginx
# /etc/nginx/conf.d/crudapp.conf
server {
    listen 80;
    server_name 10.10.10.180;
    root /var/www/crudapp/public;
    index index.php;
    location / { try_files $uri $uri/ /index.php?$query_string; }
    location ~ \.php$ {
        fastcgi_pass unix:/run/php-fpm/www.sock;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
}
```

Then skip §9 and the `httpd` commands. Keep the SELinux boolean (§5.5).

### 12.2 `Failed to start mariadb.service: Unit mariadb.service not found` on `10.10.10.180`

**Expected.** The app VM has only the `mariadb` **client**. There is no server to start. Do nothing here.

### 12.3 `SQLSTATE[HY000] [2002] Connection refused` in `www-error.log`

**Cause:** app was connecting to `10.10.10.190:3306`, but MaxScale listens on **4006**.

**Fix:** set `'db_port' => 4006` in `config.php`, then:

```bash
sudo systemctl restart php-fpm
```

### 12.4 `Connection refused` even with port 4006

Check in order:

1. **MaxScale listening:** `ss -tlnp | grep 4006` on `.190`.
2. **MaxScale firewall:** rich rule for source `10.10.10.180` to port `4006`.
3. **SELinux on app VM:** `getsebool httpd_can_network_connect_db` must be `on`.
4. **App VM outbound:** `nc -zv 10.10.10.190 4006`.

### 12.5 `Access denied for user 'appuser'@'10.10.10.190'`

**Cause:** host pattern mismatch. The DB sees MaxScale's `.190`, not the app's `.180`.

**Fix:** on a Galera node:

```sql
DROP USER 'appuser'@'10.10.10.180';     -- if it exists with the wrong host
CREATE USER IF NOT EXISTS 'appuser'@'10.10.10.%' IDENTIFIED BY '123';
GRANT SELECT, INSERT, UPDATE, DELETE ON app_data.* TO 'appuser'@'10.10.10.%';
FLUSH PRIVILEGES;
```

Then on MaxScale:

```bash
maxctrl reload service Read-Write-Service
```

### 12.6 `Access denied` on the first attempt, then works on retry

MaxScale's user cache was stale. Force a refresh:

```bash
maxctrl reload service Read-Write-Service
```

### 12.7 `SHOW DATABASES` through MaxScale does not list `app_data`

Grant scope is wrong. It must be `ON app_data.*`, not `ON *.*`:

```sql
SHOW GRANTS FOR 'appuser'@'10.10.10.%';
```

### 12.8 User exists on `.201` but not `.202` or `.203`

Galera did not replicate the DDL. Check cluster health on each node:

```sql
SHOW STATUS LIKE 'wsrep_cluster_size';   -- must be 3
SHOW STATUS LIKE 'wsrep_ready';          -- must be ON
```

Also confirm `CREATE USER` was used, **not** `INSERT INTO mysql.user`.

### 12.9 App page shows "Something went wrong"

The friendly page is intentional. The real error is logged:

```bash
sudo tail -n 30 /var/log/php-fpm/www-error.log
sudo tail -n 30 /var/log/httpd/crudapp.error.log
sudo ausearch -m avc -ts recent           # SELinux denials
```

### 12.10 `could not find driver` or `Call to undefined function mb_strlen()`

```bash
sudo dnf install php-mysqlnd php-mbstring -y
sudo systemctl restart php-fpm httpd
```

---

## 13. Security Notes

### 13.1 What this build already does

- Prepared statements (no SQL injection) via `PDO::ATTR_EMULATE_PREPARES => false`
- Escaped output via `e()` (no XSS)
- CSRF tokens on every state-changing form
- Server-side input validation
- POST for changes, GET for viewing
- Redirect-after-POST (no duplicate inserts on refresh)
- `config.php` outside the web root, `640 root:apache`
- Firewall: only the app VM reaches MaxScale on 4006
- SELinux left enforcing, with a single explicit boolean
- DB user scoped to `app_data.*`, host-restricted to `10.10.10.%`
- Friendly errors to users, full errors to logs

### 13.2 What it does **not** do, fix before production

| Gap | Fix |
|---|---|
| No login | Add PHP session login (`password_hash`/`password_verify`) or Apache `AuthType Basic` |
| No HTTPS | Add a certificate (Let's Encrypt or self-signed) |
| Plaintext app to MaxScale traffic | Enable TLS on the MaxScale listener |
| Plaintext MaxScale to Galera traffic | Enable Galera `wsrep` TLS, or keep nodes on a private network |
| `GRANT ALL` in any copy of the doc | Narrow to `SELECT, INSERT, UPDATE, DELETE` (already used in §7.2) |
| Weak password `123` | Long random password; store in a secret manager; rotate |
| No backups | Schedule `mariadb-dump` on a Galera node and test restore |

Backup example (run on a Galera node):

```bash
mysqldump --single-transaction app_data > /root/app_data_$(date +%F).sql
```

---

## 14. Reproduction Checklist

```
 [ ] Confirm architecture: 10.10.10.190 = MaxScale, .201/.202/.203 = Galera
 [ ] App VM: dnf install httpd php php-fpm php-mysqlnd php-mbstring mariadb
 [ ] App VM: free port 80 (stop/disable nginx if present)
 [ ] App VM: systemctl enable --now php-fpm httpd
 [ ] App VM: firewall-cmd --add-service=http
 [ ] App VM: setsebool -P httpd_can_network_connect_db 1
 [ ] MaxScale: firewall rich rule 10.10.10.180 -> 4006
 [ ] Galera node: CREATE DATABASE app_data
 [ ] Galera node: CREATE TABLE contacts + seed rows
 [ ] Galera node: CREATE USER 'appuser'@'10.10.10.%'
 [ ] Galera node: GRANT SELECT,INSERT,UPDATE,DELETE ON app_data.*
 [ ] Galera node: verify user replicated to .202 and .203
 [ ] MaxScale: maxctrl reload service Read-Write-Service
 [ ] App VM: config.php db_port = 4006
 [ ] App VM: copy index.php verbatim
 [ ] App VM: chown/chmod + restorecon
 [ ] App VM: /etc/httpd/conf.d/crudapp.conf
 [ ] App VM: httpd -t && systemctl reload httpd
 [ ] App VM: mariadb -h 10.10.10.190 -P 4006 -u appuser -p123 app_data -e "SELECT COUNT(*)"
 [ ] App VM: curl -sI http://10.10.10.180 -> 200 OK
 [ ] Browser: create/read/update/delete works
 [ ] Galera node: new row visible on a *different* node
 [ ] Section 11 walkthrough: insert through MaxScale, verify on all three nodes
```

---

## 15. Cheat Sheet

```
 NETWORK
   Browser --:80--> App VM 10.10.10.180 (Apache+PHP)
                        |
                        +--:4006--> MaxScale 10.10.10.190 --:3306--> Galera .201/.202/.203

 FILES
   App VM
     /var/www/crudapp/config.php              db settings (db_port = 4006)
     /var/www/crudapp/public/index.php        the app (unchanged)
     /etc/httpd/conf.d/crudapp.conf           vhost
     /var/log/httpd/crudapp.error.log
     /var/log/php-fpm/www-error.log           real DB errors appear here

   MaxScale
     /etc/maxscale.cnf                        servers, service, listener (4006)

   Galera node
     (SQL state lives here)
```

| Task | Where | Command |
|---|---|---|
| Test Apache config | App VM | `sudo httpd -t` |
| Reload Apache | App VM | `sudo systemctl reload httpd` |
| Restart PHP | App VM | `sudo systemctl restart php-fpm` |
| SELinux check | App VM | `getsebool httpd_can_network_connect_db` |
| Test path to MaxScale | App VM | `mariadb -h 10.10.10.190 -P 4006 -u appuser -p123 app_data` |
| See real app error | App VM | `sudo tail -f /var/log/php-fpm/www-error.log` |
| MaxScale health | MaxScale | `maxctrl list servers && maxctrl list services` |
| Reload MaxScale users | MaxScale | `maxctrl reload service Read-Write-Service` |
| See stored data | Galera node | `mariadb -e "SELECT * FROM app_data.contacts;"` |
| Compare all three nodes | Each Galera node | `mariadb -e "SELECT COUNT(*) FROM app_data.contacts;"` |
| Backup | Galera node | `mysqldump --single-transaction app_data > backup.sql` |

**Golden troubleshooting order:**

```
 1. Is MaxScale listening?          ss -tlnp | grep 4006            (MaxScale host)
 2. Can the app VM reach it?        nc -zv 10.10.10.190 4006        (app VM)
 3. Does auth work directly?        mariadb -h .201 -u appuser -p   (MaxScale host)
 4. Does auth work via proxy?       mariadb -h .190 -P 4006 ...     (app VM)
 5. Does the app log a real error?  /var/log/php-fpm/www-error.log  (app VM)
```

---

## 16. Key Takeaways

1. **`10.10.10.190` is MaxScale, not MariaDB.** The original guide's Part 1 does not apply to it.
2. **The app connects to port `4006`**, not `3306`. That is the MaxScale listener, not the database.
3. **The DB sees MaxScale's IP**, so the app user's host pattern must be `10.10.10.%` (or `.190`), never the app VM's `.180`.
4. **Run DDL on a Galera node**, once. Galera replicates it. Never on MaxScale.
5. **Reload MaxScale's user cache** (`maxctrl reload service Read-Write-Service`) after any user or grant change.
6. **SELinux `httpd_can_network_connect_db`** is mandatory or every DB connection fails with `Connection refused`.
7. **`mariadb` on RHEL/Alma is the client**. The server package is `mariadb-server`. `mariadb.service not found` on the app VM is correct.
8. **`index.php` is unchanged**. Only `config.php`'s `db_port` differs from the original guide.
9. **Section 11 is your proof.** Insert through MaxScale, then run the same query on all three Galera nodes. If all three return the row, the whole stack from app to proxy to cluster is working.
