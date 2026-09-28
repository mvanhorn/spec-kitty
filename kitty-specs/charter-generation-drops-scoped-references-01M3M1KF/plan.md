# Implementation Plan: Charter generation drops configured scoped references and breaks parity

**Branch**: `fix/charter-generation-drops-scoped-references-5257` | **Date**: 2026-09-28 | **Spec**: [spec.md](./spec.md)
**Input**: Mission specification from `kitty-specs/charter-generation-drops-scoped-references-01M3M1KF/spec.md`

**Note**: This template is filled in by the `/spec-kitty.plan` command. See `packs/built-in/missions/mission-steps/software-dev/plan/prompt.md` for the execution workflow.

## Summary

`_render_kind_references`/`_build_references_from_service` in `src/charter/activation/compiler.py` look up every DRG-backed activated reference against `_raw_kind_repository`, which strips only the *activation-config* filter — the underlying per-kind repository was already constructed with `active_languages=infer_repo_languages(repo_root)` baked in, so a language/scope-filtered id (present on disk, excluded by scope) misses the lookup exactly like a genuinely nonexistent id. The miss is recorded as an opaque `"Unresolved reference: <kind>/<id>"` diagnostics-list string and the reference silently vanishes from `catalog.references` (#5257 / #5253). This mission makes that loss impossible: a scope-filtered activated id (present on disk, excluded solely by language/scope — the actual #5257/#5253 defect) is uniformly preserved as a reason-bearing placeholder `CharterReference` (never a silent omission, never a second fail-closed code path), while a genuinely-nonexistent id stays on the pre-existing, currently-tested diagnostics-only path (`Contract C4`, issue #4785 — deliberately not superseded, see "WP-CORE reconciliation" below) with its diagnostic now made reason-bearing too; the reason distinguishes never-existed from scope-filtered by reusing the already-built `charter.activation._catalog_miss` classifiers, the `--json` output gains an additive structured `unresolved_references` field so automated callers stop string-matching prose, and a generator-to-parity regression test (extending `tests/charter/test_active_languages_idempotency.py`'s fixture precedent) pins the fix at the real `compile_charter`/`charter generate` entry point. A related pre-existing swallow bug in the preflight auto-refresh path (`references_refresh.py`) — a targeted `generate` subprocess whose outcome is never inspected, masking a genuine references-parity failure behind an unconditional manifest re-stamp — is folded in as a bounded, same-defect-class fix per operator doctrine (silent-success is this mission's own acceptance bar).

## Technical Context

**Language/Version**: Python 3.11+ (uv-managed; `pyproject.toml` requires-python `>=3.11`)
**Primary Dependencies**: `charter` (specifically `charter.activation.compiler`, `charter.activation._catalog_miss`, `charter.offering.base.BaseDoctrineRepository`) and `specify_cli` (specifically `specify_cli.charter_runtime.preflight.references_refresh`, `.runner`, `.result`) — both already-vendored first-party packages in this repo; no new third-party dependency is expected. `ruamel.yaml` (existing dep) continues to back the round-trip `charter.yaml` write path untouched.
**Storage**: N/A — no database; the artifact under change is the YAML file `.kittify/charter/charter.yaml` (`catalog.references` section) that `charter generate` writes, plus the JSON stdout of `charter generate --json`.
**Testing**: `.venv/bin/python -m pytest` (pytest, this repo's standard runner) — targeted module shards for `tests/charter/`, `tests/doctrine/`, and `tests/specify_cli/charter_runtime/` (preflight). No `uv run` (would re-sync the environment) and no full-suite run during mission work per `NO_FULL_HEAVY_SUITES_IN_MISSION`.
**Target Platform**: CLI, cross-platform (Linux/macOS/Windows 10+) — no platform-specific code introduced; the fix is pure Python inside the existing compiler/preflight modules.
**Project Type**: Single Python package (this repo, `spec-kitty` — the CLI/library monorepo, not a web/mobile split).
**Performance Goals**: No new perf goal beyond the charter's blanket "<2s for typical projects" CLI budget; the fix adds a bounded number of extra dict/set lookups per unresolved id (never a new network or subprocess call in the compiler seam) and does not change the auto-refresh path's step count in the success case.
**Constraints**: NFR-001 (idempotent regeneration — byte-identical `catalog.references` across repeated runs), NFR-002 (every diagnostic/placeholder names `kind/id` and a reason category), C-001–C-004 (parity guard untouched, no hardcoded six-id special case, no machine-local absolute paths reintroduced, no local-path/private-ledger citations in mission artifacts).
**Scale/Scope**: Two files change behavior (`src/charter/activation/compiler.py`, `src/specify_cli/charter_runtime/preflight/references_refresh.py` + its one caller in `runner.py`); one new/extended test module; no data migration, no schema change to any persisted file format beyond the additive JSON field described below.

## Charter Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

### Seams this mission touches (both named explicitly — DIRECTIVE_001 architectural alignment)

1. **`src/charter/activation/compiler.py`** (the `charter` package / doctrine-compiler seam) — the core FR-001/FR-002/FR-003/FR-004 fix. `_render_kind_references` and `_build_references_from_service` change so every DRG-backed activated id (directive/tactic/styleguide/toolguide/procedure/agent_profile) that misses the raw-repository lookup is classified via the existing `charter.activation._catalog_miss` primitives: a `SCOPE_FILTERED` miss is preserved as a reason-bearing placeholder `CharterReference` (never silently dropped); a `MISSING_ARTIFACT`/`TYPO_SUSPECTED` miss stays on the pre-existing diagnostics-only path (Contract C4, #4785 — see "WP-CORE reconciliation" below), also made reason-bearing.
2. **`src/specify_cli/charter_runtime/preflight/` (`references_refresh.py` + `runner.py::_attempt_auto_refresh`)** (the `specify_cli` / CLI-orchestration seam) — the auto-refresh fold-in decided below (Edge Cases "Mid-flight missions").

No CLI command reaches past a service into kernel internals here: `references_refresh.py` calls `charter generate` only via `subprocess.run` (already the existing pattern — this mission changes what the CALLER does with the `CompletedProcess`, not the invocation mechanism), and the compiler fix stays entirely inside `charter.activation`/`charter.offering`, never reaching into `specify_cli`. No unguarded core-loop → sync coupling is introduced: the preflight fold-in only tightens an existing synchronous subprocess-result check: it does not add a new sync dependency.

### What is generated, and by which command

`catalog.references` inside `.kittify/charter/charter.yaml` is recompiled ONLY by `spec-kitty charter generate` (`src/specify_cli/cli/commands/charter/generate.py` → `charter.activation.compiler.compile_charter`/`write_compiled_charter`; see `references_refresh.py`'s own module docstring: "Only `spec-kitty charter generate` recompiles that section"). This mission does not hand-patch a generated file — it changes the generator itself (`_render_kind_references`/`_build_references_from_service`) so its OUTPUT is correct on every future run, including the targeted `generate` invocation the preflight auto-refresh boundary fires internally.

### Whether any contract moves

- **`CharterPreflightResult`'s JSON shape** (`contracts/charter-preflight-json.md` in `kitty-specs/charter-ux-and-org-pack-vocabulary-01KSAF14/`, the binding doc `src/specify_cli/charter_runtime/preflight/result.py`'s own docstring cites) does **NOT** change. `blocked_reason: str | None` already exists on `CharterPreflightResult` (`result.py` line 98) and is already populated with `auto_refresh_applied=True` at multiple existing call sites inside `_attempt_auto_refresh` (e.g. lines 686–692, 704–709 in the current `runner.py`) — the fold-in below reuses that exact existing pattern for the new failure branch. No new field, no field-shape change, no change to `to_dict()`/`to_json()`.
- **`charter generate`'s `--json` output** (`src/specify_cli/cli/commands/charter/generate.py`, the `diagnostics: list[str]` field at the JSON-emit block, current lines ~538–559) DOES gain a new field. FR-004 requires the unresolved-reference id+reason to be machine-readable, not only free text a caller has to string-match. **Decision**: add a new top-level key `unresolved_references: list[{"kind": str, "id": str, "cause": str, "detail": str}]` alongside the existing `diagnostics: list[str]` field, which is left completely unchanged in shape (still a flat list of human-readable strings — its *content* per-line gains a reason suffix, per FR-002, but its type and key name are untouched). `cause` is the `CatalogMissCause` enum value (`"missing_artifact"`, `"typo_suspected"`, or `"scope_filtered"`); `detail` is the human-readable classifier suggestion text `classify_catalog_miss`/`classify_scope_filtered_miss` already produce. **Rationale for additive-alongside rather than widening `diagnostics` entries into objects**: any existing consumer that treats `diagnostics` as `list[str]` (CI scripts, readiness probes, `_finalize_sync_result`'s own sync-warnings concatenation which appends plain strings into the same list) keeps working unchanged — widening the existing field's element type to `str | dict` would be a breaking shape change for every such consumer with no compile-time signal in a dynamically-typed JSON contract. Adding a new, optional-in-practice-but-always-present key is additive and detectable (`"unresolved_references" in payload`) without touching the existing key at all. This is documented as a **versioned/additive JSON contract change**, not a silent one, in the new `contracts/charter-generate-json-diagnostics.md` (see Project Structure below).

### Gate set (authoritative — 2026-09-23 re-verification against `.github/workflows/` on `main`; supersedes any stale overlay list)

**Enforced on every PR**:
- `ruff check .` AND `ruff format --check .` (both, always-on)
- `uv lock --check`
- banned-API/import-boundary (ruff `TID251`, `import-linter`)
- `spec-kitty regen --check` (generated-artifact freshness) — **not applicable to this mission's diff**: no schema-generated file changes; stated explicitly rather than silently assumed passing.
- terminology guard (`tests/architectural/test_no_legacy_terminology.py`)
- layer rules / pyproject shape
- archive-freeze
- routed test groups, path-filtered by changed paths
- per-module test shards (`ci-modules.yml`, diff-scoped to touched module roots — `src/charter/` and `src/specify_cli/` once implementation lands)
- **diff-cover ≥90% of changed lines** (`ci-aggregate.yml` `diff-cover` job) — the REAL per-PR coverage gate, not a whole-module floor
- wheel build + `clean-install-verification`

**NOT enforced** (do not promise these; do not accept a review finding that demands them):
- commit-message lint (commitlint prints only, never fails)
- markdownlint (`|| true`, can never fail)
- Bandit/pip-audit (no workflow runs either)
- mypy (no workflow — `make typecheck` covers only 2 unrelated files, not this mission's surfaces)
- "kernel 90%"/"mission-loader ≥90%" named coverage floors (do not exist as named gates — diff-cover is the real, only per-PR coverage gate)
- SonarCloud (`sonar-pr` job is `continue-on-error`, informational, excluded from the terminal `aggregate-gate` job's `needs:` set — never a merge blocker)
- Typer JSON error surface / `patch()` target validation / Contextive glossary freshness as dedicated CI jobs (no dedicated job for any of these; if covered at all it is inside a test group or the architectural battery, never a separate promise)

**`make ci-parity` was run against the current diff** (mission-dir-only so far — spec-phase artifacts: `meta.json`, `reviews/*.yaml`, `spec.md`, `status.events.jsonl`, `tasks/.gitkeep`, `tasks/README.md`) and selected **12 jobs**: `archive-freeze`, `commit-msg` (prints only), `import-linter`, `layer-rules`, `markdownlint` (prints only), `prose-scan`, `regen-check`, `router-gate`, `ruff`, `terminology`, `tests-corpus`, `uv-lock` — with **0 code shards**, which is expected since no `src/` diff exists yet at plan time. **Once implementation touches `src/charter/activation/compiler.py` and `src/specify_cli/charter_runtime/preflight/references_refresh.py`/`runner.py`, `make ci-parity` MUST be re-run on that diff** to see which additional module shards, the `architectural-heavy` lane, and `diff-cover` scoping actually apply — the shard list above is not guessed in advance.

### The baseline (stated as fact, not a TODO)

Already captured on this branch's base commit `af847be71d97c8a126e91c31a7d05629c69c93ba` (confirmed == `git merge-base main HEAD`, and `git diff af847be71d97c8a126e91c31a7d05629c69c93ba HEAD -- src/ tests/` is empty at plan time, so HEAD == base for src/tests purposes):

```
.venv/bin/python -m pytest -q tests/doctrine/test_activation_parity_guard.py \
  tests/charter/test_active_languages_idempotency.py \
  tests/charter/test_context_catalog_miss.py \
  tests/charter/test_catalog_completeness_4785.py
```
→ **56 passed, 0 failed, 1 warning (75.20s)**.

`tests/charter/test_catalog_completeness_4785.py` (issue #4785, mission `charter-catalog-coherence-01M2XQQF`)
is added to this baseline because it pins **Contract C4** ("Recompiled catalog is complete and
stable", `kitty-specs/charter-catalog-coherence-01M2XQQF/contracts/behavior-contracts.md`) directly
against `_render_kind_references` — the exact function WP-CORE rewrites — via
`test_render_kind_references_routes_genuine_miss_to_diagnostics_not_placeholder`, which asserts a
genuinely-nonexistent id produces `references == []` (diagnostics-only, never a placeholder row).
WP-CORE's design (see "WP-CORE reconciliation: Contract C4 scope and placeholder text format" below)
scopes the new placeholder mechanism to the `SCOPE_FILTERED` cause specifically so this contract is
preserved, not superseded — this file's inclusion in the baseline is what lets WP-CORE's own
green-bar re-check actually catch a regression against it, rather than silently missing it the way
the original (pre-review) 51-test command would have.

No pre-existing red in the relevant shards — nothing needs a Pre-existing Failure Reporting Rule issue for this mission's blast radius. This is stated plainly and will not be re-run at plan time.

### Campsite-clean scope

`_render_kind_references`'s unresolved-append (`diagnostics.append(f"Unresolved reference: {kind}/{raw_id}")`, compiler.py ~line 1140) and `_build_references_from_service`'s `graph.unresolved` loop (`diagnostics.append(f"Unresolved reference: {artifact_type}/{artifact_id}")`, ~line 1251) duplicate the same literal format string at two call sites. **Decision: no separate preceding campsite-clean commit.** FR-001/FR-002's own fix necessarily rewrites BOTH sites' unresolved-handling logic (turning "append a diagnostic string and drop the reference" into "classify the cause, build a placeholder reference, append a reason-bearing diagnostic AND a structured record") — a distinct prior commit that only extracted a shared "record unresolved" helper with unchanged behavior would be immediately overwritten by the functional commit that follows it, making the two-commit split circular rather than a genuine tidy-first enabler. The unification happens naturally as part of the functional change itself (both call sites end up calling one new shared classify-and-placeholder helper). This is a considered decision under Standing Order #2, not a silently skipped bullet.

### Tracer files

Seeded now at `kitty-specs/charter-generation-drops-scoped-references-01M3M1KF/tracer-tooling-friction.md`, `tracer-approach.md`, `tracer-design-decisions.md` (see repo for full content — summarized in this plan's Phasing section below).

## Project Structure

### Documentation (this mission)

```
kitty-specs/charter-generation-drops-scoped-references-01M3M1KF/
├── plan.md                                     # This file
├── tracer-tooling-friction.md                  # Seeded this phase (C-003 mission tracer files)
├── tracer-approach.md                          # Seeded this phase
├── tracer-design-decisions.md                  # Seeded this phase
├── contracts/
│   └── charter-generate-json-diagnostics.md    # New — documents the additive `unresolved_references` JSON field (FR-004)
└── tasks.md / tasks/                           # Phase 2 output (/spec-kitty.tasks command — NOT created by /spec-kitty.plan)
```

No `research.md` — this is a targeted defect fix against already-verified, already-read code (every claim above was checked against the current `HEAD` source, not assumed); there is no open unknown a research phase would resolve. No `data-model.md` — no new persisted data model; `CharterReference` (existing dataclass) and `CharterPreflightResult` (existing, unchanged) are the only data shapes involved, and both are already fully specified in the spec's Key Entities section and this plan's Charter Check section respectively. `contracts/charter-generate-json-diagnostics.md` IS added because FR-004 is a genuine, additive JSON-contract change that downstream automated consumers need documented (mirrors the precedent `contracts/charter-preflight-json.md` sets for the sibling preflight contract).

### Source Code (repository root)

```
src/
├── charter/
│   └── activation/
│       ├── compiler.py            # FR-001/FR-002/FR-003 fix: _render_kind_references,
│       │                          #   _build_references_from_service, new
│       │                          #   _classify_and_placeholder_reference() helper (private,
│       │                          #   not added to __all__ — see C-007 section below)
│       └── _catalog_miss.py       # REUSED, not modified: CatalogMissCause, classify_catalog_miss,
│                                  #   classify_scope_filtered_miss (already built, already tested)
├── specify_cli/
│   ├── cli/commands/charter/
│   │   └── generate.py            # FR-004 fix: add `unresolved_references` to the --json payload
│   └── charter_runtime/preflight/
│       ├── references_refresh.py  # Auto-refresh fold-in: refresh_references_if_needed's return
│       │                          #   contract widens from bool to a small outcome type
│       └── runner.py               # Auto-refresh fold-in: refresh_references_if_needed wrapper +
│                                    #   _attempt_auto_refresh's post-refresh branch

tests/
├── charter/
│   ├── test_active_languages_idempotency.py  # Precedent read for FR-005's fixture scaffolding
│   ├── test_catalog_completeness_4785.py     # UNCHANGED (Contract C4, #4785) — added to WP-CORE's
│   │                                          #   blast-radius/baseline command (see "The baseline"
│   │                                          #   above); pins that a genuine miss stays
│   │                                          #   diagnostics-only, never a placeholder row. WP-CORE
│   │                                          #   scopes its new placeholder mechanism to the
│   │                                          #   SCOPE_FILTERED cause specifically so this file
│   │                                          #   stays green, not red.
│   └── test_charter_generate_scoped_reference_parity.py  # NEW — FR-005 regression test (separate,
│                                                            #   failing-first commit, ATDD C-011)
├── doctrine/
│   └── test_activation_parity_guard.py       # UNCHANGED (C-001) — the existing coherence oracle
│                                              #   FR-005's new test also drives via run_consistency_check
└── specify_cli/charter_runtime/
    └── test_references_parity_refresh.py     # EXTENDED — the REAL, already-built test module for
                                                #   preflight.references_refresh (WP06's test home; it
                                                #   already ships _git_init, _make_generate_subprocess_fake,
                                                #   and _seed_baseline_repo fixture machinery). The
                                                #   auto-refresh fold-in's red-first regression case is
                                                #   added here — adapt _make_generate_subprocess_fake so
                                                #   the faked `generate` invocation returns a non-zero
                                                #   returncode/stderr — NOT a new preflight/ subdirectory
                                                #   (no such directory exists in the current test tree).
```

**Structure Decision**: Single Python package, existing module layout unchanged — no new top-level directories, no new packages. All changes land inside the two already-existing seams named above.

## Complexity Tracking

*No Charter Check violations requiring justification.* Both touched seams already exist and already own this exact responsibility (reference resolution in `compiler.py`; preflight auto-refresh orchestration in `references_refresh.py`/`runner.py`); no new abstraction layer, no new project, no repository-pattern addition. The `__all__`/dead-symbol convention (C-007) is honored without exception (see below) — no new public API surface requiring justification.

## `__all__` Convention (C-007, binding)

`src/charter/activation/compiler.py` already declares `__all__` (line ~53: `CharterReference`, `CompiledCharter`, `WriteBundleResult`, `compile_charter`, `provision_mission_type_activations`, `resolve_config_activated_roots`, `write_compiled_charter`). The new helper this mission introduces — a shared "classify the miss cause and build a placeholder `CharterReference`" function replacing the duplicated unresolved-append logic in `_render_kind_references`/`_build_references_from_service` — is **module-private** (leading underscore, e.g. `_classify_and_placeholder_reference`), matching the existing precedent that `_render_kind_references`, `_doctrine_model_reference`, and `_doctrine_yaml_reference` are themselves NOT in `__all__` — only genuinely public, cross-module API surfaces are listed. `tests/architectural/test_no_dead_symbols.py` walks `__all__` and requires every *listed* name to have a real caller; a private helper outside `__all__` is not subject to that walk, and it DOES have real callers (both `_render_kind_references` call sites) so it would pass regardless. If implementation later finds a reason to promote any new symbol to public (not currently planned), it must be added to `__all__` at that point.

`src/specify_cli/charter_runtime/preflight/` is **NOT** under `src/charter/` or `src/kernel/`, so the C-007 binding rule does not apply there by charter text. The module (`references_refresh.py`) does not currently declare a dataclass/NamedTuple for the widened `refresh_references_if_needed` return type in `__all__` either way — `references_refresh.py`'s own `__all__` is `["refresh_references_if_needed"]` today; the new return-type NamedTuple (see Phasing below) is an implementation detail of that one function's signature, consumed only by its one in-module caller and by `runner.py`'s wrapper — it will be added to `references_refresh.py`'s `__all__` only if a second, external caller ever needs to construct/inspect it directly (not the case here: `runner.py` only reads the returned object's attributes, never constructs one).

## ATDD-First Discipline (C-011, binding)

The FR-005 generator-to-parity regression test — `tests/charter/test_charter_generate_scoped_reference_parity.py`, extending `tests/charter/test_active_languages_idempotency.py`'s `_git_init`/`_invoke_generate`/`_read_catalog` helper precedent (that module was read in full during this planning phase; its helpers already build a real `tmp_path` git repo and drive the real `charter_app` `generate` CLI via `typer.testing.CliRunner`, which is exactly what User Story 2's Acceptance Scenario 1 requires) — lands as a **separate, committed, failing-first commit BEFORE any implementation commit** touches `compiler.py`. The reviewer verifies RED on this mission's `planning_base_branch` (`af847be71d97c8a126e91c31a7d05629c69c93ba`) and GREEN on the lane's final commit. This is stated here in the phasing section, not merely mentioned in passing, per the charter's ATDD-First Discipline section.

The auto-refresh fold-in (below) gets its OWN separate red-first test (an extension to `tests/specify_cli/charter_runtime/test_references_parity_refresh.py` — the real, existing test module for `preflight.references_refresh`; NOT a new `preflight/` subdirectory, which does not exist in the current test tree) — it is a distinct defect class from FR-001–FR-005 and must not be conflated into the same red/green pair; conflating them would make it impossible for a reviewer to verify each fix independently caused its own test to flip.

## Phasing & Sequencing

1. **WP-ATDD — FR-005 regression test (failing-first, committed alone).** Author `tests/charter/test_charter_generate_scoped_reference_parity.py`: build a `tmp_path` git repo fixture whose `.kittify/config.yaml` activates the six #5257 ids (`styleguide/java-conventions`, `toolguide/maven-review-checks`, `toolguide/typescript-mutation-tools`, `agent-profile/frontend-freddy`, `agent-profile/java-jenny`, `agent-profile/node-norris`) plus at least one additional, differently language-scoped id (per C-002, proving the general mechanism, not six special cases — e.g. a language-filtered `tactic` or `procedure`, chosen at implementation time from an artifact under `packs/built-in/` whose `applies_to_languages` excludes whatever this fixture's `infer_repo_languages` detects). Drive the real `charter_app` `generate` CLI (`_invoke_generate` precedent), then feed the resulting repo through `ProjectContext.from_repo()` + `run_consistency_check` directly (spec's Acceptance Scenario 3 — NOT the hardcoded-`_REPO_ROOT` `test_this_project_charter_pack_is_coherent`, which cannot be conditioned on an arbitrary fixture path). Assert: (a) every fixture id is present in `catalog.references`; (b) `run_consistency_check` reports coherent; (c) pre-fix, this test is RED (verified by running it against `af847be71` before the compiler fix lands — the reviewer's own verification step). Commit alone.
2. **WP-CORE — FR-001/FR-002/FR-003 compiler fix.** In `src/charter/activation/compiler.py`: replace the two duplicated `diagnostics.append(f"Unresolved reference: ...")` sites with a single shared classify-and-placeholder helper that (a) checks `raw_id in repository.scope_filtered_ids` (the raw per-kind repository already exposes this via `BaseDoctrineRepository.scope_filtered_ids`) to route to `classify_scope_filtered_miss` (`CatalogMissCause.SCOPE_FILTERED`) vs. falls through to `classify_catalog_miss(raw_id, available_ids)` (`TYPO_SUSPECTED`/`MISSING_ARTIFACT`) for the never-existed case; (b) **for the `SCOPE_FILTERED` cause only**, ALWAYS preserves the id as a placeholder `CharterReference` (uniform mechanism *within that cause*, per FR-001's cardinality-agnostic requirement — never a second fail-closed exit branch for scope-filtered misses); **for `MISSING_ARTIFACT`/`TYPO_SUSPECTED`, stays on the EXISTING Contract-C4-compatible diagnostics-only path — no placeholder row, `references` unchanged at `[]` for that id** (see "WP-CORE reconciliation: Contract C4 scope and placeholder text format" below — this scoping is a deliberate, reviewed decision, not an oversight); (c) records a reason-bearing diagnostic string (`"Unresolved reference: {kind}/{id} ({cause}): {detail}"` — extends, does not replace, the existing free-text format) into the existing `diagnostics: list[str]` **for both causes** (FR-002/NFR-002's reason-bearing requirement applies whether or not the id also gets a placeholder row); (d) additionally records a structured `(kind, id, cause, detail)` record for every cause that flows through to FR-004's `unresolved_references` JSON field, so a `MISSING_ARTIFACT`/`TYPO_SUSPECTED` miss is still machine-readable even though it has no placeholder row. `--force` (FR-003) touches nothing in this path — it already only gates catalog-file overwrite permission upstream of `_build_references`, never reference resolution, so FR-003 is verified by a paired same-fixture run with/without `--force` producing identical diagnostic/placeholder content (no code change required for FR-003 itself, only a test assertion). Verify the WP-ATDD test now passes GREEN, and the existing 56-test baseline (`test_activation_parity_guard.py` + `test_active_languages_idempotency.py` + `test_context_catalog_miss.py` + `test_catalog_completeness_4785.py`, per "The baseline" above) stays green (NFR-001, C-001, FR-006) — `test_catalog_completeness_4785.py` specifically proves Contract C4 (the `MISSING_ARTIFACT`/`TYPO_SUSPECTED` diagnostics-only path) is undisturbed by this change.
3. **WP-JSON — FR-004 structured diagnostics.** In `src/specify_cli/cli/commands/charter/generate.py`'s `--json` emit block: thread the structured unresolved-reference records from `compiled` (via a new field on the compiler's own result type carrying the list WP-CORE built) into the new `unresolved_references` key, alongside the unchanged `diagnostics` list. Add/extend `contracts/charter-generate-json-diagnostics.md`. Test: Acceptance Scenario 2 (id + reason present machine-readably in `--json` output).
4. **WP-AUTOREFRESH — preflight auto-refresh swallow fold-in.** Bounded fix to `references_refresh.py`/`runner.py`'s `_attempt_auto_refresh` (full design in tracer-design-decisions.md and restated below) — its own red-first test, its own commit, sequenced independently of WP-CORE/WP-JSON (touches a disjoint file set, no ordering dependency either direction).
5. **WP-CLEANUP-VERIFY** (folded into WP-CORE/WP-JSON's own commits, not a separate step): re-run `make ci-parity` against the real `src/` diff once WP-CORE/WP-JSON/WP-AUTOREFRESH land, to confirm the actual module-shard/architectural-heavy/diff-cover selection (deferred — cannot be predicted at plan time, stated explicitly above).

## WP-CORE reconciliation: Contract C4 scope and placeholder text format

Two design decisions, made during this planning phase in response to adversarial plan-review
findings, bind WP-CORE's implementation. Full reasoning lives in `tracer-design-decisions.md`
(two new decision entries); this section states the resolutions plainly so an implementing agent
does not have to reconstruct them from the tracer.

**1. Contract C4 is NOT superseded — WP-CORE's placeholder path is scoped to `SCOPE_FILTERED` only.**
`_render_kind_references` (`src/charter/activation/compiler.py:1096`) is also the function a prior,
COMPLETED mission (`charter-catalog-coherence-01M2XQQF`, issue #4785) pinned as **Contract C4**
("Recompiled catalog is complete and stable",
`kitty-specs/charter-catalog-coherence-01M2XQQF/contracts/behavior-contracts.md`): "Given a directive
id with genuinely no bundled definition, Then it surfaces through the `graph.unresolved` diagnostics
channel, not as a silent placeholder `catalog.references` row." This is pinned today by
`tests/charter/test_catalog_completeness_4785.py::test_render_kind_references_routes_genuine_miss_to_diagnostics_not_placeholder`,
currently green (5 passed on this branch). An always-placeholder design — regardless of cause — would
flip that test red. **Decision: option (b), narrow the scope.** WP-CORE's new placeholder mechanism
applies ONLY to the `SCOPE_FILTERED` cause (the actual #5257/#5253 defect: an id present on disk,
excluded solely by language/scope). A `MISSING_ARTIFACT`/`TYPO_SUSPECTED` miss (genuinely no bundled
definition) stays on the existing Contract-C4-compatible diagnostics-only path — Contract C4 and its
pinning test are **not modified**. FR-001's disjunctive "preserve OR loudly report" bar is still met
for the genuinely-missing case: a reason-bearing diagnostics-list entry (FR-002/NFR-002 apply to this
path too) is the "loudly report" branch; it does not also need to become a placeholder row.
Consequence: `tests/charter/test_catalog_completeness_4785.py` is added to WP-CORE's blast-radius
baseline command (see "The baseline" above, now 56 tests) so this contract's continued health is
actually verified, not assumed.

**2. `SCOPE_FILTERED` placeholder summary text — extend the existing baseline string, don't replace
it.** NFR-002 requires the new `SCOPE_FILTERED` placeholder's `summary` to name the reason category.
The current baseline text (`_doctrine_yaml_reference`'s fallback, `compiler.py:1505`:
`"Definition unavailable in bundled doctrine."`, already committed verbatim in this repo's own
`.kittify/charter/charter.yaml` for the still-unresolved ids) names no reason at all, and spec.md's
Edge Case requires a repo already carrying that baseline to "not further diverge from" it on
regeneration. **Decision: option (a), extend as a stable prefix.** The new placeholder's `summary`
keeps `"Definition unavailable in bundled doctrine."` verbatim as a prefix and appends a
reason-bearing suffix sourced from `classify_scope_filtered_miss`'s own suggestion text:
`"Definition unavailable in bundled doctrine. Reason: scope_filtered — <suggestion>"`. This satisfies
NFR-002 (reason category + detail present) without diverging from the Edge Case's existing
convention (the literal prefix is preserved byte-for-byte). WP-CORE adds one concrete test: regenerate
a fixture seeded with today's exact baseline row shape for a still-unresolvable `SCOPE_FILTERED` id
and assert the post-fix `summary` matches this prefix+suffix format exactly (placed alongside the
other WP-CORE unit-level assertions in `tests/charter/test_charter_generate_scoped_reference_parity.py`
or a narrower `compiler.py`-level unit test, implementer's choice — either satisfies IC-01/IC-04).

## Auto-refresh fail-closed swallow — decision: FOLD IN (bounded)

**Verified against the actual current code** (read `src/specify_cli/charter_runtime/preflight/references_refresh.py` lines 145–189 and `runner.py` lines 553–773 in full at plan time, not restated from a prompt sketch on faith):

- `references_refresh.refresh_references_if_needed` (lines 145–189) spawns `subprocess.run(cmd, cwd=repo_root, capture_output=True, text=True, timeout=..., check=False)`, binds the `CompletedProcess` to a discarded expression (no variable capture of the return value at all — confirmed: line 175's `subprocess.run(...)` call result is not assigned), catches only `(OSError, subprocess.TimeoutExpired)` at debug-log level, and **unconditionally `return`s `True`** (line 189) regardless of whether the subprocess exited 0 or non-zero.
- Its caller, `runner.py::_attempt_auto_refresh` (lines 724–759), receives that `True` as `references_refreshed` and — with no outcome information available — unconditionally re-runs `spec-kitty charter synthesize` (the manifest re-stamp, lines 749–759) to re-stamp `synthesized_drg`'s stored content-hash against whatever is CURRENTLY in `charter.yaml`. Per the module's own "post-#2759 retirement" docstring note, `synthesized_drg` staleness is fundamentally "does the manifest's stored hash match a hash of charter.yaml's CURRENT content" — so restamping over a `charter.yaml` that `generate` FAILED to actually reconcile makes the recompute at lines 763–765 see `synthesized_drg="fresh"` even though the underlying references-parity condition was never actually resolved.
- **This is confirmed live in the current code, independent of whether this mission's own FR-001 fix ever triggers a fail-closed `generate` exit** (per the plan's placeholder-for-`SCOPE_FILTERED` design above, `generate` itself should now essentially never need a fail-closed non-zero exit for the scoped-reference defect class specifically — but the swallow is reachable for ANY reason `generate` exits non-zero at this call site: a crash, a lock conflict, a schema error, disk-full, etc.), so it is a real pre-existing defect this mission's own investigation surfaced, not a hypothetical.

**Bounded design (verified sound, adopted as proposed by the phase agent orchestrating this plan, with the verification checks above confirming each of its load-bearing claims):**

1. Widen `references_refresh.refresh_references_if_needed`'s return contract from `bool` to a small frozen outcome type — `RefreshOutcome(attempted: bool, succeeded: bool, detail: str | None)` (a `NamedTuple` or `@dataclass(frozen=True)`, implementation's choice) — where `succeeded` reflects `completed.returncode == 0` (now the `CompletedProcess` IS bound to a name and inspected) and `detail` is a short excerpt (e.g. last non-empty stderr line, or stdout if stderr is empty) on failure. This is an internal signature confined to this one module and its one in-repo caller (`runner.py`'s `refresh_references_if_needed` wrapper, lines 553–582, and `_attempt_auto_refresh`, ~line 733) — NOT a change to `CharterPreflightResult` or any documented/public contract.
2. In `runner.py::_attempt_auto_refresh`: when the outcome is `attempted and not succeeded`, return `CharterPreflightResult(passed=False, checks=initial_checks, auto_refresh_applied=True, auto_refresh_actions=actions, blocked_reason=f"references-parity refresh failed: {detail}")` **immediately, skipping the manifest-restamp step entirely** — restamping over content that was not actually regenerated is precisely the masking mechanism being removed. When `not attempted` or `attempted and succeeded`, the existing restamp+recompute flow (lines 749–773) is unchanged, byte-for-byte.
3. Red-first test (`tests/specify_cli/charter_runtime/test_references_parity_refresh.py`, extended — the real, existing test module for `preflight.references_refresh`, which already ships `_git_init`, `_make_generate_subprocess_fake`, and `_seed_baseline_repo` fixture machinery directly reusable here; NOT the nonexistent `tests/specify_cli/charter_runtime/preflight/` path): adapt `_make_generate_subprocess_fake` so the faked `generate` subprocess invocation inside the auto-refresh path returns a non-zero returncode/stderr instead of success. Assert (a) BEFORE the fix, the swallow means the failure never reaches an actionable `blocked_reason` — `passed` can report `True` despite the underlying `generate` failure (demonstrating the bug is real, RED on `af847be71`); (b) AFTER the fix, `passed=False` and `blocked_reason` names the failure detail (GREEN).

**Why this qualifies as "bounded" under operator doctrine** (fold-in, not defer-to-follow-up, not HALT): one call site (`_attempt_auto_refresh`'s post-`refresh_references_if_needed` branch), one function's return-type widening confined to a module and its single caller, testable red-first in isolation, **no contract change** to `CharterPreflightResult`/`contracts/charter-preflight-json.md` beyond reusing the field that already exists (`blocked_reason: str | None`) in exactly the pattern the function already uses elsewhere in the same branch (lines 686–692, 704–709 already return `blocked_reason=reason` with `auto_refresh_applied=True` for a failed refresh step — this fold-in makes the references-parity step behave consistently with every OTHER step in the same sequence, rather than being the one step whose failure is invisible). It does not reach into `charter generate`'s own exit-code contract, does not touch the CLI's `--json` output shape, and does not touch any file outside the two named preflight modules. This clears the fold-in bar; a HALT-for-operator framing is not warranted here.

## Implementation Concern Map

### IC-01 — Placeholder-preservation for `SCOPE_FILTERED` unresolved DRG-backed references

- **Purpose**: Close the silent-drop defect (FR-001) by making every activated-but-unresolvable id whose cause is `SCOPE_FILTERED` a placeholder `CharterReference` instead of a dropped one, uniformly regardless of cardinality (one scope-filtered id or every scope-filtered id of a kind). `MISSING_ARTIFACT`/`TYPO_SUSPECTED` misses stay on the existing Contract-C4-compatible diagnostics-only path (never a placeholder row) — see "WP-CORE reconciliation: Contract C4 scope and placeholder text format" above; this is a deliberate cause-conditioned scoping, not an unconditional "every unresolved id" rule.
- **Relevant requirements**: FR-001, FR-003, NFR-001, NFR-002, C-001, C-002, SC-001, SC-004, SC-005.
- **Affected surfaces**: `src/charter/activation/compiler.py` (`_render_kind_references`, `_build_references_from_service`, new private classify-and-placeholder helper).
- **Sequencing/depends-on**: Follows WP-ATDD (the FR-005 test must exist and be RED first).
- **Risks**: (1) The existing diagnostic string format (`"Unresolved reference: <kind>/<id>"`) is extended, not replaced — implementation must grep the test suite for any exact-string assertion on the OLD format before changing it, to avoid an unrelated test regression outside this mission's stated scope. (2) Regression risk against **Contract C4**: `tests/charter/test_catalog_completeness_4785.py` (added to the baseline command above) must stay green — a `MISSING_ARTIFACT`/`TYPO_SUSPECTED` id must never fall through to the placeholder branch; the `scope_filtered_ids` cause-check must run BEFORE any placeholder-construction code, not after.

### IC-02 — Reason classification reusing `_catalog_miss` primitives

- **Purpose**: Satisfy FR-002/NFR-002 by distinguishing never-existed from filtered-by-language-scope, reusing `CatalogMissCause`/`classify_catalog_miss`/`classify_scope_filtered_miss` rather than inventing a second classification scheme (DIRECTIVE_044).
- **Relevant requirements**: FR-002, NFR-002, SC-003 (User Story 3 acceptance scenarios).
- **Affected surfaces**: `src/charter/activation/compiler.py` — consumes `charter.activation._catalog_miss` and `BaseDoctrineRepository.scope_filtered_ids` (neither is itself modified), AND requires threading a new `active_languages: list[str] | None` parameter across three call levels: `compile_charter` already computes `active_languages` at line 424 but currently does not pass it to `_build_references` (line ~460); `_build_references` (line 1045) must accept and forward it to `_build_references_from_service` (line 1144); `_build_references_from_service` must forward it to the new classify-and-placeholder helper so `classify_scope_filtered_miss(artifact_id, active_languages=...)` can produce its richer, language-set-naming suggestion (`src/charter/activation/_catalog_miss.py:261-305`) instead of falling back to the generic, language-set-less message. `contracts/charter-generate-json-diagnostics.md`'s own worked example (the `detail` string naming `('python')`) depends on this threading actually happening.
- **Sequencing/depends-on**: IC-01 (same commit/WP — the classify step and the placeholder-build step are one cohesive change, not separable without circularity, per the Campsite-clean decision above).
- **Risks**: Without the `active_languages` threading above, FR-002/NFR-002's minimum bar (a reason category is present) still passes, but the emitted `detail`/suggestion text is the weaker, generic fallback rather than the contract doc's documented example — implementation must not skip the threading on the assumption that the reused primitives being "already tested" (`tests/charter/test_context_catalog_miss.py`, part of the baseline) makes the plumbing itself unnecessary. Add one assertion (WP-ATDD or WP-JSON, implementer's choice) that the emitted detail/suggestion text for a `SCOPE_FILTERED` case names the actual active language set, matching the contract doc's worked example.

### IC-03 — Machine-readable `--json` diagnostics

- **Purpose**: FR-004 — carry unresolved-reference id+reason structurally in `charter generate --force --json` output.
- **Relevant requirements**: FR-004, SC-001 (JSON half).
- **Affected surfaces**: `src/specify_cli/cli/commands/charter/generate.py` (JSON emit block), a new field on the compiler's result type to carry the structured records from `_build_references` through to the CLI layer, `contracts/charter-generate-json-diagnostics.md` (new).
- **Sequencing/depends-on**: IC-01/IC-02 (the structured records must exist before the CLI layer can surface them).
- **Risks**: Must not change the `diagnostics: list[str]` field's existing shape (additive-only contract change, verified against every current reader of that field before landing).

### IC-04 — Generator-to-parity regression test

- **Purpose**: FR-005 — pin this defect class at the real `compile_charter`/`charter generate` entry point so it cannot silently regress.
- **Relevant requirements**: FR-005, SC-002, SC-003, SC-004.
- **Affected surfaces**: `tests/charter/test_charter_generate_scoped_reference_parity.py` (new), extending `tests/charter/test_active_languages_idempotency.py`'s fixture helpers by reference (not by copy — import/reuse where the helpers are already suitably general, or lift near-identical private helpers into the new module per existing test-module conventions in this codebase).
- **Sequencing/depends-on**: None (this is WP-ATDD, sequenced FIRST — see Phasing above).
- **Risks**: Fixture construction must avoid hardcoding the six #5257 ids as the ONLY proof (C-002) — the extra differently-scoped id is mandatory, not optional.

### IC-05 — Preflight auto-refresh swallow fold-in

- **Purpose**: Close the pre-existing silent-success masking bug in `_attempt_auto_refresh`'s references-parity step, per operator doctrine (fold bounded same-defect-class fixes rather than defer).
- **Relevant requirements**: Edge Cases "Mid-flight missions" (spec.md), this mission's own NFR-002/FR-001 silent-success bar applied reflexively to the preflight seam.
- **Affected surfaces**: `src/specify_cli/charter_runtime/preflight/references_refresh.py`, `src/specify_cli/charter_runtime/preflight/runner.py` (`_attempt_auto_refresh` only — no other function in this module changes), `tests/specify_cli/charter_runtime/test_references_parity_refresh.py` (the real, existing test module for `preflight.references_refresh` — extended with the fold-in's red-first case via its own `_make_generate_subprocess_fake` helper, not a new `preflight/` subdirectory).
- **Sequencing/depends-on**: None — independent of IC-01–IC-04 (disjoint file set, no shared state).
- **Risks**: Must not alter `_attempt_auto_refresh`'s behavior for any OTHER step in the sequence (source-remediation, synthesize, bundle-validate) — the change is scoped exclusively to the branch immediately following `refresh_references_if_needed`'s call.

## PR Shape

**One PR for this whole mission** (the default per charter: spec-kitty carries no per-WP-PR mandate, and `spec-kitty accept` → `spec-kitty merge` assumes one mission branch / one PR). The combined compiler.py + preflight fix is not judged too large for one reviewable PR: it is five small, independently-testable, mostly-disjoint changes (IC-01/02 as one commit pair, IC-03 as one commit, IC-04 as the leading ATDD commit, IC-05 as one self-contained commit) totalling well under the scale that would warrant a split, and splitting the preflight fold-in into a separate PR would require re-litigating the "fold vs. defer" operator decision as a second review cycle for no material review-burden reduction. Default (one PR) applies; no split recommended.
