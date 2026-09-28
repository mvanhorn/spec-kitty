# Tracer: Design Decisions

Mission: `charter-generation-drops-scoped-references-01M3M1KF` (#5257)

## Decision: fold in the preflight auto-refresh fail-closed swallow (bounded)

**Context.** Spec.md's Edge Cases "Mid-flight missions" bullet identifies a confirmed fact (verified
against the actual current code during this planning phase, not taken on faith from the dispatch
prompt): `src/specify_cli/charter_runtime/preflight/references_refresh.py::refresh_references_if_needed`
spawns the targeted `generate` subprocess via `subprocess.run(cmd, cwd=repo_root,
capture_output=True, text=True, timeout=..., check=False)` (line 175), discards the returned
`CompletedProcess` without binding it to a name, catches only `(OSError, subprocess.TimeoutExpired)`
at debug-log level (lines 183–188), never inspects `.returncode`, and **unconditionally
`return`s `True`** (line 189) regardless of the subprocess's actual outcome. Its one caller,
`_attempt_auto_refresh` in `runner.py` (~line 733), treats that `True` as "an attempt was made" with
no success/failure information, and unconditionally re-runs `spec-kitty charter synthesize` (lines
749–759, flagless, `SynthesizeMode.preserve`) to re-stamp the synthesis manifest's
`bundle_content_hash` against `charter.yaml`'s CURRENT content — then recomputes freshness (lines
763–765) and reports `passed=post_passed`.

**The concrete hazard.** Per the module's own "post-#2759 retirement" docstring note,
`synthesized_drg` staleness is fundamentally "does the manifest's stored hash match a hash of
charter.yaml's CURRENT content" — so re-stamping over a `charter.yaml` that `generate` FAILED to
actually reconcile (for ANY reason: a crash, a lock conflict, a schema error, disk pressure — not
only this mission's specific scoped-reference defect class) makes the post-refresh recompute report
`synthesized_drg="fresh"` even though the underlying references-parity condition was never resolved.
This is the class this repo's charter names as its dominant failure mode: silent success. It is a
**pre-existing defect this mission's own investigation surfaced**, independent of whether this
mission's own FR-001 fix (placeholder-always, see tracer-approach.md) ever itself triggers a
fail-closed `generate` exit — which, per that design choice, it mostly should not need to for THIS
defect class, but the swallow remains reachable for any other `generate` failure mode at that call
site.

**Operator doctrine applied (not re-litigated):** prefer folding in a bounded, same-defect-class fix
over deferring to a follow-up issue, because a silently-swallowed fail-closed exit is itself a
silent-success violation of this mission's own acceptance bar (FR-001, NFR-002). Fold it in IF
bounded: one call site, testable red-first, no contract change to `CharterPreflightResult`/
`contracts/charter-preflight-json.md` beyond what already exists.

**Decision: FOLD IN**, with the following bounded design — verified sound against the actual code
(not restated from the dispatch prompt without checking):

1. Widen `references_refresh.refresh_references_if_needed`'s return contract from `bool` to a small
   frozen outcome type (`RefreshOutcome(attempted: bool, succeeded: bool, detail: str | None)`),
   where `succeeded` reflects the now-bound `completed.returncode == 0` and `detail` is a short
   excerpt of stderr/stdout on failure. Confined to this one module and its one in-repo caller
   (`runner.py`'s wrapper at ~line 553 and `_attempt_auto_refresh` at ~line 733) — not a change to
   any public/documented contract.
2. In `_attempt_auto_refresh`: when `attempted and not succeeded`, return
   `CharterPreflightResult(passed=False, checks=initial_checks, auto_refresh_applied=True,
   auto_refresh_actions=actions, blocked_reason=f"references-parity refresh failed: {detail}")`
   immediately, **skipping the manifest-restamp step entirely** — restamping over un-regenerated
   content is precisely the masking mechanism being removed. `blocked_reason` is an existing field
   (`result.py` line 98) already populated with `auto_refresh_applied=True` at multiple other branches
   in the same function (lines 686–692, 704–709) — this fold-in makes the references-parity step
   consistent with every OTHER step in the same sequence, which already surfaces its own failures
   this way, rather than adding a new mechanism.
3. Red-first test: monkeypatch/mock the `generate` subprocess to return non-zero inside the
   auto-refresh path. Assert (a) BEFORE the fix, the failure never reaches an actionable
   `blocked_reason` (RED, proving the bug is real on `af847be71`); (b) AFTER the fix, `passed=False`
   and `blocked_reason` names the failure (GREEN).

**Why this is "bounded", not a HALT-for-operator case:** one call site, one function's return-type
widening confined to a module and its single caller, testable red-first in isolation, no contract
change beyond reusing an already-existing field in an already-established pattern within the same
function, no reach outside the two named preflight files. See plan.md's own "Auto-refresh
fail-closed swallow — decision: FOLD IN (bounded)" section for the full verification trail.

## Decision: WP-CORE's placeholder path is scoped to `SCOPE_FILTERED` only — Contract C4 stands, not superseded

**Context (post-plan-review finding PLAN-GOV-001).** The plan-phase adversarial squad found that
WP-CORE's original design — "ALWAYS preserves the id as a placeholder `CharterReference` ...
uniformly regardless of cardinality/cause, never a second fail-closed exit branch" — silently
reverses **Contract C4** ("Recompiled catalog is complete and stable",
`kitty-specs/charter-catalog-coherence-01M2XQQF/contracts/behavior-contracts.md`, issue #4785, a
prior COMPLETED mission). Contract C4 requires: "Given a directive id with genuinely no bundled
definition, Then it surfaces through the `graph.unresolved` diagnostics channel, not as a silent
placeholder `catalog.references` row." This is pinned today by a real, currently-green test —
`tests/charter/test_render_kind_references_routes_genuine_miss_to_diagnostics_not_placeholder`
(`tests/charter/test_catalog_completeness_4785.py:191-216`) — which asserts `references == []`
(never a placeholder row) and the exact diagnostics string for a genuine, never-existed miss
against `_render_kind_references` (`src/charter/activation/compiler.py:1096`), the EXACT function
WP-CORE rewrites. Verified live: `.venv/bin/python -m pytest -q
tests/charter/test_catalog_completeness_4785.py` is 5 passed on this branch today. Under the
original always-placeholder design, both of that test's assertions go red the moment WP-CORE lands.

**Decision: option (b) — narrow WP-CORE's placeholder path to the `SCOPE_FILTERED` cause only.**
Contract C4 is NOT superseded; it stands as written and its pinning test is NOT modified. The
narrower design is:

- When `_render_kind_references`'s raw-repository lookup misses for `raw_id`, first check
  `raw_id in repository.scope_filtered_ids` (`BaseDoctrineRepository.scope_filtered_ids`,
  `src/charter/offering/base.py`). If **true** (the artifact exists on disk, present but excluded
  by language/scope — this mission's actual #5257/#5253 defect class), classify via
  `classify_scope_filtered_miss` (`CatalogMissCause.SCOPE_FILTERED`) and preserve the id as a
  reason-bearing placeholder `CharterReference` — uniformly regardless of *cardinality* within this
  cause (one scope-filtered id or every scope-filtered id of a kind, per Acceptance Scenario 4 /
  SC-005, is handled by the same code path).
- If **false** (the artifact genuinely has no bundled definition anywhere — `MISSING_ARTIFACT` or
  `TYPO_SUSPECTED` via `classify_catalog_miss`), stay on the EXISTING Contract-C4-compatible
  diagnostics-only path: no placeholder row, `references` stays `[]` for that id, and a
  reason-bearing diagnostic string is appended (FR-002/NFR-002 apply to this path too — the
  diagnostic text gains the `(cause): detail` suffix exactly like the scope-filtered path's
  diagnostic does; only the placeholder-row *mechanism* is withheld, not the reason-classification
  improvement).

**Why (b), not (a) [explicit supersession of Contract C4]:**

1. **Correctly scoped to the actual defect.** #5257/#5253's defect is specifically that a
   *present-on-disk, scope-filtered* id was being silently dropped as if it never existed. Genuinely
   nonexistent ids were never the defect — Contract C4's diagnostics-only behavior for that case is
   already correct, already tested, and already a "loud report" (FR-001's title is "Preserve **or**
   loudly report" — a reason-bearing diagnostics-list entry, which is what Contract C4 already
   produces and what FR-002 makes reason-bearing, satisfies the disjunctive "loudly report" branch;
   it does not need to also become a placeholder row to satisfy FR-001).
2. **DIRECTIVE_044 (single canonical authority) favors reconciliation over silent supersession.**
   Contract C4 is a real, prior-mission, currently-tested authority on this exact function. Option
   (a) would require editing both `test_catalog_completeness_4785.py`'s assertions AND
   `behavior-contracts.md`'s Contract C4 text in the same commit with a rationale note — a real,
   defensible path, but a heavier one that touches a settled prior mission's contract to fix a
   narrower, unrelated defect. Option (b) reconciles by *scoping* the new mechanism precisely to
   where it is needed, leaving the existing authority untouched and still correct.
3. **Smallest-viable-diff (charter's change-scope reconciliation order).** Narrowing to
   `SCOPE_FILTERED` is the minimal edit that closes the actual defect; it does not require touching
   a second mission's contract doc or a currently-green regression test outside this mission's
   stated blast radius.
4. **No conflict with SC-005 / Acceptance Scenario 4.** That scenario's fixture ("every activated
   `toolguide` id excluded by language scope") is, by construction, a `SCOPE_FILTERED` case — it is
   fully satisfied by the narrowed placeholder path with no special aggregate-cardinality handling
   needed (same reasoning tracer-approach.md's "Rejected alternative" section already gives for why
   always-placeholder-within-a-cause needs no cardinality branch).

**Consequence for the plan:** `tests/charter/test_catalog_completeness_4785.py` is added to
WP-CORE's stated blast-radius test command (it is NOT modified, but it now sits directly in the
diff's path and must be run to prove Contract C4 still holds after WP-CORE lands). Verified today,
with the file included: `.venv/bin/python -m pytest -q tests/doctrine/test_activation_parity_guard.py
tests/charter/test_active_languages_idempotency.py tests/charter/test_context_catalog_miss.py
tests/charter/test_catalog_completeness_4785.py` → **56 passed, 0 failed, 1 warning (75.20s)**
(the prior 51-test baseline plus this file's 5 tests). WP-CORE's own green-bar re-check must use
this 56-test command, not the original 51-test one, so a future regression against Contract C4 is
actually caught rather than silently missed by an incomplete shard list.

## Decision: SCOPE_FILTERED placeholder summary text — extend the existing baseline string, don't replace it

**Context (post-plan-review finding PLAN-ARCH-001).** NFR-002 requires every placeholder entry to
name a reason category. The CURRENT committed convention for an unresolvable id's placeholder
summary — set by `_doctrine_yaml_reference`'s fallback (`src/charter/activation/compiler.py:1505`:
`{"summary": "Definition unavailable in bundled doctrine."}`) — is the literal string
`"Definition unavailable in bundled doctrine."` with no reason category, verified against this
repo's own committed `.kittify/charter/charter.yaml` (nine rows carry this exact string today,
e.g. lines 962, 1250, 1346, 1460, 1496, 1514, 1670, 1694, 1712). Spec.md's Edge Case says a repo
already carrying this baseline "must not further diverge from" the existing placeholder convention
on regeneration. Making the new `SCOPE_FILTERED` placeholder (see the decision above — WP-CORE's
placeholder path only builds a placeholder row for the `SCOPE_FILTERED` cause) satisfy NFR-002
therefore has to change this text somehow, which is in tension with "must not diverge" unless the
change is framed as an extension.

**Decision: option (a) — extend, don't replace.** The new `SCOPE_FILTERED` placeholder's `summary`
keeps `"Definition unavailable in bundled doctrine."` as a stable, byte-identical PREFIX and appends
a reason-bearing suffix built from `classify_scope_filtered_miss`'s own suggestion text:

```
Definition unavailable in bundled doctrine. Reason: scope_filtered — <classify_scope_filtered_miss suggestion text>
```

This satisfies NFR-002 (the reason category `scope_filtered` and the specific detail are named) and
the Edge Case (the existing literal string is preserved verbatim as a prefix — a repo regenerating
today's baseline sees its existing text extended, never discarded or reworded). This is an
extension, not a replacement, of the existing convention — the same "template, not direct reuse"
relationship tracer-approach.md's "Overall approach" section already describes between
`_doctrine_yaml_reference` (paradigm placeholders, YAML-dict sourced) and the new DRG-backed-kind
placeholder path (typed-repository sourced): the new path borrows the *prefix convention*, not the
function itself. WP-CORE adds one concrete test (fixture: a still-unresolvable `SCOPE_FILTERED` id
regenerated from a repo seeded with today's exact baseline row shape) asserting the post-fix
`summary` matches this prefix+suffix format exactly — see plan.md's IC-01/WP-CORE text for the test
placement.

## Decision: add an aggregate, cause-agnostic whole-kind-unresolved fail-closed check (round-2 fix, closes PLAN-FRESH-001)

**Context (post-plan-fresh-review finding PLAN-FRESH-001, second adversarial round after the
decision above landed).** A fresh-eyes review of the fix that produced the "SCOPE_FILTERED-only"
narrowing (the decision immediately above) found that it reopened a different gap: FR-001's own
text and its matching Edge Case bullet give "a misconfigured pack root" — a MISSING_ARTIFACT-class
cause, NOT SCOPE_FILTERED — as an illustrative *whole-kind-unresolvable* scenario, and state that
only two mechanisms satisfy FR-001 "at any cardinality": placeholder-preservation of every affected
id, OR a fail-closed non-zero exit naming the kind. The SCOPE_FILTERED-only narrowing gives a
whole-kind MISSING_ARTIFACT/TYPO_SUSPECTED miss NEITHER: no placeholder row (by design, per the
decision above) and no fail-closed exit (`charter generate` still exits 0 and writes a structurally
valid but silently-empty-for-that-kind catalog section). Verified against spec.md directly during
this fix round (not taken on faith from the finding): FR-001's Functional Requirements table row and
the Edge Cases "All activated references of a kind... are unresolvable" bullet both state this
two-mechanism, any-cardinality bar in the spec's own words, and neither SC-005 nor Acceptance
Scenario 4 (User Story 1) exercises the MISSING_ARTIFACT/whole-kind combination — both are
SCOPE_FILTERED-only fixtures ("every activated toolguide id excluded by language scope"). So an
implementation built exactly to the pre-round-2 plan text would silently fail FR-001 for the
genuinely-missing whole-kind case, and nothing in the described test suite would catch it.

**Decision: add a second, independent mechanism — a per-kind aggregate check — rather than widening
the per-id placeholder scoping back out.** After `_render_kind_references` classifies every id of a
kind (the decision above's per-id SCOPE_FILTERED vs. MISSING_ARTIFACT/TYPO_SUSPECTED routing,
unchanged), `_build_references_from_service` checks, per kind: did this kind have at least one
activated id, AND did none of them produce a reference (no resolution, no SCOPE_FILTERED
placeholder)? If both hold, every one of that kind's activated ids fell through to the
diagnostics-only branch — the exact "misconfigured pack root" / whole-kind-genuinely-missing case
FR-001 names. When this holds for one or more kinds, `_build_references_from_service` raises a
fail-closed error (a private `RuntimeError` subclass) naming the affected kind(s) and their
unresolved ids, propagating up through `_build_references` → `compile_charter` so `generate.py`'s
existing broad `RuntimeError` handler (`generate.py:584`) exits non-zero **before**
`write_compiled_charter` (`generate.py:497`, called strictly after `compile_charter` at line 489)
ever runs — so no catalog is written for this case, matching FR-001's "exits non-zero before writing
any catalog" language. Full design (call-site details, the exact emptiness check, and the new test
requirement): plan.md's "WP-CORE reconciliation" section, point 3.

**Why this does not reopen the SCOPE_FILTERED decision above, and does not reintroduce C-002.**
This is a genuinely different axis from the decision above, not a widening of it:

1. **Different trigger, different mechanism.** The decision above governs whether a SCOPE_FILTERED
   miss gets a placeholder row (a per-id question, answered per the cause). This decision governs
   whether an entire KIND's activated-id set is left with zero catalog presence after all per-id
   classification completes (a per-kind aggregate question, answered by counting). A kind with even
   one SCOPE_FILTERED id mixed in never trips this new check — that id already produced a
   placeholder row under the decision above, so the kind's reference list is non-empty. The two
   decisions compose without conflict: the SCOPE_FILTERED-only narrowing still stands exactly as
   written; this decision only adds behavior for the case it never covered (a kind where literally
   every activated id is genuinely missing).
2. **A count comparison, not an id comparison — this is the C-002 distinction that matters.** C-002
   forbids special-casing the six literal id strings from #5257
   (`styleguide/java-conventions`, `toolguide/maven-review-checks`, `toolguide/typescript-mutation-tools`,
   `agent-profile/frontend-freddy`, `agent-profile/java-jenny`, `agent-profile/node-norris`) anywhere
   in the fix. This check never reads or compares against any literal id string at all — it compares
   two counts per kind (activated-id count vs. diagnostics-only-miss count for that kind), computed
   identically for every kind, on every repo, regardless of which ids are involved. "Check whether
   every id of a kind is unresolved" is a cardinality gate over an arbitrary kind's arbitrary id set;
   it is not the same shape of special-casing C-002 forbids, and tracer-approach.md's "Rejected
   alternative" section (which argues against a *per-id, cardinality-conditioned* fail-closed branch
   for SCOPE_FILTERED specifically) is not in tension with this decision — that section's objection
   was to branching WHEN a single mechanism applies based on how many ids are affected; this decision
   instead adds FR-001's own second, already-named mechanism (fail-closed exit) for a case the first
   mechanism (placeholder) was deliberately scoped never to cover.
3. **No new test-suite conflict.** `tests/charter/test_catalog_completeness_4785.py` (Contract C4)
   pins a SINGLE genuinely-missing id producing `references == []` with no placeholder row and no
   fail-closed exit for that single-id case — this check only fires when an entire KIND's ids are
   ALL genuinely missing, a materially different (and, per that test's own fixture, untested)
   cardinality; Contract C4's fixture is not a whole-kind case, so it is unaffected. WP-CORE's new
   test (plan.md, "WP-CORE reconciliation" point 3) covers the whole-kind case directly, closing the
   gap PLAN-FRESH-001 identified: SC-005/Acceptance Scenario 4 and this new test together now cover
   both whole-kind-unresolvable causes (SCOPE_FILTERED via the existing placeholder path, and
   MISSING_ARTIFACT/TYPO_SUSPECTED via this new fail-closed path).

## Decision: operator ruling on PLAN-FRESH2-001 — widen the whole-kind check to `graph.unresolved`, and make graph-load failure loud+structured (not fail-closed)

**Context (round 3 of the plan-phase adversarial squad, `reviews/plan-fresh-2.yaml`, finding
`PLAN-FRESH2-001`, severity 4).** A further fresh-eyes review of the decision immediately above
(the per-kind aggregate whole-kind check) found it inspects only the per-kind list
`_render_kind_references` returns. Two verified-in-code gaps fall entirely outside its field of
view: (1) `_build_references_from_service`'s OWN separate `graph.unresolved` loop
(`compiler.py:1250-1251`) — populated by `resolve_transitive_refs`
(`charter/offering/drg/query.py`) when a visited URN's DRG node is absent from an
otherwise-successfully-loaded graph — never reaches `_render_kind_references` and so never
contributes to the aggregate check; (2) `_resolve_transitive_reference_graph`'s
`except Exception: return fallback` branch (`compiler.py:1298-1307`), hit when the WHOLE DRG graph
fails to load, drops anything reachable only via directive-transitive closure with zero trace —
not even a diagnostic. This also left the "Campsite-clean scope" section's claim ("both call sites
end up calling one new shared classify-and-placeholder helper") false as written, since the
`graph.unresolved` site was never mentioned in the WP-CORE reconciliation design.

**The operator ruled on this finding directly (binding — replaces the finding's own remediation
options (a)/(b)):**

> "Widen, bounded: route `graph.unresolved` ids through the same classify-and-placeholder helper
> so they count toward the whole-kind fail-closed check and get a reason; graph-load failure emits
> a loud, structured diagnostic (also in `--json`) instead of vanishing, without attempting to
> reconstruct the transitive closure. Stays in compiler.py, same defect class, keeps the plan's
> 'one shared helper' campsite claim true."

**Explicit constraint from the operator:** graph-load failure does **NOT** make `generate` fail
closed (the operator explicitly declined the stricter option) — it must be loud and structured,
not fail-closed. This is a deliberate, bounded widening, not a reopening of decision 1 above (the
`SCOPE_FILTERED`-only placeholder scoping) or a general "make everything fail closed" reversal.

**Decision: implement exactly this ruling, nothing broader.**

1. Route `_build_references_from_service`'s `graph.unresolved` loop through the SAME
   classify-and-placeholder helper decision 3's per-kind loop already uses, so both unresolved-
   tracking sites genuinely share one helper (making the Campsite-clean section's claim true).
2. Because `graph.unresolved` ids now flow into the same per-kind `references` list
   `_render_kind_references` populates, the existing aggregate whole-kind check (decision 3, above)
   automatically counts them — no separate counting logic needed.
3. A total graph-load failure (`_resolve_transitive_reference_graph`'s `except Exception` branch)
   gets ONE unconditional, structured diagnostic — present in both the plain `diagnostics: list[str]`
   and the `--json` `unresolved_references` field — instead of vanishing. It does NOT attempt to
   reconstruct the transitive closure (the existing fallback's direct-root-only behavior is
   unchanged) and does NOT trip the whole-kind aggregate fail-closed check: that check counts
   per-kind ids that were actually classified (`SCOPE_FILTERED` vs. `MISSING_ARTIFACT`/
   `TYPO_SUSPECTED`, each attributable to one kind); a total graph-load failure has no per-kind
   breakdown to classify, so it gets its own unconditional loud diagnostic instead of being folded
   into a count it cannot honestly populate.

Full design (the exact helper call sites, the diagnostic message shape, the `--json` entry shape,
and the extended test requirement) lives in plan.md's "WP-CORE reconciliation" point 3, "Round-3
amendment" subsection, and in `contracts/charter-generate-json-diagnostics.md`'s "Round-3 addition"
section — this tracer entry records the ruling and the reasoning; those files record the
implementable design. This also closes PLAN-FRESH2-004 (idempotency): the whole-kind aggregate
check's fail-closed branch (now widened to include `graph.unresolved`) writes no catalog on either
invocation, so NFR-001's literal byte-identical-catalog guarantee does not apply to it — the
relevant property is that the same unresolved-kind repo fails the same way (same exit code, same
kind named) on a second invocation, with any pre-existing catalog left untouched. The graph-load-
failure diagnostic path (not fail-closed, still writes a catalog) remains subject to ordinary
NFR-001 unchanged.
