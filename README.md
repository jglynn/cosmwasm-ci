# cosmwasm-ci

Explore a CI pipeline for cosmwasm Smart Contracts

## Pipeline Phases

### Caching

Rust/Cargo downloads the world -- let's cache as much as we can.

```bash
~/.cargo/bin/
~/.cargo/registry/index/
~/.cargo/registry/cache/
~/.cargo/git/db/
target/  
```

### Install Rust

Using toolchain plugin, install Rust v1.69 including:

* cargo
* clippy
* rust-std
* rustc
* rust-analyzer
* rustfmt
* cargo-fmt

### Install Tools

Anything we cannot get from standard toolchain we need to install manually.

```bash
cargo install cargo-wasm --version 0.4.1 --locked
cargo install cosmwasm-check --version 1.4.1 --locked
cargo install cargo-llvm-cov --version 0.5.35 --locked 
cargo install cargo-sbom --version 0.8.4 --locked 
cargo install cargo-sonar --version 0.21.0 --locked
```

### Build

Build WASM and perform a cosmwasm-check

```bash
# cargo build --release
RUSTFLAGS='-C link-arg=-s' cargo wasm build
cosmwasm-check target/wasm32-unknown-unknown/release/cw_escrow.wasm
```

### Test

Run tests and generate coverage reports

```bash
cargo test --verbose
cargo llvm-cov --html 
cargo llvm-cov report --lcov --output-path target/lcov.info
```

### Inspect

Run static analysis scans and assemble test coverage report in Sonar-friendly format.

```bash
cargo clippy --message-format=json > target/clippy-report.json
cargo sonar --clippy-path target/clippy-report.json
mv sonar-issues.json target/sonar-issues.json
```

### Scan

Perform audit (failing on any vulnerable deps) and generate a Software Bill of Materials 

```bash
cargo audit
cargo sbom --output-format cyclone_dx_json_1_4 > target/cdx-sbom.json  
```

## Example Contract: Escrow

This is a simple single-use escrow contract. It creates a contract that can hold some
native tokens and gives the power to an arbiter to release them to a pre-defined
beneficiary. They can release all tokens, or only a fraction. If an optional
timeout is reached, the tokens can no longer be released, rather they can only
be returned to the original funder. Tokens can be added to the contract at any
time without causing any errors, or losing access to them.

This contract is mainly considered as a simple tutorial example. In the real
world, you would probably want one contract to manage many escrows and allow
some global configuration options on it. It is generally simpler to rely on
some well-known address for handling all escrows securely than checking each
deployed escrow is using the proper wasm code.

As of v0.2.0, this was rebuilt from
[`cosmwasm-template`](https://github.com/confio/cosmwasm-template),
which is the recommended way to create any contracts.

Source: https://github.com/InterWasm/cw-contracts