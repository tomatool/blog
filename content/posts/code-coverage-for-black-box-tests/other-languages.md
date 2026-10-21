+++
title = 'Code coverage of tomato tests: other languages'
linkTitle = 'Other languages'
weight = 60
tool = '.NET · Ruby · PHP · C/C++'
date = 2026-10-21T03:00:00+02:00
summary = "The same shape works wherever the runtime can write coverage when the process ends: .NET, Ruby, PHP and C/C++, with what is different about each."
tags = ['coverage', 'dotnet', 'ruby', 'php', 'c']
+++

The approach does not depend on the language: turn coverage on for the
process tomato starts, make sure it exits cleanly on SIGTERM, and read the
data afterwards. These are the usual tools for languages the demo shop does
not use.

| Language | Turn it on | Read it |
|---|---|---|
| .NET | `dotnet-coverage collect` around the command, or Coverlet's instrumented build | `reportgenerator` |
| Ruby | SimpleCov started first thing in the process | SimpleCov writes HTML and JSON at exit |
| PHP | `pcov` or Xdebug, with a collector started per request | `phpcov` merges the per-request files |
| C, C++ | `--coverage` (gcov) or `-fprofile-instr-generate -fcoverage-mapping` (clang) | `gcovr`, or `llvm-cov` as for [Rust](../rust/) |

## .NET

`dotnet-coverage` wraps the process and writes the report when it exits:

```yaml
# tomato.yml
app:
  command: dotnet-coverage collect --output coverage/blackbox.cobertura.xml --output-format cobertura -- dotnet bin/Orders.dll
```

That puts coverage into the command, so keep a second config, or set the
command through an environment variable (`command: ${APP_COMMAND}`). An
ASP.NET Core host stops gracefully on SIGTERM by default, within its
`ShutdownTimeout`; keep that under tomato's five seconds.

## Ruby

SimpleCov has to start before your code loads, and writes its results in an
`at_exit` hook:

```ruby
# at the very top of config/boot.rb, or the server's entry point
if ENV["BLACKBOX_COVERAGE"]
  require "simplecov"
  SimpleCov.command_name "tomato"
  SimpleCov.start { add_filter "/vendor/" }
end
```

Puma and Unicorn fork workers: start SimpleCov in each worker
(`on_worker_boot`) and use `SimpleCov.command_name` per worker, so the results
merge instead of overwriting each other. Both servers shut down gracefully on
SIGTERM.

## PHP

PHP is the odd one out. A PHP-FPM worker serves many requests and seldom
exits, so coverage has to be collected per request. With `pcov` loaded, an
`auto_prepend_file` starts collection and a shutdown function writes one file
per request; `phpcov merge` combines them after the run. Make sure the
collector only runs when an environment variable says so.

## C and C++

Both GCC and Clang can instrument a binary, which writes its data in an
`atexit` handler, so the program must return from `main` or call `exit`, not
die from the signal:

```bash
clang -fprofile-instr-generate -fcoverage-mapping -O1 -o server server.c
LLVM_PROFILE_FILE=$PWD/coverage/server-%p.profraw tomato run
llvm-profdata merge -sparse coverage/*.profraw -o coverage/server.profdata
llvm-cov report ./server -instr-profile=coverage/server.profdata
```

A SIGTERM handler that sets a flag your main loop checks is enough.
