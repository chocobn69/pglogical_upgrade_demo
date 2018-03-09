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

```
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

You have to generate postgresql.conf and pg_hba.conf file according to logical replication tutorial

Let's launch the stack : `docker-compose up -d`

Now we can connect to first db1 db : `docker exec -ti divers_db1_1 psql -U postgres`

And second (db2) `docker exec -ti divers_db2_1 psql -U postgres -p 6432`

## Creating replication

Create a logical replication slote (??)
```
SELECT * FROM pg_create_logical_replication_slot('my_logical_replication_slot', 'test_decoding');
```

```
          slot_name          | xlog_position 
-----------------------------+---------------
 my_logical_replication_slot | 0/14EEE90
 ```
 
Then on db2, we should be able to create subscription

```create subscription sub_test connection 'host=db1 port=5432 user=postgres dbname=postgres' publication my_logical_replication_slot;```

But right now I only get this error :

```ERROR:  could not receive list of replicated tables from the publisher: ERROR:  syntax error```


I think we can't easily use pg 10 logical replication with pg 9.6 logical replication slot. We may have to use an output plugins (instead of 'test_decoding' one.

