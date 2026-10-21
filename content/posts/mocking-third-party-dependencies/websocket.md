+++
title = 'Mocking a WebSocket feed'
linkTitle = 'WebSocket'
weight = 30
tool = 'websocket-server'
date = 2026-10-21T05:30:00+02:00
summary = "websocket-server stands in for a feed your service listens to: it greets, replies to messages, pushes updates, and remembers what your service sent. And it hangs up between scenarios, so reconnecting gets tested too."
tags = ['stubs', 'websocket']
+++

{{< mock-flow kind="ws" >}}

## The suite

The service here connects to a price feed when it starts, sends
`subscribe BTC`, and keeps the latest price.

```yaml
app:
  command: node app.js
  env:
    PRICES_URL: ws://localhost:9103/stream   # any path is accepted

resources:
  shop-client:    { type: http, base_url: http://localhost:3000 }
  prices-stub: { type: websocket-server, options: { port: 9103 } }
```

```gherkin
Background:
  Given "prices-stub" on message "subscribe BTC" replies:
    """
    {"symbol": "BTC", "price": 64000}
    """

Scenario: The service subscribes when it connects
  Then "prices-stub" has "1" connections
  And "prices-stub" received message "subscribe BTC"

Scenario: The service keeps the latest price the feed pushes
  Given "prices-stub" has "1" connections
  When "prices-stub" broadcasts:
    """
    {"symbol": "BTC", "price": 65000}
    """
  And "shop-client" sends "GET" to "/prices/BTC"
  Then "shop-client" response json "price" is "65000"
```

## What it can do

| To | Step |
|---|---|
| greet a new connection | `on connect sends:` |
| reply to a message | `on message "subscribe BTC" replies:`, or `on message matching "subscribe .*" replies:` |
| push to everyone connected | `broadcasts:` |
| check the connection | `has "1" connections` |
| check what arrived | `received message "subscribe BTC"`, `received "3" messages` |

## The one thing to know: it hangs up

Before every scenario the stub **closes every open connection**, as well as
forgetting its replies and messages. A service that keeps one long-lived
connection must reconnect, and it does so a moment after the scenario starts.
So:

- **Wait for the connection before pushing.** `Given "prices-stub" has "1"
  connections` waits until the service is back. Broadcasting before that
  fails with "no clients are connected".
- **That's a feature.** Real feeds drop connections too. Every scenario
  starts with your service reconnecting and resubscribing, so a service that
  can't is caught on the first scenario, not in production.

## Notes

- **No address template yet**: name the port in `options.port` and write the
  same address in `app.env`.
- **Messages match exactly**, or by regular expression with `matching`.
- **Text and JSON both work**; JSON is compared as text, so match the exact
  bytes your service sends.
