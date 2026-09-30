# Tracer: design-decisions

Mission single-rollback-authority-01M3RCP4 (#5385). Seeded at planning 2026-09-30.

- DM-01M3RCPP2CR19QH3GZVPJT52BH: one mission, rollback routing + preflight (triage ruling).
- DM-01M3RCRDBS2RKWVVZH07AZ1B4M: committed done bookkeeping is rolled back with everything else; #1826 keep-done tests re-pinned.
- Preflight reuses the transaction's policy gate (extracted), not a second protection rule (research R-2).
- Wrapper re-raises original exception types; only BookkeepingPolicyRefused gets a command-layer rendering (research R-3).
- WP01: the in-phase `_restore_and_guard_coord_coherence` no longer heals (a forward `git revert` would commit inside the span and move the coordination branch off its recorded post tip); the heal runs only at resume start in the driver (AST-pinned).
- WP01: the #2786 successor trigger is another actor's commit on the coordination branch: the door reports it NOT restored, keeps the record and the strand marker, and the resume heal reconciles.
- WP01: `--abort` repro residue comes from a hard-kill wrapper (`_report_rollback` -> `os._exit(137)`), the only way a real run still leaves residue; the killed run's own global lock is released in the foreign-lock case.
