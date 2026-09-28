---
work_package_id: "WP18"
subtasks:
  - "T095"
  - "T096"
  - "T097"
  - "T098"
  - "T099"
  - "T100"
title: "Single-authority gate closure and end-to-end proof"
task_type: "implement"
phase: "Phase 5 - Closure"
execution_mode: "code_change"
owned_files:
  - "tests/architectural/test_owned_checkout_single_authority.py"
  - "tests/architectural/_baselines.yaml"
  - "tests/integration/test_owned_lifecycle_acceptance_e2e.py"
  - "tests/performance/test_owned_checkout_perf.py"
authoritative_surface: "tests/architectural/"
create_intent:
  - "tests/architectural/test_owned_checkout_single_authority.py"
  - "tests/integration/test_owned_lifecycle_acceptance_e2e.py"
  - "tests/performance/test_owned_checkout_perf.py"
dependencies:
  - "WP01"
  - "WP02"
  - "WP03"
  - "WP04"
  - "WP05"
  - "WP06"
  - "WP07"
  - "WP08"
  - "WP09"
  - "WP10"
  - "WP11"
  - "WP12"
  - "WP13"
  - "WP14"
  - "WP15"
  - "WP16"
  - "WP17"
requirement_refs:
  - "FR-001"
  - "NFR-001"
  - "NFR-002"
  - "NFR-003"
  - "NFR-005"
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

# Work Package Prompt: WP18 – Single-authority gate closure and end-to-end proof

## ⚡ Do This First: Load Agent Profile

Use the `/ad-hoc-profile-load` skill to load the agent profile specified in the frontmatter (or any user-defined profile), and behave according to its guidance before parsing the rest of this prompt.

- **Profile**: `python-pedro`
- **Role**: `implementer`
- **Agent/tool**: `claude`

If no profile is specified, run `spec-kitty agent profile list` and select the best match for this work package's `task_type` and `authoritative_surface`.

Then load the action-scoped governance: `spec-kitty charter context --action implement --json`. The charter (`.kittify/charter/charter.md`) is binding; its "Architectural gate discipline" standing order (DIRECTIVE_043) governs T095–T097.

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

This is IC-11, the closing WP. It turns the mission's promise into an enforced invariant and proves the lifecycle end to end.

Done means:

1. **Red first (T095).** The **first commit** of this WP adds `tests/architectural/test_owned_checkout_single_authority.py` with the G1–G6 assertions of `contracts/architectural-gate.md`, and they are **red** against the tree as it stands after WP01–WP17 (the transitional surfaces still exist). The commit message records the offender count per gate.
2. **Transitional surfaces deleted (T096).** The legacy `effective_root=` keyword on the six dual-keyword seams, `effective_root_kwargs` / `_EffectiveRootKwargs`, the `OwnedMission` alias, and `OwnedCheckout`'s transitional legacy properties are gone, each deletion declared as an out-of-map closure edit with a one-line rationale.
3. **Green (T097, FR-001 / SC-004).** Every G1–G6 assertion passes with an **empty** allowlist; `tests/architectural/_baselines.yaml` carries `owned_root_bare_path_params: 0` under a `test_owned_checkout_single_authority` section, and the ratchet meta-test enforces it.
4. **End to end (T098, SC-001 / NFR-001).** `tests/integration/test_owned_lifecycle_acceptance_e2e.py` walks the `quickstart.md` flow through the real CLI (create → spec-commit → setup-plan → status → finalize → `next` to implement WP01 → move-task claimed → in_progress → for_review → `next` review prompt → context resolve), for cwd ∈ {repository root checkout R, owned checkout P, an unrelated directory}, with and without a stale copy of the mission in R, with 0 errors and 0 R-snapshot differences.
5. **Measurements (T099).** NFR-002 exactly-one validation per owned command (counted at `checkout_ownership.resolve_ownership_claim`) and 0 extra git subprocesses per owned status read, per PR; NFR-003 median-of-5 latency < 2 s for owned `agent tasks status`, `setup-plan` and `context resolve`, `performance`-marked (nightly).
6. **Closure hygiene (T100, NFR-005).** Terminology guard green, docs freshness green, the targeted gate list green, `make test-fast` green, ruff check/format and `mypy --strict` clean on every file this WP touched.

## Context & Constraints

- **Authoritative contract**: `kitty-specs/owned-checkout-lifecycle-authority-01M3M2ZB/contracts/architectural-gate.md`. Read it in full before T095; this prompt restates it but the contract wins on any difference.
- **Other mission docs**: `spec.md` (FR-001, SC-001, SC-004, NFR-001..NFR-003, NFR-005, C-008), `plan.md` (Staging Strategy — "the legacy keyword is deleted in the closing WP"; "The G1–G5 assertions are committed red as the first commit of the closing WP"; Test Layout and Markers; IC-11), `research.md` R-03 and R-16 (Gate, R-snapshot, Sizing), `contracts/owned-checkout-carrier.md` (§1 sole construction, §3 no bare owned roots, §7 the consumer list G6 pins), `quickstart.md` (the SC-001 script T098 automates), `data-model.md` (`stale_repository_root_copy` payload field; error-code registry), `occurrence_map.yaml` (note the `tests/architectural/_baselines.yaml` exception: "may only shrink or gain the new gate's zero cap; never loosen").
- **Prerequisite artefacts you build on** (all merged; read them first):
  - WP01 `tests/architectural/_owned_checkout_scan.py` — the AST scanner (reference forms, identifier ban, annotation forms, FQN exemption) — and its green self-mutation tests in `tests/architectural/test_owned_checkout_gate_selftest.py`. **Do not re-implement scanning.** The gate file composes the scanner's functions over `src/`; if a rule the contract needs is missing from the scanner, add it in the scanner as an out-of-map edit with a new self-mutation case in the self-test (rationale: "G<n> needs <rule>; scanner is WP01's single scanning authority").
  - WP02 fixtures in `tests/integration/conftest.py`: `owned_checkouts`, `r_snapshot`, `stale_root_copy` (self-tested in `tests/integration/test_owned_fixtures_selftest.py`). Use them; do not build parallel fixtures.
  - Per-concern acceptance files from WP08–WP13 (`tests/integration/test_owned_lifecycle_acceptance_{context,status,cli,next,review,finalize}.py`). The e2e file chains the steps they test individually; reuse their helpers by importing fixtures through `conftest.py`, never by importing test modules.
  - `tests/architectural/_ast_scan.py` (`parse_file`, `read_and_parse`, `qualname_by_line`) — the parser every gate uses; comments and docstrings never count as offenders.
- **Out-of-map closure edits.** T096/T097 edit files owned by earlier WPs. This is sanctioned by the plan's Staging Strategy and the WP ownership notes ("declared as out-of-map closure edits, each with a one-line rationale"). Every such edit is listed in the commit message and the Activity Log as `<path>: <one-line rationale>`. Keep each edit to the deletion or the re-point it names; no opportunistic refactors in other WPs' files.
- **No ledger, no allowlist.** Operator decision `01M3M65D…`: empty allowlists, no ratchet ledger. The only exemptions are the two named rules the contract defines (the `src/charter` module rule for the org-pack sense of `effective_root`, and the FQN rule for the fields of `mission_runtime.owned_checkout.OwnedCheckout`). Adding any other exemption is out of scope and will be rejected.
- **Test policy (`NO_FULL_HEAVY_SUITES_IN_MISSION`)**: run the specific architectural gate files implicated (T100 lists them), never the bare `tests/architectural/` directory; the full sweep is CI's.
- **Terminology (C-008)**: "repository root checkout", "owned checkout", "coordination worktree", "lane worktree", "Mission"; never bare "primary" as a checkout alias, never "feature".

## Branch Strategy

- **Strategy**: Planning artifacts were generated on `claude/sleepy-hamilton-5lelee`; completed changes must merge back into `claude/sleepy-hamilton-5lelee`.
- **Planning base branch**: `claude/sleepy-hamilton-5lelee`
- **Merge target branch**: `claude/sleepy-hamilton-5lelee`
- **Execution lane**: assigned in `kitty-specs/owned-checkout-lifecycle-authority-01M3M2ZB/lanes.json` by `spec-kitty agent mission finalize-tasks`. Start with `spec-kitty implement WP18` and use only the workspace path it prints. Every WP01–WP17 lane must be merged into your lane base before you start (check `spec-kitty agent tasks status`).

> These fields are populated automatically by `spec-kitty agent mission finalize-tasks`.
> Do NOT change them manually unless you are certain the branch topology has changed.

## Subtasks & Detailed Guidance

### Subtask T095 – Commit the G1–G6 gate assertions red (record offender counts)

- **Purpose**: DIRECTIVE_043 architectural-gate discipline and C-007 red-first: the gate must be demonstrably non-vacuous against a real tree before the fix that turns it green. The contract requires the G1–G6 assertions to be the **first commit** of this WP, red.
- **Steps**:
  1. Create `tests/architectural/test_owned_checkout_single_authority.py` with `pytestmark = [pytest.mark.architectural]`, a module docstring naming the contract file, and one test per gate:
     - **G1** `resolve_ownership_claim` is referenced only in `src/specify_cli/core/owned_mission.py` plus its definition in `src/specify_cli/core/checkout_ownership.py` (`:190` on the planning base). Every reference form counts: `ast.alias.name` (including `import … as x`), `Attribute.attr`, `Name.id`, and string `Constant`s used with `getattr`/`importlib`. Planning-base floor: `cli/commands/next_cmd.py:167,170`, `core/mission_creation.py:33,848`.
     - **G2** calls `resolve_owned_mission(` / `adopt_owned_checkout(` only from `src/specify_cli/cli/commands/_owned_checkout.py` and `src/specify_cli/core/owned_mission.py`. Planning-base floor includes `coordination/status_transition.py:917`, `status/models.py:902`, `status/bootstrap.py:183`, `migration/backfill_runtime_state.py:1441`, and the CLI commands `accept.py:342,732`, `tasks_mark_status.py:180`, `tasks_move_task.py:463`, `spec_commit_cmd.py:110`, `mission_check_prerequisites.py:599`, `mission_finalize.py:202`, `mission_runtime/resolution.py:1514`.
     - **G3** `OwnedCheckout._mint` is referenced only in `owned_mission.py` (covered by WP01's self-mutation case; here it runs over the real tree).
     - **G4** the identifier `effective_root` appears nowhere in `src/` outside `src/charter`: not as a parameter, field, local annotation or local binding, keyword argument, TypedDict key, or string dict key `"effective_root"`. Two exemptions only: the module rule `src/charter/**` (org-pack root, a different concept) and the **symbol** rule for org-pack method reads — the method is `OrgPackConfig.effective_root` (`src/charter/offering/drg/org_pack_config.py:374`); its call sites in `specify_cli` are `analysis_inputs.py:207`, `charter_runtime/lint/checks/org_layer.py:330`, `cli/commands/_doctrine_collect.py:513`, `doctrine/config.py:42`, `doctrine/org_charter.py:700`, `doctrine/snapshot.py:719`. Implement the symbol rule as "an `Attribute` named `effective_root` that is the `func` of a `Call` on a receiver the scanner can prove is an org-pack config", or, if the scanner cannot prove the receiver, as the exact-call-site shape `<name>.effective_root(<repo root>)` in those modules — whichever WP01's scanner supports; document the choice in the gate's docstring. (The contract text calls the symbol `PackContext.effective_root`; the real class is `OrgPackConfig`. Use the real symbol and note the correction in the Activity Log.)
     - **G5** no parameter or field carries an owned root as a path under any of `owned_root`, `owned_checkout`, `checkout_root`, `effective_root`, annotated `Path`, `pathlib.Path`, `Path | None`, `Optional[Path]`, `Union[Path, None]`, `os.PathLike`, or the string form of any of these. Only exemption: the fields of `mission_runtime.owned_checkout.OwnedCheckout`, by fully qualified class name. Planning-base floor: `tasks_mark_status.py:114`, `tasks_move_task.py:248`, `mission_creation.py:127`.
     - **G6** positive signature pin: each consumer in `contracts/owned-checkout-carrier.md` §7 declares a parameter or field `owned: OwnedCheckout | None`: `mission_runtime.placement_seam`, `mission_runtime.mission_context_for`, `mission_runtime.resolve_action_context`, `specify_cli.task_utils.support.locate_work_package`, `specify_cli.workspace.context.resolve_workspace_for_wp`, `specify_cli.coordination.status_service.EventLogReadContract`, `specify_cli.status.models.TransitionRequest`, `runtime.next.runtime_bridge.DecideNextContext`, the prompt builder entry point in `runtime/next/prompt_builder.py` (pin the function WP12 converted — read WP12's code; `build_prompt` at `:61` on the planning base is the public entry), and the review-base helper `claim_commit_for_wp` in `src/mission_runtime/claim_commit.py` (WP12). Implement it by parsing the defining module and checking the argument or class field, not by importing (keeps the gate AST-only).
  2. For each gate, assert on the full offender list and print it in the failure message as `path:line kind detail`, so the red commit is self-describing.
  3. Add a small non-vacuity floor per gate (the contract's "floor on the planning base"): the gate must evaluate at least one real candidate site (for example G4 must have scanned ≥ 1 file under `src/specify_cli/`), so an emptied scan root can never pass vacuously.
  4. Expose the allowlist constant the ratchet will read in T097, e.g. `_OWNED_ROOT_BARE_PATH_ALLOWLIST: frozenset[str] = frozenset()`, and assert inside G4/G5 that offenders not in it fail.
  5. Run the file; confirm every gate G1, G2, G4, G5, G6 is red and G3 is green-or-red as the tree dictates. Record the counts.
  6. Commit **only this file** first: `test(arch): owned-checkout single-authority gate G1–G6, committed red (WP18/T095)` with a body such as `Offenders at red: G1=<n> G2=<n> G3=<n> G4=<n> G5=<n> G6=<n missing pins>`.
- **Files**: `tests/architectural/test_owned_checkout_single_authority.py` (new). Read-only: `tests/architectural/_owned_checkout_scan.py`, `tests/architectural/test_owned_checkout_gate_selftest.py`, `tests/architectural/_ast_scan.py`.
- **Parallel?**: No; must be the first commit.
- **Validation checklist**:
  - [ ] The commit contains exactly one file and it is red with the recorded counts (paste the pytest summary into the Activity Log).
  - [ ] Comments and docstrings containing `effective_root` are not counted (verify against a docstring such as `specify_cli/doctrine/snapshot.py:93`).
  - [ ] `src/charter/**` is excluded by module rule, with a written reason in the gate docstring.
- **Edge cases**: string forms of annotations (`"Path | None"`) and `from __future__ import annotations` both produce string/AST annotations — rely on the scanner's annotation normaliser. TypedDict keys are class-body `AnnAssign`s. `**{"effective_root": x}` splats are `Dict` keys inside a `Call`'s `**` keyword. A walrus binding (`(effective_root := ...)`) is a `NamedExpr` target — G4 bans it (see T097 for the one planning-base instance).

### Subtask T096 – Delete the dual keywords, `effective_root_kwargs`, the `OwnedMission` alias and the legacy properties

- **Purpose**: the plan's temporary scaffolding ends here. "The legacy keyword is deleted in the closing WP, together with `effective_root_kwargs`. Nothing ships in between." These deletions are what turns G4/G5 green.
- **Steps** (each bullet is one out-of-map closure edit; list each in the commit message as `<path>: <rationale>`; line numbers are planning-base anchors, re-locate them on your lane base):
  1. **`placement_seam` / `PlacementSeam`** — `src/mission_runtime/resolution.py:2491` (`placement_seam(repo_root, mission_slug, *, effective_root: Path | None = None)`) and the `PlacementSeam.effective_root` field (`:1981`). Delete the legacy keyword and field and whatever WP04 added to reconcile them with `owned`. Rationale: "closing WP: dual keyword retired per Staging Strategy (WP04-owned)".
  2. **`mission_context_for`** — `resolution.py:1040-1046`. Same deletion, same rationale.
  3. **`resolve_action_context`** — `resolution.py:2507-2517`. Same.
  4. **`resolve_workspace_for_wp`** — `src/specify_cli/workspace/context.py:699` (the legacy keyword was added by WP05; the planning-base signature has neither). Rationale: "closing WP: dual keyword retired (WP05-owned)".
  5. **`locate_work_package`** — `src/specify_cli/task_utils/support.py:563-564` (legacy keyword), the `effective_root_kwargs` splat at `:587`, the `if effective_root is not None` at `:594`, and the import at `:15`. Rationale: "closing WP: dual keyword retired (WP04-owned)".
  6. **`TransitionRequest`** — `src/specify_cli/status/models.py:899` (`effective_root: Path | None = None`) and, if WP07 kept it as part of the dual shape, `owned_mission: OwnedMission | None` (`:908`) plus the `TYPE_CHECKING` import at `:24-26`. Rationale: "closing WP: dual field retired (WP07-owned)".
  7. **`effective_root_kwargs` / `_EffectiveRootKwargs`** — `src/specify_cli/core/owned_mission.py:41-47`. Rationale: "closing WP: bridging helper retired (WP02-owned)".
  8. **`OwnedMission` alias** — `src/specify_cli/core/owned_mission.py` (the transitional alias WP02/T010 left at the former class site `:18`) and any re-export. Rationale: "closing WP: rename complete, no alias (occurrence_map code_symbols: rename)".
  9. **`OwnedCheckout` transitional legacy properties** (`primary`, `root`, `directory`, `slug`, `target`) — `src/mission_runtime/owned_checkout.py` (WP01/T001). Rationale: "closing WP: legacy attribute names retired (WP01-owned)". Remove any `mission_runtime` public-surface entry they added (check `tests/architectural/test_mission_runtime_surface.py` and `_PUBLIC_SURFACE`).
  10. After each deletion, `grep -rn` for the removed name across `src/` **and** `tests/` and collect the breakages for T097; do not fix them inside this step.
- **Files**: `src/mission_runtime/resolution.py`, `src/specify_cli/workspace/context.py`, `src/specify_cli/task_utils/support.py`, `src/specify_cli/status/models.py`, `src/specify_cli/core/owned_mission.py`, `src/mission_runtime/owned_checkout.py` (all out-of-map closure edits).
- **Parallel?**: No; after T095.
- **Validation checklist**:
  - [ ] `grep -rnw "effective_root_kwargs\|_EffectiveRootKwargs\|OwnedMission" src/` prints nothing.
  - [ ] `grep -rn "def placement_seam\|def mission_context_for\|def resolve_action_context\|def resolve_workspace_for_wp\|def locate_work_package" -A8 src | grep effective_root` prints nothing.
  - [ ] `tests/architectural/test_mission_runtime_surface.py` passes (MR-1/MR-2 surface unchanged apart from removed transitional names).
  - [ ] `tests/architectural/test_compat_shims.py` passes (no shim left behind).
- **Edge cases**: `mission_runtime/resolution.py` is covered by the `mission_runtime` outbound ledger (`_baselines.yaml` `test_layer_rules.mission_runtime_allowed_specify_cli: 10`); deleting code must not add an import. If any deletion leaves a function at complexity > 15 (it should reduce it), fix it in place.

### Subtask T097 – Sweep leftover owned-root identifiers → gate green; `_baselines.yaml` cap 0

- **Purpose**: drive G1–G6 to green with an empty allowlist and pin the zero with the ratchet meta-test.
- **Steps**:
  1. Run the gate; fix every remaining offender at its source by converting it to `owned: OwnedCheckout | None` (never by adding an exemption). Expected residue: the bridges earlier WPs logged in their Activity Logs (search WP13–WP17 prompt Activity Logs for "bridge"), plus anything the scanner finds that mypy never did. Each fix in another WP's file is an out-of-map closure edit with a rationale line.
  2. Rename the one org-pack **local** binding G4 catches on the planning base: `src/specify_cli/charter_runtime/lint/checks/org_layer.py:327-330` binds `effective_root := pack.effective_root(repo_root)` in a comprehension. Rename the local (for example `pack_root`), leaving the method call intact. Rationale: "G4 bans the identifier as a binding; the org-pack method call stays under the symbol rule".
  3. Re-point the **tests** broken by T096 (tests are not scanned by G4, but they must stay green). Planning-base test files that use the retired names outside the org-pack sense (27 files; confirm with `grep -rlnw "effective_root\|effective_root_kwargs\|OwnedMission" tests --include=*.py`): `tests/agent/test_context_validation_unit.py`, `tests/architectural/test_git_topology_one_copy.py`, `tests/architectural/test_read_surface_placement_guard.py` (`mission_context_for(..., effective_root=repo)` at `:447, 464, 480, 496`), `tests/architectural/test_single_mission_surface_resolver.py`, `tests/consolidation/test_pr_merge_baseline.py`, `tests/integration/test_explicit_checkout_commands.py`, `tests/migration/test_runtime_feature_dir_threading.py` (constructs `OwnedMission(...)`), `tests/mission_runtime/test_builder_fs_free_identity.py`, `tests/mission_runtime/test_coord_read_seam_callers.py`, `tests/mission_runtime/test_owned_single_branch_ssot.py`, `tests/mission_runtime/test_resolution_target_branch.py`, `tests/mission_runtime/test_status_surface_dir_deleted_arm.py`, `tests/next/test_decision_unit.py`, `tests/next/test_runtime_bridge_unit.py`, `tests/runtime/test_artifact_presence_placement.py`, `tests/specify_cli/cli/commands/agent/test_owned_checkout_move_task.py`, `tests/specify_cli/cli/commands/agent/test_tasks_mark_status_recovery.py` (`OwnedMission(...)` fixture), `tests/specify_cli/cli/commands/agent/test_tasks_move_task_pre_review_identity_read.py` (`OwnedMission(...)` at `:182`), `tests/specify_cli/cli/commands/test_next_answer_effective_root.py`, `tests/specify_cli/cli/commands/test_next_owned_commit_guard.py`, `tests/specify_cli/coordination/test_status_transition.py`, `tests/specify_cli/core/test_must_not_flip_anchors.py`, `tests/specify_cli/next/test_next_invocation_lifecycle_seam.py`, `tests/status/test_bootstrap.py`, `tests/status/test_transition_pipeline.py`. Most will already have been re-pointed by their owning WPs; fix only what is still red, re-pointing to `owned=` with a fact minted by the WP02 test helper, assertions unchanged. The org-pack test files (`tests/charter/test_pack_context.py`, `tests/doctrine/test_org_pack_*.py`, `tests/specify_cli/doctrine/*`, `tests/specify_cli/cli/commands/test_doctrine_collect.py`) use the unrelated org-pack sense and must not change.
  4. **Ratchet registration.** Add to `tests/architectural/_baselines.yaml`, following its schema (top-level section = gate test-module name, leaf = integer, `# justification:` comment):
     ```yaml
     # tests/architectural/test_owned_checkout_single_authority.py — owned-checkout
     # single-authority gate (mission owned-checkout-lifecycle-authority-01M3M2ZB,
     # FR-001/SC-004). Empty by operator decision 01M3M65D: no bare owned-root
     # path parameter or field anywhere in src/; growth is a gate evasion.
     test_owned_checkout_single_authority:
       owned_root_bare_path_params: 0  # justification: empty allowlist by construction; never grows.
     ```
     A top-level scalar is refused by `test_yaml_leaves_refuses_a_scalar_section`, so the section/leaf shape is mandatory.
  5. **Meta-test wiring (out-of-map).** `tests/architectural/test_ratchet_baselines.py` fails any YAML leaf without a `_SIZE_RATCHETS` row (`test_every_baseline_leaf_is_enforced_by_a_size_ratchet`) and pins the section count (`test_size_ratchet_table_meets_floor`: `assert len(_REQUIRED_TOP_LEVEL_KEYS) == 14`). Add one row `_SizeRatchet("test_owned_checkout_single_authority", "owned_root_bare_path_params", "tests.architectural.test_owned_checkout_single_authority", "_OWNED_ROOT_BARE_PATH_ALLOWLIST")` and bump the floor to 15 with its comment. That file is not owned by any WP in this mission: declare it as an out-of-map closure edit ("register the new gate's zero cap with the ratchet meta-test, as every _baselines.yaml leaf must be").
  6. Run the gate and the ratchet meta-test until green; commit `fix(owned-checkout): retire transitional surfaces; single-authority gate green (WP18/T096-T097)`.
- **Files**: `tests/architectural/_baselines.yaml` (owned); `tests/architectural/test_owned_checkout_single_authority.py` (owned); out-of-map: `tests/architectural/test_ratchet_baselines.py`, `src/specify_cli/charter_runtime/lint/checks/org_layer.py`, any residual source file from step 1, the test files in step 3 that are still red.
- **Parallel?**: No; after T096.
- **Validation checklist**:
  - [ ] `uv run --frozen pytest tests/architectural/test_owned_checkout_single_authority.py tests/architectural/test_ratchet_baselines.py tests/architectural/test_owned_checkout_gate_selftest.py -q` green.
  - [ ] `_OWNED_ROOT_BARE_PATH_ALLOWLIST` is empty and the YAML leaf is 0.
  - [ ] `grep -rnw "effective_root" src --include=*.py | grep -v "^src/charter/"` prints only org-pack method calls (`.effective_root(`) and docstrings/comments.
  - [ ] Each out-of-map edit appears in the commit message with its rationale.
- **Edge cases**: `occurrence_map.yaml` marks `_baselines.yaml` as `manual_review` ("may only shrink or gain the new gate's zero cap; never loosen") — do not touch any other leaf. `mypy --strict` on the files you edited is mandatory here: `follow_imports = "skip"` means only files named on the command line are fully checked.

### Subtask T098 – SC-001 end-to-end walkthrough test (quickstart), incl. stale copy and cwd matrix

- **Purpose**: SC-001 and NFR-001: using only documented `--owned-checkout` commands, take an owned Mission from create to WP01's review prompt with 0 errors and 0 differences in the repository root checkout, regardless of where the operator stands.
- **Steps**:
  1. Create `tests/integration/test_owned_lifecycle_acceptance_e2e.py` with `pytestmark = [pytest.mark.integration, pytest.mark.git_repo]`. Invoke commands in process with `typer.testing.CliRunner` against the assembled root app (`from specify_cli import app` — `__all__` exports it; confirm it resolves to the assembled Typer, otherwise use the per-command sub-apps the WP08–WP13 acceptance files use). In-process keeps the test on the per-PR path (plan: Test Layout).
  2. Use the WP02 fixtures: `owned_checkouts` (R plus a linked worktree P on its own branch), `r_snapshot` (R's working tree including ignored files, excluding exactly P's resolved subtree, R's `HEAD` and index, the shared lock root `<common-dir>/spec-kitty-locks`, an isolated `SPEC_KITTY_HOME`), `stale_root_copy` (a stale copy of the mission under R).
  3. Script the `quickstart.md` sequence, asserting exit 0 and parsing `--json` output at each step:
     1. `agent mission create demo --mission-type software-dev --owned-checkout P --target-branch <P's branch> --topology single_branch --json` → capture `mission_slug`/`mid8`.
     2. write `spec.md` in P; `spec-commit --owned-checkout P --mission <slug> <spec.md> <meta.json>`.
     3. `agent mission setup-plan --owned-checkout P --mission <slug> --json` → `plan.md` exists and is committed in P.
     4. `agent tasks status --owned-checkout P --mission <slug> --json`.
     5. write `tasks.md` and a one-WP `tasks/WP01-*.md` in P (minimal valid frontmatter; `owned_files` pointing at a file in P); `agent mission finalize-tasks --owned-checkout P --mission <slug> --json`.
     6. `next --agent claude --owned-checkout P --mission <slug> --result success --json` → decision is implement WP01 with workspace == P.
     7. `agent tasks move-task WP01 --to claimed`, then `--to in_progress`, commit an implementation change in P, then `--to for_review`, each with `--owned-checkout P --mission <slug>`.
     8. `next --agent claude --owned-checkout P --mission <slug> --json` → a review prompt whose base is WP01's claim commit (FR-010: the unique commit whose diff to `status.events.jsonl` introduced the last `to_lane=claimed` event id).
     9. `agent context resolve --owned-checkout P --action implement --mission <slug> --wp-id WP01 --json` → `resolution_kind == "owned_checkout"`, `lane_id is None`, `wp_file` and `workspace_path` under P.
  4. After every step assert: every path in the JSON payload resolves under P (walk the payload for strings that look like absolute paths); `r_snapshot` shows 0 differences.
  5. Parametrise the whole walk over `cwd ∈ {"R", "P", "elsewhere"}` (`monkeypatch.chdir`), and over `stale ∈ {False, True}`. With `stale=True`, every payload of the five FR-007 reads (`agent tasks status`, `setup-plan`, `context resolve`, `next`, `finalize-tasks`) carries a non-null top-level `stale_repository_root_copy` whose `path` is under R and whose `mission_id` matches, and no other path is under R; with `stale=False` the field is `null`.
  6. Add the two documented refusals as a short tail: `agent action implement --owned-checkout P ...` and `agent action review --owned-checkout P ...` exit ≠ 0 with `error_code == "OWNED_ACTION_UNSUPPORTED"`, and R is unchanged.
  7. Keep the helper functions small (complexity ≤ 15) and the file self-explanatory: one function per quickstart step, a single `_walk(runner, P, slug)` driver.
- **Files**: `tests/integration/test_owned_lifecycle_acceptance_e2e.py` (new). Read-only: `quickstart.md`, `tests/integration/conftest.py`, the WP08–WP13 acceptance files.
- **Parallel?**: Independent of T095–T097 in code, but it must run green on the post-T097 tree; write it after T097.
- **Validation checklist**:
  - [ ] 6 parametrisations (3 cwd × 2 stale) pass; total runtime noted in the Activity Log (target: well under the per-PR budget; if it exceeds ~60 s, keep only `cwd=P, stale=True` per PR and mark the rest `slow` with a rationale).
  - [ ] Temporarily reverting one seam fix (for example make the placement seam ignore `owned`) turns the test red (mutation check; do not commit).
  - [ ] No `sleep`, no network, `SPEC_KITTY_HOME` isolated.
- **Edge cases**: R's own lane and coordination worktrees must stay inside the snapshot (the fixture excludes exactly P's subtree, never a blanket `.worktrees/`). When cwd is "elsewhere", commands must still resolve R from `--owned-checkout`, not from cwd. The claim commit is found with `git log -S<event_id>`; the test must not assume a commit subject.

### Subtask T099 – NFR-002 / NFR-003 measurements (`performance` marker)

- **Purpose**: NFR-002 (1 ownership validation per owned command; 0 additional per-read git subprocesses for owned status reads versus the non-owned path) is the per-PR proxy; NFR-003 (median of 5 < 2 s for owned `agent tasks status`, `setup-plan`, `context resolve`) is the nightly wall-clock check.
- **Steps**:
  1. **NFR-002 validation count (per PR)** — in `tests/integration/test_owned_lifecycle_acceptance_e2e.py`: a fixture that wraps the real `specify_cli.core.checkout_ownership.resolve_ownership_claim` with a counter (module-attribute monkeypatch; `owned_mission` imports it lazily inside the validator, so the patch is seen) and calls `specify_cli.workspace.context.clear_workspace_resolution_caches()` before each command. Reuse the T098 driver with the counter enabled and assert the counter increments by **exactly 1** for every owned command in the walk (0 or ≥ 2 fails, FR-003).
  2. **NFR-002 subprocess count (per PR)** — same file: count `git` subprocess invocations (wrap `subprocess.run` and `subprocess.Popen` where argv[0] ends with `git`) during one owned `agent tasks status --json` and during one non-owned `agent tasks status --json` on an equivalent mission in a plain repository root checkout; assert `owned - non_owned <= 0` after subtracting the validator's own probes (measure the validator's probes separately with the same counter around a direct validation, and record all three numbers in the assertion message).
  3. **NFR-003 latency (nightly)** — `tests/performance/test_owned_checkout_perf.py`, every test `@pytest.mark.performance` (collected but skipped unless `SPEC_KITTY_RUN_PERFORMANCE=1`, per `tests/conftest.py`). Build the acceptance fixture once (module-scoped: R + P + a finalized one-WP owned mission), then for each of the three commands run it 5 times through the real entry point (`subprocess.run([sys.executable, "-m", "specify_cli", ...], cwd=P, timeout=60)`, mirroring `tests/performance/test_cli_startup_budget_4409.py:285-302`), take the median of the wall-clock times, and assert with `tests._perf_helpers.assert_timing_budget(median, 2.0, name="<command> owned median-of-5")`. Treat a non-zero exit as `float("inf")` with stderr in the name, like the startup test.
  4. Keep functional assertions (exit code, payload shape) out of the performance file's timing tests or duplicate them per-PR in the e2e file: `tests/architectural/test_performance_marker_guard.py` and `test_timing_coverage_invariant.py` police the split between timing and functional asserts.
- **Files**: `tests/integration/test_owned_lifecycle_acceptance_e2e.py` (append), `tests/performance/test_owned_checkout_perf.py` (new).
- **Parallel?**: After T098 (it reuses the driver).
- **Validation checklist**:
  - [ ] `SPEC_KITTY_RUN_PERFORMANCE=1 uv run --frozen pytest tests/performance/test_owned_checkout_perf.py -m performance -q` passes locally; record the three medians in the Activity Log.
  - [ ] Without the env var the performance tests are collected and skipped (not errored).
  - [ ] The count tests fail when a second validation is planted (mutation check; do not commit).
- **Edge cases**: the first run of a command in a fresh interpreter pays import cost; the median of 5 absorbs it, do not add warm-up runs that are excluded from the median. Workspace resolution caches are process-global: clear them between in-process counted commands or the counts go to 0 on reuse.

### Subtask T100 – Terminology guard, docs freshness, final targeted validation

- **Purpose**: close the mission's quality gates (NFR-005) on exactly the files and gates this mission implicates, without running whole heavy suites locally.
- **Steps**:
  1. Terminology: `uv run --frozen pytest tests/architectural/test_no_legacy_terminology.py -q`. Then grep this WP's new files and every out-of-map edit for "feature" and bare "primary" used as a checkout alias; fix prose (C-008). The guard does not check Mission vs feature, so the grep is the check.
  2. Docs freshness: `uv run --frozen pytest tests/docs/test_check_docs_freshness.py tests/docs/test_docs_freshness_invariant.py tests/docs/test_docs_index_freshness.py tests/docs/test_check_cli_reference_freshness.py -q`. The new `--owned-checkout` flags (WP08/WP09) change CLI help; if the CLI reference is stale, regenerate it with `scripts/docs/build_cli_reference.py` and name the regenerated file in the Activity Log (out-of-map, generated artefact).
  3. Specific architectural gates implicated by this mission (run these files, not the directory):
     ```bash
     uv run --frozen pytest -q \
       tests/architectural/test_owned_checkout_single_authority.py \
       tests/architectural/test_owned_checkout_gate_selftest.py \
       tests/architectural/test_ratchet_baselines.py \
       tests/architectural/test_mission_runtime_surface.py \
       tests/architectural/test_layer_rules.py \
       tests/architectural/test_compat_shims.py \
       tests/architectural/test_no_dead_symbols.py \
       tests/architectural/test_read_surface_placement_guard.py \
       tests/architectural/test_single_mission_surface_resolver.py \
       tests/architectural/test_lifted_cli_accept_coord_surface_anchor.py \
       tests/architectural/test_cli_error_surface_seam.py \
       tests/architectural/test_performance_marker_guard.py \
       tests/architectural/test_timing_coverage_invariant.py \
       tests/architectural/test_fast_tier_marker_completeness.py \
       tests/architectural/test_marker_job_completeness.py \
       tests/architectural/test_no_legacy_terminology.py \
       tests/architectural/test_ruff_format_enforcement.py
     ```
  4. Mission acceptance files end to end: `uv run --frozen pytest tests/integration/test_owned_lifecycle_acceptance_*.py tests/integration/test_owned_fixtures_selftest.py -q`.
  5. Baseline: `make test-fast`.
  6. Quality on everything this WP touched (new files + every out-of-map file):
     ```bash
     uv run --frozen ruff check <touched files>
     uv run --frozen ruff format --check <touched files>
     uv run --frozen mypy --strict <touched src files> tests/architectural/test_owned_checkout_single_authority.py \
       tests/integration/test_owned_lifecycle_acceptance_e2e.py tests/performance/test_owned_checkout_perf.py
     ```
     Pass the source files you edited in T096/T097 to mypy **together** (the `specify_cli.*` `follow_imports = "skip"` override hides cross-module errors otherwise).
  7. Occurrence-map compliance check: `grep -rnw "OwnedMission" src/ tests/ docs/context docs/adr/3.x` → only historical mentions allowed by `occurrence_map.yaml` exceptions (`kitty-specs/**`, `docs/adr/2.x/**`, `CHANGELOG.md`) and the WP03 ADR amendments' explicit "formerly `OwnedMission`" notes. Record the result.
  8. Record every command with pass/fail counts in the Activity Log for the PR's *Tests run* section. Any red that is also red on the planning base is baseline red: classify per CLAUDE.md, file an issue if new (Pre-existing Failure Reporting Rule), do not fix here.
- **Files**: no new files; possibly a regenerated CLI reference (out-of-map, generated).
- **Parallel?**: No; last.
- **Validation checklist**:
  - [ ] All commands above green (or baseline red, classified with an issue link).
  - [ ] Activity Log carries the command list and counts.
- **Edge cases**: `tests/architectural/test_marker_job_completeness.py` and `test_fast_tier_marker_completeness.py` require every new test file to carry a tier marker the CI jobs select; the three new files use `architectural`, `integration`+`git_repo`, and `performance`, which are all registered in `pytest.ini`.

## Test Strategy

- New tests: the gate (`architectural`), the e2e walkthrough plus NFR-002 counts (`integration`, `git_repo`, per PR), NFR-003 latency (`performance`, nightly).
- Commands (summary; the full list is in T100):

```bash
uv run --frozen pytest tests/architectural/test_owned_checkout_single_authority.py -q          # red at T095, green at T097
uv run --frozen pytest tests/architectural/test_ratchet_baselines.py tests/architectural/test_owned_checkout_gate_selftest.py -q
uv run --frozen pytest tests/integration/test_owned_lifecycle_acceptance_e2e.py -q
SPEC_KITTY_RUN_PERFORMANCE=1 uv run --frozen pytest tests/performance/test_owned_checkout_perf.py -m performance -q
make test-fast
uv run --frozen ruff check <touched> && uv run --frozen ruff format --check <touched>
uv run --frozen mypy --strict <touched typed files, in one invocation>
```

Record exact commands and pass/fail counts in the Activity Log and the PR's *Tests run* section. A mypy failure fails the WP even when pytest is green.

## Risks & Mitigations

- **Vacuous gate.** A scanner that silently scans nothing passes everything. Mitigation: per-gate non-vacuity floor (T095 step 3) plus WP01's self-mutation tests.
- **Exemption creep.** Pressure to "just allowlist" a hard site. Mitigation: empty allowlist by operator decision; the ratchet leaf is 0; reviewers reject any new exemption.
- **Wide out-of-map blast radius.** T096/T097 touch files owned by WP01, WP02, WP04, WP05, WP07 and possibly others, and ~27 test files. Mitigation: one rationale line per edit; deletions only; targeted tests per touched module; the gate plus the e2e walk as the end-state oracle.
- **Ratchet meta-test coupling.** Adding a `_baselines.yaml` leaf without a `_SIZE_RATCHETS` row reds `test_ratchet_baselines.py`. Mitigation: T097 step 5.
- **Slow e2e.** Mitigation: in-process CliRunner; the cwd×stale matrix trimmed to one per-PR case plus `slow` if it exceeds budget (T098 checklist).

## Review Guidance

- Verify the commit order: T095's gate file alone, red, with offender counts in the message; then the deletions and sweep; then the e2e and perf files.
- Re-run `tests/architectural/test_owned_checkout_single_authority.py` and `test_ratchet_baselines.py`; both green, allowlist empty, YAML leaf 0.
- Check each out-of-map edit is listed with a one-line rationale and is a deletion or a re-point, nothing more.
- Check G4's org-pack exemption is by symbol/module rule as the contract says, and that the walrus local in `org_layer.py` was renamed rather than exempted.
- Run the e2e test yourself for `cwd=P, stale=True` and inspect that `stale_repository_root_copy` is present and no path lands under R.
- Confirm the implementer ran mypy across all touched source files in one invocation and it passed.

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
