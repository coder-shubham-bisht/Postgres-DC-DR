# PostgreSQL DC-DR Setup Guide

## 1. Setup PostgreSQL 16.8 and PostgreSQL 16.3

### 1.1 Create a Podman Network

First, create a custom network with a defined subnet:

```bash
podman network create --subnet=192.168.100.0/24 pg_replication_net
```

### 1.2 Create a Container for PostgreSQL 16.8

```bash
podman run -d --name pg_dc --network=pg_replication_net --ip=192.168.100.44 -e POSTGRES_PASSWORD=password123 -p 5432:5432 docker.io/library/postgres:16.8
```

#### Expected Output:
```bash
<container_id>
```

### 1.3 Create a Container for PostgreSQL 16.3

```bash
podman run -d --name pg_dr --network=pg_replication_net --ip=192.168.100.45 -e POSTGRES_PASSWORD=password1234 -p 5433:5432 docker.io/library/postgres:16.3
```

#### Expected Output:
```bash
<container_id>
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

### 2.5 Exit the PostgreSQL Shell
```
exit
```

### 2.6 Configure Authentication in `pg_hba.conf`

```bash
echo 'host replication replicator 192.168.100.45/24 md5' >> /var/lib/postgresql/data/pg_hba.conf
echo 'host replication replicator 192.168.100.44/24 md5' >> /var/lib/postgresql/data/pg_hba.conf
exit
```

### 2.7 Restart the PostgreSQL 16.8 Container

```bash
podman restart pg_dc
```

## 3. Configure PostgreSQL 16.3 for Streaming Replication

### 3.1 Access the `pg_dr` Container

```bash
podman exec -it pg_dr bash
```
### 3.3 Set Up pg_dr as  Standby Server

```bash
rm -rf /var/lib/postgresql/data/*
export PGPASSWORD="rep@123"
pg_basebackup -h 192.168.100.44 -U replicator -p 5432 -D /var/lib/postgresql/data -P -Xs -R
exit
```

### 3.4 Restart the `pg_dr` Container

```bash
podman restart pg_dr
```

## 4. Testing Replication

### 4.1 Create a Table in `pg_dc` postgres database

#### Prequiste 
Access the pg_dc postgresql shell

```sql
CREATE TABLE students (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    age INT CHECK (age > 0)
);
```

### 4.2 Check Table Replication in `pg_dr`postgres database

#### Prequiste 
Access the pg_dr postgresql shell

```sql
\dt
```
#### Output
```
          List of relations
 Schema |   Name   | Type  |  Owner   
--------+----------+-------+----------
 public | students | table | postgres
(1 row)

```


### 4.3 Insert Data in `pg_dc`

```sql
INSERT INTO students (name, age) VALUES ('Ayush', 25);
```

### 4.4 Verify Data Replication in `pg_dr`

```sql
SELECT * FROM students;
```

#### Output
```
 id | name  | age 
----+-------+-----
  1 | Ayush |  25
(1 row)

```

### 4.5 Check Read-Only Mode in `pg_dr`

```sql
SHOW transaction_read_only;
```

## 5. Promote `pg_dr` to Primary Server

```sql
SELECT pg_promote();
```

#### Output

```
 transaction_read_only 
-----------------------
 on
(1 row)
```

### 5.1 Verify Recovery Mode is Off

```sql
SELECT pg_is_in_recovery();
```
### Output
```
 pg_is_in_recovery 
-------------------
 f
(1 row)
```

## 6. Convert `pg_dc` to Standby Server

```bash
rm -rf /var/lib/postgresql/data/*
export PGPASSWORD="rep@123"
pg_basebackup -h 192.168.100.45 -U replicator -p 5432 -D /var/lib/postgresql/data -P -Xs -R
exit
```

### 6.1 Restart the `pg_dc` Container

```bash
podman restart pg_dc
```

## 7. Final Testing

### 7.1 Check Read-Only Mode in `pg_dc`

```sql
SHOW transaction_read_only;
```

### 7.2 Insert Data in `pg_dr`

```sql
INSERT INTO students (name, age) VALUES ('Rishabh', 40);
SELECT * FROM students;
```

### 7.3 Verify Replication in `pg_dc`

```sql
SELECT * FROM students;
```

## 8. Convert `pg_dc` to Primary Server

```sql
SELECT pg_promote();
```

### 8.1 Verify Recovery Mode is Off in `pg_dc`

```sql
SELECT pg_is_in_recovery();
```

## 9. Convert `pg_dr` to Standby Server

```bash
rm -rf /var/lib/postgresql/data/*
export PGPASSWORD="rep@123"
pg_basebackup -h 192.168.100.44 -U replicator -p 5432 -D /var/lib/postgresql/data -P -Xs -R
exit
```

### 9.1 Restart the `pg_dr` Container

```bash
podman restart pg_dr
```

## 10. Monitoring Setup



### 10.1 Deploy Postgres Exporter for `pg_dc`

```bash

podman run -d --name pg_dc_exporter  --network=pg_replication_net --ip=192.168.100.46   -e DATA_SOURCE_NAME="postgresql://postgres:password123@192.168.100.44:5432/postgres?sslmode=disable"   -p 9187:9187 quay.io/prometheuscommunity/postgres-exporter
```


### 10.2 Create `prometheus.yml`

```yaml
global:
  scrape_interval: 5s

scrape_configs:
  - job_name: 'pg_dc_exporter'
    static_configs:
      - targets: ['192.168.100.46:9187']
```

### 10.3 Deploy Prometheus

```bash
podman run -d --name prometheus_dc_dr --network=pg_replication_net --ip=192.168.100.48 \
  -p 9090:9090 -v ./prometheus.yml:/etc/prometheus/prometheus.yml \
  docker.io/prom/prometheus
```

### 10.5 Deploy Grafana

```bash
podman run -d --name grafana_pg_dc_dr --network=pg_replication_net --ip=192.168.100.49 \
  -p 3000:3000 docker.io/grafana/grafana
```


### Monitoring stats
pg_stat_replication_pg_wal_lsn_diff

