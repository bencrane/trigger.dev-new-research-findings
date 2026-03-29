# Trigger.dev Queues and Concurrency

> Canonical reference for queue behavior, concurrency controls, concurrency keys, checkpointing, queue management SDK, deadlock prevention, and priority in Trigger.dev.
>
> **SDK version**: `@trigger.dev/sdk` 4.4.3

---

## Table of Contents

- [Trigger.dev Queues and Concurrency](#triggerdev-queues-and-concurrency)
  - [Table of Contents](#table-of-contents)
  - [Default Queue Behavior](#default-queue-behavior)
  - [Custom Queues](#custom-queues)
  - [Concurrency Limits](#concurrency-limits)
  - [Concurrency Keys](#concurrency-keys)
    - [Common concurrency key patterns](#common-concurrency-key-patterns)
  - [How Concurrency Slots Work](#how-concurrency-slots-work)
  - [Checkpointing Behavior](#checkpointing-behavior)
  - [Queue Management SDK](#queue-management-sdk)
    - [queues.list()](#queueslist)
    - [queues.retrieve()](#queuesretrieve)
    - [queues.pause() / queues.resume()](#queuespause--queuesresume)
    - [queues.overrideConcurrencyLimit()](#queuesoverrideconcurrencylimit)
    - [queues.resetConcurrencyLimit()](#queuesresetconcurrencylimit)
  - [QueueObject Schema](#queueobject-schema)
  - [Deadlock Prevention](#deadlock-prevention)
  - [Priority](#priority)
  - [Related Documentation](#related-documentation)

---

## Default Queue Behavior

Every task in Trigger.dev automatically gets its own queue. The queue name matches the task ID.

```ts
import { task } from "@trigger.dev/sdk";

export const processOrder = task({
  id: "process-order",
  // Implicitly creates a queue named "process-order"
  // No concurrency limit by default — unbounded
  run: async (payload: { orderId: string }) => {
    // ...
  },
});
```

By default, task queues have **no concurrency limit** — runs execute as fast as infrastructure allows. If you need to throttle execution, set a `concurrencyLimit` on the task or use a custom queue.

```ts
export const processOrder = task({
  id: "process-order",
  queue: {
    concurrencyLimit: 10, // At most 10 runs execute concurrently
  },
  run: async (payload) => {
    // ...
  },
});
```

---

## Custom Queues

Custom queues let you share a single concurrency pool across multiple tasks. Define a queue with the `queue()` function and reference it in one or more tasks.

```ts
import { queue, task } from "@trigger.dev/sdk";

const emailQueue = queue({
  name: "email-sending",
  concurrencyLimit: 5,
});

export const sendWelcomeEmail = task({
  id: "send-welcome-email",
  queue: emailQueue,
  run: async (payload: { email: string }) => {
    // ...
  },
});

export const sendReceiptEmail = task({
  id: "send-receipt-email",
  queue: emailQueue,
  run: async (payload: { email: string; orderId: string }) => {
    // ...
  },
});
```

Both tasks share the `email-sending` queue. At most 5 runs across *both* tasks execute at the same time.

> **Note**: A custom queue's `type` field is `"custom"` in the API, while auto-created task queues have type `"task"`.

---

## Concurrency Limits

| Where set | Scope | Example |
|---|---|---|
| `task({ queue: { concurrencyLimit: N } })` | Per-task default queue | Limits that single task |
| `queue({ name, concurrencyLimit: N })` | Custom shared queue | Limits all tasks using the queue |
| `queues.overrideConcurrencyLimit()` | Runtime API override | Temporarily changes the limit |

The effective concurrency limit (`concurrency.current`) is determined by:

1. **Override** — if set via the API, the override wins.
2. **Base** — the value defined in code (`concurrency.base`).
3. **null** — no limit (unbounded).

---

## Concurrency Keys

Concurrency keys let you create **dynamic sub-queues** within a single queue. Common patterns include per-user, per-tenant, or per-resource queuing.

<!-- UNVERIFIED: concurrencyKey API shape confirmed via Trigger.dev docs pattern, but exact runtime behavior not verified against SDK 4.4.3 source -->

```ts
import { task } from "@trigger.dev/sdk";

export const processUserAction = task({
  id: "process-user-action",
  queue: {
    concurrencyLimit: 1, // One run at a time per key
  },
  run: async (payload: { userId: string; action: string }) => {
    // ...
  },
});

// When triggering, pass a concurrencyKey:
await processUserAction.trigger(
  { userId: "user_123", action: "update-profile" },
  { concurrencyKey: "user_123" }
);
```

With `concurrencyLimit: 1` and a per-user `concurrencyKey`, only one run per user executes at a time. Other users' runs proceed independently.

### Common concurrency key patterns

| Pattern | Key value | Use case |
|---|---|---|
| Per-user | `user_${userId}` | Serialize operations per user |
| Per-tenant | `tenant_${orgId}` | Rate-limit per organization |
| Per-resource | `doc_${documentId}` | Prevent concurrent edits to same resource |
| Per-API | `api_stripe` | Respect external API rate limits |

> **WARNING**: Concurrency keys are scoped to the queue, not globally. Two different queues using the same concurrency key string operate independently.

---

## How Concurrency Slots Work

A concurrency slot is consumed **only while a run is actively executing**. Specifically:

| Run state | Consumes a slot? |
|---|---|
| **EXECUTING** | Yes |
| **QUEUED** | No |
| **DELAYED** | No |
| **WAITING** (e.g., `wait.for()`, `wait.forToken()`) | No — slot is released via checkpointing |
| **FROZEN** / checkpointed | No |
| **COMPLETED** / **FAILED** / **CANCELED** | No |

This means a queue with `concurrencyLimit: 1` can have hundreds of queued and waiting runs without blocking new work from starting.

---

## Checkpointing Behavior

When a run enters a wait state (via `wait.for()`, `wait.until()`, `wait.forToken()`, `triggerAndWait()`, or `batchTriggerAndWait()`), Trigger.dev **checkpoints** the run:

1. The run's execution state is saved to durable storage.
2. The compute container is released.
3. The concurrency slot is freed.
4. No compute costs accrue during the wait.

When the wait condition is met, the run is restored from the checkpoint and resumes execution, re-acquiring a concurrency slot.

<!-- UNVERIFIED: The 5-second threshold for checkpointing (waits > 5s checkpoint, waits <= 5s may spin-wait) is documented in Trigger.dev docs but exact behavior may vary by plan -->

> **WARNING**: Waits shorter than approximately 5 seconds may not trigger a full checkpoint. Very short waits could keep the container alive and continue to consume a concurrency slot and compute time.

```ts
import { task, wait } from "@trigger.dev/sdk";

export const longRunningTask = task({
  id: "long-running-task",
  queue: { concurrencyLimit: 2 },
  run: async (payload) => {
    await doStepOne(payload);

    // This wait checkpoints the run — the slot is released
    await wait.for({ minutes: 30 });

    // When resumed, the run re-acquires a slot
    await doStepTwo(payload);
  },
});
```

---

## Queue Management SDK

All queue management functions are available from the `queues` export:

```ts
import { queues } from "@trigger.dev/sdk";
```

Authentication uses the `TRIGGER_SECRET_KEY` environment variable, or can be configured explicitly:

```ts
import { configure } from "@trigger.dev/sdk";

configure({ accessToken: "tr_dev_..." });
```

---

### queues.list()

List all queues in the current environment with page-based pagination.

```ts
import { queues } from "@trigger.dev/sdk";

// List all queues (defaults)
const allQueues = await queues.list();

// With pagination
const pagedQueues = await queues.list({
  page: 1,
  perPage: 20,
});

console.log(pagedQueues.data);        // QueueObject[]
console.log(pagedQueues.pagination);  // { currentPage, totalPages, count }
```

**REST**: `GET /api/v1/queues`

| Parameter | Type | Required | Description |
|---|---|---|---|
| `page` | `integer` | No | Page number (1-based) |
| `perPage` | `integer` | No | Number of queues per page |

**Response**:

| Field | Type | Description |
|---|---|---|
| `data` | `QueueObject[]` | Array of queue objects |
| `pagination.currentPage` | `integer` | Current page number |
| `pagination.totalPages` | `integer` | Total number of pages |
| `pagination.count` | `integer` | Total number of queues |

---

### queues.retrieve()

Retrieve a single queue by ID, task name, or custom queue name.

```ts
import { queues } from "@trigger.dev/sdk";

// By queue ID
const queue = await queues.retrieve("queue_1234");

// By task name (get that task's default queue)
const taskQueue = await queues.retrieve({
  type: "task",
  name: "my-task-id",
});

// By custom queue name
const customQueue = await queues.retrieve({
  type: "custom",
  name: "my-custom-queue",
});
```

**REST**: `GET /api/v1/queues/{queueParam}`

| Parameter | Type | Required | Description |
|---|---|---|---|
| `queueParam` (path) | `string` | Yes | Queue ID, or name when using `type` |
| `type` (query) | `"id" \| "task" \| "custom"` | No | How to interpret `queueParam`. Default: `"id"` |

**Response**: A single [`QueueObject`](#queueobject-schema).

Returns `404` if the queue is not found.

---

### queues.pause() / queues.resume()

Pause a queue to prevent new runs from starting. Resume a paused queue to allow new runs again.

```ts
import { queues } from "@trigger.dev/sdk";

// Pause by queue ID
await queues.pause("queue_1234");

// Pause by task name
await queues.pause({ type: "task", name: "my-task-id" });

// Resume
await queues.resume("queue_1234");
await queues.resume({ type: "task", name: "my-task-id" });
```

**REST**: `POST /api/v1/queues/{queueParam}/pause`

| Body field | Type | Required | Description |
|---|---|---|---|
| `action` | `"pause" \| "resume"` | Yes | Whether to pause or resume |
| `type` | `"id" \| "task" \| "custom"` | No | How to interpret the path parameter |

> **WARNING**: Pausing a queue does **not** stop currently executing runs. They continue to completion. Only new runs are blocked from starting. Queued runs remain in the queue and will start when the queue is resumed.

**Response**: The updated [`QueueObject`](#queueobject-schema).

---

### queues.overrideConcurrencyLimit()

Temporarily override a queue's concurrency limit at runtime. Useful for scaling up during peak load or throttling down during incidents.

```ts
import { queues } from "@trigger.dev/sdk";

// Override to 5 concurrent runs
await queues.overrideConcurrencyLimit("queue_1234", 5);

// Using type and name
await queues.overrideConcurrencyLimit(
  { type: "task", name: "my-task-id" },
  20
);

// Set to 0 to effectively pause execution (no new runs can acquire a slot)
await queues.overrideConcurrencyLimit(
  { type: "custom", name: "email-sending" },
  0
);
```

**REST**: `POST /api/v1/queues/{queueParam}/concurrency/override`

| Body field | Type | Required | Description |
|---|---|---|---|
| `concurrencyLimit` | `integer` (0–100000) | Yes | The new concurrency limit |
| `type` | `"id" \| "task" \| "custom"` | No | How to interpret the path parameter |

**Response**: The updated [`QueueObject`](#queueobject-schema). The `concurrency.override` field reflects the new value, and `concurrency.current` equals the override.

---

### queues.resetConcurrencyLimit()

Reset the concurrency limit back to the base value defined in code, removing any override.

```ts
import { queues } from "@trigger.dev/sdk";

await queues.resetConcurrencyLimit("queue_1234");

await queues.resetConcurrencyLimit({
  type: "task",
  name: "my-task-id",
});
```

**REST**: `POST /api/v1/queues/{queueParam}/concurrency/reset`

| Body field | Type | Required | Description |
|---|---|---|---|
| `type` | `"id" \| "task" \| "custom"` | No | How to interpret the path parameter |

> **WARNING**: Returns `400 Bad Request` if the queue does not currently have an override applied. Check `concurrency.override` before calling this.

**Response**: The updated [`QueueObject`](#queueobject-schema) with `concurrency.override` set to `null`.

---

## QueueObject Schema

Every queue API response returns a `QueueObject` with the following shape:

| Field | Type | Description |
|---|---|---|
| `id` | `string` | Queue ID (e.g. `queue_1234`) |
| `name` | `string` | Queue name. For task queues, this is the task ID. For custom queues, this is the name passed to `queue()`. |
| `type` | `"task" \| "custom"` | Whether the queue was auto-created for a task or explicitly defined |
| `running` | `integer` | Number of runs currently executing |
| `queued` | `integer` | Number of runs currently queued (waiting for a slot) |
| `paused` | `boolean` | Whether the queue is paused |
| `concurrencyLimit` | `integer \| null` | The effective concurrency limit (`null` = unbounded) |
| `concurrency.current` | `integer \| null` | The effective/current concurrency limit |
| `concurrency.base` | `integer \| null` | The base concurrency limit defined in code |
| `concurrency.override` | `integer \| null` | The runtime override (if set via API) |
| `concurrency.overriddenAt` | `string (ISO 8601) \| null` | When the override was set |
| `concurrency.overriddenBy` | `string \| null` | Who set the override (`null` if set via API) |

**Concurrency resolution logic**:

```
effective = concurrency.override ?? concurrency.base ?? null (unbounded)
```

---

## Deadlock Prevention

When using `triggerAndWait()` or `batchTriggerAndWait()` to call a child task, if both the parent and child share the same queue with a concurrency limit, you risk a **deadlock**: the parent holds a slot while waiting for the child, but the child cannot start because all slots are taken.

<!-- UNVERIFIED: Trigger.dev has built-in deadlock prevention for cross-queue triggerAndWait since v3, but exact mechanism details (reserved slots, separate pool) are not fully specified in public docs -->

Trigger.dev includes built-in deadlock prevention for `triggerAndWait` and `batchTriggerAndWait`:

- When a parent run calls `triggerAndWait()`, it checkpoints and **releases its concurrency slot**.
- The child run can then acquire the slot and execute.
- When the child completes, the parent is restored and re-acquires a slot.

However, best practices to avoid contention:

```ts
import { queue, task } from "@trigger.dev/sdk";

// Use separate queues for parent and child tasks
const parentQueue = queue({ name: "parent-work", concurrencyLimit: 5 });
const childQueue = queue({ name: "child-work", concurrencyLimit: 10 });

export const parentTask = task({
  id: "parent-task",
  queue: parentQueue,
  run: async (payload) => {
    // Safe: child uses a different queue
    const result = await childTask.triggerAndWait({ data: payload.data });
    return result.unwrap();
  },
});

export const childTask = task({
  id: "child-task",
  queue: childQueue,
  run: async (payload) => {
    return { processed: true };
  },
});
```

> **WARNING**: Never wrap `triggerAndWait()` or `batchTriggerAndWait()` in `Promise.all()`. This is not supported and can lead to undefined behavior. Call them sequentially instead.

---

## Priority

<!-- UNVERIFIED: Priority feature details based on Trigger.dev docs; exact numeric range and behavior may vary -->

Trigger.dev supports **run priority** to influence the order in which queued runs are dequeued. Higher-priority runs are picked up before lower-priority ones when a concurrency slot becomes available.

Priority is set when triggering a run:

```ts
import { myTask } from "./trigger/my-task";

// Higher priority number = picked up sooner
await myTask.trigger(
  { orderId: "rush-123" },
  { priority: 100 }
);

// Default priority
await myTask.trigger(
  { orderId: "normal-456" }
);
```

Priority interacts with queues as follows:

- Priority applies **within a single queue**. Runs in different queues are scheduled independently.
- When a concurrency slot opens, the highest-priority queued run in that queue is dequeued next.
- Runs with the same priority are dequeued in FIFO order.
- Priority does not preempt executing runs — it only affects the order of queued runs.

---

## Related Documentation

- [Waits and Tokens](./TRIGGER_DEV_WAITS_TOKENS.md) — duration waits, date waits, and waitpoint tokens
- [Queues API — List Queues](../01-api-docs/04-queues-api/00-list-queues.md)
- [Queues API — Retrieve Queue](../01-api-docs/04-queues-api/01-retrieve-queue.md)
- [Queues API — Pause/Resume](../01-api-docs/04-queues-api/02-pause-or-resume-queue.md)
- [Queues API — Override Concurrency](../01-api-docs/04-queues-api/03-override-concurrency-limit.md)
- [Queues API — Reset Concurrency](../01-api-docs/04-queues-api/04-reset-concurrency-limit.md)
