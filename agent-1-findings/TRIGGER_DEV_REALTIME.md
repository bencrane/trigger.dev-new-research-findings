# TRIGGER_DEV_REALTIME

Realtime subscriptions, frontend hooks, streams, and token auth model.

---

## Realtime Overview

Realtime API supports:

- subscribing to one run (`runs.subscribeToRun`),
- subscribing by tag (`runs.subscribeToRunsWithTag`),
- subscribing to batches/runs in frontend hooks,
- streaming output and input-stream interactions.

Standard flow:

1. backend triggers task,
2. backend returns run id + public access token,
3. frontend subscribes and renders progress/streams.

---

## Public Access Tokens

Create scoped token in backend:

```ts
import { auth } from "@trigger.dev/sdk";

const token = await auth.createPublicToken({
  scopes: {
    read: { runs: ["run_123"], tags: ["tenant:acme"] },
    write: { waitpoints: ["waitpoint_abc"] },
  },
  expiresIn: "15m",
});
```

| Property | Type | Required | Default | Description |
|---|---|---:|---|---|
| `scopes` | object | Yes | none | Allowed read/write resources |
| `expiresIn` | duration | No | ~15m | Token expiry |

Also note: trigger responses commonly include `handle.publicAccessToken` for immediate use.

> ⚠️ WARNING
> Public Access Tokens default to no permissions unless scopes are explicitly granted.

---

## React Hooks

```ts
import { useRealtimeRun } from "@trigger.dev/react-hooks";

const { run, error } = useRealtimeRun(runId, {
  accessToken: publicAccessToken,
  onComplete: (finalRun) => console.log(finalRun.status),
});
```

### `useRealtimeRun`

| Property | Type | Required | Default | Description |
|---|---|---:|---|---|
| `accessToken` | `string` | Yes | - | Public access token |
| `onComplete` | function | No | none | Callback on terminal state |

### `useRealtimeRunsWithTag`

```ts
import { useRealtimeRunsWithTag } from "@trigger.dev/react-hooks";

const { runs } = useRealtimeRunsWithTag("tenant:acme", { accessToken: publicAccessToken });
```

### `useRealtimeBatch`

```ts
import { useRealtimeBatch } from "@trigger.dev/react-hooks";

const { batch } = useRealtimeBatch(batchId, { accessToken: publicAccessToken });
```

### `useWaitToken`

```ts
import { useWaitToken } from "@trigger.dev/react-hooks";

const { complete } = useWaitToken(waitpointId, { accessToken: publicAccessToken });
await complete({ approved: true });
```

### Streams hooks

- Legacy: `useRealtimeRunWithStreams`
- Recommended modern hook: `useRealtimeStream` (docs emphasize this for newer projects)
- Input stream send hook: `useInputStreamSend`

---

## Backend Subscription APIs

```ts
import { runs } from "@trigger.dev/sdk";

for await (const run of runs.subscribeToRun("run_123")) {
  console.log(run.status);
}

for await (const run of runs.subscribeToRunsWithTag("tenant:acme")) {
  console.log(run.id, run.status);
}
```

With streams:

```ts
for await (const update of runs.subscribeToRun("run_123").withStreams()) {
  // update.run + update.streams
}
```

---

## Streams and LLM-forwarding

Task-side stream usage example:

```ts
import { task, streams } from "@trigger.dev/sdk";

const output = streams.define<{ token: string }>({ id: "llm-output" });

export const generateText = task({
  id: "generate-text",
  run: async () => {
    output.append({ token: "Hello" });
    output.append({ token: " world" });
    return { done: true };
  },
});
```

Input-stream example in waits file: [`TRIGGER_DEV_WAITS_TOKENS.md`](./TRIGGER_DEV_WAITS_TOKENS.md).

For full stream coverage (output streams, input streams, `.withStreams()`, `useRealtimeStream`, `useInputStreamSend`, typing patterns, and production gotchas), see [`TRIGGER_DEV_STREAMS.md`](./TRIGGER_DEV_STREAMS.md).

---

## Run Object and Status Lifecycle

Common statuses:

- queued-ish: `PENDING_VERSION`, `QUEUED`, `DELAYED`
- active: `EXECUTING`, `REATTEMPTING`, `FROZEN`
- terminal: `COMPLETED`, `FAILED`, `CRASHED`, `INTERRUPTED`, `SYSTEM_FAILURE`, `CANCELED`

Run object includes:

- `id`, `status`, `createdAt/updatedAt`,
- `taskIdentifier`, `version`,
- `payload/output` (may be presigned URL when large),
- `attempts[]`, `metadata`, `tags`,
- optional `batchId`, `relatedRuns`.

---

## Auth and Security Patterns

- Generate PAT/secret-key operations only on trusted backend.
- Pass short-lived scoped public token to client.
- Scope tokens to minimum run/tag/waitpoint resources.
- Rotate tokens frequently (default short TTL).

> ⚠️ WARNING
> Realtime subscriptions from browser must never use `TRIGGER_SECRET_KEY`.

Related:

- Wait token completion: [`TRIGGER_DEV_WAITS_TOKENS.md`](./TRIGGER_DEV_WAITS_TOKENS.md)
- Management API auth matrix: [`TRIGGER_DEV_MANAGEMENT_API.md`](./TRIGGER_DEV_MANAGEMENT_API.md)

