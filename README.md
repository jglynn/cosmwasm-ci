# cosmwasm-ci

Rust is commonly used for writing Smart Contracts. [WHY?](https://use.ink/why-rust-for-smart-contracts/)

The follow project explores CI pipeline support for Rust projects.

Specifically, Rust projects that use [CosmWasm](https://github.com/CosmWasm/cosmwasm#cosmwasm) and compile to Web Assembly (wasm).

Credit: this project uses an existing contract from https://github.com/InterWasm/cw-contracts

## Pipeline Phases

### Caching

Much of the tooling, crates, registry indexing, and build output can be cached to save significant time on builds.

The following is recommended by the [GitHub Cache Action](https://github.com/actions/cache/tree/main#cache-action) for Rust/Cargo projects.

```yml
    - name: Cache
      id: cache-rust
      uses: actions/cache@v3
      continue-on-error: false
      with:
        path: |
          ~/.cargo/bin/
          ~/.cargo/registry/index/
          ~/.cargo/registry/cache/
          ~/.cargo/git/db/
          target/            
        key: ${{ runner.os }}-cargo-${{ hashFiles('**/Cargo.lock') }}
        restore-keys: ${{ runner.os }}-cargo-   
```

### Install Rust

Using the `rust-toolchain` plugin, install `Rust v1.69.0` including: `rustc, cargo, rust-std, clippy` and target `wasm32-unknown-unknown`

```yml
    - name: Install Rust
      uses: dtolnay/rust-toolchain@1.69.0
      with:
        target: wasm32-unknown-unknown
        components: clippy
```

### Install Tools

Additional tools needed for static-analysis/test-coverage/sbom/etc must be installed manually.

The installations pin versions compatible with `Rust 1.69`

They also include `|| true` to avoid build failure if the installed crate is already present (via cache)

```yml
    - name: Install Tools
      run: | 
        cargo install cosmwasm-check --version 1.4.1 --locked || true
        cargo install cargo-llvm-cov --version 0.5.35 --locked || true
        cargo install cargo-sbom --version 0.8.4 --locked || true
        cargo install cargo-sonar --version 0.21.0 --locked || true
```

### Build

We build the project using the alias we created for `wasm` in `.cargo/config` which targets `wasm32-unknown-unknown`

We then perform a `cosmwasm-check` to determine if the binary is a proper smart contract.

```yml
    - name: Build
      run: |
        RUSTFLAGS='-C link-arg=-s' cargo wasm
        cosmwasm-check target/wasm32-unknown-unknown/release/cw_escrow.wasm
```

### Test

Run tests and generate coverage reports in both HTML and LCOV formats

The lcov data will eventually be shipped off to SonarQube for reporting.

```yml
    - name: Test
      run: |
        cargo test
        cargo llvm-cov --html 
        cargo llvm-cov report --lcov --output-path target/lcov.info
```

### Inspect

Run static analysis scans via clippy and assemble findings in a Sonar-friendly format.

```yml
    - name: Inspect
      run: |
        cargo clippy --message-format=json > target/clippy-report.json
        cargo sonar --clippy-path target/clippy-report.json
        mv sonar-issues.json target/sonar-issues.json
```

### Scan

Perform an `audit` and fail on any vulnerable crates.
Generate a Software Bill of Materials `sbom` in `CycloneDX` format

```yml
    - name: Scan
      run: | 
        cargo audit
        cargo sbom --output-format cyclone_dx_json_1_4 > target/cdx-sbom.json 
```

### Publish

Comming soon.

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