# TRIGGER_DEV_MASTER

Canonical Trigger.dev reference for engineers and coding agents.

Applies to: `@trigger.dev/sdk` `4.4.x` (validated against `4.4.3`), with migration notes for v3-era APIs where relevant.

---

## What Trigger.dev Is

Trigger.dev is a durable background execution platform for TypeScript/JavaScript workflows: define tasks in your codebase, deploy them as versioned workers, trigger runs from your app/backend, and orchestrate long-running work with retries, waits, queues, checkpoint/resume semantics, realtime updates, and operational controls (runs/queues/schedules/env vars/deployments) without Lambda-style timeout pressure.

---

## Core Primitives Glossary

- **Task**: versioned unit of work defined with `task()`; every trigger creates a **run**. See [`TRIGGER_DEV_TASKS.md`](./TRIGGER_DEV_TASKS.md).
- **schemaTask**: task with explicit runtime-validated input/output schema; best for strict API contracts.
- **Run**: one execution instance of a task (status, attempts, output/error, metadata, tags).
- **Attempt**: one try inside a run; retries create additional attempts.
- **Batch**: one API submission that creates many runs. See [`TRIGGER_DEV_MANAGEMENT_API.md`](./TRIGGER_DEV_MANAGEMENT_API.md).
- **Queue**: execution lane controlling concurrency and ordering. See [`TRIGGER_DEV_QUEUES_CONCURRENCY.md`](./TRIGGER_DEV_QUEUES_CONCURRENCY.md).
- **Concurrency key**: dynamic partitioning key that effectively creates per-key queue lanes.
- **Wait**: suspension point (`wait.for`, `wait.until`, `wait.forToken`) that checkpoints and later resumes.
- **Wait token / waitpoint token**: externally completable token that unblocks `wait.forToken`.
- **Schedule**: cron-based run generation (declarative or imperative).
- **Idempotency key**: deduplicates trigger/token creation operations to avoid duplicate side effects.
- **Tags**: up to 10 labels per run used for filtering and realtime subscriptions.
- **Metadata**: mutable run-associated JSON for progress/state exposure.
- **Machine preset**: CPU/RAM class (`micro`..`large-2x`) for task execution resources.
- **Streams**: realtime outputs and input streams for bidirectional task/frontend/backend communication. See [`TRIGGER_DEV_STREAMS.md`](./TRIGGER_DEV_STREAMS.md).
- **Public Access Token**: scoped, short-lived token for frontend-safe realtime access.

---

## Architecture Overview

1. **Definition-time (code):** export tasks from your task dirs (`dirs` in `trigger.config.ts`).
2. **Build-time:** Trigger CLI bundles task graph via `esbuild`; deploy builds container image.
3. **Deploy-time:** versioned deploy becomes available in target environment (DEV/STAGING/PROD/PREVIEW).
4. **Trigger-time:** backend (or another task) calls trigger API (`trigger`, `batchTrigger`, etc.).
5. **Queue-time:** run enters queue lane (task queue/custom queue/concurrency-key partition).
6. **Execution-time:** worker executes run attempt with retries/backoff policy.
7. **Checkpoint-time:** waits and `triggerAndWait` suspend + persist state; run resumes later.
8. **Observe-time:** dashboard + management API + realtime subscriptions expose status/trace/results.

> ⚠️ WARNING
> In deployed tasks, do not run `triggerAndWait()`/`batchTriggerAndWait()` concurrently via `Promise.all`. Trigger.dev docs explicitly call this unsupported.

---

## SDK Surface Area Map

### `@trigger.dev/sdk` (v4 main surface)

- Task authoring: `task`, `schemaTask`, `schedules.task`, `queue`, `wait`, `retry`, `logger`, `metadata`, `tags`, `idempotencyKeys`.
- Runtime orchestration: `myTask.trigger`, `.triggerAndWait`, `.batchTrigger`, `.batchTriggerAndWait`.
- Management-time namespaces: `runs`, `queues`, `schedules`, `envvars`, `auth`, `wait` token management, deployments APIs.
- Config: `defineConfig` in `trigger.config.ts`.

### `@trigger.dev/sdk/v3` and v3 management naming

- Package still exports v3-shaped APIs in some contexts; many management docs remain under `/api/v1` and `/api/v3` routes.
- v3→v4 migration gotchas:
  - `triggerAndWait`/`batchTriggerAndWait` return a `Result` object (use `.unwrap()` or `if (result.ok)`).
  - Queue behavior changed to explicit queue definitions (no implicit queue mutation pattern).

### Management-time vs Task-runtime APIs

- **Management-time (backend/server):** run listing/cancel/replay/reschedule, schedule CRUD, env vars CRUD/import, queue pause/resume/override, deployments, batch APIs.
- **Task-runtime (inside executing task):** `wait.*`, `retry.*`, metadata/tags/logging, subtask triggers.

See deep endpoint matrix in [`TRIGGER_DEV_MANAGEMENT_API.md`](./TRIGGER_DEV_MANAGEMENT_API.md).

---

## Authentication Model

### Key types

- **Secret keys** (`tr_dev_*`, `tr_stg_*`, `tr_prod_*`): primary backend auth for management and triggers.
- **Personal Access Tokens** (`tr_pat_*`): usable for some management APIs (not all task-trigger surfaces).
- **Public Access Tokens**: scoped JWTs for realtime/frontend-safe subscriptions and some token completion workflows.

### `configure()` reference

```ts
import { configure } from "@trigger.dev/sdk";

configure({
  secretKey: process.env.TRIGGER_SECRET_KEY, // or PAT when supported
  previewBranch: process.env.TRIGGER_PREVIEW_BRANCH,
  requestOptions: {
    retry: { maxAttempts: 3, factor: 2, minTimeoutInMs: 500, maxTimeoutInMs: 30_000, randomize: true },
  },
});
```

| Property | Type | Required | Default | Description |
|---|---|---:|---|---|
| `secretKey` | `string` | Usually | `TRIGGER_SECRET_KEY` | API auth credential for server-side SDK usage |
| `previewBranch` | `string` | No | `TRIGGER_PREVIEW_BRANCH` | Targets preview branch environment context |
| `requestOptions.retry` | object | No | SDK default retries | Global HTTP retry tuning for management API calls |

> ⚠️ WARNING
> Never expose secret keys or PATs in browser code. Use backend-issued Public Access Tokens for frontend realtime usage.

---

## Configuration Reference (`trigger.config.ts`)

```ts
import { defineConfig } from "@trigger.dev/sdk";
import { syncVercelEnvVars, prismaExtension } from "@trigger.dev/build/extensions/core";

export default defineConfig({
  project: "proj_xxx",
  dirs: ["./trigger"],
  retries: {
    enabledInDev: false,
    default: { maxAttempts: 3, minTimeoutInMs: 1_000, maxTimeoutInMs: 10_000, factor: 2, randomize: true },
  },
  runtime: "node-22",
  defaultMachine: "small-1x",
  maxDuration: 300,
  build: { extensions: [prismaExtension(), syncVercelEnvVars()] },
});
```

| Property | Type | Required | Default | Description |
|---|---|---:|---|---|
| `project` | `string` | Yes | - | Project ref (`proj_*`) |
| `dirs` | `string[]` | Recommended | auto-detect `trigger` dirs | Task discovery roots |
| `ignorePatterns` | `string[]` | No | excludes `*.test/*spec` | Build exclusion globs |
| `retries.default` | object | No | platform defaults | Default retry policy for tasks |
| `runtime` | `"node" \| "node-22" \| "bun"` | No | `node` | Runtime selection |
| `defaultMachine` | machine enum | No | `small-1x` | Default compute preset |
| `maxDuration` | `number` seconds | No | platform default | Global run max duration cap |
| `onStart/onSuccess/onFailure/init` | function | No | none | Global lifecycle hooks |
| `build.external` | `string[]` | No | none | Keep packages out of bundle |
| `build.extensions` | extension[] | No | none | Build/container customization |

Notable built-in extensions: `additionalFiles`, `additionalPackages`, `prismaExtension`, `syncEnvVars`, `syncVercelEnvVars`, `syncSupabaseEnvVars`, `puppeteer`, `ffmpeg`, `aptGet`, `esbuildPlugin`, `python`.

---

## Deployment and CLI

Common commands:

```bash
npx trigger.dev@latest login
npx trigger.dev@latest init
npx trigger.dev@latest dev
npx trigger.dev@latest deploy --env staging
npx trigger.dev@latest deploy --dry-run
npx trigger.dev@latest whoami
```

Key env vars:

- `TRIGGER_SECRET_KEY` - environment-specific secret key for runtime/backend calls.
- `TRIGGER_ACCESS_TOKEN` - CI-friendly PAT for non-interactive deploy.
- `TRIGGER_API_URL` - override API base URL (self-hosted).
- `TRIGGER_PREVIEW_BRANCH` - preview branch targeting.

See [`TRIGGER_DEV_DEPLOYMENT.md`](./TRIGGER_DEV_DEPLOYMENT.md).

---

## Limits and Quotas (Cloud)

Snapshot from Trigger.dev limits docs (cross-checked with `llms-full`):

- API rate limit: **1,500 requests/min**.
- Batch size: **1,000 items** on SDK `4.3.1+` (500 prior).
- Payload/output: single trigger payload **3MB**, batch item **3MB**, output **10MB**.
- Max run TTL: **14 days** cloud cap.
- Tags: max **10** tags/run, each <= **128 chars**.
- Query API: **10s** max execution, **10,000 rows**, **3 concurrent queries/project**.

Detailed tier tables are in [`TRIGGER_DEV_DEPLOYMENT.md`](./TRIGGER_DEV_DEPLOYMENT.md) and referenced in queue/realtime sections.

> ⚠️ WARNING
> Some limit values are plan-dependent and can change; treat this doc as architecture guidance, and dashboard Limits page as runtime source of truth.

---

## Common Production Patterns

- Fan-out/fan-in via `batchTriggerAndWait` + result aggregation.
- Sequential orchestrators with `triggerAndWait().unwrap()`.
- Per-tenant isolation via shared custom queue + `concurrencyKey`.
- Human approval gates with `wait.createToken` + callback URL + `wait.forToken`.
- Exactly-once semantics with idempotency keys at trigger/token creation boundaries.
- AI streaming pipelines using task streams + realtime hooks.

Full cookbook: [`TRIGGER_DEV_PATTERNS.md`](./TRIGGER_DEV_PATTERNS.md).

---

## Source Gaps and Known Inconsistencies

- Local file `01-tasks-api/00-trigger.md` is mislabeled and duplicates “advanced usage” content.
- Local file `03-runs-api/06-add-tags-to-run.md` appears duplicated metadata content (not tags endpoint).
- Some auth snippets use `accessToken` naming while endpoint security tables specify `secretKey`.
- Where route details were not locally present, data was cross-checked from `llms-full` and marked as needed.

<!-- UNVERIFIED -->
Action item: verify machine preset CPU/RAM table in `https://trigger.dev/docs/machines` and add exact per-preset specs to this doc.
<!-- UNVERIFIED -->
Action item: diff this reference against the latest management index at `https://trigger.dev/docs/management/overview` (or `llms-full`) and add any missing query/body options.

---

## Appendix: Concept Map

### Primitive graph

- `task/schemaTask` -> creates **runs**
- `run` -> has **attempts**
- `run` -> assigned to **queue** (+ optional `concurrencyKey` partition)
- `run` -> can **wait** (`wait.for`, `wait.until`, `wait.forToken`)
- `wait.forToken` -> depends on **waitpoint token**
- `run` -> emits **logs/traces/metadata/tags/streams**
- `batch` -> creates many `run`s
- `schedule` -> creates run(s) over time
- `deploy version` -> determines task code used by new runs

### Two SDK surfaces

- **Definition/runtime surface**: write tasks and orchestration in task files.
- **Management surface**: backend/API operations for triggering, monitoring, mutating runs/queues/schedules/env vars/deployments.

### Deployment lifecycle map

`local dev` -> `bundle (esbuild)` -> `deploy image` -> `environment activation` -> `trigger runs` -> `observe + operate` -> `promote/rollback via deployment control`.

