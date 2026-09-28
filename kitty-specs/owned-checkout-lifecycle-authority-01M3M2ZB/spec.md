# Mission Specification: Owned-checkout lifecycle authority

**Mission Branch**: `claude/sleepy-hamilton-5lelee` (planning/base and merge target)
**Created**: 2026-09-28
**Status**: Draft
**Input**: Issues #3449, #5026, #4252, #4867, #5277. PR #5009 (@samuelgoff) is prior art. The operator-locked pre-spec decisions are in `research/pre-spec-options-memo.md`.

## Summary

An operator can explicitly give a mission its own linked checkout with `--owned-checkout <path>`: a single_branch mission with no coordination branch (ADR 2026-08-12-1 and ADR 2026-09-03-1). Mission creation already writes into that checkout.

Almost every later lifecycle read still resolves against the **primary checkout**. As a result, an owned mission cannot be advanced from planning through review. Commands fail, or report success while acting on primary state.

This mission makes the **validated owned checkout the single authority** for every lifecycle read of that mission. Ownership is established in one place and handed on as a validated fact. It is never re-derived from paths, the current directory, or which checkout happens to hold a copy of the mission.

### Observed on `main` @ `af847be7` (real CLI, 2026-09-28)

| # | Command against an owned mission | Today | Issue |
|---|---|---|---|
| O1 | `agent tasks status --mission <slug\|mid8\|id>` | `MISSION_NOT_FOUND` for every handle form | #3449 |
| O2 | `agent mission setup-plan --mission <handle>` after an owned create and spec commit | `PLAN_CONTEXT_UNRESOLVED` | #3449, #4252 |
| O3 | `agent context resolve --action implement --wp-id WP01` | `WORK_PACKAGE_UNRESOLVED` naming primary paths | #5026 |
| O4 | O3 while primary also holds a stale copy of the mission | **exit 0, success**. The WP file comes from primary and the workspace is a primary lane worktree that does not exist. | #5277 |
| O5 | `next --owned-checkout … --result success` at tasks→implement | Uncaught traceback. The run state is left stuck at `implement`. | #5277 |
| O6 | `agent mission finalize-tasks --owned-checkout` for an owned checkout located under `.worktrees/` | Refused as a "coordination worktree" **after** a partial, uncommitted write: a status event is appended and a WP file is rewritten. | #5277 |
| O7 | `agent mission create --owned-checkout` | Charter activation and templates are read from primary, not the owned checkout | #5277 |
| O8 | `next` on an owned mission whose coordination-workspace probe hits a racing or odd git common-dir | `commondir: Success` fatal | #4867 |

## User Scenarios & Testing *(mandatory)*

The actor in every story is an **operator or agent harness** that created a mission with `--owned-checkout <path>` in a linked checkout it owns. The primary checkout may be on another branch or running another mission.

### User Story 1 - Plan an owned mission (Priority: P1)

After creating a mission in its owned checkout and committing a substantive spec, the operator checks status and sets up the plan. Both operate on the owned mission. Nothing is read from or written to primary.

**Why this priority**: Without it, an owned mission cannot pass the specify stage (#3449, #4252). This is the core promise of `--owned-checkout`.

**Independent Test**: Use a real primary plus a linked owned worktree. Run owned create, then spec commit, then `agent tasks status --owned-checkout P` and `agent mission setup-plan --owned-checkout P`. Take a byte snapshot of the primary checkout before and after.

**Acceptance Scenarios**:

1. **Given** an owned mission committed in checkout P, **When** the operator runs `agent tasks status --owned-checkout P --mission H`, where H is the slug, mid8 or mission id, **Then** it exits 0, reports the owned mission's work packages, and the primary checkout is byte-identical before and after.
2. **Given** the same mission with a substantive committed spec, **When** the operator runs `agent mission setup-plan --owned-checkout P --mission H`, **Then** the plan scaffold is created and committed in P, and primary is unchanged.
3. **Given** a path that fails ownership validation (not a registered linked worktree, a protected or mismatched branch, or a non-single_branch topology), **When** any of these commands is given `--owned-checkout` with that path, **Then** it refuses with a typed error naming the failed check and writes nothing anywhere.

---

### User Story 2 - Resolve work-package context in an owned mission (Priority: P1)

After tasks are finalized, the operator (or harness) resolves the action context for a work package. The WP file, its workspace and its metadata all come from the owned checkout.

**Why this priority**: Implementation and review prompts depend on this (#5026). Today the fail-open variant (O4) can silently point an agent at the wrong mission copy.

**Independent Test**: Finalize tasks in P, then run `agent context resolve --owned-checkout P --action implement --wp-id WP01`. Repeat with a stale copy of the mission placed in primary.

**Acceptance Scenarios**:

1. **Given** finalized tasks in P, **When** WP-level context is resolved with `--owned-checkout P`, **Then** the WP file and workspace path are under P, and the resolution kind identifies an owned checkout, not a lane worktree.
2. **Given** a work package absent from P, **When** it is resolved, **Then** the result is a typed "unresolved" outcome scoped to P. It never falls back to primary.
3. **Given** primary also holds a stale copy of the same mission, **When** context is resolved for the owned mission, **Then** every returned path is under P, and a warning names the stale primary copy.

---

### User Story 3 - Drive an owned mission with `next` (Priority: P1)

The operator advances an owned mission with `next --owned-checkout P` from discovery through the tasks→implement boundary and into review. Every decision, prompt, composition policy, task-board lookup and review-base selection uses the owned checkout.

**Why this priority**: `next` is the canonical control loop. Today it crashes at tasks→implement and leaves the run stuck (O5). When primary holds a stale copy, it takes primary's lane routing (O4 variant, #5277 item 4).

**Independent Test**: Run `next --owned-checkout P` across tasks→implement and into review. Repeat with a stale primary copy.

**Acceptance Scenarios**:

1. **Given** an owned mission at the tasks→implement boundary, **When** `next --owned-checkout P --result success` runs, **Then** it returns a structured step or blocked decision (exit 0). It never produces an uncaught traceback, and the run state is never left stuck.
2. **Given** the owned mission's composition policy differs from primary's, **When** `next` dispatches composition, **Then** the owned checkout's policy is applied.
3. **Given** primary holds a stale copy of the mission, **When** `next` resolves the task board for the owned mission, **Then** the workspace is the owned checkout, never a primary lane worktree, and the stale-copy warning is emitted.
4. **Given** a work package in review, **When** the review prompt is built, **Then** the review base is a deterministic claim commit in P, and the diff is scoped to that work package's changes. If no such base can be established, the command fails closed with a typed reason and never emits an unscoped whole-checkout diff.
5. **Given** the git common-dir probe for the owned mission errors or races (#4867), **When** `next` runs, **Then** it reports a typed, actionable error or resolves correctly. It never surfaces a `commondir: Success` fatal.

---

### User Story 4 - Finalize tasks atomically in any owned location (Priority: P2)

The operator's owned checkout may live anywhere, including under the repository's `.worktrees/` directory. `finalize-tasks --owned-checkout` succeeds there, and any refusal leaves the checkout clean.

**Why this priority**: Checkouts under `.worktrees/` are common. A refusal that half-writes status events and WP files corrupts the mission's authoritative log (O6).

**Independent Test**: Place the owned checkout at `<primary>/.worktrees/<name>`, then finalize. Separately, force a refusal and check `git status` in P.

**Acceptance Scenarios**:

1. **Given** a valid owned single_branch checkout under `.worktrees/`, **When** `finalize-tasks --owned-checkout P` runs, **Then** it exits 0 and status reads target P.
2. **Given** any refusal during finalize, **When** it returns, **Then** `git status` in P shows no new or modified files and the status event log is unchanged.
3. **Given** a real coordination worktree (`-coord`) or a lanes-with-coordination mission, **When** a primary-labelled status read targets it, **Then** the existing refusal still applies. Coordination protections are unchanged.

---

### User Story 5 - Governance reads at creation follow the owned checkout (Priority: P2)

When a mission is created with `--owned-checkout P`, charter activation, mission-type context and the specification template are read from P. This matches what `next` already reads. Editing the charter remains possible only from primary.

**Why this priority**: Today an activated owned checkout is refused when primary is inactive, and an inactive owned checkout is accepted when primary is active (O7).

**Independent Test**: Compare activation in P against primary. Empty it in one and keep it in the other, then run owned create.

**Acceptance Scenarios**:

1. **Given** P's charter activates the mission type and primary's does not, **When** owned create runs, **Then** it succeeds.
2. **Given** P's charter does not activate the mission type and primary's does, **When** owned create runs, **Then** it refuses with the existing inactive-charter error.
3. **Given** any charter-authoring command is run against an owned checkout, **Then** the existing primary-only authoring refusal (#4785) still applies.

---

### User Story 6 - Clear refusals where owned missions are not supported (Priority: P3)

`implement` and `action review` do not support owned single_branch missions in this mission; full support belongs to #5100. When invoked for such a mission, they refuse with a typed error that names the supported path, instead of failing to find the mission or routing to a lane.

**Why this priority**: This turns confusing "mission not found" failures into actionable guidance, without widening scope into lane allocation.

**Independent Test**: Run `agent action implement WP01` and `agent action review WP01` for an owned mission.

**Acceptance Scenarios**:

1. **Given** an owned single_branch mission, **When** `implement` or `action review` targets it, **Then** it exits non-zero with a typed refusal code and a message that points to the supported command path. No lane worktree or branch is created, and nothing is written to primary.

### Edge Cases

- The same mission handle resolves in both P and primary. P wins because ownership was validated, and a warning is emitted (decision `01M3M2ZXH4CGY20X1PZEC18F54`).
- The owned checkout is under `.worktrees/` but is not a registered coordination worktree. It is treated as the owned checkout, never as coordination.
- A symlinked or case-variant path for P is passed. Identity is decided on the canonical path.
- P's branch changes between validation and a later read within the same command. Reads use the validated fact from the start of the command. They do not re-derive ownership per read.
- The `--owned-checkout` path is missing, not a git worktree, or points at primary itself. The command refuses with a typed error.
- The mission has a coordination branch or a non-single_branch topology. The owned path is refused, as today (ADR 2026-09-03-1).
- The same mission slug exists in two different owned checkouts. Caches and lookups are isolated per checkout.

## Requirements *(mandatory)*

### Functional Requirements

| ID | Title | User Story | Priority | Status | Delivery | No-op passable? |
|----|-------|------------|----------|--------|----------|-----------------|
| FR-001 | Single validated ownership fact | As an operator, I want ownership of an `--owned-checkout` path validated exactly once per command, through the canonical ownership resolver, and handed to every downstream read as a validated fact, so that no read re-derives or bypasses ownership. | High | Open | [build] | no — an architectural check fails if any second producer of the fact exists, paired with a positive control of the one canonical producer |
| FR-002 | Owned status listing | As an operator, I want `agent tasks status` to accept `--owned-checkout` and list the owned mission by any handle form, so that I can see its state (US1-AS1, #3449). | High | Open | [build] | no — red on main with MISSION_NOT_FOUND |
| FR-003 | Owned plan setup | As an operator, I want `agent mission setup-plan` to accept `--owned-checkout` and create and commit the plan in the owned checkout, so that planning proceeds (US1-AS2, #4252). | High | Open | [build] | no — red on main with PLAN_CONTEXT_UNRESOLVED |
| FR-004 | Owned WP context resolution | As an operator, I want `agent context resolve` to accept `--owned-checkout` and resolve the WP file, workspace and metadata from the owned checkout, returning a typed unresolved result scoped to it when the WP is missing, so that implement and review context is correct (US2, #5026). | High | Open | [build] | no — red on main with WORK_PACKAGE_UNRESOLVED |
| FR-005 | No implicit current-directory ownership | As an operator, I want context resolution to stop adopting the current directory as an owned checkout without validation, so that ownership has one source (ADR 2026-08-12-1). | High | Open | [build] | no — a test runs from an unvalidated linked checkout without the flag and asserts no owned adoption; the positive control uses the flag |
| FR-006 | Stale primary copy never wins | As an operator, I want every owned read to resolve from the owned checkout even when primary holds a copy of the same mission, with a warning naming the stale copy, so that nothing fails open (US2-AS3, US3-AS3). | High | Open | [build] | no — red on main (exit 0 with primary paths) |
| FR-007 | Owned `next` across tasks→implement | As an operator, I want `next --owned-checkout` to return a structured decision at every boundary, including tasks→implement, and never leave the run stuck, so that the control loop works (US3-AS1). | High | Open | [build] | no — red on main with a traceback |
| FR-008 | Owned composition and task-board authority | As an operator, I want `next` to apply the owned checkout's composition policy and resolve the task board and workspace from the owned checkout, so that runtime decisions match the owned mission (US3-AS2/AS3). | High | Open | [build] | no — red on main (primary policy; primary lane routing) |
| FR-009 | Owned review base | As a reviewer, I want the review prompt for an owned mission's work package to use a deterministic claim commit in the owned checkout and a WP-scoped diff, failing closed when no base exists, so that reviews see exactly the WP's change (US3-AS4). | Medium | Open | [build] | no — a test asserts that the scoped diff excludes a sibling WP's change and that the missing-base path fails closed |
| FR-010 | Owned-workspace resolution kind | As an operator, I want owned-workspace resolution to report a distinct owned-checkout kind with no lane identifier, so that tooling never treats an owned checkout as a lane worktree and the checkout-identity guard applies to it. | Medium | Open | [build] | no — a test asserts the kind and a null lane, with a lane-mission positive control |
| FR-011 | Common-dir probe robustness | As an operator, I want the owned-mission coordination-workspace probe to handle a missing or racing git common-dir with a typed, actionable result, so that `next` never surfaces `commondir: Success` (#4867, US3-AS5). | Medium | Open | [build] | no — red-first reproduction of the probe failure |
| FR-012 | Owned status source under `.worktrees/` | As an operator, I want status reads for an owned single_branch checkout located under `.worktrees/` to be accepted as owned reads, decided by the canonical surface resolver from the validated fact and registered-worktree facts rather than path shape, so that finalize works there (US4-AS1). | High | Open | [build] | no — red on main with "primary_checkout reads must not target coordination worktree paths" |
| FR-013 | Coordination protections unchanged | As a maintainer, I want primary-labelled reads of real coordination worktrees and lanes-with-coordination missions to keep being refused, so that the fix does not loosen coordination safety (US4-AS3). | High | Open | [ratchet] | yes — paired with FR-012 on the same fixture family |
| FR-014 | Atomic finalize | As an operator, I want `finalize-tasks` to validate everything before writing, so that a refusal leaves the checkout and status log unchanged (US4-AS2). | High | Open | [build] | no — red on main (partial write observed) |
| FR-015 | Governance reads at owned create | As an operator, I want owned mission creation to read charter activation, mission-type context and the spec template from the owned checkout, so that governance matches `next` (US5-AS1/AS2). | Medium | Open | [build] | no — red on main in both directions |
| FR-016 | Charter authoring stays primary-only | As a maintainer, I want charter authoring from an owned checkout to remain refused, so that there is one charter authority (US5-AS3, #4785). | High | Open | [ratchet] | yes — paired with FR-015 on the same fixture |
| FR-017 | Typed refusal for unsupported commands | As an operator, I want `implement` and `action review` to refuse owned single_branch missions with a typed code and guidance, and write nothing, so that failures are actionable (US6). | Low | Open | [build] | no — red on main (MISSION_NOT_FOUND instead of a typed refusal) |
| FR-018 | Per-checkout cache isolation | As an operator, I want WP metadata lookups to be isolated per checkout, so that the same mission slug in primary and in an owned checkout (or in two owned checkouts) never collide. | Medium | Open | [build] | no — a same-slug fixture in two checkouts must return distinct results |
| FR-019 | Invalid owned path refused | As an operator, I want every command that accepts `--owned-checkout` to refuse an invalid path (missing, not a linked worktree, primary itself, wrong or protected branch, non-single_branch topology) with a typed error and no writes (US1-AS3). | High | Open | [ratchet] | yes — each invalid case is paired with the valid-path positive control |

### Non-Functional Requirements

| ID | Title | Requirement | Category | Priority | Status |
|----|-------|-------------|----------|----------|--------|
| NFR-001 | No primary writes | In every owned-mission acceptance scenario, the primary checkout's tracked and untracked files are byte-identical before and after the command (0 differences). | Reliability | High | Open |
| NFR-002 | Validation cost | Ownership validation runs at most once per command invocation. Owned status reads add no per-read git subprocess calls beyond the non-owned path (measured by counting subprocess invocations in a test). | Performance | Medium | Open |
| NFR-003 | Command latency | Owned-mode `agent tasks status`, `setup-plan` and `context resolve` complete within the charter's < 2 s budget on the acceptance fixture. | Performance | Medium | Open |
| NFR-004 | Typed errors | 100% of owned-mode refusals introduced or touched by this mission surface a stable, documented error code in `--json` output. None surface as an uncaught exception. | Reliability | High | Open |
| NFR-005 | Quality gates | New and changed code passes ruff (check + format) and mypy with zero new findings, keeps function complexity ≤ 15, and meets the ≥ 90% diff-coverage gate. | Maintainability | High | Open |
| NFR-006 | Cross-platform identity | Owned-checkout identity comparison is correct on Linux, macOS (`/private` symlinked temp dirs) and Windows (case-insensitive paths). It is covered by at least one symlink case and one case-variant case. | Reliability | Medium | Open |

### Constraints

| ID | Title | Constraint | Category | Priority | Status |
|----|-------|------------|----------|----------|--------|
| C-001 | Ownership authority | Ownership is decided only by the existing canonical ownership resolver (ADR 2026-08-12-1). There is no inference from the current directory, the environment, or the nearest project directory. | Technical | High | Open |
| C-002 | One status-source authority | Status-source classification stays with the canonical coordination surface resolver introduced by #4959. No parallel status-read source or label is added. | Technical | High | Open |
| C-003 | Layer rules | No new outbound import edges from the runtime or mission-runtime layers into the CLI application layer. The shrink-only ledgers must not grow. | Technical | High | Open |
| C-004 | Scope boundary | Out of scope: charter authoring from owned checkouts and hosted binding (#4250); single_branch lane allocation in `implement` (#5100); the one-writer-per-checkout rule (#5099); create-from-worktree ergonomics (#3443). | Business | High | Open |
| C-005 | ADR amendment | ADR 2026-09-03-1 is amended to replace its accepted "thread the owned root through every reader" trade-off with the validated-fact design. It lands with this mission. | Technical | High | Open |
| C-006 | Prior-art provenance | Reusable commits from PR #5009 are carried with `cherry-pick -x`, keeping @samuelgoff's authorship. Re-expressed fixes credit him with `Co-authored-by`. Session-log documentation from #5009 is not carried. | Business | Medium | Open |
| C-007 | Red-first | Every `[build]` FR has a reproduction test through the pre-existing entry point that is red on the planning base before its fix lands. | Technical | High | Open |
| C-008 | Terminology | Use "Mission", "owned checkout", "primary checkout" and "coordination worktree" per the glossary. No `feature` aliases. | Business | Medium | Open |

### Key Entities

- **Owned checkout**: A linked checkout that an operator explicitly declared and that passed ownership validation for one single_branch mission. It knows both its own root and the primary checkout it belongs to.
- **Validated ownership fact**: The single proof, produced once per command, that a given checkout owns a given mission. All owned reads consume it. It carries the primary root (for primary-only concerns such as charter authoring) and the owned root (for everything else).
- **Primary checkout**: The repository root checkout. It remains the authority for charter authoring and for missions that are not owned.
- **Stale primary copy**: A copy of an owned mission's artifacts that also exists in primary. It is never authoritative for owned reads.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: An operator can take an owned mission from create through spec, plan, tasks and finalize, and reach the first implement decision, using only documented `--owned-checkout` commands, with 0 errors and 0 bytes changed in primary. — [build] · no-op passable: no
- **SC-002**: All 8 observed defects (O1–O8) have a reproduction that fails on the planning base and passes at mission end. — [build] · no-op passable: no
- **SC-003**: 0 owned-mode commands report success while returning a path outside the owned checkout, including with a stale primary copy present. — [build] · no-op passable: no
- **SC-004**: Exactly 1 code path produces the validated ownership fact, enforced by an automated architectural check. — [build] · no-op passable: no
- **SC-005**: Existing coordination and lane-topology status and charter-authoring protections pass unchanged (0 regressions in their existing test files). — [ratchet] · no-op passable: yes

## Assumptions

- Owned missions remain single_branch with no coordination branch, as ADR 2026-09-03-1 requires. Supporting other topologies is not a goal.
- `next` handle resolution and owned finalize outside `.worktrees/` already work on `main`. They are covered here only as regression guards.
- The stale-primary-copy warning is emitted on stderr (human) and as a structured warning field in `--json` output. Exact wording is decided at plan time.
- #5009's regression tests are a valid starting point. They are adapted to the real command call shape, where the owned root is passed through the validated fact.

## Dependencies

- #4959 (merged): the canonical coordination surface resolver that FR-012 extends.
- #4980 (merged): the coordination-aware task-board authority that FR-008 extends.
- #5259 (merged): canonical WP task placement in prompt building that FR-008 and FR-009 build on.
- #4785 / PR #4790 (merged): the primary-only charter authoring guard kept by FR-016.
