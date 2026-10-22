+++
title = 'Presets: Redis the way ElastiCache runs it'
date = 2026-10-22T23:30:00+02:00
summary = "A service on ElastiCache with cluster mode and TLS has one Redis client, and a plain Redis container can't answer it. A tomato preset is a container tomato knows how to run: one line in tomato.yml, and tomato does the parts a container definition can't, while it starts the run. The redis preset now runs a TLS cluster."
tags = ['redis', 'elasticache', 'presets', 'black-box testing']
+++

> **New in tomato's next release.** The `redis` preset is in review in
> [tomato#228](https://github.com/tomatool/tomato/pull/228) and ships with
> the release after it merges.

A service on ElastiCache with cluster mode and in-transit encryption usually
has exactly one Redis client, and it looks like this:

```go
rdb := redis.NewClusterClient(&redis.ClusterOptions{
	Addrs:     []string{os.Getenv("REDIS_URL")},
	TLSConfig: &tls.Config{}, // verifies against the system roots
})
```

Point it at `redis:7` in a container and it fails before the first test: the
server speaks plain TCP and isn't a cluster. The usual way out is to change
the service, with a flag that turns TLS off or a plain client when a variable
says "test". Then the tests check a service that never runs in production,
and the bugs only a cluster has ship untested.

tomato's `redis` preset now runs Redis the way ElastiCache does:

```yaml
containers:
  redis:
    preset: redis
    cluster: true   # one node owning every slot, announced on a host port
    tls: true       # TLS only, with a certificate from a CA made for the run
```

## What a preset is

A preset is a container tomato knows how to run. One line expands into the
image, command, ports, wait strategy and files, and into the settings
tomato's own resources need to reach it. Whatever the entry sets itself still
wins.

Presets exist for the part a container definition can't say: what has to be
decided while tomato starts the run. A Redis cluster has three such things:

- **The address it announces.** A cluster client asks the node where each
  slot lives (`CLUSTER SLOTS`) and dials that address. A node left alone
  answers with an empty host and its port inside the container, so go-redis
  dials `:6379` and is refused. The node has to announce the host port the
  client will use, and that port only exists once something picks it, before
  the container starts.
- **Its slots.** A node started in cluster mode owns none. Something has to
  run `CLUSTER ADDSLOTSRANGE 0 16383` after it starts and before the first
  scenario.
- **A certificate** the service trusts, for the name it dials. Without a
  preset, that's a CA and a key committed to the repo.

The `kafka` preset exists for the same reason: a broker advertises the host
port it is reached on, so tomato picks it before the start. Step through a
run:

{{< redis-flow >}}

## The suite

`coupons` redeems a coupon once, and checks several at once. Its only Redis
client is the cluster client above; nothing in it knows about tomato.

```yaml
version: 2

containers:
  redis:
    preset: redis
    cluster: true
    tls: true

app:
  command: ./coupons
  port: 8098
  ready:
    type: http
    path: /health
  env:
    PORT: "8098"
    REDIS_URL: "{{.redis.host}}:{{.redis.port.6379}}"
    SSL_CERT_FILE: "{{.redis.ca_file}}"   # the run's CA, trusted like a system root

resources:
  coupons-client:
    type: http-client
    base_url: http://localhost:8098
  coupons-cache:
    type: redis
    container: redis   # cluster and TLS too, from the preset
```

```gherkin
Feature: Redeeming coupons

  Scenario: A coupon is redeemed once
    When "coupons-client" sends "POST" to "/coupons/A1/redeem"
    Then "coupons-client" response status is "201"
    And "coupons-cache" key "redeemed:A1" exists
    When "coupons-client" sends "POST" to "/coupons/A1/redeem"
    Then "coupons-client" response status is "409"

  Scenario: Several coupons are checked at once
    Given "coupons-cache" key "redeemed:A1" is "1"
    When "coupons-client" sends "GET" to "/coupons?codes=A1,B2"
    Then "coupons-client" response status is "200"
    And "coupons-client" response json matches:
      """
      {"A1": true, "B2": false}
      """
```

```text
$ tomato run
2 scenarios (2 passed)
9 steps (9 passed)
```

The redis steps are the ones you'd use against a single server.
`coupons-cache` connects as a cluster client over TLS, trusting the same CA,
and the reset between scenarios flushes the node.

## What a one-node cluster catches

The second scenario is where this pays off. The lookup's first version
fetched the codes with one `MGET`:

```go
vals, err := rdb.MGet(ctx, "redeemed:A1", "redeemed:B2").Result()
```

A single Redis server answers that. In cluster mode every key in one command
has to hash to the same slot, and these two don't. One node is enough to
enforce the rule, so the suite fails the way ElastiCache would:

```text
Then "coupons-client" response status is "200"
  Error: expected status 200, got 503
  Body: CROSSSLOT Keys in request don't hash to the same slot
```

The fix is one `EXISTS` per code in a pipeline, which go-redis sends to the
node that owns each key's slot. Keys that share a hash tag
(`{coupons}:redeemed:A1`) also work, and keep every coupon on one shard. With
a plain Redis and the client switched for tests, both versions pass.

## Trusting the run's CA

`{{.redis.ca_file}}` is the path of the run's CA on your machine; tomato
removes it when the run ends. `SSL_CERT_FILE` adds it to what the service
trusts, without a change to its code:

- On Linux, Go and the clients built on OpenSSL read it. Node.js needs
  `NODE_EXTRA_CA_CERTS`, and Java a trust store.
- On macOS, Go reads it since Go 1.27, when the service's `go.mod` says
  `go 1.27` or later. With an older `go` line, add
  `GODEBUG: x509sslcertoverrideplatform=1` to `app.env`. A service built with
  Go 1.26 or earlier ignores it on macOS.

## Choosing a preset

Choose by the client your service runs in production, not by the server you
happen to have locally. The options exist so the client's configuration
doesn't change for tests:

| Your service talks to | Preset |
|---|---|
| Redis, one server | `preset: redis` |
| ElastiCache with cluster mode | `preset: redis` and `cluster: true` |
| ElastiCache with in-transit encryption | `preset: redis` and `tls: true` |
| Kafka | `preset: kafka` |
| MSK with IAM | `preset: kafka` and `auth: aws_msk_iam` |
| A web UI, through the `browser` resource | `preset: chromium` |
| PDF/UA or PDF/A checks, through the `pdf` resource | `preset: verapdf` |

Postgres and RabbitMQ have no preset, because an image, a port and a wait is
all they need. A preset earns its place where that isn't enough: where
something has to be picked, generated or run while tomato starts the run. An
entry can still set its own image, env or wait. With `cluster` or `tls`, the
image has to have `redis-server` and `redis-cli`.

## Limits

With `cluster` or `tls`, the app runs as a command: the announced address and
the CA are on your machine, out of reach of an app in a container. A
`network` resource can't stand in front of a cluster, because the client
follows the announced address around the proxy. Containers can't be reused
between runs either, since each run makes a new CA and port.

And it is one node. It checks that the service is a cluster client over TLS
whose commands respect slots. Resharding and failover still need a real
cluster.
