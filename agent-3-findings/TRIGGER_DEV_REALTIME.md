# Trigger.dev Realtime

> **SDK version:** 4.4.3 | Canonical reference for Trigger.dev Realtime — subscribing to run updates, streaming data, and using React hooks.
> Source: [Trigger.dev Realtime](https://trigger.dev/docs/realtime/overview)

---

## Overview

Trigger.dev Realtime lets you subscribe to task run updates and stream data from running tasks to your frontend or backend in real time. Key capabilities:

- **Subscribe to individual runs** -- get live status, metadata, and output updates.
- **Subscribe by tag** -- watch all runs that share a specific tag.
- **Subscribe to batches** -- track all runs within a batch.
- **Stream data** -- pipe continuous output (e.g., LLM token streams) from tasks to clients while they execute.

---

## Public Access Tokens

To subscribe to runs from the **frontend** (browser), you need a **Public Access Token**. These are scoped, short-lived tokens that can be safely exposed to the client without revealing your secret key.

### Creating a Public Access Token

Generate tokens from your backend using `auth.createPublicToken()`:

```ts
import { auth } from "@trigger.dev/sdk";

// Token scoped to a specific run
const token = await auth.createPublicToken({
  scopes: {
    read: {
      runs: ["run_1234"],
    },
  },
});

// Token scoped to all runs of a specific task
const taskToken = await auth.createPublicToken({
  scopes: {
    read: {
      tasks: ["my-task"],
    },
  },
});

// Token scoped to runs with a specific tag
const tagToken = await auth.createPublicToken({
  scopes: {
    read: {
      tags: ["user_abc123"],
    },
  },
});

// Token scoped to a batch
const batchToken = await auth.createPublicToken({
  scopes: {
    read: {
      batches: ["batch_1234"],
    },
  },
});
```

<!-- UNVERIFIED -->
Action item: confirm exact `createPublicToken` scopes object structure at `https://trigger.dev/docs/frontend/overview`.

### Scope Types

| Scope | Type | Description |
|---|---|---|
| `read.runs` | `string[]` | Array of run IDs (e.g., `["run_1234"]`). Token can subscribe to these specific runs. |
| `read.tasks` | `string[]` | Array of task identifiers. Token can subscribe to any run of these tasks. |
| `read.tags` | `string[]` | Array of tag strings. Token can subscribe to any run with these tags. |
| `read.batches` | `string[]` | Array of batch IDs. Token can subscribe to runs within these batches. |

### Token Expiration

Public Access Tokens have a default expiration. You can customize it:

```ts
const token = await auth.createPublicToken({
  scopes: {
    read: {
      runs: ["run_1234"],
    },
  },
  expirationTime: "1h", // e.g., "15m", "1h", "24h"
});
```

<!-- UNVERIFIED -->
Action item: verify `expirationTime` format and default duration at `https://trigger.dev/docs/frontend/overview`.

> **WARNING:** Public Access Tokens should only grant the minimum scope needed. Never create tokens with broad task or tag scopes for untrusted clients.

---

## The Run Object

When you subscribe to a run (via hooks or backend subscriptions), you receive a run object with the following key fields:

| Property | Type | Description |
|---|---|---|
| `id` | `string` | Unique run ID, prefixed with `run_`. |
| `status` | `RunStatus` | Current status of the run (see lifecycle below). |
| `taskIdentifier` | `string` | The `id` of the task that was run. |
| `payload` | `object` | The input payload sent to the task. Omitted with Public API keys. |
| `output` | `object` | The return value of the task. Populated on `COMPLETED`. Omitted with Public API keys. |
| `metadata` | `object` | Arbitrary metadata attached to the run. |
| `tags` | `string[]` | Tags attached to the run (up to 10, each 1-128 characters). |
| `createdAt` | `string` (ISO 8601) | When the run was created. |
| `updatedAt` | `string` (ISO 8601) | When the run was last updated. |
| `startedAt` | `string` (ISO 8601) | When the run started executing. |
| `finishedAt` | `string` (ISO 8601) | When the run finished (completed, failed, etc.). |
| `attempts` | `Attempt[]` | Array of attempt objects with status and error info. |
| `durationMs` | `number` | Compute duration in milliseconds (excludes waits). |
| `isTest` | `boolean` | Whether this is a test run. |
| `version` | `string` | Worker version that executed the run. |
| `idempotencyKey` | `string` | Idempotency key if provided when triggering. |
| `batchId` | `string` | Batch ID if this run is part of a batch. |
| `costInCents` | `number` | Compute cost so far in cents (not applicable to DEV). |

### Run Status Lifecycle

```
PENDING_VERSION --> QUEUED --> EXECUTING --> COMPLETED
                                   |
                                   +--> REATTEMPTING --> EXECUTING
                                   |
                                   +--> FROZEN --> EXECUTING
                                   |
                                   +--> FAILED
                                   |
                                   +--> CANCELED
                                   |
                                   +--> CRASHED
                                   |
                                   +--> INTERRUPTED
                                   |
                                   +--> SYSTEM_FAILURE
```

| Status | Description |
|---|---|
| `PENDING_VERSION` | Run is waiting for a deployed version that can execute the task. |
| `DELAYED` | Run was triggered with a delay and is waiting for the scheduled time. |
| `QUEUED` | Run is in the queue waiting for available capacity. |
| `EXECUTING` | Run is currently executing. |
| `REATTEMPTING` | Run failed and is waiting to retry (between attempts). |
| `FROZEN` | Run is paused (e.g., waiting on a `wait.for()`, `wait.until()`, or waitpoint token). |
| `COMPLETED` | Run finished successfully. `output` is populated. |
| `CANCELED` | Run was manually canceled. |
| `FAILED` | Run exhausted all retry attempts or was aborted via `AbortTaskRunError`. |
| `CRASHED` | Run crashed due to an unrecoverable system error (e.g., out of memory). |
| `INTERRUPTED` | Run was interrupted (e.g., during deployment rollover). |
| `SYSTEM_FAILURE` | Internal Trigger.dev platform failure. |

### Boolean Status Helpers

The run object includes convenience boolean properties:

| Helper | `true` when status is |
|---|---|
| `isSuccess` | `COMPLETED` |
| `isFailed` | `FAILED` |
| `isCompleted` | Any terminal status (`COMPLETED`, `FAILED`, `CANCELED`, `CRASHED`, `INTERRUPTED`, `SYSTEM_FAILURE`) |

<!-- UNVERIFIED -->
Action item: verify full list of boolean status helpers at `https://trigger.dev/docs/realtime`.

---

## React Hooks

The `@trigger.dev/react-hooks` package provides React hooks for subscribing to runs, batches, and streams from frontend components.

### Installation

```bash
npm install @trigger.dev/react-hooks
```

### `useRealtimeRun`

Subscribe to a single run by ID. Returns the live-updating run object.

```tsx
import { useRealtimeRun } from "@trigger.dev/react-hooks";

function RunStatus({ runId, publicAccessToken }: { runId: string; publicAccessToken: string }) {
  const { run, error } = useRealtimeRun(runId, {
    accessToken: publicAccessToken,
  });

  if (error) return <div>Error: {error.message}</div>;
  if (!run) return <div>Loading...</div>;

  return (
    <div>
      <p>Status: {run.status}</p>
      {run.output && <p>Output: {JSON.stringify(run.output)}</p>}
    </div>
  );
}
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `runId` | `string` | Yes | The run ID to subscribe to (e.g., `"run_1234"`). |
| `options.accessToken` | `string` | Yes | A Public Access Token with `read.runs` scope for this run. |
| `options.enabled` | `boolean` | No | Whether the subscription is active. Defaults to `true`. <!-- UNVERIFIED --> Action item: verify at `https://trigger.dev/docs/realtime`. |

**Returns:**

| Property | Type | Description |
|---|---|---|
| `run` | `Run \| undefined` | The live-updating run object, or `undefined` while loading. |
| `error` | `Error \| undefined` | Any subscription error. |

### `useRealtimeRunsWithTag`

Subscribe to all runs that share a specific tag. Useful for tracking multiple related tasks (e.g., all tasks for a specific user or workflow).

```tsx
import { useRealtimeRunsWithTag } from "@trigger.dev/react-hooks";

function UserRuns({ tag, publicAccessToken }: { tag: string; publicAccessToken: string }) {
  const { runs, error } = useRealtimeRunsWithTag(tag, {
    accessToken: publicAccessToken,
  });

  if (error) return <div>Error: {error.message}</div>;

  return (
    <ul>
      {runs.map((run) => (
        <li key={run.id}>
          {run.taskIdentifier}: {run.status}
        </li>
      ))}
    </ul>
  );
}
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `tag` | `string` | Yes | The tag to subscribe to (e.g., `"user_abc123"`). |
| `options.accessToken` | `string` | Yes | A Public Access Token with `read.tags` scope for this tag. |

**Returns:**

| Property | Type | Description |
|---|---|---|
| `runs` | `Run[]` | Array of live-updating run objects matching the tag. |
| `error` | `Error \| undefined` | Any subscription error. |

### `useRealtimeBatch`

Subscribe to all runs within a batch.

```tsx
import { useRealtimeBatch } from "@trigger.dev/react-hooks";

function BatchProgress({
  batchId,
  publicAccessToken,
}: {
  batchId: string;
  publicAccessToken: string;
}) {
  const { runs, error } = useRealtimeBatch(batchId, {
    accessToken: publicAccessToken,
  });

  if (error) return <div>Error: {error.message}</div>;

  const completed = runs.filter((r) => r.status === "COMPLETED").length;
  return (
    <div>
      Progress: {completed} / {runs.length}
    </div>
  );
}
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `batchId` | `string` | Yes | The batch ID to subscribe to (e.g., `"batch_1234"`). |
| `options.accessToken` | `string` | Yes | A Public Access Token with `read.batches` scope for this batch. |

**Returns:**

| Property | Type | Description |
|---|---|---|
| `runs` | `Run[]` | Array of live-updating run objects in the batch. |
| `error` | `Error \| undefined` | Any subscription error. |

### `useWaitToken`

<!-- UNVERIFIED -->
Action item: verify exact `useWaitToken` hook name and API at `https://trigger.dev/docs/realtime`.

Subscribe to a waitpoint token, useful for human-in-the-loop workflows where you need to display the token status and provide a UI to complete it.

```tsx
import { useWaitToken } from "@trigger.dev/react-hooks";

function ApprovalButton({
  tokenId,
  publicAccessToken,
}: {
  tokenId: string;
  publicAccessToken: string;
}) {
  const { token, error } = useWaitToken(tokenId, {
    accessToken: publicAccessToken,
  });

  if (!token) return <div>Loading...</div>;

  return (
    <div>
      <p>Status: {token.status}</p>
      {token.status === "WAITING" && (
        <button onClick={() => completeToken(tokenId)}>Approve</button>
      )}
    </div>
  );
}
```

---

## Streams

Streams let you pipe continuous data from a running task to subscribed clients. This is especially useful for forwarding LLM token-by-token responses to the frontend.

### Defining Output Streams (Streams v2)

Use `streams.define()` to create typed streams with a schema. Available since SDK 4.1.0.

```ts
// trigger/my-streams.ts
import { streams } from "@trigger.dev/sdk";
import { z } from "zod";

// Define a typed stream with a Zod schema
export const textStream = streams.define("text-output", z.object({
  token: z.string(),
  done: z.boolean().optional(),
}));
```

### Writing to Streams from a Task

```ts
import { task } from "@trigger.dev/sdk";
import { textStream } from "./my-streams";

export const generateText = task({
  id: "generate-text",
  run: async (payload: { prompt: string }) => {
    // Option 1: Pipe an async iterable directly
    const llmStream = await callLLM(payload.prompt);
    textStream.pipe(llmStream);

    // Option 2: Write individual chunks
    for await (const token of llmStream) {
      textStream.write({ token, done: false });
    }
    textStream.write({ token: "", done: true });

    return { success: true };
  },
});
```

<!-- UNVERIFIED -->
Action item: verify whether both `.pipe()` and `.write()` are available at `https://trigger.dev/docs/realtime/streams`.

### Forwarding LLM Streams (Vercel AI SDK Example)

```ts
import { task } from "@trigger.dev/sdk";
import { streams } from "@trigger.dev/sdk";
import { streamText } from "ai";
import { openai } from "@ai-sdk/openai";
import { z } from "zod";

const aiStream = streams.define("ai-response", z.object({
  text: z.string(),
}));

export const chatTask = task({
  id: "chat-task",
  run: async (payload: { prompt: string }) => {
    const result = await streamText({
      model: openai("gpt-4o"),
      prompt: payload.prompt,
    });

    // Pipe the AI SDK stream directly
    aiStream.pipe(result.textStream);

    return { success: true };
  },
});
```

<!-- UNVERIFIED -->
Action item: verify exact Vercel AI SDK integration pattern at `https://trigger.dev/docs/realtime/streams`.

### Subscribing to Streams from the Frontend

#### `useRealtimeRunWithStreams`

The older hook for subscribing to both run updates and streams simultaneously:

```tsx
import { useRealtimeRunWithStreams } from "@trigger.dev/react-hooks";

function ChatOutput({ runId, accessToken }: { runId: string; accessToken: string }) {
  const { run, streams } = useRealtimeRunWithStreams(runId, {
    accessToken,
  });

  const textChunks = streams?.["text-output"] ?? [];

  return (
    <div>
      <p>Status: {run?.status}</p>
      <div>
        {textChunks.map((chunk, i) => (
          <span key={i}>{chunk.token}</span>
        ))}
      </div>
    </div>
  );
}
```

#### `useRealtimeStream` (Recommended, SDK 4.1.0+)

The newer, simpler API with better type safety when used with defined streams:

```tsx
import { useRealtimeStream } from "@trigger.dev/react-hooks";

function ChatOutput({ runId, accessToken }: { runId: string; accessToken: string }) {
  const { data, error } = useRealtimeStream(runId, "text-output", {
    accessToken,
  });

  return (
    <div>
      {data.map((chunk, i) => (
        <span key={i}>{chunk.token}</span>
      ))}
    </div>
  );
}
```

<!-- UNVERIFIED -->
Action item: verify `useRealtimeStream` API signature at `https://trigger.dev/docs/realtime`.

> **WARNING:** `useRealtimeRunWithStreams` is the older API. For new projects, prefer `useRealtimeStream` (SDK 4.1.0+) for a simpler API and better type safety with defined streams.

---

## Backend Subscriptions

You can also subscribe to run updates from your backend (e.g., in a long-running server process) using the `runs` module.

### `runs.subscribeToRun()`

```ts
import { runs } from "@trigger.dev/sdk";

// Subscribe to a single run
for await (const update of runs.subscribeToRun("run_1234")) {
  console.log("Run status:", update.status);
  if (update.status === "COMPLETED") {
    console.log("Output:", update.output);
    break;
  }
}
```

<!-- UNVERIFIED -->
Action item: verify `subscribeToRun` return type (AsyncIterable vs callback) at `https://trigger.dev/docs/realtime`.

### `runs.subscribeToRunsWithTag()`

```ts
import { runs } from "@trigger.dev/sdk";

// Subscribe to all runs with a specific tag
for await (const update of runs.subscribeToRunsWithTag("user_abc123")) {
  console.log(`Run ${update.id}: ${update.status}`);
}
```

<!-- UNVERIFIED -->
Action item: verify `subscribeToRunsWithTag` API at `https://trigger.dev/docs/realtime`.

> **WARNING:** Backend subscriptions use your `TRIGGER_SECRET_KEY` and should never be exposed to the client. For frontend subscriptions, always use Public Access Tokens with React hooks.

---

## Common Patterns

### Full-Stack Realtime Flow (Next.js)

A typical pattern for triggering a task and subscribing to updates from a Next.js frontend:

**1. Server action -- trigger the task and create a public token:**

```ts
// app/actions.ts
"use server";

import { tasks, auth } from "@trigger.dev/sdk";
import type { myTask } from "@/trigger/my-task";

export async function startTask(input: string) {
  const handle = await tasks.trigger<typeof myTask>("my-task", { input });

  const publicToken = await auth.createPublicToken({
    scopes: {
      read: {
        runs: [handle.id],
      },
    },
  });

  return { runId: handle.id, publicAccessToken: publicToken };
}
```

**2. Client component -- subscribe to the run:**

```tsx
"use client";

import { useRealtimeRun } from "@trigger.dev/react-hooks";
import { startTask } from "./actions";
import { useState } from "react";

export function TaskRunner() {
  const [runInfo, setRunInfo] = useState<{
    runId: string;
    publicAccessToken: string;
  } | null>(null);

  const { run } = useRealtimeRun(runInfo?.runId ?? "", {
    accessToken: runInfo?.publicAccessToken ?? "",
    enabled: !!runInfo,
  });

  async function handleClick() {
    const info = await startTask("hello world");
    setRunInfo(info);
  }

  return (
    <div>
      <button onClick={() => handleClick()}>Start Task</button>
      {run && (
        <div>
          <p>Status: {run.status}</p>
          {run.metadata && <p>Progress: {JSON.stringify(run.metadata)}</p>}
          {run.output && <pre>{JSON.stringify(run.output, null, 2)}</pre>}
        </div>
      )}
    </div>
  );
}
```

### Tracking Progress with Metadata

Tasks can update their metadata during execution. Subscribed clients see these updates in real time:

```ts
import { task, metadata } from "@trigger.dev/sdk";

export const processBatch = task({
  id: "process-batch",
  run: async (payload: { items: string[] }) => {
    for (let i = 0; i < payload.items.length; i++) {
      await processItem(payload.items[i]);
      // Update metadata -- subscribers see this immediately
      metadata.set("progress", {
        current: i + 1,
        total: payload.items.length,
      });
    }
    return { processed: payload.items.length };
  },
});
```

<!-- UNVERIFIED -->
Action item: verify `metadata.set` API at `https://trigger.dev/docs/runs/metadata`.

---

## Quick Reference

| Feature | Frontend | Backend | Package |
|---|---|---|---|
| `useRealtimeRun` | Yes | -- | `@trigger.dev/react-hooks` |
| `useRealtimeRunsWithTag` | Yes | -- | `@trigger.dev/react-hooks` |
| `useRealtimeBatch` | Yes | -- | `@trigger.dev/react-hooks` |
| `useWaitToken` | Yes | -- | `@trigger.dev/react-hooks` |
| `useRealtimeRunWithStreams` | Yes | -- | `@trigger.dev/react-hooks` |
| `useRealtimeStream` | Yes | -- | `@trigger.dev/react-hooks` |
| `runs.subscribeToRun()` | -- | Yes | `@trigger.dev/sdk` |
| `runs.subscribeToRunsWithTag()` | -- | Yes | `@trigger.dev/sdk` |
| `auth.createPublicToken()` | -- | Yes | `@trigger.dev/sdk` |
| `streams.define()` | -- | Yes (task) | `@trigger.dev/sdk` |

---

## metadata.stream() Deprecation

`metadata.stream()` is **deprecated** as of SDK 4.1.0. Use Realtime Streams v2 (`streams.pipe()`) instead for all stream forwarding use cases. See [Streams](./TRIGGER_DEV_STREAMS.md) for the replacement API.

---

## See Also

- [Tasks Overview](./TRIGGER_DEV_TASKS.md)
- [Streams](./TRIGGER_DEV_STREAMS.md) — output streams, input streams, Streams v2 API
- [Errors & Retries](./TRIGGER_DEV_ERRORS_RETRIES.md)
- [Management API](./TRIGGER_DEV_MANAGEMENT_API.md)
- [Master Reference](./TRIGGER_DEV_MASTER.md)
