+++
title = 'Code coverage for black-box tests, in any language'
date = 2026-10-21T03:00:00+02:00
layout = 'series'
summary = "tomato tests a service from the outside, so it cannot see the code it exercised. The service can: every mainstream language can measure a running process and write the result when it exits. How the approach works, what the number does and doesn't tell you, and a page per language."
tags = ['coverage', 'black-box testing', 'ci']
+++

A tomato suite tests a service the way the rest of the system sees it. It
starts the service's real dependencies in containers, stands in for the
services it calls, and talks to it only over the network: HTTP, gRPC, Kafka,
RabbitMQ. No scenario imports a line of the service's code. That is the point,
and it is also why tomato cannot tell you which of that code your scenarios
actually ran.

The service can. Every mainstream language has a way to measure a running
process and write down what executed when the process exits. tomato starts the
process, hands it its environment and stops it cleanly, so all that is left is
turning the measurement on.

This page explains the approach, which is the same for every language, and
what the resulting number means. The commands for each language are on their
own pages, written against one small shop where each service is in a
different language.

## The question worth asking

Which lines of my service ran while the scenarios ran? That is the coverage
that says something about the service: it points at the error branch, the
retry, the consumer path that no scenario ever drives the service into.

> **Not step coverage.** `tomato coverage` reports something else: how much of
> tomato's step vocabulary your feature files use. That is a fact about how the
> suite is worded, not about the service. Ten of the HTTP client's 56 steps can
> be a complete suite, all 56 can still miss every error path, and the
> percentage drops whenever tomato ships new steps. Its useful parts are lints,
> not a metric: a step that matches nothing (`tomato validate` catches that
> too), or a declared resource no step ever touches.

## The approach

tomato probes the service from outside; the coverage tool sits inside the
service's process and keeps count. Step through a run of the demo's Go
service, orders:

{{< coverage-flow >}}

Two facts about tomato make this work without touching `tomato.yml`:

1. **The app inherits tomato's environment.** In command mode (`app.command`)
   the process gets everything tomato was started with, plus `app.env`. So the
   variable that switches coverage on is set on the `tomato run` step in CI, or
   in your shell, and the same suite runs with or without it.
2. **tomato stops the app with SIGTERM**, sent to its whole process group, and
   waits five seconds before it sends SIGKILL.

### The one requirement: exit cleanly on SIGTERM

Coverage data is written when the process exits normally. A process that dies
from the signal's default action, or is killed after the five-second grace
period, writes nothing, and you get an empty report rather than an error. So
**the service has to handle SIGTERM and exit within five seconds.**

That is good practice anyway, since it is how Kubernetes and Docker stop
containers too. In the demo shop, two of the five services did not do it
properly until coverage made it visible: a Rust service that only listened for
Ctrl-C, and a Node.js server whose `close()` waited on idle keep-alive
connections.

The other thing to get right is **where the data goes**: give the tool an
absolute path, and create the directory first where the tool expects it to
exist.

## Per language

{{< parts >}}

## In CI

In the demo shop each service's pipeline builds the artifact, runs tomato
against it with coverage on, and gates the result:

```text
build ──────► blackbox ──────► report
              │                job summary and PR comment: results,
              │                coverage per file, pass or fail against the floor
              └ tomato run with coverage on, then read the data
```

Every language page ends with its version of the `blackbox` job. Across the
five, only two things change: the environment variable that turns coverage
on, and the command that reads the data afterwards. One small script then
reduces lcov, Go's cover profile and JaCoCo's XML to covered lines per file,
writes the job summary and the PR comment, and fails the job when the total
falls below that service's floor:

| Service | Language | Collected with | Black-box coverage | Floor |
|---|---|---|---|---|
| gateway | [Node.js](nodejs/) | `NODE_V8_COVERAGE` + c8 | 98.2% of lines | 90% |
| payments | [Kotlin](jvm/) | JaCoCo agent via `JAVA_OPTS` | 93.0% of lines | 85% |
| notifications | [Python](python/) | coverage.py via `COVERAGE_PROCESS_START` | 91.4% of lines | 85% |
| catalog | [Rust](rust/) | `-C instrument-coverage` + llvm-cov | 81.4% of lines | 70% |
| orders | [Go](go/) | `go build -cover` + `GOCOVERDIR` | 77.5% of statements | 70% |

A floor below the current number, rather than a target, is deliberate: it
stops coverage from quietly shrinking when a scenario is deleted, without
making anyone write scenarios to chase a percentage.

## What the number tells you

A report marks every line that ran. Here is a handler from the demo, the way
to read it, and the one kind of line coverage cannot tell apart from a tested
one:

{{< coverage-meaning >}}

## What it covers

The lines that ran inside the service's process while tomato drove it, which
in practice means:

- **Everything between the network and the database**: routing, parsing and
  validation, serialization, mapping a gRPC `NOT_FOUND` to an HTTP 422, the
  consumer loop that reads a Kafka message, the code that acknowledges or
  rejects a RabbitMQ delivery.
- **Wiring that unit tests skip**: configuration from the environment,
  connection setup, migrations at startup, the retry policy on a gRPC channel,
  an SDK pointed at a stubbed Stripe.
- **Failure handling, when a scenario causes the failure**: a stub that
  answers 503 twice, a gRPC server that refuses connections, a message that is
  not JSON. These are exactly the paths that are hard to reach in a unit test
  and easy to reach with tomato.
- **The real artifact.** With an agent (JVM, Python, Node.js) it is the very
  thing you ship. With an instrumented build (Go, Rust) it is the same source
  and the same dependencies, compiled with counters.

## What it doesn't cover

This is the half that matters when you read the number.

- **Executed is not verified.** A line counts as covered if it ran, whether or
  not any step asserted anything about what it did. A scenario that sends a
  request and checks only the status code covers the whole handler. Coverage
  tells you what your suite *cannot* be testing; it never tells you what it
  *is* testing. Pair it with assertions on effects (rows, messages, calls to
  stubs) in every scenario, rather than chasing the number.
- **Only the process under test.** Not the stubs, not the database, not the
  other services. In an end-to-end run with five services you need five
  measurements, one per process, and for services running in containers that
  means instrumented images and a volume to get the data out.
- **Startup and shutdown are counted too.** Code that runs once at boot,
  migrations and client construction, is covered by the first scenario and
  inflates the total. A service that is mostly wiring shows a high number
  easily.
- **Not every build is the shipped build.** Go and Rust measure an
  instrumented binary: same logic, different machine code, slower. For a
  coverage run that is fine; for performance assertions it is not, so do not
  time things in the instrumented run.
- **Work still in flight when the suite ends** may be measured partly or not
  at all, if the service does not finish it within the shutdown grace period.
- **Numbers do not compare across languages.** Go counts statements, JaCoCo
  and c8 count lines, llvm-cov counts lines and regions, coverage.py counts
  statements and calls them lines. 80% in Go and 80% in Kotlin are not the
  same amount of testing.
- **What you exclude changes the number.** Generated code, vendored code and
  dependencies have to be left out on purpose, the same way each time.
- **Branches.** Most of these tools also report branch coverage, which is
  stricter than line coverage. A line with `a && b` can be covered while `b`
  never decided anything.

## Reading the gaps

The most useful output is not the percentage but the list of what did not
run. In the demo, the [Go](go/) orders service sits at 77.5%, and almost
everything it misses is the same kind of thing:

```text
api.go:45     health            50.0%   the branch where Postgres does not answer
api.go:141    getOrder          61.5%   a database error while reading an order
events.go:34  orderCreated      75.0%   Kafka refusing the event
events.go:59  consumePayments   60.0%   RabbitMQ dropping the connection and the reconnect
events.go:119 handlePayment     53.3%   a database error while recording a payment
```

Every scenario fails a *neighbour*: the catalog stub refuses connections, a
payment event is malformed. None fails the service's *own* infrastructure. That
is a real gap: nobody knows what orders does when Kafka goes away for ten
seconds. tomato's `lifecycle` resource can stop and start containers in the
middle of a scenario, and that is the scenario to write next. Coverage did
not write it, but it pointed at it.

That is the way to use the number: as a map of the behaviour your suite never
provokes, read file by file, rather than a score to push up.
