# Postgres-DC-DR

## 1. Setup PostgreSQL 16.8 and PostgreSQL 16.3

### 1.1 Create a Podman Network

First, create a custom network with a defined subnet:

```bash
podman network create --subnet=192.168.100.0/24 pg_replication_net
```

### 1.2 Create a Container for PostgreSQL 16.8

```bash
podman run -d --name pg_dc \
  --network=pg_replication_net \
  --ip=192.168.100.44 \
  -e POSTGRES_PASSWORD=password123 \
  -p 5432:5432 \
  docker.io/library/postgres:16.8
```

#### Output:
```bash
5a6c4a9c4433a90f13d8c63e83bb7a532440817510cf77c5ceeb09279d499fff
```

### 1.3 Create a Container for PostgreSQL 16.3

```bash
podman run -d --name pg_dr \
  --network=pg_replication_net \
  --ip=192.168.100.45 \
  -e POSTGRES_PASSWORD=password1234 \
  -p 5433:5432 \
  docker.io/library/postgres:16.3
```

#### Output:
```bash
5a6c4a9c4433a90f13d8c63e83bb7a532440817510cf77c5ceeb09279d499fff
```

## 2. Configure PostgreSQL 16.8 for Streaming Replication

### 2.1 Access the PostgreSQL 16.8 Container

```bash
podman exec -it pg_dc bash
```

#### Output:
```bash
root@5a6c4a9c4433:/#
```

### 2.2 Access PostgreSQL Shell

```bash
psql -U postgres -d postgres
```

#### Output:
```bash
psql (16.8 (Debian 16.8-1.pgdg120+1))
Type "help" for help.

postgres=#
```

### 2.3 Create Replication User

```sql
CREATE USER replicator WITH REPLICATION ENCRYPTED PASSWORD 'rep@123';
```

#### Output:
```bash
CREATE ROLE
```

### 2.4 Verify Role Creation

```sql
\du
```

#### Output:
```
                              List of roles
 Role name  |                         Attributes                         
------------+------------------------------------------------------------
 postgres   | Superuser, Create role, Create DB, Replication, Bypass RLS
 replicator | Replication
```

### 2.5 Exit the Shell

```bash
exit
```

### 2.6 Configure Authentication in `pg_hba.conf`

```bash
echo 'host replication replicator 192.168.100.45/24 md5' >> /var/lib/postgresql/data/pg_hba.conf
```

### 2.7 Exit the Postgres Container
```bash
exit
```

### 2.8 Restart the Container
```bash
podman restart pg_dc
```

## 3. Configure PostgreSQL 16.3 for Streaming Replication

### Prerequisite
Access the `pg_dr` container:

```bash
podman exec -it pg_dr bash
```

#### Output:
```bash
root@5a6c4a9c4433:/#
```

### 3.1 Remove Data Directory Contents

```bash
rm -rf /var/lib/postgresql/data/*
```

### 3.2 Set Up Replication from Primary Server

```bash
export PGPASSWORD="your_password"
pg_basebackup -h 192.168.100.44 -U replicator -p 5433 -D /var/lib/postgresql/data -P -Xs -R
```
Enter the `replicator` user password when prompted.

#### Output:
```
23160/23160 kB (100%), 1/1 tablespace
```

## 4. Testing the Replication

### Prerequisite
Access the PostgreSQL shell in `pg_dc` container:

```bash
psql -U postgres -d postgres
```

### 4.1 Create a Table in `pg_dc`

```sql
CREATE TABLE students (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    age INT CHECK (age > 0)
);
```

#### Output:
```
CREATE TABLE
```

### 4.2 Check Table Replication in `pg_dr`

**Prerequisite**: Access the PostgreSQL shell in `pg_dr` container.

```sql
\dt
```

#### Output:
```
         List of relations
 Schema |   Name   | Type  |  Owner   
--------+----------+-------+----------
 public | students | table | postgres
(1 row)
```

### 4.3 Insert Data in `pg_dc`

```sql
INSERT INTO students (name, age) VALUES ('ayush', 25);
```

### 4.4 Verify Data Replication in `pg_dr`

```sql
SELECT * FROM students;
```

### 4.5 Check if `pg_dr` is Read-Only

```sql
SHOW transaction_read_only;
```
#### Output:
```
 transaction_read_only
-----------------------
 on
(1 row)
```

## 5. Convert `pg_dr` to Primary Server

### Prerequisite
Access the PostgreSQL shell in `pg_dr` container:

```sql
SELECT pg_promote();
```

#### Output:
```
 pg_promote
------------
 t
(1 row)
```

### 5.1 Check if Recovery Mode is Off

```sql
SELECT pg_is_in_recovery();
```

#### Output:
```
 pg_is_in_recovery
-------------------
 f
(1 row)
```

### 5.2 Update `pg_hba.conf` and Restart `pg_dr`

```bash
echo 'host replication replicator 192.168.100.44/24 md5' >> /var/lib/postgresql/data/pg_hba.conf
podman restart pg_dr
```

## 6. Convert `pg_dc` to Standby Server

### Prerequisite
Access the `pg_dc` container shell:

```bash
rm -rf /var/lib/postgresql/data/*
export PGPASSWORD="replicator"
pg_basebackup -h 192.168.100.45 -U replicator -p 5432 -D /var/lib/postgresql/data -P -Xs -R
```

## 7. Final Testing

### 7.1 Check Read-Only Mode in `pg_dc`

```sql
SHOW transaction_read_only;
```

### 7.2 Verify Inserts in `pg_dr`

```sql
INSERT INTO students (name, age) VALUES ('risabh', 40);
SELECT * FROM students;
```

#### Output:
```
 id |  name  | age
----+--------+-----
  1 | ayush  |  26
 34 | risabh |  40
(2 rows)
```

---
This setup ensures PostgreSQL streaming replication with failover and switchover testing. 🚀

