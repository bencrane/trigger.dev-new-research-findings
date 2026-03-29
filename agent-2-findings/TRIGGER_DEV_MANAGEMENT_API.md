# Trigger.dev Management API Reference

> **SDK version**: `@trigger.dev/sdk` 4.4.3
> **Base URL**: `https://api.trigger.dev`
> **Source**: [trigger.dev/docs/management](https://trigger.dev/docs/management/overview)

The Trigger.dev management API is a REST/SDK interface called from your **backend** (not inside tasks). It lets you list runs, manage schedules, manipulate environment variables, control queues, and inspect deployments programmatically.

---

## Table of Contents

- [Trigger.dev Management API Reference](#triggerdev-management-api-reference)
  - [Table of Contents](#table-of-contents)
  - [Installation](#installation)
  - [Authentication](#authentication)
    - [Secret Key (environment-scoped)](#secret-key-environment-scoped)
    - [Personal Access Token (PAT)](#personal-access-token-pat)
    - [Per-request options](#per-request-options)
    - [Preview branch targeting](#preview-branch-targeting)
    - [Authentication support matrix](#authentication-support-matrix)
  - [Endpoint Inventory](#endpoint-inventory)
    - [Tasks (Trigger)](#tasks-trigger)
    - [Runs](#runs)
    - [Batches](#batches)
    - [Queues](#queues)
    - [Schedules](#schedules)
    - [Environment Variables](#environment-variables)
    - [Deployments](#deployments)
    - [Waitpoints](#waitpoints)
    - [Query](#query)
  - [Pagination](#pagination)
  - [Error Handling](#error-handling)
  - [Retry Behavior](#retry-behavior)
  - [Advanced Usage](#advanced-usage)
    - [Accessing raw HTTP responses](#accessing-raw-http-responses)
  - [ID Prefixes](#id-prefixes)
  - [API Version Namespaces](#api-version-namespaces)
  - [Run Statuses](#run-statuses)
  - [Cross-references](#cross-references)

---

## Installation

The management API ships with the same `@trigger.dev/sdk` package used for defining tasks.

```bash
npm i @trigger.dev/sdk@latest
```

```ts
import { configure, runs, schedules, envvars, queues, wait } from "@trigger.dev/sdk";
```

---

## Authentication

### Secret Key (environment-scoped)

Set `TRIGGER_SECRET_KEY` in your environment. Keys are prefixed by environment:

| Prefix       | Environment |
| :----------- | :---------- |
| `tr_dev_`    | Development |
| `tr_stg_`    | Staging     |
| `tr_prod_`   | Production  |
| `tr_preview_`| Preview     |

```ts
import { configure } from "@trigger.dev/sdk";

// Auto-detected from TRIGGER_SECRET_KEY env var; explicit call is optional
configure({
  secretKey: process.env["TRIGGER_SECRET_KEY"],
});
```

### Personal Access Token (PAT)

PATs start with `tr_pat_` and are scoped to a **user**, not an environment. When using a PAT you must supply `projectRef` (and sometimes `environment`) as additional arguments.

```ts
configure({
  secretKey: process.env["TRIGGER_ACCESS_TOKEN"], // tr_pat_...
});

// projectRef is required with PAT auth
const page = await runs.list("proj_abc123", {
  limit: 10,
  status: ["COMPLETED"],
});
```

### Per-request options

Every SDK function accepts an optional trailing `requestOptions` argument to override global config for a single call:

```ts
const run = await runs.retrieve("run_1234", {
  retry: { maxAttempts: 1 },
});
```

### Preview branch targeting

When working with preview branches, set `previewBranch` in `configure()` or use the `x-trigger-branch` HTTP header:

```ts
configure({
  secretKey: process.env["TRIGGER_ACCESS_TOKEN"],
  previewBranch: "feature-xyz",
});
```

> The `x-trigger-branch` header only applies when the `{env}` parameter is `preview`. It has no effect on `dev`, `staging`, or `prod`.

### Authentication support matrix

| Endpoint               | Secret Key | PAT |
| :--------------------- | :--------: | :-: |
| `task.trigger`         | Yes        |     |
| `task.batchTrigger`    | Yes        |     |
| `runs.list`            | Yes        | Yes |
| `runs.retrieve`        | Yes        |     |
| `runs.cancel`          | Yes        |     |
| `runs.replay`          | Yes        |     |
| `envvars.list`         | Yes        | Yes |
| `envvars.retrieve`     | Yes        | Yes |
| `envvars.upload`       | Yes        | Yes |
| `envvars.create`       | Yes        | Yes |
| `envvars.update`       | Yes        | Yes |
| `envvars.del`          | Yes        | Yes |
| `schedules.list`       | Yes        |     |
| `schedules.create`     | Yes        |     |
| `schedules.retrieve`   | Yes        |     |
| `schedules.update`     | Yes        |     |
| `schedules.activate`   | Yes        |     |
| `schedules.deactivate` | Yes        |     |
| `schedules.del`        | Yes        |     |

---

## Endpoint Inventory

### Tasks (Trigger)

| Method | Path | SDK | Description |
| :----- | :--- | :-- | :---------- |
| POST | `/api/v1/tasks/{taskId}/trigger` | `tasks.trigger()` | Trigger a single task run |
| POST | `/api/v1/tasks/{taskId}/batch` | `tasks.batchTrigger()` | Batch trigger up to 1,000 items (legacy) |
| POST | `/api/v3/batches` | Phase 1: create batch | Create a batch record (2-phase API) |
| POST | `/api/v3/batches/{batchId}/items` | Phase 2: stream items | Stream NDJSON batch items into the batch |

### Runs

| Method | Path | SDK | Description |
| :----- | :--- | :-- | :---------- |
| GET | `/api/v1/runs` | `runs.list()` | List runs with filtering and cursor pagination |
| GET | `/api/v3/runs/{runId}` | `runs.retrieve()` | Retrieve full run details (payload, output, attempts) |
| POST | `/api/v1/runs/{runId}/replay` | `runs.replay()` | Create a new run with the same payload as the original |
| POST | `/api/v2/runs/{runId}/cancel` | `runs.cancel()` | Cancel an in-progress run |
| POST | `/api/v1/runs/{runId}/reschedule` | `runs.reschedule()` | Update delay on a DELAYED run |
| PUT | `/api/v1/runs/{runId}/metadata` | `runs.updateMetadata()` | Replace the metadata object on a run |
| POST | `/api/v1/runs/{runId}/tags` | `runs.addTags()` | Add tags to an existing run <!-- UNVERIFIED --> |
| GET | `/api/v1/runs/{runId}/events` | (REST only) | Retrieve OTel span events for a run |
| GET | `/api/v1/runs/{runId}/trace` | (REST only) | Retrieve the full OTel trace tree for a run |
| GET | `/api/v1/runs/{runId}/result` | (REST only) | Retrieve the execution result of a completed run |

**List runs -- filter options:**

| Parameter | Type | Description |
| :-------- | :--- | :---------- |
| `status` | `string[]` | Filter by status (PENDING_VERSION, QUEUED, EXECUTING, REATTEMPTING, FROZEN, COMPLETED, CANCELED, FAILED, CRASHED, INTERRUPTED, SYSTEM_FAILURE) |
| `taskIdentifier` | `string[]` | Filter by task ID |
| `version` | `string[]` | Filter by deploy version |
| `from` / `to` | `Date` | Filter by creation time range |
| `period` | `string` | Shorthand period, e.g. `"1d"` |
| `tag` | `string[]` | Filter by tags |
| `isTest` | `boolean` | Filter test runs |
| `schedule` | `string` | Filter by schedule ID |
| `bulkAction` | `string` | Filter by bulk action ID |

```ts
import { runs } from "@trigger.dev/sdk";

// List with filters
const page = await runs.list({
  status: ["QUEUED", "EXECUTING"],
  taskIdentifier: ["my-task"],
  from: new Date("2024-04-01"),
  limit: 20,
});

for (const run of page.data) {
  console.log(`${run.id}: ${run.status}`);
}
```

**Retrieve a run:**

```ts
import { runs } from "@trigger.dev/sdk";

const result = await runs.retrieve("run_1234");

if (result.isSuccess) {
  console.log("Output:", result.output);
}

for (const attempt of result.attempts) {
  if (attempt.status === "FAILED") {
    console.error("Error:", attempt.error);
  }
}
```

**Reschedule a delayed run:**

```ts
import { runs } from "@trigger.dev/sdk";

await runs.reschedule("run_1234", {
  delay: new Date("2024-06-29T20:45:56.340Z"),
});
```

**Retrieve run result (REST):**

```ts
const response = await fetch(
  "https://api.trigger.dev/api/v1/runs/run_1234/result",
  {
    headers: {
      Authorization: `Bearer ${process.env.TRIGGER_SECRET_KEY}`,
    },
  }
);

const result = await response.json();
// result.ok: boolean, result.output: string, result.outputType: string
```

### Batches

The v3 batch API is a 2-phase protocol: create the batch, then stream items as NDJSON.

| Method | Path | SDK | Description |
| :----- | :--- | :-- | :---------- |
| POST | `/api/v3/batches` | Phase 1 | Create a batch record. Returns `{ id, runCount, isCached }` |
| POST | `/api/v3/batches/{batchId}/items` | Phase 2 | Stream NDJSON items. Content-Type: `application/x-ndjson` |
| GET | `/api/v1/batches/{batchId}` | (REST only) | Retrieve batch status and run IDs |
| GET | `/api/v1/batches/{batchId}/results` | (REST only) | Retrieve execution results for all completed runs |

**Create batch request body:**

| Field | Type | Required | Description |
| :---- | :--- | :------: | :---------- |
| `runCount` | `integer` | Yes | Expected number of items |
| `parentRunId` | `string` | No | Parent run ID for `batchTriggerAndWait` |
| `resumeParentOnCompletion` | `boolean` | No | Resume parent when batch completes |
| `idempotencyKey` | `string` | No | Prevents duplicate batches |

**Stream batch items NDJSON format:**

```
{"index":0,"task":"my-task","payload":{"key":"value1"}}
{"index":1,"task":"my-task","payload":{"key":"value2"}}
```

**Batch statuses:** `PENDING`, `PROCESSING`, `COMPLETED`, `PARTIAL_FAILED`, `ABORTED`

```ts
// Retrieve batch status (REST)
const response = await fetch(
  "https://api.trigger.dev/api/v1/batches/batch_1234",
  {
    headers: {
      Authorization: `Bearer ${process.env.TRIGGER_SECRET_KEY}`,
    },
  }
);

const batch = await response.json();
console.log(`Status: ${batch.status}, Runs: ${batch.runCount}`);
```

### Queues

| Method | Path | SDK | Description |
| :----- | :--- | :-- | :---------- |
| GET | `/api/v1/queues` | `queues.list()` | List all queues with pagination |
| GET | `/api/v1/queues/{queueParam}` | `queues.retrieve()` | Retrieve a single queue |
| POST | `/api/v1/queues/{queueParam}/pause` | `queues.pause()` / `queues.resume()` | Pause or resume a queue |
| POST | `/api/v1/queues/{queueParam}/concurrency/override` | `queues.overrideConcurrencyLimit()` | Override concurrency limit |
| POST | `/api/v1/queues/{queueParam}/concurrency/reset` | `queues.resetConcurrencyLimit()` | Reset to base concurrency |

Queue types: `task` (auto-created per task) and `custom` (created via `queue()`).

The `queueParam` path parameter can be interpreted in three ways using a `type` body/query parameter:
- `id` (default) -- treat as queue ID, e.g. `queue_1234`
- `task` -- treat as task ID to get the task's default queue
- `custom` -- treat as custom queue name

```ts
import { queues } from "@trigger.dev/sdk";

// List all queues
const allQueues = await queues.list();

// Pause by queue ID
await queues.pause("queue_1234");

// Pause by task name
await queues.pause({ type: "task", name: "my-task-id" });

// Resume
await queues.resume("queue_1234");

// Override concurrency limit
await queues.overrideConcurrencyLimit("queue_1234", 50);

// Reset to base value
await queues.resetConcurrencyLimit({ type: "task", name: "my-task-id" });
```

**Queue object shape:**

| Field | Type | Description |
| :---- | :--- | :---------- |
| `id` | `string` | Queue ID (`queue_` prefix) |
| `name` | `string` | Queue name (task ID or custom name) |
| `type` | `"task" \| "custom"` | Queue type |
| `running` | `integer` | Currently executing runs |
| `queued` | `integer` | Runs waiting to execute |
| `paused` | `boolean` | Whether the queue is paused |
| `concurrencyLimit` | `integer \| null` | Current concurrency limit |
| `concurrency.current` | `integer \| null` | Effective concurrency limit |
| `concurrency.base` | `integer \| null` | Base limit defined in code |
| `concurrency.override` | `integer \| null` | Override value (if set) |

> When a queue is paused, no new runs will start, but runs currently executing will continue to completion.

### Schedules

| Method | Path | SDK | Description |
| :----- | :--- | :-- | :---------- |
| GET | `/api/v1/schedules` | `schedules.list()` | List all schedules with pagination |
| POST | `/api/v1/schedules` | `schedules.create()` | Create an IMPERATIVE schedule |
| GET | `/api/v1/schedules/{scheduleId}` | `schedules.retrieve()` | Retrieve a schedule |
| PUT | `/api/v1/schedules/{scheduleId}` | `schedules.update()` | Update a schedule |
| DELETE | `/api/v1/schedules/{scheduleId}` | `schedules.del()` | Delete a schedule |
| POST | `/api/v1/schedules/{scheduleId}/deactivate` | `schedules.deactivate()` | Deactivate a schedule |
| POST | `/api/v1/schedules/{scheduleId}/activate` | `schedules.activate()` | Activate a schedule |
| GET | `/api/v1/schedules/timezones` | `schedules.timezones()` | Get list of valid IANA timezones <!-- UNVERIFIED --> |

Schedule types: `DECLARATIVE` (set via `cron` property on `schedules.task`) and `IMPERATIVE` (created via `schedules.create()`).

**Create schedule options:**

| Field | Type | Required | Description |
| :---- | :--- | :------: | :---------- |
| `task` | `string` | Yes | Task identifier |
| `cron` | `string` | Yes | Cron expression (e.g. `"0 0 * * *"`) |
| `deduplicationKey` | `string` | Yes | Prevents duplicate schedules |
| `externalId` | `string` | No | Your own identifier (e.g. user ID) |
| `timezone` | `string` | No | IANA timezone (default: `"UTC"`) |

```ts
import { schedules } from "@trigger.dev/sdk";

// Create a schedule
const schedule = await schedules.create({
  task: "my-task",
  cron: "0 0 * * *",
  deduplicationKey: "my-schedule",
  timezone: "America/New_York",
});

console.log(schedule.id); // "sched_1234"

// List all schedules
const all = await schedules.list();

// Deactivate
await schedules.deactivate(schedule.id);

// Activate
await schedules.activate(schedule.id);

// Delete
await schedules.del(schedule.id);
```

### Environment Variables

All env var endpoints are scoped to a project + environment. Valid environments: `dev`, `staging`, `prod`, `preview`.

| Method | Path | SDK | Description |
| :----- | :--- | :-- | :---------- |
| GET | `/api/v1/projects/{projectRef}/envvars/{env}` | `envvars.list()` | List all env vars |
| POST | `/api/v1/projects/{projectRef}/envvars/{env}/import` | `envvars.upload()` | Bulk import env vars |
| POST | `/api/v1/projects/{projectRef}/envvars/{env}` | `envvars.create()` | Create a single env var |
| GET | `/api/v1/projects/{projectRef}/envvars/{env}/{name}` | `envvars.retrieve()` | Retrieve a single env var |
| PUT | `/api/v1/projects/{projectRef}/envvars/{env}/{name}` | `envvars.update()` | Update a single env var |
| DELETE | `/api/v1/projects/{projectRef}/envvars/{env}/{name}` | `envvars.del()` | Delete a single env var |

```ts
import { envvars, configure } from "@trigger.dev/sdk";

// List env vars
const variables = await envvars.list("proj_yubjwjsfkxnylobaqvqz", "dev");

for (const v of variables) {
  console.log(`${v.name}: ${v.value}`);
}

// Bulk import
await envvars.upload("proj_1234", "prod", {
  variables: {
    DATABASE_URL: "postgresql://...",
    API_KEY: "sk_...",
  },
  override: true,
});

// Create a single env var
await envvars.create("proj_1234", "prod", {
  name: "MY_VAR",
  value: "my-value",
});

// Update
await envvars.update("proj_1234", "prod", "MY_VAR", {
  value: "new-value",
});

// Delete
await envvars.del("proj_1234", "prod", "MY_VAR");
```

> When called **inside a task**, `projectRef` and `env` are automatically inferred from the task context and can be omitted:
> ```ts
> const variables = await envvars.list();
> ```

### Deployments

| Method | Path | SDK | Description |
| :----- | :--- | :-- | :---------- |
| GET | `/api/v1/deployments/{deploymentId}` | (REST only) | Get deployment by ID |
| GET | `/api/v1/deployments/latest` | (REST only) | Get the latest deployment |
| POST | `/api/v1/deployments/{version}/promote` | (REST only) | Promote a version to current |

**Deployment statuses:** `PENDING`, `INSTALLING`, `BUILDING`, `DEPLOYING`, `DEPLOYED`, `FAILED`, `CANCELED`, `TIMED_OUT`

**Deployment object shape:**

| Field | Type | Description |
| :---- | :--- | :---------- |
| `id` | `string` | Deployment ID |
| `status` | `string` | Current status (see above) |
| `version` | `string` | Version string, e.g. `"20250228.1"` |
| `shortCode` | `string` | Short code for the deployment |
| `contentHash` | `string` | Hash of the deployment content |
| `imageReference` | `string \| null` | Docker image reference |
| `errorData` | `object \| null` | Error details if failed |
| `worker` | `object \| null` | Worker info (id, version, tasks[]) |

```ts
// Get a deployment
const response = await fetch(
  `https://api.trigger.dev/api/v1/deployments/${deploymentId}`,
  {
    headers: { Authorization: `Bearer ${secretKey}` },
  }
);
const deployment = await response.json();

// Get latest deployment
const latest = await fetch(
  "https://api.trigger.dev/api/v1/deployments/latest",
  {
    headers: { Authorization: `Bearer ${secretKey}` },
  }
).then((r) => r.json());

// Promote a specific version
await fetch(
  "https://api.trigger.dev/api/v1/deployments/20250228.1/promote",
  {
    method: "POST",
    headers: {
      Authorization: `Bearer ${secretKey}`,
      "Content-Type": "application/json",
    },
  }
);
```

> The `{version}` path parameter for promote uses the version **string** (e.g. `"20250228.1"`), not the deployment ID.

### Waitpoints

Waitpoints allow tasks to pause execution until an external event completes them.

| Method | Path | SDK | Description |
| :----- | :--- | :-- | :---------- |
| POST | `/api/v1/waitpoints/tokens` | `wait.createToken()` | Create a waitpoint token |
| GET | `/api/v1/waitpoints/tokens` | (REST only) | List waitpoint tokens <!-- UNVERIFIED --> |
| GET | `/api/v1/waitpoints/tokens/{waitpointId}` | (REST only) | Retrieve a waitpoint token <!-- UNVERIFIED --> |
| POST | `/api/v1/waitpoints/tokens/{waitpointId}/complete` | `wait.completeToken()` | Complete a waitpoint token <!-- UNVERIFIED --> |
| POST | `/api/v1/waitpoints/tokens/{waitpointId}/callback/{callbackHash}` | (HTTP callback) | Complete via pre-signed URL (no auth needed) |

**Create token options:**

| Field | Type | Description |
| :---- | :--- | :---------- |
| `idempotencyKey` | `string` | Prevents duplicate tokens |
| `idempotencyKeyTTL` | `string` | How long the key is valid (e.g. `"1h"`) |
| `timeout` | `string` | Auto-timeout duration (e.g. `"1h"`, `"30m"`) |
| `tags` | `string \| string[]` | Up to 10 tags, each under 128 chars |

```ts
import { wait } from "@trigger.dev/sdk";

// Create a token
const token = await wait.createToken({
  timeout: "1h",
  tags: ["user:1234567"],
});

console.log(token.id);  // "waitpoint_abc123"
console.log(token.url); // HTTP callback URL -- give to external services

// Inside a task: pause until completed
const result = await wait.forToken<{ status: string }>(token);
```

**HTTP callback** -- the `url` returned from `wait.createToken()` can be called by any external service without an API key:

```bash
curl -X POST "https://api.trigger.dev/api/v1/waitpoints/tokens/waitpoint_abc123/callback/abc123hash" \
  -H "Content-Type: application/json" \
  -d '{"status": "approved"}'
```

The entire request body is passed as output data to the waiting run.

### Query

| Method | Path | SDK | Description |
| :----- | :--- | :-- | :---------- |
| POST | `/api/v1/query` | (REST only) | Execute a query against runs <!-- UNVERIFIED --> |

---

## Pagination

All list endpoints support auto-pagination via async iteration:

```ts
import { runs } from "@trigger.dev/sdk";

// Auto-paginate through all runs
for await (const run of runs.list({ limit: 20 })) {
  console.log(run.id);
}
```

You can also paginate manually using cursor-based helpers:

```ts
let page = await runs.list({ limit: 10 });

for (const run of page.data) {
  console.log(run.id);
}

while (page.hasNextPage()) {
  page = await page.getNextPage();
  // process next page...
}
```

**Cursor-based pagination parameters (runs):**

| Parameter | Type | Description |
| :-------- | :--- | :---------- |
| `page[size]` | `integer` | Items per page (10-100, default 25) |
| `page[after]` | `string` | Run ID to start after (forward) |
| `page[before]` | `string` | Run ID to start before (backward) |

**Page-based pagination (schedules, queues):**

| Parameter | Type | Description |
| :-------- | :--- | :---------- |
| `page` | `integer` | Page number |
| `perPage` | `integer` | Items per page |

---

## Error Handling

The SDK throws `ApiError` for non-successful responses:

```ts
import { runs, ApiError } from "@trigger.dev/sdk";

try {
  const run = await runs.retrieve("run_1234");
} catch (error) {
  if (error instanceof ApiError) {
    console.error(`Status: ${error.status}`);
    console.error(`Headers: ${JSON.stringify(error.headers)}`);
    console.error(`Body: ${JSON.stringify(error.body)}`);
  }
}
```

**Common HTTP status codes:**

| Status | Meaning |
| :----- | :------ |
| 400 | Invalid request parameters |
| 401 | Invalid or missing API key |
| 404 | Resource not found |
| 422 | Validation error |
| 429 | Rate limit exceeded (see `Retry-After` header) |
| 500 | Internal server error |

---

## Retry Behavior

The SDK automatically retries on network errors and 5xx/429 responses. Default: 3 attempts with exponential backoff.

```ts
import { configure } from "@trigger.dev/sdk";

configure({
  requestOptions: {
    retry: {
      maxAttempts: 5,
      minTimeoutInMs: 1000,
      maxTimeoutInMs: 5000,
      factor: 1.8,
      randomize: true,
    },
  },
});
```

Disable retries for a specific request:

```ts
const run = await runs.retrieve("run_1234", {
  retry: { maxAttempts: 1 },
});
```

> When running inside a task, the SDK ignores customized retry options for certain functions (e.g., `task.trigger`, `task.batchTrigger`) and uses retry settings optimized for task execution.

---

## Advanced Usage

### Accessing raw HTTP responses

All SDK methods return an `ApiPromise` with helpers:

```ts
import { runs } from "@trigger.dev/sdk";

// Get both parsed data and raw response
const { data: run, response: raw } = await runs
  .retrieve("run_1234")
  .withResponse();

console.log(raw.status);
console.log(raw.headers);

// Get a standard Response object
const response = await runs.retrieve("run_1234").asResponse();
console.log(response.status);
```

---

## ID Prefixes

| Prefix | Resource |
| :----- | :------- |
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
| :-------- | :-------- |
| `/api/v1/` | Most endpoints (runs list, replay, reschedule, metadata, tags, events, trace, result, schedules, queues, env vars, deployments, waitpoints, query) |
| `/api/v2/` | `POST /api/v2/runs/{runId}/cancel` |
| `/api/v3/` | `GET /api/v3/runs/{runId}` (retrieve run), `POST /api/v3/batches` (create batch), `POST /api/v3/batches/{batchId}/items` (stream items) |

> Newer endpoints use higher version numbers but all coexist. The SDK abstracts version differences -- you do not need to specify versions when using the TypeScript SDK.

---

## Run Statuses

| Status | Description |
| :----- | :---------- |
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

## Cross-references

- [Deployment & CI/CD](./TRIGGER_DEV_DEPLOYMENT.md)
- [API docs: Overview](../01-api-docs/00-overview/00-overview.md)
- [API docs: Authentication](../01-api-docs/00-overview/01-authentication.md)
- [API docs: Runs](../01-api-docs/03-runs-api/00-list-runs.md)
- [API docs: Batches](../01-api-docs/02-batches-api/00-create-batch.md)
- [API docs: Schedules](../01-api-docs/05-schedules-api/00-list-schedules.md)
- [API docs: Env Vars](../01-api-docs/06-env-vars-api/00-list-env-vars.md)
- [API docs: Deployments](../01-api-docs/07-deployments-api/00-get-deployment.md)
- [API docs: Waitpoints](../01-api-docs/08-waitpoints-api/00-create-waitpoint-token.md)
