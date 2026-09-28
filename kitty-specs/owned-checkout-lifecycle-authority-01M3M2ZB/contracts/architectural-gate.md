# Contract: single-authority architectural gate

File: `tests/architectural/test_owned_checkout_single_authority.py`. It reuses `tests/architectural/_ast_scan.parse_file`.

| Gate | Rule | Allowlist | Floor on the planning base (non-vacuity) |
|---|---|---|---|
| G1 | `resolve_ownership_claim` is referenced only in `specify_cli/core/owned_mission.py` (plus its definition) | empty | ≥ 1 offender (`next_cmd.py`, `mission_creation.py`) |
| G2 | `resolve_owned_mission(` / `adopt_owned_checkout(` are called only from `specify_cli/cli/commands/**` and `owned_mission.py` | empty | ≥ 1 offender (`status_transition.py`, `accept.py`) |
| G3 | `OwnedCheckout._mint` is referenced only in `owned_mission.py` | empty | the symbol does not exist on the planning base, so G3 is paired with a self-mutation test |
| G4 | no param or field named `effective_root`/`owned_root` annotated `Path`/`Path \| None`/`Optional[Path]` in `src/runtime`, `src/mission_runtime` | empty | ≥ 1 offender (24 hits) |
| G5 | same as G4 across `src/specify_cli`, `src/kernel`, `src/glossary` | empty; `src/charter` excluded by module rule with a reason (org-pack sense) | ≥ 1 offender (68 hits) |

**Self-mutation tests**: synthetic sources with a stray claim call, a bare `effective_root: Path | None` param and a direct `_mint` reference must each be flagged. A clean synthetic source must pass.

**Commit discipline**: the gate is committed red first, before any conversion, and goes green only when IC-10 completes.
