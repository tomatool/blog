+++
title = 'Driving your service over gRPC'
linkTitle = 'gRPC'
weight = 20
tool = 'grpc'
date = 2026-10-21T06:00:00+02:00
summary = "The grpc client calls methods by name and writes messages as JSON. It reads the schema from the service itself, through server reflection, so no .proto sits next to the tests."
tags = ['clients', 'grpc']
+++

{{< client-flow kind="grpc" >}}

## What it needs: reflection

The client asks your service for its own schema, through **gRPC server
reflection**, so you name a method the way you'd say it, and write messages as
JSON. One line in the service turns it on:

```go
reflection.Register(server)                          // Go
```
```js
new ReflectionService(packageDefinition).addToServer(server)   // Node.js, @grpc/reflection
```

## The suite

```yaml
resources:
  grpc: { type: grpc, address: localhost:50051 }
```

```gherkin
Scenario: The service describes itself
  Then "grpc" exposes service "todos.v1.TodoService"
  And "grpc" service "todos.v1.TodoService" has method "AddTodo"

Scenario: A todo is added
  When "grpc" calls "todos.v1.TodoService/AddTodo" with:
    """
    {"title": "Buy milk"}
    """
  Then "grpc" call succeeds
  And "grpc" response json "title" is "Buy milk"
  And "grpc" response json "done" is "false"

Scenario: A todo needs a title
  When "grpc" calls "todos.v1.TodoService/AddTodo" with:
    """
    {"title": ""}
    """
  Then "grpc" call fails
  And "grpc" response status is "INVALID_ARGUMENT"
  And "grpc" response error contains "title is required"
```

## What it can do

| To | Step |
|---|---|
| send metadata | `metadata "authorization" is "Bearer …"` |
| call | `calls "pkg.Service/Method"`, `… with:` a JSON message |
| check the outcome | `call succeeds`, `call fails`, `response status is "NOT_FOUND"`, `response error contains "…"` |
| check the message | every JSON check the http client has: `response json "a.b" is`, `contains:`, `matches:` |
| check the schema | `exposes service "…"`, `service "…" has method "…"` |

## Notes

- **A failed call is a result, not a broken step.** `INVALID_ARGUMENT` is
  something to assert. Only a call that can't be made (an unknown method, a
  message that doesn't fit) fails the step.
- **Zero values are in the JSON.** `"done": false` is there to assert, though
  protobuf doesn't send it.
- **Enums are names, `int64` is a string.** `"ROLE_ADMIN"`, `"priceCents": "250"`.
- **Unary calls only.** Streaming calls need steps of their own; they're
  refused with a clear message for now.
