# TRIGGER_DEV_MANAGEMENT_API

Backend/server-side API surface (not task-runtime internals).

Sources: local management docs in `trigger.dev/*-api/*.md` plus Trigger.dev `llms-full`.

---

## Authentication and Initialization

```ts
import { configure } from "@trigger.dev/sdk";

configure({
  secretKey: process.env.TRIGGER_SECRET_KEY,
  requestOptions: {
    retry: { maxAttempts: 3, factor: 2, minTimeoutInMs: 500, maxTimeoutInMs: 30_000, randomize: true },
  },
});
```

| Auth credential | Prefix | Typical use |
|---|---|---|
| Secret key | `tr_dev_`, `tr_stg_`, `tr_prod_` | Triggering + full management |
| Personal access token | `tr_pat_` | CI/deploy and supported management APIs |
| Public access token | JWT | Realtime/read scopes + selected waitpoint completion flows |

---

## Endpoint Inventory

## Tasks / Triggering

| Method | Path | SDK function | Purpose | Confidence |
|---|---|---|---|---|
| `POST` | `/api/v1/tasks/{taskIdentifier}/trigger` | `tasks.trigger()` / `myTask.trigger()` | Trigger one task run | High (verified in `llms-full`) |
| `POST` | `/api/v1/tasks/batch` | `tasks.batchTrigger()` / `myTask.batchTrigger()` | Batch trigger mixed tasks | High (local + `llms-full`) |
| `POST` | `/api/v1/tasks/{taskIdentifier}/batch` | `task.batchTrigger()` | Batch trigger one specific task | High (verified in `llms-full`) |
| `POST` | `/api/v3/batches` | `batches.create()` | Create v3 two-phase batch container | High (local + `llms-full`) |
| `POST` | `/api/v3/batches/{batchId}/items` | `batches.streamItems()` | Stream NDJSON items into v3 batch | High (local + `llms-full`) |

> ⚠️ WARNING
> Local folder includes a mislabeled trigger file; single-trigger endpoint is verified via `llms-full` and should be rechecked against live docs if strict parity is required.

## Runs

| Method | Path | SDK function | Purpose | Confidence |
|---|---|---|---|---|
| `GET` | `/api/v1/runs` | `runs.list()` | List runs with filters/pagination | High (local + `llms-full`) |
| `GET` | `/api/v3/runs/{runId}` | `runs.retrieve()` | Retrieve full run details | High (local + `llms-full`) |
| `POST` | `/api/v1/runs/{runId}/replay` | `runs.replay()` | Create new run from existing run config | High (local + `llms-full`) |
| `POST` | `/api/v2/runs/{runId}/cancel` | `runs.cancel()` | Cancel in-progress run | High (local + `llms-full`) |
| `POST` | `/api/v1/runs/{runId}/reschedule` | `runs.reschedule()` | Change delayed run schedule | High (local + `llms-full`) |
| `PUT` | `/api/v1/runs/{runId}/metadata` | `runs.updateMetadata()` / metadata API | Replace/update run metadata | High (local + `llms-full`) |
| `POST` | `/api/v1/runs/{runId}/tags` | `runs.addTags()` | Add one or more tags to a run | High (verified in `llms-full` management index) |
| `GET` | `/api/v1/runs/{runId}/events` | `runs.retrieveEvents()` | Retrieve span events | High (local + `llms-full`) |
| `GET` | `/api/v1/runs/{runId}/trace` | `runs.retrieveTrace()` | Retrieve trace tree | High (local + `llms-full`) |
| `GET` | `/api/v1/runs/{runId}/result` | `runs.retrieveResult()` | Retrieve completed result payload/error | High (local + `llms-full`) |

## Batches

| Method | Path | SDK function | Purpose | Confidence |
|---|---|---|---|---|
| `GET` | `/api/v1/batches/{batchId}` | `batches.retrieve()` | Batch status and run IDs | High (local + `llms-full`) |
| `GET` | `/api/v1/batches/{batchId}/results` | `batches.retrieveResults()` | Completed run results only | High (local + `llms-full`) |
| `POST` | `/api/v3/batches/{batchId}/items` | `batches.streamItems()` | Stream NDJSON items for 2-phase ingestion | High (local + `llms-full`) |

## Queues

| Method | Path | SDK function | Purpose | Confidence |
|---|---|---|---|---|
| `GET` | `/api/v1/queues` | `queues.list()` | List queues | High (local docs) |
| `GET` | `/api/v1/queues/{queueParam}` | `queues.retrieve()` | Retrieve by id or type/name | High (local docs) |
| `POST` | `/api/v1/queues/{queueParam}/pause` | `queues.pause()` / `queues.resume()` | Pause or resume queue | High (local docs) |
| `POST` | `/api/v1/queues/{queueParam}/concurrency/override` | `queues.overrideConcurrencyLimit()` | Temporary limit override | High (local docs) |
| `POST` | `/api/v1/queues/{queueParam}/concurrency/reset` | `queues.resetConcurrencyLimit()` | Reset to base limit | High (local docs) |

## Schedules

| Method | Path | SDK function | Purpose | Confidence |
|---|---|---|---|---|
| `GET` | `/api/v1/schedules` | `schedules.list()` | List schedules | High (local docs) |
| `POST` | `/api/v1/schedules` | `schedules.create()` | Create imperative schedule | High (local docs) |
| `GET` | `/api/v1/schedules/{schedule_id}` | `schedules.retrieve()` | Retrieve schedule | High (local docs) |
| `PUT` | `/api/v1/schedules/{schedule_id}` | `schedules.update()` | Update imperative schedule | High (local docs) |
| `DELETE` | `/api/v1/schedules/{schedule_id}` | `schedules.del()` | Delete imperative schedule | High (local docs) |
| `POST` | `/api/v1/schedules/{schedule_id}/deactivate` | `schedules.deactivate()` | Deactivate schedule | High (local docs) |
| `POST` | `/api/v1/schedules/{schedule_id}/activate` | `schedules.activate()` | Activate schedule | High (local docs) |
| `GET` | `/api/v1/timezones` | `schedules.timezones()` | Supported IANA timezone list | High (local docs) |

## Environment Variables

| Method | Path | SDK function | Purpose | Confidence |
|---|---|---|---|---|
| `GET` | `/api/v1/projects/{projectRef}/envvars/{env}` | `envvars.list()` | List env vars | High (local docs) |
| `POST` | `/api/v1/projects/{projectRef}/envvars/{env}/import` | `envvars.upload()` | Bulk import vars | High (local docs) |
| `POST` | `/api/v1/projects/{projectRef}/envvars/{env}` | `envvars.create()` | Create env var | High (local docs) |
| `GET` | `/api/v1/projects/{projectRef}/envvars/{env}/{name}` | `envvars.retrieve()` | Retrieve var value | High (local docs) |
| `PUT` | `/api/v1/projects/{projectRef}/envvars/{env}/{name}` | `envvars.update()` | Update var value | High (local docs) |
| `DELETE` | `/api/v1/projects/{projectRef}/envvars/{env}/{name}` | `envvars.del()` | Delete env var | High (local docs) |

## Deployments

| Method | Path | SDK function | Purpose | Confidence |
|---|---|---|---|---|
| `GET` | `/api/v1/deployments/{deploymentId}` | fetch / deployment client | Retrieve deployment | High (local docs) |
| `GET` | `/api/v1/deployments/latest` | fetch / deployment client | Latest unmanaged deployment | High (local docs) |
| `POST` | `/api/v1/deployments/{version}/promote` | fetch / deployment client | Promote deployment version | High (local docs) |

## Waitpoint tokens

| Method | Path | SDK function | Purpose | Confidence |
|---|---|---|---|---|
| `POST` | `/api/v1/waitpoints/tokens` | `wait.createToken()` | Create token | High (local docs) |
| `GET` | `/api/v1/waitpoints/tokens` | `wait.listTokens()` | List tokens | High (local docs) |
| `GET` | `/api/v1/waitpoints/tokens/{waitpointId}` | `wait.retrieveToken()` | Retrieve token | High (local docs) |
| `POST` | `/api/v1/waitpoints/tokens/{waitpointId}/complete` | `wait.completeToken()` | Complete token with auth | High (local docs) |
| `POST` | `/api/v1/waitpoints/tokens/{waitpointId}/callback/{callbackHash}` | callback URL | Complete token via signed URL | High (local docs) |

---

## Pagination

Auto-pagination pattern:

```ts
import { runs } from "@trigger.dev/sdk";

for await (const run of runs.list({ limit: 10 })) {
  console.log(run.id);
}
```

Manual pagination pattern:

```ts
let page = await runs.list({ limit: 10 });
while (true) {
  for (const run of page.data) console.log(run.id);
  if (!page.hasNextPage()) break;
  page = await page.getNextPage();
}
```

---

## Error Handling and Retries

```ts
import { ApiError, runs } from "@trigger.dev/sdk";

try {
  await runs.retrieve("run_123");
} catch (error) {
  if (error instanceof ApiError && [429, 500, 502, 503, 504].includes(error.status)) {
    // custom backoff/alert path
  }
}
```

Common codes:

- `400` validation/input errors,
- `401` auth failure,
- `404` missing entity,
- `422` semantic invalid request (seen on token/batch APIs),
- `429` rate-limited,
- `5xx` transient server errors.

---

## Advanced Usage

### Raw response access

```ts
const { data, response } = await runs.retrieve("run_123").withResponse();
```

### As `fetch` response

```ts
const response = await runs.retrieve("run_123").asResponse();
```

### Client request options

```ts
import { configure } from "@trigger.dev/sdk";

configure({
  secretKey: process.env.TRIGGER_SECRET_KEY,
  requestOptions: {
    retry: { maxAttempts: 2, minTimeoutInMs: 500, maxTimeoutInMs: 5_000, factor: 2, randomize: true },
  },
});
```

<!-- UNVERIFIED -->
Action item: verify custom transport hooks in `https://trigger.dev/docs/management/overview` and SDK API docs for `@trigger.dev/sdk@4.4.3+` (look for custom `fetch` / HTTP client injection points), then replace this note with exact API signature.

---

## Practical Boundaries

- Call management APIs from backend/services, not browser clients (except explicitly token-scoped realtime/waitpoint flows).
- For high-volume trigger loops, switch to batch APIs to avoid API rate-limit pressure.
- For NDJSON batch streams, enforce content-type and line-level validation.

Related:

- Core model: [`TRIGGER_DEV_MASTER.md`](./TRIGGER_DEV_MASTER.md)
- Queue behavior: [`TRIGGER_DEV_QUEUES_CONCURRENCY.md`](./TRIGGER_DEV_QUEUES_CONCURRENCY.md)

