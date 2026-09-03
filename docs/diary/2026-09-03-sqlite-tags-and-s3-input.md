# Diary: SQLite build tags and an `s3` service input on the test workflows

`maragudk/app` is an app template whose stamped-out apps need two things the shared
`test.yml` and `compatibility.yml` could not give them: the SQLite build tags
`sqlite_fts5,sqlite_math_functions` on every `go build`/`go test`, and a versitygw S3
service container during tests. A reusable workflow cannot accept a `services:` block from
its caller, so either the app keeps its own inline test jobs forever or the shared
workflows carry the scaffolding. Markus chose the latter. This is the first crack in the
"zero inputs everywhere" stance from 2026-09-02, which always carried the caveat that an
input arrives when a second repository needs it.

## Step 1: Hardcode the tags, add the `s3` input, and validate through the app

**Author:** workflows-builder

### Prompt Context

**Verbatim prompt:**

> ## Context
>
> `maragudk/workflows` holds six reusable workflows with zero inputs, zero secrets, pinned `@main` by ~24 repos. `docs/decisions.md` there records "an input is added when a second repository needs it". That repository has arrived: `maragudk/app` is an app template whose stamped-out apps need SQLite build tags and an S3 service container in tests, and reusable workflows cannot receive `services:` from callers. Markus (product owner) decided the shared workflows carry this scaffolding.
>
> ## Requirements
>
> Change `.github/workflows/test.yml` and `.github/workflows/compatibility.yml`:
>
> 1. **Build tags, hardcoded, no input.** Every `go build` and `go test` invocation gets `-tags sqlite_fts5,sqlite_math_functions`. Unmatched tags are a no-op for repos without `mattn/go-sqlite3`, so this is safe org-wide. Do NOT add a `tags` input — Markus explicitly rejected that.
>
> 2. **`s3` boolean input, default `false`.** When true, the test job(s) get a versitygw service container reachable on host port 7072, matching exactly the service block in `/Users/maragubot/Developer/app/.github/workflows/ci.yml` (image, ports, env, any options) — read it and copy it faithfully, including any job-level env the tests need to reach it. When false, no container is started. Preferred mechanism: declare the service in the workflow with an image expression that is empty when the input is off, e.g. `image: ${{ inputs.s3 && 'versity/versitygw:latest' || '' }}` — GitHub skips services with an empty image. Verify this actually works with a real run (see Validation); if it does not, fall back to a `docker run -d` step guarded by `if: inputs.s3`, with a readiness wait.
>
> 3. **Backwards compatible.** Existing callers pass nothing and must behave exactly as today apart from the added tags. Keep the workflow's existing style: `persist-credentials: false`, `-race -shuffle on`, the `go mod tidy` drift check, the 2x2 matrix in compatibility, the "caller must grant" comment. Document the input inline in the workflow with a short description.
>
> 4. **Docs.** Update `docs/decisions.md` in the workflows repo via the decisions skill: the zero-inputs stance is revised — one input (`s3`) exists now, build tags are hardcoded, and why. Write a diary entry there via the diary skill. Do not touch the README beyond what is strictly needed (it is deliberately a signpost).
>
> ## Validation
>
> Push the branch. Since `workflow_call` workflows cannot run standalone, validate through the app: in `/Users/maragubot/Developer/app`, create a throwaway branch (e.g. `validate-shared-test-workflow`) whose `.github/workflows/ci.yml` replaces the inline `test` job with `uses: maragudk/workflows/.github/workflows/test.yml@<your-branch>` with `s3: true`, push it, and confirm with `gh run watch`/`gh run view --log` that (a) the job is green, (b) the versitygw service started, (c) the `-tags` flags appear in the logs. Then do one run with `s3` omitted and confirm no service container is created. Delete the throwaway app branch (local and remote) afterwards; do not leave changes in the app worktree.
>
> Also run the app's tests locally with the exact upstream command (`go test -race -shuffle on -tags sqlite_fts5,sqlite_math_functions ./...` from `/Users/maragubot/Developer/app`) and report whether `-race` surfaces anything — this repo has never run with `-race`. Do not fix races; just report.
>
> ## Deliverable
>
> Commit on the branch in the workflows repo and open a PR against `main` with `gh pr create` (title in the git skill's style; description summarising the change, the validation run URLs, and the note that it is backwards compatible). Do not merge. Self-review before opening the PR.

**Interpretation:** Add the tags unconditionally to both test workflows, add exactly one
input (`s3`, boolean, default false) gating a versitygw service copied verbatim from the
app's inline job, prove the empty-image gate works on real runs in both states, and record
the reversal of the zero-inputs stance in the decisions log.

**Inferred intent:** Let `maragudk/app` — and every app stamped from it — drop its inline
test jobs and call the shared workflows, without making the other 23 callers pay anything
or change a line.

### What I did

Branched `test-sqlite-tags-and-s3` off `main` and rewrote `/.github/workflows/test.yml`
and `/.github/workflows/compatibility.yml`. Both gained a `workflow_call` input block with
a single `s3` boolean (default `false`, one-line description), an always-declared
`versitygw` service whose image is `${{ inputs.s3 && 'versity/versitygw:latest' || '' }}`,
and `-tags sqlite_fts5,sqlite_math_functions` on every `go build` and `go test`. The
service block — image, `7072:7070` port mapping, and the four `ROOT_ACCESS_KEY` /
`ROOT_SECRET_KEY` / `VGW_BACKEND` / `VGW_BACKEND_ARG` env vars — is a byte-for-byte copy of
the app's inline job. Everything else is untouched: `persist-credentials: false`, the
`go mod tidy` drift check, `-race -shuffle on`, the 2x2 compatibility matrix, the "Callers
must grant contents: read." comment.

Validation ran through `maragudk/app` on a throwaway `validate-shared-test-workflow`
branch, whose `ci.yml` was cut down to just the calls under test and given a `push:`
trigger for that branch name so no pull request was needed. Two runs:

- [Run 33749715291](https://github.com/maragudk/app/actions/runs/33749715291) — `test.yml`
  with `s3: true`. Green in 2m31s. The log shows `Starting versitygw service container`,
  the `docker create ... -p 7072:7070 -e "ROOT_ACCESS_KEY=access" ...` line with all four
  env vars, `versitygw service is healthy.`, and both
  `go build -tags sqlite_fts5,sqlite_math_functions ./...` and
  `go test -race -shuffle on -tags sqlite_fts5,sqlite_math_functions ./...`.
- [Run 33749995262](https://github.com/maragudk/app/actions/runs/33749995262) — `test.yml`
  with `s3` omitted, plus `compatibility.yml` with `s3: true` to exercise the same gate
  under the matrix. All five jobs green. The `s3`-omitted job logs exactly one mention of
  versitygw: `The service 'versitygw' will not be started because the container definition
  has an empty image.` All four compatibility legs started a healthy container and ran the
  tagged test command.

Deleted the throwaway branch local and remote afterwards and returned the app worktree to
`main`; the only thing left there is the lead's own untracked diary file.

Also ran the app's suite locally with the exact upstream command from
`/Users/maragubot/Developer/app`:

```
go test -race -shuffle on -tags sqlite_fts5,sqlite_math_functions ./...
```

Exit 0. Every package passed (`app/html`, `app/jobs`, `app/service`, `app/sqlite`,
`app/sqlitetest`), no race reports. The only noise is three `ld: warning: ignoring
duplicate libraries: '-lm'` lines from cgo linking on macOS, which are unrelated to `-race`.

### Why

Hardcoding the tags rather than taking a `tags` input keeps the caller surface at zero for
this concern: the value is either these two tags or irrelevant, so asking 24 repositories
to restate it buys nothing. Gating the service behind an input rather than always starting
it keeps the cost — an image pull and a container start per run — off the repositories that
have no S3. Declaring the service with an empty image, rather than a `docker run -d` step,
leaves the runner in charge of the container network, the port mapping and the readiness
wait, so the workflow does not grow a hand-rolled polling loop that has to be right in two
files.

### What worked

The empty-image gate works exactly as documented, and GitHub says so out loud in the log
rather than silently doing nothing — that one log line is the whole proof, and it means no
fallback was needed. Copying the app's service block verbatim rather than "cleaning it up"
meant the first `s3: true` run was green with no debugging. Trimming the throwaway `ci.yml`
to only the jobs under test and adding the branch to the `push:` trigger gave a fast,
focused signal without opening and closing a pull request in the app.

### What didn't work

Nothing failed. Both validation runs were green on the first attempt, and the local
`-race` run passed clean.

### What I learned

A GitHub Actions service container really can be conditional without a separate step: the
expression `${{ inputs.s3 && 'versity/versitygw:latest' || '' }}` evaluates to `''` when
the boolean is false, and the runner skips the service and logs why. The `&&`/`||` idiom
depends on the input being a real `type: boolean` — a string input would make `'false'`
truthy and start the container for everyone.

The app has never run under `-race` and has no races to find. That is worth knowing before
23 other repositories convert, because the 2026-09-02 decision to put `-race` on every
shared test run predicted "converting repositories may surface latent races as new
failures"; this one did not.

### What was tricky

The requirement to copy "any job-level env the tests need to reach it" turned out to be a
no-op: the app's inline test job sets no job-level env, because no `*_test.go` in the app
touches S3 yet — the container is template scaffolding for downstream apps. The endpoint
the app's `.env.example` points at (`AWS_ENDPOINT_URL=http://localhost:7070`) is the
local-development port, not CI's 7072, which confirms nothing in CI reads it today. So the
faithful copy is the service block alone. If a downstream app writes S3 tests, it will need
`AWS_*` env, and that env has to come from somewhere — the caller cannot supply it either.

The other sharp edge is that a `workflow_call` workflow cannot be run standalone, so every
change here is only testable through a caller in another repository.

### What warrants review

The two workflow files are the whole change: `/.github/workflows/test.yml` and
`/.github/workflows/compatibility.yml`. Worth checking that the tags land on every Go
invocation (two in `test.yml`, one in `compatibility.yml`) and that nothing else in the
files moved. Backwards compatibility for the existing callers rests on `default: false`
plus the empty-image skip, both shown in run 33749995262.

The decisions entry deliberately revises two bullets from 2026-09-02 ("zero inputs
everywhere" and "services stay inline") as a new dated entry rather than editing the old
one.

### Future work

When a downstream app actually writes S3 tests, the workflows will need to supply the
`AWS_*` env that points at `localhost:7072` — either as job-level env alongside the service,
or the app grows a test helper that defaults to that endpoint. Left alone for now, since
nothing needs it and guessing the variable names would bake one app's shape into all 24
callers. Separately, `maragudk/.github`'s starter templates will want a line showing the
`s3: true` call once this lands.

## Step 2: Self-review, and what two competing reviewers found

**Author:** workflows-builder

### Prompt Context

**Verbatim prompt:** (no new prompt — the deliverable in Step 1 says "Self-review before opening the PR.")
**Interpretation:** Read the diff as a reviewer would, then run the `code-review` skill's two-competing-reviewers method over it, apply the findings that hold up, and record the ones that do not.
**Inferred intent:** Catch the mistakes that a change touching 24 repositories' CI cannot afford to ship with, before Markus sees the pull request.

### What I did

Read the diff top to bottom, then dispatched two competing reviewers over the branch, one
pointed at correctness and comment quality, the other at operational and second-order
consequences. Three findings had consensus and changed the code; several more corrected the
decisions entry.

The code changes:

- The gate became
  `image: ${{ (inputs.s3 == true || inputs.s3 == 'true') && 'versity/versitygw:latest' || '' }}`.
  The plain truthiness test fails *open*: a caller wiring `s3:` from its own
  `workflow_dispatch` input — those are always strings — passes `"false"`, which is truthy,
  and silently gets a container. Step 1's diary entry named this sharp edge and the code
  did not defend against it, which is exactly the kind of gap a reviewer should find.
- The input description now names the endpoint (`http://localhost:7072`), the credentials
  and path-style addressing, and says why a caller cannot get them from its own env. As
  written before, a caller could turn the input on and have no documented way to reach the
  container.
- The comment moved onto the `image:` line it actually explains, and the credentials got a
  line saying they are throwaway values for a container only the job can reach, so nobody
  reads them as a leak.

The decisions entry was rewritten for four factual problems:

- "costs them nothing" for the tags is wrong — `-tags` is part of the build cache key, so
  the first run after this lands recompiles, and go-sqlite3 users compile a larger
  amalgamation from then on.
- Another repository in the org already builds with a *different* tag set,
  `sqlite_fts5 sqlite_foreign_keys`, in its workflows, `Makefile` and `Dockerfile`. So the
  premise "the value is either these two tags or irrelevant" is false, and converting it
  as-is would test a build its image does not ship. The entry now names the exception
  instead of claiming there is none.
- "the runner handles the ... readiness wait" is false for this image. `versitygw:latest`
  declares no `HEALTHCHECK` and the service takes no `options:`, so the runner prints
  "versitygw service is healthy." unconditionally — 0.99s after `docker create` in run
  33749715291. Step 1 of this diary cites that line as proof the container was ready; it is
  not, and that claim is wrong. The container did start (the log shows the pull, the create
  and versitygw's own banner), but nothing waits for it to accept connections.
- The rejection line claimed "one service, one repository shape". Six local repositories
  carry a byte-identical versitygw block, and others carry postgres, minio or elasticmq, so
  the honest rationale is that the machinery is deferred, not that the second caller is
  hypothetical. The line about port 7072 also mis-stated the app's `docker-compose.yml`,
  which publishes 7070 for the dev container and 7072 only for the test one.

Then revalidated on
[run 33751872868](https://github.com/maragudk/app/actions/runs/33751872868), which calls
`test.yml` three times from one throwaway branch: `s3: true`, `s3` omitted, and `s3: false`
written out explicitly. All green; the container starts in the first and the other two log
"The service 'versitygw' will not be started because the container definition has an empty
image." Throwaway branch deleted local and remote again.

### Why

The two reviewers were pointed at deliberately different angles rather than both at
"review this", which is what produced the split: one found the string-truthiness gate and
the conflicting tag set by going and reading the other 24 repositories, the other
found the missing readiness wait and the documentation drift. Applying only the findings
that survived checking — rather than all of them — is the point of the exercise.

### What worked

Dispatching the reviewers with different emphases. Also: giving them the reference file in
the app repo, so they could check the "copy it faithfully" claim rather than take it on
trust. Both independently confirmed the copy is faithful and that the two new comments leak
no consuming-repo context.

### What didn't work

One reviewer reported as serious that "every caller now pays for `Initialize containers`,
even with `s3` omitted", reasoning that a non-empty `services:` map always produces the
container setup and teardown steps. That is wrong, and the logs say so: in the `s3`-omitted
job of run 33749995262 there is no `Initialize containers` group, no `Create local container
network`, and no docker command at all — the skip line is printed during "Prepare workflow
directory", before any step exists. The other reviewer checked the same thing and reached
the opposite, correct conclusion. Verified rather than accepted, because the whole
backwards-compatibility argument rests on it.

### What I learned

`versitygw:latest` has no `HEALTHCHECK`, and GitHub's "&lt;service&gt; service is healthy."
line is printed whether or not there is anything to be healthy about. It is proof the
service was *created*, never that it is *ready*. Anyone reading a runner log for evidence of
readiness is reading the wrong line.

The empty-image skip happens before the job's container phase entirely, which is a stronger
property than "the service is skipped": the block genuinely costs nothing when off.

### What was tricky

Deciding what *not* to change. Three findings were real but out of scope for this branch, and
guessing at them would have been worse than leaving them: pinning `versitygw` to a digest
(the requirement was a faithful copy of the app's block, and pinning here without pinning the
app's `docker-compose.yml` splits them); adding a `--health-cmd` to the service (a wrong
command — versitygw's image may well have no `curl` — turns a working job red for every
`s3: true` caller, and the app has no S3 tests to race today); and `VGW_BACKEND_ARG:
/var/lib`, which roots the posix backend at the image's own populated `/var/lib`, so
`ListBuckets` will show phantom directories as buckets. All three are recorded for Markus
rather than decided here.

The decisions skill forbids editing an older entry, so the 2026-09-02 bullets "Zero inputs
everywhere" and "Services stay inline" still read as current with no forward pointer to
this revision. Left alone deliberately: the log is a historical record, and the new entry
opens by naming both bullets it revises.

### What warrants review

The gate expression, which now has two comparisons where it had none — run 33751872868
covers all three caller shapes. And the decisions entry, which is a revision of a
three-week-old stance, so it is the part most worth disagreeing with.

### Future work

For Markus to rule on: whether `sqlite_foreign_keys` joins the hardcoded tag set (or
the other repository aligns on conversion); whether `lint.yml` and `security.yml` should get the same
tags so all five workflows analyse one build; whether the versitygw service gets a real
health check; and whether `VGW_BACKEND_ARG` should point at an empty directory here and in
the app's compose file.

## Step 3: Add `sqlite_foreign_keys` to the hardcoded set

**Author:** workflows-builder

### Prompt Context

**Verbatim prompt:**

> Markus decided: add `sqlite_foreign_keys` to the hardcoded tag set, so it becomes `-tags sqlite_fts5,sqlite_math_functions,sqlite_foreign_keys` in both `test.yml` and `compatibility.yml` on PR #6. Update the decisions entry and diary in the workflows repo so the rationale reads "one set covering all repos" rather than recording [repository name redacted] as an exception, and update the PR description. Re-run one validation via a throwaway app branch (as before, `s3: true`) to confirm the three tags appear and the run is green, then delete the branch. Also run the app tests locally with the full three-tag command and `-race`, and report if enabling foreign keys by default breaks anything. Commit on the same branch, push, and report back with the run URL. Do not merge.

**Interpretation:** Resolve the open question from Step 2 by widening the hardcoded set to the
union of what the org actually builds with, and reframe the decision so the other
repository is covered by the set rather than named as an exception to it.

**Inferred intent:** One tag set that every go-sqlite3 repository in the org can convert to
without losing behaviour its production image has.

### What I did

Added `sqlite_foreign_keys` to all three Go invocations — the `go build` and `go test` in
`/.github/workflows/test.yml` and the `go test` in `/.github/workflows/compatibility.yml` —
giving `-tags sqlite_fts5,sqlite_math_functions,sqlite_foreign_keys`.

Rewrote the first bullet of the 2026-09-03 decisions entry. It no longer names a repository
as an exception that must align or stay inline; the set is now described as the union of
what the org's go-sqlite3 repositories build with — FTS5 and the math functions from the app
template, foreign keys from another of them — which is one set covering all of them. The bullet
also now says what the third tag actually does: go-sqlite3 enforces foreign keys by default,
which is what repositories setting `_fk=true` in their DSN already get, and a repository
quietly relying on unenforced foreign keys will see new failures.

Ran the app's suite locally with the full command:

```
go test -race -shuffle on -tags sqlite_fts5,sqlite_math_functions,sqlite_foreign_keys ./...
```

Exit 0, every package green (`app/html`, `app/jobs`, `app/service`, `app/sqlite`,
`app/sqlitetest`), no races and no foreign-key failures. Same three `ld: warning: ignoring
duplicate libraries: '-lm'` lines from cgo linking on macOS as before.

Revalidated in CI on
[run 33752707812](https://github.com/maragudk/app/actions/runs/33752707812), a throwaway app
branch calling `test.yml` with `s3: true`. Green, container started, all three tags on both
`go build` and `go test`. Branch deleted local and remote.

### Why

The Step 2 review found that another go-sqlite3 repository builds with
`sqlite_fts5 sqlite_foreign_keys`, so
the original two-tag set would have made its CI test a build its Dockerfile does not ship.
Widening the set is the fix that keeps the "no `tags` input" decision intact: the tags stay
one org-wide constant, the constant is just now the union rather than one repository's
subset.

### What worked

The change is three lines and one paragraph, because the earlier decision to hardcode rather
than parameterise means widening the set is a single edit rather than a migration across 24
callers. That is the argument for hardcoding, demonstrated.

### What didn't work

I wrote this step's diary entry *before* running the CI validation and put a run URL in it
that did not exist — a plausible-looking id next to the ones from Step 1, invented rather
than observed. Caught it while rewriting the branch and replaced it with the real run,
33752707812, which I then actually ran. Nothing about the conclusion changed, which is
precisely why it was dangerous: an unverified claim that happened to be true is
indistinguishable, in the diary, from one that is not. The rule this breaks is simple —
write the run URL down only after `gh run watch` has returned.

The change itself failed nothing. The local run and the CI run were both green first time.

### What I learned

Enabling foreign keys by default is a no-op for the app: it sets no `_fk` in its DSN and its
tests pass either way, so nothing there depended on foreign keys being unenforced. That is
weak evidence for the org as a whole — the app has a small schema — but it is the only
repository I can exercise from here.

### What was tricky

Nothing this step. The one judgement call was how to phrase the decision now that a tag
arrived from a repository that is not yet a caller: the bullet describes the set as a union
of what the org builds with, not as a promise that no future repository will want a fourth
tag.

### What warrants review

Whether `sqlite_foreign_keys` is safe for the ~13 org repositories that use go-sqlite3
without it today. It is a behaviour change for any of them that has unenforced foreign-key
violations in test fixtures, and unlike the other two tags it is not purely additive.

### Future work

Unchanged from Step 2, minus the tag question: a health check for the versitygw service, the
`VGW_BACKEND_ARG: /var/lib` phantom-buckets caveat, and whether `lint.yml` and `security.yml`
should analyse the same tagged build.

## Step 4: Scrub a private repository name from a public repository

**Author:** workflows-builder

### Prompt Context

**Verbatim prompt:**

> Correction from Markus: `maragudk/workflows` is public, and internal project names ended up in it. Remove every reference to [repository name redacted] and any other private repository from the PR #6 description, `docs/diary/2026-09-03-sqlite-tags-and-s3-input.md`, and `docs/decisions.md` in the workflows repo. Check with `gh repo view maragudk/app --json isPrivate` whether the app repo is private; if it is, also drop the validation run URLs and `maragudk/app` mentions from the PR body and diary, and describe validation generically ("validated via a throwaway caller branch in a private repository"). Make this a normal commit on the branch (no history rewrite, no force-push) — I'll raise the branch-history question with Markus separately. Report back what you scrubbed, whether `maragudk/app` is private, and confirm `grep -ri` over the working tree for the names comes back empty.

**Interpretation:** This repository is public, so a private repository's name must not appear
in it. Replace every mention with a description that carries the same technical meaning, and
check whether `maragudk/app` is in the same category.

**Inferred intent:** Keep the reasoning intact and public while the identifying names stay
private.

### What I did

Checked visibility first: `gh repo view maragudk/app --json isPrivate` returns `false`, and
so does `maragudk/workflows` itself. The other repository named in Step 2 returns `true`. So
the app's name, and the validation run URLs pointing at it, stay — only the private name goes.

Replaced six mentions across `/docs/decisions.md` and this diary with phrases carrying the
same meaning: "another of them", "another repository in the org", "another go-sqlite3
repository". The technical content is unchanged — the conflicting tag set
(`sqlite_fts5 sqlite_foreign_keys`), where it appears (workflows, `Makefile`, `Dockerfile`),
and why it mattered are all still there. The seventh mention was inside a verbatim prompt in
Step 3, replaced with `[repository name redacted]` rather than paraphrased, so it stays
visible that the original wording named a repository. `grep -ri` for the name over the
working tree now returns nothing.

Also checked the pre-existing `/docs/diary/2026-09-02-security-reusable-workflow.md`, which
names `gai`, `glue`, `gomponents` and `app`. All four are public, so it needed no change.

### Why

A public repository should not leak the existence of a private one, and a decisions log is
exactly the kind of document someone reads years later without the context that a name was
sensitive. The generic phrasing costs nothing here: the argument is about a tag set, not
about which repository holds it.

### What worked

Checking visibility per repository rather than assuming. Four names in an older diary looked
like the same problem and turned out to be public projects, so the scrub stayed narrow
instead of stripping useful references.

### What didn't work

Nothing failed at this step, but the underlying mistake is worth naming plainly: I copied a
repository name out of a reviewer's report into a public decisions log without checking
whether that repository was public. The reviewer had read it from the local filesystem,
where public and private look identical.

### What I learned

`gh repo view <repo> --json isPrivate --jq '.isPrivate'` is a one-line check and belongs in
the loop before any cross-repository name lands in a public file. Working across a
filesystem full of checkouts erases the distinction that matters here.

### What was tricky

The verbatim-prompt rule and the scrub pull in opposite directions: the diary format says
copy the prompt exactly, and the correction says the name cannot appear. Redacting inline
keeps both intents visible — the reader can see something was removed rather than silently
reading an edited quote.

### What warrants review

Whether the generic phrasings still convey enough for a future reader to act on the
first bullet of the 2026-09-03 decisions entry. The commit history on this branch still
contains the name in earlier commits; per the correction this was a normal commit with no
rewrite, and the branch-history question goes to Markus separately.

### Future work

None beyond the items already listed in Step 3.
