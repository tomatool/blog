+++
title = 'Mocking third-party dependencies'
date = 2026-10-21T05:30:00+02:00
layout = 'series'
summary = "A stub stands in for a service you don't run: it answers the way you tell it to, fails when you ask it to, and remembers what your service sent. How tomato's HTTP, gRPC and WebSocket stubs work, in one picture."
tags = ['stubs', 'mocking', 'black-box testing']
+++

Your service calls things you don't run: a payment provider, another team's
gRPC service, a price feed. In a test you can't make them fail on purpose, and
you can't ask them what your service sent. So tomato starts a **stub** in their
place.

## How it works

{{< mock-flow >}}

Four moves, the same for every protocol:

1. **Point.** The stub listens on a port you choose. The service gets that
   address in the same variable that holds the real one in production.
2. **Arrange.** A `Given` step tells the stub what to answer.
3. **Assert.** `Then` steps check what your service sent the stub.
4. **Reset.** Before every scenario the stub forgets everything.

## Which stub

| The service calls | Stub | Point it with | Arrange | Assert |
|---|---|---|---|---|
| a REST API | [`http-server`](http/) | `{{.name.url}}` | `stub "POST" "/path" returns "201"` | `received "POST" "/path"` |
| a gRPC service | [`grpc-server`](grpc/) | `{{.name.address}}` | `stub "pkg.Svc/Method" returns json:` | `received "pkg.Svc/Method"` |
| a WebSocket feed | [`websocket-server`](websocket/) | `ws://localhost:<port>` | `on message "…" replies:` | `received message "…"` |

{{< parts >}}

## Stub, not the real thing

A stub checks **your** side of the contract: what your service sends, and
what it does with each answer. It can't tell you the third party still
answers that way. Keep a few smoke tests against its sandbox for that, and let
the stub cover the cases a sandbox won't give you on demand: the decline, the
timeout, the outage.
