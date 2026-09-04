# Diary: A `postgres` input on the test workflows

`test.yml` and `compatibility.yml` already take an `s3` boolean input that declares a versitygw service container gated by an empty image expression. `maragudk/glue` needs the same treatment for Postgres, so this task adds a second input of the same shape, on the newest stable Postgres major, and validates it by pointing glue's own CI at the branch.

## Step 1: Add the `postgres` input and validate it through glue

**Author:** workflows-builder-pg

### Prompt Context

**Verbatim prompt:**

> Work in `/Users/maragubot/Developer/workflows` (GitHub `maragudk/workflows`, PUBLIC — never mention private repositories by name; `maragudk/glue` and `maragudk/app` are public and fine to name). Branch off `main` (e.g. `test-postgres-input`). Load `fabrik:git` before committing; use `fabrik:diary` and `fabrik:decisions` in the workflows repo.
>
> ## Context
>
> `test.yml` and `compatibility.yml` already accept an `s3` boolean input (default false) that declares a versitygw service whose image expression is empty when off — GitHub then skips the container entirely (verified in PR #6; read `docs/diary/2026-09-03-sqlite-tags-and-s3-input.md` and `docs/decisions.md` for the pattern and the gating expression that handles the string `"false"`). Markus decided to add a `postgres` input the same way, because `maragudk/glue` needs it.
>
> ## Requirements
>
> 1. Add a `postgres` boolean input (default `false`) to both `test.yml` and `compatibility.yml`. When true, a Postgres service container is declared with host port `5433` -> container `5432`, env `POSTGRES_USER=test`, `POSTGRES_PASSWORD=test`, `POSTGRES_DB=template1` — matching `/Users/maragubot/Developer/glue/.github/workflows/ci.yml` and `/Users/maragubot/Developer/glue/docker-compose.yml`; copy any `options:`/health-check they use. Gate it exactly like `s3` (same expression shape). Document the input inline like `s3` is documented: what it starts, the endpoint and credentials a caller's tests should expect.
> 2. **Image version:** Markus wants the newest stable Postgres major. Verify on Docker Hub which is newest with a stable (non-beta/RC) tag: `docker manifest inspect postgres:18 >/dev/null && echo ok`, likewise `postgres:19`; also check `gh api` / `curl https://hub.docker.com/v2/repositories/library/postgres/tags?name=19&page_size=20` for non-rc/beta tags. Use the major tag (e.g. `postgres:18`), not a patch pin. Report what you found.
> 3. Backwards compatible; existing callers unchanged. Keep style consistent with the `s3` block. Note: Postgres official image has no HEALTHCHECK either; if glue's workflow uses `options: --health-cmd pg_isready ...`, keep it.
> 4. Docs: extend the decisions entry (inputs are now `s3` and `postgres`) via the decisions skill; diary entry via the diary skill.
>
> ## Validation
>
> Validate through glue: in `/Users/maragubot/Developer/glue`, on a throwaway branch (e.g. `validate-shared-test-workflow`), replace the inline `test` job in `.github/workflows/ci.yml` with `uses: maragudk/workflows/.github/workflows/test.yml@<your-branch>` with `s3: true` and `postgres: true`; push; confirm via `gh run watch`/`gh run view --log` that both containers start and the job is green (glue's `postgrestest` and `s3test` hit the containers for real). Also dispatch `compatibility.yml@<your-branch>` with both inputs on that branch and confirm 4/4 legs. If `-race`, the `go mod tidy` gate, or the `sqlite_foreign_keys` tag makes glue fail, do NOT fix glue — report the exact failure and command. Delete the throwaway glue branch (local and remote) afterwards, leaving the glue checkout on `main` and clean. Also run locally in glue with `make test-up` then `go test -race -shuffle on -tags sqlite_fts5,sqlite_math_functions,sqlite_foreign_keys ./...` and `go mod tidy && git diff --exit-code go.mod go.sum`; report results; `make test-down` after.
>
> ## Deliverable
>
> Commit on the branch, self-review, open a PR against `main` with `gh pr create` (description: the change, validation run URLs, backwards-compatibility note, the Postgres version finding). Do not merge. Report back: PR URL, run URLs, Postgres version chosen and why, glue's local `-race`/tidy/tags results, and any question you could not resolve.

**Interpretation:** Copy the `s3` input pattern verbatim for a second service, pick the Postgres image version from evidence rather than habit, and prove the whole thing on real GitHub runners against a real caller instead of reasoning about YAML.

**Inferred intent:** `maragudk/glue` should be able to drop its inline `services:` blocks and call the shared workflows, without any existing caller noticing. The validation is the deliverable as much as the diff is — an untested `services:` change is worthless because nothing about it can be checked locally.

### What I did

Branched `test-postgres-input` off `main` and added a `postgres` boolean input, default `false`, to both `/.github/workflows/test.yml` and `/.github/workflows/compatibility.yml`, plus a matching service block. The service publishes host port 5433 to container 5432, sets `POSTGRES_USER=test`, `POSTGRES_PASSWORD=test`, `POSTGRES_DB=template1`, and is gated by the same empty-image expression the `s3` input uses, string comparison included: `image: ${{ (inputs.postgres == true || inputs.postgres == 'true') && 'postgres:18' || '' }}`. Inputs and services are both in alphabetical order now, `postgres` before `s3`/`versitygw`, which also matches the order glue uses inline.

For the image version I checked Docker Hub two ways. `docker manifest inspect postgres:18` succeeded, `postgres:19` and `postgres:20` failed. Listing the tag API (`curl -s "https://hub.docker.com/v2/repositories/library/postgres/tags?name=19&page_size=25"`) returned 110 matches, all of them `19beta1`, `19beta2` or `19beta3` variants — no stable 19 tag exists. The newest stable major is 18, currently 18.6, and `docker manifest inspect postgres:latest` is byte-identical to `postgres:18`. So `postgres:18` it is, following the major rather than pinning a patch.

The one deliberate deviation from glue: I added `options: --health-cmd pg_isready --health-interval 5s --health-timeout 5s --health-retries 5`. Requirement 3 said to keep such options if glue's workflow had them; it does not. See "What was tricky".

Validation ran on a throwaway glue branch, `validate-shared-test-workflow`, since deleted. Its `ci.yml` replaced the inline `test` job with two calls to `test.yml@test-postgres-input`: one passing `postgres: true` and `s3: true`, and a second, `test-no-services`, passing nothing at all. Its `compatibility.yml` called `compatibility.yml@test-postgres-input` with both inputs. glue's `ci.yml` only triggers on pushes to `main` and on pull requests, so I temporarily added the branch to the push trigger rather than opening a throwaway PR.

- CI run <https://github.com/maragudk/glue/actions/runs/33848536437>: `Test / test` green.
- Compatibility run <https://github.com/maragudk/glue/actions/runs/33848558443>: 2 of 4 legs green.
- glue's own scheduled Compatibility run on `main` the same morning, <https://github.com/maragudk/glue/actions/runs/33848624580>: the same 2 of 4 legs fail, with the same error.

Locally, with `make test-up`, `go test -race -shuffle on -tags sqlite_fts5,sqlite_math_functions,sqlite_foreign_keys ./...` in glue exited 0 across all packages, and `go mod tidy && git diff --exit-code go.mod go.sum` exited 0. `make test-down` afterwards; glue is back on `main` and clean.

Docs: a new dated entry in `/docs/decisions.md` and this diary file.

### Why

The gating expression, the port, and the credentials are copied rather than invented because the value of a shared workflow is that it matches what the callers already run inline — a caller should be able to delete its `services:` block and change nothing else. Alphabetical ordering is the only tie-breaker that survives a third input.

The version was checked rather than assumed because "newest stable" and "newest tag" differ right now: Docker Hub's most recently pushed Postgres tags include `19beta3`, and a `postgres:19` that resolved would have been a beta.

The second, input-less `test-no-services` job existed only to answer one question the green job cannot: does adding `options:` to a service block break the empty-image skip path that every existing caller depends on? Its tests were always going to fail — glue cannot test without a database — but its *setup* is the thing under test.

### What worked

The empty-image gating extended to a second service with no surprises, and the `options:` key rides along harmlessly when the image is empty. The `test-no-services` job logged both lines and initialised no containers at all:

```
The service 'postgres' will not be started because the container definition has an empty image.
The service 'versitygw' will not be started because the container definition has an empty image.
```

and then failed exactly where it should, in `postgrestest`:

```
--- FAIL: TestHelper_Ping/can_ping
    helper_test.go:13: failed to connect to `user=test database=postgres`:
        127.0.0.1:5433 (localhost): dial error: dial tcp 127.0.0.1:5433: connect: connection refused
```

That is the backwards-compatibility proof: an existing caller passing nothing gets two log lines and no containers, `options:` or not.

The health check demonstrably earns its place. The green job's log shows the runner actually waiting on it:

```
postgres service is starting, waiting 2 seconds before checking again.
postgres service is starting, waiting 3 seconds before checking again.
healthy
postgres service is healthy.
```

Roughly five seconds that glue's inline job spends racing the server instead. The container itself came up as expected — `starting PostgreSQL 18.6 (Debian 18.6-1.pgdg13+2)` — and glue's real Postgres and S3 packages passed against both containers: `ok maragu.dev/glue/postgrestest 1.603s`, `ok maragu.dev/glue/s3test`, `ok maragu.dev/glue/sql`.

Postgres 18 caused glue no trouble at all, even though glue develops against `postgres:17` in its `docker-compose.yml`. The `template1` migration dance, the advisory lock, `create database`/`drop database` — all fine.

### What didn't work

Two of the four compatibility legs failed, both of them the `deps: latest` legs that run `go get -u -t ./...` first:

```
--- FAIL: TestMiddlewareErrors/Authenticate_should_keep_the_error_description_under_an_otelhttp_handler
    auth_test.go:734: no ended span named GET /, recorded [GET]
--- FAIL: TestMiddlewareTelemetryComposition/Authenticate_and_Authorize_should_record_on_the_span_of_a_plain_otelhttp_handler
    auth_test.go:861: no ended span named GET /, recorded [GET]
FAIL	maragu.dev/glue/http	0.093s
```

This is pre-existing and unrelated. glue's own scheduled Compatibility workflow on `main` — inline services, no `-race`, no build tags, no shared workflow anywhere near it — failed the same two legs with the same assertions the same morning, and `gh run list --workflow compatibility.yml --branch main --limit 5` shows `failure` every day back to 2026-08-31. It is an upstream otelhttp span-naming change that only surfaces once dependencies are upgraded past what `go.sum` locks. Per the brief I did not fix glue.

Everything the brief specifically warned might break glue did not: `-race`, the `go mod tidy` gate and the three `sqlite_*` tags all passed, both locally and in the green CI job.

### What I learned

`postgres:latest` and `postgres:18` currently resolve to the same manifest, which is a cheap cross-check that 18 really is the newest stable major rather than merely the newest one with a `:N` tag. The Docker Hub tag listing is the load-bearing evidence, though: it is sorted by push date, so `19beta3` sits above `18.6` and a glance at the top of that list would have been misleading.

GitHub Actions renders a `workflow_call` service block into a plain `docker create`, options and all, which the log prints verbatim — a good way to confirm an expression evaluated the way you meant:

```
/usr/bin/docker create ... -p 5433:5432 --health-cmd pg_isready --health-interval 5s --health-timeout 5s --health-retries 5 -e "POSTGRES_USER=test" ... postgres:18
```

A caller whose `ci.yml` only triggers on `main` and pull requests will run nothing at all when you push a validation branch. Adding the branch to the `push:` trigger on the throwaway branch is less ceremony than opening and closing a pull request.

### What was tricky

The health check was the only real judgement call. Requirement 3 anticipated glue having `options: --health-cmd pg_isready ...` and said to keep it; glue has no `options:` at all, so read literally the instruction was to copy nothing. I added the health check anyway, and the reasoning is worth stating because it is a deviation:

The 2026-09-03 decisions entry already recorded that versitygw's "service is healthy" line is not a readiness wait — the image ships no `HEALTHCHECK`, so a test hitting it immediately races startup — and accepted that because versitygw offers no in-image probe. Postgres does ship `pg_isready`, so the same race is fixable in four words. The shared workflow serves every caller, not just glue, and inheriting a known race into all of them to stay byte-identical with one caller seemed like the wrong trade. The logs above show the wait was real, so this is not theoretical. Still, it is a deviation from a caller Markus pointed at, and he may prefer strict parity — flagged in the report.

The other small friction: proving the *off* state needs a job that fails. `test-no-services` is red by construction, which makes the validation run red overall and needs reading per-job rather than trusting the run's conclusion.

### What warrants review

The health-check deviation above is the thing to have an opinion about. After that, `postgres:18` versus glue's `postgres:17`: the shared workflow deliberately runs one major ahead of what these repositories develop against, which is either a useful early warning or an unwanted divergence depending on taste. It cost glue nothing today.

Worth noting the cost side too: `compatibility.yml` with both inputs true now starts eight service containers per run, four Postgres and four versitygw. Both inputs default to `false`, so only callers that ask pay it.

Also confirm the decisions entry is where it should be. The decisions skill says not to modify old entries, and the brief said to extend the 2026-09-03 one; I split the difference with a new 2026-09-04 entry that states the inputs are now `s3` and `postgres` and amends the earlier one by reference, rather than backdating today's work into yesterday's paragraph.

### Future work

The image is an unpinned floating major tag that nothing watches — dependabot's `github-actions` ecosystem does not track `services:` images — so a bad upstream `postgres:18` push breaks every `postgres: true` caller at once. The same already applies to `versity/versitygw:latest`. Two unwatched images is a better argument for a watcher than one was.

glue itself can now drop the inline `services:` blocks from its `ci.yml` and `compatibility.yml` and call these workflows, which is presumably the point; that is a separate change in a separate repository. Its `deps: latest` compatibility failure is a real, unrelated bug someone should look at.
