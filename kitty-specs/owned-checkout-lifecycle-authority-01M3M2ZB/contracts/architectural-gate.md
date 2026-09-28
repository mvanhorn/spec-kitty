# Contract: single-authority architectural gate

File: `tests/architectural/test_owned_checkout_single_authority.py`. It is AST-based and reuses `tests/architectural/_ast_scan.parse_file`. Comments and docstrings never count as offenders.

| Gate | Rule | Allowlist | Floor on the planning base (non-vacuity) |
|---|---|---|---|
| G1 | `resolve_ownership_claim` is referenced only in `specify_cli/core/owned_mission.py` (plus its definition in `checkout_ownership.py`). Every reference form counts: `ast.alias.name` (including `import … as x`), `Attribute.attr`, `Name.id`, and string `Constant`s used with `getattr`/`importlib`. | empty | ≥ 1 offender: `next_cmd.py:170`, `mission_creation.py:848` |
| G2 | `resolve_owned_mission(` / `adopt_owned_checkout(` are called only from `specify_cli/cli/commands/_owned_checkout.py` and `owned_mission.py` | empty | ≥ 1 offender: `status_transition.py:917`, plus every CLI command calling it directly today |
| G3 | `OwnedCheckout._mint` is referenced only in `owned_mission.py` | empty | the symbol does not exist on the planning base; covered by self-mutation |
| G4 | The identifier `effective_root` does not appear anywhere in `src/` outside `src/charter`: not as a parameter, field, local annotation, keyword argument, TypedDict key or string dict key (`"effective_root"`). Annotation spelling is irrelevant, because the rule bans the identifier, not the annotation type. | empty | ≥ 1 offender (≈ 400 sites) |
| G5 | No parameter or field carries an owned root as a path under any name: `owned_root`, `owned_checkout`, `checkout_root` or `effective_root` annotated `Path`, `pathlib.Path`, `Path \| None`, `Optional[Path]`, `Union[Path, None]`, `os.PathLike` or the string form of any of these. The **only exemption** is the named rule for the fields of `mission_runtime.owned_checkout.OwnedCheckout` (by fully-qualified class name), which is itself pinned by a self-mutation test. | empty | ≥ 1 offender (e.g. `tasks_mark_status.py:114`, `tasks_move_task.py:248`, `mission_creation.py:127`) |
| G6 | Positive signature pin: each consumer in `contracts/owned-checkout-carrier.md` §7 declares a parameter or field `owned: OwnedCheckout \| None`. | n/a | red on the planning base (no consumer has it) |

`src/charter` uses `effective_root` for the org-pack root, a different concept. It is excluded by a module-scoped rule with a written reason. The attribute reads of that sense in `specify_cli/doctrine/*`, `_doctrine_collect.py`, `org_layer.py` and `analysis_inputs.py` are org-pack reads and are documented as out of scope for G4 by an explicit symbol rule, `PackContext.effective_root`, not by path.

**Self-mutation tests.** Synthetic sources must each be flagged:
- a stray claim call;
- `import resolve_ownership_claim as r`;
- `mod.resolve_ownership_claim` attribute access;
- `effective_root: "Path | None"`;
- `Optional[Path]`;
- a TypedDict key `effective_root`;
- a `**{"effective_root": x}` splat;
- a renamed `checkout_root: Path | None` parameter;
- a direct `OwnedCheckout._mint` reference outside the minter;
- a second class with an `owned_root: Path` field.

A clean synthetic source passes.

**Scope boundary.** G2 enforces *where* validation may be called. It does not enforce "once per command"; NFR-002's CLI-level count carries that.

**Commit discipline:**
- The self-mutation and floor tests land green in the foundation WP.
- The G1–G6 assertions are committed red as the first commit of the closing WP and go green there.
- `_baselines.yaml` gains the cap `owned_root_bare_path_params: 0`.
