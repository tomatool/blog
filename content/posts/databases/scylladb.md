+++
title = 'ScyllaDB and Cassandra in a tomato suite'
linkTitle = 'ScyllaDB, Cassandra'
weight = 20
tool = 'scylladb'
date = 2026-10-21T06:20:00+02:00
summary = "tomato creates the keyspace, runs your CQL schema once, and truncates every table in the keyspace before each scenario, except the ones you exclude."
tags = ['databases', 'scylladb', 'cassandra']
+++

## The suite

```yaml
containers:
  scylla:
    image: scylladb/scylla:6.2
    command: ["--smp", "1", "--memory", "512M", "--overprovisioned", "1", "--developer-mode", "1"]
    ports: ["9042/tcp"]
    wait_for: { type: port, target: "9042", timeout: 120s }

resources:
  shop-db:
    type: scylladb               # or: cassandra
    container: scylla
    options:
      keyspace: shop             # created if missing
      schema: [schema/001_tables.cql]
      exclude: [countries]
```

```sql
-- schema/001_tables.cql, run once before the first scenario
CREATE TABLE IF NOT EXISTS shop.orders_by_customer (
    customer text, order_id int, total int,
    PRIMARY KEY (customer, order_id)
);
CREATE TABLE IF NOT EXISTS shop.countries (code text PRIMARY KEY, name text);
INSERT INTO shop.countries (code, name) VALUES ('DE', 'Germany');
INSERT INTO shop.countries (code, name) VALUES ('NL', 'Netherlands');
```

```gherkin
Scenario: Seed, then read back by partition
  Given "shop-db" table "orders_by_customer" has values:
    | customer | order_id | total |
    | ada      | 1        | 1999  |
    | ada      | 2        | 500   |
  Then "shop-db" query "SELECT order_id, total FROM orders_by_customer WHERE customer = 'ada'" returns:
    | order_id | total |
    | 1        | 1999  |
    | 2        | 500   |

Scenario: The next scenario starts empty
  Then "shop-db" table "orders_by_customer" is empty

Scenario: Excluded reference data is kept
  Then "shop-db" table "countries" has "2" rows
```

## Notes

- **The schema can come from your service** instead: leave out `schema` and
  let its migrations create the tables, as with Postgres.
- **Order is per partition.** CQL has no global order, so `table … contains:`
  matches in any order; `query … returns:` is ordered, so query by partition
  key when order matters.
- **Several keyspaces?** `keyspaces: [shop, billing]` resets them all.
- **It's slow to start.** The flags above keep ScyllaDB small enough for a CI
  runner; the resource retries until CQL answers (`ready_timeout`, 90s).
