+++
title = 'Postgres in a tomato suite'
linkTitle = 'PostgreSQL'
weight = 10
tool = 'postgres'
date = 2026-10-21T06:20:00+02:00
summary = "Every table emptied in one TRUNCATE … RESTART IDENTITY CASCADE before each scenario, migration history kept, schemas and excludes for the rest."
tags = ['databases', 'postgres']
+++

## The suite

The service migrates at startup and records versions in `schema_migrations`.
Its first migration writes reference data, `countries`.

```yaml
containers:
  postgres:
    image: postgres:16-alpine
    env: { POSTGRES_USER: orders, POSTGRES_PASSWORD: orders, POSTGRES_DB: orders }
    ports: ["5432/tcp"]
    wait_for: { type: exec, target: "pg_isready -h 127.0.0.1 -U orders" }

app:
  command: node app.js          # runs its migrations, then listens
  env:
    DATABASE_URL: "postgres://orders:orders@{{.postgres.host}}:{{.postgres.port.5432}}/orders"

resources:
  orders-client: { type: http, base_url: http://localhost:3000 }
  orders-db:
    type: postgres
    container: postgres
    database: orders
    options:
      user: orders
      password: orders
      exclude: [countries]       # reference data the migration wrote
```

```gherkin
Scenario: An order is stored
  When "orders-client" sends "POST" to "/orders" with json:
    """
    {"email": "ada@example.com", "country": "DE", "total": 1999}
    """
  Then "orders-client" response status is "201"
  And "orders-db" table "orders" contains:
    | email           | country | total |
    | ada@example.com | DE      | 1999  |

Scenario: Every scenario starts with no orders, and ids from 1
  Then "orders-db" table "orders" is empty
  When "orders-client" sends "POST" to "/orders" with json:
    """
    {"email": "bob@example.com", "country": "NL", "total": 500}
    """
  Then "orders-client" response json "id" is "1"

Scenario: Reference data and migration history survive the reset
  Then "orders-db" table "countries" has "3" rows
  And "orders-db" table "schema_migrations" has "2" rows
```

## Wait for the real server

A new Postgres container starts a temporary server first, to create the
database and run init scripts, then restarts as the real one. Both print
`database system is ready to accept connections`, so a log wait can let your
service connect into the restart and fail with `EOF`. The temporary server
only listens on a local socket, so check over TCP:

```yaml
wait_for: { type: exec, target: "pg_isready -h 127.0.0.1 -U orders" }
```

## What the reset does

Before every scenario, one statement:

```sql
TRUNCATE <every table> RESTART IDENTITY CASCADE
```

- **Every table** in `public`, found from the catalog at each reset, so a
  table a new migration adds is reset without a config change.
- **`RESTART IDENTITY`**: serial and identity columns start from 1 again.
- **`CASCADE`**: foreign keys don't get in the way.
- **Not truncated**: migration history (`schema_migrations`,
  `goose_db_version`, `flyway_schema_history`, `databasechangelog`,
  `databasechangeloglock`) and what you `exclude`.

## Choosing what is reset

| Option | Resets |
|---|---|
| *(none)* | every table in `public` |
| `schemas: [billing, "plugin_*"]` | every table in those schemas (a `*` is a pattern) |
| `tables: [audit.events]` | with `schemas`: these as well; alone: *only* these |
| `exclude: [countries, billing.fx_rates]` | never these |

`schemas` suits a service with a schema per module: the suite doesn't list
every table, and doesn't drift from the migrations.

## Steps worth knowing

| To | Step |
|---|---|
| seed | `table "orders" has values:` |
| check rows | `table "orders" contains:`, `has "2" rows`, `is empty` |
| query | `query "SELECT …" returns:` |
| wait for async writes | `query "SELECT …" returns within "10s":` (a consumer, a job) |
| run SQL | `executes:`, `executes file "fixtures/x.sql"` |
