# workflows

Reusable GitHub Actions workflows (`on: workflow_call`) for Maragu repositories. This repository must stay public: a private one is callable only from private repositories.

## Security

Runs `govulncheck` on the calling repository. On a failed run on `main` it opens one GitHub issue labelled `govulncheck`, leaving it alone if one is already open, and closes it on the next green run on `main`. Nothing else may use that label.

Callers use the [Security starter workflow](https://github.com/maragudk/.github/blob/main/workflow-templates/security.yml), available under Actions → New workflow in every repository. The caller carries what a reusable workflow does not inherit:

- Triggers: `push`, `pull_request`, `schedule`, and `workflow_dispatch`.
- Permissions: `contents: read` and `issues: write`. The reusable workflow declares none, because a called workflow requesting more than the caller's effective grant fails validation on fork pull requests.
- `concurrency`, so a push racing the nightly run cannot file the same issue twice.
- Pinning to `@main`. Every caller runs whatever is on this repository's `main` with an `issues: write` token, so that branch is protected.

Made with ✨sparkles✨ by [maragu](https://www.maragu.dev/): independent software consulting for cloud-native Go apps & AI engineering.

[Contact me at markus@maragu.dk](mailto:markus@maragu.dk) for consulting work, or perhaps an invoice to support this project?
