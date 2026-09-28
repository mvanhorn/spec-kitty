---
work_package_id: "WP02"
subtasks:
  - "T006"
  - "T007"
  - "T008"
  - "T009"
  - "T010"
  - "T011"
title: "Sole ownership minter and shared fixtures"
task_type: "implement"
phase: "Phase 1 - Foundation"
execution_mode: "code_change"
owned_files:
  - "src/specify_cli/core/owned_mission.py"
  - "src/specify_cli/core/checkout_ownership.py"
  - "tests/core/test_checkout_ownership.py"
  - "tests/core/test_owned_mission_minter.py"
  - "tests/core/test_adopt_owned_checkout.py"
  - "tests/integration/conftest.py"
  - "tests/integration/test_owned_fixtures_selftest.py"
authoritative_surface: "src/specify_cli/core/"
create_intent:
  - "tests/core/test_owned_mission_minter.py"
  - "tests/core/test_adopt_owned_checkout.py"
  - "tests/integration/test_owned_fixtures_selftest.py"
dependencies:
  - "WP01"
requirement_refs:
  - "FR-002"
  - "FR-020"
  - "FR-021"
  - "FR-023"
  - "C-001"
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

# Work Package Prompt: WP02 – Sole ownership minter and shared fixtures

## ⚡ Do This First: Load Agent Profile

Use the `/ad-hoc-profile-load` skill to load the agent profile specified in the frontmatter (or any user-defined profile), and behave according to its guidance before parsing the rest of this prompt.

- **Profile**: `python-pedro`
- **Role**: `implementer`
- **Agent/tool**: `claude`

If no profile is specified, run `spec-kitty agent profile list` and select the best match for this work package's `task_type` and `authoritative_surface`.

Then load the action-scoped governance: `spec-kitty charter context --action implement --json`, and read `.kittify/charter/charter.md` if this session has not read it yet.

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
Use language identifiers in code blocks: ````python`,````bash`

---

## Objectives & Success Criteria

`specify_cli.core.owned_mission` becomes the **only** producer of the validated ownership fact (C-001). It exposes:

| Function | Returns | Requirement |
|---|---|---|
| `resolve_owned_mission(repository_root, checkout, handle, *, target_override=None, allowed_topologies=LIFECYCLE_OWNED_TOPOLOGIES)` | `OwnedCheckout` or raises `ActionContextError(code)` | FR-002, FR-023 |
| `resolve_owned_create_root(repository_root, checkout)` | typed `OwnedCreateRoot` | FR-016 prerequisite (WP10 wires it) |
| `adopt_owned_checkout(repository_root, cwd, handle, *, allowed_topologies)` | `OwnedCheckout \| None` | FR-021 |
| `LIFECYCLE_OWNED_TOPOLOGIES` / `NEXT_OWNED_TOPOLOGIES` | `frozenset[MissionTopology]` | FR-023 |

Also:
- `OWNED_CHECKOUT_IS_REPOSITORY_ROOT` refuses R passed as the owned checkout (FR-020, research R-14).
- The transitional alias `OwnedMission = OwnedCheckout` and `effective_root_kwargs` accepting the fact keep every current call site green (plan §Staging Strategy).
- The shared integration fixtures `owned_checkouts`, `r_snapshot` and `stale_root_copy`, with self-tests that prove each fixture detects what it claims to detect.

**Done means**:
- every new test is green, and each `[build]` behaviour had a red-first commit;
- `tests/core/test_checkout_ownership.py` is still green;
- the existing owned command tests are green unchanged: `tests/integration/test_explicit_checkout_commands.py`, `tests/integration/test_owned_checkout_mark_status.py` and `tests/specify_cli/cli/commands/agent/test_owned_checkout_move_task.py`;
- `tests/architectural/test_cold_import_status_boundary.py` is green;
- ruff, format and mypy `--strict` are clean on the touched files, and `make test-fast` is green.

## Context & Constraints

- `spec.md`: FR-002, FR-020, FR-021, FR-023, C-001, US1-AS3 (the invalid-value list), US7 (flagless adoption), §Edge Cases.
- `plan.md`: §IC-01, §Staging Strategy, §Test Layout and Markers.
- `research.md`: R-02 (topology per command), R-09 (flagless adoption), R-14 (repository root refusal), R-16 (adoption falsification, sizing).
- `data-model.md`: §Validator, §Error code registry.
- `contracts/owned-checkout-carrier.md`: §1 sole construction, §4 topology per command, §5 repository root not owned, §6 adoption guard.
- `occurrence_map.yaml`: `code_symbols` and `tests_fixtures` are `rename`; `logs_telemetry` is `do_not_change` (every existing `OWNED_*` code keeps its exact string).

**Current code (HEAD `df1588860`)**:
- `src/specify_cli/core/owned_mission.py:17-38`: the `OwnedMission` dataclass (`primary`, `root`, `directory`, `slug`, `target`, plus `files()`).
- `:41-47`: `_EffectiveRootKwargs` / `effective_root_kwargs`.
- `:50-68`: `_stored_topology`.
- `:71-128`: `resolve_owned_mission`. The topology refusal is at `:108-110` and always single_branch; the branch check is at `:111-114`; the protection check is at `:115-116`; the #3866 top-level tripwire is at `:127`.
- `:131-135`: `require_unstaged_index`.
- `src/specify_cli/core/checkout_ownership.py:190-266`: `resolve_ownership_claim`. It returns `OWNED` for the repository root itself (`:225-231`), and `tests/core/test_checkout_ownership.py:111` (`test_primary_self_ownership_is_owned`) pins that primitive behaviour.
- Other claim-primitive callers (G1 offenders, converted by later WPs, **not here**): `src/specify_cli/cli/commands/next_cmd.py:170` (WP11) and `src/specify_cli/core/mission_creation.py:848` (WP10).

**Constraints**:
- **Cold-import boundary (#1461).** `owned_mission.py` is cold-imported by `task_utils.support` and ~37 CLI modules. Keep the `mission_resolver` and `checkout_ownership` imports **function-local** (as at `:84-89`); `tests/architectural/test_cold_import_status_boundary.py:39` pins this. Any `surface_resolver`, `lanes` or `workspace` import added for adoption must also be function-local.
- **Patchability (NFR-002).** Later WPs count validations by patching `specify_cli.core.checkout_ownership.resolve_ownership_claim`. Keep calling it through a function-local `from specify_cli.core.checkout_ownership import …` so a module-attribute patch is observed. Clear the workspace caches in those tests (plan §Test Layout).
- **No second minter.** Only `owned_mission.py` may call `OwnedCheckout._mint` (gate G3).
- **Terminology.** New messages say "repository root checkout" and "owned checkout". The parameter `primary` of `resolve_owned_mission` is renamed to `repository_root`. Every current caller passes it positionally (verify with `grep -rn "resolve_owned_mission(" src tests`).

## Branch Strategy

- **Strategy**: Planning artifacts were generated on `claude/sleepy-hamilton-5lelee`; completed changes merge back into `claude/sleepy-hamilton-5lelee`.
- **Planning base branch**: `claude/sleepy-hamilton-5lelee`
- **Merge target branch**: `claude/sleepy-hamilton-5lelee`
- **Lane**: assigned in `lanes.json` by `finalize-tasks` (not yet generated). WP02 depends on WP01; use `spec-kitty implement WP02`.

> These fields are populated automatically by `spec-kitty agent mission finalize-tasks`.
> Do NOT change them manually unless you are certain the branch topology has changed.

## Subtasks & Detailed Guidance

### Subtask T006 – `resolve_owned_mission` returns the fact; `allowed_topologies`; topology sets

- **Purpose**: Make the canonical validator produce `mission_runtime.OwnedCheckout` (the validated fact), and move the topology decision to the call site's allowed set (FR-023, research R-02). This is what later lets `next` go through the same validator, with its branch and protection checks (FR-002, O9), without losing owned coordination-topology support.
- **Steps**:
  1. **Red first.** Create `tests/core/test_owned_mission_minter.py` (`pytestmark = [pytest.mark.git_repo]`). Build a local helper `_owned_pair(tmp_path, *, topology="single_branch", coordination_branch=None)`, copying the shape of `tests/integration/test_explicit_checkout_commands.py:52-100`: `git init` R, seed commit, `git worktree add -b codex/owned P`, then write `kitty-specs/<slug>/meta.json` in P and commit it in P. (The integration fixtures from T011 are not reachable from `tests/core/`.) Commit these tests before the fix, and record the red output:
     - `test_returns_validated_fact`: `isinstance(resolve_owned_mission(R, P, slug), OwnedCheckout)`, with `fact.topology is MissionTopology.SINGLE_BRANCH`, `fact.owned_root == P.resolve()`, `fact.mission_dir == P/kitty-specs/<slug>`. Red today (it returns `OwnedMission`).
     - `test_next_topologies_accept_lanes_with_coord`: a `lanes_with_coord` mission with a `coordination_branch`, called with `allowed_topologies=NEXT_OWNED_TOPOLOGIES`, returns a fact whose `topology is LANES_WITH_COORD`. Red today (`TypeError` on the unknown keyword, or the single_branch refusal).
     - `test_lifecycle_default_refuses_lanes_with_coord`: the same mission, called with the default, raises `OWNED_TOPOLOGY_UNSUPPORTED`. This is the paired control, and it is green on base.
     - Branch and protection controls through the same entry point: a mismatched branch, detached HEAD, and a `target_override` mismatch each raise `OWNED_BRANCH_REFUSED`. A protected target (via `protection: {protected_branches: [codex/owned]}` in P's `.kittify/config.yaml`) raises `OWNED_BRANCH_REFUSED`. These are ratchets, green on base.
  2. Add the module constants:
     ```python
     LIFECYCLE_OWNED_TOPOLOGIES: Final = frozenset({MissionTopology.SINGLE_BRANCH})
     NEXT_OWNED_TOPOLOGIES: Final = frozenset({MissionTopology.SINGLE_BRANCH, MissionTopology.LANES_WITH_COORD, MissionTopology.COORD})
     ```
     Note that `LANES` is deliberately absent from both (research R-02).
  3. Change the signature to `resolve_owned_mission(repository_root, checkout, handle, *, target_override=None, allowed_topologies=LIFECYCLE_OWNED_TOPOLOGIES) -> OwnedCheckout`.
  4. Replace `:108-110` with `_require_allowed_topology(meta, topology, allowed_topologies)`:
     - `meta is None` or topology `None` → `OWNED_TOPOLOGY_UNSUPPORTED`;
     - topology not in the allowed set → `OWNED_TOPOLOGY_UNSUPPORTED`, with a message naming the allowed set;
     - the `coordination_branch` refusal applies **only** when the allowed set contains no coordination-routing topology (use `mission_runtime.routes_through_coordination`).
  5. Extract `_require_owned_claim`, `_require_branch_and_protection` and `_resolve_owned_directory` so `resolve_owned_mission` stays ≤ 11 complexity (campsite; measure before and after with `.venv/bin/ruff check --isolated --select C901 --config 'lint.mccabe.max-complexity=11' src/specify_cli/core/owned_mission.py`).
  6. Construct the result with `OwnedCheckout._mint(repository_root=…, owned_root=claim.claimed_checkout, mission_dir=directory, mission_slug=mission.feature_dir.name, topology=topology, target_branch=target)`. Keep the #3866 top-level tripwire `result.files(list(directory.iterdir()))`, and keep its comment.
  7. `require_unstaged_index(context: OwnedCheckout)` reads `context.owned_root`.
- **Files**: `src/specify_cli/core/owned_mission.py`, `tests/core/test_owned_mission_minter.py` (new).
- **Validation checklist**:
  - [ ] The red commit is recorded (test ids and failure reason) in the Activity Log.
  - [ ] `pytest tests/core/test_owned_mission_minter.py -q` is green.
  - [ ] `pytest tests/integration/test_explicit_checkout_commands.py tests/integration/test_owned_checkout_mark_status.py tests/specify_cli/cli/commands/agent/test_owned_checkout_move_task.py -q` is green unchanged (FR-022 guard).
  - [ ] `resolve_owned_mission` is ≤ 11 complexity.
- **Edge cases**:
  - Non-vacuity: the lanes_with_coord acceptance test fails if the allowed-set check is removed, **and** the lifecycle refusal test fails if the check is made permissive. The pair is the proof.
  - The mission handle may be a slug, a mid8 or a mission id. Parametrise the happy path over all three (spec US1).
  - Do not change `OWNED_*` code strings (`logs_telemetry: do_not_change`).

### Subtask T007 – `OWNED_CHECKOUT_IS_REPOSITORY_ROOT` refusal

- **Purpose**: `--owned-checkout <R>` currently passes the claim primitive (`checkout_ownership.py:225-231` returns `OWNED`). The fact's invariant `owned_root != repository_root` would then raise a raw `ValueError`. It must instead be a typed refusal with a registered code (FR-020, NFR-004, research R-14).
- **Steps**:
  1. **Red first** in `tests/core/test_owned_mission_minter.py`: `test_repository_root_is_refused`. Put R on an unprotected branch equal to the mission target (for example `git checkout -b codex/root-owned` in R), with a single_branch mission committed in R, and call `resolve_owned_mission(R, R, slug)`. Expect `ActionContextError` with `code == "OWNED_CHECKOUT_IS_REPOSITORY_ROOT"`. On base this returns a value (accepted), so it is a genuine red. Add a same-fixture control: the valid linked P is accepted.
  2. In `src/specify_cli/core/checkout_ownership.py`, add a module constant `OWNED_CHECKOUT_IS_REPOSITORY_ROOT = "OWNED_CHECKOUT_IS_REPOSITORY_ROOT"` and a pure predicate `claims_repository_root(claim: OwnershipClaim) -> bool` (`claim.claimed_checkout == claim.resolved_primary`). **Do not** change `resolve_ownership_claim` or `OwnershipValidationResult`: `tests/core/test_checkout_ownership.py:111` and `:334` pin the primitive (five values; self-ownership is `OWNED`). ADR 2026-08-12-1 defines that primitive, and WP03 documents the minter-level refusal.
  3. In `owned_mission._require_owned_claim`, raise `ActionContextError(OWNED_CHECKOUT_IS_REPOSITORY_ROOT, "--owned-checkout names the repository root checkout; pass a linked owned checkout instead.")` when the predicate holds. The same guard is reused by T008 and T009.
  4. Add to `tests/core/test_checkout_ownership.py`: `claims_repository_root` is True for R, False for a linked worktree, and False for a nested or foreign claim.
- **Files**: `src/specify_cli/core/checkout_ownership.py`, `src/specify_cli/core/owned_mission.py`, `tests/core/test_checkout_ownership.py`, `tests/core/test_owned_mission_minter.py`.
- **Parallel?**: Yes with T008 after T006.
- **Validation checklist**:
  - [ ] `pytest tests/core/test_checkout_ownership.py tests/core/test_owned_mission_minter.py -q` is green.
  - [ ] Search the tests for CLI invocations that pass the repository root as `--owned-checkout` (`grep -rn "owned-checkout\", str(" tests`) and confirm none relied on it. Record the result in the Activity Log.
- **Edge cases**:
  - Symlinked R or case-variant R: the claim resolves paths, so compare the resolved paths. On Windows the case-variant path is covered by the fact's normcase comparison (WP01). Add one test passing `R` through a symlink `link -> R`; it must still refuse.
  - The refusal must fire **before** mission resolution, so no read of R's `kitty-specs` happens (NFR-001 spirit).

### Subtask T008 – `resolve_owned_create_root` → typed `OwnedCreateRoot`

- **Purpose**: `agent mission create --owned-checkout` needs a validated owned root **before** the mission exists. Today `core/mission_creation.py:848` calls the claim primitive directly (a G1 offender). WP02 provides the minter-side function; **WP10 wires it** (T053). It returns a typed value, not a bare `Path` (plan §IC-01, tasks T008). Note: `data-model.md` §Validator still says it returns `Path`; the plan and tasks supersede that, so record the correction in the Activity Log.
- **Steps**:
  1. **Red first** in `tests/core/test_owned_mission_minter.py`:
     - `resolve_owned_create_root(R, P)` returns an `OwnedCreateRoot` whose `repository_root == R.resolve()` and `checkout == P.resolve()`. Red with `ImportError` / `AttributeError`. That is an acceptable red only because there is no pre-existing entry point for this new function. The pre-existing-entry-point reproduction for create lives in WP10 (a37e9ee39).
     - The refusals: R itself → `OWNED_CHECKOUT_IS_REPOSITORY_ROOT`; a nested directory → the `OWNERSHIP_NESTED` code; a foreign repository → `OWNERSHIP_FOREIGN`.
  2. Define `OwnedCreateRoot` as a frozen dataclass in `owned_mission.py` with fields `repository_root: Path` and `checkout: Path`. **Do not name a field `owned_root` / `owned_checkout` / `checkout_root`**: gate G5 bans those names typed `Path` everywhere except on `mission_runtime.owned_checkout.OwnedCheckout` (`contracts/architectural-gate.md`). Construction goes only through `resolve_owned_create_root` (reuse the private-token pattern from WP01, or keep the dataclass module-private and export only the function plus the type for annotations).
  3. Implement `resolve_owned_create_root(repository_root, checkout)`: claim → `error_for_claim` (raise the structured error as `ActionContextError(error.error_code, str(error))`, the same translation as `:92-94`) → the repository-root refusal → return the value. No branch or topology check: the mission does not exist yet, and create keeps its own guards (`mission_creation.py:863-879`).
- **Files**: `src/specify_cli/core/owned_mission.py`, `tests/core/test_owned_mission_minter.py`.
- **Parallel?**: Yes with T007.
- **Validation checklist**:
  - [ ] Tests are green, and mypy `--strict` is clean.
  - [ ] `grep -n "resolve_owned_create_root" src` shows only the definition. WP10 adds the caller; the dead-symbol gate may flag it until then (same handling as WP01 T002: record it, do not allowlist).
- **Edge cases**:
  - The caller in WP10 passes `owned_checkout.resolve()`. The function must resolve internally anyway, and must be idempotent on already-resolved input.

### Subtask T009 – `adopt_owned_checkout`

- **Purpose**: Validated flagless adoption (FR-021, decision `01M3M4GTA4ENB73A0HDS65WZ1P`). Today `src/specify_cli/missions/operation_context.py:69-116` adopts `get_status_read_root(cwd)` with no validation (O10). The new function adopts only what the canonical validator accepts. WP08 wires it into `agent context resolve` and deletes `operation_context.py`.
- **Steps**:
  1. **Red first**: create `tests/core/test_adopt_owned_checkout.py` (`pytestmark = [pytest.mark.git_repo]`) with the cases below. Commit it red (`ImportError` on the new name), then implement. The pre-existing-entry-point red for O10 is WP08's acceptance test through `agent context resolve`; say so in the module docstring.
  2. Signature: `adopt_owned_checkout(repository_root: Path, cwd: Path, handle: str | None, *, allowed_topologies: frozenset[MissionTopology]) -> OwnedCheckout | None`.
  3. Decision order (each step returns `None` unless stated otherwise; keep it ≤ 11 complexity by extracting `_candidate_checkout_root` and `_is_lane_worktree_of_mission`):
     1. `handle` empty → `None`.
     2. `toplevel = get_status_read_root(cwd)` (`src/specify_cli/core/paths.py:610`, a pure `.git` walk with no subprocess). If it equals the resolved R → `None` **without** calling the claim primitive (NFR-002: flagless commands run from R make 0 validations).
     3. A coordination worktree: `surface_resolver.classify_worktree_topology(toplevel, repo_root=R) is WorktreeTopology.COORD_WORKTREE` → `None`.
     4. A lane worktree of the mission: resolve `handle` in `toplevel`; if the mission's `lanes.json` (read from that mission dir) names a lane whose worktree path, composed through the canonical seam (`specify_cli.lanes` / `workspace` naming helpers, never a hand-built name), equals `toplevel` → `None`. **Do not use `WorktreeTopology.LANE_WORKTREE` as the lane test.** It means "registered and not coordination" (`surface_resolver.py:152`), so a valid owned checkout placed under `R/.worktrees/` also classifies as `LANE_WORKTREE` (the carrier contract note in plan §IC-01).
     5. The mission is absent from `toplevel` → `None` (today's behaviour: resolve from R).
     6. **Mission-surface conflict (US7-AS5)**: if the handle resolves in both R and `toplevel` to **different** mission ids, raise `ActionContextError("MISSION_CONTEXT_CONFLICT", <the message text of MissionSurfaceConflictError>)`. `agent/context.py:153-154` already maps the old exception to exactly this code, so JSON output is unchanged. Do not import from `operation_context.py`; WP08 deletes it.
     7. Otherwise call `resolve_owned_mission(R, toplevel, handle, allowed_topologies=allowed_topologies)`; on `ActionContextError` → `None` (a rejected checkout falls back to today's behaviour, US7-AS3). A different repository also lands here (the claim is `FOREIGN`).
  4. Cases in `tests/core/test_adopt_owned_checkout.py` (each asserts the return value **and** the number of `resolve_ownership_claim` calls, via a counting monkeypatch on `specify_cli.core.checkout_ownership.resolve_ownership_claim`):
     - cwd = R → `None`, 0 claims;
     - cwd = a subdirectory of a valid P → a fact, 1 claim;
     - cwd = P placed at `R/.worktrees/owned-a` (a valid owned checkout) → a fact (the LANE_WORKTREE-classification trap);
     - cwd = a registered coordination worktree `R/.worktrees/<slug>-<mid8>-coord` → `None`, 0 claims;
     - cwd = a lane worktree named in the mission's `lanes.json` → `None`;
     - cwd = a linked checkout on a mismatched branch → `None`;
     - cwd = a checkout of another repository → `None`;
     - same handle, different mission ids in R and P → `MISSION_CONTEXT_CONFLICT`;
     - same handle and the same mission id in R and P (a stale copy) → a fact for P.
- **Files**: `src/specify_cli/core/owned_mission.py`, `tests/core/test_adopt_owned_checkout.py` (new).
- **Validation checklist**:
  - [ ] All nine cases are green, and the claim counts match.
  - [ ] `tests/architectural/test_cold_import_status_boundary.py` is green (all new imports are function-local).
- **Edge cases**:
  - `cwd` outside any git checkout: `get_status_read_root` falls back to `get_main_repo_root`. Treat any error as `None`, never raise.
  - A `WorktreeRegistryUnavailable` from the classifier → `None` (fail closed toward today's behaviour, never toward adoption).
  - Adoption never writes anything.

### Subtask T010 – Transitional `OwnedMission` alias plus `effective_root_kwargs` accepting the fact

- **Purpose**: Keep all ~104 `OwnedMission` attribute reads and 34 `effective_root_kwargs(...)` calls green while later WPs convert top-down (plan §Staging Strategy). WP18 deletes both.
- **Steps**:
  1. Delete the `OwnedMission` dataclass (`owned_mission.py:17-38`) and add `OwnedMission = OwnedCheckout  # Transitional alias (WP02); deleted by WP18.` The legacy attribute names resolve through WP01's transitional properties.
  2. Change `effective_root_kwargs(root: Path | OwnedCheckout | None) -> _EffectiveRootKwargs`, which returns `{"effective_root": root.owned_root}` for a fact. Update its docstring to say it is transitional.
  3. **Re-point the direct positional constructions** of `OwnedMission(...)`. They now raise `TypeError`, because the alias's constructor is sentinel-guarded and has six fields. The known sites on HEAD:
     - `tests/specify_cli/cli/commands/agent/test_tasks_move_task_pre_review_identity_read.py:182`
     - `tests/specify_cli/cli/commands/agent/test_tasks_mark_status_recovery.py:41`
     - `tests/specify_cli/coordination/test_status_transition.py:1206` (a WP07-owned file)
     - `tests/status/test_bootstrap.py:542`
     - `tests/migration/test_runtime_feature_dir_threading.py:23`

     Replace each with `OwnedCheckout._mint(...)` using **valid** paths. The move-task test passes arguments in a non-canonical order that would violate the new invariants, so rebuild its paths deliberately. These are **declared out-of-map edits** (rationale: "WP02 makes positional `OwnedMission(...)` construction raise by contract §1; the test must mint the fact"). List them in the Activity Log and the PR.
  4. Run `grep -rn "OwnedMission(" src tests` and `grep -rn "effective_root_kwargs(" src`. The first must show no hits. The second must still type-check under mypy (the `**` splats stay typed).
- **Files**: `src/specify_cli/core/owned_mission.py` plus the five out-of-map test files above.
- **Validation checklist**:
  - [ ] `pytest` on the five re-pointed files plus `tests/specify_cli/cli/commands/test_next_owned_commit_guard.py tests/next/test_runtime_bridge_unit.py tests/runtime/test_artifact_presence_placement.py -q` is green.
  - [ ] `mypy --strict src/specify_cli/core/owned_mission.py` is clean.
- **Edge cases**:
  - Tests that monkeypatch `resolve_owned_mission` to return a stub object keep working, because attribute access is duck-typed. Do not "fix" them here.
  - `OwnedCheckout` hashing differs from the old dataclass (the token has `compare=False`). No production code hashes `OwnedMission`; verify with a grep.

### Subtask T011 – Shared integration fixtures with self-tests

- **Purpose**: Every owned acceptance test (WP08, WP09, WP11–WP13, WP18) uses one fixture set. That keeps the R snapshot (NFR-001) and the stale copy identical everywhere (plan §Test Layout: fixtures live in `tests/integration/conftest.py`, never imported from test modules).
- **Steps**:
  1. Extend `tests/integration/conftest.py` (31 lines today). Keep the existing coordination-topology re-export and the SaaS-sync autouse. Put the implementation in private helpers in the same file, or in a new helper module registered the way `conftest_coord_topology.py` is. **A new helper module is not in this WP's owned files**, so prefer keeping everything in `conftest.py`.
  2. `owned_checkouts`: a factory fixture `make_owned_checkouts(*, topology="single_branch", placement="sibling" | "under_worktrees", wp_ids=("WP01", "WP02"), protected_target=False)` plus a default `owned_checkouts` fixture. It builds, without `spec-kitty init` (per-PR speed; init-built variants are `e2e` + `slow`, nightly):
     - R: `git init -b main`, test identity, `.kittify/config.yaml`, a seed commit, and `origin` refs (as in `test_explicit_checkout_commands.py:52-67`);
     - P: `git worktree add -b codex/owned <path>`, at a sibling path or at `R/.worktrees/owned-a`;
     - a sibling linked worktree S;
     - the mission in P: `meta.json` with `mission_id` / `mission_slug` / `topology` / `target_branch`, `spec.md`, `plan.md`, `tasks.md`, one WP file per id, and `.gitignore` ignoring `.kittify/derived/`; committed in P.

     It returns a frozen dataclass `OwnedCheckouts(repository_root, owned_root, sibling, mission_slug, mission_id, mid8, target_branch, mission_dir)`. **Name the fields exactly like this.** `owned_root` is fine here because tests are outside G5's `src/` scope.
  3. `owned_handle`: parametrised over `("slug", "mid8", "mission_id")`, returning the handle string for the default fixture (spec §Common fixture).
  4. `r_snapshot`: requests the canonical home fixture (`canonical_home` in `tests/conftest.py`). **Never set `SPEC_KITTY_HOME` yourself**: the home-pin scan gate (`_home_pin_scan`) polices fixtures that do. It returns an object with `take() -> RSnapshot` and `assert_unchanged(before, after)`. `RSnapshot` holds:
     - sha256 of every file under R's working tree, **including ignored files**, excluding the `.git` directory and **exactly P's resolved subtree** when P lives under R (never a blanket `.worktrees/` exclusion);
     - `git rev-parse HEAD` in R;
     - the index: `git ls-files --stage` plus `git diff --cached`;
     - the file hashes under `<git-common-dir>/spec-kitty-locks` (constant `LOCK_DIRECTORY`, `src/specify_cli/core/checkout_file_lock.py:31`);
     - the file hashes under the isolated `SPEC_KITTY_HOME`.

     `assert_unchanged` prints the added, removed and changed keys per component.
  5. `stale_root_copy`: copies P's mission dir into `R/kitty-specs/<slug>` with the **same** `mission_id`, adds WP files so R holds WP01–WP05 (spec US1-AS1), and commits it in R. It returns the copied path. Offer a `different_id=True` variant for US7-AS5.
  6. `owned_cwd`: parametrised over `("repository_root", "owned_checkout", "elsewhere")`, it `monkeypatch.chdir`s accordingly (NFR-001 cwd matrix).
  7. Create `tests/integration/test_owned_fixtures_selftest.py` (`pytestmark = [pytest.mark.integration, pytest.mark.git_repo]`). These self-tests are the fixtures' non-vacuity proof:
     - P's git common dir equals R's; P is on `codex/owned`; the meta topology is as requested;
     - under the `under_worktrees` placement, `classify_worktree_topology(P, repo_root=R)` is `LANE_WORKTREE` (pins the contract note);
     - each handle resolves via `resolve_mission(handle, P)`;
     - `r_snapshot` **detects** each of: a new untracked file in R, a new **ignored** file in R (`.kittify/derived/x`), a HEAD move, a `git add` in R, a file in the lock root, a file in `SPEC_KITTY_HOME`, and a change in **another** `R/.worktrees/<lane>` directory;
     - `r_snapshot` **ignores** a change inside P when P is under `R/.worktrees/`;
     - `stale_root_copy` makes R list WP01–WP05 while P lists WP01–WP02, with the same `mission_id`.
- **Files**: `tests/integration/conftest.py`, `tests/integration/test_owned_fixtures_selftest.py` (new).
- **Validation checklist**:
  - [ ] `pytest tests/integration/test_owned_fixtures_selftest.py -q` is green, with each detection case failing if its component is dropped from the snapshot (check it manually once and note it).
  - [ ] The whole `tests/integration` collection still imports: `pytest tests/integration --collect-only -q | tail -1`.
- **Edge cases**:
  - Windows paths: use `Path.resolve()` and `relative_to`, never string prefix checks, for the P-subtree exclusion.
  - Large trees: skip `.git` and P's subtree during the walk (prune `dirnames`) instead of filtering afterwards.

## Test Strategy

**Red-first per `[build]` requirement** (C-007; commit each red test before its fix):

| Requirement | Red-first test | Entry point | Why non-vacuous |
|---|---|---|---|
| FR-023 | `test_next_topologies_accept_lanes_with_coord` | `resolve_owned_mission` (pre-existing) | paired with the lifecycle-default refusal on the same fixture |
| FR-002 (minter half) | `test_returns_validated_fact` and the branch/protection ratchets | `resolve_owned_mission` | the fact type check is red on base; the branch and protection refusals prove the checks `next` will inherit (the CLI half is WP11) |
| FR-020 | `test_repository_root_is_refused` | `resolve_owned_mission` | red on base: R is accepted today; control: valid P accepted |
| FR-021 (helper) | `tests/core/test_adopt_owned_checkout.py` | new function (the CLI-level O10 red is WP08) | nine cases with claim-count assertions; the R case asserts 0 claims |

Markers: `tests/core/*` → `git_repo`; `tests/integration/test_owned_fixtures_selftest.py` → `integration`, `git_repo`.

Commands:

```bash
uv run --frozen pytest tests/core/test_owned_mission_minter.py tests/core/test_adopt_owned_checkout.py tests/core/test_checkout_ownership.py -q
uv run --frozen pytest tests/integration/test_owned_fixtures_selftest.py -q
uv run --frozen pytest tests/integration/test_explicit_checkout_commands.py tests/integration/test_owned_checkout_mark_status.py tests/specify_cli/cli/commands/agent/test_owned_checkout_move_task.py -q
uv run --frozen pytest tests/specify_cli/cli/commands/agent/test_tasks_move_task_pre_review_identity_read.py tests/specify_cli/cli/commands/agent/test_tasks_mark_status_recovery.py tests/specify_cli/coordination/test_status_transition.py tests/status/test_bootstrap.py tests/migration/test_runtime_feature_dir_threading.py -q
uv run --frozen pytest tests/specify_cli/missions/test_operation_context.py tests/specify_cli/cli/commands/test_next_owned_commit_guard.py tests/next/test_runtime_bridge_unit.py -q
uv run --frozen pytest tests/architectural/test_cold_import_status_boundary.py tests/architectural/test_no_dead_symbols.py tests/architectural/test_mission_runtime_surface.py -q
make test-fast
uv run --frozen ruff check src/specify_cli/core/owned_mission.py src/specify_cli/core/checkout_ownership.py tests/core tests/integration/conftest.py tests/integration/test_owned_fixtures_selftest.py
uv run --frozen ruff format --check src/specify_cli/core/owned_mission.py src/specify_cli/core/checkout_ownership.py tests/core/test_owned_mission_minter.py tests/core/test_adopt_owned_checkout.py tests/core/test_checkout_ownership.py tests/integration/conftest.py tests/integration/test_owned_fixtures_selftest.py
uv run --frozen mypy --strict src/specify_cli/core/owned_mission.py src/specify_cli/core/checkout_ownership.py
```

No `make test-full`, and no bare `tests/architectural/`.

## Risks & Mitigations

- **Hidden positional constructions.** Mitigation: T010 step 4 greps, and the out-of-map list is declared.
- **Adoption drift into a second authority.** Adoption must end in `resolve_owned_mission`; the adoption tests assert claim counts.
- **The `LANE_WORKTREE` trap.** Pinned by both the fixture self-test and the adoption test for P under `.worktrees/`.
- **Cold-import regression.** All new imports are function-local; the cold-import gate runs.
- **Home-pin gate.** The fixture requests `canonical_home` and never sets `SPEC_KITTY_HOME`.

## Review Guidance

- `resolve_owned_mission` is the only `_mint` caller, and the topology decision is by allowed set with both directions tested.
- The repository-root refusal is at the minter; the claim primitive and its five-value enum are unchanged.
- The `OwnedCreateRoot` field names avoid G5's banned names.
- The adoption tests cover all nine cases with claim counts; R costs 0 claims.
- The alias and `effective_root_kwargs` widening are transitional (WP18 deletes them); the out-of-map test edits are listed with a rationale.
- The fixtures request `canonical_home`, and the snapshot excludes exactly P's subtree.
- mypy `--strict` ran and passed.

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
