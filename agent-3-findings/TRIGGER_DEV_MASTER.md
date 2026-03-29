# Trigger.dev Master Reference

> **SDK version:** 4.4.3 | **Import path:** `@trigger.dev/sdk` | **Build package:** `@trigger.dev/build`

---

## 1. What Trigger.dev Is

Trigger.dev is an open-source background jobs and workflow engine for TypeScript that provides **durable execution** with automatic checkpointing, meaning your long-running tasks survive restarts, deployments, and infrastructure failures without losing progress. It eliminates serverless timeouts entirely -- tasks can run for seconds, minutes, hours, or days. The platform handles elastic scaling (spinning up isolated containers per task run), built-in retry with exponential backoff, queuing with per-task concurrency control, cron scheduling, and full observability via OpenTelemetry-based run logs. You define tasks in TypeScript, the CLI bundles them with esbuild, and they deploy into Trigger.dev's managed infrastructure (or your self-hosted instance) as versioned Docker images.

---

## 2. Core Primitives Glossary

| Primitive | Definition |
|---|---|
| **Task** | The fundamental unit of work. A named, exported async function with an `id`, optional retry/queue/machine config, and a `run()` handler. Defined via `task()` or `schemaTask()`. See [Tasks & Schema Tasks](#4-sdk-surface-area-map). |
| **Run** | A single execution of a task with a specific payload. Runs are versioned, retryable, and observable. Each run gets a unique ID (e.g., `run_abc123`). |
| **Attempt** | One try within a run. If a run has `maxAttempts: 5`, it can have up to 5 attempts. Each attempt is tracked independently. |
| **Queue** | A named concurrency-control mechanism. Each task gets a default queue, or you can define shared/per-tenant queues. Queues have a `concurrencyLimit` that caps how many runs execute simultaneously. See [Queues & Concurrency](./TRIGGER_DEV_QUEUES_CONCURRENCY.md). |
| **Schedule** | A cron-based recurring trigger for a task. Can be `DECLARATIVE` (defined in code) or `IMPERATIVE` (created via API). Supports timezones. See [Management API](./TRIGGER_DEV_MANAGEMENT_API.md). |
| **Wait / Waitpoint** | A durable pause mechanism. The process is checkpointed and freed while waiting. Types: `wait.for()` (duration), `wait.until()` (date), `wait.forToken()` (external callback). See [Waits & Tokens](./TRIGGER_DEV_WAITS_TOKENS.md). |
| **Batch** | A group of up to 1,000 task triggers sent in a single API call. Supports `batchTrigger()` (fire-and-forget) and `batchTriggerAndWait()` (collect all results). |
| **Environment** | An isolated deployment target. Types: `DEVELOPMENT` (local dev), `STAGING`, `PRODUCTION`, `PREVIEW` (per-branch). Each has its own secret key, versions, and concurrency limits. |
| **Version** | A snapshot of all tasks at deploy time. Format: `YYYYMMDD.N` (e.g., `20250228.1`). One version is "current" per environment. Runs are locked to a version at start. |
| **Deployment** | The act of bundling, uploading, and registering a new version of tasks. Done via `trigger.dev deploy`. |
| **Build Extension** | A plugin that hooks into the esbuild-based build system to customize bundling, add system packages, or modify the Docker image. Defined in `trigger.config.ts` under `build.extensions`. |
| **Metadata** | Arbitrary key-value data attached to a run that can be read and updated during execution. Useful for progress tracking and Realtime subscriptions. |
| **Tags** | String labels attached to runs for filtering and grouping. Can be added at trigger time or during execution. |
| **Idempotency Key** | A unique string that prevents duplicate runs. If two triggers share the same key within the idempotency window, only one run is created. |
| **Machine Preset** | The compute profile for a run (CPU + memory). Options: `micro`, `small-1x`, `small-2x`, `medium-1x`, `medium-2x`, `large-1x`, `large-2x`. See [Machine Presets](#machine-presets). |

---

## 3. Architecture Overview

### 3.1 Task Lifecycle

```
Define (TypeScript) -> Bundle (esbuild) -> Deploy (Docker image) -> Trigger (API/SDK)
   -> Queue -> Execute (isolated container) -> Checkpoint (on wait) -> Resume -> Complete
```

1. **Define**: Tasks are TypeScript files in directories specified by `dirs` in `trigger.config.ts`. Every task must be a **named export**.
2. **Bundle**: The CLI uses esbuild to bundle your task files. The `trigger.config.ts` `build` section controls externals, plugins, JSX, conditions, and extensions.
3. **Deploy**: The bundled code is packaged into a Docker image (or uploaded for remote build), registered as a new version, and promoted to "current".
4. **Trigger**: Your backend calls `myTask.trigger(payload)` or the REST API. The platform enqueues the run.
5. **Queue**: The run waits in its assigned queue. When a concurrency slot opens, execution begins.
6. **Execute**: A container starts (or reuses a warm process if `processKeepAlive` is enabled), loads the versioned code, and runs the `run()` function.
7. **Checkpoint**: If the task calls `wait.for()`, `triggerAndWait()`, or similar, the process state is checkpointed to storage and the container is freed.
8. **Resume**: When the wait condition is met, a new container loads the checkpoint and resumes execution exactly where it left off.
9. **Complete**: The run finishes (success or final failure after all retries). Output is stored and accessible via the API.

### 3.2 Build System

- **Bundler**: esbuild (with automatic JSX runtime support)
- **External packages**: Native binaries, WASM packages, and specific packages can be excluded from bundling via `build.external`
- **Build extensions**: Composable plugins from `@trigger.dev/build` that can add system packages (`aptGet`), copy files (`additionalFiles`), install npm packages (`additionalPackages`), add esbuild plugins, sync env vars, and more
- **Docker**: Deploy creates a Containerfile/Dockerfile. Remote builds are the default; local builds require Docker + Buildx installed

### 3.3 Environments

| Environment | Key Prefix | Description |
|---|---|---|
| `DEVELOPMENT` | `tr_dev_` | Local dev via `trigger.dev dev`. Each developer has their own. |
| `STAGING` | `tr_stg_` | Shared pre-production environment. Hobby/Pro plans only. |
| `PRODUCTION` | `tr_prod_` | Live production workloads. |
| `PREVIEW` | `tr_preview_` | Per-branch isolated environments. Requires `TRIGGER_PREVIEW_BRANCH` env var. |

Each environment has independent: secret keys, versions, concurrency limits, environment variables, and run history.

---

## 4. SDK Surface Area Map

### 4.1 `@trigger.dev/sdk` -- Main Package

This is the **only** import path you should use. Never import from `@trigger.dev/sdk/v3`.

#### Task Definition

```typescript
import { task, schemaTask } from "@trigger.dev/sdk";

// Basic task
export const processOrder = task({
  id: "process-order",
  queue: {
    name: "order-processing",
    concurrencyLimit: 10,
  },
  machine: "small-2x",
  maxDuration: 300, // seconds of compute time
  retry: {
    maxAttempts: 5,
    factor: 2,
    minTimeoutInMs: 1000,
    maxTimeoutInMs: 30_000,
    randomize: true,
  },
  run: async (payload: { orderId: string }, { ctx }) => {
    // ctx.run.id, ctx.environment.type, ctx.task.id, etc.
    return { processed: true, orderId: payload.orderId };
  },
  onStart: async ({ payload, ctx }) => { /* called when run starts */ },
  onSuccess: async ({ payload, output, ctx }) => { /* called on success */ },
  onFailure: async ({ payload, error, ctx }) => { /* called on final failure */ },
  catchError: async ({ payload, error, ctx, retryAt }) => {
    // Return { skipRetrying: true } or { retryAt: new Date() } to control retry behavior
  },
});

// Schema-validated task (Zod)
import { z } from "zod";

export const processVideo = schemaTask({
  id: "process-video",
  schema: z.object({
    videoUrl: z.string().url(),
    outputFormat: z.enum(["mp4", "webm"]),
  }),
  run: async (payload) => {
    // payload is typed as { videoUrl: string; outputFormat: "mp4" | "webm" }
    return { success: true };
  },
});
```

**`task()` options:**

| Option | Type | Description |
|---|---|---|
| `id` | `string` | **Required.** Unique task identifier. |
| `run` | `(payload, context) => Promise<T>` | **Required.** The task's execution function. |
| `retry` | `RetryOptions` | Retry configuration (see table below). |
| `queue` | `{ name: string; concurrencyLimit?: number }` | Queue assignment and concurrency. |
| `machine` | `MachinePreset` | Compute preset for this task. |
| `maxDuration` | `number` | Max compute seconds before the run is killed. |
| `onStart` | `(params) => Promise<void>` | Lifecycle hook: run started. |
| `onSuccess` | `(params) => Promise<void>` | Lifecycle hook: run succeeded. |
| `onFailure` | `(params) => Promise<void>` | Lifecycle hook: run failed (final). |
| `catchError` | `(params) => Promise<CatchErrorResult>` | Advanced error handling hook. |
| `middleware` | Via `tasks.middleware()` | Global or per-task middleware chain. |

**`RetryOptions`:**

| Option | Type | Default | Description |
|---|---|---|---|
| `maxAttempts` | `number` | `3` | Maximum number of retry attempts. |
| `factor` | `number` | `2` | Exponential backoff multiplier. |
| `minTimeoutInMs` | `number` | `1000` | Minimum delay between retries. |
| `maxTimeoutInMs` | `number` | `10000` | Maximum delay between retries. |
| `randomize` | `boolean` | `true` | Add jitter to retry delays. |

#### Triggering Tasks

```typescript
// === FROM YOUR BACKEND (Next.js route, Express handler, etc.) ===
import { tasks } from "@trigger.dev/sdk";
import type { processOrder } from "./trigger/orders";

// Fire and forget
const handle = await tasks.trigger<typeof processOrder>("process-order", {
  orderId: "ord_123",
});
console.log(handle.id); // run ID

// Batch trigger (up to 1,000 items)
const batchHandle = await tasks.batchTrigger<typeof processOrder>("process-order", [
  { payload: { orderId: "ord_1" } },
  { payload: { orderId: "ord_2" } },
]);

// === FROM INSIDE ANOTHER TASK ===
export const parentTask = task({
  id: "parent-task",
  run: async (payload) => {
    // Fire and forget -- does NOT version-lock the child
    await processOrder.trigger({ orderId: "ord_123" });

    // Trigger and wait for result -- version-locks the child
    const result = await processOrder.triggerAndWait({ orderId: "ord_456" });
    if (result.ok) {
      console.log(result.output); // { processed: true, orderId: "ord_456" }
    } else {
      console.error(result.error);
    }

    // Or use .unwrap() to get output directly (throws on failure)
    const output = await processOrder.triggerAndWait({ orderId: "ord_789" }).unwrap();
  },
});
```

> **WARNING:** Never wrap `triggerAndWait` or `batchTriggerAndWait` in `Promise.all`. This is not supported by Trigger.dev's execution model and will cause errors.

**Trigger options:**

| Option | Type | Description |
|---|---|---|
| `delay` | `string \| Date` | Delay the run start (e.g., `"1h"`, `"30m"`, a `Date`). |
| `ttl` | `string` | Time-to-live. If the run hasn't started within this window, it's discarded. |
| `idempotencyKey` | `string` | Prevent duplicate runs with the same key. |
| `queue` | `{ name: string }` | Override the task's default queue. |
| `concurrencyKey` | `string` | Dynamic per-key concurrency (e.g., per-tenant). |
| `tags` | `string[]` | Tags to attach to the run. |
| `metadata` | `Record<string, unknown>` | Metadata to attach to the run. |
| `maxAttempts` | `number` | Override retry max attempts for this run. |
| `version` | `string` | Pin the run to a specific deployed version. |
| `machine` | `MachinePreset` | Override the machine preset for this run. |

#### Batch Operations

```typescript
import { batch } from "@trigger.dev/sdk";

// Batch trigger multiple different task types
const results = await batch.triggerByTaskAndWait([
  { task: processOrder, payload: { orderId: "1" } },
  { task: processVideo, payload: { videoUrl: "https://...", outputFormat: "mp4" } },
]);

// Access typed results
const [orderResult, videoResult] = results.runs;
if (orderResult.ok) console.log(orderResult.output);
```

#### Wait / Waitpoints

```typescript
import { wait } from "@trigger.dev/sdk";

// Wait for a duration (checkpoints the process)
await wait.for({ seconds: 30 });
await wait.for({ minutes: 5 });
await wait.for({ hours: 1 });
await wait.for({ days: 7 });

// Wait until a specific date
await wait.until({ date: new Date("2025-06-01T00:00:00Z") });

// Wait for an external callback (human-in-the-loop, webhook, etc.)
const token = await wait.createToken({
  timeout: "1h",
  tags: ["approval:user_123"],
});
// Give token.url to an external system to POST to when ready
const result = await wait.forToken<{ approved: boolean }>(token);
console.log(result.ok, result.output); // { approved: true }
```

> **WARNING:** When a task hits a `wait`, the entire process is checkpointed and freed. Any in-memory state (database connections, open file handles, browser instances) will be lost. Use `tasks.onWait()` and `tasks.onResume()` hooks to manage stateful resources.

#### Queues & Concurrency

```typescript
import { queue } from "@trigger.dev/sdk";

// Define a shared queue
const emailQueue = queue({
  name: "email-sending",
  concurrencyLimit: 5,
});

// Use it in a task
export const sendEmail = task({
  id: "send-email",
  queue: emailQueue,
  run: async (payload: { to: string; subject: string }) => {
    // At most 5 of these run concurrently
  },
});

// Per-tenant concurrency (dynamic queues)
export const processTenantData = task({
  id: "process-tenant-data",
  run: async (payload: { tenantId: string; data: unknown }) => {
    // Each tenant gets its own concurrency slot
  },
});

// Trigger with concurrencyKey
await processTenantData.trigger(
  { tenantId: "tenant_abc", data: { /* ... */ } },
  { concurrencyKey: "tenant_abc" }
);
```

#### Schedules

```typescript
import { schedules } from "@trigger.dev/sdk";

// DECLARATIVE schedule (defined in code, auto-created on deploy)
export const dailyCleanup = schedules.task({
  id: "daily-cleanup",
  cron: "0 2 * * *", // 2am daily
  timezone: "America/New_York",
  run: async (payload) => {
    // payload.timestamp, payload.lastTimestamp, payload.timezone, etc.
    console.log("Running cleanup at", payload.timestamp);
  },
});

// IMPERATIVE schedule (created via API at runtime)
const schedule = await schedules.create({
  task: "daily-cleanup",
  cron: "0 */6 * * *", // every 6 hours
  deduplicationKey: "cleanup-schedule-prod",
  timezone: "UTC",
});

// List schedules
const allSchedules = await schedules.list();

// Update a schedule
await schedules.update(schedule.id, { cron: "0 3 * * *" });

// Delete a schedule
await schedules.del(schedule.id);
```

#### Runs Management

```typescript
import { runs } from "@trigger.dev/sdk";

// List runs with filtering
const completedRuns = await runs.list({
  status: ["COMPLETED"],
  taskIdentifier: "process-order",
  limit: 20,
});

// Auto-pagination
for await (const run of runs.list({ limit: 10 })) {
  console.log(run.id, run.status);
}

// Retrieve a specific run
const run = await runs.retrieve("run_abc123");

// Cancel a run
await runs.cancel("run_abc123");

// Replay a run (creates a new run with the same payload, locked to current version)
await runs.replay("run_abc123");

// Reschedule a delayed run
await runs.reschedule("run_abc123", { delay: "2h" });
```

#### Environment Variables

```typescript
import { envvars } from "@trigger.dev/sdk";

// List all env vars
const vars = await envvars.list("proj_ref", "prod");

// Create a new env var
await envvars.create("proj_ref", "prod", {
  name: "API_KEY",
  value: "sk-abc123",
});

// Bulk upload
await envvars.upload("proj_ref", "prod", {
  variables: { API_KEY: "sk-abc123", DB_URL: "postgres://..." },
  override: true,
});

// Delete an env var
await envvars.del("proj_ref", "prod", "API_KEY");
```

#### Other SDK Exports

| Export | Purpose |
|---|---|
| `configure(options)` | Manually set `secretKey`, `baseURL`, `previewBranch`, `requestOptions`. |
| `logger` | Structured logger that sends to Trigger.dev run logs. Levels: `debug`, `info`, `log`, `warn`, `error`. |
| `metadata` | Read/write arbitrary metadata on the current run. |
| `tags` | Add tags to the current run during execution. |
| `idempotencyKeys` | Create and manage idempotency keys for deduplication. |
| `retry` | `retry.onThrow()`, `retry.fetch()` -- fine-grained retry within a task. |
| `AbortTaskRunError` | Throw to immediately fail a run without retrying. |
| `auth` | Public access token generation for frontend Realtime subscriptions. |
| `locals` | Thread-local storage for passing data through middleware and lifecycle hooks. |
| `tasks.middleware()` | Register global middleware that runs before every task. |
| `tasks.onWait()` | Hook called when a run is about to be checkpointed (clean up resources). |
| `tasks.onResume()` | Hook called when a checkpointed run resumes (re-establish resources). |
| `queue()` | Define a named queue with a concurrency limit. |
| `batch` | `batch.triggerByTaskAndWait()` for heterogeneous batch operations. |

### 4.2 `@trigger.dev/sdk/build` -- Build Utilities

```typescript
import { defineConfig } from "@trigger.dev/sdk";
// Used in trigger.config.ts -- see Section 6
```

### 4.3 Management API vs. Task-Runtime API

| API Surface | Where It Runs | Authentication | Examples |
|---|---|---|---|
| **Management API** | Your backend server | `TRIGGER_SECRET_KEY` or PAT | `runs.list()`, `schedules.create()`, `envvars.upload()`, `tasks.trigger()` |
| **Task-Runtime API** | Inside a running task | Automatic (inherited from runtime) | `wait.for()`, `metadata.set()`, `tags.add()`, `logger.info()`, `childTask.triggerAndWait()` |

> **WARNING:** Functions like `wait.for()`, `wait.forToken()`, `triggerAndWait()`, and `batchTriggerAndWait()` can **only** be called inside a running task. Calling them from your backend will throw an error.

---

## 5. Authentication Model

### 5.1 Key Types

| Key Type | Prefix | Scope | Use Case |
|---|---|---|---|
| **Dev Secret Key** | `tr_dev_` | Single dev environment | Local development with `trigger.dev dev` |
| **Staging Secret Key** | `tr_stg_` | Staging environment | Triggering tasks against staging |
| **Prod Secret Key** | `tr_prod_` | Production environment | Triggering tasks in production |
| **Preview Secret Key** | `tr_preview_` | Preview environment | Preview branch testing (requires `TRIGGER_PREVIEW_BRANCH`) |
| **Personal Access Token (PAT)** | `tr_pat_` | All orgs/projects the user can access | CI/CD deployments, cross-project management API calls |

### 5.2 Automatic Configuration

```bash
# .env
TRIGGER_SECRET_KEY="tr_prod_abc123"
TRIGGER_PREVIEW_BRANCH="feature-xyz"  # Only needed for preview environments
```

The SDK reads `TRIGGER_SECRET_KEY` automatically. No `configure()` call needed.

### 5.3 Manual Configuration

```typescript
import { configure } from "@trigger.dev/sdk";

configure({
  secretKey: process.env.TRIGGER_SECRET_KEY,   // Never hardcode
  baseURL: "https://trigger.example.com",       // For self-hosted
  previewBranch: "feature-xyz",                 // For preview environments
  requestOptions: {
    retry: {
      maxAttempts: 5,
      minTimeoutInMs: 1000,
      maxTimeoutInMs: 5000,
      factor: 1.8,
      randomize: true,
    },
  },
});
```

### 5.4 Authentication Scopes by Endpoint

| Endpoint | Secret Key | PAT |
|---|---|---|
| `task.trigger` / `task.batchTrigger` | Yes | No |
| `runs.list` | Yes | Yes (requires `projectRef`) |
| `runs.retrieve` / `runs.cancel` / `runs.replay` | Yes | No |
| `envvars.*` | Yes | Yes (requires `projectRef` + env) |
| `schedules.*` | Yes | No |

### 5.5 Frontend Access (Realtime)

For browser-side Realtime subscriptions, generate a scoped public access token:

```typescript
import { auth } from "@trigger.dev/sdk";

// On your backend
const token = await auth.createPublicToken({
  scopes: {
    read: { runs: true },
  },
});
// Send token to frontend for Realtime subscription
```

---

## 6. Configuration Reference

The `trigger.config.ts` file lives at the root of your project:

```typescript
import { defineConfig } from "@trigger.dev/sdk";
import { prismaExtension } from "@trigger.dev/build/extensions/prisma";
import { syncVercelEnvVars } from "@trigger.dev/build/extensions/core";

export default defineConfig({
  // === Required ===
  project: "<your-project-ref>",         // Found in dashboard Project Settings

  // === Task directories ===
  dirs: ["./trigger"],                    // Where to find task files
  ignorePatterns: ["**/*.test.ts"],       // Globs to exclude

  // === Runtime ===
  runtime: "node",                        // "node" (21.7.3) | "node-22" (22.16.0) | "bun" (1.3.3, experimental)
  defaultMachine: "small-1x",            // Default machine for all tasks
  maxDuration: 60,                        // Default max compute seconds for all tasks
  logLevel: "info",                       // "debug" | "info" | "log" | "warn" | "error"

  // === Retry defaults ===
  retries: {
    enabledInDev: false,                  // Disable retries in dev mode
    default: {
      maxAttempts: 3,
      minTimeoutInMs: 1000,
      maxTimeoutInMs: 10000,
      factor: 2,
      randomize: true,
    },
  },

  // === Process management ===
  processKeepAlive: {
    enabled: true,                        // Reuse process for next task (warm start)
    maxExecutionsPerProcess: 50,
    devMaxPoolSize: 25,
  },

  // === Telemetry / Observability ===
  telemetry: {
    instrumentations: [
      // new PrismaInstrumentation(),
      // new OpenAIInstrumentation(),
    ],
    exporters: [/* OTLPTraceExporter instances */],
    logExporters: [/* OTLPLogExporter instances */],
    metricExporters: [/* OTLPMetricExporter instances */],
  },

  // === Global lifecycle hooks ===
  onStart: async ({ payload, ctx }) => {},
  onSuccess: async ({ payload, output, ctx }) => {},
  onFailure: async ({ payload, error, ctx }) => {},
  init: async ({ payload, ctx }) => {},

  // === Build ===
  build: {
    external: ["sharp", "re2"],           // Don't bundle these (native/WASM)
    autoDetectExternal: true,             // Auto-detect native dependencies
    keepNames: true,                      // Preserve function/class names
    minify: false,                        // Experimental minification
    jsx: {
      automatic: true,                    // Auto-import JSX runtime
      factory: "React.createElement",
      fragment: "React.Fragment",
    },
    conditions: ["react-server"],         // Custom import conditions
    extensions: [
      prismaExtension({ mode: "legacy", schema: "prisma/schema.prisma", migrate: true }),
      syncVercelEnvVars(),
      // additionalFiles({ files: ["./assets/**"] }),
      // additionalPackages({ packages: ["wrangler"] }),
      // aptGet({ packages: ["ffmpeg"] }),
      // ffmpeg(),
      // puppeteer(),
      // pythonExtension(),
      // playwright(),
      // emitDecoratorMetadata(),
      // esbuildPlugin(myPlugin, { target: "deploy" }),
    ],
  },

  // === Other ===
  tsconfig: "./tsconfig.json",            // Custom tsconfig path
  extraCACerts: "./certs/ca.crt",         // For self-signed certs
  enableConsoleLogging: true,             // Show console.log in dev terminal
  legacyDevProcessCwdBehaviour: false,    // Use build dir as cwd (matches prod)
});
```

### Build Extensions Reference

| Extension | Import | Purpose |
|---|---|---|
| `additionalFiles` | `@trigger.dev/build/extensions/core` | Copy files/globs to build output |
| `additionalPackages` | `@trigger.dev/build/extensions/core` | Install extra npm packages in deploy image |
| `aptGet` | `@trigger.dev/build/extensions/core` | Install Debian system packages via apt |
| `ffmpeg` | `@trigger.dev/build/extensions/core` | Install FFmpeg (default or v7 static build) |
| `syncEnvVars` | `@trigger.dev/build/extensions/core` | Sync env vars from external services at deploy time |
| `syncVercelEnvVars` | `@trigger.dev/build/extensions/core` | Sync env vars from Vercel projects |
| `syncSupabaseEnvVars` | `@trigger.dev/build/extensions/core` | Sync env vars from Supabase |
| `syncNeonEnvVars` | `@trigger.dev/build/extensions/core` | Sync env vars from Neon database |
| `prismaExtension` | `@trigger.dev/build/extensions/prisma` | Prisma client generation and migrations |
| `puppeteer` | `@trigger.dev/build/extensions/puppeteer` | Install Chromium for Puppeteer |
| `playwright` | `@trigger.dev/build/extensions/playwright` | Install Playwright browsers |
| `pythonExtension` | `@trigger.dev/python/extension` | Python runtime and pip dependencies |
| `emitDecoratorMetadata` | `@trigger.dev/build/extensions/typescript` | TypeScript decorator metadata (for TypeORM, etc.) |
| `esbuildPlugin` | `@trigger.dev/build/extensions` | Register custom esbuild plugins |
| `audioWaveform` | `@trigger.dev/build/extensions/audioWaveform` | Install BBC Audio Waveform tool |
| `lightpanda` | `@trigger.dev/build/extensions/lightpanda` | Install Lightpanda headless browser |

---

## 7. Deployment & CLI

### 7.1 CLI Commands

| Command | Description |
|---|---|
| `trigger.dev login` | Authenticate with Trigger.dev (opens browser) |
| `trigger.dev init` | Initialize a project: install SDK, create `/trigger` dir, create `trigger.config.ts` |
| `trigger.dev dev` | Run tasks locally. Watches for changes, runs each task in a separate Node process. |
| `trigger.dev deploy` | Bundle, build Docker image, upload, register new version (default: `prod`) |
| `trigger.dev deploy --env staging` | Deploy to staging environment |
| `trigger.dev deploy --env preview` | Deploy to preview branch (auto-detects git branch) |
| `trigger.dev deploy --skip-promotion` | Deploy without making the version "current" |
| `trigger.dev deploy --dry-run` | Build but don't deploy (inspect output) |
| `trigger.dev deploy --force-local-build` | Build Docker image locally instead of remote |
| `trigger.dev promote [version]` | Promote a previously deployed version to current |
| `trigger.dev preview archive` | Archive a preview branch |
| `trigger.dev whoami` | Display current user and project details |
| `trigger.dev update` | Update all `@trigger.dev/*` packages to match CLI version |
| `trigger.dev logout` | Log out of Trigger.dev |
| `trigger.dev list-profiles` | List all CLI login profiles |
| `trigger.dev switch [profile]` | Switch between CLI profiles |

### 7.2 Key CLI Environment Variables

| Variable | Purpose |
|---|---|
| `TRIGGER_SECRET_KEY` | Authenticate SDK calls (auto-detected by environment) |
| `TRIGGER_ACCESS_TOKEN` | Authenticate CLI in CI/CD (PAT, starts with `tr_pat_`) |
| `TRIGGER_API_URL` | Override API base URL (default: `https://api.trigger.dev`) |
| `TRIGGER_PREVIEW_BRANCH` | Branch name for preview environment |
| `TRIGGER_VERSION` | Pin all triggers to a specific deployed version |
| `TRIGGER_TELEMETRY_DISABLED` | Set to any non-empty value to disable CLI telemetry |
| `TRIGGER_BUILD_*` | Env vars exposed during GitHub integration builds (prefix stripped) |
| `TRIGGER_BUILD_NPM_RC` | Base64-encoded `.npmrc` for private registry auth in GitHub integration |

### 7.3 CI/CD (GitHub Actions)

```yaml
# .github/workflows/deploy-trigger.yml
name: Deploy to Trigger.dev (prod)
on:
  push:
    branches: [main]
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "20.x"
      - run: npm install
      - name: Deploy Trigger.dev
        env:
          TRIGGER_ACCESS_TOKEN: ${{ secrets.TRIGGER_ACCESS_TOKEN }}
        run: npx trigger.dev@latest deploy
```

> **WARNING:** The CLI and `@trigger.dev/*` package versions must be in sync. Version mismatches will cause deploy failures. Pin the CLI version in `devDependencies` for reproducible builds.

### 7.4 Atomic Deploys (with Vercel)

```bash
# 1. Deploy tasks without promoting
npx trigger.dev@latest deploy --skip-promotion

# 2. Deploy your app with the TRIGGER_VERSION env var
npx vercel --prod -e TRIGGER_VERSION=$DEPLOY_VERSION

# 3. Promote the task version
npx trigger.dev@latest promote $DEPLOY_VERSION
```

This ensures your app and tasks are always using matching versions.

### 7.5 Version Locking

- `trigger()` and `batchTrigger()`: child runs use the **current** version (not locked)
- `triggerAndWait()` and `batchTriggerAndWait()`: child runs are **locked** to the parent's version
- `TRIGGER_VERSION` env var: forces all triggers to use a specific version globally

---

## 8. Limits & Quotas

> These values are verified against https://trigger.dev/docs/limits as of March 2026. Values are plan-dependent; treat the dashboard Limits page as the runtime source of truth.

### 8.1 Concurrency Limits

| Plan | Max Concurrent Runs |
|---|---|
| Free | **10** |
| Hobby | **25** |
| Pro | **100+** (configurable) |

### 8.2 Payload & Output Sizes

| Limit | Value |
|---|---|
| Single trigger payload | **3 MB** max |
| Batch trigger payload | **3 MB** per item (SDK 4.3.1+); prior: 1 MB total combined |
| Task output | **10 MB** max |
| Offload threshold | **512 KB** (payloads/outputs above this get presigned URLs) |

### 8.3 Tags

- Max **10** tags per run.
- Each tag: **1–128 characters**.

### 8.4 Schedule Limits

| Plan | Max Schedules |
|---|---|
| Free | **10** per project |
| Hobby | **100** per project |
| Pro | **1,000+** per project |

### 8.5 Machine Presets

<a id="machine-presets"></a>

| Preset | vCPU | Memory | Disk | Use Case |
|---|---|---|---|---|
| `micro` | 0.25 | 0.25 GB | 10 GB | Lightweight tasks (API calls, notifications) |
| `small-1x` | 0.5 | 0.5 GB | 10 GB | General purpose (default) |
| `small-2x` | 1 | 1 GB | 10 GB | Medium compute (data processing) |
| `medium-1x` | 1 | 2 GB | 10 GB | Memory-intensive (large datasets) |
| `medium-2x` | 2 | 4 GB | 10 GB | Heavy compute (ML inference) |
| `large-1x` | 4 | 8 GB | 10 GB | Very heavy compute (video processing) |
| `large-2x` | 8 | 16 GB | 10 GB | Maximum compute (large models) |

### 8.6 Batch Limits

| Limit | Value |
|---|---|
| Batch size | **1,000** items (SDK 4.3.1+); **500** prior |
| Max run TTL | **14 days** (cloud). Runs without explicit TTL get 14 days; runs with TTL > 14 days clamped. |

**Batch trigger rate limits (token bucket):**

| Plan | Bucket Size | Refill Rate |
|---|---|---|
| Free | 1,200 runs | 100 runs / 10 sec |
| Hobby | 5,000 runs | 500 runs / 5 sec |
| Pro | 5,000 runs | 500 runs / 5 sec |

**Batch processing concurrency:**

| Plan | Limit |
|---|---|
| Free | 5 concurrent batches |
| Hobby | 10 concurrent batches |
| Pro | 50 concurrent batches |

### 8.7 Queue Depth (per queue, not per environment)

| Plan | Development | Staging/Production |
|---|---|---|
| Free | 500 | 10,000 |
| Hobby | 500 | 250,000 |
| Pro | 5,000 | 1,000,000 |

### 8.8 Rate Limits

- **API rate limit:** 1,500 requests/min per secret key.
- Rate-limited responses return HTTP 429 with `Retry-After` headers.
- The SDK automatically retries rate-limited requests with exponential backoff.

### 8.9 Realtime Connections

| Plan | Limit |
|---|---|
| Free | 10 concurrent |
| Hobby | 50 concurrent |
| Pro | 500+ concurrent |

### 8.10 Preview Branches

| Plan | Limit |
|---|---|
| Free | Not available |
| Hobby | 5 |
| Pro | 20+ |

### 8.11 Log Retention

| Plan | Limit |
|---|---|
| Free | 1 day |
| Hobby | 7 days |
| Pro | 30 days |

### 8.12 Organization Limits

| Limit | Value |
|---|---|
| Projects per org | **10** (all plans) |
| Team members | **5** / **5** / **25+** (Free/Hobby/Pro) |
| Alert destinations | **1** / **3** / **100+** (Free/Hobby/Pro) |

### 8.13 Query API

| Limit | Value |
|---|---|
| Max execution time | **10 seconds** |
| Max rows returned | **10,000** |
| Concurrent queries per project | **3** |
| Lookback window | **1d** / **7d** / **30d** (Free/Hobby/Pro) |

### 8.14 Log Size Limits

| Limit | Value |
|---|---|
| Span attribute count | 256 |
| Log attribute count | 256 |
| Span attribute value length | 131,072 chars |
| Span event count | 10 |
| I/O packet length | 128 KB |

### 8.15 Run Duration

- **No built-in timeout**: Runs can execute indefinitely by default
- **`maxDuration`**: Optional per-task or global limit on compute time (in seconds). This counts only active compute, not wait/checkpoint time
- **Wait time is free**: Time spent in `wait.for()`, `wait.until()`, or `wait.forToken()` does not count against `maxDuration`

---

## 9. Common Patterns

### 9.1 Fan-Out / Fan-In

Process many items in parallel, then collect results.

```typescript
import { task, batch } from "@trigger.dev/sdk";

export const processAllImages = task({
  id: "process-all-images",
  run: async (payload: { imageUrls: string[] }) => {
    // Fan-out: trigger processing for each image
    const results = await batch.triggerByTaskAndWait(
      payload.imageUrls.map((url) => ({
        task: processImage,
        payload: { url },
      }))
    );

    // Fan-in: collect all results
    const processed = results.runs
      .filter((r) => r.ok)
      .map((r) => r.output);

    return { totalProcessed: processed.length, results: processed };
  },
});

export const processImage = task({
  id: "process-image",
  machine: "medium-1x",
  run: async (payload: { url: string }) => {
    // Heavy image processing here
    return { url: payload.url, thumbnailUrl: "https://..." };
  },
});
```

### 9.2 Sequential Chain

Execute tasks in order, passing output from one to the next.

```typescript
export const orderPipeline = task({
  id: "order-pipeline",
  run: async (payload: { orderId: string }) => {
    // Step 1: Validate
    const validation = await validateOrder
      .triggerAndWait({ orderId: payload.orderId })
      .unwrap();

    // Step 2: Charge payment
    const payment = await chargePayment
      .triggerAndWait({ orderId: payload.orderId, amount: validation.total })
      .unwrap();

    // Step 3: Fulfill
    const fulfillment = await fulfillOrder
      .triggerAndWait({ orderId: payload.orderId, paymentId: payment.id })
      .unwrap();

    return { orderId: payload.orderId, trackingNumber: fulfillment.trackingNumber };
  },
});
```

### 9.3 Per-Tenant Queues

Isolate concurrency per customer so one tenant's workload doesn't block another.

```typescript
export const processWebhook = task({
  id: "process-webhook",
  queue: {
    name: "webhook-processing",
    concurrencyLimit: 1, // Per concurrencyKey
  },
  run: async (payload: { tenantId: string; event: unknown }) => {
    // Exactly one run per tenant at a time
  },
});

// Trigger with concurrency isolation
await processWebhook.trigger(
  { tenantId: "tenant_abc", event: webhookData },
  { concurrencyKey: "tenant_abc" }
);
```

### 9.4 Idempotent Retries

Prevent duplicate processing when your backend might trigger the same work twice.

```typescript
import { tasks, idempotencyKeys } from "@trigger.dev/sdk";
import type { sendInvoice } from "./trigger/invoices";

// In your API route handler
export async function handleInvoiceCreated(invoiceId: string) {
  // Same invoiceId = same idempotency key = only one run
  await tasks.trigger<typeof sendInvoice>("send-invoice", { invoiceId }, {
    idempotencyKey: `invoice-${invoiceId}`,
  });
}
```

### 9.5 Human-in-the-Loop Approval

Pause a task until a human approves via webhook or UI button.

```typescript
export const contentPipeline = task({
  id: "content-pipeline",
  run: async (payload: { articleId: string }) => {
    // Step 1: Generate content
    const draft = await generateArticle(payload.articleId);

    // Step 2: Create a waitpoint token for human approval
    const token = await wait.createToken({
      timeout: "24h",
      tags: [`article:${payload.articleId}`],
    });

    // Send token.url to your approval UI or Slack webhook
    await notifyEditor({
      articleId: payload.articleId,
      approvalUrl: token.url,
      draft,
    });

    // Step 3: Wait for approval (process is checkpointed here)
    const approval = await wait.forToken<{ approved: boolean; feedback?: string }>(token);

    if (!approval.ok || !approval.output?.approved) {
      throw new AbortTaskRunError("Article rejected by editor");
    }

    // Step 4: Publish
    await publishArticle(payload.articleId);
    return { published: true };
  },
});
```

### 9.6 Webhook Callbacks

Use waitpoint tokens to integrate with third-party webhooks.

```typescript
export const stripeCheckout = task({
  id: "stripe-checkout",
  run: async (payload: { userId: string; priceId: string }) => {
    const token = await wait.createToken({ timeout: "1h" });

    // Create Stripe checkout with the token URL as the success webhook
    const session = await stripe.checkout.sessions.create({
      success_url: `https://yourapp.com/success`,
      metadata: { triggerCallbackUrl: token.url },
      // ...
    });

    // Checkpoint until Stripe webhooks POST to token.url
    const result = await wait.forToken<{ sessionId: string }>(token);

    if (result.ok) {
      await provisionAccess(payload.userId, result.output.sessionId);
    }
  },
});
```

### 9.7 Long-Running AI Pipeline

Process large AI workloads that would timeout on any serverless platform.

```typescript
export const deepResearch = task({
  id: "deep-research",
  machine: "large-1x",
  maxDuration: 3600, // 1 hour of compute
  retry: { maxAttempts: 2 },
  run: async (payload: { topic: string; depth: number }) => {
    const sources = [];

    for (let i = 0; i < payload.depth; i++) {
      // Each iteration can take minutes -- no timeout
      const research = await conductResearchRound(payload.topic, i, sources);
      sources.push(...research.newSources);

      // Update metadata so frontend can track progress
      await metadata.set("progress", {
        round: i + 1,
        totalRounds: payload.depth,
        sourcesFound: sources.length,
      });
    }

    const report = await synthesizeReport(payload.topic, sources);
    return { report, sourceCount: sources.length };
  },
});
```

### 9.8 Scheduled Maintenance Job

Run recurring cleanup/maintenance tasks.

```typescript
import { schedules, logger } from "@trigger.dev/sdk";

export const weeklyCleanup = schedules.task({
  id: "weekly-cleanup",
  cron: "0 3 * * 0", // Every Sunday at 3am
  timezone: "America/Los_Angeles",
  run: async (payload) => {
    logger.info("Starting weekly cleanup", {
      scheduledAt: payload.timestamp,
      lastRun: payload.lastTimestamp,
    });

    const deletedSessions = await db.session.deleteMany({
      where: { expiresAt: { lt: new Date() } },
    });

    const deletedTempFiles = await cleanupTempStorage();

    return {
      deletedSessions: deletedSessions.count,
      deletedTempFiles,
    };
  },
});
```

### 9.9 Error Handling Patterns

```typescript
import { task, retry, AbortTaskRunError, logger } from "@trigger.dev/sdk";

export const resilientApiCall = task({
  id: "resilient-api-call",
  retry: { maxAttempts: 5 },
  run: async (payload: { url: string }) => {
    // Pattern 1: Abort without retrying (permanent errors)
    if (!payload.url.startsWith("https://")) {
      throw new AbortTaskRunError("Only HTTPS URLs are allowed");
    }

    // Pattern 2: Retry a specific block (not the whole task)
    const data = await retry.onThrow(
      async ({ attempt }) => {
        logger.info(`Fetching data, attempt ${attempt}`);
        const response = await fetch(payload.url);
        if (!response.ok) throw new Error(`HTTP ${response.status}`);
        return response.json();
      },
      { maxAttempts: 3, factor: 2, minTimeoutInMs: 1000 }
    );

    // Pattern 3: Conditional retry with retry.fetch
    const enriched = await retry.fetch("https://api.enrichment.com/v1/data", {
      method: "POST",
      body: JSON.stringify(data),
      retry: {
        byStatus: {
          "429": {
            strategy: "headers",
            limitHeader: "x-ratelimit-limit",
            remainingHeader: "x-ratelimit-remaining",
            resetHeader: "x-ratelimit-reset",
            resetFormat: "unix_timestamp_in_ms",
          },
          "500-599": {
            strategy: "backoff",
            maxAttempts: 5,
            factor: 2,
            minTimeoutInMs: 1000,
            maxTimeoutInMs: 30_000,
          },
        },
      },
    });

    return enriched.json();
  },
});
```

---

## 10. Context Object Reference

The second argument to `run()` provides the execution context:

```typescript
run: async (payload, { ctx, signal }) => {
  ctx.task.id;              // "my-task"
  ctx.task.exportName;      // "myTask"
  ctx.task.filePath;        // "./trigger/tasks.ts"

  ctx.run.id;               // "run_abc123"
  ctx.run.tags;             // ["user:123"]
  ctx.run.isTest;           // boolean
  ctx.run.createdAt;        // Date
  ctx.run.startedAt;        // Date
  ctx.run.idempotencyKey;   // string | undefined
  ctx.run.maxAttempts;      // number
  ctx.run.durationMs;       // milliseconds at time of run() call
  ctx.run.costInCents;      // compute cost at time of run() call
  ctx.run.version;          // "20250228.1"
  ctx.run.maxDuration;      // number | undefined

  ctx.attempt.id;           // attempt ID
  ctx.attempt.number;       // 1, 2, 3...
  ctx.attempt.startedAt;    // Date

  ctx.environment.type;     // "PRODUCTION" | "STAGING" | "DEVELOPMENT" | "PREVIEW"
  ctx.environment.slug;     // "prod" | "staging" | "dev"
  ctx.environment.branchName; // set for PREVIEW environments

  ctx.queue.id;             // queue ID
  ctx.queue.name;           // "my-queue"

  ctx.organization.name;    // org name
  ctx.project.ref;          // project reference

  ctx.machine?.name;        // "small-1x"
  ctx.machine?.cpu;         // 0.5
  ctx.machine?.memory;      // 0.5

  ctx.batch?.id;            // batch ID if part of a batch

  signal;                   // AbortSignal -- listen for cancellation
}
```

---

## 11. Troubleshooting Quick Reference

| Symptom | Likely Cause | Fix |
|---|---|---|
| `Importing from @trigger.dev/sdk/v3` | Old import path | Change to `@trigger.dev/sdk` |
| `client.defineJob()` in code | Deprecated v2 API | Use `task()` from `@trigger.dev/sdk` |
| Task not appearing in dashboard | Task not exported | Add `export` to task definition |
| `triggerAndWait` returns unexpected shape | Using result directly as output | Check `result.ok`, access `result.output`, or use `.unwrap()` |
| `No loader for ".node" files` | Native binary in bundle | Add package to `build.external` |
| Version mismatch on deploy | CLI and SDK versions differ | Run `trigger.dev update` or pin versions in `package.json` |
| Preview branch not archiving | Missing `closed` in GitHub Action `pull_request.types` | Add `types: [opened, synchronize, reopened, closed]` |
| `Cannot find matching keyid` | Node.js v22 + corepack bug | Use Node.js v20 or `npm i -g corepack@latest` |
| Runs queuing but not executing | Concurrency limit hit | Increase queue `concurrencyLimit` or check plan limits |
| Database connection lost after wait | Checkpoint freed the process | Use `tasks.onWait()` to close and `tasks.onResume()` to reopen connections |

---

## 12. Cross-Reference Index

| Topic | File |
|---|---|
| Tasks & Schema Tasks | [TRIGGER_DEV_TASKS.md](./TRIGGER_DEV_TASKS.md) |
| Queues & Concurrency | [TRIGGER_DEV_QUEUES_CONCURRENCY.md](./TRIGGER_DEV_QUEUES_CONCURRENCY.md) |
| Waits & Tokens | [TRIGGER_DEV_WAITS_TOKENS.md](./TRIGGER_DEV_WAITS_TOKENS.md) |
| Errors & Retries | [TRIGGER_DEV_ERRORS_RETRIES.md](./TRIGGER_DEV_ERRORS_RETRIES.md) |
| Realtime | [TRIGGER_DEV_REALTIME.md](./TRIGGER_DEV_REALTIME.md) |
| Streams | [TRIGGER_DEV_STREAMS.md](./TRIGGER_DEV_STREAMS.md) |
| Management API | [TRIGGER_DEV_MANAGEMENT_API.md](./TRIGGER_DEV_MANAGEMENT_API.md) |
| Deployment & CLI | [TRIGGER_DEV_DEPLOYMENT.md](./TRIGGER_DEV_DEPLOYMENT.md) |
| Patterns & Recipes | [TRIGGER_DEV_PATTERNS.md](./TRIGGER_DEV_PATTERNS.md) |
| Decision Tree | [TRIGGER_DEV_DECISION_TREE.md](./TRIGGER_DEV_DECISION_TREE.md) |

---

## 13. Known Source Doc Issues

These issues were identified in the underlying API doc source files:

- `01-tasks-api/00-trigger.md` — mislabeled; duplicates "advanced usage" content instead of documenting the trigger endpoint.
- `03-runs-api/06-add-tags-to-run.md` — contains duplicated metadata content instead of tags endpoint docs.
- Auth naming inconsistency: some source docs use `accessToken`, others use `secretKey` for the same credential type.

---

## Appendix: Concept Map

### Primitive Graph

- `task/schemaTask` → creates **runs**
- `run` → has **attempts**
- `run` → assigned to **queue** (+ optional `concurrencyKey` partition)
- `run` → can **wait** (`wait.for`, `wait.until`, `wait.forToken`)
- `wait.forToken` → depends on **waitpoint token**
- `run` → emits **logs/traces/metadata/tags/streams**
- `batch` → creates many `run`s
- `schedule` → creates run(s) over time
- `deploy version` → determines task code used by new runs

### Two SDK Surfaces

- **Definition/runtime surface**: write tasks and orchestration in task files.
- **Management surface**: backend/API operations for triggering, monitoring, mutating runs/queues/schedules/env vars/deployments.

### Deployment Lifecycle Map

`local dev` → `bundle (esbuild)` → `deploy image` → `environment activation` → `trigger runs` → `observe + operate` → `promote/rollback via deployment control`.

---

*This document covers Trigger.dev SDK v4.4.3. Items marked with `<!-- UNVERIFIED -->` should be validated against the latest documentation at [trigger.dev/docs](https://trigger.dev/docs).*
