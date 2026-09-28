# Data Model: Owned-checkout lifecycle authority

## OwnedCheckout (value object, `mission_runtime.owned_checkout`)

| Field | Type | Meaning |
|---|---|---|
| `repository_root` | `Path` (resolved) | The repository root checkout R. Used only for repository-root-only concerns (charter authoring, stale-copy detection). |
| `owned_root` | `Path` (resolved, canonical) | The owned checkout P. |
| `mission_dir` | `Path` | `P/kitty-specs/<mission_slug>`. |
| `mission_slug` | `str` | The resolved mission slug. |
| `topology` | `MissionTopology` | The mission's stored topology, as recorded at validation. |
| `target_branch` | `str` | The mission's target branch; P's current branch at validation time. |

**Invariants** (checked in `__post_init__`; pure path logic, no I/O):
- `owned_root != repository_root`.
- `mission_dir` is inside `owned_root / "kitty-specs"`.
- `mission_dir.name == mission_slug`.
- The object was created through `_mint` (private sentinel). Direct construction raises `TypeError`.

**Behaviour**: `files(paths) -> list[Path]` checks that paths are contained in `mission_dir` and raises `OwnedCheckoutPathRefused` on escape. It replaces `OwnedMission.files` and maps to `OWNED_MISSION_PATH_REFUSED` at the CLI edge.

**Lifecycle**:
1. A CLI entry point calls `resolve_owned_or_adopt`.
2. That calls `owned_mission.resolve_owned_mission` or `adopt_owned_checkout`, which runs the claim primitive, the containment, topology, branch and protection checks, and then `OwnedCheckout._mint`.
3. The fact is passed down, immutable, for the whole command.
4. It is never persisted.

## Validator (`specify_cli.core.owned_mission`, the sole minter)

| Function | Returns | Used by |
|---|---|---|
| `resolve_owned_mission(repository_root, checkout, handle, *, target_override=None, allowed_topologies=LIFECYCLE_OWNED_TOPOLOGIES)` | `OwnedCheckout` or raises `ActionContextError(code)` | every `--owned-checkout` command |
| `resolve_owned_create_root(repository_root, checkout)` | `Path` (a validated owned root before the mission exists) | `agent mission create` |
| `adopt_owned_checkout(repository_root, cwd, handle, *, allowed_topologies)` | `OwnedCheckout \| None` | flagless owned-capable commands (FR-021) |
| `LIFECYCLE_OWNED_TOPOLOGIES` | `frozenset({SINGLE_BRANCH})` | ADR 2026-09-03-1 lifecycle commands |
| `NEXT_OWNED_TOPOLOGIES` | `frozenset({SINGLE_BRANCH, LANES_WITH_COORD, COORD})` | `next` |

## ResolvedWorkspace (extended)

The new `resolution_kind` value is `"owned_checkout"`:
- `worktree_path = owned.owned_root`
- `lane_id = None`
- `lane_wp_ids = []`
- `branch_name = owned.target_branch`

`enforce_checkout_identity` gains an arm that refuses when the invoking cwd is outside `worktree_path`.

## Error code registry (NFR-004)

| Code | Status | Raised when |
|---|---|---|
| `OWNED_NO_OPT_IN` | existing | no opt-in claim |
| `OWNED_MISSION_PATH_REFUSED` | existing | the mission dir or a path escapes P |
| `OWNED_TOPOLOGY_UNSUPPORTED` | existing (semantics now per command) | the stored topology is outside the command's allowed set |
| `OWNED_BRANCH_REFUSED` | existing | branch mismatch, detached HEAD, or a protected target |
| `OWNED_INDEX_REFUSED` | existing | staged changes before a write operation |
| `OWNED_CHECKOUT_IS_REPOSITORY_ROOT` | **new** | `--owned-checkout` names the repository root checkout |
| `OWNED_ACTION_UNSUPPORTED` | **new** | `agent action implement` / `agent action review` with `--owned-checkout` |
| `OWNED_REVIEW_BASE_UNAVAILABLE` | **new** | no single claim commit, or no owned files |
| `OWNED_COORDINATION_WORKSPACE_UNAVAILABLE` | **new** | the coordination-workspace probe fails for owned `next` |
| `WORK_PACKAGE_UNRESOLVED` | existing (now scoped to P for owned) | the WP is absent from the owned mission dir |

## Payload field (FR-007)

`stale_repository_root_copy`: `{"path": str, "mission_id": str} | null`. It is a top-level key in the `--json` output of `agent tasks status`, `agent mission setup-plan`, `agent context resolve`, `next` and `agent mission finalize-tasks`. The human text goes to stderr, and to `warnings[]` where that list exists.

## Status event fact used by the review base (unchanged schema)

The last `WPStatusChanged` event with `to_lane == "claimed"` for the WP in P's `status.events.jsonl` supplies the `event_id`. The claim commit is the unique commit on P's `HEAD` history whose diff to `status.events.jsonl` introduced that `event_id`.

## State transitions

No lane-state-machine change. For owned WPs, the `claimed → in_progress → for_review` transitions are driven by `move-task --owned-checkout`, and the prompts come from `next --owned-checkout`.
