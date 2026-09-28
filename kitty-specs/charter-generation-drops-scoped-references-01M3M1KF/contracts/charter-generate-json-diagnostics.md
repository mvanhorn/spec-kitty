# Contract — `spec-kitty charter generate --json` (additive change)

**Backed by**: FR-004 (mission `charter-generation-drops-scoped-references-01M3M1KF`, issue #5257)

## Status

This is an **additive, backward-compatible** change to an existing, already-shipped JSON contract
(`src/specify_cli/cli/commands/charter/generate.py`'s `--json` emit block). No existing key is
removed, renamed, or changed in type. A new top-level key is added.

## Before (existing shape, unchanged fields)

```json
{
  "result": "success",
  "success": true,
  "charter_path": "...",
  "interview_source": "...",
  "mission": "...",
  "template_set": "...",
  "selected_paradigms": [...],
  "selected_directives": [...],
  "available_tools": [...],
  "references_count": 42,
  "library_files": [...],
  "files_written": [...],
  "diagnostics": ["Unresolved reference: styleguide/java-conventions (scope_filtered): ..."]
}
```

`diagnostics` remains a flat `list[str]` — every existing consumer that treats it as such continues
to work unchanged. Per-line content is extended (FR-002) to carry a `(cause)` suffix and a short
detail, but the field's own type and key name never change.

## After (new field, additive)

```json
{
  "...": "... all fields above, unchanged ...",
  "unresolved_references": [
    {
      "kind": "styleguide",
      "id": "java-conventions",
      "cause": "scope_filtered",
      "detail": "artifact 'java-conventions' is present but its applies_to_languages scope does not include the active language set ('python')."
    }
  ]
}
```

- `unresolved_references` is **always present** when `diagnostics` would otherwise be produced by this
  code path (empty list `[]` when there are no unresolved DRG-backed references) — matching
  `diagnostics`'s own always-present convention, so a consumer can rely on key presence rather than
  using `.get()` with a default.
- `kind` is one of the DRG-backed kinds `_render_kind_references` covers: `directive`, `tactic`,
  `styleguide`, `toolguide`, `procedure`, `agent_profile`.
- `id` is the bare activated id (no `KIND:` prefix — matches the raw id as configured in
  `.kittify/config.yaml`, not the `CharterReference.id` field's `KIND:id` form).
- `cause` is the `charter.activation._catalog_miss.CatalogMissCause` enum's `.value` string:
  `"missing_artifact"`, `"typo_suspected"`, or `"scope_filtered"`. (`"schema_validation_suspected"` is
  reserved for callers with ground-truth loader-drop knowledge, which this call site does not have —
  see `_catalog_miss.py`'s own docstring — so it is not expected to appear here in practice.)
- `detail` is the human-readable classifier suggestion text `classify_catalog_miss`/
  `classify_scope_filtered_miss` already produce (unchanged text from those existing, already-tested
  primitives — this contract does not introduce a new message format, only surfaces the existing one
  structurally).

## Why additive-alongside, not widening `diagnostics` entries into objects

Any existing consumer (CI scripts, readiness probes, `_finalize_sync_result`'s own sync-warning
concatenation, which appends plain strings into the same `diagnostics` list) that treats `diagnostics`
as `list[str]` keeps working unchanged. Widening the existing field's element type to `str | dict`
would be a breaking shape change for every such consumer, with no compile-time signal in a
dynamically-typed JSON contract to catch the break. A new, always-present key is additive and
detectable (`"unresolved_references" in payload`) without touching the existing key's shape at all.

## Relationship to `CharterPreflightResult`

Not related. This contract governs `charter generate --json`'s own stdout payload. The preflight
contract (`contracts/charter-preflight-json.md` in `kitty-specs/charter-ux-and-org-pack-vocabulary-01KSAF14/`)
governs a structurally different, unrelated command (`spec-kitty charter preflight`) and is
**not modified** by this mission — see plan.md's Charter Check section.
