# PostgreSQL 16 Streaming Replication Setup
Primary + Standby Configuration Guide

This repository demonstrates how to configure PostgreSQL 16 Streaming Replication 
in a Primary–Standby architecture on RHEL/Rocky/AlmaLinux 9.

# 🏗 Architecture

Primary Server (PGPROD)
IP: 192.168.127.144

Standby Server (PGSTB)
IP: 192.168.127.145

Replication Type:
- Physical Streaming Replication
- Asynchronous

# 🔹 PART 1: PRIMARY SERVER CONFIGURATION

## 1️⃣ Install PostgreSQL 16

```bash
dnf -qy module disable postgresql
dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
dnf install -y postgresql16-server
```
## 2️⃣ Create Data & Archive Directories
```
mkdir -p /data/postgresql163/datadir
mkdir -p /data/postgresql163/wal_archive
chown -R postgres:postgres /data/postgresql163
chmod 700 /data/postgresql163/datadir
```
## 3️⃣ Initialize Database
```
su - postgres
/usr/pgsql-16/bin/initdb -D /data/postgresql163/datadir
exit
```
## 4️⃣ Configure systemd
```
Edit:  /usr/lib/systemd/system/postgresql-16.service
Set:   Environment=PGDATA=/data/postgresql163/datadir

Reload & Start:

systemctl daemon-reload
systemctl start postgresql-16
systemctl enable postgresql-16
```
## 5️⃣ Configure postgresql.conf
```
Edit:
/data/postgresql163/datadir/postgresql.conf

Set:
listen_addresses = '*'
wal_level = replica
max_wal_senders = 10
wal_keep_size = 512MB
archive_mode = on
archive_command = 'cp %p /data/postgresql163/wal_archive/%f'
hot_standby = on
```
## 6️⃣ Configure pg_hba.conf
```
Add:
host replication replicator 192.168.127.145/32 md5

Reload:
SELECT pg_reload_conf();
```
## 7️⃣ Create Replication User
```
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'Test@123';
```
# 🔹 PART 2: STANDBY SERVER CONFIGURATION

## 1️⃣ Install PostgreSQL 16
```
dnf -qy module disable postgresql
dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
dnf install -y postgresql16-server
```
## 2️⃣ Create Data Directory
```
mkdir -p /data/postgresql163/datadir
chown -R postgres:postgres /data/postgresql163
chmod 700 /data/postgresql163/datadir
```
## 3️⃣ Configure systemd PGDATA
```
Edit:  /usr/lib/systemd/system/postgresql-16.service
Set:   Environment=PGDATA=/data/postgresql163/datadir

Reload:
systemctl daemon-reload
systemctl enable postgresql-16
```
## 4️⃣ Take Base Backup from Primary
```
sudo -u postgres /usr/pgsql-16/bin/pg_basebackup \
  -h 192.168.127.144 \
  -D /data/postgresql163/datadir \
  -U replicator \
  -P \
  -R

-R automatically creates standby.signal

Connection info is written to postgresql.auto.conf
```
## 5️⃣ Start Standby
```
systemctl start postgresql-16
systemctl status postgresql-16
```
# 🔍 Verification
```
### Check Primary
SELECT * FROM pg_stat_replication;

### Check Standby
SELECT pg_is_in_recovery();

Expected Result:  t

### Check WAL Receiver
SELECT * FROM pg_stat_wal_receiver;
```

# 🧪 Test Replication
```
### On Primary:

CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    age INTEGER
);

INSERT INTO employees (name, age)
VALUES ('John Doe', 30),
       ('Jane Smith', 25);

### On Standby:

SELECT * FROM employees;

**If data is visible → Replication is successful** ✅

```
















