# Tracer: tooling-friction

Mission single-rollback-authority-01M3RCP4 (#5385). Seeded at planning 2026-09-30.

- The system `python3 -m pip install -e .` fails on a Debian-owned PyJWT; `--ignore-installed PyJWT` works. Tests need `uv sync --frozen --all-extras` into `.venv` (no pytest on the system interpreter).
- `tests/terminus` repros run ~20 s each; the 18-test 5318 set took 6 minutes at `-n 4`.
