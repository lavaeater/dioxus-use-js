# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

`dioxus-use-js` provides the `use_js!` proc macro, which generates type-safe Rust bindings to JavaScript/TypeScript functions and classes for [Dioxus](https://github.com/DioxusLabs/dioxus). It works on every `eval`-supported platform (Web, Desktop, Mobile, liveview) by generating glue code over `dioxus::document::eval` — there is no `wasm-bindgen` dependency. When a `.ts` source (or a JS file with an inline/linked sourcemap) is available, the macro parses the TS types to produce accurate Rust signatures.

## Workspace layout

Three crates (`Cargo.toml` workspace, `resolver = "3"`, edition 2024, MSRV 1.88):

- **`dioxus-use-js-macro/`** — the proc macro itself. `src/lib.rs` (~2800 lines) contains all parsing/codegen logic. This is where almost all real work happens.
- **`dioxus-use-js/`** — the public-facing runtime crate. Re-exports `use_js!`, defines the runtime types the generated code depends on (`JsError`, `JsValue`), and re-exports serde items under `#[doc(hidden)]` aliases so downstream crates don't need to depend on serde directly. The optional `build` feature exposes `BunBuild` (in `src/build.rs`) for compiling TS in `build.rs`.
- **`example/`** — a runnable Dioxus app exercising the macro. Not published. Its `build.rs` uses `BunBuild` to compile `js-utils/src/*.ts` into `assets/` with linked sourcemaps.

The root `README.md` is a symlink to `dioxus-use-js/README.md`, which is included as the crate-level doc via `#![doc = include_str!("../README.md")]`. **The README is the canonical spec for type mapping and special-type behavior** — keep it in sync when changing macro codegen.

## Common commands

```bash
cargo build                 # build the workspace
cargo test                  # run all tests (the macro's unit tests live in dioxus-use-js-macro)
cargo test -p dioxus-use-js-macro test_primitives   # run a single test
```

Running the example app (from `example/`):

```bash
dx serve                    # serve the example (default feature: web)
bun build js-utils/src/example.ts --outdir assets   # manual TS->JS build (normally done by build.rs)
```

The example's `build.rs` calls `BunBuild` with `skip_if_no_bun(true)`, so the build still succeeds without `bun` installed (the committed bundle in `assets/` is used). CI just runs `cargo build` + `cargo test`.

## Macro architecture (`dioxus-use-js-macro/src/lib.rs`)

The flow, roughly top to bottom in the file:

1. **Parse macro input** (`UseJsInput`, `ImportSpec`) — supports `use_js!("bundle.js"::func)`, `::{a, b}`, `::*`, and the explicit-source form `use_js!("source.ts", "bundle.js"::{...})`.
2. **Parse the JS/TS source** (`parse_script_file` → `parse_script`) with the `swc_ecma_*` crates. `JsVisitor` (`impl Visit`) walks the AST collecting `FunctionInfo`, `ClassInfo`, `MethodInfo`, `ParamInfo`. For plain `.js` with no types, `try_extract_sourcemap` looks for an inline/linked sourcemap to recover TS types.
3. **Type mapping** — `ts_type_to_rust_type` / `ts_type_to_rust_type_helper` convert TS type strings to Rust types, distinguishing input position (borrowed, e.g. `&str`) from output position (owned, e.g. `String`). `split_top_level_union`, `split_into_args`, `strip_parenthesis` handle generic/union parsing.
4. **Codegen** — `generate_invocation` builds each function binding; `generate_class_wrapper` builds the `struct Wrapper(JsValue)` + `impl` for classes. The generated code emits a JS snippet executed via `eval`, plus serialization/deserialization glue.

### Special types

These are the load-bearing concepts; their exact runtime contracts are documented in the README and must match the codegen:

- **`JsValue<T>`** — opaque reference to a JS-side value passed by reference instead of being serialized. Backed by `Arc`; the JS value is disposed when the last Rust reference drops (see `JsValue` / `JsValueInner` in `dioxus-use-js/src/lib.rs`).
- **`RustCallback<A, R>`** — a Rust async closure (`dioxus::core::Callback`) passed into JS, lets JS call back into Rust. Lifecycle is tied to the calling component, not the function invocation; multiple invocations can be in flight concurrently.
- **`Drop`** (`Promise<void>`) — resolves when the calling component is dropped; generates no user-facing Rust parameter. A JS param literally named `drop` with no type is treated as `Drop`.
- **`i64`/`u64`** type aliases force integer (not `f64`) Rust types; **`Json`** maps to `serde_json::Value`.

When changing how any special type is generated, update both the codegen and the README's type-mapping/special-type sections, and add/adjust the `ts_type_to_rust_type` unit tests in the `#[cfg(test)] mod tests` block.

## Conventions

- `dioxus-use-js-macro` is pinned to the exact version of `dioxus-use-js` (`version = "=0.4.7"`); bump both together.
- `dioxus` itself is **not** re-exported from the runtime crate on purpose (comment in `lib.rs`): mixing two dioxus versions in the graph compiles but fails at runtime during `eval`.
- Generated Rust deliberately preserves the JS identifier casing (e.g. `createDefault`, `getCount`), so bindings are not snake_case.
