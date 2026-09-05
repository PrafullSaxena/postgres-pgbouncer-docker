# PostgreSQL 18 with PgBouncer

Local Docker Compose setup using:

- PostgreSQL `18.6-alpine3.23`
- PgBouncer `1.25.2` (`edoburu/pgbouncer:v1.25.2-p0`)
- SCRAM-SHA-256 password authentication
- Transaction pooling with protocol-level prepared-statement support
- A persistent PostgreSQL data volume

PostgreSQL does not designate an LTS release. PostgreSQL 18 is the current stable
major version and is supported by the PostgreSQL project through November 2030.

## Start the stack

The local `.env` file already contains a generated password and is ignored by Git.
To create a fresh configuration later:

```sh
cp .env.example .env
```

Replace `POSTGRES_PASSWORD` in `.env`, then start the services:

```sh
docker compose up -d
docker compose ps
```

Applications should connect to PgBouncer, not directly to PostgreSQL:

```text
postgresql://app_user:<password-from-.env>@localhost:6432/app_db
```

PostgreSQL is intentionally not published on a host port. PgBouncer is available
only on `127.0.0.1:6432`, so it is not exposed to the local network.

## Verify the connection

This command runs a query through PgBouncer without printing the password:

```sh
docker compose exec -T pgbouncer sh -lc \
  'PGPASSWORD="$DB_PASSWORD" psql -h 127.0.0.1 -p 5432 -U "$DB_USER" -d "$DB_NAME" -c "select version();"'
```

Inspect the PgBouncer pools:

```sh
docker compose exec -T pgbouncer sh -lc \
  'PGPASSWORD="$DB_PASSWORD" psql -h 127.0.0.1 -p 5432 -U "$DB_USER" -d pgbouncer -c "show pools;"'
```

View service logs:

```sh
docker compose logs -f postgres pgbouncer
```

## Pool sizing

The defaults permit 200 client connections while maintaining a normal pool of 20
PostgreSQL connections, plus 5 reserve connections. Change the `PGBOUNCER_*`
values in `.env` based on the application workload and available database memory.

Transaction pooling releases a PostgreSQL connection after every transaction.
Avoid relying on session-scoped state such as temporary tables, session advisory
locks, or `LISTEN` across transactions. Protocol-level prepared statements are
supported by `PGBOUNCER_MAX_PREPARED_STATEMENTS=100`.

## Stop or reset

Stop the containers while preserving database data:

```sh
docker compose down
```

To permanently delete the database volume and initialize a new empty database:

```sh
docker compose down --volumes
```

The second command is destructive.
