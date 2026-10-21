+++
title = 'Driving your service over HTTP'
linkTitle = 'HTTP'
weight = 10
tool = 'http'
date = 2026-10-21T06:00:00+02:00
summary = "The http client sends requests and checks status, headers and JSON. Save a value from one response, use it in the next request."
tags = ['clients', 'http']
+++

{{< client-flow kind="http" >}}

## The suite

```yaml
app:
  command: node app.js
  port: 3000

resources:
  todos-client: { type: http, base_url: http://localhost:3000 }
```

```gherkin
Scenario: A todo is added
  When "todos-client" sends "POST" to "/api/todos" with json:
    """
    {"title": "Buy milk"}
    """
  Then "todos-client" response status is "201"
  And "todos-client" response json contains:
    """
    {"title": "Buy milk", "done": false}
    """
  And "todos-client" response json "id" saved as "{{id}}"
  When "todos-client" sends "GET" to "/api/todos/{{id}}"
  Then "todos-client" response json "title" is "Buy milk"

Scenario: A todo needs a title
  When "todos-client" sends "POST" to "/api/todos" with json:
    """
    {"title": "  "}
    """
  Then "todos-client" response status is "400"
  And "todos-client" response json "error" is "title is required"
```

## What it can do

| To | Step |
|---|---|
| set up the request | `header "Authorization" is "Bearer …"`, `query param "page" is "2"`, `cookie "session" is "…"` |
| send | `sends "GET" to "/path"`, `… with json:`, `… with body:` |
| check the status | `response status is "201"`, or `is client error` |
| check JSON | `response json "a.b[0]" is "x"`, `response json contains:` (a subset), `response json matches:` (all of it) |
| match loosely | `"@notempty"`, `"@regex:^ch_"`, `"@gt:10"` inside the JSON, or `response json "id" is uuid` |
| carry a value forward | `response json "id" saved as "{{id}}"`, then use `{{id}}` in any later step |
| read a stream | `opens stream "/events"`, `receives event "tick" within "2s"` (Server-Sent Events) |
| load it | `sends "20" concurrent "GET" to "/path"`, `"20" of the concurrent responses have status "200"` |

## Notes

- **`contains` is a subset, `matches` is exact.** Use `contains` for the
  fields the scenario is about, so a new field doesn't break every scenario.
- **Headers set with `header` stay** for the rest of the scenario, and are
  cleared before the next.
- **Point it at anything**: another service in a container with
  `container:`, or the address of a service tomato doesn't start.
