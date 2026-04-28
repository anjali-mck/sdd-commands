---
description: Run CodeRabbit + Semgrep on code changes; inject findings as tasks into the feature's tasks.md (opt out Semgrep with no-semgrep).
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

- **Semgrep**: **On by default.** Run local Semgrep (MCP `semgrep_scan` when available; otherwise CLI `semgrep scan` if on PATH). **Opt out** with **`no-semgrep`** or **`semgrep-off`** in `$ARGUMENTS`.
- **Tasks**: Append findings to `tasks.md` by default. **Opt out** with **`report-only`**, **`no-tasks`**, or **`no-append`** (summary only; no file writes).
- **`scan all`**: If present in `$ARGUMENTS`, do not restrict to code-only paths; otherwise follow **Scope: code only** below.

## Context-stack MCP Navigation

Use the `context-stack` MCP server (see [`.cursor/rules/context-stack.md`](../rules/context-stack.md)) to understand the codebase context alongside CodeRabbit and to triage review findings.

| When | Tool (server: `context-stack`) | Purpose |
|------|--------------------------------|---------|
| Understanding changed symbols | For each path in `git diff --name-only`: `search_code("<filename>")` and `get_dependencies("<changed symbol>")` | Blast radius of uncommitted changes |
| Verifying fix impact | `get_dependencies("<symbol>")` for any symbol a finding wants you to modify | What else breaks if we change a finding |
| Finding affected tests | `get_dependencies("<changed symbol>")` then filter results to test files (`*_test.*`, `*.spec.*`, `tests/`) | Tests that need updating after fixes |
| Cross-referencing prior reviews / decisions | `search_docs("<rule_id> OR <CR finding gist>")` | Surfaces ADRs, runbooks, prior CR/SG fixes that inform the verdict |

## Overview

**Purpose**: Run **CodeRabbit** AI review and **Semgrep** static analysis on the same change scope, parse findings, and inject them as actionable tasks into the feature's `tasks.md` so `/sp.implement` can fix them (`CR-*` = CodeRabbit, `SG-*` = Semgrep).

**Workflow Position**: Run AFTER `/sp.implement`, BEFORE `/sp.git.commit_pr`

```
/sp.implement → /sp.review-2 → /sp.implement (fix issues) → /sp.git.commit_pr
```

## Scope: code only (default)

Unless **`scan all`** appears in `$ARGUMENTS`:

- **Include**: application source (e.g. `src/**`), tests next to code, and root app manifests changed in the diff (`package.json`, `pom.xml`, etc.).
- **Exclude**: `.cursor/`, `.specify/`, `specs/*.md`, `history/`, bare `*.md` / shell / CI config unless `scan all`.
- **CodeRabbit**: Restrict via CLI flags if supported (`--path`, `--include`); else filter findings in the parse step.
- **Semgrep**: Build `code_files` / CLI targets only from paths that pass the include rules above.

## Execution Flow

### 1. Prerequisite Checks

Run `.specify/scripts/bash/check-prerequisites.sh --json --require-tasks --include-tasks`
to identify the active feature and its FEATURE_DIR.

Verify:
- CodeRabbit CLI is installed: `cr --version` (if not found, try `~/.local/bin/coderabbit --version`)
- **Semgrep** (for non–`no-semgrep` runs): Semgrep MCP tools **or** `semgrep --version` on PATH for CLI fallback
- Active git repository: `git rev-parse --git-dir`
- Uncommitted changes exist: `git status --porcelain` (warn if no changes; Semgrep/CodeRabbit may still target last commit per step 3)
- FEATURE_DIR/tasks.md exists (required for task injection unless `report-only` / `no-tasks`)

### 2. Gather SDD Context Files

Collect specification context to pass to CodeRabbit via the `-c` flag:

```bash
CONTEXT_FILES=""

# Add spec context if available
[ -f "$FEATURE_DIR/spec.md" ] && CONTEXT_FILES="$CONTEXT_FILES -c $FEATURE_DIR/spec.md"
[ -f "$FEATURE_DIR/plan.md" ] && CONTEXT_FILES="$CONTEXT_FILES -c $FEATURE_DIR/plan.md"

# Add constitution for architectural principles
CONTEXT_FILES="$CONTEXT_FILES -c .specify/memory/constitution.md"

# Add CodeRabbit config
CONTEXT_FILES="$CONTEXT_FILES -c .coderabbit.yaml"
```

### 3. Execute CodeRabbit Review

Determine review target based on change state:

```bash
# Check if there are uncommitted changes
UNCOMMITTED=$(git status --porcelain)

if [ -n "$UNCOMMITTED" ]; then
  # Review uncommitted changes
  cr review --prompt-only -t uncommitted $CONTEXT_FILES 2>&1
else
  # No uncommitted changes — review the last commit instead
  cr review --prompt-only -t committed $CONTEXT_FILES 2>&1
fi
```

**Important**:
- Use `--prompt-only` for token-efficient, parseable output
- Use `-t uncommitted` when there are working-tree changes (standalone review)
- Use `-t committed` when changes are already committed (inside `sp.feature-build` flow, which commits before review)
- Review may take 5-30 minutes depending on the scope of changes
- Set `block_until_ms` to at least 1800000 (30 minutes) when executing
- Capture ALL output — CodeRabbit findings are in stdout

If CodeRabbit requires authentication:
1. Inform the user: "CodeRabbit requires authentication. Run `cr auth login` in your terminal."
2. Wait for user confirmation before retrying.

### 3b. Execute Semgrep

Skip this subsection entirely if `$ARGUMENTS` contains **`no-semgrep`** or **`semgrep-off`**.

**Goal**: Run a local Semgrep scan on the same **code scope** as above and capture structured findings for task injection.

**Preferred — Semgrep MCP** (when tools such as `semgrep_scan` exist; check MCP schema before calling):

1. Build **`code_files`**: absolute paths from `git diff --name-only HEAD`, `git diff --cached --name-only`, and (if no uncommitted code matches) existing files under `src/` — **only** paths that exist on disk and pass **Scope: code only** unless `scan all`.
2. Invoke **`semgrep_scan`** with `code_files` as `[{ "path": "<absolute>" }, ...]`.
3. **Batching**: If there are more than ~30 files, run multiple scans in chunks (e.g. 20–30 files) and **merge** all `results`.
4. **Optional** (non-blocking): `semgrep_findings` (AppSec platform) if the repo is connected; `semgrep_scan_supply_chain` when lockfiles/manifests changed. Merge into the same result set; dedupe by file + rule + line.

**Fallback — Semgrep CLI** (when MCP is unavailable or returns no tool):

```bash
# From repo root; scope to changed files when possible
semgrep scan --config auto --json --quiet 2>/dev/null
# Or restrict: pass explicit paths or use `git diff --name-only` to build args
```

Parse CLI **JSON** output: use the standard `results` array (or SARIF if you used `--sarif`); map each finding to path, start line, rule id (`check_id`), and message.

**Failures**: If Semgrep MCP and CLI both fail or are missing, **warn** once (`Semgrep: skipped — not available | error (non-blocking)`) and continue with CodeRabbit-only tasking.

**Timing**: Large repos may need long `block_until_ms` (e.g. 600000+); batch MCP calls to avoid payload limits.

### 4. Parse CodeRabbit Findings

Read the CodeRabbit output and categorize each finding:

**Severity Categories** (map CodeRabbit findings to task priority):
- **CRITICAL**: Security vulnerabilities, data loss risks, race conditions → Must fix
- **HIGH**: Logic errors, missing error handling, architectural violations → Must fix
- **MEDIUM**: Code quality, missing validation, incomplete patterns → Should fix
- **LOW/NIT**: Style issues, naming suggestions, minor improvements → Optional fix

**For each finding, extract**:
- File path and line number(s)
- Issue description
- Suggested fix (if provided)
- Severity/category

### 4b. Parse Semgrep Findings

Skip if Semgrep was skipped or returned no results.

For each Semgrep result (MCP or CLI JSON):

- **Path**, **start line** (and end if present)
- **Rule id** (`check_id` / rule name)
- **Message** (`extra.message` or equivalent)
- **Severity**: map Semgrep severity to task bands — `ERROR` → CRITICAL/HIGH, `WARNING` → MEDIUM, `INFO` → LOW/NIT (adjust if the payload uses different labels)

**Dedupe**: If a finding matches **same file + same line + same class of issue** as a CodeRabbit item from step 4, **prefer one task** — keep the CodeRabbit line and add a short note `(also Semgrep: <rule_id>)` rather than duplicating.

**Code-only filter**: Unless `scan all`, drop findings outside the allowed code paths (same rules as **Scope: code only**).

### 5. Inject Findings into tasks.md

If `$ARGUMENTS` contains **`report-only`**, **`no-tasks`**, or **`no-append`**, skip file writes; still print the summary from step 6.

Read the existing `$FEATURE_DIR/tasks.md` and append new section(s):

**CodeRabbit** — append:

```markdown
## CodeRabbit Review Findings (Review #{N})

**Review Date**: {ISO_DATE}
**Review Type**: uncommitted changes
**Scope**: {number of files changed}

### Critical / High Priority (Must Fix)

- [ ] CR-{N}.1 [P] Fix: {description} — `{file_path}:{line}`
- [ ] CR-{N}.2 Fix: {description} — `{file_path}:{line}`

### Medium Priority (Should Fix)

- [ ] CR-{N}.3 [P] Fix: {description} — `{file_path}:{line}`

### Low Priority / Nits (Optional)

- [ ] CR-{N}.4 [P] Nit: {description} — `{file_path}:{line}`
```

**Task ID format**: `CR-{review_number}.{finding_number}`
- `{review_number}` = 1 for first review, 2 for second, etc. (count existing `## CodeRabbit Review Findings` section headers)
- `{finding_number}` = sequential within the review

**Parallel markers**: Mark findings as `[P]` if they affect different files and have no dependencies.

**Semgrep** — after the CodeRabbit block (same run), append when there is ≥1 Semgrep finding after dedupe:

```markdown
## Semgrep Review Findings (Review #{M})

**Review Date**: {ISO_DATE}
**Source**: Semgrep MCP / CLI (as used)
**Scope**: {files scanned or "changed code + src fallback"}

### Critical / High Priority (Must Fix)

- [ ] SG-{M}.1 [Semgrep] Fix: {rule_id} — {message} — `{file_path}:{line}`
...

### Medium Priority (Should Fix)

- [ ] SG-{M}.x [Semgrep] ...

### Low Priority / Nits (Optional)

- [ ] SG-{M}.y [Semgrep] [P] Nit: ...
```

**Task ID format**: `SG-{semgrep_review_number}.{finding_number}`
- `{semgrep_review_number}` = 1 + count of existing `## Semgrep Review Findings` headers in `tasks.md`
- `{finding_number}` = sequential within this Semgrep review
- Prefix lines with **`[Semgrep]`** so `/sp.implement` can distinguish from CodeRabbit

**Alternative (align with `/sp.review`)**: If the feature already uses global **`SG###`** ids in `tasks.md`, continue that scheme instead: scan for the highest `SG` number and append `- [ ] SG### [Semgrep] ...` under a `## Phase: Semgrep review findings` section — **do not** mix `SG-{M}.n` and `SG###` in the same file; pick one style based on what is already present.

### 6. Summary Report

Display a summary to the user:

```
╔══════════════════════════════════════════╗
║     CodeRabbit + Semgrep Review Summary  ║
╠══════════════════════════════════════════╣
║  CodeRabbit review #: {N}                ║
║  Semgrep review #:    {M} (or — if skip)║
║  Files scanned:       {count}           ║
║                                         ║
║  CodeRabbit findings: {cr_total}       ║
║    Critical/High/Med/Low: {breakdown}   ║
║  Semgrep findings:     {sg_total}       ║
║    Critical/High/Med/Low: {breakdown}   ║
║                                         ║
║  Tasks injected into:                   ║
║    {FEATURE_DIR}/tasks.md               ║
║  (or report-only — no file write)       ║
╚══════════════════════════════════════════╝

Semgrep: used (MCP | CLI) | skipped (no-semgrep | unavailable | error)

Next steps:
1. Review the injected CR-* and SG-* tasks in tasks.md
2. Run /sp.implement to fix critical and high priority issues
3. Run /sp.review-2 again to verify fixes (max 3 `CodeRabbit` and 3 `Semgrep` review sections in `tasks.md` unless overridden)
```

### 7. Review Iteration Guard

Track review counts separately to prevent infinite loops:

- **CodeRabbit**: Count `## CodeRabbit Review Findings` section headers in `tasks.md`. If count >= 3, WARN and **do not** run CodeRabbit again unless the user explicitly overrides.
- **Semgrep**: Count `## Semgrep Review Findings` section headers. If count >= 3, WARN and **do not** run Semgrep again unless the user explicitly overrides.
- If one tool is blocked by the guard, still run the other when enabled and not past its own limit.

## Error Handling

| Error | Action |
|-------|--------|
| CodeRabbit CLI not installed | Provide install command: `curl -fsSL https://cli.coderabbit.ai/install.sh \| sh` |
| Authentication required | Prompt user to run `cr auth login` |
| No uncommitted changes | Auto-fallback to `-t committed` to review the last commit |
| tasks.md not found | Error: "Run /sp.tasks first to generate the task list" |
| CodeRabbit times out | Suggest reviewing a smaller scope with specific file paths |
| CodeRabbit returns no findings | Report clean review — no CR tasks to inject |
| Semgrep MCP/CLI missing or errors | Warn; inject SG tasks only if some scan succeeded; else Semgrep summary = skipped |
| Semgrep returns no findings | Report Semgrep clean for scanned scope — no SG tasks |

## Configuration

CodeRabbit behavior is configured in `.coderabbit.yaml` at the repo root.
Path-specific review instructions are defined there for:
- Adapter code (`src/adapters/**`)
- Domain code (`src/notifications/**`)
- Guards and decorators (`src/common/**`)
- Migrations (`src/database/migrations/**`)
- DTOs (`**/*.dto.ts`)
- Tests (`**/*.spec.ts`)

## Related Commands

- `/sp.implement` — Execute implementation tasks including `CR-*` and `SG-*` review items (run before re-review)
- `/sp.git.commit_pr` — Commit and create PR (run AFTER fixing review findings)
- `/sp.review` — Alternate review command (CodeRabbit + Semgrep with `CR###` / `SG###` phasing)
- `/sp.checklist` — Manual compliance checklist (complementary)
- `/sp.analyze` — Codebase analysis (complementary)

---

## SDD execution timing (Option A — `SDD-TIMELINE.md`)

1. Resolve **`FEATURE_DIR`**. Ensure **`SDD-TIMELINE.md`** exists (create from **`.specify/templates/sdd-timeline-template.md`** if missing). If legacy two-column only, add **Started** and use `—` on existing rows.
2. **Start timestamp:** Before substantive `/sp.review-2` work (CodeRabbit / Semgrep / task append), `date -u +"%Y-%m-%dT%H:%M:%SZ"` → `start_ts`.
3. Perform the main **`/sp.review-2`** work.
4. **Completed timestamp:** When that work finishes, `date -u +"%Y-%m-%dT%H:%M:%SZ"` → `completed_ts`.
5. **Append** `| review_2 | <start_ts> | <completed_ts> |` (re-runs append another row).
6. **Final reply:** paste the **full table** under **### SDD execution time**.

---

As the main request completes, you MUST create and complete a PHR (Prompt History Record) using agent-native tools when possible.

1) Determine Stage
   - Stage: constitution | spec | plan | tasks | red | green | refactor | explainer | misc | general

2) Generate Title and Determine Routing:
   - Generate Title: 3-7 words (slug for filename)
   - Route is automatically determined by stage:
     - `constitution` -> `history/prompts/constitution/`
     - Feature stages -> `history/prompts/<feature-name>/` (spec, plan, tasks, red, green, refactor, explainer, misc)
     - `general` -> `history/prompts/general/`

3) Create and Fill PHR (Shell first; fallback agent-native)
   - Run: `.specify/scripts/bash/create-phr.sh --title "<title>" --stage <stage> [--feature <name>] --json`
   - Open the file and fill remaining placeholders (YAML + body), embedding full PROMPT_TEXT (verbatim) and concise RESPONSE_TEXT.
   - If the script fails:
     - Read `.specify/templates/phr-template.prompt.md` (or `templates/...`)
     - Allocate an ID; compute the output path based on stage from step 2; write the file
     - Fill placeholders and embed full PROMPT_TEXT and concise RESPONSE_TEXT

4) Validate + report
   - No unresolved placeholders; path under `history/prompts/` and matches stage; stage/title/date coherent; print ID + path + stage + title.
   - On failure: warn, don't block. Skip only for `/sp.phr`.

--- End Command ---
