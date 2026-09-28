# Implementation Plan: Owned-checkout lifecycle authority

**Branch**: `claude/sleepy-hamilton-5lelee` (planning/base = merge target) | **Date**: 2026-09-28 | **Spec**: [spec.md](spec.md)
**Input**: Mission specification from `kitty-specs/owned-checkout-lifecycle-authority-01M3M2ZB/spec.md`

The planning questions were answered through Decision Moments: specify decisions `01M3M2ZX…`, `01M3M300…`, `01M3M4GM…`, `01M3M4GT…` and `01M3M4GZ…`; plan decisions `01M3M65D…` and `01M3M65K…`. Evidence is in [research.md](research.md) and [research/pre-spec-options-memo.md](research/pre-spec-options-memo.md).

## Summary

An owned mission's lifecycle reads fall back to the repository root checkout. Three things cause this:
- a file-placement seam with no owned arm;
- three competing ownership authorities;
- 92 bare `effective_root: Path` parameters that every reader must remember to thread.

**Approach (operator-locked):**
- Introduce one validated ownership fact, `mission_runtime.OwnedCheckout`. It is minted only by `specify_cli.core.owned_mission`.
- Every owned read consumes the fact: the placement seam, WP and workspace resolution, the status surface, the `next` runtime, prompts, the review base, and CLI entry points.
- Convert all 92 bare-path parameters in this mission.
- Delete `OwnedMission` and `effective_root_kwargs`, with no alias.
- Enforce the result with a non-vacuous architectural gate that has an **empty allowlist**. Operator rationale: "half-implemented work and ratchets have been hurting us for weeks. I'd rather spend the effort now."

The rename `OwnedMission` → `OwnedCheckout` makes this a **bulk-edit mission** (`change_mode: bulk_edit`). The classification workflow was run, and [occurrence_map.yaml](occurrence_map.yaml) governs which occurrences change.

## Technical Context

**Language/Version**: Python 3.11+ (repository floor; CI also runs the nightly interpreter matrix)
**Primary Dependencies**: typer and rich (CLI), ruamel.yaml (frontmatter), git CLI via subprocess, `spec_kitty_events` (status events; consumed, unchanged). No new dependencies, so no supply-chain review is needed (directive 051: no dependency decision).
**Storage**: Files in git. Mission artifacts are under `kitty-specs/<mission>/`; `status.events.jsonl` is the append-only status authority; `lanes.json`, `meta.json` and `.kittify/config.yaml` (charter activation).
**Testing**: pytest. The layers are:
- ATDD red-first acceptance tests through the real CLI (`typer.testing.CliRunner` / subprocess) on a real repository root checkout plus a linked owned worktree fixture;
- focused unit tests per seam;
- architectural gates in `tests/architectural/` (AST scans reusing `_ast_scan`, self-mutation non-vacuity tests).

Local runs are targeted files plus `make test-fast`; the full and architectural sweeps run in CI (`NO_FULL_HEAVY_SUITES_IN_MISSION`).
**Target Platform**: Linux, macOS and Windows 10+ developer machines and CI (`ci-windows.yml` for the case-variant identity test).
**Project Type**: single project (CLI plus libraries in `src/`).
**Performance Goals**:
- 1 ownership validation per command;
- 0 extra git subprocesses per owned status read;
- owned `agent tasks status`, `setup-plan` and `context resolve` take a median under 2 s on the acceptance fixture.
**Constraints**:
- Layer chain `kernel <- charter <- {glossary, runtime, mission_runtime} <- specify_cli` (`tests/architectural/test_layer_rules.py`). The `mission_runtime` outbound ledger (cap 10 in `_baselines.yaml`) must not grow.
- `mission_runtime` public surface gates MR-1/MR-2 (`test_mission_runtime_surface.py`).
- Complexity ≤ 15; ruff, format and mypy clean; diff coverage ≥ 90%.
- No shims (`test_compat_shims` burn-down).
**Scale/Scope**:
- about 92 parameter conversions across about 45 source files, plus about 27 files for the `OwnedMission` rename;
- 5 CLI entry points gain `--owned-checkout`;
- about 10 existing owned-checkout test files are re-pointed.

## Charter Check

*GATE: must pass before Phase 0 research; re-checked after Phase 1 design.*

| Charter rule | Status | How the plan satisfies it |
|---|---|---|
| Single canonical authority | ✅ | One minter module. Gate G1–G3 pins it (claim primitive, validator, `_mint`). Status-source classification stays in `surface_resolver` (C-002); there is no new read-source label. |
| Architectural alignment / layer rules | ✅ | The carrier lives in `mission_runtime` and imports only `pathlib`, `dataclasses` and `mission_runtime.context`, so there are 0 new edges. `runtime → mission_runtime` is already permitted (5 existing importers). |
| DDD + tiered rigour | ✅ | `OwnedCheckout` is a value object with invariants in `__post_init__`. The validator is the aggregate boundary. Core seams (placement, status) get the most tests. |
| ATDD red-first (C-011, C-007) | ✅ | Each IC starts with red acceptance tests through the pre-existing entry point. #5009's red test commits are carried first. #4867 uses deterministic fault injection (a truncated `commondir`). |
| Architectural gate discipline (DIRECTIVE_043) | ✅ | Gates G1–G5 have concrete floors (red on the planning base), self-mutation tests and **empty** allowlists. There is no ledger (operator decision `01M3M65D…`). |
| Canonical sources (DIRECTIVE_044) | ✅ | This plan was produced with the canonical CLI. Templates come from `packs/built-in/missions`. The occurrence map is seeded from the canonical template. |
| Campsite cleaning (Standing Order 2) | ✅ | A tidy-first step precedes functional change on god-surfaces (`runtime_bridge.py`, `workspace/context.py`, `mission_finalize.py`). It is behaviour-preserving extraction to keep complexity ≤ 15 wherever signatures change. |
| Terminology canon | ✅ | The carrier fields use "repository_root" and "owned_root". New codes, messages and docs use "repository root checkout". The terminology guard runs pre-push. |
| No full heavy suites in mission | ✅ | Validation is by targeted files, the specific architectural gate files and `make test-fast` only. |
| User customization preservation | ✅ | Mission-state reads and writes only; no user-owned command or skill files are touched. |
| Pre-existing failure rule | ✅ | Any baseline red found is filed as an issue before it is accepted as baseline. |

No violations, so no Complexity Tracking entries are needed.

## Project Structure

### Documentation (this mission)

```
kitty-specs/owned-checkout-lifecycle-authority-01M3M2ZB/
├── spec.md
├── plan.md                 # this file
├── research.md             # Phase 0 decisions
├── research/pre-spec-options-memo.md
├── data-model.md           # OwnedCheckout, codes, payload fields
├── contracts/
│   ├── owned-checkout-carrier.md
│   ├── cli-owned-checkout-surface.md
│   └── architectural-gate.md
├── quickstart.md           # end-to-end owned walkthrough (SC-001)
├── occurrence_map.yaml     # bulk-edit classification (DIRECTIVE_035)
├── decisions/              # Decision Moments
└── tasks.md                # produced by /spec-kitty.tasks
```

### Source Code (repository root)

```
src/mission_runtime/
├── owned_checkout.py            # NEW: OwnedCheckout value object (private mint)
├── __init__.py                  # re-export OwnedCheckout (public surface + ADR 2026-06-07-1 note)
├── resolution.py                # placement_seam / mission_context_for / resolve_action_context / _resolve_wp_bearing_fields take `owned`
└── checkout_identity.py         # owned_checkout arm in enforce_checkout_identity
src/runtime/next/
├── runtime_bridge.py            # DecideNextContext.owned, _dn_*, board authority, composition, #4867 probe
├── runtime_bridge_engine.py, runtime_bridge_io.py, decision.py, prompt_builder.py, next_invocation_lifecycle.py
src/specify_cli/
├── core/owned_mission.py        # sole minter: resolve_owned_mission(allowed_topologies), resolve_owned_create_root, adopt_owned_checkout; OwnedMission + effective_root_kwargs deleted
├── core/checkout_ownership.py   # repository-root-itself refusal
├── core/mission_creation.py     # governance reads from the owned root
├── workspace/context.py         # owned arm, resolution_kind=owned_checkout, cache keyed on the resolved tasks dir
├── coordination/{status_service,status_transition,surface_resolver,workspace}.py
├── status/models.py             # TransitionRequest.owned
├── task_utils/support.py        # locate_work_package(owned=)
├── cli/commands/_owned_checkout.py   # NEW: OwnedCheckoutOption, resolve_owned_or_adopt, emit_owned_refusal, stale-copy helper
├── cli/commands/{next_cmd,tasks_status_cmd,agent/tasks,agent/context,agent/mission_setup_plan,agent/workflow,agent/mission_finalize,…}.py
├── cli/commands/agent/workflow_executor.py   # claim-commit review base
├── missions/operation_context.py # DELETED (cwd adoption replaced by adopt_owned_checkout)
└── (accept, consolidation, review cycle, commit_router, write_seam …)  # remaining effective_root → owned conversions
tests/
├── architectural/test_owned_checkout_single_authority.py   # NEW gates G1–G5
├── integration/test_owned_lifecycle_acceptance.py          # NEW SC-001 walkthrough + O1–O10 repros
└── (existing owned test files re-pointed; #5009 tests carried)
docs/
├── adr/3.x/2026-09-03-1-…, 2026-08-12-1-…, 2026-06-07-1-… (amendments)
└── context/execution.md         # "owned checkout" glossary entry
```

**Structure Decision**: single project. The new code is confined to one new `mission_runtime` module and one new CLI helper module. Everything else is signature migration and deletion.

## Complexity Tracking

Not needed; there are no Charter Check violations.

## Implementation Concern Map

> Concerns are not work packages. `/spec-kitty.tasks` decomposes them.

### IC-01 — Validated ownership fact and sole minter
- **Purpose**: one value object and one minting module, so ownership is decided once per command.
- **Relevant requirements**: FR-001, FR-002, FR-003, FR-020, FR-023, C-001, C-003, NFR-002, NFR-006.
- **Affected surfaces**:
  - new `src/mission_runtime/owned_checkout.py` and the `mission_runtime/__init__.py` re-export;
  - `src/specify_cli/core/owned_mission.py`: `resolve_owned_mission(..., allowed_topologies)`, `LIFECYCLE_OWNED_TOPOLOGIES` / `NEXT_OWNED_TOPOLOGIES`, `resolve_owned_create_root`, `adopt_owned_checkout`; delete `OwnedMission`, `_EffectiveRootKwargs` and `effective_root_kwargs`;
  - `core/checkout_ownership.py`: refuse the repository root checkout itself with `OWNED_CHECKOUT_IS_REPOSITORY_ROOT`;
  - `test_mission_runtime_surface.py` `_PUBLIC_SURFACE`.
- **Sequencing/depends-on**: none (foundation).
- **Risks**:
  - The rename of about 27 files must follow `occurrence_map.yaml`.
  - The attribute renames `.primary`/`.root` are grep-hostile; mypy strict is the net.
  - Canonical path identity needs one helper (`Path.resolve()` plus `os.path.normcase` on Windows).

### IC-02 — Placement seam and mission-runtime resolution take the fact
- **Purpose**: close the fall-back to the repository root at its source: `placement_seam` / `get_main_repo_root` fold.
- **Relevant requirements**: FR-006, FR-007, FR-009, FR-011.
- **Affected surfaces**:
  - `mission_runtime/resolution.py`: `placement_seam(owned=)`, `PlacementSeam.owned`, `mission_context_for(owned=)`, `resolve_action_context(owned=)`, `_resolve_wp_bearing_fields(owned)`, `_require_owned_single_branch` reading `owned.topology`;
  - `task_utils/support.py::locate_work_package(owned=)`.
- **Sequencing/depends-on**: IC-01.
- **Risks**:
  - The public `mission_runtime` API changes, and callers across `specify_cli` must move in lockstep.
  - Primary-partition kinds must return `owned.mission_dir` without calling `get_main_repo_root`.

### IC-03 — Workspace resolution: owned arm, kind and cache
- **Purpose**: owned WPs resolve to P with a distinct kind and no lane, and caches never mix checkouts.
- **Relevant requirements**: FR-006, FR-008, FR-011, FR-019.
- **Affected surfaces**:
  - `workspace/context.py`: owned first arm, `ResolvedWorkspace(resolution_kind="owned_checkout", lane_id=None, worktree_path=owned.owned_root)`; coordination-topology owned keeps the lane arm through the owned seam; `_FEATURE_WP_METADATA_*` keyed on the resolved read `tasks_dir`; `clear_workspace_resolution_caches` tests;
  - `mission_runtime/checkout_identity.py` owned arm.
- **Sequencing/depends-on**: IC-01, IC-02.
- **Risks**:
  - Consumers that assume `lane_id` ⇒ lane worktree (`prompt_builder`, `workflow`) must handle `owned_checkout`.
  - Changing the cache key changes invalidation semantics.

### IC-04 — Status surfaces consume the fact
- **Purpose**: owned single_branch checkouts under `.worktrees/` read status, and the status transaction stops re-validating.
- **Relevant requirements**: FR-003, FR-013, FR-014, C-002.
- **Affected surfaces**:
  - `coordination/status_service.py`: `EventLogReadContract.owned`; the guard at `:163` delegates to the new `surface_resolver.primary_read_targets_coord_worktree(path, *, owned)`;
  - `coordination/status_transition.py`: `:917` re-validation removed, `_read_contract_from_transaction_target`;
  - `status/models.py`: `TransitionRequest.owned`;
  - `agent_tasks_ports.MissionHandle.owned`;
  - `acceptance/accept.py:342`.
- **Sequencing/depends-on**: IC-01.
- **Risks**: loosening the `.worktrees` shape guard. The same-path registered coordination worktree control (FR-014) must be added before the change. The #5009 test `test_owned_contract_validates_root_and_mission` is dropped because it pins a parallel source that C-002 forbids.

### IC-05 — `next` runtime consumes the fact
- **Purpose**: `next --owned-checkout` validates properly, crosses tasks→implement, uses P's composition policy and task board, and survives the #4867 probe fault.
- **Relevant requirements**: FR-002, FR-007, FR-008, FR-009, FR-012, FR-023.
- **Affected surfaces**:
  - `cli/commands/next_cmd.py`: via `resolve_owned_or_adopt` with `NEXT_OWNED_TOPOLOGIES`; handle-less `next` resolves the handle in P first;
  - `runtime/next/runtime_bridge.py`: `DecideNextContext.owned`, `_dn_bootstrap`, `decide_next_via_runtime`, `query_current_state`, `_build_finalized_override_query_decision`, `_mission_routes_through_coordination`, `_wrap_with_decision_git_log` (git cwd), `_dn_composition_dispatch`, `_resolve_wp_board_action` / `_implement_action` / `_review_action`;
  - `runtime_bridge_engine.py`, `runtime_bridge_io.py:1155` (the presence probe folds into the fact), `decision.py`, `prompt_builder.py`, `next_invocation_lifecycle.py`;
  - `coordination/workspace.py:163`: capture stderr on `git worktree list`, raise a typed `CoordinationWorkspaceUnavailable`, which the runtime maps to a `blocked` decision with `OWNED_COORDINATION_WORKSPACE_UNAVAILABLE`.
- **Sequencing/depends-on**: IC-01, IC-02, IC-03.
- **Risks**:
  - `runtime_bridge.py` is a god-module, so campsite-extract first.
  - O5's wedge: WP resolution must happen before the run state advances.
  - The e2e lanes_with_coord owned test must stay green (FR-022).

### IC-06 — Owned review base
- **Purpose**: a deterministic, subject-free claim commit and a WP-scoped diff, failing closed.
- **Relevant requirements**: FR-010.
- **Affected surfaces**:
  - `runtime/next/prompt_builder.py` (`_build_wp_prompt` review branch);
  - `cli/commands/agent/workflow_executor.py:2236-2362` for the owned path;
  - a new helper that finds the last `to_lane=claimed` event for the WP in P's `status.events.jsonl`, then runs `git -C P log --format=%H -S<event_id> HEAD -- <status.events.jsonl>`. The helper expects exactly 1 match; otherwise `OWNED_REVIEW_BASE_UNAVAILABLE`.
- **Sequencing/depends-on**: IC-05.
- **Risks**: the #5009 test 1be5352ee asserts `{TARGET}..HEAD` and must be rewritten to the claim-commit contract.

### IC-07 — Atomic `finalize-tasks`
- **Purpose**: every refusal happens before the first write.
- **Relevant requirements**: FR-015, FR-013.
- **Affected surfaces**: `cli/commands/agent/mission_finalize.py`:
  - build the full in-memory plan (requirement mapping, ownership manifests `:3489`, stale-canceled `:3501`, lane computation and cycle `:2992`, pin classification, status-surface resolution) by reusing the `--validate-only` path;
  - then apply the writes (`:3381` meta, `:3417` issue-matrix, `:3472` WP frontmatter, `:3478` tasks.md, `:3506` TasksStarted, `:2980–3020` pipeline) in one phase.
- **Sequencing/depends-on**: IC-04 (the status-surface read inside the plan).
- **Risks**:
  - Spec US4-AS3's refusal is amended to a **lane dependency cycle** (WP01→WP02→WP03 where WP01 and WP03 share an owned file) or an ownership-manifest overlap. Dependency-graph cycles are already atomic on main (research R-07).
  - Keep complexity ≤ 15 by extracting plan and apply phases.

### IC-08 — CLI owned surface and validated flagless adoption
- **Purpose**: one option, one resolver, one refusal emitter and one stale-copy reporter for every owned-capable command.
- **Relevant requirements**: FR-004, FR-005, FR-006, FR-007, FR-018, FR-020, FR-021, NFR-004.
- **Affected surfaces**:
  - new `cli/commands/_owned_checkout.py`: `OwnedCheckoutOption`, `resolve_owned_or_adopt(flag, cwd, handle, allowed_topologies)`, `emit_owned_refusal(code, message, json_output)`, `stale_repository_root_copy(fact)`;
  - new flags on `agent tasks status` (`tasks.py:1361` → `tasks_status_cmd._do_status`), `setup-plan` (`mission_setup_plan.py:996`), `context resolve` (`agent/context.py:110`), `agent action implement` (`workflow.py:1336`) and `agent action review` (`workflow.py:1858`, refusal only, `OWNED_ACTION_UNSUPPORTED`);
  - existing owned commands (move-task, mark-status, spec-commit, finalize, check-prerequisites, accept, next) moved onto the shared helper;
  - delete `missions/operation_context.py` and `MissionSurfaceConflictError`;
  - golden contracts `test_tasks_cli_contract.py` and `test_mission_cli_golden_contract.py`.
- **Sequencing/depends-on**: IC-01; per-command wiring after IC-02/03/04.
- **Risks**:
  - Adoption must exclude the repository root checkout, lane worktrees (`lanes.json` / workspace context) and coordination worktrees (`surface_resolver.classify_worktree_topology`).
  - The JSON error shapes differ per command today; the shared emitter must keep each command's existing envelope keys (serialized keys are `do_not_change`).

### IC-09 — Governance reads at owned create
- **Purpose**: owned create reads charter activation, mission-type context and the spec template from P.
- **Relevant requirements**: FR-016, FR-017.
- **Affected surfaces**: `core/mission_creation.py:848` (now `resolve_owned_create_root`), and `:951`, `:963`, `:968` switching from `resolved_root` to the owned root.
- **Sequencing/depends-on**: IC-01.
- **Risks**: the charter authoring guard (#4785) must stay intact. Carry #5009 a37e9ee39 red-first and re-express 1f42f76ea.

### IC-10 — Remaining `effective_root` conversion (all 92)
- **Purpose**: no bare-path owned root anywhere in `src/` (operator decision `01M3M65D…`).
- **Relevant requirements**: FR-001 (G4/G5), FR-003, FR-022.
- **Affected surfaces**: every `effective_root` parameter or field not covered by IC-02..IC-09. That includes acceptance, the consolidation baseline, the review cycle, `commit_router`, `write_seam`, `agent_tasks_ports`, move-task, mark-status, spec-commit, check-prerequisites and `next_invocation_lifecycle`. The census is in research R-03. `src/charter`'s `effective_root` (org-pack root) is a different sense and is excluded.
- **Sequencing/depends-on**: IC-01..IC-09 (last functional concern).
- **Risks**:
  - Largest blast radius. Each converted module's own tests plus FR-022's existing owned test files must pass unchanged.
  - Where a caller only has a flagless repository root, it passes `owned=None`, never a synthesized fact.

### IC-11 — Single-authority architectural gate
- **Purpose**: make the defect class impossible by construction (SC-004).
- **Relevant requirements**: FR-001, SC-004.
- **Affected surfaces**: new `tests/architectural/test_owned_checkout_single_authority.py`:
  - G1: `resolve_ownership_claim` only in `owned_mission.py`;
  - G2: validator and adoption calls only from `cli/commands/**` and `owned_mission.py`;
  - G3: `OwnedCheckout._mint` only in `owned_mission.py`;
  - G4/G5: no `effective_root`/`owned_root` parameter or field typed `Path`/`Path | None` anywhere in `src/` except `src/charter`'s unrelated org-pack sense, which is excluded by an explicit module-scoped rule with a reason. The **allowlist is empty**;
  - self-mutation tests.

  The gate is committed red first, against the planning base.
- **Sequencing/depends-on**: authored red at the start; green after IC-10.
- **Risks**: the gate must not be vacuous. It asserts a concrete floor: ≥ 1 offender on the planning base for each of G1, G2 and G4.

### IC-12 — ADRs and glossary
- **Purpose**: record the design and terminology.
- **Relevant requirements**: FR-024, C-005, C-008.
- **Affected surfaces**:
  - `docs/adr/3.x/2026-09-03-1-explicit-owned-checkout-single-branch-lifecycle.md` (validated fact, per-command topology, `owned_checkout` kind, claim-commit review base, atomic finalize, stale-copy policy);
  - `docs/adr/3.x/2026-08-12-1-checkout-ownership-for-mission-create-and-next.md` (`next` via validator, validated flagless adoption, create root);
  - `docs/adr/3.x/2026-06-07-1-execution-state-canonical-surface.md` (public `OwnedCheckout` note);
  - `docs/context/execution.md` ("owned checkout" entry).

  Run the docs freshness, terminology and docs index tooling.
- **Sequencing/depends-on**: none. Can land with IC-01.
- **Risks**: none significant.

### IC-13 — #5009 provenance
- **Purpose**: carry prior art with authorship.
- **Relevant requirements**: C-006.
- **Affected surfaces**: carry these `cherry-pick -x` onto the lane that owns their concern, keeping author Samuel Goff:
  - a37e9ee39 (IC-09);
  - edaa9cd83 (IC-03/05);
  - 1be5352ee (IC-05/06, then rewrite the review assert);
  - 4ff6ff0c0 (IC-04, drop `test_owned_contract_validates_root_and_mission`);
  - 22f380055, if the engine file is still formatter-excluded.

  6d682dce3 is dropped (pins the forbidden parallel source). The fix commits 1f42f76ea, e6923bc97, b1a1693bf and 77aa62913 are re-expressed with `Co-authored-by: Samuel Goff`. The doc commits ea68d9a47, 5a9b708e3, 75f3339d6 and 965d7fd13 are dropped.
- **Sequencing/depends-on**: each carried test lands red-first at the start of its IC.
- **Risks**: the carried tests use the `repo_root=R, effective_root=P` call shape and are adapted to the fact in the same WP.
