# TRIGGER_DEV_MANAGEMENT_API

> **SDK version:** 4.4.3 | Backend/server-side API surface (not task-runtime internals).

Sources: local management docs, Trigger.dev `llms-full`, and official docs.

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

### Preview Branch Targeting

```ts
configure({
  secretKey: process.env.TRIGGER_ACCESS_TOKEN,
  previewBranch: "feature-xyz",
});
```

### Authentication Support Matrix

| Endpoint | Secret Key | PAT |
|---|:---:|:---:|
| `task.trigger` | Yes | |
| `task.batchTrigger` | Yes | |
| `runs.list` | Yes | Yes |
| `runs.retrieve` | Yes | |
| `runs.cancel` | Yes | |
| `runs.replay` | Yes | |
| `envvars.list` | Yes | Yes |
| `envvars.retrieve` | Yes | Yes |
| `envvars.upload` | Yes | Yes |
| `envvars.create` | Yes | Yes |
| `envvars.update` | Yes | Yes |
| `envvars.del` | Yes | Yes |
| `schedules.list` | Yes | |
| `schedules.create` | Yes | |
| `schedules.retrieve` | Yes | |
| `schedules.update` | Yes | |
| `schedules.activate` | Yes | |
| `schedules.deactivate` | Yes | |
| `schedules.del` | Yes | |

---

## ID Prefixes

| Prefix | Resource |
|---|---|
| `run_` | Task run |
| `batch_` | Batch |
| `queue_` | Queue |
| `sched_` | Schedule |
| `waitpoint_` | Waitpoint token |
| `attempt_` | Run attempt |
| `proj_` | Project |
| `bulk_` | Bulk action |

---

## API Version Namespaces

The API uses multiple version namespaces:

| Namespace | Endpoints |
|---|---|
| `/api/v1/` | Most endpoints (runs list, replay, reschedule, metadata, tags, events, trace, result, schedules, queues, env vars, deployments, waitpoints, query) |
| `/api/v2/` | `POST /api/v2/runs/{runId}/cancel` |
| `/api/v3/` | `GET /api/v3/runs/{runId}` (retrieve run), `POST /api/v3/batches` (create batch), `POST /api/v3/batches/{batchId}/items` (stream items) |

> The SDK abstracts version differences — you do not need to specify versions when using the TypeScript SDK.

---

## Endpoint Inventory

### Tasks / Triggering

| Method | Path | SDK function | Purpose | Confidence |
|---|---|---|---|---|
| `POST` | `/api/v1/tasks/{taskIdentifier}/trigger` | `tasks.trigger()` / `myTask.trigger()` | Trigger one task run | High (verified in `llms-full`) |
| `POST` | `/api/v1/tasks/batch` | `tasks.batchTrigger()` / `myTask.batchTrigger()` | Batch trigger mixed tasks | High (local + `llms-full`) |
| `POST` | `/api/v1/tasks/{taskIdentifier}/batch` | `task.batchTrigger()` | Batch trigger one specific task | High (verified in `llms-full`) |
| `POST` | `/api/v3/batches` | `batches.create()` | Create v3 two-phase batch container | High (local + `llms-full`) |
| `POST` | `/api/v3/batches/{batchId}/items` | `batches.streamItems()` | Stream NDJSON items into v3 batch | High (local + `llms-full`) |

> ⚠️ WARNING
> Local folder includes a mislabeled trigger file; single-trigger endpoint is verified via `llms-full` and should be rechecked against live docs if strict parity is required.

### Runs

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

**List runs — filter options:**

| Parameter | Type | Description |
|---|---|---|
| `status` | `string[]` | Filter by status (see Run Statuses section) |
| `taskIdentifier` | `string[]` | Filter by task ID |
| `version` | `string[]` | Filter by deploy version |
| `from` / `to` | `Date` | Filter by creation time range |
| `period` | `string` | Shorthand period, e.g. `"1d"` |
| `tag` | `string[]` | Filter by tags |
| `isTest` | `boolean` | Filter test runs |
| `schedule` | `string` | Filter by schedule ID |
| `bulkAction` | `string` | Filter by bulk action ID |

### Batches

| Method | Path | SDK function | Purpose | Confidence |
|---|---|---|---|---|
| `GET` | `/api/v1/batches/{batchId}` | `batches.retrieve()` | Batch status and run IDs | High (local + `llms-full`) |
| `GET` | `/api/v1/batches/{batchId}/results` | `batches.retrieveResults()` | Completed run results only | High (local + `llms-full`) |
| `POST` | `/api/v3/batches/{batchId}/items` | `batches.streamItems()` | Stream NDJSON items for 2-phase ingestion | High (local + `llms-full`) |

### Queues

| Method | Path | SDK function | Purpose | Confidence |
|---|---|---|---|---|
| `GET` | `/api/v1/queues` | `queues.list()` | List queues | High (local docs) |
| `GET` | `/api/v1/queues/{queueParam}` | `queues.retrieve()` | Retrieve by id or type/name | High (local docs) |
| `POST` | `/api/v1/queues/{queueParam}/pause` | `queues.pause()` / `queues.resume()` | Pause or resume queue | High (local docs) |
| `POST` | `/api/v1/queues/{queueParam}/concurrency/override` | `queues.overrideConcurrencyLimit()` | Temporary limit override | High (local docs) |
| `POST` | `/api/v1/queues/{queueParam}/concurrency/reset` | `queues.resetConcurrencyLimit()` | Reset to base limit | High (local docs) |

### Schedules

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

### Environment Variables

| Method | Path | SDK function | Purpose | Confidence |
|---|---|---|---|---|
| `GET` | `/api/v1/projects/{projectRef}/envvars/{env}` | `envvars.list()` | List env vars | High (local docs) |
| `POST` | `/api/v1/projects/{projectRef}/envvars/{env}/import` | `envvars.upload()` | Bulk import vars | High (local docs) |
| `POST` | `/api/v1/projects/{projectRef}/envvars/{env}` | `envvars.create()` | Create env var | High (local docs) |
| `GET` | `/api/v1/projects/{projectRef}/envvars/{env}/{name}` | `envvars.retrieve()` | Retrieve var value | High (local docs) |
| `PUT` | `/api/v1/projects/{projectRef}/envvars/{env}/{name}` | `envvars.update()` | Update var value | High (local docs) |
| `DELETE` | `/api/v1/projects/{projectRef}/envvars/{env}/{name}` | `envvars.del()` | Delete env var | High (local docs) |

### Deployments

| Method | Path | SDK function | Purpose | Confidence |
|---|---|---|---|---|
| `GET` | `/api/v1/deployments/{deploymentId}` | fetch / deployment client | Retrieve deployment | High (local docs) |
| `GET` | `/api/v1/deployments/latest` | fetch / deployment client | Latest unmanaged deployment | High (local docs) |
| `POST` | `/api/v1/deployments/{version}/promote` | fetch / deployment client | Promote deployment version | High (local docs) |

### Waitpoint Tokens

| Method | Path | SDK function | Purpose | Confidence |
|---|---|---|---|---|
| `POST` | `/api/v1/waitpoints/tokens` | `wait.createToken()` | Create token | High (local docs) |
| `GET` | `/api/v1/waitpoints/tokens` | `wait.listTokens()` | List tokens | High (local docs) |
| `GET` | `/api/v1/waitpoints/tokens/{waitpointId}` | `wait.retrieveToken()` | Retrieve token | High (local docs) |
| `POST` | `/api/v1/waitpoints/tokens/{waitpointId}/complete` | `wait.completeToken()` | Complete token with auth | High (local docs) |
| `POST` | `/api/v1/waitpoints/tokens/{waitpointId}/callback/{callbackHash}` | callback URL | Complete token via signed URL | High (local docs) |

### Query

| Method | Path | SDK function | Purpose | Confidence |
|---|---|---|---|---|
| `POST` | `/api/v1/query` | (REST only) | Execute a query against runs | Medium <!-- UNVERIFIED --> Action item: verify query endpoint at `https://trigger.dev/docs/management/overview`. |

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

**Cursor-based pagination parameters (runs):**

| Parameter | Type | Description |
|---|---|---|
| `page[size]` | `integer` | Items per page (10-100, default 25) |
| `page[after]` | `string` | Run ID to start after (forward) |
| `page[before]` | `string` | Run ID to start before (backward) |

**Page-based pagination (schedules, queues):**

| Parameter | Type | Description |
|---|---|---|
| `page` | `integer` | Page number |
| `perPage` | `integer` | Items per page |

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

## Run Statuses

| Status | Description |
|---|---|
| `PENDING_VERSION` | Waiting for a deployed version to become available |
| `DELAYED` | Triggered with a delay; not yet enqueued |
| `QUEUED` | Enqueued and waiting for a worker |
| `EXECUTING` | Currently running |
| `REATTEMPTING` | Failed and will be retried |
| `FROZEN` | Paused (e.g. waiting on `wait.for*`) |
| `COMPLETED` | Finished successfully |
| `CANCELED` | Canceled via API or dashboard |
| `FAILED` | Failed after exhausting retries |
| `CRASHED` | Process crashed unexpectedly |
| `INTERRUPTED` | Interrupted by the system |
| `SYSTEM_FAILURE` | Infrastructure-level failure |

---

## Practical Boundaries

- Call management APIs from backend/services, not browser clients (except explicitly token-scoped realtime/waitpoint flows).
- For high-volume trigger loops, switch to batch APIs to avoid API rate-limit pressure.
- For NDJSON batch streams, enforce content-type and line-level validation.

---

## Related

- [Master Reference](./TRIGGER_DEV_MASTER.md)
- [Queue behavior](./TRIGGER_DEV_QUEUES_CONCURRENCY.md)
- [Waits & Tokens](./TRIGGER_DEV_WAITS_TOKENS.md)
- [Deployment & CLI](./TRIGGER_DEV_DEPLOYMENT.md)
- [Realtime](./TRIGGER_DEV_REALTIME.md)
