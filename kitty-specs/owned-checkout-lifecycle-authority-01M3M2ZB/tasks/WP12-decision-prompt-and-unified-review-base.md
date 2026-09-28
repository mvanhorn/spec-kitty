---
work_package_id: "WP12"
subtasks:
  - "T064"
  - "T065"
  - "T066"
  - "T067"
  - "T068"
  - "T069"
title: "Decision, prompt building and unified review base"
task_type: "implement"
phase: "Phase 3 - Commands and runtime"
execution_mode: "code_change"
owned_files:
  - "src/runtime/next/decision.py"
  - "src/runtime/next/prompt_builder.py"
  - "src/specify_cli/cli/commands/agent/workflow_executor.py"
  - "src/specify_cli/core/worktree_topology.py"
  - "src/mission_runtime/claim_commit.py"
  - "tests/mission_runtime/test_claim_commit.py"
  - "tests/integration/test_owned_lifecycle_acceptance_review.py"
authoritative_surface: "src/runtime/next/prompt_builder.py"
create_intent:
  - "src/mission_runtime/claim_commit.py"
  - "tests/mission_runtime/test_claim_commit.py"
  - "tests/integration/test_owned_lifecycle_acceptance_review.py"
dependencies:
  - "WP11"
requirement_refs:
  - "FR-008"
  - "FR-009"
  - "FR-010"
  - "FR-025"
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

# Work Package Prompt: WP12 – Decision, prompt building and unified review base

## ⚡ Do This First: Load Agent Profile

Use the `/ad-hoc-profile-load` skill to load the agent profile named in the frontmatter, then follow its guidance before you read the rest of this prompt.

- **Profile**: `python-pedro`
- **Role**: `implementer`
- **Agent/tool**: `claude`

Then load the charter (`.kittify/charter/charter.md`) and the action context: `spec-kitty charter context --action implement --json`.

---

## ⚠️ IMPORTANT: Review Feedback

- **Has review feedback?** Check `review_ref` in the event log (`spec-kitty agent tasks status --mission owned-checkout-lifecycle-authority-01M3M2ZB`) or the Activity Log below.
- **Address every item** before you move the WP back to `for_review`, and log each fix in the Activity Log.

---

## Review Feedback

*[Empty until this WP is returned from review.]*

---

## Markdown Formatting

Wrap HTML/XML tags in backticks: `` `<div>` ``, `` `<script>` ``
Use language identifiers in code blocks: ````python`, ````bash`

---

## Objectives & Success Criteria

This WP finishes the owned `next` path that WP11 started, and gives every review path one claim-commit authority (operator decision `01M3M70Y…`):

1. **FR-025.** A new helper, `mission_runtime.claim_commit_for_wp`, finds a WP's **claim commit** the same way for every review path, owned and repository-root alike:
   1. take the last `to_lane == "claimed"` status event for the WP;
   2. find the unique commit on `HEAD` whose diff to `status.events.jsonl` introduced that `event_id`.

   It fails closed when there are zero or several such commits. The three duplicate commit-subject matchers are **deleted**:
   - `workflow_executor.py:2259`
   - `prompt_builder.py:255`
   - `core/worktree_topology.py:125`
2. **FR-025 red-first.** On the planning base, a *non-owned* finalized single_branch mission shows "unavailable: no deterministic implementation claim commit found" in its review prompt. The subject matcher cannot match the event-only `move-task` commit (R-05). After this WP the prompt shows `git diff <claim>..HEAD`.
3. **FR-010 / US3-AS4.** An owned WP's review prompt uses its claim commit as the base. The diff is `claim..HEAD -- <WP owned_files>` and excludes a sibling WP's files.
4. **FR-010 / US3-AS5.** If the WP declares no `owned_files`, or has no single claim commit, `next --owned-checkout` returns a `blocked` decision with `error_code == "OWNED_REVIEW_BASE_UNAVAILABLE"`. It never emits an unscoped whole-checkout diff.
5. **FR-008 / FR-009 (prompt half).**
   - `decision.py` and `prompt_builder.py` take the fact.
   - Owned WP prompts read the WP file, workspace, governance and mission-type context from P.
   - The full US3-AS1 row (`kind == "step"` with an existing prompt file, workspace P) turns green. WP11 committed it as a strict xfail.
6. **Carried test rewrite.** #5009 1be5352ee's `test_owned_review_prompt_uses_owned_target_branch` asserts `git diff {TARGET}..HEAD --stat`. It is rewritten to the claim-commit contract.

**Done when:**
- every row above has its red-first test committed before its fix;
- `grep -rn 'claimed for implementation\|Move {wp_id} to in_progress\|Start {wp_id} implementation' src/` returns nothing;
- `decision.py` and `prompt_builder.py` contain no `effective_root`;
- ruff, format and mypy are clean on the touched files;
- `make test-fast` is green.

## Context & Constraints

- **Read first.**
  - `spec.md`: US3-AS1/AS4/AS5, the "Claim commit" definition, FR-008/009/010/025.
  - `plan.md`: Staging Strategy, IC-05, IC-06, IC-13.
  - `research.md`: R-05 (the algorithm), R-15, R-16 "Review-base duplication".
  - `data-model.md`: the review-base status event fact and the `OWNED_REVIEW_BASE_UNAVAILABLE` code.
  - `contracts/owned-checkout-carrier.md` §7: the prompt builder and the review-base helper are consumers of the fact.
- **WP11 provides** `DecideNextContext.owned`, owned board authority, and bridging calls into this WP's files. The bridging looks like `repo_root=owned.owned_root if owned else repo_root`, each call marked `# bridging: WP12 converts`. Replace every one of them.
- **Layer rules for the new helper.**
  - `src/mission_runtime/claim_commit.py` sits in the `mission_runtime` layer (`kernel <- charter <- {…, mission_runtime} <- specify_cli`, `tests/architectural/test_layer_rules.py:82-84`).
  - It may import `specify_cli.status` **lazily, inside the function**. `status` is already an allowed subpackage in `_MISSION_RUNTIME_ALLOWED_SPECIFY_CLI` (`test_layer_rules.py:116-144`), so the cap-10 ledger in `_baselines.yaml:21` does not grow.
  - A module-level import would break `test_package_root_cold_imports` (`test_mission_runtime_surface.py:180`).
- **Git access from `mission_runtime`.** No general-purpose kernel git seam exists. `kernel/git_topology.py` exposes only `git_common_dir` / `git_toplevel`, and its runner `_run_rev_parse` is private. Follow the established in-layer precedent, `mission_runtime/lifecycle_phase.py:174-185`: call `subprocess.run([...], cwd=…, capture_output=True, text=True, timeout=<module constant>, check=False)` directly, and translate `TimeoutExpired` and non-zero exits into a typed error. Do **not** import `specify_cli.git`, because `git` is not an allowed subpackage and importing it would grow the ledger. This deliberately deviates from plan IC-06's phrase "git through the kernel seam". Record the deviation and this rationale in the PR.
- **Public surface (MR-1/MR-2).** Code outside the package may import `mission_runtime` symbols only from the package root. You must therefore make two **declared out-of-map edits** to WP01-owned files, and put this rationale in the PR: "MR-1/MR-2 forbid submodule imports; FR-025 needs the helper at the package root."
  1. Re-export `claim_commit_for_wp` and its error type from `src/mission_runtime/__init__.py` and `__all__`.
  2. Add both names to `_PUBLIC_SURFACE` in `tests/architectural/test_mission_runtime_surface.py:49`. `test_public_surface_is_exactly_all` requires `__all__ == _PUBLIC_SURFACE`.

  Also add a one-line note in the ADR 2026-06-07-1 amendment if WP03 left a list of public symbols to extend. Otherwise note it in the PR for WP18.
- **Helper signature (deviation from plan IC-06).** Plan IC-06 names the helper `claim_commit_for_wp(checkout_root, mission_slug, wp_id)`. WP18's gate G5 bans any parameter named `checkout_root` that is annotated `Path` (`contracts/architectural-gate.md`). Use this signature instead:

  ```python
  claim_commit_for_wp(mission_dir: Path, wp_id: str) -> str
  ```

  `mission_dir` is the directory that holds `status.events.jsonl` in the checkout whose `HEAD` is being reviewed. Git runs with `cwd=mission_dir`, and the pathspec `status.events.jsonl` is relative to it. Record the deviation in the PR.
- **Staging.** WP11 already routes the runtime through the fact.
  - This WP converts `decision.py` (except `decide_next`, which WP11 converted as a declared edit) and `prompt_builder.py` to `owned: OwnedCheckout | None`.
  - Callers in `runtime_bridge.py` (WP11-owned) must switch from the bridging value to `owned=`. That is a mechanical, declared out-of-map edit in `src/runtime/next/runtime_bridge.py`. Rationale: "top-down completion of WP11's bridging (staging rule)."
  - The six seams keep their dual keyword until WP18.
- **Out-of-map test edits (declared).**
  - `tests/integration/test_explicit_checkout_commands.py` (WP11): rewrite the carried review test, and remove its strict-xfail marker.
  - `tests/integration/test_owned_lifecycle_acceptance_next.py` (WP11): remove the strict-xfail marker on the full US3-AS1 row.

  Both are needed because `prompt_builder.py` (this WP) is what turns those tests green.
- **Campsite first.** `_state_to_action` (`decision.py:394`, complexity 14) and `_build_wp_prompt` (`prompt_builder.py:152`, complexity 14) are extracted before their signatures change.
- **Terminology.** Use "claim commit", "owned checkout" and "repository root checkout". In user-facing prompt text, never write bare "primary" or "feature".

## Branch Strategy

- **Strategy**: planning artifacts were generated on `claude/sleepy-hamilton-5lelee`, and completed changes merge back into `claude/sleepy-hamilton-5lelee`.
- **Planning base branch**: `claude/sleepy-hamilton-5lelee`
- **Merge target branch**: `claude/sleepy-hamilton-5lelee`
- **Execution lane**: `finalize-tasks` assigns it in `kitty-specs/owned-checkout-lifecycle-authority-01M3M2ZB/lanes.json`. Run `spec-kitty implement WP12` and use the workspace it resolves. WP12 depends on WP11, so its lane is based on WP11's result.

> `spec-kitty agent mission finalize-tasks` fills these fields in automatically.

## Subtasks & Detailed Guidance

### Subtask T064 – Red-first: non-owned review base; owned review AS4/AS5

- **Purpose**: C-007. Reproduce the three review defects through the **pre-existing entry points** before any fix. The entry points are `spec-kitty next` (review prompt) and `agent action review` (repository-root review context).
- **Steps**:
  1. Create `tests/integration/test_owned_lifecycle_acceptance_review.py` with `pytestmark = [pytest.mark.integration, pytest.mark.git_repo]`. Use WP02's `owned_checkouts` and `r_snapshot` fixtures from `tests/integration/conftest.py`.
  2. **FR-025, non-owned (red on base).**
     - Build a repository-root single_branch mission with the same fixture machinery, without `--owned-checkout`.
     - Finalize, then run `move-task WP01 --to claimed`, then `--to in_progress`.
     - Commit a change to a WP01-owned file, then run `move-task WP01 --to for_review`.
     - Run `next --agent claude --mission H --json` so that it returns the WP01 review step, and read the prompt file.
     - Assert `f"git diff {claim}..HEAD --stat" in prompt`. Compute `claim` independently in the test:

       ```
       git log --format=%H -S<event_id> HEAD -- kitty-specs/<slug>/status.events.jsonl
       ```

       where `event_id` is the last `claimed` event read from the log.
     - On the base, the prompt contains "unavailable: no deterministic implementation claim commit found" (`prompt_builder.py:266`).
  3. **US3-AS4, owned.**
     - Using `owned_checkouts` (P with WP01 and WP02; the fixture is not finalized, so run `finalize-tasks --owned-checkout P` first), run `move-task WP01 --to claimed/in_progress --owned-checkout P`.
     - Commit a change to a WP01 file in P, then commit a **WP02** file change in P, then run `move-task WP01 --to for_review --owned-checkout P`.
     - Run `next --owned-checkout P --mission H --json`.
     - Assert that the review prompt contains `git diff {claim}..HEAD --stat -- <WP01 owned_files…>` and does not name WP02's files.
     - Run the printed diff command in P and assert that WP01's change is present and WP02's is absent.
  4. **US3-AS5, owned.**
     - **(a)** WP01 declares `owned_files: []`.
     - **(b)** No claim event exists (the WP was moved straight into `for_review` with `--force`, or its claim event was stripped from the fixture log).
     - **(c)** Ambiguous: two commits match. Simulate this by committing a second change that removes and re-adds the event line.

     Each case must end with `next` returning `kind == "blocked"`, `error_code == "OWNED_REVIEW_BASE_UNAVAILABLE"`, and a prompt text (if any) containing no bare `git diff … ..HEAD` without a pathspec.
  5. **Repository-root review context (red on base).** Run `agent action review WP01 --mission H` on the non-owned fixture, through the workflow app and CliRunner. Assert that the printed review context names the claim commit as its base. On the base, the subject matcher at `workflow_executor.py:2259` finds nothing and leaves `ctx` without a base.
  6. Wrap each owned scenario in `r_snapshot` (NFR-001: 0 differences).
  7. Commit: `test(review): red-first unified claim-commit review base (FR-010, FR-025)`.
- **Files**: `tests/integration/test_owned_lifecycle_acceptance_review.py` (new).
- **Validation checklist**:
  - [ ] Every row is red on the WP11 result for the stated reason, recorded in the Activity Log.
  - [ ] Every negative has a same-fixture positive control (AS4 is the control for AS5).
  - [ ] No assertion depends on commit-subject wording.
- **Edge cases**: `move-task` claim commits touch only `status.events.jsonl` and `status.json` (R-05). The `-S` pathspec must be the events file, **not** `status.json`, which may also carry the event id.

### Subtask T065 – `claim_commit_for_wp` helper plus uniqueness pin

- **Purpose**: FR-025. One fail-closed authority for "which commit claimed this WP", replacing three subject matchers that have been broken for every mission since the event-only cutover (R-16).
- **Steps**:
  1. Create `src/mission_runtime/claim_commit.py`:
     - Define `class ClaimCommitUnresolved(Exception)` with a `reason: Literal["no_claim_event", "no_commit", "ambiguous", "git_unavailable"]` attribute and the WP id. Do not reuse `ActionContextError`: repository-root callers must not see an `OWNED_*` code.
     - Define `def claim_commit_for_wp(mission_dir: Path, wp_id: str) -> str`:
       1. Read events with a lazy `from specify_cli.status import read_events`. Pick the **last** event with `wp_id == wp_id` and `to_lane == Lane.CLAIMED` (compare on the enum or its value, the way `status.models.StatusEvent` stores it). If there is none, raise `no_claim_event`.
       2. Run `git log --format=%H -S<event_id> HEAD -- status.events.jsonl` with `cwd=mission_dir`, `capture_output=True`, `text=True`, `check=False`, and a module-level timeout constant.
       3. Handle the result: `TimeoutExpired` or a non-zero exit raises `git_unavailable`; zero lines raises `no_commit`; two or more lines raise `ambiguous`; exactly one line returns that SHA.
     - Add a docstring that states the definition from the spec ("never identified by matching commit subjects") and the squash risk (see Risks).
  2. Out-of-map (declared): re-export both names from `mission_runtime/__init__.py` and `__all__`, and add them to `_PUBLIC_SURFACE` in `tests/architectural/test_mission_runtime_surface.py`.
  3. Create `tests/mission_runtime/test_claim_commit.py` (`unit` for pure-parsing cases, `git_repo` for real-git cases). Cover:
     - a single claim → its SHA;
     - claim → rejected → re-claimed → the **last** claim's SHA;
     - no claim event → `no_claim_event`;
     - a claim event whose commit is not on `HEAD` (the event exists only in the working tree, uncommitted) → `no_commit`;
     - two commits introducing the same id → `ambiguous`;
     - git failure (`mission_dir` is not a repository) → `git_unavailable`.
  4. **Uniqueness pin.** This is the "no later event cites a claim `event_id`" assumption behind `-S`. Drive a full real lifecycle with `move-task` through CliRunner: claimed → in_progress → for_review → planned (reject) → claimed → in_progress → for_review → approved. For **every** claim `event_id`, assert that `git log -S<id> HEAD -- status.events.jsonl` returns exactly one commit. Also assert that no later event in the log contains that id as a substring (`review_ref`, `evidence`, `reason`). If a future schema starts citing event ids, this test goes red deliberately.
- **Files**: `src/mission_runtime/claim_commit.py` (new), `tests/mission_runtime/test_claim_commit.py` (new), plus the declared `mission_runtime/__init__.py` and `test_mission_runtime_surface.py` edits.
- **Validation checklist**:
  - [ ] `.venv/bin/python -m pytest -q tests/architectural/test_mission_runtime_surface.py tests/architectural/test_layer_rules.py` is green, and the ledger is not grown.
  - [ ] **Non-vacuity**: change the helper to return the *first* match instead of refusing on ≥ 2, and confirm the `ambiguous` test goes red. Record this in the Activity Log.
  - [ ] Coverage of the new module is ≥ 90% (diff-cover gate).
- **Edge cases**:
  - Repositories with `log.showSignature` or a custom `format.pretty` config: always pass `--format=%H` explicitly, and use `--no-show-signature` if your git supports it.
  - An event id is a ULID, which is safe as a `-S` literal (no regex; `-S` is a literal match unless `--pickaxe-regex` is given).

### Subtask T066 – Campsite: `_state_to_action`, `_build_wp_prompt`

- **Purpose**: Standing Order 2. Both functions are at complexity 14, and both change signature in T067/T068.
- **Steps**:
  1. `_state_to_action` (`decision.py:394-493`). Extract:
     - `_implement_state_action(mission_slug, feature_dir, repo_root)` for the `state == "implement"` block (`:408-435`);
     - `_review_state_action(...)` for the review block (`:440-444`);
     - `_template_state_action(state, repo_root, mission_name)` for the generic template + `_ALIASES` resolution (`:451-493`).

     The outer function becomes a 4-way dispatch.
  2. `_build_wp_prompt` (`prompt_builder.py:152-302`). Extract:
     - `_workspace_header_lines(...)`;
     - `_isolation_rule_lines(wp_id, action)`;
     - `_review_command_lines(...)`, the whole `if action == "review":` block (`:222-275`) that T068/T069 rewrite;
     - `_completion_lines(action, wp_id, mission_slug, subtask_ids)`.
  3. Commit: `refactor(next): campsite extractions in decision/prompt_builder (C901)`. There is no behaviour change, and the existing tests stay green unmodified.
- **Files**: `src/runtime/next/decision.py`, `src/runtime/next/prompt_builder.py`.
- **Validation checklist**:
  - [ ] `.venv/bin/ruff check --isolated --select C901 --config 'lint.mccabe.max-complexity=11' src/runtime/next/decision.py src/runtime/next/prompt_builder.py` no longer reports these two functions.
  - [ ] `tests/next/` is green and unchanged. Grep `tests/` for monkeypatch strings `decision._state_to_action` / `prompt_builder._build_wp_prompt` and keep both names.
- **Edge cases**: `_state_to_action` is imported by name in `runtime_bridge.py` and in the carried #5009 test. Keep the name and positional order.

### Subtask T067 – `decision.py` takes the fact

- **Purpose**: FR-008/FR-009. The pre-finalize fallback and prompt building in `decision.py` must stop folding to the repository root checkout (#5009 e6923bc97 is re-expressed here for `decision.py`).
- **Steps**:
  1. `_state_to_action(state, mission_slug, feature_dir, repo_root, mission_name, *, owned: OwnedCheckout | None = None)`. Each `resolve_workspace_for_wp(repo_root, mission_slug, wp)` (`:425`, `:434`, `:443`) becomes `resolve_workspace_for_wp(repo_root, mission_slug, wp, owned=owned)`. `resolve_command(f"{state}.md", …)` and the alias variant use `owned.owned_root if owned else repo_root`, because a command template is a P-local governance read (e6923bc97).
  2. `_build_prompt_safe` (`:496`) and `_build_prompt_or_error` (`:524`) take `owned` and forward it to `build_prompt`. The composed-action probe `resolve_mission_type_context(repo_root, …)` (`:559`) uses the owned root.
  3. `decide_next` (WP11 converted its signature): remove any remaining `effective_root` compatibility if it has no caller left (`grep -rn "decide_next(" src/ tests/`).
  4. `_with_guard_failure_paths(decision, repo_root)` (`:344`): if guard paths must name owned artifacts, pass the owned root. Otherwise leave it unchanged and say why in the Activity Log.
  5. Switch the bridging call sites in `runtime_bridge.py` (WP11-owned, declared edit) from `repo_root=owned.owned_root if owned else repo_root` to `repo_root=repo_root, owned=ctx.owned` or `owned=owned`. The sites are `:1913`, `:1923`, `:1978`, `:1986`, `:3200`, `:3264`, `:3388`, `:3436` and `:3446` (line numbers are from the planning base and shift after WP11).
  6. Add a direct unit pin, the one that WP11 deferred from carried edaa9cd83: `_state_to_action("implement", SLUG, P/kitty-specs/SLUG, repository_root, "software-dev", owned=fact)` returns `("implement", "WP01", str(fact.owned_root))`.
- **Files**: `src/runtime/next/decision.py`, plus the declared `src/runtime/next/runtime_bridge.py` call-site edits.
- **Validation checklist**:
  - [ ] `grep -n "effective_root" src/runtime/next/decision.py` returns nothing.
  - [ ] `grep -n "bridging: WP12" src/runtime/next/` returns nothing.
  - [ ] The non-owned behaviour of `tests/next/` (lane and coordination missions) is unchanged.
- **Edge cases**: the in_review / for_review arbitration rules (FR-012a comments) must survive the extraction unchanged.

### Subtask T068 – `prompt_builder` takes the fact; scoped review diff; fail closed

- **Purpose**: FR-008 (the prompt half), FR-009 and FR-010. `_build_wp_prompt` calls `mission_context_for(repo_root, mission_slug)` (`prompt_builder.py:162-164`) and `resolve_workspace_for_wp(repo_root, …)` (`:167`), and it reads governance from `repo_root` (`:193-194`). For an owned mission, all of these read the repository root checkout.
- **Steps**:
  1. `build_prompt(action, feature_dir, mission_slug, wp_id, agent, repo_root, mission_type, *, owned: OwnedCheckout | None = None)` (`:61`). Its tasks:
     - forward `owned` to `_build_wp_prompt`;
     - when owned, give `_build_template_prompt` the owned root;
     - write the temp prompt through `_write_to_temp(... repo_root=…)`. It is keyed by root, so use the owned root to keep R untouched (NFR-001).
  2. `_build_wp_prompt(..., owned=None)` does the following:
     - Calls `mission_context_for(repo_root, mission_slug, owned=owned)` and `resolve_workspace_for_wp(repo_root, mission_slug, wp_id, owned=owned)`.
     - Reads governance and mission-type lines from the owned root. This re-expresses the e6923bc97 prompt half with credit.
     - When `workspace.resolution_kind == "owned_checkout"`, writes the workspace contract line as "owned checkout" (not "repository root planning workspace").
     - When owned, emits completion commands with `--owned-checkout <P>` (`move-task`, `mark-status`), because that is the supported owned path (US3 and the quickstart).
  3. The review block (`_review_command_lines`, T066) handles three cases:
     - **Lane workspace** (`workspace.lane_id` set, not owned): unchanged. The base is `workspace.context.base_branch`, or `get_feature_target_branch`.
     - **Owned** (`resolution_kind == "owned_checkout"`):
       - `owned_files` empty → raise a typed `OwnedReviewBaseUnavailable` (in `prompt_builder.py`, or reuse `ActionContextError("OWNED_REVIEW_BASE_UNAVAILABLE", …)` from the `mission_runtime` root).
       - Otherwise call `claim_commit_for_wp(owned.mission_dir, wp_id)`. On `ClaimCommitUnresolved`, raise the same typed error.
       - Otherwise emit `git log {claim}..HEAD --oneline -- <pathspecs>` and `git diff {claim}..HEAD --stat -- <pathspecs>`, keeping the existing `:(exclude)` handling for mission-dir pathspecs (`:229-239`).
     - **Repository root** (`resolution_kind == "repo_root"`): same algorithm with `claim_commit_for_wp(feature_dir-or-status-dir, wp_id)`. On failure, keep today's fail-closed text line "unavailable: …" (the non-owned contract), never an unscoped diff.
  4. Map `OWNED_REVIEW_BASE_UNAVAILABLE` to a `blocked` decision. `_build_prompt_or_error` (`decision.py`) catches the typed error and returns `(None, message)`, while carrying the code. The decision then goes through `step_or_blocked` (`runtime_bridge_cores.py:901`) as `blocked`. Make sure `next`'s JSON carries `error_code: "OWNED_REVIEW_BASE_UNAVAILABLE"` through the same typed attribute that WP11 used for `OWNED_COORDINATION_WORKSPACE_UNAVAILABLE`. Do not parse the reason string.
  5. Remove WP11's strict-xfail marker on the full US3-AS1 row in `tests/integration/test_owned_lifecycle_acceptance_next.py` (declared out-of-map edit).
- **Files**: `src/runtime/next/prompt_builder.py`, `src/runtime/next/decision.py`, plus the declared test-marker edit.
- **Validation checklist**:
  - [ ] T064's AS4 and AS5 rows are green, and US3-AS1 is green, with the xfail removed and `strict` flipping it red if it ever regresses.
  - [ ] `grep -n "effective_root" src/runtime/next/prompt_builder.py` returns nothing.
  - [ ] The prompt temp file is not written under R (`r_snapshot` shows 0 differences).
- **Edge cases**:
  - A WP whose `owned_files` are all under `kitty-specs/<slug>/` (a planning-artifact WP) still excludes `tasks/**`, `tasks.md` and the status files from the pathspec (`:231-238`).
  - If there is a stale copy of M in R, P still wins: the prompt must not name R's `target_branch` (the carried test checks that `"wrong-primary-base" not in prompt`).

### Subtask T069 – Replace and delete the three commit-subject matchers

- **Purpose**: FR-025 / R-16. The matchers are unified onto `claim_commit_for_wp` for every review path, and the carried 1be5352ee review assertion is rewritten to the claim-commit contract.
- **Steps**:
  1. `workflow_executor.review_context_for_repo_root_workspace` (`workflow_executor.py:2236-2279`). Replace the `git log --format=%H%x00%s` scan (`:2244-2261`) with `claim_commit_for_wp(feature_dir, wp_id)`.
     - On `ClaimCommitUnresolved`, return `ctx` unchanged. That is today's fail-closed "no base" behaviour; do not fabricate a base.
     - Keep the `rev-list --count` and `ctx` population (`:2264-2279`).
     - This file is **formatter-excluded** (`pyproject.toml` `[tool.ruff.format] exclude`), so do not reformat it wholesale.
  2. `core/worktree_topology._planning_claim_commit` (`worktree_topology.py:103-128`). Its caller is `worktree_topology.py:206`. Delete the matcher and call `claim_commit_for_wp(<status read dir>, wp_id)`, returning `None` on `ClaimCommitUnresolved`. Use the seam's `MissionArtifactKind.STATUS_STATE` read dir, because repository-root planning WPs' claim events live in that mission's status log. Keep the `str | None` contract for `base_branch`.
  3. `prompt_builder.py:241-257`: this was already replaced in T068. Confirm that no subject scan remains.
  4. **Rewrite the carried test** (declared out-of-map edit in WP11's `tests/integration/test_explicit_checkout_commands.py`). `test_owned_review_prompt_uses_owned_target_branch` currently asserts `f"git diff {TARGET}..HEAD --stat" in prompt`. Rewrite it so that:
     - the fixture claims WP01 with `move-task --to claimed --owned-checkout owned` (event-only commit, R-05);
     - it commits an `app.py` change;
     - it calls `build_prompt("review", …, repository_root, "software-dev", owned=fact)`;
     - it asserts `f"git diff {claim}..HEAD --stat -- app.py" in prompt`, where `claim` comes from the `-S<event_id>` algorithm (b1a1693bf's shape, re-expressed);
     - it keeps `"wrong-primary-base" not in prompt`, with the variable renamed in canonical terms;
     - it removes the strict xfail.

     Commit with a `Co-authored-by: Samuel Goff <samuel@defpix.com>` trailer. That address is the author email on every #5009 commit, while tasks.md names a noreply address; use the commit email.
  5. Final guard: `grep -rn "claimed for implementation\|to in_progress\" in subject\|Start {wp_id} implementation" src/` returns nothing.
- **Files**: `src/specify_cli/cli/commands/agent/workflow_executor.py`, `src/specify_cli/core/worktree_topology.py`, `src/runtime/next/prompt_builder.py`, plus the declared test edit.
- **Validation checklist**:
  - [ ] The T064 repository-root review-context row (step 5) is green.
  - [ ] The `worktree_topology` tests are green: `grep -rl "worktree_topology\|materialize_worktree_topology" tests/ --include="*.py"`, then run the hits.
  - [ ] The carried test is green without xfail, and its commit carries the trailer.
- **Edge cases**:
  - **Coordination-topology missions with repository-root planning WPs.** Their claim events are committed on the coordination branch, which is not on the repository root checkout's `HEAD`. The helper returns `no_commit`, and the path fails closed (`None` / "unavailable"). This is correct: no fabricated base. Pin it with one test.
  - `workflow.py:1676` (WP09-owned) calls `review_context_for_repo_root_workspace`. The signature is unchanged, so no WP09 edit is needed.

## Test Strategy

Red-first order:
1. T064 reds, committed before any `src/` edit.
2. T065 helper, with its own unit tests written first.
3. T066 campsite (green).
4. T067 → T069, which turn the reds green.

Record the exact commands and pass counts under *Tests run*:

```bash
PWHEADLESS=1 .venv/bin/python -m pytest -q \
  tests/integration/test_owned_lifecycle_acceptance_review.py \
  tests/mission_runtime/test_claim_commit.py \
  tests/integration/test_owned_lifecycle_acceptance_next.py \
  tests/integration/test_explicit_checkout_commands.py
PWHEADLESS=1 .venv/bin/python -m pytest -q tests/next/ tests/mission_runtime/     # owning subsystems
grep -rl "prompt_builder\|runtime.next.decision\|workflow_executor\|worktree_topology" tests/ --include="*.py"   # run each hit not already covered (skip e2e/slow)
make test-fast
.venv/bin/ruff check src/runtime/next/decision.py src/runtime/next/prompt_builder.py src/specify_cli/cli/commands/agent/workflow_executor.py src/specify_cli/core/worktree_topology.py src/mission_runtime/claim_commit.py src/mission_runtime/__init__.py tests/mission_runtime/test_claim_commit.py tests/integration/test_owned_lifecycle_acceptance_review.py
.venv/bin/ruff format --check src/runtime/next/decision.py src/runtime/next/prompt_builder.py src/specify_cli/core/worktree_topology.py src/mission_runtime/claim_commit.py src/mission_runtime/__init__.py tests/mission_runtime/test_claim_commit.py tests/integration/test_owned_lifecycle_acceptance_review.py
.venv/bin/mypy src/runtime/next/decision.py src/runtime/next/prompt_builder.py src/specify_cli/cli/commands/agent/workflow_executor.py src/specify_cli/core/worktree_topology.py src/mission_runtime/claim_commit.py src/mission_runtime/__init__.py
.venv/bin/python -m pytest -q tests/architectural/test_mission_runtime_surface.py tests/architectural/test_layer_rules.py tests/architectural/test_no_legacy_terminology.py
```

- `workflow_executor.py` is formatter-excluded, so it is not in the format check.
- Do **not** run `make test-full` or the bare `tests/architectural/` directory.

## Risks & Mitigations

- **Squash-rebasing P's branch** folds the claim commit into an implementation commit. `-S` then returns that combined commit, and the diff `claim..HEAD` silently omits the work inside it. Document this in the helper docstring and in the PR, as the plan's IC-06 does. The helper cannot detect it. Rewriting history before review is out of contract.
- **A future event schema that cites event ids** would make `-S` ambiguous. The helper then fails closed with `ambiguous`, and the T065 uniqueness pin goes red first, deliberately.
- **Ownership split with WP11.** The runtime files are WP11's. Every edit to them here is mechanical (bridging → `owned=`) and declared. If WP11's final code has no `# bridging: WP12` markers, grep for `owned.owned_root if owned else` in `runtime_bridge.py` to find the sites.
- **Out-of-map surface edits.** `mission_runtime/__init__.py` and `test_mission_runtime_surface.py` are WP01's files. Keep each edit to the added names only.

## Review Guidance

- Check that the three matchers are gone (grep in "Done when"), and that every review path calls `claim_commit_for_wp`.
- Check that there is no unscoped diff on any owned failure path (AS5 cases a, b and c).
- Check that `claim_commit.py` has no module-level `specify_cli` import, uses an explicit timeout, and raises a typed error with a reason.
- Check the declared out-of-map edits (`__init__.py`, `_PUBLIC_SURFACE`, `runtime_bridge.py` call sites, the two WP11 test files) and their one-line rationales in the PR.
- Check that the carried-test rewrite has the `Co-authored-by` trailer.
- Check that mypy ran on every touched source file.

## Activity Log

> Append entries at the END, in chronological order: `- YYYY-MM-DDTHH:MM:SSZ – <agent_id> – <action>`.

- 2026-09-28T15:00:00Z – system – Prompt created.

---

### Updating Status

Use `spec-kitty agent tasks move-task WP12 --to <status> --mission owned-checkout-lifecycle-authority-01M3M2ZB` and `spec-kitty agent tasks mark-status T064 … --status done`.
