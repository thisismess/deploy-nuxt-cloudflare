# Deploy Nuxt to Cloudflare Workers

A reusable GitHub Action to build and deploy Nuxt applications to Cloudflare Workers with versioned deployments.

## Features

- Builds Nuxt apps for Cloudflare Workers
- Versioned deployments with tags and messages
- Support for build-time environment variables
- Automatic secrets upload to Cloudflare Workers
- Configurable Node.js version
- Works with npm, pnpm, or yarn
- Build/deploy modes for parallel CI workflows

## Usage

### Basic Usage

```yaml
- uses: thisismess/deploy-nuxt-cloudflare@v1
  with:
    cloudflare-api-token: ${{ secrets.CLOUDFLARE_API_TOKEN }}
    cloudflare-account-id: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
    worker-name: ${{ secrets.CLOUDFLARE_WORKER_NAME }}
```

> **Note:** The `worker-name` must match your Cloudflare Worker name exactly. This is the name shown in the Cloudflare dashboard under Workers & Pages.

### With Build Environment Variables

```yaml
- uses: thisismess/deploy-nuxt-cloudflare@v1
  with:
    cloudflare-api-token: ${{ secrets.CLOUDFLARE_API_TOKEN }}
    cloudflare-account-id: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
    worker-name: ${{ secrets.CLOUDFLARE_WORKER_NAME }}
    working-directory: src/nuxt
    build-env: |
      {
        "NUXT_PUBLIC_API_BASE": "https://api.example.com",
        "NUXT_PUBLIC_SITE_URL": "https://example.com"
      }
```

### With Worker Secrets

```yaml
- uses: thisismess/deploy-nuxt-cloudflare@v1
  with:
    cloudflare-api-token: ${{ secrets.CLOUDFLARE_API_TOKEN }}
    cloudflare-account-id: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
    worker-name: ${{ secrets.CLOUDFLARE_WORKER_NAME }}
    secrets-json: |
      {
        "API_SECRET_KEY": "${{ secrets.API_SECRET_KEY }}",
        "DATABASE_URL": "${{ secrets.DATABASE_URL }}"
      }
```

### With Environment-Specific Bindings

Use GitHub environments to manage different Cloudflare bindings (KV, D1, R2, etc.) for production vs staging.

**1. Create environment-specific config files:**

Nuxt/Nitro generates the base wrangler config during build (with `main`, `assets`, etc.). You only need to create environment-specific files with your bindings:

```jsonc
// wrangler.production.jsonc (production bindings)
{
  "kv_namespaces": [
    { "binding": "CACHE", "id": "abc123-prod-kv-id" }
  ],
  "d1_databases": [
    { "binding": "DB", "database_id": "xyz789-prod-d1-id" }
  ],
  "vars": {
    "ENVIRONMENT": "production"
  }
}
```

```jsonc
// wrangler.staging.jsonc (staging bindings)
{
  "kv_namespaces": [
    { "binding": "CACHE", "id": "def456-staging-kv-id" }
  ],
  "d1_databases": [
    { "binding": "DB", "database_id": "uvw321-staging-d1-id" }
  ],
  "vars": {
    "ENVIRONMENT": "staging"
  }
}
```

The action merges your environment file into the Nuxt-generated `.output/server/wrangler.json` after the build step.

**2. Configure your workflow with GitHub environments:**

```yaml
name: Deploy

on:
  push:
    branches: [main, staging]

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ github.ref_name == 'main' && 'production' || 'staging' }}
    steps:
      - uses: actions/checkout@v4

      - uses: thisismess/deploy-nuxt-cloudflare@v1
        with:
          cloudflare-api-token: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          cloudflare-account-id: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
          worker-name: ${{ secrets.CLOUDFLARE_WORKER_NAME }}
          github-environment: ${{ github.ref_name == 'main' && 'production' || 'staging' }}
```

The action will deep-merge `wrangler.{github-environment}.jsonc` into the base `wrangler.jsonc` before deploying. This allows multiple branches to share the same environment config, or unique branches to have unique configs.

### Full Example (Staging + Production)

```yaml
name: Deploy

on:
  push:
    branches: [main, staging]

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ github.ref_name == 'main' && 'production' || 'staging' }}
    steps:
      - uses: actions/checkout@v4

      - uses: thisismess/deploy-nuxt-cloudflare@v1
        with:
          cloudflare-api-token: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          cloudflare-account-id: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
          worker-name: ${{ secrets.CLOUDFLARE_WORKER_NAME }}
          github-environment: ${{ github.ref_name == 'main' && 'production' || 'staging' }}
          build-env: |
            {
              "NUXT_PUBLIC_SITE_URL": "${{ vars.SITE_URL }}",
              "NUXT_PUBLIC_API_BASE": "${{ vars.API_BASE }}"
            }
```

### Separate Build and Deploy Steps

Use `mode: build` and `mode: deploy` to separate the build phase from the upload/deploy phase. This is useful when you want to:
- Run other jobs between build and deploy
- Control exactly when the upload to Cloudflare happens
- Coordinate deployments with other systems

```yaml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4

      # Build step: prepares .output directory with merged config
      - uses: thisismess/deploy-nuxt-cloudflare@v1
        with:
          cloudflare-api-token: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          cloudflare-account-id: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
          worker-name: ${{ secrets.CLOUDFLARE_WORKER_NAME }}
          github-environment: production
          mode: build  # Build only, no upload to Cloudflare

      # ... other steps (e.g., tests, other deployments) ...

      # Deploy step: uploads and deploys the prepared build
      - uses: thisismess/deploy-nuxt-cloudflare@v1
        with:
          cloudflare-api-token: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          cloudflare-account-id: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
          worker-name: ${{ secrets.CLOUDFLARE_WORKER_NAME }}
          mode: deploy  # Upload and deploy from .output
```

**Modes:**
- `full` (default): Build, upload, and deploy in one step
- `build`: Build only - prepares `.output` directory with merged config, no Cloudflare interaction
- `deploy`: Upload and deploy from `.output` directory (must run after `build` in the same job)

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `cloudflare-api-token` | Cloudflare API Token with Workers permissions | Yes | - |
| `cloudflare-account-id` | Cloudflare Account ID | Yes | - |
| `worker-name` | Name of your Cloudflare Worker (must match the name in Cloudflare dashboard) | Yes | - |
| `working-directory` | Path to Nuxt project directory | No | `.` |
| `node-version` | Node.js version to use | No | `20` |
| `build-command` | Command to build the Nuxt app | No | `npm run build` |
| `install-command` | Command to install dependencies (auto-detected from `package-manager` if not set) | No | `npm ci` |
| `package-manager` | Package manager for caching (`npm`, `pnpm`, `yarn`, or `none` to disable) | No | `npm` |
| `build-env` | JSON object of environment variables for build | No | `{}` |
| `secrets-json` | JSON object of secrets to upload to Worker | No | - |
| `deploy-tag` | Tag for the deployment | No | Short SHA |
| `deploy-message` | Message for the deployment | No | `{actor} - {sha}` |
| `github-environment` | GitHub environment name - merges `wrangler.{environment}.jsonc` into base config | No | - |
| `mode` | Action mode: `full` (build+upload+deploy), `build` (build only), or `deploy` (upload+deploy from .output) | No | `full` |

## Outputs

| Output | Description |
|--------|-------------|
| `version-id` | The uploaded Worker Version ID (set in `full` and `deploy` modes) |
| `deploy-tag` | The tag used for the deployment |

## Requirements

- Nuxt must be configured to build for Cloudflare Workers (Nitro generates the wrangler config automatically)
- To use `github-environment` for environment-specific bindings, create `wrangler.{environment}.jsonc` files
- The Cloudflare API token needs `Workers Scripts:Edit` permission

## License

MIT
