# linkchar-backend-workflows

Shared, versioned GitHub Actions workflows for Linkchar's **backend** projects (Django + uv, the ones started from `django-base-project`).

**Backend only, on purpose.** Tags version the whole repository, so a release here becomes a Dependabot bump PR — review, auto-merge, production redeploy — in every project that calls it. Frontend workflows live in [`linkchar-frontend-workflows`](https://github.com/Linkcharsoft/linkchar-frontend-workflows), which follows the same pattern; keeping them apart means a release of one never opens PRs in the other's projects.

Each project calls these workflows from its `ci.yml`, pinned to a release tag. Dependabot bumps that tag like any other action, so improving a workflow here means one release, and every project receives it as a reviewed PR. There is no shared CI workflow: each backend keeps its own Lint and Tests jobs, because they depend on the project's services and settings.

| Workflow | What it does |
| --- | --- |
| [`dependabot-review.yml`](.github/workflows/dependabot-review.yml) | On Dependabot PRs only: Claude reads the changelog of every bumped package (`pyproject.toml`, `uv.lock`, the Dockerfile's base image, workflow actions), checks whether the project uses what changed — and, when CI fails, reads the failed jobs' logs — and returns a verdict; the workflow posts it as a comment and squash-merges patch/minor bumps that passed review **and** CI. Fix commits pushed to the PR by a [trusted actor](#fix-commits-from-a-trusted-actor) are reviewed as a code change in the same pass. |

## Usage

`.github/workflows/ci.yml` in the project — its own `lint` and `test` jobs stay as they are, and a `dependabot` job is added after them:

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

permissions:
  contents: read

jobs:
  lint:
    name: Lint
    runs-on: ubuntu-latest
    steps:
      # ... the project's own lint steps (uv lock --check, ruff check, ruff format --check)

  test:
    name: Tests
    runs-on: ubuntu-latest
    steps:
      # ... the project's own test steps (Postgres service, migrations check, pytest)

  dependabot:
    needs: [lint, test]
    # Dependabot's PRs, whoever pushed last: the called workflow decides which pushers it trusts.
    if: ${{ !cancelled() && github.event.pull_request.user.login == 'dependabot[bot]' }}
    uses: Linkcharsoft/linkchar-backend-workflows/.github/workflows/dependabot-review.yml@v1.0.0
    permissions:
      actions: read
      contents: write
      pull-requests: write
      id-token: write
    secrets:
      CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
    with:
      # "success" only when every job above succeeded; failed, cancelled or skipped all count as not.
      ci-result: ${{ (needs.lint.result == 'success' && needs.test.result == 'success') && 'success' || 'failure' }}
      automerge: ${{ vars.DEPENDABOT_AUTOMERGE || 'minor' }}
```

When a project adds a CI job, add it to `needs` **and** to the `ci-result` expression, or a failure in it will not block the merge.

A project that still has its own inline `.github/workflows/dependabot-review.yml` should delete it when adopting this one, or every Dependabot PR gets reviewed twice.

Plus `.github/dependabot.yml`, which GitHub reads only from the project itself and cannot be shared — `django-base-project`'s version is the canonical one. Add a `github-actions` entry if it is missing: that is what bumps the tag above.

### Inputs

| Input | Default | Purpose |
| --- | --- | --- |
| `ci-result` | — (required) | `success` only when every CI job succeeded (see the expression above). Only `success` can merge. |
| `automerge` | `minor` | `minor` merges patch/minor bumps; any other value (use `off`) only reviews. Set the `DEPENDABOT_AUTOMERGE` repo variable to change it per project without touching the YAML. |
| `model` | `claude-sonnet-5-5` | Model used for the review. |
| `max-turns` | `40` | Caps the agent's turns on large grouped PRs. |
| `trusted-actors` | `inferencia-runner[bot]` | Comma-separated logins, besides Dependabot, allowed to push fix commits to a Dependabot PR and still get it reviewed and merged. See [below](#fix-commits-from-a-trusted-actor). |

### Merge policy

A Dependabot PR is merged only when **all** of these hold:

1. CI (`Lint` + `Tests`) succeeded.
2. Claude's verdict is `ok`.
3. The highest update in the PR is a patch or minor — the highest of `dependabot/fetch-metadata` and of the `update-type` lines in every Dependabot commit message, never the agent's word. Majors are reviewed and commented, never merged; an update type that cannot be determined never merges either.
4. Every commit in the PR is Dependabot's (with a verified signature) or a trusted actor's, the run was triggered by one of them, and the head is still the reviewed commit (`--match-head-commit`).
5. `automerge` is `minor`.

The squash commit is titled with the PR title (which carries the prefix set in the project's `dependabot.yml`) plus `(#N)`.

> **Merging to `main` can deploy to production.** In a project whose deploy pipeline runs on push to `main`, an auto-merged bump ships without a human in the loop. Lint, the test suite and the migrations check are the gate; the Claude verdict is a changelog review, not a security control — a `cooldown` in `dependabot.yml` is what keeps a freshly published malicious release out.

### Fix commits from a trusted actor

A PR the review does not clear stays open for a human — or for a bot that fixes it. When an actor listed in `trusted-actors` pushes a commit to the Dependabot branch, CI and the review run again on the new head:

- The run must be triggered by Dependabot or a trusted actor, and the PR must be Dependabot's. A push by anyone else skips the review, so nothing merges.
- Every commit is classified by its GitHub author: Dependabot's (signature verified), a trusted actor's, or anyone else's. The trusted actor must author its commits as its own account (for a GitHub App, `<id>+<app>[bot]@users.noreply.github.com`); a commit whose email maps to no account counts as someone else's.
- The trusted actor's commits are listed for the review, which reviews them as a code change: the verdict is `ok` only if the fix is correct and minimal for the bump, and CI passed.
- Any commit by someone else blocks the merge; the verdict comment says which.
- The update type comes from Dependabot's own commit messages, so extra commits do not let a major through.

## Setup

1. **Secret, in every calling project** — create `CLAUDE_CODE_OAUTH_TOKEN` (from `claude setup-token`) **twice**, with the same value: under the project's *Settings → Secrets and variables → **Dependabot*** and under *Settings → Secrets and variables → **Actions***. Runs triggered by Dependabot only read Dependabot secrets; runs triggered by a trusted actor's push (e.g. a GitHub App) only read Actions secrets. The caller passes it in, so a secret stored in **this** repository never reaches anyone. On GitHub Free organization secrets do not reach private repositories — on a paid plan one organization secret of each kind, shared with the backend repositories, replaces the per-repo ones. Without it the review is skipped with a warning and nothing merges.
2. **Access** — keep this repository **public** (it holds no secrets): callers and Dependabot then reach it on any plan. If it must be private, allow access from the organization's repositories under *Settings → Actions → General → Access*, and add it under *Organization settings → Advanced Security → Grant Dependabot access to private repositories*.

## Releasing

1. Change the workflow on a branch. To try it before releasing, point a project's `ci.yml` at the branch (`@my-branch`) in a throwaway PR — `django-base-project` is the natural test bed.
2. Add an entry to [`CHANGELOG.md`](CHANGELOG.md): the review of each project's bump PR reads it to explain what changed.
3. Merge, then tag `vX.Y.Z` and publish a GitHub release with the same notes.
   - **patch / minor**: compatible — projects auto-merge the bump.
   - **major**: anything a caller must adapt to (a renamed or required input, a renamed job, a new required secret or permission). Projects get it as a PR that waits for a human.

Never move or reuse a tag: projects pin exact tags so that every change reaches them as a reviewable PR.
