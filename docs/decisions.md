# Project Decisions

This document records significant architectural and design decisions made throughout the project's development.

## 2026-09-02: Job-sized reusable workflows with zero inputs; CD reusable but amd64-only

Extend this repository beyond `security.yml` with reusable workflows for the jobs that are identical across the org's Go repositories: `lint.yml`, `test.yml`, `compatibility.yml`, `build.yml` (CI-time docker build, no push), and `cd.yml` (docker publish to ghcr.io). Each is a job-sized unit that a repository's own `ci.yml`/`cd.yml`/`compatibility.yml` composes; repo-specific jobs stay inline in the caller.

Context: a survey of 24 local repositories with `ci.yml` (15 with `compatibility.yml`, 11 with `cd.yml`) showed the lint job is byte-identical modulo drift (older action versions, a missing dependabot guard, one missing setup-go); the test and compatibility jobs are identical in the 12 and 8 repositories that need no `services:`; 10 repositories share an identical arm64+amd64 CI docker build pair; and the 11 CD files share one skeleton differing on six axes (QEMU, platforms, checkout presence, `context:`, `build-args`, env indirection).

Decisions:

- **Zero inputs everywhere.** The first-wave callers need none: the only test-command variation among service-free repositories is a `-coverprofile` flag nothing consumes. Fewer inputs is the boring choice; an input is added when a second repository needs it, not before.
- **No secrets either.** A called workflow inherits neither secrets nor the caller's `env:`. gai's test and compatibility jobs need provider API keys (and a Vertex credentials file) for live client tests, so they stay inline; gai proves `lint.yml` only, and the secret-free, service-free repositories are the first callers of `test.yml` and `compatibility.yml`. Declaring gai's key names as `secrets:` on a generic workflow was rejected as leaking one repository's needs into all.
- **Services stay inline.** A called workflow cannot accept a `services:` block from its caller, and canned service recipes behind boolean inputs were judged too much machinery. Repositories with postgres/versitygw/minio/elasticmq keep an inline test job (and compatibility job) for now.
- **CD is reusable, not template-only**, after weighing its six axes: `platforms` collapses to `linux/amd64` only going forward (no QEMU; the six repositories publishing arm64 images lose them on conversion, accepted); `context: .` is universal because without it Go's `vcs.revision` stamp is missing; `build-args` is a single repository (golang.dk) and stays inline; the env indirection was cosmetic. Callers carry `push: main` and `contents: read, packages: write`.
- **Permissions and trust.** As with `security.yml`, no reusable workflow declares `permissions:`; callers grant them. The called workflow runs with the caller's `GITHUB_TOKEN`, so `cd.yml` publishes to the caller's own registry. Whatever is on this repository's `main` runs with every caller's grant — for CD that includes `packages: write`. Judged no worse than the trust already extended to third-party actions pinned by floating tag; `main` is branch-protected and has a single committer.
- **`persist-credentials: false`** on every checkout, normalizing the repositories that lacked it.
- **`-race` on every test run** in `test.yml` and `compatibility.yml`. None of the surveyed repositories used it; the cost is CPU time and memory on hosted runners, the benefit is finding real data races in the exercised paths. Converting repositories may surface latent races as new failures, which is the intended effect.
- **Pinning stays `@main`.** A bad push here breaks up to 24 repositories' CI at once; accepted for a single-maintainer org, and the failure is loud.
- **Starter templates** in `maragudk/.github`: `ci.yml` (lint + test, with the `build` job present but commented out), `compatibility.yml`, `cd.yml`. Repositories convert on their own schedule from the templates; gai gets a conversion PR as the proving caller for lint/test/compatibility, left for a human to merge.

Rejected: one reusable CI pipeline with many inputs (every repository's `ci.yml` differs in which jobs it has); `secrets: inherit` (nothing in the first wave needs secrets, and explicit declarations keep the surface auditable); a `platforms` input on `cd.yml` (kept amd64-only instead); CD as template-only (the six axes turned out to be one decision, two fixes, and two single-repository quirks).
