# Diary: Add reusable `lint`, `test`, `compatibility`, `build`, and `cd` workflows

With `security.yml` proven on gai (maragudk/gai#351), extend this repository with job-sized reusable workflows for the rest of the org's common CI, plus matching starter templates in `maragudk/.github`. Design recorded in `/docs/decisions.md` (2026-09-02).

## Step 1: Survey the variation and settle the design

**Author:** main

### Prompt Context

**Verbatim prompt:** "And if that all works, do the same for the compatibility, CI, and CD workflows. Don't merge anything yourself." Then, one unit at a time via `/fabrik:one-at-a-time`: "yes" (lint); "Let's discuss the inputs" → "yes, zero inputs" (test); "yes" (compatibility); "yes, just call it \"build\"" (docker build); "Let's discuss that one" → "- No qemu, linux/amd64 only going forward - context included always - No build args / Any other divergences?" → "Definitely keep the persist-credentials fix. Also, don't convert any repos yourself. We need to talk through the CD permission stuff, I don't understand." → "Ah, right. Yeah, my repo, not worse than any other repo exposed for whatever reason." (cd); "Approved, commented-out job" (templates); "yes, keep @main"; "b" (gai conversion PR, unmerged).
**Interpretation:** Generalize the Security pattern to the other shared jobs, with Markus deciding each workflow's shape individually.
**Inferred intent:** One maintained copy of each common CI job across ~24 repositories, without a mega-workflow and without me touching the other repositories.

### What I did

A research agent read every `.github/workflows/{ci,cd,compatibility}.yml` under `/Users/maragubot/Developer/` (24, 11, and 15 files) and produced a variation matrix: the lint job identical modulo drift; test/compatibility identical wherever no `services:` are needed; a shared arm64+amd64 docker build pair in 10 repos; CD sharing a skeleton with six axes of variation. Walked the design with Markus one decision at a time: five reusable workflows, all zero inputs; services stay inline; CD amd64-only with `context: .` and no build args; `persist-credentials: false` everywhere; callers carry triggers, concurrency, permissions; `@main`; starter templates `ci.yml` (build job commented out), `compatibility.yml`, `cd.yml`; gai gets a conversion PR left unmerged. Explained the CD token mechanics (called workflow runs with the caller's `GITHUB_TOKEN` and grant; the only concern is trust in this repo's `main`, which Markus judged acceptable). Recorded the decision, started this diary, delegated to two builders in parallel: one here, one in `maragudk/.github`.

### Why

The survey turned "do the same for CI" from a vague ask into a list of jobs with known variation; each zero-input decision rests on a count of actual callers, not a guess.

### What worked

Presenting one decision per message made the two genuinely contested points (test inputs, CD) get real discussion instead of being nodded through in a list.

### What didn't work

Nothing failed in this step.

### What I learned

Three of CD's six "axes" were fixes in disguise (missing `context: .`, missing checkout, cosmetic env indirection); only platforms was a real choice, and it was made by fiat rather than by input.

### What was tricky

Explaining reusable-workflow permissions without jargon: the caller's token, the caller's grant, the called workflow only able to inherit or reduce — and separating that mechanical fact from the trust question about this repo's `main`.

### What warrants review

After the builders' changes: five reusable workflows with no `inputs:`, no `permissions:`, no `concurrency:`, no triggers beyond `workflow_call`; `persist-credentials: false` on every checkout; `cd.yml` with `platforms: linux/amd64`, `context: .`, `latest` + sha tags, GHA cache; three templates in `.github` using `$default-branch`; the gai conversion PR's checks green under the reusable jobs.

### Future work

Repositories convert from the templates on their own schedule (not by this session). Repositories with `services:` keep inline test/compatibility jobs until a services story exists. golang.dk keeps its inline CD for the `RELEASE` build arg.

## Step 2: Write the five reusable workflows

**Author:** reusable-builder

### Prompt Context

**Verbatim prompt:** "Requirements — five files in `.github/workflows/`, each `on: workflow_call:` with NO inputs, NO permissions block, NO concurrency, NO other triggers; `persist-credentials: false` on every checkout … 1. `lint.yml` — job `lint`, `ubuntu-latest`: checkout; setup-go (`go-version-file: go.mod`, `check-latest: true`); `golangci/golangci-lint-action@v9` with `version: latest`. Keep the dependabot guard `if: ${{ github.triggering_actor != 'dependabot[bot]' }}` on the job as gai has it. … Each file gets a short header comment in the style of security.yml stating which permissions the caller must grant … Do not add a README section per workflow — the README is deliberately a two-sentence pointer; leave it alone."
**Interpretation:** Extract five job-sized workflows verbatim from the reference repositories, wrap each in the `security.yml` shape, and change nothing that the design decisions do not call for.
**Inferred intent:** Callers should be able to swap an inline job for a `uses:` line and get byte-identical behaviour, so every deviation from the source has to be a decision someone already made.

### What I did

Branched `reusable-ci-workflows` from `main` and wrote `/github/workflows/{lint,test,compatibility,build,cd}.yml`. Each carries `name:`, `on: workflow_call:`, and a header comment naming the caller-granted permissions, then the job copied from its source: `lint` and `test` from gai's `ci.yml`, the four-legged matrix from gai's `compatibility.yml`, the `build-arm64`/`build-amd64` pair from app's `ci.yml`, and `publish` from app's `cd.yml`. The repository-specific parts of the sources were dropped: the GCP credentials step and the `ANTHROPIC_KEY`/`GOOGLE_KEY`/`GOOGLE_VERTEX_KEY`/`OPENAI_KEY` env blocks in gai's test and compatibility jobs, app's `-tags sqlite_fts5,sqlite_math_functions` and its `services:`, and app's `permissions:`, triggers, and `concurrency:`.

Validated with `go run github.com/rhysd/actionlint/cmd/actionlint@v1.7.12` (exit 0), a `yaml.safe_load` of each file, and a grep for `inputs:|permissions:|concurrency:|qemu|build-args` across the five files (no matches, exit 1) plus a check that all six checkouts carry `persist-credentials: false`.

Self-reviewed by dispatching two competing reviewers over the diff, the decision record, `security.yml`, and the four source files. Every change they produced was to a comment; no workflow logic moved. The header comments originally reused `security.yml`'s justification — "a called workflow that requests more than the caller's effective grant fails validation outright, which would break fork pull requests" — which is only true where the workflow wants a write permission; for four `contents: read` workflows nothing would break, and `cd.yml` is never called from a fork pull request at all. All five now say the accurate, general thing instead: a called workflow can only narrow the caller's grant, so permissions are the caller's to declare. `cd.yml`'s header gained a second paragraph stating that callers must only call it on a push to the default branch, because `push: true` is unconditional and on a pull request `github.sha` is the ephemeral merge commit, which would tag an image against a commit that is not in the repository. `lint.yml`'s dependabot guard got two lines explaining both why it is there and that the resulting `skipped` cascades into any caller job with `needs: lint`. The `context: .` comment copied from app's `cd.yml` mentioned `service.version`, an OTel attribute belonging to a calling application and meaningless to a reader of this repository; both files now give the reason that holds here — the default Git-URL context drops `.git`, so `go build` cannot stamp `vcs.revision`.

### Why

Copying faithfully is what makes the conversion safe: a caller can read its old job next to the reusable one and see that nothing moved. Every deviation is therefore either in the decision record (`context: .`, `persist-credentials: false`, amd64-only CD, no inputs, no permissions) or a comment.

### What didn't work

Nothing failed outright. `actionlint` was green on the first run and stayed green through both comment passes.

### What I learned

Passing setup-go both `go-version` and `go-version-file` from the compatibility matrix, where one is always the empty string, is not the accident it looks like: an unset object key renders as `''`, `core.getInput` returns `''` for it, and setup-go falls through to the other input without emitting its "both specified" warning. The gai matrix is correct as written and copying it literally was the right call.

### What was tricky

Deciding where "copy the source faithfully" stops. Both reviewers independently flagged the same three places where the source is arguably wrong — the GHA cache scope collision between `build-arm64` and `build-amd64` (both write the default `buildkit` scope and evict each other, and `cd.yml` joins them on `main`), `ghcr.io/${{ github.repository }}` not being lowercased for a repository name with a capital letter, and `push: true` having no ref guard — and only the last was cheap enough to address with a comment rather than a behaviour change. The other two are reported to the lead rather than fixed, because the requirements pinned the cache and tag lines exactly, and both are inherited defects rather than something this change introduced.

### What warrants review

The header comment wording in all five files, since it now diverges from `security.yml`'s and arguably `security.yml` should follow. The job-level `name:` fields from the sources (`Lint`, `Test`, `Build arm64`, `Publish`) were deliberately dropped to match `security.yml`, which means converted repositories will see check names change from `Lint` to `lint / lint`, and branch protection rules that name the old checks will need updating. And the secrets question in "Future work" below, which decides whether gai can be the proving caller for `test.yml` and `compatibility.yml` at all.

### Future work

gai's test and compatibility jobs read `ANTHROPIC_KEY`, `GOOGLE_KEY`, `GOOGLE_VERTEX_KEY`, and `OPENAI_KEY` from the environment, plus a `Set up GCP credentials` step that writes `GOOGLE_VERTEX_CREDENTIALS_PATH`. A called workflow inherits no secrets, and the design rejected `secrets: inherit`, so gai converting to `test.yml`/`compatibility.yml` would run its client tests with empty keys. Either gai is the proving caller for `lint.yml` only, or those two workflows need declared secrets — a decision for the lead.

Two source-inherited weaknesses worth a follow-up: `cache-to: type=gha,mode=max` uses the default scope in all three build steps, so `build-arm64` and `build-amd64` evict each other, and `ghcr.io/${{ github.repository }}` breaks for any repository name with a capital letter. One reviewer proposed collapsing `build.yml`'s two jobs into a `matrix.include` over `{runner, platform}`, which would also give each leg a natural cache scope; kept as two jobs because the requirements named `build-arm64` and `build-amd64` as job ids, and a matrix would rename the checks.

## Step 3: Review the builder's output and settle the gai secrets gap

**Author:** main

### Prompt Context

**Verbatim prompt:** (No new prompt; the builder's report raised four open questions.)
**Interpretation:** Decide each before opening the PR.
**Inferred intent:** Ship the five workflows as designed, correcting the design record where the builder's review proved it wrong.

### What I did

Reviewed all five files against the decision record: `workflow_call` only, no inputs, no permissions, no concurrency, `persist-credentials: false` on every checkout, `cd.yml` amd64-only with `context: .`. Decisions on the builder's questions: (1) gai's test and compatibility jobs need provider secrets and a Vertex credentials step, which a called workflow cannot receive without `secrets:` declarations — so gai keeps those two jobs inline and proves `lint.yml` only; the decision record gained a "No secrets either" bullet, since its "nothing in the first wave needs secrets" was wrong for gai. (2) The bare `type=gha` cache scope shared by the two build legs is left as-is: it is today's behaviour in ten repositories, and a `scope=` per leg is a cheap later fix if eviction shows up. (3) Job `name:`s stay dropped, matching `security.yml`; checks render as `lint / lint`, and conversion PRs must mention the branch-protection rename. (4) `ghcr.io/${{ github.repository }}` is not lowercased; every org repository is lowercase, noted and left.

### Why

Each call keeps the workflows generic; gai's needs are gai's, and the two review-flagged behaviours that were inherited from working repositories are not regressions.

### What worked

The builder's two competing reviewers independently caught the secrets gap, which my survey had missed by looking only for `services:`.

### What didn't work

The design record was wrong on one point and is now amended rather than silently corrected.

### What I learned

"Which jobs are identical" needs to be asked of the job's environment (secrets, env, services), not just its steps.

### What was tricky

Deciding not to fix the cache-scope finding: it is a real improvement, but not a decision anyone made, and the brief's rule was that every deviation from source must be one.

### What warrants review

`/docs/decisions.md`'s new bullet; the gai conversion PR should touch only the lint job.

### Future work

`scope=` on the build cache legs if eviction is observed; a services and secrets story before the repositories that need them can convert test/compatibility.

## Step 4: Address Markus's review of PR #3

**Author:** main

### Prompt Context

**Verbatim prompt:** `/fabrik:second-opinion` then `/fabrik:address-code-review`; per comment: "Agree, and file an issue in the workflows repo to follow up"; "yes"; "yes"; "yes"; "yes, also existing workflow not part of this PR"; "yes"; "yes"; "ok"
**Interpretation:** Eight inline comments, all but two asking to delete explanatory comments from the workflow files; one about runner images; one raising the go.mod-vs-stable Go version question.
**Inferred intent:** Workflow files should be code, not prose — the reasoning lives in the decision record — and open questions become issues rather than in-PR debates.

### What I did

Ran `codex exec review --base main` (gpt-5.6-sol, xhigh) on both this PR and `maragudk/.github#2` first: no actionable findings on either. Then triaged the eight comments one at a time, replying to and resolving each. Outcomes: runners stay `ubuntu-24.04`/`-arm` because `ubuntu-26.04` is still marked preview in `actions/runner-images` (follow-up issue #4); all six header comments — the five new files and `security.yml`, which Markus asked to include — trimmed to the single sentence naming the caller's required grants; the three `context: .` comments, `cd.yml`'s second header paragraph about default-branch-only calls, and `lint.yml`'s dependabot-guard comment deleted; the go.mod-versus-stable question filed as issue #5 with the trade-off written out, no change. Applied the deletions with one script over `.github/workflows/*.yml`, re-ran actionlint locally (exit 0).

### Why

Every deleted comment restated something the decision record or the code already says; the two questions that were not about comments are real and belong in issues with room for discussion.

### What worked

The one-at-a-time triage made eight comments take eight short exchanges; most were a single word each way.

### What didn't work

Nothing failed.

### What I learned

Nothing new.

### What was tricky

Nothing.

### What warrants review

The six files' header comments are one sentence each and otherwise the files contain no comments; issues #4 and #5 exist.

### Future work

Issues #4 (Ubuntu 26.04 runners) and #5 (Go version policy for CI).

## Step 5: Add `-race`

**Author:** main

### Prompt Context

**Verbatim prompt:** "Should we add a -race flag to the tests? Any downsides?" then "add now"
**Interpretation:** Enable the race detector in the shared test and compatibility workflows as part of PR #3.
**Inferred intent:** A shared default is the cheapest place to raise the bar for every repository at once.

### What I did

Changed `go test -shuffle on ./...` to `go test -race -shuffle on ./...` in `/.github/workflows/test.yml` and `/.github/workflows/compatibility.yml`; added a bullet to `/docs/decisions.md`. Discussed downsides with Markus: slower and more memory (minor for network-bound suites, noticeable for pure-Go libraries), detection limited to exercised paths, and latent races in converting repositories showing up as new failures.

### Why

None of the 24 surveyed repositories ran the race detector; a zero-input shared workflow is exactly where such a default belongs, since it applies everywhere on the next run.

### What worked

The question arrived while the PR was still open, so the change is one line per file rather than a second PR.

### What didn't work

Nothing failed.

### What I learned

Nothing new.

### What was tricky

Nothing.

### What warrants review

The two `go test` lines; gai's inline test job is unchanged and does not use `-race`.

### Future work

Consider `-race` for gai's inline test job when its secrets story is settled.
