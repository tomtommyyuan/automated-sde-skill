# Commit conventions

## Which style — read the repo before writing

`git log --format=%s -n 40`. If half or more match
`^(feat|fix|docs|style|refactor|perf|test|build|ci|chore|revert)(\(.+\))?!?: `, the repo uses
Conventional Commits — use them. Otherwise mimic what is there (capitalized or not, a ticket
prefix like `ABC-123:`, a bare sentence) and record it in `.claude/sde.json` as
`"commitStyle": "plain"`. A repo with no history yet gets Conventional Commits.

## Conventional Commits, as used here

`<type>(<scope>): <subject>`

- type — `feat` (user-visible behavior), `fix` (a defect), `refactor` (no behavior change),
  `perf`, `test`, `docs`, `style` (formatting only), `build` (deps, packaging), `ci`, `chore`
  (nothing above; use sparingly).
- scope — the area, optional, the noun a teammate would use: `(dataloader)`, `(api)`, `(train)`.
- `!` after the type/scope for a breaking change, plus a `BREAKING CHANGE:` footer saying what
  breaks and what to do about it.
- subject — imperative, present tense ("add", not "added" / "adds"); ≤ 72 characters, 50 is
  better; no trailing period; lowercase first word unless the repo capitalizes.

Body (blank line after the subject; wrap at 72):
- why the change exists — the problem, the constraint, the thing that was wrong;
- what a reader needs that the diff does not show — the alternative rejected, the invariant now
  relied on, the measurement that justified it;
- not a list of files or functions. The diff already says that.

Footers, in this order: `Closes #N` / `Refs #N`; `BREAKING CHANGE:`; the attribution trailer
this environment requires, last.

Good:

```
fix(train): stop LR schedule restarting on resume

Resume restored model and optimizer state but rebuilt the scheduler from
step 0, so every resumed run got a second warmup. Save scheduler state
with the checkpoint and restore it before the first step.

Refs #42
```

Bad, and why:
- `fix: bug` — says nothing.
- `Update train.py` — names the file, not the change.
- `feat(api): add endpoint, fix README typo, bump deps` — three commits.
- `WIP` — never pushed. If it must exist locally, squash it before the push.

## Atomic commits — one concern each

A commit is atomic when reverting it removes exactly one idea and leaves the tree green.

Cutting the working tree:
1. List the concerns first, then assign files to them. A refactor that enables a feature is its
   own commit, before the feature. A rename is its own commit. Tests go with the code they test —
   do not split tests from implementation unless the repo does.
2. Order so every commit builds and passes: enabling change → main change → follow-ups.
3. One file holding two concerns: stage it with the concern it mostly serves and say in the body
   that it also carries the other. Do not fight for hunk-level purity at the cost of an hour; do
   not use it as a reason to merge unrelated concerns either.
4. Fixups after review are new commits on a pushed branch — never amend a pushed commit.

## Never commit

- Secrets. Before every `git add`, scan the diff for `-----BEGIN (RSA|EC|OPENSSH|PGP) PRIVATE KEY`,
  `AKIA[0-9A-Z]{16}`, `ghp_`, `github_pat_`, `sk-[A-Za-z0-9_-]{20,}`, `xox[bpsa]-`,
  `AIza[0-9A-Za-z_-]{35}`, `(password|passwd|secret|token|api_key)\s*[:=]\s*['"][^'"]{6,}`, and
  the files `.env*`, `*.pem`, `*.key`, `*.p12`, `credentials.json`, `id_rsa*`. After staging, when
  installed: `gitleaks git --pre-commit --staged`. On a hit: do not stage the file, report the path
  and the pattern; if the secret is already in a commit on this branch, say so plainly — rotating
  it is the fix, deleting the line is not.
- Generated output the repo does not track: build directories, `__pycache__`, `.pytest_cache`,
  `node_modules`, notebook outputs when the repo strips them, checkpoints, datasets, logs, `wandb/`.
- Files over 5 MB. Ask; the answer is usually LFS or "not in git".
- IDE and OS files: `.idea/`, `.vscode/` unless tracked, `.DS_Store`.
- Debug leftovers: `print(` / `console.log` added this session, `breakpoint()`, `import pdb`,
  commented-out code, `TODO` without an issue number.

## Hooks

Pre-commit hooks are part of the repo's contract. `.pre-commit-config.yaml` present but no
`.git/hooks/pre-commit` → run `pre-commit install` once and say so. A hook that modifies files →
re-stage, commit again. A hook that fails → fix the cause. `--no-verify` is never the answer.
