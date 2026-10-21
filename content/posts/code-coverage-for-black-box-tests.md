+++
title = 'Code coverage for black-box tests, in any language'
date = 2026-10-21T03:00:00+02:00
draft = false
summary = "tomato tests a service from the outside, so it cannot see the code it exercised. The service can: every mainstream language can measure a running process and write the result when it exits. Here is how to do that for Go, Rust, the JVM, Python and Node.js, and what the number does and doesn't tell you."
tags = ['coverage', 'black-box testing', 'go', 'rust', 'kotlin', 'python', 'nodejs', 'ci']
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
turning the measurement on. This post shows how for five languages, using one
small shop where each service is written in a different one, and then spends
as long on what the resulting number means, because it is easy to read too
much into it.

## Two kinds of coverage

tomato already reports one kind of coverage, and it is a different thing:

| | Step coverage | Code coverage of the black-box run |
|---|---|---|
| Question | How much of tomato's vocabulary for my resources do my scenarios use? | Which lines of my service ran while the scenarios ran? |
| Measured by | `tomato coverage`, from the feature files alone | The language's own coverage tool, inside the service |
| Needs a run | No | Yes |
| Catches | Typos in steps, resources nobody asserts on | Behaviour no scenario reaches |

Step coverage is cheap and catches a scenario that says "kafka" but never
checks a message. Code coverage is what this post is about: it points at the
error branch, the retry, the consumer path that no scenario ever drives the
service into.

## The approach

The coverage tool lives in the service's process, not in the test:

```text
build       │ the artifact, instrumented or run under an agent
            ▼
tomato run  │ starts the containers, then the app, with tomato's environment
            │   + COVERAGE_SETTING=/where/to/write      ← the only addition
            ▼
scenarios   │ HTTP, gRPC, Kafka, RabbitMQ… exercise the running service
            ▼
SIGTERM     │ tomato stops the app; it exits, and its runtime writes the data
            ▼
report      │ the language's tool turns the data into lcov, HTML, a number
```

Two facts about tomato make this work without touching `tomato.yml`:

1. **The app inherits tomato's environment.** In command mode (`app.command`)
   the process gets everything tomato was started with, plus `app.env`. So the
   variable that switches coverage on is set on the `tomato run` step in CI, or
   in your shell, and the same suite runs with or without it.
2. **tomato stops the app with SIGTERM**, sent to its whole process group, and
   waits five seconds before it sends SIGKILL.

The second one is the only real requirement on the service: **it has to exit
cleanly on SIGTERM, within those five seconds.** Coverage data is written when
the process exits normally. A process that dies from the signal's default
action, or is killed after the grace period, writes nothing, and you get an
empty report rather than an error. Handling SIGTERM is good practice anyway,
since that is how Kubernetes and Docker stop containers too. In the demo shop
two of the five services did not handle it properly until coverage made it
visible.

The other thing to get right is **where the data goes**: give the tool an
absolute path, and create the directory first where the tool expects it to
exist.

## Go

Since Go 1.20, `go build -cover` builds a binary that counts the statements it
runs and writes them to the directory in `GOCOVERDIR` when it exits. Without
`GOCOVERDIR` it only prints a warning, so the instrumented binary is safe to
run anywhere.

```bash
go build -cover -o bin/orders .

mkdir -p coverage/raw
GOCOVERDIR=$PWD/coverage/raw tomato run

go tool covdata percent -i=coverage/raw
go tool covdata textfmt -i=coverage/raw -o coverage/cover.out   # for go tool cover, or other tools
go tool cover -html=coverage/cover.out
```

```text
	github.com/tomatool/demo/services/orders		coverage: 77.5% of statements
	github.com/tomatool/demo/services/orders/gen/catalog/v1		coverage: 25.3% of statements
```

Notes:

- By default only the packages of your main module are instrumented. Add
  `-coverpkg=./...,example.com/lib/...` to count other packages too.
- Data is written when `main` returns or `os.Exit` is called. Handle SIGTERM
  with `signal.NotifyContext`, shut the server down, and return.
- Generated code (protobuf, gRPC stubs) shows up as its own package, usually
  with a low number. Leave it out of the total; it is not code you wrote.
- `GOCOVERDIR` data from unit tests (`go test -cover -args
  -test.gocoverdir=...`) can be merged with `go tool covdata merge`, for the
  union of what either reaches.

## Rust

Rust instruments with LLVM's source-based coverage. Build with
`-C instrument-coverage`, and the binary writes a `.profraw` file to
`LLVM_PROFILE_FILE` when it exits:

```bash
rustup component add llvm-tools-preview
RUSTFLAGS="-C instrument-coverage" cargo build --release

mkdir -p coverage
LLVM_PROFILE_FILE=$PWD/coverage/catalog-%p-%m.profraw tomato run

LLVM="$(rustc --print sysroot)/lib/rustlib/$(rustc -vV | sed -n 's/host: //p')/bin"
"$LLVM/llvm-profdata" merge -sparse coverage/*.profraw -o coverage/catalog.profdata
"$LLVM/llvm-cov" report target/release/catalog -instr-profile=coverage/catalog.profdata \
  --ignore-filename-regex='(\.cargo/registry|/rustc/|/rustlib/)'
"$LLVM/llvm-cov" export -format=lcov target/release/catalog \
  -instr-profile=coverage/catalog.profdata > coverage/lcov.info
```

Notes:

- `%p` (the process id) and `%m` (the binary's signature) in the file name
  keep runs from overwriting each other.
- tokio's `ctrl_c()` handles SIGINT, not SIGTERM. Listen for both:

  ```rust
  async fn shutdown() {
      let mut term = tokio::signal::unix::signal(SignalKind::terminate()).unwrap();
      tokio::select! {
          _ = tokio::signal::ctrl_c() => {}
          _ = term.recv() => {}
      }
  }
  // Server::builder()...serve_with_shutdown(addr, shutdown())
  ```

- `RUSTFLAGS` applies to build scripts too, so a `build.rs` leaves `.profraw`
  files in the crate directory when it runs. Ignore `*.profraw` in git.
- Read the data with the LLVM tools of the toolchain that built the binary.
  They link against that toolchain's `libLLVM`, so copying the two binaries
  somewhere else does not work, and a different LLVM version may not read the
  file. In CI, install the same toolchain with `llvm-tools-preview` in the job
  that reads the coverage.
- [cargo-llvm-cov](https://github.com/taiki-e/cargo-llvm-cov) wraps all of this:
  `cargo llvm-cov show-env` prints the environment for an external run, and
  `cargo llvm-cov report` reads it afterwards.

## The JVM: Kotlin, Java, Scala

On the JVM nothing has to be rebuilt. The [JaCoCo](https://www.jacoco.org/)
agent instruments classes as they load and writes its data from a shutdown
hook. The JVM runs shutdown hooks on SIGTERM by itself, so this works even if
the application does nothing special.

```bash
JACOCO=0.8.13
curl -fsSLO https://repo1.maven.org/maven2/org/jacoco/org.jacoco.agent/$JACOCO/org.jacoco.agent-$JACOCO-runtime.jar
curl -fsSLO https://repo1.maven.org/maven2/org/jacoco/org.jacoco.cli/$JACOCO/org.jacoco.cli-$JACOCO-nodeps.jar

# Gradle's start script (installDist) passes JAVA_OPTS to the JVM.
export JAVA_OPTS="-javaagent:$PWD/org.jacoco.agent-$JACOCO-runtime.jar=destfile=$PWD/coverage/blackbox.exec,includes=demo.*"
tomato run

java -jar org.jacoco.cli-$JACOCO-nodeps.jar report coverage/blackbox.exec \
  --classfiles build/install/payments/lib/payments.jar \
  --sourcefiles src/main/kotlin \
  --xml coverage/jacoco.xml --html coverage/html
```

Notes:

- `includes=demo.*` limits it to your packages. Without it, every library
  class that loads is measured too.
- If your `app.command` is a plain `java -jar`, use `JAVA_TOOL_OPTIONS`
  instead: every JVM reads it, whatever starts it.
- Shutdown hooks race with tomato's five seconds. Keep your own short: stop
  the consumer, join it for a few seconds at most, close connections.
- JaCoCo can also serve its data over TCP (`output=tcpserver`), so you can
  dump coverage per scenario without stopping the service. That is more
  plumbing than most suites need.

## Python

[coverage.py](https://coverage.readthedocs.io/) can measure a process that
starts it. The obvious way is to change the command to
`coverage run -m notifications`, but then `tomato.yml` differs between
coverage runs and normal ones. A smaller change is coverage.py's startup hook:
any Python process that sees `COVERAGE_PROCESS_START` starts measuring itself,
once a `.pth` file in the environment calls `coverage.process_startup()`.

```ini
# .coveragerc
[run]
source = notifications
data_file = coverage/.coverage
parallel = true
sigterm = true
```

```bash
.venv/bin/pip install coverage
echo "import coverage; coverage.process_startup()" \
  > "$(.venv/bin/python -c 'import site; print(site.getsitepackages()[0])')/coverage-blackbox.pth"

mkdir -p coverage
COVERAGE_PROCESS_START=$PWD/.coveragerc tomato run

.venv/bin/coverage combine
.venv/bin/coverage report
.venv/bin/coverage lcov -o coverage/lcov.info
```

```text
Name                        Stmts   Miss  Cover
-----------------------------------------------
notifications/__main__.py       3      0   100%
notifications/app.py          127     12    91%
notifications/messages.py      21      1    95%
-----------------------------------------------
TOTAL                         151     13    91%
```

Notes:

- `parallel = true` gives every process its own data file, and
  `coverage combine` merges them. This also covers subprocesses and
  `multiprocessing` workers.
- `sigterm = true` makes coverage.py save its data when SIGTERM ends a
  process that does not handle the signal itself. An asyncio service that
  handles SIGTERM and returns from `main` saves on its own.
- Point `source` at the package, not a path, so it is measured wherever it is
  installed: from a checkout, or from the wheel your build produced.

## Node.js

Node has coverage built in. With `NODE_V8_COVERAGE` set, V8 records what runs
and Node writes it to that directory as JSON when the process exits.
[c8](https://github.com/bcoe/c8) turns it into reports.

```bash
mkdir -p coverage/v8
NODE_V8_COVERAGE=$PWD/coverage/v8 tomato run

npx c8 report --temp-directory coverage/v8 --reports-dir coverage \
  --reporter=text --reporter=lcov --src src --include 'src/**'
```

```text
------------|---------|----------|---------|---------|-------------------
File        | % Stmts | % Branch | % Funcs | % Lines | Uncovered Line #s
------------|---------|----------|---------|---------|-------------------
All files   |   98.19 |    87.71 |     100 |   98.19 |
 catalog.js |     100 |      100 |     100 |     100 |
 index.js   |   97.43 |     85.1 |     100 |   97.43 | 98,103-104
 price.js   |     100 |      100 |     100 |     100 |
------------|---------|----------|---------|---------|-------------------
```

Notes:

- Node's default action on SIGTERM ends the process without running exit
  handlers, so nothing is written. Handle the signal and exit:

  ```js
  process.on('SIGTERM', () => {
    server.close(() => process.exit(0));
    server.closeIdleConnections();          // keep-alive sockets would hold close() open
    setTimeout(() => process.exit(0), 2000).unref();
  });
  ```

  The `closeIdleConnections()` matters: tomato's HTTP client keeps connections
  alive, and without it `server.close()` waits past the five seconds.
- Every Node process that sees `NODE_V8_COVERAGE` writes into it, including
  `npx` itself. `--include 'src/**'` keeps the report to your code.
- `NODE_V8_COVERAGE` also works for TypeScript compiled with source maps; c8
  maps the coverage back to the `.ts` files.

## Other languages

The same shape works wherever the runtime can write coverage at exit:

| Language | Turn it on | Read it |
|---|---|---|
| .NET | `dotnet-coverage collect --output cov.xml -- dotnet app.dll` as the command, or Coverlet's instrumented build | `reportgenerator` |
| Ruby | `require "simplecov"; SimpleCov.start` first thing, behind an environment variable | SimpleCov writes HTML and JSON at exit |
| PHP | `pcov` or Xdebug, with `auto_prepend_file` starting the collector per request | `phpcov` merges the per-request files |
| C, C++ | `--coverage` (gcov) or `-fprofile-instr-generate -fcoverage-mapping` (clang) | `gcovr`, or `llvm-cov` as for Rust |

PHP is the odd one out: a PHP-FPM worker serves many requests and seldom exits,
so coverage has to be written per request instead of at exit.

## In CI

In the demo shop each service's pipeline builds the artifact, runs tomato
against it with coverage on, and gates the result. For orders, the Go service,
that is:

```yaml
build:
  steps:
    - run: CGO_ENABLED=0 go build -cover -o bin/orders .   # the artifact blackbox tests
    - uses: actions/upload-artifact@v7
      with: { name: build, path: services/orders/bin/orders }

blackbox:
  needs: build
  env:
    GOCOVERDIR: ${{ github.workspace }}/services/orders/coverage/raw
  steps:
    - uses: actions/download-artifact@v8
      with: { name: build, path: services/orders/bin/ }
    - run: chmod +x services/orders/bin/orders && mkdir -p "$GOCOVERDIR"
    - run: tomato run
      working-directory: services/orders
    - if: always()
      run: go tool covdata textfmt -i="$GOCOVERDIR" -o coverage/cover.out
      working-directory: services/orders
    # then: turn the report into a job summary, and fail below the floor
```

The other four services differ only in the environment variable and the
command that reads the data. One small script reduces lcov, Go's cover profile
and JaCoCo's XML to lines per file, writes the job summary and PR comment, and
fails the job when the total falls below that service's floor. These are the
results:

| Service | Language | Collected with | Black-box coverage | Floor |
|---|---|---|---|---|
| gateway | Node.js | `NODE_V8_COVERAGE` + c8 | 98.2% of lines | 90% |
| payments | Kotlin | JaCoCo agent via `JAVA_OPTS` | 93.0% of lines | 85% |
| notifications | Python | coverage.py via `COVERAGE_PROCESS_START` | 91.4% of lines | 85% |
| catalog | Rust | `-C instrument-coverage` + llvm-cov | 81.4% of lines | 70% |
| orders | Go | `go build -cover` + `GOCOVERDIR` | 77.5% of statements | 70% |

A floor below the current number, rather than a target, is deliberate: it
stops coverage from quietly shrinking when a scenario is deleted, without
making anyone write scenarios to chase a percentage.

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
  stubs), and with step coverage for what you never check.
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
- **Work that is still in flight when the suite ends** may be measured partly
  or not at all, if the service does not finish it within the shutdown grace
  period.
- **Numbers do not compare across languages.** Go counts statements, JaCoCo
  and c8 count lines, llvm-cov counts lines and regions, coverage.py counts
  statements and calls them lines. 80% in Go and 80% in Kotlin are not the
  same amount of testing.
- **What you exclude changes the number.** Generated code, vendored code and
  dependencies have to be left out on purpose, the same way each time.
- **Branches.** Most of the tools above also report branch coverage, which is
  stricter than line coverage. A line with `a && b` can be covered while `b`
  never decided anything.

## Reading the gaps

The most useful output is not the percentage but the list of what did not
run. In the demo, the orders service sits at 77.5%, and almost everything it
misses is the same kind of thing:

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
