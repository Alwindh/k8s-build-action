# k8s-build-action

Shared CI for every app repo, in two parts:

- A reusable GitHub Actions workflow: build a Docker image, push it to GHCR, and
  (optionally) trigger an ArgoCD restart + sync of a specific Deployment. On a pull
  request it builds without publishing, as the repo's PR check.
- A Renovate preset, `renovate/default.json`, that every repo's `renovate.json`
  extends — see "Dependency updates" below.

The workflow is used via `workflow_call` from other repos - it has no repo-specific
assumptions baked in, so the same file backs every calling repo/environment.

## Usage

```yaml
name: Build and Deploy

on:
  push:
    branches: [main]
  pull_request:
  workflow_dispatch:

concurrency:
  group: deploy-${{ github.repository }}-${{ github.ref }}
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

`pull_request:` makes the same workflow the repo's PR check — see "Pull request builds"
below. Keep `github.ref` in the concurrency group if you have one: without it a PR run
and a `main` run share a group, and a Renovate PR opening would cancel a deploy halfway.

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

## Pull request builds

A caller that adds `pull_request:` to its triggers gets a PR check from this workflow:
`build` runs in full against the PR's merge commit, and nothing is published — no
tags, no `:buildcache` write — and `deploy` is skipped. Callers that do not trigger on
`pull_request` are unaffected.

That check is what Renovate's automerge waits on (next section). Most app repos have
no CI of their own, and until now their `automerge.yml` merged Renovate PRs without
building them at all, so a dependency that broke the build was found by the deploy on
`main`. Now it is found on the PR, and the PR does not merge.

The check is exactly as strong as the Dockerfile: for a front end that runs
`npm ci && npm run build` (typecheck + bundle) it catches most breakage; for a back
end whose image only runs `npm ci --omit=dev` it proves the lockfile installs and
little else. A repo that wants more adds its own `ci.yml`, as mars-tracker and
magicounter do — Renovate waits for every check, not just this one.

## Dependency updates

`renovate/default.json` is the dependency policy for every app repo, in the same way
`docker-build-deploy.yaml` is the build: each repo's `renovate.json` is a stub that
extends it, and a change here applies to every repo on its next Renovate run.

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["github>Alwindh/k8s-build-action//renovate/default"]
}
```

Anything after `extends` is that repo's own exception (an `allowedVersions` cap, a
`baseBranchPatterns` for a repo that deploys from `development`) and nothing else.

### What it does

| Update | What happens |
|---|---|
| minor / patch / digest | One grouped PR per repo per week (Mondays), merges itself once every check is green |
| npm releases | Not proposed until 3 days old (`security:minimumReleaseAgeNpm`) — the window in which compromised npm releases have historically been caught and pulled |
| lock file maintenance | Refreshes transitive deps on the 1st of the month, merges itself on green |
| major | No PR. Listed on the repo's Dependency Dashboard issue; tick it and the PR opens immediately and waits for you |
| GitHub Actions major (`uses:`) | Treated as a minor — the PR's own CI runs on the new version |
| security fix (GitHub advisory or OSV) | Immediately, outside the schedule and the release-age wait. Non-major fixes merge on green |
| anything that fails CI | Stays open and is assigned to you — the only PRs that ask for anything |

So the list at [github.com/pulls/assigned](https://github.com/pulls/assigned) is the
whole inbox: a red group PR, or a major you asked for.

### Why Renovate merges, not GitHub and not a workflow

**`platformAutomerge: false`.** GitHub's native auto-merge merges when the branch's
*required* checks pass. None of these repos has branch protection (a private repo on a
free personal plan cannot), so there are no required checks and native auto-merge has
nothing to wait for — where it is available at all, it would merge before CI finished.
Renovate's own merge looks at every check on the commit and merges only when all of
them are green, and it needs no repo setting to do it. `k8s-manifests` already worked
this way: its PRs are merged by `renovate[bot]`.

**No `automerge.yml`.** The per-repo `pascalgn/automerge-action` workflows merged
anything labelled `automerge`, with a PAT, without building it, and each was its own
copy with its own bugs: a misspelled filename that never ran, a label that never
matched, a `return` that did not stop the merge step, majors merging because the label
was on them too. Renovate's own automerge has none of those moving parts, and the
"needs a check" rule is what makes a broken update stay a PR.

**Renovate merges one PR per repo per run**, and the Mend-hosted app runs active repos
about every four hours. That is why non-majors are grouped: one weekly PR merges in one
run. Expect a few hours between "checks green" and "merged", not minutes.

**Repos without any PR check never automerge.** Renovate will not merge something it
has not seen pass (`ignoreTests` stays false). For a repo that calls this workflow,
adding `pull_request:` to its triggers is the fix.

### npm stays below 12 for lockfile updates

`constraints.npm: "<12"` pins the npm that Renovate uses to rewrite `package-lock.json`.
npm 12 refuses to download a package that declares `bundleDependencies` while it builds a
lockfile ([npm/cli#9800](https://github.com/npm/cli/issues/9800)), so in any repo with
Tailwind 4 every lockfile update failed with `EALLOWREMOTE` and an "Artifact update
problem" comment. npm 11 is also the npm that Node 24 bundles, so the lockfile comes
from the same npm major that the Node 24 images run. Remove the constraint once a fixed
npm 12 is out.

### Dormant repos

`renovate/dormant.json` is for repos that are neither deployed nor worked on. It
disables every dependency, which makes Renovate close the PRs it already opened there
and close the dashboard issue. Uninstalling the app would stop Renovate before it got
that far and leave every PR open. To revive a repo, point its stub back at `default`.

### Changing the preset

`.github/workflows/validate-renovate.yaml` validates both presets on every PR that
touches them. Every repo reads `renovate/default.json` from `main` — there is no
pinning, by the same reasoning as `@main` above — so a preset that fails validation
stops Renovate in every repo at once, each with an "Action Required" issue. That
failure is loud, not silent; a preset that is *valid but wrong* (an `automerge: true`
in the wrong rule) is the one to read twice.

Nothing in this repo merges itself, Renovate PRs included: see `renovate.json`.

### Not Dependabot

Dependabot was briefly added alongside Renovate and opened duplicates of everything
Renovate already proposed — plus one PR per *transitive* package, from
`allow: dependency-type: all`. It has no equivalent of a shared preset (every repo needs
its own `dependabot.yml`), and it cannot merge anything on a private repo without branch
protection, which is the problem the old `automerge.yml` files existed to work around.

Leave **Dependabot alerts** on (Settings → Code security): Renovate reads them for its
security PRs. Turn **Dependabot security updates** off, account-wide, or every advisory
arrives twice.
