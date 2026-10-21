+++
title = 'Testing what your service does when the network fails'
date = 2026-10-22T01:00:00+02:00
summary = "Databases fail over, caches hang, brokers restart. A test suite on localhost never sees any of it, so the timeouts, retries and fallbacks written for those moments never run until production. tomato's network resource puts a proxy between your service and its dependencies and breaks the link on purpose, one scenario at a time."
tags = ['network', 'resilience', 'black-box testing']
+++

> **New in tomato's next release.** The `network` resource is merged on
> `main` and ships with the next release. Until then, build tomato from
> `main` to try it.

Every service has code for a bad network: a query timeout, a retry loop, a
cache that's allowed to fail. Almost none of it runs in tests. On
`localhost` the database answers in a millisecond, so the timeout never
fires, the retry never retries, and the fallback never falls back. The first
real run is the night the database fails over.

tomato's `network` resource changes that. It sits between your service and a
dependency's container, and a step breaks the link: slow it down, hang it,
cut it, take it down. Then you check what the service did. Pick a fault:

{{< network-lab >}}

## How it fits in

Add one `network` resource per dependency you want to break:

```yaml
resources:
  orders-db:
    type: postgres
    container: postgres
    database: orders
    options: { user: orders, password: orders }

  postgres-network:               # the app's link to Postgres
    type: network
    container: postgres
  redis-network:                  # and to Redis
    type: network
    container: redis
```

Nothing else changes. The service's `DATABASE_URL` still uses
`{{.postgres.port.5432}}`, which now resolves to the proxy. tomato prints
where each one listens:

```text
  ⇄ postgres-network  app → 127.0.0.1:61442 → postgres:5432  (seed 98866)
  ⇄ redis-network     app → 127.0.0.1:61443 → redis:6379     (seed 253200)
```

Three things make it work for testing:

- **Only the service goes through it.** tomato's own resources, like
  `orders-db`, connect to the container directly. While the service's link
  is down, a step can still read the table and see what really happened.
- **It speaks TCP, not a protocol.** The same steps work for Postgres, Redis,
  Kafka, RabbitMQ, MinIO, ScyllaDB, or anything else in a container. With
  the `kafka` preset, tomato even makes the broker advertise the proxy, so
  the client can't route around it.
- **Every scenario starts healthy.** Before each scenario tomato turns every
  fault off and resets its counts. A fault can't leak into the next test.

## The service

`orders` is a small Node.js service with Postgres and a Redis cache in front
of product reads. These are the lines that decide how it behaves on a bad
network:

```js
const db = new pg.Pool({
  connectionString: process.env.DATABASE_URL,
  connectionTimeoutMillis: 1000,      // a hung database costs a second, not forever
  query_timeout: 1000,
});

const cache = createClient({
  url: process.env.REDIS_URL,
  disableOfflineQueue: true,          // fail now instead of queueing while Redis is down
  socket: { connectTimeout: 500, reconnectStrategy: () => 200 },
});

// A query that fails on the connection is tried again; a timeout is not.
async function query(sql, args) {
  for (let attempt = 1; ; attempt++) {
    try {
      return await db.query(sql, args);
    } catch (e) {
      const transient = ["ECONNREFUSED", "ECONNRESET", "57P01"].includes(e.code)
        || /Connection terminated/.test(e.message);
      if (!transient || attempt === 5) throw e;
      await sleep(250 * attempt);
    }
  }
}

// The cache is an optimisation, not a dependency: 200 ms, then Postgres.
async function product(id) {
  try {
    const hit = await within(200, cache.get(`product:${id}`));
    if (hit) return JSON.parse(hit);
  } catch {}
  return (await query("SELECT id, name, price FROM products WHERE id = $1", [id])).rows[0];
}

// The order id makes a retry safe.
await query("INSERT INTO orders (id, product, qty) VALUES ($1, $2, $3) ON CONFLICT (id) DO NOTHING", …);
```

Any error ends in `503 {"error": "try again shortly"}`.

## The scenarios

Each scenario sets up one fault and checks one behaviour:

```gherkin
Feature: Placing orders when the database misbehaves

  Scenario: A database failover doesn't lose the order
    Given "postgres-network" resets connections for "2s"
    When "orders-client" sends "POST" to "/orders" with json:
      """
      {"id": "ord-1", "product": "tomato-seeds", "qty": 2}
      """
    Then "orders-client" response status is "201"
    And "orders-db" table "orders" has "1" rows
    And "postgres-network" has seen "4" connections

  Scenario: A hung database gets a quick answer, not a hang
    Given "postgres-network" stops responding
    When "orders-client" sends "POST" to "/orders" with json:
      """
      {"id": "ord-1", "product": "tomato-seeds", "qty": 2}
      """
    Then "orders-client" response status is "503"
    And "orders-client" response time is less than "2s"
```

```gherkin
Feature: Reading products when the cache or the database is slow

  Background:
    Given "orders-db" table "products" has values:
      | id           | name         | price |
      | tomato-seeds | Tomato seeds | 299   |

  Scenario: A cached product doesn't wait for a slow database
    Given "orders-cache" key "product:tomato-seeds" is:
      """
      {"id": "tomato-seeds", "name": "Tomato seeds", "price": 299}
      """
    And "postgres-network" adds latency "2s"
    When "orders-client" sends "GET" to "/products/tomato-seeds"
    Then "orders-client" response status is "200"
    And "orders-client" response time is less than "300ms"

  Scenario: Products are still served while the cache is down
    Given "redis-network" refuses connections
    When "orders-client" sends "GET" to "/products/tomato-seeds"
    Then "orders-client" response status is "200"
    And "orders-client" response json contains:
      """
      {"name": "Tomato seeds"}
      """

  Scenario: A hung cache costs 200ms, not the request
    Given "redis-network" stops responding
    When "orders-client" sends "GET" to "/products/tomato-seeds"
    Then "orders-client" response status is "200"
    And "orders-client" response time is less than "1s"
```

```text
$ tomato run
6 scenarios (6 passed)
32 steps (32 passed)
11.2s
```

`has seen "4" connections` is how a scenario proves a retry happened. During
the 2-second reset, the pool's connection was cut and the service opened
four new ones: three were reset, and the fourth, after the network healed,
went through. A service that didn't retry would have opened none.

## The same suite, without the safeguards

The point of these scenarios is the bugs they catch. Here is the same suite
against the same service with its timeouts, retries and fallback taken out,
which is the default behaviour of most client libraries. Five of six fail:

| Scenario | Fault | Without the safeguard |
|---|---|---|
| A database failover doesn't lose the order | `resets connections for "2s"` | `expected status 201, got 503`: one cut connection, and the order is lost |
| A hung database gets a quick answer | `stops responding` | no answer within the client's 10 s: the request hangs, and so does its caller |
| Products are still served while the cache is down | `refuses connections` on Redis | hangs: the Redis client queues commands while offline, so a down cache becomes a hung request |
| A hung cache costs 200ms | `stops responding` on Redis | hangs, for the same reason |
| An order retried after a timeout is stored once | `stops responding`, then `is healthy` | hangs; and with timeouts but no duplicate protection, the retry fails (below) |

The fifth row isn't in the scenarios above, because it needs its own section.

## The timeout that lied

`stops responding` lets queries through and swallows the replies. The first
time this suite ran, the hung-database scenario also checked that no order
was stored, and it failed:

```text
table orders has 1 rows, expected 0
while the network was faulty:
  postgres-network stops responding (step 1)
```

The service answered 503, but the order was in the table. Step through it:

{{< network-timeout >}}

A timeout means *I don't know*, not *it didn't happen*. The caller was told
to try again, so the honest test is to do exactly that:

```gherkin
  Scenario: An order retried after a timeout is stored once
    Given "postgres-network" stops responding
    And "orders-client" sends "POST" to "/orders" with json:
      """
      {"id": "ord-1", "product": "tomato-seeds", "qty": 2}
      """
    And "orders-client" response status is "503"
    When "postgres-network" is healthy
    And "orders-client" sends "POST" to "/orders" with json:
      """
      {"id": "ord-1", "product": "tomato-seeds", "qty": 2}
      """
    Then "orders-client" response status is "201"
    And "orders-db" table "orders" has "1" rows
```

Without `ON CONFLICT (id) DO NOTHING`, the retry hits the primary key, and
the customer gets an error for an order that exists:

```text
expected status 201, got 503
Body: {"error":"try again shortly"}
```

Without a primary key, they get two orders. Neither shows up on a healthy
network.

## Every fault

| Step | Stands in for | Check that the service… |
|---|---|---|
| `adds latency "300ms" with jitter "100ms"` | a slow or distant dependency | stays within its budget; caches help |
| `limits bandwidth to "64KB/s"` | a congested link | doesn't give up on a slow but healthy transfer |
| `stops responding` | a hung dependency, a partition | times out, and keeps retries safe |
| `resets connections` | a crash, a failover | retries on a new connection |
| `refuses connections` | the dependency is down | fails fast, or degrades |
| `flaps every "2s"` | a restart loop | backs off, and recovers when it ends |
| `fails the next "2" connections` | a dependency coming back up | retries enough times |
| `fails "30%" of new connections` | an unreliable link | survives; the seed makes a failure repeatable |
| `loses "30%" of the data` | a corrupting link | errors out instead of hanging |
| `is healthy` | the fix | recovers without a restart |

Each fault lasts until the scenario ends, or `for "<duration>"`, after which
it heals by itself, which is what retry code has to survive. Two steps check
what the service did about it: `has seen "3" connections` counts the
connections it opened (a retry or a reconnect is a new one), and
`sees a new connection within "10s"` waits for a consumer or a worker to
reconnect after a fault.

## When a scenario fails

A step that fails while a fault is on says so, in every report format:

```text
expected status 201, got 503
Body: {"error":"try again shortly"}
while the network was faulty:
  postgres-network resets connections (step 1, 2s left)
  postgres-network has seen 1 connection this scenario, 0 failed
```

`tomato ui` marks every step that ran on a faulty network, and the step during
which a fault was applied or healed.

## Limits

- **The service runs as a process** (`app.command`). A service in a container
  reaches its dependencies over the Docker network, past the proxy, so
  tomato refuses that combination.
- **TCP, blind to the protocol.** A fault hits every connection to the
  container, so it can't fail only the `INSERT`s. For an HTTP or gRPC
  dependency, [stub it]({{< relref "/posts/mocking-third-party-dependencies" >}})
  and fail single requests there.
- **One network resource per container.**

The full reference is in the
[network configuration](https://tomatool.github.io/tomato/configuration/network/)
docs.
