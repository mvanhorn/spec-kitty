# Tracer: tooling-friction

Mission single-rollback-authority-01M3RCP4 (#5385). Seeded at planning 2026-09-30.

- The system `python3 -m pip install -e .` fails on a Debian-owned PyJWT; `--ignore-installed PyJWT` works. Tests need `uv sync --frozen --all-extras` into `.venv` (no pytest on the system interpreter).
- `tests/terminus` repros run ~20 s each; the 18-test 5318 set took 6 minutes at `-n 4`.
- WP01: `ruff format` on a file listed in pyproject's format-exclude ratchet trips `test_ruff_format_exclude_ratchet.py` (every excluded entry must still reformat); checking via a copy outside the repo hides the exclude. Use `ruff format --check --force-exclude` on explicit paths.
- WP01: `pytest.ini` sets `pythonpath = src`, so a scratch copy of `src` on `PYTHONPATH` is shadowed; a planted-break proof must edit the worktree itself (and restore it).
- WP01: first pytest run in a fresh process spends ~60 s warming up; re-runs take seconds.
