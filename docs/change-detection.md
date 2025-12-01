# Monorepo Change Detection

This document explains how the `detect-changes` action works for monorepo setups.

## How It Works

The action uses [dorny/paths-filter](https://github.com/dorny/paths-filter) under the hood to detect file changes between commits.

```
┌─────────────────────────────────────────────────────────────────┐
│                     Your Monorepo                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  services/                                                      │
│    ├── api/          ◄─── Changed? ───► Build api image        │
│    ├── web/          ◄─── Changed? ───► Build web image        │
│    └── worker/       ◄─── Changed? ───► Build worker image     │
│                                                                 │
│  shared/             ◄─── Changed? ───► Build ALL images       │
│                                                                 │
│  .github/            ◄─── Changed? ───► Build ALL images       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Configuration

```yaml
- uses: MatteoSava/cicd-actions/actions/detect-changes@v1
  with:
    services: |
      {
        "api": ["services/api/**", "proto/**"],
        "web": ["services/web/**"],
        "worker": ["services/worker/**"]
      }
    shared_paths: '["shared/**", ".github/**", "package.json"]'
```

### Services Object

A JSON object mapping service names to arrays of glob patterns:

```json
{
  "service-name": ["path/to/service/**", "other/path/**"]
}
```

- Service names become matrix values
- Paths support glob patterns (`**`, `*`, etc.)
- Multiple paths per service are OR'd together

### Shared Paths

A JSON array of paths that trigger ALL services:

```json
["shared/**", ".github/**", "Makefile"]
```

Use for:
- Shared libraries
- CI/CD configuration
- Root config files that affect all services

## Using with Matrix Strategy

```yaml
jobs:
  detect:
    runs-on: ubuntu-latest
    outputs:
      matrix: ${{ steps.changes.outputs.matrix }}
      any_changed: ${{ steps.changes.outputs.any_changed }}
    steps:
      - uses: actions/checkout@v4
      - uses: MatteoSava/cicd-actions/actions/detect-changes@v1
        id: changes
        with:
          services: '{"api": ["api/**"], "web": ["web/**"]}'

  build:
    needs: detect
    if: needs.detect.outputs.any_changed == 'true'
    strategy:
      matrix:
        service: ${{ fromJson(needs.detect.outputs.matrix) }}
    runs-on: ubuntu-latest
    steps:
      - run: echo "Building ${{ matrix.service }}"
```

## Outputs

| Output | Type | Description |
|--------|------|-------------|
| `matrix` | JSON array | `["api", "web"]` — changed services |
| `any_changed` | boolean | `true` if anything changed |
| `services_json` | JSON object | `{"api": true, "web": false}` |
| `changed_list` | string | `api,web` — comma-separated |

## Edge Cases

### Release Tags

On release tags (`v*`), you typically want to build everything:

```yaml
build:
  if: needs.detect.outputs.any_changed == 'true' || startsWith(github.ref, 'refs/tags/v')
```

### Force Build All

To force-build all services, you can check for specific commit messages or labels:

```yaml
- name: Check for force build
  id: force
  run: |
    if [[ "${{ github.event.head_commit.message }}" == *"[build-all]"* ]]; then
      echo "force=true" >> $GITHUB_OUTPUT
    fi

build:
  if: needs.detect.outputs.any_changed == 'true' || steps.force.outputs.force == 'true'
```

### Empty Matrix

If no services changed, the matrix will be `[]`. Use `any_changed` to skip:

```yaml
build:
  needs: detect
  if: needs.detect.outputs.any_changed == 'true'
  # This job won't run if nothing changed
```

## Common Patterns

### Microservices with Shared Proto

```yaml
services: |
  {
    "user-service": ["services/user/**", "proto/user.proto"],
    "order-service": ["services/order/**", "proto/order.proto", "proto/user.proto"],
    "gateway": ["services/gateway/**", "proto/**"]
  }
```

### Frontend + Backend with Shared Types

```yaml
services: |
  {
    "api": ["backend/**"],
    "web": ["frontend/**"]
  }
shared_paths: '["shared/types/**", "openapi.yaml"]'
```

### Multiple Dockerfiles per Service

```yaml
services: |
  {
    "app": ["src/**"],
    "app-worker": ["src/**", "worker/**"],
    "app-migrations": ["migrations/**", "src/db/**"]
  }
```
