# Tracer: design-decisions

Mission single-rollback-authority-01M3RCP4 (#5385). Seeded at planning 2026-09-30.

- DM-01M3RCPP2CR19QH3GZVPJT52BH: one mission, rollback routing + preflight (triage ruling).
- DM-01M3RCRDBS2RKWVVZH07AZ1B4M: committed done bookkeeping is rolled back with everything else; #1826 keep-done tests re-pinned.
- Preflight reuses the transaction's policy gate (extracted), not a second protection rule (research R-2).
- Wrapper re-raises original exception types; only BookkeepingPolicyRefused gets a command-layer rendering (research R-3).
