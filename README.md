# DevOps Learning

## Documentation Index

### Basics: Linux

- [Users & Groups](Basics/01-users-and-groups.md)
- [File Viewing Commands](Basics/02-file-viewing-commands.md)
- [File Operations: Redirection, Counting, Disk Usage](Basics/03-redirection-wc-du.md)
- [Linux File Permissions: Fundamentals to Technical Reference](Basics/04-file-permissions-overview.md)

### Concepts

- [NFS (Linux) Concepts and MySQL Data Directory Case Study](Concepts/nfs-concepts-mysql-case-study.md)
- [Pacemaker + Corosync High Availability Cluster for MySQL on AlmaLinux and Hyper-V](Concepts/pacemaker-corosync-mysql-high-availability.md)

### Implementation

#### Infrastructure

- [Hyper-V VM Setup (AlmaLinux 10)](Implementation/infrastructure/01-hyperv-vm-setup.md)
- [Simple Pacemaker Cluster](Implementation/infrastructure/02-pacemaker-cluster.md)
- [Simple NFS Server Setup](Implementation/infrastructure/03-nfs-server-setup.md)

#### Web Servers

- [LAMP Stack & WordPress Installation](Implementation/web-servers/01-lamp-wordpress-installation.md)
- [SSL Certificate & HTTPS Configuration](Implementation/web-servers/02-ssl-https-configuration.md)
- [Hosting Multiple Sites](Implementation/web-servers/03-multiple-sites-vhosts.md)
- [Nginx: Install and Deploy a Simple Web Page](Implementation/web-servers/04-nginx-web-page-deployment.md)
- [Enterprise Nginx Infrastructure and Proxy Configuration Reference](Implementation/web-servers/05-nginx-implementation-operations.md)
- [Nginx Next Steps: Virtual Hosts, HTTPS, Proxies and Load Balancing](Implementation/web-servers/06-nginx-vhosts-https-proxy-load-balancing.md)
- [Apache HTTP Server: Implementation and Operations Reference](Implementation/web-servers/07-httpd-implementation-operations.md)
- [Apache httpd Fundamentals (and How It Differs from Nginx)](Implementation/web-servers/08-httpd-fundamentals-and-nginx-comparison.md)

#### MySQL

- [Installing Oracle MySQL 8.0.46 on AlmaLinux 10.2](Implementation/mysql/01-mysql-8.0.46-installation-almalinux-10.md)
- [MySQL Administration Notes](Implementation/mysql/02-mysql-administration-notes.md)
- [MySQL Master-Slave Replication on AlmaLinux](Implementation/mysql/03-mysql-master-slave-replication.md)
- [MySQL Tuning and Configuration (Production, No-Restart Edition)](Implementation/mysql/04-mysql-tuning-and-configuration.md)

#### MySQL High Availability

- [MySQL on Shared NFS Storage Without Pacemaker](Implementation/mysql-high-availability/01-mysql-nfs-shared-storage-without-pacemaker.md)
- [MySQL NFS Shared Storage: Single-Node Setup and Verification](Implementation/mysql-high-availability/02-nfs-shared-storage-troubleshooting.md)
- [MySQL High Availability (NFS + Pacemaker) on Hyper-V](Implementation/mysql-high-availability/03-mysql-ha-nfs-pacemaker.md)
- [MySQL HA: Quick Reference & Test Commands](Implementation/mysql-high-availability/04-mysql-ha-quick-reference.md)
- [Failover Recovery & Automatic Cleanup](Implementation/mysql-high-availability/05-failover-troubleshooting.md)

#### MariaDB, MaxScale and Galera

- [MariaDB MaxScale Fundamentals](Implementation/mariadb/01-maxscale-fundamentals.md)
- [MaxScale + Single MariaDB Node](Implementation/mariadb/02-maxscale-single-node.md)
- [MaxScale: Adding Replica Nodes (Node 2, then Node 3)](Implementation/mariadb/03-maxscale-adding-replica-nodes.md)
- [MaxScale Step by Step with MariaDB: From One Database to Primary, Replica and Automatic Failover](Implementation/mariadb/04-maxscale-primary-replica-failover.md)
- [MariaDB Galera Cluster Implementation](Implementation/mariadb/05-galera-cluster-three-node.md)
- [MaxScale with a Galera Cluster](Implementation/mariadb/06-maxscale-galera-configuration.md)
- [CRUD Contacts App: Complete Build and Reproduction](Implementation/mariadb/07-crud-app-maxscale-galera.md)
- [Application Code (Apache + PHP CRUD Contacts App)](Implementation/mariadb/08-crud-app-application-code.md)
- [High Availability: MaxScale with Keepalived (VIP)](Implementation/mariadb/09-maxscale-keepalived-vip.md)
- [Galera Cluster Recovery and Restart Procedure](Implementation/mariadb/10-galera-cluster-recovery-procedures.md)
