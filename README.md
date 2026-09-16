# automated-sde-skill

Four [Claude Code](https://claude.com/claude-code) skills that cover the everyday software-engineering loop — plan → implement → self-review → pull request → act on the review — and run it **end to end without stopping to ask**. They commit, push, open the PR, reply on review threads and re-request review on their own. What they refuse is only the footgun list a senior engineer refuses too.

| Skill | What it does | How far it goes on its own |
|---|---|---|
| `sde-pr` | Cuts the working tree into atomic conventional commits, runs the repo's format / lint / typecheck / tests (autofixes committed), pushes, writes a reviewer-ready description from the actual diff and the repo's PR template, links the issue | All the way; `--merge` enables auto-merge so GitHub merges when checks are green |
| `sde-review` | Reviews the current branch or someone else's PR: verified, severity-rated findings (blocker / should-fix / nit) with `file:line` and a concrete fix, across correctness, security, tests, ML & data pitfalls, and repo conventions | Self-review fixes blockers and should-fixes and commits each; PR review posts one review with inline comments |
| `sde-resolve` | Fetches every unresolved review thread and failing check, gives each a verdict (agree / disagree with evidence / ask), fixes, pushes, replies in-thread, resolves what is done, re-requests review | All the way; disagreements are stated plainly with evidence, never silently ignored |
| `sde-plan` | Turns a request or GitHub issue into a grounded plan file in `tasks/` whose checkbox steps each map to one commit; executes or resumes it step by step with tests and a commit per step. Has an experiment variant (hypothesis, baseline, metric, budget) for ML work | Asks only for design decisions the code does not settle; everything else becomes a written assumption |

Combined description cost in the system prompt: about 420 tokens for all four.

## Install

```bash
git clone https://github.com/tomtommyyuan/automated-sde-skill.git
cp -R automated-sde-skill/sde-* ~/.claude/skills/
```

Then `/sde-pr`, `/sde-review`, `/sde-resolve`, `/sde-plan` — or just ask ("commit and open a PR", "address the review comments") and the matching skill is picked up.

Requirements: `git`, the [GitHub CLI](https://cli.github.com/) authenticated (`gh auth status`). `gitleaks` is used for the staged-secrets scan when installed.

## The stance

Automated by default:

- commit and push without asking — that is the job
- open or update the PR, reply on threads, resolve them, re-request review
- fix your own review findings and commit each fix
- `--merge` turns on auto-merge; nothing merges red

Never, even when automating:

- force-push a pushed branch (a squash or rebase the user asks for uses `--force-with-lease`)
- `--no-verify` — a failing hook is a finding
- push to the base branch — a change lands through a PR
- approve a PR, or resolve a thread that was not actually fixed
- commit a secret, a `.env`, or `git add -A`

The one question the skills ask on their own initiative: whether to split a branch that is really two pull requests, because splitting rewrites the branch.

## Repo profile

On first run a skill writes `.claude/sde.json` in the repository with what it detected — base branch, format / lint / typecheck / test commands, commit style, merge method, generated paths. Edit it; hand edits win over detection. `--reprofile` re-detects.

## Layout

```
sde-pr/        SKILL.md + references/ (commit-conventions, pr-body, repo-profile)
sde-review/    SKILL.md + references/ (checklist, writing-feedback)
sde-resolve/   SKILL.md
sde-plan/      SKILL.md + references/ (plan-template)
```

`SKILL.md` is what loads when a skill is invoked; `references/` files load only when a step points at them. The commit conventions and repo-profile rules in `sde-pr/references/` are shared by the other skills via their `~/.claude/skills/...` paths, which is why the install target matters.

## License

MIT.
