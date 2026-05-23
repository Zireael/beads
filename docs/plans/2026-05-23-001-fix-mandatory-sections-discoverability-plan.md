---
title: "Fix mandatory section discoverability and completeness for all bead types"
type: fix
status: draft
created: 2026-05-23
---

## Problem Frame

An agent (or human) relying only on `bd` CLI help output cannot discover what mandatory sections are required for each bead type. The `bd lint --help`, `bd lint --type`, `bd create --help --type`, and `docs/CLI_REFERENCE.md` all omit types that have validated mandatory sections (decision, spike, story). The only place the complete mapping exists is in source code (`internal/types/types.go`) or in validation error messages after a failed creation attempt.

Additionally, the reference document `references/bead-checklists.md` in the beads skill has factual discrepancies with the code: it lists "Chosen option" for decision (code requires "Rationale"), and lists chore as requiring acceptance criteria (code says none).

## Scope

**In scope:**
- Fix CLI help text for `bd lint --help` (Long) to list all types with mandatory sections
- Fix `bd lint --type` flag description to list all valid filter values
- Fix `bd create --help --type` help text to list all valid types
- Fix `docs/CLI_REFERENCE.md` section requirements table to match code
- Fix `references/bead-checklists.md` to match code's `RequiredSections()`
- Update `bd lint --type` flag validation to accept all valid types (already works at the type level but flag help doesn't reflect it)
- Add a `bd types --sections` flag to display the mandatory section requirements mapping from code
- Update `CLI_REFERENCE.md` generated docs to match

**Out of scope:**
- Changing `RequiredSections()` logic or adding/removing mandatory sections (the plan fixes documentation, not validation rules)
- Adding `--validate` as default behavior on create (that's a separate behavioral change)
- Fixing server-mode Dolt init issues (separate concern)
- Adding built-in templates for decision/spike/story (nice-to-have, not a correctness issue)

## Research Summary

Source of truth: `internal/types/types.go` L617-L647, `RequiredSections()` method.

| Type | Required sections | Enforced via |
|------|------------------|-------------|
| bug | Steps to Reproduce, Acceptance Criteria | validation + lint |
| task | Acceptance Criteria | validation + lint |
| feature | Acceptance Criteria | validation + lint |
| epic | Success Criteria | validation + lint |
| decision | Decision, Rationale, Alternatives Considered | validation + lint |
| spike | Goal, Findings | validation + lint |
| story | Acceptance Criteria | validation + lint |
| chore | (none) | lint (shows "none") |
| milestone, message, molecule, gate, event, custom | (none) | — |

Gaps in CLI docs:
- `lint.go:26-35` (Long help) — missing decision, spike, story sections
- `lint.go:164` (--type flag) — lists only 4 types
- `create.go:836` (--type flag) — lists 6 types, missing spike, story, milestone
- `docs/CLI_REFERENCE.md:1624-1629` — same omissions as lint.go

Gaps in external reference:
- `references/bead-checklists.md` says decision needs "Chosen option" (code: "Rationale")
- `references/bead-checklists.md` says chore needs acceptance criteria (code: none)

Both validation error messages (create.go L259-272) and lint runtime output correctly use the code's section requirements. Only help text and docs are wrong.

## Implementation Units

### U1. Fix `bd lint` help text (Long section)

**Goal:** `bd lint --help` accurately lists all types that have mandatory sections.

**Files:**
- `cmd/bd/lint.go` (L26-35 — Long string)

**Approach:** Replace the hardcoded Long string's section-requirements table to include all types from `RequiredSections()`. The table should list bug, task, feature, epic, decision, spike, story and indicate chore/milestone/etc. have none.

**Test scenarios:**
- `bd lint --help` output includes decision, spike, and story sections
- Output still shows bug/task/feature/epic sections correctly
- Output shows chore as "(none)"

### U2. Fix `bd lint --type` flag description

**Goal:** `-t, --type` help text lists all valid filter values.

**Files:**
- `cmd/bd/lint.go` (L164 — flag definition)

**Approach:** Update the flag description from `"Filter by issue type (bug, task, feature, epic)"` to enumerate all types with mandatory sections.

**Test scenarios:**
- `bd lint --help` shows all valid types in the `--type` flag description
- `bd lint --type decision` works (already does, just undocumented)
- `bd lint --type spike` works (already does, just undocumented)

### U3. Fix `bd create --type` flag description

**Goal:** `bd create --help` shows all valid types.

**Files:**
- `cmd/bd/create.go` (L836 — flag definition)

**Approach:** Update the `--type` flag description to include spike, story, and milestone in the documented list. Currently says `"Issue type (bug|feature|task|epic|chore|decision)"`.

**Test scenarios:**
- `bd create --help` shows all 9 standard types
- `bd create --type spike --validate` works and validates correctly
- `bd create --type story --validate` works and validates correctly
- `bd create --type milestone` works

### U4. Add `bd types --sections` flag

**Goal:** A discoverable command to print the required sections per type.

**Files:**
- `cmd/bd/types.go` (add `--sections` flag and handler)

**Approach:** Add a `--sections` flag to the existing `bd types` command. When used, instead of listing type descriptions, print:
```
Required sections by type:
  bug:      ## Steps to Reproduce, ## Acceptance Criteria
  task:     ## Acceptance Criteria
  feature:  ## Acceptance Criteria
  epic:     ## Success Criteria
  decision: ## Decision, ## Rationale, ## Alternatives Considered
  spike:    ## Goal, ## Findings
  story:    ## Acceptance Criteria
  chore:    (none)
  milestone: (none)
  message:  (none)
  molecule: (none)
  gate:     (none)
  event:    (none)
```

Use `RequiredSections()` from the types package to avoid duplication. The `types.IssueType` has a `RequiredSections()` method that returns `[]RequiredSection` — iterate `coreWorkTypes` and call it.

Also support `--json` output format.

**Test scenarios:**
- `bd types --sections` prints all types with their required sections
- `bd types --sections --json` outputs valid JSON
- `bd types --sections` matches the output of `RequiredSections()` for each type
- No sections shown for chore, milestone, message, molecule, gate

### U5. Fix `docs/CLI_REFERENCE.md`

**Goal:** The generated CLI reference doc matches the updated help text.

**Files:**
- `docs/CLI_REFERENCE.md` (L1624-1629 and L498)

**Approach:** Update the section-requirements table in L1624-1629 and the `--validate` flag description to include all types. This file is ostensibly auto-generated — check if there's a generation script and regenerate, otherwise edit manually.

**Test scenarios:**
- CLI_REFERENCE.md section-requirements table includes decision, spike, story
- Matches the lint help text after U1
- `--validate` flag description references section requirements

### U6. Fix external bead-checklists.md

**Goal:** The skills reference matches code.

**Files:**
- `C:\Users\zir\.config\opencode\skills\beads\references\bead-checklists.md` (L9 — decision sections, L10 — chore)

**Approach:**
- Change "Chosen option" to "Rationale" for decision type
- Change chore from listing "Acceptance criteria" to "(none)"

**Test scenarios:**
- bead-checklists.md decision row lists: Decision, Rationale, Alternatives Considered
- bead-checklists.md chore row says "(none)" or omits it

## Dependencies
- U1, U2, U3, U5 are independent — can be done in parallel
- U4 is independent
- U6 is independent

## Key Technical Decisions

1. **String changes only for U1-U3, U5, U6** — these are help text and doc edits with no behavioral impact.
2. **`bd types --sections` uses the code's `RequiredSections()` method** — this keeps the single source of truth and auto-syncs if sections change. The output format mirrors `bd lint --help` styling for consistency.
3. **`--sections` on `types` rather than a new command** — avoids adding another top-level command. `bd types` already lists types; `--sections` adds the natural next level of detail.
4. **Not auto-triggering `--validate` on create** — that would be a breaking behavioral change. The plan fixes discoverability, not defaults.

## Deferred to Implementation

- Whether `docs/CLI_REFERENCE.md` has a regeneration script or needs manual editing. Check `scripts/` for doc generation.
- Whether there are test files asserting the old help text strings that need updating.

## Verification

- `bd lint --help` output matches the `RequiredSections()` table from `internal/types/types.go`
- `bd create --help` `--type` flag shows all 9 standard types
- `bd types --sections` output matches `RequiredSections()` for every type
- All existing tests pass (help text string updates will break test assertions — update them)
- `go vet ./cmd/bd/...` passes
