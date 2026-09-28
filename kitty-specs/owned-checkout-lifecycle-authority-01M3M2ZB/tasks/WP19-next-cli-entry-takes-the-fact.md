---
work_package_id: WP19
title: next CLI entry takes the fact
dependencies:
- WP11
- WP08
requirement_refs:
- FR-002
- FR-007
- FR-008
- FR-022
- FR-023
planning_base_branch: claude/sleepy-hamilton-5lelee
merge_target_branch: claude/sleepy-hamilton-5lelee
branch_strategy: Planning artifacts for this mission were generated on claude/sleepy-hamilton-5lelee. During /spec-kitty.implement this WP may branch from a dependency-specific base, but completed changes must merge back into claude/sleepy-hamilton-5lelee unless the human explicitly redirects the landing branch.
subtasks:
- T101
- T102
- T103
- T104
- T105
phase: Phase 3 - Commands and runtime
history:
- at: '2026-09-28T16:00:00Z'
  actor: system
  action: Prompt generated via /spec-kitty.tasks (split from WP11)
agent_profile: python-pedro
authoritative_surface: src/specify_cli/cli/commands/next_cmd.py
create_intent:
- tests/integration/test_owned_lifecycle_acceptance_next.py
execution_mode: code_change
model: claude-sonnet-5
owned_files:
- src/specify_cli/cli/commands/next_cmd.py
- tests/integration/test_explicit_checkout_commands.py
- tests/specify_cli/cli/commands/test_next_answer_effective_root.py
- tests/integration/test_owned_lifecycle_acceptance_next.py
role: implementer
tags: []
task_type: implement
tracker_refs: []
---

# Work Package Prompt: WP19 – next CLI entry takes the fact

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

WP11 made `runtime/next` carry the `OwnedCheckout` fact and proved it at the runtime entry points. This WP makes the **CLI entry**, `spec-kitty next`, validate ownership once through the sole minter, hand the fact to the runtime, and proves the rows through the real command:

| Row | Outcome | Proof (red on base → green here) |
|---|---|---|
| FR-002 / O9 | `next --owned-checkout` goes through `resolve_owned_or_adopt(…, NEXT_OWNED_TOPOLOGIES)` and gets branch and protection checks. A protected target is refused with `OWNED_BRANCH_REFUSED`. | CLI test with a protected target, plus a valid control |
| FR-023 | A `lanes_with_coord` mission is accepted by `next`, and the same mission is refused by a lifecycle command with `OWNED_TOPOLOGY_UNSUPPORTED`. A plain `lanes` mission is accepted by `next` the same way (U1: `next` keeps accepting every topology it accepts today). | paired CLI test |
| FR-008 / O5 | `next --result success` at tasks→implement raises no traceback, the workspace is P, and the run is not wedged. The full US3-AS1 row (`kind == "step"` with a prompt file) is a strict xfail that WP12 removes. | CLI test |
| FR-007 | With a stale copy of M in R, the `next` JSON carries `stale_repository_root_copy` (owned runs only), and no workspace path is under R. A non-owned payload is byte-identical to the base. | CLI test with `stale_root_copy` + non-owned control |
| FR-022 | Coordination-topology owned `next` still works: the per-PR in-process CLI twin (owned `lanes_with_coord` create → `next`). | twin (green on base, ratchet) |
| FR-003 | Exactly one ownership validation per `next --owned-checkout` invocation. | call-count assertion |
| FR-012 (CLI envelope) | WP11's `Decision.error_code` reaches `next --json` as `error_code`. | one subprocess-seam injection through the real command |
| US3-AS6 | R's branch is flipped by a test hook between validation and prompt build; all reads still use the fact from the start of the command, and validation count stays 1. | monkeypatch hook on the AS-1 fixture (T102 step 10) |

**Done when:**
- every row above has a red-first test **committed before** its fix commit (reviewers verify the commit order in `git log`);
- `grep -n "effective_root" src/specify_cli/cli/commands/next_cmd.py` is empty;
- `grep -n "resolve_ownership_claim\|error_for_claim\|_emit_checkout_ownership_error" src/specify_cli/cli/commands/next_cmd.py` is empty (G1/G2 floor offender `next_cmd.py:170` removed);
- `grep -rn "bridging: WP19" src` is empty; this WP adds **no** `TRANSITIONAL(WP18)` marker (`grep -c "TRANSITIONAL(WP18)" src/specify_cli/cli/commands/next_cmd.py` is 0);
- WP11's six legacy `effective_root` markers in the runtime have **no caller left** in `src/` (`grep -rn "effective_root=" src/specify_cli/cli/commands/next_cmd.py` empty), so WP18 T096 deletes them safely;
- ruff, format and mypy `--strict` are clean; `make test-fast` is green.

## Context & Constraints

- **Read first:**
  - `spec.md`: US3, US7, O5, O9, FR-002/003/007/008/022/023.
  - `plan.md`: Staging Strategy, Test Layout, IC-05, IC-08, IC-13.
  - `research.md`: R-02, R-12, R-15.
  - `data-model.md`: registry codes and the `stale_repository_root_copy` payload.
  - `contracts/cli-owned-checkout-surface.md`: the `next` row.
  - `occurrence_map.yaml`.
- **What the dependencies provide.** Read their merged code, not the plan.
  - **WP01**: `OwnedCheckout` and the `OwnedRefusalCode` `StrEnum`. Import codes from it; never repeat a code literal.
  - **WP02**: `resolve_owned_mission`, `NEXT_OWNED_TOPOLOGIES`, and the shared fixtures `owned_checkouts` / `make_owned_checkouts`, `owned_handle`, `owned_cwd`, `r_snapshot`, `stale_root_copy` in `tests/integration/conftest.py`.
  - **WP05**: owned workspace cache isolation (FR-019). **No fallback:** if WP05's owned API that the carried cache test needs is missing, this WP is **blocked**; file it against WP05 and stop.
  - **WP08**: `cli/commands/_owned_checkout.py` with `OwnedCheckoutOption`, `resolve_owned_or_adopt`, `emit_owned_refusal` and the stale-copy reporter. Use its **actual** signatures.
  - **WP11**: the runtime entry points accept `owned=`: `decision.decide_next`, `runtime_bridge.query_current_state`, `runtime_bridge.answer_decision_via_runtime`, and the three `next_invocation_lifecycle` entry points. They still carry a marked legacy `effective_root` keyword for this file; after this WP nothing passes it. `Decision.error_code` carries typed refusal codes.
- **Root discipline.** `next_cmd` hands the runtime `repo_root = owned.repository_root` (R) plus `owned`. Non-owned-aware CLI-local callees (`safe_commit`, the charter preflight) receive `owned.owned_root`, which is today's behaviour (today `repo_root` *is* P, `next_cmd.py:176-178`).
- **Do not edit** WP11's runtime files. If a runtime change is needed, file it against WP11.
- **Terminology.** "repository root checkout" and "owned checkout"; rename test locals `primary` to `repository_root` in tests you touch; no `feature` aliases.
- **Error contract.** Refusals carry `error_code` from `OwnedRefusalCode` (NFR-004). Tests assert on `error_code` only, never on git stderr.

## Branch Strategy

- **Strategy:** the planning artifacts were generated on `claude/sleepy-hamilton-5lelee`, and completed changes merge back into `claude/sleepy-hamilton-5lelee`.
- **Planning base branch:** `claude/sleepy-hamilton-5lelee`
- **Merge target branch:** `claude/sleepy-hamilton-5lelee`
- **Execution lane:** assigned by `finalize-tasks` in `kitty-specs/owned-checkout-lifecycle-authority-01M3M2ZB/lanes.json`. Start with `spec-kitty implement WP19` and use the workspace it resolves. Never reconstruct the path yourself.

> These fields are populated automatically by `spec-kitty agent mission finalize-tasks`.

## Subtasks & Detailed Guidance

### Subtask T101 – Carry #5009 edaa9cd83 and 1be5352ee; adapt the call shape (was T056)

- **Purpose:** C-006 provenance. Both commits (author Samuel Goff) add tests to `tests/integration/test_explicit_checkout_commands.py` and reproduce real defects:
  - **edaa9cd83**: `test_finalized_owned_tasks_resolve_for_next_implementation` (O5 at the decision seam) and `test_wp_cache_is_scoped_to_selected_checkout` (FR-019 cache isolation).
  - **1be5352ee**: extends the cache test; adds `test_owned_composition_policy_reaches_executor_before_advancing` (FR-009 composition, R-11) and `test_owned_review_prompt_uses_owned_target_branch`.
- **Steps:**
  1. Carry both commits verbatim, in order: `git cherry-pick -x edaa9cd83` then `git cherry-pick -x 1be5352ee` (from the local ref `pr-5009`). Author and date are preserved.
  2. Run them and record in the Activity Log why each is red on the WP11 result (expected: `TypeError`, because the carried shape passes `effective_root=` to functions that now take `owned=`). The **behavioural** red-first proof for these defects is WP11's T057 (runtime level) and this WP's T102 (CLI level); the carried commits are provenance.
  3. **Adapt in a separate commit.** Change the call shape "`repo_root`=R, `effective_root`=P" to "`repo_root`=R, `owned`=fact", minted by the **real validator** `resolve_owned_mission(repository_root, owned, SLUG, allowed_topologies=NEXT_OWNED_TOPOLOGIES)` (or WP02's fixture helper). Never construct `OwnedCheckout` directly (tests may reference `_mint` — gate G3 scans `src/` only — but prefer the fixture helper).
     - `test_finalized_owned_tasks_resolve_for_next_implementation`: `_state_to_action("implement", SLUG, owned / "kitty-specs" / SLUG, repository_root, "software-dev", owned=fact)` returns `("implement", "WP01", str(fact.owned_root))`, plus the board-authority assertion through `runtime_bridge._wp_iteration_action_and_state(..., owned=fact)`.
     - The cache test: use the owned keyword WP05 exposed (`get_normalized_wp(..., owned=fact)` or `resolve_workspace_for_wp(repository_root, SLUG, "WP01", owned=fact)`). **No fallback** (Dependencies).
     - The composition test: `context = SimpleNamespace(..., repo_root=repository_root, owned=fact, ...)`; the `advance` stub asserts `kwargs["owned"] is fact` and `kwargs["repo_root"] == repository_root`; `inputs`/`execute` assert `repo_root == fact.owned_root`.
     - `test_owned_review_prompt_uses_owned_target_branch` stays as carried, marked `pytest.mark.xfail(strict=True, reason="WP12 T069: review assertion rewritten to the claim-commit contract (FR-025)")`. WP12 owns that rewrite.
  4. **Optional provenance: 22f380055** (formatter exclusion for `runtime_bridge_engine.py`) is WP11's file; do not carry it here. Record `DROP (WP11-owned file)` in the Activity Log.
- **Files:** `tests/integration/test_explicit_checkout_commands.py`.
- **Validation checklist:**
  - [ ] `git log --format='%an%n%b' -2` on the carried commits shows Samuel Goff and the `(cherry picked from commit …)` trailers.
  - [ ] The adapted tests are green; the review test is a strict xfail, not a skip.
- **Edge cases:** the carried fixture `checkouts` creates a `sibling` worktree. Keep it: it guards "not read or written".

### Subtask T102 – Red-first CLI acceptance: O5, O9, stale copy, FR-023 pairing, lanes_with_coord CLI twin, FR-003 count

- **Purpose:** C-007. Every row gets a reproduction through the **pre-existing entry point**, the real `next` Typer command driven in-process with `typer.testing.CliRunner`, in one new file committed **before** any `next_cmd.py` fix.
- **Steps:**
  1. Create `tests/integration/test_owned_lifecycle_acceptance_next.py` with `pytestmark = [pytest.mark.integration, pytest.mark.git_repo]`. Use WP02's shared fixtures (never import fixtures from test modules). `make_owned_checkouts(...)` is not finalized: run `agent mission finalize-tasks --owned-checkout P --mission H` in setup. Invoke the command through the `next` app/command object from `specify_cli.cli.commands.next_cmd`, as `tests/specify_cli/cli/commands/` does.
  2. **O9 / FR-002.** `make_owned_checkouts(protected_target=True)` (or set `protection.protected_branches: [main, <P's target>]` in P's `.kittify/config.yaml`, following `test_explicit_checkout_commands.py:247-260`). `next --owned-checkout P --mission H --json` → non-zero exit and `json["error_code"] == OwnedRefusalCode.OWNED_BRANCH_REFUSED`. Same-fixture control without protection passes. Red on the base: the claim-only path at `next_cmd.py:161-178` accepts it.
  3. **FR-023 pairing.** A `lanes_with_coord` owned mission (step 6): `next --owned-checkout P` passes validation; `agent tasks status --owned-checkout P` (WP09) refuses it with `OWNED_TOPOLOGY_UNSUPPORTED`. **U1 pin (`LANES` in `NEXT_OWNED_TOPOLOGIES`):** a plain `lanes` (no coordination) owned mission also passes `next --owned-checkout P` validation (green on the base, since today's `next` applies no topology check at all — this is the ratchet that keeps that behaviour), while the same mission is refused by `agent tasks status --owned-checkout P` with `OWNED_TOPOLOGY_UNSUPPORTED`, same as the `lanes_with_coord` pairing.
  4. **O5 / FR-008.** On a finalized owned mission, `next --agent claude --owned-checkout P --mission H --result success --json`. Assert: no traceback (`result.exception` is `None` or `SystemExit`); no `ValueError` in the output; not `blocked` for a workspace-resolution reason; `workspace_path == str(P)`, `action == "implement"`, `wp_id == "WP01"`; a following query does **not** return the same unadvanced decision.

     The full US3-AS1 row goes in a **separate** test: `kind == "step"` with an existing `prompt_file` whose workspace is P, marked `xfail(strict=True, reason="WP12 T068: prompt builder owned arm")`. WP12 removes the marker (declared out-of-map edit in this file).
  5. **O8 CLI envelope.** Monkeypatch `specify_cli.coordination.workspace.subprocess.run` to fail with exit 128 only for `"worktree", "list"` on an owned `lanes_with_coord` mission; assert `json["error_code"] == OwnedRefusalCode.OWNED_COORDINATION_WORKSPACE_UNAVAILABLE`. If `_print_decision` already serializes `Decision.error_code`, this row is green on the WP11 result; record that.
  6. **FR-022 in-process CLI twin.** `agent mission create <slug> --topology lanes_with_coord --owned-checkout P --start-branch <b> --pr-bound --branch-strategy already-confirmed --json` through `CliRunner`, mirroring `tests/e2e/test_worktree_owned_root_concurrency.py:395-431`; then `next --owned-checkout P --mission <slug> --json`. Expect exit 0 and a non-error decision. Green on the base (ratchet); record that.
  7. **Stale copy (FR-007).** With `stale_root_copy`, `next --owned-checkout P --result success --json`: `json["stale_repository_root_copy"]["path"]` names R's copy, and `workspace_path` is not under R (`Path.is_relative_to`, excluding P's subtree when P lives under R). Non-owned control on the same repository: the key is **absent** and the payload is byte-identical to the base.
  8. **NFR-001.** Wrap each owned scenario in `r_snapshot`; 0 differences. Run the owned scenarios across the `owned_cwd` matrix (cwd = R, P, unrelated directory; `monkeypatch.chdir`).
  9. **FR-003 count.** Patch `specify_cli.core.checkout_ownership.resolve_ownership_claim` with a counting wrapper; clear the workspace caches first with `clear_workspace_resolution_caches()`. Assert exactly **1** call per `next --owned-checkout` invocation (red on the WP11 result with a count ≥ 2: the claim-only CLI check plus each runtime legacy arm; record the observed count).
  10. **US3-AS6 / fact-not-re-derived.** On the fixture of step 4 (AS-1: a finalized owned mission, `next --owned-checkout P --result success --json`), install a monkeypatch hook on `specify_cli.core.paths.get_main_repo_root` (or the equivalent site `resolve_owned_or_adopt` calls after validation, whichever the merged WP08/WP11 code actually reads for R's branch) that flips R's current branch (`git -C R checkout -q -b <other>`) after the single `resolve_owned_or_adopt` call and before the prompt is built. Reuse the same `resolve_ownership_claim` counting wrapper from step 9. Assert:
     - the validation count stays exactly **1** (no second validation is triggered by the branch flip);
     - `workspace_path == str(P)` and no output path is under R;
     - the command still exits 0 with `action == "implement"`.

     This is deliberately a stronger, more direct proof than the FR-003 call-count assertion alone (analysis C1): the call-count check would not catch re-derivation of R's identity through `get_main_repo_root(cwd)` or `locate_project_root`, whereas flipping R's branch mid-command and asserting the count stays 1 **and** all outputs stay under P catches both a second validation and a same-fact-but-re-derived-context bug.
  11. Commit: `test(next): red-first owned next CLI acceptance (O5, O9, FR-007/022/023, FR-003, US3-AS6)`.
- **Validation checklist:**
  - [ ] Every row is red on the WP11 result for the stated reason (or recorded as a green ratchet), recorded in the Activity Log; the red commit precedes every fix commit.
  - [ ] No assertion reads git stderr text. `SPEC_KITTY_ENABLE_SAAS_SYNC=0` comes from the integration conftest autouse fixture.
  - [ ] US3-AS6's branch-flip row is green: the validation count stays 1 and every output path stays under P after R's branch changes mid-command.

### Subtask T103 – `next_cmd` through `resolve_owned_or_adopt(NEXT_OWNED_TOPOLOGIES)` (was T059, plus the CLI campsites)

- **Purpose:** FR-002 / FR-003 / O9. Replace the claim-only path (`next_cmd.py:159-178`: a direct `resolve_ownership_claim` plus `error_for_claim`, then `repo_root = effective_root = claim.claimed_checkout`) with the single validator.
- **Steps:**
  0. **Campsite first (own commit, behaviour-preserving).**
     - `next_step` (`next_cmd.py:119-268`, complexity 12): extract `_resolve_slug_or_exit(mission, repo_root, json_output, ...)` (the five-arm `except` ladder at `:198-224`, unchanged) and `_resolve_next_owned(...)` (the owned block `:159-178`).
     - `_print_standard_human` (`next_cmd.py:1202`, complexity 14): one small printer per decision kind; the outer function is a dispatch. The stale-copy warning is printed at command level (T104), not inside the printers.
     - Commit: `refactor(next): campsite extractions in next_cmd (C901)`. Grep `tests/` for `next_cmd.<name>` patch strings first and keep patched names.
  1. Replace the inline `owned_checkout: Annotated[Path | None, typer.Option("--owned-checkout", …)]` (`:132-137`) with WP08's `OwnedCheckoutOption`.
  2. In `_resolve_next_owned`, call `resolve_owned_or_adopt(...)` with `allowed_topologies=NEXT_OWNED_TOPOLOGIES` and the `--mission` handle. On refusal, call `emit_owned_refusal(...)`, keeping `next`'s envelope `{success, error_code, error}`, and exit 1. On success, `repo_root = owned.repository_root`, and hold `owned`.
  3. **Handle-less owned `next` (resolution in P).** `resolve_owned_mission` requires a handle. If WP08's helper supports `handle=None` (discovery inside the claimed checkout after the claim check), use it. Otherwise run the sole-mission discovery `_missing_handle_or_sole_slug` (`:575`) against the **claimed** checkout *only after* the helper has confirmed the claim, then validate with that handle. **Never** list P's missions before the claim check. Record which path you took.
  4. **Flagless adoption.** Rework `_require_main_repo_unless_owned` (`:80-98`) so that a flagless invocation from inside a linked checkout first attempts validated adoption through `resolve_owned_or_adopt`: a fact → proceed owned; `None` (lane worktree, coordination worktree, rejected checkout) → fall through to the **unchanged** legacy guard (US7-AS2/AS3).
  5. Delete `_emit_checkout_ownership_error` (`:101-115`) and the `TYPE_CHECKING` import of `CheckoutOwnershipError` (`:42-44`) once they have no users.
  6. `_run_charter_preflight_for_next` (`:487`): the charter `next` enforces is P's, so pass `owned.owned_root` when owned. Pin it with a unit test.
- **Validation checklist:**
  - [ ] T102's O9 row turns green, and its valid control stays green; the FR-003 count is exactly 1.
  - [ ] The campsite commit precedes the signature changes; `ruff check --isolated --select C901 --config 'lint.mccabe.max-complexity=11' src/specify_cli/cli/commands/next_cmd.py` no longer lists `next_step` or `_print_standard_human`.

### Subtask T104 – `stale_repository_root_copy` in the `next` payload (owned runs only)

- **Purpose:** FR-007 / R-12.
- **Steps:**
  1. After the decision is computed, and only when `owned` is set, call WP08's stale-copy reporter. Add the top-level `stale_repository_root_copy: {"path", "mission_id"} | null` to the `--json` decision output in `_print_decision` (`:1061`) in owned runs only. A non-owned run never emits the key.
  2. In human mode, print the warning on **stderr** at command level. Detection compares the repository-root placement against `owned.mission_dir` and makes no git calls.
  3. If a golden JSON contract (WP09's files) pins `next`'s key set: the key is emitted only in owned runs, so a non-owned golden needs no change. If an owned golden exists, the additive key is a declared out-of-map edit with a rationale.
- **Validation checklist:**
  - [ ] T102's stale-copy row is green; the field is `null` when R holds no copy; the non-owned control payload is byte-identical.
  - [ ] Occurrence map: the key is **new** (additive). No existing key or `OWNED_*` code string changes.

### Subtask T105 – Remaining `next_cmd` conversions (the CLI part of the former T063)

- **Purpose:** leave `next_cmd.py` with no bare owned roots (G4/G5) and no caller of WP11's legacy keywords.
- **Steps:**
  1. Thread `owned` (never a bare path) into every downstream call in `next_step`: `_resolve_mission_slug`, `_maybe_handle_answer`, `_dispatch_query_mode` and `_dispatch_advancing_mode`, replacing their `effective_root: Path | None` keywords (45 sites on HEAD).
  2. `decide_next` (`next_cmd.py:59-72`, the patchable wrapper) forwards `owned=` to `decision.decide_next`. `query_current_state`, `answer_decision_via_runtime` (`_handle_answer`, `:1074`) and the three lifecycle wrappers (`_pair_previous_lifecycle_record`, `_write_issuance_lifecycle_record`, `_emit_mission_next_invoked`) pass `owned=`. `_handle_answer`'s `mission_context_for(..., effective_root=)` fork (`:1118-1130`) becomes `mission_context_for(repo_root, slug, owned=owned)`.
  3. `_commit_owned_next_mutations(effective_root, mission_slug)` (`:371-419`) becomes `_commit_owned_next_mutations(owned)`: `owned.mission_dir` instead of `compose_meta_json_path(effective_root, slug).parent`, the owned root for `safe_commit(repo_root=…, worktree_root=…)`, and `mission_context_for(owned.repository_root, owned.mission_slug, owned=owned)` for the commit target.
  4. `_resolve_mission_slug` (`:598-662`) with a fact returns `owned.mission_slug`.
  5. Re-point `tests/specify_cli/cli/commands/test_next_answer_effective_root.py` to `owned` (keep the filename, which is pinned; update its docstring).
  6. Commit the conversions separately from the functional fixes.
- **Validation checklist:**
  - [ ] `grep -n "effective_root" src/specify_cli/cli/commands/next_cmd.py` is empty.
  - [ ] `grep -rn "effective_root=" src/ | grep -v "TRANSITIONAL(WP18)"` shows no call into WP11's six legacy keywords (they are now callerless; WP18 deletes them).
  - [ ] mypy `--strict` is clean on the invocation below.

## Test Strategy

Red-first order (each red is a **commit** preceding its fix commit):
1. T101 carried commits (provenance) and T102 reds.
2. T103 step 0 campsite (green).
3. T103, T104 fixes; T105 conversion.

```bash
PWHEADLESS=1 .venv/bin/python -m pytest -q \
  tests/integration/test_owned_lifecycle_acceptance_next.py \
  tests/integration/test_explicit_checkout_commands.py \
  tests/specify_cli/cli/commands/test_next_answer_effective_root.py \
  tests/integration/test_owned_next_runtime.py
PWHEADLESS=1 .venv/bin/python -m pytest -q tests/specify_cli/cli/commands/ -k next
grep -rl "next_cmd" tests/ --include="*.py"  # run each hit not already covered (excluding e2e/slow)
PWHEADLESS=1 .venv/bin/python -m pytest -q tests/next/
make test-fast
.venv/bin/ruff check src/specify_cli/cli/commands/next_cmd.py tests/integration/test_owned_lifecycle_acceptance_next.py tests/integration/test_explicit_checkout_commands.py tests/specify_cli/cli/commands/test_next_answer_effective_root.py
.venv/bin/ruff format --check src/specify_cli/cli/commands/next_cmd.py tests/integration/test_owned_lifecycle_acceptance_next.py tests/integration/test_explicit_checkout_commands.py tests/specify_cli/cli/commands/test_next_answer_effective_root.py
.venv/bin/mypy --strict src/specify_cli/cli/commands/next_cmd.py src/specify_cli/cli/commands/_owned_checkout.py src/specify_cli/core/owned_mission.py src/runtime/next/decision.py src/runtime/next/runtime_bridge.py src/runtime/next/next_invocation_lifecycle.py src/mission_runtime/owned_checkout.py
.venv/bin/python -m pytest -q tests/architectural/test_layer_rules.py tests/architectural/test_cold_import_status_boundary.py
```

- **mypy discipline:** never drop `--strict`; callers and callees in one invocation. If the base has pre-existing errors in these files, record the pre-existing count (same command on the planning base) and show it did not grow.
- Do **not** run `make test-full` or the bare `tests/architectural/` directory. The nightly e2e `tests/e2e/test_worktree_owned_root_concurrency.py` is covered per-PR by the T102 twin.

## Risks & Mitigations

- **Validation count.** Adoption plus explicit validation must not both run. `resolve_owned_or_adopt` is the single call site; the FR-003 counter guards it, and it can only reach 1 once every runtime legacy arm has lost its caller (T105).
- **Monkeypatch churn.** Tests patch `next_cmd.<name>`; keep names stable.
- **Handle discovery before the claim check** would read P before validating it. The ordering rule in T103 step 3 prevents it.

## Review Guidance

- Verify the red-first commit order in `git log` (T102 red commit before the T103/T104 fix commits) and the recorded red reasons.
- Verify the carried commits' authorship and `-x` trailers, and that the review test is a strict xfail.
- Verify that `next_cmd.py` has no `effective_root`, no claim-primitive call, and no `_emit_checkout_ownership_error`; and that no WP11 runtime file was edited.
- Verify the non-owned `next` payload is byte-identical (no `stale_repository_root_copy` key).
- Verify mypy ran with `--strict` over callers and callees in one invocation.

## Activity Log

> Append entries at the END in chronological order: `- YYYY-MM-DDTHH:MM:SSZ – <agent_id> – <action>`.

- 2026-09-28T16:00:00Z – system – Prompt created (split from WP11; carries the former T056/T059 as T101/T103).

---

### Updating Status

Use `spec-kitty agent tasks move-task WP19 --to <status> --mission owned-checkout-lifecycle-authority-01M3M2ZB` and `spec-kitty agent tasks mark-status T101 … --status done`.
