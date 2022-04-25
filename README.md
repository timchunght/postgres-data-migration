## Postgres Data Migration Guide

### Dump source database


```sql
docker run -it postgres:13.6-alpine /bin/bash
PGPASSWORD=sourcepass pg_dump -Fc --no-acl --no-owner -h sourcehost -U sourceuser --port=sourceport sourcedbname > latest.dump
```

### Restore dump to destination database

```sql
PGPASSWORD=destpass pg_restore --verbose --clean --no-acl --no-owner -h destdbhost -U destdbuser -d destdbname --port=destport latest.dump
```

### Verify database objects

Here are some things that can be verified to ensure that the migration was successful.

* Verify that all the tables and indexes have been created in destination database
* Ensure that triggers and constraints are migrated and are working as expected
* Verify row counts for tables

Run a COUNT(*) command to verify that the total number of rows match between the source database and destination DB. This can be done as shown below using a PLPGSQL function.

#### Step 1. Create the following function to print the number of rows in a single table.

```sql
create function
cnt_rows(schema text, tablename text) returns integer
as
$body$
declare
  result integer;
  query varchar;
begin
  query := 'SELECT count(1) FROM ' || schema || '.' || tablename;
  execute query into result;
  return result;
end;
$body$
language plpgsql;
```

#### Example

Below is an example illustrating the output of running the above on the Northwind database.

```sql
example=# SELECT table_schema, table_name, cnt_rows(table_schema, table_name)
    FROM information_schema.tables
    WHERE table_schema NOT IN ('pg_catalog', 'information_schema')
    AND table_type='BASE TABLE' order by cnt_rows desc;


 table_schema |       table_name       | cnt_rows
--------------+------------------------+----------
 public       | order_details          |     2155
 public       | orders                 |      830
 public       | customers              |       91
 public       | products               |       77
 public       | territories            |       53
 public       | us_states              |       51
 public       | employee_territories   |       49
 public       | suppliers              |       29
 public       | employees              |        9
 public       | categories             |        8
 public       | shippers               |        6
 public       | region                 |        4
 public       | customer_customer_demo |        0
 public       | customer_demographics  |        0
(14 rows)
```