# pg logical test
We will test logical replication between db with different version and locale

- db1 : 9.6 and en_US.UTF-8
- db2 : 10.3 and C

## links

- https://www.postgresql.org/docs/10/static/sql-createsubscription.html
- https://www.postgresql.org/docs/9.6/static/logicaldecoding-example.html
- https://www.2ndquadrant.com/en/resources/pglogical/pglogical-docs/

## Initial setup

Here is the docker-compose.yml I will use:

```yaml
version: '3.5'

services:

  db1:
    image: postgres:9
    restart: always
    environment:
        POSTGRES_PASSWORD: db1
        POSTGRES_INITDB_ARGS: '--locale=en_US.UTF-8'
    volumes:
        - ./my-postgres-db1.conf:/etc/postgresql/postgresql.conf
        - ./my-pg_hba.conf:/etc/postgresql/pg_hba.conf
    command: -c config_file=/etc/postgresql/postgresql.conf

  db2:
    image: postgres:10
    restart: always
    environment:
        POSTGRES_PASSWORD: db2
        POSTGRES_INITDB_ARGS: '--locale=C'
    volumes:
        - ./my-postgres-db2.conf:/etc/postgresql/postgresql.conf
        - ./my-pg_hba.conf:/etc/postgresql/pg_hba.conf
    command: -c config_file=/etc/postgresql/postgresql.conf
```

You have to generate postgresql.conf and pg_hba.conf file according to pglogical replication tutorial
https://www.2ndquadrant.com/en/resources/pglogical/pglogical-installation-instructions/

Let's launch the stack : `docker-compose up -d`

Now we can connect to first db1 db : `docker exec -ti divers_db1_1 psql -U postgres`

And second (db2) `docker exec -ti divers_db2_1 psql -U postgres -p 6432`

## Initial structure / datas

As we will migrate an existing database, we will create initial tables and datas on db1.

Structure :
```sql
create table table1(id serial primary key, comment text);
create table table2(id serial primary key, comment text);
```

Note: The structure must be created both on db1 and db2, as pglogical does not handle
initial DDL.


Datas:
```sql
insert into table1(comment)
select md5(random()::text)
from generate_series (1,10);

insert into table2(comment)
select md5(random()::text)
from generate_series (1,10);
```


## pglogical install

postgresql.conf

```conf
wal_level = 'logical'
max_worker_processes = 10   # one per database needed on provider node
                            # one per node needed on subscriber node
max_replication_slots = 10  # one per node needed on provider node
max_wal_senders = 10        # one per node needed on provider node
shared_preload_libraries = 'pglogical'
track_commit_timestamp = on
```

On each node

```sql
CREATE EXTENSION pglogical;
```

Then on db1 (provider node)

```sql
SELECT pglogical.create_node(
    node_name := 'provider1',
    dsn := 'host=db1 port=5432 dbname=postgres'
);
```

Add all tables in replication set on db1

```sql
SELECT pglogical.replication_set_add_all_tables('default', ARRAY['public']);
```

Then on db2 (subscriber)

```sql
SELECT pglogical.create_node(
    node_name := 'subscriber1',
    dsn := 'host=db2 port=6432 dbname=postgres'
);
```

Then start replication on db2

```sql
SELECT pglogical.create_subscription(
    subscription_name := 'subscription1',
    provider_dsn := 'host=db1 port=5432 dbname=postgres'
);
```

## Errors

Sadly, it looks like we can't use pglogical with two different locale

```
ERROR:  encoding conversion for binary datum not supported yet
DETAIL:  expected_encoding SQL_ASCII must be unset or match server_encoding UTF8
CONTEXT:  slot "pgl_postgres_provider1_subscription1", output plugin "pglogical_output", in the startup callback
```
