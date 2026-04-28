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
/sp.prd … --full
```

| Flag | Meaning |
|------|---------|
| `--jira` / `--ticket` / `-j` | `getJiraIssue` |
| `--confluence` | `getConfluencePage` markdown |
| `--name` | First `PRD-BB-MM`; auto-increment siblings |
| `--single-prd` | Force **one** Confluence doc when the agent would otherwise split a single journey |
| `--full` | Long-form draft (extra prose per section). **Still** shrink before Confluence publish if body exceeds safe size (below). Default is **compact**. |

**Defaults:** A BRD may yield **K PRDs** only when slices are **independently shippable** or **ownership/context** differs. **One buyer journey** (e.g. the same list screen with **filter + search** together) → **one PRD**; do **not** split into separate PRDs per control unless the BRD explicitly separates scope. **`--single-prd`** forces one doc when the agent would otherwise over-split. **Confluence body = compact PRD** unless `--full` and under safe size.

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
| Scope | One **primary user goal / journey** per PRD. Multiple **mechanics** on the same journey (e.g. **optional filter** and **optional search** on **My Orders**) stay **in the same PRD**—one requirements table, one AC set, one publish. |
| BRD | Many **distinct** capabilities or domains → many PRDs. Do **not** mint separate PRDs for each UI control when the BRD describes one incremental list experience. |
| Header | `Related PRDs:` + Confluence URLs when siblings exist. |
| Split if | Distinct journeys, different owning service/context, different shippable stories, **~400 lines / ~8k words** (main sections), or BRD feature matrix with **explicit** independent delivery. |
| One doc OK | Single cohesive journey (including filter+search-style bundles) or `--single-prd`. |

## Compact PRD (default for publish)

**Goal:** One Confluence page that is PO-complete but **short** (see template size target). **One Confluence write per PRD outcome**—see **Single MCP upload** below.

| Rule | Requirement |
|------|-------------|
| **Single MCP upload** | Use **exactly one** Confluence **write** MCP call per PRD delivered in the run: **`createConfluencePage`** (new page) **or** **`updateConfluencePage`** (replace body on an existing page the user pointed at or a duplicate title match). Do **not** chain create + update to “finish” the same PRD; **not** multiple updates; prepare the **final** compact markdown **before** that single call. Retries after tool/network failure are OK (same intent, one successful write). |
| **Target size** | Aim **≤ ~8 KB UTF-8** for the markdown `body` (heuristic: ~≤220 lines). If larger, **shorten** before the upload call. |
| **Template** | Follow **§1–15** in `.specify/templates/prd-template.md` (compact tables). Do not drop sections—**compress** them. |
| **Techniques** | Same as template: summary + metadata tables; one table each for goals, requirements, AC, stories; short §6 bullets; §8 minimal or N/A; appendix = tight implementation map. |
| **`--full`** | Staging prose only; **before Confluence**, trim so **one** upload carries the full final body (or split into **separate PRDs** only when § “One PRD” says so—each PRD still gets **one** write). |
| **Split vs shorten** | Prefer **one compact PRD** when the BRD is one buyer journey (e.g. filter + search on the same list). **Split** when capabilities are independently shippable or ownership differs—not merely because the doc is long. |
| **Publish** | **New:** **`createConfluencePage`** with full compact `body` (no H1 duplicate of title), correct `parentId`, `contentFormat: markdown`. **Existing page refresh / user-supplied id:** **`updateConfluencePage`** with the same constraints. **Do not** use update as a second step after create for the same content in one run. |

## Storage

- **SoT:** Confluence (`createConfluencePage` / `updateConfluencePage`). No new `docs/prds/**` or `docs/brds/**`.
- **Drafts:** `docs/.specify-staging/prd|brd/*.md` (gitignored). **Delete** after successful publish; **WARN** + keep if publish skipped/failed.
- Titles: `<PRD-ID>: <Feature>`, `BRD-NN: <Domain>`.

## Inputs

Combine `--jira` + `--confluence` allowed. Fetch before draft. No fetch + no usable text → **ERROR**. Structure = **`.specify/templates/prd-template.md`** only (no wiki crawl for layout).

## Grounding

Match **repo reality** (master-spec, constitution, `roam_understand`). Missing auth/DB/RPC → prerequisite or **NEW** in deps/files — never assume.

| When | Roam |
|------|------|
| Before draft | `roam_understand` |
| Symbols | `roam_search_symbol` |
| Service | `roam_explore` |

## Outline

1. **Parse/fetch/merge** — `getJiraIssue`, `getConfluencePage`, free text → one source doc.

2. **Repo** — Read master-spec, constitution; optional `roam_understand`. Checklist: services, auth, persistence, gRPC/HTTP, frontend patterns. PRD must align.
   - **Bounded context (before 4–draft):** Short/generic/vague (*sort*, *list*, …) or multi-domain → **AskQuestion** first. No silent domain from code layout.

3. **Confluence (narrow)** — One `getConfluencePage` per id; **no** space-wide BRD crawl. BRD + at most one canonical follow-up. Siblings: narrow CQL `PRD-<BB>-*` or metadata only; `getConfluencePage` only to resolve conflicts.

   **Publish parent (PRD folder — mandatory)**  
   New PRDs MUST live under the **domain’s PRD folder** in Confluence (the parent of pages like `PRD-02-01`, `PRD-03-01`, etc.). They MUST **NOT** be created under the **BRD library folder** (the parent shared by `BRD-01`, `BRD-02`, `BRD-07`, …). Those two folders are different: a BRD’s `parentId` from `getConfluencePage` is usually the BRD tree — **do not** reuse it as `parentId` for `createConfluencePage` on a PRD.

   **How to resolve `parentId` for publish (in order):**
   1. From the source BRD, read **domain** (e.g. *Order Management*, *Identity & Authentication*).
   2. Find **any existing PRD** in that domain in the same space (`getPagesInConfluenceSpace` / title prefix `PRD-` + domain keyword, or narrow CQL). Open it; set **`parentId` = that PRD’s `parentId`** (the domain PRD folder).
   3. If no PRD exists yet: locate the space’s **PRD** hierarchy and the **folder** for that domain (title aligned with domain), via space navigation or admin — use that folder’s id. If it cannot be found, **stop** and ask the user for the correct PRD folder page id (do not fall back to the BRD folder).
   4. **Validate:** After create, the new PRD’s `parentId` must equal a known domain PRD sibling’s `parentId`, not the BRD page’s `parentId`.

4. **IDs** — `PRD-BB-MM`; K slices unless `--single-prd`; numbering from **Confluence** narrow search, not `docs/prds/`. No BRD: `PRD-<NN>-<MM>` from Confluence titles.

5. **BRD** — Exists → header ref. Else create in Confluence (concise domain doc); `spaceId` FKPOC `490405892`; delete staging BRD after create.

6. **Questions** — Max 5; priority: bounded context, personas, success, scope, deps, constraints. Skip only if source substantive + domain explicit + no alternate domain.

7. **Template** — Read `prd-template.md`.

8. **Draft** — **Compact** (default): all **§1–15** per slice in digest form (**Compact PRD** table above). `--full`: richer prose, then compress for publish if needed. Strip sibling scope → **Related PRDs**. §4 real paths/RPCs; §10/`src/` paths in appendix map; prefer tables over walls of text.

9. **Validate** — Bounded context confirmed if 2f applied; header; no invented infra; ≤3 `[DECISION NEEDED]`; one **journey** per PRD (mechanics bundled per § “One PRD”); extra slices only with `--single-prd` or true multi-capability BRD. Fix failures (≤3 passes).

10. **Staging** — `docs/.specify-staging/prd/PRD-…-kebab.md` (and `brd/` if new BRD).

11. **Publish** — `searchConfluenceUsingCql` duplicate by title; `parentId` = **domain PRD folder** from §3 (**never** the BRD page, **never** the BRD library folder, **never** the source BRD’s `parentId` unless you have confirmed that id is the PRD folder). **One write MCP call** per PRD (§ Compact PRD **Single MCP upload**): either **`createConfluencePage`** (new; ask before overwrite on duplicate) **or** **`updateConfluencePage`** when updating an existing PRD page—**not** both in one run for the same doc. **Body** = compact markdown per template. Delete staging on success.

12. **Report** — BRD/PRD URLs + page ids; next: `/sp.backlog --confluence <id>`, `/sp.design` if §6 UI, `/sp.pdlc`.

## Ambiguity

Domain first (ask). Then master-spec/constitution for **facts**, not domain choice. Product ambiguity → resolve before draft; implementation ambiguity → `[DECISION NEEDED]` (≤3) with options table.

## Reference

Structure: **`.specify/templates/prd-template.md`**.

## Pipeline

`BRD → /sp.prd → /sp.backlog → /sp.design? → /sp.pdlc → /sp.feature-build`

## Related

`/sp.prd-validate` · `/sp.backlog` · `/sp.pdlc` · `/sp.feature-build` · `/sp.specify` · `/sp.clarify`
