---
name: rust-idiomatic-review
description: Use before calling any Rust code done, or when asked to review Rust code. Checks for unwrap()/panic! in non-test production paths, proper error propagation, ownership/borrowing cleanliness, unnecessary cloning, and clippy-level idiom issues.
---

# Rust idiomatic code review

Run this as a self-review pass on any Rust code you just wrote or are asked to review.

## Error handling

- **No `.unwrap()` or `.expect()` on a `Result`/`Option` in a production code path** (library/binary logic reachable from real input or runtime conditions). These are acceptable only in: tests, truly-infallible cases you can justify in a comment (e.g. a regex compiled from a string literal known at compile time to be valid), or genuine "this is a programmer bug if it happens" invariants — and even then, prefer `.expect("reason")` over bare `.unwrap()` so a panic says why.
- **Propagate errors with `?`**, not manual `match`/`if let Err` that just re-wraps and returns — use `?` plus `From`/`map_err` conversions to keep call sites short.
- A function that can fail should return `Result<T, E>`, not an `Option<T>` that silently discards *why* it failed, unless the caller genuinely doesn't need to know why (rare).
- At FFI/API boundaries (e.g. a `flutter_rust_bridge` function), errors should be converted into whatever error shape the boundary expects (often `Result<T, String>`) at the boundary itself — not deep inside internal logic, so internal functions keep using proper typed errors (or `anyhow`/`thiserror` if the project already uses one) and only the boundary function does the final conversion.
- Don't swallow errors with `let _ = fallible_call();` unless the failure is genuinely inconsequential and ideally logged — silent failure is worse than a visible one in almost every case.

## Ownership & borrowing

- Don't `.clone()` to work around a borrow-checker complaint without first checking whether a reference, a different scope, or restructuring the function would avoid the need to clone at all. A clone is sometimes the right, cheap answer (e.g. cloning an `Arc`-backed or otherwise reference-counted type is intentionally cheap) — but a clone of a large owned structure (e.g. a `Vec`, a `String`, a deeply nested struct) to dodge a lifetime issue is usually a sign the function's signature or the call site's structure should change instead.
- Prefer borrowing (`&T`/`&mut T`) over taking ownership in function signatures unless the function actually needs to own/consume/store the value.
- Prefer `&str` over `String` in function parameters when the function only needs to read the string.
- If a type is `Clone` only because something deep in the call chain needed one clone, consider whether a shared-ownership type (`Arc<T>`, `Rc<T>`) at the point where sharing is actually needed is a better fit than making the type cheaply-cloned-by-convention.

## API design

- New types/functions should default to `pub(crate)` or private, and only become `pub` (or cross an FFI boundary) when there's an actual external caller — don't widen visibility "just in case."
- Prefer `impl Trait` or generics over `Box<dyn Trait>` unless dynamic dispatch or heterogeneous collections are actually needed.
- Struct fields should be private with accessor methods unless the struct is a plain data-holder meant for direct field access (and that should be a deliberate choice, not a default).

## Async

- Don't block the async executor with synchronous blocking calls (`std::thread::sleep`, synchronous file/network I/O, heavy CPU-bound work) inside an `async fn` — use the async runtime's equivalents (e.g. `tokio::time::sleep`, `tokio::fs`) or `spawn_blocking` for CPU-bound work.
- Check that a shared resource accessed from multiple async tasks is wrapped appropriately (`Arc<Mutex<T>>`/`Arc<RwLock<T>>`/an actor pattern) rather than relying on incidental single-threaded execution.

## General idiom / clippy-equivalent checks

- No `match` with a single meaningful arm and a catch-all that could be an `if let`.
- Use iterator chains (`.map()`, `.filter()`, `.fold()`, etc.) over manual index-based loops where it's at least as readable — but don't force an iterator chain that becomes harder to read than a loop.
- Avoid needless `.to_string()`/`.to_owned()` calls where a borrow would do.
- If `cargo clippy` is available in the project (check for it in CI config or just try running it), run it and address its warnings rather than relying only on manual review — clippy catches many of the above mechanically and more besides.

## How to run this review

1. Re-read every `.rs` file touched in the current task.
2. Walk the checklist above against each file.
3. If `cargo clippy` is runnable in the project, run it and fold its output into the review.
4. Fix findings inline. If a fix would meaningfully change the function's public signature or behavior in a way that affects callers outside the current task's scope, flag it rather than changing it silently.
