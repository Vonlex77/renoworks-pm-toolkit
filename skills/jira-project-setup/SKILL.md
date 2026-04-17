---
name: jira-project-setup
description: Set up a structured Jira initiative inside an existing project (RW2, S4, MAINT, etc.) by creating Epics, Tasks, and Release issues. Use this skill when the user wants to create a new Jira project, initiative, or feature set — including when they provide a Confluence page with project details, or when they want to create Epics/Tasks/Releases in Jira. Automatically generates User Story, Expected Behaviour, and QA Acceptance Criteria for each Task. Checks for existing Epics to avoid duplicates.
---

# Jira Project Setup

Sets up a structured Jira initiative by reading a Confluence brief, then interactively creating Epics, Tasks (with AI-generated User Story, Expected Behaviour, and QA Acceptance Criteria), and Release issues inside an existing Jira project.

**CloudId:** `renoworks.atlassian.net`

## Workflow

### Step 1 — Get the Confluence page and team composition

Ask the user for the following before doing anything else:

1. "Please provide the **Project Charter** Confluence page URL or page ID. This is the single-page alignment document that outlines what we're building, why, success criteria, risks, and team responsibilities — it will be used to extract project structure and also linked in the final Teams summary."
2. "How many developers will be working on this project, broken down by team? (e.g. 1 Backend, 1 Frontend, 1 QA)"

Both are required before proceeding. Store the developer counts per team — they are used in Step 10 to schedule tasks correctly based on available daily capacity.

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

### Step 4 — Present summary and confirm

Show a structured summary before doing anything:

```
Project: {KEY}

Epics ({N}):
  • {Epic 1 name}
  • {Epic 2 name}

Tasks ({N}):
  Under {Epic 1}: {Task A} (~Xh), {Task B} (~Xh)
  Under {Epic 2}: {Task C} (~Xh)

Releases ({N}):
  • {Release 1}

Total estimated effort: ~Xh
```

Include a preliminary effort estimate (see Step 6b) for each task in this summary. If no estimates are available yet, note that they will be determined in the next step.

Ask: "Shall I proceed with creating these? Any changes?"

Wait for explicit confirmation before continuing.

### Step 5 — Check for existing Epics

For each Epic, search before creating:

```jql
project = {KEY} AND issuetype = Epic AND summary ~ "{epic name}" ORDER BY created DESC
```

- If an exact or close match is found: reuse it, inform the user ("Found existing Epic: {KEY} — {summary}")
- If not found: create using `mcp__claude_ai_Atlassian__createJiraIssue` with `labels: ["WBS"]`

### Step 6 — Generate Task content with AI

For each Task, before creating it, generate the following fields using all available context (Confluence page content, Epic description, task summary):

**User Story**
```
As a [specific user type], I want to [action], so that [benefit/outcome].
```

**Expected Behaviour**
A bullet list of observable outcomes when the feature/task works correctly. Be specific and technical where context allows.

**QA Acceptance Criteria**
A checklist of testable, verifiable conditions that must all be true for the task to be accepted as done.

Format the Task description using Jira wiki markup:

```
h2. User Story
As a [user type], I want to [action], so that [benefit].

h2. Expected Behaviour
* [behaviour 1]
* [behaviour 2]
* [behaviour 3]

h2. QA Acceptance Criteria
* [criterion 1]
* [criterion 2]
* [criterion 3]
```

Present the generated content for each Task and ask: "Does this look correct? Any edits before I create it?" Proceed task-by-task or ask if the user wants to review all at once.

### Step 6b — Estimate effort per Task

**Team conventions — apply these to every estimate:**
- **1 day = 6 hours** of effort. Always convert day-based estimates using this rate (e.g. "2 days" = 12h, "half a day" = 3h → round up to 4h).
- **Estimates must be in 2-hour increments**: 2h, 4h, 6h, 8h, 10h, 12h, etc. Always round up to the next even increment (e.g. 1h → 2h, 3h → 4h, 13h → 14h, 17h → 18h). Never produce an odd-hour estimate.

For each Task, determine a time estimate using this priority order:

**1. Explicit page value** — if the Confluence page stated a specific estimate for this task (e.g. "3 days", "8h", "5 points"), use it directly. Convert days to hours at 6h/day, then round up to the nearest 2-hour increment.

**2. Relative page signal** — if the page used a t-shirt size or complexity label, map it:

| Signal | Hours |
|--------|-------|
| XS / trivial | 2h |
| S / small | 4h |
| M / medium | 8h |
| L / large | 18h |
| XL / extra large | 30h |
| Complex / multi-sprint | 42h+ (flag for user review) |

**3. AI inference** — if no effort signal exists on the page, infer based on the task's User Story, Expected Behaviour, and team context:

- **Backend tasks**: API endpoints, DB changes, and business logic typically range 4–16h depending on complexity and number of endpoints.
- **Frontend tasks**: UI components and integrations typically range 4–12h; full flows or new pages range 12–24h.
- **QA tasks**: End-to-end testing suites typically range 8–16h; smoke/regression for a single feature 4–8h.
- Consider dependencies: if a task blocks or depends on multiple others, add buffer.
- When uncertain between two bands, choose the higher one and note the assumption.
- After arriving at a raw estimate, apply the 2-hour increment rounding rule before finalizing.

**Output format** — record each estimate as:
- Hours (e.g. `8h`) for the Jira `timeoriginalestimate` field
- A one-line rationale (e.g. "Inferred: 2 new API endpoints + auth middleware, medium complexity → 14h")

Add a `h2. Effort Estimate` section to the Task description:

```
h2. Effort Estimate
*Estimated:* 8h
*Basis:* [Page-specified | Page signal: M | Inferred: reason]
```

If the estimate is AI-inferred (not from the page), flag it clearly so the user can adjust.

### Step 7 — Create Tasks

Create each Task using `mcp__claude_ai_Atlassian__createJiraIssue` with:
- `issuetype`: Task (or Story if that's the appropriate type)
- `summary`: formatted as `[TEAM] - [RW PRODUCT] - [TASK SUMMARY]` (see Step 2 naming convention)
- `description`: the generated wiki-markup content from Step 6 (including the `h2. Effort Estimate` section from Step 6b)
- `timeoriginalestimate`: the estimate in seconds (e.g. 8h → `28800`). Always set this field — it maps to the "Original Estimate" field in Jira. If Step 3 metadata indicates it is unavailable, notify the user rather than silently skipping it.
- `labels`: `["WBS"]` — always include this label on every Task so it appears on the master workback schedule
- Epic link: use `parent` field (team-managed) or `customfield_10014` (company-managed) based on Step 3 metadata

### Step 8 — Create Release issues and link Tasks

Create each Release issue using `mcp__claude_ai_Atlassian__createJiraIssue` with:
- `issuetype`: Deployment Release (or the matching type from Step 3 metadata)
- `summary`: formatted as `{Initiative Name} - Dev Release` (e.g. `RW Pro - Instant Estimate - Homeowner/Contractor Toggle Feature - Dev Release`)
- `description`: any description from the Confluence page
- `labels`: `["WBS"]` — always include this label on every Release so it appears on the master workback schedule
- Link to parent Epic if the field is available

After creating the Dev Release, link every Task created in Step 7 to it using `mcp__claude_ai_Atlassian__createIssueLink`:
- `inwardIssue`: the Task key (the blocker)
- `outwardIssue`: the Dev Release key (the blocked issue)
- `linkType`: `Blocker`

This results in each Task showing **"blocks" → Dev Release**, and the Dev Release showing **"is blocked by" ← Task**.

Create all Task→Release links in parallel for efficiency.

### Step 9 — Identify and create dependencies

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

  S4-101 (Backend - API Endpoints) → blocks → S4-103 (Frontend - UI Integration)
    Reason: Frontend consumes the API built in S4-101

  S4-101 (Backend - API Endpoints) → blocks → S4-104 (QA - End-to-End Testing)
  S4-103 (Frontend - UI Integration) → blocks → S4-104 (QA - End-to-End Testing)
    Reason: QA cannot test until both Backend and Frontend are complete

  S4-102 (Backend - DB Migration) → blocks → S4-101 (Backend - API Endpoints)
    Reason: API depends on the schema introduced in S4-102
```

Ask: "Does this dependency map look correct? Any changes before I create the links?"

Wait for explicit confirmation.

**Create the links** using `mcp__claude_ai_Atlassian__createIssueLink`:
- `inwardIssue`: the blocking task key
- `outwardIssue`: the blocked task key
- `linkType`: `Blocker`

Create all dependency links in parallel for efficiency. Do not re-create any links already established in Step 8 (Task → Release blocker links).

### Step 10 — Build workback schedule and set dates

#### 10a — Get scheduling inputs

Ask the user:
> "To build the workback schedule, I need two things:
> 1. What is the **project start date** (the first day work can begin)?
> 2. Is there a **target delivery date** (hard deadline)? If so, I'll flag any schedule risk."

Wait for the response before continuing.

#### 10b — Scheduling rules

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

#### 10c — Present the schedule for confirmation

Output a schedule table before setting any dates:

```
Workback Schedule (start: YYYY-MM-DD)

| Key | Summary | Est | Start | Due | Depends On |
|-----|---------|-----|-------|-----|------------|
| S4-102 | DB Migration | 6h | 2025-06-02 | 2025-06-02 | — |
| S4-101 | API Endpoints | 12h | 2025-06-03 | 2025-06-04 | S4-102 |
| S4-103 | Frontend UI Integration | 8h | 2025-06-03 | 2025-06-04 | S4-101 |
| S4-104 | QA End-to-End Testing | 14h | 2025-06-05 | 2025-06-07 | S4-101, S4-103 |
| S4-105 | Dev Release | — | 2025-06-10 | 2025-06-10 | all tasks |

Total project duration: X working days (YYYY-MM-DD → YYYY-MM-DD)
```

If a target delivery date was provided and the calculated end date exceeds it, flag the risk clearly:
> "⚠ Schedule risk: projected completion is YYYY-MM-DD, which is X days past the target of YYYY-MM-DD."

Ask: "Does this schedule look correct? Any adjustments before I apply the dates to Jira?"

Wait for explicit confirmation.

#### 10d — Apply dates to Jira

For each Task and Release issue, call `mcp__claude_ai_Atlassian__editJiraIssue` to set:
- `startDate` — the calculated start date in `YYYY-MM-DD` format
- `duedate` — the calculated due date in `YYYY-MM-DD` format

If either field is not available for the project's issue type (per Step 3 metadata), notify the user which field is missing rather than silently skipping it.

> **Known limitation — company-managed projects:** The built-in `startDate` / `customfield_10015` field is system-managed ("LOCKED") in company-managed Jira projects and cannot be set via the REST API. If it fails, notify the user and ask them to make the field editable via Jira admin settings before retrying. The `duedate` field is standard and always works.

Apply all date updates in parallel for efficiency.

### Step 11 — Print summary

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

**Example output (for reference only — do not copy verbatim):**

```
RW Pro - Instant Estimate - New Toggle Feature - Project Charter - https://renoworks.atlassian.net/wiki/x/HoAB3Q
The following Project Charter is intended to be a single-page alignment that outlines the project — what we're building and why, clear success criteria so we know what "done" looks like, known risks and mitigations, a phased delivery timeline (backend → frontend → QA), and a RACI matrix so there's no ambiguity on decision-making or responsibilities. The goal is for anyone on the team to get up to speed without needing a separate kickoff conversation.

RW Pro - Instant Estimate - New Toggle Feature - Jira Project - https://renoworks.atlassian.net/browse/S4-23077
Based on the scope defined within the Project Charter, all required tasks for development have been defined in the following Jira Epic — 2 backend tasks for updating the RW Pro API endpoint, with 5 frontend tasks for Surfaces and AppBuilder to surface the Instant Estimate feature to Homeowners via a toggle control within the CMS. As of today, the total estimated development effort is currently ~56 hours to reach dev completion.
```

## Notes

- Always confirm with the user before creating any issues
- If a Release issue type does not exist in the project, inform the user and ask how to proceed
- If the Confluence page is ambiguous, ask clarifying questions rather than guessing
- Prefer creating fewer, well-defined issues over many vague ones
