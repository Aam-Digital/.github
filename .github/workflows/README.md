# Shared GitHub Actions workflows

Reusable workflows for all Aam-Digital repositories, so that each repository keeps only what is specific to it
(see Aam-Digital/internal#392).

| Workflow                                         | Purpose                                                       | Check name in the caller      |
| ------------------------------------------------ | ------------------------------------------------------------- | ----------------------------- |
| [`docker-image.yml`](docker-image.yml)           | Build (and push) a container image for PRs and releases       | `<caller job> / build`        |
| [`pr-image-cleanup.yml`](pr-image-cleanup.yml)   | Delete the `pr-<n>` image(s) from ghcr.io when a PR is closed | `<caller job> / cleanup`      |
| [`pr-title-lint.yml`](pr-title-lint.yml)         | Enforce Conventional Commits in PR titles                     | `<caller job> / Validate PR title` |
| [`actionlint.yml`](actionlint.yml)               | Lint the repository's own workflow files                      | `<caller job> / actionlint`   |

`ci.yml` is this repository's own CI (actionlint on the workflows here); it is not meant to be called.

## General rules

- **Pin `@v1`**, never `@main` or a branch:
  `uses: Aam-Digital/.github/.github/workflows/<file>@v1`.
  This way a push to this repository (e.g. an edit of the org profile README) does not change CI everywhere.
- **Permissions:** a called workflow cannot get more permissions than the calling job has.
  The calling job (or the caller workflow's top-level `permissions`) must grant at least what is listed for each
  workflow below, otherwise the run fails before it starts.
- **Secrets and `env`** of the caller are not passed through. Everything a workflow needs is an input or an explicitly
  passed secret.
- **Check names:** GitHub names the check of a job in a called workflow `<caller job> / <called job>`, where each part is
  the job's `name` if it has one, else its id. The called job ids/names listed above are part of the `v1` contract.
  If you move an existing job into one of these workflows, either keep the caller job's name so that the check name is
  what the repository's ruleset expects, or update the required status checks in the same PR. Otherwise PRs wait
  forever for a check that never reports.

## `docker-image.yml`

One workflow for PR, prerelease (`edge`) and release images. It checks out the calling repository, optionally downloads
an artifact and runs a preparation command, then builds with Buildx (QEMU only if a non-amd64 platform is requested),
using the GitHub Actions cache (one scope per image) and setting the OCI labels from `docker/metadata-action`.

Required permissions of the calling job: `contents: read`, `packages: write` (also with `push: false`).

| Input           | Default                   | Description |
| --------------- | ------------------------- | ----------- |
| `images`        | (required)                | Image name(s) without tag, one per line or comma-separated. All get the same tags. Images on ghcr.io are pushed with the `GITHUB_TOKEN`; images on Docker Hub (`docker.io/…` or no registry host) need the `DOCKERHUB_*` secrets. |
| `context`       | `.`                       | Build context |
| `file`          | `<context>/Dockerfile`    | Dockerfile |
| `version`       | –                         | Tags the image `<version>` and sets the `org.opencontainers.image.version` label (unless `labels` sets it) |
| `stable`        | `false`                   | Also tag `stable` and `latest`. Requires `version`. Pass `${{ !github.event.release.prerelease }}`; never derive it from the version string. |
| `tags`          | –                         | Extra tags, one per line or comma-separated, e.g. `edge`, `pr-123` |
| `labels`        | –                         | Extra or overriding labels (`key=value`, one per line) |
| `platforms`     | `linux/amd64,linux/arm64` | PR builds pass `linux/amd64` |
| `build-args`    | –                         | `KEY=value`, one per line |
| `push`          | `true`                    | `false` only checks that the image builds (e.g. PRs from forks) |
| `cache-scope`   | last path segment of the first image | GitHub Actions cache scope |
| `artifact-name` / `artifact-path` | – / `.` | Download a workflow artifact (e.g. a web app built in an earlier job) before building |
| `prepare`       | –                         | Shell commands run in the checkout before building; `$VERSION` holds the `version` input |

| Secret               | Description |
| -------------------- | ----------- |
| `DOCKERHUB_USERNAME` | Only for images on Docker Hub |
| `DOCKERHUB_TOKEN`    | Only for images on Docker Hub |

Outputs: `digest` (of the pushed image or multi-platform index), `tags` (all `<image>:<tag>` references, one per line).

A PR image must carry only its `pr-<n>` tag (do not pass `version` for PR builds; set the version label via `labels` if
needed). `pr-image-cleanup.yml` never deletes an image that carries another tag.

### Example: PR image + release image

```yaml
# .github/workflows/pr-release.yml
on:
  pull_request:

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
      # PRs from forks cannot push; only check that the image builds
      push: ${{ github.event.pull_request.head.repo.full_name == github.repository }}
```

```yaml
# .github/workflows/on-release.yml
on:
  release:
    types: [published] # the release must be created with a PAT, GITHUB_TOKEN releases trigger no workflows

jobs:
  build:
    uses: Aam-Digital/.github/.github/workflows/docker-image.yml@v1
    permissions:
      contents: read
      packages: write
    with:
      images: ghcr.io/aam-digital/my-service
      version: ${{ github.event.release.tag_name }}
      # full release: <version>, stable, latest; prerelease (X.Y.Z-edge.N): <version>, edge
      stable: ${{ !github.event.release.prerelease }}
      tags: ${{ github.event.release.prerelease && 'edge' || '' }}
```

### Example: several registries, artifact from an earlier job

```yaml
jobs:
  build-webapp:
    # ... uploads the artifact "webapp-files"
  build:
    needs: build-webapp
    uses: Aam-Digital/.github/.github/workflows/docker-image.yml@v1
    permissions:
      contents: read
      packages: write
    secrets:
      DOCKERHUB_USERNAME: ${{ secrets.DOCKER_USERNAME }}
      DOCKERHUB_TOKEN: ${{ secrets.DOCKER_PASSWORD }}
    with:
      images: |
        aamdigital/ndb-server
        ghcr.io/aam-digital/ndb-core
      file: build/Dockerfile
      artifact-name: webapp-files
      artifact-path: dist
      version: ${{ github.event.release.tag_name }}
      stable: ${{ !github.event.release.prerelease }}
```

## `pr-image-cleanup.yml`

Deletes the image(s) tagged `pr-<n>` when the PR is closed, using
[`dataaxiom/ghcr-cleanup-action`](https://github.com/dataaxiom/ghcr-cleanup-action). It reads all package versions
(not just the first page) and is manifest-aware: deleting a multi-platform index also deletes its per-platform and
attestation manifests, unless another image still uses them; a version that also carries another tag is only untagged.
Untagged versions that are not children of the deleted image are never touched.

Most leaked PR images are not failed deletes but a race: the build of the PR's last push is still running when
the PR is merged, and pushes `pr-<n>` only after the cleanup ran. Pass the PR build workflow file(s) as
`build-workflows`: their unfinished runs on the PR's head branch are then cancelled, and the cleanup waits (up to
15 minutes) until they have stopped before deleting.

Required permissions of the calling job: `actions: write` (cancel PR builds), `packages: write`.
The default `GITHUB_TOKEN` can delete versions of packages linked to the repository (Admin role in the package's
"Manage Actions access" settings). Otherwise pass a classic PAT
(`read:packages`, `delete:packages`) as the `token` secret.

| Input             | Default                            | Description |
| ----------------- | ---------------------------------- | ----------- |
| `packages`        | repository name                    | ghcr.io package name(s), comma-separated |
| `tags`            | `pr-<number>` of the triggering PR | Exact tag(s), comma-separated (no wildcards) |
| `build-workflows` | –                                  | PR build workflow file name(s), comma-separated, to cancel and wait for first |
| `dry-run`         | `false`                            | Only log what would be deleted |

Note: this deletes the image that is tagged `pr-<n>` when the PR closes. Images of earlier pushes to the PR lost the tag
when they were replaced and are untagged by then; they are not touched here.

```yaml
# .github/workflows/pr-closed.yml
on:
  pull_request:
    types: [closed]

jobs:
  cleanup-image:
    # PRs from forks have no image (and their token cannot delete packages)
    if: github.event.pull_request.head.repo.full_name == github.repository
    uses: Aam-Digital/.github/.github/workflows/pr-image-cleanup.yml@v1
    permissions:
      actions: write
      packages: write
    with:
      build-workflows: pr-release.yml
    #   packages: aam-services,keycloak-third-party-authentication
    # secrets:
    #   token: ${{ secrets.GH_TOKEN }} # only if the repository is not linked to the package
```

## `pr-title-lint.yml`

Checks that the PR title follows [Conventional Commits](https://www.conventionalcommits.org/), using
`amannn/action-semantic-pull-request`. This only guarantees conventional commits on the default branch if the
repository allows squash merging only, with "Default commit message" = "Pull request title"
(`squash_merge_commit_title = PR_TITLE`).

Required permissions of the calling job: `pull-requests: read`.

```yaml
# .github/workflows/pr-title-lint.yml
on:
  # pull_request_target so that it also runs for PRs from forks. Safe here: the workflow never checks out PR code
  # and only reads the PR title from the event payload. The zizmor comment keeps `qlty check` (dangerous-triggers,
  # blocking with qlty's cloud defaults) from failing on this line.
  pull_request_target: # zizmor: ignore[dangerous-triggers] never checks out PR code, only reads the PR title from the event payload
    types: [opened, edited, synchronize, reopened]

jobs:
  pr-title:
    uses: Aam-Digital/.github/.github/workflows/pr-title-lint.yml@v1
    permissions:
      pull-requests: read
```

Check name: `pr-title / Validate PR title`.

## `actionlint.yml`

Runs [actionlint](https://github.com/rhysd/actionlint) (with shellcheck) on the calling repository's workflows.

Required permissions of the calling job: `contents: read`.

```yaml
on:
  pull_request:
  push:
    branches: [main]

jobs:
  actionlint:
    uses: Aam-Digital/.github/.github/workflows/actionlint.yml@v1
    permissions:
      contents: read
```

Check name: `actionlint / actionlint`.

## Versioning: how `v1` is moved and when `v2` is cut

Consumers reference a moving major tag (`v1`). Each change merged to `main` that should reach the consumers is released
like this:

1. Tag the merge commit with the next immutable version and move the major tag to it:

   ```shell
   git fetch origin --tags
   git tag v1.2.0 origin/main
   git tag -f v1 origin/main
   git push origin v1.2.0
   git push --force origin v1
   ```

2. Create a GitHub release for `v1.2.0` with a short changelog.

Changes that are compatible with all existing callers may go into `v1`: bug fixes, new optional inputs/outputs,
action updates. Everything else needs a new major tag (`v2`), and the consumers are switched one by one:

- removing or renaming an input, output or secret, or changing an input's default or meaning;
- renaming a job id or job `name` (this changes check names, see above);
- requiring more permissions from the caller;
- changing which image tags are produced for the same inputs.

Rules for the workflows in this folder:

- Keep shell inline. Do not fetch or run scripts from this repository at runtime.
- Use fully qualified `uses:` references (`owner/repo/...@ref`). A local `./` reference is ambiguous once the
  workflow is called from another repository (for a step's action it points into the caller's checkout).
- Actions of large trusted providers (`actions/*`, `docker/*`, `github/*`, …) use their plain major tag.
  Other actions are pinned to a commit SHA with a `# vX.Y.Z` comment.
- Pass inputs to `run:` scripts via `env:`, never by `${{ }}` interpolation into the script.
