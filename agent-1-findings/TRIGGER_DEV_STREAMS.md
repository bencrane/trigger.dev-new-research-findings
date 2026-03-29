# TRIGGER_DEV_STREAMS

Dedicated stream reference for Trigger.dev output streams and input streams.

---

## Stream Types

- **Output streams**: task emits incremental events/data to subscribers.
- **Input streams**: external systems/UI send typed data into a running task.

Use streams when:

- progress/result data is incremental (LLM tokens, step events),
- users must interact with in-flight runs (approve/cancel/feedback),
- you need realtime UX without polling loops.

---

## Defining Output Streams

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
| payload type | generic | No | `unknown` | Event payload shape |

---

## Defining Input Streams

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

| Property | Type | Required | Default | Description |
|---|---|---:|---|---|
| `.wait({ timeout })` | function | No | no timeout | Suspend until message/timeout |
| `.once()` | function | No | n/a | Receive next message once |
| `.on(handler)` | function | No | n/a | Register ongoing listener |
| `.send(runId, data)` | function | Yes | - | Push data into target run |

> ⚠️ WARNING
> Use `.wait()` for checkpoint-friendly suspension; avoid custom polling loops.

---

## Subscribing to Streams (Backend)

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

```ts
import { useRealtimeStream } from "@trigger.dev/react-hooks";

const { data, error } = useRealtimeStream("tokens", runId, {
  accessToken: publicAccessToken,
});
```

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

- Define stream schemas once in shared module.
- Reuse stream ids/types in task and UI to avoid drift.
- Keep payloads compact and append-only when possible.

---

## Production Gotchas

- Token scopes must allow stream read/write on targeted run.
- Excessive high-frequency emits can overwhelm UI render loops; throttle/coalesce in frontend.
- Keep stream IDs stable across deploys to reduce client breakage.

<!-- UNVERIFIED -->
Action item: verify exact hook signatures and generics for `useRealtimeStream` in `https://trigger.dev/docs/realtime/react-hooks/streams` and update this doc if argument order/options changed after `4.4.3`.

Related:

- Realtime overview: [`TRIGGER_DEV_REALTIME.md`](./TRIGGER_DEV_REALTIME.md)
- Wait/input stream waiting: [`TRIGGER_DEV_WAITS_TOKENS.md`](./TRIGGER_DEV_WAITS_TOKENS.md)

