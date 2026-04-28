---
description: Validate a PRD (file, Confluence page, or pasted markdown) for developer readiness—structure, testability, traceability, and repo alignment.
---

## Usage

```
/sp.prd-validate path/to/prd.md
/sp.prd-validate --confluence <pageId>
/sp.prd-validate
```

- With **no path**: use `$ARGUMENTS` if it looks like a path or page id; otherwise ask once for **file path** or **Confluence page id**.
- **`--confluence` / `-c`**: `getConfluencePage` (markdown body); validate that content (strip agent-only blocks if present).

## User input

```text
$ARGUMENTS
```

Honor non-empty `$ARGUMENTS` for path, flags, or pasted excerpt (if the user pastes a full PRD inline, validate that).

## Purpose

Act as a **PRD quality gate for implementation**: the doc should let a developer start **`/sp.specify`** / **`/sp.backlog`** / **`/sp.pdlc`** without chasing the PO for missing intent, scope, or acceptance tests.

This is **not** a full legal/compliance review; it is **developer pickup readiness**.

## Grounding (before scoring)

1. Read **`.specify/templates/prd-template.md`** — the canonical section list and table shapes.
2. **Verify implementation claims with `context-stack`** (see [`.cursor/rules/context-stack.md`](../rules/context-stack.md)). For every service / RPC / endpoint / file path / table the PRD names:
   - `search_code("<claim>")` → **Pass** if found in code, **Fail (invented)** if not, **Partial (stale)** if only outdated references appear.
   - `get_dependencies("<symbol>")` for any symbol the PRD says will be modified — used to score **Section 15 Implementation Map**.
3. **Verify cross-document consistency with `context-stack`**:
   - `search_specs("<domain or capability>")` → confirms the PRD aligns with master-spec / sibling PRDs; flags duplication or contradiction.
   - `search_docs("<topic> ADR OR runbook")` → flags PRD claims that contradict an existing ADR or runbook.
4. Optionally skim **`.specify/memory/master-spec.md`** and **`.specify/memory/constitution.md`** when the PRD references capabilities, SDD gates, or cross-service behavior — `search_specs` already surfaces these but a direct read is fine for short files.

When a `context-stack` lookup contradicts the PRD, record the citation (path or URL) in the **Blockers** or **Improvements** bullet so the PO can act on it.

## Validation dimensions

Score each dimension **Pass / Partial / Fail** and give **evidence** (quote or section reference). Use **Section 1–15** names from the template (not shorthand symbols in user-visible summaries).

| Dimension | What “good” means for developers |
|-----------|-----------------------------------|
| **Structure** | All template sections present with real content (not placeholders). Summary block filled; agent-note boilerplate may be ignored for Confluence drafts. |
| **Intent & scope** | Objective, in/out scope, and non-goals are concrete enough to reject random feature creep. One primary journey per PRD; no unrelated bundled epics. |
| **Requirements** | Section 7 rows are **testable** (“shall” with observable behavior). Every **P0** in the summary appears as a **P0** requirement. Priorities consistent. |
| **Acceptance criteria** | Section 10: **Given / When / Then** filled (not empty cells). Each **P0** requirement has at least one linked or implied AC. |
| **Stories** | Section 11: stories map to PRs and AC columns (no orphan stories). |
| **UX / NFRs** | Section 6–8 and 12: if UI — journeys and UX table meaningful; if backend-only — explicitly **N/A** where template allows. NFRs not blank when latency, security, or reliability matter. |
| **Dependencies & risks** | Section 13: dependencies name **what** and **why**; risks have mitigation. **`[DECISION NEEDED]`**: **at most 3** open decisions; each has options and owner (per `/sp.prd` guidance). |
| **Implementation hints** | Section 15 appendix: implementation map lists **credible** paths or services; **MUST** match `context-stack` `search_code` hits when checked (or be tagged **NEW** with rationale). |
| **Traceability** | Metadata (section 1): BRD / Jira / Figma / related PRDs present when the org expects them; capability / master-spec id if required by team process. |

## Automatic red flags (usually Fail)

- Empty **Given/When/Then** for most AC rows.
- **P0** in summary with no matching P0 requirement or AC.
- Scope section empty or only buzzwords (“improve UX”, “better performance”) without measurable intent.
- More than **three** `[DECISION NEEDED]` items without prioritization.
- Contradictions (e.g. out-of-scope item repeated as a requirement).
- **Wrong bounded context** (e.g. order feature described but identity-only systems listed) — call out even if structure is complete.

## Output format (mandatory)

Produce a short report the user can paste into Confluence or a PR:

1. **Verdict**: **Ready** / **Ready with gaps** / **Not ready** (one line rationale).
2. **Scorecard table**: Dimension | Pass/Partial/Fail | Notes (1 line each).
3. **Blockers** (must fix before dev pickup): bulleted, actionable.
4. **Improvements** (should fix): bulleted.
5. **Suggested next command**: e.g. edit PRD → `/sp.backlog --confluence <id>`, `/sp.clarify`, or `/sp.specify` when on a feature branch.

Keep the report **compact** (target under ~40 lines). No full PRD rewrite unless the user asks.

## Related

**`/sp.prd`** (authoring) · **`.specify/templates/prd-template.md`** · **`/sp.backlog`** · **`/sp.clarify`**
