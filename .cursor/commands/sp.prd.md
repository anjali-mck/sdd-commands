---
description: Generate PRD(s) from BRD/brief/Jira/Confluence. One journey per PRD (bundle related mechanics). Compact body; one Confluence write MCP call per PRD. Feeds /sp.backlog and /sp.pdlc.
---

## Usage

```
/sp.prd "brief"
/sp.prd --jira PROJ-123
/sp.prd --confluence <pageId>
/sp.prd --confluence <pageId> --name "PRD-02-01"
/sp.prd --confluence <pageId> --single-prd
/sp.prd ... --full
```

| Flag | Meaning |
|------|---------|
| `--jira` / `--ticket` / `-j` | `getJiraIssue` |
| `--confluence` | `getConfluencePage` markdown |
| `--name` | First `PRD-BB-MM`; auto-increment siblings |
| `--single-prd` | Force **one** Confluence doc when the agent would otherwise split a single journey |
| `--full` | Longer narrative in sections where the template allows prose. Same section structure and style as the default; no compression at publish time. |

**Defaults:** A BRD may yield **K PRDs** only when slices are **independently shippable** or **ownership/context** differs. **One user journey** (e.g. the same list screen with **filter + search** together) maps to **one PRD**; do **not** split into separate PRDs per control unless the BRD explicitly separates scope. **`--single-prd`** forces one doc when the agent would otherwise over-split. The Confluence body always follows the active `prd-template.md` structure -- see **PRD shape and writing style** below.

## User Input

```text
$ARGUMENTS
```

Must honor non-empty `$ARGUMENTS`.

## Persona & token economy

Senior PO: WHAT/WHY, testable reqs, explicit rules, measurable success. **Do not** paste BRD walls into chat; **compact session summary** after first big fetch. **Links** over bodies in completion output.

## One PRD = one feature *(one journey, possibly multiple mechanics)*

| Rule | Requirement |
|------|-------------|
| Scope | One **primary user goal / journey** per PRD. Multiple **mechanics** on the same journey (e.g. **optional filter** and **optional search** on **My Orders**) stay **in the same PRD** -- one requirements table, one AC set, one publish. |
| BRD | Many **distinct** capabilities or domains map to many PRDs. Do **not** mint separate PRDs for each UI control when the BRD describes one incremental list experience. |
| Header | `Related PRDs:` + Confluence URLs when siblings exist. |
| Split if | Distinct journeys, different owning service/context, different shippable stories, or a BRD feature matrix that **explicitly** lists independent delivery slices. Length alone is **not** a reason to split. |
| One doc OK | Single cohesive journey (including filter+search-style bundles) or `--single-prd`. |

## PRD shape and writing style

**Goal:** One Confluence page that is **PO-complete and easy to read**. The body must look like a product document a stakeholder can pick up cold -- not a digest, not a table dump, not a glyph soup.

| Rule | Requirement |
|------|-------------|
| **Template is authoritative** | Follow `.specify/templates/prd-template.md` **exactly**: same section order, same section names, same prose-vs-table conventions. Do **not** invent a parallel section scheme, do **not** renumber, do **not** drop or merge sections without an explicit reason. |
| **Match the template's prose-to-table ratio** | If the template renders a section as narrative paragraphs (Problem Statement, User Persona, What It Does, How It Works), write **paragraphs** -- not bullet fragments and not tables. Use tables **only** in the sections where the template uses tables (Business Rules, UI, Edge Cases, NFRs, Dependencies, Files, Known Gaps, Success Indicators). |
| **Writing style** | Full sentences in plain English. The body should read like a product document a stakeholder can pick up cold. Use abbreviations only when they are **widely understood in software product writing** (e.g. API, UI, HTTP, gRPC, SLO, P0/P1/P2); spell out anything that is project-specific or that a non-engineer might not recognise (write "out of scope" instead of `OOS`, "version 1" instead of `v1` in narrative prose, etc.). Spell out boolean logic in prose ("any of the selected statuses"; "matches both the search term and the status filter"); do not substitute logic or math glyphs for words. Refer to sections by name or number ("see Section 7"); do not use the section-sign glyph. **ASCII-only in the body** -- avoid logic glyphs, math glyphs, smart quotes, smart dashes, non-breaking spaces, and any other characters outside the printable ASCII range; use the plain-text equivalents (`-`, `"..."`, `'`, `<=`, `->`, `~=`, `--`). Examples are illustrative, not exhaustive -- the rule is "ASCII only" everywhere except code identifiers in backticks. |
| **Size** | Soft cap: **12 to 20 KB UTF-8** for the markdown `body` (a normal PRD is several pages). Do **not** abbreviate sentences or strip vowels to fit. If a single section is genuinely bloated, tighten **that** section; if the whole feature is too large, the answer is to **split into multiple PRDs** (see **One PRD** rules below), not to compress. |
| **`--full`** | Same shape as default; just longer narrative where the template allows prose. Still ASCII-safe and still one section structure. |
| **Single MCP upload** | Use **exactly one** Confluence **write** MCP call per PRD delivered in the run: **`createConfluencePage`** (new page) **or** **`updateConfluencePage`** (replace body on an existing page the user pointed at or a duplicate title match). Do **not** chain create + update to "finish" the same PRD; **not** multiple updates; prepare the **final** markdown **before** that single call. Retries after tool/network failure are OK (same intent, one successful write). |
| **Publish** | **New:** **`createConfluencePage`** with the full body (no H1 duplicate of title), correct `parentId`, `contentFormat: markdown`. **Existing page refresh / user-supplied id:** **`updateConfluencePage`** with the same constraints. Do **not** use update as a second step after create for the same content in one run. |
| **Split vs shorten** | Prefer **one PRD** when the BRD describes one buyer journey (e.g. filter + search on the same list). **Split** when capabilities are independently shippable or ownership differs -- never to fit a size cap. |

## Storage

- **SoT:** Confluence (`createConfluencePage` / `updateConfluencePage`). No new `docs/prds/**` or `docs/brds/**`.
- **Drafts:** `docs/.specify-staging/prd|brd/*.md` (gitignored). **Delete** after successful publish; **WARN** + keep if publish skipped/failed.
- Titles: `<PRD-ID>: <Feature>`, `BRD-NN: <Domain>`.

## Inputs

Combine `--jira` and `--confluence` is allowed. Fetch before drafting. No fetch and no usable text is an **ERROR**. Body structure always comes from the active `.specify/templates/prd-template.md` (do not crawl Confluence for layout).

## Grounding (`context-stack` MCP)

Match **repo reality** and **org reality** via the **`context-stack`** MCP when it is available -- see [`.cursor/rules/context-stack.md`](../rules/context-stack.md). Use it lightly: a couple of targeted lookups before drafting, not a survey. Never assume infra; missing auth/DB/RPC is called out as a prerequisite in the **Dependencies** section or **NEW** in the **Files** section of the PRD template.

| When | Tool (server: `context-stack`) |
|------|--------------------------------|
| Codebase orientation before drafting | `get_context("how does <domain> work today; key services + flows")` |
| Verify a specific symbol / RPC / file path the PRD will name | `search_code("<symbol or RPC name>")` |
| Find sibling PRDs/BRDs to cross-link in the header | `search_specs("<domain keywords>")` + `search_docs("PRD OR BRD <domain>")` |
| Master-spec capability bucket | `search_specs("master-spec <capability keyword>")` |

**Conflict rule (code wins over docs):** If a Confluence/BRD/Jira source describes a service, RPC, schema, or behavior that disagrees with what `search_code` / `get_context` shows in the actual codebase, **the code is the source of truth for "what is"**. Use the doc only for "what is intended". Surface the gap in the PRD's **Known Gaps** section (or the equivalent section in the active `prd-template.md`) with both citations -- do not silently parrot the stale doc.

**Citation discipline:** Reference real paths inline only when needed; do **not** include `context-stack` result lists in the PRD body. The PRD reads as a product document, not a search transcript.

## Outline

1. **Parse/fetch/merge** -- `getJiraIssue`, `getConfluencePage`, free text combine into one source doc.

2. **Repo** -- Read master-spec, constitution; when the `context-stack` MCP is available, run **one** `get_context("<domain> services + flows")` to orient (and `search_code` only if a specific path/symbol needs verification). Checklist: services, auth, persistence, gRPC/HTTP, frontend patterns. PRD must align -- and where doc and code disagree, **code wins** (see Grounding above).
   - **Bounded context (before drafting):** if the source is short, generic, or vague (e.g. "sort", "list"), or covers multiple domains, **AskQuestion** first. No silent domain inference from code layout.

3. **Confluence (narrow)** -- One `getConfluencePage` per id; **no** space-wide BRD crawl. BRD plus at most one canonical follow-up. Siblings: narrow CQL `PRD-<BB>-*` or metadata only; `getConfluencePage` only to resolve conflicts.

   **Publish parent (PRD folder -- mandatory)**  
   New PRDs MUST live under the **domain's PRD folder** in Confluence (the parent of pages like `PRD-02-01`, `PRD-03-01`, etc.). They MUST **NOT** be created under the **BRD library folder** (the parent shared by `BRD-01`, `BRD-02`, `BRD-07`, ...). Those two folders are different: a BRD's `parentId` from `getConfluencePage` is usually the BRD tree -- **do not** reuse it as `parentId` for `createConfluencePage` on a PRD.

   **How to resolve `parentId` for publish (in order):**
   1. From the source BRD, read **domain** (e.g. *Order Management*, *Identity & Authentication*).
   2. Find **any existing PRD** in that domain in the same space (`getPagesInConfluenceSpace` / title prefix `PRD-` + domain keyword, or narrow CQL). Open it; set **`parentId` = that PRD's `parentId`** (the domain PRD folder).
   3. If no PRD exists yet: locate the space's **PRD** hierarchy and the **folder** for that domain (title aligned with domain), via space navigation or admin -- use that folder's id. If it cannot be found, **stop** and ask the user for the correct PRD folder page id (do not fall back to the BRD folder).
   4. **Validate:** After create, the new PRD's `parentId` must equal a known domain PRD sibling's `parentId`, not the BRD page's `parentId`.

4. **IDs** -- `PRD-BB-MM`; K slices unless `--single-prd`; numbering from **Confluence** narrow search, not `docs/prds/`. No BRD: `PRD-<NN>-<MM>` from Confluence titles.

5. **BRD** -- If one exists, reference it in the header. Otherwise create one in Confluence (concise domain doc); `spaceId` FKPOC `490405892`; delete the staging BRD draft after create.

6. **Questions** -- Max 5; priority: bounded context, personas, success, scope, deps, constraints. Skip only if the source is substantive, the domain is explicit, and there is no alternate domain candidate.

7. **Template** -- Read `prd-template.md`.

8. **Draft** -- Render every section that the active `.specify/templates/prd-template.md` defines, in the order and style that template prescribes (narrative where it uses paragraphs; tables only where it uses tables -- see **PRD shape and writing style** above). Use full sentences, ASCII-safe punctuation, no logic glyphs, no section-sign glyph, no aggressive abbreviation. Strip sibling-scope items into the **Related PRDs** header. The "How It Works" / files / implementation sections name **real** paths and RPCs; anything that does not exist yet is tagged **NEW**. `--full` simply allows longer prose in the same sections.

9. **Validate** -- Run the gates below before staging. Iterate up to 3 fix passes. If a gate still fails after 3 passes, **stop and surface the unresolved items to the user** rather than papering over them or splitting the PRD as a workaround.

   **Structural gate**
   - Bounded context confirmed (if 2f applied), header populated, no invented infrastructure.
   - At most 3 `[DECISION NEEDED]` items.
   - One journey per PRD (mechanics bundled per the **One PRD** rules); extra slices only with `--single-prd` or a true multi-capability BRD.
   - Every section that the active `prd-template.md` defines is present, in the right order, with the right name.

   **Writing-style gate**
   - Narrative sections contain paragraphs, not tables or bullet fragments; tables appear only in sections where the template uses tables.
   - Body is ASCII-only (no logic, math, or section glyphs; no smart quotes/dashes; no non-breaking characters); see the **Writing style** rule above.
   - Abbreviations are limited to the widely-understood software-product set; project-specific terms are spelled out.
   - No orphan labels: every table header has rows, every list has items, every section has body content.

10. **Staging** -- `docs/.specify-staging/prd/PRD-...-kebab.md` (and `brd/` if new BRD).

11. **Publish** -- Use `searchConfluenceUsingCql` to check for a duplicate by title. Set `parentId` to the **domain PRD folder** resolved in step 3 (never the BRD page, never the BRD library folder, never the source BRD's `parentId` unless you have confirmed that id is the PRD folder). Use exactly one write MCP call per PRD (see the **Single MCP upload** rule under **PRD shape and writing style**): either **`createConfluencePage`** (new; ask before overwrite on duplicate) or **`updateConfluencePage`** when updating an existing PRD page -- not both in one run for the same doc. Body is markdown rendered per the active `prd-template.md`. Delete staging on success.

12. **Report** -- BRD/PRD URLs and page ids. Next: `/sp.backlog --confluence <id>`, `/sp.design` if the PRD's UI section is non-empty, `/sp.pdlc`.

## Ambiguity

Domain first (ask). Then master-spec/constitution for **facts**, not domain choice. Resolve product ambiguity before drafting; capture implementation ambiguity as `[DECISION NEEDED]` items (at most 3) with an options table.

## Reference

Structure: **`.specify/templates/prd-template.md`**.

## Pipeline

`BRD -> /sp.prd -> /sp.backlog -> /sp.design? -> /sp.pdlc -> /sp.feature-build`

## Related

`/sp.prd-validate`, `/sp.backlog`, `/sp.pdlc`, `/sp.feature-build`, `/sp.specify`, `/sp.clarify`
