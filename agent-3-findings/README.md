# Trigger.dev SDK v4.4.3 — Canonical Reference

> **SDK version:** 4.4.3 | **Generated:** 2026-03-29 | **Reconciled output from two independent research agents**

This folder contains the reconciled canonical reference documentation for the Trigger.dev SDK v4.4.3. It was produced by merging and correcting two independently-generated research sets (`agent-1-findings/` and `agent-2-findings/`) according to the reconciliation directive, with verified corrections applied from official Trigger.dev documentation.

---

## Files

| File | Description |
|---|---|
| [`TRIGGER_DEV_MASTER.md`](./TRIGGER_DEV_MASTER.md) | Master reference: architecture, SDK surface area, authentication, configuration, limits & quotas, troubleshooting, and cross-reference index. |
| [`TRIGGER_DEV_TASKS.md`](./TRIGGER_DEV_TASKS.md) | Task definition, schema tasks, lifecycle hooks, triggering methods, subtask patterns, metadata API, machine types, and quick reference. |
| [`TRIGGER_DEV_QUEUES_CONCURRENCY.md`](./TRIGGER_DEV_QUEUES_CONCURRENCY.md) | Queue definitions, concurrency limits, concurrency keys, per-tenant isolation, queue management API, checkpointing behavior, and priority. |
| [`TRIGGER_DEV_WAITS_TOKENS.md`](./TRIGGER_DEV_WAITS_TOKENS.md) | Duration waits, date waits, waitpoint tokens, HTTP callbacks, idempotency, and input streams. |
| [`TRIGGER_DEV_ERRORS_RETRIES.md`](./TRIGGER_DEV_ERRORS_RETRIES.md) | Retry configuration, `catchError` hook, `retry.onThrow()`, `retry.fetch()`, `AbortTaskRunError`, `onFailure`, OOM retry, and error handling quick reference. |
| [`TRIGGER_DEV_REALTIME.md`](./TRIGGER_DEV_REALTIME.md) | Public access tokens, React hooks (`useRealtimeRun`, `useRealtimeRunWithStreams`, `useRealtimeStream`), backend subscriptions, run status lifecycle, and streams integration. |
| [`TRIGGER_DEV_STREAMS.md`](./TRIGGER_DEV_STREAMS.md) | Output streams (`streams.define()`), input streams (`streams.input()`), `streams.pipe()`, Zod schemas, frontend hooks, backend subscriptions, and Vercel AI SDK integration. |
| [`TRIGGER_DEV_MANAGEMENT_API.md`](./TRIGGER_DEV_MANAGEMENT_API.md) | REST API endpoint inventory with confidence ratings, authentication matrix, ID prefixes, API version namespaces, pagination, error handling, and query API. |
| [`TRIGGER_DEV_DEPLOYMENT.md`](./TRIGGER_DEV_DEPLOYMENT.md) | CLI commands, build extensions, CI/CD workflows, GitHub App integration, atomic deploys, self-hosting, troubleshooting, and Node.js version matrix. |
| [`TRIGGER_DEV_PATTERNS.md`](./TRIGGER_DEV_PATTERNS.md) | Production patterns: fan-out/fan-in, sequential pipelines, per-tenant rate limiting, idempotent retries, human-in-the-loop, webhook callbacks, scheduled sync, realtime progress, error recovery, and M2M triggers. |
| [`TRIGGER_DEV_DECISION_TREE.md`](./TRIGGER_DEV_DECISION_TREE.md) | Decision trees for common design choices: task vs inline, trigger vs triggerAndWait, custom queues, wait types, batch strategies, and schedules. |

---

## How This Was Produced

1. Two independent agents researched the Trigger.dev SDK v4.4.3 from official docs and source materials.
2. A reconciliation directive specified base selection per file, 10 verified corrections, content presence requirements, and formatting standards.
3. A third pass merged the two sets, applied all corrections, normalized cross-references, and verified content completeness.

## Verified Corrections Applied

- `handleError` → `catchError` (current SDK hook name)
- Machine presets corrected: `large-1x` = 4 vCPU / 8 GB, `large-2x` = 8 vCPU / 16 GB, 10 GB disk for all
- Limits section fully replaced with verified values (concurrency, payloads, schedules, queue depth, rate limits, etc.)
- Priority: "higher = picked up sooner, FIFO within same priority" (not "seconds offset")
- M2M trigger: `tasks.trigger("task-id", payload, options)` — 3 positional args
- OOM retry (`retry.outOfMemory`) and `OutOfMemoryError` added
- Metadata API: `metadata.decrement()`, `metadata.parent.set()`, `metadata.flush()` documented; `metadata.stream()` deprecated
- `defineConfig` import: `@trigger.dev/sdk` for `trigger.config.ts`
- Source doc issues preserved from Agent 1 research
- All cross-references normalized to sibling `./TRIGGER_DEV_*.md` links

## UNVERIFIED Markers

Items marked with `<!-- UNVERIFIED -->` could not be confirmed from available documentation. Each marker includes an action item with a specific URL to check for verification.
