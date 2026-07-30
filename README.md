# k8s-build-action

Shared, reusable GitHub Actions workflow: build a Docker image, push it to GHCR,
and (optionally) trigger an ArgoCD restart + sync of a specific Deployment.

Used via `workflow_call` from other repos - it has no repo-specific assumptions
baked in, so the same file backs every calling repo/environment.

## Usage

```yaml
name: Build and Deploy

on:
  push:
    branches: [main]
  workflow_dispatch:

concurrency:
  group: deploy-${{ github.repository }}
  cancel-in-progress: true

jobs:
  call-shared-workflow:
    uses: Alwindh/k8s-build-action/.github/workflows/docker-build-deploy.yaml@main
    with:
      argo_app_name: 'my-app'
      deployment_name: 'my-app'
      image_name: 'my-app'
    secrets:
      GHCR_PAT: ${{ secrets.GHCR_PAT }}
      ARGOCD_TOKEN: ${{ secrets.ARGOCD_TOKEN }}
      ARGOCD_SERVER: ${{ secrets.ARGOCD_SERVER }}
```

Pin `@main` to a released tag (see "Versioning" below) once you're past initial setup -
a bug pushed to `main` here takes effect on every caller's very next run.

### Inputs

| Input | Default | Notes |
|---|---|---|
| `ref` | *(unset)* | Git ref to check out and build. **Required if your caller is triggered by `workflow_run`** - see below. |
| `image_name` | repo name | GHCR image name |
| `argo_app_name` | repo name | ArgoCD Application name |
| `deployment_name` | repo name | K8s Deployment to restart |
| `target_namespace` | `default` | K8s namespace |
| `build_context` | `.` | Docker build context |
| `dockerfile_path` | `./Dockerfile` | Path to Dockerfile |

### Secrets

| Secret | Required | Notes |
|---|---|---|
| `GHCR_PAT` | No | If omitted, GHCR login falls back to the caller's `GITHUB_TOKEN`. That token needs `packages: write` - either rely on this workflow's own `permissions: packages: write` (sufficient unless your org/repo restricts default token permissions further) or grant it explicitly on the calling job. |
| `ARGOCD_TOKEN` / `ARGOCD_SERVER` | No | Omit both to skip the `deploy` job entirely (build-only mode). |

### Triggering from `workflow_run` (deploy-after-CI pattern)

If your build/deploy workflow fires on another workflow's completion, e.g.:

```yaml
on:
  workflow_run:
    workflows: ['CI']
    types: [completed]
    branches: [development]

jobs:
  call-shared-workflow:
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    uses: Alwindh/k8s-build-action/.github/workflows/docker-build-deploy.yaml@main
    with:
      ref: ${{ github.event.workflow_run.head_sha }}
      argo_app_name: 'my-app-dev'
      deployment_name: 'my-app-dev'
      image_name: 'my-app-dev'
    secrets: inherit
```

**you must pass `ref: ${{ github.event.workflow_run.head_sha }}`.** Without it,
checkout resolves to the repository's default branch rather than the commit that
triggered the run - the build would silently build/deploy the wrong commit.

### Concurrency

The `deploy` job carries its own `concurrency` group (keyed off `argo_app_name` /
`target_namespace`), so two overlapping runs targeting the same ArgoCD app can't race
each other's restart/sync calls regardless of what concurrency settings (if any) the
caller workflow sets at its own level. It's intentionally set to
`cancel-in-progress: false` - once a deploy starts, it runs to completion rather than
being killed mid-way through mutating a live Deployment.

You can still (and should) set your own `concurrency` block in the calling workflow to
control the build/CI side (e.g. cancelling a stale build when a newer commit lands);
that's independent of the deploy-side guard described above.

### Branch/ref gating is the caller's responsibility

This workflow does not hardcode or assume any branch name (`main`, `development`, etc.)
- different callers legitimately deploy from different branches. Whether a given
invocation should be allowed to deploy is controlled entirely by how the *caller*
wires up `on:` / `if:` (as in the examples above). Review your trigger conditions
carefully: anyone able to run `workflow_dispatch` against your caller workflow can
choose any branch/ref to build and deploy - if you need to restrict that, add your
own `if: github.ref == 'refs/heads/...'` check in the caller job.

### Known limitation: mutable `:latest` tag

Every build pushes both `:<sha>` and `:latest`, and `deploy` restarts + syncs
whatever the Deployment's manifest currently resolves `:latest` to - it does not tell
ArgoCD which sha to run. This means the manifest repo can't tell you which commit is
actually deployed, and an ArgoCD rollback won't revert the image. If you need real
rollback/audit guarantees, have your deploy step pin an explicit image tag/digest as
an ArgoCD Application parameter instead of relying on `:latest` + restart. Not
addressed here since it depends on how each consuming repo's Application/manifests
are structured.

### Versioning

Consumers should pin to a released tag rather than `@main` once this workflow is
past initial iteration, so a change here doesn't silently alter every caller's next
run with no rollback point.
