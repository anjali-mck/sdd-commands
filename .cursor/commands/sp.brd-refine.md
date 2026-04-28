---
description: Review a BRD for business completeness—gaps, stakeholders, outcomes, and readiness for downstream specs—without rewriting the full document unless asked.
handoffs:
  - label: Build specification
    agent: sp.specify
    prompt: I want to build the following feature. Use the refined BRD context from our last step.
    send: true
  - label: Clarify specification
    agent: sp.clarify
    prompt: Run clarification on the current feature spec before planning.
    send: true
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty). Use it for **Confluence page id**, **local file path**, **focus areas** (e.g. compliance, geography), or pasted excerpts.

## Usage

```text
/sp.brd-refine
/sp.brd-refine --confluence <pageId>
/sp.brd-refine path/to/brd.md
/sp.brd-refine --confluence <pageId> "focus on compliance and data residency"
```

| Flag / form | Meaning |
|-------------|---------|
| `--confluence` / `-c` | Load the BRD from Confluence (page id as accepted by your Confluence MCP/API); fetch markdown or equivalent body. |
| *(positional path)* | Local BRD file (markdown or text). |
| *(none)* | If `$ARGUMENTS` looks like a path or numeric page id, use it; otherwise **ask once** for file path or Confluence page id. |
| Trailing free text | **Narrows the review** (themes, audience, regulatory focus). |

## Outline

1. **Resolve source**  
   Parse flags and `$ARGUMENTS`. Require exactly one BRD source (Confluence page or local file) before scoring.

2. **Load BRD content**  
   Read the full BRD from disk or Confluence. Record a short **source label** (path or page URL) for the report. **Do not** paste the entire BRD into chat.

3. **Grounding (before scoring)**  
   - Read **`.specify/templates/brd-template.md`** if it exists — use it as the **rubric** (score coverage against its themes, not literal heading text if the BRD uses different labels). If the template is missing, use the **Refinement dimensions** table below as the rubric only.  
   - Optionally read **`.specify/memory/master-spec.md`** (or repo README / architecture docs) when the BRD names **specific systems, services, or repositories** — flag **stale, vague, or inconsistent** references.  
   - Optionally read **`.specify/memory/constitution.md`** when the BRD touches **cross-cutting security, quality gates, or engineering policy** — flag **misalignment** as a gap or risk.

4. **Persona & scope**  
   Act as a **senior product / business analyst** reviewing a **Business Requirements Document (BRD)**.  
   - **In scope:** Whether the BRD is **sufficient** for downstream **feature specifications**, backlog shaping, and stakeholder alignment; **actionable gaps** with evidence.  
   - **Out of scope:** Line-by-line copy editing, **full BRD rewrite** (unless the user asks), **PRD-level acceptance tests** (handle those in the spec/PRD workflow).  
   - **Out of scope:** Legal sign-off; surface **compliance gaps** as **risks** only.

5. **Score** each **Refinement dimension** as **Strong / Adequate / Weak / Missing** with **evidence** (short quote or section reference).

6. **Apply red-flag checks**  
   Treat the **Automatic red flags** list below as signals—usually **Weak** or **Missing** on the relevant dimensions.

7. **Emit mandatory report**  
   Produce a **compact** report (target **under ~55 lines**) suitable to paste into Confluence, Slack, or a PR comment. Follow **Output format** exactly.

8. **Suggested next steps**  
   Point to **commands that exist in this repo** (e.g. after BRD fixes: **`/sp.specify`**, then **`/sp.clarify`** before **`/sp.plan`**; **`/sp.backlog`** when the team tracks work in Jira from PRDs). Do **not** reference slash commands that are not present under `.cursor/commands/` unless the user has added them.

## Refinement dimensions

Score each dimension **Strong / Adequate / Weak / Missing** with **evidence** (quote or section reference).

| Dimension | What “good” means for a BRD |
|-----------|------------------------------|
| **Executive clarity** | Problem, opportunity, and “why now” are understandable without deep product context. |
| **Stakeholders & governance** | Sponsor, decision-makers, and affected groups are identifiable; escalation path is plausible. |
| **Outcomes & success** | Business goals and **measurable** success (KPIs or clear qualitative signals) are stated—not only features. |
| **Scope boundaries** | Domain-level **in / out** and phasing reduce ambiguity; non-goals explicit. |
| **Customers / segments** | Who benefits is clear enough to prioritize and to derive personas for specs/PRDs. |
| **Capability map** | Major capability themes or journeys exist at **business** granularity (suitable to split into specs or PRDs). |
| **Compliance & policy** | Regulatory, data, and brand constraints called out when relevant; **N/A** stated when truly irrelevant. |
| **Dependencies & risks** | Cross-domain, vendor, data, and org dependencies named; top risks have mitigation or owner. |
| **Traceability to delivery** | Path to specs/epics/backlog is clear (table, epic list, or unambiguous capability list). |
| **Spec / PRD readiness** | A PO could draft the next artifact **without inventing scope**; **no critical unknowns** block slicing. |

## Automatic red flags *(usually Weak or Missing)*

- Only **solution / tech** language with no **problem or outcome** narrative.
- **Success** is vague (“better UX”, “more revenue”) with **no** guardrails or measures.
- **Scope** missing or “everything in the domain” without phasing.
- **Stakeholders** absent—no owner, no customers, no operational actors.
- **Compliance**-sensitive domain (payments, identity, health, minors, etc.) with **no** dedicated treatment in the BRD.
- **Contradictions** (e.g. capability listed as in-scope and out-of-scope).
- **Orphan capabilities**: themes with no owner, metric, or delivery line of sight.

## Output format *(mandatory)*

1. **Verdict:** **BRD-complete** / **BRD-usable with gaps** / **Not BRD-ready** (one line rationale).
2. **Scorecard:** table — Dimension | Strong/Adequate/Weak/Missing | Notes (one line).
3. **Must-fix gaps** *(before treating the BRD as source of truth for specs):* bulleted, actionable; each with **suggested BRD section** (template section or theme name).
4. **Should-fix improvements:** bulleted.
5. **Suggested slices** *(if applicable):* 1–5 bullets—how to split into **specs or PRDs** without duplicating the domain.
6. **Suggested next command:** e.g. update BRD in Confluence → **`/sp.specify`**, **`/sp.clarify`**, **`/sp.plan`**, or **`/sp.backlog`** per team workflow.

**Do not** paste the full BRD into chat. **Do not** rewrite the entire BRD unless the user asks.

## Related

`/sp.specify` · `/sp.clarify` · `/sp.plan` · `/sp.backlog` · `.specify/templates/brd-template.md` · `.specify/memory/constitution.md`

---

As the main request completes, you MUST create and complete a PHR (Prompt History Record) using agent‑native tools when possible.

1) Determine Stage  
   - Stage: constitution | spec | plan | tasks | red | green | refactor | explainer | misc | general

2) Generate Title and Determine Routing:  
   - Generate Title: 3–7 words (slug for filename)  
   - Route is automatically determined by stage:  
     - `constitution` → `history/prompts/constitution/`  
     - Feature stages → `history/prompts/<feature-name>/` (spec, plan, tasks, red, green, refactor, explainer, misc)  
     - `general` → `history/prompts/general/`

3) Create and Fill PHR (Shell first; fallback agent‑native)  
   - Run: `.specify/scripts/bash/create-phr.sh --title "<title>" --stage <stage> [--feature <name>] --json`  
   - Open the file and fill remaining placeholders (YAML + body), embedding full PROMPT_TEXT (verbatim) and concise RESPONSE_TEXT.  
   - If the script fails:  
     - Read `.specify/templates/phr-template.prompt.md` (or `templates/…`)  
     - Allocate an ID; compute the output path based on stage from step 2; write the file  
     - Fill placeholders and embed full PROMPT_TEXT and concise RESPONSE_TEXT

4) Validate + report  
   - No unresolved placeholders; path under `history/prompts/` and matches stage; stage/title/date coherent; print ID + path + stage + title.  
   - On failure: warn, don't block. Skip only for `/sp.phr`.
