+++
title = 'Driving your service from a shell'
linkTitle = 'Shell'
weight = 40
tool = 'shell'
date = 2026-10-21T06:00:00+02:00
summary = "The shell client runs commands: your CLI, a migration, a script a cron job runs. It checks the exit code, what was printed, and the files left behind."
tags = ['clients', 'shell', 'cli']
+++

{{< client-flow kind="shell" >}}

## The suite

The service here ships a small command-line tool, `cli.js`, that talks to its
API.

```yaml
resources:
  shell:
    type: shell
    options:
      env: { TODOS_API: http://localhost:3000 }   # what every command sees
```

```gherkin
Scenario: A todo added from the command line
  When "shell" runs "node cli.js add Buy milk"
  Then "shell" succeeds
  And "shell" stdout contains "Buy milk"
  When "shell" runs "node cli.js list"
  Then "shell" stdout contains "Buy milk"

Scenario: A todo without a title is refused
  When "shell" runs "node cli.js add"
  Then "shell" exit code is "1"
  And "shell" stderr contains "title is required"
  And "shell" stdout is empty
```

## What it can do

| To | Step |
|---|---|
| set up | `env "API_KEY" is "…"`, `workdir is "./fixtures"` |
| run | `runs "cmd"`, `runs:` (a multi-line script), `runs script "scripts/seed.sh"`, `runs with timeout "60s":` |
| check the exit | `succeeds`, `fails`, `exit code is "2"` |
| check output | `stdout contains "…"`, `stdout is:`, `stdout is empty`, `stderr contains "…"` |
| check files | `file "out/report.csv" exists`, `file "…" contains "…"` |

## Notes

- **It runs on the machine tomato runs on**, in tomato's directory, with
  tomato's environment plus `options.env` and `env` steps.
- **Use it for anything with a command**: your CLI, a migration tool, a
  backup script, `curl` against an endpoint no client covers.
- **Check stderr and the exit code on the failure path.** A CLI that prints
  an error and exits 0 breaks every script that calls it; `exit code is "1"`
  catches that.
