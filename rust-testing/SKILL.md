---
name: rust-testing
description: Guidance for writing, reviewing, and running Rust tests. Use when adding or reviewing tests in Rust, choosing between unit, integration, and doc tests, writing async or HTTP handler tests, or reaching for insta snapshots, proptest, cargo-nextest, cargo-fuzz, cargo-mutants, sqlx::test, or tempfile. Trigger on real sleeps or wall-clock timing in tests, tests that open sockets or ports, mock-heavy tests that assert calls instead of outcomes, unreviewed snapshot updates, env::set_var or fixed paths in tests, example-only tests for parsers or roundtrips, and CI that runs cargo nextest but never doc tests. Expands the test-double preference in rust-advanced-patterns (its #17); fault-injection cases stay in rust-resilient-services and malicious-input cases stay in rust-security-untrusted-input.
---

# Rust Testing

Core idea: **a test is a claim about observable behaviour, proven at the cheapest layer that can actually falsify it.** If a refactor that preserves behaviour breaks the test, the test was asserting implementation. If no plausible bug in the code under test can make it fail, it is decoration.

Domain-specific cases live elsewhere and are not repeated here: fault tests (duplicates, timeouts, shutdown) in `rust-resilient-services`, malicious-input fixtures in `rust-security-untrusted-input`, concurrency/UB tools (`loom`, Miri) in `rust-advanced-patterns`. This skill covers test mechanics and doubles.

## Calibrate before applying

- **Tests are code, with relaxed rules where the other skills say so:** `unwrap`/`expect` and indexing are fine in tests (`rust-defensive-programming`, Clippy notes). Panicking on a failed precondition is how tests report.
- **Assert outcomes, not choreography:** returned values, persisted state, emitted responses, error kinds. Call counts and sequences only when the interaction itself is the contract (#8).
- **Deterministic or it gets deleted:** no real sleeps, no wall-clock deadlines, no reliance on test order, ports, or shared mutable state. A flaky test trains the team to ignore red, which is worse than no test.
  - **Process-global state:** `std::env::set_var` / `remove_var` are `unsafe` in the 2024 edition because they race with concurrent reads, and `cargo test` runs tests on threads in one process. Inject configuration (pass a config struct or a lookup closure) instead of mutating the environment; the same applies to the current directory (`set_current_dir`).
  - **Filesystem:** use `tempfile::tempdir()` / `NamedTempFile` per test, never fixed paths under `/tmp` or the repo, which collide under parallel runs and leak state between runs. Keep the guard alive for the test's duration — dropping it deletes the directory.
- **One behaviour per test,** named for the claim (`expired_token_is_rejected`, not `test_token_2`). Parametrize or generate (via #3) instead of copy-pasting near-identical examples.

## Severity (report in this order)

1. **False confidence:** tests that cannot fail for a real bug — mock echoes, tautologies, snapshots accepted without reading the diff, assertions on values the test itself just configured.
2. **Flakiness:** real time, real network, shared state, order dependence. These erode trust in the whole suite.
3. **Wrong layer / slow:** HTTP round-trips for pure logic, unit tests reaching a real database, everything in integration tests because unit tests felt hard to set up.

Verify with the commands in #5, plus deliberately breaking the code once (mutate the behaviour, watch the intended test go red) for any test you suspect of being decorative. `cargo-mutants` automates this: it applies small mutations (flip a condition, return a default) and reports the ones no test catches. Run it scoped to changed code (`cargo mutants --in-diff`) rather than on the whole crate per commit; a surviving mutant is a named unproven claim.

## Test layout

### 1. Pick the layer by what you are proving

- **Unit tests** (`#[cfg(test)] mod tests` in the same file): prove internal logic and invariants. The child module sees the parent's private items, so internals can be tested without making them `pub`. Default choice for pure functions, state machines, parsers' internals, edge cases. Fast, many, narrow.
- **Integration tests** (`tests/*.rs`): prove the **public API** works as a consumer uses it. Each file compiles as its own crate against only the public surface — that friction is the point; it catches APIs that are unusable from outside. Shared helpers go in `tests/common/mod.rs` and are pulled in with `mod common;` (a bare `tests/common.rs`, or helpers as a direct child file of `tests/`, would be built as a test target of its own).
  - A binary-only crate (`src/main.rs`, no `src/lib.rs`) cannot be imported by integration tests. Keep the logic in a library crate and `main.rs` thin — tests are one more reason, alongside `rust-advanced-patterns` #15's composition root.
- **Doc tests** (examples in `///` docs): prove the documented example compiles and does what the docs claim. They are documentation first and coverage second — keep them short, one happy path, and do not bury bulk behaviour coverage in them; they run slowly and render in docs. Use hidden `#` lines for setup that would distract the reader. Reach for `compile_fail` to pin API misuse (a typestate or sealed-trait rejection is a behaviour worth testing), `no_run` when the example must compile but not execute, and `should_panic` rarely. Avoid `ignore`: ignored examples rot silently, which is worse than no example.

## Snapshots

### 2. `insta` for large structured output, explicit asserts for everything else

Snapshots pay off when the output is big and the *diff* is the reviewable unit: rendered CLI help, serialized JSON shapes, generated code, error reports, syntax trees.

- Review a snapshot update like a code change, because it is one: the accepted snapshot **becomes** the assertion. `cargo insta test --review` (or `--accept` after reading) exists so a human reads the diff; blind acceptance converts a regression into the new expected output. Never set `INSTA_UPDATE=always` in CI — CI runs with `INSTA_UPDATE=no` (or `cargo insta test --check`) so a mismatch fails instead of writing files. Unreviewed `.snap.new` files are never committed.
- **Redact nondeterminism** before it reaches the snapshot: timestamps, generated IDs, UUIDs, absolute paths. Use `insta` redactions (`assert_json_snapshot!(value, { ".id" => "[id]", ".created_at" => "[ts]" })`) and sort unordered collections first. A snapshot that changes every run is flaky by construction (severity 2).
- Do not snapshot what one `assert_eq!` says better: a status code, a single field, a short error string. Snapshot-everything suites produce thousand-line diffs nobody reads, which is severity 1 — the review step silently stops happening.
- Name snapshots explicitly (`assert_snapshot!("cli_help", out)`) so reordering tests does not rename and churn every file.

## Generated inputs

### 3. `proptest` when the claim is an invariant, examples when it is a case

- State the property, not the input: roundtrips (`parse(display(x)) == x`), never panics, idempotence, commutativity/ordering, equivalence with a simple model implementation. Parsers, serializers, arithmetic, and sanitizers are the natural targets — exactly the code where the interesting input is one you would not think to type.
- Generation must reach interesting values. `Arbitrary` (derive via `proptest_derive`, or hand-written strategies with `prop_compose!`) should produce *valid* domain values when the property assumes validity, plus a separate strategy for invalid ones. A property tested only over near-empty strings and small positive integers is example testing with extra steps.
- Commit the `proptest-regressions/` file: it records the seeds that once failed, replaying them before random generation. When shrinking finds a failure, also add the minimized case as a plain unit test so the bug has a fast, legible name.
- Tune case count to the budget: defaults locally, more via `PROPTEST_CASES` in CI. If a property needs minutes to say anything, narrow the strategy instead of accepting a slow suite.

### 4. `cargo-fuzz` for parsers that face bytes, run longer than CI can

`proptest` generates typed values inside `cargo test`; `cargo-fuzz` (libFuzzer, nightly) mutates raw bytes with coverage guidance for hours and finds the crash class property tests miss. They complement, not replace, each other.

- One fuzz target per external parser (file format, protocol frame, config, wire type): the invariant is **never panic, never hang, never OOM** on arbitrary bytes; if parsing succeeds, the roundtrip/validation properties from `rust-security-untrusted-input` apply to the result.
- Minimize any crash (`cargo fuzz tmin`) and land the failing input as a regression: a unit test or a seed in the corpus, so the bug stays fixed under plain `cargo test`, not only under the fuzzer.
- Schedule honestly: a short bounded run (`-- -max_total_time=60`) per merge is a smoke check, the real value is a long nightly (or OSS-Fuzz for open source) run with the corpus persisted between runs. A fuzz job whose corpus is discarded nightly restarts from zero every time.

## Running the suite

### 5. `cargo-nextest` for execution, `cargo test` still owns doc tests

- Nextest runs each test in its own process: faster wall-clock via aggressive parallelism, and isolation that exposes (rather than shares) global-state and environment leakage between tests. Tests that only pass under `cargo test`'s shared process, or only in declaration order, are flaky tests nextest just caught for you.
- **Nextest does not run doc tests.** A CI that only runs `cargo nextest run` silently stops testing every documented example. The pair is:

```sh
cargo nextest run --all-features
cargo test --doc
```

- Use nextest profiles (`.config/nextest.toml`) for CI policy: `slow-timeout` to kill hung tests instead of hung jobs, and test-groups to serialize the few tests competing for one scarce resource (a database, a fixed port you failed to eliminate) instead of serializing the whole suite.
- Retries are a flake *measurement*, not a cure: a retry profile in CI is acceptable only while the flaky test is being fixed, and its flake rate should be visible. A suite kept green by retries is severity 2 with paperwork.
- Keep plain `cargo test` working: contributors and IDEs run it, and it remains the reference for doc tests and for tests that depend on libtest behaviour nextest does not emulate.

## Async and HTTP

### 6. Paused Tokio time instead of real sleeps

- `#[tokio::test]` defaults to a current-thread runtime; use `flavor = "multi_thread"` only when the test is *about* cross-thread behaviour. Ordinary async logic tested multi-threaded just gains scheduling nondeterminism.
- For anything involving timers — timeouts, retries, debounce, TTLs — use paused time (requires Tokio's `test-util` feature, dev-dependencies only; only works on the current-thread runtime — `start_paused` with `flavor = "multi_thread"` panics):

```rust
#[tokio::test(start_paused = true)]
async fn cache_entry_expires() {
    let cache = Cache::new(Duration::from_secs(60));
    cache.insert("k", "v").await;
    tokio::time::advance(Duration::from_secs(61)).await; // no real waiting
    assert!(cache.get("k").await.is_none());
}
```

- With `start_paused = true` the clock also auto-advances to the next pending timer when the runtime goes idle, so `sleep`-based code completes without explicit `advance`. Two limits, both worth a comment when hit: only **Tokio's** clock pauses (`std::time::Instant`, external processes, and a real database keep real time — code reading those is not controlled by this test), and a task that never yields (busy loop) never lets the clock advance.
- Combine with injected backoff/clock where `rust-resilient-services` asks for it: paused time controls Tokio timers, injection controls values. What you must never do is `tokio::time::sleep(real_duration)` and a prayer — real-sleep timing assertions are the canonical flaky test.

### 7. Axum routers via `oneshot`: full stack, no socket

Most handler behaviour — routing, extractors, middleware order, status codes, headers, error mapping — does not need a network. Build the real `Router` with test state and drive it in-process (`tower` with the `util` feature, dev-dependencies):

```rust
use axum::{body::Body, http::{Request, StatusCode}};
use tower::ServiceExt; // for `oneshot`

#[tokio::test]
async fn create_user_returns_201() {
    let app = router(AppState::for_tests());
    let res = app
        .oneshot(
            Request::builder()
                .method("POST")
                .uri("/users")
                .header("content-type", "application/json")
                .body(Body::from(r#"{"name":"Ada","email":"ada@example.com"}"#))
                .unwrap(),
        )
        .await
        .unwrap();
    assert_eq!(res.status(), StatusCode::CREATED);
    let bytes = axum::body::to_bytes(res.into_body(), 1 << 20).await.unwrap();
    // assert on the parsed body, not on substrings
}
```

- `oneshot` consumes the router; build (or `clone`) a fresh `app` per request. State it with fakes (#8), so the whole handler path runs without a database.
- Know what this layer does **not** prove, and cover the gap with exactly one or two real-server smoke tests: socket binding and composition-root wiring, TLS/hyper server configuration, graceful shutdown under load, and real client behaviour. Those are wiring claims; the dozens of handler cases belong at `oneshot` speed.
- Do not drop below this layer to calling handler functions directly for cases where extractor rejection, middleware, or routing *is* the behaviour — a handler invoked by hand cannot fail the way the framework would make it fail.

## Test doubles

### 8. Fakes beat stubs beat mocks; assert interactions only when they are the contract

Expands `rust-advanced-patterns` #17's preference (fake > stub > recording mock); the wiring rules there (consumer-owned port traits, #15; stored vs per-call dependencies, #16) are prerequisites — you cannot fake a dependency you do not own a trait for.

- **Fake** — a working in-memory implementation with real semantics (`HashMap`-backed repository honouring uniqueness, ordering, and error cases). Default choice for repositories and stores. It is reusable across tests, survives refactors of the code under test, and lets tests assert *state outcomes* (`repo.get(id)` returns the saved value) instead of call choreography.
  - A fake that drifts from the real adapter is severity 1: it proves the code works against an implementation that does not exist. Keep fakes behaviourally faithful (same errors, same ordering guarantees) and, where the adapter matters, run one shared contract test suite against both the fake and the real adapter (the real one in integration tests, against a real database) so drift fails a test instead of shipping. With `sqlx`, `#[sqlx::test]` gives each test its own freshly created and migrated database (from `DATABASE_URL`), so real-database tests are isolated and parallel-safe without hand-written cleanup.
- **Stub** — returns canned answers, no semantics: a clock fixed at one instant, an ID generator yielding a sequence, a client returning one prepared response. Right for operation-scoped dependencies passed per call (`rust-advanced-patterns` #16) and for forcing one specific branch (the error path, the expired-token path) that a fake would make awkward to reach.
- **Mock / recording double** — asserts the interaction itself: expectations on calls, counts, or order (`mockall`, or a hand-rolled recording fake). Justified only when the call **is** the observable behaviour and there is no state to inspect afterwards: an email was sent exactly once, a charge was attempted, an external write did *not* happen on the validation-failure path, an event was published. Everywhere else it couples the test to internal call sequence, so a correct refactor goes red — severity 3 turning into severity 1 as the team learns to update expectations instead of reading failures.
- Never mock a type you do not own by wrapping a database or HTTP client in a trait *solely* to mock it; that path mocks away the SQL or the request shape, which was the behaviour worth testing. Test repositories against a real database (or the fake at the repository's own consumers, with contract tests keeping the fake honest), and HTTP handlers at the `oneshot` layer (#7).
- Sharing mechanics: a double behind `&self` records through interior mutability — `RefCell` for single-threaded tests, `Mutex` when it must be `Send` across tasks (from #17). Prefer returning state the test can read (`recorded.lock().unwrap().len()`) over expectation frameworks that fail inside the double, where the panic message points at the mock instead of the claim.

## Choosing a tool

| Need | Tool | Not |
|---|---|---|
| Internal logic, invariants, edge cases | Unit tests in `mod tests` | Integration tests "to be safe" |
| Public API as consumers see it | `tests/*.rs` | Doc tests as bulk coverage |
| Documented example correctness | Doc tests (`compile_fail` for misuse) | `ignore`d examples |
| Large structured output review | `insta` with redactions | Snapshots of everything, auto-accepted |
| Invariant over many inputs | `proptest` + regression seeds | More hand-typed examples |
| Crash/hang discovery on raw bytes | `cargo-fuzz`, short CI + long nightly | Fuzzing instead of properties |
| Fast, isolated suite execution | `cargo nextest run` **plus** `cargo test --doc` | Nextest alone |
| Timer, timeout, retry behaviour | `#[tokio::test(start_paused = true)]` | Real `sleep` |
| Handler, routing, middleware behaviour | Axum `oneshot`, fake state | Binding ports per test; calling handlers directly |
| Repository/store dependency | Fake, contract-tested against the real adapter | Mocking the database client |
| One forced branch, fixed clock/IDs | Stub | A fake with special cases |
| Real-database adapter tests | `#[sqlx::test]` per-test database | Shared database with manual cleanup |
| Files and directories | `tempfile` per test | Fixed paths |
| Configuration | Injected config | `env::set_var` in tests |
| Is this test decorative? | `cargo mutants --in-diff` | Coverage percentage alone |
| Side effect with no readable state | Recording mock, sparingly | Mocking to assert internal call order |

## Workflow

- **Writing:** state the claim in the test name, pick the cheapest layer from #1 that can falsify it, reach for a fake before a mock (#8), and make time, randomness, and IDs injected or paused before writing the first assertion.
- **Reviewing:** look for real sleeps and wall-clock assertions, `TcpListener`/port binding in handler tests, mocks asserting call order on internal steps, snapshot diffs accepted in the same commit that changed the code, properties whose strategies only generate trivial inputs, missing `cargo test --doc` beside nextest in CI, `env::set_var` or fixed filesystem paths in tests, and tests whose failure message would not tell you which behaviour broke.
- **Justify each suggestion** with the failure it exposes or the flake it removes. "Add more tests" without a named unproven claim is not a finding.
- **Verify:** run the suite the way CI does (#5), and for any new test guarding a fixed bug, confirm it fails on the pre-fix code. Report anything you could not run.
- **Output format for reviews:** a short list ordered by severity above: location, the false claim or flake risk, the cheapest layer or double that fixes it. Don't dump the whole checklist.
