# TRIGGER_DEV_STREAMS

> **SDK version:** 4.4.3 | Dedicated stream reference for Trigger.dev output streams, input streams, and Streams v2.

---

## Stream Types

- **Output streams**: task emits incremental events/data to subscribers.
- **Input streams**: external systems/UI send typed data into a running task.

Use streams when:

- progress/result data is incremental (LLM tokens, step events),
- users must interact with in-flight runs (approve/cancel/feedback),
- you need realtime UX without polling loops.

---

## Defining Output Streams with `streams.define()` (Streams v2)

Use `streams.define()` to create typed streams with a Zod schema. Available since SDK 4.1.0. This replaces the deprecated `metadata.stream()` approach.

### Basic Definition

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

### Simple (Generic) Definition

```ts
import { streams, task } from "@trigger.dev/sdk";

const tokens = streams.define<{ token: string }>({ id: "tokens" });

export const generate = task({
  id: "generate",
  run: async () => {
    tokens.append({ token: "Hello" });
    tokens.append({ token: " world" });
    return { done: true };
  },
});
```

| Property | Type | Required | Default | Description |
|---|---|---:|---|---|
| `id` | `string` | Yes | - | Stream identifier used by subscribers |
| `schema` | `ZodSchema` | No | - | Zod schema for type-safe stream payloads |
| payload type | generic | No | `unknown` | Event payload shape (when not using Zod) |

---

## Writing to Streams (`streams.pipe()` and `.write()`)

### `streams.pipe()` — Forward an Async Iterable

`streams.pipe()` replaces the deprecated `metadata.stream()`. Use it to forward LLM token streams or any async iterable directly:

```ts
import { task } from "@trigger.dev/sdk";
import { textStream } from "./my-streams";

export const generateText = task({
  id: "generate-text",
  run: async (payload: { prompt: string }) => {
    const llmStream = await callLLM(payload.prompt);

    // Pipe the async iterable directly to the stream
    textStream.pipe(llmStream);

    return { success: true };
  },
});
```

### `.write()` — Write Individual Chunks

```ts
export const generateText = task({
  id: "generate-text",
  run: async (payload: { prompt: string }) => {
    const llmStream = await callLLM(payload.prompt);

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

> ⚠️ WARNING
> `metadata.stream()` is **deprecated** as of SDK 4.1.0. Use `streams.pipe()` or `.write()` instead for all stream forwarding use cases.

---

## Vercel AI SDK Integration

Forward AI SDK streams directly to Trigger.dev streams:

```ts
import { task, streams } from "@trigger.dev/sdk";
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

---

## Defining Input Streams

Input streams allow external systems or UI to send typed data into a running task:

```ts
import { streams, task } from "@trigger.dev/sdk";

export const approval = streams.input<{ approved: boolean; reviewer: string }>({
  id: "approval",
});

export const publish = task({
  id: "publish",
  run: async () => {
    const result = await approval.wait({ timeout: "7d" });
    if (!result.ok) return { published: false, reason: "timeout" };
    return { published: result.output.approved, reviewer: result.output.reviewer };
  },
});
```

| Property | Type | Required | Default | Description |
|---|---|---:|---|---|
| `id` | `string` | Yes | - | Input stream identifier |
| payload type | generic | No | `unknown` | Incoming message shape |

---

## Input Stream Runtime Methods

| Method | Type | Required | Default | Description |
|---|---|---:|---|---|
| `.wait({ timeout })` | function | No | no timeout | Suspend until message/timeout |
| `.once()` | function | No | n/a | Receive next message once |
| `.on(handler)` | function | No | n/a | Register ongoing listener |
| `.send(runId, data)` | function | Yes | - | Push data into target run |

> ⚠️ WARNING
> Use `.wait()` for checkpoint-friendly suspension; avoid custom polling loops.

---

## Subscribing to Streams (Backend)

Use `.withStreams()` on a run subscription to include stream events:

```ts
import { runs } from "@trigger.dev/sdk";

for await (const update of runs.subscribeToRun("run_123").withStreams()) {
  console.log(update.run.status);
  console.log(update.streams);
}
```

| Property | Type | Required | Default | Description |
|---|---|---:|---|---|
| `runId` | `string` | Yes | - | Run to subscribe to |
| `.withStreams()` | method | No | run-only updates | Include stream events in subscription |

---

## Subscribing to Streams (Frontend)

### `useRealtimeStream` (Recommended, SDK 4.1.0+)

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
Action item: verify exact `useRealtimeStream` API signature and return shape at `https://trigger.dev/docs/realtime/react-hooks/streams`.

### `useRealtimeRunWithStreams` (Legacy)

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

> ⚠️ WARNING
> `useRealtimeRunWithStreams` is the older API. For new projects, prefer `useRealtimeStream` (SDK 4.1.0+) for a simpler API and better type safety with defined streams.

### Sending to Input Streams from Frontend

```ts
import { useInputStreamSend } from "@trigger.dev/react-hooks";

const { send, isReady, isLoading } = useInputStreamSend("approval", runId, {
  accessToken: publicAccessToken,
});

await send({ approved: true, reviewer: "alice@example.com" });
```

| Property | Type | Required | Default | Description |
|---|---|---:|---|---|
| `streamId` | `string` | Yes | - | Stream identifier |
| `runId` | `string` | Yes | - | Target run |
| `accessToken` | `string` | Yes | - | Public Access Token |
| `baseURL` | `string` | No | Trigger default | Realtime API URL override |

---

## Typing and Contract Strategy

- Define stream schemas once in a shared module using `streams.define()` with Zod.
- Reuse stream ids/types in both task and UI code to avoid type drift.
- Keep payloads compact and append-only when possible.

---

## Production Gotchas

- Token scopes must allow stream read/write on the targeted run.
- Excessive high-frequency emits can overwhelm UI render loops; throttle/coalesce in the frontend.
- Keep stream IDs stable across deploys to reduce client breakage.
- `metadata.stream()` is deprecated — migrate to `streams.pipe()` for all new code.

---

## Related

- [Realtime overview](./TRIGGER_DEV_REALTIME.md)
- [Waits & Tokens](./TRIGGER_DEV_WAITS_TOKENS.md) — input stream waiting
- [Tasks Overview](./TRIGGER_DEV_TASKS.md) — metadata API deprecation notes
- [Master Reference](./TRIGGER_DEV_MASTER.md)
