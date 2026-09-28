---
schema_version: 1
artifact_type: spec-kitty.analysis-report
command: /spec-kitty.analyze
mission_slug: charter-generation-drops-scoped-references-01M3M1KF
mission_id: 01M3M1KFFPC04BHFA54SSZ4CMG
generated_at: '2026-09-28T18:16:15.755280+00:00'
analyzer_agent: claude
input_artifacts:
  spec.md:
    path: kitty-specs/charter-generation-drops-scoped-references-01M3M1KF/spec.md
    sha256: e7ab25e91ba3c33831e1238ad1831bafff583c940a9324039e88aae73cd244ef
  plan.md:
    path: kitty-specs/charter-generation-drops-scoped-references-01M3M1KF/plan.md
    sha256: 3e4ba34c704b933c76acbb37276dbf78965461696a90fb0ad77e2a36580f5abe
  tasks.md:
    path: kitty-specs/charter-generation-drops-scoped-references-01M3M1KF/tasks.md
    sha256: 928ebd62351578588d721cb2133db8c7adaa0c63b57f3e07a8e5019a7df8a5c8
  charter:
    path: .kittify/charter/charter.yaml
    sha256: cae74c9f8d1895556c96391ed8d733a628ffd85fe6b57c9b5a1f05ab6c873539
verdict: ready
issue_counts:
  high: 0
  low: 0
  medium: 1
  critical: 0
  info: 0
findings:
- id: W1
  severity: medium
  category: inconsistency
  summary: lanes.json lane-c's write_scope lists only src/specify_cli/cli/commands/charter/generate.py, omitting tests/specify_cli/cli/commands/test_charter_generate_autotrack.py, which wps.yaml and tasks/WP03-json-diagnostics.md's frontmatter both declare as a WP03-owned file.
---

## Specification Analysis Report

| ID | Category | Severity | Location(s) | Summary | Recommendation |
|----|----------|----------|-------------|---------|----------------|
| W1 | Inconsistency (write-scope ledger) | MEDIUM | kitty-specs/charter-generation-drops-scoped-references-01M3M1KF/lanes.json (lane-c); wps.yaml (WP03); tasks/WP03-json-diagnostics.md frontmatter | `lanes.json`'s `lane-c.write_scope` is `["src/specify_cli/cli/commands/charter/generate.py"]` only. Both `wps.yaml`'s WP03 entry and `tasks/WP03-json-diagnostics.md`'s own frontmatter declare `owned_files: [src/specify_cli/cli/commands/charter/generate.py, tests/specify_cli/cli/commands/test_charter_generate_autotrack.py]` — the test file WP03's own T014/T015/T016 all write to (per the WP prompt's Subtask sections) is present in every other authoritative source but absent from `lanes.json`'s write-scope ledger. All three other lanes (lane-a/WP01, lane-b/WP02, lane-d/WP04) have write_scope that matches their WP's owned_files exactly — lane-c is the sole outlier. Because write_scope (unlike the `predicted_surfaces` field, which is generator-owned cosmetic metadata per this mission's own tracer-tooling-friction.md L1 entry) is the field the "one checkout, many agents"/write-scope-disjointness safety mechanism actually reads, an operator or future re-planning pass that trusts `lanes.json` as complete could believe no lane touches that test file and permit a conflicting concurrent write. The mission's strict linear dependency chain (lane-a → lane-b → lane-c) means there is no live collision today, but the ledger itself is wrong. | Regenerate `lanes.json` via the sanctioned `spec-kitty agent mission finalize-tasks` path (which recomputes write_scope from wps.yaml/owned_files) rather than hand-editing the JSON; if that command's current output would also touch unrelated WP frontmatter/status as tracer-tooling-friction.md's L1 entry notes for the sibling predicted_surfaces gap, weigh that against leaving the gap recorded until a narrower recompute path exists upstream. |

**Coverage Summary Table:**

| Requirement Key | Has Task? | Task IDs | Notes |
|-----------------|-----------|----------|-------|
| FR-001 (preserve-or-report, incl. whole-kind + graph-load carve-out) | Yes | T001-T013 (WP01, WP02) | I1-I5 invariant table + WP02 T007/T009/T010/T011; round-8 fix verified against live source (`_diagnose_catalog_miss`, `_active_languages`, `scope_filtered_ids` all match compiler.py/catalog_diagnosis.py/base.py exactly) |
| FR-002 (distinguish never-existed vs. filtered) | Yes | T007, T008, T012 | D1/C1/U1 (prior analyze findings) closed this round by reuse of the canonical `_diagnose_catalog_miss` gate — independently re-verified live: gate is in `catalog_diagnosis.py.__all__`, guards `scope_filtered_ids` with `getattr(..., frozenset())`, sources fuzzy-match corpus from `_available_catalog_ids`, reads `_active_languages` off the repository with no threading |
| FR-003 (--force unaffected) | Yes | WP02 (test-only assertion, no code branch) | Verified by paired with/without-force run |
| FR-004 (machine-readable --json) | Yes | T014-T016 (WP03) | Contract doc's shape (kind/id/cause/detail, sentinel values `_graph`/`_unattributed`) is internally consistent with plan.md's I1-I5 and WP01's fixture table |
| FR-005 (generator-to-parity regression test) | Yes | T001 (WP01) | |
| FR-006 (backward-compatible, no forced regen) | Yes (indirect) | Existing baseline `test_active_languages_idempotency.py` | |
| NFR-001 (idempotent regeneration) | Yes | T005 (I5), WP02 baseline re-run | |
| NFR-002 (actionable diagnostic content) | Yes | T007-T013, T008's language-naming assertion | |
| C-001 (parity guard untouched) | Yes | Baseline command re-run at every WP | |
| C-002 (no six-id special case) | Yes | T001's extra differently-scoped id requirement | |
| C-003 (no absolute source_path) | Implicit | Not exercised by a dedicated new test | Mission does not touch `source_path` construction; risk low |
| C-004 (public-repo hygiene) | N/A (process) | Enforced by mission conduct, not a WP task | Verified: the only `/home/` string in spec.md is inside C-004's own illustrative placeholder text, not a real leak |
| Invariants I1-I5 + carve-out + `_raw_kind_repository` degrade | Yes | T002-T006 (WP01), T009-T012 (WP02) | Full 7-row ATDD table mapped; live-source line numbers for `_raw_kind_repository` (compiler.py:1073-1093), `_render_kind_references` (1096-1141), `graph.unresolved` loop (1250-1251), and `_resolve_transitive_reference_graph`'s except branch (1298-1307/1306-1307) all re-verified against HEAD and match plan.md's citations |
| Operator rulings, plan.ruling.md rounds 3-7 | Yes | Confirmed present in reviews/plan.ruling.md (rounds 3-7 headings all exist) and restated faithfully in plan.md/WP02 | |

**Charter Alignment Issues:**

- DIRECTIVE_044 (single canonical authority) — prior HIGH finding D1 is now closed: WP02 reuses `_diagnose_catalog_miss`/`_available_catalog_ids` instead of a parallel classify helper, verified against live source.
- C-011 (ATDD-first) — satisfied: WP01's two red-first commits, WP03's T016, and WP04's T019 are each sequenced before their implementation commits with explicit RED-verification steps; WP02 legitimately has no internal red-first commit since it turns WP01's already-red tests green (does not violate C-011).
- C-007 (`__all__`) — satisfied on paper: new classify-and-placeholder helper and `RefreshOutcome` are both explicitly module-private, matching existing `compiler.py` precedent (`_render_kind_references`, `_doctrine_model_reference`, `_doctrine_yaml_reference` are likewise unlisted).
- Campsite/tracer files — plan explicitly declines a separate campsite-clean commit with stated reasoning; three tracer files exist and are being appended each phase (confirmed non-empty, substantive content).
- Pre-existing Failure Reporting Rule — plan.md states no pre-existing red in this mission's blast radius (56 passed, 0 failed); not independently re-run this pass (out of scope for a non-remediating cross-artifact analysis), but the claim is internally consistent with the stated baseline command and file set.

**Unmapped Tasks:** None found — every subtask T001-T019 maps to a named WP, requirement, or plan concern; no orphaned subtask IDs and no gaps in the T001-T019 sequence.

**Metrics:**

- Total Requirements (FR+NFR+C): 12 (FR-001..006, NFR-001..002, C-001..004)
- Total Success Criteria: 5 (SC-001..005)
- Total Invariants: 5 (I1-I5) + 1 carve-out row + 1 residual-fixture row = 7 ATDD table rows
- Total Tasks/Subtasks: 19 (T001-T019) across 4 WPs
- Coverage %: 100% of FR/NFR/C/SC/invariants have at least one mapped WP/subtask
- Ambiguity Count: 0 (no vague adjectives or unresolved placeholders found in spec.md/plan.md/tasks.md/WP files)
- Duplication Count: 0 (prior HIGH finding D1 closed this round)
- Critical Issues Count: 0
- High Issues Count: 0
- Medium Issues Count: 1 (W1)

## Next Actions

No CRITICAL or HIGH issues remain. One MEDIUM finding (W1, a `lanes.json` write-scope ledger gap for WP03) does not block `/implement` given the mission's strict linear WP dependency chain (WP01 → WP02 → WP03; WP04 independent), but should be corrected via a sanctioned `finalize-tasks` regeneration before or during WP03's lane setup so the write-scope disjointness ledger stays trustworthy for any future re-planning.

Suggested command: proceed to `/spec-kitty.implement` (or the mission's `agent action implement` entry point) for WP01; optionally re-run `spec-kitty agent mission finalize-tasks` first if the operator wants W1's `lanes.json` gap closed before implementation starts.

## tooling_notes

- **lanes.json lane-a `predicted_surfaces: ["api", "artifact-rendering", "legacy-cleanup"]` mischaracterizes a tests-only WP (prior finding L1).** WP01 owns only two new files under `tests/charter/` with no `src/` change, yet is tagged with generic dashboard-classification surfaces that plainly do not describe a red-first test-authoring WP. This is `lanes.json`'s `predicted_surfaces` metadata specifically — the field this mission's dispatch explicitly names as generator output no sanctioned command can regenerate in isolation (`spec-kitty agent mission finalize-tasks --help` was checked this round per `tracer-tooling-friction.md`'s "Analyze fix-round phase" entry: it has no flag to recompute `lanes.json` metadata alone without also mutating WP frontmatter/status, and `--validate-only` skips all writes). Already recorded in `tracer-tooling-friction.md`; carried forward here as a tooling note, not a finding, per this run's dispatch instructions. Not blocking.
