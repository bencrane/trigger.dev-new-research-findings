# Trigger.dev Patterns Cookbook

> SDK version: **4.4.3** | Import from `@trigger.dev/sdk` (never `@trigger.dev/sdk/v3`)

This document provides production-ready TypeScript patterns for common Trigger.dev workflows. Each pattern includes a complete code example, key constraints, and links to related reference docs.

---

## Table of Contents

- [Trigger.dev Patterns Cookbook](#triggerdev-patterns-cookbook)
  - [Table of Contents](#table-of-contents)
  - [1. Fan-out: Batch Trigger N Subtasks](#1-fan-out-batch-trigger-n-subtasks)
  - [2. Sequential Pipeline: Task A to B to C](#2-sequential-pipeline-task-a-to-b-to-c)
  - [3. Per-Tenant Rate Limiting](#3-per-tenant-rate-limiting)
  - [4. Idempotent Processing](#4-idempotent-processing)
  - [5. Human-in-the-Loop](#5-human-in-the-loop)
  - [6. Webhook Callback](#6-webhook-callback)
  - [7. Scheduled Data Sync](#7-scheduled-data-sync)
  - [8. Long-Running AI Agent with Realtime Streaming](#8-long-running-ai-agent-with-realtime-streaming)
  - [9. Retry with Conditional Logic](#9-retry-with-conditional-logic)
  - [10. M2M Triggering: Cross-Service Task Invocation](#10-m2m-triggering-cross-service-task-invocation)
  - [Quick Reference: Key Limits and Formats](#quick-reference-key-limits-and-formats)

---

## 1. Fan-out: Batch Trigger N Subtasks

Use `batchTriggerAndWait` to fan work out to child tasks and collect all results before continuing. This is ideal for parallel processing of independent items (image processing, API calls, data enrichment).

**Constraints:**
- Batch trigger limit: **1,000 items** per call
- Child tasks are version-locked to the parent's deploy version when using `batchTriggerAndWait`
- Never wrap `batchTriggerAndWait` in `Promise.all` -- this is not supported

```typescript
// trigger/fan-out.ts
import { task, logger } from "@trigger.dev/sdk";

// --- Child task: processes a single item ---
export const processItem = task({
  id: "process-item",
  retry: { maxAttempts: 3, factor: 2, minTimeoutInMs: 1000 },
  run: async (payload: { itemId: string; data: string }) => {
    logger.info("Processing item", { itemId: payload.itemId });
    // Simulate work (API call, transformation, etc.)
    const result = await transformData(payload.data);
    return { itemId: payload.itemId, result };
  },
});

// --- Parent task: fans out to N children, collects results ---
export const fanOutOrchestrator = task({
  id: "fan-out-orchestrator",
  run: async (payload: { items: Array<{ itemId: string; data: string }> }) => {
    logger.info(`Fanning out to ${payload.items.length} subtasks`);

    // batchTriggerAndWait sends up to 1,000 items and waits for all to complete
    const batchResult = await processItem.batchTriggerAndWait(
      payload.items.map((item) => ({
        payload: item,
      }))
    );

    // Filter successful runs and extract outputs
    const successes = batchResult.runs
      .filter((run) => run.ok)
      .map((run) => run.output);

    const failures = batchResult.runs.filter((run) => !run.ok);

    logger.info("Fan-out complete", {
      total: payload.items.length,
      succeeded: successes.length,
      failed: failures.length,
    });

    return { successes, failedCount: failures.length };
  },
});
```

**When to use:** Processing lists of items in parallel (CSV rows, image batches, multi-model LLM evaluations).

**Related:** [Management API](./TRIGGER_DEV_MANAGEMENT_API.md) | [Tasks](./TRIGGER_DEV_TASKS.md)

---

## 2. Sequential Pipeline: Task A to B to C

Use `triggerAndWait` to chain tasks sequentially. Each task has independent retry settings and appears as a separate run in the dashboard for debugging.

**Constraints:**
- `triggerAndWait` returns a `Result` object, not the output directly
- Use `.unwrap()` to get output directly (throws on failure), or check `result.ok`
- Child tasks are version-locked to the parent version

```typescript
// trigger/pipeline.ts
import { task, logger } from "@trigger.dev/sdk";

// Step 1: Fetch raw data from an external API
export const fetchData = task({
  id: "pipeline-fetch-data",
  retry: { maxAttempts: 5, factor: 2, minTimeoutInMs: 2000 },
  run: async (payload: { sourceUrl: string }) => {
    const response = await fetch(payload.sourceUrl);
    if (!response.ok) throw new Error(`Fetch failed: ${response.status}`);
    const rawData = await response.json();
    logger.info("Fetched raw data", { recordCount: rawData.length });
    return { records: rawData as Record<string, unknown>[] };
  },
});

// Step 2: Transform / clean the data
export const transformData = task({
  id: "pipeline-transform-data",
  retry: { maxAttempts: 3 },
  run: async (payload: { records: Record<string, unknown>[] }) => {
    const cleaned = payload.records
      .filter((r) => r.email != null)
      .map((r) => ({
        email: String(r.email).toLowerCase().trim(),
        name: String(r.name ?? ""),
      }));
    logger.info("Transformed data", { cleanedCount: cleaned.length });
    return { cleaned };
  },
});

// Step 3: Load into destination (database, API, etc.)
export const loadData = task({
  id: "pipeline-load-data",
  retry: { maxAttempts: 3 },
  run: async (payload: { cleaned: Array<{ email: string; name: string }> }) => {
    // Insert into database or call destination API
    for (const record of payload.cleaned) {
      await db.upsertContact(record);
    }
    logger.info("Loaded data", { count: payload.cleaned.length });
    return { loadedCount: payload.cleaned.length };
  },
});

// Orchestrator: A -> B -> C
export const etlPipeline = task({
  id: "etl-pipeline",
  run: async (payload: { sourceUrl: string }) => {
    // Step A: Fetch
    const fetchResult = await fetchData.triggerAndWait({ sourceUrl: payload.sourceUrl });
    if (!fetchResult.ok) {
      throw new Error(`Fetch step failed: ${fetchResult.error}`);
    }

    // Step B: Transform
    const transformResult = await transformData.triggerAndWait({
      records: fetchResult.output.records,
    });
    if (!transformResult.ok) {
      throw new Error(`Transform step failed: ${transformResult.error}`);
    }

    // Step C: Load -- using .unwrap() for brevity (throws on failure)
    const loadOutput = await loadData
      .triggerAndWait({ cleaned: transformResult.output.cleaned })
      .unwrap();

    return {
      status: "complete",
      loadedCount: loadOutput.loadedCount,
    };
  },
});
```

**When to use:** ETL pipelines, multi-step workflows where each step depends on the previous one and needs independent retry/observability.

**Related:** [Tasks](./TRIGGER_DEV_TASKS.md)

---

## 3. Per-Tenant Rate Limiting

Use custom queues with concurrency keys to isolate work per customer. Each tenant gets its own concurrency slot, preventing noisy-neighbor problems.

**Constraints:**
- Queue concurrency limits apply per unique `concurrencyKey` value
- Queue names must be unique within a project
- Concurrency can be overridden at runtime via the Queues API

```typescript
// trigger/per-tenant.ts
import { task, queue, logger } from "@trigger.dev/sdk";

// Define a shared queue with per-tenant concurrency
const perTenantQueue = queue({
  name: "per-tenant-processing",
  concurrencyLimit: 2, // Each tenant gets up to 2 concurrent runs
});

export const processForTenant = task({
  id: "process-for-tenant",
  queue: perTenantQueue,
  run: async (payload: { tenantId: string; data: unknown }) => {
    logger.info("Processing for tenant", { tenantId: payload.tenantId });
    // Do tenant-specific work (API calls, DB writes, etc.)
    await performTenantWork(payload.tenantId, payload.data);
    return { success: true };
  },
});

// --- Triggering from your backend ---
// Each trigger passes the tenantId as the concurrencyKey.
// Runs with the same concurrencyKey share the queue's concurrency limit.

// In your backend (e.g., Next.js API route):
import type { processForTenant } from "./trigger/per-tenant";
import { tasks } from "@trigger.dev/sdk";

async function handleRequest(tenantId: string, data: unknown) {
  await tasks.trigger<typeof processForTenant>("process-for-tenant", {
    tenantId,
    data,
  }, {
    queue: {
      name: "per-tenant-processing",
      concurrencyKey: tenantId, // Isolates concurrency per tenant
    },
  });
}
```

**Advanced: Dynamic concurrency override**

You can dynamically adjust a queue's concurrency limit at runtime using the Queues API, useful for temporarily throttling a specific tenant or bursting capacity:

```typescript
import { queues } from "@trigger.dev/sdk";

// Temporarily increase concurrency for a high-priority tenant
await queues.overrideConcurrencyLimit("per-tenant-processing", 10);

// Reset back to the base limit defined in code
await queues.resetConcurrencyLimit("per-tenant-processing");
```

**When to use:** Multi-tenant SaaS platforms, per-user job isolation, global rate limiting against external APIs.

**Related:** [Queues & Concurrency](./TRIGGER_DEV_QUEUES_CONCURRENCY.md) | [Management API](./TRIGGER_DEV_MANAGEMENT_API.md)

---

## 4. Idempotent Processing

Use `idempotencyKey` to ensure exactly-once semantics. If you trigger a task with the same idempotency key twice, the second call returns the original run handle instead of creating a new run.

**Constraints:**
- Idempotency keys are scoped to the task and environment
- Keys are valid for 24 hours by default; you can set a custom TTL with `idempotencyKeyTTL`
- TTL format: `"1h"`, `"1m"`, `"1h42m"`, or a number of seconds

```typescript
// trigger/idempotent.ts
import { task, logger } from "@trigger.dev/sdk";

export const processPayment = task({
  id: "process-payment",
  retry: { maxAttempts: 5 },
  run: async (payload: { orderId: string; amount: number; currency: string }) => {
    logger.info("Processing payment", { orderId: payload.orderId });

    // Call payment provider -- safe to retry because
    // the task-level idempotency prevents duplicate runs
    const charge = await paymentProvider.createCharge({
      amount: payload.amount,
      currency: payload.currency,
      metadata: { orderId: payload.orderId },
    });

    await db.updateOrder(payload.orderId, {
      status: "paid",
      chargeId: charge.id,
    });

    return { chargeId: charge.id, orderId: payload.orderId };
  },
});

// --- Triggering with idempotency from your backend ---
import type { processPayment } from "./trigger/idempotent";
import { tasks } from "@trigger.dev/sdk";

async function handleOrderPayment(orderId: string, amount: number) {
  // Using the orderId as the idempotency key ensures this payment
  // is only processed once, even if the endpoint is called multiple times.
  const handle = await tasks.trigger<typeof processPayment>(
    "process-payment",
    { orderId, amount, currency: "usd" },
    {
      idempotencyKey: `payment-${orderId}`,
      idempotencyKeyTTL: "24h", // Key expires after 24 hours
    }
  );

  return handle;
}
```

**When to use:** Payment processing, webhook handlers that may receive duplicates, any operation that must not run more than once.

**Related:** [Tasks](./TRIGGER_DEV_TASKS.md)

---

## 5. Human-in-the-Loop

Use wait tokens to pause a task until a human takes action. Create a waitpoint token, surface it in your UI, and complete it when the human approves/rejects.

**Constraints:**
- Tokens have a `.url` property for HTTP callback (no API key needed)
- Tokens support up to 10 tags, each under 128 characters
- Token timeout format: `"1h"`, `"30m"`, `"1d"`, etc.
- When a token times out, `wait.forToken()` returns `{ ok: false }`

```typescript
// trigger/approval-workflow.ts
import { task, wait, logger, metadata } from "@trigger.dev/sdk";

export const contentApprovalWorkflow = task({
  id: "content-approval-workflow",
  run: async (payload: { contentId: string; authorId: string; draft: string }) => {
    // Step 1: Generate AI-enhanced version
    logger.info("Generating enhanced content", { contentId: payload.contentId });
    const enhanced = await generateEnhancedContent(payload.draft);

    // Step 2: Create a waitpoint token for human review
    const reviewToken = await wait.createToken({
      timeout: "4h", // Reviewer has 4 hours
      tags: [`content:${payload.contentId}`, `author:${payload.authorId}`],
      idempotencyKey: `review-${payload.contentId}`,
    });

    // Step 3: Update metadata so the frontend knows the token ID
    metadata.set("reviewTokenId", reviewToken.id);
    metadata.set("status", "awaiting-review");
    metadata.set("enhanced", enhanced);

    // Step 4: Notify the reviewer (email, Slack, in-app notification, etc.)
    await notifyReviewer({
      contentId: payload.contentId,
      reviewUrl: `https://app.example.com/review/${payload.contentId}?token=${reviewToken.id}`,
    });

    // Step 5: Pause execution until the reviewer acts or timeout
    const result = await wait.forToken<{
      approved: boolean;
      feedback?: string;
    }>(reviewToken);

    // Step 6: Handle the review outcome
    if (!result.ok) {
      logger.warn("Review timed out", { contentId: payload.contentId });
      metadata.set("status", "timed-out");
      return { status: "timed-out", contentId: payload.contentId };
    }

    if (result.output.approved) {
      await publishContent(payload.contentId, enhanced);
      metadata.set("status", "published");
      return { status: "published", contentId: payload.contentId };
    } else {
      metadata.set("status", "rejected");
      metadata.set("feedback", result.output.feedback);
      return {
        status: "rejected",
        contentId: payload.contentId,
        feedback: result.output.feedback,
      };
    }
  },
});
```

**Completing the token from your frontend/API:**

```typescript
// app/api/review/route.ts (Next.js)
import { wait } from "@trigger.dev/sdk";

export async function POST(req: Request) {
  const { tokenId, approved, feedback } = await req.json();

  await wait.completeToken(
    { id: tokenId },
    { approved, feedback }
  );

  return Response.json({ success: true });
}
```

**When to use:** Content approval, compliance review, manual QA gates, any workflow that requires a human decision.

**Related:** [Waits & Tokens](./TRIGGER_DEV_WAITS_TOKENS.md)

---

## 6. Webhook Callback

Use `wait.createToken()` and pass the token's `.url` to an external service. When that service sends an HTTP POST to the URL, the task resumes. No API key is needed for the callback URL.

**Constraints:**
- The callback URL is pre-signed with an HMAC hash -- do not construct it manually
- The entire POST body is passed as the output data to the waiting task
- If the token is already completed, a repeat POST is a no-op

```typescript
// trigger/webhook-callback.ts
import { task, wait, logger } from "@trigger.dev/sdk";

export const orderFulfillment = task({
  id: "order-fulfillment",
  run: async (payload: { orderId: string; items: string[] }) => {
    // Step 1: Create a waitpoint token to receive the shipping callback
    const shippingToken = await wait.createToken({
      timeout: "7d", // Wait up to 7 days for shipping confirmation
      tags: [`order:${payload.orderId}`],
      idempotencyKey: `shipping-callback-${payload.orderId}`,
    });

    // Step 2: Send order to fulfillment provider, passing the callback URL
    // The fulfillment provider will POST to this URL when the order ships
    await fulfillmentProvider.createShipment({
      orderId: payload.orderId,
      items: payload.items,
      webhookUrl: shippingToken.url, // Pre-signed URL, no API key needed
    });

    logger.info("Shipment created, waiting for callback", {
      orderId: payload.orderId,
      callbackUrl: shippingToken.url,
    });

    // Step 3: Task pauses here until the webhook fires or timeout
    const result = await wait.forToken<{
      trackingNumber: string;
      carrier: string;
      estimatedDelivery: string;
    }>(shippingToken);

    if (!result.ok) {
      logger.error("Shipping callback timed out", { orderId: payload.orderId });
      return { status: "timeout", orderId: payload.orderId };
    }

    // Step 4: Process the shipping confirmation
    const { trackingNumber, carrier, estimatedDelivery } = result.output;

    await db.updateOrder(payload.orderId, {
      status: "shipped",
      trackingNumber,
      carrier,
      estimatedDelivery,
    });

    await sendShippingNotification({
      orderId: payload.orderId,
      trackingNumber,
      carrier,
    });

    return {
      status: "shipped",
      orderId: payload.orderId,
      trackingNumber,
      carrier,
    };
  },
});
```

**What the external service sends:**

```bash
# The fulfillment provider POSTs to the token URL when the order ships.
# The entire JSON body becomes the output of wait.forToken().
curl -X POST "$CALLBACK_URL" \
  -H "Content-Type: application/json" \
  -d '{"trackingNumber": "1Z999AA10123456784", "carrier": "UPS", "estimatedDelivery": "2025-04-05"}'
```

**When to use:** Third-party webhook integration (payment providers, shipping, document signing), any scenario where an external system needs to signal completion.

**Related:** [Waits & Tokens](./TRIGGER_DEV_WAITS_TOKENS.md)

---

## 7. Scheduled Data Sync

Use `schedules.task()` to define cron-based tasks that run on a recurring schedule. Trigger.dev manages deduplication and timezone handling.

**Constraints:**
- Cron pattern follows standard 5-field format (`minute hour day-of-month month day-of-week`)
- Timezone is optional; defaults to UTC
- Schedule tasks receive a payload with `timestamp`, `lastTimestamp`, `externalId`, etc.
- Schedules are automatically deduped -- each cron tick produces exactly one run

```typescript
// trigger/daily-sync.ts
import { schedules, logger } from "@trigger.dev/sdk";

// Runs every day at 2:00 AM Eastern Time
export const dailyDataSync = schedules.task({
  id: "daily-data-sync",
  cron: {
    pattern: "0 2 * * *",
    timezone: "America/New_York",
  },
  retry: { maxAttempts: 3 },
  run: async (payload) => {
    // payload.timestamp is the scheduled time for this run
    // payload.lastTimestamp is the previous run's scheduled time (if any)
    const syncSince = payload.lastTimestamp ?? new Date(Date.now() - 86400000);

    logger.info("Starting daily sync", {
      syncSince: syncSince.toISOString(),
      scheduledFor: payload.timestamp.toISOString(),
    });

    // Fetch records modified since last sync
    const records = await externalApi.getModifiedRecords({
      since: syncSince,
    });

    logger.info(`Found ${records.length} records to sync`);

    // Process each record
    let synced = 0;
    for (const record of records) {
      await db.upsertRecord(record);
      synced++;
    }

    logger.info("Sync complete", { synced });
    return { synced, syncedAt: new Date().toISOString() };
  },
});

// Runs every hour at minute 0
export const hourlyHealthCheck = schedules.task({
  id: "hourly-health-check",
  cron: "0 * * * *", // Simple string format (UTC)
  run: async (payload) => {
    const services = ["api", "database", "cache", "search"];
    const results: Record<string, string> = {};

    for (const service of services) {
      try {
        await checkServiceHealth(service);
        results[service] = "healthy";
      } catch (error) {
        results[service] = "unhealthy";
      }
    }

    logger.info("Health check complete", results);
    return results;
  },
});
```

**When to use:** Daily/hourly data syncs, report generation, cleanup jobs, health checks, any recurring background work.

**Related:** [Management API](./TRIGGER_DEV_MANAGEMENT_API.md) — schedules CRUD

---

## 8. Long-Running AI Agent with Realtime Streaming

Use `metadata` to push progress updates to the frontend in real time. Combine with `streams` to forward LLM token streams directly to the client.

**Constraints:**
- Tags: max 10 per run, each under 128 characters
- Machine types: `micro`, `small-1x`, `small-2x`, `medium-1x`, `medium-2x`, `large-1x`, `large-2x`
- Use `metadata.set()` / `metadata.parent.set()` for real-time progress
- Use `metadata.stream()` to forward readable streams to the frontend

```typescript
// trigger/ai-agent.ts
import { task, metadata, logger } from "@trigger.dev/sdk";
import { openai } from "@ai-sdk/openai";
import { generateText, streamText } from "ai";

export const deepResearchAgent = task({
  id: "deep-research-agent",
  machine: "medium-1x", // More memory for large context windows
  maxDuration: 600, // 10 minutes max
  retry: { maxAttempts: 2 },
  run: async (payload: { query: string; userId: string }) => {
    // Phase 1: Generate search queries
    metadata.set("status", { progress: 10, label: "Generating search queries..." });

    const queriesResult = await generateText({
      model: openai("gpt-4o"),
      messages: [
        {
          role: "system",
          content: "Generate 3 search queries to research the given topic.",
        },
        { role: "user", content: payload.query },
      ],
    });

    const queries = queriesResult.text.split("\n").filter(Boolean);

    // Phase 2: Search and analyze (update progress as we go)
    const findings: string[] = [];
    for (let i = 0; i < queries.length; i++) {
      metadata.set("status", {
        progress: 10 + ((i + 1) / queries.length) * 50,
        label: `Researching: "${queries[i]}"`,
      });

      const searchResults = await searchWeb(queries[i]);
      findings.push(...searchResults);
    }

    // Phase 3: Generate final report with streaming
    metadata.set("status", { progress: 70, label: "Writing report..." });

    // Stream the final report to the frontend in real time
    const reportStream = streamText({
      model: openai("gpt-4o"),
      messages: [
        {
          role: "system",
          content: "Write a comprehensive research report based on the findings.",
        },
        {
          role: "user",
          content: `Query: ${payload.query}\n\nFindings:\n${findings.join("\n\n")}`,
        },
      ],
    });

    // Forward the LLM stream to the frontend via Realtime
    const stream = await metadata.stream("report", reportStream.textStream);

    // Collect the full text for return value
    let fullReport = "";
    for await (const chunk of stream) {
      fullReport += chunk;
    }

    metadata.set("status", { progress: 100, label: "Complete" });

    return {
      query: payload.query,
      report: fullReport,
      sourcesCount: findings.length,
    };
  },
});
```

**Frontend (React) -- subscribing to the run:**

```typescript
// components/ResearchAgent.tsx
import { useRealtimeTaskTrigger } from "@trigger.dev/react-hooks";
import type { deepResearchAgent } from "@/trigger/ai-agent";

export function ResearchAgent({ accessToken }: { accessToken: string }) {
  const { submit, run } = useRealtimeTaskTrigger<typeof deepResearchAgent>(
    "deep-research-agent",
    { accessToken }
  );

  const status = run?.metadata?.status as
    | { progress: number; label: string }
    | undefined;

  return (
    <div>
      <button onClick={() => submit({ query: "AI safety in 2025", userId: "123" })}>
        Start Research
      </button>
      {status && (
        <div>
          <progress value={status.progress} max={100} />
          <p>{status.label}</p>
        </div>
      )}
    </div>
  );
}
```

**When to use:** AI agents, deep research, document generation, any long-running task where the user needs live progress or streaming output.

**Related:** [Management API](./TRIGGER_DEV_MANAGEMENT_API.md) — runs endpoints | [Realtime](./TRIGGER_DEV_REALTIME.md)

---

## 9. Retry with Conditional Logic

Use `catchError` to implement conditional retry logic based on error type. Skip retries on permanent failures (4xx), retry on transient failures (5xx), and customize retry timing.

**Constraints:**
- `catchError` runs on every uncaught error before the retry decision
- Return `{ skipRetrying: true }` to abort immediately
- Return `{ retryAt: Date }` to retry at a specific time
- Return `undefined` to use the task's default retry settings
- `AbortTaskRunError` prevents retries when thrown inside `run()`

```typescript
// trigger/conditional-retry.ts
import { task, logger, retry, AbortTaskRunError } from "@trigger.dev/sdk";

export const callExternalApi = task({
  id: "call-external-api",
  retry: {
    maxAttempts: 8,
    factor: 2,
    minTimeoutInMs: 1000,
    maxTimeoutInMs: 60_000,
    randomize: true,
  },
  run: async (payload: { endpoint: string; body: unknown }) => {
    const response = await fetch(payload.endpoint, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(payload.body),
    });

    if (!response.ok) {
      // Throw with status info so catchError can inspect it
      const error = new Error(`API error: ${response.status}`);
      (error as any).status = response.status;
      (error as any).headers = Object.fromEntries(response.headers.entries());
      throw error;
    }

    return await response.json();
  },

  catchError: async ({ error, ctx, retryAt }) => {
    const status = (error as any)?.status;
    const headers = (error as any)?.headers;

    // 400 Bad Request -- our payload is wrong, retrying won't help
    if (status === 400) {
      logger.error("Bad request, skipping retries", { error });
      return { skipRetrying: true };
    }

    // 401/403 -- auth issue, retrying won't help
    if (status === 401 || status === 403) {
      logger.error("Auth error, skipping retries", { status });
      return { skipRetrying: true };
    }

    // 404 -- resource not found, permanent failure
    if (status === 404) {
      return { skipRetrying: true };
    }

    // 429 Rate Limited -- respect Retry-After header
    if (status === 429) {
      const retryAfter = headers?.["retry-after"];
      if (retryAfter) {
        const retryDate = new Date(Date.now() + parseInt(retryAfter) * 1000);
        logger.warn("Rate limited, retrying at", { retryDate });
        return { retryAt: retryDate };
      }
    }

    // 5xx server errors -- use default retry backoff
    if (status >= 500) {
      logger.warn("Server error, will retry with backoff", { status });
      return undefined; // Use default retry settings
    }

    // Unknown errors -- use default retry behavior
    return undefined;
  },
});

// --- Alternative: Using retry.onThrow for scoped retries ---
export const resilientMultiStepTask = task({
  id: "resilient-multi-step",
  run: async (payload: { userId: string }) => {
    // Step 1: Retry this specific block up to 3 times
    const userData = await retry.onThrow(
      async ({ attempt }) => {
        logger.info("Fetching user data", { attempt });
        return await userService.getUser(payload.userId);
      },
      { maxAttempts: 3, factor: 2, minTimeoutInMs: 500 }
    );

    // Step 2: Validate -- permanent failure if invalid
    if (!userData.email) {
      throw new AbortTaskRunError("User has no email, cannot continue");
    }

    // Step 3: Retry HTTP fetch with built-in status-based retrying
    const enriched = await retry.fetch(
      `https://api.enrichment.com/v1/users?email=${userData.email}`,
      {
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
      }
    );

    return await enriched.json();
  },
});
```

**When to use:** Calling external APIs with varying error semantics, payment processors, rate-limited APIs, any task where different errors require different retry strategies.

**Related:** [Management API](./TRIGGER_DEV_MANAGEMENT_API.md) — `runs.replay()`

---

## 10. M2M Triggering: Cross-Service Task Invocation

One backend service can trigger tasks in another service's Trigger.dev project using the REST API or the SDK with a different project's secret key. This enables microservice architectures where services communicate through Trigger.dev tasks.

**Constraints:**
- The calling service needs the target project's `TRIGGER_SECRET_KEY`
- Use `tasks.trigger()` with explicit task IDs (string-based, not imports)
- Use `type` imports only for type safety -- the actual task code lives in the other project
- Tags are useful for correlating runs across services (max 10, each under 128 chars)
- TTL format: `"1h"`, `"1m"`, `"1h42m"`, or number of seconds

```typescript
// Service A: triggers a task in Service B's Trigger.dev project
// service-a/src/trigger-service-b.ts
import { configure, tasks } from "@trigger.dev/sdk";

// Configure the SDK to point at Service B's project
configure({
  secretKey: process.env.SERVICE_B_TRIGGER_SECRET_KEY!,
});

// Type-only import for payload type safety (optional)
// This type mirrors what Service B's task expects
interface GenerateInvoicePayload {
  orderId: string;
  customerId: string;
  lineItems: Array<{ description: string; amount: number }>;
}

interface GenerateInvoiceOutput {
  invoiceId: string;
  pdfUrl: string;
}

export async function requestInvoiceGeneration(orderId: string, customerId: string) {
  // Fire-and-forget trigger to Service B's task
  const handle = await tasks.trigger<{
    payload: GenerateInvoicePayload;
    output: GenerateInvoiceOutput;
  }>("generate-invoice", {
    orderId,
    customerId,
    lineItems: [
      { description: "Pro Plan - Monthly", amount: 4900 },
      { description: "Extra seats (3)", amount: 1500 },
    ],
  }, {
    tags: [`order:${orderId}`, `customer:${customerId}`],
    ttl: "1h", // Run must start within 1 hour or it's discarded
  });

  return handle; // { id: "run_abc123" }
}
```

**Service B: the task being triggered remotely**

```typescript
// service-b/trigger/generate-invoice.ts
import { task, logger } from "@trigger.dev/sdk";

export const generateInvoice = task({
  id: "generate-invoice",
  retry: { maxAttempts: 3 },
  run: async (payload: {
    orderId: string;
    customerId: string;
    lineItems: Array<{ description: string; amount: number }>;
  }) => {
    logger.info("Generating invoice", { orderId: payload.orderId });

    const pdf = await invoiceService.generate({
      customerId: payload.customerId,
      items: payload.lineItems,
    });

    const pdfUrl = await storage.upload(`invoices/${payload.orderId}.pdf`, pdf);

    return { invoiceId: `inv_${payload.orderId}`, pdfUrl };
  },
});
```

**Alternatively, use the REST API directly (any language):**

```bash
curl -X POST "https://api.trigger.dev/api/v1/tasks/generate-invoice/trigger" \
  -H "Authorization: Bearer $SERVICE_B_TRIGGER_SECRET_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "payload": {
      "orderId": "ord_123",
      "customerId": "cust_456",
      "lineItems": [{"description": "Pro Plan", "amount": 4900}]
    },
    "options": {
      "tags": ["order:ord_123"],
      "ttl": "1h"
    }
  }'
```

**When to use:** Microservice architectures, cross-project communication, triggering background work from services written in other languages, event-driven integrations.

**Related:** [Tasks](./TRIGGER_DEV_TASKS.md) | [Management API](./TRIGGER_DEV_MANAGEMENT_API.md) | [Master Reference](./TRIGGER_DEV_MASTER.md)

---

## Quick Reference: Key Limits and Formats

| Parameter | Limit / Format |
| :--- | :--- |
| Batch trigger | Max 1,000 items per call |
| Tags per run | Max 10 tags, each under 128 characters |
| TTL format | `"1h"`, `"1m"`, `"1h42m"`, or number of seconds |
| Machine types | `micro`, `small-1x`, `small-2x`, `medium-1x`, `medium-2x`, `large-1x`, `large-2x` |
| Wait token `.url` | Pre-signed HTTP callback URL, no API key needed |
| Idempotency key TTL | `"30s"`, `"1m"`, `"2h"`, `"3d"`, `"4w"` |
| SDK import | Always `@trigger.dev/sdk` (never `@trigger.dev/sdk/v3`) |
| Config import | `@trigger.dev/sdk` for `defineConfig` in `trigger.config.ts` |
| `triggerAndWait` result | Returns `Result` object -- check `.ok` then `.output`, or use `.unwrap()` |
| `Promise.all` | Never wrap `triggerAndWait` / `batchTriggerAndWait` in `Promise.all` |
