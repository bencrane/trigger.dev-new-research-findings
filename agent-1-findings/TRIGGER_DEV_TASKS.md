# TRIGGER_DEV_TASKS

Deep reference for task definition, orchestration, scheduling, and triggering APIs.

Version scope: `@trigger.dev/sdk` `4.4.x`.

---

## `task()` Definition

```ts
import { task, logger } from "@trigger.dev/sdk";

export const processInvoice = task({
  id: "process-invoice",
  queue: { name: "billing", concurrencyLimit: 20 },
  machine: "small-1x",
  maxDuration: 60 * 30,
  retry: { maxAttempts: 5, factor: 2, minTimeoutInMs: 1_000, maxTimeoutInMs: 30_000, randomize: true },
  run: async (payload: { invoiceId: string }, { ctx }) => {
    logger.info("Processing", { runId: ctx.run.id, invoiceId: payload.invoiceId });
    return { ok: true };
  },
});
```

| Property | Type | Required | Default | Description |
|---|---|---:|---|---|
| `id` | `string` | Yes | - | Stable unique task identifier |
| `run` | function | Yes | - | Task body |
| `queue` | queue ref/object | No | per-task default queue | Queue lane and concurrency limit |
| `retry` | object | No | config/default platform | Retry policy for uncaught failures |
| `machine` | machine enum | No | config default | CPU/RAM preset |
| `maxDuration` | `number` sec | No | config/platform | Max runtime before timeout fail |
| `middleware` | array | No | none | Task middleware chain |
| `onStart/onSuccess/onFailure` | function | No | none | Lifecycle hooks |

> ⚠️ WARNING
> Tasks must be exported from files included in configured `dirs`, or the build will not register them.

---

## `schemaTask()` (Typed contracts)

```ts
import { schemaTask } from "@trigger.dev/sdk";
import { z } from "zod";

const Input = z.object({ userId: z.string(), plan: z.enum(["free", "pro"]) });
const Output = z.object({ upgraded: z.boolean(), appliedAt: z.string() });

export const upgradePlan = schemaTask({
  id: "upgrade-plan",
  schema: Input,
  output: Output,
  run: async (payload) => ({ upgraded: true, appliedAt: new Date().toISOString() }),
});
```

| Property | Type | Required | Default | Description |
|---|---|---:|---|---|
| `id` | `string` | Yes | - | Task identifier |
| `schema` | Zod schema | Yes | - | Input validation |
| `output` | Zod schema | No | none | Output validation/inference |
| `run` | function | Yes | - | Typed task body |

---

## Scheduled Tasks

### Declarative schedule attached in code

```ts
import { schedules, task } from "@trigger.dev/sdk";

export const dailyReconcile = task({
  id: "daily-reconcile",
  run: async () => ({ done: true }),
});

schedules.task({
  id: "daily-reconcile-schedule",
  task: dailyReconcile,
  cron: "0 2 * * *",
  timezone: "America/New_York",
  deduplicationKey: "daily-reconcile-nyc",
});
```

### Imperative schedule (management-time)

```ts
import { schedules } from "@trigger.dev/sdk";

await schedules.create({
  task: "daily-reconcile",
  cron: "0 2 * * *",
  timezone: "America/New_York",
  deduplicationKey: "daily-reconcile-nyc",
});
```

| Property | Type | Required | Default | Description |
|---|---|---:|---|---|
| `task` | `string` task id | Yes | - | Task to invoke |
| `cron` | cron `string` | Yes | - | Trigger cadence |
| `timezone` | IANA timezone | No | `UTC` | Cron timezone (DST aware) |
| `deduplicationKey` | `string` | Create: Yes | - | Prevent duplicate schedule creation |
| `externalId` | `string` | No | none | External mapping id |

> ⚠️ WARNING
> For dynamic schedule creation, always set `deduplicationKey` to avoid duplicate schedule fan-out and accidental cost spikes.

---

## Lifecycle Hooks

- **Task-level hooks:** `onStart`, `onSuccess`, `onFailure`.
- **Global hooks in config:** `init`, `onStart`, `onSuccess`, `onFailure`.
- **Error control hook:** `handleError` (see [`TRIGGER_DEV_ERRORS_RETRIES.md`](./TRIGGER_DEV_ERRORS_RETRIES.md)).

```ts
import { task } from "@trigger.dev/sdk";

export const sendEmail = task({
  id: "send-email",
  onStart: async ({ ctx }) => console.log("start", ctx.run.id),
  onSuccess: async ({ output }) => console.log("ok", output),
  onFailure: async ({ error }) => console.error("failed", error.message),
  run: async (payload: { to: string }) => ({ sent: true, to: payload.to }),
});
```

---

## `run` Function Context

Inside `run(payload, { ...context })`, commonly used context primitives include:

- `ctx` (run/task identifiers, environment details),
- logging (`logger`),
- `wait` APIs,
- metadata and tags APIs,
- subtask triggering helpers.

Use progress metadata for realtime UI:

```ts
import { task, metadata } from "@trigger.dev/sdk";

export const importData = task({
  id: "import-data",
  run: async () => {
    await metadata.set({ stage: "download", progress: 10 });
    await metadata.set({ stage: "transform", progress: 60 });
    return { imported: true };
  },
});
```

---

## Triggering Methods

### Fire-and-forget

```ts
const handle = await processInvoice.trigger(
  { invoiceId: "inv_123" },
  {
    idempotencyKey: "inv_123:v1",
    concurrencyKey: "tenant:acme",
    delay: "5m",
    ttl: "2h",
    tags: ["tenant:acme", "workflow:billing"],
    metadata: { source: "api" },
    machine: "medium-1x",
    queue: { name: "billing", concurrencyLimit: 25 },
  }
);
```

### Wait for completion

```ts
const result = await processInvoice.triggerAndWait({ invoiceId: "inv_123" });
const output = result.unwrap(); // throws if failed
```

### Batch variants

```ts
const batch = await processInvoice.batchTrigger([
  { payload: { invoiceId: "inv_1" } },
  { payload: { invoiceId: "inv_2" }, options: { idempotencyKey: "inv_2:v1" } },
]);

const results = await processInvoice.batchTriggerAndWait([
  { payload: { invoiceId: "inv_1" } },
  { payload: { invoiceId: "inv_2" } },
]);
```

| Property | Type | Required | Default | Description |
|---|---|---:|---|---|
| `idempotencyKey` | `string` | No | none | Deduplicate duplicate trigger calls |
| `concurrencyKey` | `string` | No | none | Per-key concurrency partition |
| `queue` | `string` or queue object | No | task queue | Override queue routing |
| `delay` | duration/date | No | immediate | Delay start |
| `ttl` | duration/seconds | No | platform default | Drop if not started before expiry |
| `tags` | `string[]` | No | none | Run classification/filtering |
| `metadata` | `Record<string, unknown>` | No | none | Initial run metadata |
| `machine` | machine enum | No | task/config | Override compute class |
| `priority` | `number` | No | `0` | Priority offset in seconds; higher values dequeue sooner within your organization |

> ⚠️ WARNING
> `batchTrigger` item cap is 1,000 on SDK `4.3.1+` (500 before that).

> ⚠️ WARNING
> Do not wrap `triggerAndWait` or `batchTriggerAndWait` in `Promise.all` inside a running task.

---

## Subtask Patterns

```ts
import { task } from "@trigger.dev/sdk";
import { validateOrder } from "./validate-order";

export const placeOrder = task({
  id: "place-order",
  run: async (payload: { orderId: string }) => {
    const validation = await validateOrder.triggerAndWait(payload).unwrap();
    if (!validation.ok) throw new Error("Invalid order");
    return { accepted: true };
  },
});
```

Guidance:

- Use `triggerAndWait` for strict dependency chains.
- Use `trigger` + tags/metadata when downstream task can be eventual.
- Use idempotency keys in parent and child task boundaries.

---

## Hidden Tasks

Hidden tasks are useful for internal orchestration tasks not intended for manual/dashboard triggering.

<!-- UNVERIFIED -->
Action item: verify hidden-task option name and shape in `https://trigger.dev/docs/tasks/overview` (or current equivalent) and update this section with the exact property signature and example.

---

## Related

- Queues/concurrency: [`TRIGGER_DEV_QUEUES_CONCURRENCY.md`](./TRIGGER_DEV_QUEUES_CONCURRENCY.md)
- Waits/tokens: [`TRIGGER_DEV_WAITS_TOKENS.md`](./TRIGGER_DEV_WAITS_TOKENS.md)
- Errors/retries: [`TRIGGER_DEV_ERRORS_RETRIES.md`](./TRIGGER_DEV_ERRORS_RETRIES.md)

