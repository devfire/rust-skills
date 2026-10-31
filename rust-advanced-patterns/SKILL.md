---
name: rust-advanced-patterns
description: Advanced Rust guidance for concurrency/async, unsafe code, type-enforced invariants, library API and error design, dependency wiring, and performance. Use when writing or reviewing Rust involving tokio, shared state, `unsafe`/FFI, public traits or errors, DI/ports/test doubles, or production-grade design reviews. Pair with rust-defensive-programming.
---

# Advanced Rust Patterns

These patterns pay off in concurrent, unsafe, or multi-crate code. Each has a cost. **Apply one only when the problem in front of you has the failure mode it prevents**, and say which failure mode that is. Do not stack patterns to make code look sophisticated; a simple design that is correct beats a clever one.

Run `rust-defensive-programming` first for the baseline checklist, then use this for the areas below.

## Severity (report in this order)

1. **Soundness / data races / deadlocks / cancellation bugs**
2. **Public API mistakes that are hard to undo** (semver breaks, leaked dependency types)
3. **Performance and structure improvements**, only with measurements

Limit changes to what was asked. Verify with `cargo check`, `cargo clippy --all-targets`, `cargo test`, and where relevant `cargo miri test` or `loom`.

## Concurrency and async

### 1. Choose shared state deliberately
- **`Arc<Mutex<T>>` is often the right answer** for simple shared state with short critical sections. Tokio's own guidance says so. Use `std::sync::Mutex` (not `tokio::sync::Mutex`) when the guard is never held across an `.await`.
- **Actor/channel ownership** (one task owns the state, others send messages over `mpsc`) fits when: the state has a multi-step lifecycle, operations must be serialized with I/O in between, or you would otherwise hold a lock across `.await`.
- **Costs of actors to weigh:** bounded channels can deadlock in cycles, unbounded channels hide missing backpressure, and you must handle the actor task dying (`send` errors, `oneshot` reply dropped).
- Reach for `RwLock`, `DashMap`, or atomics only when contention is measured or the data is a simple counter/flag.
- **Channel kind by topology:** `tokio::sync::mpsc` is multiple-producer/single-consumer — cloning the sender adds producers, never consumers. `oneshot` for a single request/reply, `broadcast` for many consumers of every value, `watch` for many consumers of only the latest value, and an MPMC channel (`crossbeam-channel`, `flume`) for a multi-worker queue.

### 2. Never hold a lock guard across `.await`
A `std::sync::MutexGuard` across `.await` makes the future `!Send` or risks deadlock. Scope the guard in a block, clone the data out, or switch to the actor pattern.

### 3. Cancellation safety
Any future can be dropped at an `.await`. In `tokio::select!`, branches that lose the race are dropped. Check that partially completed work (half-read buffers, dequeued-but-unprocessed items) isn't lost. Use cancel-safe APIs, or move the work into a spawned task and select on its `JoinHandle`.

### 4. Don't block the runtime
CPU-heavy or blocking I/O inside `async fn` stalls the executor. Use `tokio::task::spawn_blocking`, or `rayon` for data-parallel CPU work called from a blocking context.

### 5. Backpressure and shutdown
Prefer bounded channels unless you can justify unbounded. Design graceful shutdown up front (`CancellationToken`, or dropping senders so receivers see `None`), and collect spawned tasks — a dynamic set with `JoinSet` (`join_next` as tasks finish), propagating `JoinError` rather than discarding it: a panicked task is a bug surfaced, not a result to drop. `JoinHandle::abort()` is not graceful shutdown: the future is dropped at its next yield point, `Drop` runs but no async cleanup does. Use it only when token-plus-join cannot work.
- **Limit concurrency with a semaphore, permit outside the spawn:** `Arc::clone` the `tokio::sync::Semaphore`, `acquire_owned().await` *before* `tokio::spawn`, and move the permit into the task. Acquiring inside an already-spawned task limits only execution — spawning and queued task state stay unbounded.

### 6. Atomics and `Send`/`Sync`
Default to `SeqCst`; weaken ordering only with a written justification and a `loom` test. A compound update must be a single read-modify-write — `fetch_add`/`fetch_update` or a `compare_exchange` retry loop; a separate `load`, modify, `store` races even though each step is atomic. Don't `unsafe impl Send/Sync` without a `// SAFETY:` argument.

## Type-system-enforced invariants

### 7. Typestate for protocols
`Connection<Disconnected>` has no `send()`; `.connect()` returns `Connection<Connected>`. Use for handshakes and builders where misuse is a realistic bug. Skip it for simple internal structs; a runtime enum is often clearer, especially when the state must be stored in collections.

### 8. Newtypes and `#[repr(transparent)]`
A plain newtype (`UserId(u64)`) is already zero-cost in practice; its value is preventing mix-ups. Add `#[repr(transparent)]` only when you need a **layout/ABI guarantee**: FFI signatures, or pointer casts/transmutes between the wrapper and inner type.

### 9. Sealed traits
```rust
pub trait Shape: private::Sealed { /* ... */ }
mod private { pub trait Sealed {} }
```
Use for a `pub` trait that only your crate should implement, so adding methods later isn't a breaking change. If downstream implementation is intended, don't seal.
Same trick seals constructors: a field of a type private to an inner module (`_seal: Seal` with `struct Seal;` inside `mod inner`, type re-exported via `pub use inner::S;`) makes struct-literal construction impossible outside that module — see `rust-defensive-programming` #8 for when to prefer this over `_private: ()` or `#[non_exhaustive]`.

### 10. `PhantomData` for units/markers
`Length<Meters>` vs `Length<Feet>` turns unit mix-ups into compile errors. Worth it when unit confusion is a real risk in the domain.

## `unsafe` discipline

### 11. Small, encapsulated, justified
- Keep each `unsafe` block minimal, and wrap it in a safe API.
- Put a `// SAFETY:` comment on every block, stating *every* invariant relied on: validity, alignment, initialization, aliasing, lifetime, and where each was established.
- Put a `# Safety` rustdoc section on every `unsafe fn` and `unsafe trait`, describing what the *caller* must guarantee.
- Weak: `// SAFETY: we checked len > 0` before `*ptr`. Better: state that `ptr` came from a live `&mut [T]` of length `len`, is properly aligned and initialized, and no other reference exists for the block's duration.
- Enable lints: `clippy::undocumented_unsafe_blocks`, `unsafe_op_in_unsafe_fn`. Test with Miri.
- Prefer a safe abstraction or a well-audited crate before writing new `unsafe`.

## Errors and API design

### 12. Match the error strategy to the audience
- **Library crates:** expose a concrete error type callers can inspect. `thiserror` enums are the usual choice; an opaque struct with a `kind()` accessor (as `std::io::Error`) is also legitimate and hides internals.
- **Binaries (anywhere in the binary crate, not only `main`):** `anyhow` with `.context(...)` is fine.
- **Library pitfalls:** add `#[non_exhaustive]` to public error enums so new variants aren't breaking; avoid `#[from]` on third-party error types that would leak dependency versions into your public API; implement `source()` so chains are preserved.
- **Panic vs `Result`:** panic for broken internal invariants (bugs), `Result` for anything the environment or caller can cause.

### 13. Flexible parameters, honest signatures
Accept `impl AsRef<str>`, `impl Into<String>`, or `&[T]` where callers benefit; prefer generics for hot paths and `dyn Trait` when compile-time or binary size matters or heterogeneous collections are needed. Don't add generics that have one call site.

### 14. `Cow` to defer allocation
When a function usually returns its input unchanged but sometimes must modify it (sanitizers, normalizers), return `Cow<'_, str>` so the common path doesn't allocate. Skip it when the function always allocates or callers need owned data anyway.

## Dependency wiring

### 15. Composition root and consumer-owned ports
- Construct concrete adapters (DB, HTTP, queue) in exactly one place — `main` or the framework layer. Domain/application code names only the trait it needs, never `PostgresRepo`, `PgPool`, or framework types.
- Define the port trait in the consumer's layer, shaped by the caller's needs. No adapter types in its signatures (`PgPool`, `sqlx::Error`, HTTP status types); its error is the consumer-layer error from #12, not `Box<dyn Error>` or `String`.
- A trait with one implementation and no test double is indirection without payoff: use the concrete type until substitution, runtime polymorphism, or a fake is actually needed.

### 16. Ownership form of an injected dependency
- `Box<dyn Trait>`: owned, one implementation chosen at wiring time. `&dyn Trait`: borrowed for a single call — use for per-operation deps (clock, transaction, caller context) instead of storing them in the struct.
- `Arc<dyn Trait + Send + Sync + 'static>`: shared across tasks/threads; required when the dependency is cloned into `tokio::spawn`. Async port traits are not dyn-compatible natively — `#[async_trait]` (boxed futures) or generic (static dispatch) parameters, chosen per #13.
- Store lifetime-long deps as struct fields via the constructor; pass operation-scoped deps as method arguments.

### 17. Scopes and test doubles — no DI container by default
- Translate DI vocabulary to ownership: **singleton** = one `Arc<T>` cloned around; **transient** = call the constructor/factory per use; **scoped** (per-request) = an explicit value created at the request boundary and passed down. No container magic, no thread-locals.
- A `TypeId`/`Any`-downcast container turns missing registrations into runtime `None`/panics, discarding compile-time wiring checks. Wire manually in the composition root; a container needs a named failure mode that manual wiring actually caused.
- Test doubles, in preference order: **fake** (in-memory impl with real semantics, e.g. `HashMap`-backed repository) > **stub** (canned answers) > **recording mock** (asserts calls; most brittle). Hand-rolled mocks recording through `&self` need `RefCell`, or `Mutex` when the double must be `Send`.

## Performance

### 18. Measure first
- Indexed loops are not inherently slower; LLVM often removes the bounds checks. The reason to prefer iterator chains is usually clarity and fewer panic sites. Only claim a speedup with a benchmark.
- Struct-of-arrays layouts, manual SIMD, `unsafe` unchecked indexing, and similar changes need profiling evidence (`criterion`, `cargo flamegraph`, `perf`). Without it, don't suggest them.
- Report before/after numbers when you make a performance change.

## Testing tools for this area

`loom` (concurrency interleavings), Miri (undefined behavior), `proptest`/`cargo fuzz` (parsers, invariants), `cargo-semver-checks` (public API). Suggest the one matching the risk you found.

## Workflow

- **Writing:** pick the simplest design that avoids the failure modes actually present.
- **Reviewing:** look for locks across `.await`, blocking in async, unbounded channels, spawned tasks nobody joins, `abort()` used as shutdown, semaphore permits acquired inside already-spawned tasks, `select!` cancellation, `unsafe` without justification, `anyhow` in a library crate, unsealed `pub` traits, adapter types in port-trait signatures, DI containers where manual wiring suffices, and unmeasured optimizations. Don't flag `Arc<Mutex<T>>` on its own.
- **Justify each suggestion** with the failure mode it removes, plus the cost it introduces. Keep review tone neutral and specific; describe tradeoffs, not the author's experience level.
- **Output:** a short list ordered by severity: location, risk, minimal fix, how to verify.
