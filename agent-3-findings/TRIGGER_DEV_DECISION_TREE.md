# Trigger.dev Decision Trees

> SDK version: **4.4.3** | These decision trees help you make the right architectural choice fast.

Use this document when you are designing a new feature and need to decide how to structure your Trigger.dev tasks. Each section contains a key question, a decision flowchart, concrete examples, and links to reference docs.

---

## Table of Contents

- [Trigger.dev Decision Trees](#triggerdev-decision-trees)
  - [Table of Contents](#table-of-contents)
  - [1. Should this be a Trigger.dev task or inline code?](#1-should-this-be-a-triggerdev-task-or-inline-code)
    - [Decision Flowchart](#decision-flowchart)
    - [Decision Criteria](#decision-criteria)
    - [Concrete Examples](#concrete-examples)
  - [2. Should I use `trigger()` or `triggerAndWait()`?](#2-should-i-use-trigger-or-triggerandwait)
    - [Decision Flowchart](#decision-flowchart-1)
    - [Key Constraints](#key-constraints)
    - [Concrete Examples](#concrete-examples-1)
  - [3. Should I use a custom queue or the default?](#3-should-i-use-a-custom-queue-or-the-default)
    - [Decision Flowchart](#decision-flowchart-2)
    - [When to Use Each Option](#when-to-use-each-option)
    - [Concrete Examples](#concrete-examples-2)
  - [4. Should I use `wait.for` or `wait.forToken`?](#4-should-i-use-waitfor-or-waitfortoken)
    - [Decision Flowchart](#decision-flowchart-3)
    - [Key Differences](#key-differences)
    - [Concrete Examples](#concrete-examples-3)
  - [5. Should I use batch trigger or a loop of individual triggers?](#5-should-i-use-batch-trigger-or-a-loop-of-individual-triggers)
    - [Decision Flowchart](#decision-flowchart-4)
    - [Key Differences](#key-differences-1)
    - [Concrete Examples](#concrete-examples-4)
  - [6. Should I use schedules or an external cron?](#6-should-i-use-schedules-or-an-external-cron)
    - [Decision Flowchart](#decision-flowchart-5)
    - [Key Differences](#key-differences-2)
    - [Concrete Examples](#concrete-examples-5)
    - [Recommendation](#recommendation)
  - [Cross-Reference Index](#cross-reference-index)

---

## 1. Should this be a Trigger.dev task or inline code?

**Key question:** Does this work need to happen in the background, survive failures, or run longer than your request timeout?

### Decision Flowchart

```
START: You have work to do
  |
  v
Does it need to complete in < 1 second for a user response?
  |
  |-- YES --> Will it ever fail and need retrying?
  |              |
  |              |-- YES --> Use a Trigger.dev task (fire-and-forget with `trigger()`)
  |              |-- NO  --> Inline code is fine
  |
  |-- NO ----> Could it take > 10 seconds?
                 |
                 |-- YES --> Trigger.dev task (always)
                 |-- NO  --> Does it need retries, observability, or queuing?
                               |
                               |-- YES --> Trigger.dev task
                               |-- NO  --> Inline code is fine
```

### Decision Criteria

| Factor | Inline Code | Trigger.dev Task |
| :--- | :--- | :--- |
| **Duration** | < 10s, within request timeout | Any duration, minutes to hours |
| **Failure handling** | Try/catch, manual retry | Automatic retries with backoff |
| **Observability** | Application logs only | Full run traces, dashboard, OpenTelemetry |
| **User response** | Must return immediately | Can run in background |
| **Queuing** | No built-in queuing | Concurrency limits, per-tenant isolation |
| **Cost** | Runs on your server | Runs on Trigger.dev infra (or self-hosted) |

### Concrete Examples

**Use inline code:**
- Validating a form input
- Reading a single database row to populate a page
- Checking if a user is authenticated

**Use a Trigger.dev task:**
- Sending a welcome email sequence (uses `wait.for` for delays between emails)
- Processing an uploaded CSV with 10,000 rows
- Calling an external AI API that might take 30+ seconds and could rate-limit you
- Generating a PDF report and uploading it to storage
- Syncing data from a third-party API nightly

**Hybrid approach:** Trigger the task from your API route, return a run handle to the client, and let the client poll or subscribe via Realtime for the result:

```typescript
// API route: fast response, background work
export async function POST(req: Request) {
  const handle = await tasks.trigger<typeof generateReport>("generate-report", {
    userId: "123",
  });
  return Response.json({ runId: handle.id }); // Return immediately
}
```

**Related:** [TRIGGER_DEV_PATTERNS.md](./TRIGGER_DEV_PATTERNS.md) -- all 10 patterns use tasks for background work

---

## 2. Should I use `trigger()` or `triggerAndWait()`?

**Key question:** Does the calling code need the result of the child task before it can continue?

### Decision Flowchart

```
START: You want to run a child task
  |
  v
Do you need the child task's return value?
  |
  |-- YES --> Is the caller itself a Trigger.dev task?
  |              |
  |              |-- YES --> Use triggerAndWait() or .unwrap()
  |              |-- NO  --> You CANNOT use triggerAndWait() from backend code.
  |                          Use trigger() + poll/subscribe for the result.
  |
  |-- NO ----> Do you need to know if the child succeeded?
                 |
                 |-- YES --> Use triggerAndWait() to check result.ok
                 |-- NO  --> Use trigger() (fire-and-forget)
```

### Key Constraints

- `triggerAndWait()` can **only** be called from inside another Trigger.dev task
- `triggerAndWait()` version-locks the child to the parent's deploy version
- `trigger()` can be called from anywhere (API routes, serverless functions, other tasks)
- `trigger()` does NOT version-lock the child -- it runs on the current deployed version
- Never wrap `triggerAndWait()` or `batchTriggerAndWait()` in `Promise.all`

### Concrete Examples

**Use `trigger()` (fire-and-forget):**
- Your API route receives a webhook and kicks off background processing
- A user action triggers an async job; the user does not wait for the result
- You want to fan out to many children but do not need to aggregate their results

```typescript
// Backend API route -- fire and forget
const handle = await tasks.trigger<typeof sendWelcomeEmail>(
  "send-welcome-email",
  { userId: "123", email: "user@example.com" }
);
// Returns immediately with { id: "run_abc123" }
```

**Use `triggerAndWait()` (synchronous subtask):**
- A parent task needs to chain steps: fetch data, then transform, then load
- You need to collect results from child tasks before sending a final response
- You are building a pipeline where Step B depends on Step A's output

```typescript
// Inside a parent task
const result = await fetchDataTask.triggerAndWait({ url: sourceUrl });
if (!result.ok) {
  throw new Error("Fetch failed");
}
// Now use result.output in the next step
const transformed = await transformTask.triggerAndWait({
  data: result.output.records,
});
```

**Use `batchTriggerAndWait()` (parallel fan-out with collection):**
- Processing N items in parallel and collecting all results
- Running multiple AI models simultaneously and comparing outputs

```typescript
// Fan out to up to 1,000 children, wait for all
const batch = await processItem.batchTriggerAndWait(
  items.map((item) => ({ payload: item }))
);
const successes = batch.runs.filter((r) => r.ok).map((r) => r.output);
```

**Related:** [Tasks](./TRIGGER_DEV_TASKS.md) | [Management API](./TRIGGER_DEV_MANAGEMENT_API.md) | [Fan-out Pattern](./TRIGGER_DEV_PATTERNS.md#1-fan-out-batch-trigger-n-subtasks)

---

## 3. Should I use a custom queue or the default?

**Key question:** Do you need to control concurrency across tasks, isolate tenants, or rate-limit against an external API?

### Decision Flowchart

```
START: You are defining a task
  |
  v
Do multiple tasks share a concurrency limit?
  |
  |-- YES --> Use a shared custom queue
  |
  |-- NO ----> Do you need per-tenant isolation (concurrencyKey)?
                 |
                 |-- YES --> Use a custom queue with concurrencyKey
                 |
                 |-- NO ----> Do you need to set a specific concurrency limit?
                                |
                                |-- YES --> Does only this one task need the limit?
                                |             |
                                |             |-- YES --> Set concurrency on the task's
                                |             |            default queue (no custom queue needed)
                                |             |-- NO  --> Use a shared custom queue
                                |
                                |-- NO ----> Use the default queue (no configuration needed)
```

### When to Use Each Option

| Scenario | Queue Type | Example |
| :--- | :--- | :--- |
| Simple task, no concurrency concerns | Default (no config) | A one-off data transform task |
| Limit this task to N concurrent runs | Task-level queue config | An API call task limited to 5 concurrent runs |
| Multiple tasks share one API rate limit | Shared custom queue | 3 different tasks all call the same external API (limit: 10 total) |
| Per-tenant concurrency isolation | Custom queue + concurrencyKey | SaaS platform where each customer gets 2 concurrent runs |
| Dynamic concurrency at runtime | Custom queue + API override | Temporarily burst capacity for a VIP customer |

### Concrete Examples

**Default queue (no configuration):**

```typescript
// No queue config needed -- uses the automatic per-task queue
export const simpleTask = task({
  id: "simple-task",
  run: async (payload) => { /* ... */ },
});
```

**Custom queue with shared concurrency:**

```typescript
import { queue, task } from "@trigger.dev/sdk";

// Both tasks share this queue's concurrency limit
const openaiQueue = queue({
  name: "openai-calls",
  concurrencyLimit: 10, // Max 10 concurrent OpenAI calls across all tasks
});

export const summarizeText = task({
  id: "summarize-text",
  queue: openaiQueue,
  run: async (payload) => { /* calls OpenAI */ },
});

export const generateImage = task({
  id: "generate-image",
  queue: openaiQueue,
  run: async (payload) => { /* calls DALL-E */ },
});
```

**Per-tenant isolation with concurrencyKey:**

```typescript
const tenantQueue = queue({
  name: "per-tenant",
  concurrencyLimit: 3, // Each tenant gets max 3 concurrent runs
});

export const tenantTask = task({
  id: "tenant-work",
  queue: tenantQueue,
  run: async (payload: { tenantId: string }) => { /* ... */ },
});

// When triggering, pass the concurrencyKey
await tenantTask.trigger(
  { tenantId: "tenant_abc" },
  {
    queue: { name: "per-tenant", concurrencyKey: "tenant_abc" },
  }
);
```

**Related:** [Queues & Concurrency](./TRIGGER_DEV_QUEUES_CONCURRENCY.md) | [Management API](./TRIGGER_DEV_MANAGEMENT_API.md) | [Per-Tenant Pattern](./TRIGGER_DEV_PATTERNS.md#3-per-tenant-rate-limiting)

---

## 4. Should I use `wait.for` or `wait.forToken`?

**Key question:** Do you know *when* the task should resume, or are you waiting for an *external event*?

### Decision Flowchart

```
START: Your task needs to pause
  |
  v
Do you know the exact duration or date to wait?
  |
  |-- YES --> Is it a fixed duration (e.g., "wait 3 days")?
  |              |
  |              |-- YES --> Use wait.for({ days: 3 })
  |              |-- NO  --> Is it a specific date/time?
  |                            |
  |                            |-- YES --> Use wait.until({ date: targetDate })
  |
  |-- NO ----> Are you waiting for an external system or human action?
                 |
                 |-- YES --> Does the external system support webhooks?
                 |              |
                 |              |-- YES --> Use wait.createToken() + pass token.url
                 |              |            as the webhook callback (Pattern 6)
                 |              |
                 |              |-- NO  --> Use wait.createToken() + complete it
                 |                          from your own API/UI (Pattern 5)
                 |
                 |-- NO ----> You probably know the duration. Use wait.for().
```

### Key Differences

| Feature | `wait.for` / `wait.until` | `wait.forToken` |
| :--- | :--- | :--- |
| **Resume trigger** | Clock (duration or date) | External event (HTTP POST or SDK call) |
| **Use case** | Email sequences, delayed processing | Approvals, webhook callbacks, human-in-the-loop |
| **Cost during wait** | Zero -- task is suspended | Zero -- task is suspended |
| **Timeout** | N/A (always resumes at the specified time) | Configurable; returns `{ ok: false }` on timeout |
| **Callback URL** | N/A | Token has a `.url` for no-auth HTTP callbacks |

### Concrete Examples

**`wait.for` -- Known delay:**

```typescript
// Send an email, wait 3 days, send follow-up
await sendEmail(user, "Welcome!");
await wait.for({ days: 3 }); // Task suspends, costs nothing
await sendEmail(user, "How's it going?");
await wait.for({ days: 7 });
await sendEmail(user, "Check out our pro plan");
```

**`wait.until` -- Known date:**

```typescript
// Wait until a specific date (e.g., campaign launch)
await wait.until({ date: new Date("2025-07-01T09:00:00Z") });
await launchCampaign();
```

**`wait.forToken` -- External event:**

```typescript
// Create token, send callback URL to external service, wait for completion
const token = await wait.createToken({ timeout: "24h" });
await sendToExternalService({ callbackUrl: token.url });
const result = await wait.forToken<{ status: string }>(token);
if (result.ok) {
  console.log(result.output.status); // Data from the external service's POST body
}
```

**Related:** [Waits & Tokens](./TRIGGER_DEV_WAITS_TOKENS.md) | [Human-in-the-Loop Pattern](./TRIGGER_DEV_PATTERNS.md#5-human-in-the-loop) | [Webhook Callback Pattern](./TRIGGER_DEV_PATTERNS.md#6-webhook-callback)

---

## 5. Should I use batch trigger or a loop of individual triggers?

**Key question:** Are you triggering many instances of the same task with different payloads?

### Decision Flowchart

```
START: You need to trigger multiple task runs
  |
  v
Are all runs for the SAME task ID?
  |
  |-- YES --> How many items?
  |              |
  |              |-- <= 1,000 --> Do you need to wait for all results?
  |              |                  |
  |              |                  |-- YES --> batchTriggerAndWait() (from a task)
  |              |                  |-- NO  --> batchTrigger() (from anywhere)
  |              |
  |              |-- > 1,000 ----> Chunk into batches of 1,000 and loop
  |                                 with sequential batchTrigger() calls
  |
  |-- NO ----> Are they different task types you want to run in parallel?
                 |
                 |-- YES --> Use batch.triggerByTaskAndWait() to run
                 |            heterogeneous tasks in parallel
                 |
                 |-- NO  --> Use individual trigger() calls
```

### Key Differences

| Approach | Max Items | Wait for Results | Error Handling | Performance |
| :--- | :--- | :--- | :--- | :--- |
| Loop of `trigger()` | Unlimited | No (fire-and-forget) | Per-run retries only | N API calls |
| `batchTrigger()` | 1,000 | No | Per-run retries only | 1 API call |
| `batchTriggerAndWait()` | 1,000 | Yes (all results) | Filter by `run.ok` | 1 API call + wait |
| `batch.triggerByTaskAndWait()` | Mixed tasks | Yes (all results) | Filter by `run.ok` | 1 API call + wait |

### Concrete Examples

**`batchTrigger()` -- fire-and-forget, 1,000 items:**

```typescript
// From your backend (API route, server action, etc.)
import { tasks } from "@trigger.dev/sdk";

const handles = await tasks.batchTrigger<typeof processRow>("process-row",
  rows.map((row) => ({
    payload: { rowId: row.id, data: row },
  }))
);
// Returns immediately with an array of run handles
```

**`batchTriggerAndWait()` -- collect results from inside a parent task:**

```typescript
// Inside a parent task
const batch = await processRow.batchTriggerAndWait(
  rows.map((row) => ({
    payload: { rowId: row.id, data: row },
    options: {
      tags: [`import:${importId}`], // Tag for grouping
      idempotencyKey: `row-${row.id}`, // Prevent duplicate processing
    },
  }))
);

const results = batch.runs.filter((r) => r.ok).map((r) => r.output);
const errors = batch.runs.filter((r) => !r.ok);
```

**`batch.triggerByTaskAndWait()` -- heterogeneous parallel tasks:**

```typescript
// Run different task types in parallel and wait for all
import { batch } from "@trigger.dev/sdk";

const { runs: [basicInfo, industry, funding] } = await batch.triggerByTaskAndWait([
  { task: getBasicInfo, payload: { companyName: "Acme" } },
  { task: getIndustry, payload: { companyName: "Acme" } },
  { task: getFundingRound, payload: { companyName: "Acme" } },
]);
```

**Chunking for > 1,000 items:**

```typescript
// If you have more than 1,000 items, chunk them
const BATCH_SIZE = 1000;
for (let i = 0; i < allItems.length; i += BATCH_SIZE) {
  const chunk = allItems.slice(i, i + BATCH_SIZE);
  await processItem.batchTrigger(
    chunk.map((item) => ({ payload: item }))
  );
}
```

**Related:** [Management API](./TRIGGER_DEV_MANAGEMENT_API.md) — batch endpoints | [Fan-out Pattern](./TRIGGER_DEV_PATTERNS.md#1-fan-out-batch-trigger-n-subtasks)

---

## 6. Should I use schedules or an external cron?

**Key question:** Do you want Trigger.dev to manage the schedule lifecycle, or do you already have a cron system you prefer?

### Decision Flowchart

```
START: You need recurring background work
  |
  v
Do you already have a cron system (AWS EventBridge, Vercel Cron, GitHub Actions)?
  |
  |-- YES --> Do you want to consolidate into Trigger.dev?
  |              |
  |              |-- YES --> Use schedules.task() in Trigger.dev
  |              |-- NO  --> Keep external cron, have it call tasks.trigger()
  |
  |-- NO ----> Does the schedule need to be created/updated dynamically at runtime?
                 |
                 |-- YES --> Use the Schedules API (schedules.create/update/delete)
                 |            to manage schedules programmatically
                 |
                 |-- NO ----> Is the schedule known at development time?
                                |
                                |-- YES --> Use schedules.task() with inline cron
                                |            (simplest approach)
                                |
                                |-- NO  --> Use the Schedules API for dynamic creation
```

### Key Differences

| Feature | `schedules.task()` (Trigger.dev) | External Cron + `trigger()` |
| :--- | :--- | :--- |
| **Schedule management** | Defined in code or via API | Managed in your cron system |
| **Deduplication** | Built-in (one run per cron tick) | You must handle it yourself |
| **Timezone support** | Native (per-schedule timezone) | Depends on your cron system |
| **Dynamic schedules** | Yes, via Schedules API | Depends on your cron system |
| **Dashboard visibility** | Full schedule + run history | Run history only |
| **Payload** | Auto-populated with `timestamp`, `lastTimestamp`, etc. | You define the payload |
| **Testing** | Test from dashboard with "Now" option | Must trigger manually |
| **Cost** | Included in Trigger.dev pricing | Separate cron system cost |

### Concrete Examples

**`schedules.task()` -- inline cron definition (recommended for most cases):**

```typescript
import { schedules, logger } from "@trigger.dev/sdk";

export const dailyReport = schedules.task({
  id: "daily-report",
  cron: {
    pattern: "0 9 * * 1-5", // 9 AM weekdays
    timezone: "America/New_York",
  },
  run: async (payload) => {
    // payload.timestamp: when this run was scheduled
    // payload.lastTimestamp: when the previous run was scheduled
    logger.info("Generating daily report", {
      for: payload.timestamp.toISOString(),
    });
    const report = await generateReport();
    await emailReport(report);
    return { sent: true };
  },
});
```

**Schedules API -- dynamic schedule management (for user-configurable schedules):**

```typescript
import { schedules } from "@trigger.dev/sdk";

// Create a schedule for a specific user
const schedule = await schedules.create({
  task: "send-digest",
  cron: "0 8 * * *", // 8 AM daily
  timezone: "Europe/London",
  externalId: `user-digest-${userId}`, // Your own identifier
  deduplicationKey: `digest-${userId}`, // Prevents duplicate schedules
});

// Later: update the user's preferred time
await schedules.update(schedule.id, {
  cron: "0 18 * * *", // Changed to 6 PM
});

// User unsubscribes: deactivate or delete
await schedules.del(schedule.id);
```

**External cron triggering a Trigger.dev task:**

```typescript
// Vercel cron or AWS EventBridge calls this endpoint
// app/api/cron/daily-sync/route.ts
import type { dailySync } from "@/trigger/daily-sync";
import { tasks } from "@trigger.dev/sdk";

export async function GET(req: Request) {
  // Verify cron secret to prevent unauthorized calls
  if (req.headers.get("Authorization") !== `Bearer ${process.env.CRON_SECRET}`) {
    return Response.json({ error: "Unauthorized" }, { status: 401 });
  }

  const handle = await tasks.trigger<typeof dailySync>("daily-sync", {
    triggeredAt: new Date().toISOString(),
  });

  return Response.json({ runId: handle.id });
}
```

### Recommendation

For most teams, **`schedules.task()` is the best default**. It gives you:
- Deduplication out of the box (no duplicate runs if the schedule fires twice)
- Timezone-aware scheduling
- Full visibility in the Trigger.dev dashboard
- `lastTimestamp` in the payload for incremental syncs
- No external infrastructure to maintain

Use an **external cron** only if you already have one and want to keep your scheduling logic centralized there, or if you need sub-minute precision that Trigger.dev schedules do not support.

**Related:** [Management API](./TRIGGER_DEV_MANAGEMENT_API.md) — schedules CRUD | [Scheduled Data Sync Pattern](./TRIGGER_DEV_PATTERNS.md#7-scheduled-data-sync)

---

## Cross-Reference Index

| Decision | Primary Pattern | Reference Docs |
| :--- | :--- | :--- |
| Task vs inline code | All patterns | [Tasks](./TRIGGER_DEV_TASKS.md) |
| `trigger()` vs `triggerAndWait()` | [Sequential Pipeline](./TRIGGER_DEV_PATTERNS.md#2-sequential-pipeline-task-a-to-b-to-c) | [Tasks](./TRIGGER_DEV_TASKS.md) |
| Custom queue vs default | [Per-Tenant Rate Limiting](./TRIGGER_DEV_PATTERNS.md#3-per-tenant-rate-limiting) | [Queues & Concurrency](./TRIGGER_DEV_QUEUES_CONCURRENCY.md) |
| `wait.for` vs `wait.forToken` | [Human-in-the-Loop](./TRIGGER_DEV_PATTERNS.md#5-human-in-the-loop), [Webhook Callback](./TRIGGER_DEV_PATTERNS.md#6-webhook-callback) | [Waits & Tokens](./TRIGGER_DEV_WAITS_TOKENS.md) |
| Batch vs loop | [Fan-out](./TRIGGER_DEV_PATTERNS.md#1-fan-out-batch-trigger-n-subtasks) | [Management API](./TRIGGER_DEV_MANAGEMENT_API.md) |
| Schedules vs external cron | [Scheduled Data Sync](./TRIGGER_DEV_PATTERNS.md#7-scheduled-data-sync) | [Management API](./TRIGGER_DEV_MANAGEMENT_API.md) |
