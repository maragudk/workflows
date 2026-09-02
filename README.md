# workflows

Reusable GitHub Actions workflows for the Maragu org. Each one declares `on: workflow_call`, so other repositories call it instead of keeping a copy-pasted version. Roughly half the org's repositories carry their own `security.yml`; those copies are being replaced by one-job callers of the `Security` workflow below, which is so far the only one here. This repository has to stay public: a private one is callable only from other private repositories.

## Security

Runs `govulncheck` over the calling repository, which must be a Go module rooted at the repository root. On a failed run on `main` it opens a single GitHub issue, deduplicated by label so that later failures leave the open issue alone, and closes that issue on the next green run on `main`.

| Input | Type | Default | Description |
| --- | --- | --- | --- |
| `go-version` | string | `stable` | Go version passed to `actions/setup-go`. |

The issue carries the `govulncheck` label, which the workflow creates if it is missing. Nothing else may use that label: every open issue carrying it counts as the same report, so a collision both suppresses new reports and closes unrelated issues.

### Caller

Add this as `.github/workflows/security.yml` in the calling repository:

```yaml
name: Security

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main
  schedule:
    - cron: "14 7 * * *"
  workflow_dispatch:

concurrency:
  group: ${{ github.workflow }}-${{ github.ref_name }}
  cancel-in-progress: ${{ github.ref_name != 'main' }}

permissions:
  contents: read
  issues: write

jobs:
  govulncheck:
    uses: maragudk/workflows/.github/workflows/security.yml@main
```

Inside the called workflow the `github` context resolves to the caller, so issues are filed in the calling repository.

### Rules the caller carries

- Triggers are not inherited. The called workflow has no `schedule` of its own, so the caller declares its `push`, `pull_request`, `schedule` and `workflow_dispatch`.
- Permissions are not inherited from anywhere else, so the caller must grant `contents: read` and `issues: write`; with less, the `gh` steps fail. The reusable workflow declares no `permissions` of its own on purpose: a called workflow that requests more than the caller's effective grant fails validation before any step runs, which would turn every fork pull request red.
- Keep the `concurrency` block. The deduplication reads the open issues and then creates one, so without a shared group a `push` racing the nightly cron can file the same issue twice.
- The issue steps run only when the branch is `main`. A repository whose default branch has another name still gets the scan, but never gets an issue.
- Pin `@main`. Dependabot ignores branch refs, so there is no bump noise; switch to tags if per-repo control over when changes land is ever wanted. Every caller runs whatever sits on this repository's `main` with an `issues: write` token, so protect that branch.

Migrating an existing `security.yml` renames its status check: a caller job named `govulncheck` reports as `govulncheck / govulncheck`. Update any branch protection rule that requires the old name.

Made with ✨sparkles✨ by [maragu](https://www.maragu.dev/): independent software consulting for cloud-native Go apps & AI engineering.

[Contact me at markus@maragu.dk](mailto:markus@maragu.dk) for consulting work, or perhaps an invoice to support this project?
