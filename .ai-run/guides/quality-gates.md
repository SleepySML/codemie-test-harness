# Quality Gates

Run every command from the repository root. The "Ran" column records what was executed on 2026-11-05 in a scratch clone of this branch, with Poetry 2.5.1 and Python 3.12 (`poetry run`) or Python 3.15 (`python3`). "not run" means nobody ran the command for this guide.

## Commands

| Command | Ran | Pass means |
|---|---|---|
| `poetry install --no-interaction` | exit 0 | dependencies installed from the tracked `poetry.lock` |
| `poetry run ruff check .` | exit 0 | `All checks passed!` (CI gate) |
| `poetry run ruff format --check .` | exit 0, 914 files already formatted | no file would change (not a CI gate; the pre-commit hook runs it) |
| `poetry build` | exit 0 | wheel and sdist in `dist/` |
| `poetry run codemie-test-harness --help` | exit 0 | the CLI starts and lists its commands |
| `make ruff-fix` | exit 0 | `ruff check --fix` (rewrites files) |
| `make ruff-format` | exit 0 | `ruff format` (rewrites files) |
| `make ruff` | exit 0 | fix, check, then format (rewrites files) |
| `make license-check` | exit 1 on the scratch clone: 12 tracked `.py` files lack the header | exit 0 when every tracked `.py` file has the header (`scripts/license_template.txt`) |
| `make license-fix` | exit 0 | inserts the missing headers (rewrites files) |
| `make license-scan` | exit 1 on the scratch clone: `cffi` 2.1.1 reports `MIT-0`, which is not in `allow-only` | exit 0 when every dependency licence is in `[tool.pip-licenses]` of `pyproject.toml` |
| `make scan-internal` | exit 2 (no scan config set) | exit 0 only when `HARNESS_SCAN_PATTERNS` and/or `HARNESS_SCAN_CUSTOMER_NAMES` is set and nothing matches; exit 1 on a hit |
| `make install-hooks FORCE=1` | exit 0 | the pre-commit hook is installed (see below) |
| `make test` | exit 0 | nothing ran: `Nothing to be done for test` (the target has no recipe) |
| `make install`, `make publish` | not run | `make install` is `poetry install` plus an optional `SDK_VERSION` pin; `make publish` uploads to PyPI, never run it |

The two failing rows (`license-check`, `license-scan`) are the state of `main` at the time of the run. They are not gates in CI. Do not report them as caused by your change without comparing against `main`.

Python scripts run as `python3` (`Makefile`, targets `scan-internal`, `license-check`, `license-fix`). Where only `python` exists, run the scripts with that.

## Tests

CI does not run the test suites: they need a running CodeMie installation (`CONTRIBUTING.md`, intro). Commands to run one test or a suite are in [testing/strategy.md](testing/strategy.md), "Run one test" and "Run suite". Markers, traps and environments are in [testing/strategy.md](testing/strategy.md) and [testing/environments.md](testing/environments.md). Test commands were not run for this guide.

## Pre-commit hook

`make install-hooks` copies `scripts/pre-commit-hook.sh` to the hook path of the clone. It does not link the file, so run `make install-hooks FORCE=1` again after the script changes. Without `FORCE=1` the target refuses to replace a different existing hook; with it, the old hook is saved as `pre-commit.bak`.

The hook runs on every commit:

1. The internal-value scan, when `HARNESS_SCAN_PATTERNS` or `HARNESS_SCAN_CUSTOMER_NAMES` is set. Otherwise it prints a notice and goes on.
2. For staged `.py` files only: `ruff check`, `ruff format --check` and the licence-header check, all on the staged content, not the working tree.

Checked in a scratch clone on 2026-11-05: a ruff error blocks the commit, a missing header blocks it, a formatted file with a header passes, and a commit of only a non-`.py` file passes with the scan notice.

## CI jobs

| Job name | Definition | Fail means |
|---|---|---|
| `Build` | `.github/workflows/ci.yml`, job `build`, lines 17-43 | install, build or the CLI smoke test failed |
| `Lint` | `.github/workflows/ci.yml`, job `lint`, lines 45-66 | `ruff check .` reported a finding |
| `Internal references` | `.github/workflows/ci.yml`, job `internal-refs`, line 68 | a private host, ticket key or personal e-mail is in the tree |
| `Security` | `.github/workflows/ci.yml`, job `secrets-detection`, line 74; rules in `.gitleaks.toml` | a secret is in the change |
| `PR Title` | `.github/workflows/pr-title-lint.yml`, job `pr-title`, line 17 | the PR title is not Conventional Commits |
| `CodeQL (<language>)` | `.github/workflows/codeql.yml`, job `codeql`, line 22 | CodeQL reported a finding |

`Internal references`, `Security`, `PR Title` and `CodeQL` call reusable workflows from `codemie-ai/ci-workflows`; their rules are not in this repository. The local `make scan-internal` is a separate check and does not replace the CI job.

## Gaps

| Gate | State | Evidence |
|---|---|---|
| Tests | CI runs none | `.github/workflows/ci.yml` comment above step "Install dependencies" |
| Format check | CI runs `ruff check .` only, not `ruff format --check`; only the hook runs it | `.github/workflows/ci.yml`, step "Lint" |
| Lock file | `poetry.lock` is tracked and `poetry install` uses it. The comment in `ci.yml` and `CONTRIBUTING.md`, "Local Checks", say the project has no lock file. The text is stale. | `git ls-files poetry.lock` |
| `.PHONY` | the targets `ruff`, `ruff-format` and `ruff-fix` are not in `.PHONY` in the `Makefile` | `Makefile`, first line |
| Rewriting targets | `make ruff`, `make ruff-fix`, `make ruff-format` and `make license-fix` change files; they are not checks | `Makefile` |
| Type check | none configured | `pyproject.toml` |

## SDK changes

When a test needs an SDK change, develop it in the public SDK repository and follow its `CONTRIBUTING.md`. Its own checks are listed in the `sdk` entry of [testing/config.json](testing/config.json), key `verify`; run them from the root of the SDK clone, not from this repository. Do not copy them here.
