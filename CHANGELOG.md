# Changelog

Releases follow semver from the caller's point of view: a **major** is anything a project's `ci.yml` must adapt to.

## 1.1.0

- `dependabot-review.yml`: `deploy-workflow` input (default empty). A merge made with `GITHUB_TOKEN` starts no `push` workflow, so a project that deploys on push to `main` never deployed an auto-merged bump. When set, the named workflow is dispatched on the base branch after a successful merge; if the dispatch fails, the PR gets a comment asking for a manual deploy and the Merge job fails. The deploy workflow needs a `workflow_dispatch` trigger and jobs that accept it on `main`, and the calling job needs `actions: write`.
- `dependabot-review.yml`: the Merge job no longer declares its own permissions and inherits the calling job's (the workflow-level `contents: read` is gone; the Review job still declares its own). Callers that do not set `deploy-workflow` need no change.

## 1.0.0

- `dependabot-review.yml`: review of Dependabot PRs on Django/uv backends with a structured verdict, posted by the workflow; squash-merge of patch/minor bumps when the verdict is `ok` and CI succeeded, gated by `automerge`. When CI fails, the tail of each failed job's log is handed to the review.
- `dependabot-review.yml`: `trusted-actors` input (default `inferencia-runner[bot]`). Fix commits a trusted actor pushes to a Dependabot PR are reviewed as a code change, and the PR can merge only if the fix is correct and minimal for the bump. Commits by anyone else block the merge. The update type is the highest of `dependabot/fetch-metadata` and of the metadata in every Dependabot commit message.
