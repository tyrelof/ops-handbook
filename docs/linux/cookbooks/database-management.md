# Cookbook: Database Management

> [!IMPORTANT]
> This guide covers **MySQL 8.0/8.4**, **MariaDB**, and **MongoDB 7.0** for Ubuntu 22.04/24.04 LTS. 
> Key modernizations include **GTID-based replication** and **authentication by default**.

## 1. MySQL & MariaDB: Installation & Security

### Installation
Ubuntu 24.04 defaults to MySQL 8.0.x or MariaDB 10.11.x.
```bash
# For MySQL
sudo apt update
sudo apt install mysql-server -y

# For MariaDB
sudo apt install mariadb-server -y
```

### Security Hardening
Run the security script immediately after installation:
```bash
sudo mysql_secure_installation
```
- **MySQL 8.x Note**: By default, the root user uses the `auth_socket` plugin, allowing passwordless access via `sudo mysql`.

---

## 2. Modern User Management (MySQL 8.0+)

MySQL 8.0+ no longer allows implicit user creation via `GRANT`. You MUST create the user first.

### Creating a User
```sql
CREATE USER 'app_user'@'localhost' IDENTIFIED BY 'StrongPassword!123';
GRANT ALL PRIVILEGES ON app_db.* TO 'app_user'@'localhost';
FLUSH PRIVILEGES;
```

### Remote Access
1.  Update `bind-address` in `/etc/mysql/mysql.conf.d/mysqld.cnf`:
    `bind-address = 0.0.0.0` (or your specific private IP).
2.  Restart: `sudo systemctl restart mysql`.

---

## 3. Replication: Source to Replica (GTID-based)

Modern replication uses **Global Transaction Identifiers (GTIDs)** instead of binary log file offsets. This is much more robust for failover.

### Configuration (Source Server)
Add to `/etc/mysql/mysql.conf.d/mysqld.cnf`:
```text
server-id                = 1
log_bin                  = /var/log/mysql/mysql-bin.log
gtid_mode                = ON
enforce_gtid_consistency = ON
```

### Configuration (Replica Server)
Add to `/etc/mysql/mysql.conf.d/mysqld.cnf`:
```text
server-id                = 2
log_bin                  = /var/log/mysql/mysql-bin.log
gtid_mode                = ON
enforce_gtid_consistency = ON
read_only                = ON
```

### Establishing the Link
1.  **On Source**: Create a replication user.
    ```sql
    CREATE USER 'replica_user'@'%' IDENTIFIED WITH 'caching_sha2_password' BY 'ReplicaPass!123';
    GRANT REPLICATION SLAVE ON *.* TO 'replica_user'@'%';
    ```
2.  **On Replica**: Point to the source.
    ```sql
    CHANGE REPLICATION SOURCE TO
      SOURCE_HOST='SOURCE_IP',
      SOURCE_USER='replica_user',
      SOURCE_PASSWORD='ReplicaPass!123',
      SOURCE_AUTO_POSITION=1;
    START REPLICA;
    ```
3.  **Check Status**: `SHOW REPLICA STATUS\G`

---

## 4. Maintenance & Backups

### Basic Backup (`mysqldump`)
```bash
mysqldump -u root -p --all-databases > full_backup.sql
```

### Modern Backups (Percona XtraBackup)
For large databases where `mysqldump` is too slow, use XtraBackup for non-blocking, physical backups.
```bash
# Install
sudo apt install percona-xtrabackup-80 -y
# Perform Backup
sudo xtrabackup --backup --target-dir=/data/backups/
```

---

## 5. NoSQL: MongoDB 7.0

### Installation (Official Repo)
```bash
wget -qO- https://www.mongodb.org/static/pgp/server-7.0.asc | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/mongodb-server-7.0.gpg
echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list
sudo apt update
sudo apt install mongodb-org -y
```

### Basic CRUD (Mongo Shell: `mongosh`)
Modern MongoDB uses `mongosh` instead of the legacy `mongo` shell.
```javascript
use test_db
db.users.insertOne({ name: "Alice", Role: "Admin" })
db.users.find().pretty()
```

---

## 6. Performance Tuning

### InnoDB Buffer Pool
The most important setting. Set to ~70% of available RAM if the server is dedicated to MySQL.
`/etc/mysql/mysql.conf.d/mysqld.cnf`:
```text
innodb_buffer_pool_size = 4G # Adjust based on RAM
innodb_flush_log_at_trx_commit = 2 # Better performance, slight crash risk
```

### Slow Query Log
```text
slow_query_log = 1
long_query_time = 2
log_queries_not_using_indexes = 1
```

---

## 7. Troubleshooting

- **Check Process**: `sudo systemctl status mysql`
- **Error Log**: `tail -f /var/log/mysql/error.log`
- **Process List**: `SHOW FULL PROCESSLIST;` in MySQL shell.
- **Explain Plan**: `EXPLAIN SELECT ...` to debug slow queries.
