---
name: jira-ticket-creator
description: Create one-off Jira Tasks and/or Stories (with Sub-tasks) for a project already in progress. Use this skill when the user wants to add a small number of tickets (1–10) to an existing Jira project and Epic — NOT for setting up a new initiative from scratch. Accepts plain-language descriptions and interprets them into structured Jira fields: summary, assignee, labels, epic link, start date, end date, and original estimate.
---

# Jira Ticket Creator

Creates Tasks and/or Stories (with optional Sub-tasks) inside an existing Jira project and Epic, from plain-language descriptions.

**CloudId:** `renoworks.atlassian.net`

## Workflow

### Step 1 — Gather inputs

Ask for all of the following before proceeding:

1. **Jira project key** (e.g. `S4`, `RW2`, `MAINT`) — if the user provides an Epic key (e.g. `S4-1234`), infer the project key from it automatically and do not ask separately
2. **Epic** — name or key of the existing Epic to link all tickets to
3. **Ticket descriptions** — plain language descriptions of each ticket, including any known details (assignee, dates, sub-tasks, estimates, labels)

All three are required before proceeding.

### Step 2 — Find the Epic

Search for the Epic using `mcp__claude_ai_Atlassian__searchJiraIssuesUsingJql`:

```jql
project = {KEY} AND issuetype = Epic AND summary ~ "{epic name}" ORDER BY created DESC
```

- If an exact or close match is found, confirm with the user: "Found Epic {KEY} — {summary}. Is this the correct one?"
- If multiple matches are found, list them and ask the user to choose
- If not found, ask the user to provide the Epic key directly

### Step 3 — Fetch project metadata and resolve assignees

Run all of the following in parallel:

**Metadata:** Call `mcp__claude_ai_Atlassian__getJiraProjectIssueTypesMetadata` with the project key to confirm available issue types (Task, Story, Sub-task, etc.). Then call `mcp__claude_ai_Atlassian__getJiraIssueTypeMetaWithFields` for each issue type you'll use (these can also run in parallel) to determine:
- Required fields
- How to link to the Epic: look for a `parent` field (team-managed) or `customfield_10014` (company-managed)
- Whether `startDate`, `duedate`, and `timeoriginalestimate` are available

**Assignees:** For any named assignees in the ticket descriptions, call `mcp__claude_ai_Atlassian__lookupJiraAccountId` for each name now (in parallel). Track resolved account IDs and flag any names that could not be matched — the confirmation summary in Step 5 will show resolved vs. unresolved status so the user can correct names before committing.

### Step 4 — Interpret tickets

From the plain-language descriptions, extract the following for each ticket:

| Field | Notes |
|---|---|
| `issuetype` | Task or Story |
| `summary` | Formatted per naming convention below |
| `assignee` | Match to Jira account if mentioned; leave blank if not |
| `labels` | Any labels mentioned, plus `WBS` always |
| `parent` | The Epic found in Step 2 (for Tasks/Stories); the parent Task/Story (for Sub-tasks) |
| `startDate` | In `YYYY-MM-DD` format; infer from context if mentioned |
| `duedate` | In `YYYY-MM-DD` format; infer from context if mentioned |
| `timeoriginalestimate` | See estimation rules below |
| `description` | Generated per description template below |
| `sub-tasks` | List of sub-task summaries if mentioned |

#### Description Template

Every Task, Story, and Sub-task must include a `description` field. Generate the content as follows:

Use Jira wiki markup for all descriptions so they render correctly in every Jira view.

**For Tasks and Stories:**

```
h2. User Story
As a [user type], I want to [action], so that [benefit].

h2. Acceptance Criteria
* [Testable condition 1]
* [Testable condition 2]
* [Testable condition 3]
```

**For Sub-tasks:**

```
[1–3 sentences describing the specific work to be done and relevant context.]

h2. Acceptance Criteria
* [Testable condition 1]
* [Testable condition 2]
```

Rules:
- User stories should be written from the perspective of the end user or stakeholder
- Acceptance criteria must be specific and testable — avoid vague language like "works correctly"
- Sub-task descriptions should reference the parent story context where helpful
- If no estimate is given, note it as AI-inferred in the description below the acceptance criteria: `_⚠ Estimate is AI-inferred._`

#### Summary Naming Convention

All Task and Story summaries must follow:

```
[TEAM] - [RW PRODUCT] - [TASK SUMMARY]
```

Examples:
- `Backend - RW Pro - Permission Override Endpoints`
- `Frontend - Surfaces - Role-based Toggle Controls`
- `QA - RW Pro - End-to-End Roles Testing`

- **TEAM**: Derive from context (Backend, Frontend, QA). Ask if unclear.
- **RW PRODUCT**: Derive from Epic or project context (e.g. `RW Pro`, `Surfaces`). Confirm if ambiguous.
- **TASK SUMMARY**: 4–7 word noun phrase. Lead with the subject/outcome, not a verb.

Sub-task summaries do not need to follow this convention — use a concise descriptive phrase.

#### Estimation Rules

- **1 day = 6 hours**
- **Estimates in 2-hour increments only**: 2h, 4h, 6h, 8h, etc. Always round up (e.g. 3h → 4h, 7h → 8h)
- T-shirt sizes: XS=2h · S=4h · M=8h · L=18h · XL=30h

- If no estimate is given, infer based on task type and complexity, then flag it as AI-inferred in the description
- Store as seconds in `timeoriginalestimate` (e.g. 8h → `28800`)

### Step 5 — Present summary and confirm

Show a structured summary before creating anything. Include assignee resolution status from Step 3 so the user can correct any unmatched names before committing:

```
Project: {KEY} | Epic: {EPIC-KEY} — {Epic Summary}
Tickets to create ({N}):
  [{Type}] {Summary} | Assignee: {Name ✓ or ⚠ not found} | Labels: {labels} | Est: {Xh} | {start} → {due}
    Sub-tasks: {list if any}
```

Ask: "Shall I create these? Any changes?"

Wait for explicit confirmation before continuing.

### Step 6 — Create Tasks and Stories

Create each Task or Story using `mcp__claude_ai_Atlassian__createJiraIssue` with:
- `issuetype`: Task or Story
- `summary`: formatted per naming convention
- `description`: generated per description template (User Story + Acceptance Criteria)
- `assignee`: account ID resolved in Step 3 (if resolved)
- `labels`: always include `WBS`, plus any others
- Epic link: use `parent` (team-managed) or `customfield_10014` (company-managed) per Step 3 metadata
- `startDate`: in `YYYY-MM-DD` format (if provided)
- `duedate`: in `YYYY-MM-DD` format (if provided)
- `timeoriginalestimate`: in seconds (if estimated)

Create all top-level tickets in parallel for efficiency.

### Step 7 — Create Sub-tasks

For each Sub-task, create using `mcp__claude_ai_Atlassian__createJiraIssue` with:
- `issuetype`: Sub-task
- `summary`: concise descriptive phrase
- `description`: generated per description template (Description + Acceptance Criteria)
- `parent`: the key of the Task or Story created in Step 6

Create all Sub-tasks in parallel for efficiency.

### Step 8 — Print summary

Output a table of all created issues:

| Type | Key | Summary | Est | Start | Due | Assignee |
|------|-----|---------|-----|-------|-----|----------|
| Story | S4-101 | Frontend - RW Pro - Role-based Homepage | 8h | 2025-06-02 | 2025-06-03 | Jane Smith |
| Sub-task | S4-102 | Set up route | — | — | — | — |
| Task | S4-103 | Backend - RW Pro - Permission Validation | 12h | 2025-06-02 | 2025-06-04 | John Doe |

Include direct Jira links: `https://renoworks.atlassian.net/browse/{KEY}`

## Notes

- Always confirm with the user before creating any issues
- If an issue type (e.g. Sub-task, Story) is not available in the project, inform the user and ask how to proceed
- If start/end dates are not provided, leave them blank — do not invent dates
- If a field is not available for the project type (per Step 3 metadata), notify the user rather than silently skipping it
