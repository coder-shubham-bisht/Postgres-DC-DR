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

### 2.2 Access PostgreSQL Shell

```bash
psql -U postgres -d postgres
```

### 2.3 Create Replication User

```sql
CREATE USER replicator WITH REPLICATION ENCRYPTED PASSWORD 'rep@123';
```

### 2.4 Verify Role Creation

```sql
\du
```

### 2.5 Configure Authentication in `pg_hba.conf`

```bash
echo 'host replication replicator 192.168.100.45/24 md5' >> /var/lib/postgresql/data/pg_hba.conf
```

### 2.6 Restart the Container

```bash
podman restart pg_dc
```

## 3. Configure PostgreSQL 16.3 for Streaming Replication

### 3.1 Access the PostgreSQL 16.3 Container

```bash
podman exec -it pg_dr bash
```

### 3.2 Remove Data Directory Contents

```bash
rm -rf /var/lib/postgresql/data/*
```

### 3.3 Set Up Replication from Primary Server

```bash
export PGPASSWORD="rep@123"
pg_basebackup -h 192.168.100.44 -U replicator -p 5433 -D /var/lib/postgresql/data -P -Xs -R
```

## 4. Testing the Replication

### 4.1 Create a Table in `pg_dc`

```sql
CREATE TABLE students (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    age INT CHECK (age > 0)
);
```

### 4.2 Check Table Replication in `pg_dr`

```sql
\dt
```

### 4.3 Insert Data in `pg_dc`

```sql
INSERT INTO students (name, age) VALUES ('ayush', 25);
```

### 4.4 Check Data Replication in `pg_dr`

```sql
SELECT * FROM students;
```

### 4.5 Verify `pg_dr` is Read-Only

```sql
SHOW transaction_read_only;
```

Attempt an insert:

```sql
INSERT INTO students (name, age) VALUES ('Test', 20);
```

#### Expected Output:
```bash
ERROR:  cannot execute INSERT in a read-only transaction
```

## 5. Promote `pg_dr` as Primary Server

### 5.1 Promote Standby to Primary

```sql
SELECT pg_promote();
```

Verify promotion:

```sql
SELECT pg_is_in_recovery();
```

### 5.2 Configure Authentication in `pg_hba.conf`

```bash
echo 'host replication replicator 192.168.100.44/24 md5' >> /var/lib/postgresql/data/pg_hba.conf
podman restart pg_dr
```

## 6. Convert `pg_dc` to Standby Server

### 6.1 Reinitialize `pg_dc`

```bash
podman exec -it pg_dc bash
rm -rf /var/lib/postgresql/data/*
export PGPASSWORD="rep@123"
pg_basebackup -h 192.168.100.45 -U replicator -p 5432 -D /var/lib/postgresql/data -P -Xs -R
```

## 7. Final Testing

### 7.1 Check `pg_dc` is Read-Only

```sql
SHOW transaction_read_only;
```

Attempt an insert:

```sql
INSERT INTO students (name, age) VALUES ('risabh', 40);
```

#### Expected Output:
```bash
ERROR:  cannot execute INSERT in a read-only transaction
```

### 7.2 Check `pg_dr` is Writable

```sql
INSERT INTO students (name, age) VALUES ('risabh', 40);
SELECT * FROM students;
```

This setup ensures high availability with PostgreSQL streaming replication. 🚀

