---
work_package_id: WP11
title: next runtime takes the fact
dependencies:
- WP05
- WP07
- WP08
requirement_refs:
- FR-002
- FR-007
- FR-008
- FR-009
- FR-012
- FR-022
- FR-023
planning_base_branch: claude/sleepy-hamilton-5lelee
merge_target_branch: claude/sleepy-hamilton-5lelee
branch_strategy: Planning artifacts for this mission were generated on claude/sleepy-hamilton-5lelee. During /spec-kitty.implement this WP may branch from a dependency-specific base, but completed changes must merge back into claude/sleepy-hamilton-5lelee unless the human explicitly redirects the landing branch.
subtasks:
- T056
- T057
- T058
- T059
- T060
- T061
- T062
- T063
- T066
- T067
phase: Phase 3 - Commands and runtime
history:
- at: '2026-09-28T15:00:00Z'
  actor: system
  action: Prompt generated via /spec-kitty.tasks
agent_profile: python-pedro
authoritative_surface: src/runtime/next/
create_intent:
- tests/integration/test_owned_lifecycle_acceptance_next.py
execution_mode: code_change
model: claude-sonnet-5
owned_files:
- src/specify_cli/cli/commands/next_cmd.py
- src/runtime/next/runtime_bridge.py
- src/runtime/next/runtime_bridge_engine.py
- src/runtime/next/runtime_bridge_io.py
- src/runtime/next/next_invocation_lifecycle.py
- src/specify_cli/coordination/workspace.py
- tests/integration/test_explicit_checkout_commands.py
- tests/next/test_runtime_bridge_unit.py
- tests/specify_cli/cli/commands/test_next_answer_effective_root.py
- tests/integration/test_owned_lifecycle_acceptance_next.py
- src/runtime/next/decision.py
- src/runtime/next/runtime_bridge_composition.py
role: implementer
tags: []
task_type: implement
tracker_refs: []
---

# Work Package Prompt: WP11 – next runtime takes the fact

## ⚡ Do This First: Load Agent Profile

Use the `/ad-hoc-profile-load` skill to load the agent profile in the frontmatter, and follow its guidance before you read the rest of this prompt.

- **Profile**: `python-pedro`
- **Role**: `implementer`
- **Agent/tool**: `claude`

Then load the charter (`.kittify/charter/charter.md`) and the action context: `spec-kitty charter context --action implement --json`.

---

## ⚠️ IMPORTANT: Review Feedback

- **Has review feedback?** Check `review_ref` in the event log (`spec-kitty agent tasks status --mission owned-checkout-lifecycle-authority-01M3M2ZB`) or the Activity Log below.
- **Address every item** before moving back to `for_review`, and log each fix in the Activity Log.

---

## Review Feedback

*[Empty until this WP is returned from review.]*

---

## Markdown Formatting

Wrap HTML/XML tags in backticks: `` `<div>` ``, `` `<script>` ``
Use language identifiers in code blocks: ````python`, ````bash`

---

## Objectives & Success Criteria

`spec-kitty next --owned-checkout P` is the canonical owned control loop (decision `01M3M4GZ…`). This WP makes it validate once, through the sole minter, and carry the resulting `OwnedCheckout` fact through the whole runtime:

| Row | Outcome | Proof (red on base → green here) |
|---|---|---|
| FR-002 / O9 | `next --owned-checkout` goes through `resolve_owned_or_adopt(…, NEXT_OWNED_TOPOLOGIES)` and gets branch and protection checks. A protected target is refused with `OWNED_BRANCH_REFUSED`. | CLI test with a protected target |
| FR-023 | A `lanes_with_coord` mission is accepted by `next`, and the same mission is refused by a lifecycle command with `OWNED_TOPOLOGY_UNSUPPORTED`. | paired test |
| FR-008 / O5 | `next --result success` at tasks→implement raises no traceback. The WP workspace is resolved **before** the advance is persisted, and the run is not wedged. `decision._state_to_action` takes the fact (T067). | CLI test (runtime half green here; the full-prompt US3-AS1 assertion is a strict xfail that WP12 removes, declared) |
| FR-009 | Composition policy, task-board resolution and the git-log working directory are read from P. | carried 1be5352ee composition test plus the board test |
| FR-007 | With a stale copy of M in R, the `next` payload carries `stale_repository_root_copy`, and no workspace path is under R. | CLI test with the `stale_root_copy` fixture |
| FR-012 / O8 | A failing coordination-workspace probe produces `error_code == "OWNED_COORDINATION_WORKSPACE_UNAVAILABLE"` and never a `commondir` fatal. The transient-lock retry is kept. | two deterministic injections |
| FR-022 | Coordination-topology owned `next` still works. There is a per-PR in-process twin (owned `lanes_with_coord` create → `next`). | new twin plus the unchanged unit tests |
| FR-003 | Exactly one ownership validation per `next` invocation. | call-count assertion |

**Done when:** every row above has a red-first test committed before its fix; all targeted files are green; `_emit_checkout_ownership_error` and every `resolve_ownership_claim` call are gone from `next_cmd.py`; `next_cmd.py`, `decision.py`, `runtime_bridge_composition.py` and the `runtime/next` bridge modules owned here contain no `effective_root` parameter, field or keyword that this WP could convert (see T063 for the permitted bridging into `prompt_builder.py`, and the six shared seams' `TRANSITIONAL(WP18)` dual keyword, which stays); ruff, format and mypy are clean; and `make test-fast` is green.

## Context & Constraints

- **Read first:**
  - `spec.md`: US3, O5, O8, O9, FR-002/007/008/009/012/022/023.
  - `plan.md`: Staging Strategy, Test Layout, IC-05, IC-13.
  - `research.md`: R-02, R-06, R-08, R-11, R-12, R-15, R-16.
  - `data-model.md`: registry codes and the `stale_repository_root_copy` payload.
  - `contracts/owned-checkout-carrier.md` §2 and §7.
  - `contracts/cli-owned-checkout-surface.md`: the `next` row.
  - `occurrence_map.yaml`.
- **What the dependencies provide.** Read their merged code, not the plan, before you start.
  - **WP01**: `mission_runtime.OwnedCheckout` with `repository_root`, `owned_root`, `mission_dir`, `mission_slug`, `topology`, `target_branch`, plus transitional legacy properties.
  - **WP02**: `resolve_owned_mission(..., allowed_topologies)` and `NEXT_OWNED_TOPOLOGIES`; the shared fixtures `owned_checkouts`, `r_snapshot` and `stale_root_copy` in `tests/integration/conftest.py`.
  - **WP04/WP05**: `placement_seam`, `mission_context_for`, `resolve_action_context`, `resolve_workspace_for_wp` and `locate_work_package` accept `owned=` (dual keyword, marked `TRANSITIONAL(WP18)`, until WP18). Workspace caches are keyed on the resolved tasks dir.
  - **WP07**: `TransitionRequest.owned`.
  - **WP08**: `cli/commands/_owned_checkout.py` with `OwnedCheckoutOption`, `resolve_owned_or_adopt`, `emit_owned_refusal` and the stale-copy reporter. Use its **actual** signatures.
- **Root discipline (the core rule of this WP).** The CLI hands the runtime `repo_root = owned.repository_root` (R) plus `owned`. Inside the runtime, when `owned` is set:
  - owned-aware consumers (contract §7: `placement_seam`, `mission_context_for`, `resolve_workspace_for_wp`, `locate_work_package`, `DecideNextContext`) receive `owned=`;
  - callees that are not yet owned-aware receive `owned.owned_root` in the root argument they already have: the run store (`get_or_start_run`), the lifecycle store, `PackContext`/composition, and git cwd. This preserves today's owned behaviour, because today `repo_root` *is* P (`next_cmd.py:176-178`);
  - `repo_root` (R) is used only for repository-root-only concerns and never for mission-state reads or writes.

  Consequence: the run state, the lifecycle record and the coordination worktree stay where they live today for owned missions, which is under P.
- **Staging (top-down).**
  - `next_cmd.py` gets the fact first and the runtime entry points take `owned`.
  - **`src/runtime/next/decision.py` is owned by this WP** (T066 campsite `_state_to_action`, T067 takes the fact, including `decide_next`, the only path from `next_cmd` to `decide_next_via_runtime`). **`src/runtime/next/runtime_bridge_composition.py` is owned by this WP** too (composition policy reads and the presence-probe call site, T060).
  - Callees in the file owned by **WP12** (`src/runtime/next/prompt_builder.py`) are not converted here. They receive the bridging value that preserves today's behaviour: `owned.owned_root if owned else repo_root` as their existing `repo_root` argument, each marked `# bridging: WP12 converts`.
  - Transitional surfaces are the six shared seams plus every other function marked `TRANSITIONAL(WP18)`. Do not remove `effective_root=` from the six seams. Any legacy parameter you must keep for a not-yet-converted caller carries `# TRANSITIONAL(WP18): <reason>`.
- **Campsite first.** Before changing their signatures, extract these functions (behaviour-preserving, in their own commit):
  - `query_current_state` (`runtime_bridge.py:2661`, complexity 15), T058;
  - `next_step` (`next_cmd.py:119`, complexity 12), T058;
  - `_print_standard_human` (`next_cmd.py:1202`, complexity 14), T058;
  - `_state_to_action` (`decision.py:394`, complexity 14), T066.

  Check with `.venv/bin/ruff check --isolated --select C901 --config 'lint.mccabe.max-complexity=11' <file>`.
  - `_run_bootstrap_loop` lives in `src/specify_cli/cli/commands/agent/mission_finalize.py:1480` and is WP13's (T071). `_build_wp_prompt` is WP12's (T068).
- **Terminology.** Use "repository root checkout" and "owned checkout" in code, messages and tests. Rename test locals `primary` to `repository_root` in any test you touch (the `tests_fixtures` rename). Do not use `feature` aliases.
- **Error contract.** Refusals always carry `error_code`, one of the registered codes (NFR-004). Tests assert on `error_code` only, never on git's stderr wording.

## Branch Strategy

- **Strategy:** the planning artifacts were generated on `claude/sleepy-hamilton-5lelee`, and completed changes merge back into `claude/sleepy-hamilton-5lelee`.
- **Planning base branch:** `claude/sleepy-hamilton-5lelee`
- **Merge target branch:** `claude/sleepy-hamilton-5lelee`
- **Execution lane:** assigned by `finalize-tasks` in `kitty-specs/owned-checkout-lifecycle-authority-01M3M2ZB/lanes.json`. Start with `spec-kitty implement WP11` and use the workspace it resolves. Never reconstruct the path yourself.

> These fields are populated automatically by `spec-kitty agent mission finalize-tasks`.

## Subtasks & Detailed Guidance

### Subtask T056 – Carry #5009 edaa9cd83 and 1be5352ee; adapt the call shape

- **Purpose:** C-006 provenance. Both commits (author Samuel Goff) add tests to `tests/integration/test_explicit_checkout_commands.py` and reproduce real defects:
  - **edaa9cd83**: `test_finalized_owned_tasks_resolve_for_next_implementation` (O5 at the decision seam) and `test_wp_cache_is_scoped_to_selected_checkout` (FR-019 cache isolation).
  - **1be5352ee**: extends the cache test; adds `test_owned_composition_policy_reaches_executor_before_advancing` (FR-009 composition, R-11) and `test_owned_review_prompt_uses_owned_target_branch`.
- **Steps:**
  1. Carry both commits verbatim, in order: `git cherry-pick -x edaa9cd83` then `git cherry-pick -x 1be5352ee` (from the local ref `pr-5009`). Author and date are preserved.
  2. Run them and record in the Activity Log why each is red. R-15 says edaa9cd83 is red only through a `TypeError` (`effective_root=` is not accepted), and 1be5352ee's composition test is a genuine red.
  3. **Adapt in a separate commit.** Change the call shape "`repo_root`=R, `effective_root`=P" to "`repo_root`=R, `owned`=fact", where the fact is minted by the **real validator**: `resolve_owned_mission(repository_root, owned, SLUG, allowed_topologies=NEXT_OWNED_TOPOLOGIES)`. Never construct `OwnedCheckout` directly.
     - `test_finalized_owned_tasks_resolve_for_next_implementation` calls `runtime.next.decision._state_to_action(..., effective_root=owned)`. `decision.py` is owned here, so re-point it to `_state_to_action("implement", SLUG, owned / "kitty-specs" / SLUG, repository_root, "software-dev", owned=fact)` (T067) and assert `("implement", "WP01", str(fact.owned_root))`. Add a second assertion through the board authority that O5 actually goes through: `runtime_bridge._wp_iteration_action_and_state("implement", SLUG, "software-dev", owned / "kitty-specs" / SLUG, repository_root, owned=fact)`.
     - The cache test calls `get_normalized_wp(..., effective_root=owned)`. Use the owned keyword that WP05 exposed. If WP05 did not add `owned=` to `get_normalized_wp`, assert the same isolation through `resolve_workspace_for_wp(repository_root, SLUG, "WP01", owned=fact)` and file the gap against WP05 in the Activity Log. Do not edit `workspace/context.py`.
     - Adapt the composition test: `context = SimpleNamespace(..., repo_root=repository_root, owned=fact, ...)`, the `advance` stub asserts `kwargs["owned"] is fact` and `kwargs["repo_root"] == repository_root`, and `inputs`/`execute` assert `repo_root == fact.owned_root`.
     - `test_owned_review_prompt_uses_owned_target_branch` stays as carried. Mark it `pytest.mark.xfail(strict=True, reason="WP12 T069: review assertion rewritten to the claim-commit contract (FR-025)")`. **WP12 owns that rewrite**, so do not rewrite the `{TARGET}..HEAD` assertion here.
  4. **Optional provenance: 22f380055** (retires the formatter exclusion for `runtime_bridge_engine.py`). `runtime_bridge_engine.py` is still in `[tool.ruff.format] exclude` in `pyproject.toml`, and `ruff format --check src/runtime/next/runtime_bridge_engine.py` currently reports "Would reformat". If you edit the engine in T060, you may carry 22f380055 with `-x`. If you do, add a separate pure-format commit of the engine file, and declare the `pyproject.toml` one-line removal as an out-of-map edit with the rationale "occurrence_map pyproject manual_review: formatter-exclude housekeeping (IC-13)". Otherwise record `DROP (engine not reformatted)` in the Activity Log.
- **Files:** `tests/integration/test_explicit_checkout_commands.py`, optionally `pyproject.toml` (out-of-map, declared).
- **Validation checklist:**
  - [ ] `git log --format='%an%n%b' -2` shows Samuel Goff and the `(cherry picked from commit …)` trailers.
  - [ ] Each adapted test is red before its fix commit (T060/T061), and the red reasons are recorded.
  - [ ] The review test is a strict xfail, not a skip.
- **Edge cases:** the carried fixture `checkouts` creates a `sibling` worktree. Keep it: it guards "not read or written".

### Subtask T057 – Red-first acceptance: O5, O8 (both injections), O9, stale-copy board, lanes_with_coord twin

- **Purpose:** C-007. Every `[build]` row gets a reproduction through the **pre-existing entry point**, the real `next` Typer command driven in-process with `typer.testing.CliRunner`. The reproductions go into one new file, committed **before** any `src/` fix.
- **Steps:**
  1. Create `tests/integration/test_owned_lifecycle_acceptance_next.py` with `pytestmark = [pytest.mark.integration, pytest.mark.git_repo]`.
     - Use WP02's shared fixtures from `tests/integration/conftest.py`:
       - `owned_checkouts`, or its factory `make_owned_checkouts(*, topology=…, placement="sibling"|"under_worktrees", wp_ids=…, protected_target=…)`. It returns `OwnedCheckouts(repository_root, owned_root, sibling, mission_slug, mission_id, mid8, target_branch, mission_dir)`. It is **not** finalized, so run `agent mission finalize-tasks --owned-checkout P --mission H` in the test setup;
       - `owned_handle`, which parametrises over slug, mid8 and id;
       - `owned_cwd`, the NFR-001 cwd matrix;
       - `r_snapshot`, used as `before = r_snapshot.take()` … `r_snapshot.assert_unchanged(before, r_snapshot.take())`;
       - `stale_root_copy`.
     - Do not import fixtures from test modules.
     - Invoke the command the way `tests/specify_cli/cli/commands/` invokes it: through the `next` app/command object from `specify_cli.cli.commands.next_cmd`.
  2. **O9 / FR-002.** Use `make_owned_checkouts(protected_target=True)`. If that flag is missing, set `protection.protected_branches: [main, <P's target>]` in P's `.kittify/config.yaml`, following the existing pattern in `test_explicit_checkout_commands.py:247-260`. Run `next --owned-checkout P --mission H --json`. Expect a non-zero exit and `json["error_code"] == "OWNED_BRANCH_REFUSED"`. Add a same-fixture control without the protection that passes the validator. On the base this is red: the claim-only path at `next_cmd.py:161-178` accepts it.
  3. **FR-023 pairing.** Create a `lanes_with_coord` owned mission (see step 6). `next --owned-checkout P` passes validation. `agent tasks status --owned-checkout P` (WP09) or another lifecycle command refuses the same mission with `OWNED_TOPOLOGY_UNSUPPORTED`. If WP09 has not merged yet, pair with `finalize-tasks --owned-checkout`, which already exists.
  4. **O5 / FR-008.** On a finalized owned mission, run `next --agent claude --owned-checkout P --mission H --result success --json`. Assert:
     - no traceback (`result.exception` is `None` or `SystemExit`);
     - no `ValueError` anywhere in the output;
     - the decision is not `blocked` for a workspace-resolution reason;
     - `workspace_path == str(P)`, `action == "implement"`, `wp_id == "WP01"`;
     - a following `next --owned-checkout P --json` query does **not** return the same unadvanced decision.

     Add the full US3-AS1 row as a **separate** test: `kind == "step"` with an existing `prompt_file` whose workspace is P. Mark it `xfail(strict=True, reason="WP12 T068: prompt builder owned arm")`, because prompt building lives in `prompt_builder.py` (WP12). WP12 removes the marker as a declared out-of-map edit in this file. See Risks.
  5. **O8 / FR-012, two deterministic injections.** Assert only on `error_code == "OWNED_COORDINATION_WORKSPACE_UNAVAILABLE"`.
     - **(a) Zero-byte `commondir`.** Build an owned `lanes_with_coord` mission (step 6). Remove its coordination worktree directory. Add a sibling registered worktree with `git worktree add`, then truncate `<common-dir>/worktrees/<name>/commondir` to **zero bytes**. Pre-assert that `subprocess.run(["git", "-C", R, "worktree", "list", "--porcelain"]).returncode != 0`, and fail loudly with `pytest.fail` (never `skip`) if git tolerates it. R-16: a missing file or a lone newline gives rc 0.
     - **(b) Subprocess seam.** Monkeypatch `specify_cli.coordination.workspace.subprocess.check_output` with a wrapper that raises `subprocess.CalledProcessError(128, argv, output="", stderr="fatal: injected")` **only** when `argv` contains `"worktree", "list"`, and delegates to the real function otherwise. This is the probe at `coordination/workspace.py:163`.

     On the base, both are red: `DecisionGitLogUnavailable` escapes from `runtime_bridge.py:389`.
  6. **FR-022 in-process twin.** Run owned create with `agent mission create <slug> --topology lanes_with_coord --owned-checkout P --start-branch <b> --pr-bound --branch-strategy already-confirmed --json` through `CliRunner`, mirroring `tests/e2e/test_worktree_owned_root_concurrency.py:395-431`. Then run `next --owned-checkout P --mission <slug> --json` (query). Expect exit 0 and a non-error decision. This one should be **green on the base** as a ratchet. Record that it is green.
  7. **Stale-copy board (FR-007).** With the `stale_root_copy` fixture (R holds a copy of M with a different lane map or WP set), run `next --owned-checkout P --result success --json`. Assert that `json["stale_repository_root_copy"]["path"]` names R's copy, and that `workspace_path` is not under R (use `Path.is_relative_to`, and exclude P's subtree when P lives under R).
  8. **NFR-001.** Wrap each owned scenario in the `r_snapshot` fixture and assert 0 differences.
  9. **FR-003 count.** Patch `specify_cli.core.checkout_ownership.resolve_ownership_claim` with a counting wrapper. Clear the workspace caches first with `clear_workspace_resolution_caches()`. Assert exactly **1** call per `next --owned-checkout` invocation.
  10. Commit: `test(next): red-first owned next acceptance (O5, O8, O9, FR-007/022/023)`.
- **Files:** `tests/integration/test_owned_lifecycle_acceptance_next.py` (new).
- **Validation checklist:**
  - [ ] Every `[build]` row is red on the planning base for the stated reason, recorded in the Activity Log.
  - [ ] Each row has a non-vacuity control on the same fixture (the valid P passes, and the twin is green).
  - [ ] No assertion reads git stderr text.
- **Edge cases:**
  - `next` is decorated with `_require_main_repo_unless_owned` (`next_cmd.py:80-98`). With `--owned-checkout`, the guard is bypassed. Run CliRunner with `cwd` = R, P and an unrelated directory (NFR-001 cwd matrix), using `monkeypatch.chdir`.
  - Set `SPEC_KITTY_ENABLE_SAAS_SYNC=0`. The integration conftest autouse fixture already does this.

### Subtask T058 – Campsite: `query_current_state`, `next_step`, `_print_standard_human`

- **Purpose:** Standing Order 2. Functions at complexity ≥ 12 whose signature this WP changes are extracted first, without behaviour change.
- **Steps:**
  1. **`query_current_state`** (`runtime_bridge.py:2661-2855`, complexity 15). Extract:
     - `_query_resolve_mission_context(repo_root, mission_slug, **owned_kw)`, which holds the try/except `ActionContextError` → read-path pass-through / `MissionNotFoundError` mapping (`:2698-2720`);
     - `_query_read_runtime_plan(run_ref, mission_slug, mission_type, repo_root)`, which holds the nested try around the ephemeral run and the planner (`:2754-2777`) and returns `(run_ref, ephemeral_run_store, snapshot, runtime_decision)`;
     - `_query_dispatch_decision(...)`, which holds the finalized-override / initial / decision-required / runtime branch ladder.

     Keep the `finally` rmtree of the ephemeral store in the outer function.
  2. **`next_step`** (`next_cmd.py:119-268`, complexity 12). Extract `_resolve_slug_or_exit(mission, repo_root, json_output, ...)`, which holds the five-arm `except` ladder at `:198-224` and its typed emitters, unchanged. Also extract the owned resolution block (`:159-178`) into `_resolve_next_owned(...)`, which T059 will rewrite.
  3. **`_print_standard_human`** (`next_cmd.py:1202`, complexity 14). Split it by decision kind (one small printer per kind, the outer function a dispatch). This campsite is assigned here, not to WP09. Print the stale-copy warning at command level (T063), not inside the printers.
  4. Commit: `refactor(next): campsite extractions before owned conversion (C901)`.
- **Files:** `src/runtime/next/runtime_bridge.py`, `src/specify_cli/cli/commands/next_cmd.py`.
- **Validation checklist:**
  - [ ] `.venv/bin/ruff check --isolated --select C901 --config 'lint.mccabe.max-complexity=11'` on both files no longer lists the extracted functions, or lists them at ≤ 11.
  - [ ] `tests/next/test_runtime_bridge_unit.py`, `tests/next/` and `tests/specify_cli/cli/commands/test_next*` are green and unchanged.
  - [ ] No monkeypatch target used by existing tests has moved. Grep `tests/` for `runtime_bridge.<name>` and `next_cmd.<name>` patch strings before renaming anything, and keep the names that tests patch.
- **Edge cases:** `query_current_state` is monkeypatched by name in many tests. Keep the public name and its signature (plus the new `owned` keyword in T060).

### Subtask T059 – `next_cmd` goes through `resolve_owned_or_adopt(NEXT_OWNED_TOPOLOGIES)`

- **Purpose:** FR-002 / FR-003 / O9. Replace the claim-only path (`next_cmd.py:159-178`: a direct `resolve_ownership_claim` plus `error_for_claim`, then `repo_root = effective_root = claim.claimed_checkout`) with the single validator. Then delete `_emit_checkout_ownership_error` (`:101-115`) in favour of WP08's `emit_owned_refusal`.
- **Steps:**
  1. Replace the inline `owned_checkout: Annotated[Path | None, typer.Option("--owned-checkout", …)]` (`:132-137`) with WP08's `OwnedCheckoutOption`.
  2. In `_resolve_next_owned` (T058), call `resolve_owned_or_adopt(...)` with `allowed_topologies=NEXT_OWNED_TOPOLOGIES` and the `--mission` handle. On refusal, call `emit_owned_refusal(...)`, keeping `next`'s existing envelope `{success, error_code, error}`, and exit 1. On success, keep `repo_root = owned.repository_root` and hold `owned`.
  3. **Handle-less owned `next`.** `resolve_owned_mission` requires a handle (`owned_mission.py:101-102`). Plan IC-05 says "resolves the handle in P for handle-less `next`". If WP08's helper already supports `handle=None` (discovery inside the claimed checkout after the claim check), use it. If not, run the sole-mission discovery `_missing_handle_or_sole_slug` (`:575`) against the **claimed** checkout path *only after* the helper has confirmed the claim, then validate with that handle. **Never** list P's missions before the claim check. Record which path you took.
  4. **Flagless adoption.** Contract: "owned-capable commands run without the flag call `adopt_owned_checkout`". `next` is currently wrapped by `require_main_repo`, which refuses any linked worktree (`core/context_validation.py:190-225`). Rework `_require_main_repo_unless_owned` (`:80-98`) so that a flagless invocation from inside a linked checkout first attempts validated adoption through `resolve_owned_or_adopt`:
     - a fact → proceed owned;
     - `None` (lane worktree, coordination worktree, rejected checkout) → fall through to the **unchanged** legacy guard. This keeps US7-AS2/AS3.

     Pin both paths in T057's file: adoption from a valid P, and the unchanged refusal from a lane worktree.
  5. Delete `_emit_checkout_ownership_error` and the `TYPE_CHECKING` import of `CheckoutOwnershipError` (`:42-44`) once they have no users.
  6. Thread `owned` (never a bare path) into every downstream call in `next_step`: `_resolve_mission_slug`, `_maybe_handle_answer`, `_dispatch_query_mode` and `_dispatch_advancing_mode`, replacing their `effective_root: Path | None` keywords.
- **Files:** `src/specify_cli/cli/commands/next_cmd.py`.
- **Validation checklist:**
  - [ ] T057's O9 row turns green, and its valid control stays green.
  - [ ] `grep -n "resolve_ownership_claim\|error_for_claim\|_emit_checkout_ownership_error" src/specify_cli/cli/commands/next_cmd.py` returns nothing (G1/G2 floor offender `next_cmd.py:170` removed).
  - [ ] The FR-003 count is exactly 1.
- **Edge cases:** `_run_charter_preflight_for_next` (`:487`) receives `repo_root`. The charter that `next` enforces is P's (consistent with WP10's create-time reads, R-11), so pass `owned.owned_root` when owned. Pin it with a unit test.

### Subtask T060 – `DecideNextContext.owned`, the `_dn_*` phases, composition policy and git cwd

- **Purpose:** FR-009 / R-11. The runtime carries the fact, not a bare root, and reads composition policy from P.
- **Steps:**
  1. Add `owned: OwnedCheckout | None = None` to `DecideNextContext` (`runtime_bridge.py:1561-1588`). Do not add an `effective_root` field.
  2. `decide_next_via_runtime` (`:2447`) and `_dn_bootstrap` (`:1591`) take `owned` instead of `effective_root`. In `_dn_bootstrap`, replace `mission_context_for(repo_root, slug, effective_root=…)` (`:1610-1622`) with `mission_context_for(repo_root, slug, owned=owned)`. Apply the root discipline:
     - `get_or_start_run` (`:1702`) → `owned.owned_root if owned else repo_root`;
     - `_build_operational_context_for_decision` → the same;
     - `resolve_mission_path(mission_type, …)` → the same.
  3. `_dn_composition_dispatch` (`:2020-2138`): let `config_root = ctx.owned.owned_root if ctx.owned else ctx.repo_root`. This is the re-expression of b1a1693bf's runtime half. Use it for `_should_dispatch_via_composition(..., repo_root=config_root)`, `_composition._composition_dispatch_inputs(repo_root=config_root)` and `_dispatch_via_composition(repo_root=config_root)`. Pass `owned=ctx.owned` to `_advance_run_state_after_composition` (`:1314`), which forwards to `runtime_bridge_engine.advance_run_state_after_composition` (`runtime_bridge_engine.py:318`). There, `_emit_terminal` uses the owned root for retrospective policy (`:249`).
  4. `_wrap_with_decision_git_log` (`:293`) and `_mission_routes_through_coordination` (`:240`) take `owned`. Every git cwd, and the `CoordinationWorkspace.resolve` root for an owned coordination mission, uses `owned.owned_root`, which is today's behaviour. The file `DecisionGitLog(repo_root=…)` uses the same root.
  5. `_dn_decision_materialize` (`:2242`) and `_map_runtime_decision` (`:3478`) thread `owned` down to the WP-iteration builders (T061).
  6. Update `next_invocation_lifecycle.py` in the same way: `pair_previous_lifecycle_record` (`:127`), `write_issuance_lifecycle_record` (`:225`) and `emit_mission_next_invoked` (`:316`) take `owned` and call `mission_context_for(..., owned=owned)` instead of the `effective_root` forks (`:167-178`, `:265-276`, `:341-352`).
  7. `runtime_bridge_io.py:1155`: the artifact-presence probe builds a second seam with `effective_root=repo_root`, which guesses at owned placement (R-16: "the presence probe folds into the fact"). Thread `owned` from `gather_artifact_presence` (`:1292`) callers (`runtime_bridge.py:907`, `runtime_bridge_composition.py:543`) into `_artifact_presence_seam` and build the owned seam with `owned=owned`. Remove the guess.
     - `runtime_bridge_composition.py` is owned by this WP. Thread `owned` into its presence-probe call site (`:543`) and make its composition-policy reads (`resolve_mission_type_context(repo_root, …)` `:189`, `:353`; `resolve_org_dirs(repo_root, …)` `:292`; `StepContractExecutor(repo_root=…)` `:654`) take the owned root through `config_root` when owned. Keep its public names stable (tests patch them).
- **Files:** `src/runtime/next/runtime_bridge.py`, `runtime_bridge_engine.py`, `runtime_bridge_io.py`, `runtime_bridge_composition.py`, `next_invocation_lifecycle.py`.
- **Validation checklist:**
  - [ ] The carried composition test (T056, adapted) is green.
  - [ ] Every existing `tests/next/test_runtime_bridge_unit.py` owned-threading test (`:1775-1990`) is re-pointed to `owned=` and green, including `_resolve_owned_coordination_workspace` retry tests at `:1789-1930`, which stay unchanged in behaviour.
  - [ ] No `effective_root` remains in `runtime_bridge.py`, `runtime_bridge_engine.py`, `runtime_bridge_io.py`, `runtime_bridge_composition.py` or `next_invocation_lifecycle.py`, except `TRANSITIONAL(WP18)`-marked lines (T063).
- **Edge cases:**
  - `answer_decision_via_runtime` (`:2858`) has no owned path today, and `_handle_answer` (`next_cmd.py:1074`) passes R. Thread `owned` into it too. Its `resolve_action_context(repo_root, …)` becomes `resolve_action_context(repo_root, …, owned=owned)`, and `get_or_start_run` uses the owned root.
  - Re-point `tests/specify_cli/cli/commands/test_next_answer_effective_root.py` to `owned`. Keep the filename (it is pinned) and update its docstring.

### Subtask T061 – Board authority with the fact; resolve before persisting the advance

- **Purpose:** FR-008 / O5 / R-06. Today `_dn_decision_materialize` persists the advance through `runtime_next_step(...)` (`runtime_bridge.py:2288`). The WP workspace is resolved **afterwards**, in `_map_runtime_decision` → `_build_wp_iteration_decision` (`:3204`) → `_wp_iteration_action_and_state` (`:3172`). On an owned mission:
  - `_resolve_wp_board_action(repo_root=P)` reads the repository root checkout through `placement_seam(P, …)`;
  - it declines on `ActionContextError`;
  - it falls back to `_state_to_action`, which folds to R and raises `ValueError`;
  - the run is left at `implement` (the wedge).
- **Steps:**
  1. `_resolve_wp_board_action(*, mission_slug, repo_root, owned=None)` (`:3081`) builds `placement_seam(repo_root, slug, owned=owned)` and `mission_context_for(repo_root, slug, owned=owned)`. `_resolve_wp_board_implement_action` (`:3032`) and `_resolve_wp_board_review_action` (`:3058`) call `resolve_workspace_for_wp(repo_root, slug, wp, owned=owned)`.
  2. `_wp_iteration_action_and_state` (`:3172`) and `_build_wp_iteration_decision` (`:3204`) take `owned`. The `_state_to_action` fallback (`:3200`) and `_build_prompt_or_error` (`:3264`, `:3388`, `:3446`), plus the sites at `:1913`, `:1923`, `:1978` and `:1986` (`_state_to_action`, `_build_prompt_or_error`, `_build_prompt_safe`), all live in `decision.py`, owned here: pass `repo_root=repo_root, owned=owned` (T067). No bridging into `decision.py` remains.
  3. **Resolve before persisting.** In `_dn_decision_materialize`, before calling `runtime_next_step`:
     - preview the next step read-only with `_engine_adapter.plan_next(snapshot, template, policy, live_template_path=…)`, exactly as `query_current_state` does;
     - if the previewed step is a WP-iteration step (`_is_wp_iteration_step`, `:657`), run `_wp_iteration_action_and_state(..., owned=ctx.owned)` **first**;
     - if it raises, the exception propagates **before** any state is persisted, and the typed error reaches `next_cmd`, which emits it with its `error_code` and exit 1;
     - on success, persist the advance and pass the pre-resolved action into mapping, so the workspace is not resolved twice.

     **Do not** catch the resolution error into a `blocked` decision. FR-008 says wrapping into `blocked` fails the row.
  4. Keep the NFR-003 fail-closed arms (`CoordinationWorktreeUnmaterialized` / `CoordinationBranchDeleted`, `:3121-3143`) exactly as they are.
- **Files:** `src/runtime/next/runtime_bridge.py`.
- **Validation checklist:**
  - [ ] T057's O5 WP11-half test is green: no traceback, workspace P, WP01 implement, and the run advances exactly once.
  - [ ] A unit test forces the pre-resolution to raise and asserts that the run snapshot (`state.json` / `run.events.jsonl` in the run dir) is **byte-identical** before and after. This is the "never persisted" oracle.
  - [ ] The lane-mission and coordination-mission board tests in `tests/next/` stay green (positive control for the non-owned arm).
- **Edge cases:**
  - Stale copy: with `owned` set, the board never consults R, so O4-style fold-back cannot occur. Assert it in T057 step 7.
  - A pre-finalize owned mission (no finalized board) takes the `_state_to_action(owned=)` fallback (T067), which reads from P.

### Subtask T062 – #4867: `CoordinationWorkspaceUnavailable` → blocked `OWNED_COORDINATION_WORKSPACE_UNAVAILABLE`

- **Purpose:** FR-012 / R-08. `_has_stale_worktree_registration` (`coordination/workspace.py:161-184`) runs `subprocess.check_output([... "worktree", "list", "--porcelain"], text=True)` with stderr **not captured**. Exit 128 then escapes as `CalledProcessError`. `_wrap_with_decision_git_log`'s `except Exception` (`runtime_bridge.py:387-398`) re-raises it as `DecisionGitLogUnavailable`, which nothing catches, so the output is `fatal: … commondir: Success`.
- **Steps:**
  1. In `coordination/workspace.py`, add `class CoordinationWorkspaceUnavailable(subprocess.CalledProcessError)` with a stable `error_code = "OWNED_COORDINATION_WORKSPACE_UNAVAILABLE"`. It **must** subclass `CalledProcessError`: that is how `_resolve_owned_coordination_workspace` (`runtime_bridge.py:402-430`) keeps retrying transient lock contention through `_is_transient_git_worktree_contention` (`:433-456`), which reads `returncode` and `stderr`.
  2. Run the probe with `subprocess.run([...], capture_output=True, text=True, check=False)`. On a non-zero exit, raise `CoordinationWorkspaceUnavailable(returncode, argv, output=stdout, stderr=stderr)`. Apply the same wrapping to the `git worktree add` call in `resolve` (`:293-299`, already `capture_output=True`, `check=True`) so that both probe failures are typed.
     - This file is formatter-excluded (`pyproject.toml` `[tool.ruff.format] exclude`). Do **not** reformat it wholesale. Keep your edit locally formatted.
  3. In `_wrap_with_decision_git_log`, let `CoordinationWorkspaceUnavailable` propagate **unwrapped** when `owned` is set. Do not convert it to `DecisionGitLogUnavailable`.
  4. In `_dn_bootstrap` (where the wrap is called, `:1674-1682`), catch `CoordinationWorkspaceUnavailable` when `owned` is set and return a `blocked` Decision. Its `reason` names the code and remediation (for example, `git worktree prune` / `spec-kitty doctor coordination --fix`), and the command's JSON carries `error_code: "OWNED_COORDINATION_WORKSPACE_UNAVAILABLE"`.
     - If `Decision` has no `error_code` field, add the code to the JSON emission in `next_cmd._print_decision` from a typed attribute. Do not scrape the reason text.
     - The transient-lock path still retries first, so only a durable failure reaches this arm.
  5. Keep the non-owned behaviour unchanged: a non-owned coordination mission still raises `DecisionGitLogUnavailable` as today. Pin this with a unit test.
- **Files:** `src/specify_cli/coordination/workspace.py`, `src/runtime/next/runtime_bridge.py`, `src/specify_cli/cli/commands/next_cmd.py`.
- **Validation checklist:**
  - [ ] Both T057 injections turn green and assert on `error_code` only.
  - [ ] `tests/next/test_runtime_bridge_unit.py::…_resolve_owned_coordination_workspace…` (`:1789-1930`) is green: transient contention still retries, and permanent failure still raises after the window.
  - [ ] A new unit test shows that `CoordinationWorkspaceUnavailable` is a `CalledProcessError` and is recognised by `_is_transient_git_worktree_contention` when its stderr carries the lock text.
- **Edge cases:**
  - `CoordinationWorkspace.teardown` (`:302-331`) also calls the probe. With the typed exception, teardown on a broken registry raises `CoordinationWorkspaceUnavailable` instead of a bare `CalledProcessError`. It is a subclass, so existing `except CalledProcessError` callers are unaffected.
  - Grep: `grep -rn "_has_stale_worktree_registration\|CoordinationWorkspace.teardown" src/`.

### Subtask T063 – Remaining conversions in the `runtime/next` bridge modules and `next_cmd`

- **Purpose:** leave the WP11-owned files free of bare owned roots, carry the stale-copy field, and prepare for WP18's gate (G4/G5).
- **Steps:**
  1. Convert the remaining `effective_root` sites. Current counts on HEAD: `next_cmd.py` 45, `runtime_bridge.py` 22, `next_invocation_lifecycle.py` 9, `runtime_bridge_io.py` 1 (plus `decision.py`, handled in T067). Parameters and fields become `owned: OwnedCheckout | None`. Keyword arguments into the six seams become `owned=`. Arguments into WP12-owned `prompt_builder.py` become the bridging expression.
     - `_commit_owned_next_mutations(effective_root, mission_slug)` (`next_cmd.py:371-419`) becomes `_commit_owned_next_mutations(owned)`. It uses `owned.mission_dir` instead of `compose_meta_json_path(effective_root, slug).parent`, the owned root for `safe_commit(repo_root=…, worktree_root=…)`, and `mission_context_for(owned.repository_root, owned.mission_slug, owned=owned)` for the commit target.
     - `_resolve_mission_slug` (`:598-662`) with a fact returns `owned.mission_slug`; there is nothing to re-derive.
     - `decide_next` (`next_cmd.py:59-72`, the patchable wrapper) forwards `owned=`, and so does `decision.decide_next` (T067).
  2. **Stale-copy field (FR-007 / R-12).** After the decision is computed, and only when `owned` is set, call WP08's stale-copy reporter. Add the top-level `stale_repository_root_copy: {"path", "mission_id"} | null` to the `--json` decision output in `_print_decision` (`:1061`) in owned runs only; a non-owned run never emits the key, so its payload stays byte-identical (pin this with a non-owned control). In human mode, print the warning on **stderr** at command level. Detection compares the repository-root placement against `owned.mission_dir` and makes no git calls.
  3. Update the golden JSON contract for `next`, if one pins the key set, by adding the new additive key. Serialized keys may be added but must not be renamed. The golden contract test files are owned by WP09. If the key set is pinned there, coordinate: this is an additive, declared out-of-map edit. The key is emitted only in owned runs (data-model §Payload field), so a non-owned `next` payload stays byte-identical and its golden needs no change.
  4. Commit the conversions separately from the functional fixes, so review can read them as mechanical.
- **Files:** `src/specify_cli/cli/commands/next_cmd.py`, `src/runtime/next/runtime_bridge.py`, `runtime_bridge_io.py`, `runtime_bridge_composition.py`, `next_invocation_lifecycle.py`, `tests/next/test_runtime_bridge_unit.py`, `tests/specify_cli/cli/commands/test_next_answer_effective_root.py`.
- **Validation checklist:**
  - [ ] `grep -n "effective_root" src/specify_cli/cli/commands/next_cmd.py src/runtime/next/runtime_bridge.py src/runtime/next/runtime_bridge_io.py src/runtime/next/runtime_bridge_composition.py src/runtime/next/next_invocation_lifecycle.py src/runtime/next/runtime_bridge_engine.py src/runtime/next/decision.py` lists nothing except `TRANSITIONAL(WP18)`-marked lines. Bridging into `prompt_builder.py` (WP12) passes the owned root as `repo_root`, each call marked `# bridging: WP12 converts`.
  - [ ] T057's stale-copy row is green, and the field is `null` when R holds no copy.
  - [ ] mypy is clean on every touched file.
- **Edge cases:** occurrence map: the JSON key `stale_repository_root_copy` is **new** (additive). No existing key or `OWNED_*` code string changes.

### Subtask T066 – Campsite: `_state_to_action` (decision.py)

- **Purpose:** Standing Order 2. `_state_to_action` (`decision.py:394-493`, complexity 14) changes signature in T067, so it is extracted first. (`_build_wp_prompt`'s campsite stays with WP12's T068.)
- **Steps:**
  1. Extract:
     - `_implement_state_action(mission_slug, feature_dir, repo_root)` for the `state == "implement"` block (`:408-435`);
     - `_review_state_action(...)` for the review block (`:440-444`);
     - `_template_state_action(state, repo_root, mission_name)` for the generic template + `_ALIASES` resolution (`:451-493`).

     The outer function becomes a 4-way dispatch.
  2. Commit: `refactor(next): campsite extraction of decision._state_to_action (C901)`. No behaviour change; existing tests stay green unmodified.
- **Files:** `src/runtime/next/decision.py`.
- **Validation checklist:**
  - [ ] `.venv/bin/ruff check --isolated --select C901 --config 'lint.mccabe.max-complexity=11' src/runtime/next/decision.py` no longer reports `_state_to_action`.
  - [ ] `tests/next/` is green and unchanged. Grep `tests/` for the monkeypatch string `decision._state_to_action` and keep the name.
- **Edge cases:** `_state_to_action` is imported by name in `runtime_bridge.py` and in the carried #5009 test. Keep the name and positional order. The in_review / for_review arbitration rules (FR-012a comments) survive the extraction unchanged.

### Subtask T067 – `decision.py` takes the fact

- **Purpose:** FR-008/FR-009. The pre-finalize fallback and the `decide_next` bridge in `decision.py` must stop folding to the repository root checkout (#5009 e6923bc97 is re-expressed here for `decision.py`, with a `Co-authored-by: Samuel Goff <samuel@defpix.com>` trailer).
- **Steps:**
  1. `decide_next` (`:313-341`): accept `owned: OwnedCheckout | None = None` and forward `owned=owned` to `decide_next_via_runtime`. It is the only path from `next_cmd` to the runtime; without it the runtime would re-derive ownership (FR-003). Drop its `effective_root` once no caller needs it (`grep -rn "decide_next(" src/ tests/`); if a caller still does, mark the keyword `# TRANSITIONAL(WP18): <caller>`.
  2. `_state_to_action(state, mission_slug, feature_dir, repo_root, mission_name, *, owned: OwnedCheckout | None = None)`. Each `resolve_workspace_for_wp(repo_root, mission_slug, wp)` (`:425`, `:434`, `:443`) becomes `resolve_workspace_for_wp(repo_root, mission_slug, wp, owned=owned)`. `resolve_command(f"{state}.md", …)` and the alias variant use `owned.owned_root if owned else repo_root`, because a command template is a P-local governance read (e6923bc97).
  3. `_build_prompt_safe` (`:496`) and `_build_prompt_or_error` (`:524`) take `owned`. The composed-action probe `resolve_mission_type_context(repo_root, …)` (`:559`) uses the owned root. Their calls into `prompt_builder.build_prompt` (WP12) pass the bridging value `repo_root=owned.owned_root if owned else repo_root`, marked `# bridging: WP12 converts`; WP12 switches them to `owned=` as a declared out-of-map edit.
  4. **Typed refusal mapping (prepares WP12).** `_build_prompt_or_error` catches `ActionContextError` raised by `build_prompt` and returns `(None, message)` while carrying the error's `code` on the same typed attribute T062 uses for `OWNED_COORDINATION_WORKSPACE_UNAVAILABLE`, so `next`'s JSON gets `error_code` without parsing reason text. WP12 raises `ActionContextError("OWNED_REVIEW_BASE_UNAVAILABLE", …)` through this path. Pin it with a unit test that stubs `build_prompt` to raise `ActionContextError("OWNED_REVIEW_BASE_UNAVAILABLE", "x")`.
  5. `_with_guard_failure_paths(decision, repo_root)` (`:344`): if guard paths must name owned artifacts, pass the owned root. Otherwise leave it unchanged and say why in the Activity Log.
  6. Convert every `runtime_bridge.py` call site into `decision.py` (`:1913`, `:1923`, `:1978`, `:1986`, `:3200`, `:3264`, `:3388`, `:3436`, `:3446`) to `repo_root=repo_root, owned=…` (see T061).
  7. Unit pin: `_state_to_action("implement", SLUG, P/kitty-specs/SLUG, repository_root, "software-dev", owned=fact)` returns `("implement", "WP01", str(fact.owned_root))`, with the fact minted by the real validator.
- **Files:** `src/runtime/next/decision.py`, `src/runtime/next/runtime_bridge.py`, `tests/integration/test_explicit_checkout_commands.py`.
- **Validation checklist:**
  - [ ] `grep -n "effective_root" src/runtime/next/decision.py` lists nothing except `TRANSITIONAL(WP18)`-marked lines.
  - [ ] `grep -n "bridging: WP12" src/runtime/next/decision.py` lists only the `build_prompt` calls.
  - [ ] The non-owned behaviour of `tests/next/` (lane and coordination missions) is unchanged.

## Test Strategy

Red-first order:
1. T056 carried reds.
2. T057 reds (commit before any `src/` edit).
3. T058 and T066 campsites (green).
4. T059 → T062 and T067 fixes (reds turn green).
5. T063 conversion.

Record exact commands and counts under *Tests run*:

```bash
PWHEADLESS=1 .venv/bin/python -m pytest -q \
  tests/integration/test_owned_lifecycle_acceptance_next.py \
  tests/integration/test_explicit_checkout_commands.py \
  tests/next/test_runtime_bridge_unit.py \
  tests/specify_cli/cli/commands/test_next_answer_effective_root.py
PWHEADLESS=1 .venv/bin/python -m pytest -q tests/next/            # owning subsystem
grep -rl "next_cmd\|runtime_bridge\|runtime.next.decision\|next_invocation_lifecycle\|coordination.workspace\|CoordinationWorkspace" tests/ --include="*.py"  # run each hit not already covered (excluding e2e/slow)
PWHEADLESS=1 .venv/bin/python -m pytest -q tests/specify_cli/coordination/ tests/runtime/
make test-fast
.venv/bin/ruff check src/specify_cli/cli/commands/next_cmd.py src/runtime/next/ src/specify_cli/coordination/workspace.py tests/integration/test_owned_lifecycle_acceptance_next.py tests/integration/test_explicit_checkout_commands.py tests/next/test_runtime_bridge_unit.py
.venv/bin/ruff format --check src/specify_cli/cli/commands/next_cmd.py src/runtime/next/runtime_bridge.py src/runtime/next/runtime_bridge_io.py src/runtime/next/runtime_bridge_composition.py src/runtime/next/decision.py src/runtime/next/next_invocation_lifecycle.py tests/integration/test_owned_lifecycle_acceptance_next.py tests/integration/test_explicit_checkout_commands.py tests/next/test_runtime_bridge_unit.py
.venv/bin/mypy src/specify_cli/cli/commands/next_cmd.py src/runtime/next/runtime_bridge.py src/runtime/next/runtime_bridge_engine.py src/runtime/next/runtime_bridge_io.py src/runtime/next/runtime_bridge_composition.py src/runtime/next/decision.py src/runtime/next/prompt_builder.py src/runtime/next/next_invocation_lifecycle.py src/specify_cli/coordination/workspace.py src/specify_cli/cli/commands/_owned_checkout.py src/specify_cli/core/owned_mission.py  # callers and callees in ONE invocation (follow_imports = "skip" for specify_cli.*)
.venv/bin/python -m pytest -q tests/architectural/test_layer_rules.py tests/architectural/test_mission_runtime_surface.py tests/architectural/test_cold_import_status_boundary.py  # the specific gates this WP implicates
```

- `ruff format --check` deliberately omits the formatter-excluded `runtime_bridge_engine.py` and `coordination/workspace.py` (unless you carried 22f380055).
- Do **not** run `make test-full` or the bare `tests/architectural/` directory.
- The nightly e2e `tests/e2e/test_worktree_owned_root_concurrency.py` is covered per-PR by the T057 twin; do not run it locally.

## Risks & Mitigations

- **FR-008's prompt half is owned by WP12.** `prompt_builder._build_wp_prompt` resolves `mission_context_for(repo_root, slug)` with no owned arm (`prompt_builder.py:162-170`). An owned WP prompt cannot be built until WP12 T068, and `step_or_blocked` (`runtime_bridge_cores.py:901`) will never issue `kind=step` without a prompt file. Mitigation: WP11 proves the runtime half (no wedge, workspace P, advance, `decision.py` on the fact) and commits the full US3-AS1 row as a strict xfail; WP12 removes that marker (declared out-of-map one-line edit in this WP's test file). Flag this to the reviewer; do not paper over it with a `blocked` wrap.
- **Coordination worktree placement.** For owned coordination-topology missions, the coordination worktree is created under the root passed to `CoordinationWorkspace.resolve`. Today that is P, because `repo_root` = P. The root discipline keeps it at P. If you pass R there, R's `.worktrees/` changes, and NFR-001 reddens.
- **Monkeypatch churn.** Many tests patch `runtime_bridge.*` names. Keep names stable; change only keyword parameters.
- **Validation count.** Adoption plus explicit validation must not both run. `resolve_owned_or_adopt` is the single call site, and the FR-003 counter guards it.
- **Transient lock retry.** If `CoordinationWorkspaceUnavailable` does not subclass `CalledProcessError`, concurrent owned creates regress. The unit test in T062 pins the subclass.

## Review Guidance

- Verify the red-first commit order (T056/T057 before fixes) and the recorded red reasons.
- Verify that both #4867 injections exist, that injection (a) pre-asserts git failure with `pytest.fail` (never `skip`), and that no assertion reads git stderr.
- Verify that no `effective_root` survives in WP11-owned modules (including `decision.py` and `runtime_bridge_composition.py`) except `TRANSITIONAL(WP18)`-marked lines, and that the only bridging left targets `prompt_builder.py` (`# bridging: WP12 converts`).
- Verify the declared out-of-map edits (optional `pyproject.toml`; the additive golden-contract key if pinned in WP09's files), each with its one-line rationale in the PR description.
- Verify the `_state_to_action` (T066) and `_print_standard_human` (T058) campsite commits precede the signature changes.
- Verify that resolution happens before `runtime_next_step` persists, and that the byte-identical snapshot oracle exists.
- Verify that the implementer ran mypy on every touched source file.

## Activity Log

> Append entries at the END in chronological order: `- YYYY-MM-DDTHH:MM:SSZ – <agent_id> – <action>`.

- 2026-09-28T15:00:00Z – system – Prompt created.

---

### Updating Status

Use `spec-kitty agent tasks move-task WP11 --to <status> --mission owned-checkout-lifecycle-authority-01M3M2ZB` and `spec-kitty agent tasks mark-status T056 … --status done`.
