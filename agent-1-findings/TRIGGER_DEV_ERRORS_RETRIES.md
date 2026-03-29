# TRIGGER_DEV_ERRORS_RETRIES

Failure modes, retries, and error-control patterns for production Trigger.dev tasks.

---

## Default Retry Behavior

- Uncaught task errors are retried using task/config default retry policy.
- Management API client retries network/server failures (`ApiError` handling).
- `triggerAndWait` returns `Result`; failure may be handled without throwing unless `.unwrap()` is used.

---

## Retry Configuration (`retry` option)

```ts
import { task } from "@trigger.dev/sdk";

export const syncCatalog = task({
  id: "sync-catalog",
  retry: {
    maxAttempts: 5,
    factor: 2,
    minTimeoutInMs: 1_000,
    maxTimeoutInMs: 30_000,
    randomize: true,
  },
  run: async () => ({ ok: true }),
});
```

| Property | Type | Required | Default | Description |
|---|---|---:|---|---|
| `maxAttempts` | `number` | No | platform default | Total attempts including first run |
| `factor` | `number` | No | exponential factor | Backoff multiplier |
| `minTimeoutInMs` | `number` | No | platform default | Minimum retry delay |
| `maxTimeoutInMs` | `number` | No | platform default | Maximum retry delay |
| `randomize` | `boolean` | No | true-ish | Jitter to reduce thundering herds |

---

## `handleError` Hook (Conditional Retry)

```ts
import { task } from "@trigger.dev/sdk";

export const callPartnerApi = task({
  id: "call-partner-api",
  retry: { maxAttempts: 8 },
  handleError: async ({ error }) => {
    // Example pattern: do not retry on 4xx business errors
    const status = (error as any)?.status;
    if (status && status >= 400 && status < 500) return { retry: false };
    return { retry: true };
  },
  run: async () => {
    throw new Error("transient");
  },
});
```

| Property | Type | Required | Default | Description |
|---|---|---:|---|---|
| `retry` | `boolean` | No | task retry policy | Continue retries or stop now |
| `delay` | duration string | No | automatic backoff | Optional override delay before next attempt |

---

## `retry.fetch()`

```ts
import { task, retry } from "@trigger.dev/sdk";

export const fetchWithResilience = task({
  id: "fetch-with-resilience",
  run: async () => {
    const res = await retry.fetch("https://api.example.com/data", {
      retry: { maxAttempts: 10 },
      timeoutInMs: 5_000,
    });
    if (!res.ok) throw new Error(`Bad response: ${res.status}`);
    return await res.json();
  },
});
```

| Property | Type | Required | Default | Description |
|---|---|---:|---|---|
| `retry.maxAttempts` | `number` | No | SDK default | HTTP retry attempts |
| `retry` status map | object | No | SDK defaults | Retry classes (e.g. 500-599) |
| `timeoutInMs` | `number` | No | SDK default | Per-attempt timeout |

---

## `retry.onThrow()`

```ts
import { retry, task } from "@trigger.dev/sdk";

export const parseEventuallyConsistent = task({
  id: "parse-eventually-consistent",
  run: async () =>
    retry.onThrow(
      async () => {
        const ready = await checkRemoteIndex();
        if (!ready) throw new Error("Not ready");
        return getPayload();
      },
      { maxAttempts: 6, minTimeoutInMs: 500, maxTimeoutInMs: 10_000 }
    ),
});
```

---

## `AbortTaskRunError` (Fail Fast, No Retry)

```ts
import { task, AbortTaskRunError } from "@trigger.dev/sdk";

export const validatePayload = task({
  id: "validate-payload",
  run: async (payload: { userId?: string }) => {
    if (!payload.userId) {
      throw new AbortTaskRunError("Missing userId; not retriable");
    }
    return { ok: true };
  },
});
```

Use for deterministic business-invalid input where retries will never succeed.

---

## `onFailure` Hook

Runs after retries are exhausted.

```ts
import { task } from "@trigger.dev/sdk";

export const notifyOps = task({
  id: "notify-ops",
  onFailure: async ({ error, ctx }) => {
    await postToPagerDuty({ runId: ctx.run.id, message: error.message });
  },
  run: async () => {
    throw new Error("boom");
  },
});
```

---

## Error Objects and Attempts

From runs APIs:

- Run statuses include `FAILED`, `CRASHED`, `INTERRUPTED`, `SYSTEM_FAILURE`, `CANCELED`.
- Attempts include `PENDING`, `EXECUTING`, `PAUSED`, `COMPLETED`, `FAILED`, `CANCELED`.
- Retrieve run details to inspect `attempts[]` and serialized error fields (`message`, `name`, `stackTrace`).

```ts
import { runs } from "@trigger.dev/sdk";

const run = await runs.retrieve("run_123");
for (const attempt of run.attempts ?? []) {
  if (attempt.status === "FAILED") console.error(attempt.error?.message);
}
```

---

## Config-level Defaults (`trigger.config.ts`)

```ts
import { defineConfig } from "@trigger.dev/sdk";

export default defineConfig({
  retries: {
    enabledInDev: false,
    default: {
      maxAttempts: 3,
      minTimeoutInMs: 1_000,
      maxTimeoutInMs: 10_000,
      factor: 2,
      randomize: true,
    },
  },
});
```

---

## Management API Error Handling

```ts
import { runs, ApiError } from "@trigger.dev/sdk";

try {
  await runs.retrieve("run_123");
} catch (error) {
  if (error instanceof ApiError) {
    console.error(error.status, error.body);
  }
}
```

> ⚠️ WARNING
> In-task SDK calls may use task-optimized retry semantics; per-request client retry overrides are not always honored for task-triggering helper methods.

Related:

- Task contracts/orchestration: [`TRIGGER_DEV_TASKS.md`](./TRIGGER_DEV_TASKS.md)
- Queue/release behavior during waits: [`TRIGGER_DEV_QUEUES_CONCURRENCY.md`](./TRIGGER_DEV_QUEUES_CONCURRENCY.md)

