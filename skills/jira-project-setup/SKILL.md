---
name: jira-project-setup
description: Set up a structured Jira initiative inside an existing project (RW2, S4, MAINT, etc.) by creating Epics, Tasks, and Release issues. Use this skill when the user wants to create a new Jira project, initiative, or feature set — including when they provide a Confluence page with project details, or when they want to create Epics/Tasks/Releases in Jira. Automatically generates User Story, Expected Behaviour, and QA Acceptance Criteria for each Task. Checks for existing Epics to avoid duplicates.
---

# Jira Project Setup

Sets up a structured Jira initiative by reading a Confluence brief, then interactively creating Epics, Tasks (with AI-generated User Story, Expected Behaviour, and QA Acceptance Criteria), and Release issues inside an existing Jira project.

**CloudId:** `renoworks.atlassian.net`

## Workflow

### Step 1 — Get the Confluence page, team composition, and scheduling inputs

Ask the user for the following before doing anything else:

1. "Please provide the **Project Charter** Confluence page URL or page ID. This is the single-page alignment document that outlines what we're building, why, success criteria, risks, and team responsibilities — it will be used to extract project structure and also linked in the final Teams summary."
2. "How many developers will be working on this project, broken down by team? (e.g. 1 Backend, 1 Frontend, 1 QA)"
3. "What is the **project start date** (the first day work can begin)?"
4. "Is there a **target delivery date** (hard deadline)? If so, I'll flag any schedule risk."

All four are required before proceeding. Store the developer counts per team and the scheduling dates — they are used throughout the workflow to estimate task dates and flag delivery risk.

Read the page using `mcp__claude_ai_Atlassian__getConfluencePage`.

### Step 2 — Extract project structure

From the page content, extract using your best judgment:
- **Target Jira project key** (e.g. `S4`, `RW2`, `MAINT`) — ask user to confirm if unclear
- **RW product name** (e.g. `RW Pro`, `Surfaces`, `RW 2.0`) — used in Task summary formatting
- **Epics**: name, description/goal
- **Tasks per Epic**: summary, description, assignee (if mentioned), and **team** (e.g. `Backend`, `Frontend`, `QA`)
- **Release issues**: summary, description, which Epic they belong to (if mentioned)
- **Effort signals**: any time estimates, story points, t-shirt sizes (S/M/L/XL), complexity notes, or sprint capacity hints mentioned anywhere on the page — at the initiative level, per-epic, or per-task

If the page does not mention specific tasks or releases, ask the user what they'd like created before proceeding.

#### Task Summary Naming Convention

All Task summaries must follow this structure:

```
[TEAM] - [RW PRODUCT] - [TASK SUMMARY]
```

Examples:
- `Backend - RW Pro - Implement Roles and Permissions List Endpoint`
- `Frontend - RW Pro - AppBuilder: Add per-role homeowner/contractor Instant Estimate toggles`
- `QA - RW Pro - End-to-End Testing: AppBuilder, Surfaces, and Java API`

- **TEAM**: Derive from the deliverables/team table on the Confluence page (e.g. Backend, Frontend, QA). If a task spans teams or is unclear, ask the user.
- **RW PRODUCT**: Extract from the page title or project description (e.g. `RW Pro`, `Surfaces`). Confirm with user if ambiguous.
- **TASK SUMMARY**: A concise noun phrase — 4 to 7 words maximum. Avoid filler words like "Implement", "Add", "Update", "Handle". Lead with the subject or outcome instead. Bad: `Implement Permission Override Endpoints for CAN_ORDER_EV_ESTIMATE`. Good: `CAN_ORDER_EV_ESTIMATE Permission Override Endpoints`.

This convention does **not** apply to Epics or Deployment Release summaries.

### Step 3 — Fetch project metadata

Call `mcp__claude_ai_Atlassian__getJiraProjectIssueTypesMetadata` with the project key to confirm available issue types (Epic, Task, Story, Release, etc.).

For issue types you'll use (Epic, Task, Release), call `mcp__claude_ai_Atlassian__getJiraIssueTypeMetaWithFields` to determine:
- Required fields
- How to link Tasks to Epics: look for a `parent` field (team-managed projects) or `Epic Link` / `customfield_10014` (company-managed projects)

### Step 4 — Generate Task content and confirm

Generate all task content before doing anything in Jira. Use all available context: Confluence page content, Epic description, task summary.

**For each Task, generate:**
- **User Story**: `As a [user type], I want to [action], so that [benefit].`
- **Expected Behaviour**: bullet list of observable outcomes when the feature works correctly — specific and technical
- **QA Acceptance Criteria**: checklist of testable, verifiable conditions that must all be true for the task to be accepted
- **Effort estimate**: see rules below

**Estimation rules — apply to every task:**
- 1 day = 6h. Increments of 2h only, always round up (e.g. 3h → 4h, 13h → 14h).
- Priority order: (1) explicit page value — convert at 6h/day; (2) t-shirt size: XS=2h · S=4h · M=8h · L=18h · XL=30h · Complex=42h+; (3) AI inference — Backend 4–16h, Frontend 4–24h, QA 4–16h; choose higher band when uncertain
- Flag AI-inferred estimates

**Task description format (Jira wiki markup):**
```
h2. User Story
As a [user type], I want to [action], so that [benefit].

h2. Expected Behaviour
* [behaviour]

h2. QA Acceptance Criteria
* [criterion]

h2. Effort Estimate
*Estimated:* Xh
*Basis:* [Page-specified | Page signal: M | Inferred: reason]
```

**Then present the full plan for confirmation:**
```
Project: {KEY}
Epics ({N}): {Epic 1}, {Epic 2}
Tasks ({N}):
  Under {Epic 1}: [{Team}] {Summary} (~Xh) — [one-line User Story]
Releases ({N}): {Release 1}
Total estimated effort: ~Xh | Schedule: {start} → ~{projected end}
```

Ask: "Shall I proceed? Any changes to epics, tasks, or estimates?"

Wait for explicit confirmation before continuing.

### Step 5 — Check for existing Epics

Run all Epic duplicate checks in parallel — each JQL lookup is independent:

```jql
project = {KEY} AND issuetype = Epic AND summary ~ "{epic name}" ORDER BY created DESC
```

- If an exact or close match is found: reuse it, inform the user ("Found existing Epic: {KEY} — {summary}")
- If not found: create using `mcp__claude_ai_Atlassian__createJiraIssue` with `labels: ["WBS"]`

Epic creations can also run in parallel once all lookups complete.

### Step 6 — Create Tasks

Create all Tasks in parallel for efficiency using `mcp__claude_ai_Atlassian__createJiraIssue` with:
- `issuetype`: Task (or Story if that's the appropriate type)
- `summary`: formatted as `[TEAM] - [RW PRODUCT] - [TASK SUMMARY]` (see Step 2 naming convention)
- `description`: the generated wiki-markup content from Step 4 (User Story, Expected Behaviour, QA Acceptance Criteria, and Effort Estimate sections)
- `timeoriginalestimate`: the estimate in seconds (e.g. 8h → `28800`). Always set this field — it maps to the "Original Estimate" field in Jira. If Step 3 metadata indicates it is unavailable, notify the user rather than silently skipping it.
- `labels`: `["WBS"]` — always include this label on every Task so it appears on the master workback schedule
- Epic link: use `parent` field (team-managed) or `customfield_10014` (company-managed) based on Step 3 metadata

### Step 7 — Create Release issues and link Tasks

Create each Release issue using `mcp__claude_ai_Atlassian__createJiraIssue` with:
- `issuetype`: Deployment Release (or the matching type from Step 3 metadata)
- `summary`: formatted as `{Initiative Name} - Dev Release` (e.g. `RW Pro - Instant Estimate - Homeowner/Contractor Toggle Feature - Dev Release`)
- `description`: any description from the Confluence page
- `labels`: `["WBS"]` — always include this label on every Release so it appears on the master workback schedule
- Link to parent Epic if the field is available

After creating the Dev Release, link every Task created in Step 6 to it using `mcp__claude_ai_Atlassian__createIssueLink`:
- `inwardIssue`: the Task key (the blocker)
- `outwardIssue`: the Dev Release key (the blocked issue)
- `linkType`: `Blocker`

This results in each Task showing **"blocks" → Dev Release**, and the Dev Release showing **"is blocked by" ← Task**.

Create all Task→Release links in parallel for efficiency.

### Step 8 — Identify and create dependencies

After all Tasks and Releases are created, review every ticket holistically and infer dependencies using the following signals:

**Dependency signals to look for:**
- **Team sequencing**: Backend API tasks must typically be completed before Frontend tasks that consume those APIs. QA tasks can only begin after the relevant Backend and Frontend tasks are done.
- **Data / schema order**: DB migrations or schema changes must precede any task that reads from or writes to those tables.
- **Auth / permissions gates**: Any task that requires a new role, permission, or token must wait on the task that creates it.
- **Shared components**: If two tasks both mention a shared UI component, service, or library — the task building it blocks the task consuming it.
- **Explicit page language**: Words like "after", "once", "depends on", "requires", "prerequisite", or "then" in the Confluence page are strong signals.
- **Logical ordering**: Setup/scaffolding tasks block feature tasks; feature tasks block integration/testing tasks.

**Output a dependency map before creating any links:**

```
Dependency Map:
  {KEY} ({Summary}) → blocks → {KEY} ({Summary})
    Reason: [one-line explanation]
```

Ask: "Does this dependency map look correct? Any changes before I create the links?"

Wait for explicit confirmation.

**Create the links** using `mcp__claude_ai_Atlassian__createIssueLink`:
- `inwardIssue`: the blocking task key
- `outwardIssue`: the blocked task key
- `linkType`: `Blocker`

Create all dependency links in parallel for efficiency. Do not re-create any links already established in Step 7 (Task → Release blocker links).

### Step 9 — Build workback schedule and set dates

#### 9a — Scheduling rules

Use the project start date and target delivery date collected in Step 1.

Apply these rules consistently when calculating dates:

- **Working days only**: Monday–Friday. Skip weekends when advancing dates.
- **Daily capacity**: 6 hours per working day per developer (1 day = 6h).
- **Resource-constrained scheduling**: Tasks assigned to the same team are scheduled sequentially against that team's developer(s). If a team has 1 developer, they can only work on 1 task at a time — tasks queue up and the next task starts the day after the previous one ends.
- **Daily capacity packing**: If a task's estimate is less than 6h, and the same developer has remaining capacity that day, a second task can be scheduled on the same day — as long as their combined hours ≤ 6h. Both tasks share the same start and due date in this case.
- **Cross-team parallelism**: Tasks assigned to different teams (e.g. Backend vs Frontend) can overlap if their dependencies allow it.
- **Dependency gates**: A task cannot start until all tasks that block it are complete, regardless of developer availability.
- **Duration**: `ceil(estimate_hours / 6)` working days. A 4h task = 1 day; a 8h task = 2 days; a 14h task = 3 days (always round up).
- **Due date** = start date + duration in working days − 1 (i.e. the last working day the task is active).
- The **Release issue** start and due date = the day all its blocking Tasks are complete.

#### 9b — Present the schedule for confirmation

Output a schedule table before setting any dates. Columns: Key | Summary | Est | Start | Due | Depends On. End with total project duration and date range.

If a target delivery date was provided and the calculated end date exceeds it, flag the risk clearly:
> "⚠ Schedule risk: projected completion is YYYY-MM-DD, which is X days past the target of YYYY-MM-DD."

Ask: "Does this schedule look correct? Any adjustments before I apply the dates to Jira?"

Wait for explicit confirmation.

#### 9c — Apply dates to Jira

For each Task and Release issue, call `mcp__claude_ai_Atlassian__editJiraIssue` to set:
- `startDate` — the calculated start date in `YYYY-MM-DD` format
- `duedate` — the calculated due date in `YYYY-MM-DD` format

If either field is unavailable (per Step 3 metadata), notify the user rather than silently skipping it.

> **Known limitation — company-managed projects:** The built-in `startDate` / `customfield_10015` field is system-managed ("LOCKED") in company-managed Jira projects and cannot be set via the REST API. If it fails, notify the user and ask them to make the field editable via Jira admin settings before retrying. The `duedate` field is standard and always works.

Apply all date updates in parallel for efficiency.

### Step 10 — Print summary

Output a table of all created and reused issues:

| Type | Key | Summary | Estimate | Start | Due | Blocks |
|------|-----|---------|----------|-------|-----|--------|
| Epic (existing) | S4-123 | ... | — | — | — | — |
| Epic (created) | S4-456 | ... | — | — | — | — |
| Task | S4-457 | ... | 8h | 2025-06-03 | 2025-06-04 | S4-460 |
| Release | S4-458 | ... | — | 2025-06-10 | 2025-06-10 | — |

Include direct Jira links where possible: `https://renoworks.atlassian.net/browse/{KEY}`

#### Teams Channel Summary

After the table, generate two ready-to-paste paragraphs for posting in the Teams project channel.

**Format:**

```
{Initiative Full Name} - Project Charter - {Confluence page URL}
{Project Charter paragraph}

{Initiative Full Name} - Jira Project - {Epic URL}
{Jira Epic paragraph}
```

**Project Charter paragraph** — Write 2–3 sentences describing the purpose of the Confluence page. Use this structure as a guide:
- What the charter is (a single-page alignment document)
- What it covers (what we're building and why, success criteria, risks and mitigations, phased delivery timeline, RACI matrix)
- What it's for (anyone on the team can get up to speed without needing a separate kickoff conversation)

Tailor the specifics to the actual project — mention the delivery phases and team structure as they appear on the Confluence page, rather than using generic language.

**Jira Epic paragraph** — Write 2–3 sentences summarizing the scope of work created in Jira. Use this structure as a guide:
- Reference the Project Charter as the source of scope
- Describe what was created: number of tasks per team, what each team's work covers
- State the total estimated effort to reach dev completion

Use the actual task counts, team names, and total hours from the session. Format the total as "~Xh".

Format (do not copy verbatim — tailor to actual project phases, team structure, and task counts):

`{Initiative Name} - Project Charter - {URL}` → 2–3 sentences: what the charter is, what it covers (building rationale, success criteria, risks, delivery phases, RACI), and that it lets anyone get up to speed without a kickoff.

`{Initiative Name} - Jira Project - {Epic URL}` → 2–3 sentences: reference the charter as scope source, describe tasks created per team and what each covers, state total estimated hours as `~Xh`.

## Notes

- Always confirm with the user before creating any issues
- If a Release issue type does not exist in the project, inform the user and ask how to proceed
- If the Confluence page is ambiguous, ask clarifying questions rather than guessing
- Prefer creating fewer, well-defined issues over many vague ones
