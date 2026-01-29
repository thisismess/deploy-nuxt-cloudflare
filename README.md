# Deploy Nuxt to Cloudflare Workers

A reusable GitHub Action to build and deploy Nuxt applications to Cloudflare Workers with versioned deployments.

## Features

- Builds Nuxt apps for Cloudflare Workers
- Versioned deployments with tags and messages
- Support for build-time environment variables
- Automatic secrets upload to Cloudflare Workers
- Configurable Node.js version
- Works with npm, pnpm, or yarn

## Usage

### Basic Usage

```yaml
- uses: thisismess/deploy-nuxt-cloudflare@v1
  with:
    cloudflare-api-token: ${{ secrets.CLOUDFLARE_API_TOKEN }}
    cloudflare-account-id: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
    worker-name: my-nuxt-app
```

### With Build Environment Variables

```yaml
- uses: thisismess/deploy-nuxt-cloudflare@v1
  with:
    cloudflare-api-token: ${{ secrets.CLOUDFLARE_API_TOKEN }}
    cloudflare-account-id: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
    worker-name: my-nuxt-app
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
    worker-name: my-nuxt-app
    secrets-json: |
      {
        "API_SECRET_KEY": "${{ secrets.API_SECRET_KEY }}",
        "DATABASE_URL": "${{ secrets.DATABASE_URL }}"
      }
```

### With Environment-Specific Bindings

Use GitHub environments to manage different Cloudflare bindings (KV, D1, R2, etc.) for production vs staging.

**1. Create environment-specific config files:**

```jsonc
// wrangler.jsonc (base config)
{
  "name": "my-nuxt-app",
  "main": ".output/server/index.mjs",
  "compatibility_date": "2024-01-01",
  "assets": { "directory": ".output/public" }
}
```

```jsonc
// wrangler.production.jsonc (production overrides)
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
// wrangler.staging.jsonc (staging overrides)
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
          worker-name: my-nuxt-app
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
          worker-name: my-nuxt-app
          github-environment: ${{ github.ref_name == 'main' && 'production' || 'staging' }}
          build-env: |
            {
              "NUXT_PUBLIC_SITE_URL": "${{ vars.SITE_URL }}",
              "NUXT_PUBLIC_API_BASE": "${{ vars.API_BASE }}"
            }
```

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `cloudflare-api-token` | Cloudflare API Token with Workers permissions | Yes | - |
| `cloudflare-account-id` | Cloudflare Account ID | Yes | - |
| `worker-name` | Name of the Cloudflare Worker | Yes | - |
| `working-directory` | Path to Nuxt project directory | No | `.` |
| `node-version` | Node.js version to use | No | `20` |
| `build-command` | Command to build the Nuxt app | No | `npm run build` |
| `install-command` | Command to install dependencies | No | `npm ci` |
| `build-env` | JSON object of environment variables for build | No | `{}` |
| `secrets-json` | JSON object of secrets to upload to Worker | No | - |
| `deploy-tag` | Tag for the deployment | No | Short SHA |
| `deploy-message` | Message for the deployment | No | `{actor} - {sha}` |
| `github-environment` | GitHub environment name - merges `wrangler.{environment}.jsonc` into base config | No | - |

## Outputs

| Output | Description |
|--------|-------------|
| `version-id` | The deployed Worker Version ID |
| `deploy-tag` | The tag used for the deployment |

## Requirements

- Your Nuxt project must have a wrangler config file (`wrangler.toml`, `wrangler.jsonc`, or `wrangler.json`)
- To use `github-environment` for environment-specific bindings, you must use `wrangler.jsonc` or `wrangler.json` (not `.toml`)
- The Cloudflare API token needs `Workers Scripts:Edit` permission

## License

MIT
