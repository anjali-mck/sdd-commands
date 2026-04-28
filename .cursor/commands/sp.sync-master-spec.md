---
description: Sync master-spec.md with the current feature implementation by analyzing spec.md and code changes
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Roam MCP Navigation

Use the `user-roam-code` MCP server to understand implementation details for accurate master spec sync.

| When | MCP Tool (`user-roam-code`) | Purpose |
|------|----------------------------|---------|
| Understand feature implementation | `roam_explore` (symbol) | Codebase overview + symbol deep-dive for implementation mapping |
| Get precise file context | `roam_context` (symbol) | Minimal files + line ranges for key symbols |
| Trace dependencies | `roam_deps` (path) | File-level imports to map integration patterns |
| Find all related symbols | `roam_search_symbol` (query) | Find symbols by name for comprehensive mapping |

## Outline

1. **Setup**: Run `.specify/scripts/bash/setup-plan.sh --json` from repo root and parse JSON for FEATURE_SPEC, SPECS_DIR, BRANCH. Extract JIRA ticket ID from BRANCH name (e.g., "scrum-8-menu-updates" → "SCRUM-8").

2. **Load context**: 
   - Read FEATURE_SPEC (the spec.md for current feature)
   - Read `.specify/memory/master-spec.md`
   - Read `documentation/master-spec-template.md` (integration pattern templates)
   - Read `.specify/memory/constitution.md` (Constitution - includes TDD and Active Rules principles)
   - Call `roam_explore` (MCP: `user-roam-code`) to understand codebase structure relevant to the feature
   - Call `roam_context` (MCP: `user-roam-code`) for key symbols to get precise file + line ranges

3. **Analyze implementation**:
   - Analyze FEATURE_SPEC to identify what was implemented
   - Map features to master-spec capability sections:
     - OB-001: Product Catalog (productcatalogservice)
     - OB-002: Shopping Cart (cartservice)
     - OB-003: Checkout & Order Placement (checkoutservice)
     - OB-004: Currency Conversion (currencyservice)
     - OB-005: Payment Processing (paymentservice)
     - OB-006: Shipping & Fulfillment (shippingservice)
     - OB-007: Email Notifications (emailservice)
     - OB-008: Product Recommendations (recommendationservice)
     - OB-009: Contextual Advertising (adservice)
     - OB-010: Frontend & User Experience (frontend)
   - Within each capability, identify impacted integration patterns:
     - gRPC APIs: New or modified RPC methods in `pb/demo.proto`
     - HTTP Routes: Frontend HTTP endpoint changes (GET/POST)
     - Proto Messages: New or modified message types
     - Kubernetes: New or modified manifests in `kubernetes-manifests/`
     - Configuration: New environment variables or Secrets
     - Health & Observability: Health check changes, new metrics, logging
   - Determine if each impacted integration pattern needs NEW entry or UPDATE to existing

4. **Generate documentation**:
   - Use templates from `documentation/master-spec-template.md`
   - For each impacted integration pattern, generate:
     - **User Story**: As a [user type], I want to [action] so that [benefit]
     - **Business Requirements**: 5-12 bullets with reasoning (WHAT and WHY in parentheses)
     - **Implementation Details**: Technical specifics (endpoint, DTOs, validation, SLOs, errors)
     - **JIRA line**: Extract JIRA ticket ID from branch name

5. **Update master-spec.md**:
   - Add new subsections to appropriate capability/integration pattern sections (maintain numbering: 1.6, 1.7, etc.)
   - Update Capability Catalogue table if new capability
   - Update Table of Contents if new capability added
   - Update Last Updated date in header

6. **Report**: Display what was updated with section numbers and brief summary

## Analysis Phase

### Step 1: Extract Feature Details

From FEATURE_SPEC, identify:
- **Feature Type**: API, Database, Event, Batch Job, Health Check, Webhook
- **Capability**: Which existing capability this belongs to (Restaurant Discovery, Menu Management, System Observability, Restaurant Notifications) or if it's a new capability
- **Endpoints**: Methods and paths (if API)
- **User Story**: From spec.md
- **Acceptance Criteria**: Convert to business requirements
- **Implementation**: Look for technical details in spec.md

### Step 2: Map to Master-Spec Structure

Create analysis report:

```text
📋 Analysis Results for [JIRA-ID]:

Capability: [Capability Name] ([Existing/New])
Integration Pattern: APIs

✅ APIs - [NEW/UPDATE]
   - Feature: [METHOD] /api/vX/path
   - Justification: [Why this belongs here]
   - Details: [Key implementation aspects]

✅ Database Access - [NEW/UPDATE]
   - Feature: [Table/Index name]
   - Justification: [Why this belongs here]
   - Details: [Schema changes]

⚠️ Asynchronous Messaging - NOT APPLICABLE
   - No event publishing/consuming in this spec
```

## Documentation Generation Phase

### Template Selection

**Always** reference `documentation/master-spec-template.md` for the correct template based on integration pattern type:
- APIs (REST Endpoints)
- Asynchronous Messaging (Event-Driven)
- Real-Time Communication (WebSockets/SSE)
- Webhooks (Outbound Notifications)
- Data Exports & Batch Operations
- Database Access (Direct/Shared)
- Health & Observability Endpoints

### Business Requirements Guidelines

**Format**: System [what it does] for [why it matters in parentheses]

**Coverage**:
1. Core functionality (what system does)
2. Versioning/audit trail (if applicable)
3. Data integrity (business rules)
4. Validation rules (data quality)
5. Edge cases (special scenarios)
6. User feedback (confirmations, notifications)
7. Prerequisites (dependencies)
8. Compliance requirements (regulations)
9. Performance expectations (from user perspective)
10. Scale limits (business constraints)

**Example**:
```markdown
- System creates a new menu version every time owner updates for version control (audit trail and compliance requirement)
- Previous menu versions are automatically archived but never deleted (historical record preserved for compliance and rollback capability)
- Only one menu version is active at any time so customers always see the latest validated menu (business rule for customer experience)
```

### Implementation Details Guidelines

**For APIs**:
- **Endpoint**: Full path with method
- **Method**: Idempotent/non-idempotent, safe/unsafe
- **Path/Query Parameters**: Types and descriptions
- **Request Body**: DTO name with key fields OR None
- **Response**: DTO name with key fields
- **Validation**: Bean/Custom/Database OR None
- **Transaction**: @Transactional details OR None
- **Query Logic**: Key WHERE/ORDER BY clauses (if complex)
- **Performance SLO**: P95/P99 < Xms for scenario
- **Error Codes**: 400, 404, 409, 500 with reasons

**For Database**:
- **Database**: PostgreSQL version
- **Schema**: Schema name
- **Tables**: Table names with key columns
- **Constraints**: Unique, FK, check constraints
- **Indexes**: Index names with purpose
- **Access Pattern**: Read/write characteristics
- **Migrations**: Flyway script names

**For Observability**:
- **Endpoint**: Health check path OR None
- **Framework**: Spring Boot Actuator, Prometheus, etc.
- **Log Levels**: INFO, DEBUG, WARN, ERROR usage
- **Metrics**: What's being tracked
- **Format**: JSON, plain text, etc.

## Update Phase

### Step 1: Identify Capability and Integration Pattern

- Scan master-spec.md for the relevant capability section (e.g., "## Restaurant Discovery")
- Identify which integration pattern section to update (APIs, Events, etc.)
- Find highest existing subsection number (e.g., 1.5)
- Use next number for NEW entries (e.g., 1.6)

### Step 2: Insert Documentation

- Add generated documentation to appropriate capability/integration pattern section
- Use templates from `documentation/master-spec-template.md`
- Maintain existing formatting and structure
- Preserve all existing content
- If integration pattern section shows "_Not yet implemented for this capability_", replace with actual implementation

### Step 3: Update Cross-References

**Capability Catalogue Table** (if new capability):
```markdown
| PRD-XXX-XXX | [Capability Name] | [Summary] | ✅ Implemented / 🔵 Planned |
```

**Table of Contents** (if new capability):
```markdown
X. [[Capability Name]](#capability-name)
```

**Last Updated Date** (header):
```markdown
Last Updated: YYYY-MM-DD
```

## Validation

Before saving, verify:
- [ ] User Story starts with "As a..."
- [ ] Business Requirements are 5-12 bullets
- [ ] Each requirement has reasoning in parentheses
- [ ] Implementation Details follow template from documentation/master-spec-template.md
- [ ] JIRA ticket ID is correctly formatted (e.g., SCRUM-8, KAN-94)
- [ ] Capability Catalogue updated (if new capability)
- [ ] Table of Contents updated (if new capability)
- [ ] Last Updated date in header is current
- [ ] No broken markdown formatting

## Key Rules

- Use absolute paths for all file operations
- Extract JIRA ticket ID from branch name automatically
- Follow existing master-spec.md formatting exactly
- **Always use templates from** `documentation/master-spec-template.md`
- Preserve all existing content (only ADD, don't modify existing entries unless user specifies UPDATE)
- If spec.md doesn't exist or is incomplete, ERROR and ask for clarification
- Document ONLY what actually EXISTS in the implementation (no assumptions or planned features)
- Business Requirements focus on WHAT and WHY (no technical HOW)
- Implementation Details focus on HOW (technical specifics)
- Replace "_Not yet implemented for this capability_" with actual implementation when adding first feature to an integration pattern

## Error Handling

**If FEATURE_SPEC not found**:
```text
❌ ERROR: No spec.md found for current branch.
Please run `/sp.specify` first to create a specification.
```

**If master-spec.md not found**:
```text
❌ ERROR: master-spec.md not found at .specify/memory/master-spec.md
This is a critical file that should always exist.
```

**If template file not found**:
```text
❌ ERROR: Template file not found at documentation/master-spec-template.md
This file contains required integration pattern templates.
```

**If unable to determine JIRA ticket**:
```text
⚠️ WARNING: Cannot extract JIRA ticket ID from branch name "[BRANCH]"
Please provide JIRA ticket ID manually: 
```

## Success Output

```text
✅ Master Spec Synced Successfully!

📋 Summary:
- Branch: [BRANCH]
- JIRA: [TICKET-ID]
- Spec: [FEATURE_SPEC path]

📝 Capability Updated:
✅ [Capability Name]
   Integration Pattern: APIs
   - Added: [METHOD] /api/vX/path (Section X.Y)

   Integration Pattern: Database Access
   - Updated: [Detail] (Section X.Y)

📊 Also Updated:
- Capability Catalogue table (if new capability)
- Table of Contents (if new capability)
- Last Updated date (header)

🔗 Files Modified:
- .specify/memory/master-spec.md

⏭️  Next Steps:
1. Review the generated documentation
2. Run `/sp.checklist` to validate completeness
3. Commit changes with: git add .specify/memory/master-spec.md
4. After the feature **PR has merged**, run **`sp.feature-build.md` Phase 10** (or Jira–Git automation) to set the Jira **Story** to **Done** — not when the PR is only opened
```

## Related Resources

- `documentation/master-spec-template.md` - Integration pattern templates (REQUIRED)
- `.cursor/prompts/PROMPT_LIBRARY.md` - Prompt #3 (manual alternative)
- `.specify/memory/constitution.md` - Constitution v2.2.0 (TDD, Active Rules, Audit Logging, JWT Security)
- `.specify/memory/master-spec.md` - Target file

---

## SDD execution timing (Option A — `SDD-TIMELINE.md`)

1. Resolve **`FEATURE_DIR`** for the feature whose **`spec.md`** / implementation you are rolling into **`master-spec.md`** (e.g. `specs/$(git rev-parse --abbrev-ref HEAD)` when that path exists, or **`check-prerequisites.sh --json`** / **`SPECIFY_FEATURE`**). Ensure **`FEATURE_DIR/SDD-TIMELINE.md`** exists (create from **`.specify/templates/sdd-timeline-template.md`** if missing). If legacy two-column only, add **Started** and use `—` on existing rows.
2. **Start timestamp:** Before substantive **`/sp.sync-master-spec`** work, `date -u +"%Y-%m-%dT%H:%M:%SZ"` → `start_ts`.
3. Perform the main **`/sp.sync-master-spec`** work.
4. **Completed timestamp:** When that work finishes, `date -u +"%Y-%m-%dT%H:%M:%SZ"` → `completed_ts`.
5. **Append** `| sync_master_spec | <start_ts> | <completed_ts> |` (re-runs append another row).
6. **Final reply:** paste the **full table** under **### SDD execution time**.


