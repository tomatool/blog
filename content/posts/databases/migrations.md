+++
title = 'Setting up the schema with your migration tool'
linkTitle = 'Migrations'
weight = 5
tool = 'in the app · a tool before it · init SQL'
date = 2026-10-21T06:20:00+02:00
summary = "Whether your service migrates itself, or a tool like Flyway, golang-migrate, Alembic or Prisma does it before the service starts, the schema is made once per run, before the first scenario. Each case, in each language and tool, and the one line most tools need: keep their history table."
tags = ['databases', 'migrations', 'postgres']
+++

The reset empties tables but never drops them, so the schema is made **once
per run**, before the first scenario. Make it the way production does, and
every run tests your migrations too. Which case are you?

| In production, migrations run… | In tomato | |
|---|---|---|
| inside the service, as it starts (a library) | nothing to add: tomato starts the service | [Case 1](#case-1-the-service-runs-its-migrations) |
| as a separate step before the service (a CLI: an init container, a release step) | a start script runs the CLI, then the service | [Case 2](#case-2-a-migration-tool-runs-before-the-service) |
| from SQL files: a dump, `schema.sql` | the database container runs them when it starts | [Case 3](#case-3-plain-sql-files) |

Whichever it is, most tools keep a **history table**, and some of those need
one line in `tomato.yml`: [keep the history table](#keep-the-history-table).

## Case 1: the service runs its migrations

tomato starts the database, then your service, which migrates before it
listens. The ready check waits until it does. Nothing in `tomato.yml` is about
migrations:

```yaml
app:
  command: ./orders
  ready: { type: http, path: /health, timeout: 120s }
  env:
    DATABASE_URL: "postgres://orders:orders@{{.postgres.host}}:{{.postgres.port.5432}}/orders?sslmode=disable"
```

The call, in your language:

{{< code-tabs >}}
{{< tab "Go" >}}
```go
// golang-migrate, migrations embedded in the binary
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
History: `schema_migrations`, kept by tomato. With goose, `goose.Up(db, "migrations")`; history `goose_db_version`, kept too.
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
History: `flyway_schema_history`, kept by tomato. Spring Boot runs Flyway or Liquibase on its own at startup when it finds them on the classpath; Liquibase's `databasechangelog` tables are kept too. The same code works from Kotlin.
{{< /tab >}}
{{< tab "Python" >}}
```python
# Alembic, before the app serves
from alembic import command
from alembic.config import Config

command.upgrade(Config("alembic.ini"), "head")

# Django: django.core.management.call_command("migrate")
```
History: `alembic_version`, add `exclude: [alembic_version]`. Django: `exclude: [django_migrations]`.
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

// Knex: await knex.migrate.latest();
```
History: `pgmigrations`, add `exclude: [pgmigrations]`. Knex: `exclude: [knex_migrations, knex_migrations_lock]`. Prisma has no API for this; run its CLI as in Case 2.
{{< /tab >}}
{{< tab "Rust" >}}
```rust
// sqlx, migrations embedded at compile time
let pool = PgPool::connect(&std::env::var("DATABASE_URL")?).await?;
sqlx::migrate!("./migrations").run(&pool).await?;
```
History: `_sqlx_migrations`, add `exclude: [_sqlx_migrations]`.
{{< /tab >}}
{{< tab ".NET" "dotnet" >}}
```csharp
// Entity Framework Core, before app.Run()
using (var scope = app.Services.CreateScope())
{
    scope.ServiceProvider.GetRequiredService<OrdersDb>().Database.Migrate();
}
```
History: `__EFMigrationsHistory`, add `exclude: [__EFMigrationsHistory]`.
{{< /tab >}}
{{< /code-tabs >}}

## Case 2: a migration tool runs before the service

When production migrates in a separate step, a Kubernetes init container or a
release command, do the same: a start script runs the tool, then becomes the
service.

```sh
#!/bin/sh
# scripts/start.sh
set -e
<the migration command, below>
exec ./orders          # or: node app.js, java -jar app.jar, …
```

```yaml
app:
  command: ./scripts/start.sh
  ready: { type: http, path: /health, timeout: 120s }
  env:
    DATABASE_URL: "postgres://orders:orders@{{.postgres.host}}:{{.postgres.port.5432}}/orders?sslmode=disable"
```

The command, for your tool:

{{< code-tabs >}}
{{< tab "Flyway" "flyway" >}}
```sh
# reads FLYWAY_URL, FLYWAY_USER, FLYWAY_PASSWORD from the environment
flyway -locations=filesystem:db/migration migrate
```
```yaml
env:
  FLYWAY_URL: "jdbc:postgresql://{{.postgres.host}}:{{.postgres.port.5432}}/orders"
  FLYWAY_USER: orders
  FLYWAY_PASSWORD: orders
```
History `flyway_schema_history`: kept. *Not run for this post.*
{{< /tab >}}
{{< tab "Liquibase" "liquibase" >}}
```sh
liquibase update --changelog-file=db/changelog.xml
```
```yaml
env:
  LIQUIBASE_COMMAND_URL: "jdbc:postgresql://{{.postgres.host}}:{{.postgres.port.5432}}/orders"
  LIQUIBASE_COMMAND_USERNAME: orders
  LIQUIBASE_COMMAND_PASSWORD: orders
```
History `databasechangelog`, `databasechangeloglock`: kept. *Not run for this post.*
{{< /tab >}}
{{< tab "golang-migrate" "golang-migrate" >}}
```sh
migrate -path migrations -database "$DATABASE_URL" up
```
History `schema_migrations`: kept. Install: `go install -tags postgres github.com/golang-migrate/migrate/v4/cmd/migrate@latest`. ✓ Run for this post.
{{< /tab >}}
{{< tab "goose" "goose" >}}
```sh
goose -dir migrations postgres "$DATABASE_URL" up
```
History `goose_db_version`: kept (it holds a version-0 row as well as yours). Install: `go install github.com/pressly/goose/v3/cmd/goose@latest`. ✓ Run for this post.
{{< /tab >}}
{{< tab "dbmate" "dbmate" >}}
```sh
npx dbmate --no-dump-schema up      # reads DATABASE_URL
```
History `schema_migrations`: kept. ✓ Run for this post.
{{< /tab >}}
{{< tab "Alembic" "alembic" >}}
```sh
alembic upgrade head
# without a venv: uvx --with psycopg2-binary alembic upgrade head
```
```yaml
orders-db:
  options: { exclude: [alembic_version] }
```
✓ Run for this post.
{{< /tab >}}
{{< tab "Django" "django" >}}
```sh
python manage.py migrate --no-input
```
```yaml
orders-db:
  options: { exclude: [django_migrations] }
```
*Not run for this post.*
{{< /tab >}}
{{< tab "Prisma" "prisma" >}}
```sh
npx prisma migrate deploy           # reads the url from schema.prisma's env()
```
```yaml
orders-db:
  options: { exclude: [_prisma_migrations] }
```
✓ Run for this post.
{{< /tab >}}
{{< tab "Knex" "knex" >}}
```sh
npx knex migrate:latest --knexfile knexfile.cjs
```
```yaml
orders-db:
  options: { exclude: [knex_migrations, knex_migrations_lock] }
```
✓ Run for this post.
{{< /tab >}}
{{< tab "node-pg-migrate" "node-pg-migrate" >}}
```sh
npx node-pg-migrate up --migrations-dir migrations   # reads DATABASE_URL
```
```yaml
orders-db:
  options: { exclude: [pgmigrations] }
```
✓ Run for this post.
{{< /tab >}}
{{< tab "Rails" "rails" >}}
```sh
bin/rails db:prepare
```
```yaml
orders-db:
  options: { exclude: [ar_internal_metadata] }   # schema_migrations is kept already
```
*Not run for this post.*
{{< /tab >}}
{{< tab "sqlx-cli" "sqlx" >}}
```sh
sqlx migrate run                     # reads DATABASE_URL
```
```yaml
orders-db:
  options: { exclude: [_sqlx_migrations] }
```
*Not run for this post.*
{{< /tab >}}
{{< /code-tabs >}}

Three details of the start script:

- **A script, not `sh -c "… && …"`.** `app.command` is split on spaces and
  run without a shell, so `&&` and quotes don't work there.
- **`exec` the service.** It replaces the script, so the SIGTERM tomato sends
  at the end reaches the service, and so does any coverage it writes on exit.
- **`set -e`.** A failed migration stops the script, and tomato reports that
  the service didn't start, with the migration's error in its log.

> **Not yet: the tool in a container of its own.** Running the tool's image
> (`flyway/flyway`, `migrate/migrate`) under `containers:` would mirror a
> Kubernetes Job, but tomato v2.7.0 treats a container that finishes and exits
> as one that failed to start. Install the CLI on the runner and use the
> start script.

## Case 3: plain SQL files

A `schema.sql`, a `pg_dump --schema-only`, Rails' `structure.sql`: the
Postgres image runs every file in `/docker-entrypoint-initdb.d` when it
creates the database, before your service starts.

```yaml
containers:
  postgres:
    image: postgres:16-alpine
    env: { POSTGRES_USER: shop, POSTGRES_PASSWORD: shop, POSTGRES_DB: orders }
    ports: ["5432/tcp"]
    volumes:
      - ${PWD}/initdb:/docker-entrypoint-initdb.d
    wait_for: { type: exec, target: "pg_isready -h 127.0.0.1 -U shop" }
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
container, as above. MySQL and MariaDB images read the same directory.

## Keep the history table

The reset empties every table, and a migration tool's history is a table.
Empty it and the tool forgets what it applied: the next migration run, a
restart in a `lifecycle` scenario say, tries to create tables that exist.

| Tool | History table | In `tomato.yml` |
|---|---|---|
| golang-migrate, dbmate, Rails | `schema_migrations` | nothing: kept |
| goose | `goose_db_version` | nothing: kept |
| Flyway | `flyway_schema_history` | nothing: kept |
| Liquibase | `databasechangelog`, `databasechangeloglock` | nothing: kept |
| Alembic | `alembic_version` | `exclude: [alembic_version]` |
| Django | `django_migrations` | `exclude: [django_migrations]` |
| Prisma | `_prisma_migrations` | `exclude: [_prisma_migrations]` |
| Knex | `knex_migrations`, `knex_migrations_lock` | `exclude: [knex_migrations, knex_migrations_lock]` |
| node-pg-migrate | `pgmigrations` | `exclude: [pgmigrations]` |
| sqlx (Rust) | `_sqlx_migrations` | `exclude: [_sqlx_migrations]` |
| Entity Framework | `__EFMigrationsHistory` | `exclude: [__EFMigrationsHistory]` |

Guard it with one scenario that fails the moment the history is emptied:

```gherkin
Scenario: The migration history survives the reset
  Then "orders-db" table "orders" is empty
  And "orders-db" table "pgmigrations" has "1" rows
```

Without `exclude: [pgmigrations]`, it fails with
`table pgmigrations has 0 rows, expected 1`.
