+++
title = 'Databases in a tomato suite: migrations, resets and clean state'
date = 2026-10-21T06:20:00+02:00
layout = 'series'
summary = "tomato empties your databases before every scenario, and there is no switch to turn that off. Why that is the cure for flaky suites, how it fits with migrations, how to keep a table when you must, and how it works for Postgres, ScyllaDB, Redis and the databases tomato has no resource for."
tags = ['databases', 'postgres', 'redis', 'scylladb', 'flakiness']
+++

A flaky suite is almost always a suite with **state that leaks**: one scenario
leaves a row, a counter or a cache entry behind, and another scenario passes
or fails because of it. Run the files in another order, or one scenario on
its own, and the result changes.

tomato's answer is one rule: **every scenario starts from the same state.**
Before each scenario it empties every database it manages. There is no option
to turn that off.

{{< flaky-vs-clean >}}

## A run, table by table

{{< db-flow >}}

Three things happen, and only one of them per scenario:

| When | What | Who |
|---|---|---|
| once, at start | the schema: tables, indexes, reference data | your service's migrations (or a schema file) |
| before every scenario | every table emptied, ids restarted | tomato |
| in each scenario | the rows *this* scenario needs | a `Background`, or your service |

## Where the schema comes from

The reset empties tables; it never drops them. So the schema is made once,
and the best place is **the way production does it**:

1. **Your service migrates at startup.** tomato starts the database, then
   your service, which runs its migrations. The migration code is tested by
   every run. *Recommended.*
2. **A schema file tomato runs.** ScyllaDB's `schema:` option, or an init
   script the database container runs (Postgres reads
   `/docker-entrypoint-initdb.d`). For services that don't migrate themselves.
3. **A `before_all` hook.** `sql_file:` against a resource. It runs *after*
   your service has started, so only for a service that doesn't need the
   tables to boot.

[Migrations](migrations/) shows each with real tools: golang-migrate, Flyway,
Alembic, node-pg-migrate and dbmate, a start script, and init SQL.

## Seeding: a scenario builds its own state

A scenario that needs data declares it, in the scenario or its `Background`,
which runs after the reset:

```gherkin
Background:
  Given "orders-db" table "orders" has values:
    | id | email           | country | total |
    | 42 | ada@example.com | ID      | 700   |
```

Seed the ids you assert on. Ids restart from 1 after every reset in Postgres,
but a scenario asserting `id is "1"` still depends on how many rows it created
first; `id is "42"` after seeding 42 does not.

## Keeping a table (when you must)

Some data shouldn't be emptied:

- **Migration history.** Empty it and the next start re-runs every migration.
  tomato **keeps it automatically** for golang-migrate, Rails and sqlx
  (`schema_migrations`), goose, Flyway and Liquibase.
- **Reference data a migration wrote**: countries, currencies, roles. Without
  it, every scenario would have to seed it again.

Exclude those, and only those:

```yaml
orders-db:
  type: postgres
  container: postgres
  options:
    exclude: [countries]
```

It's an escape hatch, not a pattern:

- **Never exclude a table a scenario writes to.** What one scenario adds, the
  next one sees: the flakiness the reset exists to prevent.
- **A migration tool tomato doesn't know** (Alembic, Django, Prisma, Knex,
  node-pg-migrate, sqlx…) needs its history table in `exclude`, or the history
  is emptied after the first scenario: [the table](migrations/#keep-the-history-table).
- **Prefer a seed over an exclude.** Data a scenario needs belongs in its
  `Background`, where a reader can see it.

{{< parts >}}

## The checklist

- The schema comes from your service's own migrations.
- Each scenario seeds what it needs, and asserts on what it seeded.
- `exclude` holds the migration history (if tomato doesn't already) and
  read-only reference data. Nothing a scenario writes.
- A database tomato has no resource for gets a `before_scenario` hook that
  empties it.
- Running any one scenario alone gives the same result as running the suite.
