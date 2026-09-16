# Repo profile — `.claude/sde.json`

One small file so the sde skills stop re-detecting the same things, and so you can override them.
Created on first run by whichever sde skill needs it; edited by hand after that; a hand edit
always wins. Re-detect with `--reprofile` on any sde skill.

## Schema

```json
{
  "base": "main",
  "commitStyle": "conventional",
  "mergeMethod": "squash",
  "plansDir": "tasks",
  "resolveThreads": true,
  "commands": {
    "format": "ruff format . && ruff check --fix .",
    "lint": "ruff check .",
    "typecheck": "mypy src",
    "test": "pytest -q",
    "testFast": "pytest -q -x -m 'not slow'"
  },
  "areas": {
    "frontend/": { "lint": "npm --prefix frontend run lint", "test": "npm --prefix frontend test" }
  },
  "generatedPaths": ["package-lock.json", "poetry.lock", "uv.lock", "*.snap", "dist/", "**/__snapshots__/**"]
}
```

Every key is optional. A command set to `null` means "this repo has none — stop looking".
`commitStyle` is `conventional` or `plain` (mimic the log). `areas` maps a path prefix to command
overrides, used when a diff touches only that prefix. `resolveThreads: false` makes `/sde-resolve`
leave thread resolution to the reviewer.

## Detection

Base branch: `git symbolic-ref --short refs/remotes/origin/HEAD` minus `origin/`; failing that,
`gh repo view --json defaultBranchRef -q .defaultBranchRef.name`; failing that, `main` if it
exists, else `master`.

Commit style: `git log --format=%s -n 40`; ≥ 50 % conventional prefixes → `conventional`, else
`plain`; empty history → `conventional`.

Merge method: `gh repo view --json squashMergeAllowed,rebaseMergeAllowed,mergeCommitAllowed`;
prefer squash → rebase → merge among those allowed.

Commands — first match per slot wins, and a `Makefile` / `justfile` target with the obvious name
(`format`, `lint`, `typecheck`, `test`) beats everything below:

Python
- `pyproject.toml` with `[tool.ruff]` → format `ruff format . && ruff check --fix .`, lint
  `ruff check .`; with `[tool.black]` → format `black .` (plus `isort .` when configured).
- `[tool.mypy]` / `mypy.ini` → typecheck `mypy <package or src dir>`; `pyrightconfig.json` →
  `pyright`.
- `pytest` among the dependencies, or `tests/`, or `[tool.pytest.ini_options]` → test `pytest -q`;
  `testFast` = `pytest -q -x`, plus `-m "not slow"` when a `slow` marker is registered.
- Runner prefix from the lockfile: `uv.lock` → `uv run`, `poetry.lock` → `poetry run`,
  `Pipfile.lock` → `pipenv run`, else none.
- `.pre-commit-config.yaml` with nothing more specific → format `pre-commit run --all-files`.

JavaScript / TypeScript
- `package.json` scripts `format`, `lint`, `typecheck` / `type-check`, `test` — used verbatim via
  the package manager the lockfile implies (`pnpm-lock.yaml` → pnpm, `yarn.lock` → yarn,
  `bun.lock*` → bun, else npm).
- No typecheck script but `tsconfig.json` → `npx tsc --noEmit`.

Rust: `cargo fmt`, `cargo clippy --all-targets -- -D warnings`, `cargo test`.
Go: `gofmt -l . && go vet ./...`, `go test ./...`.

Anything else: the CI workflow (`.github/workflows/*.yml`) is the best hint — the `run:` lines
under jobs named test / lint / check are what the repo treats as the gate. Copy them.

Nothing found for a slot → `null`, said once in the report. Do not write config files into the
repo to fill a slot; suggesting one is fine.

Generated paths: lockfiles, `dist/`, `build/`, `*.min.js`, `*.snap`, `__snapshots__/`, `*.pb.go`,
`*_pb2.py`, auto-generated `migrations/`, anything `.gitattributes` marks `linguist-generated`.

## Writing it

`mkdir -p .claude` and write `.claude/sde.json` with only the keys you detected. Tell the user it
exists and that edits to it win. Whether it is committed follows the repo's `.gitignore`; do not
stage it unless `.claude/` is already tracked or the user asks.
