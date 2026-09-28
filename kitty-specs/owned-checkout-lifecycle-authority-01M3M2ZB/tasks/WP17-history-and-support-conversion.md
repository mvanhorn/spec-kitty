---
work_package_id: "WP17"
subtasks:
  - "T090"
  - "T091"
  - "T092"
  - "T093"
  - "T094"
title: "History and support module conversion"
task_type: "implement"
phase: "Phase 4 - Conversion sweep"
execution_mode: "code_change"
owned_files:
  - "src/specify_cli/tasks/issue_matrix.py"
  - "src/specify_cli/review/cycle.py"
  - "src/specify_cli/consolidation/baseline.py"
  - "src/specify_cli/git/commit_helpers.py"
  - "src/specify_cli/missions/_read_path_resolver.py"
  - "src/specify_cli/migration/runtime_state_cutover.py"
  - "src/specify_cli/migration/backfill_runtime_state.py"
authoritative_surface: "src/specify_cli/"
create_intent: []
dependencies:
  - "WP04"
  - "WP07"
requirement_refs:
  - "FR-001"
  - "FR-022"
agent_profile: "python-pedro"
role: "implementer"
agent: "claude"
model: "claude-sonnet-5"
assignee: ""
shell_pid: ""
history:
  - at: "2026-09-28T15:00:00Z"
    actor: "system"
    action: "Prompt generated via /spec-kitty.tasks"
---

# Work Package Prompt: WP17 – History and support module conversion

## ⚡ Do This First: Load Agent Profile

Use the `/ad-hoc-profile-load` skill to load the agent profile specified in the frontmatter (or any user-defined profile), and behave according to its guidance before parsing the rest of this prompt.

- **Profile**: `python-pedro`
- **Role**: `implementer`
- **Agent/tool**: `claude`

If no profile is specified, run `spec-kitty agent profile list` and select the best match for this work package's `task_type` and `authoritative_surface`.

Then load the action-scoped governance: `spec-kitty charter context --action implement --json`. The charter (`.kittify/charter/charter.md`) is binding.

---

## ⚠️ IMPORTANT: Review Feedback

**Read this first if you are implementing this task!**

- **Has review feedback?**: Check the `review_ref` field in the event log (via `spec-kitty agent tasks status`) or the Activity Log below.
- **You must address all feedback** before your work is complete. Feedback items are your implementation TODO list.
- **Report progress**: As you address each feedback item, update the Activity Log explaining what you changed.

---

## Review Feedback

*[If this WP was returned from review, the reviewer feedback reference appears in the Activity Log below or in the status event log.]*

---

## Markdown Formatting

Wrap HTML/XML tags in backticks: `` `<div>` ``, `` `<script>` ``
Use language identifiers in code blocks: ````python`, ````bash`

---

## Objectives & Success Criteria

This is the support-module tail of IC-10b: the issue matrix, the review cycle, the consolidation baseline, the commit helper, the read-path resolver and the two runtime-state migrations. It is a **behaviour-preserving signature migration**; the proof is each module's existing tests passing unchanged plus `mypy --strict`.

Done means:

1. The seven owned files contain **zero** occurrences of the identifier `effective_root`, zero `effective_root_kwargs`, zero `OwnedMission` references, and zero legacy attribute reads on the fact (`.root`, `.primary`, `.directory`, `.slug`, `.target`).
2. Every converted function takes `owned: OwnedCheckout | None = None` (keyword-only, default `None`) where it took `effective_root: Path | None` or `owned: OwnedMission | None`; `safe_commit`'s dead `effective_root` parameter is deleted, not converted (T093).
3. `create_rejected_review_cycle` (complexity 12) is campsite-extracted before its signature changes and stays ≤ 15, like every function you touch.
4. Every existing test of these modules passes; the only test edit allowed is the single out-of-map re-point in T092 (explained there).
5. ruff check, ruff format, `mypy --strict` clean on the touched files; `make test-fast` green.

## Context & Constraints

- **Mission docs**: `kitty-specs/owned-checkout-lifecycle-authority-01M3M2ZB/` → `spec.md` (FR-001, FR-022), `plan.md` (Staging Strategy; IC-10b lists exactly these modules; Campsite list names `create_rejected_review_cycle` at 12), `research.md` R-03, `contracts/architectural-gate.md` (G4/G5), `contracts/owned-checkout-carrier.md` §3, `data-model.md`, `occurrence_map.yaml`.
- **Landed prerequisites** (verify shapes on your lane base):
  - WP01 `from mission_runtime import OwnedCheckout` (public import only; MR-1/MR-2). Canonical fields `repository_root`, `owned_root`, `mission_dir`, `mission_slug`, `topology`, `target_branch`; the legacy properties are deleted by WP18, so never read them.
  - WP04 `placement_seam(..., owned=)` (dual keyword until WP18) and the converted `mission_runtime.resolve_artifact_surface` (verify: if it still takes only `effective_root`, use the bridge rule).
  - WP07 `agent_tasks_ports.MissionHandle(owned=)`, `coordination/write_seam.write_artifact(owned=)`, `coordination/commit_router` and `status/transition_pipeline` conversions.
- **Callee rule.** For each call out of these files: callee takes `owned` → `owned=owned`; callee is a dual-keyword seam → `owned=owned` (never the legacy keyword); callee still takes only `effective_root` → bridging keyword `effective_root=owned.owned_root if owned else None` as a plain keyword, logged in the Activity Log for WP18/T097.
- **Caller rule (cross-WP call sites).** Converting a public function breaks callers that pass the old keyword. On the planning base:

  | Callee (this WP) | Caller passing the keyword | Caller's WP | Dependency of WP17? |
  |---|---|---|---|
  | `scaffold_issue_matrix` | `cli/commands/agent/mission_finalize.py:922-930` (`**({"effective_root": owned.root} if owned else {})`) | WP13 | no |
  | `create_rejected_review_cycle` | `cli/commands/agent/tasks_verdict_persistence.py:901, 936` | WP16 | no |
  | `verify_pr_merge_evidence` | `cli/commands/accept.py:906` (`**scope`) | WP14 | no |
  | `record_pr_merge_baseline_for_mission` | `cli/commands/accept.py:398-402` | WP14 | no |
  | `stamp_accept_cutover` | `cli/commands/accept.py:346` (already passes `owned`) | WP14 | no (type-only change, no call break) |
  | `resolve_subtasks_gate_dir` | `status/transition_pipeline.py:102` | WP07 | yes |
  | `safe_commit` | `coordination/commit_router.py:438`, `coordination/transaction.py:836` | WP07, WP06 | yes (WP06 via WP07) |

  For callers whose WP is a dependency (already merged), change the call expression to `owned=...` (or delete the keyword, for `safe_commit`) as a **one-line out-of-map edit**, with the rationale "WP17 converted the callee; caller already holds the fact" in the commit message and Activity Log. For callers whose WP is **not** a dependency, check your lane base with `git log --oneline`: if that WP has merged, make the same one-line out-of-map edit; if not, do **not** touch the caller file (a parallel lane owns it) — convert everything else, leave that public signature for last, and ask the orchestrator to sequence (record it in the Activity Log). Callers that pass no keyword (`issue_verdict.py`, `issue_matrix_migration.py`, `tasks_materialization.py:234`, `migrate_cmd.py`, `consolidation/executor.py`, `consolidation/ordering.py`, `_cutover_doctor.py`, `upgrade/migrations/m_zz_runtime_state_backfill.py`, `status/cutover_eligibility.py`) are unaffected.
- **Behaviour preservation.** The owned arms and the non-owned arms of each function stay as they are; only the parameter's type and the forwarding keyword change. `serialized_keys`/`logs_telemetry` are `do_not_change` (issue-matrix JSON, review-cycle frontmatter, `meta.json` fields, `pr_merge_evidence`, error messages such as "Owned issue matrix write failed.").
- **Terminology (C-008)**: update docstring sentences that name the removed parameter to "owned checkout"; no "feature", no bare "primary" as a checkout alias in new prose (existing `PRIMARY` partition vocabulary stays).
- **No suppressions**; complexity ≤ 15.

## Branch Strategy

- **Strategy**: Planning artifacts were generated on `claude/sleepy-hamilton-5lelee`; completed changes must merge back into `claude/sleepy-hamilton-5lelee`.
- **Planning base branch**: `claude/sleepy-hamilton-5lelee`
- **Merge target branch**: `claude/sleepy-hamilton-5lelee`
- **Execution lane**: assigned in `kitty-specs/owned-checkout-lifecycle-authority-01M3M2ZB/lanes.json` by `spec-kitty agent mission finalize-tasks`. Start with `spec-kitty implement WP17` and use only the workspace path it prints.

> These fields are populated automatically by `spec-kitty agent mission finalize-tasks`.
> Do NOT change them manually unless you are certain the branch topology has changed.

## Occurrence census (planning base `df1588860`)

```bash
F="src/specify_cli/tasks/issue_matrix.py src/specify_cli/review/cycle.py src/specify_cli/consolidation/baseline.py \
   src/specify_cli/git/commit_helpers.py src/specify_cli/missions/_read_path_resolver.py \
   src/specify_cli/migration/runtime_state_cutover.py src/specify_cli/migration/backfill_runtime_state.py"
grep -nw "effective_root\|effective_root_kwargs\|OwnedMission" $F
grep -nE "owned\.(root|primary|directory|slug|target)\b" $F
```

Planning-base identifier lines: `issue_matrix.py` 8, `review/cycle.py` 15, `consolidation/baseline.py` 17, `git/commit_helpers.py` 2, `missions/_read_path_resolver.py` 3, `migration/runtime_state_cutover.py` 10, `migration/backfill_runtime_state.py` 5. The end state is empty apart from logged bridges.

## Subtasks & Detailed Guidance

### Subtask T090 – Convert `tasks/issue_matrix.py`

- **Purpose**: the issue-matrix writer and its finalize-time scaffold thread the owned root into the write seam and the placement seam.
- **Steps**:
  1. Import (`:51`): replace `from specify_cli.core.owned_mission import effective_root_kwargs` with `OwnedCheckout` from `mission_runtime`.
  2. `write_issue_matrix(..., effective_root)` (`:358`) → `owned: OwnedCheckout | None = None`; `:393-404` `write_artifact(..., effective_root=effective_root)` (the keyword is on `:403`) → `owned=owned` (WP07 `write_seam`, callee rule).
  3. `scaffold_issue_matrix(..., effective_root)` (`:445`) → `owned`; `:478` `if effective_root is not None:` → `if owned is not None:`; `:481` `placement_seam(repo_root, mission_slug, effective_root=effective_root)` → `owned=owned`; `:495-503` `write_issue_matrix(**effective_root_kwargs(effective_root))` → `owned=owned`; `:511` `if effective_root is not None: raise RuntimeError(...)` → `if owned is not None:` (same message).
  4. Apply the caller rule for `mission_finalize.py:922-930` (WP13).
- **Files**: `src/specify_cli/tasks/issue_matrix.py`.
- **Parallel?**: `[P]`.
- **Validation checklist**:
  - [ ] `tests/specify_cli/tasks/test_issue_matrix_scaffold.py`, `test_issue_matrix_structured.py`, `test_issue_ref_url_provenance.py`, `tests/mission_runtime/test_issue_matrix_content_source.py`, `tests/mission_runtime/test_issue_matrix_ref_read.py`, `tests/specify_cli/cli/commands/review/test_issue_matrix_partition.py`, `test_zero_reference_not_applicable.py`, `tests/integration/test_issue_verdict_concurrent_lock.py`, `tests/integration/test_issue_verdict_coord_legacy_md_preservation.py`, `tests/policy/test_issue_matrix_cross_site_consistency.py`, `tests/tasks/test_issue_reference_classification.py` pass unchanged.
  - [ ] `tests/integration/test_explicit_checkout_commands.py::test_finalize_issue_matrix_uses_owned_writer` passes unchanged.
- **Edge cases**: the non-owned arm uses `coord_read_dir_for(...) or feature_dir`, the owned arm the placement seam's `ISSUE_MATRIX` read dir; keep both. The idempotent early return (existing JSON or legacy `.md`) must fire before any write in both arms.

### Subtask T091 – Campsite plus conversion: `review/cycle.py`

- **Purpose**: the review-cycle writer is on the owned review path (`move-task --to planned` with feedback, the rejection cycle). `create_rejected_review_cycle` is at complexity 12 and listed in the plan's campsite set; tidy it first, then convert.
- **Steps**:
  1. **Campsite commit first** (behaviour-preserving, no signature change): in `create_rejected_review_cycle` (`:1125-1290`) the expression `effective_root or main_repo_root` is computed in three functions (`_commit_review_cycle_artifact` `:705`, `_adopt_or_allocate_review_cycle_locked` `:1010`, `create_rejected_review_cycle` `:1179`). Extract one private helper, for example `_operation_root(main_repo_root, owned)`, and any sub-block needed to bring `create_rejected_review_cycle` comfortably under 15 (the sub-artifact directory resolution and the persistence-outcome construction are natural seams). Commit: `refactor(review): campsite create_rejected_review_cycle before owned conversion (WP17/T091)`.
  2. **Conversion commit**:

     | Line | Function | Change |
     |---|---|---|
     | 179 | `_review_cycle_wp_dir(..., effective_root)` | param → `owned`; `:281` `placement_seam(..., effective_root=effective_root)` → `owned=owned` |
     | 683 | `_commit_review_cycle_artifact(..., effective_root)` | param → `owned`; `:699` `MissionHandle(effective_root=...)` → `owned=owned` (WP07); `:702-705` operation root via the helper; `:717-718` `placement_seam(effective_root=)` → `owned=owned` |
     | 1001 | `_adopt_or_allocate_review_cycle_locked(..., effective_root)` | param → `owned`; `:1010-1012` operation root + `placement_seam` |
     | 1125 | `create_rejected_review_cycle(..., effective_root)` (public) | param → `owned`; forwards at `:1180`, `:1231`, `:1246`, `:1272`; operation root `:1179, 1184, 1240, 1247, 1280` |

  3. The operation-root helper returns `owned.owned_root if owned is not None else main_repo_root`; name its return value something other than `owned_root`/`effective_root` if you bind it to a local (gate G4 bans the identifier `effective_root` everywhere in `src/`).
  4. Apply the caller rule for `tasks_verdict_persistence.py:901, 936` (WP16).
- **Files**: `src/specify_cli/review/cycle.py`.
- **Parallel?**: No internal dependency on the other subtasks.
- **Validation checklist**:
  - [ ] `uv run --frozen ruff check --isolated --select C901 --config 'lint.mccabe.max-complexity=15' src/specify_cli/review/cycle.py` is clean and `create_rejected_review_cycle` is below its planning-base 12.
  - [ ] `tests/review/test_cycle.py`, `tests/specify_cli/review/test_cycle_kind_flip.py`, `tests/review/test_artifacts.py`, `tests/review/test_arbiter.py`, `tests/review/test_verdict_status_lock_bound.py`, `tests/review/test_verdict_save_performance.py`, `tests/mission_runtime/test_coord_read_seam_callers.py`, `tests/agent/test_workflow_review_cycle_pointer.py` pass unchanged.
  - [ ] `tests/architectural/test_single_mission_surface_resolver.py` still passes (it pins the review-cycle sub-artifact routing).
- **Edge cases**: the commit retry loop in `_commit_review_cycle_artifact` (`attempt` counter) must keep its exact retry semantics. `evidence_ref` is computed relative to the operation root; for owned runs that must stay the owned checkout so evidence paths remain relative to P.

### Subtask T092 – Convert `consolidation/baseline.py`

- **Purpose**: `accept --mode pr --merge-commit` records the PR merge baseline from the owned checkout's own `meta.json`; that read and the write leg are threaded through `effective_root` today.
- **Steps**:
  1. Convert the five functions; each `effective_root` parameter becomes `owned: OwnedCheckout | None = None`:

     | Line | Function | Forwards / reads |
     |---|---|---|
     | 447 | `_resolve_pr_target_ref` | `:476` `resolve_primary_meta_dir(..., effective_root=effective_root)` |
     | 497 | `verify_pr_merge_evidence` (public) | `:617` `_resolve_pr_target_ref(..., effective_root=effective_root)` |
     | 724 | `_record_pr_merge_baseline` | `:759` `verify_pr_merge_evidence(effective_root=)`; `:765` read |
     | 792 | `resolve_primary_meta_dir` (public) | `:804` lazy import of `effective_root_kwargs`; `:806` `placement_seam(**effective_root_kwargs(effective_root))` → `placement_seam(..., owned=owned)` |
     | 814 | `record_pr_merge_baseline_for_mission` (public) | `:833, 834, 841` forwards |

  2. Delete the lazy `effective_root_kwargs` import at `:804`. Update the docstrings at `:466` and `:828-831` to describe `owned`.
  3. Apply the caller rule for `accept.py:398-402` and `accept.py:906` (WP14).
  4. **Out-of-map test re-point (the one test edit this WP makes).** `tests/consolidation/test_pr_merge_baseline.py:842-894` (`test_verify_effective_root_resolves_declared_target_from_owned_checkout`) calls `verify_pr_merge_evidence(..., effective_root=worktree_root)` directly. No WP owns that file, and the call cannot survive the signature change. Re-point exactly that one call to `owned=<fact>`, building the fact with the WP02 test helper for minting a fact in tests (the checkout in that test is a bare `git worktree add` without an ownership claim, so the production validator would refuse it; never call `OwnedCheckout._mint`, gate G3). Keep the test name, the docstring's intent, and both assertions unchanged. Rationale for the commit message and Activity Log: "WP17: verify_pr_merge_evidence now takes the validated fact; the one direct unit call is re-pointed; assertions unchanged (FR-022)."
- **Files**: `src/specify_cli/consolidation/baseline.py`; `tests/consolidation/test_pr_merge_baseline.py` (one call, out-of-map).
- **Parallel?**: `[P]`.
- **Validation checklist**:
  - [ ] `tests/consolidation/test_pr_merge_baseline.py` (all tests), `tests/consolidation/test_baseline_committed_meta_decode.py`, `tests/specify_cli/cli/commands/review/test_dead_code_baseline.py`, `tests/specify_cli/cli/commands/test_accept_merge_commit.py`, `tests/cli/commands/test_merge_status_commit.py`, `tests/consolidation/test_merge_done_recording.py` pass.
  - [ ] `git diff -- tests/consolidation/test_pr_merge_baseline.py` touches only the one call (plus the fact construction lines).
- **Edge cases**: the bare (non-owned) call in the same test must still raise `PrMergeEvidenceError` for the bogus repository-root target branch — that proves the owned arm reads P's `meta.json`, which is the whole point of the arm.

### Subtask T093 – Convert `git/commit_helpers.py`, `missions/_read_path_resolver.py`

- **Purpose**: two small, isolated sites. `safe_commit` accepts an `effective_root` it immediately discards (`del effective_root` at `commit_helpers.py:1091`, "Compatibility-only routing hint after retirement of the ambient sync emitter"), so the right conversion is deletion. `resolve_subtasks_gate_dir` is on the status-transition subtask gate.
- **Steps**:
  1. `git/commit_helpers.py`: delete the `effective_root: Path | None = None` parameter (`:1003`) and the `del effective_root` line plus its comment (`:1090-1091`). Do not add an `owned` parameter: nothing in `safe_commit` uses the owned checkout (FR-001 bans bare owned roots; it does not require dead parameters).
  2. Remove the keyword from its two callers (both in merged prerequisite WPs; one-line out-of-map edits each, with rationale "dead parameter deleted in `safe_commit` (WP17/T093)"):
     - `coordination/commit_router.py:438` (`**effective_root_kwargs(effective_root)` on the planning base; WP07 may already have changed it — delete whatever owned-root keyword it now passes to `safe_commit`);
     - `coordination/transaction.py:836` (`effective_root=self.worktree_root if self._primary_root is not None else None`).
     Confirm with `grep -rn -A10 "safe_commit(" src --include=*.py | grep effective_root` that no other caller passes it (30 call sites on the planning base; only those two do). Also check tests: `grep -rn "safe_commit(" tests --include=*.py | xargs -r grep -l effective_root` (none on the planning base).
  3. `missions/_read_path_resolver.py`: `resolve_subtasks_gate_dir(..., effective_root)` (`:1485`) → `owned: OwnedCheckout | None = None`; `:1520` `if effective_root is not None:` → `if owned is not None:`; `:1523-1524` `placement_seam(primary_root, mission_slug, effective_root=effective_root)` → `owned=owned`. Caller `status/transition_pipeline.py:102` (WP07, merged): switch its keyword to `owned=` in a one-line out-of-map edit if WP07 left a bridge there.
- **Files**: `src/specify_cli/git/commit_helpers.py`, `src/specify_cli/missions/_read_path_resolver.py`; out-of-map one-liners in `src/specify_cli/coordination/commit_router.py`, `src/specify_cli/coordination/transaction.py`, `src/specify_cli/status/transition_pipeline.py`.
- **Parallel?**: `[P]`.
- **Validation checklist**:
  - [ ] `uv run --frozen mypy --strict src/specify_cli/git/commit_helpers.py` (it is in the `make typecheck` target) plus the multi-file command in Test Strategy.
  - [ ] `tests/git_ops/test_safe_commit_helper_integration.py`, `tests/git_ops/test_safe_commit_commit_failure_classification.py`, `tests/integration/git/test_safe_commit_backstop.py`, `tests/git/test_guard_capability_regression.py`, `tests/git/test_protection_preserved.py`, `tests/specify_cli/status/test_subtasks_gate_dir_seam.py`, `tests/specify_cli/status/test_infer_subtasks_primary.py`, `tests/specify_cli/missions/test_read_path_resolver_redundant_cast_gate.py`, `tests/task_utils/test_set_scalar_retired.py` pass unchanged.
  - [ ] `tests/architectural/test_no_read_side_bypass.py` passes (it references `resolve_subtasks_gate_dir`).
- **Edge cases**: `_read_path_resolver.py` is referenced by 110 test files; most do not exercise `resolve_subtasks_gate_dir`. Run the targeted list above plus `make test-fast`, not all 110.

### Subtask T094 – Convert `migration/runtime_state_cutover.py`, `migration/backfill_runtime_state.py`

- **Purpose**: the birth-cutover and backfill already receive the validated value object (`owned: OwnedMission | None`, #3866). Conversion here is a type change plus canonical attribute names plus removing the `{"owned": owned}` splats and one owned-root keyword.
- **Steps**:
  1. `runtime_state_cutover.py`: the `TYPE_CHECKING` import at `:50-51` becomes `from mission_runtime import OwnedCheckout` (keep it under `TYPE_CHECKING` if runtime does not need it). Retype `owned: OwnedMission | None` → `OwnedCheckout | None` in `_seed_phase` (`:129`), `_verify_phase` (`:149`), `_resolve_primary_home_or_degrade` (`:199`), `_flip_target` (`:253`), `_already_at_snapshot_authority` (`:271`), `_flip_phase` (`:291`), `cutover_mission` (`:347`), `stamp_accept_cutover` (`:486`).
  2. `_resolve_primary_home_or_degrade` `:226-230`: `resolve_artifact_surface(owned.primary, owned.slug, MissionArtifactKind.PRIMARY_METADATA, effective_root=owned.root)` → `resolve_artifact_surface(owned.repository_root, owned.mission_slug, MissionArtifactKind.PRIMARY_METADATA, owned=owned)` (callee rule for `resolve_artifact_surface`).
  3. `cutover_mission` `:406`: delete `scope = {"owned": owned} if owned is not None else {}` and pass `owned=owned` at `:413, 421, 435, 448` (every callee already defaults `owned=None`, so the plain keyword is equivalent).
  4. `backfill_runtime_state.py`: `TYPE_CHECKING` import at `:84-85`; retype `_runtime_feature_dir` (`:1433`), `backfill_runtime_state` (`:1453`), `_invocation_write_refusal` (`:2081`), `verify_backfill` (`:2129`). `_runtime_feature_dir` `:1445-1447`: `owned.directory` → `owned.mission_dir` (keep the exact-directory guard and its `OWNED_MISSION_PATH_REFUSED` code). `verify_backfill` `:2169-2170`: delete the `scope` splat, pass `owned=owned`.
  5. Update the #3866 comments (`backfill_runtime_state.py:1438-1442`, `runtime_state_cutover.py` equivalents) so they name the fact, not a re-resolution that no longer exists.
- **Files**: `src/specify_cli/migration/runtime_state_cutover.py`, `src/specify_cli/migration/backfill_runtime_state.py`.
- **Parallel?**: `[P]`.
- **Validation checklist**:
  - [ ] `tests/specify_cli/migration/test_runtime_state_cutover.py`, `test_backfill_cutover_guard.py`, `test_backfill_migration_coexistence.py`, `test_backfill_writer_locking.py`, `test_cutover_partition_decouple.py`, `test_dogfood_corpus_backfilled.py`, `tests/migration/test_birth_cutover.py`, `tests/migration/test_corpus_frontload_idempotent.py`, `tests/migration/test_runtime_feature_dir_threading.py`, `tests/integration/test_migration_backfill.py`, `tests/specify_cli/cli/commands/test_backfill_runtime_state_cli.py`, `tests/specify_cli/cli/commands/test_cutover_doctor.py`, `tests/specify_cli/cli/test_accept_birth_cutover.py`, `tests/specify_cli/cli/commands/test_accept_birth_cutover_seam.py`, `tests/specify_cli/upgrade/test_runtime_state_backfill_migration.py` pass unchanged.
  - [ ] `tests/integration/test_explicit_checkout_commands.py::test_accept_commits_metadata_matrix_and_cutover_only_in_owned` passes unchanged.
- **Edge cases**: `tests/migration/test_runtime_feature_dir_threading.py:16-38` constructs `OwnedMission(...)` directly and asserts `_runtime_feature_dir(owned.directory, owned) == owned.directory`. You do not own that file. If WP02's transitional alias still lets it construct an object exposing `mission_dir`, it keeps passing; if it goes red only because the alias object lacks `mission_dir`, report it as a WP02 alias gap in the Activity Log (do not edit the file) — WP18/T097 re-points every remaining `OwnedMission` test construction when the alias is deleted.

## Test Strategy

Targeted (existing files unchanged except the one T092 re-point):

```bash
uv run --frozen pytest -q \
  tests/specify_cli/tasks/test_issue_matrix_scaffold.py tests/specify_cli/tasks/test_issue_matrix_structured.py \
  tests/specify_cli/tasks/test_issue_ref_url_provenance.py tests/mission_runtime/test_issue_matrix_content_source.py \
  tests/mission_runtime/test_issue_matrix_ref_read.py tests/specify_cli/cli/commands/review/test_issue_matrix_partition.py \
  tests/specify_cli/cli/commands/review/test_zero_reference_not_applicable.py tests/integration/test_issue_verdict_concurrent_lock.py \
  tests/integration/test_issue_verdict_coord_legacy_md_preservation.py tests/policy/test_issue_matrix_cross_site_consistency.py \
  tests/tasks/test_issue_reference_classification.py \
  tests/review/test_cycle.py tests/specify_cli/review/test_cycle_kind_flip.py tests/review/test_artifacts.py \
  tests/review/test_arbiter.py tests/review/test_verdict_status_lock_bound.py tests/review/test_verdict_save_performance.py \
  tests/mission_runtime/test_coord_read_seam_callers.py tests/agent/test_workflow_review_cycle_pointer.py \
  tests/consolidation/test_pr_merge_baseline.py tests/consolidation/test_baseline_committed_meta_decode.py \
  tests/specify_cli/cli/commands/review/test_dead_code_baseline.py tests/specify_cli/cli/commands/test_accept_merge_commit.py \
  tests/cli/commands/test_merge_status_commit.py tests/consolidation/test_merge_done_recording.py \
  tests/git_ops/test_safe_commit_helper_integration.py tests/git_ops/test_safe_commit_commit_failure_classification.py \
  tests/integration/git/test_safe_commit_backstop.py tests/git/test_guard_capability_regression.py tests/git/test_protection_preserved.py \
  tests/specify_cli/status/test_subtasks_gate_dir_seam.py tests/specify_cli/status/test_infer_subtasks_primary.py \
  tests/specify_cli/missions/test_read_path_resolver_redundant_cast_gate.py tests/task_utils/test_set_scalar_retired.py \
  tests/specify_cli/migration/ tests/migration/test_birth_cutover.py tests/migration/test_corpus_frontload_idempotent.py \
  tests/migration/test_runtime_feature_dir_threading.py tests/integration/test_migration_backfill.py \
  tests/specify_cli/cli/commands/test_backfill_runtime_state_cli.py tests/specify_cli/cli/commands/test_cutover_doctor.py \
  tests/specify_cli/cli/test_accept_birth_cutover.py tests/specify_cli/cli/commands/test_accept_birth_cutover_seam.py \
  tests/specify_cli/upgrade/test_runtime_state_backfill_migration.py \
  tests/specify_cli/coordination/test_commit_router_partition.py tests/specify_cli/coordination/test_status_transition.py \
  tests/integration/test_explicit_checkout_commands.py
```

To find any further test for a touched module: `grep -rl "<module_name>" tests --include=*.py` (planning-base counts: `issue_matrix` 16, `review.cycle` 155, `consolidation.baseline` 24, `commit_helpers` 38, `_read_path_resolver` 110, `runtime_state_cutover` 20, `backfill_runtime_state` 28). Run the ones that exercise the functions you changed; `make test-fast` covers the rest of the fast tier.

Specific architectural gates implicated (files, not the directory):

```bash
uv run --frozen pytest -q \
  tests/architectural/test_single_mission_surface_resolver.py \
  tests/architectural/test_no_read_side_bypass.py \
  tests/architectural/test_lifted_cli_accept_birth_cutover.py \
  tests/architectural/test_issue_matrix_json_migration_completeness.py \
  tests/architectural/test_owned_checkout_gate_selftest.py
```

Baseline and quality gates:

```bash
make test-fast
uv run --frozen ruff check $F tests/consolidation/test_pr_merge_baseline.py
uv run --frozen ruff format --check $F tests/consolidation/test_pr_merge_baseline.py
uv run --frozen mypy --strict $F src/specify_cli/coordination/commit_router.py src/specify_cli/coordination/transaction.py \
  src/specify_cli/status/transition_pipeline.py src/specify_cli/cli/commands/accept.py \
  src/specify_cli/cli/commands/agent/tasks_verdict_persistence.py src/specify_cli/cli/commands/agent/mission_finalize.py
```

(`$F` is the seven-file list from the census.) Callers and callees go into **one** mypy invocation: `follow_imports = "skip"` for `specify_cli.*` makes cross-module keyword mismatches invisible otherwise. Record commands and counts in the Activity Log and the PR's *Tests run* section.

## Risks & Mitigations

- **Parallel-lane caller conflicts** (WP13, WP14, WP16 call into this WP's modules and are not dependencies). Mitigation: caller rule in Context; flagged to the planner as missing dependency edges.
- **Dead-parameter deletion touching two other WPs' files** (`safe_commit`). Mitigation: both owners (WP06, WP07) are merged prerequisites; each edit is one line with rationale.
- **Review-cycle retry semantics** during the campsite extraction. Mitigation: separate refactor commit; `tests/review/test_cycle.py` and the verdict durability tests run after each commit.
- **Alias-constructed test objects** (`OwnedMission(...)` in unowned test files). Mitigation: report, do not edit; WP18 owns the final re-point.

## Review Guidance

- Re-run the census; only logged bridges may remain.
- Check that the T091 campsite commit is refactor-only and precedes the conversion.
- Check every out-of-map edit is one line (or one call) with a written rationale, and that none landed in a caller file whose WP had not merged.
- Check the `tests/consolidation/test_pr_merge_baseline.py` diff is limited to the one call and keeps both assertions.
- Confirm the multi-file mypy command ran clean.

## Activity Log

> **CRITICAL**: Activity log entries MUST be in chronological order (oldest first, newest last).

### How to Add Activity Log Entries

**When adding an entry**:

1. Scroll to the bottom of this Activity Log section
2. **APPEND the new entry at the END** (do NOT prepend or insert in middle)
3. Use exact format: `- YYYY-MM-DDTHH:MM:SSZ – agent_id – <action>`
4. Timestamp MUST be current time in UTC (check with `date -u "+%Y-%m-%dT%H:%M:%SZ"`)
5. Agent ID should identify who made the change (claude-sonnet-4-5, codex, etc.)

**Format**:

```
- YYYY-MM-DDTHH:MM:SSZ – <agent_id> – <brief action description>
```

**Initial entry**:

- 2026-09-28T15:00:00Z – system – Prompt created.

---

### Updating Status

Status is managed via `status.events.jsonl`. Use `spec-kitty agent tasks move-task <WPID> --to <status>` to change WP status.
