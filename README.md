# 🚀 CI/CD Actions

Reusable GitHub Actions for Docker-based CI/CD pipelines. Build once, use everywhere.

## Features

- 📦 **Monorepo Support** — Build only what changed
- 🏷️ **Smart Tagging** — Consistent tags based on Git context (PR, branch, release)
- 🔄 **Environment Promotion** — Dev → Staging → Production flow
- 🧹 **Automatic Cleanup** — Retention policies for images
- ⚡ **Caching** — GHA cache, registry cache, or local cache
- 🔒 **Secure** — No secrets in logs, minimal permissions

## Quick Start

```yaml
# .github/workflows/ci.yml
name: CI/CD

on:
  push:
    branches: [main]
    tags: ['v*']
  pull_request:

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4

      - uses: MatteoSava/cicd-actions/actions/docker-metadata@v1
        id: meta
        with:
          image_name: ghcr.io/${{ github.repository }}

      - uses: MatteoSava/cicd-actions/actions/docker-build@v1
        with:
          tags: ${{ steps.meta.outputs.tags }}
          push: ${{ github.event_name != 'pull_request' }}
          registry: ghcr.io
          registry_username: ${{ github.actor }}
          registry_password: ${{ secrets.GITHUB_TOKEN }}
```

## Actions

| Action | Description |
|--------|-------------|
| [`detect-changes`](#detect-changes) | Detect which services changed in a monorepo |
| [`docker-metadata`](#docker-metadata) | Generate Docker tags and labels based on Git context |
| [`docker-build`](#docker-build) | Build and push Docker images with caching |
| [`promote-image`](#promote-image) | Retag images for environment promotion |
| [`cleanup-images`](#cleanup-images) | Apply retention policy to container images |

---

## Actions Reference

### detect-changes

Detects which services changed in a monorepo based on file paths.

```yaml
- uses: MatteoSava/cicd-actions/actions/detect-changes@v1
  id: changes
  with:
    services: |
      {
        "api": ["backend/**", "proto/**"],
        "web": ["frontend/**"],
        "worker": ["worker/**"]
      }
    shared_paths: '["shared/**", ".github/**"]'

# Use in matrix
- if: needs.changes.outputs.any_changed == 'true'
  strategy:
    matrix:
      service: ${{ fromJson(needs.changes.outputs.matrix) }}
```

**Inputs:**

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `services` | Yes | - | JSON map of service name → paths |
| `shared_paths` | No | `[]` | Paths that trigger all services |

**Outputs:**

| Name | Description |
|------|-------------|
| `matrix` | JSON array of changed services |
| `any_changed` | `true` if any service changed |
| `services_json` | JSON object with `service: boolean` |
| `changed_list` | Comma-separated list |

---

### docker-metadata

Generates consistent Docker tags and OCI labels based on Git context.

```yaml
- uses: MatteoSava/cicd-actions/actions/docker-metadata@v1
  id: meta
  with:
    image_name: ghcr.io/owner/app
    tag_prefix: ''  # Optional prefix for monorepo

# Output tags based on context:
# PR:      pr-42-abc1234
# Branch:  main-abc1234
# Release: v1.2.3-abc1234, v1.2.3, v1.2, latest
```

**Inputs:**

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `image_name` | Yes | - | Full image name with registry |
| `tag_prefix` | No | `''` | Prefix for all tags |
| `sha_length` | No | `7` | Git SHA length |
| `tag_pr` | No | `pr-{{number}}-{{sha}}` | PR tag template |
| `tag_branch` | No | `{{branch}}-{{sha}}` | Branch tag template |
| `tag_release_immutable` | No | `v{{version}}-{{sha}}` | Release immutable tag |
| `tag_release_floating` | No | `v{{version}}` | Release floating tag |
| `tag_latest` | No | `true` | Add `latest` on release |

**Outputs:**

| Name | Description |
|------|-------------|
| `tags` | Newline-separated tags |
| `labels` | OCI labels |
| `version` | Primary version string |
| `primary_tag` | First tag (for promotions) |
| `is_release` | `true` if release tag |

---

### docker-build

Builds and pushes Docker images with caching and best practices.

```yaml
- uses: MatteoSava/cicd-actions/actions/docker-build@v1
  with:
    context: ./backend
    tags: ${{ steps.meta.outputs.tags }}
    labels: ${{ steps.meta.outputs.labels }}
    push: true
    registry: ghcr.io
    registry_username: ${{ github.actor }}
    registry_password: ${{ secrets.GITHUB_TOKEN }}
    cache_key: backend
```

**Inputs:**

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `context` | No | `.` | Build context path |
| `dockerfile` | No | `Dockerfile` | Dockerfile path |
| `tags` | Yes | - | Image tags |
| `labels` | No | `''` | OCI labels |
| `push` | No | `true` | Push to registry |
| `platforms` | No | `linux/amd64` | Target platforms |
| `build_args` | No | `''` | Build arguments |
| `registry` | No | `''` | Registry for login |
| `registry_username` | No | `''` | Registry username |
| `registry_password` | No | `''` | Registry password |
| `cache_mode` | No | `gha` | `gha`, `registry`, `local`, `none` |
| `cache_key` | No | `''` | Cache key suffix |

**Outputs:**

| Name | Description |
|------|-------------|
| `digest` | Image digest |
| `image_id` | Image ID |
| `metadata` | Build metadata JSON |

---

### promote-image

Retags images for environment promotion (dev → staging → prod).

```yaml
- uses: MatteoSava/cicd-actions/actions/promote-image@v1
  with:
    source_tag: ghcr.io/owner/app:main-abc1234
    target_tag: ghcr.io/owner/app:staging-abc1234
    registry: ghcr.io
    registry_username: ${{ github.actor }}
    registry_password: ${{ secrets.GITHUB_TOKEN }}
```

**Inputs:**

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `source_tag` | Yes | - | Full source image tag |
| `target_tag` | Yes | - | Full target image tag |
| `additional_tags` | No | `''` | Extra tags to apply |
| `registry` | No | `ghcr.io` | Registry for login |
| `registry_username` | Yes | - | Registry username |
| `registry_password` | Yes | - | Registry password |
| `fail_if_missing` | No | `false` | Fail if source not found |

**Outputs:**

| Name | Description |
|------|-------------|
| `promoted` | `true` if successful |
| `digest` | Image digest |

---

### cleanup-images

Applies retention policy to container images in GitHub Packages.

```yaml
- uses: MatteoSava/cicd-actions/actions/cleanup-images@v1
  with:
    image_name: my-app
    owner: ${{ github.repository_owner }}
    owner_type: user
    token: ${{ secrets.GITHUB_TOKEN }}
    retention_main: '20'
    retention_staging: '10'
    retention_pr_days: '7'
    dry_run: 'false'
```

**Inputs:**

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `image_name` | Yes | - | Image name (no registry prefix) |
| `owner` | Yes | - | Package owner |
| `owner_type` | No | `user` | `user` or `org` |
| `token` | Yes | - | Token with `packages:write` |
| `retention_release` | No | `true` | Keep releases forever |
| `retention_main` | No | `20` | Keep last N main images |
| `retention_staging` | No | `10` | Keep last N staging |
| `retention_pr_days` | No | `7` | Delete PRs after N days |
| `dry_run` | No | `false` | Log without deleting |

---

## Tagging Strategy

| Context | Tag Pattern | Example |
|---------|-------------|---------|
| PR | `pr-<number>-<sha>` | `pr-42-abc1234` |
| Main branch | `main-<sha>` | `main-abc1234` |
| Staging | `staging-<sha>` | `staging-abc1234` |
| Release | `v<semver>-<sha>`, `v<semver>`, `latest` | `v1.2.3-abc1234` |

## Examples

See [`examples/`](./examples/) for complete workflow files:

- [`monorepo-cicd.yml`](./examples/monorepo-cicd.yml) — Full monorepo setup
- [`single-service-cicd.yml`](./examples/single-service-cicd.yml) — Single app
- [`image-cleanup.yml`](./examples/image-cleanup.yml) — Scheduled cleanup

## Versioning

Use tags for stability:

```yaml
# Recommended: pin to major version
uses: MatteoSava/cicd-actions/actions/docker-build@v1

# Or pin to exact version
uses: MatteoSava/cicd-actions/actions/docker-build@v1.2.3

# Development only
uses: MatteoSava/cicd-actions/actions/docker-build@main
```

## License

MIT
