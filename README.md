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
| `runner` | `k8s` | Runner pool for all three jobs. Defaults to the in-cluster ARC scale set; **callers should not set this** — see "Where these jobs run". |
| `ref` | *(unset)* | Git ref to check out and build. **Required if your caller is triggered by `workflow_run`** - see below. |
| `image_name` | repo name | GHCR image name |
| `argo_app_name` | repo name | ArgoCD Application name |
| `deployment_name` | repo name | K8s Deployment to restart |
| `target_namespace` | `default` | K8s namespace |
| `build_context` | `.` | Docker build context |
| `dockerfile_path` | `./Dockerfile` | Path to Dockerfile |

## Where these jobs run

All three jobs run on a **self-hosted runner inside the cluster** (`runs-on: k8s`),
not on GitHub-hosted runners. GitHub bills no minutes for self-hosted runners, which
is the entire point: these repos are private, so every GitHub-hosted minute counts
against the free allowance.

`k8s` is an [Actions Runner Controller](https://github.com/actions/actions-runner-controller)
runner scale set. A queued job causes a throwaway pod to start, run the job, and be
deleted. Logs, checks and PR statuses are unaffected — GitHub sees an ordinary job.

### Why the runner name is the same in every repo

A self-hosted runner can only be registered to **one** repository at a time; sharing
one across repos requires a GitHub organization. So every repo gets its own scale set
— but they are all *named* `k8s`. Scale-set names only have to be unique within a
runner group, and each repo has its own. That is what lets `runs-on` be a single
constant here instead of a per-repo variable.

Registration lives in `Alwindh/k8s-manifests` under `apps/arc/`. Onboarding a repo
means adding one line there — never editing the repo itself.

### Why this is two jobs and not three

`check-secrets` used to be its own job, just to run a five-line `if` and hand a
boolean to `deploy`. A job is a whole runner. On GitHub-hosted that was a billed
minute rounded up from about five seconds of work; on the in-cluster runners it is a
pod creation, an image check, a runner registration and a teardown — while holding
one of the `maxRunners` slots that every other queued job is waiting for.

`build` already runs and already has the secrets, so it answers the question for
free. Behaviour is unchanged: omit `ARGOCD_TOKEN`/`ARGOCD_SERVER` and `deploy` is
still skipped, not run-and-does-nothing.

The general point is worth remembering when adding to this workflow: **on self-hosted
runners a job is not free, it is a pod.** Prefer a step in an existing job over a new
job whenever the work does not need its own runner, its own concurrency group, or to
fail independently. `deploy` keeps all three, which is why it stays separate.

### The escape hatch

The cluster is now a CI dependency. If it is down, mid-rebuild, or ARC is broken,
change the `runner` input's **default** in `docker-build-deploy.yaml`:

```yaml
runner:
    default: "ubuntu-latest"
```

Every caller on `@main` reverts on its next run. One line, one file, no repo settings
and no cluster access — which is exactly when you need it.

> **Note the tension with "Versioning" below.** That centrality only holds for callers
> tracking `@main`. A caller pinned to a tag or SHA keeps the old default until it is
> bumped, so it will *keep trying the cluster runner* during an outage. Pin for
> stability or track `@main` for one-switch control — you cannot have both.

### Timeouts, and why they matter more here

Both jobs set `timeout-minutes` (25 for `build`, 10 for `deploy`). GitHub's default is
**360**.

That default is reasonable when a runner is a disposable VM someone else pays for. It
is not reasonable when the runner is one of a small number of slots on a single node:
a hung job holds a slot for a working day while reporting nothing but *"in progress"*,
and every other repo's jobs queue behind it. The visible symptom is unrelated repos
mysteriously stuck in "Waiting for a runner", which sends you looking in the wrong
place entirely.

This is not hypothetical. A `RUN apk add` reaching `dl-cdn.alpinelinux.org` from inside
dind stalled a build here for 20+ minutes and would have run the full six hours.

If you add a job to this workflow, give it a timeout. On self-hosted runners an
unbounded job is a capacity leak, not just a slow build.

### If a job hangs in "Waiting for a runner"

The repo has no scale set registered. Add it to `apps/arc/repos.txt` in
`k8s-manifests` and re-run the generator; the queued job is picked up as soon as the
listener registers. Nothing needs re-triggering.

### Secrets

| Secret | Required | Notes |
|---|---|---|
| `ARGOCD_TOKEN` / `ARGOCD_SERVER` presence | — | Checked by a step in `build`, not by a job of its own. `deploy` keys its `if:` off `needs.build.outputs.has_argo_secrets`, so build-only mode still *skips* the deploy job rather than running it empty. |
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
