# Diary: Create the shared `Security` reusable workflow

`maragudk/workflows` is the org's central repository for reusable GitHub Actions workflows (`on: workflow_call`). Its first workflow is `Security`: `govulncheck` plus a deduplicated GitHub issue on failure, ported from `maragudk/gai` (PRs #348 and #350 there), where the pattern was verified end-to-end on 2026-09-02. Roughly half of the org's repositories carry a copy-pasted `security.yml`; this repo replaces the copies with callers.

## Step 1: Decide the shape of the central repo

**Author:** main

### Prompt Context

**Verbatim prompt:** "Is it possible to have org-level shared workflows in Github? Half my projects have this security workflow." Then: "Merge 350. What's the idiomatic central repo? Launch subagent to research?" Then: "Yes, spawn the builder. I created a workflows repo, see ../workflows"
**Interpretation:** Centralize the `Security` workflow as a reusable workflow in a dedicated org repository, following the established GitHub convention, and convert `gai` to the first caller.
**Inferred intent:** Stop maintaining N copies of the same CI workflow; fixes and improvements land once and propagate.

### What I did

A research agent surveyed GitHub's documentation and real-world org layouts. Findings that shaped the design: reusable workflows must live flat in `.github/workflows/` of the central repo; the org `.github` repo can host them but its documented role is community-health files and starter workflows (copy-once templates, which would recreate the drift); a private central repo is callable only from private repos, so a mostly-public org needs a public one; the `github.*` context inside a called workflow resolves to the caller, so the issue-filing shell from `gai` needs no changes; `schedule` and `permissions` are not inherited, so each caller carries its own triggers and must grant `issues: write`; `@main` pinning suits a self-owned org (Dependabot ignores branch refs, so no bump noise). Closest analogue: `charmbracelet/meta`. Markus created `maragudk/workflows` (public, `main`, LICENSE and README only). Started this diary and delegated the port to a builder; `gai`'s conversion to a caller follows as a separate change once this repo's `main` has the workflow.

### Why

The pattern just proved itself in `gai` and is wanted everywhere; a reusable workflow is the mechanism GitHub provides for exactly this, and the dedicated-public-repo shape is what comparable Go orgs use.

### What worked

Researching before building: the private-repo access rule and the non-inheritance of `schedule`/`permissions` would each have cost a debugging round if discovered by trial.

### What didn't work

Nothing failed in this step.

### What I learned

Starter workflows and reusable workflows are different features with different homes; conflating them is the common mistake.

### What was tricky

Nothing technical; the only judgement call was `.github` vs a dedicated repo, settled by the awkward double path and the feature conflation above.

### What warrants review

After the builder's change: the reusable `security.yml` is the `gai` file with `on: workflow_call` and inputs in place of triggers, the issue-filing shell is byte-for-byte the verified version, the README's caller snippet declares `issues: write` and its own `schedule`, and `actionlint` is clean on the repo's own CI.

### Future work

Convert `gai` to the first caller (separate PR there), then `glue` and `app` (their `security.yml` is byte-identical to `gai`'s pre-#350 file); `gomponents` has no security workflow yet and would be a new adopter. `Conformance` and the issue steps as a composite action are candidates if a second consumer appears.

CI is the harder case and deliberately not attempted here: the repos' `ci.yml` files differ materially (`gomponents` runs a Go version matrix 1.18–1.27 with codecov and benchmark jobs; `app` has a docker build; `gai` has evaluate and lint jobs). The right shape, per `charmbracelet/meta`, is job-sized reusable workflows (`test.yml`, `lint.yml`, ...) that each repo's `ci.yml` composes, with repo-specific jobs staying inline in the caller — not one reusable CI pipeline with a dozen inputs.

## Step 2: Port `Security` into the reusable workflow, plus this repo's own CI

**Author:** workflows-builder

### Prompt Context

**Verbatim prompt:** "Work on a new branch `security-reusable-workflow` created from `main`. […] 1. `.github/workflows/security.yml` — the reusable workflow: `name: Security`, `on: workflow_call:` with inputs `go-version` (string, default `stable`) and `label` (string, default `govulncheck`). […] `permissions: contents: read, issues: write` at the workflow level (the caller must still grant them; document this). Job `govulncheck` with the same steps as gai's file […] Keep the shell byte-for-byte identical to gai's except for substituting `${{ inputs.label }}` for the literal `govulncheck` label (put it in an env var `LABEL` rather than interpolating into the script body — never inline `${{ }}` inside `run:`). Keep the dedup as an assignment, not a pipeline […] Keep the `github.ref_name == 'main'` gating on both issue steps (context resolves to the caller). Do NOT add `concurrency` or `schedule` here […] 2. `.github/workflows/ci.yml` — this repo's own CI […] run `actionlint` […] pick the boring one and pin a version. 3. `.github/dependabot.yml` […] 4. `README.md` […] 5. Do not add a Makefile, Go module, or anything else." Then, mid-run: "one correction to requirement 3: gai's dependabot.yml is the org's outlier, not the convention. Mirror /Users/maragubot/Developer/glue/.github/dependabot.yml instead (github-actions, directory "/", weekly, `cooldown: default-days: 7`; no ignore rules). Also for your information: the pre-#350 security.yml is byte-identical across gai, glue, and app, so no additional inputs are needed for per-repo variation — keep the input set minimal."
**Interpretation:** Lift `gai`'s verified `security.yml` into a `workflow_call` workflow with two inputs, stand up the minimum scaffolding this repo needs to maintain itself (actionlint CI, Dependabot), and write the README that tells a caller what to paste and what the call does not give them for free.
**Inferred intent:** One copy of the pattern, maintained centrally, with the caller's obligations written down — because the parts a reusable workflow does *not* inherit (`schedule`, `permissions`) are exactly the parts a copy-paste migration silently drops.

### What I did

Created `/.github/workflows/security.yml`, `/.github/workflows/ci.yml` and `/.github/dependabot.yml`, and replaced the two-line placeholder `/README.md`.

`security.yml` is `gai`'s file with the trigger block swapped for `on: workflow_call` and two inputs. I verified the port mechanically rather than by eye: parsing both files with `yaml.safe_load` and diffing the *parsed steps* shows exactly three differences across five steps — `go-version` now reads `${{ inputs.go-version }}`, and each of the two `gh` steps gained `LABEL: ${{ inputs.label }}` in `env:` with `"$LABEL"` replacing the three literal `govulncheck` label arguments in the shell. The checkout and `govulncheck` steps are byte-identical. `printf -v body`, the assignment-form dedup, the `--limit 30` close loop and both `if:` conditions came across untouched.

For the README I briefed a writer sub-agent rather than drafting in my own context, then reviewed and corrected what came back. The writer independently noticed that the brief's purpose line put `main` only on the closing half while the workflow gates both steps on it, and wrote the accurate version.

`ci.yml` runs `go run github.com/rhysd/actionlint/cmd/actionlint@v1.7.12` after `setup-go`. I chose that over `raven-actions/actionlint` or the Docker image because the org already has `setup-go` in every workflow and this adds no third-party action to a repository that is about to become a supply-chain root for the org. The trade-off is recorded under Future work.

Per the lead's mid-run correction, `dependabot.yml` carries no `ignore` rules. I wrote it from the description in that message rather than reading `/Users/maragubot/Developer/glue`, which is outside this worktree.

### Why

The brief's one non-obvious instruction — the label goes in an env var, never inline in `${{ }}` — is what keeps the `run:` blocks free of expression injection. Moving it to `env:` also made the port mechanically checkable: the shell either matches `gai`'s modulo `"$LABEL"` or it does not.

### What worked

Diffing the *parsed* YAML steps rather than the text. It turns "I copied it carefully" into a three-line diff a reviewer can check in ten seconds, and it caught nothing — which is the point.

Reusing `gai`'s stub-`gh` harness idea. I extracted both `run:` blocks with `yaml.safe_load`, put a stub `gh` on `PATH` that logs argv and is driven by env vars, and ran them under `bash -e` across seven cases. Exit codes: no open issue → 0 and files one; issue already open → 0 and files nothing; `gh issue list` errors → **1 and files nothing**; close step with two open issues → closes both. I also ran every case with the label `security alert`, logging argv as `[%s]` per argument, and confirmed it arrives as a single argv element in all three positions — so the `"$LABEL"` quoting is right and the input cannot word-split or glob.

### What didn't work

The CI workflow I had just written would have failed on its own first run, and the local `actionlint` said it was fine.

`shellcheck` is not installed on this machine, so `actionlint` disables that rule silently — `-verbose` says `Rule "shellcheck" was disabled: exec: "shellcheck": executable file not found in $PATH`. GitHub's `ubuntu-latest` *does* ship shellcheck, so CI would have enabled it. A reviewer flagged this; I downloaded shellcheck 0.10.0 to a scratch directory and reproduced it:

```
$ PATH=/scratch/bin:$PATH go run github.com/rhysd/actionlint/cmd/actionlint@v1.7.12
.github/workflows/security.yml:43:9: shellcheck reported issue in this script: SC2016:info:7:16: Expressions don't expand in single quotes, use double quotes for that [shellcheck]
exit status 1
```

Script line 7 is the `printf -v body '...'` line: the Markdown backticks around `` `Security` `` and `` `go run …` `` inside a single-quoted format string look to shellcheck like intended command substitution. It is a false positive — the backticks must stay literal, they are Markdown for the issue body — but `SC2016` is not in actionlint's default exclusion list, and actionlint reports shellcheck findings as errors regardless of severity, so the Lint job goes red. Fixed with a `# shellcheck disable=SC2016` directive and a comment saying why. `actionlint` with shellcheck on `PATH` is now exit 0.

This is the first time the shell in this pattern has ever been linted. `gai` has no actionlint step, which is precisely why the diary for `gai` PR #350 lists adding one as future work.

### What I learned

`actionlint`'s exit 0 means two different things depending on the machine it ran on, and it only tells you which if you pass `-verbose`. On a developer laptop without shellcheck it checks YAML and expression syntax; on a GitHub runner it also checks every `run:` block. Any workflow whose shell has never met shellcheck should be assumed to have a finding waiting in it — adding an actionlint job and porting shell in the same change is how you find out the hard way.

Also: `actions/setup-go`'s cache is enabled by default and `findDependencyFile` throws when there is no `go.mod` in the workspace root, but `main.ts` catches that and downgrades it to `core.warning`. So a repository with no Go module gets a warning per run, not a failure. `cache: false` is the honest setting.

### What was tricky

Deciding what to fix and what to report. Two findings that both reviewers raised independently are things the brief specified explicitly, so I implemented the brief and wrote them up instead of pre-empting the lead — the same split `gai`'s builder settled on: deviate when the repository itself answers the question, report when only the person who wrote the brief can. The SC2016 fix went the other way; it is not a judgement call, it is a red build.

The lead's mid-run correction pointed at a file in a sibling repository, outside this worktree. The message described its contents completely, so I wrote `dependabot.yml` from the description and did not read across.

### Self-review

I read the diff myself, then ran the `code-review` skill, which put two competing reviewers over the change. Consensus findings, and what I did with each:

**Fixed.** The SC2016 build break (above, single reviewer, verified by me and blocking). `cache: false` on `setup-go`. A `name: Govulncheck` on the previously anonymous run step, so a caller's Actions UI shows a step name rather than the full `go run` command line. Five README corrections: the migration is described in the future tense rather than as done; the permissions bullet now says a caller that under-grants fails the *run*, not just the issue steps; a new bullet says the `concurrency` block is load-bearing, because the dedup is a read-then-create that a `push` racing the cron can double; a new bullet states the `main`-only gate as a known limitation; and the intro records that this repository has to stay public, since a private one is callable only from private repositories. I also noted that migrating renames the status check to `govulncheck / govulncheck`, which silently stops enforcing any branch protection rule that requires the old name.

**Confirmed correct, no change.** Both reviewers independently hand-checked the shell and agreed with my harness: `"$LABEL"` is safely quoted in all three positions, the dedup still fails closed, no `${{ }}` reaches any `run:` block, `RUN_URL` and `github.token` resolve to the caller, no `secrets: inherit` is needed, `persist-credentials: false` still does real work on same-repo PRs, and the close step's fail-open pipeline fails in the benign direction.

**Reported, not changed** — see What warrants review. The two serious ones are the workflow-level `permissions:` block and the hardcoded `main` gate; both were specified in the brief, and both change meaning now that the workflow is shared rather than in-repo.

### What warrants review

Three things, in order of risk.

**The `permissions:` block may hard-fail fork pull requests.** A called workflow requesting a permission the caller does not hold is a workflow-validation error — `The workflow is requesting 'issues: write', but is only allowed 'issues: none'` — that fails the run before any step executes, not a silent downgrade. On a fork PR the caller's token is capped read-only no matter what its own `permissions:` block declares. Whether GitHub compares the callee's request against the caller's *declared* or *effective* permissions is not documented, and I could not settle it without a live fork PR. If it is the effective ones, every outside contributor's PR gets a red `Security` check and `govulncheck` never runs — a regression `gai` cannot have hit, because its non-reusable file only ever declared permissions for itself. Both reviewers reached this independently. The fix, if confirmed, is to delete the `permissions:` block: a called workflow without one inherits whatever the calling job passes, the README already tells callers to grant both, and the issue steps skip on PRs anyway. Worth one throwaway fork PR before rolling this out org-wide.

**`github.ref_name == 'main'` is now a literal in a workflow meant for arbitrary repositories.** In any caller whose default branch is `master` or `trunk`, the nightly run goes red and no issue is ever filed — the exact silent-failure hole this pattern exists to close, reintroduced by centralising it. `gai`'s diary already listed this as an open question; making the workflow shared turns it from theoretical into likely. `github.ref_name == github.event.repository.default_branch` fixes it in one line and keeps every property the current gate has (`pull_request` is still `N/merge`, branch dispatches are still excluded). The brief said to keep `main`, so I did.

**The `label` input probably should not exist.** Its own description says nothing else may use the label, and the safest value is the bespoke default. The title, body, colour and label description are all still hardcoded, so overriding `label` changes only the collision surface, not the report. Meanwhile `label: ""` passes through unvalidated to `gh issue list --label ''`, whose filter semantics I have not verified — if an empty filter means "no filter", the close step closes every open issue in the repository. Either drop the input, or guard it with `[ -n "$LABEL" ] || exit 1`.

Smaller, all recorded here rather than fixed: `govulncheck@latest` is now an unpinned dependency imposed on every caller, and by `gai`'s own exit-code analysis a proxy hiccup files a false "govulncheck found vulnerabilities" issue in all of them at once; a label beginning with `-` would be parsed by `gh` as a flag (`gh label create -- "$LABEL"` costs nothing); and `--limit 30` on the close step is inherited from `gai`.

### Future work

Add a self-test to `ci.yml`. Nothing in this repository ever *executes* `security.yml` — actionlint checks syntax only — yet callers pin `@main`, so any merge here fans out to every org repository with zero runtime verification. A job that does `uses: ./.github/workflows/security.yml` against a tiny fixture module would catch a runtime break before the org sees it.

The `actionlint@v1.7.12` pin in `/.github/workflows/ci.yml` is invisible to Dependabot: a `go run pkg@version` string is not the `github-actions` ecosystem, and there is no `go.mod` for the `gomod` one. It will rot until someone notices. Accept it, add a tools module, or switch to an action reference Dependabot tracks.

Branch protection on this repository's `main`, with required review, is now a prerequisite rather than hygiene: a push here executes in every caller's checkout with an `issues: write` token.

### Resolution: the lead's rulings on the four open questions

The three "reported, not changed" items above went back to the lead, who ruled on all four. Two are now applied in this branch; the sections above are left as written, since the reasoning that produced the questions is the useful part.

**Applied — the `permissions:` block is gone.** The reusable workflow declares no `permissions` at all and inherits whatever the calling job passes. This removes the fork-pull-request hazard entirely rather than betting on undocumented behaviour: with nothing requested, there is nothing that can exceed the caller's effective grant, so the validation error cannot fire. The README now states the reason in place, so the next person does not "helpfully" add the block back. A comment in `/.github/workflows/security.yml` says the same thing at the point of absence, which is the only place someone about to re-add it will be looking.

**Applied — the `label` input is gone.** `LABEL` survives as an env var set to the literal `govulncheck`, which keeps the shell readable and keeps the three call sites in step, but it is no longer caller-controllable. This also disposes of the `label: ""` footgun without needing the guard I proposed: there is no longer an untrusted path to `gh issue list --label ''`. The README's inputs table is down to one row, with the label's collision rule kept as a sentence beneath it, since callers still need to know not to use that label by hand.

**Kept — the literal `main` gate.** The lead's reasoning is better than mine: every `maragudk` repository uses `main`, and `github.event.repository.default_branch` is not reliably present on `schedule` events, which is exactly the trigger the whole pattern exists to serve. Swapping in the "more correct" expression would have traded a limitation that does not apply for a failure mode on the one event that matters. The README limitation note stays, because it is still true and cheap.

**Kept — `go-version` defaults to `stable`.**

I also diffed `/.github/dependabot.yml` against `glue`'s on the lead's instruction: identical, so writing it from the description in the correction message rather than reading across worktrees cost nothing.

Re-validated after the edits. `actionlint` v1.7.12 with shellcheck 0.10.0 on `PATH` is exit 0 on both workflows and on the README's caller snippet extracted to a scratch repository. The parsed workflow now has three top-level keys (`name`, `on`, `jobs`) and one input. The stub-`gh` harness returns the same exit codes as before the edits — `0, 0, 1, 0, 0, 0, 0` across the seven cases, with the third still confirming the dedup fails closed when `gh issue list` errors.

One consequence worth noting for the `gai` conversion: with the `permissions:` block removed here, the caller's block is now the *only* place permissions are declared. A caller that omits `issues: write` no longer fails loudly at validation time — it fails inside `gh` at the moment an issue needs filing, which is a red nightly rather than a red pull request, and only on the day something is actually wrong. The README bullet is the mitigation. A self-test job that exercises the workflow against a fixture module, already listed under Future work, would catch it earlier.

## Step 3: Address Markus's review of PR #2

**Author:** main

### Prompt Context

**Verbatim prompt:** `/fabrik:address-code-review`, then per comment: "Apply, and make sure all references are deleted"; "Ah. Do we even need that then? It's not like running govulncheck does anything in this repo."; "I don't think I want the action lint from an unknown third-party"; "Yes, delete entirely"; "ok"; "ok"; "a, and make the PR"; "Keep the README minimal"
**Interpretation:** Walk the five inline review comments one at a time, then apply the agreed changes in one batch: drop the `go-version` input, delete the repo's own CI, move the caller example to a starter workflow in `maragudk/.github`, and trim the README.
**Inferred intent:** The central repo should carry the minimum: one reusable workflow with no knobs, no third-party tooling in the repo every caller trusts, and documentation that points at the canonical template instead of duplicating it.

### What I did

Fetched the five unresolved review threads via GraphQL, presented each with an assessment, replied and resolved each after Markus's decision. Outcomes: (1) `go-version` input dropped, `stable` hardcoded, inputs table removed — govulncheck should scan with the newest toolchain and no repo needs otherwise; (2)–(4) `/.github/workflows/ci.yml` deleted entirely — the `concurrency` question was moot for a plain workflow, and Markus does not want `rhysd/actionlint`, a third-party tool, in this repository; callers are the integration test; (5) the inline caller file in `/README.md` replaced by a link to a starter workflow at `maragudk/.github/workflow-templates/security.yml`, created in a separate PR with a `security.properties.json` (`$default-branch` placeholders in the triggers, `filePatterns: ["go.mod$"]`, Go category), so the template shows up in every repository's New workflow picker and there is one copy of the caller. Rewrote `/README.md` to one paragraph per concern. A prior codex second opinion (`gpt-5.6-sol`, xhigh, `codex exec review --uncommitted`) had found no functional defects and one P3 — a callee comment stating a caller requirement as fact — which was fixed before the PR was opened.

### Why

Each removal takes a maintenance surface out of the one repository whose `main` runs with `issues: write` in every caller. The starter workflow is the GitHub-native home for a copy-once caller file; the README linking to it means the caller has exactly one canonical form.

### What worked

Triage before touching code kept the five comments from turning into five commits; three of them collapsed into a single deletion.

### What didn't work

Nothing failed. `actionlint` is no longer run in CI; it is still useful locally before pushing.

### What I learned

For a reusable-workflow repo, the repo's own CI has little to check without a third-party linter; the callers' runs are the real validation.

### What was tricky

Comment 5 pulled in a second repository (`maragudk/.github`, which had no `workflow-templates/` directory) and the question of where the single copy of the caller lives; the answer is the template, with the README carrying only the contract.

### What warrants review

`/.github/workflows/security.yml` has no `inputs:` and no `permissions:`; `/.github/workflows/ci.yml` is gone; `/README.md` links to the template. In `maragudk/.github`: `workflow-templates/security.yml` uses `$default-branch` in the triggers and calls `maragudk/workflows/.github/workflows/security.yml@main`; `security.properties.json` is valid JSON with the documented keys.

### Future work

Convert `gai`, `glue` and `app` to callers using the template; `gomponents` as a new adopter. Job-sized reusable CI workflows (`test`, `lint`) as recorded in Step 1's future work.

## Step 4: Second review round on PR #2

**Author:** main

### Prompt Context

**Verbatim prompt:** `/fabrik:address-code-review`, then "apply", "yes", "yes"
**Interpretation:** Three more inline comments on PR #2, all pushing the same direction as the first round: fewer words, one canonical location for each fact.
**Inferred intent:** The README should be a signpost, not a manual; the workflow file should be self-explanatory to whoever opens it while debugging.

### What I did

Triaged three comments one at a time, replied and resolved each. Outcomes: the required caller permissions are now stated in the workflow file's own comment instead of via a README pointer; the README intro is the single line "Reusable GitHub Actions workflows."; the Security section is one descriptive sentence plus a single link to the starter workflow in `maragudk/.github`, with the caller-rules bullet list removed so nothing is duplicated between the two repositories. Corrected one path in the discussion: starter workflows live at `workflow-templates/` in the root of the `.github` repository, not `.github/.github/workflows/`.

### Why

Every duplicated fact between the README, the workflow comment, and the template is a future drift; after this round each fact has exactly one home.

### What worked

Comment 1 (document permissions in the workflow) made comment 3 (drop the README rules) free, since the rationale had somewhere to go.

### What didn't work

Nothing failed.

### What I learned

Nothing new.

### What was tricky

Nothing.

### What warrants review

`/README.md` is four lines of content; `/.github/workflows/security.yml`'s header comment names the two required grants.

### Future work

Unchanged from Step 3.
