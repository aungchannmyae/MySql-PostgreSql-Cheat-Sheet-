# MySQL ↔ PostgreSQL Command Cheatsheet

Each entry is three lines: **what it does**, the **MySQL** form, the **PostgreSQL** form.

Backslash commands (`\l`, `\dt`, `\du`) are `psql` meta-commands — they only work
inside the `psql` client, not in a SQL file or an application connection.

---

## Connecting

```
Connect to a server
MySQL       mysql -h 127.0.0.1 -P 3306 -u root -p
PostgreSQL  psql -h 127.0.0.1 -p 5432 -U postgres -d postgres

Connect straight into one database
MySQL       mysql -u medi -p medi
PostgreSQL  psql -U medi -d medi

Run a single statement and exit
MySQL       mysql -u root -p -e "SHOW DATABASES;"
PostgreSQL  psql -U postgres -c "\l"

Run a .sql file
MySQL       mysql -u root -p medi < script.sql
PostgreSQL  psql -U postgres -d medi -f script.sql

Open a client inside a Docker container
MySQL       docker exec -it <container> mysql -u root -p
PostgreSQL  docker exec -it <container> psql -U postgres

Quit the client
MySQL       EXIT;
PostgreSQL  \q
```

---

## Databases

```
List all databases
MySQL       SHOW DATABASES;
PostgreSQL  \l

Create a database with UTF-8
MySQL       CREATE DATABASE medi CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
PostgreSQL  CREATE DATABASE medi ENCODING 'UTF8' TEMPLATE template0;

Switch to another database
MySQL       USE medi;
PostgreSQL  \c medi

Show which database you are in
MySQL       SELECT DATABASE();
PostgreSQL  SELECT current_database();

Rename a database
MySQL       -- not supported; dump and restore into a new name
PostgreSQL  ALTER DATABASE medi RENAME TO medi_old;

Change the owner of a database
MySQL       -- no concept of database owner
PostgreSQL  ALTER DATABASE medi OWNER TO medi;

Drop a database
MySQL       DROP DATABASE IF EXISTS medi;
PostgreSQL  DROP DATABASE IF EXISTS medi;
```

---

## Users and roles

```
List all users
MySQL       SELECT user, host FROM mysql.user;
PostgreSQL  \du

Create a user
MySQL       CREATE USER 'medi'@'%' IDENTIFIED BY 'secret';
PostgreSQL  CREATE USER medi WITH PASSWORD 'secret';

Create a user that may create databases
MySQL       CREATE USER 'medi'@'%' IDENTIFIED BY 'secret'; GRANT CREATE ON *.* TO 'medi'@'%';
PostgreSQL  CREATE USER medi WITH PASSWORD 'secret' CREATEDB;

Create a superuser / full admin
MySQL       GRANT ALL PRIVILEGES ON *.* TO 'medi'@'%' WITH GRANT OPTION;
PostgreSQL  CREATE USER medi WITH SUPERUSER PASSWORD 'secret';

Change a user's password
MySQL       ALTER USER 'medi'@'%' IDENTIFIED BY 'newsecret';
PostgreSQL  ALTER USER medi WITH PASSWORD 'newsecret';

Rename a user
MySQL       RENAME USER 'old'@'%' TO 'new'@'%';
PostgreSQL  ALTER USER old RENAME TO new;

Show who you are connected as
MySQL       SELECT CURRENT_USER();
PostgreSQL  SELECT current_user;

Drop a user
MySQL       DROP USER IF EXISTS 'medi'@'%';
PostgreSQL  DROP USER IF EXISTS medi;
```

---

## Permissions

```
Grant full access to one database
MySQL       GRANT ALL PRIVILEGES ON medi.* TO 'medi'@'%';
PostgreSQL  GRANT ALL PRIVILEGES ON DATABASE medi TO medi;

Grant access to all existing tables (Postgres needs this separately)
MySQL       -- covered by the database-level grant above
PostgreSQL  GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO medi;

Grant access to tables created in the future
MySQL       -- covered by the database-level grant above
PostgreSQL  ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON TABLES TO medi;

Grant only read/write, no DDL
MySQL       GRANT SELECT, INSERT, UPDATE, DELETE ON medi.* TO 'medi'@'%';
PostgreSQL  GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO medi;

Grant read-only
MySQL       GRANT SELECT ON medi.* TO 'readonly'@'%';
PostgreSQL  GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly;

Grant on a single table
MySQL       GRANT SELECT ON medi.medicines TO 'medi'@'%';
PostgreSQL  GRANT SELECT ON medicines TO medi;

Show a user's privileges
MySQL       SHOW GRANTS FOR 'medi'@'%';
PostgreSQL  \dp

Revoke privileges
MySQL       REVOKE ALL PRIVILEGES ON medi.* FROM 'medi'@'%';
PostgreSQL  REVOKE ALL PRIVILEGES ON DATABASE medi FROM medi;

Apply privilege changes
MySQL       FLUSH PRIVILEGES;
PostgreSQL  -- takes effect immediately, nothing to flush
```

---

## Tables and schema

```
List tables
MySQL       SHOW TABLES;
PostgreSQL  \dt

Describe a table
MySQL       DESCRIBE medicines;
PostgreSQL  \d medicines

Describe a table with sizes and indexes
MySQL       SHOW FULL COLUMNS FROM medicines;
PostgreSQL  \d+ medicines

Show the CREATE statement
MySQL       SHOW CREATE TABLE medicines\G
PostgreSQL  pg_dump -U postgres -d medi -t medicines --schema-only

Count rows
MySQL       SELECT COUNT(*) FROM medicines;
PostgreSQL  SELECT COUNT(*) FROM medicines;

Create a table
MySQL       CREATE TABLE notes (id BIGINT AUTO_INCREMENT PRIMARY KEY, body TEXT);
PostgreSQL  CREATE TABLE notes (id BIGSERIAL PRIMARY KEY, body TEXT);

Rename a table
MySQL       RENAME TABLE notes TO remarks;
PostgreSQL  ALTER TABLE notes RENAME TO remarks;

Add a column
MySQL       ALTER TABLE medicines ADD COLUMN price INT NULL;
PostgreSQL  ALTER TABLE medicines ADD COLUMN price integer NULL;

Drop a column
MySQL       ALTER TABLE medicines DROP COLUMN price;
PostgreSQL  ALTER TABLE medicines DROP COLUMN price;

Empty a table and reset auto-increment
MySQL       TRUNCATE TABLE medicines;
PostgreSQL  TRUNCATE TABLE medicines RESTART IDENTITY CASCADE;

Drop a table
MySQL       DROP TABLE IF EXISTS medicines;
PostgreSQL  DROP TABLE IF EXISTS medicines;
```

---

## Indexes and constraints

```
List indexes on a table
MySQL       SHOW INDEX FROM medicines;
PostgreSQL  SELECT * FROM pg_indexes WHERE tablename = 'medicines';

Create a unique index
MySQL       CREATE UNIQUE INDEX medicines_slug_unique ON medicines (slug);
PostgreSQL  CREATE UNIQUE INDEX medicines_slug_unique ON medicines (slug);

Create an index without locking writes
MySQL       CREATE INDEX idx_name ON medicines (generic_name) ALGORITHM=INPLACE LOCK=NONE;
PostgreSQL  CREATE INDEX CONCURRENTLY idx_name ON medicines (generic_name);

Drop an index
MySQL       DROP INDEX medicines_slug_unique ON medicines;
PostgreSQL  DROP INDEX medicines_slug_unique;

Show foreign keys
MySQL       SELECT * FROM information_schema.KEY_COLUMN_USAGE WHERE TABLE_NAME = 'favourites';
PostgreSQL  \d favourites

Explain a query plan
MySQL       EXPLAIN SELECT * FROM medicines WHERE slug = 'Aspirin';
PostgreSQL  EXPLAIN ANALYZE SELECT * FROM medicines WHERE slug = 'Aspirin';
```

---

## Connections, locks and processes

```
Show running queries
MySQL       SHOW FULL PROCESSLIST;
PostgreSQL  SELECT pid, state, query FROM pg_stat_activity;

Count open connections
MySQL       SHOW STATUS LIKE 'Threads_connected';
PostgreSQL  SELECT count(*) FROM pg_stat_activity;

Kill a connection
MySQL       KILL 1234;
PostgreSQL  SELECT pg_terminate_backend(1234);

Show the max connection limit
MySQL       SHOW VARIABLES LIKE 'max_connections';
PostgreSQL  SHOW max_connections;

Show server version
MySQL       SELECT VERSION();
PostgreSQL  SELECT version();
```

---

## Sizes and maintenance

```
Size of one database
MySQL       SELECT SUM(data_length + index_length) / 1024 / 1024 AS mb FROM information_schema.tables WHERE table_schema = 'medi';
PostgreSQL  SELECT pg_size_pretty(pg_database_size('medi'));

Size of one table
MySQL       SELECT (data_length + index_length) / 1024 / 1024 AS mb FROM information_schema.tables WHERE table_name = 'medicines';
PostgreSQL  SELECT pg_size_pretty(pg_total_relation_size('medicines'));

List tables by size
MySQL       SELECT table_name, (data_length + index_length) / 1024 / 1024 AS mb FROM information_schema.tables WHERE table_schema = 'medi' ORDER BY mb DESC;
PostgreSQL  \dt+

Reclaim space / rebuild
MySQL       OPTIMIZE TABLE medicines;
PostgreSQL  VACUUM FULL ANALYZE medicines;

Refresh planner statistics
MySQL       ANALYZE TABLE medicines;
PostgreSQL  ANALYZE medicines;
```

---

## Backup and restore

```
Dump one database
MySQL       mysqldump -u root -p medi > medi.sql
PostgreSQL  pg_dump -U postgres medi > medi.sql

Dump every database
MySQL       mysqldump -u root -p --all-databases > all.sql
PostgreSQL  pg_dumpall -U postgres > all.sql

Dump schema only, no rows
MySQL       mysqldump -u root -p --no-data medi > schema.sql
PostgreSQL  pg_dump -U postgres --schema-only medi > schema.sql

Dump rows only, no schema
MySQL       mysqldump -u root -p --no-create-info medi > data.sql
PostgreSQL  pg_dump -U postgres --data-only medi > data.sql

Dump a single table
MySQL       mysqldump -u root -p medi medicines > medicines.sql
PostgreSQL  pg_dump -U postgres -t medicines medi > medicines.sql

Restore a dump
MySQL       mysql -u root -p medi < medi.sql
PostgreSQL  psql -U postgres -d medi -f medi.sql

Dump from a Docker container to the host
MySQL       docker exec <container> mysqldump -u root -pSECRET medi > medi.sql
PostgreSQL  docker exec <container> pg_dump -U postgres medi > medi.sql
```

---

## Laravel connection settings

```
Driver name in .env
MySQL       DB_CONNECTION=mysql
PostgreSQL  DB_CONNECTION=pgsql

Default port
MySQL       DB_PORT=3306
PostgreSQL  DB_PORT=5432

Required PHP extension
MySQL       pdo_mysql
PostgreSQL  pdo_pgsql

Check the connection from the app
MySQL       php artisan db:show
PostgreSQL  php artisan db:show

Open a SQL shell through Laravel
MySQL       php artisan db
PostgreSQL  php artisan db
```

---

## Gotchas worth knowing

**MySQL — the host part of a username is part of the identity.**
`'medi'@'localhost'` and `'medi'@'%'` are two different users with separate
passwords and grants. A user created as `@'localhost'` only works for connections
originating inside the database server itself, so an app in another container is
rejected with `Host '...' is not allowed to connect`. Use `@'%'` for containerised
apps.

**PostgreSQL — `GRANT ALL ON DATABASE` does not grant table access.**
It only allows connecting and creating schemas. Reading and writing tables needs
`GRANT ... ON ALL TABLES IN SCHEMA public`, and tables created *later* need
`ALTER DEFAULT PRIVILEGES`. Forgetting the second one is the classic "permission
denied for table users" right after a migration.

**A hang is not a permission problem.**
Authentication, missing-database and privilege errors all return immediately with
a specific error code. If a connection hangs until timeout instead, the traffic is
never reaching the server — that is a network, firewall or wrong-host issue.

**`utf8` in MySQL is not UTF-8.**
It is a 3-byte subset that cannot store emoji or some CJK characters. Always use
`utf8mb4` with `utf8mb4_unicode_ci`.

**`TRUNCATE` does not reset sequences in PostgreSQL by default.**
Add `RESTART IDENTITY` or the next insert continues from the old sequence value.
MySQL resets `AUTO_INCREMENT` automatically.
