# TRIGGER_DEV_QUEUES_CONCURRENCY

Execution control reference for queues, queue overrides, and concurrency semantics.

---

## Default Queue Behavior

- Each task has a default queue.
- Default task queue concurrency is effectively unbounded at task level, constrained by environment concurrency limits.
- Only **actively executing** runs consume queue/env concurrency slots.

> ⚠️ WARNING
> Waiting/checkpointed/delayed runs do not consume active slots; assume bursts can resume later and reacquire slots.

---

## Defining Custom Queues

```ts
import { queue, task } from "@trigger.dev/sdk";

const billingQueue = queue({
  name: "billing",
  concurrencyLimit: 25,
});

export const chargeCustomer = task({
  id: "charge-customer",
  queue: billingQueue,
  run: async (payload: { customerId: string }) => ({ charged: true }),
});
```

| Property | Type | Required | Default | Description |
|---|---|---:|---|---|
| `name` | `string` | Yes | - | Queue identifier |
| `concurrencyLimit` | `number` | No | platform/env | Max concurrent executions in queue |

---

## Concurrency Keys

Use `concurrencyKey` to partition execution by resource/user/tenant.

```ts
await chargeCustomer.trigger(
  { customerId: "cus_123" },
  { concurrencyKey: "tenant:acme", queue: "billing" }
);
```

Behavioral model:

- Unique key values create independent serialized lanes under queue constraints.
- Great for per-tenant rate protection and data-race prevention.

> ⚠️ WARNING
> `concurrencyKey` can dramatically increase lane count; design key cardinality intentionally.

---

## Slot Semantics and Checkpointing

- Run starts -> slot acquired.
- Run hits waitpoint (`wait.*`, `triggerAndWait`) -> run checkpoints and transitions to waiting state -> slot released.
- Run resumes -> slot reacquired.

This is why cross-queue `triggerAndWait` avoids environment deadlock in common parent/child patterns.

---

## Deadlock Prevention in Cross-Queue Orchestration

```ts
import { task } from "@trigger.dev/sdk";
import { childTask } from "./child";

export const parentTask = task({
  id: "parent-task",
  queue: { name: "parent", concurrencyLimit: 10 },
  run: async (payload: { id: string }) => {
    // Parent checkpoints and releases slot while waiting
    return await childTask.triggerAndWait(payload).unwrap();
  },
});
```

When parent waits on child:

1. parent suspends and releases queue/env slot,
2. child runs,
3. parent resumes and reacquires slot.

---

## Queue Management SDK

```ts
import { queues } from "@trigger.dev/sdk";

const page = await queues.list({ page: 1, perPage: 20 });
const q = await queues.retrieve("queue_123");
await queues.pause("queue_123");
await queues.resume("queue_123");
await queues.overrideConcurrencyLimit("queue_123", 100);
await queues.resetConcurrencyLimit("queue_123");
```

### `queues.list(options)`

| Property | Type | Required | Default | Description |
|---|---|---:|---|---|
| `page` | `number` | No | API default | Page number |
| `perPage` | `number` | No | API default | Page size |

### `queues.retrieve(queue)`

| Input variant | Shape | Description |
|---|---|---|
| by id | `"queue_123"` | Direct queue id |
| by name/type | `{ type: "task" \| "custom", name: string }` | Resolve by logical name |

### Pause/resume/override/reset

| Function | Input | Purpose |
|---|---|---|
| `queues.pause(queue)` | id or `{type,name}` | Stop starting new runs (running runs continue) |
| `queues.resume(queue)` | id or `{type,name}` | Resume scheduling |
| `queues.overrideConcurrencyLimit(queue, n)` | id or `{type,name}`, `n: number` | Temporary override |
| `queues.resetConcurrencyLimit(queue)` | id or `{type,name}` | Return to base configured limit |

> ⚠️ WARNING
> `queues.resetConcurrencyLimit` returns 400 when queue is not currently overridden.

---

## Queue Object Fields

Typical queue object fields from management docs:

- `id`, `name`, `type` (`task` or `custom`)
- `paused`
- `running`, `queued`
- `concurrencyLimit`
- `concurrency.base`, `concurrency.current`, `concurrency.override`, `overriddenAt`, `overriddenBy`

---

## Priority and Queues

Priority is a numeric dequeue offset in seconds (`priority: 10` means "treat this run as if it were queued 10 seconds earlier"). No priority defaults to `0`.

Verified behavior from Trigger docs:

- Higher priority runs start sooner within the same organization.
- Priority does not let one organization bypass another organization's queues.

<!-- UNVERIFIED -->
Action item: verify ordering semantics for `priority` + `concurrencyKey` in `https://trigger.dev/docs/queue-concurrency` and `https://trigger.dev/docs/runs/priority`, then document the exact dequeue precedence rules here.

---

## Operational Recommendations

- Put shared external dependency callers in one custom queue.
- Add `concurrencyKey` for per-tenant fairness.
- Keep queue names stable and semantic (`billing`, `email`, `sync`).
- Use temporary override during incidents/backfills, then reset.

Related:

- Task orchestration: [`TRIGGER_DEV_TASKS.md`](./TRIGGER_DEV_TASKS.md)
- Wait/release semantics: [`TRIGGER_DEV_WAITS_TOKENS.md`](./TRIGGER_DEV_WAITS_TOKENS.md)

