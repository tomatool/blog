+++
title = 'Driving your service: HTTP, gRPC, a browser and a shell'
date = 2026-10-21T06:00:00+02:00
layout = 'series'
summary = "Clients are how a tomato scenario uses your service: through its HTTP API, its gRPC API, its web page in a real browser, or its command line. One idea, four doors."
tags = ['clients', 'black-box testing']
+++

A scenario never calls your code. It uses your service the way a person or
another system does, through a **client**: an HTTP request, a gRPC call, a
click in a browser, a command in a shell. The client is the scenario's hands;
the `Then` steps are its eyes.

## How it works

{{< client-flow >}}

The same loop for every client:

1. **Act.** A `When` step makes the client do one thing.
2. **Answer.** The service answers through the same door.
3. **Assert.** `Then` steps check the answer.
4. **Reset.** The client forgets everything before the next scenario.

## Which client

| Your service has | Client | Acts with | Checks |
|---|---|---|---|
| an HTTP API | [`http`](http/) | `sends "POST" to "/api/todos" with json:` | status, headers, JSON |
| a gRPC API | [`grpc`](grpc/) | `calls "todos.v1.TodoService/AddTodo" with:` | status code, JSON of the message |
| a web page | [`browser`](browser/) | `fills "New todo" with "…"`, `clicks "Add"` | what the page shows |
| a command line | [`shell`](shell/) | `runs "todo add Buy milk"` | exit code, stdout, stderr, files |

{{< parts >}}

## The answer is half of it

A client sees what comes back through its door. What the service *did* lives
elsewhere: the row it wrote, the event it published, the call it made to a
third party. Check those with the resources that see them (`postgres`,
`kafka`, the [stubs](../mocking-third-party-dependencies/)), so a scenario
proves the effect, not only the reply.
