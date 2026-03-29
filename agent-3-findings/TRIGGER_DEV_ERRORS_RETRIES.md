# Trigger.dev Errors & Retries

> **SDK version:** 4.4.3 | Canonical reference for error handling and retry behavior.
> Source: [Trigger.dev Errors & Retrying](https://trigger.dev/docs/errors-retrying)

---

## Overview

When an uncaught error is thrown inside a task, the current **attempt** fails. Trigger.dev can automatically retry the task according to configurable rules. You can set retry behavior in two places:

1. **`trigger.config.ts`** -- project-wide defaults for all tasks.
2. **Individual task definitions** -- per-task overrides (take precedence over config-level defaults).

> **NOTE:** By default, the CLI `init` command disables retrying in the DEV environment. Enable it in your `trigger.config.ts` if needed.

---

## Default Retry Behavior

When you create a project with the CLI, the default retry configuration (set in `trigger.config.ts`) is:

```ts
// trigger.config.ts
import { defineConfig } from "@trigger.dev/sdk";

export default defineConfig({
  project: "<project ref>",
  dirs: ["./trigger"],
  retries: {
    enabledInDev: false,
    default: {
      maxAttempts: 3,
      minTimeoutInMs: 1000,
      maxTimeoutInMs: 10000,
      factor: 2,
      randomize: true,
    },
  },
});
```

Task-level `retry` settings override the `retries.default` in `trigger.config.ts`.

---

## Retry Configuration Options

### Task-Level Retry

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `maxAttempts` | `number` | No | `3` | Maximum number of attempts (including the initial run). |
| `factor` | `number` | No | `2` | Exponential backoff multiplier. Each successive delay is multiplied by this factor. |
| `minTimeoutInMs` | `number` | No | `1000` | Minimum delay (in ms) before the first retry. |
| `maxTimeoutInMs` | `number` | No | `10000` | Maximum delay (in ms) between retries. The backoff will never exceed this. |
| `randomize` | `boolean` | No | `true` | Whether to add jitter to the delay to avoid thundering-herd problems. |

```ts
import { task } from "@trigger.dev/sdk";

export const myTask = task({
  id: "my-task",
  retry: {
    maxAttempts: 10,
    factor: 1.8,
    minTimeoutInMs: 500,
    maxTimeoutInMs: 30_000,
    randomize: false,
  },
  run: async (payload: { prompt: string }) => {
    // Task logic that may throw
  },
});
```

### Config-Level Defaults (`trigger.config.ts`)

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `retries.enabledInDev` | `boolean` | No | `false` | Whether retries are enabled in the DEV environment. |
| `retries.default.maxAttempts` | `number` | No | `3` | Default max attempts for all tasks. |
| `retries.default.factor` | `number` | No | `2` | Default backoff factor. |
| `retries.default.minTimeoutInMs` | `number` | No | `1000` | Default minimum timeout between retries. |
| `retries.default.maxTimeoutInMs` | `number` | No | `10000` | Default maximum timeout between retries. |
| `retries.default.randomize` | `boolean` | No | `true` | Default jitter setting. |

---

## Management API Retry Config

When using the SDK management API (e.g., `runs.retrieve()`, `tasks.trigger()` from your backend), the SDK automatically retries requests that fail due to network or server errors. The default retry settings for API requests are:

| Property | Type | Default | Description |
|---|---|---|---|
| `maxAttempts` | `number` | `3` | Maximum number of retry attempts for the API request. |
| `minTimeoutInMs` | `number` | `1000` | Minimum delay between retries. |
| `maxTimeoutInMs` | `number` | `5000` | Maximum delay between retries. |
| `factor` | `number` | `1.8` | Exponential backoff factor. |
| `randomize` | `boolean` | `true` | Whether to add jitter to the delay. |

You can customize the management API retry behavior globally via `configure()`:

```ts
import { configure } from "@trigger.dev/sdk";

configure({
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

Or per-request as the last argument to any SDK function:

```ts
import { runs } from "@trigger.dev/sdk";

const run = await runs.retrieve("run_1234", {
  retry: {
    maxAttempts: 1, // Disable retries for this request
  },
});
```

> **WARNING:** When running **inside a task**, the SDK ignores customized retry options for certain functions (e.g., `task.trigger`, `task.batchTrigger`), and uses retry settings optimized for task execution. This means your `requestOptions.retry` overrides will not apply in that context.

---

## `catchError` Hook -- Conditional Retry Logic

The `catchError` callback runs whenever an uncaught error is thrown in your task. It lets you inspect the error, log it, modify the retry behavior, or skip retrying entirely.

> **NOTE:** The older name `handleError` was renamed to `catchError` in the current SDK. They serve the same purpose.

### Return Values

| Return | Effect |
|---|---|
| `undefined` (or no return) | Use the normal retry settings defined on the task or config. |
| `{ skipRetrying: true }` | Immediately fail the run; do not retry further. |
| `{ retryAt: Date }` | Retry at a specific date/time instead of using exponential backoff. |
| `{ retry: { ... } }` | Override the retry options for the next retry attempt. <!-- UNVERIFIED --> Action item: verify `{ retry }` return shape in `https://trigger.dev/docs/errors-retrying#catcherror` for SDK 4.4.3+. |
| `{ error: Error }` | Replace the error with a different one. <!-- UNVERIFIED --> Action item: verify `{ error }` return shape in `https://trigger.dev/docs/errors-retrying#catcherror` for SDK 4.4.3+. |

### Task-Level `catchError`

```ts
import { task } from "@trigger.dev/sdk";
import OpenAI from "openai";

export const openaiTask = task({
  id: "openai-task",
  retry: {
    maxAttempts: 10,
  },
  run: async (payload: { prompt: string }) => {
    const chatCompletion = await openai.chat.completions.create({
      messages: [{ role: "user", content: payload.prompt }],
      model: "gpt-4o",
    });
    return chatCompletion.choices[0].message.content;
  },
  catchError: async ({ payload, error, ctx, retryAt }) => {
    if (error instanceof OpenAI.APIError) {
      // No HTTP status means a network-level error -- skip retrying
      if (!error.status) {
        return { skipRetrying: true };
      }

      // Out of credits -- retrying won't help
      if (error.status === 429 && error.type === "insufficient_quota") {
        return { skipRetrying: true };
      }

      // Rate-limited -- check headers to determine when to retry
      if (error.headers) {
        const remaining = error.headers["x-ratelimit-remaining-requests"];
        const resets = error.headers["x-ratelimit-reset-requests"];
        if (typeof remaining === "string" && Number(remaining) === 0) {
          return {
            retryAt: calculateResetDate(resets),
          };
        }
      }
    }

    // Return undefined to use default retry behavior
  },
});
```

### Config-Level `catchError`

You can also define `catchError` in `trigger.config.ts` to apply to all tasks. The task-level `catchError` takes precedence if both are defined.

```ts
// trigger.config.ts
import { defineConfig } from "@trigger.dev/sdk";

export default defineConfig({
  project: "<project ref>",
  catchError: async ({ payload, error, ctx }) => {
    // Global error handling logic
    console.error("Task error:", error);
    // Return undefined to use default retry behavior
  },
});
```

<!-- UNVERIFIED -->
Action item: verify whether config-level `catchError` supports the same return values as task-level `catchError` in `https://trigger.dev/docs/errors-retrying#catcherror`.

---

## `retry.onThrow()` -- Fine-Grained In-Task Retry Blocks

Retry a specific block of code within a task, independent of the task's overall retry configuration. This is useful when you want to retry a single operation (like an API call) without restarting the entire task.

```ts
import { task, logger, retry } from "@trigger.dev/sdk";

export const retryOnThrow = task({
  id: "retry-on-throw",
  run: async (payload: any) => {
    // Retries the inner function up to 3 times
    const result = await retry.onThrow(
      async ({ attempt }) => {
        if (attempt < 3) throw new Error("Transient failure");
        return { foo: "bar" };
      },
      { maxAttempts: 3, randomize: false }
    );

    logger.info("Result", { result });
  },
});
```

The options for `retry.onThrow()` accept the same retry configuration properties as the task-level `retry` object (`maxAttempts`, `factor`, `minTimeoutInMs`, `maxTimeoutInMs`, `randomize`).

> **WARNING:** If all attempts within `retry.onThrow` fail, an error is thrown. You can catch this or let it propagate to trigger a full task-level retry.

---

## `retry.fetch()` -- Automatic HTTP Retry

A convenience wrapper around `fetch` that adds conditional retry logic based on HTTP status codes and timeouts.

### Status-Based Retry Strategies

#### `headers` Strategy (Rate-Limit Aware)

Uses rate-limit headers from the response to determine when to retry:

```ts
import { retry } from "@trigger.dev/sdk";

const response = await retry.fetch("https://api.example.com/data", {
  retry: {
    byStatus: {
      "429": {
        strategy: "headers",
        limitHeader: "x-ratelimit-limit",
        remainingHeader: "x-ratelimit-remaining",
        resetHeader: "x-ratelimit-reset",
        resetFormat: "unix_timestamp_in_ms",
      },
    },
  },
});
```

| Property | Type | Required | Description |
|---|---|---|---|
| `strategy` | `"headers"` | Yes | Use response headers to determine retry timing. |
| `limitHeader` | `string` | Yes | Header name for the rate limit ceiling. |
| `remainingHeader` | `string` | Yes | Header name for the remaining request count. |
| `resetHeader` | `string` | Yes | Header name for when the limit resets. |
| `resetFormat` | `string` | Yes | Format of the reset value (e.g., `"unix_timestamp_in_ms"`). |

#### `backoff` Strategy (Exponential Backoff)

Standard exponential backoff for server errors:

```ts
const response = await retry.fetch("https://api.example.com/data", {
  retry: {
    byStatus: {
      "500-599": {
        strategy: "backoff",
        maxAttempts: 10,
        factor: 2,
        minTimeoutInMs: 1_000,
        maxTimeoutInMs: 30_000,
        randomize: false,
      },
    },
  },
});
```

| Property | Type | Required | Description |
|---|---|---|---|
| `strategy` | `"backoff"` | Yes | Use exponential backoff. |
| `maxAttempts` | `number` | No | Maximum retry attempts. |
| `factor` | `number` | No | Backoff multiplier. |
| `minTimeoutInMs` | `number` | No | Minimum delay. |
| `maxTimeoutInMs` | `number` | No | Maximum delay. |
| `randomize` | `boolean` | No | Add jitter. |

#### Timeout Retry

Retry when a request exceeds a time limit:

```ts
const response = await retry.fetch("https://api.example.com/slow", {
  timeoutInMs: 1000,
  retry: {
    timeout: {
      maxAttempts: 5,
      factor: 1.8,
      minTimeoutInMs: 500,
      maxTimeoutInMs: 30_000,
      randomize: false,
    },
  },
});
```

> **WARNING:** If all retry attempts within `retry.fetch` fail, an error is thrown. You can catch it or let it propagate to trigger a task-level retry.

---

## `AbortTaskRunError` -- Immediate Failure Without Retry

Throw `AbortTaskRunError` to immediately fail a task attempt and **skip all remaining retries**. Use this for permanent errors where retrying would be pointless.

```ts
import { task, AbortTaskRunError } from "@trigger.dev/sdk";

export const processPayment = task({
  id: "process-payment",
  retry: { maxAttempts: 5 },
  run: async (payload: { userId: string; amount: number }) => {
    if (payload.amount <= 0) {
      // Permanent error -- retrying won't fix invalid input
      throw new AbortTaskRunError("Invalid payment amount, will not retry");
    }

    // ... process payment
  },
});
```

`AbortTaskRunError` is imported from `@trigger.dev/sdk`.

---

## `onFailure` Hook -- Runs After All Retries Exhausted

The `onFailure` lifecycle hook is called when a task has **permanently failed** -- all retry attempts have been exhausted or the run was aborted.

### Task-Level `onFailure`

<!-- UNVERIFIED -->
Action item: verify exact `onFailure` callback signature and parameter types in `https://trigger.dev/docs/errors-retrying#onfailure`.

```ts
import { task } from "@trigger.dev/sdk";

export const myTask = task({
  id: "my-task",
  retry: { maxAttempts: 3 },
  run: async (payload) => {
    // task logic
  },
  onFailure: async ({ payload, error, ctx }) => {
    console.error("Task permanently failed:", ctx.task.id, error);
    // Send alert, update database, etc.
  },
});
```

### Config-Level `onFailure` (Global)

Defined in `trigger.config.ts`, runs for every task that fails:

```ts
// trigger.config.ts
import { defineConfig } from "@trigger.dev/sdk";

export default defineConfig({
  project: "<project ref>",
  onFailure: async ({ payload, error, ctx }) => {
    console.log("Task failed:", ctx.task.id);
  },
});
```

### Global `onFailure` via `init.ts`

You can also register a global `onFailure` hook programmatically, useful for error tracking integrations like Sentry:

```ts
// trigger/init.ts
import { tasks } from "@trigger.dev/sdk";
import * as Sentry from "@sentry/node";

Sentry.init({
  defaultIntegrations: false,
  dsn: process.env.SENTRY_DSN,
  environment: process.env.NODE_ENV === "production" ? "production" : "development",
});

tasks.onFailure(({ payload, error, ctx }) => {
  Sentry.captureException(error, {
    extra: { payload, ctx },
  });
});
```

The `trigger/init.ts` file is automatically loaded when your tasks execute.

---

## Error Objects

### `ApiError` (Management SDK)

When the SDK management API cannot connect to the server or receives a non-successful response, it throws an `ApiError`:

```ts
import { runs, ApiError } from "@trigger.dev/sdk";

async function main() {
  try {
    const run = await runs.retrieve("run_1234");
  } catch (error) {
    if (error instanceof ApiError) {
      console.error(`Status: ${error.status}`);
      console.error(`Headers: ${JSON.stringify(error.headers)}`);
      console.error(`Body: ${JSON.stringify(error.body)}`);
    }
  }
}
```

| Property | Type | Description |
|---|---|---|
| `status` | `number` | HTTP status code (e.g., `401`, `404`, `500`). |
| `headers` | `Headers` | Response headers from the API server. |
| `body` | `unknown` | The response body, typically an object with an `error` string. |

### Accessing Errors via Run Attempts

When retrieving a run, the `attempts` array contains error information for each failed attempt:

```ts
import { runs } from "@trigger.dev/sdk";

const result = await runs.retrieve("run_1234");

for (const attempt of result.attempts) {
  if (attempt.status === "FAILED") {
    console.log("Attempt failed:", attempt.error);
    // attempt.error has: { message, name?, stackTrace? }
  }
}
```

#### `SerializedError` Structure

| Property | Type | Required | Description |
|---|---|---|---|
| `message` | `string` | Yes | Human-readable error message. |
| `name` | `string` | No | Error class name (e.g., `"Error"`, `"TypeError"`). |
| `stackTrace` | `string` | No | Stack trace string. |

---

## Combining Retry Strategies

Break work into smaller [tasks](./TRIGGER_DEV_TASKS.md) with independent retry configurations for maximum reliability:

```ts
import { task } from "@trigger.dev/sdk";

export const parentTask = task({
  id: "parent-task",
  retry: { maxAttempts: 10 },
  run: async (payload: string) => {
    const result = await childTask.triggerAndWait("some data");
    // ... continue processing
  },
});

export const childTask = task({
  id: "child-task",
  retry: { maxAttempts: 5 },
  run: async (payload: string) => {
    return { foo: "bar" };
  },
});
```

Each child task retries independently. You can view logs and retry each task separately from the Trigger.dev dashboard.

---

## Using try/catch as an Alternative

Standard try/catch prevents task-level retries for caught errors. Useful for implementing fallback strategies:

```ts
import { task } from "@trigger.dev/sdk";

export const resilientTask = task({
  id: "resilient-task",
  run: async (payload: { prompt: string }) => {
    try {
      const result = await primaryService(payload.prompt);
      return result;
    } catch (error) {
      // Fall back to secondary service instead of retrying
      const fallback = await secondaryService(payload.prompt);
      return fallback;
    }
  },
});
```

---

## Quick Reference

| Mechanism | Scope | When it runs | Key behavior |
|---|---|---|---|
| `retry` (task option) | Single task | Automatically on uncaught error | Exponential backoff with configurable limits |
| `retries.default` (config) | All tasks | Automatically on uncaught error | Project-wide defaults; overridden by task `retry` |
| `catchError` | Single task or config | On uncaught error, before retry decision | Can skip retry, set custom `retryAt`, or modify error |
| `retry.onThrow()` | Code block within task | On throw within the block | Does not restart the entire task; only the block |
| `retry.fetch()` | HTTP request within task | On HTTP error or timeout | Status-code-aware retry with headers or backoff |
| `AbortTaskRunError` | Thrown in task | Immediately | Skips all retries; fails the run permanently |
| `onFailure` | Single task or config | After all retries exhausted | For alerts, cleanup, error tracking; does not affect retry |
| `requestOptions.retry` | Management API calls | On network/server errors | Configurable per-request or globally via `configure()` |

---

## OOM Retry and `OutOfMemoryError`

When a task runs out of memory, Trigger.dev can automatically retry it on a larger machine. This is configured with the `retry.outOfMemory` option.

### `retry.outOfMemory` Option

```ts
export const heavyTask = task({
  id: "heavy-task",
  machine: "medium-1x",
  retry: {
    outOfMemory: {
      machine: "large-1x",
    },
  },
  run: async (payload: any, { ctx }) => {
    // If this OOMs, it retries once on large-1x
  },
});
```

### `OutOfMemoryError` Explicit Throw

If you detect impending OOM conditions (e.g., from a native package or memory monitoring), you can throw `OutOfMemoryError` explicitly to trigger the OOM retry path:

```ts
import { OutOfMemoryError } from "@trigger.dev/sdk";

// Throw when you detect impending OOM from a native package
throw new OutOfMemoryError();
```

These are verified features from https://trigger.dev/docs/machines.

---

## See Also

- [Tasks Overview](./TRIGGER_DEV_TASKS.md)
- [Realtime](./TRIGGER_DEV_REALTIME.md)
- [Master Reference](./TRIGGER_DEV_MASTER.md)
- [Patterns & Recipes](./TRIGGER_DEV_PATTERNS.md)
