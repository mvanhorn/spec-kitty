---
schema_version: 1
artifact_type: spec-kitty.analysis-report
command: /spec-kitty.analyze
mission_slug: charter-generation-drops-scoped-references-01M3M1KF
mission_id: 01M3M1KFFPC04BHFA54SSZ4CMG
generated_at: '2026-09-28T17:42:20.727414+00:00'
analyzer_agent: unknown
input_artifacts:
  spec.md:
    path: kitty-specs/charter-generation-drops-scoped-references-01M3M1KF/spec.md
    sha256: e7ab25e91ba3c33831e1238ad1831bafff583c940a9324039e88aae73cd244ef
  plan.md:
    path: kitty-specs/charter-generation-drops-scoped-references-01M3M1KF/plan.md
    sha256: 0ec1779f43e4b7d96d60e51360ee82eb150d6446e360948fc826807a63652f83
  tasks.md:
    path: kitty-specs/charter-generation-drops-scoped-references-01M3M1KF/tasks.md
    sha256: 928ebd62351578588d721cb2133db8c7adaa0c63b57f3e07a8e5019a7df8a5c8
  charter:
    path: .kittify/charter/charter.yaml
    sha256: cae74c9f8d1895556c96391ed8d733a628ffd85fe6b57c9b5a1f05ab6c873539
verdict: blocked
issue_counts:
  critical: 0
  low: 1
  high: 2
  medium: 1
  info: 0
findings:
- id: D1
  severity: high
  category: duplication
  summary: WP02 (IC-02/T007/T008) builds a new classify-and-placeholder helper plus 4-level active_languages threading in compiler.py, duplicating the already-built, already-used _diagnose_catalog_miss gate (src/charter/activation/context_renderers/catalog_diagnosis.py) which already does 'check scope_filtered_ids else classify_catalog_miss' and already reads active_languages off the repository's own _active_languages attribute with zero threading.
- id: C1
  severity: high
  category: coverage
  summary: T007's 'raw_id in repository.scope_filtered_ids' check will raise AttributeError against the mandatory baseline test tests/charter/test_catalog_completeness_4785.py::test_render_kind_references_routes_genuine_miss_to_diagnostics_not_placeholder, whose _EmptyRepository test double exposes only .get() and no scope_filtered_ids attribute; neither plan.md nor WP02's prompt names this incompatibility or the existing defensive getattr(repository, 'scope_filtered_ids', frozenset()) precedent (catalog_diagnosis.py:51), and WP02 does not own that test file to fix the fake itself.
- id: U1
  severity: medium
  category: underspecification
  summary: "WP02's T007 says the non-SCOPE_FILTERED branch 'falls through to classify_catalog_miss(raw_id, available_ids)' but classify_catalog_miss requires an available_ids: Iterable[str] fuzzy-match corpus that _render_kind_references has no parameter for; the existing _available_catalog_ids(repository) helper (catalog_diagnosis.py) solves exactly this (defensive against list_all/all-less stubs) but is never cited as the source."
- id: L1
  severity: low
  category: inconsistency
  summary: "lanes.json's lane-a (WP01, which only adds two new test files under tests/charter/) is tagged predicted_surfaces: [api, artifact-rendering, legacy-cleanup] -- surface labels that do not describe a test-only work package and appear to be stale/generic dashboard metadata rather than a considered classification."
---

## Specification Analysis Report

| ID | Category | Severity | Location(s) | Summary | Recommendation |
|----|----------|----------|-------------|---------|----------------|
| D1 | Duplication / DIRECTIVE_044 reuse gap | HIGH | plan.md "WP-CORE reconciliation"/IC-02; tasks/WP02-compiler-fix.md T007/T008; src/charter/activation/context_renderers/catalog_diagnosis.py `_diagnose_catalog_miss`, `_available_catalog_ids` | Plan/WP02 design a brand-new classify helper and thread `active_languages` through 4 call levels (`compile_charter` to `_build_references` to `_build_references_from_service` to `_render_kind_references`), duplicating the existing FR-013 canonical gate `_diagnose_catalog_miss(missing_id, repository)`, which already implements "scope_filtered_ids check, else classify_catalog_miss" and reads `active_languages` straight from the repository's own `_active_languages` attribute (set at construction in `BaseDoctrineRepository.__init__`, `src/charter/offering/base.py:106`) -- no threading needed. Plan's Key Entities "reuse target" bullet and WP02's "Reuse, do not reinvent (DIRECTIVE_044)" section cite only the raw `_catalog_miss.py` primitives, never `_diagnose_catalog_miss`/`_available_catalog_ids`, despite `_render_kind_references`'s own docstring confirming `repository` is exactly the raw per-kind repository object `_diagnose_catalog_miss` expects. | Before WP02 starts, decide explicitly whether to call `_diagnose_catalog_miss(raw_id, repository)` directly (or extract/share its logic) instead of building a parallel mechanism and IC-02's 4-level threading; if there is a deliberate reason not to reuse it (e.g. a layering rule between `activation/compiler.py` and `activation/context_renderers/`), state that reason in plan.md -- it is currently unaddressed, not ruled out. |
| C1 | Coverage / test-compatibility | HIGH | src/charter/activation/compiler.py `_render_kind_references`; tests/charter/test_catalog_completeness_4785.py:191-216 (`_EmptyRepository`); tasks/WP02-compiler-fix.md T007 | T007 requires the new helper's first step to be `raw_id in repository.scope_filtered_ids`. The mandatory, must-stay-green Contract C4 baseline test calls `_render_kind_references` directly with `_EmptyRepository()`, a bare test double exposing only `.get()`. Unguarded attribute access on `.scope_filtered_ids` raises `AttributeError` on this fixture, breaking the pinned Contract-C4 regression oracle every WP is told to treat as its regression gate -- unless the implementer independently discovers and applies the `getattr(repository, "scope_filtered_ids", frozenset())` defensive pattern already used at `catalog_diagnosis.py:51` (not mentioned in plan.md, the WP02 prompt, or the contract doc). | Add an explicit binding note (plan.md WP-CORE reconciliation, or WP02 prompt Context/Risks) requiring the defensive `getattr(..., "scope_filtered_ids", frozenset())` form, citing the existing precedent -- or resolve via D1's reuse of `_diagnose_catalog_miss`, which already applies it. |
| U1 | Underspecification | MEDIUM | src/charter/activation/_catalog_miss.py:225-228 (`classify_catalog_miss` signature); tasks/WP02-compiler-fix.md T007 | `classify_catalog_miss(missing_id, available_ids)` requires an `Iterable[str]` fuzzy-match corpus. T007's text ("falls through to `classify_catalog_miss(raw_id, available_ids)`") never states where `available_ids` comes from inside `_render_kind_references`'s scope -- that function only receives the requested `ids` list and `repository`, not a full listing. The existing `_available_catalog_ids(repository)` helper already solves this defensively (falls back through `list_all`/`all`/`_items`) but is not cited. | Name `_available_catalog_ids` (or an equivalent) explicitly as the corpus source in plan.md/WP02, or resolve automatically via D1's reuse of `_diagnose_catalog_miss` (which already calls it internally). |
| L1 | Inconsistency (metadata) | LOW | kitty-specs/charter-generation-drops-scoped-references-01M3M1KF/lanes.json lane-a | WP01 (two new test files only, no `src/` change) is tagged with `predicted_surfaces: ["api", "artifact-rendering", "legacy-cleanup"]` -- labels that do not describe a red-first test authoring WP and read as generic/stale classification. | Cosmetic; not blocking. Worth a one-line correction pass on `predicted_surfaces` if lanes.json is regenerated, otherwise safe to ignore. |

**Coverage Summary Table:**

| Requirement Key | Has Task? | Task IDs | Notes |
|-----------------|-----------|----------|-------|
| FR-001 (preserve-or-report, incl. whole-kind + graph-load carve-out) | Yes | T001-T013 (WP01, WP02) | I1-I5 invariant table + WP02 T007/T009/T010/T011 |
| FR-002 (distinguish never-existed vs. filtered) | Yes | T007, T008, T012 | Subject to D1/U1 above for the mechanism's soundness |
| FR-003 (--force unaffected) | Yes | WP02 (test-only assertion, no code branch) | Verified by paired with/without-force run |
| FR-004 (machine-readable --json) | Yes | T014-T016 (WP03) | |
| FR-005 (generator-to-parity regression test) | Yes | T001 (WP01) | |
| FR-006 (backward-compatible, no forced regen) | Yes (indirect) | Existing baseline `test_active_languages_idempotency.py` (`test_charter_generate_is_idempotent_*`) | No dedicated new fixture, but pre-existing regression coverage already in the pinned 56-test baseline addresses the byte-identical/no-mutation claim |
| NFR-001 (idempotent regeneration) | Yes | T005 (I5), WP02 baseline re-run | |
| NFR-002 (actionable diagnostic content) | Yes | T007-T013, T008's language-naming assertion | |
| C-001 (parity guard untouched) | Yes | Baseline command re-run at every WP | |
| C-002 (no six-id special case) | Yes | T001's extra differently-scoped id requirement | |
| C-003 (no absolute source_path) | Implicit | Not exercised by a dedicated new test; relies on unchanged provenance convention | Minor -- no dedicated assertion, but the mission does not touch `source_path` construction, so risk is low |
| C-004 (public-repo hygiene) | N/A (process) | Enforced by mission conduct, not a WP task | |
| Invariants I1-I5 + carve-out + `_raw_kind_repository` degrade | Yes | T002-T006 (WP01), T009-T012 (WP02) | Full 7-row ATDD table mapped |
| Operator rulings, plan.ruling.md rounds 3-7 | Yes | All rulings are baked into plan.md's binding sections and restated verbatim in WP02's Context/Binding-mechanism-decisions section; round-7's spec.md wording carve-outs are textual-only and already present in spec.md (verified) | |

**Charter Alignment Issues:**

- DIRECTIVE_044 (single canonical authority) -- see D1. The charter marks this principle as "reviewers weigh it; not an automated rejection," so it is scored HIGH rather than an automatic CRITICAL, but it is a substantive, concrete miss across 7 review rounds, not a style nit.
- C-011 (ATDD-first) -- verified satisfied on paper: WP01's two red-first commits, WP03's T016, and WP04's T019 are each explicitly sequenced before their implementation commits, with RED-verification steps spelled out.
- C-007 (`__all__`) -- verified satisfied on paper: the new classify-and-placeholder helper and `RefreshOutcome` are both explicitly kept module-private, matching existing precedent.
- Campsite/tracer files -- the plan explicitly declines a separate campsite-clean commit with stated reasoning (would be immediately overwritten), and tracer files are seeded per the Project Structure section. Both are considered decisions, not silently skipped.
- Pre-existing Failure Reporting Rule -- plan.md's "The baseline" section states plainly there is no pre-existing red in this mission's blast radius (56 passed, 0 failed), so no PEFR issue is owed.

**Unmapped Tasks:** None found -- every subtask T001-T019 maps to a named WP, requirement, or plan concern; no orphaned subtask IDs and no gaps in the T001-T019 sequence.

**Metrics:**

- Total Requirements (FR+NFR+C): 12 (FR-001..006, NFR-001..002, C-001..004)
- Total Success Criteria: 5 (SC-001..005)
- Total Invariants: 5 (I1-I5) + 1 carve-out row + 1 residual-fixture row = 7 ATDD table rows
- Total Tasks/Subtasks: 19 (T001-T019) across 4 WPs
- Coverage %: 100% of FR/NFR/C/SC/invariants have at least one mapped WP/subtask
- Ambiguity Count: 0 (no vague adjectives or unresolved placeholders found in spec.md/plan.md/tasks.md/WP files)
- Duplication Count: 1 (D1)
- Critical Issues Count: 0
- High Issues Count: 2 (D1, C1)

## Next Actions

Two HIGH findings exist (D1, C1) plus one MEDIUM (U1) that is closely coupled to D1's remedy. Recommend resolving D1 and C1 before `/implement` starts on WP02:

- D1: decide explicitly whether WP02 reuses `_diagnose_catalog_miss`/`_available_catalog_ids` (`src/charter/activation/context_renderers/catalog_diagnosis.py`) instead of building a new parallel classify helper + 4-level `active_languages` threading (IC-02/T008). This may eliminate T008 entirely and resolve U1 as a side effect.
- C1: whether or not D1's reuse path is taken, pin explicitly (in plan.md or the WP02 prompt) that any `repository.scope_filtered_ids` access must be defensive (`getattr(..., "scope_filtered_ids", frozenset())`) so the Contract C4 baseline test's bare `_EmptyRepository` double does not crash `_render_kind_references`.
- U1: name the `available_ids` corpus source for `classify_catalog_miss`'s non-`SCOPE_FILTERED` branch explicitly.
- L1: cosmetic; safe to defer or fix opportunistically.

Suggested command: return to `/spec-kitty.plan` (or a targeted plan amendment) to add an explicit ruling on D1/C1/U1's reuse-vs-reinvent question before WP02 implementation begins, then re-run `/spec-kitty.analyze`.
