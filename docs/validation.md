# Validation evidence: vnpy

Run date: 2026-10-12. Branch: `feature/ayu_develop`. Interpreter: `3.14.8 (main, Oct  9 2026, 15:11:09) [Clang 22.1.3 ]`.

| Check | Result |
|---|---|
| lock | passed |
| sync | passed |
| compile | passed |
| build | passed |
| tests | passed |

## Current result and compatibility decisions

- The complete162-case upstream suite passes. Tests use an absolute directory because conftest changes the working directory before testpath resolution.
- Alpha all-NaN rolling windows return NaN using guarded counts, sums and variances rather than warning-producing reductions. RPC worker shutdown closes sockets and terminates its owned ZeroMQ context.
- Current statsmodels and the SQLite database plugin are explicit alpha/dev dependencies needed by supplied tests. Qt/native ML extras constrain Python support.

Current unresolved checks:

All supplied checks recorded below passed.

## Latest full-suite output

```text
$ uv run --no-sync python -m pytest /Users/alex/workspaces/trading/vnpy/tests
........................................................................ [ 44%]
........................................................................ [ 88%]
..................                                                       [100%]
162 passed in 54.00s

```

## Exact commands and exit codes

Unsuccessful rows are retained as historical compatibility attempts; they are not claims that a current passing check remains blocked. Source-build probes use isolated temporary copies to preserve the selected repository environment.

| Command | Exit | Seconds | Evidence |
|---|---:|---:|---|
| `uv lock --upgrade --python 3.15` | 0 | 1.41 | [check record](maintenance/index.html) |
| `uv sync --all-extras --all-groups --python 3.15 --no-build --no-install-project` | 2 | 0.06 | [check record](maintenance/index.html) |
| `uv lock --upgrade --python 3.14` | 0 | 1.27 | [check record](maintenance/index.html) |
| `uv sync --all-extras --all-groups --python 3.14 --no-build --no-install-project` | 0 | 20.55 | [check record](maintenance/index.html) |
| `uv sync --all-extras --all-groups --python 3.14` | 0 | 0.83 | [check record](maintenance/index.html) |
| `uv build --python 3.14` | 0 | 1.94 | [check record](maintenance/index.html) |
| `/Users/alex/.local/share/uv/python/cpython-3.14-macos-aarch64-none/bin/python3.14 -m compileall -q src` | 0 | 0.3 | [check record](maintenance/index.html) |
| `uv run --no-sync python -m pytest` | 1 | 5.86 | [check record](maintenance/index.html) |
| `uv run --no-sync python -m pytest src/vnpy` | 5 | 0.55 | [check record](maintenance/index.html) |
| `uv run --no-sync python -m pytest /Users/alex/workspaces/trading/vnpy/tests` | 2 | 6.39 | [check record](maintenance/index.html) |
| `uv lock --upgrade --python 3.14` | 0 | 1.49 | [check record](maintenance/index.html) |
| `uv sync --all-extras --all-groups --python 3.14` | 0 | 1.39 | [check record](maintenance/index.html) |
| `uv run --no-sync python -m pytest /Users/alex/workspaces/trading/vnpy/tests` | 1 | 59.67 | [check record](maintenance/index.html) |
| `uv run --no-sync python -m pytest /Users/alex/workspaces/trading/vnpy/tests` | 0 | 56.3 | [check record](maintenance/index.html) |
| `uv run --no-sync python -m compileall -q src` | 0 | 0.13 | [check record](maintenance/index.html) |
| `uv build --python 3.14` | 0 | 1.21 | [check record](maintenance/index.html) |
| `uv run --no-sync python -c import sys,importlib.metadata,json; print(sys.version); print(json.dumps({d.metadata["Name"]: d.version for d in importlib.metadata.distributions()},indent=2,sort_keys=True))` | 0 | 0.18 | [check record](maintenance/index.html) |
| `uv sync --directory /tmp/trading-python315-probes/vnpy --all-extras --all-groups --python 3.15` | 2 | 1.9 | [check record](maintenance/index.html) |

The versions-final log records the exact interpreter patch release and every installed dependency version. The machine-readable [verification record](maintenance/verification.json) includes branch/default ancestry, status, artifacts and unresolved checks.

## Dependency floor refresh

| Before | After |
|---|---|
| `pyarrow>=21.0.0` | `pyarrow>=26.0.0` |

The table records the initial metadata refresh. Later compatibility constraints and vendor sources are explained above and the final authoritative requirements are in [pyproject.toml](../pyproject.toml) and [uv.lock](../uv.lock). No test cases were added, removed or disabled. Existing deprecated-API cases retain their assertions while explicitly checking the documented warning.
