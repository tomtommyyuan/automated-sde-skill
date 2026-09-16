# Pull request title and body

Written for the reviewer who has ten minutes, and for the engineer running `git blame` in a
year. Both need the why first.

## Title

- What the change does, imperative, ≤ 70 characters, no trailing period.
- If the repo's merged PR titles use conventional prefixes (`gh pr list --state merged --limit 20`),
  use one; otherwise a plain sentence. `feat(train): resume LR schedule from checkpoint` or
  `Resume LR schedule from checkpoint`.
- Not the branch name. Not the issue title pasted, unless it already names the change.

## Body

Use the repo's template when there is one — `.github/PULL_REQUEST_TEMPLATE.md`,
`.github/pull_request_template.md`, or `.github/PULL_REQUEST_TEMPLATE/*.md` (several: pick by
change type, else the default). Fill every section; keep its headings and order; drop a section
only when it is clearly N/A and the template allows that. Without a template, this shape:

```
## Why
<the problem or need, one paragraph; the issue: Closes #N / Refs #N>

## What
<the change at the level of decisions: the approach, the alternative rejected and why, the
invariant or contract that changed. Not a file list.>

## How I verified it
<the exact commands run and what they showed; manual steps a reviewer can repeat; for UI,
before/after screenshots; for a model or data change, the run id or log path and the metric
before and after>

## Risk
<what breaks if this is wrong, who is affected, migration or flag, how to roll back.
"Low — internal helper, covered by tests" is a complete answer when true.>

## Out of scope
<what a reviewer might expect here and will not find, and where it is going instead>
```

When the quality gate was red, then:

```
## Status
Draft — <check name> failing:
<the failing output, verbatim, trimmed to the relevant lines>
```

Then the attribution line this environment requires for PR descriptions, last.

## Writing rules

- Lead with why. A reviewer who understands the problem reads the diff in half the time.
- Describe decisions, not the diff. "Moved scheduler state into the checkpoint dict rather than a
  sidecar file so old checkpoints keep loading" is content; "modified `save_checkpoint` in
  `train.py`" is not.
- Verification is what you did, not what one could do: commands and their outcome. If you did not
  run something, say so — the reviewer plans their own checks around that.
- Risk names the failure mode. "None" is almost never true; "low" with a reason is.
- Say what is deliberately not here. It pre-empts "why didn't you also…".
- Length follows the change: a one-line fix earns four lines; a new subsystem earns a page. Over
  ~300 words, put a two-line summary at the top.
- Plain sentences. No "This PR aims to…", no "Please review", no emoji, no restating the title.
- Over ~400 non-generated lines: one line saying so, and a reading order or a proposed split.

## Updating an existing PR

Regenerate only Why / What / How I verified it / Risk / Out of scope / Status. Everything else
in the body — a human's notes, a reviewer's ticked checklist, a bot's block — stays byte-for-byte.
`gh pr view <n> --json body -q .body` first; rebuild the owned sections; write the whole body back
with `gh pr edit <n> --body-file <tmp>`.
