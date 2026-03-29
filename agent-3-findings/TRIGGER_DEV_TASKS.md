# Trigger.dev Tasks Reference

> **SDK version**: 4.4.3 | **Import**: `@trigger.dev/sdk` (never `@trigger.dev/sdk/v3`)

Tasks are the core building block of Trigger.dev. A task is a function that runs in the Trigger.dev runtime -- it can be triggered from your backend, from other tasks, on a schedule, or via the REST API.

Every task **must be exported** from a file inside a configured `dirs` directory (set in `trigger.config.ts`).

---

## Table of Contents

1. [Defining a Task with `task()`](#1-defining-a-task-with-task)
2. [Schema Tasks with `schemaTask()`](#2-schema-tasks-with-schematask)
3. [Scheduled Tasks](#3-scheduled-tasks)
4. [Lifecycle Hooks](#4-lifecycle-hooks)
5. [The `run` Function Context](#5-the-run-function-context)
6. [Triggering Methods](#6-triggering-methods)
7. [Subtask Patterns](#7-subtask-patterns)
8. [Hidden Tasks](#8-hidden-tasks)
9. [Machine Types](#9-machine-types)
10. [Quick Reference: Limits and Formats](#10-quick-reference-limits-and-formats)

---

## 1. Defining a Task with `task()`

```ts
import { task } from "@trigger.dev/sdk";

export const processOrder = task({
  id: "process-order",
  retry: {
    maxAttempts: 5,
    factor: 1.8,
    minTimeoutInMs: 500,
    maxTimeoutInMs: 30_000,
    randomize: true,
  },
  queue: {
    name: "order-processing",
    concurrencyLimit: 10,
  },
  machine: "small-2x",
  maxDuration: 300, // seconds
  run: async (payload: { orderId: string; userId: string }, { ctx }) => {
    // Task logic here
    return { success: true, orderId: payload.orderId };
  },
});
```

### `task()` Options Object

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `id` | `string` | **Yes** | -- | Unique identifier for the task. Must be stable across deploys. |
| `run` | `(payload, { ctx }) => Promise<T>` | **Yes** | -- | The function that executes when the task runs. Receives the payload and a context object. |
| `retry` | `RetryOptions` | No | Config default | Retry behavior when the task throws an uncaught error. See [Retry Options](#retry-options). |
| `queue` | `QueueOptions` | No | Auto-generated | Queue configuration. See [Queues & Concurrency](./TRIGGER_DEV_QUEUES_CONCURRENCY.md). |
| `machine` | `MachinePreset` | No | `"small-1x"` | Compute resources for the task. See [Machine Types](#9-machine-types). |
| `maxDuration` | `number` | No | Config default | Maximum wall-clock seconds the task can run before being stopped. |
| `init` | `(payload, { ctx }) => Promise<any>` | No | -- | Runs before `run`. Return value is available in lifecycle hooks. |
| `onStart` | `(params) => Promise<void>` | No | -- | Called when a task run starts. |
| `onSuccess` | `(params) => Promise<void>` | No | -- | Called when a task run completes successfully. |
| `onFailure` | `(params) => Promise<void>` | No | -- | Called when all retry attempts are exhausted and the task fails permanently. |
| `catchError` | `(params) => Promise<CatchErrorResult \| void>` | No | -- | Advanced error handler. Can modify retry behavior per-error. |
| `middleware` | `(params) => Promise<void>` | No | -- | Task-level middleware. Wraps the `run` function. <!-- UNVERIFIED --> Action item: verify at `https://trigger.dev/docs/middleware`. |

### Retry Options

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `maxAttempts` | `number` | No | `3` | Maximum number of retry attempts (including the first attempt). |
| `factor` | `number` | No | `2` | Exponential backoff multiplier. |
| `minTimeoutInMs` | `number` | No | `1000` | Minimum delay between retries in milliseconds. |
| `maxTimeoutInMs` | `number` | No | `10000` | Maximum delay between retries in milliseconds. |
| `randomize` | `boolean` | No | `true` | Add jitter to retry delays to avoid thundering herd. |

> Task-level retry settings override the defaults in your `trigger.config.ts` file.

> By default, retrying is **disabled** in the DEV environment when you create a project via `trigger init`.

### Queue Options

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `name` | `string` | No | Auto-generated from task id | Name of the queue. Multiple tasks can share a queue. |
| `concurrencyLimit` | `number` (0-1000) | No | Unlimited | Maximum concurrent run executions in this queue. |

See [Queues & Concurrency](./TRIGGER_DEV_QUEUES_CONCURRENCY.md) for shared queue patterns.

---

## 2. Schema Tasks with `schemaTask()`

`schemaTask()` adds Zod schema validation for input (and optionally output) payloads. The payload is validated **before** the `run` function executes, and TypeScript types are inferred from the schema.

```ts
import { schemaTask } from "@trigger.dev/sdk";
import { z } from "zod";

export const processVideo = schemaTask({
  id: "process-video",
  schema: z.object({
    videoUrl: z.string().url(),
    format: z.enum(["mp4", "webm", "mov"]).default("mp4"),
    maxResolution: z.number().max(4096).optional(),
  }),
  run: async (payload) => {
    // payload is fully typed: { videoUrl: string; format: "mp4" | "webm" | "mov"; maxResolution?: number }
    const result = await transcode(payload.videoUrl, payload.format);
    return { outputUrl: result.url, duration: result.durationMs };
  },
});
```

### `schemaTask()` Options

`schemaTask()` accepts all the same options as `task()` plus:

| Property | Type | Required | Description |
|---|---|---|---|
| `schema` | `ZodSchema` | **Yes** | Zod schema for the input payload. Validated before `run()` executes. |
| `outputSchema` | `ZodSchema` | No | Zod schema for the return value. Validated after `run()` completes. <!-- UNVERIFIED --> Action item: verify at `https://trigger.dev/docs/tasks`. |

> **WARNING**: If the payload does not match the schema, the run will fail immediately without retrying (the error is treated as an `AbortTaskRunError`). <!-- UNVERIFIED --> Action item: verify at `https://trigger.dev/docs/tasks`.

---

## 3. Scheduled Tasks

Scheduled tasks run on a cron schedule. They use the `schedules.task()` function.

### 3.1 Declarative Schedules (In-Code)

Define the cron expression directly on the task. The schedule is created/updated automatically when you deploy.

```ts
import { schedules } from "@trigger.dev/sdk";

export const dailyReport = schedules.task({
  id: "daily-report",
  cron: "0 9 * * 1-5", // 9 AM UTC, Monday-Friday
  run: async (payload, { ctx }) => {
    // payload includes: timestamp, lastTimestamp, externalId, upcoming
    console.log(`Running scheduled report at ${payload.timestamp}`);
    await generateAndSendReport();
  },
});
```

#### Cron with Timezone

```ts
export const dailyDigest = schedules.task({
  id: "daily-digest",
  cron: {
    pattern: "0 9 * * 1-5",
    timezone: "America/New_York", // IANA timezone format
  },
  run: async (payload) => {
    // Runs at 9 AM Eastern Time, respects daylight saving time
    await sendDigestEmail();
  },
});
```

### 3.2 Dynamic (Imperative) Schedules via Management API

Create schedules at runtime using the SDK. This is useful for user-specific or tenant-specific schedules.

```ts
import { schedules } from "@trigger.dev/sdk";

// Create a dynamic schedule
const schedule = await schedules.create({
  task: "daily-report",          // Must match a schedules.task() id
  cron: "0 9 * * *",
  deduplicationKey: "report-tenant-123", // Prevents duplicate schedules
  timezone: "America/New_York",
  externalId: "tenant-123",      // Optional external reference
});

// Update an existing schedule
await schedules.update(schedule.id, {
  cron: "0 10 * * *", // Change to 10 AM
});

// Delete a schedule
await schedules.del(schedule.id);

// List all schedules
const allSchedules = await schedules.list();

// Deactivate / reactivate
await schedules.deactivate(schedule.id);
await schedules.activate(schedule.id);
```

#### Deduplication Keys

The `deduplicationKey` ensures only one schedule exists per key. If you call `schedules.create()` with a `deduplicationKey` that already exists, the existing schedule is returned (not duplicated). This is essential for idempotent schedule creation in multi-deploy environments.

### Scheduled Task Payload

The `run` function of a `schedules.task()` receives a special payload:

| Property | Type | Description |
|---|---|---|
| `timestamp` | `Date` | The time this scheduled run was triggered. |
| `lastTimestamp` | `Date \| undefined` | The time the previous run was triggered (if any). |
| `externalId` | `string \| undefined` | The external ID set when creating an imperative schedule. |
| `upcoming` | `Date[]` | Array of the next few scheduled run times. <!-- UNVERIFIED --> Action item: verify at `https://trigger.dev/docs/triggering#scheduled-tasks`. |

---

## 4. Lifecycle Hooks

Lifecycle hooks let you execute code at specific points in a task's execution. They can be defined at three levels:

1. **Task-level** -- on the task options object.
2. **Global (config)** -- in `trigger.config.ts` via `defineConfig()`.
3. **Global (runtime)** -- in an `init.ts` file via `tasks.onStart()`, `tasks.onFailure()`, etc.

### 4.1 Task-Level Lifecycle Hooks

```ts
export const myTask = task({
  id: "my-task",
  init: async (payload, { ctx }) => {
    // Runs BEFORE the run function. Return value is passed to lifecycle hooks.
    const dbConnection = await connectToDatabase();
    return { db: dbConnection };
  },
  onStart: async ({ payload, ctx }) => {
    console.log(`Task ${ctx.task.id} started, run: ${ctx.run.id}`);
  },
  onSuccess: async ({ payload, output, ctx }) => {
    console.log(`Task ${ctx.task.id} succeeded with output:`, output);
  },
  onFailure: async ({ payload, error, ctx }) => {
    console.error(`Task ${ctx.task.id} failed permanently:`, error);
    await notifyOpsTeam(ctx.task.id, error);
  },
  catchError: async ({ payload, error, ctx, retryAt }) => {
    // Called on EACH failed attempt (before retry logic).
    // Return values can modify retry behavior:
    if (error instanceof RateLimitError) {
      return { retryAt: new Date(Date.now() + error.retryAfterMs) };
    }
    if (error instanceof InvalidInputError) {
      return { skipRetrying: true };
    }
    // Return undefined to use default retry behavior
    return undefined;
  },
  run: async (payload: { data: string }) => {
    return { processed: true };
  },
});
```

### 4.2 Global Lifecycle Hooks (trigger.config.ts)

These apply to **all tasks** in the project:

```ts
// trigger.config.ts
import { defineConfig } from "@trigger.dev/sdk";

export default defineConfig({
  project: "<project ref>",
  onStart: async ({ payload, ctx }) => {
    console.log("Task started:", ctx.task.id);
  },
  onSuccess: async ({ payload, output, ctx }) => {
    console.log("Task succeeded:", ctx.task.id);
  },
  onFailure: async ({ payload, error, ctx }) => {
    console.error("Task failed:", ctx.task.id, error);
  },
  init: async ({ payload, ctx }) => {
    console.log("Initializing before task run");
  },
});
```

### 4.3 Global Runtime Hooks (init.ts)

Create a file in your trigger directory (e.g., `trigger/init.ts`). It is automatically loaded at runtime. Use `tasks.onStart()`, `tasks.onFailure()`, etc. to register hooks:

```ts
// trigger/init.ts
import { tasks } from "@trigger.dev/sdk";
import * as Sentry from "@sentry/node";

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  defaultIntegrations: false,
});

tasks.onFailure(({ payload, error, ctx }) => {
  Sentry.captureException(error, {
    extra: { payload, taskId: ctx.task.id, runId: ctx.run.id },
  });
});
```

### 4.4 Middleware

Middleware wraps the `run` function and executes before and after it. Use `tasks.middleware()` for global middleware:

```ts
// trigger/init.ts
import { tasks, locals, logger } from "@trigger.dev/sdk";

const TimingLocal = locals.create<{ startedAt: number }>("timing");

tasks.middleware("request-timing", async ({ next }) => {
  locals.set(TimingLocal, { startedAt: Date.now() });
  try {
    await next(); // Execute the run function
  } finally {
    const elapsed = Date.now() - locals.getOrThrow(TimingLocal).startedAt;
    logger.info(`Task completed in ${elapsed}ms`);
  }
});
```

#### Middleware with Resource Cleanup (Playwright Example)

```ts
import { tasks, locals, logger } from "@trigger.dev/sdk";
import { chromium, type Browser } from "playwright";

const BrowserLocal = locals.create<{ browser: Browser }>("playwright-browser");

export function getBrowser() {
  return locals.getOrThrow(BrowserLocal).browser;
}

tasks.middleware("playwright-browser", async ({ next }) => {
  const browser = await chromium.launch();
  locals.set(BrowserLocal, { browser });
  logger.log("Browser launched");
  try {
    await next();
  } finally {
    await browser.close();
    logger.log("Browser closed");
  }
});

// Handle waits (browser must be closed while waiting to free resources)
tasks.onWait("playwright-browser", async () => {
  const browser = getBrowser();
  await browser.close();
});

// Re-launch browser when resuming after a wait
tasks.onResume("playwright-browser", async () => {
  const browser = await chromium.launch();
  locals.set(BrowserLocal, { browser });
});
```

### Lifecycle Hook Execution Order

1. `init` -- runs first, return value available to other hooks
2. `onStart` -- fires when the run begins
3. `middleware` -- wraps the `run` function
4. `run` -- your task logic
5. `onSuccess` **or** `catchError` / `onFailure`
   - `catchError` fires on **each** failed attempt
   - `onFailure` fires only after **all retries are exhausted**
   - `onSuccess` fires when the task completes successfully

> Task-level hooks and global hooks both fire. Global hooks fire first, then task-level hooks. <!-- UNVERIFIED --> Action item: verify hook ordering at `https://trigger.dev/docs/lifecycle-hooks`.

---

## 5. The `run` Function Context

The second argument to the `run` function is a destructurable object containing `ctx`:

```ts
export const myTask = task({
  id: "my-task",
  run: async (payload: { userId: string }, { ctx }) => {
    // ctx is a snapshot -- values are fixed at the moment run() is called
    console.log(ctx.task.id);              // "my-task"
    console.log(ctx.run.id);               // "run_abc123"
    console.log(ctx.run.tags);             // ["user_123"]
    console.log(ctx.environment.type);     // "PRODUCTION" | "STAGING" | "DEVELOPMENT" | "PREVIEW"
    console.log(ctx.attempt.number);       // 1, 2, 3, ...
    console.log(ctx.machine?.name);        // "small-1x"
  },
});
```

### Context Properties

| Path | Type | Description |
|---|---|---|
| `ctx.task.id` | `string` | The task ID. |
| `ctx.task.exportName` | `string` | The exported variable name (e.g., `myTask`). |
| `ctx.task.filePath` | `string` | File path of the task. |
| `ctx.run.id` | `string` | Unique run ID (prefixed `run_`). |
| `ctx.run.tags` | `string[]` | Tags attached to the run. |
| `ctx.run.isTest` | `boolean` | Whether this is a test run from the dashboard. |
| `ctx.run.createdAt` | `Date` | When the run was created. |
| `ctx.run.startedAt` | `Date` | When the run started executing. |
| `ctx.run.idempotencyKey` | `string \| undefined` | Idempotency key if one was provided. |
| `ctx.run.maxAttempts` | `number` | Maximum retry attempts configured. |
| `ctx.run.durationMs` | `number` | Duration at the moment `run()` was called (snapshot). |
| `ctx.run.costInCents` | `number` | Cost at the moment `run()` was called (snapshot). |
| `ctx.run.version` | `string` | Deployment version (e.g., `"20250228.1"`). |
| `ctx.run.maxDuration` | `number` | Maximum allowed duration in seconds. |
| `ctx.run.context` | `any` | Custom context passed when triggering. |
| `ctx.attempt.id` | `string` | Attempt ID. |
| `ctx.attempt.number` | `number` | Attempt number (starts at 1). |
| `ctx.attempt.startedAt` | `Date` | When this attempt started. |
| `ctx.attempt.status` | `string` | Current attempt status. |
| `ctx.queue.id` | `string` | Queue ID. |
| `ctx.queue.name` | `string` | Queue name. |
| `ctx.environment.id` | `string` | Environment ID. |
| `ctx.environment.slug` | `string` | Environment slug (`"dev"`, `"prod"`, `"staging"`). |
| `ctx.environment.type` | `string` | `"PRODUCTION"`, `"STAGING"`, `"DEVELOPMENT"`, or `"PREVIEW"`. |
| `ctx.environment.branchName` | `string \| undefined` | Branch name if environment is `PREVIEW`. |
| `ctx.organization.id` | `string` | Organization ID. |
| `ctx.organization.name` | `string` | Organization name. |
| `ctx.project.id` | `string` | Project ID. |
| `ctx.project.ref` | `string` | Project reference. |
| `ctx.project.slug` | `string` | Project slug. |
| `ctx.project.name` | `string` | Project name. |
| `ctx.batch?.id` | `string \| undefined` | Batch ID if the run is part of a batch. |
| `ctx.machine?.name` | `string` | Machine preset name. |
| `ctx.machine?.cpu` | `number` | CPU allocation. |
| `ctx.machine?.memory` | `number` | Memory allocation. |
| `ctx.machine?.centsPerMs` | `number` | Cost in cents per millisecond. |

> **WARNING**: The `ctx` object is a **snapshot** taken when `run()` is called. Values like `ctx.run.durationMs` do not update during execution. For live usage data, use the [run usage SDK functions](https://trigger.dev/docs/run-usage).

### Available Imports Inside `run`

These SDK modules are available inside your `run` function:

```ts
import {
  logger,     // Structured logging (logger.info, logger.warn, logger.error, logger.debug)
  metadata,   // Read/write run metadata (metadata.set, metadata.get, metadata.parent.set)
  tags,       // Manage run tags at runtime
  wait,       // Pause execution (wait.for, wait.until, wait.forToken)
  retry,      // Fine-grained retries (retry.onThrow, retry.fetch)
  AbortTaskRunError, // Throw to abort without retrying
  batch,      // Batch trigger operations
} from "@trigger.dev/sdk";
```

---

## 6. Triggering Methods

### 6.1 From Your Backend (Outside Tasks)

Use the task handle directly, or use the `tasks` namespace for type-safe triggering by string ID:

```ts
// Direct import (preferred when you have access to the task)
import { myTask } from "./trigger/my-task";

const handle = await myTask.trigger({ userId: "user_123" });
console.log(handle.id); // run_abc123

// Type-safe by string ID (when you can't import the task directly)
import type { myTask } from "./trigger/my-task";
import { tasks } from "@trigger.dev/sdk";

const handle = await tasks.trigger<typeof myTask>("my-task", { userId: "user_123" });
```

### 6.2 From Inside Other Tasks

```ts
import { task } from "@trigger.dev/sdk";
import { childTask } from "./child-task";

export const parentTask = task({
  id: "parent-task",
  run: async (payload: { items: string[] }) => {
    // Fire and forget -- does NOT wait for the child to finish
    const handle = await childTask.trigger({ item: payload.items[0] });

    // Trigger and wait -- blocks until the child completes
    const result = await childTask.triggerAndWait({ item: payload.items[1] });
    if (result.ok) {
      console.log("Child succeeded:", result.output);
    } else {
      console.error("Child failed:", result.error);
    }

    // Shorthand: unwrap throws on failure, returns output directly
    const output = await childTask.triggerAndWait({ item: payload.items[2] }).unwrap();

    return { processed: true };
  },
});
```

### 6.3 Batch Triggering

```ts
// Batch trigger -- fire and forget up to 1,000 items
const batchHandle = await myTask.batchTrigger(
  items.map((item) => ({
    payload: { itemId: item.id },
    options: {
      tags: [`item_${item.id}`],
      concurrencyKey: `tenant_${item.tenantId}`,
    },
  }))
);
console.log(batchHandle.batchId); // batch_xyz789
console.log(batchHandle.runs);    // ["run_1", "run_2", ...]

// Batch trigger and wait -- blocks until ALL items complete
const batchResult = await myTask.batchTriggerAndWait(
  items.map((item) => ({
    payload: { itemId: item.id },
  }))
);
const successfulOutputs = batchResult.runs
  .filter((run) => run.ok)
  .map((run) => run.output);
```

### 6.4 `batch.triggerByTaskAndWait` -- Heterogeneous Batch

Trigger **different task types** in a single batch call and wait for all results:

```ts
import { batch } from "@trigger.dev/sdk";
import { taskA, taskB, taskC } from "./tasks";

const { runs } = await batch.triggerByTaskAndWait([
  { task: taskA, payload: { url: "https://example.com" } },
  { task: taskB, payload: { query: "test" } },
  { task: taskC, payload: { id: 42 } },
]);

// runs[0] is typed as the result of taskA, etc.
```

### Trigger Options

All trigger methods accept an options object as the second argument (or inside `options` for batch items):

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `idempotencyKey` | `string` | No | -- | Prevents duplicate runs. If a run with this key already exists, the existing run ID is returned. |
| `concurrencyKey` | `string` | No | -- | Scopes queue concurrency to this key. Useful for per-user or per-tenant limits. |
| `queue` | `QueueOptions` | No | Task default | Override the queue for this specific run. |
| `delay` | `string \| Date` | No | -- | Delay before execution. See [Delay Format](#delay-format). |
| `ttl` | `string \| number` | No | -- | Time-to-live. If the run hasn't started within this window, it is discarded. See [TTL Format](#ttl-format). |
| `tags` | `string \| string[]` | No | `[]` | Tags to attach to the run. Max 10 per run, each < 128 chars. |
| `metadata` | `Record<string, any>` | No | -- | Arbitrary metadata attached to the run. <!-- UNVERIFIED --> Action item: verify at `https://trigger.dev/docs/runs/metadata`. |
| `machine` | `MachinePreset` | No | Task default | Override the machine preset for this run. |
| `version` | `string` | No | Current | Lock the run to a specific deployment version. |
| `maxAttempts` | `number` | No | Task default | Override max retry attempts for this run. <!-- UNVERIFIED --> Action item: verify at `https://trigger.dev/docs/tasks`. |
| `priority` | `number` | No | -- | Priority within the queue (higher = sooner). <!-- UNVERIFIED --> Action item: verify at `https://trigger.dev/docs/queue-concurrency`. |

```ts
await myTask.trigger(
  { orderId: "order_123" },
  {
    idempotencyKey: "order_123_process",
    concurrencyKey: "tenant_456",
    delay: "30m",
    ttl: "2h",
    tags: ["order_123", "tenant_456"],
    machine: "medium-1x",
  }
);
```

> **WARNING**: `triggerAndWait()` and `batchTriggerAndWait()` can **only** be called from inside a task's `run` function. They cannot be used from your backend code.

> **WARNING**: Never wrap `triggerAndWait()` or `batchTriggerAndWait()` in `Promise.all()`. This is **not supported** and will cause failures. If you need parallel child tasks, use `batch.triggerByTaskAndWait()` instead.

### Version Locking Behavior

| Method | Child Version | Locked? |
|---|---|---|
| `trigger()` | Current deployed version | No |
| `batchTrigger()` | Current deployed version | No |
| `triggerAndWait()` | Same version as parent | Yes |
| `batchTriggerAndWait()` | Same version as parent | Yes |

---

## 7. Subtask Patterns

### Parent/Child with `triggerAndWait`

```ts
export const parentTask = task({
  id: "parent-task",
  run: async (payload: { urls: string[] }) => {
    const results: string[] = [];

    for (const url of payload.urls) {
      // Each child runs independently with its own retry config
      const result = await fetchAndParse
        .triggerAndWait({ url })
        .unwrap(); // Throws if the child fails
      results.push(result.content);
    }

    return { totalProcessed: results.length };
  },
});

export const fetchAndParse = task({
  id: "fetch-and-parse",
  retry: { maxAttempts: 3 },
  run: async (payload: { url: string }) => {
    const response = await fetch(payload.url);
    const html = await response.text();
    return { content: html.substring(0, 1000) };
  },
});
```

### Parallel Children with `batch.triggerByTaskAndWait`

```ts
import { batch, task } from "@trigger.dev/sdk";

export const orchestrator = task({
  id: "orchestrator",
  run: async (payload: { companyName: string }) => {
    const { runs } = await batch.triggerByTaskAndWait([
      { task: getBasicInfo, payload: { name: payload.companyName } },
      { task: getFinancials, payload: { name: payload.companyName } },
      { task: getSocialMedia, payload: { name: payload.companyName } },
    ]);

    const [basicInfo, financials, social] = runs;

    return {
      basic: basicInfo.ok ? basicInfo.output : null,
      financials: financials.ok ? financials.output : null,
      social: social.ok ? social.output : null,
    };
  },
});
```

### Communicating Between Parent and Child via Metadata

```ts
import { metadata, task } from "@trigger.dev/sdk";

export const childTask = task({
  id: "enrichment-subtask",
  run: async (payload: { query: string }) => {
    const result = await enrichData(payload.query);
    // Update the PARENT task's metadata (visible in realtime)
    metadata.parent.set("enrichmentStatus", "complete");
    metadata.parent.set("enrichmentResult", result.summary);
    return result;
  },
});
```

> **WARNING**: There is a depth limit on nested `triggerAndWait` calls. Tasks calling `triggerAndWait` on children that also call `triggerAndWait` will work, but deeply nested chains (typically beyond ~5 levels) may hit platform limits. <!-- UNVERIFIED --> Action item: verify nesting depth limits at `https://trigger.dev/docs/tasks`.

### Metadata API Reference

| Method | Description |
|---|---|
| `metadata.set(key, value)` | Set a metadata key-value pair on the current run |
| `metadata.get(key)` | Get a metadata value by key |
| `metadata.current()` | Get the entire metadata object |
| `metadata.decrement(key, amount)` | Decrement a numeric metadata value |
| `metadata.parent.set(key, value)` | Update the parent task's metadata from a child task |
| `metadata.flush()` | Force flush pending metadata updates (normally flushed periodically in background) |

> ⚠️ WARNING
> `metadata.stream()` is **deprecated** as of SDK 4.1.0 — replaced by Realtime Streams v2 (`streams.pipe()`). See [Streams](./TRIGGER_DEV_STREAMS.md).

Metadata updates are synchronous and non-blocking; they are flushed periodically in the background. Use `metadata.flush()` only when you need to guarantee the update is visible before a checkpoint or return.

---

## 8. Hidden Tasks

Hidden tasks are not displayed in the Trigger.dev dashboard task list but can still be triggered and executed normally. Mark a task as hidden when it is an internal implementation detail (e.g., a subtask that should not be triggered manually).

<!-- UNVERIFIED -->
Action item: verify hidden tasks API syntax at `https://trigger.dev/docs/tasks`.

```ts
export const internalProcessingStep = task({
  id: "internal-processing-step",
  // hidden: true, // UNVERIFIED -- check latest SDK docs for exact syntax
  run: async (payload: { data: string }) => {
    // This task won't appear in the dashboard task list
    return { processed: true };
  },
});
```

---

## 9. Machine Types

Machine types control the CPU and memory available to your task. Set at the task level, config level (`defaultMachine`), or per-trigger.

| Machine | vCPU | Memory | Disk | Best For |
|---|---|---|---|---|
| `micro` | 0.25 | 0.25 GB | 10 GB | Lightweight HTTP calls, queue workers |
| `small-1x` | 0.5 | 0.5 GB | 10 GB | Default. Most tasks. |
| `small-2x` | 1 | 1 GB | 10 GB | Moderate processing, API orchestration |
| `medium-1x` | 1 | 2 GB | 10 GB | Data processing, image manipulation |
| `medium-2x` | 2 | 4 GB | 10 GB | AI inference, heavy computation |
| `large-1x` | 4 | 8 GB | 10 GB | Large model inference, video processing |
| `large-2x` | 8 | 16 GB | 10 GB | ML training, massive data sets |

```ts
// Set default machine for all tasks in trigger.config.ts
export default defineConfig({
  project: "<project ref>",
  defaultMachine: "small-2x",
});

// Override per task
export const heavyTask = task({
  id: "heavy-task",
  machine: "large-1x",
  run: async (payload) => { /* ... */ },
});

// Override per trigger call
await heavyTask.trigger(payload, { machine: "large-2x" });
```

---

## 10. Quick Reference: Limits and Formats

### Batch Limits

| SDK Version | Max Items per Batch |
|---|---|
| >= 4.3.1 | **1,000** |
| < 4.3.1 | **500** |

### TTL Format

Time-to-live defines how long a run can sit in the queue before being discarded.

| Format | Example | Description |
|---|---|---|
| Duration string | `"1h"`, `"30m"`, `"1h42m"` | Human-readable duration. Supports `h` (hours), `m` (minutes), `s` (seconds). |
| Number | `3600` | Seconds (minimum 1). |

```ts
await myTask.trigger(payload, { ttl: "1h42m" });
await myTask.trigger(payload, { ttl: 3600 }); // 1 hour in seconds
```

### Delay Format

Delay defines when the task should begin execution (relative or absolute).

| Format | Example | Description |
|---|---|---|
| `"Nh"` | `"1h"` | 1 hour from now |
| `"Nd"` | `"30d"` | 30 days from now |
| `"Nm"` | `"15m"` | 15 minutes from now |
| `"Nw"` | `"2w"` | 2 weeks from now |
| `"Ns"` | `"60s"` | 60 seconds from now |
| ISO 8601 string | `"2025-07-01T00:00:00Z"` | Specific date/time |

```ts
await myTask.trigger(payload, { delay: "30m" });
await myTask.trigger(payload, { delay: "2025-12-25T00:00:00Z" });
```

### Tags

- Maximum **10 tags** per run.
- Each tag must be less than **128 characters**.
- Convention: prefix with a namespace using underscore or colon (`user_123456`, `org:987654`).

```ts
await myTask.trigger(payload, {
  tags: ["user_123456", "org:987654", "priority:high"],
});
```

### Error Handling Quick Reference

| Technique | Use Case |
|---|---|
| `retry` option on task | Automatic retries with exponential backoff on uncaught errors |
| `catchError` hook | Per-error custom retry logic (skip, delay, modify) |
| `AbortTaskRunError` | Throw to fail immediately without retrying |
| `retry.onThrow()` | Retry a specific block within a task (not the whole task) |
| `retry.fetch()` | HTTP requests with status-based retry strategies |
| `try/catch` | Standard error handling with fallback logic |

```ts
import { task, retry, AbortTaskRunError } from "@trigger.dev/sdk";

export const resilientTask = task({
  id: "resilient-task",
  retry: { maxAttempts: 5 },
  run: async (payload: { url: string; isValid: boolean }) => {
    // Abort immediately -- no retries
    if (!payload.isValid) {
      throw new AbortTaskRunError("Invalid payload, will not retry");
    }

    // Retry a specific block (not the whole task)
    const data = await retry.onThrow(
      async ({ attempt }) => {
        const res = await fetch(payload.url);
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        return res.json();
      },
      { maxAttempts: 3 }
    );

    return data;
  },
});
```

---

## Cross-Reference

- [Master Reference](./TRIGGER_DEV_MASTER.md)
- [Queues & Concurrency](./TRIGGER_DEV_QUEUES_CONCURRENCY.md)
- [Waits & Tokens](./TRIGGER_DEV_WAITS_TOKENS.md)
- [Errors & Retries](./TRIGGER_DEV_ERRORS_RETRIES.md)
- [Realtime](./TRIGGER_DEV_REALTIME.md)
- [Streams](./TRIGGER_DEV_STREAMS.md)
- [Deployment & Versioning](./TRIGGER_DEV_DEPLOYMENT.md)
- [Management API](./TRIGGER_DEV_MANAGEMENT_API.md)
- [Patterns & Recipes](./TRIGGER_DEV_PATTERNS.md)
