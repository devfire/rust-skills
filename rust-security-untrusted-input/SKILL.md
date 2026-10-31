---
name: rust-security-untrusted-input
description: Security guidance for Rust code handling untrusted input or secrets. Use when writing or reviewing Rust that deserializes external data (JSON, bincode, form, config, IPC), builds filesystem paths from user input, accepts request bodies or file uploads, handles or generates passwords/tokens/API keys/session IDs/signatures, chooses cryptography or randomness, builds SQL or other queries, compiles/runs regular expressions on external input, fetches user-supplied URLs, renders user content as HTML, or verifies JWTs. Trigger on serde structs without deny_unknown_fields or size limits, Path::join with user-controlled segments, read_to_end/read_to_string on untrusted streams, == comparison of tokens or MACs, predictably generated tokens, format! inside SQL, user-supplied regex, HTTP fetches of user URLs, dangerous_inner_html/inner_html/|safe, or secret types deriving Debug/Clone/Serialize. Pair with rust-defensive-programming, which covers only secret redaction in Debug (its #16); this skill covers the rest.
---

# Rust Security and Untrusted Input

Core idea: **every byte crossing a trust boundary is attacker-controlled until validated, bounded, and interpreted in exactly one context.** File contents, HTTP bodies, CLI args, environment variables, config files, queue payloads, and data read back from your own database (stored earlier from user input) are all untrusted. Service-to-service traffic is not automatically trusted.

Run `rust-defensive-programming` first for local invariants (this skill extends only its #16, secret redaction in `Debug`) and `rust-resilient-services` for timeouts/retries across boundaries; this skill covers what an attacker does with the input itself.

## Calibrate before applying

- **Trust boundary first:** name where the input came from and who controls it. A config file only the operator can write needs less defence than a public request body; a config file shipped in a user-uploaded archive needs the same defence as a request body.
- **Defence in depth, ordered by cost:** reject oversize before parsing, parse into a bounded wire type, validate into a domain type (`rust-defensive-programming` #8), and let each sink (filesystem, SQL, regex engine) apply its own rule. One layer failing must not be exploitable.
- **Tests, examples, local fixtures:** relaxed handling is fine, provided the same code path is not reachable with real external input.
- **Don't oversell a control:** note what it does not stop (see zeroize and canonicalization caveats below) instead of implying safety.

## Severity (report in this order)

1. **Injection / escape / disclosure:** SQL injection, XSS, path traversal, SSRF, token forgery or confusion, secrets leaked or guessable via timing or logs.
2. **Resource exhaustion:** unbounded allocation, decompression bombs, regex CPU — remotely triggerable denial of service.
3. **Hardening gaps:** unknown fields silently ignored, secrets clonable/serializable by accident.

Verify with `cargo check`, `cargo clippy --all-targets`, `cargo test`, plus malicious-input tests (see *Testing untrusted input*).

## Deserialization of untrusted data

### 1. Bound the input before and during `serde` parsing

`serde` derive happily allocates whatever the input describes: a `Vec`, `String`, or `HashMap<String, Value>` grows until the input ends. Validation *after* `from_slice` runs only after the allocation already happened.

- **Cap bytes first** (see #4): never `serde_json::from_slice` / `from_reader` on an unbounded source. Read at most `MAX + 1` bytes and reject when the extra byte exists — `Read::take(MAX)` alone silently truncates, which can turn an oversize payload into a different valid one.
- **Cap collections and strings semantically too:** byte caps bound memory, but a legal-size payload can still carry 100k elements. Check `len()` on vectors/maps/key strings immediately after parsing, or use a custom `Deserialize` that errors past the cap. Prefer fixed fields over open-ended `HashMap<String, Value>`; an arbitrary map accepts arbitrary keys, count, and nesting.
- **Nesting depth:** keep the parser's recursion limit on. `serde_json` errors past its default depth (128); do not raise or disable it for untrusted input, and prefer iterative or streaming parsing for legitimately deep formats.
- **Length-prefixed binary formats** (`postcard`, `bincode`, protobuf bytes): a length prefix is an allocation instruction. Deserialize from an already byte-capped slice (`postcard::from_bytes`) or use the format's own limit API instead of defaults, and validate embedded lengths before allocating — never `Vec::with_capacity(untrusted_len)`. `bincode` is unmaintained (RUSTSEC-2025-0141); prefer `postcard` (or `bitcode`/`rkyv`) for new code, and don't add it as a new dependency.
- **Parse into a wire type, then validate** into the domain type with `TryFrom`. Deserializing directly into persistence/domain structs couples your schema to the attacker's payload and skips validation.

### 2. `#[serde(deny_unknown_fields)]` on closed request types

Unknown fields are otherwise silently ignored, so a misspelled `is_admin` — or a smuggled field a later layer interprets — passes unnoticed.

```rust
#[derive(Deserialize)]
#[serde(deny_unknown_fields)]
pub struct CreateUserRequest {
    pub name: String,
    pub email: String,
}
```

- Apply it to **closed, versioned-by-you request DTOs**. Do *not* apply it where forward compatibility is the contract: payloads from newer clients, evolving third-party webhooks, or stored data that must survive schema additions. There, ignoring unknown fields is the feature; say so in a comment.
- It does not compose with `#[serde(flatten)]`: the flattened capture swallows unknown fields anyway, so the attribute gives false confidence. Avoid `flatten` of arbitrary maps on untrusted input; if you must flatten, validate the captured keys against an allowlist.
- Duplicate fields in derive already error; unknown-field rejection is the missing half, not a substitute for validation of the known fields.

## Filesystem paths

### 3. Never join user input onto a base path unchecked

`base.join(user)` escapes `base` two ways: `user = "../../etc/passwd"` climbs out, and an absolute `user` (`/etc/passwd`) discards `base` entirely. String checks (`contains("..")`, `starts_with`) are bypassable and operate on the wrong representation.

**Fix:** validate components — only plain names may survive:

```rust
use std::path::{Component, Path, PathBuf};

fn safe_join(base: &Path, user: &Path) -> Result<PathBuf, PathError> {
    if user.components().any(|c| !matches!(c, Component::Normal(_))) {
        return Err(PathError::Escape); // rejects .., /, root, prefix, .
    }
    Ok(base.join(user))
}
```

- Decode first, validate second: percent-decode / normalize the input *before* this check, or the check inspects a different string than the filesystem receives. Reject empty and separator-containing values when a single file name is expected (`Path::file_name()` must round-trip).
- **Reading an existing file:** `std::fs::canonicalize` the result and require `canonical.starts_with(canonical_base)`. This resolves symlinks, which component checks cannot see.
- **Caveats, state them:** canonicalize-then-open has a TOCTOU race — a symlink swapped between check and open escapes. For serving a directory tree to untrusted callers, prefer a capability filesystem (`cap-std` / `openat2`-style open-beneath) that enforces containment at open time instead of check-then-open.
- Same rule for archive entries: a zip/tar member name is a user path. Validate each entry with the same component rule before extracting (zip-slip), and cap entry count and total extracted size (#4).

## Size and resource caps

### 4. Every ingress has an explicit byte, count, and expansion cap

- **Bodies:** set the framework limit explicitly (e.g. `tower_http::limit::RequestBodyLimitLayer`, Axum's `DefaultBodyLimit`) rather than relying on a default that may differ per extractor. Reject early on `Content-Length` when present, but never trust it — count bytes as they stream, because chunked/lying headers bypass header checks.
- **Streams:** never `read_to_end` / `read_to_string` on an untrusted stream without a cap (`Read::take(MAX + 1)` and reject the extra byte). This includes response bodies from downstream services.
- **Expansion:** compressed size is not expanded size. When decompressing or extracting archives, cap the *output* bytes, the entry/file count, and per-entry size; a 42 KB zip bomb is a legal small request.
- **Derived counts:** cap pagination (`page_size.min(MAX)` server-side, never honour the client's unbounded value), batch sizes, uploaded file count, and header/field counts where the framework exposes them.
- **Allocation from declared sizes:** a length field, `Content-Length`, or capacity hint from the wire must be checked against a maximum *before* it sizes an allocation; otherwise one small header forces a huge allocation.

## Secrets

### 5. Secrets are a distinct type for their whole lifetime

`rust-defensive-programming` #16 covers redacted `Debug`. The rest of the lifetime leaks too: cloning, serializing, logging fields, error messages, and comparisons.

- Hold secrets in `secrecy::SecretString` / `SecretBox<T>`, or a newtype wrapping `zeroize::Zeroizing<T>` for temporaries. Access is explicit and greppable (`expose_secret()`), instead of a `String` that flows anywhere.
- **Do not derive** `Clone`, `Serialize`, or `Display` on secret types unless a call site genuinely needs it; each derive is a future leak. Errors and logs must never include the secret value — log that a credential was rejected, not which one was presented.
- `zeroize` / `ZeroizeOnDrop` overwrite memory on drop, including heap buffers. **Limits, say them:** they cannot erase copies already made (a `String` that reallocated, a serialized copy, a clone). Minimize copies and secret lifetime rather than trusting zeroization to clean up; swap, core dumps, and the OS are outside its reach entirely.
- Keep secrets out of URLs and query strings (logged by proxies and servers) — headers or body only. Never commit them; load from the environment or a secret store at startup and fail closed when absent.

### 6. Generate secrets from a CSPRNG, compare them in constant time, hash passwords with a password KDF

`==` on byte strings exits at the first difference, leaking match length to anyone who can time repeated attempts.

- Compare MACs, signatures, API tokens, and reset codes with `subtle::ConstantTimeEq` (`a.ct_eq(b)`), or the verifier provided by the crypto crate (`hmac`'s `verify_slice`, signature crates' `verify`) — never `==` on the raw values, and never hand-rolled early-exit loops. Compare fixed-length digests/tags where you can; `ct_eq` on differing lengths still reveals length through timing, so treat length as public or reject on length first only when length is not secret.
- **Passwords are never stored or compared directly.** Hash with Argon2id (default choice) via the `password-hash` / `argon2` crates with per-password salt; verification (`PasswordHash::verify_password`) is constant-time internally. Fast hashes (SHA-2, bcrypt with low cost, unsalted anything) are wrong even if compared in constant time. Never implement a KDF or a MAC yourself.
- **A password KDF is deliberately expensive CPU work** (tens of milliseconds and megabytes per call). Run hashing and verification in `spawn_blocking` (`rust-advanced-patterns` #4), bound concurrent verifications with a semaphore, and rate-limit unauthenticated login/registration per account and per client — otherwise the hash itself is a remotely triggerable denial of service (severity 2).
- Constant time matters only where an attacker can make repeated timed guesses (network auth, token checks). Don't pay for it on non-secret equality; flag its *absence* on secret comparisons, not its absence everywhere.
- Related timing leak, note when reviewing auth: unknown-user vs wrong-password should return the same response (and ideally comparable work, e.g. verifying against a dummy hash), or the endpoint enumerates valid accounts.
- **Generation is half the secret.** Session IDs, API keys, reset/invite tokens, nonces, salts, and IVs come from an OS-backed CSPRNG — `getrandom` / `rand`'s `OsRng` — with at least 128 bits of entropy for bearer tokens: 32 random bytes is the simple default. A UUIDv4 carries only 122 random bits, so don't use one where this bar applies. Never a non-cryptographic PRNG, a counter, a timestamp, or a generator seeded from time/pid/process state: a predictable token is an authentication bypass no comparison discipline fixes. Encode the full entropy (hex/base64url of the raw bytes); truncating to a short numeric code silently cuts it.
- **Choose cryptography, don't construct it.** Take AEAD, signatures, hashing, and TLS from maintained crates (RustCrypto family, `ring`, `rustls`) with their default constructions. Never assemble a cipher mode, padding scheme, MAC-then/encrypt-then ordering, or nonce-reuse scheme yourself, and never implement a KDF or MAC (above). A novel combination of vetted primitives is still novel cryptography.
- **Verifying JWTs and other signed tokens:** pin the accepted algorithm(s) on the server side — never take the algorithm from the token header (algorithm confusion, `alg: none`). With `jsonwebtoken`, build `Validation::new(Algorithm::…)` explicitly and set `set_audience` / `set_issuer`: `exp` is checked by default, `aud` and `iss` are not, so a token minted for another service is otherwise accepted. Never use `dangerous::insecure_decode` (or decode-without-verify) output for an authorization decision.

## Queries and interpreters

### 7. SQL values are always bound parameters; only identifiers may be interpolated, from an allowlist

```rust
// Wrong: attacker controls the value's meaning
let q = format!("SELECT * FROM users WHERE email = '{email}'");
// Right: value is data, never parsed as SQL
let user = sqlx::query_as!(User, "SELECT * FROM users WHERE email = $1", email)
    .fetch_one(&pool).await?;
```

- Every value — including numbers, `LIMIT` where the driver supports binding it, and values read back from the database earlier (second-order injection) — goes through `bind` / `params![]` / `&dyn ToSql`. String formatting, concatenation, or `format!` producing any part of a statement that includes a value is a finding, full stop.
- **Identifiers cannot be bound** (table names, column names, `ORDER BY` direction). Map user choice through a closed match to static strings, and interpolate only the match result:

```rust
let order_by = match sort {
    Sort::Name => "name",
    Sort::CreatedAt => "created_at DESC",
};
let q = format!("SELECT id, name FROM users ORDER BY {order_by}"); // safe: closed set
```

- `IN` lists: bind an array (`= ANY($1)` in Postgres) or generate exactly one placeholder per element and bind each; never join values into the statement text.
- `LIKE` patterns: bound parameters still let the user supply `%` / `_` wildcards. Escape them (`\%`, `\_`, `\\`, with an explicit `ESCAPE`) when the input is meant as a literal substring.
- Same discipline at every interpreter boundary, not just SQL: shell commands (argument arrays, never `sh -c` with interpolated strings), LDAP/XPath/template queries, and `format!`-built URLs (encode each segment). The sink defines the escaping; parameterize or encode at that sink.
- **Argument arrays stop shell injection, not option injection.** `Command::new("convert").arg(user_value)` never invokes a shell, but a value starting with `-` is still parsed as a flag by the child program (`convert`, `git`, `tar`, `ffmpeg` all have dangerous options). Where the program supports it, put `--` before user-supplied positional arguments; otherwise reject values starting with `-` or make them unambiguous paths (`./name`). The program name itself must never come from user input.

### 8. Treat regular expressions on untrusted input as a CPU budget

- Prefer the `regex` crate: it is a finite-automata engine with linear-time matching, so catastrophic backtracking (`(a+)+$`) does not apply. That guarantee disappears with backtracking engines (`fancy-regex`, PCRE bindings) — a user-influenced pattern there is a classic ReDoS. Flag backtracking engines on untrusted patterns or inputs.
- Linear time is not free: cost still scales with input length and pattern size. Cap the input length *before* matching, and bound user-supplied patterns: reject empty/oversize patterns and compile with `RegexBuilder::size_limit` and `dfa_size_limit` set below defaults when the pattern itself is external.
- Best case, don't accept patterns at all: accept a literal and `regex::escape` it, or a choice from a fixed set of precompiled patterns (`LazyLock<Regex>`), compiled once — never per request.
- A CPU-bound match cannot be cancelled by dropping a future or timing out the request (`rust-advanced-patterns` #4). Bound the work (input caps) instead of wrapping it in a timeout; if it must run on large inputs, isolate it in `spawn_blocking` so it cannot stall the executor.

## Outbound requests and rendered output

### 9. A user-supplied URL is a request from inside your network (SSRF)

Webhooks, image/link previews, "import from URL", OAuth/OIDC discovery, and PDF/HTML renderers that fetch subresources all make *your* server send a request to an address the attacker picked. From inside the network that reaches what the attacker cannot: cloud metadata (`169.254.169.254`, which hands out credentials), admin ports on `localhost`, internal services, and private ranges.

- **Parse, then allowlist:** parse with `url::Url`, require `https` (or an explicit scheme allowlist — never `file:`, `gopher:`, or whatever the HTTP client happens to support), and where the product allows it, an allowlist of hosts beats any denylist.
- **Check the resolved address, not the hostname string:** resolve the host yourself and reject every address that is loopback, private (RFC 1918, IPv6 ULA `fc00::/7`), link-local (`169.254.0.0/16`, `fe80::/10` — this covers metadata), unspecified, multicast/broadcast, CGNAT (`100.64.0.0/10`), and IPv4-mapped IPv6 forms of all of these (`::ffff:127.0.0.1`). String checks on the host miss decimal/octal/hex IP encodings (`http://2130706433/`) and DNS names pointing at internal addresses.
- **Pin the checked address (DNS rebinding):** if the client resolves the name again when connecting, the second answer can be `127.0.0.1`. Connect to the IP you validated — e.g. `reqwest::ClientBuilder::resolve(host, addr)` or a custom DNS resolver that applies the filter — while keeping the original `Host`/SNI.
- **Redirects re-open the hole:** disable them (`redirect::Policy::none()`) or re-run the full check on every hop with a hop cap. Also disable proxies inherited from the environment for this client, or the check validates the wrong connection.
- **Bound the fetch** like any other ingress (#4): connect and total timeouts, response byte cap, and content-type check; never return the raw response or detailed connection errors to the caller (they turn the feature into an internal port scanner).
- When stakes are high, run fetches from an egress-restricted network segment or proxy; the in-process filter is then defence in depth, not the only control.

### 10. User content in HTML is escaped by the renderer, and opt-outs are findings

- Templating that escapes by default (`askama`, `maud`, `minijinja` with HTML autoescape, Dioxus/Leptos text nodes and attribute bindings) is safe for text content. The risk is the explicit opt-out: `|safe` / `PreEscaped`, autoescape disabled, `dangerous_inner_html` (Dioxus), `inner_html` (Leptos), or `format!`-built HTML strings. Each one rendering anything user-influenced — including data read back from your database — is stored XSS.
- If users must supply rich text, sanitize with an allowlist sanitizer (`ammonia`) at render time, not a hand-written tag stripper or regex.
- Context matters beyond text: a user value in an `href`/`src` needs a scheme allowlist (`javascript:` URLs survive HTML escaping), and a value inside a `<script>` block or inline event handler needs JSON/JS encoding, not HTML escaping. Prefer passing data through `data-` attributes or a JSON endpoint.
- Set a `Content-Security-Policy` that forbids inline script as defence in depth, and serve uploaded files with `Content-Disposition: attachment` and an explicit, non-sniffed content type (`X-Content-Type-Options: nosniff`) — an uploaded `.html` or SVG served inline from your origin is XSS.

## Minimal dependency set

Add only what a finding needs: `secrecy` + `zeroize` (secret types), `subtle` (constant-time equality), `getrandom` / `rand` with `OsRng` (token generation), `argon2` / `password-hash` (passwords), RustCrypto / `ring` / `rustls` (cryptography and TLS), `regex` (safe engine), `cap-std` (directory-tree serving under untrusted paths), `url` (URL parsing before SSRF checks), `ammonia` (rich-text HTML sanitizing), `jsonwebtoken` with explicit `Validation` (JWTs). Each is small and audited; hand-rolling any of these behaviours is a finding in itself. Dependency hygiene (`cargo audit` / `cargo deny`) stays with `rust-defensive-programming`, *Beyond the code*.

## Testing untrusted input

- Keep a malicious-fixtures test set per boundary: `../` and absolute-path payloads (also percent-encoded and mixed-separator forms), unknown and duplicate JSON fields, a body one byte over the cap, a nested payload past depth limits, an archive entry escaping the target dir, a small archive expanding past the size cap.
- Injection probe: `"' OR 1=1 --"` (and per-sink equivalents) must be treated as a literal value — assert the query returns no row / the file is not found, not merely that nothing panicked.
- Secret handling: assert `Debug` output and error messages do not contain the secret value; this is a test, not an intention.
- Regex: time the pathological pattern for your engine (nested quantifiers, long near-miss input) at the maximum allowed input size; assert it completes within the request budget.
- SSRF: requests to `http://127.0.0.1/`, `http://169.254.169.254/`, `http://[::ffff:127.0.0.1]/`, a decimal-encoded IP, and an allowed host that redirects to an internal address must all be refused before any connection to the internal address is made.
- XSS: render `<script>alert(1)</script>` and a `javascript:` URL through every view that shows user content; assert the output contains the escaped text, not the live markup.
- Tokens: a JWT with `alg: none`, a different algorithm, a wrong `aud`, and an expired `exp` must each be rejected.
- Every parser of external bytes (file formats, protocol frames, config, wire types from #1) gets a `cargo fuzz` target whose only invariant is **never panic, never hang** on arbitrary input. Run it briefly in CI on merge and longer on a nightly schedule; a parser tested only with hand-picked malicious fixtures is tested against yesterday's payloads.
- Follow the permanent-test rules: these tests assert observable behaviour (rejected, contained, literal), not that a particular validation function was called.

## Workflow

- **Writing:** for each input, name its source, its byte/count caps, the wire type it parses into, and each sink it reaches. Apply the sink's rule (components for paths, binding for SQL, limits for regex, resolved-address checks for outbound fetches, escaping for HTML) at the sink, not by pre-sanitizing strings upstream.
- **Reviewing:** search for smells (`from_slice`/`from_str`/`from_reader` on external data, `deny_unknown_fields` absent on request DTOs, `Path::join` / `push` with non-literal input, `read_to_end` / `read_to_string`, `format!` inside query strings, `Command` args built from unchecked user values starting with `-`, `==` on tokens/hashes/MACs, tokens generated from counters/timestamps/non-CSPRNG sources, hand-assembled cipher/MAC/KDF constructions, `Regex::new` on external patterns, HTTP clients fetching user-supplied URLs without resolved-address checks or with redirects on, `dangerous_inner_html` / `inner_html` / `|safe` / `PreEscaped` on user data, JWT `Validation` without pinned algorithm or audience, derived `Debug`/`Clone`/`Serialize` on secret-bearing types), then triage by severity above.
- **For each suggestion**, name the concrete attack it enables (escape, injection, exhaustion, disclosure) and the boundary it crosses. "Sanitize input" without a sink and a payload is not a finding.
- **Verify:** after edits run `cargo check`, `cargo clippy --all-targets`, and `cargo test`, including the malicious fixtures. Report anything you could not run.
- **Output format for reviews:** a short list ordered by severity, each item with location, the attack it enables, and a minimal fix. Don't dump the whole checklist.
