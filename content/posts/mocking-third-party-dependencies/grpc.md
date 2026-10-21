+++
title = 'Mocking a gRPC service'
linkTitle = 'gRPC'
weight = 20
tool = 'grpc-server'
date = 2026-10-21T05:30:00+02:00
summary = "Give grpc-server your .proto, compiled to a descriptor set, and stub methods in JSON: answers, status codes, delays, streams, and the metadata your service sent."
tags = ['stubs', 'grpc']
+++

{{< mock-flow kind="grpc" >}}

## What it needs: the schema

A gRPC stub has to know the messages to answer with. Give it your `.proto`
files compiled to a **descriptor set**:

```bash
buf build -o proto/users.binpb
# or
protoc --include_imports --descriptor_set_out=proto/users.binpb -I proto proto/users/v1/users.proto
```

With that, you write stubs and assertions in **JSON**, and a typo in a method
or field fails at the step that has it.

## The suite

```yaml
app:
  command: node app.js
  env:
    USERS_ADDR: "{{.users.address}}"   # localhost:9102

resources:
  api:   { type: http, base_url: http://localhost:3000 }
  users: { type: grpc-server, options: { port: 9102, descriptor_set: proto/users.binpb } }
```

```gherkin
Scenario: A known user
  Given "users" stub "users.v1.UserService/GetUser" with request json "id" "42" returns json:
    """
    {"id": "42", "name": "Ada", "plan": "pro"}
    """
  When "api" sends "GET" to "/users/42"
  Then "api" response json "name" is "Ada"
  And "users" last "users.v1.UserService/GetUser" request metadata "authorization" is "Bearer secret"

Scenario: An unknown user
  Given "users" stub "users.v1.UserService/GetUser" returns status "NOT_FOUND" with message "no user 7"
  When "api" sends "GET" to "/users/7"
  Then "api" response status is "404"
```

## What it can do

| To | Step |
|---|---|
| answer | `stub "users.v1.UserService/GetUser" returns json:` |
| answer only some calls | `… with request json "id" "42" returns …`, `… with metadata "authorization" "Bearer x" returns …` |
| fail | `returns status "NOT_FOUND" with message "…"` (any gRPC status) |
| be slow, or vary | `returns in sequence:` a table of `json`, `status`, `message`, `delay` |
| stream | `streams:` one message per row, for server-streaming methods |
| be down | `refuses connections`, then `accepts connections` |
| check what arrived | `received "…/GetUser" "2" times`, `last "…/GetUser" request json contains:` |

## Notes

- **The most specific stub wins**, then the latest, so a scenario can override
  what its `Background` stubbed.
- **A delay respects the caller's deadline**: give the call one second and it
  gets `DEADLINE_EXCEEDED` after one second.
- **After `refuses connections`**, a gRPC client backs off before it
  reconnects. A service that retries `UNAVAILABLE` with a short backoff
  recovers at once; one with the library defaults may not, which is worth a
  scenario of its own.
- **Unary and server-streaming only** for now; client and bidirectional
  streams can't be stubbed yet.
