+++
title = 'Setting up the schema with your migration tool'
linkTitle = 'Migrations'
weight = 5
tool = 'migrations · start script · init SQL'
date = 2026-10-21T06:20:00+02:00
summary = "Three ways to get the schema in place before the first scenario, with real tools: the service migrates itself, a migration CLI runs before it, or Postgres runs plain SQL when its container starts. And the one line most tools need: keep their history table."
tags = ['databases', 'migrations', 'postgres']
+++

The reset empties tables but never drops them, so the schema is made **once
per run**, before the first scenario. Do it the way production does, and every
run tests your migrations too.

| Production migrates… | In tomato | |
|---|---|---|
| inside the service, at startup | `app.command` starts the service; nothing else | [1](#1-the-service-migrates-itself) |
| with a CLI before the service starts (an init container, a release step) | a start script: migrate, then `exec` the service | [2](#2-a-migration-cli-then-the-service) |
| from a SQL dump or `schema.sql` | mount it into the database's init directory | [3](#3-plain-sql-run-by-postgres) |

## 1. The service migrates itself

Nothing to configure: tomato starts Postgres, then the service, which
migrates before it listens. The ready check waits for it.

```yaml
app:
  command: ./orders
  ready: { type: http, path: /health, timeout: 120s }
  env:
    DATABASE_URL: "postgres://orders:orders@{{.postgres.host}}:{{.postgres.port.5432}}/orders?sslmode=disable"
```

{{< code-tabs >}}
{{< tab "Go" >}}
```go
// golang-migrate, with the migrations embedded in the binary
//go:embed migrations/*.sql
var migrations embed.FS

func migrateUp(url string) error {
	src, err := iofs.New(migrations, "migrations")
	if err != nil {
		return err
	}
	m, err := migrate.NewWithSourceInstance("iofs", src, url)
	if err != nil {
		return err
	}
	if err := m.Up(); err != nil && err != migrate.ErrNoChange {
		return err
	}
	return nil
}
```
History table `schema_migrations`: kept by tomato.
{{< /tab >}}
{{< tab "Java" >}}
```java
// Flyway, before the server starts
Flyway.configure()
    .dataSource(System.getenv("DATABASE_URL"), "orders", "orders")
    .locations("classpath:db/migration")
    .load()
    .migrate();
```
History table `flyway_schema_history`: kept by tomato. Spring Boot runs Flyway
or Liquibase at startup on its own when it finds them on the classpath.
{{< /tab >}}
{{< tab "Python" >}}
```python
# Alembic, before the app starts serving
from alembic import command
from alembic.config import Config

command.upgrade(Config("alembic.ini"), "head")
```
History table `alembic_version`: **add it to `exclude`**.
{{< /tab >}}
{{< tab "Node.js" "nodejs" >}}
```js
// node-pg-migrate, before the server listens
import { runner } from 'node-pg-migrate';

await runner({
  databaseUrl: process.env.DATABASE_URL,
  dir: 'migrations',
  direction: 'up',
  migrationsTable: 'pgmigrations',
});
```
History table `pgmigrations`: **add it to `exclude`**.
{{< /tab >}}
{{< /code-tabs >}}

## 2. A migration CLI, then the service

When production runs migrations as a separate step, a Kubernetes init
container or a release command, run the same CLI right before the service,
from a start script:

```sh
#!/bin/sh
# scripts/start.sh: migrate, then become the service
set -e
npx dbmate --no-dump-schema up
npx node-pg-migrate up --migrations-dir inventory-migrations --database-url-var INVENTORY_URL
exec node app.js
```

```yaml
app:
  command: ./scripts/start.sh
  port: 3000
  ready: { type: http, path: /health, timeout: 120s }
  env:
    DATABASE_URL: "postgres://shop:shop@{{.postgres.host}}:{{.postgres.port.5432}}/orders?sslmode=disable"
    INVENTORY_URL: "postgres://shop:shop@{{.postgres.host}}:{{.postgres.port.5432}}/inventory?sslmode=disable"
```

Three details:

- **A script, not `sh -c "… && …"`.** `app.command` is split on spaces and
  run without a shell, so `&&` and quotes don't work there.
- **`exec` the service.** It replaces the script with the service, so the
  SIGTERM tomato sends at the end reaches it, and so does any coverage it
  writes on exit.
- **`set -e`.** A failed migration stops the script; tomato reports that the
  service didn't start, with the migration's error in the log.

The same script works for any CLI: `migrate -path migrations -database
"$DATABASE_URL" up` (golang-migrate), `goose up`, `flyway migrate`,
`alembic upgrade head`, `npx prisma migrate deploy`.

## 3. Plain SQL, run by Postgres

A `schema.sql`, a `pg_dump --schema-only`, Rails' `structure.sql`: Postgres
runs every file in `/docker-entrypoint-initdb.d` when its container creates
the database, before your service starts.

```yaml
containers:
  postgres:
    image: postgres:16-alpine
    env: { POSTGRES_USER: shop, POSTGRES_PASSWORD: shop, POSTGRES_DB: orders }
    ports: ["5432/tcp"]
    volumes:
      - ${PWD}/initdb:/docker-entrypoint-initdb.d
    wait_for: { type: log, target: "database system is ready to accept connections" }
```

```sql
-- initdb/001_databases.sql
CREATE DATABASE inventory;
CREATE DATABASE legacy;
\connect legacy
CREATE TABLE products (sku text PRIMARY KEY, name text NOT NULL);
```

Use `${PWD}/…`, not `./…`: tomato v2.7.0 hands a relative path to Docker as a
named volume, and Postgres then finds an empty directory. It's also the
simplest way to give **each service its own database** in one Postgres
container, as above.

## Keep the history table

The reset empties every table, and a migration tool's history is a table.
Empty it, and the tool forgets what it applied: the next migration run (a
restart in a `lifecycle` scenario, say) tries to create tables that exist.

tomato keeps the history of the tools it knows. For the rest, one line:

| Tool | History table | Kept by tomato? |
|---|---|---|
| golang-migrate, dbmate, Rails | `schema_migrations` | yes |
| goose | `goose_db_version` | yes |
| Flyway | `flyway_schema_history` | yes |
| Liquibase | `databasechangelog`, `databasechangeloglock` | yes |
| Alembic | `alembic_version` | `exclude: [alembic_version]` |
| Django | `django_migrations` | `exclude: [django_migrations]` |
| Prisma | `_prisma_migrations` | `exclude: [_prisma_migrations]` |
| node-pg-migrate | `pgmigrations` | `exclude: [pgmigrations]` |
| Knex | `knex_migrations`, `knex_migrations_lock` | `exclude: [knex_migrations, knex_migrations_lock]` |
| sqlx (Rust) | `_sqlx_migrations` | `exclude: [_sqlx_migrations]` |
| Entity Framework | `__EFMigrationsHistory` | `exclude: [__EFMigrationsHistory]` |

Check it with one scenario, which fails the moment the history is emptied:

```gherkin
Scenario: Each tool's history survives the reset
  Then "orders-db" table "schema_migrations" has "1" rows
  And "inventory-db" table "pgmigrations" has "1" rows
```

Without `exclude: [pgmigrations]` on `inventory-db`, that scenario fails with
`table pgmigrations has 0 rows, expected 1`.
