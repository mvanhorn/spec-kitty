---
work_package_id: "WP15"
subtasks:
  - "T080"
  - "T081"
  - "T082"
  - "T083"
title: "Acceptance package conversion"
task_type: "implement"
phase: "Phase 4 - Conversion sweep"
execution_mode: "code_change"
owned_files:
  - "src/specify_cli/acceptance/__init__.py"
  - "src/specify_cli/acceptance/execution_context.py"
  - "src/specify_cli/acceptance/gates_core.py"
  - "src/specify_cli/acceptance/matrix.py"
  - "tests/acceptance/test_gate_execution_context.py"
authoritative_surface: "src/specify_cli/acceptance/"
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

# Work Package Prompt: WP15 – Acceptance package conversion

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

This is the `acceptance/*` half of IC-10a. It is a **behaviour-preserving signature migration**: every bare owned-root parameter in the acceptance package becomes `owned: OwnedCheckout | None`, and nothing else changes.

Done means:

1. The four owned modules contain **zero** occurrences of the identifier `effective_root` (parameter, keyword, local read, `"effective_root"` key), zero `effective_root_kwargs` imports or calls, and zero `OwnedMission` references. Each converted function takes `owned: OwnedCheckout | None = None` (keyword-only, default `None`) in the position the old `effective_root` had.
2. Every existing test of the acceptance package passes **unchanged** (FR-022), and `tests/architectural/test_read_surface_placement_guard.py::test_declared_home_surface_delegates_not_reimplements` still sees exactly one delegated call.
3. The owned arm of each converted function is covered by at least one focused test in `tests/acceptance/test_gate_execution_context.py` (diff coverage ≥ 90%, NFR-005).
4. `mypy --strict`, ruff check and ruff format are clean on the touched files; `make test-fast` passes.

## Context & Constraints

- **Mission docs**: `kitty-specs/owned-checkout-lifecycle-authority-01M3M2ZB/` → `spec.md` (FR-001, FR-022), `plan.md` (Staging Strategy; IC-10a; Scale/Scope), `research.md` R-03 (the 92-parameter census and "convert all 92 … G4/G5 empty allowlist"), `data-model.md` (OwnedCheckout fields), `contracts/owned-checkout-carrier.md` §3 and §7, `contracts/architectural-gate.md` (G4/G5 are what this WP is burning down), `occurrence_map.yaml`.
- **Landed prerequisites** (verify their actual shapes on your lane base before editing):
  - WP01: `from mission_runtime import OwnedCheckout` (public import only — MR-1/MR-2 forbid `mission_runtime.owned_checkout` submodule imports). Canonical fields: `owned_root`, `repository_root`, `mission_dir`, `mission_slug`, `topology`, `target_branch`. The legacy properties (`root`, `primary`, `directory`, `slug`, `target`) are transitional and deleted by WP18 — do not read them.
  - WP04: `placement_seam(..., owned=)` (dual keyword until WP18), and whatever WP04/T021 did to `mission_runtime.resolve_artifact_surface` (`resolution.py:2320` on the planning base) and `mission_runtime.declared_read_surface` (`resolution.py:2125`).
  - WP07: `specify_cli.coordination.write_seam.write_artifact` converted to the fact (called from `matrix.py:524`).
- **Callee rule (plan: Staging Strategy).** For each call you make **out of** the acceptance package, read the callee's current signature first:
  - takes `owned` → pass `owned=owned`;
  - is one of the six dual-keyword seams → pass `owned=owned` (never the legacy keyword);
  - still takes only `effective_root` → pass the bridging expression `effective_root=owned.owned_root if owned else None` as a plain keyword and log the site in the Activity Log for WP18/T097.
- **Caller rule (cross-WP call sites).** Converting a public function breaks every caller that passes `effective_root=` to it. On the planning base those callers are **in other WPs' files**:
  - `src/specify_cli/cli/commands/accept.py` (WP14) → `collect_feature_summary` (`accept.py:679, 703` via `**scope`) and `normalize_feature_encoding` (`accept.py:695`);
  - `src/specify_cli/cli/commands/agent/mission_finalize.py:2636` (WP13) → `scaffold_acceptance_matrix(..., **({"effective_root": owned.root} if owned else {}))`.
  Neither WP13 nor WP14 is a dependency of this WP. Before converting those two public entry points, check `git log --oneline` on your lane base for the WP13/WP14 merges. If the caller WP has merged, change that one call expression to `owned=owned` as a **one-line out-of-map edit**, with the rationale "WP15 converted the callee signature; caller already holds the fact" in the commit message and Activity Log. If it has not merged, do **not** edit the caller file (a parallel lane owns it): convert every private function and leave the public function's parameter for the last commit, then ask the orchestrator to sequence it after the caller WP (and say so in the Activity Log). Callers that do not pass the keyword (`orchestrator_api/commands.py:2135`, `consolidation/executor.py`, `cli/commands/agent/acceptance_verdict.py`, `acceptance/post_consolidation.py`) are unaffected because the new keyword defaults to `None`.
- **Occurrence map**: `code_symbols` rename; `serialized_keys` do_not_change (the acceptance summary JSON and `acceptance-matrix.json` keys stay byte-identical); `logs_telemetry` do_not_change.
- **Terminology (C-008)**: new docstrings say "owned checkout" / "repository root checkout". Do not rewrite unrelated legacy docstrings; do update any docstring sentence that names the parameter you removed (for example `matrix.py:495-500`, `__init__.py:1204`).
- **No suppressions**, complexity ≤ 15 for every touched function (none of the touched acceptance functions is ≥ 12 on the planning base; keep it that way).

## Branch Strategy

- **Strategy**: Planning artifacts were generated on `claude/sleepy-hamilton-5lelee`; completed changes must merge back into `claude/sleepy-hamilton-5lelee`.
- **Planning base branch**: `claude/sleepy-hamilton-5lelee`
- **Merge target branch**: `claude/sleepy-hamilton-5lelee`
- **Execution lane**: assigned in `kitty-specs/owned-checkout-lifecycle-authority-01M3M2ZB/lanes.json` by `spec-kitty agent mission finalize-tasks`. Start with `spec-kitty implement WP15` and use only the workspace path it prints.

> These fields are populated automatically by `spec-kitty agent mission finalize-tasks`.
> Do NOT change them manually unless you are certain the branch topology has changed.

## Occurrence census (planning base `df1588860`)

Re-run at the end; it must print nothing except logged bridges:

```bash
grep -nw "effective_root\|effective_root_kwargs\|OwnedMission" \
  src/specify_cli/acceptance/__init__.py src/specify_cli/acceptance/execution_context.py \
  src/specify_cli/acceptance/gates_core.py src/specify_cli/acceptance/matrix.py
```

Totals on the planning base: 27 + 5 + 9 + 6 identifier lines; 19 parameters (11 + 2 + 4 + 2), 15 `effective_root_kwargs(...)` calls, 5 plain `effective_root=` keywords, 24 local reads, 0 `"effective_root"` dict keys, 0 `OwnedMission` references. The per-file tables are in each subtask.

## Subtasks & Detailed Guidance

### Subtask T080 – Convert `acceptance/__init__.py`

- **Purpose**: `acceptance/__init__.py` holds the accept read path (`collect_feature_summary`) and all of its private readers. It carries 11 bare owned-root parameters, the largest single cluster in the package (R-03).
- **Steps**:
  1. Replace `from specify_cli.core.owned_mission import effective_root_kwargs` (`:18`) with `from mission_runtime import OwnedCheckout` (under `TYPE_CHECKING` if the module keeps runtime imports lazy; follow the file's existing import style).
  2. Convert each function below; for each, the parameter becomes `owned: OwnedCheckout | None = None` and every forward becomes `owned=owned`:

     | Line | Function | Owned-root uses inside |
     |---|---|---|
     | 229 | `_encoding_backup_scope_prefix` | `:252` `_planning_read_dir(..., effective_root=effective_root)` |
     | 298 | `_mission_routes_through_coordination` | `:324` `if effective_root is None:` branch; `:330-334` `declared_home_surface(..., effective_root=effective_root)` |
     | 345 | `_accept_dirty_gate` | `:392` keyword; `:401-405` `_filter_coordination_residue(**effective_root_kwargs(...))` |
     | 414 | `_filter_coordination_residue` | `:432-435` `_mission_routes_through_coordination(**effective_root_kwargs(...))` |
     | 741 | `_iter_work_packages` | `:752-755` `_wp_tasks_read_dir(**effective_root_kwargs(...))` |
     | 1031 | `normalize_feature_encoding` (public) | `:1051-1054` `_planning_read_dir(**effective_root_kwargs(...))` |
     | 1138 | `_status_read_feature_dir` | `:1157` `if effective_root is not None:`; `:1160` `placement_seam(..., effective_root=effective_root)` |
     | 1185 | `_planning_read_dir` | `:1226-1229` `placement_seam(**effective_root_kwargs(...))` |
     | 1234 | `_wp_tasks_read_dir` | `:1271-1274` `placement_seam(**effective_root_kwargs(...))` |
     | 1284 | `_primary_anchor_feature_dir` | `:1319-1322` `placement_seam(**...)`; **`:1334` `resolve_mission(feature, effective_root or repo_root)`** |
     | 1388 | `collect_feature_summary` (public) | `:1396` `scope = effective_root_kwargs(effective_root)`; splats at `:1397, 1398, 1414, 1415, 1442, 1484, 1538` |

  3. Replace the `None`-checks with `owned`-checks: `if effective_root is None:` → `if owned is None:`; `if effective_root is not None:` → `if owned is not None:`. Keep the branch bodies identical apart from the keyword.
  4. `:1334` is the only site that uses the owned root as a **value**: `resolve_mission(feature, effective_root or repo_root)` becomes `resolve_mission(feature, owned.owned_root if owned is not None else repo_root)`. Do not introduce a local named `effective_root` or `owned_root: Path` to hold it (gate G4/G5 scan locals and annotations).
  5. In `collect_feature_summary`, delete the `scope` local and pass `owned=owned` explicitly at each of the seven former splat sites; `_check_lane_gates` (`:1538`) is converted in T081 — convert both in the same commit or the call breaks.
  6. Apply the **caller rule** from Context for `collect_feature_summary` and `normalize_feature_encoding` (callers in `accept.py`).
- **Files**: `src/specify_cli/acceptance/__init__.py`.
- **Parallel?**: T081 and T082 are independent files, but `collect_feature_summary` → `_check_lane_gates` crosses into `gates_core.py`, so land T080 and T081 in one commit, or T081 first.
- **Validation checklist**:
  - [ ] `grep -nw "effective_root\|effective_root_kwargs" src/specify_cli/acceptance/__init__.py` prints nothing (or only logged bridges).
  - [ ] `tests/specify_cli/acceptance/test_status_read_feature_dir_canonical_mid8.py`, `test_accept_gate_read_surface.py`, `test_accept_dirty_kitty_ops.py`, `test_trio_read_seam_migration.py` pass unchanged.
  - [ ] `tests/integration/test_explicit_checkout_commands.py -k accept` passes unchanged (owned accept reads the owned checkout's documents without writes).
- **Edge cases**: `_mission_routes_through_coordination` deliberately uses a different predicate for the owned arm (declared home surface of `ACCEPTANCE_MATRIX`) than for the non-owned arm (`resolve_topology` + `routes_through_coordination`). Keep that asymmetry; it is not a bug to "unify" here. `_status_read_feature_dir`'s owned arm returns the seam dir unconditionally, while the non-owned arm falls back to `feature_dir` when the status dir is absent — preserve both.

### Subtask T081 – Convert `acceptance/execution_context.py` and `gates_core.py`

- **Purpose**: The gate-execution context (GEC) and the lane gates are the acceptance gate's single surface authority. They must receive the fact rather than a path so the gate judges the owned checkout it was handed (GEC-1, C1).
- **Steps**:
  1. `execution_context.py`: replace the import at `:59`; convert
     - `declared_home_surface` (`:237`) — forward at `:266-268` into `mission_runtime.declared_read_surface(..., **effective_root_kwargs(effective_root))`;
     - `build_gate_execution_context` (`:318`) — forward at `:336-338` into `mission_runtime.resolve_artifact_surface(..., **effective_root_kwargs(effective_root))`.
     Both callees live in `src/mission_runtime/resolution.py` (WP04). Apply the **callee rule**: if WP04 converted them, pass `owned=owned`; otherwise pass the bridging keyword and log it.
  2. `gates_core.py`: replace the import at `:37`; convert

     | Line | Function | Owned-root uses inside |
     |---|---|---|
     | 311 | `_acceptance_gate_context` | `:345` `scope = effective_root_kwargs(...)`; `:346` `build_gate_execution_context(**scope)` |
     | 453 | `_matrix_surface_cannot_hold` | `:479` `scope = ...`; `:480` `declared_home_surface(**scope)` |
     | 640 | `_evaluate_acceptance_matrix` | `:671` `scope = ...`; splats at `:672, 678` |
     | 743 | `_check_lane_gates` | `:774-782` `_evaluate_acceptance_matrix(**effective_root_kwargs(...))` |

     Delete every `scope` local; pass `owned=owned`.
  3. Keep the `from typing import Any` import only if something else still uses it after the `scope: dict[str, Any]` annotations disappear (ruff F401 will tell you).
  4. Do not widen `build_gate_execution_context`'s contract: it stays "the ONE construction door" and must stay pure (no HEAD read). `owned` only selects the surface.
- **Files**: `src/specify_cli/acceptance/execution_context.py`, `src/specify_cli/acceptance/gates_core.py`.
- **Parallel?**: Yes with T082; coupled with T080 through `_check_lane_gates`.
- **Validation checklist**:
  - [ ] `tests/architectural/test_read_surface_placement_guard.py::test_declared_home_surface_delegates_not_reimplements` passes unchanged. Its spy forwards `**kwargs` to the real `declared_read_surface`, so whatever keyword you pass must be one the real function accepts.
  - [ ] `tests/acceptance/test_gate_execution_context.py` (all 30 tests) passes before you add anything in T083.
  - [ ] `tests/specify_cli/acceptance/test_acceptance_cores.py`, `test_issue_1834.py`, `test_issue_4974_accept_concurrent_verdict.py`, `test_negative_invariant_authoring.py`, `test_find_unchecked_tasks_canon.py` pass unchanged.
- **Edge cases**: `resolve_artifact_surface` raises `CoordinationBranchDeleted` for a deleted coordination branch (C3 fail-loud); with `owned` set on a single_branch mission that path is unreachable, but do not add a `try/except` that would swallow it for the non-owned case. `_matrix_surface_cannot_hold` must keep returning the same GEC-5 outcome type for flat and coord missions.

### Subtask T082 – Convert `acceptance/matrix.py`

- **Purpose**: The acceptance-matrix writer is the only **write** path in the package. Its owned arm decides where the matrix is committed and whether a failed write is fatal.
- **Steps**:
  1. Replace the import at `:24`.
  2. `write_and_commit_acceptance_matrix` (`:478`): the parameter becomes `owned: OwnedCheckout | None = None`; the forward at `:524-533` into `specify_cli.coordination.write_seam.write_artifact(..., effective_root=effective_root)` becomes `owned=owned` if WP07 converted `write_artifact` (it should have; `write_seam.py` is WP07-owned), otherwise the bridging keyword (log it).
  3. `scaffold_acceptance_matrix` (`:802`): the parameter becomes `owned`; `:910-918` `write_and_commit_acceptance_matrix(**effective_root_kwargs(effective_root))` becomes `owned=owned`; `:922` `if effective_root is not None: raise RuntimeError(...)` becomes `if owned is not None:` with the same message ("Owned acceptance matrix write failed.").
  4. Update the docstring lines that describe the removed parameter (around `:495-500` and the scaffold docstring) to describe `owned`.
  5. Apply the **caller rule** to `scaffold_acceptance_matrix` (caller `mission_finalize.py:2629-2637`, WP13). Note that caller also passes `repo_root=owned.primary` and `policy=ProtectionPolicy.resolve(owned.primary ...)`; those legacy attribute reads are WP13's to convert, not yours. Only the owned-root keyword on that call is in scope for your one-line edit, and only if WP13 has merged.
- **Files**: `src/specify_cli/acceptance/matrix.py`.
- **Parallel?**: Yes with T081.
- **Validation checklist**:
  - [ ] `tests/architectural/test_acceptance_matrix_write_seam.py`, `tests/specify_cli/acceptance/test_matrix_write_seam.py`, `tests/specify_cli/acceptance/test_matrix_marker_reject.py`, `tests/acceptance/test_overall_verdict_scaffold.py`, `tests/acceptance/test_issue_3231_scaffold_pending_poisons_acceptance.py`, `tests/integration/test_issue_2404_acceptance_matrix_write_surface.py`, `tests/lanes/test_acceptance_matrix.py` pass unchanged.
  - [ ] `tests/integration/test_explicit_checkout_commands.py::test_finalize_seeds_owned_status_only` and `::test_accept_no_commit_updates_only_owned_matrix` pass unchanged.
- **Edge cases**: the non-owned zero-write refusal path (FR-011) must still never fall back to a bare repository-root write; only the owned arm raises `RuntimeError`. `write_and_commit_acceptance_matrix` has non-owned callers (`acceptance_verdict.py`, `post_consolidation.py`) that pass no keyword; their behaviour must not change.

### Subtask T083 – Acceptance-package test updates plus the FR-022 ratchet

- **Purpose**: Diff coverage ≥ 90% (NFR-005) requires the owned arms to be exercised directly, and FR-022 requires proof that nothing existing changed. The only acceptance test file this WP owns is `tests/acceptance/test_gate_execution_context.py`; add the new focused tests there.
- **Steps**:
  1. Add a section `# --- owned checkout (WP15) ---` to `tests/acceptance/test_gate_execution_context.py` with focused tests that build a real fact through the WP02 fixture `owned_checkouts` from `tests/integration/conftest.py` (or, if that fixture is integration-scoped only, through the WP02 test helper that WP02 documents for minting a fact in tests; never call `OwnedCheckout._mint` directly — gate G3). Cover:
     - `build_gate_execution_context(..., owned=fact)` stamps the surface under `fact.owned_root` and never under the repository root checkout, even when the repository root checkout holds a stale copy of the same mission slug (use the WP02 `stale_root_copy` fixture).
     - `declared_home_surface(..., owned=fact)` for a single_branch mission answers `PRIMARY`.
     - `collect_feature_summary(repo_root, slug, owned=fact, mutate_matrix=False)` reads WPs from `fact.mission_dir / "tasks"`.
     - `scaffold_acceptance_matrix(..., owned=fact)` raising `RuntimeError` when the write seam reports a non-committed status (monkeypatch `write_and_commit_acceptance_matrix` to return a refused result).
     - `normalize_feature_encoding(..., owned=fact)` only touches files under `fact.mission_dir`.
  2. Mark the new tests `integration` + `git_repo` if they create repositories, `unit` + `fast` otherwise; keep the file's existing module-level markers intact.
  3. Assert the repository root checkout snapshot is unchanged in each owned test (WP02 `r_snapshot`).
  4. Run the full FR-022 ratchet list (Test Strategy). No existing test file may change. If one must, that is a behaviour change: stop and record it.
- **Files**: `tests/acceptance/test_gate_execution_context.py` (append only).
- **Parallel?**: No; after T080–T082.
- **Validation checklist**:
  - [ ] `uv run --frozen pytest tests/acceptance/test_gate_execution_context.py --cov=specify_cli.acceptance --cov-report=term-missing` shows every new owned branch covered.
  - [ ] `git diff --stat -- tests/ ':!tests/acceptance/test_gate_execution_context.py'` is empty.
- **Edge cases**: the file's existing fixture `flat_topology_mission` (`ctf.FlatTopologyContext`) is not an owned fixture; do not repurpose it. `test_fixture_smoke_no_resolver_patched` (`:793`) asserts nothing is patched globally — make sure your new monkeypatches are function-scoped.

## Test Strategy

Targeted (existing files must pass unchanged; the owned test file gains tests):

```bash
uv run --frozen pytest -q \
  tests/acceptance/ \
  tests/specify_cli/acceptance/ \
  tests/specify_cli/test_acceptance.py tests/specify_cli/test_canonical_acceptance.py \
  tests/specify_cli/test_accept_gate_convergence.py tests/specify_cli/test_acceptable_ending.py \
  tests/specify_cli/test_terminal_state_gate_integrity.py tests/specify_cli/test_acceptance_regressions.py \
  tests/specify_cli/test_accept_no_commit_readonly.py \
  tests/agent/test_orchestrator_commands_integration.py \
  tests/integration/test_accept_matrix_coord_partition.py \
  tests/integration/test_issue_2404_acceptance_matrix_write_surface.py \
  tests/integration/test_deferral_enforcement_and_disclosure.py \
  tests/integration/test_explicit_checkout_commands.py \
  tests/lanes/test_acceptance_matrix.py \
  tests/characterization/test_trio_pure_cores.py \
  tests/mission_runtime/test_coord_read_seam.py \
  tests/status/test_wp01_terminus_authority_parity.py \
  tests/consolidation/test_issue_2804_merge_resets_gate_artifacts.py \
  tests/specify_cli/cli/commands/test_accept_normalize_encoding.py \
  tests/specify_cli/cli/commands/test_issue_4891_accept_missing_lanes.py
```

Specific architectural gates implicated (files, not the directory):

```bash
uv run --frozen pytest -q \
  tests/architectural/test_read_surface_placement_guard.py \
  tests/architectural/test_acceptance_matrix_write_seam.py \
  tests/architectural/test_single_mission_surface_resolver.py \
  tests/architectural/test_owned_checkout_gate_selftest.py
```

Baseline and quality gates:

```bash
make test-fast
uv run --frozen ruff check src/specify_cli/acceptance/ tests/acceptance/test_gate_execution_context.py
uv run --frozen ruff format --check src/specify_cli/acceptance/ tests/acceptance/test_gate_execution_context.py
uv run --frozen mypy --strict src/specify_cli/acceptance/__init__.py src/specify_cli/acceptance/execution_context.py \
  src/specify_cli/acceptance/gates_core.py src/specify_cli/acceptance/matrix.py \
  src/specify_cli/coordination/write_seam.py src/specify_cli/cli/commands/accept.py \
  src/specify_cli/cli/commands/agent/mission_finalize.py
```

Run mypy on the callee and caller files **in one invocation**: `pyproject.toml` sets `follow_imports = "skip"` for `specify_cli.*`, so a mismatched keyword between two `specify_cli` modules is invisible unless both are on the command line. Record the exact commands and pass/fail counts in the Activity Log and the PR's *Tests run* section.

## Risks & Mitigations

- **Cross-WP caller breakage.** `accept.py` (WP14) and `mission_finalize.py` (WP13) pass the old keyword to two public functions here. Mitigation: the caller rule in Context; never edit a caller file whose WP has not merged.
- **Seam asymmetries mistaken for bugs.** Several readers treat the owned arm differently from the non-owned arm on purpose (see T080 edge cases). Mitigation: behaviour-preserving means both arms keep their current logic; only the parameter type changes.
- **Spy/monkeypatch compatibility.** Architectural tests spy on `declared_read_surface` and forward `**kwargs`. Mitigation: pass only keywords the real callee accepts.
- **mypy blind spot** (`follow_imports = "skip"`). Mitigation: multi-file invocation above.

## Review Guidance

- Re-run the occurrence census; anything left must be a logged bridge with a named callee.
- Check that every converted parameter is keyword-only with default `None` and typed `OwnedCheckout | None` (gate G6 pins signatures only for the §7 consumers, but consistency here is what lets WP18 go green).
- Check that any edit to `accept.py` or `mission_finalize.py` is one line, carries the rationale, and happened only after that WP merged.
- Confirm `git diff --stat -- tests/` touches only `tests/acceptance/test_gate_execution_context.py`.
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
