# Contributing to Aam Digital

_Aam Digital_ is built in the open because we believe in building things together
and letting people benefit from and adapt things rather than everybody reinventing their own wheel.

To build great software for small social impact organisations, we welcome anybody willing to contribute.
We are a small core team of full-time developers as well as a few regular volunteer contributors.
Whether you want to extend our code for your own use case or just help out -
we welcome any contributions to make this project better!

This is the default contribution guide for the repositories of the Aam-Digital organisation.
A repository can replace it with its own `CONTRIBUTING.md`; its README describes how to run, test and lint it.

---

## Get started

- The repository's **README** explains how to run it locally and how to run its tests.
- The [Developer Documentation](https://aam-digital.github.io/ndb-core/documentation/additional-documentation/overview.html)
  of the main application gives an overview of the architecture, and its
  [Tutorial](https://aam-digital.github.io/ndb-core/documentation/additional-documentation/tutorial.html)
  walks you through the setup and the technologies involved.
- Don't hesitate to ask questions!

## Get in touch

Comment on an issue, or open one yourself, also to ask a question.
For more general discussions or questions, use the [Discussions section](https://github.com/orgs/Aam-Digital/discussions).
Within the team we also use Slack. Write us an e-mail to get an invitation to our workspace.

Security vulnerabilities are not reported through issues; see the [security policy](https://github.com/Aam-Digital/.github/blob/main/SECURITY.md).

## Where can I help?

Have a look at our issues. Some of them are marked as suitable for a new contributor:
[Community Help Wanted](https://github.com/orgs/Aam-Digital/projects/2/views/4?filterQuery=label%3A%22Status%3A+Community+Help+Wanted+%28good+first+issue%29%22+).

Before you start working on an issue, please always comment on it first to help us coordinate.
We will then assign you to the issue and set its status in the project to "In Progress"
(as an outside contributor you may not have the permission to assign yourself at first).

We organise our work in the [Project Board](https://github.com/orgs/Aam-Digital/projects/2),
which covers the issues of all our repositories:

- Issues move through the columns of the board based on their status, and are sorted by priority (most important on top).
- Please do not work on issues in the "Triage / Analysis" status: these still need a clearer definition and
  approval from the core team.
- Issues have a type (Bug, New Feature, Task, ...); labels add a status or area.

## Workflow

We follow the [GitHub Flow](https://docs.github.com/en/get-started/using-github/github-flow): every change is made on its
own branch and merged into the default branch through a pull request.

1. Discuss the bug or feature in its issue first, including possible approaches.
2. Create a **new branch** from the current default branch for each issue, with a descriptive name.
   Do not use the same branch for several features or fixes.
3. Make commits on this branch with descriptive messages.
4. Run the repository's tests and linting (see its README) and add tests for your change where it makes sense.
   All checks must pass before your contribution can be accepted.
5. Open a **pull request** once you are done or want feedback. Use the "Draft" state while it is still work in progress.
   - The **PR title** must follow [Conventional Commits](https://www.conventionalcommits.org/)
     (e.g. `fix(export): handle empty values`, `feat!: ...` for a breaking change).
     PRs are squash-merged with the PR title as the commit message, and the next version number and the
     release notes are derived from these titles.
   - Reference the issue in the description (e.g. `closes #15`).
6. A **different developer reviews** your PR. Discussions about the code take place in the PR; please follow it and
   explain or adapt your code where necessary.
   For repositories that build a container image, each PR gets an image tagged `pr-<number>` for testing.
7. A core team member merges the PR after the review. The branch is deleted automatically.
   Depending on the repository, each merge into the default branch is either released right away, or published as a
   prerelease (e.g. `1.4.0-edge.3`, image tag `edge`) until the team creates the next full release
   (image tags `<version>`, `latest` and `stable`).

## License

Please note that by contributing, you make your work available under the license of the repository you
contribute to (see its `LICENSE` file).
