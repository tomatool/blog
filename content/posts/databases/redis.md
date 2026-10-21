+++
title = 'Redis in a tomato suite'
linkTitle = 'Redis'
weight = 30
tool = 'redis'
date = 2026-10-21T06:20:00+02:00
summary = "Redis holds the state that leaks most easily: counters, sessions, caches. tomato flushes it before every scenario, or deletes only the keys matching a pattern."
tags = ['databases', 'redis', 'flakiness']
+++

Redis is where the sneakiest leaks live: a rate-limit counter, a session, a
cached response. They have a TTL, so a suite run slowly passes and the same
suite run quickly fails. tomato flushes Redis before every scenario.

## The suite

The service allows three orders per email per minute, counted in Redis.

```yaml
containers:
  redis:
    image: redis:7-alpine
    ports: ["6379/tcp"]
    wait_for: { type: port, target: "6379" }

resources:
  orders-cache: { type: redis, container: redis }
```

```gherkin
Scenario: The fourth order in a minute is refused
  When "orders-client" sends "POST" to "/orders" with json: …   # three times
  And "orders-client" sends "POST" to "/orders" with json: …
  Then "orders-client" response status is "429"
  And "orders-cache" key "ratelimit:ada@example.com" has value "4"

Scenario: The counter does not leak into the next scenario
  Then "orders-cache" key "ratelimit:ada@example.com" does not exist
  When "orders-client" sends "POST" to "/orders" with json: …
  Then "orders-client" response status is "201"
```

Without the reset, the second scenario fails whenever it runs within a minute
of the first: [the flaky pair](../) at the top of the overview.

## Choosing what is reset

| Option | Resets |
|---|---|
| `reset_strategy: flush` *(default)* | every key in the database (`db`, 0 by default) |
| `reset_strategy: pattern`, `reset_pattern: "app:*"` | only keys matching the pattern |

`pattern` is the Redis way to keep something: put the keys scenarios write
under a prefix, reset that prefix, and leave the rest. As with tables, keep
only what no scenario writes.

## Steps worth knowing

| To | Step |
|---|---|
| seed | `key "k" is "v"`, `key "k" is "v" with TTL "60s"`, `hash "h" has fields:` |
| check | `key "k" has value "v"`, `key "k" exists`, `does not exist`, `has TTL greater than "30" seconds` |
| structures | `hash "h" field "f" is "v"`, `list "l" has "3" items`, `set "s" has "x"` |
