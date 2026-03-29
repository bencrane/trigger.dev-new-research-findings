# Trigger.dev Waits and Tokens

> Canonical reference for duration-based waits, date-based waits, waitpoint tokens (external event / approval workflows), token lifecycle, listing tokens, checkpointing during waits, idempotency, and input streams.
>
> **SDK version**: `@trigger.dev/sdk` 4.4.3

---

## Table of Contents

- [Trigger.dev Waits and Tokens](#triggerdev-waits-and-tokens)
  - [Table of Contents](#table-of-contents)
  - [Duration-Based Waits — wait.for()](#duration-based-waits--waitfor)
    - [wait.for() options](#waitfor-options)
  - [Date-Based Waits — wait.until()](#date-based-waits--waituntil)
    - [wait.until() options](#waituntil-options)
  - [Waitpoint Tokens Overview](#waitpoint-tokens-overview)
  - [Creating Tokens — wait.createToken()](#creating-tokens--waitcreatetoken)
    - [CreateToken options](#createtoken-options)
    - [CreateToken response](#createtoken-response)
  - [Waiting for Tokens — wait.forToken()](#waiting-for-tokens--waitfortoken)
    - [wait.forToken() return value](#waitfortoken-return-value)
  - [Completing Tokens — wait.completeToken()](#completing-tokens--waitcompletetoken)
    - [Authentication](#authentication)
    - [CompleteToken request body](#completetoken-request-body)
    - [CompleteToken response](#completetoken-response)
  - [HTTP Callback Completion](#http-callback-completion)
    - [How it works](#how-it-works)
    - [Callback-specific responses](#callback-specific-responses)
  - [Listing Tokens — wait.listTokens()](#listing-tokens--waitlisttokens)
    - [ListTokens filter options](#listtokens-filter-options)
    - [Pagination (REST)](#pagination-rest)
    - [ListTokens response (REST)](#listtokens-response-rest)
  - [Retrieving Tokens — wait.retrieveToken()](#retrieving-tokens--waitretrievetoken)
  - [WaitpointTokenObject Schema](#waitpointtokenobject-schema)
  - [Checkpointing During Waits](#checkpointing-during-waits)
    - [Idempotency keys on waits](#idempotency-keys-on-waits)
  - [Input Streams](#input-streams)
  - [Complete Example: Human-in-the-Loop Approval](#complete-example-human-in-the-loop-approval)
  - [Related Documentation](#related-documentation)

---

## Duration-Based Waits — wait.for()

Pause a run for a specified duration. The run is checkpointed and no compute costs accrue during the wait.

```ts
import { task, wait } from "@trigger.dev/sdk";

export const drip = task({
  id: "drip-campaign",
  run: async (payload: { email: string }) => {
    await sendWelcomeEmail(payload.email);

    // Pause for 3 days — run is checkpointed, slot released
    await wait.for({ days: 3 });

    await sendFollowUpEmail(payload.email);

    await wait.for({ hours: 12 });

    await sendFinalEmail(payload.email);
  },
});
```

### wait.for() options

| Option | Type | Description |
|---|---|---|
| `seconds` | `number` | Wait for N seconds |
| `minutes` | `number` | Wait for N minutes |
| `hours` | `number` | Wait for N hours |
| `days` | `number` | Wait for N days |

You can pass one or combine multiple (they are additive):

```ts
await wait.for({ hours: 1, minutes: 30 }); // Wait 1h 30m
```

<!-- UNVERIFIED -->
Action item: verify whether combining multiple duration fields in `wait.for()` is additive in `https://trigger.dev/docs/wait`.

---

## Date-Based Waits — wait.until()

Pause a run until a specific date/time.

```ts
import { task, wait } from "@trigger.dev/sdk";

export const scheduledReport = task({
  id: "scheduled-report",
  run: async (payload: { reportDate: string }) => {
    // Wait until a specific date
    await wait.until({ date: new Date(payload.reportDate) });

    await generateAndSendReport();
  },
});
```

### wait.until() options

| Option | Type | Description |
|---|---|---|
| `date` | `Date` | The date/time to wait until |

If the date is in the past, the wait resolves immediately.

---

## Waitpoint Tokens Overview

Waitpoint tokens enable **external event** and **human-in-the-loop** workflows. The flow is:

1. **Create a token** — get back an ID and a callback URL.
2. **Wait for the token** — the run pauses (checkpoints) until the token is completed.
3. **Complete the token** — from your backend, frontend, or an external webhook.
4. **Run resumes** — with the data passed during completion.

```
┌─────────────┐      ┌──────────────┐      ┌─────────────────┐
│ Task run     │      │ External     │      │ Task run        │
│ creates token│─────▶│ service/user │─────▶│ resumes with    │
│ & waits      │      │ completes it │      │ completion data │
└─────────────┘      └──────────────┘      └─────────────────┘
```

---

## Creating Tokens — wait.createToken()

Create a waitpoint token from within a task or from your backend.

```ts
import { wait } from "@trigger.dev/sdk";

const token = await wait.createToken({
  timeout: "1h",
  tags: ["user:1234567", "org:9876543"],
  idempotencyKey: "approval-user-1234567",
  idempotencyKeyTTL: "24h",
});

console.log(token.id);       // "waitpoint_abc123"
console.log(token.isCached); // false (true if idempotency key matched an existing token)
console.log(token.url);      // "https://api.trigger.dev/api/v1/waitpoints/tokens/waitpoint_abc123/callback/abc123hash"
```

**REST**: `POST /api/v1/waitpoints/tokens`

### CreateToken options

| Option | Type | Required | Description |
|---|---|---|---|
| `idempotencyKey` | `string` | No | If the same key is passed again before it expires, the original token is returned. The returned token may already be completed, in which case `wait.forToken()` continues immediately. |
| `idempotencyKeyTTL` | `string` | No | How long the idempotency key is valid. After expiry, the same key creates a new token. Accepts duration shorthand: `"30s"`, `"1m"`, `"2h"`, `"3d"`. |
| `timeout` | `string` | No | How long before the token times out. When a run is waiting for a timed-out token, `wait.forToken()` returns with `ok: false`. Accepts ISO 8601 date or duration shorthand: `"30s"`, `"1m"`, `"2h"`, `"3d"`, `"4w"`. |
| `tags` | `string \| string[]` | No | Tags to attach. Up to 10 tags, each under 128 characters. Namespace with a prefix like `user:1234567`. |

### CreateToken response

| Field | Type | Description |
|---|---|---|
| `id` | `string` | Unique waitpoint token ID (e.g. `waitpoint_abc123`) |
| `isCached` | `boolean` | `true` if an existing token was returned due to idempotency key match |
| `url` | `string` | Pre-signed HTTP callback URL for completing the token without an API key |

---

## Waiting for Tokens — wait.forToken()

Pause the current run until a token is completed or times out.

```ts
import { task, wait } from "@trigger.dev/sdk";

export const approvalWorkflow = task({
  id: "approval-workflow",
  run: async (payload: { userId: string; requestId: string }) => {
    const token = await wait.createToken({
      timeout: "24h",
      tags: [`user:${payload.userId}`],
    });

    // Send the callback URL to the approver
    await sendApprovalEmail({
      to: "manager@example.com",
      callbackUrl: token.url,
      requestId: payload.requestId,
    });

    // Run pauses here — checkpointed, no compute cost
    const result = await wait.forToken<{ status: string; comment: string }>(token);

    if (result.ok) {
      console.log(result.output); // { status: "approved", comment: "Looks good!" }
      await processApproval(payload.requestId);
    } else {
      // Token timed out
      console.log("Approval timed out");
      await handleTimeout(payload.requestId);
    }
  },
});
```

### wait.forToken() return value

`wait.forToken()` returns a `Result` object:

| Field | Type | Description |
|---|---|---|
| `ok` | `boolean` | `true` if the token was completed, `false` if it timed out |
| `output` | `T` | The data passed during completion (only when `ok` is `true`) |

You can also use `.unwrap()` to get the output directly (throws on timeout):

```ts
const output = await wait.forToken<{ status: string }>(token).unwrap();
// Throws if the token timed out
```

---

## Completing Tokens — wait.completeToken()

Complete a waitpoint token from your backend or frontend, unblocking any run waiting on it.

```ts
import { wait } from "@trigger.dev/sdk";

// Complete with data
await wait.completeToken(
  { id: "waitpoint_abc123" },
  {
    status: "approved",
    comment: "Looks good to me!",
  }
);

// Complete with no data
await wait.completeToken({ id: "waitpoint_abc123" }, {});
```

**REST**: `POST /api/v1/waitpoints/tokens/{waitpointId}/complete`

### Authentication

This endpoint accepts **two** authentication methods:

| Method | When to use |
|---|---|
| **Secret API key** (`Authorization: Bearer tr_dev_...`) | Backend calls |
| **Public access token** (short-lived JWT) | Frontend / client-side calls — safe to expose |

This makes it possible to complete waitpoints from browser-based approval UIs without exposing your secret key.

### CompleteToken request body

| Field | Type | Required | Description |
|---|---|---|---|
| `data` | `any (JSON-serializable)` | No | Payload returned to the waiting run via `wait.forToken()` |

### CompleteToken response

| Field | Type | Description |
|---|---|---|
| `success` | `boolean` | Always `true` on success |

> **Note**: If the token is already completed, this is a no-op and still returns `success: true`.

---

## HTTP Callback Completion

Every token includes a pre-signed `url` that can complete the token via a simple HTTP POST — **no API key required**. The `callbackHash` embedded in the URL authenticates the request.

This is ideal for:
- Webhook callbacks from external services (e.g., Replicate, Stripe)
- Approval links in emails
- Third-party integrations that can POST to a URL

```ts
import { task, wait } from "@trigger.dev/sdk";

export const generateImage = task({
  id: "generate-image",
  run: async (payload: { prompt: string }) => {
    const token = await wait.createToken({ timeout: "10m" });

    // Give the callback URL to an external service
    await replicate.predictions.create({
      model: "stability-ai/sdxl",
      input: { prompt: payload.prompt },
      webhook: token.url,
      webhook_events_filter: ["completed"],
    });

    // Run pauses here until the webhook fires
    const result = await wait.forToken<{ output: string }>(token).unwrap();

    return { imageUrl: result.output };
  },
});
```

**REST**: `POST /api/v1/waitpoints/tokens/{waitpointId}/callback/{callbackHash}`

### How it works

1. The **entire request body** is passed as the output data to the waiting run.
2. If the body is not valid JSON, an empty object is used.
3. No `Authorization` header is needed — the `callbackHash` in the URL is the authentication.

### Callback-specific responses

| Status | Description |
|---|---|
| `200` | Token completed successfully |
| `401` | Invalid callback URL or hash mismatch |
| `404` | Waitpoint token not found |
| `405` | Method not allowed (must be POST) |
| `411` | `Content-Length` header required |
| `413` | Request body too large |

> **WARNING**: Do not construct the `callbackHash` manually. Always use the `url` returned from `wait.createToken()`. The hash is an HMAC signature that Trigger.dev generates and verifies server-side.

**cURL example**:

```bash
curl -X POST \
  "https://api.trigger.dev/api/v1/waitpoints/tokens/waitpoint_abc123/callback/abc123hash" \
  -H "Content-Type: application/json" \
  -d '{"status": "approved", "comment": "Looks good to me!"}'
```

---

## Listing Tokens — wait.listTokens()

List waitpoint tokens with cursor-based pagination and filters. Results are ordered by creation date, newest first.

```ts
import { wait } from "@trigger.dev/sdk";

// Iterate over all tokens (auto-paginated)
for await (const token of wait.listTokens()) {
  console.log(token.id, token.status);
}

// Filter by status and tags
for await (const token of wait.listTokens({
  status: ["WAITING"],
  tags: ["user:1234567"],
})) {
  console.log(token.id);
}

// Filter by creation date
for await (const token of wait.listTokens({
  createdAt: { period: "24h" },
})) {
  console.log(token.id, token.createdAt);
}
```

**REST**: `GET /api/v1/waitpoints/tokens`

### ListTokens filter options

| Option | Type | Description |
|---|---|---|
| `status` | `string[]` | Filter by status: `"WAITING"`, `"COMPLETED"`, `"TIMED_OUT"` |
| `tags` | `string[]` | Filter by tags (comma-separated in REST) |
| `idempotencyKey` | `string` | Filter by idempotency key |
| `createdAt.period` | `string` | Shorthand period: `"1h"`, `"24h"`, `"7d"` |
| `createdAt.from` | `string (ISO 8601)` | Tokens created at or after this timestamp |
| `createdAt.to` | `string (ISO 8601)` | Tokens created at or before this timestamp |

> **WARNING**: `createdAt.period` cannot be combined with `createdAt.from` or `createdAt.to`. Use one approach or the other.

### Pagination (REST)

| Query parameter | Type | Description |
|---|---|---|
| `page[size]` | `integer` (1–100) | Tokens per page |
| `page[after]` | `string` | Cursor from `pagination.next` |
| `page[before]` | `string` | Cursor from `pagination.previous` |

### ListTokens response (REST)

| Field | Type | Description |
|---|---|---|
| `data` | `WaitpointTokenObject[]` | Array of token objects |
| `pagination.next` | `string \| null` | Next page cursor |
| `pagination.previous` | `string \| null` | Previous page cursor |

---

## Retrieving Tokens — wait.retrieveToken()

Retrieve a single waitpoint token by ID.

```ts
import { wait } from "@trigger.dev/sdk";

const token = await wait.retrieveToken("waitpoint_abc123");

console.log(token.status); // "WAITING" | "COMPLETED" | "TIMED_OUT"

if (token.status === "COMPLETED") {
  console.log(token.output);
}
```

**REST**: `GET /api/v1/waitpoints/tokens/{waitpointId}`

Returns `404` if the token is not found.

---

## WaitpointTokenObject Schema

Token objects returned by list, retrieve, and create operations:

| Field | Type | Description |
|---|---|---|
| `id` | `string` | Unique token ID (e.g. `waitpoint_abc123`) |
| `url` | `string` | Pre-signed HTTP callback URL |
| `status` | `"WAITING" \| "COMPLETED" \| "TIMED_OUT"` | Current token status |
| `idempotencyKey` | `string \| null` | Idempotency key used during creation |
| `idempotencyKeyExpiresAt` | `string (ISO 8601) \| null` | When the idempotency key expires |
| `timeoutAt` | `string (ISO 8601) \| null` | When the token will time out |
| `completedAt` | `string (ISO 8601) \| null` | When the token was completed |
| `output` | `string \| null` | Serialized output data (only when `COMPLETED`) |
| `outputType` | `string \| null` | Content type of the output (e.g. `"application/json"`) |
| `outputIsError` | `boolean \| null` | Whether the output represents an error |
| `tags` | `string[]` | Tags attached to the token |
| `createdAt` | `string (ISO 8601)` | When the token was created |

---

## Checkpointing During Waits

All wait operations (`wait.for()`, `wait.until()`, `wait.forToken()`) trigger checkpointing:

1. **Run state is persisted** — execution context, variables, and call stack are saved.
2. **Compute is released** — the container shuts down, no compute costs accrue.
3. **Concurrency slot is freed** — other queued runs can proceed.
4. **On wake-up** — the run is restored from checkpoint and re-acquires a slot.

<!-- UNVERIFIED -->
Action item: verify the ~5-second checkpoint threshold in `https://trigger.dev/docs/wait` — short waits may spin-wait instead of checkpointing.

> **WARNING**: Waits shorter than approximately 5 seconds may not trigger a full checkpoint. The container may stay alive, continuing to consume compute and holding its concurrency slot. For cost-efficient waits, use durations of 5 seconds or longer.

### Idempotency keys on waits

<!-- UNVERIFIED -->
Action item: verify exact idempotency key skip-on-retry behavior in `https://trigger.dev/docs/idempotency`.

When a task retries (due to an error after the wait), you may not want to wait again. Idempotency keys solve this:

```ts
import { task, wait } from "@trigger.dev/sdk";

export const processWithApproval = task({
  id: "process-with-approval",
  retry: { maxAttempts: 3 },
  run: async (payload: { requestId: string }) => {
    const token = await wait.createToken({
      timeout: "24h",
      idempotencyKey: `approval-${payload.requestId}`,
      idempotencyKeyTTL: "48h",
    });

    // On retry: if the token was already completed, forToken() resolves immediately
    const result = await wait.forToken(token);

    if (!result.ok) {
      throw new Error("Approval timed out");
    }

    // This part might fail, causing a retry
    await processApprovedRequest(payload.requestId, result.output);
  },
});
```

On retry:
1. `wait.createToken()` returns the **same token** (because the idempotency key matches).
2. `isCached` is `true`.
3. If the token was already completed, `wait.forToken()` resolves immediately — the run skips the wait.

---

## Input Streams

<!-- UNVERIFIED -->
Action item: verify input stream API shape and availability in SDK 4.4.3 at `https://trigger.dev/docs/realtime/streams`.

`inputStream` provides a higher-level abstraction over waitpoint tokens for scenarios where you want to stream data into a running task from an external source.

```ts
import { task, inputStream } from "@trigger.dev/sdk";

export const interactiveTask = task({
  id: "interactive-task",
  run: async (payload, { ctx }) => {
    // Wait for input from outside the task
    const userInput = await inputStream.wait({
      timeout: "5m",
      schema: z.object({
        action: z.enum(["approve", "reject"]),
        reason: z.string().optional(),
      }),
    });

    if (userInput.action === "approve") {
      await doApprovedWork();
    } else {
      await handleRejection(userInput.reason);
    }
  },
});
```

Under the hood, `inputStream.wait()` creates a waitpoint token and waits for it, providing:

- **Schema validation** on the incoming data.
- **A simpler API** — no need to manually create and forward token IDs.
- **Integration with Realtime** — frontends can subscribe and push data into the stream.

> **WARNING**: `inputStream` is a newer API. Consult the [latest Trigger.dev documentation](https://trigger.dev/docs) to confirm availability and exact API shape for your SDK version.

---

## Complete Example: Human-in-the-Loop Approval

```ts
import { task, wait } from "@trigger.dev/sdk";

export const orderApproval = task({
  id: "order-approval",
  run: async (payload: { orderId: string; amount: number; userId: string }) => {
    // Step 1: Create a token for the approval
    const token = await wait.createToken({
      timeout: "48h",
      tags: [`order:${payload.orderId}`, `user:${payload.userId}`],
      idempotencyKey: `order-approval-${payload.orderId}`,
    });

    // Step 2: Notify the approver (they can POST to token.url to approve)
    await notifyApprover({
      orderId: payload.orderId,
      amount: payload.amount,
      approvalUrl: token.url, // No API key needed to call this URL
    });

    // Step 3: Wait — run is checkpointed, no compute cost
    const result = await wait.forToken<{
      approved: boolean;
      approverName: string;
    }>(token);

    // Step 4: Handle the result
    if (!result.ok) {
      // Timed out after 48 hours
      await cancelOrder(payload.orderId, "Approval timed out");
      return { status: "timed_out" };
    }

    if (result.output.approved) {
      await fulfillOrder(payload.orderId);
      return { status: "approved", by: result.output.approverName };
    } else {
      await cancelOrder(payload.orderId, `Rejected by ${result.output.approverName}`);
      return { status: "rejected", by: result.output.approverName };
    }
  },
});
```

---

## Related Documentation

- [Queues and Concurrency](./TRIGGER_DEV_QUEUES_CONCURRENCY.md) — queue behavior, concurrency limits, checkpointing
- [Tasks Overview](./TRIGGER_DEV_TASKS.md)
- [Management API](./TRIGGER_DEV_MANAGEMENT_API.md) — waitpoint token endpoints
- [Errors & Retries](./TRIGGER_DEV_ERRORS_RETRIES.md)
- [Master Reference](./TRIGGER_DEV_MASTER.md)
