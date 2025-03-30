# Postgres-DC-DR

## Setup postgres16.8 and postgres16.3

podman inspect my_static_pg | grep '"IPAddress"'

```
podman network create --subnet=192.168.100.0/24 my_custom_network

```

create a container for postgres-16.8
```bash
podman run -d --name pg_dc     --network=pg_replication_net --ip=192.168.100.44   -e POSTGRES_PASSWORD=password    -p 5432:5432   docker.io/library/postgres:16.8

```

ouput

```bash

5a6c4a9c4433a90f13d8c63e83bb7a532440817510cf77c5ceeb09279d499fff
```

create a container for postgres-16.3
```bash
podman run -d --name pg_dr     --network=pg_replication_net --ip=192.168.100.45   -e POSTGRES_PASSWORD=password    -p 5433:5432   docker.io/library/postgres:16.3
```
ouput
```bash
5a6c4a9c4433a90f13d8c63e83bb7a532440817510cf77c5ceeb09279d499fff

```
## Setup postgres16.8 for streaming replication
```bash
podman exec -it pg_dc bash

```
output
```
root@5a6c4a9c4433:/#
```

```bash
root@5a6c4a9c4433:/# psql -U postgres -d postgres

```
ouput
```
psql (16.8 (Debian 16.8-1.pgdg120+1))
Type "help" for help.

postgres=# 
```

```bash
postgres=# CREATE USER replicator
WITH REPLICATION
ENCRYPTED PASSWORD 'replicator';
```
ouput
```
CREATE ROLE

```
```
postgres=# \du

```
```
                              List of roles
 Role name  |                         Attributes                         
------------+------------------------------------------------------------
 postgres   | Superuser, Create role, Create DB, Replication, Bypass RLS
 replicator | Replication

```

```

SHOW wal_level;
SHOW max_wal_senders;
SHOW listen_addresses;
SHOW hot_standby;

```
```
postgres=# ALTER SYSTEM SET wal_level TO 'replica';
ALTER SYSTEM SET max_wal_senders TO '5';
ALTER SYSTEM SET listen_addresses TO '*';
ALTER SYSTEM SET hot_standby TO 'ON';

```
ouput
```
ALTER SYSTEM
ALTER SYSTEM
ALTER SYSTEM
ALTER SYSTEM

```

echo 'host replication replicator 192.168.0.28/32 md5'>> /var/lib/postgresql/data/pg_hba.conf
pg_basebackup -h 192.168.1.85 -U replicator -p 5432 -D /var/lib/postgresql/data -P -Xs -R


