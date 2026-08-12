# gh-actions-check-environment

A minimal, reproducible demonstration of a GitHub Actions risk: referencing an
undefined Environment name from a workflow causes GitHub to silently create
that Environment **without any protection rules**, allowing the job to run
without approval.

refs.
- https://docs.github.com/en/actions/how-tos/deploy/configure-and-manage-deployments/manage-environments
- https://docs.github.com/en/actions/reference/workflows-and-actions/deployments-and-environments

## The problem

When a job in a GitHub Actions workflow declares:

```yaml
jobs:
  deploy:
    environment: production
```

the implicit assumption is usually "use the existing `production`
Environment." In reality, GitHub's behavior is:

> If an Environment with that name does not already exist in the repository,
> create it — with no protection rules (Required Reviewers, wait timers,
> deployment branch policies, etc.) and no secrets.

Anyone who can edit a workflow file can therefore create a new Environment
simply by referencing an unused name. If that name is a typo, or an
Environment that has not been provisioned yet, the job runs immediately with
no approval gate — even though the workflow *looks* like it is protected by
an Environment.

This is not specific to Terraform or any particular deployment tool. It
applies to **any** job that uses `environment:` to gate a destructive or
sensitive action. This repository reproduces the issue in the abstract,
using a dummy "destructive operation" step instead of a real tool.

Reference:
[Deployments and environments](https://docs.github.com/en/actions/reference/workflows-and-actions/deployments-and-environments)

## What this repository verifies

The fix under test is **not** "add Required Reviewers to every Environment."
Required Reviewers remain the correct mechanism for gating an approved
Environment — but they cannot help if the workflow references an Environment
that doesn't exist yet, because the auto-created Environment has no
protection rules to begin with.

Instead, this repository verifies a **pre-flight check**: before any job
references `environment: <stage>`, a `check-environment` job confirms that
`<stage>` already exists as a real, pre-provisioned GitHub Environment (via
the GitHub API), and fails the workflow if it does not. This closes the gap
that Required Reviewers alone cannot cover — an Environment name that
doesn't exist yet has no reviewers to bypass in the first place.

The list of stages a workflow is allowed to use (`development`, `staging`,
`production`, …) is statically enumerated in the calling workflow. The
`check-environment` job does not invent or derive this list — it only
verifies, one stage at a time, that the statically declared name is backed
by a real Environment.

### Files

| File | Role |
|---|---|
| `.github/workflows/deploy.yml` | Guarded entry point. Statically lists stages in `matrix.stage` and calls `deploy-stage.yml` once per stage. |
| `.github/workflows/deploy-stage.yml` | Guarded reusable workflow. Runs `check-environment` before `plan`/`apply`; `plan`/`apply` are gated on `needs: check-environment` so an undefined stage never reaches them. |
| `.github/workflows/deploy-cancel-on-close.yml` | Cancels any `deploy.yml` run still pending Required Reviewers approval when its PR is closed. See "Concurrency and PR close" below. |
| `.github/workflows/deploy-unguarded.yml` | Comparison entry point, kept for reference only. Calls `deploy-stage-unguarded.yml` with `preview`, a stage that is intentionally **not** pre-created. Triggered only via `workflow_dispatch` so it never runs as a side effect of opening a PR. |
| `.github/workflows/deploy-stage-unguarded.yml` | Comparison reusable workflow. No pre-flight check — `environment: ${{ inputs.stage }}` is referenced directly, reproducing the original incident. |

### Concurrency and PR close

`deploy.yml` sets `concurrency: { group: deploy-<PR number>, cancel-in-progress: true }`,
scoped per PR so unrelated PRs never cancel each other's runs. A new push to
the same PR cancels any run still in progress for that PR — including one
left waiting on Required Reviewers approval.

Closing a PR does not by itself cancel its in-progress workflow runs; GitHub
has no built-in behavior for that. A run left pending on Required Reviewers
approval (as in scenario 1 below) stays pending indefinitely otherwise.
`deploy-cancel-on-close.yml` closes that gap: it triggers on
`pull_request: types: [closed]` and shares the same concurrency group
(`deploy-<PR number>`), so GitHub cancels the corresponding `deploy.yml` run
for that PR as soon as this job starts.

The guarded entry point (`deploy.yml`) triggers on `pull_request` with
`paths: ['.github/workflows/**']`, so the reproduction is faithful to the
original scenario: a PR that changes a workflow file is what exposes the
risk, not a manual dispatch. `deploy-unguarded.yml`, by contrast, is
intentionally *not* wired to `pull_request` — it reproduces an unprotected
Environment being auto-created, so it must only run when deliberately
dispatched by hand.

## How to verify

### 0. Prerequisite

The Environments `development`, `staging`, and `production` must already
exist in this repository (created via `gh api --method PUT
repos/<owner>/<repo>/environments/<name>`). This reflects the assumption
stated in the incident report: Environments are assumed to be pre-provisioned,
and the goal is to detect when a workflow references one that is *not* in
that pre-provisioned set.

All three Environments additionally have a Required Reviewer configured (via
`gh api --method PUT repos/<owner>/<repo>/environments/<name>` with a
`reviewers` payload). This is what scenario 4 below verifies: every
pre-provisioned stage is expected to be protected, and the audit confirms
none of them was missed.

### 1. Baseline: guarded workflow, all stages already defined

Open a PR that touches `.github/workflows/**` without changing the stage
list in `deploy.yml`. Expected result: `check-environment` passes for each of
`development`, `staging`, `production`, and `plan`/`apply` run normally for
all three matrix entries.

This confirms the guard does not produce false positives when every declared
stage is legitimate.

### 2. Reproduction target: guarded workflow, undefined stage added

On a branch, add an unlisted stage (e.g. `preview`, which does not exist as
an Environment) to `matrix.stage` in `deploy.yml`, and open it as a PR.

Expected result: for the `preview` matrix entry, `check-environment` fails
with an error such as:

```
Environment 'preview' is not defined in this repository. Create it before referencing it from a workflow.
```

and neither `plan` nor `apply` runs for that entry (they are blocked by
`needs: check-environment`). The other matrix entries — `development`,
`staging`, `production` — are unaffected, since `fail-fast: false` isolates
each stage.

This is the core claim under test: **an unexpected Environment is prevented
from being used, before any destructive step runs.**

### 3. Contrast: unguarded workflow, undefined stage

Manually dispatch `deploy-unguarded.yml` (already wired to the undefined
stage `preview` by default):

```bash
gh workflow run deploy-unguarded.yml
```

Expected result: `plan` and `apply` both run for `preview` with no approval
gate, and GitHub auto-creates the `preview` Environment.

Confirm the auto-created Environment has no protection rules:

```bash
gh api repos/<owner>/<repo>/environments/preview --jq '.protection_rules'
# => []
```

An empty `protection_rules` array is the direct evidence of the original
incident: the Environment exists, but nothing gates access to it.

### 4. Protection rules: are all pre-provisioned Environments actually guarded?

The `check-environment` job only verifies that a stage *exists* as an
Environment — it says nothing about whether that Environment has
`Required Reviewers` configured. Existence and protection are independent
properties, and both matter: an Environment can exist with no protection
rules at all, in which case `check-environment` passes but the subsequent
`plan`/`apply` still run without approval. Passing scenario 1 is therefore
not sufficient evidence that deployments are actually gated — it only shows
the referenced stages are legitimate, not that they are protected.

Check `protection_rules` across all pre-provisioned Environments:

```bash
gh api repos/<owner>/<repo>/environments --jq \
  '.environments[] | {name, protection_rules: (.protection_rules | length)}'
```

Expected result in this repository:

```
{"name":"development","protection_rules":1}
{"name":"production","protection_rules":1}
{"name":"staging","protection_rules":1}
```

Every pre-provisioned Environment has a `required_reviewers` rule attached.
This confirms that `check-environment` and Required Reviewers are two
independent, complementary controls — the pre-flight check guards against an
*undefined* Environment being used, while Required Reviewers guard against
an *unapproved* deployment to an Environment that does exist. Neither one
substitutes for the other, and this audit step is what actually verifies the
latter; scenario 1 alone does not.

### Summary of expected outcomes

| Scenario | Stage | Guard present? | Environment protected? | Result |
|---|---|---|---|---|
| 1. Baseline | `development`/`staging`/`production` | Yes | Yes (all three) | `check-environment` passes; `plan`/`apply` run |
| 2. Reproduction target | `preview` (undefined) | Yes | N/A (does not exist) | `check-environment` fails; `plan`/`apply` never run |
| 3. Contrast | `preview` (undefined) | No | No | `plan`/`apply` run; Environment auto-created with `protection_rules: []` |
| 4. Protection audit | all pre-provisioned stages | N/A | Verified via API | All three have `required_reviewers` configured |

Scenario 2 is the reproduction of the fix's intended behavior. Scenario 3 is
the reproduction of the original incident. Scenario 1 confirms the fix does
not block legitimate, pre-provisioned stages. Scenario 4 confirms that
"exists" and "protected" are separate properties, and that this repository's
pre-provisioned stages satisfy both.
