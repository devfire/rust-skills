# Rust Skills

Agent skills for writing and reviewing Rust. Each skill is a directory containing a `SKILL.md` whose YAML frontmatter (`name`, `description`) determines when it triggers; the body is a severity-ordered checklist to apply — and to calibrate, not apply mechanically.

## Skills

| Skill | Use when |
|---|---|
| [`rust-defensive-programming`](rust-defensive-programming/SKILL.md) | Baseline for any Rust writing/review: compiler-enforced invariants, indexing, `unwrap`/`expect`, wildcard matches, `..Default::default()`, lossy casts, swallowed `Result`s, bool parameters, Clippy lints. **Start here.** |
| [`rust-advanced-patterns`](rust-advanced-patterns/SKILL.md) | Concurrency/async (tokio, shared state, cancellation), `unsafe`/FFI, type-enforced invariants, public API and error design, dependency wiring/ports/test doubles, performance. |
| [`rust-resilient-services`](rust-resilient-services/SKILL.md) | Cross-boundary failure in HTTP/gRPC clients/servers, queues, background jobs: retries, timeouts/deadlines, idempotency, circuit breakers/bulkheads, graceful shutdown, health checks, tracing/metrics. |
| [`rust-security-untrusted-input`](rust-security-untrusted-input/SKILL.md) | Untrusted input or secrets: deserialization, filesystem paths, uploads/size caps, tokens/passwords/crypto, SQL/regex, SSRF, XSS, JWT verification. |
| [`rust-testing`](rust-testing/SKILL.md) | Test layout and mechanics: unit/integration/doc tests, snapshots (`insta`), generated inputs (`proptest`), async/HTTP tests, test doubles, `cargo-nextest`/`cargo-fuzz`/`cargo-mutants`. |

## How they compose

The skills are layered; later ones assume the earlier ones and don't repeat them:

1. `rust-defensive-programming` — local invariants, always first.
2. `rust-advanced-patterns` — in-process async, `unsafe`, and API design.
3. `rust-resilient-services` — failure across process/network boundaries.
4. `rust-security-untrusted-input` — what an attacker does with input at those boundaries.
5. `rust-testing` — test mechanics and doubles; domain cases stay in the skill that owns them (fault injection in `rust-resilient-services`, malicious-input fixtures in `rust-security-untrusted-input`, `loom`/Miri in `rust-advanced-patterns`).

## Conventions shared by all skills

- **Calibrate before applying.** Tests, prototypes, and application-internal code get simpler versions of the patterns; sealed/public-API machinery is for published libraries.
- **Apply a pattern only when its failure mode is reachable**, and name that failure mode. No stacking patterns for sophistication.
- **Report findings in the skill's severity order** (soundness/correctness first, style last), scoped to the diff or area asked about.
- **Verify** with `cargo check`, `cargo clippy --all-targets`, and `cargo test` — plus the skill-specific check it names (Miri/`loom`, a fault test, malicious-input tests, or a mutation check).

## Adding a skill

Create `<name>/SKILL.md` with frontmatter:

```markdown
---
name: <name>
description: <what it covers and the concrete triggers for using it>
---
```

Write the `description` as trigger conditions, not a summary — it is what decides activation. Cross-reference existing skills instead of duplicating their content, and state explicitly what the new skill assumes from them.
