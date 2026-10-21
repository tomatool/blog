+++
title = 'Code coverage of tomato tests: Go'
linkTitle = 'Go'
weight = 10
tool = 'go build -cover · GOCOVERDIR'
date = 2026-10-21T03:00:00+02:00
summary = "Build with -cover, point GOCOVERDIR at a directory, and go tool covdata reads what ran. No agent, nothing in the code."
tags = ['coverage', 'go']
+++

Since Go 1.20, `go build -cover` builds a binary that counts the statements it
runs and writes them to the directory in `GOCOVERDIR` when it exits. Without
`GOCOVERDIR` it only prints a warning, so the instrumented binary is safe to
run anywhere.

## Run it

```bash
go build -cover -o bin/orders .

mkdir -p coverage/raw
GOCOVERDIR=$PWD/coverage/raw tomato run

go tool covdata percent -i=coverage/raw
go tool covdata textfmt -i=coverage/raw -o coverage/cover.out   # for go tool cover and other tools
go tool cover -func=coverage/cover.out
go tool cover -html=coverage/cover.out
```

```text
	github.com/tomatool/demo/services/orders		coverage: 77.5% of statements
	github.com/tomatool/demo/services/orders/gen/catalog/v1		coverage: 25.3% of statements
```

`tomato.yml` does not change: tomato passes `GOCOVERDIR` on to the app.

## Exit cleanly

Data is written when `main` returns or `os.Exit` is called, not when the
process dies from a signal. Stop on SIGTERM and return:

```go
func run() error {
	ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
	defer stop()

	srv := &http.Server{Addr: ":8080", Handler: routes()}
	go func() {
		<-ctx.Done()
		shutdown, cancel := context.WithTimeout(context.Background(), 3*time.Second)
		defer cancel()
		srv.Shutdown(shutdown)
	}()
	if err := srv.ListenAndServe(); err != http.ErrServerClosed {
		return err
	}
	return nil
}
```

## Notes

- **What is instrumented.** By default, only the packages of your main
  module. Add `-coverpkg=./...,example.com/lib/...` to count other packages
  too.
- **Generated code** (protobuf, gRPC stubs) shows up as its own package,
  usually with a low number. Leave it out of the total; it is not code you
  wrote.
- **Statements, not lines.** Go counts statements in basic blocks, so its
  number is not directly comparable with the line coverage of other languages.
- **Merging with unit tests.** `go test -cover ./... -args
  -test.gocoverdir=$PWD/coverage/unit` writes the same format, and
  `go tool covdata merge -i=coverage/raw,coverage/unit -o coverage/all` gives
  the union of what either reaches.

## In CI

The build job ships the instrumented binary; the black-box job needs Go only
to read the data.

```yaml
build:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v7
    - uses: actions/setup-go@v7
      with: { go-version-file: go.mod }
    - run: go test ./...
    - run: CGO_ENABLED=0 go build -cover -o bin/orders .
    - uses: actions/upload-artifact@v7
      with: { name: build, path: bin/orders }

blackbox:
  needs: build
  runs-on: ubuntu-latest
  env:
    GOCOVERDIR: ${{ github.workspace }}/coverage/raw
  steps:
    - uses: actions/checkout@v7
    - uses: actions/download-artifact@v8
      with: { name: build, path: bin/ }
    - uses: actions/setup-go@v7
      with: { go-version-file: go.mod, cache: false }
    - run: chmod +x bin/orders && mkdir -p "$GOCOVERDIR"
    - run: tomato run
    - if: always()
      run: |
        go tool covdata textfmt -i="$GOCOVERDIR" -o coverage/cover.out
        go tool cover -func=coverage/cover.out | tail -1
```
