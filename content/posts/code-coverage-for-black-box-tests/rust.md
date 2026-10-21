+++
title = 'Code coverage of tomato tests: Rust'
linkTitle = 'Rust'
weight = 20
tool = '-C instrument-coverage · llvm-cov'
date = 2026-10-21T03:00:00+02:00
summary = "Build with LLVM's source-based coverage, set LLVM_PROFILE_FILE, and read the profile with the toolchain's own llvm-profdata and llvm-cov."
tags = ['coverage', 'rust']
+++

Rust instruments with LLVM's source-based coverage. Build with
`-C instrument-coverage`, and the binary writes a `.profraw` file to
`LLVM_PROFILE_FILE` when it exits.

## Run it

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

`%p` (the process id) and `%m` (the binary's signature) in the file name keep
runs from overwriting each other.

## Exit cleanly

The profile is written when `main` returns. tokio's `ctrl_c()` handles SIGINT,
not SIGTERM, so a server that only waits for it is killed by tomato after five
seconds and writes nothing. Listen for both:

```rust
use tokio::signal::unix::{signal, SignalKind};

async fn shutdown() {
    let mut term = signal(SignalKind::terminate()).expect("installing the SIGTERM handler");
    tokio::select! {
        _ = tokio::signal::ctrl_c() => {}
        _ = term.recv() => {}
    }
}

// Server::builder().add_service(...).serve_with_shutdown(addr, shutdown()).await?;
```

## Notes

- **Build scripts are instrumented too.** `RUSTFLAGS` applies to `build.rs`,
  so it leaves `.profraw` files in the crate directory when it runs. Add
  `*.profraw` to `.gitignore`.
- **Use the tools of the toolchain that built the binary.** `llvm-profdata` and
  `llvm-cov` link against that toolchain's `libLLVM`, so copying the two
  binaries to another job fails to start, and an LLVM of another version may
  not read the profile. In CI, install the same toolchain with
  `llvm-tools-preview` in the job that reads the coverage.
- **Leave out dependencies.** Without `--ignore-filename-regex`, the report
  includes the standard library and every crate from the registry.
- **cargo-llvm-cov** wraps all of this:
  `cargo llvm-cov show-env --export-prefix` prints the environment for an
  external run, and `cargo llvm-cov report` reads it afterwards.

## In CI

```yaml
build:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v7
    - uses: dtolnay/rust-toolchain@stable
    - run: cargo test --locked
    - run: cargo build --release --locked
      env:
        RUSTFLAGS: -C instrument-coverage
    - uses: actions/upload-artifact@v7
      with: { name: build, path: target/release/catalog }

blackbox:
  needs: build
  runs-on: ubuntu-latest
  env:
    LLVM_PROFILE_FILE: ${{ github.workspace }}/coverage/catalog-%p-%m.profraw
  steps:
    - uses: actions/checkout@v7
    - uses: actions/download-artifact@v8
      with: { name: build, path: target/release/ }
    - run: chmod +x target/release/catalog && mkdir -p coverage
    # The same stable toolchain that built the binary, for its LLVM tools.
    - uses: dtolnay/rust-toolchain@stable
      with: { components: llvm-tools-preview }
    - run: tomato run
    - if: always()
      run: |
        LLVM="$(rustc --print sysroot)/lib/rustlib/$(rustc -vV | sed -n 's/host: //p')/bin"
        "$LLVM/llvm-profdata" merge -sparse coverage/*.profraw -o coverage/catalog.profdata
        "$LLVM/llvm-cov" export -format=lcov target/release/catalog \
          -instr-profile=coverage/catalog.profdata > coverage/lcov.info
```
