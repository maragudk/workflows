# Diary: A `secret-env` input on the test workflows

The shared `test.yml` and `compatibility.yml` have carried "still no secrets" since 2026-09-02,
because the only credentials they needed were the fixed throwaway pairs for containers the job
starts itself. A repository whose tests need real API keys from repository secrets cannot be
served that way, and the 2026-09-02 entry parked it by keeping such jobs inline. This task adds
the general mechanism instead: the caller lists its own secret names, passes `secrets: inherit`,
and the shared workflow resolves the names at run time without ever declaring one.

## Step 1: Add the input, the export step, and prove both on a runner

**Author:** fabrik:builder

### Prompt Context

**Verbatim prompt:**

> You are building in the repo `maragudk/workflows`, local checkout at /Users/maragubot/Developer/workflows (currently on `main`, clean). This is NOT the c6 repo. Work on a feature branch in a git worktree of that repo (e.g. `git -C /Users/maragubot/Developer/workflows worktree add ../workflows-secret-env -b secret-env-input`), never commit on `main`. Open a PR at the end; do not merge it. Do not sign up for or create any account, key, or service. Use `gh` with the git and diary conventions this repo already follows (read `docs/decisions.md` and the latest entries in `docs/diary/` first — the `postgres` input PR #7 is the model for how an input gets added, decided, and diaried here).
>
> ## Requirement
>
> Add a general way for a calling repository to expose repository secrets to the test step as environment variables, without the shared workflow ever learning any repo's secret names.
>
> Design (decided by the product owner, Markus — do not redesign):
>
> - New `workflow_call` input `secret-env` on `.github/workflows/test.yml`, type `string`, default `""`. Value: newline- or whitespace-separated names of secrets. Each named secret is exported into the test step's environment under the same name. Requires the caller to pass `secrets: inherit`; document that in the input description, following the style of the existing `postgres`/`s3` descriptions.
> - Implement as a step before `Test`, guarded on `inputs.secret-env != ''`, that reads `toJSON(secrets)` from an env var and writes each listed name to `$GITHUB_ENV` with a heredoc delimiter (random suffix) so multiline values survive. Unset names become empty strings, which matches fork-PR behaviour where secrets are empty anyway. Only the listed names reach `go test`; never export everything.
> - If `compatibility.yml` in this repo also runs `go test`, give it the same input and step so the two stay consistent (check `docs/decisions.md` for whether the two are meant to mirror each other).
> - README/docs: whatever this repo uses to document inputs, update it.
> - Record a decision in `docs/decisions.md` via the decisions skill, and a diary entry via the diary skill, in this repo's own style. The decision's rationale, briefly: `workflow_call` inputs are string/number/boolean only; the `secrets` context is not available in a caller's `with:`, and job outputs containing secrets are redacted, so secret values cannot travel as inputs; the only route is `secrets: inherit` with the called workflow resolving names dynamically. Alternatives rejected: declaring each secret by name upstream (per-repo churn in the shared repo), and exporting all inherited secrets with no input (every secret lands in the test process's environment).
>
> Reference sketch of the step:
>
> ```yaml
> - name: Export secrets to env
>   if: inputs.secret-env != ''
>   env:
>     ALL_SECRETS: ${{ toJSON(secrets) }}
>     NAMES: ${{ inputs.secret-env }}
>   run: |
>     d="ghadelim$RANDOM$RANDOM"
>     for n in $NAMES; do
>       { printf '%s<<%s\n' "$n" "$d"
>         jq -r --arg n "$n" '.[$n] // ""' <<<"$ALL_SECRETS"
>         printf '%s\n' "$d"; } >> "$GITHUB_ENV"
>     done
> ```
>
> Adapt to this repo's conventions; keep comments sparse and inside their layer (say what the step does for the caller, not GitHub trivia the docs already cover). The first consumer will be a private app repo whose tests need two API keys from repository secrets; you don't need to name it.
>
> ## Validation
>
> There is no local runner for reusable workflows. Validate the shell logic locally by running the loop against a fake `ALL_SECRETS` JSON (including a multiline value and a missing name) with `GITHUB_ENV` pointed at a temp file, and check the file parses as the runner expects. Run `actionlint` if it's available in this repo's toolchain (check the Makefile/CI). If this repo has a way to exercise a workflow on a branch (a `workflow_dispatch` or a self-test caller), use it and report the run URL.
>
> Self-review before opening the PR. Report back: branch, PR URL, what you validated, and any open questions.

**Interpretation:** Implement the design as given, in this repository's established style, and
prove the shell logic rather than reason about it — first against a local fake, then, if there
is any way to do it, on a real runner where `toJSON(secrets)` is a real secrets context.

**Inferred intent:** A repository that needs API keys in its tests should be able to call the
shared workflows instead of keeping an inline test job forever, and this repository should stay
ignorant of what those keys are called.

### What I did

Branched `secret-env-input` off `main` in a worktree at `/Users/maragubot/Developer/workflows-secret-env`
and added, to both `/.github/workflows/test.yml` and `/.github/workflows/compatibility.yml`, a
`secret-env` string input defaulting to `""` and an `Export secrets to the environment` step
immediately before `Test`. Inputs stay alphabetical, so it sits after `s3`. The step is the
briefed sketch plus one addition, a guard rejecting a name that is not a valid environment
variable identifier; see "What was tricky".

Documentation of inputs in this repository is the inline `description:` on the input itself —
the README is deliberately a signpost, and the 2026-09-03 brief said not to touch it — so the
description carries the contract: the caller must pass `secrets: inherit`, a name with no secret
behind it is exported as an empty string, and nothing but the listed names reaches the tests.

Validation came in three layers.

**Local, against the real YAML.** A harness pulls the step's `run:` body straight out of the
workflow file with PyYAML and executes it under `bash -e` — the runner's default shell — with a
fake `ALL_SECRETS`, then parses the resulting `$GITHUB_ENV` the way actions/runner does.
Eighteen assertions cover a simple value, a multiline PEM, a value containing `=`, an
empty-string secret, an unset name, a value ending in a newline, a value containing a
delimiter-shaped line, whitespace- and newline-separated input, whitespace-only input, that only
the listed names appear, that `github_token` does not appear when unlisted, that nothing reaches
stdout or stderr, and that `compatibility.yml` carries a byte-identical step. A second pass
covered a 48 KB value (GitHub's per-secret cap), a 2.6 KB PEM, non-ASCII, tabs, backslashes,
shell metacharacters (`$HOME`, backticks, `$(id)`), and the two values jq's `//` operator could
plausibly mangle, the literal strings `"null"` and `"false"`. All round-trip byte-exact.

**Mutation testing**, because a green harness proves nothing about the harness. Five mutants of
the step, run from a scratch copy: dropping jq's `// ""` default, a fixed rather than random
delimiter, plain `NAME=value` instead of the heredoc, a weakened name guard, and exporting every
key in `ALL_SECRETS` rather than the listed ones. All five were caught, by between one and seven
assertions each.

**`actionlint` v1.7.12**, which is not in this repository's toolchain (there is no Makefile and
no CI of its own), run via `go run github.com/rhysd/actionlint/cmd/actionlint@latest` from a
scratch module so nothing was added to the repository. Clean on the branch and on `main`.

**A real runner.** There is no `workflow_dispatch` or self-test caller here, so I made one on a
throwaway branch, `secret-env-input-validation`: a `secret-env-probe.yml` holding the input and
the export step verbatim plus an assertion step, and a `secret-env-selftest.yml` calling it four
ways. Run
[34099209667](https://github.com/maragudk/workflows/actions/runs/34099209667):

- `listed` — green. `secret-env: "github_token\nNO_SUCH_SECRET"` with `secrets: inherit`.
  `github_token: set=yes length=377` proves the inherited context reached the step;
  `NO_SUCH_SECRET: set=yes length=0` proves an unset name is exported empty rather than skipped;
  `MULTI: set=` proves an unlisted key is not exported.
- `omitted` — green. No input, so the step is skipped and nothing is exported at all.
- `no-inherit` — green, and see "What I learned".
- `malformed` — red by construction, `secret-env: "github_token,NO_SUCH_SECRET"`, failing with
  `secret-env: "github_token,NO_SUCH_SECRET" is not a valid environment variable name; separate names with spaces or newlines`.

The throwaway branch is deleted local and remote; the run and its logs survive it. Neither probe
file is on the feature branch.

Docs: a new dated entry in `/docs/decisions.md` and this file.

### Why

The heredoc with a random delimiter, rather than `NAME=value`, is the only form that survives a
multiline secret — and the first consumer's kind of credential (API keys today, private keys and
service-account JSON tomorrow) is exactly where that bites. The mutation run shows the plain
`NAME=value` form failing two assertions, one of them "the file parses at all".

The step sits after `Build` in `test.yml` rather than at the top of the job, so the secrets are
in the environment for `go test` and for nothing else.

Extracting the step body from the YAML instead of copying it into the harness is what makes the
local layer worth anything: the thing under test is the shipped text, and the harness fails the
moment the two workflows drift apart.

### What worked

Building the runner validation as a separate probe workflow rather than pointing a caller at
`test.yml` itself. The shared workflow would have needed a Go module to get past `setup-go`, and
the interesting behaviour is entirely in the one step; copying that step into a probe let all
four caller shapes run in about a minute, in this repository, with no other repository involved.

`github_token` turned out to be the one secret that can be tested on a runner without creating
one. It is always in the `secrets` context, it is long enough to distinguish "inherited" from
"empty", and its value is masked in the log — the run shows `github_token: ***` where the runner
echoes the step environment, which incidentally confirms that values exported this way are still
masked.

### What didn't work

`actionlint` caught a mistake in the throwaway self-test caller before it ever ran:

```
secret-env-selftest.yml:24:5: when a reusable workflow is called with "uses", "continue-on-error" is not available. only following keys are allowed: "name", "uses", "with", "secrets", "needs", "if", and "permissions" in job "no-inherit" [syntax-check]
```

I had wanted the two deliberately-failing jobs marked `continue-on-error` so the run would be
green overall. A `uses:` job cannot take it, so — as in the 2026-09-03 entry, where proving the
off state also needed a job that fails — the run has to be read per job rather than by its
conclusion.

Nothing else failed. The step was green on the first local run and the first CI run.

### What I learned

**`toJSON(secrets)` is not empty without `secrets: inherit`.** The `no-inherit` job passed its
`github_token` assertion, because `GITHUB_TOKEN` is in a called workflow's `secrets` context
whether or not the caller inherits. So a caller that sets `secret-env` and forgets
`secrets: inherit` gets empty strings for its own secrets and no warning — the failure surfaces
in the test, as a missing API key, not in the export step. That is what the input description
has to be clear about, and it is why the description says the caller must pass it rather than
merely should. Worth being precise about what the run proved: it proves `GITHUB_TOKEN` is always
present, not that repository secrets are absent without `inherit` — testing that half would need
a repository secret, which I did not create.

**jq's `//` operator is falsy on `false` and `null`, not on `""`.** A secret whose value is the
string `"false"` is a JSON string and therefore truthy, so `.[$n] // ""` returns it intact;
an empty secret is likewise returned intact. Only a genuinely absent key falls through to `""`,
which is precisely the wanted behaviour, but it is a coincidence of jq's semantics rather than
something the expression says out loud — hence the two literal-string cases in the harness.

**A deleted branch does not delete its run.** The validation run URL still resolves after the
throwaway branch is gone, so the cleanup and the evidence do not trade off against each other.

### What was tricky

The one deviation from the briefed sketch is a guard, before the write, rejecting a name that is
not `[A-Za-z_][A-Za-z0-9_]*`:

```
secret-env: "github_token,NO_SUCH_SECRET" is not a valid environment variable name; separate names with spaces or newlines
```

Without it, the most likely caller mistake — writing a comma-separated list, given every other
list-shaped thing in YAML — produces `FOO,BAR<<delim` in `$GITHUB_ENV`. The runner does not
reject that; it sets a variable named `FOO,BAR` and consumes the heredoc happily, so the
variables the caller actually asked for are simply missing and the tests fail somewhere far away
with an empty API key. Turning a silent misconfiguration into a loud one costs four lines. It is
still an addition to a design the product owner specified, so it is flagged rather than assumed:
if the preference is strict fidelity to the sketch, deleting the `case` block is the whole
revert and the harness records exactly which assertions then fail.

The other judgement call was leaving `for n in $NAMES` unquoted with no `set -f`. The unquoted
expansion is the mechanism — it is how whitespace and newlines both separate names — but it also
means a name containing a glob character is expanded against the checkout first. In practice the
guard then rejects whatever comes back, because filenames in a Go repository contain dots and
slashes; a caller passing a bare `*` in a repository with a file called `README` would get an
empty `README` variable and no error. That seemed too remote to spend a line of shell on, but it
is the sharpest remaining edge.

### What warrants review

The guard, above — it is the only thing in the diff that was not specified.

After that, the input description, which is the entire user-facing documentation of the feature
and makes three claims a caller will rely on: that `secrets: inherit` is required, that an
unmatched name is an empty string, and that nothing else reaches the tests. The third is the
security property; the runner log for the `listed` job (`MULTI: set=`) and the "export every
inherited secret" mutant are the evidence for it.

Also worth an opinion: whether `compatibility.yml` should have this at all. It mirrors `test.yml`
by convention, and a repository whose tests need API keys needs them in all four matrix legs, so
consistency says yes — but it does mean the `deps: latest` legs run third-party code that has
just been upgraded, with the caller's secrets in the environment.

### Future work

The starter templates in `maragudk/.github` will want a `secret-env:` example with the
`secrets: inherit` line beside it, since a caller that copies a template and adds only the input
gets empty values and a confusing test failure. That is a change in another repository.

Nothing here warns a caller that set `secret-env` without `secrets: inherit`. It cannot be
detected from inside the called workflow — `GITHUB_TOKEN` is present either way — so if it turns
out to be a common mistake, the fix is a note in the starter template rather than a check here.

## Step 2: Self-review, and what a reviewer found

**Author:** fabrik:builder

### Prompt Context

**Verbatim prompt:** (no new prompt — the deliverable in Step 1 says "Self-review before opening the PR.")
**Interpretation:** Read the diff as a reviewer would, then dispatch reviewers at deliberately
different angles, apply the findings that survive checking, and record the ones that do not.
**Inferred intent:** Catch the mistakes a change that ~24 repositories pin `@main` cannot afford
to ship with.

### What I did

Read the diff, then dispatched two reviewers, one at correctness and secret leakage, one at
operational consequences and house style. Four findings changed the code and two changed the
decisions entry.

- **The description overclaimed.** It said the secrets go into "the test step's environment".
  `$GITHUB_ENV` sets variables for every step that follows, not one; that reads as true today
  only because `Test` happens to be last in both files. Reworded to "the environment of the
  steps that follow, which is the test run", which stays true if a coverage upload is ever added
  after `Test` — and makes it visible that such a step would inherit the secrets.
- **The caller obligation was in the wrong place.** Every workflow here carries a top-of-file
  `# Callers must grant …` comment; `secrets: inherit` is an obligation of exactly that kind and
  was buried in an input description. The comment in both files now reads
  `# Callers must grant contents: read, and pass secrets: inherit to use secret-env.`
- **The one comment explained the wrong half.** It read "A random heredoc delimiter, so multiline
  secrets such as private keys survive" — but multiline values survive because it is a heredoc at
  all. The randomness is there for a reason a reader cannot derive from the code: a value
  containing a line equal to the delimiter closes the heredoc early, and its remaining lines are
  then read as further assignments. That is variable injection out of a secret's own contents.
  The comment now says that, and drops "such as private keys", which was this repository
  guessing at what a particular caller keeps in its secrets.
- **The step was loud about the rare mistake and silent about the likely ones.** The guard fails
  a malformed name, but a name that simply resolves to nothing exported an empty string with no
  signal — and four different caller errors land there: a forgotten `secrets: inherit`, a
  mis-cased name (`toJSON(secrets)` keys are uppercase), a genuinely empty secret, and a fork
  pull request. Only the last is legitimate. Two lines now log
  `::warning::secret-env: no secret named $n, so it is exported empty`, naming it; secret names
  are not secret, and the semantics the description promises are unchanged.

The decisions entry gained the concession it was glossing over — `secrets: inherit` means a
reader of a caller's `ci.yml` can no longer see which secrets cross the boundary, which is
exactly the "explicit declarations keep the surface auditable" half of the 2026-09-02 rejection
— and a note that `GITHUB_TOKEN` is in the context and therefore exportable by name.

Then re-ran everything: 22 assertions green, the edge-case pass (48 KB value, PEM, non-ASCII,
tabs, backslashes, shell metacharacters, the literal strings `"null"` and `"false"`, a value
ending in a newline) byte-exact, and `actionlint` clean.

### Why

The reviewers were pointed at different angles rather than both at "review this", which is what
produced the split: the operational reviewer found the `$GITHUB_ENV` scope overclaim and the
misplaced caller obligation by reading the other four workflows for convention, neither of which
is visible from the diff alone.

The warning was the finding worth the most. It is two lines, it has no ongoing cost, and it
turns the single most likely misconfiguration — a caller who adds `secret-env:` and forgets
`secrets: inherit`, which the runner validation in Step 1 showed is undetectable from inside the
called workflow — from a silent empty string into a line in the log.

### What didn't work

Nothing regressed. The one thing worth recording is a limit of the mutation testing rather than
a failure: after the warning landed, weakening the name guard no longer breaks the assertion
that the error names the offending input, because the warning names it too. The guard is still
covered by the assertion that a malformed name fails the step, but the two checks now overlap,
and that overlap is an argument for the reviewer's position that the warning subsumes part of
the guard.

Also worth naming: two of the reviewer's three "major" findings — that no decision or diary
entry existed, and that there was no evidence the workflow had ever been run — were artefacts of
reviewing a mid-flight branch. Both existed by the time the report arrived. That is a cost of
dispatching a reviewer before the docs commit, not a finding.

### What I learned

`toJSON(secrets)` keys are the stored names, which GitHub uppercases, so `secret-env: my_api_key`
resolves to nothing while passing every validity check the step makes. That case is why the
warning had to key on "resolved to nothing" rather than on name shape — no amount of syntactic
checking would have caught it.

An error message and a warning covering overlapping ground is easy to arrive at accidentally and
shows up as mutants that stop biting. Mutation testing is as useful for spotting redundant checks
as for spotting missing ones.

### What was tricky

Deciding what not to take. Four findings were real and were left alone deliberately:

- **`set -f` before the loop.** Recorded in Step 1 and unchanged: a name containing a glob
  character expands against the checkout first. The reviewer reached the same conclusion — the
  accident is benign — so it stays a known sharp edge rather than a line of shell.
- **A sentence saying a listed name overrides an environment variable of the same name.** True
  — a secret named `GOFLAGS` or `PATH` would quietly change how `go test` runs — but the
  description is already the longest of the three in the file, and "exported under the same
  names" says it. A denylist was rejected by the reviewer too.
- **Trimming the guard's error message,** which repeats the description's "separated by
  whitespace or newlines". It is duplication, but an error message a caller reads at 5pm should
  carry its own fix rather than send them back to the input docs.
- **Comments on the `if:` gate and the `case` guard.** The gate is worth understanding — a
  skipped step's `env:` is never evaluated, so it is what keeps `toJSON(secrets)` from
  materialising for the callers that do not use this — but `security.yml` carries twenty lines
  of comparable shell with no comments at all, and the step now has one comment where it had
  one. Adding two more would be the comment volume this repository has trimmed before.

### What warrants review

Unchanged from Step 1: the guard and the warning are the two things in the step that were not in
the specified design, and they are now deliberately overlapping. If only one is wanted, the
warning is the one that covers more ground; the guard is what stops a malformed line reaching a
file the runner parses.

The reworded input description is the whole user-facing contract and is worth reading as a
caller would.

### Future work

Unchanged from Step 1.

## Step 3: A second review round, and the transport changes

**Author:** fabrik:builder

### Prompt Context

**Verbatim prompt:**

> Your git isolation is restored to the workflows checkout (the lead has left the c6 worktree); carry on in /Users/maragubot/Developer/workflows-secret-env on branch `secret-env-input`. Decisions on your three questions, plus one new finding:
>
> 1. Keep both hardenings: `set -f` and the `openssl rand -hex 16` delimiter.
> 2. An absent secret is NOT exported (not exported empty). Keep the warning. Update the input description, the decision text and the diary to match.
> 3. `compatibility.yml` does NOT get the input. Revert whatever you added there; only `test.yml` carries `secret-env`.
> 4. New finding from an independent codex review of the branch: `env: ALL_SECRETS: ${{ toJSON(secrets) }}` puts the whole secrets context in one environment string, and Linux caps a single env string at MAX_ARG_STRLEN = 128 KiB. GitHub allows 48 KB per secret, so a caller with three large inherited secrets (repo + org) gets `Argument list too long` at step launch regardless of what `secret-env` lists. Fix: don't transport the JSON through `env:`; write it to a file under `$RUNNER_TEMP` from the script body instead (a `run:` script has no such limit), e.g. a quoted heredoc `cat > "$RUNNER_TEMP/secrets.json" <<'EOF'` with `${{ toJSON(secrets) }}` on the following lines — `toJSON` pretty-prints one key per line with values as JSON strings, so a bare `EOF` line cannot occur inside a value; confirm that reasoning yourself and add a local test with a value containing a literal "EOF" line. Then `jq` reads from the file. Delete the file after export. Say in the diary why the transport changed.
>
> Then: run your local test suite and actionlint, re-run the runner probe (confirm `openssl` and the file transport on a real runner), commit, push, refresh the PR body, and report back with the final PR state. Do not merge.

**Interpretation:** Take all four rulings as given, verify the fourth's reasoning rather than
apply it on faith, and re-prove the whole step on a runner now that both the transport and the
absent-name semantics have changed.

**Inferred intent:** The step should be correct for a caller with large secrets, not only for
one with short API keys, and `compatibility.yml` should stop mirroring `test.yml` where the
mirror is actively harmful.

### What I did

- **Reverted `compatibility.yml` to `main`.** Only `test.yml` carries `secret-env` now. The
  harness gained two assertions that the file has neither the input nor the step, so the mirror
  cannot come back by accident.
- **An absent name is no longer exported.** `jq -e 'has($n)'` decides; a miss warns and
  `continue`s. The input description, the decision entry and the reasoning all changed with it:
  the old text defended the empty string by pointing at fork pull requests, and that was the
  argument that undid it — a fork gets no secrets at all, so with the old behaviour every listed
  name arrived set-but-empty and `if _, ok := os.LookupEnv(k); !ok { t.Skip() }` stopped
  skipping. Leaving the name unset restores what a test sees when this input is not used.
- **Moved the secrets context out of `env:`.** `ALL_SECRETS` is gone. The step now writes
  `${{ toJSON(secrets) }}` into a file under `$RUNNER_TEMP` through a quoted heredoc in the
  script body, `jq` reads the file, and `trap 'rm -f "$f"' EXIT` deletes it on every exit path,
  the guard's `exit 1` included.
- Kept `set -f` and the `openssl rand -hex 16` delimiter.

### Why

I checked the `env:` finding rather than taking it, and it holds exactly as stated. Locally it
looked like a non-issue — macOS has no per-string cap, and 144 KiB in one variable works fine —
so I put the question on the runner instead, in a plain `ubuntu-latest` job:

```
100 KiB: ok
127 KiB: ok
129 KiB: FAILS (argument list too long)
144 KiB: FAILS (argument list too long)
```

That is `MAX_ARG_STRLEN`, 32 pages, on the nose. Three 48 KB secrets — repository plus
organisation, which a caller does not choose — make `toJSON(secrets)` about 144 KB and the step
would have died at launch before running a line, no matter what `secret-env` listed. The bug was
invisible to every test I had written, because all of them used short values.

The heredoc that carries the payload is safe for a reason worth stating, since it looks like the
same delimiter-collision problem the export heredoc has: it is not, because JSON escapes
newlines inside strings. A multiline secret is one physical line, `"KEY": "line1\nline2"`. So
every line of the payload is `{`, `}`, or an indented `"key": "value"` pair, and a bare
`SECRETSJSON` line cannot occur — not from a value, and not from a key, since a key always
arrives quoted and indented. I asserted it rather than trusted it: the harness reads the
terminator out of the workflow, feeds in a secret whose value contains that exact word on its
own line, and checks both that the value survives intact and that the secret after it in the
payload still arrives. Quoting the heredoc is what stops `$` in a value being expanded, and a
mutant that unquotes it is caught.

Only `test.yml` gets the input because `compatibility.yml`'s `deps: latest` legs run
`go get -u -t ./...` and then execute third-party code that was upgraded seconds earlier, with
the caller's API keys in the environment. That is a real difference in kind, not a lapse in
consistency.

### What worked

Re-running the harness against a 147 KB payload took one line, and it passes — the same shape
that breaks through `env:`. Putting the cap probe in the validation run as its own job means the
premise is recorded as a measurement rather than as a citation.

The `trap` covers the guard's `exit 1` path for free, which an `rm` at the end of the loop
would not have.

### What didn't work

I destroyed my own uncommitted work and had to recover it. Wanting a clean tree before branching
for the probe, I ran `git stash && git checkout -b secret-env-input-validation && git stash pop`,
committed everything onto the throwaway branch — the real change included — and then deleted
that branch with `git branch -D` as the cleanup step. `git status` came back clean, which is
what a finished branch and an emptied branch look like alike. The tip hash was still in the
scrollback, so `git checkout 4e1f4e0 -- .github/workflows/test.yml .github/workflows/compatibility.yml`
brought it back, and the harness confirmed it. The lesson is narrow and sharp: commit the real
change on its own branch *before* creating a throwaway one, never carry a working tree across.

Earlier in the same step, every `Bash` call started failing with

```
This session is isolated in the worktree /Users/maragubot/Developer/c6v2/.claude/worktrees/merge-upstream, but this command's working directory resolved to the shared checkout (/Users/maragubot/Developer/c6v2). Refusing to run it there
```

because the lead had moved worktrees mid-task, and the isolation target moves with them. There
is no fix from inside the sub-agent: the right move is to stop and report rather than commit
somewhere else, which is what I did.

### What I learned

`MAX_ARG_STRLEN` is a per-string limit and is Linux-only; macOS enforces a total `ARG_MAX` but
no per-variable cap. So a shell test on a development machine cannot see this class of bug at
all, and a local harness that runs the real script under the real shell is still the wrong
environment. Anything about process-launch limits has to be measured on the runner.

`toJSON`'s pretty-printing is what makes a quoted heredoc a safe transport, and the property is
JSON's rather than GitHub's — any conforming serializer escapes newlines in strings. That is
worth knowing because it means the safety does not depend on a formatting detail that could
change.

### What was tricky

Choosing the predicate for "no secret behind it". `has($n)` distinguishes absent from
present-and-empty, and only absent now skips the export; a secret that exists with an empty
value is exported empty, faithfully, and does not warn. That is a real behavioural seam and the
warning no longer covers it, which is a deliberate narrowing from Step 2 — the warning's job is
to catch a name that resolves to nothing, and an empty secret is a value the caller chose.

### What warrants review

The transport. It is the largest change in the branch and the least like the specified sketch,
and the argument for its safety is the JSON-escaping property above — worth disagreeing with if
you think it does not hold.

After that, `test.yml` and `compatibility.yml` no longer mirror each other for the first time.
The decision entry says why; if that reasoning is wrong, the fix is to add the input to
`compatibility.yml` and it is a copy of one block.

### Future work

Unchanged from Step 1: the starter templates in `maragudk/.github` want a `secret-env:` example
with `secrets: inherit` beside it.

Four smaller things a reviewer flagged and I did not act on, recorded so they are not
rediscovered: GitHub stores secret names uppercase, so `secret-env: my_key` never resolves and
now warns rather than exporting an empty value; `secrets: inherit` does not carry *environment*
secrets, because `on.workflow_call` has no `environment` keyword, so a caller whose keys live in
a GitHub Environment gets warnings and nothing else; a secret name cannot begin with `GITHUB_`,
so `secret-env: GITHUB_TOKEN` can never resolve, while lowercase `github_token` does and hands
the job token to `go test`; and `NODE_OPTIONS` is on the runner's `$GITHUB_ENV` blocklist, so
listing it logs a runner error and sets nothing.
