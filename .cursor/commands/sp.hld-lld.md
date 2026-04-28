---
description: Generate HLD and LLD from spec/plan/codebase; repo mode, or /sp.hld-lld --confluence PRD page id to load PRD and publish HLD+LLD to Confluence (SoT) unless --write-repo.
---

## Usage

```text
/sp.hld-lld
/sp.hld-lld --hld-only
/sp.hld-lld --lld-only
/sp.hld-lld --confluence <pageId>
/sp.hld-lld --confluence <pageId> --write-repo
/sp.hld-lld --confluence <pageId> --confluence-hld-parent <folderId> --confluence-lld-parent <folderId>
/sp.hld-lld "emphasize checkout latency and payment failure modes"
```

| Flag | Effect |
|------|--------|
| *(none)* | **Repo mode:** create or refresh **`FEATURE_DIR/hld.md`** and **`FEATURE_DIR/lld.md`** only (no Confluence). |
| `--hld-only` / `--lld-only` | Limit which artifact(s) are produced (repo and/or Confluence per rules below). |
| `--confluence <pageId>` | **Confluence mode:** (1) **Source PRD** — **`getConfluencePage`** (`contentFormat: markdown`); **`cloudId`** via **`getAccessibleAtlassianResources`**. (2) **Publish** compact HLD + LLD to Confluence (**`createConfluencePage`** via **`call_mcp_tool`**, **`user-Atlassian-MCP-Server`**, **twice**) — **Confluence is the design SoT**; **no** local **`hld.md`/`lld.md`** unless **`--write-repo`**. **Do not** use Python/Node/curl for payloads; pass **`body`** in MCP arguments. |
| `--write-repo` | **With `--confluence` only:** also write **`FEATURE_DIR/hld.md`** and **`FEATURE_DIR/lld.md`** as a mirror. Omit for Confluence-only. |
| `--confluence-hld-parent <pageId>` | **Optional override** for **HLD** **`parentId`**. Omit to **auto-resolve** (see below). |
| `--confluence-lld-parent <pageId>` | **Optional override** for **LLD** **`parentId`**. Omit to **auto-resolve**. |

Trailing free text **supplements** the brief (emphasis areas, audience, constraints).

## User Input

Free text after flags (e.g. emphasis areas). Do **not** rely on a literal `$ARGUMENTS` placeholder in agent execution.

You **MUST** consider the user input before proceeding (if not empty).

## Preconditions

1. Run `.specify/scripts/bash/check-prerequisites.sh --json` from the repo root and parse **`FEATURE_DIR`** and **`AVAILABLE_DOCS`**.
2. **Requirements input (pick a path):**
   - **Repo-first (default):** **`FEATURE_DIR/spec.md`** and **`FEATURE_DIR/plan.md`** are **required**. If **`plan.md`** is missing: print a clear error — *"Run `/sp.plan` first."* — and stop without writing files.
   - **Confluence PRD augment:** If **`--confluence <pageId>`** is set, **fetch the PRD** and merge it into traceability (tables, links). **`plan.md` still required** unless the user is explicitly doing a **PRD-only design pass** (see below).
   - **PRD-only pass (exception):** **`--confluence <pageId>`** and **no `plan.md`** — allowed **only** when the user’s intent is to produce HLD/LLD **from the Confluence PRD + codebase** (e.g. before `/sp.plan` exists). **WARN** that full SDD prefers **`/sp.specify`** + **`/sp.plan`** in-repo; output must still reference the PRD URL and PRD ID. If **`spec.md`** is also missing, same WARN and proceed only with PRD + Roam + master-spec/constitution as grounding.
3. **Recommended inputs** (use if present): `research.md`, `data-model.md`, `contracts/`, existing ADRs under the feature or `docs/adr/`.

## Confluence publish (when `--confluence <pageId>`)

| Rule | Requirement |
|------|-------------|
| **MCP** | **`getAccessibleAtlassianResources`** → **`cloudId`**; PRD page supplies **`spaceId`** for **`createConfluencePage`**. |
| **Titles** | Page title = **`<PRD-ID or feature name>: HLD`** and **`<PRD-ID or feature name>: LLD`** (or user-provided pattern). **Body** must not repeat the title as a duplicate **H1** (same convention as **`/sp.prd`**). |
| **Folder placement (mandatory)** | **HLD** page **`parentId`** = **HLD folder** in the same space as the PRD. **LLD** page **`parentId`** = **LLD folder** in that space. **Do not** put both under the PRD page; **do not** use the BRD library folder. |
| **Auto-resolve (default — no user input)** | **Always** resolve **HLD** and **LLD** folder ids automatically unless **`--confluence-hld-parent`** / **`--confluence-lld-parent`** is passed. Do **not** ask the user for folder ids unless every step below fails. |
| **Auto-resolve algorithm** | 1) **Space context:** From **`getConfluencePage`** on the source PRD, read **`spaceId`** and resolve the **space key** if CQL needs it (e.g. **`getConfluenceSpaces`** or page metadata). 2) **CQL passes** (run in order until a unique **folder** is found for each): `title = "HLD"` / `"LLD"` with **`type = folder`**; then `title ~ "HLD"` / `"LLD"` narrowing to **folder**; then title variants **High-Level Design** / **Low-Level Design** / **High level** / **Low level** (space-scoped). Use **`space = <SpaceKey>`** in CQL when the API requires the key (numeric **`spaceId`** alone may not work in CQL — mirror **`/sp.prd`** behavior). 3) **Sibling pairing:** If one folder (e.g. HLD) is found, **`getConfluencePage`** on that item to read **`parentId`**, then **`getConfluencePageDescendants`** on the **parent** or **`searchConfluenceUsingCql`** for siblings under the same **`parentId`** to locate the missing **LLD** / **HLD** folder by title. 4) **Tie-break:** Prefer **`type = folder`** over page; prefer exact title **HLD** / **LLD**; prefer folders in the same parent as the **PRD** folder’s design siblings when the PRD lives under a domain tree. 5) **Failure:** Only if **no** candidate exists after (2)–(4), **WARN**, list what was searched, and **then** ask once for **`--confluence-hld-parent`** / **`--confluence-lld-parent`** or space admin to create the folders — do **not** attach HLD/LLD to the PRD page as a silent fallback. |
| **Writes** | Up to **two** successful **`createConfluencePage`** (one under **HLD** folder, one under **LLD**), or **`updateConfluencePage`** when updating by existing page id. Retries after failure are OK. |
| **Cross-links** | In each page body, link to the **source PRD** page and, where useful, to the sibling design page (HLD ↔ LLD). |
| **Size** | Keep markdown **small enough for one MCP `body`** per page (compact tables; short sections). Shorten before upload if needed. |
| **Source of truth** | **With `--confluence`:** **Confluence pages** are authoritative; **no** repo files unless **`--write-repo`**. **Without `--confluence`:** **repo** files under **`FEATURE_DIR`** only. |

## Roam MCP Navigation

Use the `user-roam-code` MCP server to ground HLD/LLD in the real codebase — **`call_mcp_tool`** / **`roam_*`** per **`roam.mdc`**; no shell-only workarounds when MCP is available.

| When | MCP Tool (`user-roam-code`) | Purpose |
|------|----------------------------|---------|
| Start | `roam_explore` | Codebase overview + optional symbol deep-dive |
| Boundaries / impact | `roam_impact` (symbol) | Blast radius for major entry points |
| File-level deps | `roam_deps` (path) | Imports and importers for LLD module map |
| Health context | `roam_health` | Optional baseline for complexity hotspots |

## Atlassian MCP (PRD + publish)

Use **`user-Atlassian-MCP-Server`** via **`call_mcp_tool`** when **`--confluence <pageId>`** is present — **direct tool calls only** (see **`.cursor/rules/mcp-integration.mdc`**).

| When | Tool | Purpose |
|------|------|---------|
| Source PRD | `getConfluencePage` | `cloudId`, `pageId`, `contentFormat: markdown` — also **`spaceId`**, **`parentId`** for navigation |
| Space key for CQL | `getConfluenceSpaces` | Map **`spaceId`** → **space key** when CQL needs **`space = KEY`** |
| New design pages | `createConfluencePage` | `spaceId`, `parentId`, `title`, `body`, `contentFormat: markdown` — **pass `body` as the markdown string in MCP arguments** |
| Refresh existing | `updateConfluencePage` | `pageId`, `body`, … |
| Auto-find folders | `searchConfluenceUsingCql`, `getPagesInConfluenceSpace`, `getConfluencePageDescendants` | Resolve **HLD** / **LLD** folder **ids** without user input |
| Duplicate check | `searchConfluenceUsingCql` | Optional: title collision before create |

## Execution

1. **Resolve scope** from arguments: `--hld-only`, `--lld-only`, or both (default both). If **`--confluence <pageId>`**, fetch PRD first and keep **page URL + id** and **`spaceId`**; **automatically** resolve **HLD** and **LLD** folder **`parentId`** values (overrides optional) before **`createConfluencePage`**.

2. **Load artifacts** (absolute paths): `spec.md`, `plan.md` (if preconditions allow), plus any of `research.md`, `data-model.md`, listings under `contracts/`. **Merge** Confluence PRD content for requirements/traceability when **`--confluence`** is set.

3. **LLD-only rule:** If **`--lld-only`** and no prior HLD artifact: **with `--confluence`**, publish **LLD** Confluence page only (and link to PRD); **repo-only**, create **`lld.md`** and note if HLD is pending. With **`--confluence`** and full package, publish **HLD** then **LLD** unless flags restrict.

4. **Templates** (repo root paths):
   - HLD: **`.specify/templates/hld-template.md`**
   - LLD: **`.specify/templates/lld-template.md`**

5. **Fill templates**:
   - Replace `{{FEATURE_NAME}}` with the feature folder name or title from **`spec.md`**, or from the Confluence PRD title/PRD ID when using **`--confluence`** without spec.
   - Replace `{{DATE_ISO}}` with `date -u +"%Y-%m-%d"` (or document local date if constitution prefers).
   - **HLD** must stay **system-oriented**: boundaries, major components, integrations, NFR mapping — **not** per-function implementation detail.
   - **LLD** must tie **HLD components** to **concrete modules/paths/symbols** (from Roam + repo), interfaces, sequences, errors, and test focus.

6. **Write repo (optional):** Write **`FEATURE_DIR/hld.md`** and **`lld.md`** when **repo-only** (no `--confluence`) **or** when **`--confluence --write-repo`**. With **`--confluence`** alone, **skip** local files unless **`--write-repo`**. If writing and files exist, **merge intelligently** where appropriate.

7. **Publish (if `--confluence <pageId>`):** Build **compact** HLD and LLD markdown (no duplicate **H1** with page title). Invoke **`createConfluencePage`** **directly** — **HLD** under the **HLD** folder, **LLD** under the **LLD** folder. Report **web** URLs + page **id**s. **Do not** require prior local files.

8. **Consistency:** HLD/LLD content must align; cross-link in Confluence bodies (PRD + sibling design page). If **`--write-repo`**, ensure **`lld.md`** references **`hld.md`** when both exist.

## Quality gates (self-check before finishing)

- [ ] HLD has no low-level function lists; LLD has no vague-only architecture without repo anchors.
- [ ] Traceability: requirements in spec/plan (and Confluence PRD if used) are reflected or explicitly marked out-of-scope.
- [ ] Open questions in HLD/LLD are actionable or owned.
