---
description: End-to-end feature build using Plan Mode as the SDD orchestrator. Optional Jira (--jira) or Linear (--linear issueid) MCP for ticket-backed specs. Runs the full SDD lifecycle (specify → plan → tasks → analyze → implement → review → commit → sync → ship → post-merge Jira Done), marks Jira Sub-tasks Done when their tasks.md slices are committed, and delivers a PR-ready feature.
---

## Usage

```
/sp.feature-build --name "<kebab-slug>" --jira TST-12
/sp.feature-build --name "<kebab-slug>" --linear ANU-5
/sp.feature-build --name "<kebab-slug>" --jira PROJ-123 "Optional extra context for spec/plan"
/sp.feature-build --name "<kebab-slug>" --jira TST-12 --no-jira-status
/sp.feature-build --name "<branch-name>" <feature description>
/sp.feature-build <feature description>
```

**Quick Start**:
- **`--jira <KEY>`** or **`--ticket <KEY>`** (alias **`-j`**): load the issue via **Atlassian MCP** (`getJiraIssue`); use as **source of truth** for **`/sp.specify`** (same rules as [`sp.specify.md`](sp.specify.md)). Unless **`--no-jira-status`**: set **assignee** to the current MCP user and transition to **In Progress**.
- **`--linear <issueid>`**: Linear issue identifier passed to **Linear MCP** **`get_issue`** as `id` (e.g. `ANU-5` / `TEAM-42`, or UUID if your Linear integration requires it). Same behavior as [`sp.specify.md`](sp.specify.md) **`--linear <issueid>`**.
- Do **not** pass both **`--jira`** and **`--linear`** in one invocation — **ERROR** and ask the user to choose one ticket source.

**Quick Start (branch naming)**:
- Use `--name` for the feature **slug** (e.g., `--name "notification-status"`); numbering follows `create-new-feature.sh` / existing specs (see Step 1).
- Omit `--name` for auto-numbered naming derived from the description (or from ticket title when using Jira/Linear)

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Purpose

This command is the **single entry point** for building a complete feature from design input to final deliverable. It orchestrates the entire SDD lifecycle through Cursor Plan Mode, so the developer describes the feature once, reviews the plan once, and says "implement" — the AI handles the rest.

## Outline

### Step 1: Parse Arguments

Extract from `$ARGUMENTS` (flags order-agnostic; remaining text = optional **free-form description**):

| Flag | Meaning |
|------|--------|
| `--name "<kebab-slug>"` | Feature short name passed to **`/sp.specify`** / `create-new-feature.sh` (see `sp.specify.md`). |
| `--jira <KEY>` or `--ticket <KEY>` | Jira issue key; **MUST** fetch via Atlassian MCP (`getJiraIssue`). **Aliases:** `-j` → `--jira`. |
| `--linear <issueid>` | Linear issue identifier for **`get_issue`** (`id` parameter). |
| `--no-jira-status` | With Jira: fetch issue for spec content but **do not** transition workflow **or** set assignee (see `sp.specify.md`). |

**Ticket vs free text**

- If **`--jira`** or **`--linear`** is set: ticket **summary, description, acceptance criteria** (and URLs from the API) are the **primary** source of truth for the feature. Append **Jira:** / **Linear:** traceability in **`spec.md`** per `sp.specify.md`. Any remaining free text **supplements** the spec (do not ignore it).
- If **both** `--jira` and `--linear` are present: **ERROR** — use only one ticket source per run.
- If **neither** ticket flag is set: use free-form description only (existing behavior).
- If after parsing there is **no** `--name`, **no** usable description, and **no** successful ticket fetch: **ERROR** — provide `--name` + description, `--jira`, or `--linear`.

If `--name` is not provided, generate a branch slug:

1. Prefer deriving a 2–4 word **kebab-slug** from the **Jira summary** or **Linear title** if a ticket was fetched; else from the free-form description (action–noun style).
2. Scan `specs/` and branches per **`sp.specify.md`** / `create-new-feature.sh` rules for the next number for that slug.
3. Result: `{next-number}-{short-name}` (e.g., `002-notification-status`).

**Pass-through for later steps:** Keep resolved values for the execution phase:

- `BRANCH_SLUG` / `branch-name` (kebab short name, with or without numeric prefix in todos—match how `sp.specify` names the branch after `create-new-feature.sh`).
- `JIRA_KEY` (if any), `LINEAR_ISSUEID` (if any), `NO_JIRA_STATUS` (boolean); append **`--no-jira-status`** on **`/sp.specify`** when set.

**Jira story depends on another (`Depends on: FP-XX`, Blocks, or linked issue):**

- Do **not** require the blocker’s PR to be merged to `main` before starting this feature.
- **Before Phase 1 (`/sp.specify`)**: `git fetch origin`; create the new branch **from `origin/<blocker-feature-branch>`** (stacked PR). Resolve `<blocker-feature-branch>` from the blocker’s Jira PR comment, remote branches, or `specs/` folder name — **AskQuestion** if ambiguous.
- Open the PR with **base = blocker branch**; merge order: blocker PR first, then dependent (or retarget to `main` after blocker merges), per team workflow.

### Step 2: Switch to Plan Mode

**You MUST switch to Plan Mode immediately** using the `SwitchMode` tool with `target_mode_id: "plan"`.

Explain to the user:
> "Switching to Plan Mode to design the feature before writing any code. I'll create a comprehensive plan with architecture, design decisions, and SDD phase todos. Review it, ask questions, request changes — then say **'implement the plan'** to kick off execution."

### Step 3: Create the Cursor Plan (in Plan Mode)

In Plan Mode, create a comprehensive plan that covers:

#### 3a. Context Gathering

- Read `.specify/memory/constitution.md` (project principles)
- Read `.specify/memory/master-spec.md` (existing capabilities)
- Read `design-inputs/` for relevant contracts and schemas
- **Use the `context-stack` MCP** when available (see [`.cursor/rules/context-stack.md`](../rules/context-stack.md)) — a few targeted lookups to ground the plan in real code + real org knowledge. Where doc and code disagree, **code wins** for current behavior:
  - `get_context("<feature description in 1 sentence>")` — hybrid pack: code + relevant PRDs/BRDs/ADRs/Jira in one shot.
  - `search_specs("<feature short-name OR domain>")` — sibling specs/plans/tasks to mirror or extend (and to avoid duplicate features).
  - `search_code("<feature keyword OR likely symbol>")` — existing implementations to extend; real file paths to seed plan §File Structure.
  - `get_dependencies("<service or major symbol the plan will modify>")` — blast radius for each major touchpoint; informs §Architecture, §Testing Strategy, and parallelism in tasks.
  - `search_docs("<feature topic> ADR OR runbook")` — prior architectural decisions and operational guidance; cite them in §Design Decisions instead of re-deciding.

#### 3b. Plan Content

The plan MUST include:

1. **Overview**: What's being built and why
2. **Current State**: What exists in the codebase relevant to this feature
3. **Design Decisions**: Key architectural choices with rationale (e.g., endpoint path, auth strategy, data model, adapter pattern)
4. **Architecture Diagram**: Mermaid flowchart showing component interactions
5. **UI Design** (if feature has user-facing screens): Whether to run `/sp.design` to create Figma mockups before implementation. Recommend design phase when PRD Section 6 (UI Requirements) is non-empty or when the feature adds/modifies frontend routes.
6. **File Structure**: List of files to create/modify
7. **Environment Variables**: Any new config needed
8. **Testing Strategy**: Unit, E2E, functional test approach
9. **Constitution Check**: Which of the 17 principles apply and how they'll be satisfied

#### 3c. SDD Phase Todos

The plan MUST include these todos (the AI will execute them in order when the user says "implement the plan"). **Substitute** `{branch-name}` with the resolved slug; **substitute** the specify line with the correct flags:

- Ticket-backed: `Phase 1: Run /sp.specify --name '{branch-name}' --jira {KEY} [--no-jira-status]` **or** `/sp.specify --name '{branch-name}' --linear {issueid}`
- Text-only: `Phase 1: Run /sp.specify --name '{branch-name}' {optional quoted description}`

```yaml
todos:
  - id: specify
    content: "Phase 1: Run /sp.specify with --name '{branch-name}' and --jira/--linear/free text per parsed $ARGUMENTS — create feature branch, write spec.md, Jira/Linear traceability, PHR"
    status: pending
  - id: plan
    content: "Phase 2: Run /sp.plan — create architecture plan, research.md, data-model.md, quickstart.md, contracts/"
    status: pending
  - id: design
    content: "Phase 2.5 (conditional): Run /sp.design --spec specs/{branch-name}/spec.md — create Figma screens, design.md, attach to Jira. SKIP if feature is backend-only (no frontend routes or UI changes)."
    status: pending
  - id: tasks
    content: "Phase 3: Run /sp.tasks — break down into granular implementation tasks (TDD red-green-refactor order). If design.md exists, include UI implementation tasks referencing Figma frames."
    status: pending
  - id: analyze
    content: "Phase 4: Run /sp.analyze — cross-artifact consistency check; inject remediation tasks into tasks.md before implementation"
    status: pending
  - id: implement
    content: "Phase 5: Run /sp.implement — build all layers. Reference design.md for UI implementation."
    status: pending
  - id: verify
    content: "Phase 5.5: Build Verification — ./scripts/go-workspace-verify.sh (+ proto/docker table per sp.feature-build); MUST pass before /sp.review and commit"
    status: pending
  - id: review
    content: "Phase 6: Run /sp.review — CodeRabbit + Semgrep on uncommitted changes (do NOT pass no-semgrep); inject CR-* and SG-* findings; fix via /sp.implement and re-review until clean (max 3 passes)"
    status: pending
  - id: commit
    content: "Phase 7: Commit all changes (git add -A && git commit) after review is clean"
    status: pending
  - id: sync
    content: "Phase 8: Run /sp.sync-master-spec — sync business logic into master-spec.md"
    status: pending
  - id: ship
    content: "Phase 9: Push branch and create PR with summary and test plan; Jira PR comment per sp.pdlc §G; In Review if not --no-jira-status"
    status: pending
  - id: post_merge_jira
    content: "Phase 10: After PR merges — transition Jira Story to Done (unless --no-jira-status); optional merge comment"
    status: pending
```

### Step 4: Developer Reviews the Plan (Still in Plan Mode)

Wait for the developer to:
- Ask questions about design decisions
- Request changes ("put this under /private/", "add pagination", etc.)
- Confirm the approach

Update the plan based on feedback. Do NOT proceed to implementation until the developer explicitly says to implement.

### Step 5: Execute the Plan (Agent Mode)

When the developer says **"implement the plan"** (or similar), switch to Agent Mode and execute each todo in order:

#### SDD-TIMELINE.md — all phases (not only `/sp.plan`)

**`FEATURE_DIR/SDD-TIMELINE.md` is mandatory for feature-build.** It is **not** owned solely by `sp.plan`: almost every SDD command defines the same **“SDD execution timing (Option A — `SDD-TIMELINE.md`)"** section — create the file from **`.specify/templates/sdd-timeline-template.md`** if missing, then append **`| Phase | Started | Completed |`** rows (UTC) when **that** step finishes.

| Feature-build phase | What runs | Where timeline rules live |
|---------------------|-----------|---------------------------|
| 1 | `/sp.specify` | [`sp.specify.md`](sp.specify.md) — Option A |
| 2 | `/sp.plan` | [`sp.plan.md`](sp.plan.md) — Option A |
| 2.5 | `/sp.design` (conditional — UI features only) | Append **design** row; skip if backend-only |
| 3 | `/sp.tasks` | [`sp.tasks.md`](sp.tasks.md) — Option A |
| 4 | `/sp.analyze` | [`sp.analyze.md`](sp.analyze.md) — Option A |
| 5 | `/sp.implement` | [`sp.implement.md`](sp.implement.md) — Option A |
| 5.5 | Build Verification (`./scripts/go-workspace-verify.sh` + applicable proto/docker checks) | Append **verify** row |
| 6 | `/sp.review` | [`sp.review.md`](sp.review.md) — Option A |
| 7 | `git commit` (this command uses raw git, not `/sp.git.commit_pr`) | Append **commit** row using the **same** Phase / Started / Completed UTC pattern as Option A |
| 8 | `/sp.sync-master-spec` | [`sp.sync-master-spec.md`](sp.sync-master-spec.md) — Option A |
| 9 | `git push` / `gh pr create` | Append **ship** row (same table format) |
| 10 | Jira Story → **Done** after PR **merged** | Append **post_merge_jira** row when executed |

[`sp.clarify.md`](sp.clarify.md) and [`sp.git.commit_pr.md`](sp.git.commit_pr.md) also include Option A when those commands are used on their own.

1. **Phase 1 — Specify**: Run **`/sp.specify`** with the **same** flags parsed in Step 1 (do not drop ticket context):
   - With **`--jira KEY`**:  
     `/sp.specify --name "<kebab-slug>" --jira KEY` plus any trailing free text; add **`--no-jira-status`** if the user passed it.  
     Before or as part of specify: use Atlassian MCP **`getJiraIssue`**; **assignee** + **In Progress** per **`sp.specify.md`** unless **`--no-jira-status`**.
   - With **`--linear <issueid>`**:  
     `/sp.specify --name "<kebab-slug>" --linear <issueid>` plus optional free text. Linear MCP **`get_issue`** with `id` = **`<issueid>`** for title/description.
   - **Text-only:**  
     `/sp.specify --name "<kebab-slug>" {feature description}`  
   - Creates `specs/{NNN}-{slug}/spec.md` (actual folder from `create-new-feature.sh` JSON), user stories, FRs, SCs; Jira/Linear link in front matter when applicable
   - Records PHR per `sp.specify.md`

2. **Phase 2 — Plan**: Run `/sp.plan`
   - Creates `plan.md`, `research.md`, `data-model.md`, `quickstart.md`, `contracts/`
   - Runs constitution check (17/17 must pass)

2.5. **Phase 2.5 — Design** (conditional): Run `/sp.design --spec specs/{branch-name}/spec.md`

   **When to run**: The feature adds or modifies user-facing screens (frontend HTTP routes, HTML templates). Check the spec for user stories that mention UI, pages, forms, or screens.

   **When to skip**: Backend-only features (new gRPC services, proto contracts, data migrations). Log: `"Phase 2.5: Design — SKIPPED (backend-only feature)"`

   **What it does**:
   - Creates Figma screens from the spec's user stories and UI requirements
   - Generates `specs/{branch-name}/design.md` with Figma frame links per screen
   - Attaches design links to the Jira story (if `--jira` was used)
   - If Figma MCP is not connected: **WARN** and skip (do not block the pipeline)

   **Design → Tasks handoff**: When `design.md` exists, `/sp.tasks` will include UI implementation tasks that reference specific Figma frames, ensuring each screen and state has a corresponding template and handler.

   **Capture baseline**: If no Figma reference exists yet (`docs/design/design.md` is missing or has no captured frames), run `/sp.design-capture` first to capture existing screens as a style baseline before designing new ones.

3. **Phase 3 — Tasks**: Run `/sp.tasks`
   - Creates `tasks.md` with ordered, dependency-aware tasks
   - TDD phases: Red (write failing tests) → Green (implement) → Refactor
   - If `design.md` exists, include tasks for each screen: create template, wire handler, implement states

4. **Phase 4 — Analyze**: Run `/sp.analyze`
   - Cross-artifact consistency check across spec.md, plan.md, and tasks.md
   - Detects coverage gaps, ambiguities, constitution violations, terminology drift
   - If CRITICAL or HIGH findings exist:
     - Inject remediation tasks into `tasks.md` (append an "Analysis Remediation" phase)
     - These tasks will be resolved during implementation
   - If only LOW/MEDIUM findings: log them and proceed
   - This is a **quality gate** — catching issues here avoids rework during implementation

5. **Phase 5 — Implement**: Run `/sp.implement`.
   - Executes all tasks from `tasks.md` in order (including any remediation tasks from analyze)
   - Follows TDD Red-Green-Refactor cycle
   - Achieves >90% test coverage
   - Creates Postman collection and update scripts/curl-local-test.sh scripts if present else create the same
   - **Jira Sub-tasks** (`--jira`, not `--no-jira-status`): when a `tasks.md` row maps to a **Sub-task key** (see **`sp.implement.md`** and **`sp.tasks.md`** — e.g. `(Jira: FP-80)`), transition that issue to **Done** after the **commit** that completes that slice (Phase 7 — not only when the whole PR merges)

6. **Phase 5.5 — Build Verification (before review and commit)**

   After implementation, run **applicable checks** for what you changed before committing. This is a **quality gate** — if any check fails, fix it before proceeding.

   **Scan changed/created files** and run the matching checks:

   | Trigger (files exist / changed) | Verification | Command |
   |--------------------------------|--------------|---------|
   | Any Go change under `src/<svc>/` (`*.go`, `go.mod`, `*_test.go`) | Workspace build + vet + test (one pass; modules from `go.work`) | From repo root: `./scripts/go-workspace-verify.sh` |
   | `package.json` | Node install + build | `npm ci && npm run build` (if build script exists) |
   | `requirements.txt` / `setup.py` | Python deps install | `pip install -r requirements.txt` |
   | `*.proto` modified | Proto / genproto | `protoc` per service `genproto.sh`; run `genproto.sh`; `go mod tidy` in affected `src/<svc>/` if imports changed; then `./scripts/go-workspace-verify.sh` from repo root |
   | `*.html` templates created/modified | Template wiring | `{{ define "..." }}` matches `ExecuteTemplate()` names; templates render via `bytes.Buffer`, not straight to `ResponseWriter` |
   | `src/frontend/templates/*.html` or `src/frontend/static/styles/*.css` | Design fidelity | Match **`specs/*/design.md`** tokens and **`.cursor/rules/active-rules/design-patterns.mdc`** (**Token-accurate implementation**); use feature CSS / shared badge patterns — do not substitute generic Bootstrap colors for token-defined badges or teal accent |

   **On failure:** fix, re-run the failed check, then proceed to **`/sp.review`** only when applicable checks pass.

   **Report format** (append to completion output):
   ```
   Build Verification:
     ✅ ./scripts/go-workspace-verify.sh (go build + vet + test for all `go.work` modules)
   ```

7. **Phase 5.6 — Smoke Test (when service is running)**

   If the service is running locally (via `scripts/run-*-local.sh` or `skaffold dev`), execute quick smoke tests:

   | Target | Test | Pass criteria |
   |--------|------|---------------|
   | HTTP endpoints (new/changed) | `curl -s -o /dev/null -w "%{http_code} size=%{size_download}" <url>` | HTTP 200 **and** `size_download > 0` |
   | gRPC endpoints (new/changed) | `grpcurl -plaintext -proto pb/demo.proto -d '<json>' <addr> <method>` | Expected status code; non-empty response |
   | Error paths | `curl` / `grpcurl` with invalid input | Proper error response, not blank page or panic |

   This step is best-effort — skip if no local instance is available, but **always** run when available.

   **Phase 5.7 — Design Re-sync** (conditional): If the feature modified frontend templates/routes and the frontend is running locally, run `/sp.design-capture --update --pages "<changed-routes>"` to re-sync the Figma reference with the implemented UI. Skip if backend-only or no local instance.

8. **Phase 6 — Review**: Run **`/sp.review`** (full command — **not** CodeRabbit alone)
   - **CodeRabbit** reviews **uncommitted** changes (`-t uncommitted` when the working tree has changes — same as standalone `/sp.review`; see **`sp.review.md`**)
   - **Semgrep** MUST run the same pass (MCP `semgrep_scan` or CLI `semgrep scan`) on code scope per `sp.review.md` — **do not pass `no-semgrep`** in feature-build
   - If Semgrep cannot run (MCP + CLI both unavailable), **WARN** prominently in the review summary (`Semgrep: SKIPPED — not available`) and proceed with CodeRabbit-only review — this is an environment gap, not a code issue
   - Inject **CR-*** and **SG-*** findings into `tasks.md`
   - Fix critical/high findings via **`/sp.implement`** (or minimal direct edits), then re-run **`/sp.review`** until clean
   - Max 3 review iterations per change set

9. **Phase 7 — Commit**: Stage and commit all changes after review is clean
   - `git add -A && git commit -m "feat: {feature description}"`

10. **Phase 8 — Sync**: Run `/sp.sync-master-spec`
   - Maps feature to master-spec.md capability sections
   - Generates business-level documentation
   - Amend commit if master-spec changes were made

11. **Phase 9 — Ship**: Push branch and create PR
    - `git push -u origin HEAD`
    - `gh pr create` (or repo-standard PR flow) with summary and test plan
    - Return PR URL to the developer
    - **Jira (`--jira`)**: after the PR URL exists, **`addCommentToJiraIssue`** is **required** — PR link, branch, and `specs/…` path (markdown template: **`sp.pdlc.md` §G**). This applies **even with** **`--no-jira-status`** (comment is traceability, not a workflow transition).
    - **Jira (`--jira`)** when **`--no-jira-status` is not set**: then call `getTransitionsForJiraIssue` → `transitionJiraIssue` to a **review** state (In Review, Code Review, Review, Ready for Review, etc.). **Do not** transition the **Story** to **Done** here — that happens **after the PR merges** (step 12).

12. **Phase 10 — Post-merge Jira (Story → Done)**  
    When the GitHub PR is **merged** (user confirms, or `gh pr view <url> --json state` shows `MERGED`, or default branch contains the merge), and **`--jira`** is set **without** **`--no-jira-status`**:
    - Call **`getTransitionsForJiraIssue`** on the **Story** key → **`transitionJiraIssue`** to **Done** / **Closed** / the team’s **terminal** workflow state (pick the available transition whose name matches **Done** or **Closed** first).
    - Optionally **`addCommentToJiraIssue`**: e.g. `PR merged: <url>`.
    - **`--no-jira-status`**: skip **Done** transition; optional merge comment only if the team wants traceability.

Mark each todo `in_progress` → `completed` as you go. If a phase fails, stop and report.

## Key Rules

- **Jira / Linear MCP**: If Atlassian or Linear MCP is missing or a call fails, follow **`sp.specify.md`** failure rules (e.g. Jira transition failures are **WARNING**, not fatal; missing issue fetch may **ERROR** if there is no fallback description). **Jira assignee**: with **`--jira`** and without **`--no-jira-status`**, **`/sp.specify`** sets assignee to the **current Atlassian user** when moving to **In Progress** (see `sp.specify.md`). **Jira Story lifecycle**: **In Review** after PR opens (Phase 9); **Done** after **PR merges** (Phase 10). **Jira Sub-tasks**: transition to **Done** when their mapped **`tasks.md`** work is **committed** (Phase 7), not deferred to merge — see **`sp.implement.md`**.
- **Jira `Depends on`**: Branch from the blocker’s **feature branch** (stacked PR); do **not** require merging the blocker PR to `main` before opening the dependent PR (see Step 1).
- **Plan Mode first, always**: Never skip straight to implementation
- **Developer approval required**: Don't implement until the developer confirms the plan
- **All phases through 10**: Every phase must run — including Phase 5.5 build verification before **`/sp.review`** (analyze before implement, **review before commit** are still critical). **Phase 10** runs when the PR is **merged** (may be a follow-up session).
- **SDD-TIMELINE.md (mandatory)**: Maintain **`FEATURE_DIR/SDD-TIMELINE.md`** across **all** phases — see **Step 5** table. Each of `sp.specify`, `sp.plan`, `sp.tasks`, `sp.analyze`, `sp.implement`, `sp.review`, and `sp.sync-master-spec` documents its own Option A section; **commit** and **ship** rows are appended by the feature-build executor when those steps are not delegated to `sp.git.commit_pr`.
- **Phase 6 Semgrep**: `/sp.review` includes **mandatory Semgrep** alongside CodeRabbit; waiving Semgrep with `no-semgrep` is **not** allowed in feature-build
- **Constitution compliance**: Every implementation decision must trace to a constitution principle
- **TDD mandatory**: Tests before implementation (Red-Green-Refactor)
- **>90% coverage**: Non-negotiable coverage target
- **No hardcoded values**: All config from `.env`
- **PII masking**: Phone numbers, credentials never in logs

## Error Handling

If any phase fails:
1. Stop execution
2. Report which phase failed and why
3. Suggest corrective action
4. Wait for developer input before retrying

## Success Output

```text
✅ Feature Build Complete!

📋 Branch: {branch-name}
📝 Spec: specs/{branch-name}/spec.md
🏗️ Plan: specs/{branch-name}/plan.md
🎨 Design: specs/{branch-name}/design.md → Figma: {URL} (or "SKIPPED — backend-only")
📊 Tasks: specs/{branch-name}/tasks.md (X tasks completed)
🔎 Analyze: X findings (Y critical, Z remediation tasks added)
🧪 Coverage: XX.X% (target: >90%)
🔍 Review: CodeRabbit — 0 critical findings | Semgrep — scan completed (0 findings or SG tasks listed)
📖 Master Spec: Synced (Section X.Y added)
🔗 PR: {PR URL}

Artifacts:
- specs/{branch-name}/ (spec, plan, design.md, tasks, research, data-model, quickstart, contracts, SDD-TIMELINE.md)
- history/prompts/{branch-name}/ (PHR records)
- postman/ (API collection)
- scripts/ (curl test scripts)
```

## Related Commands

| Command | When to Use Instead |
|---------|-------------------|
| `/sp.clarify` | Spec already exists but needs refinement; runs clarify → plan → ... → ship |
| `/sp.specify` | Only need to create/update a spec (no full build); **`--jira` / `--linear`** behavior is identical—this command forwards them |
| `/sp.analyze` | Only need a cross-artifact consistency check (no full build) |
| `/sp.implement` | Already have tasks.md, just need to code |
| `/sp.review` | Already have code, just need review |
| `/sp.sync-master-spec` | Already shipped, just need to update master spec |