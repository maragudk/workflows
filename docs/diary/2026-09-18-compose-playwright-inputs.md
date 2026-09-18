# Diary: `compose` and `playwright` inputs on the test workflow

`test.yml` takes `postgres`, `s3` and `secret-env`. A repository whose integration tests need a
local atproto network — PLC, Postgres, PDS and Caddy, with `depends_on`, healthchecks and a
`build:` from a git URL — cannot get that from a `services:` block, and its browser tests need
a Chromium that matches the playwright-go version in its `go.mod`. This task adds two booleans
that let the caller's own compose file and its own `go.mod` supply both.

## Step 1: Add the inputs, prove the pieces locally, and open the PR

**Author:** workflows-builder

### Prompt Context

**Verbatim prompt:**

> You are "workflows-builder". Work in the repository /Users/maragubot/Developer/workflows (branch off `main`, currently at `d2327f6`; create a branch like `compose-playwright-inputs` and work there; do NOT touch /Users/maragubot/Developer/audioadastra). It holds reusable GitHub Actions workflows; read `.github/workflows/test.yml`, `README.md`, `docs/decisions.md`, and the entries in `docs/diary/` first, and follow the conventions you find there (the workflow's inputs have careful multi-line descriptions and comments explaining non-obvious YAML). Start a diary for this task with the `fabrik:diary` skill (`docs/diary/<date>-compose-playwright-inputs.md`).
>
> ## Goal
>
> Extend the reusable test workflow so a calling repository can (a) bring up a docker compose stack before the tests and (b) install Playwright browsers for Go browser tests. Motivation: a project needs a local atproto network (PLC + Postgres + PDS + Caddy, with `depends_on`, healthchecks and a `build:` from a git URL) for its integration tests -- `services:` containers can't express that, docker compose can, and the same compose file then serves dev, `make test` and CI.
>
> ## Requirements
>
> 1. New boolean input `compose` (default false) on `test.yml`'s `workflow_call`: when true, add a step after checkout/setup-go and before the tests that runs `docker compose up -d --wait` in the caller's checkout (the called workflow's steps run in the caller's repository checkout; document that the caller's `docker-compose.yml` is what gets used, and that `--wait` fails the job if any service never becomes healthy). Description should state the semantics like the existing `postgres`/`s3` inputs do. Add a matching teardown as a best-effort final step (`if: always()`, `docker compose down --volumes`) -- cheap, and it keeps logs tidy; or argue in the diary why not.
>    - Consider whether a `compose-file` string input (path, default empty meaning compose's own default resolution) is worth adding now; I lean no -- YAGNI -- but note it as future work. Don't add it unless there's a concrete reason.
>    - Consider printing `docker compose ps` after up (and `docker compose logs` on failure via `if: failure()`) so a failing job is debuggable. Do it if it's a couple of lines.
> 2. New boolean input `playwright` (default false): when true, install Chromium for `github.com/playwright-community/playwright-go` before the tests. Find the correct, current installation command for playwright-go in CI (it's typically `go run github.com/playwright-community/playwright-go/cmd/playwright@<version> install --with-deps chromium`; check the playwright-go README on GitHub via `gh api` or WebFetch for the recommended way, and think about version alignment: the driver version must match the playwright-go module version the caller uses. The cleanest is running the command from the caller's module -- `go run github.com/playwright-community/playwright-go/cmd/playwright install --with-deps chromium` without `@version` resolves to the version in the caller's go.mod, which is what we want; verify this works when the module is a dependency of the caller). Cache the browser download if a standard cache action step is simple (`~/.cache/ms-playwright`); otherwise skip caching and note it.
> 3. Keep every existing behaviour and input intact. Preserve the `# Callers must ...` comment style and the input description style.
> 4. Update `README.md` if it documents inputs (check); add a `docs/decisions.md` entry only if the repo's decisions convention says this rises to that level (read it -- "compose over services for multi-container test dependencies" might be one; be conservative).
> 5. Validate YAML syntax (e.g. `python3 -c 'import yaml,sys; yaml.safe_load(open(".github/workflows/test.yml"))'` or `actionlint` if installed -- check `which actionlint`). You cannot run the workflow itself; say so in the report.
> 6. Commit with the `fabrik:git` skill conventions (concise subject, backticks around identifiers, no attribution lines; no GitHub issue references unless you find a relevant open issue in that repo -- check `gh issue list --repo maragudk/workflows`), push the branch, and open a PR with `gh pr create --repo maragudk/workflows --reviewer markuswustenberg` -- bullet-point body, no summary header, no test-plan section. Report the PR URL, the exact input names and semantics, and anything uncertain (especially the playwright install command/version story).

**Interpretation:** Two opt-in booleans in the established shape, with the playwright install
command chosen from evidence about how playwright-go resolves its driver today, and every claim
in the input descriptions checked locally since the workflow itself cannot run here.

**Inferred intent:** The first caller should be able to point its CI at `test.yml` with
`compose: true` and `playwright: true` and delete its inline test job, keeping one compose file
for development, `make test` and CI — and no existing caller should notice anything.

### What I did

Branched `compose-playwright-inputs` off `main` at `d2327f6`. In
`/.github/workflows/test.yml` I added two boolean inputs, `compose` and `playwright`, both
defaulting to `false`, keeping the inputs alphabetical (they land before `postgres`), and three
steps:

- `Install Chromium for Playwright`, after `Build`, guarded by the same
  `inputs.x == true || inputs.x == 'true'` double comparison the service images use, running
  `go run github.com/mxschmitt/playwright-go/cmd/playwright install --with-deps chromium`.
- `Start the compose stack`, next, with `id: compose`, running
  `docker compose up --wait --wait-timeout 300` then `docker compose ps`.
- `Show compose state and logs`, after `Test`, on
  `failure() && steps.compose.outcome != 'skipped'`, running `docker compose ps --all` and
  `docker compose logs`.

That is the shape after self-review; the first cut differed in ways recorded under "What
didn't work" and "What was tricky".

There is no teardown step; see "Why". The `Export secrets to the environment` step stays
immediately before `Test`, so its description ("the steps that follow, which is the test run")
stays exact. The `# Callers must ...` comment is unchanged because neither input needs a
permission or a secret. `README.md` does not document inputs, so it is untouched; the input
descriptions carry the contract, as they have since the `s3` input.

Docs: a new entry in `/docs/decisions.md` and this file.

Evidence gathered before writing anything, all with `gh api` against
`playwright-community/playwright-go`, which now redirects to `mxschmitt/playwright-go`:

- The README's install instruction is `go run github.com/mxschmitt/playwright-go/cmd/playwright@v0.xxxx.x install --with-deps`, with the note "replace the version number with the version used in your current go.mod. Each minor version upgrade requires a specific Playwright driver version."
- `go.mod` at tags: `v0.5200.0` and `v0.6000.0` say `module github.com/playwright-community/playwright-go`; `v0.6100.0`, `v0.6201.0` and `v0.6201.1` (latest, 2026-08-17) say `module github.com/mxschmitt/playwright-go`. So the path moved at v0.6100.0.
- `cmd/playwright/main.go` imports only `log`, `os` and the module's root package, so running it from a caller needs no `go.sum` entries the caller does not already have.
- playwright-go's own `build.yml` runs `playwright install --with-deps ${{ matrix.browser }}` on `ubuntu-latest`, so `--with-deps` (which runs `apt-get` under `sudo`) works on hosted runners.
- Playwright's CI docs: "Caching browser binaries is not recommended, since the amount of time it takes to restore the cache is comparable to the time it takes to download the binaries", and the Linux OS dependencies are not cacheable.
- Docker's reference for `compose up --wait`: "Wait for services to be running|healthy. Implies detached mode." (so `-d` is redundant and left out).

Validation, since a reusable workflow cannot be run from here:

- `python3 -c 'import yaml; ...'` parses the file and lists the inputs as
  `['compose', 'playwright', 'postgres', 's3', 'secret-env']`.
- `actionlint` is not installed, so I ran `go run github.com/rhysd/actionlint/cmd/actionlint@latest .github/workflows/test.yml` (v1.7.12): no findings.
- Two scratch modules in the session scratchpad, one requiring `github.com/playwright-community/playwright-go v0.6000.0` and one `github.com/mxschmitt/playwright-go v0.6201.1`. In the new-path one, `go run github.com/mxschmitt/playwright-go/cmd/playwright --version` compiled and ran the CLI from `go.mod` with `go.sum` unchanged and printed `Version 1.62.1`. In the old-path one, and in a third module with no playwright-go at all, the same command fails with `no required module provides package github.com/mxschmitt/playwright-go/cmd/playwright; to add it: go get ...`, which is the error a misconfigured caller will see.
- A scratch `compose.yaml` with a healthy service and one whose healthcheck is `false`: `docker compose up --wait` printed `container compose-bad-sick-1 is unhealthy` and exited 1; with `retries: 100` and `--wait-timeout 5` it printed `application not healthy after 5s` instead. In both cases `docker compose ps --all` and `docker compose logs` still worked afterwards, showing the sick container `Up`.

### Why

`go run` without `@version` is the whole version story: in module mode it resolves the package
through the main module's requirements, so the driver the CLI downloads is the one the caller's
playwright-go release pins. Any version written in this repository would drift from every
caller's `go.mod`, which is the one thing the driver must match. Only the post-rename module
path is supported because the pre-rename releases cannot download a driver from anywhere any
more; see "What didn't work".

`--wait` rather than `-d` because it implies detached mode and adds the readiness gate that a
`services:` block gets from `options: --health-cmd`; `--wait-timeout 300` because a
`depends_on` chain on a service that never finishes otherwise holds `--wait` until the
six-hour job limit; `ps` after it costs one line and shows the state of every service in the
log; `ps --all` and `logs` on failure are the debugging aid that would otherwise be lost with
the VM, gated on the compose step's outcome so they run exactly when the stack was started.

Compose starts before `secret-env` is exported, so the compose file sees no secrets. That keeps
the export step immediately before `Test`, where its own description says it is, and the
`compose` description says so; a caller that needs a secret in its compose file is a future
change to both descriptions, not a silent limitation.

No teardown, against the lead's lean, and the argument is short: the job runs on
`ubuntu-latest`, a fresh VM discarded with everything in it, so `docker compose down --volumes`
would run on every `compose: true` job and remove nothing anyone could observe. It does not tidy
the logs either — the runner does not print anything about compose containers at job end. On a
self-hosted runner it becomes necessary, and it is one `if: always()` step then.

Chromium only because that is what the callers' browser tests use, and every extra browser is
another download on every run; no cache because Playwright's own guidance says it does not pay.

### What worked

Every piece was checkable locally without a runner: the module resolution with real scratch
modules, the compose exit code with a deliberately sick service, the YAML with actionlint via
`go run`. The old-path scratch module was also a useful negative: its `go run ... --version`
got as far as the CLI and then failed downloading its driver, `could not download driver: error:
got non 200 status code: 404 (404 Not Found) from
https://playwright.azureedge.net/builds/driver/playwright-1.60.0-mac-arm64.zip`, which proves
the resolution and shows that very old playwright-go releases have a driver CDN problem of their
own, unrelated to this workflow.

### What didn't work

The first cut supported both module paths with a `go list -m` probe loop, on the reasoning that
callers on either side of the v0.6100.0 rename were plausible. Self-review (below) showed the
old-path arm could never succeed: every `github.com/playwright-community/playwright-go` release
lists only the three `azureedge.net` mirrors as driver sources, and
`curl -I https://playwright.azureedge.net/builds/driver/playwright-1.60.0-linux.zip` returns
404 — as do `cdn.playwright.dev` and the `playwright.download.prss.microsoft.com` path — while
v0.6100.0's `run.go` fetches `playwright-core` from `https://registry.npmjs.org` and Node from
`https://nodejs.org/dist`. The 404 I had already seen locally on the old-path scratch module and
filed as "that old version's problem" was in fact the whole story: a caller on the old path
would get a red step every run, with a worse error than the plain `go run` one. The loop went,
the step is one line, and the description says why the new path is required.

The lead's suggested module path, `github.com/playwright-community/playwright-go`, was also a
dead end on its own: a step written against it would fail for every caller on v0.6100.0 or
newer with `no required module provides package`.

### What I learned

- `go run <pkg>` inside a module resolves `<pkg>` from the build list, so a CLI shipped inside a dependency runs at exactly the dependency's version with no extra `go.sum` traffic, as long as its imports are already covered — which is worth checking, since a CLI with its own dependencies would need `go mod tidy` to have seen it.
- "Compiles and runs" is not "works": a probe that gets as far as the tool's own first network call has proven resolution and nothing else. When the tool then fails, look at whether that failure is universal before filing it as someone else's problem.
- `docker compose up --wait` with no `--wait-timeout` is unbounded for a dependency that never resolves; healthcheck `retries` bound the unhealthy case but not the `service_completed_successfully` one.
- `docker compose up --wait` fails on an unhealthy service but leaves the stack up, so a follow-up `ps`/`logs` still has something to show.
- The `# Callers must ...` line is about permissions and secrets only; an input that needs neither does not touch it.

### What was tricky

Deciding how much of the rename to encode, and getting it wrong first. The two-name probe looked
like the careful choice — it handled both paths that exist — and it took a reviewer checking the
old releases' download URLs to show that "handles" meant "fails later with a 404". The lesson is
in "What I learned".

Self-review was the code-review skill's two competing reviewers over the diff. Consensus
findings, all taken: the undocumented fact that compose runs before `secret-env`; `ps` never
running when `up --wait` fails under `bash -e`, so the failure step now prints `ps --all`; the
description narrowing compose's file resolution to two names; the caching rationale and "the
same one used for development" being decisions-document material rather than contract; the
missing `--wait-timeout` (one reviewer reproduced a `depends_on: service_completed_successfully`
chain holding `--wait` past two minutes); and gating the failure step on
`steps.compose.outcome != 'skipped'` instead of repeating the input comparison. Single-reviewer
findings taken: the dead old-path arm (serious, verified above), and the description claiming a
service container cannot have a healthcheck when the file's own `postgres` service has one — the
thing it cannot have is `depends_on` gated on one. Not taken: `-mod=mod` on the playwright step
against a future CLI dependency (recorded as the known failure mode in the decisions entry
instead), a `vendor/` caller (none exist), and overlapping the Chromium download with the compose
build by splitting `up -d` from `up --wait` (future work).

Step order also took a moment: playwright install before compose up, and both after `Build`, so
a build failure never pays for either.

### What warrants review

- `/.github/workflows/test.yml`: the two descriptions, since they are the documentation; the
  `if:` on `Show compose state and logs` (`failure() && steps.compose.outcome != 'skipped'`),
  the only step gated on another step's outcome; whether five minutes is the right
  `--wait-timeout` for a stack that builds an image from a git URL (the timeout covers the
  wait, not the build, as far as compose's reference says); and whether the missing teardown
  argument holds for you.
- `/docs/decisions.md`: whether the "compose is the escape hatch" framing is the stance you want
  recorded, since it settles the question the 2026-09-03 entry deferred.
- Nothing here has run on a GitHub runner. The first real check is the first caller pointing
  `uses:` at this branch with `compose: true` and `playwright: true`.

### Future work

- Mirror both inputs into `compatibility.yml` when a caller wants them there; they carry over unchanged.
- A `compose-file` string input if a caller ever needs a file other than compose's default resolution.
- A teardown step if the workflow ever runs on a self-hosted runner.
- Overlap the Chromium download with the compose build (`docker compose up -d` before the playwright step, `docker compose up --wait` after) if the serial wall time turns out to matter.
- `-mod=mod` on the playwright step, if a playwright-go release ever gives `cmd/playwright` a dependency of its own.

## Step 2: Drop the `playwright` input and trim the `compose` prose after review

**Author:** workflows-builder

### Prompt Context

**Verbatim prompt:**

> Review outcome on workflows PR #9, both threads triaged with Markus; apply as one commit on the same branch and push:
>
> 1. Remove the `playwright` input and its install step entirely (the consuming project will use chromedp with the Chrome preinstalled on ubuntu-latest, so no browser install is needed). Remove the corresponding text from the decisions entry and add a line in the diary saying why it was dropped (community binding + runtime Node driver download vs. chromedp with system Chrome). Make sure the decisions entry only argues what's actually shipped.
> 2. Make the `compose` input description and the step comments concise: a few sentences, no diary-level detail. Suggested description: "Run `docker compose up --wait` on the caller's compose file before the tests, for dependencies a service container can't express. Fails if a service isn't healthy within five minutes. Starts before secret-env is exported." Keep the existing inputs' descriptions as they are. Trim the step comments to one line each; keep the `"false"`-string comparison comment since it's load-bearing.
> 3. Update the PR title/body to match (compose input only). actionlint + YAML parse again. Report the new commit hash.

**Interpretation:** Ship only the `compose` input, in the terse register the other inputs use, and leave the playwright investigation as history in this diary rather than as a contract in the workflow.

**Inferred intent:** The browser question has been settled on the consuming side — chromedp against the Chrome that `ubuntu-latest` already ships — so a step that downloads a Node driver at run time for a community binding is machinery nobody will call.

### What I did

In `/.github/workflows/test.yml`: removed the `playwright` input and the `Install Chromium for Playwright` step; replaced the `compose` description with the suggested three sentences, quoting the command with double quotes as the `postgres` and `s3` descriptions do; cut the comment on `Show compose state and logs` to one line. The `"false"`-string comparison comment on the compose step stays. The steps and commands are otherwise as Step 1 left them.

In `/docs/decisions.md`: retitled the 2026-09-18 entry to the `compose` input alone, dropped the two playwright bullets and the playwright items from the rejected list, and added one rejected item recording that the input was carried and dropped, and why.

PR #9's title and body updated to describe only the `compose` input. `actionlint` (via `go run`, v1.7.12) and a YAML parse both clean.

### Why

The `playwright` input was dropped because the consuming project chose chromedp with the system Chrome on `ubuntu-latest` over playwright-go. The trade is a community-maintained binding that downloads a Node.js driver at run time — with the module-path rename and the dead CDN for older releases that Step 1 ran into — against a Go-native library that talks to a browser the runner already has, so there is nothing for the workflow to install. A shared workflow should not carry an input whose only consumer went another way.

The prose was trimmed because the other inputs' descriptions state the contract in three or four sentences and let the decisions entry carry the argument; Step 1's description had drifted into explaining itself.

### What worked

The playwright removal was a clean cut: the step had no `id`, nothing referenced it, and the `compose` steps were already independent of it.

### What didn't work

Nothing failed in this step.

### What I learned

The Step 1 investigation was still worth doing: the rename and the dead driver CDN are the concrete reasons a runtime-driver binding is a liability in a shared workflow, and they are what made the chromedp decision easy to argue.

### What was tricky

Only the quoting of the suggested description: the coordinator's text used backticks around the command, the file's existing descriptions use double quotes, and "keep the existing inputs' descriptions as they are" argues for matching them rather than introducing a second style.

### What warrants review

- `/.github/workflows/test.yml`: the new `compose` description, and that nothing playwright-related remains.
- `/docs/decisions.md`: the entry now argues only what ships; the playwright drop is one rejected item.

### Future work

- Unchanged from Step 1 for compose: `compatibility.yml` mirroring, a `compose-file` input, a teardown step on self-hosted runners.
