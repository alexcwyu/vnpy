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
| `uv lock --upgrade --python 3.15` | 0 | 1.41 | [log](../../docs/maintenance-2026-10-12/vnpy-lock-3.15.log) |
| `uv sync --all-extras --all-groups --python 3.15 --no-build --no-install-project` | 2 | 0.06 | [log](../../docs/maintenance-2026-10-12/vnpy-wheel-probe-3.15.log) |
| `uv lock --upgrade --python 3.14` | 0 | 1.27 | [log](../../docs/maintenance-2026-10-12/vnpy-lock-3.14.log) |
| `uv sync --all-extras --all-groups --python 3.14 --no-build --no-install-project` | 0 | 20.55 | [log](../../docs/maintenance-2026-10-12/vnpy-wheel-probe-3.14.log) |
| `uv sync --all-extras --all-groups --python 3.14` | 0 | 0.83 | [log](../../docs/maintenance-2026-10-12/vnpy-sync-3.14.log) |
| `uv build --python 3.14` | 0 | 1.94 | [log](../../docs/maintenance-2026-10-12/vnpy-build.log) |
| `/Users/alex/.local/share/uv/python/cpython-3.14-macos-aarch64-none/bin/python3.14 -m compileall -q src` | 0 | 0.3 | [log](../../docs/maintenance-2026-10-12/vnpy-compile.log) |
| `uv run --no-sync python -m pytest` | 1 | 5.86 | [log](../../docs/maintenance-2026-10-12/vnpy-tests.log) |
| `uv run --no-sync python -m pytest src/vnpy` | 5 | 0.55 | [log](../../docs/maintenance-2026-10-12/vnpy-tests-existing-source.log) |
| `uv run --no-sync python -m pytest /Users/alex/workspaces/trading/vnpy/tests` | 2 | 6.39 | [log](../../docs/maintenance-2026-10-12/vnpy-tests-absolute-suite.log) |
| `uv lock --upgrade --python 3.14` | 0 | 1.49 | [log](../../docs/maintenance-2026-10-12/vnpy-lock-missing-dependencies.log) |
| `uv sync --all-extras --all-groups --python 3.14` | 0 | 1.39 | [log](../../docs/maintenance-2026-10-12/vnpy-sync-missing-dependencies.log) |
| `uv run --no-sync python -m pytest /Users/alex/workspaces/trading/vnpy/tests` | 1 | 59.67 | [log](../../docs/maintenance-2026-10-12/vnpy-tests-missing-dependencies.log) |
| `uv run --no-sync python -m pytest /Users/alex/workspaces/trading/vnpy/tests` | 0 | 56.3 | [log](../../docs/maintenance-2026-10-12/vnpy-tests-alpha-rpc-compatibility.log) |
| `uv run --no-sync python -m compileall -q src` | 0 | 0.13 | [log](../../docs/maintenance-2026-10-12/vnpy-compile-final.log) |
| `uv build --python 3.14` | 0 | 1.21 | [log](../../docs/maintenance-2026-10-12/vnpy-build-final.log) |
| `uv run --no-sync python -c import sys,importlib.metadata,json; print(sys.version); print(json.dumps({d.metadata["Name"]: d.version for d in importlib.metadata.distributions()},indent=2,sort_keys=True))` | 0 | 0.18 | [log](../../docs/maintenance-2026-10-12/vnpy-versions-final.log) |
| `uv sync --directory /tmp/trading-python315-probes/vnpy --all-extras --all-groups --python 3.15` | 2 | 1.9 | [log](../../docs/maintenance-2026-10-12/vnpy-python315-source-probe.log) |

The versions-final log records the exact interpreter patch release and every installed dependency version. The machine-readable [result](../../docs/maintenance-2026-10-12/vnpy-result.json) includes branch/default ancestry, status, artifacts and unresolved checks.

## Dependency floor refresh

| Before | After |
|---|---|
| `pyarrow>=21.0.0` | `pyarrow>=26.0.0` |

The table records the initial metadata refresh. Later compatibility constraints and vendor sources are explained above and the final authoritative requirements are in [pyproject.toml](../pyproject.toml) and [uv.lock](../uv.lock). No test cases were added, removed or disabled. Existing deprecated-API cases retain their assertions while explicitly checking the documented warning.
