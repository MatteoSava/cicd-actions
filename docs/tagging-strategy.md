# Tagging Strategy

This document explains the image tagging conventions used by these actions.

## Overview

The tagging strategy provides:
- **Traceability** — Every tag includes Git SHA
- **Environment clarity** — Tags indicate where images should run
- **Immutability** — Production tags are never overwritten
- **Automation** — Tags are generated based on Git context

## Tag Patterns

### Pull Request Builds

```
pr-<number>-<sha:7>
```

**Example:** `pr-42-abc1234`

- Built on every PR push
- NOT pushed by default (build verification only)
- Deleted when PR closes
- Short-lived, ephemeral

### Branch Builds (Main/Develop)

```
<branch>-<sha:7>
```

**Example:** `main-abc1234`, `develop-abc1234`

- Built on every push to branch
- Pushed to registry
- Retained based on policy (default: last 20)
- Used for dev/integration environments

### Staging

```
staging-<sha:7>
```

**Example:** `staging-abc1234`

- Created by retagging a main build
- Requires environment approval
- Retained based on policy (default: last 10)
- Used for pre-production validation

### Release

Multiple tags are applied:

| Tag | Purpose | Mutable? |
|-----|---------|----------|
| `v1.2.3-abc1234` | Immutable, exact version | No |
| `v1.2.3` | Floating patch version | Yes |
| `v1.2` | Floating minor version | Yes |
| `latest` | Most recent release | Yes |

**Use in Kubernetes:** Always use the immutable tag (`v1.2.3-abc1234`) in manifests.

## Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        Development                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PR Created                                                     │
│      │                                                          │
│      ▼                                                          │
│  pr-42-abc1234  (build only, no push)                          │
│      │                                                          │
│      ▼                                                          │
│  PR Merged                                                      │
│      │                                                          │
└──────┼──────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────────────┐
│                        Integration                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  main-abc1234  ───► Deploy to Dev Environment                  │
│      │                                                          │
│      ▼                                                          │
│  [Manual Approval]                                              │
│      │                                                          │
└──────┼──────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────────────┐
│                         Staging                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  staging-abc1234  ───► Deploy to Staging Environment           │
│      │                                                          │
│      ▼                                                          │
│  [QA Validation]                                                │
│      │                                                          │
└──────┼──────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────────────┐
│                        Production                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  git tag v1.2.3                                                │
│      │                                                          │
│      ▼                                                          │
│  v1.2.3-abc1234  ───► Deploy to Production                     │
│  v1.2.3                                                         │
│  v1.2                                                           │
│  latest                                                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Retention Policy

| Tag Pattern | Default Retention | Configurable? |
|-------------|-------------------|---------------|
| `pr-*` | 7 days after PR close | Yes |
| `main-*` | Last 20 | Yes |
| `staging-*` | Last 10 | Yes |
| `v*` (releases) | Forever | No |
| Untagged | 1 day | Yes |

## Custom Tag Templates

The `docker-metadata` action supports custom tag templates:

```yaml
- uses: MatteoSava/cicd-actions/actions/docker-metadata@v1
  with:
    image_name: ghcr.io/owner/app
    # Customize templates with placeholders:
    # {{number}} - PR number
    # {{sha}} - Short git SHA
    # {{branch}} - Branch name (sanitized)
    # {{version}} - Semver (from tag)
    # {{major}}, {{minor}}, {{patch}} - Semver parts
    tag_pr: 'pr-{{number}}-{{sha}}'
    tag_branch: '{{branch}}-{{sha}}'
    tag_release_immutable: 'v{{version}}-{{sha}}'
    tag_release_floating: 'v{{version}}'
```

## Best Practices

1. **Never use `latest` in Kubernetes manifests** — It's mutable and unpredictable
2. **Use immutable tags for deployments** — `v1.2.3-abc1234` guarantees exact code
3. **Clean up aggressively** — PR images are disposable
4. **Keep staging images short-term** — They're intermediate, not permanent
5. **Protect release tags** — Never delete or overwrite them
