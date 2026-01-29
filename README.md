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

### Full Example (Staging + Production)

```yaml
name: Deploy

on:
  push:
    branches: [main, staging]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set environment
        id: env
        run: |
          if [ "${{ github.ref_name }}" == "main" ]; then
            echo "worker=my-app-production" >> $GITHUB_OUTPUT
            echo "site_url=https://example.com" >> $GITHUB_OUTPUT
            echo "api_base=https://api.example.com" >> $GITHUB_OUTPUT
          else
            echo "worker=my-app-staging" >> $GITHUB_OUTPUT
            echo "site_url=https://staging.example.com" >> $GITHUB_OUTPUT
            echo "api_base=https://api.staging.example.com" >> $GITHUB_OUTPUT
          fi

      - uses: thisismess/deploy-nuxt-cloudflare@v1
        with:
          cloudflare-api-token: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          cloudflare-account-id: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
          worker-name: ${{ steps.env.outputs.worker }}
          working-directory: src/nuxt
          build-env: |
            {
              "NUXT_PUBLIC_SITE_URL": "${{ steps.env.outputs.site_url }}",
              "NUXT_PUBLIC_API_BASE": "${{ steps.env.outputs.api_base }}"
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

## Outputs

| Output | Description |
|--------|-------------|
| `version-id` | The deployed Worker Version ID |
| `deploy-tag` | The tag used for the deployment |

## Requirements

- Your Nuxt project must have a `wrangler.toml` configured for Cloudflare Workers
- The Cloudflare API token needs `Workers Scripts:Edit` permission

## License

MIT
