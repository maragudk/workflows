# workflows

Reusable GitHub Actions workflows.

## Security

Runs `govulncheck` on the calling repository. On a failed run on `main` it opens one GitHub issue labelled `govulncheck`, leaving it alone if one is already open, and closes it on the next green run on `main`.

Use the [Security starter workflow](https://github.com/maragudk/.github/blob/main/workflow-templates/security.yml) to call it.

Made with ✨sparkles✨ by [maragu](https://www.maragu.dev/): independent software consulting for cloud-native Go apps & AI engineering.

[Contact me at markus@maragu.dk](mailto:markus@maragu.dk) for consulting work, or perhaps an invoice to support this project?
