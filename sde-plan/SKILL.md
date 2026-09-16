---
name: sde-plan
description: Turn a multi-step task — a sentence, a brief, or a GitHub issue — into a grounded plan file in tasks/ whose checkbox steps each map to one commit, then execute or resume it step by step with tests and a commit per step. Use when work spans several steps or sessions, when asked to plan, scope, or break down a task, or to continue an existing plan; also plans ML experiments (hypothesis, baseline, metric, budget).
argument-hint: "<request | issue-url | tasks/file.md> [--execute] [--resume] [--issue] [--branch <name>]"
allowed-tools: Bash(git *), Bash(gh *), Read, Glob, Grep, Edit, Write, AskUserQuestion
---

# /sde-plan — plan it, then work the plan

The plan file is the unit of work management: what we are doing, why, in what order, and how far
we got. It lives in the repo, so it survives the session, and it is short enough that a reviewer
reads it in two minutes. Steps are the size of a commit.

Arguments: `$ARGUMENTS`

Mode: a plan file path, `--execute`, `--resume`, or an ask phrased as implement / continue /
resume / pick up → **execute**. Anything else → **plan**.

## Plan mode

### 1. Classify the ask

bug · feature · refactor · chore · question (answer it, no plan file) · experiment (success is a
measured number — hypothesis, baseline, metric; the template's experiment variant).

A GitHub issue URL → `gh issue view <n> --json title,body,labels,comments`; the number goes into
the plan and later into the PR.

### 2. Ground it in the code

Before writing a step, read what the step will touch: grep the symbols the ask names, read the
files that own them, read the last commits that changed them (`git log -n 10 -- <path>`), find the
tests that cover them. Every claim in Context cites `path:line`. A plan that says "update the
loader" without having read the loader is a guess.

### 3. Decide what needs the user

Ask (AskUserQuestion, one question, recommended option first) only for a choice that changes the
design and that the code does not settle — which API survives, migrate versus shim. Everything
else is an assumption written under Assumptions, where it can be corrected.

### 4. Write the plan

`<plansDir>/YYYY-MM-DD-<slug>.md` (`plansDir` from `.claude/sde.json`, default `tasks/`), from
`references/plan-template.md`. Steps are:

- one concern each — the size of one commit, one green test run, roughly under 200 lines;
- ordered so the tree is green after every step — enabling refactors first, in their own step;
- verifiable — each ends with `verify: <command or observation>`;
- for a bug, step 1 is a failing test that reproduces it;
- for an experiment: implement → smoke run on a tiny subset → full run → analyze → write up.

### 5. Register it

- `--issue`, or the ask says to create an issue → `gh issue create --title "<title>" --body-file
  <tmp>` with Goal, Context and Steps; record the number in the plan.
- Branch name in the plan: `<type>/<slug>`, or `--branch`.
- Commit the plan file on its own — `docs: plan <slug>` — unless the plans directory is
  gitignored, in which case it stays local and the report says so.

Report: the plan path, the steps one per line, the assumptions, the open questions.

## Execute mode

### 1. Find the plan

The path given, else the most recently modified `<plansDir>/*.md` with `Status: planning` or
`in-progress`. Read it whole, Log included. None → say so and offer plan mode.

### 2. Branch

The plan's `Branch:` — create it from base if it does not exist (`git switch -c`), switch to it if
it does. Set `Status: in-progress`; it goes into the first step's commit.

### 3. Work the steps

For the first unchecked step, then the next:

1. Implement exactly that step. Read before writing; match the neighbors.
2. Run the step's verify line, then the profile's `testFast` (`test` when there is no fast variant).
3. Commit — message per `~/.claude/skills/sde-pr/references/commit-conventions.md`, body naming
   the plan step. Tick the box in the plan file in the same commit.
4. Something the plan did not foresee → update the plan, not just the code: revise the step or add
   one, and append a dated Log line saying what changed and why. A plan that still matches the code
   is the point.
5. A step needs a decision the plan does not settle → ask, record the answer in the Log, continue.
   A step turns out wrong or impossible → stop, say what you found, propose the revision. Do not
   improvise past a wrong plan.

### 4. Finish

All boxes ticked → run the full gate (format, lint, typecheck, full tests), set `Status: done`, and
hand off: "Ready for `/sde-pr`" — or run it, when the ask was to ship. `/sde-pr` moves the file to
`<plansDir>/done/` when it opens the PR.

## Rules

- Steps map to commits. "1. implement 2. test 3. fix" is not a plan.
- The plan file is the state. Update it in the same commit as the work it describes.
- Read before planning; cite `path:line`. No step touches code you have not opened.
- Ask for design decisions; assume the rest and write the assumption down.
- Never tick a step whose verify line did not pass.
