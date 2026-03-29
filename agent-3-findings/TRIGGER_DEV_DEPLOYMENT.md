# Trigger.dev Deployment, Environments & CI/CD Reference

> **SDK version**: `@trigger.dev/sdk` 4.4.3
> **CLI**: `trigger.dev@latest` (or pinned in `devDependencies`)
> **Source**: [trigger.dev/docs/deployment](https://trigger.dev/docs/deployment/overview)

This document covers everything needed to deploy Trigger.dev tasks to production: environments, the CLI, `trigger.config.ts`, build extensions, CI/CD integration, and self-hosting.

---

## Table of Contents

- [Trigger.dev Deployment, Environments \& CI/CD Reference](#triggerdev-deployment-environments--cicd-reference)
  - [Table of Contents](#table-of-contents)
  - [Environments](#environments)
  - [CLI Commands](#cli-commands)
    - [dev](#dev)
    - [deploy](#deploy)
    - [login / logout / whoami](#login--logout--whoami)
    - [init](#init)
    - [promote](#promote)
    - [preview archive](#preview-archive)
  - [trigger.config.ts](#triggerconfigts)
    - [Core Options](#core-options)
    - [Build Configuration](#build-configuration)
  - [Build Extensions](#build-extensions)
    - [Built-in Extensions](#built-in-extensions)
    - [Custom Extensions](#custom-extensions)
  - [Build Process](#build-process)
  - [Deployment Versions](#deployment-versions)
  - [Atomic Deploys](#atomic-deploys)
  - [Preview Branches](#preview-branches)
  - [GitHub Actions Integration](#github-actions-integration)
    - [Production deployment](#production-deployment)
    - [Staging deployment](#staging-deployment)
    - [Preview branch deployment](#preview-branch-deployment)
    - [Creating a Personal Access Token (PAT)](#creating-a-personal-access-token-pat)
    - [Version pinning](#version-pinning)
  - [GitHub Integration (App)](#github-integration-app)
  - [Vercel Integration](#vercel-integration)
    - [Automatic env var syncing](#automatic-env-var-syncing)
    - [Atomic deploys with Vercel](#atomic-deploys-with-vercel)
  - [Deployment API](#deployment-api)
  - [Self-Hosted Deployment](#self-hosted-deployment)
    - [CLI deployment](#cli-deployment)
    - [CI/CD for self-hosted](#cicd-for-self-hosted)
    - [Self-hosted options](#self-hosted-options)
  - [Environment Variables](#environment-variables)
    - [Setting env vars](#setting-env-vars)
    - [syncEnvVars build extension](#syncenvvars-build-extension)
    - [Local development](#local-development)
  - [Troubleshooting](#troubleshooting)
    - [Dry run](#dry-run)
    - [Debug logs](#debug-logs)
    - [Common issues](#common-issues)
    - [Local builds](#local-builds)
  - [Node.js Versions](#nodejs-versions)
  - [Cross-references](#cross-references)

---

## Environments

Trigger.dev provides four environment types. Each has its own secret key, deployed versions, and configuration.

| Environment | Key Prefix | Description |
| :---------- | :--------- | :---------- |
| **DEV** | `tr_dev_` | Local development; tasks run on your machine via `trigger dev` |
| **STAGING** | `tr_stg_` | Pre-production testing; deployed via `deploy --env staging` |
| **PROD** | `tr_prod_` | Production workloads; deployed via `deploy` (default) |
| **PREVIEW** | `tr_preview_` | Isolated per-branch environments; deployed via `deploy --env preview` |

Each environment has independent:
- Deployed versions (current version)
- Concurrency limits
- Environment variables
- Schedules
- Queues

> When using preview branches, you must also set `TRIGGER_PREVIEW_BRANCH` alongside `TRIGGER_SECRET_KEY`.

---

## CLI Commands

### dev

Runs a local development server that executes tasks on your machine. Each task runs in a separate Node.js process.

```bash
npx trigger.dev@latest dev
```

| Option | Description |
| :----- | :---------- |
| `--config \| -c` | Config file name (default: `trigger.config.ts`) |
| `--project-ref \| -p` | Project ref (required if no config file) |
| `--env-file` | Load env vars for the CLI process (not tasks) |
| `--skip-update-check` | Skip package version check |
| `--analyze` | Display detailed import timings for debugging cold starts |
| `--api-url \| -a` | Override API URL (default: `https://api.trigger.dev`) |
| `--log-level \| -l` | CLI log level: `debug`, `info`, `log`, `warn`, `error`, `none` |

The dev command automatically loads environment variables from `.env`, `.env.development`, `.env.local`, `.env.development.local`, and `dev.vars` (in that order; later files override earlier ones).

**Concurrent dev scripts:**

```json
{
  "scripts": {
    "dev": "concurrently --raw --kill-others npm:dev:*",
    "dev:trigger": "npx trigger.dev@latest dev",
    "dev:next": "next dev"
  }
}
```

### deploy

Compiles, bundles, and deploys your tasks. Defaults to the `prod` environment.

```bash
npx trigger.dev@latest deploy
```

| Option | Description |
| :----- | :---------- |
| `--env \| -e` | Target environment: `prod` (default), `staging`, `preview` |
| `--branch \| -b` | Branch name for preview deploys (auto-detected from git) |
| `--dry-run` | Build without deploying; prints build path for inspection |
| `--skip-promotion` | Deploy without promoting to current version |
| `--skip-sync-env-vars` | Skip syncing environment variables |
| `--force-local-build` | Force local Docker build instead of remote build |
| `--config \| -c` | Config file name (default: `trigger.config.ts`) |
| `--project-ref \| -p` | Project ref (required if no config file) |
| `--env-file` | Load env vars for the CLI process |

> **WARNING**: `deploy` will fail in CI if any version mismatches are detected between the CLI and `@trigger.dev/*` packages. Always ensure packages are in sync.

**Deploy steps:**
1. (Optional) Update packages when running locally
2. Compile and bundle code via esbuild
3. Build a Docker image (remote or local)
4. Deploy the image to the Trigger.dev instance
5. Register tasks as a new version in the target environment

### login / logout / whoami

```bash
npx trigger.dev@latest login     # Authenticate with Trigger.dev
npx trigger.dev@latest logout    # Log out
npx trigger.dev@latest whoami    # Display current user and project details
```

All support `--profile` for managing multiple accounts and `--api-url` for self-hosted instances.

### init

Initialize a project for Trigger.dev:

```bash
npx trigger.dev@latest init
```

| Option | Description |
| :----- | :---------- |
| `--javascript` | Initialize as JavaScript (default: TypeScript) |
| `--project-ref \| -p` | Project ref to use |
| `--tag \| -t` | SDK version tag (default: `latest`) |
| `--skip-package-install` | Skip installing `@trigger.dev/sdk` |
| `--override-config` | Override existing config file |

### promote

Promote a previously deployed version to be the current version:

```bash
npx trigger.dev@latest promote 20250228.1
```

### preview archive

Archive a preview branch:

```bash
npx trigger.dev@latest preview archive
npx trigger.dev@latest preview archive --branch my-branch
```

---

## trigger.config.ts

The `trigger.config.ts` file lives at the root of your project and configures all aspects of building and deploying tasks.

### Core Options

```ts
import { defineConfig } from "@trigger.dev/sdk";

export default defineConfig({
  // Required: your project ref from the dashboard
  project: "<project-ref>",

  // Directories containing task files (default: auto-detect "trigger" dirs)
  dirs: ["./trigger"],

  // Runtime: "node" (default), "node-22", or "bun" (experimental)
  runtime: "node",

  // Log level for the `logger` API: "debug" | "info" | "log" | "warn" | "error"
  logLevel: "log",

  // Default machine size for all tasks
  defaultMachine: "small-1x",

  // Default max duration in seconds
  maxDuration: 300,

  // Default retry settings for all tasks
  retries: {
    enabledInDev: false,
    default: {
      maxAttempts: 3,
      minTimeoutInMs: 1000,
      maxTimeoutInMs: 10000,
      factor: 2,
      randomize: true,
    },
  },

  // Glob patterns to exclude from task detection
  ignorePatterns: ["**/*.test.ts"],

  // Custom tsconfig path
  tsconfig: "./tsconfig.json",

  // Keep worker processes alive between runs
  processKeepAlive: true,

  // CA certificate for self-signed certs
  extraCACerts: "./certs/ca.crt",

  // Console logging in dev
  enableConsoleLogging: true,

  // Build configuration (see below)
  build: { /* ... */ },

  // OpenTelemetry instrumentations
  telemetry: {
    instrumentations: [/* ... */],
    exporters: [/* ... */],
    logExporters: [/* ... */],
    metricExporters: [/* ... */],
  },

  // Global lifecycle hooks
  onStart: async ({ payload, ctx }) => {},
  onSuccess: async ({ payload, output, ctx }) => {},
  onFailure: async ({ payload, error, ctx }) => {},
  init: async ({ payload, ctx }) => {},
});
```

**Config options table:**

| Option | Type | Default | Description |
| :----- | :--- | :------ | :---------- |
| `project` | `string` | -- | Project ref (required) |
| `dirs` | `string[]` | Auto-detect | Task source directories |
| `runtime` | `string` | `"node"` | `"node"`, `"node-22"`, or `"bun"` |
| `logLevel` | `string` | `"log"` | Log level for `logger` API |
| `defaultMachine` | `string` | `"small-1x"` | Default machine preset |
| `maxDuration` | `number` | -- | Default max duration (seconds) |
| `retries` | `object` | See above | Default retry config |
| `tsconfig` | `string` | `"tsconfig.json"` | Custom tsconfig path |
| `processKeepAlive` | `boolean \| object` | `false` | Keep worker processes alive |
| `extraCACerts` | `string` | -- | Path to CA cert file |
| `build` | `object` | -- | Build configuration |
| `telemetry` | `object` | -- | OTel instrumentations and exporters |

### Build Configuration

```ts
build: {
  // Packages to exclude from bundling (installed at runtime instead)
  external: ["sharp", "re2", "sqlite3"],

  // Auto-detect externals from node_modules (default: true)
  autoDetectExternal: true,

  // Preserve function/class names (default: true)
  keepNames: true,

  // Minify output (default: false, experimental)
  minify: false,

  // JSX settings (esbuild automatic runtime by default)
  jsx: {
    factory: "React.createElement",
    fragment: "React.Fragment",
    automatic: true,
  },

  // Custom import conditions
  conditions: ["react-server"],

  // Build extensions (see next section)
  extensions: [/* ... */],
}
```

> Packages that use native binaries or WebAssembly **must** be added to `external` since they cannot be bundled by esbuild. Examples: `sharp`, `re2`, `sqlite3`, WASM packages.

> The `build` configuration is stripped from the final bundle. Imports used only within `build` config are tree-shaken out.

---

## Build Extensions

Build extensions hook into the build system to customize bundling and the resulting Docker image. Install `@trigger.dev/build` as a `devDependency`:

```bash
npm i -D @trigger.dev/build@latest
```

### Built-in Extensions

| Extension | Import Path | Description |
| :-------- | :---------- | :---------- |
| `prismaExtension` | `@trigger.dev/build/extensions/prisma` | Generate Prisma client and copy engine |
| `syncEnvVars` | `@trigger.dev/build/extensions/core` | Sync env vars from external services at deploy time |
| `syncVercelEnvVars` | `@trigger.dev/build/extensions/core` | Sync env vars from Vercel |
| `syncSupabaseEnvVars` | `@trigger.dev/build/extensions/core` | Sync env vars from Supabase |
| `puppeteer` | `@trigger.dev/build/extensions/puppeteer` | Install Chromium and Puppeteer dependencies |
| `playwright` | `@trigger.dev/build/extensions/playwright` | Install Playwright browsers |
| `ffmpeg` | `@trigger.dev/build/extensions/core` | Install FFmpeg (default version or 7.x static build) |
| `aptGet` | `@trigger.dev/build/extensions/core` | Install system packages via `apt-get` |
| `additionalFiles` | `@trigger.dev/build/extensions/core` | Copy extra files to the build directory |
| `additionalPackages` | `@trigger.dev/build/extensions/core` | Install additional npm packages at deploy time |
| `pythonExtension` | `@trigger.dev/build/extensions/python` | Execute Python scripts in tasks <!-- UNVERIFIED --> Action item: verify at `https://trigger.dev/docs/cli-deploy`. |
| `esbuildPlugin` | `@trigger.dev/build/extensions` | Add custom esbuild plugins |
| `emitDecoratorMetadata` | `@trigger.dev/build/extensions/typescript` | Enable TypeScript decorator metadata |
| `audioWaveform` | `@trigger.dev/build/extensions/audioWaveform` | Install Audio Waveform |
| `lightpanda` | `@trigger.dev/build/extensions/lightpanda` | Install Lightpanda browser |

**Examples:**

```ts
import { defineConfig } from "@trigger.dev/sdk";
import { prismaExtension } from "@trigger.dev/build/extensions/prisma";
import { syncEnvVars } from "@trigger.dev/build/extensions/core";
import { puppeteer } from "@trigger.dev/build/extensions/puppeteer";
import { ffmpeg } from "@trigger.dev/build/extensions/core";
import { aptGet } from "@trigger.dev/build/extensions/core";
import { additionalFiles } from "@trigger.dev/build/extensions/core";
import { additionalPackages } from "@trigger.dev/build/extensions/core";
import { esbuildPlugin } from "@trigger.dev/build/extensions";

export default defineConfig({
  project: "<project-ref>",
  dirs: ["./trigger"],
  build: {
    extensions: [
      // Prisma -- generate client and copy query engine
      prismaExtension({
        schema: "./prisma/schema.prisma",
      }),

      // Sync env vars from a third-party service
      syncEnvVars(async (ctx) => {
        // ctx.environment: "dev" | "staging" | "prod" | "preview"
        // ctx.branch: string | undefined (preview only)
        return await fetchEnvVars(ctx.environment, ctx.branch);
      }),

      // Install Puppeteer
      puppeteer(),

      // Install FFmpeg 7.x
      ffmpeg({ version: "7" }),

      // Install system packages
      aptGet({ packages: ["imagemagick", "ghostscript"] }),

      // Copy additional files
      additionalFiles({ files: ["./assets/**", "./templates/*.html"] }),

      // Install extra npm packages (e.g. CLI tools)
      additionalPackages({ packages: ["wrangler@1.19.0"] }),

      // Add a custom esbuild plugin
      esbuildPlugin(myPlugin(), {
        placement: "last",
        target: "deploy",
      }),
    ],
  },
});
```

### Custom Extensions

Create your own extension by implementing the `BuildExtension` interface:

```ts
import { defineConfig } from "@trigger.dev/sdk";
import type { BuildExtension } from "@trigger.dev/build";

function myExtension(): BuildExtension {
  return {
    name: "my-extension",

    // Add packages to the externals list
    externalsForTarget: async (target) => {
      return ["my-native-dep"];
    },

    // Runs before the build starts
    onBuildStart: async (context) => {
      // context.target: "dev" | "deploy"
      // context.registerPlugin(esbuildPlugin)
      // context.resolvePath(relativePath)
      console.log(`Building for ${context.target}`);
    },

    // Runs after the build completes
    onBuildComplete: async (context, manifest) => {
      context.addLayer({
        id: "my-layer",
        // Additional npm dependencies
        dependencies: { "my-dep": "^1.0.0" },
        // Commands to run in the Docker build stage
        commands: ["echo 'Hello from build'"],
        // System packages to install
        image: { pkgs: ["libpng-dev"] },
        // Build-time env vars
        build: { env: { MY_VAR: "value" } },
        // Runtime env vars (synced to Trigger.dev project)
        deploy: { env: { RUNTIME_VAR: "value" } },
      });
    },
  };
}

export default defineConfig({
  project: "<project-ref>",
  build: {
    extensions: [myExtension()],
  },
});
```

---

## Build Process

The deployment build process follows these steps:

1. **Bundle** -- esbuild compiles and bundles your task code, respecting `external` and `conditions` settings
2. **Generate Dockerfile** -- A multi-stage Dockerfile is created with system packages, dependencies, and the bundled code
3. **Build Docker image** -- Either remotely (default on Trigger.dev Cloud) or locally (`--force-local-build`)
4. **Push and deploy** -- The image is pushed to the registry and registered as a new version

Use `--dry-run` to inspect the build output without deploying:

```bash
npx trigger.dev@latest deploy --dry-run
# Output: .trigger/tmp/<build-dir>
```

---

## Deployment Versions

Every deployment creates a new version with a timestamp-based identifier, e.g. `20250228.1`.

- **Current version**: The version that new runs execute against. Only one current version per environment.
- **Version locking**: When a run starts, it is locked to the current version. Retries use the same version.
- **Explicit version**: You can specify a version when triggering:

```ts
await myTask.trigger({ foo: "bar" }, { version: "20250228.1" });
```

Or globally via environment variable:

```bash
TRIGGER_VERSION=20250228.1
```

**Child task version locking:**

| Function | Child Version | Locked? |
| :------- | :------------ | :------ |
| `trigger()` | Current | No |
| `batchTrigger()` | Current | No |
| `triggerAndWait()` | Parent's version | Yes |
| `batchTriggerAndWait()` | Parent's version | Yes |

> When you Replay a run in the dashboard, the new run uses the **current** version, not the version of the original run.

---

## Atomic Deploys

Atomic deploys let you synchronize your application deployment with a specific task version:

1. Deploy tasks with `--skip-promotion` (creates a version without making it current)
2. Capture the version from CLI output
3. Deploy your application with `TRIGGER_VERSION` set to the captured version
4. Promote the task version

```bash
# Step 1: Deploy without promotion
npx trigger.dev@latest deploy --skip-promotion
# Output includes: deploymentVersion=20250228.1

# Step 2: Deploy your app with the version pinned
TRIGGER_VERSION=20250228.1 npx vercel --prod

# Step 3: Promote the task version
npx trigger.dev@latest promote 20250228.1
```

---

## Preview Branches

Preview branches create isolated environments per git branch with independent versions, concurrency limits, env vars, and schedules.

**Workflow:**
1. Create a preview branch (auto from git or manual)
2. Deploy to it one or more times
3. Trigger runs using `TRIGGER_SECRET_KEY` (preview key) + `TRIGGER_PREVIEW_BRANCH`
4. Archive when done

```bash
# Deploy to preview (auto-detects git branch)
npx trigger.dev@latest deploy --env preview

# Explicitly specify branch
npx trigger.dev@latest deploy --env preview --branch feature-xyz

# Archive a preview branch
npx trigger.dev@latest preview archive
npx trigger.dev@latest preview archive --branch feature-xyz
```

**Triggering runs against a preview branch:**

```bash
TRIGGER_SECRET_KEY="tr_preview_1234567890"
TRIGGER_PREVIEW_BRANCH="feature-xyz"
```

Or via SDK configuration:

```ts
import { configure } from "@trigger.dev/sdk";

configure({
  secretKey: "tr_preview_1234567890",
  previewBranch: "feature-xyz",
});
```

**Active branch limits (Trigger.dev Cloud):**

| Plan | Active Preview Branches |
| :--- | :---------------------: |
| Free | 0 |
| Hobby | 5 |
| Pro | 20 (then paid) |

**Env vars for preview branches**: Set variables at the "Preview" level (applies to all branches) or per-branch (overrides preview-level). Use `syncEnvVars()` or `syncVercelEnvVars()` for automatic syncing.

---

## GitHub Actions Integration

### Production deployment

```yaml
# .github/workflows/release-trigger-prod.yml
name: Deploy to Trigger.dev (prod)

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Use Node.js 20.x
        uses: actions/setup-node@v4
        with:
          node-version: "20.x"
      - name: Install dependencies
        run: npm install
      - name: Deploy Trigger.dev
        env:
          TRIGGER_ACCESS_TOKEN: ${{ secrets.TRIGGER_ACCESS_TOKEN }}
        run: npx trigger.dev@latest deploy
```

### Staging deployment

```yaml
# .github/workflows/release-trigger-staging.yml
name: Deploy to Trigger.dev (staging)

on:
  workflow_dispatch:

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "20.x"
      - run: npm install
      - name: Deploy Trigger.dev
        env:
          TRIGGER_ACCESS_TOKEN: ${{ secrets.TRIGGER_ACCESS_TOKEN }}
        run: npx trigger.dev@latest deploy --env staging
```

### Preview branch deployment

```yaml
# .github/workflows/trigger-preview-branches.yml
name: Deploy to Trigger.dev (preview branches)

on:
  pull_request:
    types: [opened, synchronize, reopened, closed]

jobs:
  deploy-preview:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "20.x"
      - run: npm install
      - name: Deploy preview branch
        run: npx trigger.dev@latest deploy --env preview
        env:
          TRIGGER_ACCESS_TOKEN: ${{ secrets.TRIGGER_ACCESS_TOKEN }}
```

> **WARNING**: Include `closed` in the `pull_request.types` list. Without it, preview branches will not be archived when PRs are merged, and you may hit active branch limits.

### Creating a Personal Access Token (PAT)

1. Go to your [Trigger.dev account settings](https://cloud.trigger.dev/account/tokens) and create a new token (starts with `tr_pat_`)
2. In your GitHub repo: Settings > Secrets and variables > Actions > New repository secret
3. Name: `TRIGGER_ACCESS_TOKEN`, Value: your PAT

### Version pinning

Add `trigger.dev` to `devDependencies` and use scripts to ensure version consistency:

```json
{
  "scripts": {
    "deploy:trigger-prod": "trigger deploy",
    "deploy:trigger-staging": "trigger deploy --env staging"
  },
  "devDependencies": {
    "trigger.dev": "4.4.3"
  }
}
```

```yaml
- name: Deploy Trigger.dev
  env:
    TRIGGER_ACCESS_TOKEN: ${{ secrets.TRIGGER_ACCESS_TOKEN }}
  run: npm run deploy:trigger-prod
```

---

## GitHub Integration (App)

Trigger.dev offers a GitHub app integration for automatic deployments without custom workflows:

1. Install the GitHub app from your project settings
2. Connect a repository
3. Configure branch tracking:
   - **Production**: branch that deploys to prod (e.g. `main`)
   - **Staging**: branch that deploys to staging
   - **Preview**: toggle for PR-based preview deployments
4. (Optional) customize build settings: config file path, install command, pre-build command

Every push to a tracked branch triggers an automatic deployment.

---

## Vercel Integration

### Automatic env var syncing

```ts
import { defineConfig } from "@trigger.dev/sdk";
import { syncVercelEnvVars } from "@trigger.dev/build/extensions/core";

export default defineConfig({
  project: "<project-ref>",
  build: {
    extensions: [syncVercelEnvVars()],
  },
});
```

Requires `VERCEL_ACCESS_TOKEN`, `VERCEL_PROJECT_ID`, and `VERCEL_TEAM_ID` environment variables.

### Atomic deploys with Vercel

For coordinated deployments, combine `--skip-promotion` with Vercel's deployment promotion:

```yaml
steps:
  - name: Deploy Trigger.dev
    id: deploy-trigger
    env:
      TRIGGER_ACCESS_TOKEN: ${{ secrets.TRIGGER_ACCESS_TOKEN }}
    run: npx trigger.dev@latest deploy --skip-promotion

  - name: Deploy to Vercel
    run: npx vercel --yes --prod -e TRIGGER_VERSION=$TRIGGER_VERSION --token $VERCEL_TOKEN
    env:
      VERCEL_TOKEN: ${{ secrets.VERCEL_TOKEN }}
      TRIGGER_VERSION: ${{ steps.deploy-trigger.outputs.deploymentVersion }}

  - name: Promote Trigger.dev Version
    run: npx trigger.dev@latest promote $TRIGGER_VERSION
    env:
      TRIGGER_ACCESS_TOKEN: ${{ secrets.TRIGGER_ACCESS_TOKEN }}
      TRIGGER_VERSION: ${{ steps.deploy-trigger.outputs.deploymentVersion }}
```

---

## Deployment API

The REST API provides three deployment endpoints:

| Method | Path | Description |
| :----- | :--- | :---------- |
| GET | `/api/v1/deployments/{deploymentId}` | Get deployment by ID |
| GET | `/api/v1/deployments/latest` | Get the latest deployment |
| POST | `/api/v1/deployments/{version}/promote` | Promote a version to current |

**Deployment statuses:**

| Status | Description |
| :----- | :---------- |
| `PENDING` | Deployment created, waiting to start |
| `INSTALLING` | Installing dependencies |
| `BUILDING` | Building the image |
| `DEPLOYING` | Pushing the image and registering tasks |
| `DEPLOYED` | Successfully deployed |
| `FAILED` | Deployment failed |
| `CANCELED` | Deployment was canceled |
| `TIMED_OUT` | Deployment exceeded time limit |

**Deployment object fields:**

| Field | Type | Description |
| :---- | :--- | :---------- |
| `id` | `string` | Deployment ID |
| `status` | `string` | Current status |
| `version` | `string` | Version string (e.g. `"20250228.1"`) |
| `shortCode` | `string` | Short code identifier |
| `contentHash` | `string` | Hash of deployment content |
| `imageReference` | `string \| null` | Docker image reference |
| `imagePlatform` | `string` | Image platform |
| `errorData` | `object \| null` | Error details (if failed) |
| `worker` | `object \| null` | Worker info with task list |

```ts
// Get latest deployment
const response = await fetch(
  "https://api.trigger.dev/api/v1/deployments/latest",
  {
    headers: { Authorization: `Bearer ${process.env.TRIGGER_SECRET_KEY}` },
  }
);
const deployment = await response.json();
console.log(`Version: ${deployment.version}, Status: ${deployment.status}`);

// Promote a specific version
await fetch(
  "https://api.trigger.dev/api/v1/deployments/20250228.1/promote",
  {
    method: "POST",
    headers: {
      Authorization: `Bearer ${process.env.TRIGGER_SECRET_KEY}`,
      "Content-Type": "application/json",
    },
  }
);
```

> The promote endpoint uses the version **string** (e.g. `"20250228.1"`) as the path parameter, not the deployment ID.

---

## Self-Hosted Deployment

When self-hosting, builds are performed locally by default. Set `TRIGGER_API_URL` to point to your instance.

### CLI deployment

```bash
# Set your self-hosted API URL
export TRIGGER_API_URL="https://trigger.example.com"

# Login
npx trigger.dev@latest login

# Deploy
npx trigger.dev@latest deploy
```

### CI/CD for self-hosted

Requires Docker Buildx in your CI environment:

```yaml
name: Deploy to Trigger.dev (self-hosted)

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "20.x"
      - run: npm install

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
        with:
          version: latest

      - name: Login to DockerHub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Deploy Trigger.dev
        env:
          TRIGGER_ACCESS_TOKEN: ${{ secrets.TRIGGER_ACCESS_TOKEN }}
          TRIGGER_API_URL: ${{ secrets.TRIGGER_API_URL }}
        run: npx trigger.dev@latest deploy
```

### Self-hosted options

<!-- UNVERIFIED -->
Action item: verify self-hosting options at `https://trigger.dev/docs/self-hosting/overview`.

Trigger.dev can be self-hosted via:
- **Docker Compose** -- for single-node setups
- **Kubernetes / Helm** -- for production-grade orchestration

Refer to the [self-hosting documentation](https://trigger.dev/docs/self-hosting/overview) for detailed setup instructions.

---

## Environment Variables

Environment variables for deployed tasks are managed separately from your local `.env` files.

### Setting env vars

1. **Dashboard**: Project > Environment Variables > New variable (set per environment)
2. **SDK**: Use `envvars.create()`, `envvars.upload()`, `envvars.update()`, `envvars.del()`
3. **Build extensions**: `syncEnvVars()`, `syncVercelEnvVars()`, `syncSupabaseEnvVars()`

### syncEnvVars build extension

Automatically sync env vars from external services on every deploy:

```ts
import { defineConfig } from "@trigger.dev/sdk";
import { syncEnvVars } from "@trigger.dev/build/extensions/core";

export default defineConfig({
  project: "<project-ref>",
  build: {
    extensions: [
      syncEnvVars(async (ctx) => {
        // ctx.environment: "dev" | "staging" | "prod" | "preview"
        // ctx.branch: string | undefined (set for preview deploys)
        const secrets = await fetchFromVault(ctx.environment);
        return secrets.map((s) => ({ name: s.key, value: s.value }));
      }),
    ],
  },
});
```

> `syncEnvVars` runs during **deploy** only. It has no effect during `dev`. For local development, use `.env` files or your secret service's CLI.

### Local development

The dev CLI auto-loads from these files in order (later overrides earlier):
- `.env`
- `.env.development`
- `.env.local`
- `.env.development.local`
- `dev.vars`

---

## Troubleshooting

### Dry run

Inspect what will be built and deployed without actually deploying:

```bash
npx trigger.dev@latest deploy --dry-run
```

### Debug logs

```bash
npx trigger.dev@latest deploy --log-level debug
```

> Do NOT share debug logs publicly -- they may contain private information.

### Common issues

| Error | Cause | Fix |
| :---- | :---- | :-- |
| `Failed to build project image` | Remote build provider issue or broken dependency | Try `--force-local-build` |
| `No loader is configured for ".node" files` | Native package not externalized | Add package to `build.external` |
| `Cannot find matching keyid` | Node.js v22 + corepack incompatibility | Use Node.js v20 or `npm i -g corepack@latest` |
| Version mismatch on deploy | CLI and SDK package versions differ | Run `npx trigger.dev@latest update` |

### Local builds

Requires Docker and Docker Buildx installed locally. Useful as a fallback when the remote build provider has issues:

```bash
npx trigger.dev@latest deploy --force-local-build
```

---

## Node.js Versions

| SDK Version | Default Runtime | Available Runtimes |
| :---------- | :-------------- | :----------------- |
| v3 | Node.js 21.7.3 | -- |
| v4 | Node.js 21.7.3 | `node` (21.7.3), `node-22` (22.16.0), `bun` (1.3.3) |

Set via the `runtime` field in `trigger.config.ts`:

```ts
export default defineConfig({
  runtime: "node-22",
  project: "<project-ref>",
});
```

---

## Cross-references

- [Management API Reference](./TRIGGER_DEV_MANAGEMENT_API.md) — deployment endpoints, env var CRUD
- [Master Reference](./TRIGGER_DEV_MASTER.md)
- [Tasks Overview](./TRIGGER_DEV_TASKS.md)
- [Errors & Retries](./TRIGGER_DEV_ERRORS_RETRIES.md)
