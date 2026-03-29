# Trigger.dev Canonical Reference Set

Generated: 2026-03-29
Primary source: local `trigger.dev/` management API docs (50 files) plus Trigger.dev public docs and release metadata cross-checks.

## Files

- [`TRIGGER_DEV_MASTER.md`](./TRIGGER_DEV_MASTER.md) - Canonical mental model, glossary, architecture map, auth/config/CLI/deploy/limits, and concept map appendix.
- [`TRIGGER_DEV_TASKS.md`](./TRIGGER_DEV_TASKS.md) - Task authoring and orchestration reference (`task`, `schemaTask`, scheduled tasks, triggering APIs, hooks, subtasks).
- [`TRIGGER_DEV_QUEUES_CONCURRENCY.md`](./TRIGGER_DEV_QUEUES_CONCURRENCY.md) - Queue architecture, concurrency slots, concurrency keys, deadlock behavior, and queue management APIs.
- [`TRIGGER_DEV_WAITS_TOKENS.md`](./TRIGGER_DEV_WAITS_TOKENS.md) - Wait APIs (`wait.for`, `wait.until`, tokens, callback URLs, input streams) and checkpoint semantics.
- [`TRIGGER_DEV_ERRORS_RETRIES.md`](./TRIGGER_DEV_ERRORS_RETRIES.md) - Error model, retry/backoff configuration, `retry.*` helpers, abort semantics, and failure hooks.
- [`TRIGGER_DEV_REALTIME.md`](./TRIGGER_DEV_REALTIME.md) - Realtime subscriptions, run lifecycle, Public Access Tokens, React hooks, and stream integrations.
- [`TRIGGER_DEV_STREAMS.md`](./TRIGGER_DEV_STREAMS.md) - Dedicated streams reference (output streams, input streams, backend/frontend subscription and send patterns).
- [`TRIGGER_DEV_MANAGEMENT_API.md`](./TRIGGER_DEV_MANAGEMENT_API.md) - Backend Management API endpoint inventory, auth, pagination, and operational usage.
- [`TRIGGER_DEV_DEPLOYMENT.md`](./TRIGGER_DEV_DEPLOYMENT.md) - Environments, CLI, `trigger.config.ts`, build system/extensions, CI/CD, Vercel, atomic deploys, preview branches.
- [`TRIGGER_DEV_PATTERNS.md`](./TRIGGER_DEV_PATTERNS.md) - End-to-end production patterns cookbook (fan-out/fan-in, approval gates, M2M, retries, long-running AI pipelines).
- [`TRIGGER_DEV_DECISION_TREE.md`](./TRIGGER_DEV_DECISION_TREE.md) - Practical decision tree for choosing Trigger.dev primitives and architecture paths.

## Version scope

- Primary scope: `@trigger.dev/sdk` v4.4.x (validated against `4.4.3` metadata and Trigger.dev changelog).
- Legacy notes: v3/v4 migration differences are called out where materially relevant.

## Presence and completeness checks

- `TRIGGER_DEV_WAITS_TOKENS.md`: present; covers duration waits, date waits, wait tokens, callback completion, token listing, and input-stream wait flows.
- `README.md`: present; enumerates every canonical doc in this set and generation date/version scope.

