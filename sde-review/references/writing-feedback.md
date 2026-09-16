# Writing review feedback

## Severity

- **blocker** — merging ships a defect: wrong behavior, data loss or corruption, a security hole,
  a broken build or test, a violation of a rule the repo states as hard. Fixed before merge.
- **should-fix** — a real defect or a maintainability cost with a concrete consequence: the next
  person will misuse it, the next change will break it, the failure will be undiagnosable. Merge
  waits for the fix or for an explicit reason not to.
- **nit** — polish or preference. The author may ignore it without replying. Always prefixed
  `nit:`. Batched: five nits are one comment.

Severity is a claim about consequence, not about confidence. Unsure → verify first. Cannot
verify → not a finding (security excepted, marked *unverified*).

## The shape of a finding

```
**[blocker]** `src/train.py:142` — scheduler state is not restored on resume.
`load_checkpoint` restores model and optimizer but `scheduler` is rebuilt at step 0, so a run
resumed at step 10k repeats warmup (any checkpoint with `step > warmup_steps`).
Fix: save `scheduler.state_dict()` in `save_checkpoint`; call `scheduler.load_state_dict` after
the optimizer is restored.
```

Four parts in this order: severity and location · one-line claim · why, with the concrete
triggering input or state · the fix, concrete enough to apply. A code suggestion when the fix is
under ~5 lines. No preamble.

## Tone

- About the code, in the indicative: "this drops the exception", not "I feel like this might maybe
  drop the exception?". Hedging on a verified finding wastes the reader's time; on an unverified
  one, verify instead of hedging.
- A real question when you have one: "Is `batch_size` guaranteed non-zero here? I could not find
  the check." Never a question that is an assertion in disguise.
- No praise padding, no "great work overall", no emoji. One true sentence in the summary about
  what is good is fine when it helps the author keep it.
- Never argue style the formatter owns. Never restate a bot; agree or disagree with it in one
  line if it matters.
- Disagreeing with the approach is a finding like any other: the cost of the chosen path and the
  alternative, with the rigor you would demand of a bug report.

## The summary

1. What the change does, in your own words, two sentences at most. If that differs from the PR
   body, the gap is your first finding.
2. The verdict — **Ready** / **Ready after the fixes above** / **Not ready** — and one sentence why.
3. Counts by severity, and the single risk you would watch after merge.

## Posting

Line-anchored findings go inline (path, line, `RIGHT` side of the diff). The summary is the
review body; findings about the change as a whole — structure, missing tests, scope — go in the
body under the summary. One review per pass, never a trickle of single comments.
