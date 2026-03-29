# TRIGGER_DEV_DEPLOYMENT

Deployment lifecycle, environments, CLI workflow, config/build system, and CI/CD patterns.

---

## Environments

Trigger.dev supports:

- `DEV` (local task execution with remote scheduler/control plane),
- `STAGING`,
- `PROD`,
- `PREVIEW` branches (plan-dependent limits).

Each environment has distinct API keys and run/deploy context.

---

## CLI Commands

```bash
npx trigger.dev@latest login
npx trigger.dev@latest init
npx trigger.dev@latest dev
npx trigger.dev@latest deploy
npx trigger.dev@latest deploy --env staging
npx trigger.dev@latest deploy --env preview
npx trigger.dev@latest deploy --dry-run
npx trigger.dev@latest whoami
```

| Command | Purpose | Common flags |
|---|---|---|
| `login` | Authenticate CLI session | `--api-url` (self-hosted), `--profile` |
| `init` | Scaffold Trigger.dev in project | `--runtime bun` |
| `dev` | Build/run tasks locally | `--log-level`, `--profile` |
| `deploy` | Build/deploy worker version | `--env`, `--branch`, `--dry-run`, `--skip-promotion` |
| `whoami` | Show authenticated account/profile | `--api-url` |
| `update` | Upgrade Trigger.dev packages | n/a |

> ⚠️ WARNING
> `deploy --env preview` depends on branch detection or explicit branch env vars; misconfigured branch metadata can deploy to wrong preview context.

---

## `trigger.config.ts` (Deployment-critical fields)

```ts
import { defineConfig } from "@trigger.dev/sdk";
import { syncVercelEnvVars, ffmpeg, aptGet } from "@trigger.dev/build/extensions/core";

export default defineConfig({
  project: "proj_123",
  dirs: ["./trigger"],
  runtime: "node-22",
  defaultMachine: "small-1x",
  logLevel: "info",
  maxDuration: 60 * 20,
  build: {
    external: ["sharp"],
    autoDetectExternal: true,
    keepNames: true,
    minify: false,
    extensions: [syncVercelEnvVars(), ffmpeg(), aptGet({ packages: ["imagemagick"] })],
  },
});
```

| Property | Type | Required | Default | Description |
|---|---|---:|---|---|
| `project` | `string` | Yes | - | Project reference |
| `dirs` | `string[]` | Recommended | auto detect | Task code locations |
| `runtime` | `"node" \| "node-22" \| "bun"` | No | node | Runtime |
| `defaultMachine` | machine enum | No | `small-1x` | Default compute |
| `logLevel` | log level enum | No | `info` | Logger API verbosity |
| `maxDuration` | seconds | No | platform | Global run cap |
| `build.external` | `string[]` | No | none | Keep native/WASM deps unbundled |
| `build.extensions` | extension[] | No | none | Build/image customization |

---

## Build Extensions

Frequently used built-ins:

- `prismaExtension()`
- `syncEnvVars()`
- `syncVercelEnvVars()`
- `syncSupabaseEnvVars()` (new in `4.4.3`)
- `additionalFiles({ files })`
- `additionalPackages({ packages })`
- `puppeteer()`
- `ffmpeg()`
- `aptGet({ packages })`
- `esbuildPlugin(...)`
- `python()`

<!-- UNVERIFIED -->
Action item: verify the exact extension package/import path for Python support in `https://trigger.dev/docs/config/extensions` (current v4.4.x docs), then update this list with the canonical import statement.

```ts
import { defineConfig } from "@trigger.dev/sdk";
import { additionalFiles, additionalPackages, syncSupabaseEnvVars } from "@trigger.dev/build/extensions/core";

export default defineConfig({
  build: {
    extensions: [
      additionalFiles({ files: ["./assets/**"] }),
      additionalPackages({ packages: ["wrangler@1.19.0"] }),
      syncSupabaseEnvVars(),
    ],
  },
});
```

---

## Build Process

- `dev`: bundle task files locally with Trigger build system (`esbuild`-based), execute locally.
- `deploy`: bundle + generate container build (Containerfile/Docker image) + publish worker deployment.
- `deploy --dry-run`: inspect bundle/container output without pushing deploy.

---

## GitHub Actions / CI

Required env vars in CI:

- `TRIGGER_ACCESS_TOKEN` (PAT),
- `TRIGGER_API_URL` for self-hosted.

```yaml
name: Deploy Trigger.dev
on:
  push:
    branches: [main]
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npx trigger.dev@latest deploy --env prod
        env:
          TRIGGER_ACCESS_TOKEN: ${{ secrets.TRIGGER_ACCESS_TOKEN }}
```

For preview branches, include PR `closed` events so branch environments get archived automatically.

---

## Vercel Integration

Integration paths:

- sync env vars from Vercel into Trigger via `syncVercelEnvVars()`,
- coordinate atomic deploy strategy using Trigger version + Vercel promotion flow.

Recent updates:

- dedicated Vercel integration entry in changelog (`2026-03-10`),
- env sync + atomic deploy guidance.

---

## Atomic Deploys and Versioning

- New deploy versions do not mutate currently running runs.
- Existing runs continue on original deployed version.
- Promote deployment/version when app and tasks must move together.

Pattern:

1. deploy Trigger tasks,
2. capture task/deploy version,
3. deploy app with matching `TRIGGER_VERSION`,
4. promote app deployment once task deploy is healthy.

---

## Preview Branches

- Isolated env per branch (Hobby/Pro plans).
- Requires branch metadata + preview env enabled.
- Limits are plan-based (e.g. Hobby ~5 active, Pro ~20+ per docs snapshot).

Env variables:

- `TRIGGER_PREVIEW_BRANCH`
- `TRIGGER_SECRET_KEY` (preview-scoped)

---

## Deployment Limits Snapshot

From Trigger limits docs snapshot:

- API: `1,500 req/min`
- Batch size: `1,000` (SDK `4.3.1+`)
- Max run TTL: `14d` (cloud cap)
- Queue depth per queue (plan-dependent): Dev `500/500/5000` (Free/Hobby/Pro), Staging/Prod higher.
- Realtime connection caps and preview branch caps are plan-based.

See full limits table in [`TRIGGER_DEV_MASTER.md`](./TRIGGER_DEV_MASTER.md).

---

## Operational Checklist

- Pin SDK/build packages consistently in monorepos.
- Keep `dirs` explicit in config.
- Put native/WASM packages in `build.external`.
- Use `deploy --dry-run` for build-debug loops.
- In CI, use PAT (`TRIGGER_ACCESS_TOKEN`) not interactive login.

Related:

- Task/runtime controls: [`TRIGGER_DEV_TASKS.md`](./TRIGGER_DEV_TASKS.md)
- Management deploy endpoints: [`TRIGGER_DEV_MANAGEMENT_API.md`](./TRIGGER_DEV_MANAGEMENT_API.md)

