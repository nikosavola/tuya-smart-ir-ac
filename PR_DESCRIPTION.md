## Chore: lint/compile CI + housekeeping

- Adds a `ruff` config and a GitHub Actions workflow (ruff + `compileall` smoke test).
  Ruff is **report-only** for now since the codebase has pre-existing style findings;
  flip off `--exit-zero` once those are cleaned up to make it a hard gate.
- Cleanups: drop the unused `TEST_MODE` constant, add a `loggers` manifest key, fix a
  wrong return type hint in `entity.py`, translate stray Italian comments in
  `openapi.py`, and guard ECB depadding against a bad padding length.
