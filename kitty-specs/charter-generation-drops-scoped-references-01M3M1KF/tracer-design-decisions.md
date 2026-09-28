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
