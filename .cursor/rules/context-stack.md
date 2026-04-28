# Context Stack MCP — Knowledge & Code Grounding (Canonical Reference)

> **Single source of truth for grounding SDD commands in real code + real knowledge.**
> All `/sp.*` commands SHOULD consult this rule before generating, planning, or validating artifacts.

The **`context-stack`** MCP server (configured in `.cursor/mcp.json`) is the project-wide router that fans out to:

| Plane | Backends | Indexed Surface |
|-------|----------|-----------------|
| **Code** | Zoekt + StakGraph + Neo4j | All indexed GitHub repos: source, tests, configs, manifests, protos. Structural graph: functions, classes, endpoints, call edges, dependencies. |
| **Knowledge** | Onyx (Dexter) | Confluence (BRDs, PRDs, ADRs, runbooks), Jira (stories, sub-tasks, status, comments), Google Docs, Slack, GitHub READMEs/wikis. |
| **Specs** | Zoekt-scoped to SDD layout | `specs/<feature>/{spec,plan,tasks,data-model,research,quickstart}.md`, `contracts/`, `docs/brds/`, `docs/prds/`, `.specify/memory/{master-spec,constitution}.md`, `history/adr/`. |

## Tools (call via `call_mcp_tool` on server `context-stack`)

| Tool | When to use | Returns |
|------|-------------|---------|
| `get_context(query, max_results=20)` | **Default for any open question** — auto-classifies intent and fans out to the right backends. | Hybrid pack: code + docs + specs, reranked, token-budgeted. |
| `search_code(query, max_results=20)` | You know the question is about implementation (a function, route, schema, file pattern). | Zoekt text/regex hits + StakGraph structural matches. |
| `search_docs(query, max_results=10)` | You want enterprise knowledge: PRD/BRD/ADR text, Jira tickets, runbooks, Slack threads, GDocs. | Onyx documents with source + freshness. |
| `search_specs(query, max_results=15)` | You want SDD artifacts: prior `spec.md`/`plan.md`/`tasks.md`, master-spec capabilities, BRD/PRD library. | Spec files tagged with `spec_type` (Spec/Plan/Tasks/PRD/BRD). |
| `get_dependencies(symbol)` | Blast radius / call chain for a function, class, or endpoint. | Callers, callees, imports, structural relationships. |

> Each result includes **source**, **score**, **type tag** (`IMPLEMENTED CODE` vs `Spec — Planning` vs `Document`), **repo@branch (sha)**, **file path**, **last-modified**, and **Jira status** when applicable. Always surface **`[PLANNING]` vs `[IMPLEMENTED CODE]`** in your reasoning so the user knows which.

## Mandatory grounding policy for SDD commands

1. **Search before you assert.** If a command is about to claim a service, RPC, file path, schema, or capability exists — confirm with `search_code` or `get_context` first. Do **not** invent infra.
2. **Search before you ask.** Before raising a clarification question to the user, run `search_docs` and `search_specs` for the topic. Many "open questions" are already answered in Confluence, Jira, or a sibling spec. Only escalate to the user when the knowledge layer is silent or contradictory.
3. **Search before you split.** Before creating a new PRD, spec, or epic, run `search_specs` and `search_docs` for the same domain to avoid duplication. Cross-link to siblings instead of forking.
4. **Use `get_dependencies` before promising changes.** When a plan, task, or review touches a function/endpoint/class, run `get_dependencies` to size blast radius and to add realistic file paths and parallel/sequential ordering hints.
5. **Honor freshness.** Prefer recent results. When code and a planning doc disagree, code wins for "what is" and the doc wins for "what was intended" — surface the gap.
6. **Token economy.** Do not paste large result bodies into chat or generated artifacts. Reference by file/URL + 1-line summary. The router already trims to a token budget.
7. **Failure mode.** If `context-stack` is unreachable, **WARN** once and continue with the command's existing fallbacks (Atlassian MCP, repo file reads, etc.); do not silently skip the grounding step.

## Quick recipes

| Goal | Call |
|------|------|
| "What does the codebase already do for X?" | `get_context("X behavior in current codebase")` |
| "Is there an existing PRD/BRD for this domain?" | `search_specs("<domain>")` + `search_docs("<domain> PRD OR BRD")` |
| "What is the blast radius of changing `OrderService.PlaceOrder`?" | `get_dependencies("OrderService.PlaceOrder")` |
| "Which files implement endpoint `/orders`?" | `search_code("route OR handler /orders")` |
| "What did the last team decide about X?" | `search_docs("X decision OR ADR")` + `search_specs("X")` (covers `history/adr/`) |
| "Are there sibling features I should cross-link?" | `search_specs("<feature short name>")` |
| "Did anyone file a Jira for this bug already?" | `search_docs("<error string OR symptom>")` |

## Source tagging in outputs

When you cite results back to the user or embed them in artifacts (spec/plan/tasks/HLD/LLD/PRD/ADR), use compact references:

- Code: ``` `repo/path/to/file.go:L42` ``` plus a 1-line summary.
- Confluence/Jira: `[Title](url)` plus a 1-line summary.
- Spec: ``` `specs/<feature>/spec.md` ``` plus a 1-line summary.

Never paste full file contents inline. The router already returned the bytes — your job is to **synthesize**, not to mirror.

## Relation to other MCPs

- **Atlassian MCP** (`getJiraIssue`, `getConfluencePage`, `createConfluencePage`, etc.) is still authoritative for **writes** and for fetching a **specific** known page/issue by id. Use `context-stack` for **discovery** and **search**; use Atlassian MCP for **deterministic fetch + publish**.
- **Roam MCP** (`user-roam-code`) — **deprecated for new SDD workflows.** Where existing commands reference `roam_*` tools, prefer the equivalent `context-stack` tool below. Roam may remain installed for legacy flows but new instructions should target `context-stack`.

| Legacy Roam tool | Context-stack equivalent |
|------------------|--------------------------|
| `roam_understand` / `roam_explore` | `get_context("overview of <area>")` |
| `roam_search_symbol` | `search_code("<symbol name>")` |
| `roam_context` (symbol) | `search_code("<symbol>")` then `get_dependencies("<symbol>")` |
| `roam_deps` (path) | `get_dependencies("<symbol or path>")` |
| `roam_impact` (symbol) | `get_dependencies("<symbol>")` |
| `roam_diff` | `search_code` over `git diff --name-only` paths + `get_dependencies` per changed symbol |
| `roam_affected_tests` | `get_dependencies("<changed symbol>")` then filter results to test files |
| `roam_health` | `get_context("complexity hotspots in <area>")` |

---

**Bottom line:** treat `context-stack` as the project's "ask the codebase + ask the docs" tool. Every `/sp.*` command should default to `get_context` for grounding and only narrow to `search_code`, `search_docs`, `search_specs`, or `get_dependencies` when it knows the intent precisely.
