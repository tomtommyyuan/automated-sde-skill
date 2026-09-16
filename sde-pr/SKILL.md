---
name: sde-pr
description: Commit, push, and open or update a pull request in one automated pass — atomic conventional commits cut from the working tree, the repo's formatter / linter / type-checker / tests run first with autofixes committed, a reviewer-ready description written from the actual diff and the repo's PR template, the issue linked, optional auto-merge. Use when asked to commit, push, open / update / ship a PR, or write a PR title or description.
argument-hint: "[title] [--base <branch>] [--draft] [--merge] [--reviewer <login>] [--commit-only] [--dry-run] [--reprofile]"
allowed-tools: Bash(git *), Bash(gh *), Bash(gitleaks *), Read, Glob, Grep, Edit, Write, AskUserQuestion
---

# /sde-pr — commit, push, open or update the pull request

Takes a branch from "the change is done" to "a reviewer has something worth reading" without
stopping on the way: it commits, it pushes, it opens the PR. It asks for exactly one kind of
decision — whether to split a branch that is really two PRs — and decides everything else,
then reports what it did.

The ask sets the stopping point. "Commit" stops after step 2. "Push" stops after step 5.
"PR" / "ship" / "open a pull request" runs everything. `--commit-only` is the first case as a flag.

Arguments: `$ARGUMENTS`

## Preflight

Gathered when the skill loaded (if these lines are blank, run the commands yourself):

Branch: !`git branch --show-current 2>/dev/null || echo "(detached)"`
Working tree: !`git status --short 2>/dev/null | head -40`
Existing PR: !`gh pr view --json number,state,url,isDraft,baseRefName 2>/dev/null || echo "none"`

## Step 0 — Repo profile

Read `.claude/sde.json`. If it is missing (or `--reprofile`), detect and write it following
`references/repo-profile.md`. It gives you the base branch, the format / lint / typecheck / test
commands, the commit-message style the repo actually uses, generated paths to exclude from size
counts, and the merge method. A value the user edited by hand wins over detection.

## Step 1 — Be on a branch a PR can come from

- On the base branch with changes → `git switch -c <type>/<slug>` (`feat/`, `fix/`, `refactor/`,
  `docs/`, `chore/`; slug says what the change does, ≤ 5 words, kebab-case). Uncommitted work
  follows the switch: nothing is lost and nothing lands on base.
- On the base branch, tree clean, nothing unpushed → there is nothing to ship; say so and stop.
- Detached HEAD → `git switch -c <type>/<slug>` at HEAD.
- Otherwise stay where you are.

## Step 2 — Commit the working tree

Skip if clean. Otherwise cut it into atomic commits per `references/commit-conventions.md`:

1. `git status --short` and `git diff` — read every change; group by concern, not by file type.
2. Secrets scan before staging anything: the patterns in the reference, plus
   `gitleaks git --pre-commit --staged` after staging when gitleaks is installed. A hit is never
   committed — leave that file unstaged and report the path and pattern.
3. Stage by path — `git add <paths>` — one concern at a time. Never `git add -A` / `git add .`;
   the working tree may hold things that are not yours to commit.
4. Message in the repo's own style (the profile says conventional or plain). Subject says what,
   body says why. Append the attribution trailer this environment requires — the harness states
   it in the session context; do not invent one.
5. Pre-commit hooks run on their own. A hook that rewrites files → re-stage, commit again. A hook
   that fails → fix the cause. Never `--no-verify`.

Stop here if the ask was only to commit. Report the commits.

## Step 3 — Quality gate

Run, in order, what the profile knows: format (autofix) → lint (autofix) → typecheck → tests.
When the profile maps areas to commands, run only the areas the diff touches; otherwise run all.

- Autofix changed files → commit as `style: apply <tool>`, or fold into the last commit with
  `git commit --amend --no-edit` when that commit has not been pushed yet.
- A failure the diff caused that you can fix with confidence → fix, commit, rerun. Two rounds at
  most.
- Still red → keep going, but the PR opens as a **draft** and its body gets a `## Status` section
  quoting the failing check verbatim. Red work on a draft is honest; a red PR asking for review
  is not.
- No tests in the repo → say so once in the report. Do not write a test suite as a side quest.

## Step 4 — One PR, one concern

Read `git log --oneline <base>..HEAD` and `git diff --stat <base>...HEAD` (three dots: from the
merge-base, so commits that landed on base since the branch point do not appear as changes).

The branch is two PRs when a reviewer could approve one part while rejecting the other: a
feature plus an unrelated fix; a refactor of module A plus new behavior in module B; a
formatting sweep with three lines of logic buried in it. Signals: commits whose files never
overlap and whose scopes differ; `fix:` beside `feat:` in different areas; a file rewritten for
style in a commit that also changes behavior.

Found one → ask with AskUserQuestion, one question, **Split** first and marked recommended, each
option describing the resulting PRs in a line. On Split: branch again from base, `git cherry-pick`
that concern's commits onto it, push both, open both PRs, and note the dependency in the second
body when there is one. On Keep: proceed and say in the body that the PR carries two concerns
and why.

Size: `git diff --numstat <base>...HEAD` minus lockfiles, the profile's generated paths, vendored
code, snapshots and pure renames. Over ~400 lines → say so in the body and suggest a reading
order. Over ~1000 → fold a split into the question above even when the concern is single.

## Step 5 — Push

`git push -u origin HEAD`. From here the branch's history is append-only (Rules).

Stop here if the ask was to push, not to open a PR.

## Step 6 — Write the pull request

Per `references/pr-body.md`: title from the change, not the branch name; body from the diff and
the commits, in reviewer order — why → what → how it was verified → risk → out of scope.
Respect `.github/PULL_REQUEST_TEMPLATE.md` (or `pull_request_template.md`, or a
`PULL_REQUEST_TEMPLATE/` directory): fill its sections, keep its headings.

Link the issue. Sources, in order: a plan file (below), the branch name (`123-`, `issue-123`,
`/123-`), the commit messages, then `gh issue list --search "<keywords>" --state open`.
`Closes #N` when the PR finishes the issue, `Refs #N` when it only advances it.

Plan file: if `<plansDir>/*.md` (default `tasks/`) holds a plan whose `Branch:` names this
branch — or whose slug matches — use its Goal and Steps for the body, set `Status: done`, move it
to `<plansDir>/done/`, and commit the move (`docs: close plan <slug>`) before pushing again.

## Step 7 — Open or update

- No PR → `gh pr create --base <base> --title "<title>" --body-file <tmp> [--draft]
  [--reviewer <logins>]`. Draft if asked, or if the gate is red.
- An open PR → `gh pr edit <n> --title "<title>" --body-file <tmp>`. Regenerate only the sections
  you own (Why / What / How I verified it / Risk / Out of scope / Status); every other line a
  human or bot added stays byte-for-byte.
- A merged or closed PR for this branch → do not reopen it; say the branch is stale and stop.
- `--dry-run` → print the commits you would make, the title and the body. Change nothing, push
  nothing.

## Step 8 — After opening

- `gh pr checks <n>` — one snapshot for the report. Do not wait for CI unless asked.
- `--merge` → `gh pr merge <n> --auto --<method>` with the profile's merge method (squash when the
  repo allows it). GitHub merges when checks pass; nothing merges red.
- Report, one screen: PR URL · commits one per line · gate results · checks snapshot · anything
  skipped and why.

## Rules

- Commit and push without asking. That is the job. The only question is the split in step 4.
- A pushed branch is append-only. Rewrite it (`--force-with-lease`, never `--force`) only when the
  user asks for a squash or rebase, and never on a branch that is not yours.
- Never `--no-verify`. A failing hook is a finding, not an obstacle.
- Never push to the base branch. A change lands through a pull request.
- Never merge without `--merge` or the user's word; `--auto` is the sanctioned merge because it
  waits for green. Never approve a PR.
- Never commit a secret, a `.env`, or a file over 5 MB (ask — the answer is usually LFS or "no").
- Never `git add -A`. Never delete a human's text from a PR body.
- The attribution trailer is whatever the harness requires this session. No invented co-authors.
