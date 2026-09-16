---
name: sde-review
description: Review a diff or a pull request the way a senior engineer would — verified, severity-rated findings (blocker / should-fix / nit), each anchored to file:line with a concrete fix, across correctness, security, tests, ML and data pitfalls, and the repo's own conventions. Use to self-review the current branch before opening a PR (fixes are applied and committed), or to review someone else's PR by number or URL and post the review on GitHub.
argument-hint: "[<pr-number|pr-url>] [--post] [--request-changes|--approve] [--report-only] [--deep]"
allowed-tools: Bash(git *), Bash(gh *), Read, Glob, Grep, Edit, Write
---

# /sde-review — review like the senior engineer on the team

Two modes, chosen by the argument:

- **Self-review** (no argument): the current branch against its base, uncommitted changes included.
  Findings are reported, then blockers and should-fixes are fixed and committed — reviewing your
  own branch exists to make it ready. `--report-only` skips the fixing.
- **PR review** (a number or URL): someone else's pull request, checked out in a worktree so your
  own tree stays untouched. Findings are shown in chat; they are posted to GitHub when the user
  asked for that (review / comment on / post to the PR) or with `--post`.

Arguments: `$ARGUMENTS`

## Step 1 — Get the change in front of you

Self: base from `.claude/sde.json` (detect per `~/.claude/skills/sde-pr/references/repo-profile.md`
when missing). `git diff <base>...HEAD`, plus `git diff` and `git diff --cached` for the tree.

PR: `gh pr view <n> --json number,title,body,author,headRefName,baseRefName,files,reviews,comments,reviewDecision`.
Then `git fetch origin pull/<n>/head:pr-<n>` and `git worktree add ../<repo>-pr-<n> pr-<n>`; work
inside that worktree; `git worktree remove` it at the end. Diff: `git diff <base>...pr-<n>`.

## Step 2 — Understand what it claims to do

Read the PR body, the commit messages, the linked issue (`gh issue view <n>`), and any plan file
under `tasks/` that names the branch. Write one sentence: what this change is supposed to do. The
review checks the code against that sentence; drift between the two is a finding in its own right.

## Step 3 — Read enough to know

Reading the hunk is not reading the change. For every changed function or class, read the whole
thing; for every changed public symbol, grep its callers and read the call sites; for every
changed test, read the code it tests. Up to ~15 files read whole; beyond that, rank by risk
(auth, money, data writes, concurrency, migrations, anything inside a training or evaluation loop)
and read the rest by hunk.

`--deep` adds: run the tests locally, trace one end-to-end path through the change by hand, and
follow every error path to what actually happens when it is taken.

## Step 4 — The passes

Work through `references/checklist.md` in order: correctness → security → tests → ML and data
(when the diff touches models, data, metrics or training) → conventions → performance (only what
is obvious) → API surface and docs. Every candidate finding is verified before it is written:
name the concrete input or state that triggers it and confirm the code path takes it; if a cheap
test would settle it, run the test. What you cannot confirm, drop — except a security concern,
which is reported marked *unverified* with what would confirm it.

Skip what a formatter or linter would say; the gate owns that. Skip preference.

## Step 5 — Deduplicate

PR mode: read the existing review comments and bot comments
(`gh api repos/{o}/{r}/pulls/{n}/comments`). Do not repeat a point already made; if you agree with
an unresolved one, one line in the summary says so.

## Step 6 — Write it

Format, severity and tone per `references/writing-feedback.md`. Order: blockers, should-fixes,
nits. Then the summary: what the change does in your words, the verdict, the top risk.

Verdict is one of **Ready** · **Ready after the fixes above** · **Not ready**, with one sentence why.

## Step 7 — Act

Self-review, unless `--report-only`: fix every blocker and should-fix now, one commit per concern
(`fix:` / `refactor:` / `test:` per `~/.claude/skills/sde-pr/references/commit-conventions.md`),
rerun the affected tests, and add to the report which finding was fixed in which commit. Nits: fix
the one-line ones; leave the rest listed. Push if the branch was already pushed.

PR review, when posting: one review carrying an inline comment per line-anchored finding and the
summary as its body:

```bash
gh api repos/{owner}/{repo}/pulls/{n}/reviews --input review.json
# review.json:
# {"event":"COMMENT","body":"<summary>",
#  "comments":[{"path":"src/x.py","line":42,"side":"RIGHT","body":"<finding>"}]}
```

Event: `COMMENT` by default. `REQUEST_CHANGES` only with `--request-changes` and at least one
blocker. `APPROVE` only with `--approve` and zero blockers and zero should-fixes — otherwise refuse
and name the finding that stops it. Never approve your own change.

When the `code-review` skill is available in this session, running it too on anything over ~200
lines is a cheap second opinion on bugs; merge its confirmed findings into yours, deduplicated.

## Rules

- Verified findings only. A review that cries wolf is ignored next time.
- Severity means something: a blocker stops the merge. Do not inflate a nit to look thorough; do
  not soften a blocker to be polite.
- The code, not the author. Direct sentences. No questions that are really assertions.
- Never approve without `--approve`; never approve over an open blocker.
- Never edit or delete anyone else's comment; never resolve threads from a review.
- In self-review, fixing is the default, and every fix is its own commit the user can inspect or
  revert.
