---
name: jira-project-setup
description: Set up a structured Jira initiative inside an existing project (RW2, S4, MAINT, etc.) by creating Epics, Tasks, and Release issues. Supports two work streams — Platform (net-new features/products) and Enterprise (client Surfaces implementations). Use this skill when the user wants to create a new Jira project, initiative, or feature set — including when they provide a Confluence page with project details, or when they want to create Epics/Tasks/Releases in Jira. Automatically generates User Story, Expected Behaviour, and QA Acceptance Criteria for Platform tasks. Uses the standard TM-1 Jira template for Enterprise builds.
---

# Jira Project Setup

Sets up a structured Jira initiative for either a **Platform** project (net-new feature or product build) or an **Enterprise** project (client Surfaces implementation). Branches into the appropriate workflow based on the PM's answer to the first question.

**CloudId:** `renoworks.atlassian.net`

---

## Step 0 — Identify the work stream

When this skill loads, greet the user with the following message before asking anything else:

> "Hi! I'll help you set up a new Jira project. I can handle two types of builds:
> - **Platform** — net-new features, product improvements, or client feature requests. I'll read your Project Charter from Confluence and generate structured tickets with User Stories and QA Acceptance Criteria.
> - **Enterprise** — setting up Surfaces for a new enterprise client. I'll use the standard TM-1 ticket template and walk you through which tickets apply for this build.
>
> Is this an **Enterprise** or **Platform** project?"

- If **Platform** → follow the [Platform Path](#platform-path) below.
- If **Enterprise** → follow the [Enterprise Path](#enterprise-path) below.

---

## Platform Path

Used for net-new product builds, feature development, or platform improvements. Tasks are derived from a Confluence Project Charter and include AI-generated User Story, Expected Behaviour, and QA Acceptance Criteria.

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

---

## Enterprise Path

Used for enterprise client Surfaces implementations. The ticket structure follows the official TM-1 Jira template and is consistent across every client build. Tasks are templated — no User Story or QA Acceptance Criteria generation is needed.

The standard workflow is: create all tickets from the template → transition any out-of-scope tickets to **Archived** status so they remain accessible if needed later.

### Step 1E — Gather Enterprise inputs

Ask for all of the following before doing anything else:

1. "What is the **client name**? (e.g. `ProDoor Systems`)"
2. "What is the **SOW reference number**? (e.g. `SOW-PRODOOR-002`)"
3. "Is this a **New Build** or an **Upgrade** for this client?"
4. "Please provide the **App Configuration Confluence page URL**. This is the page that describes the client's app setup, branding, product catalogue, and configuration requirements."
5. "Please provide the **Basecamp project folder URL** for this client."
6. "Is there a **Figma design link**? (Optional — include if branding mock-ups or UI designs exist)"
7. "**If CRM integration is needed**: what CRM is being integrated? (e.g. Salesforce, HubSpot — leave blank if not applicable)"
8. "Which **Jira project key** should these tickets be created under? (e.g. `S4`)"
9. "What is the **team composition**? (e.g. 1 DevOps, 1 Backend, 1 DPL artist, 1 SP masker — used to build the workback schedule)"
10. "What is the **project start date** (the first day work can begin)?"
11. "Is there a **target delivery date** (hard deadline)? If so, I'll flag any schedule risk."

Required: all except Figma, CRM name, and target delivery date. Read the App Configuration Confluence page using `mcp__claude_ai_Atlassian__getConfluencePage` to inform task descriptions.

### Step 2E — Confirm ticket set and plan

Present the full template ticket list and ask the PM to confirm scope in a single round. Tickets marked **[Archived in template]** are off by default — include them only if the PM explicitly confirms they're needed. All others are on by default.

```
Enterprise Template Tickets — [Client Name] [SOW-REF]

EPIC
  [✓] [Client Name] [SOW-REF]  (Epic)

DEV
  [✓] DevOps - Vanity URL  (Story)
  [✓] Backend - CRM Integration  (Story)  ← CRM: [name from Step 1E, or "TBD" if not provided]

DPL
  [✓] DPL - [PROJECT SET UP] Asset Review & Requirements  (DPL Task)

SAMPLE PHOTO
  [✓] SP - Mask and deploy new sample images  (DS Task)  ← How many exterior scenes?
  [✓] SP - Prepare shadow files for new sample images  (DS Task)
  [✓] SP - Implement sample photo tabs  (DS Task)
  [✓] SP - Update and deploy repo images  (DS Task)
  [✓] SP - Hotspot positioning  (DS Task)

RELEASE
  [✓] Release [Client Name] New Build OR Upgrade  (Deployment Release)

--- Archived in template (include only if in scope for this build) ---
  [ ] Initial Build - DSA Analytics Setup  (Task)
  [ ] Design Team - Create branding mock-ups  (Task)
  [ ] Backend - 3D Product Viewer Deployment  (Task)
  [ ] Frontend - Complete all S4 Configurations and Styling  (Task)
  [ ] Projects - Configure S4 App in Configuration Portal  (Task)
  [ ] Backend - Configure New Build S4 App  (Task)
```

Ask: "Please confirm which tickets are in scope and how many exterior sample scenes SP will need to mask. If any of the archived tickets should be included, let me know. I'll show the full plan with estimates before creating anything."

Once the PM responds, generate the full plan summary using the confirmed ticket set and the default estimates from Step 9E, then present it:

```
Project: {KEY}
Epic (1): [Client Name] [SOW-REF]

Tickets to create — Active ({N}):
  Story     | DevOps - Vanity URL                                    | 4h
  Story     | Backend - CRM Integration                              | 16h
  DPL Task  | DPL - [PROJECT SET UP] Asset Review & Requirements     | 4h
  DS Task   | SP - Mask and deploy new sample images ({N} scenes)    | {N×4}h
  DS Task   | SP - Prepare shadow files for new sample images        | 4h
  DS Task   | SP - Implement sample photo tabs                       | 4h
  DS Task   | SP - Update and deploy repo images                     | 4h
  DS Task   | SP - Hotspot positioning                               | 4h
  Release   | Release [Client Name] New Build                        | —

Tickets to create — Archived ({M}):
  Task      | [any archived-by-default tickets included]

Total estimated effort: ~Xh
Schedule: {start date} → ~{projected end date}
```

Ask: "Does this look right? Any changes before I create the tickets?"

Wait for explicit confirmation before continuing.

### Step 3E — Fetch project metadata

Call `mcp__claude_ai_Atlassian__getJiraProjectIssueTypesMetadata` with the project key to confirm available issue types. For Enterprise projects, confirm availability of:
- `DPL Task` — Digital Product Library work
- `DS Task` — Design Services / Sample Photo work
- `Story` — DevOps and Backend/Frontend setup
- `Task` — standalone dev or design tasks
- `Deployment Release` — release milestone

Run all of the following calls in parallel:
- `mcp__claude_ai_Atlassian__getJiraIssueTypeMetaWithFields` for each issue type in use — to determine required fields and Epic linking method
- `mcp__claude_ai_Atlassian__getTransitionsForJiraIssue` on any existing issue in the target project — to find the transition ID for the **Archived** status (needed in Step 6E)

### Step 4E — Check for existing Epic

Search for an existing Epic to avoid duplicates:

```jql
project = {KEY} AND issuetype = Epic AND summary ~ "{Client Name}" ORDER BY created DESC
```

- If a match is found: confirm with the PM whether to reuse it or create a new one.
- If not found: create using `mcp__claude_ai_Atlassian__createJiraIssue`.

**Epic description (Jira wiki markup):**
```
This is an epic for creation of site: `[app-name derived from App Config page]`

Related documentation:

* Confluence Page: [Client Name - App Configuration|{Confluence App Config URL}]
* Basecamp Folder: [{Basecamp URL}|{Basecamp URL}]

*TO-DO*:

# Fill in site name and documentation.
# Fill in sub-ticket details for all sub-tickets. (if applicable)
# Mark tickets not needed as "Not Accepted".
# Set End-Date for ticket when DOD is sent for project closure.
```

Include `labels: ["WBS"]` on the Epic.

### Step 5E — Create all tickets

Create all confirmed tickets using `mcp__claude_ai_Atlassian__createJiraIssue`. Run parallel creation calls for tickets with no dependencies between them. Use the exact descriptions below — replace placeholders in `[brackets]` with actual client values from Step 1E and the App Config Confluence page.

**Epic link:** use `parent` field (team-managed) or `customfield_10014` (company-managed) based on Step 3E metadata.

Always include `WBS` on every Enterprise ticket, plus the per-ticket label defined below.

---

#### DevOps - Vanity URL (Story) — label: `Dev-Backend`

```
Create a vanity URL (URL to be confirmed by `[client-name]`).

URL: `visualizer.[app-name].com`

Vanity URLs

{code}
visualizer.[app-name].com => [app-name].renoworks.com
visualizer-api.[app-name].com => api.renoworks.com
{code}

[Setting up Vanity URLs|https://renoworks.atlassian.net/wiki/spaces/S3/pages/1770291250/Setting+up+Vanity+URLs]

Add the API vanity domain to the "connect-src" and "img-src" CSP directives in default.js in surfaces config
```

---

#### Backend - CRM Integration (Story) — label: `Dev-Frontend`

```
`[CRM Name]` Integration is required for `[app-name]`. ← Fill these out before assigning.

Link to CRM confluence page: INSERT LINK HERE PRIOR TO ASSIGNING OUT (Should be provided by [TM-44|https://renoworks.atlassian.net/browse/TM-44])

Add to Documentation: [CRM Integration Platform|https://renoworks.atlassian.net/wiki/spaces/COR/pages/318079092/CRM+Integration+Platform]

Back-End Documentation (Reference): [CRM platform|https://renoworks.atlassian.net/wiki/spaces/BAC/pages/315064549/CRM+platform], [Backend - Steps for CRM Integration|https://renoworks.atlassian.net/wiki/spaces/BAC/pages/1720680515/Backend+-+Steps+for+CRM+Integration]

Back-End Package: [https://bitbucket.org/renoworks/framework/src/master/src/com/renoworks/framework/objects/crm/|https://bitbucket.org/renoworks/framework/src/master/src/com/renoworks/framework/objects/crm/]
```

---

#### DPL - [PROJECT SET UP] Asset Review & Requirements (DPL Task) — label: `DPL`

```
[{Basecamp URL}|{Basecamp URL}]

h3. ASSET LIST ---

_*Instructions:*_

* _*Assign to Project Coordinator once asset list is provided.*_
* _*Provide Asset List and update the date in the log once the list is provided. This asset sheet may also be provided as an external google sheet.*_
* _*Ensure the use of the following statuses:*_
  *[ PENDING ]* *[ RECEIVED ]* *[ CLARIFICATION NEEDED ]*

* *Asset* *[ PENDING ]*
** Manufacturer & product ID

* *Asset* *[ CLARIFICATION NEEDED ]*
* *Asset* *[ RECEIVED ]*

----

h3. QUESTIONS & CLARIFICATIONS

_*Instructions*_

* _*Ensure the use of the following statuses:*_
  *[ PENDING ]* *[ RECEIVED ]* *[ CLARIFICATION NEEDED ]*
* _*Use the format defined below*_

* _Question:_ *[ PENDING ]* *[ RECEIVED ]* *[ CLARIFICATION NEEDED ]*
** Answer:

----

h3. NOTES

* _Additional Notes For A Project_

----

h3. LOG

|| *Project Set Up Date* || *Date Asset List Provided* || *Date Assets Received* || *Date Assets Reviewed* || *Date Assets Approved* ||
| {today's date} | | | | |
```

---

#### SP - Mask and deploy new sample images (DS Task) — label: `SampleProjects`

```
Mask new images found in client basecamp folder. Shadow files will need to be prepared. Supergrouping as needed.

Single source of truth is app configuration page (linked in main epic)

Layer names are displayed in layer mapping table
```

---

#### SP - Prepare shadow files for new sample images (DS Task) — label: `SampleProjects`

```
Prepare shadow files for new sample images found in client basecamp folder. Ensure these are uploaded to the correct GUID and set opacity.
```

---

#### SP - Implement sample photo tabs (DS Task) — label: `SampleProjects`

*(No description — leave blank, matching template)*

---

#### SP - Update and deploy repo images (DS Task) — label: `SampleProjects`

```
Selected repo images found on app configuration page, linked in main epic

Please update xml to align with the appropriate layer names as referenced in the layer mapping table.
```

---

#### SP - Hotspot positioning (DS Task) — label: `SampleProjects`

*(No description — leave blank, matching template)*

---

#### Release [Client Name] New Build OR Upgrade (Deployment Release) — label: `WBS`

Use `Release [Client Name] New Build` or `Release [Client Name] Upgrade` based on the PM's answer from Step 1E.

```
h2. Deployment task

# Execute a *[DEV/SP/DPL]** deployment of the App to the requested Server (see status).
# Check [Deployment Specific Instructions|https://renoworks.atlassian.net/wiki/spaces/S3/pages/317390995/Deployments+Specific+Instructions] (if Applicable).
# Run the app through the automated/manual testing to confirm that the site is ready for testing.
# For releases to a Production environment only - Before executing the deployment script, please notify DS team to ensure that maskers are not actively using the app. This is not applicable if the App uses the Scene Editor.

----

h2. QA task

# Execution of [Deployment Release QA Checklist|https://renoworks.atlassian.net/wiki/spaces/QA1/pages/313851932/Deployment+Release+QA+Checklist]
# Confirmation that App satisfies all requested verifications on the requested environment

*Exceptions*

* DS / DPL tickets when there is no explicit request
* Known Issues according to App backlog / Core backlog

----

*Purpose*: Implementation of baseline scope

----

*Release setup:**
# This release has to be assigned to the team who is responsible of the SOW/CR.
# If required work from other teams, a subrelease task has to be setup properly.
# Please ensure that all team-specific blockers are linked to this subrelease, including non-team-specific tickets which block team tasks.
```

---

#### Archived-by-default tickets

For each ticket confirmed as in scope from the "Archived in template" list, create it with the description below:

**Initial Build - DSA Analytics Setup (Task) — label: `WBS`**
```
The sub tasks within this ticket need to be completed by the DSA team before the APP goes live.

*Go Live Date:* TBD

h2. Ticket Details

*Prod URL / Staging URL:*

*Confluence:*
```

**Design Team - Create branding mock-ups (Task) — label: `WBS`**

Fill in the Basecamp link and target completion date from Step 1E context. Leave feature toggles (ON/OFF) as written — the design team fills these in:
```
*Branding Assets & Background image:* [{Basecamp URL}|{Basecamp URL}]

*Particularities:*

* Version 4.20
* New Toolbar
* React Landing page
* Legacy Link ON/OFF
* Sample Homes ON/OFF (Button copy: "Sample Gallery")
* AI ON/OFF (Button copy: "AI Instant Design")
* DIY ON/OFF (Button copy: "Upload Home") _*only enabled if separate access from AI is requested in the SOW_
* Design Services ON/OFF (Button copy: "Design Services")
* Get Started Page ON/OFF (Button copy: "Design Your Home") _*i.e. as seen in Ply Gem_
* Header CTA ON/OFF (Button copy: TBD)

Please aim to have this completed by [target date from Step 1E, or XX 202X if not specified]. Let me know if you have any questions.
```

**Backend - 3D Product Viewer Deployment (Task) — label: `WBS`**
```
_BA to add information when defining DPL components post asset collection._
```

**Frontend - Complete all S4 Configurations and Styling (Task) — label: `WBS`**
```
Complete all S4 configurations and styling in alignment with the app configuration page: [Client Name - App Configuration|{Confluence App Config URL}]
```

**Projects - Configure S4 App in Configuration Portal (Task) — label: `WBS`**
```
Configure S4 App in Configuration Portal
```

**Backend - Configure New Build S4 App (Task) — label: `WBS`**
```
Configure a new build S4 visualizer in alignment with the app configuration page [Client Name - App Configuration|{Confluence App Config URL}]

This includes any applicable:

* Implementing AI mapping
* Assign a product manufacturer
```

### Step 6E — Transition out-of-scope tickets to Archived

For any tickets that were created but are **not** in scope for this build (i.e., those the PM indicated are not needed), transition them to the **Archived** status using `mcp__claude_ai_Atlassian__transitionJiraIssue` with the Archived transition ID found in Step 3E.

Run all archive transitions in parallel for efficiency.

### Step 7E — Link all tickets to the Release

Link every active (non-Archived) ticket to the Release using `mcp__claude_ai_Atlassian__createIssueLink`:
- `inwardIssue`: the ticket key (the blocker)
- `outwardIssue`: the Release key (the blocked issue)
- `linkType`: `Blocker`

Create all links in parallel for efficiency.

### Step 8E — Standard dependency map

For Enterprise projects, present this standard dependency map for PM confirmation before creating links. These are inter-ticket dependencies only — ticket-to-Release links were already created in Step 7E and must not be duplicated here.

```
Dependency Map:
  DPL Asset Review → blocks → [all other DPL tickets]
    Reason: Asset requirements must be confirmed before DPL production work begins
  SP - Mask and deploy → blocks → SP - Prepare shadow files
    Reason: Shadow files are prepared from the same images being masked
  SP - Prepare shadow files → blocks → SP - Update and deploy repo images
    Reason: Repo images depend on shadow files being ready
```

Ask: "Does this dependency map look correct for this project? Any additions or removals?"

Wait for explicit confirmation, then create links using `mcp__claude_ai_Atlassian__createIssueLink` with `linkType: Blocker`. Do not re-create any links already created in Step 7E.

### Step 9E — Build workback schedule and set dates

#### 9Ea — Scheduling rules

Use the project start date and target delivery date from Step 1E.

Use these default estimates for each ticket unless the PM provides alternatives:

| Ticket | Team | Default Estimate |
|--------|------|-----------------|
| DevOps - Vanity URL | DevOps | 4h |
| Backend - CRM Integration | Backend | 16h |
| DPL - Asset Review & Requirements | DPL | 4h |
| SP - Mask and deploy new sample images | SP | 8h (adjust based on scene count) |
| SP - Prepare shadow files | SP | 4h |
| SP - Implement sample photo tabs | SP | 4h |
| SP - Update and deploy repo images | SP | 4h |
| SP - Hotspot positioning | SP | 4h |
| Initial Build - DSA Analytics Setup | DSA | 8h |
| Design Team - Create branding mock-ups | Design | 8h |
| Backend - 3D Product Viewer Deployment | Backend | 8h |
| Frontend - Complete all S4 Configurations | Frontend | 16h |
| Projects - Configure S4 App | Projects | 8h |
| Backend - Configure New Build S4 App | Backend | 16h |

Apply the same scheduling rules as the Platform path:
- Working days only (Mon–Fri), 6h per day per developer
- Resource-constrained: same-team tasks are sequential
- Cross-team tasks can overlap if no dependency blocks them
- Duration = `ceil(estimate_hours / 6)` working days, rounded up

#### 9Eb — Present the schedule for confirmation

Output a schedule table before setting any dates. Columns: Key | Summary | Type | Est | Start | Due | Depends On.

If a target delivery date was provided and the calculated end date exceeds it, flag the risk:
> "⚠ Schedule risk: projected completion is YYYY-MM-DD, which is X days past the target of YYYY-MM-DD."

Ask: "Does this schedule look correct? Any adjustments before I apply the dates to Jira?"

Wait for explicit confirmation.

#### 9Ec — Apply dates to Jira

For each active ticket, call `mcp__claude_ai_Atlassian__editJiraIssue` to set:
- `startDate` — calculated start date in `YYYY-MM-DD` format
- `duedate` — calculated due date in `YYYY-MM-DD` format

Skip date-setting on any tickets transitioned to Archived.

> **Known limitation:** `startDate` may be locked in company-managed projects. Notify the user if it fails.

Apply all date updates in parallel for efficiency.

### Step 10E — Print summary

Output a table of all created issues:

| Type | Key | Summary | Status | Estimate | Start | Due |
|------|-----|---------|--------|----------|-------|-----|
| Epic | S4-xxx | [Client Name] [SOW-REF] | To Do | — | — | — |
| Story | S4-xxx | DevOps - Vanity URL | To Do | 4h | ... | ... |
| DPL Task | S4-xxx | DPL - [PROJECT SET UP] Asset Review & Requirements | To Do | 4h | ... | ... |
| Task | S4-xxx | Design Team - Create branding mock-ups | Archived | — | — | — |
| ... | | | | | | |
| Release | S4-xxx | Release [Client Name] New Build | To Do | — | ... | ... |

Include direct Jira links where possible: `https://renoworks.atlassian.net/browse/{KEY}`

#### Teams Channel Summary

After the table, generate two ready-to-paste paragraphs for posting in the Teams project channel.

**Format:**
```
[Client Name] [SOW-REF] - App Configuration - {Confluence App Config URL}
{App Config paragraph}

[Client Name] [SOW-REF] - Jira Epic - {Epic URL}
{Jira Epic paragraph}
```

**App Config paragraph** — 2–3 sentences: what the App Config page is, what it covers (product catalogue, branding, configuration requirements, sample scenes), and that it's the single source of truth for all implementation work.

**Jira Epic paragraph** — 2–3 sentences: reference the SOW and App Config as scope sources, describe the ticket breakdown (how many Dev/DPL/SP tickets are active), and state the total estimated effort as `~Xh`.

---

## Notes

- Always confirm with the user before creating any issues
- If a required issue type (e.g. DPL Task, DS Task) does not exist in the target project, inform the user and ask how to proceed
- If the Confluence page is ambiguous, ask clarifying questions rather than guessing
- Prefer creating fewer, well-defined issues over many vague ones
