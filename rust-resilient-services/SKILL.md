---
name: rust-resilient-services
description: >-
  Resilience and observability guidance for Rust services that communicate across process or network boundaries. Use when writing or reviewing Rust HTTP/gRPC clients or servers, message-queue producers/consumers, background jobs, or service startup/shutdown involving retries, timeouts, deadlines, idempotency keys, duplicate delivery, circuit breakers, bulkheads, load shedding, downstream dependency failure, health/readiness checks, structured logging, metrics, or distributed tracing/OpenTelemetry. Trigger on: a retry loop without idempotency, jitter, a cap, or an overall deadline; an outbound call with no timeout; an in-memory idempotency cache; acknowledging a message before its side effect is durable; a liveness check that calls dependencies; user/request IDs used as metric labels or span names; or shutdown that aborts tasks before draining. Pair with rust-defensive-programming for local invariants and rust-advanced-patterns for in-process async/concurrency mechanics.
---

# Rust Resilient Services

These patterns govern failure **across boundaries**: a request may time out after it succeeded, a message may be delivered twice, a dependency may be slow rather than down, and the process may be told to exit mid-request. Apply a pattern only when that failure mode is actually reachable, and name it. A local function call needs none of this.

Run `rust-defensive-programming` for local invariants and `rust-advanced-patterns` for in-process async mechanics (`JoinSet`, cancellation safety, semaphore placement, lock discipline) first; this skill assumes both and covers only what they don't.

## Severity (report in this order)

1. **Duplicate or lost effects** (double charge, acked-but-unprocessed message, retry of a non-idempotent operation)
2. **Unbounded waiting, retrying, or queueing** (no timeout, no deadline, retry storms, unbounded drain)
3. **Unobservable or misleading operation** (no propagated context, high-cardinality labels, liveness that fails when a dependency does)

Verify with `cargo check`, `cargo clippy --all-targets`, `cargo test`, plus a fault test that exercises the failure named (see *Testing resilience*).

## Retries, timeouts, and deadlines

### 1. Retry only what is safe to repeat
- Retry an operation only if it is **naturally idempotent** (reads, deletes by ID, upserts to a known final state) or carries an **idempotency key** the server honours (#4). Otherwise a timeout means "outcome unknown," not "failed — try again": surface that ambiguity or reconcile with a follow-up read.
- Classify before retrying. Retryable: connection failures, timeouts, `429`, `502`/`503`/`504`. Not retryable: validation and other `4xx` (except `409` idempotency-in-progress handling in #4), authentication failures, deterministic domain rejections. When a framework retries for you, confirm its classification matches; default policies rarely know your domain errors.
- Parse failures on a success response are not a free retry: the server may have completed the operation. Retry only under an idempotency key, else reconcile first.

### 2. Bound every retry loop in time, not just attempts
- Exponential backoff with **full jitter** (`delay = random(0, min(cap, base * 2^attempt))`), a **cap**, and an **overall deadline** shared across attempts — a bare attempt count still permits pathological elapsed time when each attempt can block.
- Honour `Retry-After` on `429`/`503` when present; it overrides your computed backoff.
- Make the backoff sleep **cancellable** (`tokio::select!` against the shutdown/cancellation token). A task sleeping between retries must not delay shutdown.
- **Exactly one layer retries.** If the client retries, disable retries in the transport/framework beneath it and in callers above it; stacked layers multiply attempts geometrically. Where many callers share a dependency, prefer a retry budget (retries as a fraction of total requests) over per-caller generosity.
- Inject the clock/RNG or the backoff policy in tests; a retry loop that can only be tested by real sleeping will not be tested.

### 3. Every outbound call has a timeout inside a deadline
- No unbounded waits on network, queue, or dependency calls. Set a **per-attempt timeout**, and ensure attempt timeout × attempts + backoff fits the **overall deadline**.
- Propagate the caller's remaining deadline downstream (e.g. gRPC/HTTP timeout headers, context) instead of granting each hop a fresh full timeout; otherwise tail latency compounds hop by hop.
- Timeouts are cancellation: the dropped future's cleanup rules from `rust-advanced-patterns` #3 apply. Work that must not be interrupted mid-effect belongs in a task whose handle is awaited, not in the timed future itself.

## Idempotency

### 4. Server-side idempotency is a durable, atomic claim
The client generates **one key per logical operation**, reuses it across all retries of that operation, and never reuses it for a different one. If the client itself can restart mid-operation, the key must be persisted before the first attempt.
- Store the key in **durable storage shared by all replicas**, claimed **atomically with the effect**: a unique constraint on `(key)` inserted in the same transaction as the business mutation and the stored response; for an external side effect that can't join the transaction, forward the same key to the provider's own idempotency mechanism, or record intent first and reconcile (outbox/inbox pattern). An in-memory `HashMap` cache fails every requirement: lost on restart, invisible to other replicas, unbounded, and recorded after the effect, leaving a crash window that duplicates it.
- Fingerprint the payload (a hash of the canonical request). Same key + different payload is a **conflict error**, never a cached success.
- Model the record as a state machine: `in_progress` → `completed(response)` / `failed_final`. A duplicate arriving during `in_progress` gets the stored response when complete, an explicit in-progress response (e.g. `409`/`425` or wait briefly), **never a concurrent second execution**. Cache only deterministic final failures as `failed_final`; transient failures release the claim or mark it retryable.
- A crashed worker can strand `in_progress` forever: give the claim a lease/TTL longer than the maximum retry window but finite, after which a retry may reclaim it — and make sure the reclaimed execution is itself safe if the original was merely slow.

## Containing dependency failure

### 5. Fail fast and shed load instead of queueing collapse
- When a dependency is failing, a **circuit breaker** (per dependency/endpoint, counting dependency failures only — caller-caused `4xx` must not trip it) fails calls fast during a cooldown, then admits a small number of half-open probes. Don't hand-roll the state machine from loose atomics without tests for the open/half-open transitions.
- **Bulkhead:** bound concurrency per dependency (semaphore acquired before spawning — `rust-advanced-patterns` #5), with its own queue limit, so one slow dependency cannot consume every task/connection. Queueing beyond the bound must be rejected, not parked: when saturated, shed with `429`/`503` rather than accepting work you will time out anyway.
- Degrade deliberately where the product allows it (cached/stale reads, dropping optional enrichment) and make the degraded path observable (#9); silent degradation is an incident discovered by users.

### 6. Message delivery is at-least-once: design for duplicates
- Assume every message is delivered **at least once** and may be redelivered after partial processing. Consumers dedup with the same durable-claim discipline as #4 (an inbox record in the same transaction as the effect), not with an in-memory seen-set.
- **Ack/commit only after the side effect is durable** — never on receipt, on parse, or on successful spawn of the processing task. Acknowledge-after-spawn converts crashes into silent loss.
- Set the visibility/ack deadline above realistic processing time or heartbeat during long work; a message that times out mid-processing will be processed twice, which is why the dedup above is mandatory rather than optional.
- Poison messages: cap delivery attempts and route to a dead-letter queue with the failure recorded. Infinite redelivery of one bad message must not stall the partition/queue behind it.

### 7. Know your dependency's partition behaviour; never hand-roll consensus
- For every stateful dependency, know what it does during a partition: refuse service to preserve consistency, or serve possibly stale/divergent data. Your service inherits that choice; design explicitly for the stale-read or refused-write path, including how divergence is reconciled and how a caller knows data may be stale.
- Never implement Paxos/Raft/Zab or a lock-free replicated log yourself. Use the dependency's guarantees or a maintained consensus/coordination system (etcd, a Raft library). The algorithm being "conceptually simple" is how split-brain ships.

## Graceful shutdown

### 8. Shutdown is an ordered, deadline-bounded sequence
Task mechanics (tokens, `JoinSet`, `abort()`) live in `rust-advanced-patterns` #5; the service-level order is:
1. **Flip readiness to failing** and stop accepting new work (close listeners, stop polling consumers) while liveness stays green — failing liveness during a drain gets the process killed mid-drain.
2. **Signal cancellation** (`CancellationToken` or sender-drop) so loops and retry sleeps exit.
3. **Drain in-flight work under an overall deadline:** await handlers/tasks, let consumers finish and commit the current message, flush buffers/batches and telemetry.
4. **On deadline expiry,** abort remaining tasks as the last resort and exit non-zero or with a logged incomplete drain — hanging forever waiting for one stuck task is worse than a bounded, visible truncation.
- A second termination signal may force immediate exit, but treat `std::process::exit` as skipping every `Drop` and flush: it is the operator's emergency brake, not a step in the normal path.
- Keep liveness dependency-free (event loop is alive) and readiness honest (able to serve: dependencies reachable, not draining, not shed). A liveness check that calls the database restarts healthy processes during a database outage and turns it into a full outage.

## Observability

### 9. Structure first, propagate always
- Emit **structured** logs (`tracing` with fields, JSON to **stdout**; the platform ships and indexes them — the application does not open log files or talk to the aggregator). Initialize the subscriber once at startup with an environment-driven filter (`EnvFilter`). Log at boundaries with stable field names (`request_id`, `route`, `status`, `error.kind`); never interpolate IDs into the message string where a field belongs.
- Propagate trace context on **every** outbound call and extract it on ingress (W3C `traceparent` via the runtime's propagator/`opentelemetry` integration); a trace that breaks at one hop is two useless traces. Instrument boundaries (inbound handler, outbound client, queue produce/consume, DB) rather than every function; span per loop iteration is noise and cost.
- **Cardinality discipline:** span names and metric labels come from small closed sets (route *templates* like `/users/{id}`, status classes, dependency names). User IDs, request IDs, raw URLs, and error strings go in log/span **fields**, never in span names or metric labels — each distinct label value is a stored time series.
- Record failures on the span (status + error kind) before propagating them, and never log secrets, tokens, full payloads, or PII to "debug later" — redaction at the `Debug`/field level, per `rust-defensive-programming` #16. Sample traces if volume demands it, but keep errors observable: tail/error-biased sampling or full-fidelity error logs.
- Expose **RED** metrics per endpoint/dependency (request **R**ate, **E**rror ratio, **D**uration histogram) — histograms, not averages, so percentiles exist. Alert on symptoms (error ratio, p99 latency, saturation) rather than causes. The same failure should be findable from one ID: request/trace ID in the response, the logs, and the trace.

## Testing resilience

- Fault tests, not just happy paths: duplicate request with the same key (sequentially **and** concurrently), duplicate message delivery, kill/restart between side effect and ack/response, dependency that sleeps past the timeout, dependency returning `503` + `Retry-After`.
- Assert counts: exactly-once *effect* (one charge row, one created resource), bounded attempts, bounded elapsed time. "Eventually succeeded" hides duplicate effects.
- Exercise the shutdown path in a test: start a slow request, initiate shutdown, assert it drains (or is truncated at the deadline) and readiness flipped first.
- Deterministic time where possible (paused Tokio time, injected backoff) — resilience tests that rely on real sleeps are flaky and get deleted.

## Workflow

- **Writing:** for each boundary, state the failure you're designing for (timeout-after-success, duplicate delivery, dependency down, shutdown mid-flight) before choosing a mechanism.
- **Reviewing:** look for retry loops without idempotency/jitter/deadline, outbound calls without timeouts, fresh timeouts granted per hop, in-memory idempotency or dedup stores, ack-before-durable, liveness calling dependencies, shutdown that aborts before draining, hand-rolled consensus or circuit breakers, and request/user IDs in metric labels or span names.
- **Justify each suggestion** with the failure mode it removes and the cost it adds (latency, storage, operational surface). Describe tradeoffs, not the author's experience level.
- **Output:** a short list ordered by the severity above: location, failure mode, minimal fix, how to verify (which fault test).
