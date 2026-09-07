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
