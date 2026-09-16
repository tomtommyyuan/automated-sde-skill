# Plan template

File: `<plansDir>/YYYY-MM-DD-<slug>.md`. Under ~80 lines; a longer plan is two plans.

```markdown
# <Title — what will be true when this is done>

- Issue: #N | none
- Branch: <type>/<slug>
- Status: planning | in-progress | done
- Created: YYYY-MM-DD

## Goal
<one paragraph: the outcome, and how we will know we have it>

## Context
<what exists today, path:line for every claim; the constraint that shapes the approach; what was
tried before (git log, issue history) when relevant>

## Approach
<the design in a few sentences; the alternative considered and why not>

## Assumptions
- <decided without asking; correct here if wrong>

## Non-goals
- <what a reader might expect that this deliberately excludes, and where it goes instead>

## Steps
- [ ] 1. <concern> — verify: <command or observation>
- [ ] 2. …

## Risks
- <what could go wrong, how we would notice, what we would do>

## Open questions
- <needs the user; blocks step N>

## Log
- YYYY-MM-DD: plan written.
```

## What makes a step good

- Its name is the commit subject waiting to happen: "add scheduler state to checkpoint dict", not
  "checkpoint work".
- It ends in a checkable state — a test passes, a command prints X, a file exists with Y.
- It leaves the tree green. When it cannot (a two-part migration), the two parts are one step.
- Under ~200 changed lines. Bigger → split by layer (schema → model → API → UI) or by case (happy
  path → error paths).
- Enabling changes first, on their own: the rename, the extraction, the new helper.
- For a bug: step 1 the failing test; step 2 the fix; step 3, if any, the guard that stops the
  class of bug.

## Experiment variant

Replace Goal / Approach / Steps with:

```markdown
## Hypothesis
<one sentence: changing X will improve Y by about Z because W>

## Baseline
- command: <exact>
- commit: <sha>
- config: <path>
- metric: <name> = <value> on <split> (run id / log path)

## Change
<what differs from baseline, and only that>

## Success
<metric threshold; and what result would falsify the hypothesis>

## Budget
<GPU-hours / wall-clock / number of seeds; the stop rule>

## Steps
- [ ] 1. implement the change behind a config flag — verify: unit test or shape check
- [ ] 2. smoke run on a tiny subset, flag on and off — verify: both complete, losses finite
- [ ] 3. full run(s), N seeds — verify: run ids in the Log
- [ ] 4. analyze: metric with confidence interval versus baseline — verify: table in the Log
- [ ] 5. write up: keep / drop / iterate, and why — verify: Log entry, Status: done
```

The Log holds run ids and numbers as they arrive, so the plan doubles as the experiment record
the paper's method section will need.
