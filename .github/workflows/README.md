# Shared GitHub Actions workflows

Reusable workflows for all Aam-Digital repositories, so that each repository keeps only what is specific to it
(see Aam-Digital/internal#392).

| Workflow                                       | Purpose                                                          | Check name(s) in the caller |
| ---------------------------------------------- | ---------------------------------------------------------------- | --------------------------- |
| [`docker-image.yml`](docker-image.yml)         | Build (and push) a container image for PRs, prereleases and releases; delete the image it replaces | `<caller job> / build`, or with `native-runners`: `<caller job> / build-native (linux/amd64)`, `<caller job> / build-native (linux/arm64)`, `<caller job> / merge` |
| [`release.yml`](release.yml)                   | Create the next release or `edge` prerelease with semantic-release | `<caller job> / release` |
| [`image-prune.yml`](image-prune.yml)           | Delete old prerelease images (`*-edge.*`) after a full release   | `<caller job> / prune` |
| [`pr-image-cleanup.yml`](pr-image-cleanup.yml) | Delete the `pr-<n>` image(s) from ghcr.io when a PR is closed    | `<caller job> / cleanup` |
| [`pr-title-lint.yml`](pr-title-lint.yml)       | Enforce Conventional Commits in PR titles                        | `<caller job> / Validate PR title` |
| [`actionlint.yml`](actionlint.yml)             | Lint the repository's own workflow files                         | `<caller job> / actionlint` |

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
  forever for a check that never reports. Jobs of the mode that is not used (e.g. `merge` without `native-runners`)
  report as skipped; a skipped required check counts as passed.

## Release model

Every repository versions with [semantic-release](https://semantic-release.gitbook.io/) from Conventional Commit PR
titles (squash merge, see `pr-title-lint.yml`). There are two modes:

- **auto**: every push to the default branch that contains a `fix`/`feat`/breaking change is released right away
  (`release.yml` with `type: release` on push).
- **manual**: pushes to the default branch create **prereleases** `<next version>-edge.<n>`, e.g. `1.4.0-edge.3`
  (`release.yml` with `type: prerelease` on push). As soon as there is any `fix`/`feat`/breaking change since the last
  full release, *every* later push creates the next prerelease (and image), also a push with only `chore`/`docs`
  commits; see `release.yml` below. A full release is created by running the release workflow by hand
  (`workflow_dispatch`, `type: release`). It first deletes the `*-edge.*` prereleases and tags (and the `*-master.*` ones
  of the old naming), then releases the next version. After the full release, `image-prune.yml` deletes the
  prerelease images.

Releases are created with a PAT (`GH_TOKEN`), because releases created with the `GITHUB_TOKEN` trigger no workflows.
The `release: published` event then builds the image (`docker-image.yml`) with these tags:

| GitHub release                     | Image tags                                  |
| ---------------------------------- | ------------------------------------------- |
| prerelease `1.4.0-edge.3`          | `1.4.0-edge.3`, `edge`                      |
| full release `1.4.0`               | `1.4.0`, `stable`, `latest`                 |
| pull request #123 (not a release)  | `pr-123`                                    |

`stable`/`latest` come from the release's **prerelease flag** (`stable: ${{ !github.event.release.prerelease }}`),
never from parsing the version string.

## `docker-image.yml`

One workflow for PR, prerelease (`edge`) and release images. It checks out the calling repository, optionally downloads
an artifact and runs a preparation command, then builds with Buildx, using the GitHub Actions cache (one scope per
image) and setting the OCI labels from `docker/metadata-action`.

Two build modes:

- **default** (job `build`): all platforms in one job; QEMU is set up if a non-amd64 platform is requested.
- **`native-runners: true`**: one job per platform on a native GitHub-hosted runner (`linux/amd64` on `ubuntu-latest`,
  `linux/arm64` on `ubuntu-24.04-arm`; no other platforms), each pushes its image by digest only. Then the job `merge`
  creates the multi-platform image with all tags. Much faster than QEMU for arm64 builds that compile code.
  Check names: `<caller job> / build-native (linux/amd64)`, `<caller job> / build-native (linux/arm64)`,
  `<caller job> / merge`. With `push: false` only the per-platform builds run.

**Replaced images are deleted right away** (`cleanup-replaced`, default `true`, ghcr.io images only): before pushing,
the workflow records the digest each tag points to; after pushing, every such image that is not the new one and has
**no tag left** is deleted, together with its platform and attestation manifests unless the new image or another tagged
image still references them (with reproducible builds a platform manifest can be the same between builds). So the
previous `pr-<n>` image disappears as soon as a new push to the PR replaces it, while moving `edge`, `latest` or
`stable` deletes nothing as long as every build also passes `version`: the previous image keeps its version tag.
An image that carries **only** moving tags (e.g. a branch build tagged just `main`) is deleted when the next push
moves its last tag; pass `version` or set `cleanup-replaced: false` if such images must be kept. With `native-runners`, the per-platform
images pushed by digest are deleted after merging as well (their manifests are part of the new image). Every deletion
is logged in the step "Delete replaced images"; a failure there only produces a warning, never a failed build.

Required permissions of the calling job: `contents: read`, `packages: write` (also with `push: false`).
`cleanup-replaced` uses the same `GITHUB_TOKEN`, which can delete versions of packages linked to the repository
(Admin role in the package's "Manage Actions access" settings, which is the default for a package that the
repository's workflow pushed first). Without that access it only warns.

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
| `native-runners` | `false`                  | Build each platform on a native runner and merge (see above). Only `linux/amd64` and `linux/arm64`. |
| `cleanup-replaced` | `true`                 | Delete the ghcr.io images this push replaced (see above) |
| `build-args`    | –                         | `KEY=value`, one per line |
| `push`          | `true`                    | `false` only checks that the image builds (e.g. PRs from forks) |
| `cache-scope`   | last path segment of the first image | GitHub Actions cache scope. With `native-runners` it also names the digest artifacts, so it must differ between calls in the same workflow run (the default does for different images). |
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
      # native-runners: true # arm64 on its own runner instead of QEMU

  prune-prerelease-images:
    # manual release mode: after a full release, the edge images are no longer needed
    if: ${{ !github.event.release.prerelease }}
    needs: build
    uses: Aam-Digital/.github/.github/workflows/image-prune.yml@v1
    permissions:
      packages: write
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

(`cleanup-replaced` only covers the ghcr.io image; Docker Hub tags are left as they are.)

## `release.yml`

Creates the next release or prerelease with semantic-release
([`cycjimmy/semantic-release-action`](https://github.com/cycjimmy/semantic-release-action)). Only the semantic-release
`branches` setting comes from this workflow (the caller's default branch); the plugins and everything else stay in the
repository's own configuration (`.releaserc*`, `release.config.*` or `release` in package.json), including `tagFormat`
(which must end with the version). Releases are only created from the default branch; on any other ref the job does
nothing (`published: false`).

- `type: release`: a full release of the default branch (`branches: [{name: <default>, prerelease: false}]`).
  With `cleanup-prereleases` (default), the GitHub prereleases and git tags matching `*-<prerelease-id>.*` or
  `*-master.*` are deleted first.
- `type: prerelease`: `<next version>-<prerelease-id>.<n>`, e.g. `1.4.0-edge.3`. semantic-release cannot configure the
  default branch alone as a prerelease branch (it requires an existing release branch next to it), so a **dry run**
  with the default branch as release branch computes the next version and release notes, and this workflow creates
  the tag and the GitHub prerelease (with the PAT, so that `release: published` builds the image). `<n>` continues
  after the highest existing `<next version>-<prerelease-id>.*` tag. If the commit already has such a prerelease tag
  (e.g. a re-run), nothing is created. Unlike semantic-release's own prerelease branches, which only release when
  there are releasable commits since the last *prerelease*, this compares with the last *full* release: once there is
  a releasable change, every push to the default branch creates a new prerelease. The repository's `prepare`/`publish`/`success` plugins run only for full
  releases, so prereleases do not comment on issues or PRs.

Required permissions of the calling job: `contents: read`. Everything that writes uses the `token` secret
(PAT with `repo` scope, or a fine-grained token with contents (and, if the plugins comment, issues and pull requests)
write access). Releases of one repository and type are serialised (concurrency group, never cancelled).

| Input                 | Default  | Description |
| --------------------- | -------- | ----------- |
| `type`                | (required) | `release` or `prerelease` |
| `prerelease-id`       | `edge`   | Prerelease identifier |
| `cleanup-prereleases` | `true`   | Only for `type: release`: delete the `*-<prerelease-id>.*` and `*-master.*` prereleases and tags first |
| `extra-plugins`       | –        | Plugins to install that the configuration uses, one per line (e.g. `@semantic-release/git`, `semantic-release-replace-plugin@1.2.6`) |
| `node-version-file`   | `.nvmrc` | Node.js version for semantic-release; if the file does not exist, the latest LTS. If package.json has `"packageManager": "npm@x.y.z"`, that npm is installed. |

| Secret          | Description |
| --------------- | ----------- |
| `token`         | (required) PAT for tags, releases (and comments). Not the `GITHUB_TOKEN`. |
| `slack-webhook` | Optional, available to plugins as `SLACK_WEBHOOK` (e.g. `semantic-release-slack-bot`) |

Outputs: `version` (the released version, empty if none), `published` (`true`/`false`).

### Example: manual release mode

```yaml
# .github/workflows/on-push.yml (part)
on:
  push:
    branches: [main]

jobs:
  # ... tests
  prerelease:
    needs: [test]
    uses: Aam-Digital/.github/.github/workflows/release.yml@v1
    permissions:
      contents: read
    with:
      type: prerelease
    secrets:
      token: ${{ secrets.GH_TOKEN }}
```

```yaml
# .github/workflows/release.yml
on:
  workflow_dispatch:

jobs:
  release:
    uses: Aam-Digital/.github/.github/workflows/release.yml@v1
    permissions:
      contents: read
    with:
      type: release
      extra-plugins: |
        semantic-release-slack-bot
    secrets:
      token: ${{ secrets.GH_TOKEN }}
      slack-webhook: ${{ secrets.SLACK_WEBHOOK_URL }}
```

### Example: auto release mode

```yaml
on:
  push:
    branches: [main]

jobs:
  # ... tests
  release:
    needs: [test]
    uses: Aam-Digital/.github/.github/workflows/release.yml@v1
    permissions:
      contents: read
    with:
      type: release
    secrets:
      token: ${{ secrets.GH_TOKEN }}
```

## `image-prune.yml`

Deletes old prerelease images from ghcr.io, typically after a full release (see the release example under
`docker-image.yml`), using [`dataaxiom/ghcr-cleanup-action`](https://github.com/dataaxiom/ghcr-cleanup-action) with
wildcard tags. Like `pr-image-cleanup.yml` it is manifest-aware (children of a deleted image are deleted unless another
image uses them). An image that carries one of the `exclude-tags`, e.g. the image `edge` currently points to, is
neither deleted nor untagged, even if another of its tags matches. A version whose tags all match but are several
(e.g. `1.4.0-edge.3` and `1.4.0-master.2`) is only untagged by the action. Untagged versions are never touched.

Required permissions of the calling job: `packages: write` (same token rules as `pr-image-cleanup.yml`).

| Input          | Default                      | Description |
| -------------- | ---------------------------- | ----------- |
| `packages`     | repository name              | ghcr.io package name(s), comma-separated |
| `tag-patterns` | `*-edge.*,*-master.*`        | Tags to delete, comma-separated, `*` wildcards. Every pattern must contain a letter or digit. |
| `exclude-tags` | `edge,master,stable,latest`  | Images with one of these tags (wildcards allowed) are never deleted or untagged |
| `dry-run`      | `false`                      | Only log what would be deleted |

Secret `token`: optional, as in `pr-image-cleanup.yml`. Docker Hub images are not covered.

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

Note: this deletes the image that is tagged `pr-<n>` when the PR closes. The images of earlier pushes to the PR are
deleted by `docker-image.yml` (`cleanup-replaced`) as soon as they are replaced.

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
like this (`gh` logged in with push access to this repository):

```shell
sha=$(gh api repos/Aam-Digital/.github/commits/main --jq .sha)
# first release: immutable v1.0.0 plus the moving v1 tag
gh release create v1.0.0 -R Aam-Digital/.github --target "$sha" --title v1.0.0 --generate-notes
gh api -X POST repos/Aam-Digital/.github/git/refs -f ref=refs/tags/v1 -f sha="$sha"
# every later release: the next v1.x.y, then move v1 to it
gh release create v1.1.0 -R Aam-Digital/.github --target "$sha" --title v1.1.0 --generate-notes
gh api -X PATCH repos/Aam-Digital/.github/git/refs/tags/v1 -f sha="$sha" -F force=true
```

Use a minor version for new inputs or workflows, a patch version for fixes. Until `v1` exists, a consumer PR can test
against a branch of this repository (`@<branch>`), but must switch to `@v1` before it is merged.

Changes that are compatible with all existing callers may go into `v1`: bug fixes, new optional inputs/outputs,
new workflows, action updates. Everything else needs a new major tag (`v2`), and the consumers are switched one by one:

- removing or renaming an input, output or secret, or changing an input's default or meaning;
- renaming a job id or job `name` (this changes check names, see above);
- requiring more permissions from the caller;
- changing which image tags are produced for the same inputs.

Rules for the workflows in this folder:

- Keep shell inline. Do not fetch or run scripts from this repository at runtime.
- Use fully qualified `uses:` references (`owner/repo/...@ref`) in callers. A local `./` reference is ambiguous once the
  workflow is called from another repository (for a step's action it points into the caller's checkout).
- Steps that several jobs of one workflow share are written once and reused with YAML anchors (`&name` / `*name`).
- Actions of large trusted providers (`actions/*`, `docker/*`, `github/*`, …) use their plain major tag.
  Other actions are pinned to a commit SHA with a `# vX.Y.Z` comment.
- Pass inputs to `run:` scripts via `env:`, never by `${{ }}` interpolation into the script.
