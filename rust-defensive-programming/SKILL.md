---
name: rust-defensive-programming
description: Patterns for writing and reviewing defensive Rust that makes the compiler enforce invariants instead of relying on runtime checks or comments like "this should never happen". Use whenever writing, reviewing, or refactoring Rust, adding fields or variants to structs/enums, hand-implementing PartialEq/Hash/Debug/Clone, writing constructors or From/TryFrom conversions, handling errors, or configuring Clippy lints. Especially relevant when the code contains slice indexing in non-obvious places, `..Default::default()`, wildcard `_ =>` match arms on your own enums, multiple bool parameters, `unwrap`/`expect` in non-test code, lossy `as` casts, `let _ =` on a Result, or `From` impls that swallow failure. Also use when asked to "harden", "lint", or "make this more robust" for a Rust project.
---

# Defensive Programming in Rust

Core idea: **implicit invariants the compiler doesn't enforce are where bugs hide.** When you would write `// this should never happen`, ask how to make the type system or a compile error enforce it instead.

## Calibrate before applying

- **Application code** (all call sites under your control): prefer the simple version of each pattern. Sealed constructors and `#[non_exhaustive]` are for **public library APIs** that evolve over time.
- **Tests, scripts, prototypes, examples:** `unwrap`, indexing and `todo!()` are fine. Don't flag them.
- **Hot loops:** if profiling shows a bounds check matters, indexing behind a local, obvious length check is acceptable. Say so in a comment.
- **Review scope:** only touch code in the diff or the area the user asked about. Don't rewrite a whole codebase to satisfy the checklist.

## Severity (report in this order)

1. **Correctness / panic risk:** unchecked indexing on runtime data, `unwrap` on fallible input, lossy casts or overflow on untrusted values, swallowed errors, hand-written trait impls that miss fields.
2. **Silent-drift risk:** `..Default::default()`, wildcard match arms, `From` that hides failure.
3. **Readability / API quality:** bool parameters, unnamed `_` fields, missing `#[must_use]`, mutable-after-setup bindings.

## Checklist

### 1. Indexing (`v[0]`, `v[i]`)
The length check and the index are decoupled, so a refactor can turn one into a panic.
**Fix:** match on the slice so access exists only in the arm that proves the length:
```rust
match matching_users.as_slice() {
    [] => return Err(RepositoryError::NotFound),
    [existing_user] => { /* safe, no indexing */ }
    _ => return Err(RepositoryError::DuplicateUsers),
}
```
Also consider `.first()`, `.get(i)`, `.split_first()`, `.chunks_exact()`. Avoid `todo!()` in the empty arm of production code: it is a panic.

### 2. `..Default::default()` in struct literals
Newly added fields silently take defaults.
**Fix:** set every field, or destructure the default so new fields break compilation:
```rust
let Foo { field1, field2, field3, field4 } = Foo::default();
let foo = Foo { field1: value1, field2: value2, field3, field4 };
```
Config structs whose whole point is "override a few defaults" can reasonably keep the spread. Prefer a builder there.

### 3. Hand-written trait impls that name fields
`self.a == other.a && self.b == other.b` compiles after you add field `c`, and is probably wrong.
**Fix:** destructure fully so a new field is a compile error until you decide:
```rust
impl PartialEq for User {
    fn eq(&self, other: &Self) -> bool {
        let Self { id, name, cached_at: _ } = self;
        let Self { id: o_id, name: o_name, cached_at: _ } = other;
        id == o_id && name == o_name
    }
}
```
Same for `Hash`, `Debug`, `Clone`. Keep `Hash` consistent with `Eq`.

### 4. `From` that hides failure
`From` advertises infallibility. `unwrap_or_else(|_| default)` inside it swallows errors.
**Fix:** implement `TryFrom` and let callers handle `Err`. Panicking inside `From` is equally wrong.

### 5. Wildcard `_ =>` on enums you own
A new variant falls into the catch-all silently.
**Fix:** list every variant, grouping with `|`.
**Exception:** matching a *foreign* `#[non_exhaustive]` enum requires a wildcard. In that case use a deliberate arm (log, return an error) rather than ignoring it.

### 6. Anonymous `_` in destructuring
`Self::Rocket { _, _, .. }` hides what was skipped. Write `Self::Rocket { has_fuel: _, has_crew: _, .. }`.

### 7. Freeze mutable bindings after setup
Lightweight: `let data = data;` shadows and freezes after mutation. When temporaries must not leak, confine setup to a block:
```rust
let data = {
    let mut data = get_vec();
    data.sort();
    data
};
```
Minor. Apply when writing new code, not as a review finding.

### 8. Validation that can be bypassed by struct literals
If callers can write `Email { value: "" }`, validation is a convention.
**Fix (application code):** make fields private and expose a fallible constructor:
```rust
pub struct Email(String);
impl Email {
    pub fn parse(s: &str) -> Result<Self, EmailError> { /* validate */ Ok(Self(s.to_owned())) }
    pub fn as_str(&self) -> &str { &self.0 }
}
```
**Fix (public library):** additionally use `#[non_exhaustive]` on structs meant to gain fields, and seal traits you don't want downstream to implement (pattern: `rust-advanced-patterns` #9).
**Sealing boundary — pick by who must be blocked:** `_private: ()` blocks struct-literal construction outside the *module*; `#[non_exhaustive]` only blocks it outside the *crate* (underscore is convention; non-`pub` visibility does the work). Neither blocks same-module code — Rust privacy is module-level. If internal bypass matters, seal construction: put the type in a private `mod inner` with a private `struct Seal` field and `pub use inner::S;`, so only the constructor inside `inner` can build it. Same private-module trick as sealed traits, applied to a field instead of a supertrait.
Sealed construction is moot if fields stay `pub` (callers mutate past validation afterwards), so pair it with private fields + getters.

### 9. Easy-to-ignore return values
Annotate builders, guards, and result-like types with `#[must_use = "reason"]`. Put it on the *type* when discarding the value itself is the bug (e.g. a builder `Config` whose methods return `Self`): `#[must_use = "must be applied to take effect"] pub struct Config { /* … */ }`.

### 10. Multiple `bool` parameters
`process(&data, true, false, true)` is unreadable. Use enums (`Compression::Strong`), or a params struct with named presets (`ProcessParams::production()`).

### 11. Make illegal states unrepresentable
`is_active: bool` plus `expires_at: Option<DateTime>` allows nonsense combinations.
**Fix:** `enum Status { Active { expires_at: DateTime }, Inactive }`.
For lifecycles (connect before send), use typestate: `Conn<Closed>` has no `send`, and `.open()` returns `Conn<Open>`. Only use typestate when misuse is a realistic risk.

### 12. `unwrap` / `expect` / `panic!` on fallible paths
Reserve them for true bugs (a broken internal invariant), and prefer `expect("why this cannot fail")` over bare `unwrap`. Recoverable failures (I/O, parsing, user input) return `Result`.

### 13. Lossy numeric conversions and overflow
`as` silently truncates and wraps. Use `TryFrom`/`try_into()` for narrowing, and `checked_*`, `saturating_*`, or `wrapping_*` deliberately when arithmetic involves untrusted input or sizes. Note that release builds wrap on overflow by default.
As a safety net, also set `overflow-checks = true` under `[profile.release]` in `Cargo.toml`: per-operation discipline misses a spot eventually, and a silent wrap on a length or count is a bypassed bounds check. Caveat: this converts wrap into panic, which in a service is an availability trade — pair it with panic isolation per task (`rust-advanced-patterns` #5: propagate `JoinError`, never let one task's panic take the process), or scope the profile setting to the crates whose arithmetic guards memory safety.

### 14. Swallowed errors
`let _ = fallible();` and `.ok()` discard failures. Either handle, propagate with `?`, or log with a comment explaining why ignoring is safe.

### 15. Primitive obsession
Wrap IDs, units and validated strings in newtypes (`UserId(u64)`) so they can't be swapped. Derive only the traits that make sense.

### 16. Secrets in `Debug`
Types holding tokens or passwords should implement `Debug` manually and redact, so logs don't leak them.

### 17. Forbid unsafe where unneeded
Add `#![forbid(unsafe_code)]` to crates with no `unsafe`. Any future `unsafe` then requires a deliberate decision.

## Clippy lints

None of these are on by default. **Start with `warn`, fix the findings, then promote to `deny` in CI.** Turning all of them on as `deny` in an existing codebase produces a flood.

```toml
[lints.clippy]
# restriction group
indexing_slicing = "warn"
wildcard_enum_match_arm = "warn"
unneeded_field_pattern = "warn"
unwrap_used = "warn"
expect_used = "warn"
let_underscore_must_use = "warn"
# pedantic group
fn_params_excessive_bools = "warn"
cast_possible_truncation = "warn"
# nursery group
fallible_impl_from = "warn"   # catches panics inside From, NOT unwrap_or_else fallbacks (review #4 manually)
```

Notes:
- `must_use_candidate` (pedantic) is very noisy. Enable it only for library crates.
- `arithmetic_side_effects` is powerful but noisy; enable for code parsing untrusted input.
- Tests: set `allow-indexing-slicing-in-tests = true`, `allow-unwrap-in-tests = true`, `allow-expect-in-tests = true` in `clippy.toml`.
- Rust 1.74+ supports the `[lints]` table; older toolchains need `#![warn(...)]` crate attributes.

## Beyond the code

For a full "harden" pass, also suggest: `cargo deny` / `cargo audit` for dependencies, property tests (`proptest`) for parsers and invariants, and Miri if any `unsafe` exists.

## Workflow

- **Writing new code:** apply the checklist as you write.
- **Reviewing:** search for smells (`\[0\]`, `\.\.Default::default\(\)`, `_ =>`, `unwrap`, ` as u`, `let _ =`, multiple `bool` params), then triage by severity above.
- **For each suggestion**, name the implicit invariant being made explicit. That is the point, not style.
- **Verify:** after edits run `cargo check`, `cargo clippy --all-targets`, and `cargo test`. Report anything you could not run.
- **Output format for reviews:** a short list ordered by severity, each item with location, the invariant at risk, and a minimal fix. Don't dump the whole checklist.

## Source

Based on ["Patterns for Defensive Programming in Rust"](https://corrode.dev/blog/defensive-programming/) by Matthias Endler, extended with items 11-17.
