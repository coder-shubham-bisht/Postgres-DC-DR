# Postgres-DC-DR

## Setup postgres16.8 and postgres16.3

podman inspect my_static_pg | grep '"IPAddress"'

```
podman network create --subnet=192.168.100.0/24 pg_replication_net

```

create a container for postgres-16.8
```bash
podman run -d --name pg_dc  --network=pg_replication_net --ip=192.168.100.44   -e POSTGRES_PASSWORD=password  -p 5432:5432   docker.io/library/postgres:16.8

```

ouput

```bash

5a6c4a9c4433a90f13d8c63e83bb7a532440817510cf77c5ceeb09279d499fff
```

create a container for postgres-16.3
```bash
podman run -d --name pg_dr   --network=pg_replication_net --ip=192.168.100.45   -e POSTGRES_PASSWORD=password    -p 5433:5432   docker.io/library/postgres:16.3
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

```sql
CREATE USER replicator
WITH REPLICATION
ENCRYPTED PASSWORD 'replicator';
```
ouput
```
CREATE ROLE

```
check if role is created
```
postgres=# \du

```
ouput
```
                              List of roles
 Role name  |                         Attributes                         
------------+------------------------------------------------------------
 postgres   | Superuser, Create role, Create DB, Replication, Bypass RLS
 replicator | Replication

```
**exit the shell**
``
exit
``

add authentication to pg_hba.conf
```
echo 'host replication replicator 192.168.100.45/24 md5'>> /var/lib/postgresql/data/pg_hba.conf
```




pg_basebackup -h 192.168.1.85 -U replicator -p 5432 -D /var/lib/postgresql/data -P -Xs -R

SELECT pg_promote();
SELECT pg_is_in_recovery();

## Setup postgres16.3 for streaming replication
```bash
podman exec -it pg_dr bash

```
output
```
root@5a6c4a9c4433:/#
```

removing data direcory content
```
rm -rf rm -rf /var/lib/postgresql/data/*
```
adding replication
```
pg_basebackup -h 192.168.1.85 -U replicator -p 5433 -D /var/lib/postgresql/data -P -Xs -R

```
ouput Password pops up ,enter the replicator user password

```
23160/23160 kB (100%), 1/1 tablespace

```
