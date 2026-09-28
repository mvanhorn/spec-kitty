---
work_package_id: "WP14"
subtasks:
  - "T075"
  - "T076"
  - "T077"
  - "T078"
  - "T079"
title: "Decompose and convert accept"
task_type: "implement"
phase: "Phase 4 - Conversion sweep"
execution_mode: "code_change"
owned_files:
  - "src/specify_cli/cli/commands/accept.py"
  - "tests/specify_cli/cli/commands/test_accept_decomposition.py"
authoritative_surface: "src/specify_cli/cli/commands/accept.py"
create_intent:
  - "tests/specify_cli/cli/commands/test_accept_decomposition.py"
dependencies:
  - "WP07"
  - "WP08"
  - "WP10"
requirement_refs:
  - "FR-003"
  - "FR-022"
  - "FR-026"
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

# Work Package Prompt: WP14 – Decompose and convert accept

## ⚡ Do This First: Load Agent Profile

Use the `/ad-hoc-profile-load` skill to load the agent profile specified in the frontmatter (or any user-defined profile), and behave according to its guidance before parsing the rest of this prompt.

- **Profile**: `python-pedro`
- **Role**: `implementer`
- **Agent/tool**: `claude`

If no profile is specified, run `spec-kitty agent profile list` and select the best match for this work package's `task_type` and `authoritative_surface`.

Then load the action-scoped governance: `spec-kitty charter context --action implement --json`. The charter (`.kittify/charter/charter.md`) is binding; this prompt does not repeat it.

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

This WP is the IC-10a + IC-14a slice of the plan: the `accept` command stops being the largest over-limit function in the repository and stops carrying bare owned roots.

Done means all of the following hold at the tip of the WP:

1. **Characterisation first (T075).** `tests/specify_cli/cli/commands/test_accept_decomposition.py` exists, was committed **green against the undecomposed `accept()`** in its own commit, and pins every exit path of the command (exit code, JSON keys, side-effect order).
2. **Complexity restored (T076, FR-026).** `accept()` is ≤ 15 (it is 59 on the planning base, `src/specify_cli/cli/commands/accept.py:740`). No new helper exceeds 15. `C901` is removed from the `ruff.toml` per-file-ignore entry for `src/specify_cli/cli/commands/accept.py` (line 27 today: `["B904", "C901", "F401", "UP035"]` becomes `["B904", "F401", "UP035"]`). `uv run --frozen ruff check src/specify_cli/cli/commands/accept.py` is clean with that ignore gone.
3. **No re-validation (T077, FR-003).** `_stamp_birth_cutover_for_accept` no longer calls `resolve_owned_mission` (today `accept.py:339-342`). The fact validated once at the CLI edge is the only fact the command uses.
4. **Converted (T078, FR-001 via FR-022).** `accept.py` contains **zero** occurrences of the identifier `effective_root`, zero calls to `effective_root_kwargs`, zero `"effective_root"` string keys, zero `OwnedMission` references, zero legacy attribute reads (`.root`, `.primary`, `.directory`, `.slug`, `.target`) on the fact, and no direct `resolve_owned_mission` call. The `--owned-checkout` option is declared through WP08's `OwnedCheckoutOption` and resolved through `resolve_owned_or_adopt`.
5. **Ratchet (T079, FR-022).** Every existing accept test file listed under Test Strategy passes **without modification**, including the owned accept scenarios in `tests/integration/test_explicit_checkout_commands.py`.
6. ruff check, ruff format and `mypy --strict` are clean on the touched files; `make test-fast` passes.

## Context & Constraints

- **Mission docs**: `kitty-specs/owned-checkout-lifecycle-authority-01M3M2ZB/` → `spec.md` (FR-003, FR-022, FR-026), `plan.md` (Staging Strategy; IC-10a; IC-14a; "One owning WP per shared file": `cli/commands/accept.py` is owned by IC-10a = this WP), `research.md` R-16 (Complexity: `accept` 59 is the only over-limit function in its per-file-ignored file), `data-model.md` (OwnedCheckout fields), `contracts/owned-checkout-carrier.md` (§2 once per command, §3 no bare owned roots), `contracts/cli-owned-checkout-surface.md` (accept row: existing flag, single_branch, unchanged behaviour), `occurrence_map.yaml`.
- **What the earlier WPs give you** (read their landed code first; do not assume the shapes below, verify them):
  - WP01: `mission_runtime.OwnedCheckout` with canonical fields `repository_root`, `owned_root`, `mission_dir`, `mission_slug`, `topology`, `target_branch`, plus **transitional** legacy properties (`primary`, `root`, `directory`, `slug`, `target`) that WP18 deletes. Do not read the legacy properties in new code.
  - WP02: `specify_cli.core.owned_mission` is the sole minter; `OwnedMission` is a transitional alias; `require_unstaged_index` accepts the fact.
  - WP08: `src/specify_cli/cli/commands/_owned_checkout.py` exposes `OwnedCheckoutOption`, `resolve_owned_or_adopt(...)` and `emit_owned_refusal(...)`. Read the module and its test file (`tests/specify_cli/cli/commands/test_owned_checkout_helper.py`) before wiring.
  - WP07: `TransitionRequest.owned` and friends. accept does not build a `TransitionRequest` directly, but its callees in `specify_cli.acceptance` and `specify_cli.consolidation.baseline` are converted by WP15 and WP17.
  - WP10: owns `ruff.toml` and removed the `core/mission_creation.py` C901 entry. Your `ruff.toml` edit happens after WP10 merged (that is why WP10 is a dependency).
- **Staging (plan: Staging Strategy)**: conversion is top-down. `accept()` holds the fact; any callee that is **not yet** converted when you start receives the bridging expression `effective_root=owned.owned_root if owned else None` — but only in a callee's **own** file, never in `accept.py`. Inside `accept.py` every helper you own takes `owned: OwnedCheckout | None`. For calls leaving `accept.py` into WP15/WP17 modules, see T078 step 5.
- **Behaviour preservation is the contract.** FR-026 is a pure refactor; FR-022 says the existing accept tests pass unchanged. The JSON payload keys (`error`, `error_code`, `diagnose`, the `result.to_dict()` shape, advisory notes) are `serialized_keys: do_not_change` in `occurrence_map.yaml`. Error-code strings (`OWNED_*`) are `logs_telemetry: do_not_change`.
- **Terminology (C-008)**: in any new docstring, comment or message write "repository root checkout" and "owned checkout", never bare "primary" as a checkout alias and never "feature". Existing identifiers such as `_primary_dirty_paths` / `_commit_primary_residuals` name the PRIMARY partition (a governed term, see `docs/context/orchestration.md#primary-partition`) and stay as they are.
- **Complexity ceiling 15, Sonar S1192**: repeated error-emission literals become one helper, not three copies.
- **No suppressions**: no new `# noqa`, `# type: ignore` or per-file ignores. The three existing `# noqa: BLE001` lines (`accept.py:76`, `accept.py:352`, `accept.py:1095`) keep their inline rationale and move with their code.

## Branch Strategy

- **Strategy**: Planning artifacts were generated on `claude/sleepy-hamilton-5lelee`; completed changes must merge back into `claude/sleepy-hamilton-5lelee`.
- **Planning base branch**: `claude/sleepy-hamilton-5lelee`
- **Merge target branch**: `claude/sleepy-hamilton-5lelee`
- **Execution lane**: assigned in `kitty-specs/owned-checkout-lifecycle-authority-01M3M2ZB/lanes.json` by `spec-kitty agent mission finalize-tasks`. Start with `spec-kitty implement WP14` and work only in the workspace path it prints; never reconstruct the worktree path yourself.

> These fields are populated automatically by `spec-kitty agent mission finalize-tasks`.
> Do NOT change them manually unless you are certain the branch topology has changed.

## Occurrence census for `accept.py` (planning base `df1588860`)

Every site below must be gone or converted at the end of the WP (T077/T078). Line numbers are for the planning base; they drift after T076, so re-run the census command at the end:

```bash
grep -nw "effective_root\|effective_root_kwargs\|OwnedMission\|resolve_owned_mission" src/specify_cli/cli/commands/accept.py
grep -nE "owned\.(root|primary|directory|slug|target)\b" src/specify_cli/cli/commands/accept.py
```

| Line | Kind | Site |
|---|---|---|
| 29-34 | import | `OwnedMission`, `effective_root_kwargs`, `resolve_owned_mission` from `specify_cli.core.owned_mission` |
| 120 | param | `_coord_worktree_root(..., *, effective_root: Path \| None)` |
| 147-148 | `effective_root_kwargs` + splat | `scope = effective_root_kwargs(effective_root)` → `resolve_artifact_surface(**scope)` |
| 167 | param | `_coord_status_feature_dir(..., *, effective_root)` |
| 208-209 | `effective_root_kwargs` + splat | into `resolve_artifact_surface` |
| 217 | `effective_root_kwargs` splat | `placement_seam(repo_root, mission_slug, **effective_root_kwargs(effective_root))` |
| 220 | param | `_coord_dirty_paths(..., *, effective_root)` |
| 232-235 | splat | `_coord_worktree_root(**effective_root_kwargs(effective_root))` |
| 270 | param | `_stamp_birth_cutover_for_accept(..., effective_root)` |
| 271 | param | `_stamp_birth_cutover_for_accept(..., owned: OwnedMission \| None)` |
| 320-321 | `effective_root_kwargs` + splat | `placement_seam(..., **scope)` |
| 334 | splat | `_coord_status_feature_dir(..., **scope)` |
| 337-342 | **re-validation** | `if owned is None and effective_root is not None: owned = resolve_owned_mission(...)` |
| 370 | param | `_record_pr_merge_for_accept(..., effective_root)` |
| 398-402 | keyword | `record_pr_merge_baseline_for_mission(effective_root=effective_root)` (callee in WP17) |
| 483 | param | `_commit_residual_acceptance_artifacts(..., *, effective_root)` |
| 498-501 | splat | `_coord_dirty_paths(**effective_root_kwargs(effective_root))` |
| 665 | param | `_collect_summary_with_optional_repair(..., effective_root)` |
| 677, 679, 695, 703 | `effective_root_kwargs` + 3 splats | into `collect_feature_summary` (×2) and `normalize_feature_encoding` (callees in WP15) |
| 721-737 | `OwnedMission` + direct validator call | `_owned_accept_context` → `resolve_owned_mission(primary, checkout, mission)` |
| 763 | inline option | `owned_checkout: Annotated[Path \| None, typer.Option("--owned-checkout", ...)]` |
| 825 | legacy attr | `repo_root = owned.root` |
| 864 | legacy attrs | `owned.slug, owned.directory` |
| 868 | **dict key** | `scope = {"effective_root": owned.root} if owned is not None else {}` |
| 906 | splat | `verify_pr_merge_evidence(..., **scope)` (callee in WP17) |
| 925 | splat | `_collect_summary_with_optional_repair(..., **scope)` |
| 1075 | splat | `_stamp_birth_cutover_for_accept(repo_root, mission_slug, **scope, owned=owned)` |
| 1086 | splat | `_record_pr_merge_for_accept(..., **scope)` |
| 1105 | splat | `_commit_residual_acceptance_artifacts(repo_root, mission_slug, **scope)` |

Totals: 7 parameters, 1 inline option, 7 `effective_root_kwargs(...)` calls, 14 splats/keywords, 1 dict key, 4 legacy attribute reads, 2 validator calls (`:342`, `:732`).

## Subtasks & Detailed Guidance

### Subtask T075 – Characterisation tests for `accept()`

- **Purpose**: Freeze the observable behaviour of `accept()` before it is taken apart, so T076 is provably behaviour-preserving (DIRECTIVE_034 test-first; FR-026 "behaviour-preserving … with focused tests"). The existing suites cover many paths end to end, but none pins the side-effect **order** in the `finally` block or every JSON error envelope in one place.
- **Steps**:
  1. Create `tests/specify_cli/cli/commands/test_accept_decomposition.py` with `pytestmark = [pytest.mark.unit, pytest.mark.fast]`. Invoke the command through Typer exactly as `tests/integration/test_explicit_checkout_commands.py:24-25` does (`accept_app = typer.Typer(); accept_app.command()(accept)`; `CliRunner().invoke(accept_app, [...])`).
  2. Isolate the command from git and the filesystem by monkeypatching **module attributes of `specify_cli.cli.commands.accept`** only: `show_banner`, `find_repo_root`, `resolve_mission_dir_with_bare_modern_fold`, `choose_mode`, `_collect_summary_with_optional_repair`, `verify_pr_merge_evidence`, `perform_acceptance`, `resolve_acceptance_actor`, `_stranded_verdict_provenance_note`, `_stamp_birth_cutover_for_accept`, `_record_pr_merge_for_accept`, `_commit_residual_acceptance_artifacts`. Use tiny fakes (a `SimpleNamespace` summary with `ok`, `to_dict()`, `warnings`; a fake `AcceptanceResult` with `to_dict()`, `notes`, `commit_created`, `summary`). Record every helper call into one `calls: list[str]` so ordering is assertable.
  3. Write one test per exit path; each asserts exit code, and for `--json` runs, the exact top-level key set of the printed JSON:
     - `--mission` omitted → exit 2, `{"error": "--mission <slug> is required"}`.
     - `find_repo_root` raises `TaskCliError` → exit 1, `{"error"}` only.
     - owned entry raises `ActionContextError("OWNED_BRANCH_REFUSED", ...)` → exit 1, keys `{"error_code", "error"}` (simulate by patching `_owned_accept_context`; after T078 you will patch the WP08 helper instead — see step 5).
     - `--merge-commit` with a resolved mode other than `pr` → exit 2, message contains `only valid with --mode pr`.
     - `verify_pr_merge_evidence` raises `PrMergeEvidenceError` → exit 1, `Cannot record PR merge for`.
     - summary collection raises each of `Pre30LayoutError`, `AcceptanceError`, `AcceptanceMatrixParseError` → exit 1, `{"error"}` (parametrise).
     - `--diagnose --json` → exit 0, payload has `"diagnose": true`; `perform_acceptance` never called.
     - `--mode checklist` with `summary.ok` True/False → exit 0/1; `perform_acceptance` never called.
     - `summary.ok` False, with and without `--allow-fail` → exit 1 both ways.
     - `perform_acceptance` raises `AcceptanceError` → exit 1; **pin that `_commit_residual_acceptance_artifacts` still runs** (today's `finally` runs the residual commit whenever `commit_required`) and that the stamp and PR-merge recording do **not** run.
     - stamp raises `AcceptanceError` → exit 1, `Birth-cutover stamp refused:`; pr-merge raises → `PR merge recording failed:`; residual raises → `Residual artifact commit failed:`. When more than one fails, the stamp message wins over the PR-merge message, which wins over the residual message (today's `if` order at `accept.py:1108-1130`).
     - happy path, `--mode pr --merge-commit abc`: `calls == ["perform_acceptance", "stamp", "record_pr_merge", "residual"]`; `result.notes` gains the `PR merge recorded:` note; `--no-commit` instead gains the `re-run without --no-commit` note and calls neither stamp nor residual (`perform_acceptance(auto_commit=False)`).
  4. Assert the stamp is called with the `owned` value object that the CLI edge produced (identity: `is`), which T077 relies on.
  5. Keep each fake minimal and centralise the patching in one fixture so T078 only has to swap the owned-entry patch target in one place.
  6. Run the file against the **planning base code** and commit it on its own: `test(accept): characterise accept() exit paths before decomposition (WP14/T075)`.
- **Files**: `tests/specify_cli/cli/commands/test_accept_decomposition.py` (new). Read-only references: `src/specify_cli/cli/commands/accept.py:740-1159`, `tests/specify_cli/cli/commands/test_accept_merge_commit.py` (merge-commit fakes to borrow), `tests/specify_cli/cli/commands/test_accept_clean_tree.py`.
- **Parallel?**: No. It must land before T076.
- **Validation checklist**:
  - [ ] File is green on the undecomposed `accept()` (run before touching `accept.py`).
  - [ ] Every `raise typer.Exit(...)` in `accept()` (there are 16 today) is reached by at least one test; check with `uv run --frozen pytest tests/specify_cli/cli/commands/test_accept_decomposition.py --cov=specify_cli.cli.commands.accept --cov-report=term-missing` and confirm no uncovered line inside `accept()`.
  - [ ] Tests assert key sets and codes, not rich console formatting.
- **Edge cases**: the human (non-JSON) path calls `show_banner()` and a `StepTracker`; test at least one human-mode failure so the tracker-error branches are covered. `normalize_encoding` and `diagnose` together are refused only in owned mode (`_owned_accept_context`), so pin that refusal too.

### Subtask T076 – Decompose `accept()` to ≤ 15; drop the `accept.py` C901 ignore

- **Purpose**: FR-026. `accept()` sits at complexity 59 (`ruff check --isolated --select C901` on the planning base). The plan (IC-14a) requires a behaviour-preserving decomposition **before** the functional conversion (tidy-first), and removal of the per-file suppression that has hidden it.
- **Steps**:
  1. Introduce one small frozen carrier for the run, for example `@dataclass(frozen=True) class _AcceptRun` holding `json_output`, `tracker`, `repo_root`, `mission_slug`, `mission_dir`, `owned`, `actual_mode`, `commit_required`, `provenance_note`. (Keep `owned` typed as the WP02 alias at this point if T078 has not run; T078 retypes it.)
  2. Extract one error emitter, e.g. `_fail(run_or_json, message, *, code=1, tracker_step=None)`, replacing the eight near-identical `if json_output: print(json.dumps({"error": ...})) else: tracker.error(...); console.print(...)` blocks (`accept.py:852-859, 897-903, 912-917, 944-952, 953-960, 971-978, 1044-1052, 1109-1130`). It raises `typer.Exit(code)`; preserve each block's exact message text and whether the tracker is rendered.
  3. Extract phase helpers, each ≤ 15 and each with a single responsibility (suggested names; keep them private):
     - `_resolve_accept_entry(...)` → `(repo_root, owned)`; owns the `ActionContextError`/`TaskCliError` handling currently at `accept.py:815-836`.
     - `_resolve_mission_identity(...)` → `(mission_slug, mission_dir)`; owns the exit-2 missing-handle branch (`:849-859`) and the owned/non-owned split (`:863-867`).
     - `_verify_merge_commit(...)` → `PrMergeEvidence | None` (`:888-917`).
     - `_collect_summary_or_exit(...)` → `AcceptanceSummary`; the three `except` arms (`:936-978`) collapse onto `_fail` via one `except (Pre30LayoutError, AcceptanceError, AcceptanceMatrixParseError)` **only if** the three arms are byte-identical in behaviour (they are today; confirm).
     - `_exit_early_for_report_modes(...)` for diagnose / checklist / not-ok (`:982-1015`).
     - `_perform_and_finalize(...)` → a small `_FinalizeOutcome` dataclass with `result`, `accept_exc`, `stamp_exc`, `pr_merge_exc`, `residue_exc`, `pr_merge_recorded`; it owns the `try/except/finally` at `:1031-1107` and **must keep the same ordering** (stamp → PR merge → residual, residual still running after an `AcceptanceError`).
     - `_raise_on_finalize_errors(...)` (`:1108-1130`) and `_render_accept_result(...)` (`:1134-1159`).
  4. `accept()` itself becomes a linear sequence of those calls plus the Typer option declarations. The Typer signature, option names, help strings, defaults and metavars do **not** change (`cli_commands: manual_review` — no flag is renamed or removed).
  5. Remove `"C901"` from the `src/specify_cli/cli/commands/accept.py` entry at `ruff.toml:27`, leaving `["B904", "F401", "UP035"]`. `ruff.toml` is owned by WP10: this is a **one-line out-of-map edit**. Record it in the commit message and the PR/Activity Log with the rationale: "FR-026 / IC-14a: accept() decomposed to ≤15; its per-file C901 suppression is removed in the same WP that removes the violation (ruff.toml is WP10-owned; one-line out-of-map edit)." Do not touch any other `ruff.toml` line.
  6. Commit as a pure refactor: `refactor(accept): decompose accept() to complexity ≤15 (WP14/T076, FR-026)`. No behaviour change, no owned-root conversion in this commit.
- **Files**: `src/specify_cli/cli/commands/accept.py` (functions `accept` `:740-1159` and new private helpers placed above it); `ruff.toml:27` (out-of-map, one line).
- **Parallel?**: No; depends on T075.
- **Validation checklist**:
  - [ ] `uv run --frozen ruff check --isolated --select C901 --config 'lint.mccabe.max-complexity=15' src/specify_cli/cli/commands/accept.py` reports nothing.
  - [ ] `uv run --frozen ruff check src/specify_cli/cli/commands/accept.py` clean with the ignore removed.
  - [ ] T075 file green, unchanged. Every accept test file in the Test Strategy list green, unchanged.
  - [ ] `git diff` of the `accept()` Typer parameter block is empty apart from moved lines.
  - [ ] `_print_acceptance_result` (13 today) is not made more complex.
- **Edge cases**: `typer.Exit` raised inside a helper must propagate unchanged (do not wrap helpers in `try/except Exception`). The `finally` semantics matter: an `AcceptanceError` inside `perform_acceptance` still triggers the residual commit; an unexpected exception type from `perform_acceptance` (not `AcceptanceError`) still propagates after the `finally` ran — keep that. `assert result is not None` (`:1132`) should become an explicit narrowing in the outcome object rather than an `assert` removed silently.

### Subtask T077 – Stop re-validating at `accept.py:342`; consume the fact

- **Purpose**: FR-003 / contracts/owned-checkout-carrier.md §2: ownership is validated exactly once per command; every downstream function receives the fact and never re-validates. `_stamp_birth_cutover_for_accept` still carries a fallback that re-runs `resolve_owned_mission` whenever it holds only an `effective_root` (`accept.py:336-342`), which is also a gate G2 offender (a validator call outside the allowed modules).
- **Steps**:
  1. Change the signature to `_stamp_birth_cutover_for_accept(repo_root: Path, mission_slug: str, *, owned: OwnedCheckout | None = None) -> None`. Delete the `effective_root` parameter and the whole `if owned is None and effective_root is not None:` block with its `resolve_canonical_root` import.
  2. Inside, replace `scope = effective_root_kwargs(effective_root)` + `placement_seam(repo_root, mission_slug, **scope)` with the seam's owned keyword from WP04: `placement_seam(repo_root, mission_slug, owned=owned)`. Replace `_coord_status_feature_dir(repo_root, mission_slug, **scope)` with `_coord_status_feature_dir(repo_root, mission_slug, owned=owned)` (converted in T078).
  3. Keep the `stamp_accept_cutover(..., **({"owned": owned} if owned is not None else {}))` call shape only if WP17 has not yet converted `stamp_accept_cutover`; if `runtime_state_cutover.stamp_accept_cutover` already takes `owned: OwnedCheckout | None = None`, pass `owned=owned` directly (it is a plain keyword with a `None` default, so no splat is needed either way — prefer `owned=owned`).
  4. Keep the owned-vs-best-effort error semantics exactly: owned → `AcceptanceError("Owned birth-cutover failed: ...")`; non-owned → `logger.warning` and return.
  5. Update the two comments that justify the fallback (`accept.py:335-338` and `:1073-1074`) to say the fact comes from the CLI edge; do not leave a comment referring to a code path that no longer exists.
- **Files**: `src/specify_cli/cli/commands/accept.py` (`_stamp_birth_cutover_for_accept`, planning-base `:266-362`, and its call site in the finalize helper from T076).
- **Parallel?**: No; after T076.
- **Validation checklist**:
  - [ ] `grep -n "resolve_owned_mission\|resolve_canonical_root" src/specify_cli/cli/commands/accept.py` returns nothing.
  - [ ] Positional callers still work: `tests/architectural/test_lifted_cli_accept_coord_surface_anchor.py:74` and `tests/specify_cli/cli/commands/test_accept_birth_cutover_seam.py:184` call `_stamp_birth_cutover_for_accept(repo_root, handle)` and must pass unchanged.
  - [ ] `tests/integration/test_explicit_checkout_commands.py::test_accept_commits_metadata_matrix_and_cutover_only_in_owned` passes (it wraps the real `stamp_accept_cutover`).
- **Edge cases**: a non-owned run must still pass `owned=None` all the way down (today's `**scope` is `{}`); make sure the seam call with `owned=None` is exactly equivalent to the old call with no keyword. The architectural test `tests/architectural/test_single_mission_surface_resolver.py:341-347` pins that the COORD leg comes from `_coord_status_feature_dir`; keep that routing.

### Subtask T078 – Convert `accept.py` owned roots; option migration

- **Purpose**: FR-001 (no bare owned roots anywhere in `src/`), contracts/cli-owned-checkout-surface.md (all eight inline `--owned-checkout` declarations move onto the shared option; `accept.py:763` is one of them), gate G2 (no direct validator call outside `_owned_checkout.py`/`owned_mission.py`).
- **Steps**:
  1. **Imports.** Replace the `specify_cli.core.owned_mission` import block (`:29-34`) with `from mission_runtime import OwnedCheckout` (public surface, MR-1/MR-2 forbid submodule imports) plus `require_unstaged_index` from `owned_mission` and `OwnedCheckoutOption` / `resolve_owned_or_adopt` / `emit_owned_refusal` from `specify_cli.cli.commands._owned_checkout`. Drop `effective_root_kwargs` and `OwnedMission`.
  2. **Option.** Replace the inline `owned_checkout: Annotated[Path | None, typer.Option("--owned-checkout", ...)]` at `:763` with `owned_checkout: OwnedCheckoutOption = None`. Check the WP08 help text; if it differs from today's ("Explicit owned checkout for a single-branch mission."), the golden contract tests owned by WP09 decide which wins — run `tests/specify_cli/cli/commands/agent/test_mission_cli_golden_contract.py` and report any help-text drift in the Activity Log rather than editing goldens here.
  3. **Entry.** Rewrite `_owned_accept_context` (planning-base `:721-737`) to call `resolve_owned_or_adopt(...)` with `LIFECYCLE_OWNED_TOPOLOGIES` (accept is single_branch only per the CLI contract), keep the `OWNED_OPTION_UNSUPPORTED` refusal for `--diagnose --normalize-encoding` and the `require_unstaged_index(owned)` call for non-diagnose runs, and return `OwnedCheckout | None`. Route owned refusals through `emit_owned_refusal` **only if** it prints the same `{"error_code", "error"}` envelope accept prints today (`:826-831`); if not, keep accept's own printing — the envelope keys are `serialized_keys: do_not_change`. Flagless runs follow FR-021 via the helper; run from the repository root checkout they must behave exactly as today (`test_accept_without_opt_in_keeps_primary_resolution`).
  4. **Helpers.** Convert every parameter in the census table to `owned: OwnedCheckout | None = None` (keyword-only, default `None`, so positional test callers keep working): `_coord_worktree_root`, `_coord_status_feature_dir`, `_coord_dirty_paths`, `_record_pr_merge_for_accept`, `_commit_residual_acceptance_artifacts`, `_collect_summary_with_optional_repair`. Inside, calls to the six dual-keyword seams (`placement_seam`, `resolve_artifact_surface` if WP04 gave it `owned`, `mission_context_for`, …) use `owned=owned`. Delete every `scope = effective_root_kwargs(...)` local and every `**scope` splat.
  5. **Calls that leave `accept.py`.** The callees `collect_feature_summary` / `normalize_feature_encoding` (WP15, `acceptance/__init__.py:1388,1031`), `verify_pr_merge_evidence` / `record_pr_merge_baseline_for_mission` (WP17, `consolidation/baseline.py:497,814`) and `stamp_accept_cutover` (WP17) are converted by other WPs that run in the same wave. At the moment you implement, check each callee's signature on your lane base:
     - callee already takes `owned` → pass `owned=owned`;
     - callee still takes `effective_root` → pass the bridging expression `effective_root=owned.owned_root if owned else None` **as a plain keyword argument** (never a dict splat and never `effective_root_kwargs`). Do not add a marker comment; list every such bridge (file:line, callee) in the Activity Log so the callee WP or WP18/T097 can flip it. This is the only place the identifier may survive in `accept.py`, and only until the callee converts.
  6. **Legacy attributes.** `owned.root` → `owned.owned_root` (`:825`, `:868`), `owned.slug` → `owned.mission_slug`, `owned.directory` → `owned.mission_dir` (`:864`). Delete the `scope = {"effective_root": owned.root} ...` dict at `:868` outright — the run carrier from T076 already holds `owned`.
  7. Commit: `refactor(accept): consume the validated OwnedCheckout fact end to end (WP14/T078)`.
- **Files**: `src/specify_cli/cli/commands/accept.py` only (all census rows).
- **Parallel?**: No; after T077.
- **Validation checklist**:
  - [ ] Census command from the section above prints only bridge lines you listed in the Activity Log (ideally nothing).
  - [ ] `grep -n '"effective_root"' src/specify_cli/cli/commands/accept.py` prints nothing.
  - [ ] `uv run --frozen mypy --strict src/specify_cli/cli/commands/accept.py src/specify_cli/cli/commands/_owned_checkout.py src/specify_cli/acceptance/__init__.py src/specify_cli/consolidation/baseline.py src/specify_cli/migration/runtime_state_cutover.py` is clean. Pass all of them in **one** invocation: `pyproject.toml` sets `follow_imports = "skip"` for `specify_cli.*`, so a signature mismatch between `accept.py` and a callee module is only visible when both files are on the command line.
  - [ ] T075 characterisation file green (only the owned-entry patch target moved, in the shared fixture).
- **Edge cases**: `repo_root` is reassigned to the owned root today (`:825`) and every later read uses it (`choose_mode(requested_mode, repo_root)`, `_primary_dirty_paths(repo_root, ...)`, `_commit_primary_residuals`). Keep that exact behaviour: the run carrier's `repo_root` is `owned.owned_root` when owned. `_coord_worktree_root` compares `worktree_root.resolve() == repo_root.resolve()`; do not start comparing against `owned.repository_root`.

### Subtask T079 – FR-022 accept ratchet

- **Purpose**: FR-022 / SC-005: the existing owned accept tests and all non-owned accept tests pass **unchanged** after the rewire onto the single validator. This subtask is verification plus the one new assertion the ratchet needs: accept validates exactly once.
- **Steps**:
  1. Run the full accept list from the Test Strategy with no test-file edits. Any red that also reproduces on the planning base is baseline red: classify it per CLAUDE.md "Test-run baseline-red gotcha" and note it; do not fix it here.
  2. Add one test to `tests/specify_cli/cli/commands/test_accept_decomposition.py` (your owned file) that counts ownership validations: monkeypatch `specify_cli.core.checkout_ownership.resolve_ownership_claim` with a counting wrapper around the real function, call `specify_cli.workspace.context.clear_workspace_resolution_caches()` first, and run an owned `--diagnose --json` accept on the WP02 fixture `owned_checkouts` (from `tests/integration/conftest.py`). Assert the count is exactly 1. Mark it `integration` + `git_repo` (it builds real repositories), not `fast`.
  3. Assert the repository root checkout is untouched on that same run with the WP02 `r_snapshot` fixture (0 differences).
  4. If any existing test **must** change, stop: that is a behaviour change, which FR-022 forbids. Record it in the Activity Log and ask the reviewer before editing a file you do not own.
- **Files**: `tests/specify_cli/cli/commands/test_accept_decomposition.py` (append). Read-only: `tests/integration/conftest.py` (WP02 fixtures), `tests/integration/test_explicit_checkout_commands.py:388-530`.
- **Parallel?**: No; last.
- **Validation checklist**:
  - [ ] Every file in the Test Strategy list is green with `git diff --stat` showing no change under those paths.
  - [ ] The exactly-once test fails if you temporarily re-add a second `resolve_owned_mission` call (mutation check, do not commit it).
- **Edge cases**: the diagnose path does not call `require_unstaged_index`, so the owned diagnose run must succeed even with staged changes in the owned checkout; do not stage anything in the fixture to "help" it.

## Test Strategy

Targeted tests (run all; none of the existing files may change):

```bash
uv run --frozen pytest -q \
  tests/specify_cli/cli/commands/test_accept_decomposition.py \
  tests/integration/test_explicit_checkout_commands.py -k accept \
  tests/specify_cli/cli/commands/test_accept_birth_cutover_seam.py \
  tests/specify_cli/cli/commands/test_accept_clean_tree.py \
  tests/specify_cli/cli/commands/test_accept_guidance_liveness.py \
  tests/specify_cli/cli/commands/test_accept_lenient_help.py \
  tests/specify_cli/cli/commands/test_accept_merge_commit.py \
  tests/specify_cli/cli/commands/test_accept_normalize_encoding.py \
  tests/specify_cli/cli/commands/test_accept_readiness_no_write.py \
  tests/specify_cli/cli/commands/test_accept_residual_partition.py \
  tests/specify_cli/cli/commands/test_accept_stranded_verdict_note.py \
  tests/specify_cli/cli/commands/test_accept_warnings_render.py \
  tests/specify_cli/cli/commands/test_accept_guard.py \
  tests/specify_cli/cli/commands/test_issue_4891_accept_missing_lanes.py \
  tests/specify_cli/cli/commands/test_bare_modern_slug_ambiguity_cli.py \
  tests/specify_cli/cli/commands/test_lifecycle_read_seam_migration.py \
  tests/specify_cli/cli/test_accept_birth_cutover.py \
  tests/specify_cli/cli/test_accept_coord_surface_anchor.py \
  tests/specify_cli/test_accept_no_commit_readonly.py \
  tests/specify_cli/test_acceptance_regressions.py \
  tests/specify_cli/acceptance/test_accept_idempotency.py \
  tests/specify_cli/acceptance/test_accept_contracts_dedup.py \
  tests/regressions/test_issue_4968_accept_encoding.py \
  tests/integration/test_accept_matrix_coord_partition.py \
  tests/agent/test_commands.py
```

Specific architectural gates implicated by `accept.py` (run these files, not the directory):

```bash
uv run --frozen pytest -q \
  tests/architectural/test_lifted_cli_accept_coord_surface_anchor.py \
  tests/architectural/test_single_mission_surface_resolver.py \
  tests/architectural/test_cli_error_surface_seam.py \
  tests/architectural/test_owned_checkout_gate_selftest.py \
  tests/architectural/test_ruff_format_enforcement.py
```

Baseline and quality gates:

```bash
make test-fast
uv run --frozen ruff check src/specify_cli/cli/commands/accept.py tests/specify_cli/cli/commands/test_accept_decomposition.py ruff.toml
uv run --frozen ruff format --check src/specify_cli/cli/commands/accept.py tests/specify_cli/cli/commands/test_accept_decomposition.py
uv run --frozen mypy --strict src/specify_cli/cli/commands/accept.py src/specify_cli/cli/commands/_owned_checkout.py src/specify_cli/acceptance/__init__.py src/specify_cli/consolidation/baseline.py src/specify_cli/migration/runtime_state_cutover.py tests/specify_cli/cli/commands/test_accept_decomposition.py
```

Record the exact commands and pass/fail counts in the Activity Log and the PR's *Tests run* section. A passing test run does not replace the mypy result: a mypy failure fails the WP.

## Risks & Mitigations

- **Silent behaviour drift during decomposition.** Mitigation: T075 lands first and green; T076 is a separate refactor-only commit; reviewers diff behaviour against the T075 file, not against intuition.
- **`finally`-block ordering.** The residual commit must still run after an `AcceptanceError`, and the stamp must run before the residual commit so its writes are swept into it (R4 in the docstring). Mitigation: the call-order assertions in T075.
- **Parallel callee conversion (WP15, WP17).** Both touch functions `accept.py` calls, in the same wave, with no dependency edge between them. Mitigation: T078 step 5 decides per callee at implementation time and logs every surviving bridge; WP18/T097 removes any leftover.
- **mypy blind spot.** `follow_imports = "skip"` for `specify_cli.*` hides cross-module signature errors unless both files are checked together. Mitigation: the single multi-file mypy command above.
- **Help-text or envelope drift from the shared helper.** Mitigation: T078 step 2/3; the golden-contract test is the arbiter.

## Review Guidance

- Confirm the T075 commit precedes the T076 commit and that the T075 file did not change in T076 (except the one fixture patch target in T078).
- Run `uv run --frozen ruff check --isolated --select C901 --config 'lint.mccabe.max-complexity=15' src/specify_cli/cli/commands/accept.py` yourself; expect no output.
- Check the `ruff.toml` diff is exactly one line (C901 removed from the `accept.py` entry) and carries the out-of-map rationale.
- Grep for `effective_root`, `effective_root_kwargs`, `OwnedMission`, `resolve_owned_mission` and legacy attribute reads in `accept.py`; anything left must be a logged bridge to an unconverted callee.
- Confirm no existing test file changed (`git diff --stat -- tests/ ':!tests/specify_cli/cli/commands/test_accept_decomposition.py'` is empty).
- Confirm the implementer ran the multi-file mypy command and it passed.
- Terminology: new prose says "owned checkout" / "repository root checkout", never "feature".

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
