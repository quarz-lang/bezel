# Working on bezel

A cog of Quarz (github.com/quarz-lang/quarz): its rules apply here.
Code, comments, commit messages and the docs are in English.

## Layout

- The library cog `bezel` at the root: `Cog.toml` (`module = "Bezel"`),
  `src/` (declarations only; `src/plug/` the built-in plugs),
  `test/*_test.qz` (`require "test"`, Test::Case), README.md, LICENSE.
  The application cog `bezel-cli` (the `bezel` command) will live in
  `cli/`.
- Depends on rotor (`Rotor`) and pg (`PG`), their main branches. To
  develop against local checkouts, write `Cog.local.toml` (ignored by
  git): `[patch]` / `Rotor = { path = "../quarz-rotor" }`. CI uses their
  main branches, as users do: a bezel change that needs a rotor or pg
  change waits for that PR.
- The design and its decisions are in issue #1 and the issues of each
  part (#3 to #9); a decision taken later is a comment on its issue.
- Test an endpoint by calling it with requests built in memory
  (`HTTP::Server::Request.get/post`, `MyEndpoint.call(req)`); a test
  needs a socket only for what the server itself does.

## Before committing

```sh
quarz check
quarz test              # interpreted
quarz test --release    # compiled
quarz test --doc        # the doc comments' examples
quarz doc --check
```

Use a `quarz` built from Quarz's main (`cargo build --release` there).
The tests that need a database will use pg's `PG::Testing`:
`PG_TEST_URL` when it is set (CI's `postgres:16` service sets it), else a
throwaway cluster; the others need none.

- Work on a branch and open a pull request (below); stage files by explicit path.
- Review the branch's diff with a fresh subagent before the pull request
  leaves draft (commits on a draft are fine without it), and fix what it
  finds that needs no decision.
- Never print secrets (passwords, tokens).
- Do not put a `Claude-Session:` line in commit messages, nor a Claude
  session URL in pull request descriptions.

## Pull requests

- Open a draft pull request as soon as the branch has its first commit,
  while the local checks run: CI then runs in parallel with `quarz test`.
- Force-pushing after the subagent review or the local checks changed
  the commits is fine at any stage (amend or rebase as needed, onto
  main too): the merge names the head commit it was approved on.
- A rebase onto main that applied without conflicts needs no local rerun
  of the full suite: CI runs it on the new head. Rerun the affected tests
  locally when the rebase resolved a conflict or main changed a path the
  branch touches.
- Mark it ready for review only when the subagent review of the branch's
  diff and every local check above have passed; from then on
  review every change, added or rewritten, before pushing it.
- The description says what was verified.

## Issues

- Open work lives in GitHub issues of the repository that owns the code,
  one per defect or change. The title is a sentence in the commit-message
  style, saying what is wrong or what is wanted.
- The body has the headings **Symptom**, **Reproducer**, **Cause** (when
  known; say when it is a guess), **Direction** (`Decided (date): ...`,
  or the options with a recommendation) and **Found in** (the pull
  request or review that found it).
- Labels: `bug`, `enhancement`, `documentation`; `soundness` when a
  guarantee of the type or ownership system breaks (the highest
  priority); `performance`; `design` when a language or API decision is
  needed; `needs-decision` while it waits for the maintainer;
  `after-0.1.0`.
- A bug found while implementing or reviewing that is outside the pull
  request's scope is filed as an issue that says which pull request
  found it, not fixed in that pull request nor left in a comment.
- A pull request that fixes an issue says `Closes #N` in its description.
  A decision taken about an issue is recorded as a comment on it.
