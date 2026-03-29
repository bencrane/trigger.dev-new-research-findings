# TRIGGER_DEV_DECISION_TREE

Architecture decision tree for choosing Trigger.dev primitives.

---

## 1) Trigger.dev Task vs Inline Code

Use **Trigger.dev task** when any of these are true:

- operation may exceed request latency budget (> a few seconds),
- needs retries/backoff/idempotency and durable recovery,
- involves orchestration/waits/human approval/webhooks,
- requires run-level observability (trace/log/metadata/replay),
- can benefit from queue-based concurrency control.

Use **inline code** when:

- work is truly short and synchronous,
- failure can be handled in-request without durability,
- no need for background observability or replay semantics.

---

## 2) `trigger()` vs `triggerAndWait()`

- Choose `trigger()` for fire-and-forget side effects and throughput.
- Choose `triggerAndWait()` when parent logic must branch on child output before proceeding.

Heuristic:

- If parent response must include child output now -> `triggerAndWait`.
- If parent can continue asynchronously -> `trigger`.

---

## 3) Default Queue vs Custom Queue

- Use default queue for isolated task-level control.
- Use custom queue when multiple tasks must share a global concurrency/rate budget.
- Add `concurrencyKey` for per-tenant or per-resource serialization.

---

## 4) `wait.for` vs `wait.forToken`

- `wait.for` / `wait.until`: known delay/time-based orchestration.
- `wait.forToken`: external event or human decision must unblock execution.

Choose token wait when an external system/UI submits completion payload.

---

## 5) Batch Trigger vs Loop of Single Triggers

- Prefer `batchTrigger` when dispatching many independent runs in one burst.
- Use single trigger loop only for low volume or per-item custom transaction logic.

Batch advantages:

- lower API overhead,
- better rate-limit behavior,
- cleaner aggregate tracking with batch ids.

> ⚠️ WARNING
> Batch cap is 1,000 items per call on SDK `4.3.1+` (500 older).

---

## 6) Schedules vs External Cron

Use Trigger schedules when you want:

- schedule state visible in Trigger dashboard,
- environment-aware execution behavior,
- schedule CRUD from management SDK,
- easy dedup and run observability.

Use external cron when:

- organizational policy centralizes scheduling elsewhere,
- schedule source-of-truth must remain in another platform.

If creating schedules programmatically, include `deduplicationKey`.

---

## 7) Result Object Handling (`triggerAndWait`)

- Need explicit success/failure branch? check `result.ok`.
- Need throw-on-fail ergonomics? use `.unwrap()`.

Avoid:

- assuming raw output return from `triggerAndWait` (v4 returns `Result` wrapper).

---

## 8) Realtime Integration Choice

- Task progress/status only -> `useRealtimeRun`.
- Streamed output (LLM tokens/events) -> `useRealtimeStream` (or legacy `useRealtimeRunWithStreams`).
- Multi-run workflow per tenant -> `useRealtimeRunsWithTag`.

Always authenticate frontend with scoped Public Access Token.

---

## 9) Retry Strategy Choice

- transient/network/server errors -> retry enabled.
- deterministic business-invalid input -> `AbortTaskRunError` or `handleError` with `retry: false`.
- mixed error surfaces -> classify in `handleError`.

---

## 10) Deployment Target Choice

- `dev` for rapid task iteration/debug.
- `staging` before production cutover.
- `preview` for branch-isolated QA.
- `prod` for customer traffic.

Use atomic deploy patterns when app release and task version must move together.

Related:

- Canonical model: [`TRIGGER_DEV_MASTER.md`](./TRIGGER_DEV_MASTER.md)
- Patterns cookbook: [`TRIGGER_DEV_PATTERNS.md`](./TRIGGER_DEV_PATTERNS.md)

