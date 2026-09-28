# Contract: validated ownership fact

**Scope**: `mission_runtime.OwnedCheckout` and its sole minter, `specify_cli.core.owned_mission`.

1. **Sole construction.** The only way to obtain an `OwnedCheckout` in `src/` is through `owned_mission.resolve_owned_mission`, `owned_mission.adopt_owned_checkout`, or a `tests/` helper. `OwnedCheckout(...)` called directly raises `TypeError`. Gates G1–G3 enforce this.
2. **Once per command.** A CLI entry point validates at most once. Every downstream function receives `owned: OwnedCheckout | None` and never re-validates. Gate G2 enforces this, and NFR-002 counts it.
3. **No bare owned roots.** No function parameter or dataclass field in `src/` carries the owned root as `Path` / `Path | None`. Gates G4/G5 enforce this with an empty allowlist. `src/charter`'s org-pack `effective_root` is a different concept and is excluded by module rule.
4. **Topology per command.** Each call site passes its `allowed_topologies`. The fact records the stored topology.
5. **Repository root is not owned.** Passing R as `--owned-checkout` raises `OWNED_CHECKOUT_IS_REPOSITORY_ROOT`.
6. **Adoption guard.** `adopt_owned_checkout` returns `None` in each of these cases, never a fact:
   - the repository root checkout;
   - a lane worktree;
   - a coordination worktree;
   - another repository;
   - any checkout the validator rejects.
7. **Consumers.** The following accept `owned` and, when it is set, read only under `owned.owned_root`, never calling `get_main_repo_root`:
   - `placement_seam`
   - `mission_context_for`
   - `resolve_action_context`
   - `locate_work_package`
   - `resolve_workspace_for_wp`
   - `EventLogReadContract`
   - `TransitionRequest`
   - `DecideNextContext`
   - the prompt builder
   - the review-base helper
