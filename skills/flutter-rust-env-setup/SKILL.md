---
name: flutter-rust-env-setup
description: Use when setting up a Flutter project with a Rust core via flutter_rust_bridge and Cargokit from scratch, or when diagnosing a build/run failure in such a project — content hash mismatches, linker errors (missing native libraries), "cannot find type in this scope" / SseEncode errors in frb_generated.rs, crate-not-found errors, or integrate scaffolding conflicts.
---

# Flutter + Rust (flutter_rust_bridge) environment setup & troubleshooting

This skill covers the `flutter_rust_bridge` + Cargokit toolchain: a Flutter app with a Rust crate as its core logic, bridged via generated Dart/Rust glue code.

## Initial project setup (from scratch)

1. **Create the Rust crate as a library**, not a binary: `cargo new --lib rust` inside the Flutter project root (name the folder `rust/` by convention, adjust if the project differs).

2. **Crate naming**: Cargokit expects the crate name to match a `rust_lib_<flutter_package_name>` pattern (snake_case). Check `pubspec.yaml`'s `name:` field and name the crate `rust_lib_<that_name>`. A mismatched name will cause the native library to not be found at build/integrate time — this is a common silent-failure point, check it first if `integrate` or a build can't find the library.

3. **Run `flutter_rust_bridge_codegen integrate`** to wire up Cargokit and generate the initial scaffolding. **This generates a demo `api/` module** — delete this demo scaffolding before writing real code; don't try to merge hand-written modules into it, since the demo structure usually doesn't match the project's intended module layout.

4. **Module layout** — modern style (an `api.rs` file beside an `api/` folder, not the older `api/mod.rs` style), unless the project already established the older style:
   ```rust
   // lib.rs
   mod frb_generated;
   pub mod api;

   // api.rs
   pub mod some_module;
   ```

5. **Required system libraries** — install before the first build, per target platform:
   - Linux (desktop target): `libsqlite3-dev`, `libsecret-1-dev`, `libgtk-3-dev` (exact list depends on which crates are pulled in — e.g. a crate needing SQLite needs `libsqlite3-dev`; a crate needing secret storage needs `libsecret-1-dev`). If a linker error names a missing `-l<name>` library, the fix is almost always `apt install lib<name>-dev` (or the distro equivalent), not a Cargo-side fix.
   - macOS/Windows: check the specific crates in use for their native dependency requirements (e.g. `matrix-sdk-sqlite` needs SQLite; the OS package manager equivalents are Homebrew/vcpkg).

## The `pub` vs `pub(crate)` rule — critical, check this first on bridge errors

Only `pub` items in the `api` module tree are exposed across the FFI bridge to Dart, and **every type that crosses the bridge must be bridge-safe** (serializable/encodable by the generated `Sse*` code — primitives, `String`, `Vec<u8>`, and structs/enums made of bridge-safe types; **not** arbitrary external-crate types like an SDK's `Client`, connection objects, etc.).

**Diagnostic signature**: if you see any of:
- `error[E0425]: cannot find type '<TypeName>' in this scope` inside `frb_generated.rs`
- `the trait bound '<crate>::<Type>: SseEncode' is not satisfied`
- a cascade of ~10+ errors all centered on one external type name appearing throughout `frb_generated.rs`

...the root cause is a `pub` function somewhere whose signature exposes a non-bridge-safe type (commonly: returning a client/connection object from an SDK, or taking one as a parameter). **Fix**: change that function to `pub(crate)` so it's usable internally but not bridge-exposed, and have the actual `pub` bridge functions take/return only bridge-safe types (strings, primitives, bridge-safe structs), doing the conversion internally.

Do not try to make the external type bridge-safe by hand (implementing `SseEncode` etc. for a type you don't own) — restructure the API surface instead.

## Common singleton pattern for a shared SDK client

When a Rust core wraps a stateful SDK client that should persist across multiple bridge calls (e.g. one authenticated client reused by login, data-fetch, and other calls), use a process-wide `OnceLock`, and keep the function that reads/builds it `pub(crate)`:

```rust
use std::sync::OnceLock;

static CLIENT: OnceLock<SomeSdkClient> = OnceLock::new();

pub(crate) async fn get_or_create_client(config: &str) -> Result<SomeSdkClient, String> {
    if let Some(client) = CLIENT.get() {
        return Ok(client.clone()); // cheap if the client is internally Arc/Rc-based — verify for your SDK
    }
    let client = SomeSdkClient::builder(config).build().await.map_err(|e| e.to_string())?;
    let _ = CLIENT.set(client.clone());
    Ok(client)
}
```

All `pub` bridge functions that need the client should call this instead of constructing their own.

## Known failure signatures and fixes

| Symptom | Cause | Fix |
|---|---|---|
| `Bad state: Content hash on Dart side (...) is different from Rust side (...)` | Dart-side generated bindings and the compiled Rust lib are out of sync (e.g. hot-reloaded without a full rebuild) | Full clean rebuild: `flutter clean && rm -rf rust/target build && flutter pub get && <regen + build>` |
| `error: linking with 'cc' failed` / `unable to find library -l<name>` | Missing system dev package for a native dependency | Install the matching `lib<name>-dev` (or platform equivalent); re-run build, no Cargo/code change needed |
| `cannot find type '<Type>' in this scope` in `frb_generated.rs`, or `SseEncode is not satisfied` | A `pub` function exposes a non-bridge-safe external type | See "pub vs pub(crate)" above — downgrade visibility / restructure the signature |
| Native library not found at runtime/build despite code being correct | Crate name doesn't match Cargokit's expected `rust_lib_<package_name>` pattern | Rename the crate in `Cargo.toml` (and the folder if it matches) to the expected pattern |
| Duplicate/conflicting `api/` files after running `integrate` | `integrate` scaffolds a demo module that collides with hand-written modules | Delete the generated demo files before adding real modules — don't merge |
| A newly added `pub` function doesn't appear in the generated Dart bindings | Codegen wasn't re-run after the Rust change | Re-run the bridge codegen generate step; this is required after every change to anything under `pub mod api` |

## Standard dev loop

After any change under the Rust `api` module tree:
1. Regenerate bridge bindings (`flutter_rust_bridge_codegen generate`, or the project's wrapper script if one exists — check for a `check.sh` or similar before running raw commands).
2. Build/run the Flutter app.

If a project provides its own build script wrapping these steps in the right order, prefer it over running the raw commands separately — order matters (codegen before Rust build before Flutter run) and a wrapper script usually encodes a fix for a specific ordering bug already hit on that project.

## Don'ts

- Don't manually copy compiled `.so`/`.dylib`/`.dll` files into place — Cargokit's build integration handles this; a missing-library error almost always means a missing system package or a crate-naming mismatch, not a missing manual copy step.
- Don't try to pass a raw SDK object (client, connection, session handle) across the bridge — convert to/from bridge-safe data (strings, primitives, plain structs) at the API boundary.
- Don't skip the full-clean-rebuild step when debugging a content-hash mismatch by trying partial fixes first — it wastes more time than it saves.
