+++
title = 'Mocking an HTTP API, and how WireMock does it'
linkTitle = 'HTTP'
weight = 10
tool = 'http-server'
date = 2026-10-21T05:30:00+02:00
summary = "tomato's http-server works like WireMock: stub a request, let the service call it, check the journal of what arrived. Here it stands in for a payment provider."
tags = ['stubs', 'http', 'wiremock']
+++

{{< mock-flow kind="http" >}}

## How WireMock works

[WireMock](https://wiremock.org) is the best-known HTTP stub. It is a server
with two parts:

- **Mappings**: "a request like this gets this response", loaded from JSON
  files or posted to its admin API.
- **A journal**: every request it received, which a test queries afterwards
  to check what was sent.

tomato's `http-server` is the same idea, inside the suite. A mapping becomes a
`Given` step, a journal query becomes a `Then` step, and tomato clears both
before every scenario, which with WireMock you do yourself.

{{< side-by-side >}}
{{% side label="WireMock: mappings/charge.json" %}}
```json
{
  "request": { "method": "POST", "url": "/v1/charges" },
  "response": {
    "status": 201,
    "jsonBody": { "id": "ch_1" }
  }
}
```
{{% /side %}}
{{% side label="tomato: a step" %}}
```gherkin
Given "payments" stub "POST" "/v1/charges" returns "201" with json:
  """
  {"id": "ch_1"}
  """
```
{{% /side %}}
{{< /side-by-side >}}

## The suite

```yaml
app:
  command: node app.js
  env:
    PAYMENTS_URL: "{{.payments.url}}"   # http://localhost:9101

resources:
  api:      { type: http, base_url: http://localhost:3000 }
  payments: { type: http-server, options: { port: 9101 } }
```

```gherkin
Scenario: A successful charge
  Given "payments" stub "POST" "/v1/charges" returns "201" with json:
    """
    {"id": "ch_1", "status": "succeeded"}
    """
  When "api" sends "POST" to "/checkout" with json:
    """
    {"amount": 1999}
    """
  Then "api" response status is "201"
  And "payments" received "POST" "/v1/charges" with header "Authorization" "Bearer sk_test_123"
  And "payments" last "POST" "/v1/charges" request json contains:
    """
    {"amount": 1999, "currency": "eur"}
    """

Scenario: The provider is down
  Given "payments" refuses connections
  When "api" sends "POST" to "/checkout" with json:
    """
    {"amount": 1999}
    """
  Then "api" response status is "503"
```

## What it can do

| To | Step |
|---|---|
| answer | `stub "POST" "/v1/charges" returns "201" with json:` |
| answer only some requests | `stub "GET" "/users" with query "page" "2" returns "200"` (also `with header`, `with body containing`) |
| answer differently each call | `stub "POST" "/v1/charges" returns in sequence:` a table of `status`, `body` |
| be down | `refuses connections`, then `accepts connections` |
| check what arrived | `received "POST" "/v1/charges" "2" times`, `last "POST" "/v1/charges" request json contains:` |
| reuse a value the service generated | `last "POST" "/v1/charges" request json "id" saved as "{{charge}}"` |

## Notes

- **Give it a fixed port.** `{{.payments.url}}` needs one to resolve.
- **A request no stub matches gets a 404** that says so, which usually means a
  wrong path in the stub or in the service.
- **The service runs as a process** (`app.command`) for this. Services tomato
  runs *in containers* can't reach a stub on the host on Linux; for those, a
  WireMock container with mapping files is the way, as the end-to-end suite of
  the [coverage demo](../../code-coverage-for-black-box-tests/) does.
