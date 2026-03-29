# Trigger.dev Canonical Reference Documentation

> Generated: 2026-03-29 | SDK Version: @trigger.dev/sdk@4.4.3

These files serve as the **authoritative layer above the individual endpoint-level docs** in `../01-api-docs/`. Read `TRIGGER_DEV_MASTER.md` first, then dive into the topic-specific files as needed.

## Files

| File | Description |
|------|-------------|
| [TRIGGER_DEV_MASTER.md](./TRIGGER_DEV_MASTER.md) | **Start here.** Full platform overview — primitives glossary, architecture, SDK surface map, auth, config, CLI, limits, and common patterns. |
| [TRIGGER_DEV_TASKS.md](./TRIGGER_DEV_TASKS.md) | Deep reference on `task()`, `schemaTask()`, scheduled tasks, lifecycle hooks, triggering methods, subtask patterns, and machine types. |
| [TRIGGER_DEV_QUEUES_CONCURRENCY.md](./TRIGGER_DEV_QUEUES_CONCURRENCY.md) | Queues, concurrency limits, concurrency keys, checkpointing slot behavior, deadlock prevention, and the queue management SDK. |
| [TRIGGER_DEV_WAITS_TOKENS.md](./TRIGGER_DEV_WAITS_TOKENS.md) | All wait mechanisms — duration, date, wait tokens, HTTP callbacks, token listing, checkpointing, idempotency, and input streams. |
| [TRIGGER_DEV_ERRORS_RETRIES.md](./TRIGGER_DEV_ERRORS_RETRIES.md) | Retry configuration, `catchError`/`handleError`, `retry.fetch()`, `retry.onThrow()`, `AbortTaskRunError`, `onFailure`, and error objects. |
| [TRIGGER_DEV_REALTIME.md](./TRIGGER_DEV_REALTIME.md) | Realtime API — public access tokens, React hooks, streams, run status lifecycle, backend subscriptions, and frontend integration. |
| [TRIGGER_DEV_MANAGEMENT_API.md](./TRIGGER_DEV_MANAGEMENT_API.md) | Full REST/SDK management API — endpoint inventory, auth, pagination, error handling, and advanced usage patterns. |
| [TRIGGER_DEV_DEPLOYMENT.md](./TRIGGER_DEV_DEPLOYMENT.md) | Environments, CLI commands, `trigger.config.ts`, build extensions, CI/CD, GitHub Actions, Vercel, preview branches, and self-hosted. |
| [TRIGGER_DEV_PATTERNS.md](./TRIGGER_DEV_PATTERNS.md) | Patterns cookbook — fan-out, sequential pipelines, per-tenant queues, idempotency, human-in-the-loop, webhooks, AI agents, and more. |
| [TRIGGER_DEV_DECISION_TREE.md](./TRIGGER_DEV_DECISION_TREE.md) | Decision trees — task vs inline, trigger vs triggerAndWait, custom vs default queue, wait.for vs wait.forToken, batch vs loop, schedules vs cron. |

## Notes

- Items marked `<!-- UNVERIFIED -->` could not be confirmed from the source docs or internet research — verify before relying on them.
- Cross-references between files use relative markdown links.
- Source API docs live in `../01-api-docs/`.
