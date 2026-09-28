---
schema_version: 1
artifact_type: spec-kitty.analysis-report
command: /spec-kitty.analyze
mission_slug: owned-checkout-lifecycle-authority-01M3M2ZB
mission_id: 01M3M2ZB5D4A33B8070F97DKW7
generated_at: '2026-09-28T16:10:49.146874+00:00'
analyzer_agent: unknown
input_artifacts:
  spec.md:
    path: kitty-specs/owned-checkout-lifecycle-authority-01M3M2ZB/spec.md
    sha256: 5f437d3bc340be2827ec544f581e91f9e2539e1e86b4a72a4a9251ff3bb0b21e
  plan.md:
    path: kitty-specs/owned-checkout-lifecycle-authority-01M3M2ZB/plan.md
    sha256: ce4663bf193ac75724fa76970f47e03930ddcb81dbad8413c741cca1df311325
  tasks.md:
    path: kitty-specs/owned-checkout-lifecycle-authority-01M3M2ZB/tasks.md
    sha256: 043d71af6804ddea01e758ae389bf0de0ea9d06b6975def258239860aaedf853
  charter:
    path: .kittify/charter/charter.yaml
    sha256: cae74c9f8d1895556c96391ed8d733a628ffd85fe6b57c9b5a1f05ab6c873539
verdict: ready
issue_counts:
  critical: 0
  low: 3
  medium: 5
  high: 0
  info: 0
findings:
- id: C1
  severity: medium
  category: coverage
  summary: US3-AS6 (the fact is reused when the repository root checkout's branch changes mid-command, via a test hook) has no task or test in any WP.
- id: I1
  severity: medium
  category: inconsistency
  summary: The error-code registry differs across artifacts. Spec US1-AS3 and NFR-004 omit OWNED_CHECKOUT_IS_MISSION_WORKTREE and the OWNERSHIP_* claim codes, and WP09 T049 step 2 says the registry has no lane/coordination code, which contradicts its own table and data-model.md.
- id: I2
  severity: medium
  category: inconsistency
  summary: NFR-004 requires every owned refusal's code in --json output, but the FR-018 refusals on agent action implement/review have no --json flag, and WP09 forbids adding one.
- id: U1
  severity: medium
  category: underspecification
  summary: LANES is left out of NEXT_OWNED_TOPOLOGIES 'deliberately (research R-02)', but R-02 gives no LANES rationale, the spec's topology rule does not mention LANES, and no FR or test pins the new refusal for owned next.
- id: U2
  severity: medium
  category: underspecification
  summary: No artifact specifies or pins the refusal code for a missing --owned-checkout path. WP09 T049 needs WP02 to have pinned it, but WP02 has no such test.
- id: I3
  severity: low
  category: inconsistency
  summary: Spec's Common-fixture 'R unchanged' definition (working tree/HEAD/index, shared git dir excluded) is narrower than NFR-001 (lock root inside common dir, SPEC_KITTY_HOME, P-subtree exclusion).
- id: I4
  severity: low
  category: inconsistency
  summary: The bare owned-root census reads 92 in spec FR-001, plan Summary and research R-03, but 94 in plan Scale/Scope.
- id: A1
  severity: low
  category: ambiguity
  summary: FR-001 says 'allowlist is empty' and 'any parameter or field anywhere in src/', but the gate contract has five named exemptions (carrier fields, CLI_CLAIM_INPUT_RULE, ORG_PACK_MODULE_RULE, migrations rule, OrgPackConfig call rule).
---

## Specification Analysis Report

Mission `owned-checkout-lifecycle-authority-01M3M2ZB`. Artifacts analysed: spec.md, plan.md, tasks.md, tasks/WP01..WP19, data-model.md, contracts/, research.md, quickstart.md, occurrence_map.yaml, meta.json, and `.kittify/charter/charter.md`. Governance applied: reviewer-renata profile, and the review action charter context (terminology canon, regression vigilance, code-review checklist, ATDD-first C-011).

The following are intentional design choices and are not flagged: the temporary dual keywords and `TRANSITIONAL(WP18)` markers that WP18 deletes, the retired subtask ids T056/T059 (moved to WP19 as T101/T103), WP03 as a `planning_artifact`, and C-004/C-006/C-007 being enforced through prose.

| ID | Category | Severity | Location(s) | Summary | Recommendation |
|----|----------|----------|-------------|---------|----------------|
| C1 | Coverage | MEDIUM | spec.md:L129 (US3-AS6); tasks.md (no row); WP11/WP19/WP18 | US3-AS6 requires a test hook that changes R's current branch between ownership validation and prompt build, then shows that all reads use the original fact. No subtask or DoD item implements this hook. FR-003's "exactly 1 validation" count (WP19 T102, WP07 T037) is a partial proxy only: it would not catch re-derivation by other means, such as `get_main_repo_root(cwd)` or `locate_project_root`. | Add a row to WP19 T102 (or WP18 T098) with the branch-flip hook, or explicitly map US3-AS6 to the FR-003 count test in spec. |
| I1 | Inconsistency | MEDIUM | spec.md:L76-83 (US1-AS3), L239 (NFR-004); data-model.md "Error code registry"; WP09:L388-389 vs L396 | The spec's registered set is "existing `OWNED_*` codes plus four new ones". It leaves out `OWNED_CHECKOUT_IS_MISSION_WORKTREE` (new in data-model, WP01 S7, WP02 T009, WP03) and the claim-primitive `OWNERSHIP_*` / `WORKTREE_INVOCATION_REFUSED` codes. WP09 T049's table expects `OWNED_CHECKOUT_IS_MISSION_WORKTREE` for lane and coordination worktrees. Step 2 of the same subtask says "the NFR-004 registry does not list a lane or coordination refusal code". | Amend NFR-004 and US1-AS3 to cite data-model.md's registry as authoritative, including `OWNED_CHECKOUT_IS_MISSION_WORKTREE`. Drop the stale sentence in WP09 T049 step 2. |
| I2 | Inconsistency | MEDIUM | spec.md:L239 (NFR-004) vs L222 (FR-018); WP09:L145 | NFR-004 requires every owned refusal to return a registered code "in `--json` output". `agent action implement`/`review` have no `--json` flag. WP09 makes their refusal a stderr line `Error: [OWNED_ACTION_UNSUPPORTED] …` and forbids adding `--json`. | Add an NFR-004 carve-out: commands without `--json` emit the code in the `Error: [<CODE>]` human line. |
| U1 | Underspecification | MEDIUM | data-model.md (`NEXT_OWNED_TOPOLOGIES`); WP02:L162-166; research.md:L19-21 (R-02); spec.md:L42-47 | `NEXT_OWNED_TOPOLOGIES = {SINGLE_BRANCH, LANES_WITH_COORD, COORD}`, and WP02 says LANES is "deliberately absent (research R-02)". R-02 gives no LANES rationale. Today `next --owned-checkout` applies no topology check (`src/specify_cli/cli/commands/next_cmd.py:160-176` calls only the claim primitive), so after WP19 an owned LANES mission would be newly refused with `OWNED_TOPOLOGY_UNSUPPORTED`. No FR, acceptance scenario or FR-022 ratchet names or pins this change. | State the LANES decision and its rationale in spec (topology rule) and R-02, and pin it with a paired test in WP02 or WP19. Otherwise, add LANES to the set. |
| U2 | Underspecification | MEDIUM | spec.md:L77-78 (US1-AS3 "a missing path", "a non-worktree directory"); WP09:L385-386, L396; WP02 (no missing-path test) | US1-AS3 says each invalid value "exits non-zero with the listed code", but no code is listed for a missing path or a non-worktree directory. WP09 T049 expects "the code WP02's minter test pins for a missing path (expected `OWNERSHIP_BROKEN_POINTER`)". WP02 has no missing-path test. WP09's own rule is to stop and escalate, so this is a known stop point in the implement loop. | Specify the missing-path and non-worktree-directory codes in spec or data-model, and add a missing-path case to WP02 T006/T009's minter tests. |
| I3 | Inconsistency | LOW | spec.md:L53-58 vs L236 (NFR-001); quickstart.md "Expected throughout" | The Common-fixture definition of "R unchanged" excludes the shared git directory and covers only the working tree, `HEAD` and the index. NFR-001 adds `<common-dir>/spec-kitty-locks` (inside the shared git dir), `SPEC_KITTY_HOME`, and the exclusion of exactly P's subtree. WP02 T011 correctly implements the NFR-001 version. | Make the Common-fixture paragraph (and quickstart) reference NFR-001's definition. |
| I4 | Inconsistency | LOW | spec.md:L205; plan.md:L13, L18, L47; research.md:L24-25 | The same census is given as 92 in spec FR-001, plan Summary and research R-03, and as 94 "in 30 files" in plan Scale/Scope. Execution is unaffected because the G4/G5 floor is ≥1 and the allowlist is empty. | Reconcile on the post-plan-squad number, or say "all" rather than a count. |
| A1 | Ambiguity | LOW | spec.md:L205 (FR-001); contracts/architectural-gate.md (G5 exemptions, ORG_PACK_MODULE_RULE) | FR-001 reads as absolute ("any parameter or field anywhere in `src/`", "allowlist is empty"). The gate contract has named exemptions: `OwnedCheckout` fields, `CLI_CLAIM_INPUT_RULE`, the org-pack module rule, the migrations `checkout_root` rule, and the `OrgPackConfig.effective_root` call rule. These are justified, but not visible from spec. | Add one sentence to FR-001: "named exemptions per contracts/architectural-gate.md; no allowlist entries". |

**Coverage Summary Table:**

| Requirement Key | Has Task? | Task IDs / WPs | Notes |
|-----------------|-----------|----------------|-------|
| FR-001 one-ownership-validator | Yes | WP01 T004-T005; WP18 T095-T097 (+WP15, WP17 sweep) | Gate floors plus self-mutation |
| FR-002 next-uses-validator | Yes | WP02 T006; WP19 T102-T103 | O9 |
| FR-003 validated-once | Yes | WP07 T037; WP16 T089; WP14; WP19 T102 | See C1 |
| FR-004 owned-status-listing | Yes | WP09 T044-T045 | O1 |
| FR-005 owned-plan-setup | Yes | WP09 T046 | O2 |
| FR-006 owned-wp-context | Yes | WP04 T016-T018; WP05; WP08 T038-T040 | O3 |
| FR-007 stale-copy-never-wins | Yes | WP04, WP08 T041, WP09 T045, WP11, WP13 T074, WP19 T104 | All 5 reads covered |
| FR-008 next-crosses-tasks-implement | Yes | WP11 T061; WP19 T102; WP12 (xfail removal) | O5 |
| FR-009 owned-runtime-reads | Yes | WP11 T057, T060; WP12 T068 | |
| FR-010 owned-review-base | Yes | WP12 T064, T065, T068 | |
| FR-011 owned-workspace-kind | Yes | WP05 T022-T025; WP04 | |
| FR-012 coordination-probe-hardening | Yes | WP11 T057, T062 | O8 |
| FR-013 owned-status-under-worktrees | Yes | WP06 T028-T031; WP13 | |
| FR-014 coordination-protections-unchanged | Yes | WP06 T027, T031 | ratchet |
| FR-015 atomic-finalize | Yes | WP13 T070-T073 | O6 |
| FR-016 governance-reads-at-owned-create | Yes | WP10 T052-T054 | O7 |
| FR-017 charter-authoring-stays-at-root | Yes | WP10 T054 | ratchet |
| FR-018 typed-refusal-unsupported-actions | Yes | WP09 T047 | See I2 |
| FR-019 per-checkout-lookup-isolation | Yes | WP05 T022, T024 | |
| FR-020 invalid-owned-path-refused | Yes | WP02 T007, T009; WP08; WP09 T049 | See I1, U2 |
| FR-021 validated-flagless-adoption | Yes | WP02 T009; WP08 T038, T040 | O10 |
| FR-022 existing-owned-commands-unchanged | Yes | WP14 T079; WP15 T083; WP16 T088; WP17; WP19 T102 | In-process lanes_with_coord twin |
| FR-023 topology-per-command | Yes | WP02 T006; WP04 T019; WP11; WP19 T102 | See U1 |
| FR-024 glossary-and-adrs | Yes | WP03 T012-T015 | |
| FR-025 unified-review-base | Yes | WP12 T064, T065, T069 | |
| FR-026 complexity-ceiling-restored | Yes | WP10 T050-T051; WP14 T075-T076 | |
| NFR-001 repository-root-untouched | Yes | WP02 T011 (r_snapshot); WP18 T098 | See I3 |
| NFR-002 validation-count | Yes | WP07 T037; WP16 T089; WP18 T099 | |
| NFR-003 command-latency | Yes | WP18 T099 | nightly `performance` |
| NFR-004 typed-errors | Yes | WP01 S7 (OwnedRefusalCode); WP08 T039; WP09 T049 | See I1, I2 |
| NFR-005 quality-gates | Yes | every WP DoD; WP18 T100 | |
| NFR-006 cross-platform-identity | Yes | WP01 T003 | ci-windows.yml filter |
| US3-AS6 fact-not-re-derived (scenario) | No | none | C1 |

**Charter Alignment Issues:** none found. The following were checked:
- ATDD-first (C-011) is honoured through red-first commits in every code WP. WP01 uses construction and self-mutation reds. WP14 has a characterisation commit plus the FR-003 committing-path red. WP03 is `planning_artifact` and runs the docs gates.
- The terminology canon holds: no `--feature` usage appears in the artifacts, and the canonical checkout terms are used.
- Single canonical authority holds: G1-G3, with C-002 kept in `surface_resolver`.
- Layer rules hold: the carrier sits in `mission_runtime`, with 0 new edges.
- `NO_FULL_HEAVY_SUITES_IN_MISSION` holds: targeted gates plus `make test-fast`.

**Unmapped Tasks:** none. T056 and T059 are retired ids, intentionally mapped to WP19 T101/T103.

**Structural checks (pass):**
- owned_files overlap across the 19 WPs: 0.
- The dependency graph and the 10 parallel waves are consistent: each WP's dependencies land in an earlier wave.
- WP frontmatter `requirement_refs` and `subtasks` match tasks.md.
- Every WP that adds `TRANSITIONAL(WP18)` markers declares an exact marker budget in its DoD.
- prerequisites: `valid: true`.

**Issue-matrix heads-up (non-gating, #3469):** the artifacts cite #3449, #5026, #4252, #4867 and #5277 as the mission's input issues. They also cite #4250, #5100, #5099, #3443, #3474, #4959, #4980, #5259 and #4785 as out-of-scope items or dependencies. Record out-of-scope and dependency citations as context-only or `not-applicable` rows at approval time.

**Metrics:**

- Total Requirements: 32 (26 FR + 6 NFR); 8 constraints tracked separately (C-004, C-006 and C-007 are enforced through prose, by design)
- Total Tasks: 103 active subtasks (T001-T105 minus retired T056, T059) across 19 WPs
- Coverage: 100% (32/32 requirements have ≥1 task). 1 acceptance scenario (US3-AS6) is uncovered.
- Ambiguity Count: 1
- Duplication Count: 0
- Critical Issues Count: 0

### Next Actions

- No CRITICAL or HIGH findings. The verdict is ready, and implementation may proceed.
- Fix U2 before WP09 T049 starts; otherwise WP09 hits its own "stop and escalate" rule. The smallest fix is a missing-path case in WP02's minter tests, plus the code in data-model.md.
- Decide U1 (LANES under owned `next`) before WP19 lands, because it is a user-visible behaviour change.
- The remaining findings are wording and definition alignment and can be folded in one editing pass:
  - I1 and I2: spec NFR-004 / US1-AS3, and WP09 T049 step 2.
  - C1: add a row to WP19 T102 or WP18 T098.
  - I3, I4, A1: spec prose.
