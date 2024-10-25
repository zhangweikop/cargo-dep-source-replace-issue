# cargo-dep-source-replace-issue

This "crate" demonstrate the conflict issue caused by same package from replaced source.
This issue is discussed in [cargo/pull/14713](https://github.com/rust-lang/cargo/pull/14713).


## Steps to reproduce the conflict issue

1. Checkout this repo.
   The "bin" crate `deps-conflict` has very simple dependency.
   The crate itself is just a "hello world" and doesn't use those dependency at all.

2. Setup a registry mirror as sourece replacement of `crates-io`
   https://github.com/rust-lang/cargo/wiki/Third-party-registries

   It's convenient to start [Kellnr ](https://kellnr.io/documentation#cratesio-proxy) locally at `localhost:8000` as a mirror. 
   ```
   docker run --rm -it \
    -p 8000:8000 \
    -e "KELLNR_PROXY__ENABLED=true" \
    -v ./tmp/mirrordata:/opt/kdata ghcr.io/kellnr/kellnr:5.0.0
   ```

   The address `localhost:8000` has been specified in the `.cargo/config.toml`
    ```
    [source.crates-io]
    replace-with = "test-mirror"

    [registries]
    [registries.test-mirror]
    index = "sparse+http://localhost:8000/api/v1/cratesio/"
    ```

3. Now build the test crate to see the result:
   `cd deps-conflict && cargo build`

   It should fail with the following message:
    ```
    error: failed to select a version for `wasm-bindgen-shared`.
        ... required by package `wasm-bindgen-macro-support v0.2.40`
        ... which satisfies dependency `wasm-bindgen-macro-support = "=0.2.40"` of package `wasm-bindgen-macro v0.2.40`
        ... which satisfies dependency `wasm-bindgen-macro = "=0.2.40"` of package `wasm-bindgen v0.2.40`
        ... which satisfies dependency `wasm-bindgen = "^0.2.23"` of package `js-sys v0.3.0`
        ... which satisfies dependency `js-sys = "^0.3"` of package `chrono v0.4.20`
        ... which satisfies dependency `chrono_0_4 = "^0.4.20"` of package `serde_with v3.3.0`
        ... which satisfies dependency `serde_with = "^3.3.0"` of package `testdeps v0.1.0 (/Users/wzhang1/workspace/rsworkspace/testdeps)`
    versions that meet the requirements `=0.2.40` are: 0.2.40

    the package `wasm-bindgen-shared` links to the native library `wasm_bindgen`, but it conflicts with a previous package which links to `wasm_bindgen` as well:
    package `wasm-bindgen-shared v0.2.81 (registry `test-mirror`)`
        ... which satisfies dependency `wasm-bindgen-shared = "=0.2.81"` of package `wasm-bindgen-macro-support v0.2.81 (registry `test-mirror`)`
        ... which satisfies dependency `wasm-bindgen-macro-support = "=0.2.81"` of package `wasm-bindgen-macro v0.2.81 (registry `test-mirror`)`
        ... which satisfies dependency `wasm-bindgen-macro = "=0.2.81"` of package `wasm-bindgen v0.2.81 (registry `test-mirror`)`
        ... which satisfies dependency `wasm-bindgen = "^0.2"` of package `chrono v0.4.21 (registry `test-mirror`)`
        ... which satisfies dependency `chrono = "^0.4.21"` of package `testdeps v0.1.0 (/Users/wzhang1/workspace/rsworkspace/testdeps)`
    Only one package in the dependency graph may specify the same links value. This helps ensure that only one copy of a native library is linked in the final binary. Try to adjust your dependencies so that only one package uses the `links = "wasm_bindgen"` value. For more information, see https://doc.rust-lang.org/cargo/reference/resolver.html#links.

    failed to select a version for `wasm-bindgen-shared` which could resolve this conflict
    ```

## Test Cargo fix by [cargo/pull/14713](https://github.com/rust-lang/cargo/pull/14713)
The folder `deps-conflict/result-with-cargo-pr14713` contain result files generated using rust toolchain with local cargo fix.
