+++
title = 'Databases tomato has no resource for'
linkTitle = 'Other databases'
weight = 40
tool = 'hooks · exec'
date = 2026-10-21T06:20:00+02:00
summary = "MongoDB, MySQL, Elasticsearch: tomato has no resource for them, but the rule still holds. A before_scenario hook empties them, and scenarios check them through your service."
tags = ['databases', 'mongodb', 'mysql', 'hooks']
+++

tomato has resources for Postgres, ScyllaDB, Cassandra and Redis. For any
other database the rule is the same, **every scenario starts clean**, and you
enforce it with a hook.

## A `before_scenario` hook

It runs after tomato's own resets and before the `Background`, inside the
database's container:

```yaml
containers:
  mongo:
    image: mongo:7
    ports: ["27017/tcp"]
    wait_for: { type: log, target: "Waiting for connections" }

hooks:
  before_scenario:
    - exec: mongosh --quiet --eval 'db.getSiblingDB("orders").dropDatabase()'
      container: mongo
```

```gherkin
Scenario: An order is audited
  When "orders-client" sends "POST" to "/orders" with json:
    """
    {"email": "ada@example.com", "country": "DE", "total": 1}
    """
  And "orders-client" sends "GET" to "/audit"
  Then "orders-client" response json "events" is "1"

Scenario: The audit log starts empty
  When "orders-client" sends "GET" to "/audit"
  Then "orders-client" response json "events" is "0"
```

The same pattern for others (the MongoDB hook above is the one run for this post; adapt these to your names):

| Database | `exec` in its container |
|---|---|
| MongoDB | `mongosh --quiet --eval 'db.getSiblingDB("app").dropDatabase()'` |
| MySQL, MariaDB | `mysql -uroot -p$MYSQL_ROOT_PASSWORD app -e "SET FOREIGN_KEY_CHECKS=0; TRUNCATE orders; …"` |
| Elasticsearch, OpenSearch | `curl -s -XDELETE 'localhost:9200/orders-*'` |

## Notes

- **Drop what scenarios write, not the schema the service made.** Dropping a
  Mongo database is fine (collections come back on the next insert); in MySQL,
  truncate tables rather than dropping them.
- **Check through your service**, or with a `shell` resource running the
  database's CLI, since there are no steps for these databases.
- **Use `exec`, with `container:`.** The hook runs inside that container, so
  its CLI is there.
