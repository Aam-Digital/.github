# Shared GitHub Actions workflows

Reusable workflows for all Aam-Digital repositories, so that each repository keeps only what is specific to it.
Inputs, secrets and outputs are documented in the `workflow_call` section at the top of each file.

| Workflow | What it does | Typical trigger in the caller | Permissions of the calling job | Check name |
| --- | --- | --- | --- | --- |
| [`docker-image.yml`](docker-image.yml) | Builds and pushes a container image (PR, edge, release) | `pull_request`, `push`, `release: published` | `contents: read`, `packages: write` | `<job> / build`; with `native-runners`: `<job> / build-native (<platform>)`, `<job> / merge` |
| [`edge-version.yml`](edge-version.yml) | Computes the version of an edge build (`1.4.1-edge.5`) | `push` to the default branch | `contents: read` | `<job> / version` |
| [`release.yml`](release.yml) | Creates the next full release with semantic-release | `push` to the default branch, or `workflow_dispatch` | `contents: read` (+ PAT as secret `token`) | `<job> / release` |
| [`image-prune.yml`](image-prune.yml) | Deletes old edge images (`*-edge.*`) after a full release | `release: published` | `packages: write` | `<job> / prune` |
| [`pr-image-cleanup.yml`](pr-image-cleanup.yml) | Deletes the `pr-<n>` image when a PR closes, then untagged images older than 2 days | `pull_request: closed` | `actions: write`, `packages: write` | `<job> / cleanup` |
| [`pr-title-lint.yml`](pr-title-lint.yml) | Checks that PR titles follow Conventional Commits | `pull_request_target` | `pull-requests: read` | `<job> / Validate PR title` |
| [`actionlint.yml`](actionlint.yml) | Lints the repository's workflows (with shellcheck) | `pull_request`, `push` | `contents: read` | `<job> / actionlint` |

`ci.yml` is this repository's own CI; it is not meant to be called.

## Calling a workflow

```yaml
permissions: {} # nothing at the top level; each job grants what it needs

jobs:
  build:
    uses: Aam-Digital/.github/.github/workflows/docker-image.yml@v1
    permissions:
      contents: read
      packages: write
    with:
      images: ghcr.io/aam-digital/my-service
      tags: pr-${{ github.event.number }}
      platforms: linux/amd64
```

- Always reference **`@v1`**, never a branch.
- The calling job must grant the **permissions** from the table. A called workflow cannot get more, and the run fails
  before it starts. `permissions: {}` at the top level keeps `qlty check` (checkov) happy.
- The caller's **secrets and `env` are not passed on**: pass secrets explicitly under `secrets:`, everything else as
  inputs.
- **Check names** become `<caller job> / <called job>`. Moving an existing job into a shared workflow renames its
  check, so update the required checks of the ruleset when that PR merges. Skipped jobs (e.g. `build-native` and
  `merge` when `native-runners` is off) count as passed.
- `pr-title-lint.yml` only works with squash-only merging and the PR title as commit title. Call it on
  `pull_request_target` with the comment `# zizmor: ignore[dangerous-triggers]` on that line, otherwise `qlty check`
  fails.
- PR images carry only their `pr-<n>` tag (otherwise the cleanup keeps them). PRs from forks build with `push: false`.

Complete examples: [replication-backend](https://github.com/Aam-Digital/replication-backend/tree/master/.github/workflows)
(auto release mode, PR images) and [ndb-core](https://github.com/Aam-Digital/ndb-core/tree/master/.github/workflows)
(manual release mode, two registries, web app built in an earlier job and passed as an artifact).

## Release model

Every push to the default branch builds an **edge** image; full releases come from Conventional Commit PR titles via
[semantic-release](https://semantic-release.gitbook.io/) (plugins in each repository's own `.releaserc*`).

- **Edge build:** `edge-version.yml` computes `<last release, patch + 1>-edge.<commits since>`, e.g. `1.4.1-edge.5`.
  There is no tag and no GitHub release for it. Move the `edge` tag only if `is-head` is `true`, and give the image job
  a `concurrency` group with `cancel-in-progress: true`, so that an older run never moves `edge` back.
- **Full release**, in one of two modes: **auto** (`release.yml` on every push; a `fix`/`feat`/breaking change is
  released right away) or **manual** (`release.yml` via `workflow_dispatch`). Releases are created with a PAT, because
  releases created with the `GITHUB_TOKEN` trigger no workflows. The `release: published` event builds the image, and
  `image-prune.yml` then deletes the edge images of the finished cycle.

| Build | Image tags |
| --- | --- |
| push to the default branch | `1.4.1-edge.5`, `edge` |
| full release `1.4.1` | `1.4.1`, `stable`, `latest` |
| pull request #123 | `pr-123` |

Pass `stable: ${{ !github.event.release.prerelease }}` for release images; never derive it from the version string.

## Versioning (`v1`)

There are no releases of this repository. After every push to `main`, CI moves `v1` to the new commit once actionlint
has passed, so a merged PR (Renovate's included) reaches every repository on its next run. Merging here is the release
step: check what a change does to the callers before merging.

A change that breaks callers needs `v2`: set `MAJOR_TAG: v2` in `ci.yml` in the same PR, so that `v1` stays where it
is, then switch the callers one by one. Breaking means:

- removing or renaming an input, output or secret, or changing an input's default or meaning;
- renaming a job id or job `name` (this changes check names);
- requiring more permissions from the caller;
- changing which image tags are produced for the same inputs.

To test a change before merging, point a caller PR at the branch (`@<branch>`), and switch back to `@v1` before
merging it. In an emergency, `v1` can be reset by hand:
`gh api -X PATCH repos/Aam-Digital/.github/git/refs/tags/v1 -f sha=<good commit> -F force=true`.

## Rules for the workflows here

- Keep shell inline; never fetch scripts from this repository at runtime.
- Pass inputs to `run:` scripts via `env:`, never by `${{ }}` interpolation into the script.
- Actions of large trusted providers (`actions/*`, `docker/*`, `github/*`, …) use their major tag; other actions are
  pinned to a commit SHA with a `# vX.Y.Z` comment. Renovate (`renovate.json` in the repository root) updates both.
- Steps shared between jobs of one workflow are written once and reused with YAML anchors (`&name` / `*name`).
- Explain *why* in comments in the workflow file, next to the code; keep this README to what callers need.
