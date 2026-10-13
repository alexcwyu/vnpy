# vnpy: portable verification records

<!-- current-documentation-provenance -->
Documentation reviewed: 2026-10-13 / Git source base: `fbe7d7940e45206491063d56b6c4a6d581d35d0a` / [Current architecture and feature guide](../ayu-develop.md).
Historical execution commands/results below retain their original dates and scope; this metadata update is not a new application test run.
<!-- /current-documentation-provenance -->


Full raw logs are retained in the workspace maintenance evidence, not distributed here. Hashes identify those original logs. A successful build or selected test subset does not imply a full test-suite pass. Commands containing <workspace> describe the original execution environment.

[Machine-readable records](verification.json) · [Repository validation summary](../validation.md)

## Executed commands

| Check | Exit | Duration | Command | Outcomes |
| --- | ---: | ---: | --- | --- |
| build | 0 | 1.94s | `uv build --python 3.14` | Successfully built dist/vnpy-4.5.0.tar.gz<br>Successfully built dist/vnpy-4.5.0-py3-none-any.whl |
| build-final | 0 | 1.21s | `uv build --python 3.14` | Successfully built dist/vnpy-4.5.0.tar.gz<br>Successfully built dist/vnpy-4.5.0-py3-none-any.whl |
| compile | 0 | 0.3s | `/Users/alex/.local/share/uv/python/cpython-3.14-macos-aarch64-none/bin/python3.14 -m compileall -q src` | See exit status; no test-count assertion. |
| compile-final | 0 | 0.13s | `uv run --no-sync python -m compileall -q src` | See exit status; no test-count assertion. |
| lock-3.14 | 0 | 1.27s | `uv lock --upgrade --python 3.14` | See exit status; no test-count assertion. |
| lock-3.15 | 0 | 1.41s | `uv lock --upgrade --python 3.15` | See exit status; no test-count assertion. |
| lock-missing-dependencies | 0 | 1.49s | `uv lock --upgrade --python 3.14` | See exit status; no test-count assertion. |
| python315-source-probe | 2 | 1.9s | `uv sync --directory /tmp/trading-python315-probes/vnpy --all-extras --all-groups --python 3.15` | See exit status; no test-count assertion. |
| sync-3.14 | 0 | 0.83s | `uv sync --all-extras --all-groups --python 3.14` | See exit status; no test-count assertion. |
| sync-missing-dependencies | 0 | 1.39s | `uv sync --all-extras --all-groups --python 3.14` | See exit status; no test-count assertion. |
| tests | 1 | 5.86s | `uv run --no-sync python -m pytest` | See exit status; no test-count assertion. |
| tests-absolute-suite | 2 | 6.39s | `uv run --no-sync python -m pytest <workspace>/vnpy/tests` | See exit status; no test-count assertion. |
| tests-alpha-rpc-compatibility | 0 | 56.3s | `uv run --no-sync python -m pytest <workspace>/vnpy/tests` | 162 passed in 54.00s |
| tests-existing-source | 5 | 0.55s | `uv run --no-sync python -m pytest src/vnpy` | See exit status; no test-count assertion. |
| tests-missing-dependencies | 1 | 59.67s | `uv run --no-sync python -m pytest <workspace>/vnpy/tests` | 6 failed, 156 passed in 57.55s |
| versions-final | 0 | 0.18s | `uv run --no-sync python -c import sys,importlib.metadata,json; print(sys.version); print(json.dumps({d.metadata["Name"]: d.version for d in importlib.metadata.distributions()},indent=2,sort_keys=True))` | See exit status; no test-count assertion. |
| wheel-probe-3.14 | 0 | 20.55s | `uv sync --all-extras --all-groups --python 3.14 --no-build --no-install-project` | See exit status; no test-count assertion. |
| wheel-probe-3.15 | 2 | 0.06s | `uv sync --all-extras --all-groups --python 3.15 --no-build --no-install-project` | See exit status; no test-count assertion. |
