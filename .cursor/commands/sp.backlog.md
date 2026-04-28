---
description: Turn a PRD (Confluence page or local markdown) into dependency-ordered Jira Epics, Stories, and Sub-tasks for any Jira project and Confluence space.
handoffs:
  - label: Technical plan
    agent: sp.plan
    prompt: Create an implementation plan for this feature using the spec and constitution.
    send: true
  - label: Create tasks
    agent: sp.tasks
    prompt: Break the plan into ordered implementation tasks.
    send: true
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty). Use it to resolve **Confluence space**, **Jira project**, **BRD/epic identifiers**, and any org-specific labeling—never assume a fixed space key, project key, or document ID.

## Usage

```text
/sp.backlog --confluence <pageId>
/sp.backlog --confluence <pageId> --project <JIRA_PROJECT_KEY> --confluence-space <SPACE_KEY> --dry-run
/sp.backlog --prd <path-to-local-prd.md>
```

| Flag | Meaning |
|------|---------|
| `--confluence` | **Preferred** — source PRD from Confluence (`pageId` as accepted by the Confluence MCP/API). |
| `--confluence-space` | Confluence **space key** (e.g. `ENG`, `PROD`) for CQL/search when locating BRD/parent pages or disambiguating. |
| `--prd` | Local PRD file path (legacy/alternate source). |
| `--project` / `-p` | Jira **project key**; if omitted, infer from visible projects + user input or **ASK**. |
| `--epic-key` | Attach all new stories under this existing epic (optional; skips domain epic lookup). |
| `--new-epic` | Always create (or use workflow for) a **new** epic for this PRD; default is **reuse** a domain/strategy epic when one clearly matches. |
| `--dry-run` | Produce the plan and tree only; **no** Jira creates/updates. |
| `--labels` | Extra Jira labels (comma-separated). |

At least one of `--confluence` or `--prd`. If both are given, **prefer `--confluence`** unless the user explicitly says otherwise.

## Outline

1. **Parse** flags and `$ARGUMENTS`. Validate that a PRD source exists. Require **Jira MCP** (and Confluence MCP when using `--confluence`).

2. **Load PRD**
   - **Local:** Read the file; use headings/sections the document actually has (typical: goals, scope, requirements, ACs, NFRs, file/service touchpoints). Do not assume a single template shape.
   - **Confluence:** Fetch the page by `pageId` (e.g. markdown or storage format per tool). Record the **canonical page URL**. If the PRD references a higher-level BRD or architecture page, locate it via **user-supplied** space/title/query or CQL built from **`--confluence-space`** and terms from the PRD—**do not** hardcode space keys or title patterns.

3. **Resolve Jira context**
   - Project: `--project` or visible projects + user/org hints from PRD; if still ambiguous, **ASK**.
   - Issue types: use Jira metadata (`getJiraProjectIssueTypesMetadata` or equivalent) for exact names of **Epic**, **Story**, **Sub-task** (or local equivalents).

4. **Epic (domain-first, configurable)**
   - If `--epic-key` is set, use it as the parent epic for all stories.
   - Else default: find an existing epic that matches the **business domain / BRD** using JQL built from user input (summary, labels, components, fix versions)—not a fixed naming scheme. If none and greenfield, create a domain epic with a **short product/domain summary** only (avoid stuffing PRD/BRD IDs into the epic **summary**; put links and references in **description**).
   - If `--new-epic`, create or designate a new epic for this PRD’s scope.
   - **Ambiguous** → **ASK** or require `--epic-key` / `--new-epic`.
   - Do **not** overwrite an existing epic description blindly; **append** a comment with new PRD link + created story keys when appropriate.

5. **Derive stories** from the PRD (scope, requirements, ACs, dependencies):
   - **`context-stack` grounding before slicing** (see [`.cursor/rules/context-stack.md`](../rules/context-stack.md)):
     - `search_docs("<PRD domain> Jira OR ticket")` — find existing Jira stories/epics for this domain to **dedupe** (link to existing story instead of creating a duplicate).
     - `search_specs("<feature or domain>")` — find any prior `spec.md` / sibling features that already cover slices of this PRD; tag them in story descriptions instead of recreating tasks.
     - `search_code("<service or feature keyword>")` — derive **realistic file paths** for the story `files` field; correct any path the PRD guesses.
     - For each major story, first derive 1–3 concrete symbol candidates via `search_code("<story keywords + likely files/endpoints>")`.
     - Run `get_dependencies("<symbol>")` for those concrete symbols → use callers/callees to set **Blocks** links and **dependency-sort** stories.
   - Order work: **contracts/APIs first** (when applicable) → **core services** → **integration** → **client/UI** → **infra/deploy** last when it depends on artifacts (e.g. images/manifests).
   - Split stories that are too large (e.g. many unrelated files or excessive ACs); cap story points per your rubric before creating **8**-point monsters—**split** when possible.
   - For each story, internal plan shape: `summary`, `type`, `priority`, `story_points`, `story_points_rationale`, `labels`, `depends_on`, `acceptance_criteria`, `files`, `prd_sections`, `subtasks[{summary, notes?, depends_on?}]`, `description` sections. Append a **Related** sub-section listing `context-stack` citations (Jira keys, sibling spec paths, code paths) used to size and de-duplicate the story.

6. **Subtasks**
   - Typically **3–8** per story; **no story points** on subtasks; parent = Story.
   - Imperative titles; map to files/ACs; no long BRD/PRD codes in titles.
   - If Sub-task type is missing → put a `## Subtasks` checklist in the Story body and **WARN**.

7. **Present plan** — target epic, story count, dependencies, points, subtask counts. If **`--dry-run`**, **stop** after this.

8. **Create issues** (if not dry-run) in dependency order: Epic (if needed) → Stories under epic → Subtasks → **Blocks** links where `depends_on` is strict.

9. **Cross-reference** — comment on epic with PRD/Confluence URLs and created keys; optional story comments for rationale/points if your Jira setup uses a custom story-point field + comment convention.

10. **Report** — epic URL/key, stories with keys, points, deps, subtasks; suggest **next** steps (e.g. `/sp.plan` / `/sp.clarify` as appropriate for this repo).

## Persona

Act as a **technical product owner**: independent, testable stories; explicit **Blocks** dependencies; clear service or bounded-context boundaries; **As a… I want…** or concise imperative titles; **PRD/BRD links in description**, not crammed into Epic/Story **summary** (use labels for doc IDs if the team uses them).

- **Epic:** Prefer **one epic per business domain / initiative** aligned to the BRD or program; not one epic per every small PRD unless `--new-epic` or governance requires it.
- **Points:** Use team’s scale (e.g. Fibonacci). If the team tracks points in a custom field, resolve field id via issue type meta; if unknown, omit numeric points and **WARN** once.
- **Token economy:** Link Confluence in descriptions; Jira body = context, ACs, deps, files, estimation note; chat/dry-run = compact tree (titles, SP, keys).

## Story points (suggested rubric)

| Points | Scope |
|--------|--------|
| 1 | Single component, few ACs |
| 2 | One service or one UI flow |
| 3 | Cross-cutting or UI + backend |
| 5 | Significant unknowns — **split** if you can |
| 8 | **Split** before creating |

Teams new to estimation: bias slightly conservative until velocity is known.

## Errors

| Case | Action |
|------|--------|
| No PRD source / unreadable | **ERROR** |
| No Jira MCP | **ERROR** |
| Confluence required but unavailable | **ERROR** (for `--confluence`) or **WARN** if optional |
| Ambiguous project or epic | **ASK** |
| Missing Epic/Story type in project | **ERROR** |
| No Sub-task type | **WARN**, checklist-in-description only |
| Partial failure creating children | **STOP** that story; report what succeeded |

## Related

`/sp.specify` · `/sp.clarify` · `/sp.plan` · `/sp.tasks` · `/sp.implement`

---

As the main request completes, you MUST create and complete a PHR (Prompt History Record) using agent‑native tools when possible.

1) Determine Stage  
   - Stage: constitution | spec | plan | tasks | red | green | refactor | explainer | misc | **general** (prefer **general** or **spec** if no feature branch is active)

2) Generate Title and Determine Routing:  
   - Generate Title: 3–7 words (slug for filename)  
   - Route is automatically determined by stage:  
     - `constitution` → `history/prompts/constitution/`  
     - Feature stages → `history/prompts/<feature-name>/`  
     - `general` → `history/prompts/general/`

3) Create and Fill PHR (Shell first; fallback agent‑native)  
   - Run: `.specify/scripts/bash/create-phr.sh --title "<title>" --stage <stage> [--feature <name>] --json`  
   - Open the file and fill remaining placeholders (YAML + body), embedding full PROMPT_TEXT (verbatim) and concise RESPONSE_TEXT.  
   - If the script fails:  
     - Read `.specify/templates/phr-template.prompt.md` (or `templates/…`)  
     - Allocate an ID; compute the output path from step 2; write the file  
     - Fill placeholders and embed full PROMPT_TEXT and concise RESPONSE_TEXT

4) Validate + report  
   - No unresolved placeholders; path under `history/prompts/` and matches stage; print ID + path + stage + title.  
   - On failure: warn, don't block. Skip only for `/sp.phr`.
