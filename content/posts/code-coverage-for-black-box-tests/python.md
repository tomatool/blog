+++
title = 'Code coverage of tomato tests: Python'
linkTitle = 'Python'
weight = 40
tool = 'coverage.py · COVERAGE_PROCESS_START'
date = 2026-10-21T03:00:00+02:00
summary = "coverage.py's startup hook measures any Python process that sees COVERAGE_PROCESS_START, so the command in tomato.yml stays exactly as it is."
tags = ['coverage', 'python']
+++

[coverage.py](https://coverage.readthedocs.io/) can measure a process that
starts it. The obvious way is to change the command to
`coverage run -m notifications`, but then `tomato.yml` differs between
coverage runs and normal ones. A smaller change is coverage.py's startup hook:
any Python process that sees `COVERAGE_PROCESS_START` starts measuring itself,
once a `.pth` file in the environment calls `coverage.process_startup()`.

## Run it

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

Without `COVERAGE_PROCESS_START`, the `.pth` hook does nothing, so it is safe
to leave in a development environment.

## Exit cleanly

`sigterm = true` makes coverage.py save its data when SIGTERM ends a process
that does not handle the signal itself. An asyncio service that handles
SIGTERM and returns from `main` saves on its own:

```python
stop = asyncio.Event()
for sig in (signal.SIGINT, signal.SIGTERM):
    asyncio.get_running_loop().add_signal_handler(sig, stop.set)
await stop.wait()
await runner.cleanup()
await connection.close()
```

## Notes

- **`parallel = true`** gives every process its own data file, and
  `coverage combine` merges them. That also covers subprocesses and
  `multiprocessing` workers, and gunicorn or uvicorn workers.
- **Point `source` at the package, not a path**, so it is measured wherever it
  is installed: from a checkout, or from the wheel your build produced.
- **The data directory has to exist** before the run when `data_file` points
  into it.
- **Merging with unit tests.** Run `pytest --cov` with the same `data_file`
  and `parallel = true`, and `coverage combine` gives the union.

## In CI

The build job builds a wheel; the black-box job installs it into a fresh
virtualenv along with coverage.py and the startup hook.

```yaml
build:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v7
    - uses: actions/setup-python@v7
      with: { python-version: "3.12" }
    - run: pip install -e '.[test]' && python -m pytest
    - run: pip wheel --no-deps -w dist .
    - uses: actions/upload-artifact@v7
      with: { name: build, path: dist/ }

blackbox:
  needs: build
  runs-on: ubuntu-latest
  env:
    COVERAGE_PROCESS_START: ${{ github.workspace }}/.coveragerc
  steps:
    - uses: actions/checkout@v7
    - uses: actions/download-artifact@v8
      with: { name: build, path: dist/ }
    - uses: actions/setup-python@v7
      with: { python-version: "3.12" }
    - run: |
        mkdir -p coverage
        python -m venv .venv
        .venv/bin/pip install dist/*.whl coverage
        echo "import coverage; coverage.process_startup()" \
          > "$(.venv/bin/python -c 'import site; print(site.getsitepackages()[0])')/coverage-blackbox.pth"
    - run: tomato run
    - if: always()
      run: |
        .venv/bin/coverage combine
        .venv/bin/coverage lcov -o coverage/lcov.info
```
