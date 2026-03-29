# TRIGGER_DEV_PATTERNS

Production patterns cookbook for Trigger.dev workflows.

---

## 1) Fan-out / Fan-in

```ts
import { task } from "@trigger.dev/sdk";
import { processItem } from "./process-item";

export const processDataset = task({
  id: "process-dataset",
  run: async (payload: { itemIds: string[] }) => {
    const results = await processItem.batchTriggerAndWait(
      payload.itemIds.map((itemId) => ({ payload: { itemId } }))
    );

    const outputs = results.flatMap((r) => (r.ok ? [r.output] : []));
    const failures = results.flatMap((r) => (r.ok ? [] : [r.error]));
    return { outputs, failures };
  },
});
```

---

## 2) Sequential Pipeline (`A -> B -> C`)

```ts
const a = await taskA.triggerAndWait(input).unwrap();
const b = await taskB.triggerAndWait(a).unwrap();
const c = await taskC.triggerAndWait(b).unwrap();
return c;
```

> ⚠️ WARNING
> Do not parallelize wait calls with `Promise.all` inside a running task.

---

## 3) Per-tenant Rate Limiting

```ts
await syncTenant.trigger(
  { tenantId, cursor },
  {
    queue: "sync",
    concurrencyKey: `tenant:${tenantId}`,
    idempotencyKey: `sync:${tenantId}:${cursor}`,
  }
);
```

Combines shared queue + per-tenant partition key.

---

## 4) Idempotent Processing

```ts
await settleInvoice.trigger(
  { invoiceId },
  { idempotencyKey: `invoice:settle:${invoiceId}` }
);
```

Use deterministic keys from business identity/version.

---

## 5) Human-in-the-loop Approval

```ts
import { task, wait } from "@trigger.dev/sdk";

export const approveAndPublish = task({
  id: "approve-and-publish",
  run: async (payload: { draftId: string }) => {
    const token = await wait.createToken({
      idempotencyKey: `approve:${payload.draftId}`,
      timeout: "7d",
      tags: [`draft:${payload.draftId}`],
    });

    await notifyReviewer({ draftId: payload.draftId, callbackUrl: token.url });
    const result = await wait.forToken<{ approved: boolean; reason?: string }>(token.id);

    if (!result.ok || !result.output.approved) return { published: false, reason: result.ok ? result.output.reason : "timeout" };

    await publishDraft(payload.draftId);
    return { published: true };
  },
});
```

---

## 6) Webhook Callback Completion

External system POSTs to `token.url`:

```bash
curl -X POST "$TOKEN_URL" \
  -H "Content-Type: application/json" \
  -d '{"approved":true,"reviewer":"ops@example.com"}'
```

This unblocks `wait.forToken`.

---

## 7) Scheduled Data Sync

```ts
import { task, schedules } from "@trigger.dev/sdk";

export const hourlySync = task({
  id: "hourly-sync",
  run: async () => syncAllTenants(),
});

schedules.task({
  id: "hourly-sync-cron",
  task: hourlySync,
  cron: "0 * * * *",
  timezone: "UTC",
  deduplicationKey: "hourly-sync",
});
```

---

## 8) Long-running AI Agent with Realtime Progress

```ts
import { task, metadata, streams } from "@trigger.dev/sdk";

const tokens = streams.define<{ token: string }>({ id: "tokens" });

export const researchAgent = task({
  id: "research-agent",
  run: async (payload: { topic: string }) => {
    await metadata.set({ stage: "planning", progress: 10, topic: payload.topic });
    tokens.append({ token: "Planning..." });
    await metadata.set({ stage: "executing", progress: 60 });
    tokens.append({ token: "Executing..." });
    await metadata.set({ stage: "finalizing", progress: 95 });
    return { reportId: "rep_123" };
  },
});
```

Frontend subscribes with `useRealtimeRun` + `useRealtimeStream`.

---

## 9) Conditional Retry Strategy

```ts
import { task, AbortTaskRunError } from "@trigger.dev/sdk";

export const callApi = task({
  id: "call-api",
  retry: { maxAttempts: 6 },
  handleError: async ({ error }) => {
    const status = (error as any)?.status;
    if (status >= 400 && status < 500) return { retry: false };
    return { retry: true };
  },
  run: async () => {
    const res = await fetch("https://partner.example.com");
    if (res.status === 422) throw new AbortTaskRunError("Validation failure");
    if (!res.ok) throw new Error(`HTTP ${res.status}`);
    return res.json();
  },
});
```

---

## 10) M2M Triggering (Service A -> Service B)

```ts
// service-a backend
import { configure, tasks } from "@trigger.dev/sdk";

configure({ secretKey: process.env.SERVICE_B_TRIGGER_SECRET_KEY });

await tasks.trigger("service-b:sync-customer", {
  customerId: "cus_123",
}, {
  idempotencyKey: "sync-customer:cus_123:v1",
});
```

Security notes:

- Store remote service key in secrets manager.
- Scope by environment (staging vs prod keys).
- Use strict idempotency and payload validation.

Related:

- Task API: [`TRIGGER_DEV_TASKS.md`](./TRIGGER_DEV_TASKS.md)
- Waits: [`TRIGGER_DEV_WAITS_TOKENS.md`](./TRIGGER_DEV_WAITS_TOKENS.md)
- Realtime: [`TRIGGER_DEV_REALTIME.md`](./TRIGGER_DEV_REALTIME.md)

