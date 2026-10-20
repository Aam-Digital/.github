# Renovate presets

Shared [Renovate](https://docs.renovatebot.com/) presets for the repositories of the Aam-Digital organisation.
A repository keeps only what is specific to it in its own `renovate.json` and extends these presets.

| Preset | What it does |
| --- | --- |
| [`renovate/default`](default.json) | Base for every repo: `config:recommended`, all minor/patch updates in one `weekly-non-major-updates` group, Sundays (UTC), no dependency dashboard, max. 5 open PRs / 2 per hour, vulnerability and OSV alerts (any time, labelled `security`), label `dependencies`, reviewer, major updates as draft PRs. All GitHub Actions and reusable workflow updates (majors included) in one ready `github-actions` PR. Updates to this repo's own reusable workflows in a separate `aam-digital-shared-workflows` group, never pinned to a digest. |
| [`renovate/node`](node.json) | Node.js repos: `rangeStrategy: bump`, npm `^11.0.0` constraint (the npm major that comes with the Node.js major in `.nvmrc`; change both together), monthly lock file maintenance, and one `node` group for `.nvmrc`, Dockerfile `FROM node:` and `node-version` inputs in workflows. |
| [`renovate/docker`](docker.json) | Repos that build Docker images: minor/patch base image updates in a `docker-base-image` group (the `node` image stays in the `node` group). |

## Usage

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "local>Aam-Digital/.github//renovate/default",
    "local>Aam-Digital/.github//renovate/node",
    "local>Aam-Digital/.github//renovate/docker"
  ],
  "enabledManagers": ["npm", "nvm", "dockerfile", "github-actions"]
}
```

- Always extend `renovate/default` first; `node` and `docker` are add-ons.
- If the repo sets `enabledManagers`, list every manager the presets rely on, e.g. `nvm` for `.nvmrc` with `renovate/node`. Otherwise that part is silently skipped.
- Renovate reads presets from the **default branch** of this repo. A change here takes effect for all repos on their next Renovate run, so validate it before merging.
- The Renovate GitHub app must have access to this repository. Otherwise every repo that extends a preset fails to load its config.
- The local `renovate-config-validator` does not fetch remote presets. Check the result of a change in the Renovate run log (Mend developer portal) of a repo that extends it: the resolved config and the branch each update is assigned to.

## How repo settings combine with the presets

- **`packageRules`** are concatenated: the presets' rules come first, then the repo's own. For each option, the last matching rule wins, so a repo rule overrides a preset rule.
- **Most top-level options**, such as `labels`, `reviewers`, `schedule`, `timezone`, `rangeStrategy` and `enabledManagers`, are replaced as a whole by the repo's value (arrays are not merged).
- A few list options are **concatenated** instead: `packageRules`, `customManagers`, `ignoreDeps`, `addLabels` and `additionalReviewers`.
- **Object options** such as `vulnerabilityAlerts`, `lockFileMaintenance` and `constraints` are merged key by key: the repo only needs to set the keys it changes. Setting `vulnerabilityAlerts.labels` therefore replaces only those labels.
- To add reviewers without repeating the preset's, use `additionalReviewers` (it is added to `reviewers`).
- **`labels` in a package rule beat the top-level `labels`.** To put an extra label on some PRs, use `addLabels` in the rule. It adds to whatever labels apply instead of replacing them. For the same reason, the presets set no `labels` inside package rules.
