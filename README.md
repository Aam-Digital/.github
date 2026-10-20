# Aam-Digital/.github

Shared configuration for the repositories of the Aam-Digital organisation.
(The organisation profile shown on GitHub lives in [`profile/README.md`](profile/README.md).)

| Folder / file | What |
| --- | --- |
| [`.github/workflows/`](.github/workflows) | Reusable GitHub Actions workflows (container images, releases, image cleanup, PR title and workflow linting). See its [README](.github/workflows/README.md) for how to call them, the release model and how `v1` is moved. |
| [`renovate/`](renovate) | Shared Renovate presets. See its [README](renovate/README.md) for the presets, how to extend them and how repo settings combine with them. |
| [`renovate.json`](renovate.json) | Renovate config of this repository: keeps the actions used by the shared workflows up to date. After a merge, CI moves `v1`, so the updates reach every repository without a release step. |
| [`.github/ISSUE_TEMPLATE/`](.github/ISSUE_TEMPLATE), [`SECURITY.md`](SECURITY.md), [`CONTRIBUTING.md`](CONTRIBUTING.md) | Organisation-wide defaults (below) |

## Default community health files

GitHub uses the issue templates, `SECURITY.md` and `CONTRIBUTING.md` of this repository for every repository of the
organisation that has no file of its own (public and private ones). A repository with its own
`.github/ISSUE_TEMPLATE/` folder (e.g. ndb-core) uses only its own templates.

- The issue templates (Bug report, Feature request, Technical task) set the organisation's **issue types**
  (`Bug`, `New Feature`, `Task`) instead of labels, because labels differ between repositories.
- There is deliberately no default pull request template: PR templates are repository-specific.
