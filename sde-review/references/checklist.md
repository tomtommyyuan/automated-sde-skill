# Review checklist

Each line is a prompt to check, not a finding. A finding needs a concrete triggering input and a
confirmed code path.

## Correctness

- Boundaries: empty input, one element, the maximum, zero, negative, `None` / `null` /
  `undefined`, empty string versus missing key.
- Error paths: every `except` / `catch` — what is swallowed, what is half-written when it fires,
  what the caller sees. `except Exception: pass` is a finding. An error logged and then execution
  continuing as if it succeeded is a finding.
- Partial failure: a loop that writes N things and fails at k — are the first k−1 rolled back,
  retried, or left inconsistent? Is the operation safe to retry?
- Concurrency: shared mutable state; check-then-act without a lock; a blocking call inside
  `async def`; a global mutated by a worker; a cache with no invalidation.
- Resources: files, sockets, DB sessions, subprocesses, CUDA tensors held past their use — opened
  in a `with`? Closed on the error path?
- Off-by-one: inclusive versus exclusive ends, slicing, pagination, `len − 1`.
- Time and text: naive versus aware datetimes, local versus UTC, DST, assumed encoding, locale.
- Python: mutable default arguments; `is` versus `==`; late-binding closures in loops; a dict
  mutated while iterated; float equality; `__eq__` without `__hash__`; `sorted` stability assumed.
- JS/TS: `==`; an unawaited promise; `forEach` with async; `parseInt` without a radix; optional
  chaining hiding a real null; `any` crossing a boundary.
- Contract changes: a changed signature, return shape, exception type, config key or default —
  grep every caller and every config file. A silently changed default is a finding.
- The sentence from step 2: does the code do that, all of it, and nothing else?

## Security

- Injection: string-built SQL; `subprocess(..., shell=True)` with outside input; `eval` / `exec`;
  templates with autoescape off; `os.system`; an f-string into a query.
- Deserialization: `pickle.load`; `yaml.load` without `SafeLoader`; `torch.load` without
  `weights_only=True` on a file that can come from outside; `marshal`; `jsonpickle`.
- Files: a path built from input without normalization (`../`); predictable temp names;
  world-writable outputs; symlinks followed.
- Secrets: any literal token, key, password or connection string; secrets in logs or error
  messages; secrets in test fixtures.
- Auth: a new endpoint, command or handler — who may call it, is that checked, and before the
  work? Object-level authorization (can user A read B's row)?
- Network: outside-supplied URLs fetched server-side (SSRF); TLS verification off; no timeouts on
  outbound calls.
- Dependencies: a new package — the well-known one (typosquats exist), pinned, needed?
- Crypto: home-made hashing; `random` where `secrets` belongs.

## Tests

- Would the test fail without the change? Mentally revert the fix: still green means it tests
  nothing.
- Assertions present and specific: `assert result == expected`, not `assert result`.
- Behavior, not implementation: a test that mocks the function under test, or asserts internal
  call order for no reason, breaks on every refactor and catches no bug.
- Flakiness: wall-clock time, network, unseeded randomness, filesystem order, fixtures mutated
  across tests, sleeps.
- The new branch of logic — the `else`, the exception, the empty case — has a test, where the repo
  tests that kind of thing.
- Names say the behavior: `test_resume_restores_scheduler_step`, not `test_resume`.
- A removed or weakened test needs a stated reason.

## ML and data (when the diff touches models, data, metrics or training)

- Leakage: normalization stats, vocab, tokenizer or scaler fit before the split or on the full
  set; the test set used for early stopping, model selection or threshold tuning; duplicates
  across splits; a time-series split that shuffles.
- Determinism: seeds for `random`, `numpy`, `torch` and the CUDA / cuDNN flags when the PR claims
  reproducibility; DataLoader worker seeding; nondeterministic ops where the claim matters.
- Train/eval hygiene: `model.eval()` and `torch.no_grad()` / `inference_mode` around evaluation;
  augmentation applied at eval; dropout and BN state; `optimizer.zero_grad()` placement;
  gradient accumulation scaled; clipping before the step.
- Shapes and dtypes: silent broadcasting; `view` on a non-contiguous tensor; a batch dimension
  assumed; `argmax` on the wrong axis; loss reduction (`mean` versus `sum`) changed without an LR
  change; mixed-precision overflow; integer division.
- Metrics: aggregated over the whole split, not averaged per batch with unequal batch sizes;
  padding and `ignore_index` excluded; computed on the split the PR names; a claimed number
  matches a logged run (path, run id or link).
- Memory: a running stat that keeps the graph (`.item()` / `.detach()` missing); lists of tensors
  growing across epochs; CPU tensors created inside the hot loop.
- Checkpoints: model, optimizer, scheduler, scaler and RNG state saved together; loading with
  `map_location`; `strict` key handling; old checkpoints still load, or a migration exists.
- Config: a hyperparameter hardcoded that the config owns; a default changed silently; the run
  logs its commit hash and config.
- Numerics: `log(0)`; division by a count that can be zero; softmax without log-sum-exp; NaN
  propagating silently — is there a check that would notice?

## Conventions

- Matches its neighbors: naming, module layout, error-handling style, logging, how config is
  read. Code that looks foreign to the file is a should-fix even when it works.
- No drive-by changes: a whole-file reformat, an unrelated rename, a dependency bump — each is its
  own PR.
- Type hints where the repo has them; docstrings on public functions where the repo has them.
- No dead code, commented-out code, debug prints, leftover `breakpoint()`, `TODO` without an
  issue, or `# type: ignore` / `eslint-disable` without a reason.
- Changelog, docs, README and example configs updated where the repo keeps them current.
- A new dependency has its reason in the PR body.

## Performance (only what is obvious)

- N+1 queries; a query inside a loop; loading a whole table to filter in Python.
- O(n²) on something that grows: `in list` inside a loop; repeated string concatenation;
  `pd.concat` / `df.append` in a loop; `iterrows` on a large frame.
- Work that could be hoisted out of a hot loop; a regex compiled per call.
- Blocking IO on the request path or the training step.
- No micro-optimizations; no benchmarks demanded that the PR did not claim.

## API surface and docs

- A public function, CLI flag, config key, endpoint or exported type changed: backward
  compatible, deprecated with a path, or a break that is called out and versioned?
- Error messages tell the user what to do.
- The body's "How I verified it" matches what the diff could actually have verified.
