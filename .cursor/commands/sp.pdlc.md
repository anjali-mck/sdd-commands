---
description: Product delivery orchestration — discovers Jira stories and Confluence PRD/BRD context, runs sp.feature-build per story through push + PR, posts PR links on stories, and marks Sub-tasks Done on commit. Merge and Story → Done are not part of PDLC (senior engineer owns merge).
---

## Usage

```
/sp.pdlc --jira FP-26
/sp.pdlc --jira FP-28
/sp.pdlc --confluence 510001204
/sp.pdlc --jira FP-26 --story FP-28
/sp.pdlc --jira FP-26 --all
/sp.pdlc --jira FP-26 --story FP-28 --no-jira-status
/sp.pdlc --jira FP-77 --jira FP-78 --unattended
```

**Flags** — everything **except** the first group is passed through or only affects discovery; **SDD execution** is always [`sp.feature-build.md`](sp.feature-build.md).

| Flag | Role |
|------|------|
| `--jira <KEY>` / `-j` | Epic or story (see §B). May appear **multiple** times to build a merged, dependency-sorted queue. |
| `--story <KEY>` | With **one** epic `--jira`, build only this child story. |
| `--all` | With **one** epic `--jira`, build all child stories (dependency-sorted). |
| `--confluence <pageId>` | Start from a Confluence PRD page; **must** fetch that page; resolve Jira stories from the page + JQL (see §B). |
| `--no-jira-status` | Forward to **every** `/sp.feature-build` invocation — skips assignee, **In Review** / **Done** transitions, and **Sub-task Done** transitions; **PR comment** still required. |
| `--unattended` | Skip [`sp.feature-build.md`](sp.feature-build.md) **Steps 2–4** (Plan Mode + approval) for this PDLC run; after §D, go straight to §F per story. |

For **one** story and **no** Confluence round-trip, use **`/sp.feature-build`** directly.

## User Input

```text
$ARGUMENTS
```

## Purpose

`/sp.pdlc` is **`/sp.feature-build`** with extra **inputs**: Jira epic/multi-story selection, **optional** Confluence PRD/BRD body fetch (only when §C says it is required), and a **trailing context blob** per story for `/sp.specify`. It does **not** define a second SDD pipeline — **`sp.feature-build.md` is authoritative** for Plan Mode, SDD phases, verification, review, `SDD-TIMELINE.md`, and PR open.

**PDLC execution ends at Phase 9** (push + open PR + Jira comment + **In Review** when workflow updates apply). **`sp.feature-build.md` Phase 10** (post-merge Story → **Done**) exists for teams that automate it, but **`/sp.pdlc` does not require the PR author to merge the PR or transition the Story to Done** — a **senior engineer** (or designated merge owner) performs merge and downstream Jira updates per team process.

**PDLC adds hard gates:** a delivery slice is **not** complete until **Phase 9** produced a **PR URL** **and** the Jira issue has an **`addCommentToJiraIssue`** note with that link (§F.1 + §G). **Sub-tasks** close when their **`tasks.md`** work is **committed** (§G.2). Do not stop after commit only.

## Pipeline position

```
BRD / Brief → /sp.prd → /sp.backlog → /sp.pdlc → (per story) /sp.feature-build lifecycle
```

---

## PDLC-only steps

### A. Parse arguments

- Flags order-agnostic; at least one of `--jira` or `--confluence` required; else **ERROR**.
- **`--story` / `--all`**: only when **exactly one** `--jira` names an **Epic** (not with multi-`--jira` unless each is a Story).

### B. Resolve Jira → ordered story queue

Use Atlassian MCP (`getAccessibleAtlassianResources`, `getJiraIssue`, `searchJiraIssuesUsingJql`).

**`--jira` path**

1. Fetch issue; if **Epic**, load children:  
   `project = <project> AND ("Epic Link" = <KEY> OR parent = <KEY>) ORDER BY priority DESC, created ASC`
2. Select stories: `--story` / `--all` / **AskQuestion** (multi-select; prefer To Do; Done excluded by default).
3. **Multiple `--jira`**: fetch each; stories go on the queue; epics expand; **dedupe keys**; **dependency-sort** using `Depends on: FP-XX` in descriptions.

**`--confluence` path** (no `--jira`)

1. `getConfluencePage(pageId, contentFormat: "markdown")`
2. Find Jira keys / links; `searchJiraIssuesUsingJql` to attach stories.
3. If **no** stories: report and suggest creating backlog or `/sp.backlog`.

### C. Confluence context (per story) — fetch only when required

**Default:** Prefer **Jira** (summary, description, acceptance criteria, technical notes, file tables). **Do not** call `getConfluencePage` for linked PRD/BRD pages unless a case below applies.

**When you MUST fetch** (`getConfluencePage`, `contentFormat: "markdown"`):

1. **§B `--confluence` path** — the starting `pageId` (and any page you must read to resolve stories from the doc).
2. **Jira is insufficient for `/sp.specify`** — e.g. no real acceptance criteria or scope, only “see Confluence” without actionable detail; then fetch the **minimum** set of pages (usually the linked PRD first) until the story is spec-ready.
3. **The developer asks** for PRD/BRD excerpts in the plan or execution phase.

**When you MUST NOT fetch** (traceability without API):

- Jira already contains enough to write `spec.md` (AC, scope, files, semantics, links to PRD/BRD for audit). **Parse PRD/BRD URLs from the description** and record **id / title / url** from the link text and href; leave `content` empty or omit the `prd`/`brd` object’s body.

**Dedupe:** if you do fetch, at most **one** `getConfluencePage` per unique page ID for the whole run.

Bundle shape (omit `content` on `prd`/`brd` when not fetched):

```yaml
story: { key, summary, description, acceptance_criteria, priority, status }
prd:  { id, title, url, content? }   # content present only if fetched
brd:  { id, title, url, content? }
epic: { key, summary }
```

### D. Log batch inventory

Print:

```text
PDLC Build Plan
===============
Epic / PRD / BRD lines…
Stories (order): 1. FP-… — summary …
```

### E. Plan and approve (skip if `--unattended`)

Run **[`sp.feature-build.md`](sp.feature-build.md) Steps 2–4** verbatim:

- Switch to Plan Mode; build the Cursor plan (§3a–3c there).
- **PDLC-only additions to the plan:** (1) cover **all** queued stories in order; (2) under context, summarize §C per story (**Jira-first**; Confluence excerpts **only if** §C fetched those pages — never paste full dumps); (3) todos must list **per story** the same phase chain as that doc’s §3c, with concrete invocations for §F.
- Wait until the developer says **"implement the plan"** (that doc Step 4).

### F. Execute per story

After §E approval, or immediately after §D if `--unattended`.

For **each** story in order:

1. Derive `--name` slug from summary (kebab-case, 2–4 words; strip “As a buyer…”, etc.).
2. Build **enriched trailing context** for `/sp.specify` (Jira is always included; Confluence body only if §C fetched):

```text
Context from Jira:
- Story: <key> — <summary>
- Acceptance criteria & technical notes: <from description>

PRD/BRD traceability (links from Jira; add excerpts only if Confluence was fetched in §C):
- PRD: <id> — <title> (<url>) [optional: short excerpts]
- BRD: <id> — <title> (<url>) [optional: short excerpts]

Design: <Figma URL if any>
```

3. Run **`/sp.feature-build`** with the same arguments the §E plan lists for that story (typically):

```text
/sp.feature-build --name "<slug>" --jira <STORY-KEY> [--no-jira-status] "<enriched context>"
```

**Execution mode:** §E already produced the batch Cursor plan and the user approved it. For each story, perform **`sp.feature-build.md` Step 5** for that story only—run the SDD phase todos **through Phase 9** **without** repeating **`sp.feature-build` Steps 2–4** (Plan Mode + plan review) unless the user explicitly asks to replan that story.

**One PR per story.** **Jira dependencies (`Depends on: FP-XX` / Blocks links):** do **not** require the blocker story’s PR to be **merged to `main` first**. After the blocker’s **feature branch exists on `origin`** (PR may still be open), **create this story’s branch from that branch** — stacked PRs:

1. `git fetch origin`
2. Resolve the blocker branch name (Jira PR comment, `specs/<feature>/`, or ask the developer).
3. `git checkout -b <this-story-branch> origin/<blocker-branch>` (or equivalent), **then** run `/sp.specify` / `create-new-feature.sh` on that tip.

Open the dependent PR with **base branch = blocker branch** (not `main`) until the stack lands; document **Depends on** the blocker PR in the PR body. Merge **blocker first**, then rebase or retarget the dependent PR to `main` per team practice. Note branch stacking in the §E plan.

**Inner loop:** scoped `go build` while iterating; full `./scripts/go-workspace-verify.sh` where that doc says (Phase 5.5).

#### F.1 Ship — create the PR (required)

A PDLC story is **incomplete** until **`sp.feature-build.md` Phase 9** finishes:

1. Push the feature branch to the remote.
2. **Open a pull request** (`gh pr create`, hosting UI, or [`sp.git.commit_pr.md`](sp.git.commit_pr.md) if that is the repo workflow).
3. Capture and record the **PR URL** for §G and the §H batch report.

If the developer uses a stacked branch, put that in the PR description (base branch / depends on PR-#).

### G. Jira after each PR (comment + workflow)

Use Atlassian MCP **`addCommentToJiraIssue`** (`cloudId` from `getAccessibleAtlassianResources`, `issueIdOrKey` = **story key**, `commentBody` in **markdown**, optional `contentFormat: "markdown"`).

**Required for every story** that has a Jira key (all normal `/sp.pdlc` runs): post a comment **as soon as the PR exists**, for example:

```markdown
**PR opened** (PDLC / feature-build)

- **PR**: <full PR URL>
- **Branch**: `<branch-name>`
- **Spec**: `specs/<feature-dir>/`
- **Note**: Story is **In Review** while the PR is open. **Merge** and moving the Story to **Done** are handled by the senior engineer (or merge owner), not by the PR author as part of PDLC.
```

**`--no-jira-status`**: still **MUST** add the comment above (traceability). Only **skip** assignee changes and **workflow transitions** that [`sp.feature-build.md`](sp.feature-build.md) Phase 9 ties to Jira.

**With Jira workflow updates** (default, no `--no-jira-status`): after the comment, follow **`sp.feature-build.md` Phase 9** — e.g. `getTransitionsForJiraIssue` → `transitionJiraIssue` toward **In Review** (or equivalent). Do **not** transition the **Story** to **Done** here; that happens **after merge** by whoever merges (see §G.1).

#### G.1 Merge and Story → Done (out of scope for PDLC)

**`/sp.pdlc` does not include merging the PR or transitioning the Story to Done.** The **senior engineer** (or designated merge owner) reviews, merges the PR, and updates Jira (**Done** / **Closed**) per team practice. The PR author’s PDLC obligation stops at **§F.1** and **§G** (open PR + traceability comment + **In Review** when applicable).

Teams that want automation may still follow **`sp.feature-build.md` Phase 10** when the merge happens, but that step is **not** part of the `/sp.pdlc` run for the developer who raised the PR.

#### G.2 Sub-tasks during implementation

While executing §F, **`tasks.md`** should list **`(Jira: FP-xx)`** on rows (see **`sp.tasks.md`**). When a subtask’s slice is **committed**, transition that **Sub-task** to **Done** per **`sp.implement.md`** — do not wait until the PR merges for children.

### H. Batch report and testing

When the queue finishes (or stop on first failure):

```text
PDLC Build Complete
===================
Stories: ✅ / ⏳ / ⬚ + PR URLs
Confluence fetched: <none | page ids> (see §C — default is none when Jira suffices)
```

#### How to test (simple)

At the end of the run, give the developer this **short checklist** (adjust services/ports to what the stories changed).

**Important:** `go test`, `go vet`, and `go-workspace-verify.sh` only exercise **in-process / compile-time** checks. They **do not** prove that a **live HTTP route or gRPC RPC** behaves correctly. Always include **§H.1 endpoint verification** when the story adds or changes an API surface.

##### H.1 Static checks (no running endpoint)

- Per touched module, e.g. `cd src/<service> && go test ./... && go vet ./...`.
- From repo root, when many `go.work` modules changed: `bash ./scripts/go-workspace-verify.sh` (may still fail on unrelated modules; scope to what you changed when needed).

##### H.2 Build and run the service (required to test endpoints)

1. **Build** — e.g. `cd src/<service> && go build -o /tmp/<service> .` **or** from repo root `skaffold build -b <image>` where `<image>` is from [`skaffold.yaml`](../../skaffold.yaml) (`orderservice`, `frontend`, …).
2. **Run** — start the process or container with the env vars and DB/deps that service expects (see service `main.go` / manifests). Examples:
   - **Binary**: run the built binary on the service’s default port (e.g. **orderservice** → **5250**; set `DATABASE_URL` / `DISABLE_TRACING` / `DISABLE_PROFILER` as needed).
   - **Cluster**: `skaffold dev -b orderservice` or `skaffold run -b orderservice` (full stack: `skaffold dev`).
   - **Frontend + backends**: `bash scripts/run-frontend-local.sh` when the change is HTTP-facing.

##### H.3 Endpoint smoke (call the real route or RPC)

After the service is **listening**:

- **gRPC**: e.g. `grpcurl -plaintext -proto pb/demo.proto -d '{"user_id":"…", …}' localhost:5250 hipstershop.OrderService/ListOrders` — extend `-d` with new fields (`status_filter`, `search_query`, etc.) when the story changed the request message.
- **HTTP**: e.g. `curl -sf http://localhost:8080/_healthz` and the route under test (`GET /orders?…` for My Orders).

Confirm status codes / response shape / errors (e.g. `INVALID_ARGUMENT`) match the spec.

Details, port-forwards, and copy-paste examples: [`.cursor/rules/active-rules/local-dev.mdc`](../rules/active-rules/local-dev.mdc).

---

## Faster runs

| Goal | Action |
|------|--------|
| One story | `--jira EPIC --story KEY` or use `/sp.feature-build` only |
| Less Jira API | `--no-jira-status` |
| Skip §E gate | `--unattended` (use sparingly) |
| Skip Confluence API | §C — use Jira only when it is spec-ready; still record PRD/BRD URLs from links |

## Key rules (PDLC-only)

1. **Authoritative SDD** — [`sp.feature-build.md`](sp.feature-build.md); no duplicate phase definitions here.
2. **PR required** — each story ends with a **merged-ready PR** (Phase 9); no “implementation only” handoff without a PR URL.
3. **Jira comment required** — each story’s issue gets **`addCommentToJiraIssue`** with the **PR link** (and branch + spec path) when the PR exists, even with `--no-jira-status`.
4. **Sub-tasks → Done on commit** — Jira **Sub-tasks** when their **`tasks.md`** slice is committed (**§G.2**). **Story → Done after merge** is **not** executed by `/sp.pdlc` (see **§G.1**); the merge owner handles it.
5. **Confluence** — fetch bodies only per §C. When not fetched, pass **Jira + PRD/BRD URLs** into `/sp.feature-build`. If a **required** fetch fails, **WARNING** and fall back to Jira text.
6. **Dependency order** — respect Jira `Depends on` / queue order: **branch from the blocker’s feature branch** once it exists on `origin` (stacked PRs). Do **not** gate starting work on merging the blocker PR to `main` unless the team explicitly requires it.
7. **Fail-fast** — if one `/sp.feature-build` fails, stop the queue.
8. **Idempotency** — if a story already has an open PR, ask whether to skip or continue; if continuing, still ensure the ticket has the current PR link (comment or update).

## Error handling

| Case | Action |
|------|--------|
| Jira MCP down | **ERROR** |
| Epic empty | **ERROR** |
| Confluence missing | If §C did not require fetch: **OK** (Jira-only). If required fetch fails: **WARNING**, degraded context |
| `/sp.feature-build` fails | **STOP**, report phase; wait for user |

## Related commands

| Command | Use when |
|---------|----------|
| `/sp.feature-build` | Single story; no PDLC discovery |
| `/sp.prd` / `/sp.backlog` | Upstream of PDLC |
| `/sp.specify` | Spec only |
