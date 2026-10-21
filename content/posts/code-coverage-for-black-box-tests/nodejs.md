+++
title = 'Code coverage of tomato tests: Node.js'
linkTitle = 'Node.js'
weight = 50
tool = 'NODE_V8_COVERAGE · c8'
date = 2026-10-21T03:00:00+02:00
summary = "Coverage is built into Node: set NODE_V8_COVERAGE and V8 writes what ran when the process exits. c8 turns it into reports. Just make sure the process does exit."
tags = ['coverage', 'nodejs', 'typescript']
+++

Node has coverage built in. With `NODE_V8_COVERAGE` set, V8 records what runs
and Node writes it to that directory as JSON when the process exits.
[c8](https://github.com/bcoe/c8) turns it into reports.

## Run it

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

## Exit cleanly

This is the part that bites. Node's default action on SIGTERM ends the
process without running exit handlers, so nothing is written. Handle the
signal and call `process.exit`:

```js
process.on('SIGTERM', () => {
  server.close(() => process.exit(0));
  server.closeIdleConnections();          // keep-alive sockets would hold close() open
  setTimeout(() => process.exit(0), 2000).unref();
});
```

`closeIdleConnections()` matters: tomato's HTTP client keeps connections
alive, and without it `server.close()` waits until tomato gives up after five
seconds and kills the process, coverage and all.

## Notes

- **Every Node process that sees `NODE_V8_COVERAGE` writes into it**,
  including `npx` and any build tool in the same job. `--include 'src/**'`
  keeps the report to your code.
- **TypeScript.** Compile with source maps and c8 maps the coverage back to
  the `.ts` files. Running through `tsx` or `ts-node` works too, with their
  source maps.
- **Bundled code** (esbuild, webpack) needs its source map shipped next to the
  bundle for the report to point at your files rather than at `dist/`.
- **Branches.** c8 reports V8's branch coverage, which is often the more
  useful number for request handlers full of early returns.

## In CI

```yaml
build:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v7
    - uses: actions/setup-node@v7
      with: { node-version: 22, cache: npm }
    - run: npm ci && npm test
    - uses: actions/upload-artifact@v7
      with:
        name: build
        path: |
          src/
          node_modules/
          package.json

blackbox:
  needs: build
  runs-on: ubuntu-latest
  env:
    NODE_V8_COVERAGE: ${{ github.workspace }}/coverage/v8
  steps:
    - uses: actions/checkout@v7
    - uses: actions/download-artifact@v8
      with: { name: build, path: . }
    - uses: actions/setup-node@v7
      with: { node-version: 22 }
    - run: tomato run
    - if: always()
      run: |
        npx --yes c8@10 report --temp-directory "$NODE_V8_COVERAGE" --reports-dir coverage \
          --reporter=lcov --src src --include 'src/**'
```
