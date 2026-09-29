---
work_package_id: WP07
title: Status transition pipeline takes the fact
dependencies:
- WP04
- WP06
requirement_refs:
- FR-003
- NFR-002
planning_base_branch: claude/sleepy-hamilton-5lelee
merge_target_branch: claude/sleepy-hamilton-5lelee
branch_strategy: Planning artifacts for this mission were generated on claude/sleepy-hamilton-5lelee. During /spec-kitty.implement this WP may branch from a dependency-specific base, but completed changes must merge back into claude/sleepy-hamilton-5lelee unless the human explicitly redirects the landing branch.
subtasks:
- T032
- T033
- T034
- T035
- T036
- T037
phase: Phase 2 - Seams
history:
- at: '2026-09-28T15:00:00Z'
  actor: system
  action: Prompt generated via /spec-kitty.tasks
agent_profile: python-pedro
authoritative_surface: src/specify_cli/coordination/status_transition.py
create_intent:
- tests/status/test_transition_request_owned.py
execution_mode: code_change
model: claude-sonnet-5
owned_files:
- src/specify_cli/coordination/status_transition.py
- src/specify_cli/coordination/commit_router.py
- src/specify_cli/coordination/write_seam.py
- src/specify_cli/status/models.py
- src/specify_cli/status/transition_pipeline.py
- src/specify_cli/status/emit.py
- src/specify_cli/status/bootstrap.py
- src/specify_cli/agent_tasks_ports.py
- tests/specify_cli/coordination/test_status_transition.py
- tests/status/test_transition_request_owned.py
role: implementer
tags: []
task_type: implement
tracker_refs: []
---

# Work Package Prompt: WP07 – Status transition pipeline takes the fact

## ⚡ Do This First: Load Agent Profile

Use the `/ad-hoc-profile-load` skill to load the agent profile specified in the frontmatter, and behave according to its guidance before parsing the rest of this prompt.

- **Profile**: `python-pedro`
- **Role**: `implementer`
- **Agent/tool**: `claude`

Then load the action-scoped governance with `spec-kitty charter context --action implement --json`. Read `.kittify/charter/charter.md` if this session has not read it yet.

---

## ⚠️ IMPORTANT: Review Feedback

**Read this first if you are implementing this task!**

- **Has review feedback?** Check the `review_ref` field in the event log (via `spec-kitty agent tasks status`) or the Activity Log below.
- **You must address all feedback** before your work is complete. Feedback items are your implementation TODO list.
- **Report progress.** As you address each feedback item, update the Activity Log explaining what you changed.

---

## Review Feedback

*[If this WP was returned from review, the reviewer feedback reference appears in the Activity Log below or in the status event log.]*

---

## Markdown Formatting

Wrap HTML/XML tags in backticks: `` `<div>` ``, `` `<script>` ``
Use language identifiers in code blocks: ````python`,````bash`

---

## Objectives & Success Criteria

The status pipeline carries the **validated ownership fact** end to end and never re-derives ownership when it holds one (FR-003). Its readers collapse three things into one `owned: OwnedCheckout | None`:

- the parallel `effective_root: Path | None`;
- the `owned_mission: OwnedMission | None`;
- the transaction's `primary_root`.

Done means:

1. **`TransitionRequest.owned`.**
   - `TransitionRequest` has `owned: OwnedCheckout | None`.
   - The legacy `effective_root` and `owned_mission` fields remain as the **transitional dual keyword**: `TransitionRequest` is one of the six shared seams, and the legacy fields are deleted in WP18.
   - Every pipeline reader reads only the collapsed fact.
2. **No re-validation with a fact.** When a fact is present, nothing in `coordination/status_transition.py`, `status/transition_pipeline.py`, `status/emit.py`, `status/bootstrap.py`, `coordination/commit_router.py`, `coordination/write_seam.py` or `agent_tasks_ports.py` calls `resolve_owned_mission` or `resolve_ownership_claim`. The re-validation at `status_transition.py:897-918` survives **only** as the clearly marked legacy-keyword branch that WP18 deletes.
3. **Transaction converted.**
   - `BookkeepingTransaction.acquire` takes `owned=` (this is the out-of-map completion of WP06 T030; see Context).
   - An owned transaction appends through `EventLogWriteContract.primary_checkout_append(feature_dir, owned=fact)`.
   - `_TransactionIdentity.primary_root` is gone; `identity.owned` carries it.
4. **Ports carry the fact.** `MissionHandle.owned` is added, and the ports route placement and commits through it.
5. **NFR-002 / FR-003 through the CLI.** `agent mission finalize-tasks --owned-checkout P` performs **exactly one** ownership validation. This is **red on the base**, where the bootstrap read re-validates (see T037). `agent tasks move-task --owned-checkout P` performs exactly one as well.
6. **Nothing changes for non-owned missions.** All status-transition, bootstrap, pipeline, commit-router and write-seam tests stay green. The existing owned `move-task`/`mark-status`/finalize tests (FR-022) stay green without edits.

**Mission-wide DoD (post-tasks squad):**
- `grep -rn "bridging: WP07 converts" src` is empty: every bridging call site marked for this WP is converted, including sites in other WPs' files (declared out-of-map edits).
- Every bridging call site this WP adds carries `# bridging: WP<n> converts`, naming the WP that converts it (never a free-form comment, never no marker).
- `TRANSITIONAL(WP18)` markers added by this WP (exact list; any deviation is amended here in the same PR, and WP18 T096 fails on unlisted markers): **12** markers: `status/models.py` 3 (`TransitionRequest.effective_root`, `.owned_mission`, `has_owned_root_only`); `coordination/status_transition.py` 3 (the legacy re-validation branch, `read_events_transactional(effective_root=)`, `emit_inner_state_changed_transactional(effective_root=)`); `status/bootstrap.py` 2 (`effective_root`, `owned_mission`); `coordination/commit_router.py` 1 (`commit_for_mission`); `coordination/write_seam.py` 1 (`write_artifact`); `agent_tasks_ports.py` 2 (`MissionHandle.effective_root`, the `_placement_for` legacy arm).
- Red-first proofs are **commits**: the red test commit precedes its fix commit, and the reviewer verifies the order in `git log`.
- Error codes are imported from `OwnedRefusalCode` (WP01); no `OWNED_*` string literal is repeated in `src/` or new tests (Sonar S1192).

## Context & Constraints

**Mission documents.** All live under `kitty-specs/owned-checkout-lifecycle-authority-01M3M2ZB/`:
- `spec.md`: FR-003, NFR-002, FR-022, C-001.
- `plan.md`: IC-04, Staging Strategy, Test Layout.
- `research.md`: R-03, R-16.
- `contracts/owned-checkout-carrier.md`: §2 "Once per command" and §7 (`TransitionRequest` is a named consumer).
- `contracts/architectural-gate.md`: G2's floor names `status_transition.py:917`, and G4 bans `effective_root`.
- `occurrence_map.yaml`.

**Prerequisites.**
- **WP04** exposes `owned=` on `placement_seam` / `mission_context_for` / `resolve_action_context`, and possibly on `resolve_placement_only`.
  - Before calling a `mission_runtime` function, grep its current signature.
  - If it takes `owned=`, pass the fact.
  - If it still takes only `effective_root=`, pass the **bridging expression** `effective_root=owned.owned_root if owned else None`, marked `# bridging: WP<n> converts` with the WP that owns the callee. This is not a new signature.
- **WP06** added:
  - `_TransactionIdentity.owned` (set in `_identity_for_request`);
  - the owned arm of `_read_contract_from_transaction_target`;
  - `EventLogReadContract.primary_checkout(..., owned=)` and `EventLogWriteContract.primary_checkout_append(..., owned=)`;
  - the `transaction.py:644` segment swap.
- **WP02** makes `resolve_owned_mission` return the fact and turns `OwnedMission` into a transitional legacy factory function (marked `TRANSITIONAL(WP18)`) that mints an `OwnedCheckout`; the legacy attribute names resolve through `OwnedCheckout`'s `TRANSITIONAL(WP18)` properties. **New code uses canonical fields only:** `repository_root`, `owned_root`, `mission_dir`, `mission_slug`, `target_branch`, `topology`.

**Staging rules (binding).**
- Conversion is top-down.
- Transitional surfaces are the six shared seams plus every other function marked TRANSITIONAL(WP18). The six shared seams keep a dual keyword until WP18; `TransitionRequest` is one of them.
- **Additional transitional dual keywords introduced here (declared design extension).** These public callees are called with a bare owned root by files converted in later WPs:

  | Callee | Legacy caller(s) with the owner WP |
  |---|---|
  | `commit_for_mission` | `mission_finalize.py:2752` (WP13), `spec_commit_cmd.py:207` (WP16) |
  | `write_seam.write_artifact` | `tasks/issue_matrix.py:403` (WP17), `acceptance/matrix.py:533` (WP15) |
  | `bootstrap_canonical_state` | `mission_finalize.py:194-204` (WP13) |
  | `read_events_transactional` | `tasks_move_task.py:2458,2524` (WP16), `tasks_finalize_validation.py:105` (WP13) |
  | `emit_inner_state_changed_transactional` | `tasks_mark_status.py:415`, `tasks_move_task.py:3122` (WP16) |
  | `MissionHandle` | `tasks_move_task.py:550,2888,2925`, `tasks_mark_status.py:232` (WP16), `review/cycle.py:699` (WP17) |

  The rules for these callees:
  - Each gains `owned: OwnedCheckout | None = None` as the authoritative parameter and keeps its legacy `effective_root` / `owned_mission` parameter with the **same lifetime as the six shared seams' dual keywords**.
  - Converting those callers here would be a large out-of-map edit into five WPs' files, which is why the legacy parameter stays.
  - Mark every legacy parameter with `# TRANSITIONAL(WP18): delete with the dual keywords`.
  - Record this list in the Activity Log as a cross-check. Under the plan's convention these are "other functions marked TRANSITIONAL(WP18)"; WP18 T096 deletes exactly what `grep -rn "TRANSITIONAL(WP18)" src tests` finds.
- The transitional `OwnedMission` legacy factory function and the legacy attribute names live until WP18. Do not add new reads of `.root`/`.primary`/`.directory`/`.slug`/`.target`.

**Declared out-of-map edits.** WP06 owns the first two; the last two have no owner in the pinned map. Declare each in its commit body with a one-line rationale.
- `src/specify_cli/coordination/transaction.py`: convert `acquire(effective_root=)` to `acquire(owned=)` and make the owned append label honest.
  - The sole caller `_acquire_status_transaction` gets the fact only in this WP.
  - The acquire-shape test is WP07-owned.
  - WP06 T030 explicitly deferred this, and WP06 is complete because it is a dependency.
- `tests/specify_cli/coordination/test_owned_status_read_contract.py`: only if a WP06 test pins `acquire` kwargs. None is expected; grep first.
- `tests/status/test_transition_pipeline.py`: its fake resolver takes `effective_root=` (`:75-76`, `:172-176`), and its test targets `transition_pipeline.py`, which this WP owns.
- `tests/status/test_bootstrap.py`: `:549-556` asserts `request.effective_root == owned.root`; the test targets `bootstrap.py`, which this WP owns.

**Occurrence map.**
- `code_symbols` / `tests_fixtures`: rename.
- `serialized_keys`: do not change. This covers status event fields and `CommitReceipt`, and `TransitionRequest` is not serialized.
- `logs_telemetry`: do not change. The `OWNED_TRANSACTION_UNAVAILABLE`, `OWNED_MISSION_PATH_REFUSED` and `OWNED_TOPOLOGY_UNSUPPORTED` strings stay exact.

**Complexity.** The probe on HEAD `df1588860` flags `commit_router._commit_partition_group` at **15** and `_stage_artifacts_in_coord_worktree` at **14** (`.venv/bin/ruff check --isolated --select C901 --config 'lint.mccabe.max-complexity=11' src/specify_cli/coordination/commit_router.py`).
- `_commit_partition_group` is touched, so a **campsite extraction commit comes first**.
- `status_transition.py`, `write_seam.py`, `status/*.py` and `agent_tasks_ports.py` show no function at ≥12.
- Re-run the probe after each conversion. A function that would reach ≥12 gets a behaviour-preserving extraction first.

**Terminology.** New prose uses "repository root checkout" and "owned checkout". Keep the existing public names (`primary_checkout_append`, `commit_to_primary_target`).

## Branch Strategy

- **Strategy**: Planning artifacts were generated on `claude/sleepy-hamilton-5lelee`; completed changes must merge back into `claude/sleepy-hamilton-5lelee`.
- **Planning base branch**: `claude/sleepy-hamilton-5lelee`
- **Merge target branch**: `claude/sleepy-hamilton-5lelee`
- **Execution lane**: allocated by `spec-kitty agent mission finalize-tasks` in `kitty-specs/owned-checkout-lifecycle-authority-01M3M2ZB/lanes.json`, which had not been generated when this prompt was written. Use `spec-kitty implement WP07` and the workspace it resolves. Never hand-construct the path.

> These fields are populated automatically by `spec-kitty agent mission finalize-tasks`.
> Do NOT change them manually unless you are certain the branch topology has changed.

**Commit order**:
1. T037 red CLI-count test.
2. T035 campsite extraction.
3. T032.
4. T033 (with the `transaction.py` out-of-map hunk). T037 finalize count turns green here or at T034.
5. T034.
6. T035 conversion.
7. T036.

## Subtasks & Detailed Guidance

### Subtask T032 – `TransitionRequest.owned` (collapses `effective_root` and `owned_mission`; dual keyword)

- **Purpose**: one field carries the validated fact through the status pipeline (contract §7). The two legacy fields keep not-yet-converted callers green until WP18.
- **Steps**:
  1. **Red first** in `tests/status/test_transition_request_owned.py` (markers `unit` + `fast`). In tests, mint facts with `OwnedCheckout._mint(...)`, as WP01's `tests/mission_runtime/test_owned_checkout.py` does (contract §1 allows a `tests/` helper, and G3 scans `src/` only), or take them from the WP02 minter on a real fixture. Never call `OwnedCheckout(...)`, which raises `TypeError` by design. Cover these cases:
     - `TransitionRequest(owned=fact).owned_fact() is fact`.
     - `TransitionRequest(owned_mission=fact).owned_fact() is fact` (legacy alias collapses).
     - Mismatch: `owned=f1, owned_mission=f2` with differing checkout or slug raises `TypeError` at `owned_fact()`.
     - `effective_root=P` alone: `owned_fact()` is `None`, and `has_owned_root_only()` is `True`. This legacy shape is the only one that may still validate.
     - A request built positionally or with no owned fields is unchanged.
  2. Implement in `src/specify_cli/status/models.py` (`TransitionRequest`, `:858-908`):
     - Add `owned: OwnedCheckout | None = None` **after** `owned_mission`, so positional construction is unchanged. Import it under `TYPE_CHECKING` from `mission_runtime`, replacing the `OwnedMission` import at `:22-26`, which keeps the cold-import guarantee (#3866 comment).
     - Add `owned_fact(self) -> OwnedCheckout | None`. It returns `self.owned` or `self.owned_mission`, and raises `TypeError` when both are set and are not the same object and not equal. Do the collapse **at read time, not in `__post_init__`**: existing tests and callers mutate `request.effective_root` / `request.owned_mission` after construction (for example `tests/specify_cli/coordination/test_status_transition.py:1008-1011`, `:1240-1241`).
     - Mark `effective_root` and `owned_mission` with `# TRANSITIONAL(WP18)` and update the class docstring.
  2a. Sketch (read-time collapse; no identity checks here):
     ```python
     def owned_fact(self) -> OwnedCheckout | None:
         """The validated ownership fact; the legacy ``owned_mission`` alias collapses into it (WP18 deletes the alias)."""
         if self.owned is not None and self.owned_mission is not None:
             if self.owned is not self.owned_mission and self.owned != self.owned_mission:
                 raise TypeError("TransitionRequest carries two different ownership facts")
         return self.owned if self.owned is not None else self.owned_mission

     def has_owned_root_only(self) -> bool:
         """TRANSITIONAL(WP18): a legacy caller passed a bare owned root without the fact."""
         return self.effective_root is not None and self.owned_fact() is None
     ```
  3. Do **not** add identity checks (checkout or slug vs `effective_root`) here. They stay at the identity seam (T033), preserving today's `OWNED_MISSION_PATH_REFUSED` behaviour and location.
- **Files**: `src/specify_cli/status/models.py`, `tests/status/test_transition_request_owned.py`.
- **Parallel?**: no. T033 and T034 build on it.
- **Validation checklist**:
  - [ ] The new tests are committed red (`AttributeError`/`TypeError` on the new API), then green.
  - [ ] `tests/status/` stays green, including the `TransitionRequest` users in `test_transition_pipeline.py`.
  - [ ] mypy is clean on `status/models.py` (the dataclass remains non-frozen).
- **Edge cases**:
  - `dataclasses.replace(request, ...)` copies all three fields. `owned_fact()` stays consistent.
  - Equality on the frozen `OwnedCheckout`: use `is` first, then `==`, so two facts minted separately for the same checkout do not raise.

### Subtask T033 – `status_transition`: consume the fact, remove re-validation, owned read contract

- **Purpose**: the transactional status layer stops deriving ownership. It takes the fact from the request and hands it to the transaction and the read contract (FR-003, contract §2).
- **Steps**:
  1. **Red first**: add tests to `tests/specify_cli/coordination/test_status_transition.py`.
     - (a) `_identity_for_request(TransitionRequest(owned=fact, ...))`:
       - it builds the identity from the fact: `feature_dir == fact.mission_dir`, `repo_root == fact.owned_root`, `identity.owned is fact`;
       - it runs with a **tripwire** on `specify_cli.core.checkout_ownership.resolve_ownership_claim` and `specify_cli.core.owned_mission.resolve_owned_mission` that raises `AssertionError` if called;
       - it never calls `_repo_root_for_feature`.
     - (b) Mismatch: `owned=fact` with `effective_root` set to another path, or with `mission_slug` naming another mission, raises `ActionContextError("OWNED_MISSION_PATH_REFUSED")`. This keeps the #3866 guard.
     - (c) `read_events_transactional(feature_dir=..., mission_slug=..., owned=fact)` returns P's events under the same tripwire.
     - (d) The acquire-shape parity test (`:1046-1083`) rewritten to `single["owned"] is fact` and `single["repo_root"] == fact.repository_root`.
     - Rewrite the #3866 tests (`:1198-1310`) from `OwnedMission(...)` construction and `request.owned_mission` onto `owned=`. Keep one test that still threads `owned_mission=` to pin the dual-keyword collapse until WP18.
  2. Implement in `src/specify_cli/coordination/status_transition.py`:
     - Replace the `TYPE_CHECKING` `OwnedMission` import (`:84-86`) with `from mission_runtime import OwnedCheckout`.
     - `_identity_for_request` (`:884-988`):
       - `fact = request.owned_fact()`.
       - If `fact` is set: apply the identity guard (the `request.effective_root` checkout and `mission_slug` must match `fact.owned_root` and `fact.mission_slug`), then `feature_dir, repo_root = fact.mission_dir, fact.owned_root`.
       - `elif request.effective_root is not None`: this is the **legacy keyword branch**. Keep the existing `resolve_owned_mission` call (it yields a fact) inside a block commented `# TRANSITIONAL(WP18): legacy effective_root-only callers; G2 floor offender`.
       - Else: the non-owned path is unchanged.
       - The destination-ref block (`:966-975`) uses `fact`: pass `owned=` or the bridging expression to `resolve_placement_only` (see Context).
     - `_TransactionIdentity` (`:92-101`): delete `primary_root`. `identity.owned` (added by WP06) is the only owned carrier. Replace every `identity.primary_root` read with `identity.owned.repository_root if identity.owned else None`, or restructure (`_resolve_transaction_entry` `:1003`, `_acquire_status_transaction` `:1028-1037`, `emit_inner_state_changed_transactional` `:1654-1700`).
     - `_acquire_status_transaction`: `BookkeepingTransaction.acquire(repo_root=identity.owned.repository_root if identity.owned else identity.repo_root, ..., owned=identity.owned)`.
     - `read_events_transactional` (`:1345-1364`): add `owned: OwnedCheckout | None = None` and keep the legacy `effective_root` (`# TRANSITIONAL(WP18)`). Build the internal `TransitionRequest(owned=owned, effective_root=effective_root, ...)`.
     - `emit_inner_state_changed_transactional` (`:1585-1706`): add `owned=` and keep the legacy `effective_root` and `owned_mission`. Use `request.owned_fact()` / `identity.owned` for the owned checks at `:1654` and `:1670`, and the `BookkeepingWorktreeMissing` re-raise at `:1698`.
  3. Out-of-map `src/specify_cli/coordination/transaction.py` (WP06-owned; declare it):
     - `acquire(..., owned: OwnedCheckout | None = None)` replaces `effective_root` (`:249`). The lock root is `owned.owned_root if owned else repo_root` (`:291-293`).
     - `_acquire_locked(..., owned=)` replaces `primary_root=` (`:299-312`, `:547`). `txn._owned` replaces `_primary_root` (`:195`), and `commit()` reads `self._owned.repository_root` (`:830`).
     - The `safe_commit(effective_root=...)` keyword (`:836`) stays as a bridging expression marked `# bridging: WP17 converts`, because `git/commit_helpers.py` converts in WP17: `effective_root=self.worktree_root if self._owned is not None else None`.
     - `append_events` (`:643`): `if self._owned is not None: primary_checkout_append(self.feature_dir, owned=self._owned)`. The legacy and coordination arms are unchanged.
- **Files**: `src/specify_cli/coordination/status_transition.py`, `tests/specify_cli/coordination/test_status_transition.py`, and the declared hunk in `src/specify_cli/coordination/transaction.py`.
- **Parallel?**: no.
- **Validation checklist**:
  - [ ] `grep -n "resolve_owned_mission" src/specify_cli/coordination/status_transition.py` finds exactly one call site, inside the `TRANSITIONAL(WP18)` legacy branch.
  - [ ] `grep -n "primary_root" src/specify_cli/coordination/status_transition.py src/specify_cli/coordination/transaction.py` finds only comments or unrelated identifiers (`commit_to_primary_target` is fine).
  - [ ] The carried WP06 test `test_finalize_and_read_owned_mission_below_worktrees` stays green, and the owned append now uses `primary_checkout_append(owned=)`. Assert this in one T033 test by recording the contract passed to `append_event_stream_log`.
  - [ ] `tests/specify_cli/coordination/` and `tests/coordination/` stay green.
- **Edge cases**:
  - `OWNED_TRANSACTION_UNAVAILABLE` must still fire for an owned request whose mission has no transactional metadata. Both doors must keep it (`test_batch_door_refuses_owned_mission_without_transaction_like_single`).
  - The lock file must stay at the same path for owned and non-owned missions, because NFR-001 snapshots the lock root.
  - `_read_contract_from_transaction_target`'s owned arm, added by WP06, must stay the first branch.

### Subtask T034 – Pipeline readers: `transition_pipeline`, `emit`, `bootstrap`

- **Purpose**: the pure pipeline and its two shells read the collapsed fact, and bootstrap stops re-validating on every read. This is the concrete source of the extra validation in finalize (T037).
- **Steps**:
  1. **Red first**:
     - `tests/status/test_transition_request_owned.py`: `bootstrap_canonical_state(P_mission_dir, slug, owned=fact, repo_root=fact.repository_root)` seeds WP rows with **zero** calls to `resolve_ownership_claim` (tripwire).
     - `prepare_transition` with `owned=fact` on `in_progress → for_review` passes the fact to the injected resolver as `owned=`.
  2. `src/specify_cli/status/transition_pipeline.py`:
     - `_default_resolve_subtasks_dir(..., owned: OwnedCheckout | None = None)` (`:84-104`) bridges into `resolve_subtasks_gate_dir(..., effective_root=owned.owned_root if owned else None)`, because `missions/_read_path_resolver.py` converts in WP17. Mark the call `# bridging: WP17 converts`.
     - `_infer_review_gates` (`:131-176`) passes `owned=request.owned_fact()` at `:163`.
     - Update `SubtasksDirResolver` (`:64`) and its docstring to say the resolver receives `owned=`.
  3. `src/specify_cli/status/emit.py`: rename `_flat_subtasks_dir_resolver`'s dropped parameter (`:690`) to `owned: OwnedCheckout | None = None`. Keep the `noqa: ARG001` with its D-1 rationale; it is still deliberately dropped. Update the docstring (`:694-699`). There is no other `effective_root` in `emit.py` (`grep -n effective_root src/specify_cli/status/emit.py`).
  4. `src/specify_cli/status/bootstrap.py` `bootstrap_canonical_state` (`:92-198`):
     - Add `owned: OwnedCheckout | None = None`. Keep `effective_root` and `owned_mission` as `# TRANSITIONAL(WP18)`; the owned caller `mission_finalize.py:194-204` converts in WP13.
     - Collapse: `fact = owned or owned_mission`. If `effective_root` is also set, it must equal `fact.owned_root`; raise `TypeError` otherwise.
     - The read at `:151-155` becomes `read_events_transactional(..., repo_root=repo_root, owned=fact)` whenever `fact` is set. This removes the per-read re-validation.
     - The seeding requests (`:170-186`) become `TransitionRequest(..., owned=fact)`.
     - Replace the `TYPE_CHECKING` `OwnedMission` import (`:28`).
  5. Declared out-of-map test updates:
     - `tests/status/test_transition_pipeline.py:75-76,172-176`: the fake resolver keyword becomes `owned`, and the test passes a fact.
     - `tests/status/test_bootstrap.py:549-556`: assert `request.owned_fact() is owned` instead of `request.effective_root == owned.root`.
- **Files**: `src/specify_cli/status/transition_pipeline.py`, `src/specify_cli/status/emit.py`, `src/specify_cli/status/bootstrap.py`, and the tests above.
- **Parallel?**: after T032/T033.
- **Validation checklist**:
  - [ ] `grep -n "effective_root" src/specify_cli/status/*.py` finds only the `TRANSITIONAL(WP18)` legacy parameters in `bootstrap.py` and the `TransitionRequest` fields in `models.py`.
  - [ ] `tests/status/` is fully green.
  - [ ] The T037 finalize count is now 1.
- **Edge cases**:
  - The flat shell must keep **not** threading the root to the subtasks resolver (D-1 parity). Do not "fix" that here.
  - `bootstrap_canonical_state` with `dry_run=True` must also read without re-validating.

### Subtask T035 – `commit_router` / `write_seam` conversions

- **Purpose**: the canonical commit and write seams carry the fact internally, so every owned artifact commit decides placement from `owned` and never from a bare path.
- **Steps**:
  1. **Campsite first** (its own commit, behaviour-preserving): `_commit_partition_group` in `src/specify_cli/coordination/commit_router.py` (`:278-470`, complexity **15**). Extract:
     - the placement/`use_coord` decision (`:298-321`) into `_resolve_group_placement(...)`;
     - the empty-commit-paths / wrong-surface classification (`:388-422`) into `_classify_no_commit_paths(...)`.
     Target ≤ 11. Add focused unit tests for each helper in `tests/coordination/test_commit_router.py`; note that as an out-of-map test edit, or use the WP-owned `tests/status/test_transition_request_owned.py` if you prefer to stay in-map. Run `tests/coordination/test_commit_router*.py` and `tests/specify_cli/coordination/test_commit_router_*.py` before and after with identical results.
  2. **Red first**: in `tests/status/test_transition_request_owned.py`, test that `commit_for_mission(R, slug, (P/plan.md,), msg, policy, kind=PLAN, owned=fact)`:
     - commits on P's branch;
     - makes zero claim calls (tripwire);
     - returns `status == "committed"`.
     Add the same-fixture control `effective_root=P` (legacy) with an identical result.
  3. Convert `commit_router.py`:
     - `commit_for_mission(..., owned: OwnedCheckout | None = None, effective_root: Path | None = None)` (`:187-197`). The legacy keyword is `# TRANSITIONAL(WP18)`. Internally, normalise once: if `effective_root` is set without `owned`, keep today's path-based behaviour by bridging to the internals' legacy branch.
     - The private internals `_commit_partition_group`, the new helpers and `_group_files_by_partition` take `owned` only when the caller holds a fact.
     - Where only a bare root arrives (legacy callers), keep one private bridging variable `owned_root: Path | None` confined to `commit_for_mission`. Do not introduce a new bare-path **parameter** anywhere.
     - Placement calls (`:298-301`, `:242`) use `owned=` or the bridging expression, according to WP04's signature.
     - `safe_commit(...)` (`:432-439`): `**effective_root_kwargs(owned.owned_root if owned else legacy_root)`, marked `# bridging: WP17 converts`. This is the bridging expression, because `commit_helpers` converts in WP17. Drop the module-level `effective_root_kwargs` import (`:44`) if it is no longer needed. Otherwise keep it with a `# bridging` comment.
  4. Convert `write_seam.py` the same way:
     - `write_artifact(..., owned=None, effective_root=None)` (`:462-472`, legacy keyword transitional);
     - `_probe_write_target(..., owned=...)` (`:239-268`, private, fact only);
     - the owned gates at `:533` and `:540` read `owned is None`;
     - the `commit_for_mission(...)` call at `:549-559` passes `owned=`.
- **Files**: `src/specify_cli/coordination/commit_router.py`, `src/specify_cli/coordination/write_seam.py`, `tests/status/test_transition_request_owned.py` (plus any declared out-of-map test edits for the campsite helpers).
- **Parallel?**: yes with T036 after T032, but not with T033 (both touch the transaction flow; keep the commits separate).
- **Validation checklist**:
  - [ ] `.venv/bin/ruff check --isolated --select C901 --config 'lint.mccabe.max-complexity=11' src/specify_cli/coordination/commit_router.py src/specify_cli/coordination/write_seam.py` flags nothing you touched. `_stage_artifacts_in_coord_worktree` (14) is untouched unless you edit it; if you do, extract first.
  - [ ] `tests/coordination/test_commit_router.py`, `test_commit_router_fail_loud.py`, `test_commit_router_layering.py`, `test_write_seam_adoption.py`, `tests/specify_cli/coordination/test_commit_router_partition*.py`, `test_commit_router_placement.py` and `test_write_seam_thunk.py` are green.
  - [ ] The existing owned spec-commit and finalize tests (`tests/integration/test_explicit_checkout_commands.py`) are green unchanged. They pass the legacy keyword.
- **Edge cases**:
  - An owned commit must never split into partition groups. Today `groups = [(kind, files)]` when an owned root is set (`:242`); preserve that.
  - The owned write gate skip (`write_seam.py:529-535`, owned single-branch writes never route through coordination) must hold for both keywords.
  - `commit_for_mission` is patched in many tests via `tasks_command_adapters._seam_commit_for_mission` (`*args, **kwargs` passthrough). Adding a keyword does not break those patches; confirm with `grep -rn "commit_for_mission" tests/ | wc -l` and a run of the patched test files.

### Subtask T036 – `agent_tasks_ports.MissionHandle.owned`

- **Purpose**: the task-command ports carry the fact, so `move-task` / `mark-status` placement and commits read P through the fact rather than a bare root (IC-04).
- **Steps**:
  1. **Red first**, in `tests/status/test_transition_request_owned.py`:
     - `RealFsReader().planning_read_dir(MissionHandle(R, slug, owned=fact), kind=TASKS_INDEX)` returns `fact.mission_dir`;
     - `RealCoordCommitRouter().feature_write_dir(MissionHandle(..., owned=fact))` returns P's STATUS dir;
     - `commit_artifact` passes `owned=fact` to the injected `commit_fn` (record the kwargs).
     Add the same-fixture legacy control, `MissionHandle(R, slug, effective_root=P)`, with the same results.
  2. Implement in `src/specify_cli/agent_tasks_ports.py`:
     - `MissionHandle` (`:68-80`): add `owned: OwnedCheckout | None = None` after `effective_root`, which stays as `# TRANSITIONAL(WP18)` because its constructors live in WP16/WP17 files.
     - Add `__post_init__` validation: if both are set, `effective_root.resolve() == owned.owned_root`, else `TypeError`. The dataclass is frozen and immutable after construction, so `__post_init__` is safe here.
     - Replace the three `**({"effective_root": ...})` dict splats (`:249-252`, `:256-259`, `:278-281`) with one private helper, `_placement_for(mission) -> PlacementSeam`, that calls `placement_seam(mission.repo_root, mission.mission_slug, owned=mission.owned)` when `owned` is set and falls back to the legacy keyword otherwise. This removes three `"effective_root"` dict-key sites (G4).
     - `feature_write_dir` (`:336-346`) and `commit_artifact` (`:356-390`): pass `owned=mission.owned` to `commit_fn`, and the legacy `effective_root_kwargs` only when `owned` is `None`.
       - C-001 byte-parity of the mock call shape matters for `test_tasks_cli_contract_coord.py`: a non-owned handle must produce **no** extra keyword.
       - Pass `owned=` only when set, using a small `_owned_kwargs(mission)` typed helper.
  3. Sketch of the private helpers (the names are suggestions):
     ```python
     def _placement_for(mission: MissionHandle) -> PlacementSeam:
         if mission.owned is not None:
             return placement_seam(mission.repo_root, mission.mission_slug, owned=mission.owned)
         # TRANSITIONAL(WP18): legacy bare-root handles (review/cycle.py, WP17)
         return placement_seam(mission.repo_root, mission.mission_slug, **effective_root_kwargs(mission.effective_root))
     ```
     The three `RealFsReader` methods and `feature_write_dir` each call `_placement_for(mission).read_dir(kind)`, which removes the triplicated splat.
- **Files**: `src/specify_cli/agent_tasks_ports.py`, `tests/status/test_transition_request_owned.py`.
- **Parallel?**: yes, after T032.
- **Validation checklist**:
  - [ ] `tests/specify_cli/cli/commands/agent/test_tasks_cli_contract_coord.py`, `test_tasks_cli_contract.py`, `test_owned_checkout_move_task.py` and `tests/integration/test_owned_checkout_mark_status.py` are green unchanged (FR-022).
  - [ ] `grep -n '"effective_root"' src/specify_cli/agent_tasks_ports.py` finds nothing.
  - [ ] mypy is clean; the `follow_imports = "skip"` note (`:241-244`) still applies to the annotated locals.
- **Edge cases**:
  - Non-owned handles must keep calling `resolve_feature_dir_for_mission` in `feature_write_dir` (`:342-345`), byte-identical.
  - The `thread_target_branch` split (`:357-381`) must keep its two distinct call shapes.

### Subtask T037 – NFR-002 exactly-one-validation test (`move-task`, `finalize-tasks`)

- **Purpose**: pin FR-003 / NFR-002 through the real CLI entry point. One owned command invocation performs exactly one ownership validation, and 0 or ≥2 fails.
- **Steps**:
  1. **First commit of the WP (red)**. In `tests/status/test_transition_request_owned.py`, add an `integration` + `git_repo` test class. It uses the WP02 fixture `owned_checkouts`, imported explicitly with `from tests.integration.conftest import owned_checkouts, r_snapshot` and re-exported through `__all__`, because conftest fixtures in `tests/integration/` are not visible from `tests/status/`. The class:
     - clears the workspace caches first (`specify_cli.workspace.context.clear_workspace_resolution_caches` or the name WP05 settled on) (plan: Test Layout);
     - wraps `specify_cli.core.checkout_ownership.resolve_ownership_claim` in a counting wrapper that delegates to the original. `resolve_owned_mission` imports it lazily inside the function (`owned_mission.py:88`), so patching the module attribute intercepts it;
     - invokes through `CliRunner` exactly as a user would.
  2. Cases:
     - **`finalize-tasks`**: `agent mission finalize-tasks --mission <slug> --owned-checkout P --json` gives `count == 1`. This is **red on the base**: `mission_finalize.py:3327` validates once, then `bootstrap_canonical_state` reads through `read_events_transactional(..., effective_root=P)` without the fact (`bootstrap.py:151-155`), which re-validates at `status_transition.py:917`. Record the base count in the Activity Log (expected ≥ 2).
     - **`move-task`**: `agent tasks move-task WP01 --to claimed --mission <slug> --owned-checkout P --json` after a finalize gives `count == 1`. Run it on the base and record the count. If the base is already 1 (the seeding path threads `owned_mission`), this case is the `[ratchet]` guard and the finalize case carries the red-first evidence. Say so in the test docstring.
     - **Positive control on the same fixture**: the same commands without `--owned-checkout`, against a non-owned mission in R, give `count == 0`. This proves the counter actually intercepts: a counter that never fires would make the owned assertion vacuous.
     - `r_snapshot` is unchanged around each owned invocation (NFR-001).
  2a. Counting-wrapper sketch:
     ```python
     @pytest.fixture
     def claim_counter(monkeypatch):
         from specify_cli.core import checkout_ownership
         calls: list[Path | None] = []
         original = checkout_ownership.resolve_ownership_claim
         def _counting(claimed_checkout, *, resolved_primary):
             calls.append(claimed_checkout)
             return original(claimed_checkout, resolved_primary=resolved_primary)
         monkeypatch.setattr(checkout_ownership, "resolve_ownership_claim", _counting)
         return calls
     ```
     Request `claim_counter` **after** the fixture that prepares the mission, or reset `calls.clear()` right before the invocation.
  3. Keep the counting wrapper local to this test module. WP16 T089 re-implements its own copy (about 10 lines) rather than importing from a test module.
- **Files**: `tests/status/test_transition_request_owned.py`.
- **Parallel?**: no. It is the first commit.
- **Validation checklist**:
  - [ ] The red run on the base is recorded with the observed count(s).
  - [ ] It is green after T033/T034 with no edits to the test.
  - [ ] The `count == 0` control is present and green on base and after.
  - [ ] No test-module import. Fixtures come only from `tests/integration/conftest.py`.
- **Edge cases**:
  - `move-task` may run the pre-review gate on `for_review` hops. Keep to the `claimed` hop, or set `SPEC_KITTY_SKIP_PRE_REVIEW_GATE=1` explicitly if you add later hops.
  - The counter must be installed **after** fixture setup (the fixture itself may validate) and **before** the invocation.
  - Different checkouts per test: the workspace caches are module-level (FR-019). Clear them per test.

## Test Strategy

**Red-first map**:

| Requirement | Red-first test | Entry point | Non-vacuity |
|---|---|---|---|
| FR-003 / NFR-002 | T037 `finalize-tasks` count == 1 | real CLI via `CliRunner` | base count ≥ 2 recorded; the same-fixture non-owned control counts 0, which proves the counter fires |
| FR-003 (pipeline) | T033 (a)/(c) and T034 tripwire tests | `_identity_for_request`, `read_events_transactional`, `bootstrap_canonical_state` | a tripwire raises if validation runs; the legacy-keyword control still validates (count 1) |
| FR-022 `[ratchet]` | existing owned tests unchanged | real CLI | must pass without edits |

**Commands** (record commands and counts in the Activity Log and the PR's *Tests run*):

```bash
.venv/bin/python -m pytest tests/status/test_transition_request_owned.py tests/specify_cli/coordination/test_status_transition.py -q
.venv/bin/python -m pytest tests/status/ tests/specify_cli/coordination/ tests/coordination/ -q     # owning subsystem dirs
.venv/bin/python -m pytest tests/integration/test_explicit_checkout_commands.py tests/integration/test_owned_checkout_mark_status.py tests/specify_cli/cli/commands/agent/test_owned_checkout_move_task.py tests/specify_cli/cli/commands/agent/test_tasks_cli_contract.py tests/specify_cli/cli/commands/agent/test_tasks_cli_contract_coord.py -q   # FR-022 consumers
.venv/bin/python -m pytest tests/architectural/test_layer_rules.py tests/architectural/test_status_module_boundary.py tests/architectural/test_status_unsafe_allowlist.py tests/architectural/test_no_dead_symbols.py -q   # implicated gates only
make test-fast
.venv/bin/ruff check <every touched src and test file>
.venv/bin/ruff format --check <every touched src and test file>
.venv/bin/mypy --strict src/specify_cli/status/models.py src/specify_cli/status/transition_pipeline.py src/specify_cli/status/emit.py src/specify_cli/status/bootstrap.py src/specify_cli/coordination/status_transition.py src/specify_cli/coordination/transaction.py src/specify_cli/coordination/commit_router.py src/specify_cli/coordination/write_seam.py src/specify_cli/agent_tasks_ports.py src/specify_cli/core/owned_mission.py src/specify_cli/git/commit_helpers.py src/specify_cli/missions/_read_path_resolver.py src/specify_cli/coordination/surface_resolver.py src/specify_cli/coordination/status_service.py
```

mypy discipline (plan: Staging Strategy): never drop `--strict`; list callers **and callees** in the same invocation (`follow_imports = "skip"` for `specify_cli.*`). If `--strict` reports pre-existing errors in these files, run the identical command on the planning base, record that count, and show it did not grow; never narrow the file list to hide them.

Do not run `make test-full` or the bare `tests/architectural/` directory. Classify unrelated reds per the `CLAUDE.md` baseline-red gotcha.

## Risks & Mitigations

- **Hidden legacy callers.** Before merging, run `grep -rn "effective_root=" src --include=*.py | grep -v "TRANSITIONAL\|bridging"` and list every remaining site in the Activity Log with its owning WP. Each must be a documented legacy caller from the Context table, a bridging expression, or outside this WP's files.
- **Mock call-shape drift (C-001).** Several tests assert exact `commit_for_mission` kwargs. Pass `owned=` only when set (T036).
- **Silent behaviour change in the legacy branch.** The legacy `effective_root` branch must remain byte-identical until WP18. Do not "simplify" it.
- **Import cycles.** `status/models.py` and `status/bootstrap.py` import `OwnedCheckout` only under `TYPE_CHECKING`, and `status_transition.py` already defers heavy imports. Run `python -c "import specify_cli.status"` in a fresh interpreter to check.

## Review Guidance

- With a fact present, nothing in the pipeline validates. Check the tripwire tests and grep for `resolve_owned_mission`.
- Every legacy parameter carries `# TRANSITIONAL(WP18)`, and the Activity Log lists them for WP18 T096. Check the list against the Context table.
- The `transaction.py` edit is the declared out-of-map completion of WP06 T030, and nothing else in that file changed.
- The campsite extraction commit precedes the `commit_router` conversion, and its helpers have focused tests.
- T037's base count is recorded, and the non-owned counter control exists.
- The implementer ran mypy on every touched source file, and it passed.
- Do not reject the transitional dual keywords before WP18.

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
- 2026-09-29T01:12:56Z – claude – shell_pid=19362 – Status pipeline now carries the collapsed owned fact (owned_fact()) end to end. Summary:
- T032: TransitionRequest.owned + owned_fact()/has_owned_root_only() (models.py, 3 TRANSITIONAL(WP18) markers).
- T033: _identity_for_request builds from the fact with zero re-validation; _TransactionIdentity.primary_root deleted (identity.owned carries it); BookkeepingTransaction.acquire/commit/append_events converted to owned= (out-of-map completion of WP06 T030, declared).
- T034: transition_pipeline/emit/bootstrap thread owned=; bootstrap_canonical_state's read no longer re-validates -- this was the observed finalize-tasks double-validation gap.
- T035: campsite split of _commit_partition_group (complexity 15->11) then commit_for_mission/write_seam gained owned=.
- T036: MissionHandle.owned + _placement_for/_owned_kwargs helpers in agent_tasks_ports.py (C-001 byte-parity preserved).
- T037: CLI-level NFR-002 proof -- finalize-tasks --owned-checkout now performs exactly ONE ownership validation (was >=2 on base); move-task recorded as [ratchet] (its own re-validation sites are WP16-owned, out of this WP's files).
Fixed a self-introduced dead field: _TransactionIdentity.primary_root was converted-away-from but not deleted; deleted in a follow-up commit.
Tests: tests/status/ tests/specify_cli/coordination/ tests/coordination/ = 1951 passed, 18 skipped. FR-022 consumers (test_owned_checkout_move_task.py, test_owned_checkout_mark_status.py, test_tasks_cli_contract*.py, test_explicit_checkout_commands.py) all green unchanged. mypy --strict on the 14 listed files: 11 pre-existing errors, 0 new (verified against the lane base). Complexity gate clean on touched functions.
Known open item: tests/architectural/test_no_dead_symbols.py::test_no_public_symbol_in_all_is_unimported goes red on this branch (OwnedMission, NEXT_OWNED_TOPOLOGIES, adopt_owned_checkout, resolve_owned_create_root, OwnedCheckoutPathRefused lose their last/only src/ import-site reference once status_transition.py's TYPE_CHECKING OwnedMission import is replaced with OwnedCheckout, per this WP's own explicit T032 instruction). CLAUDE.md lists this exact test as "judged at WP18" (pre-existing-known-red category); flagging for the reviewer/WP18 rather than silently reintroducing the OwnedMission import to paper over it.
