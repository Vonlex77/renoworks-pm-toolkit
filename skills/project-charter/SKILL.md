---
name: project-charter
description: Fill in sections of a Confluence implementation/project page by reading a source discovery or requirements page. Use this skill when the user wants to populate sections like "Project Objective", "Business Objectives", "Success Criteria", "Risk Assessment", or similar structured sections on a Confluence page using content from another Confluence page.
---

# Project Charter

Reads a source Confluence page (e.g. a discovery or requirements doc) and uses the content to fill in structured sections on a target Confluence page (e.g. an implementation or project brief).

**CloudId:** `renoworks.atlassian.net`

## Workflow

### Step 1 — Get source and target page URLs

If the user has not already provided both URLs, ask:
- "What is the URL of the **source** page (e.g. discovery doc, requirements doc)?"
- "What is the URL of the **target** page to fill in?"
- "Which sections would you like me to fill in, or shall I fill all empty ones I find?"

Extract the numeric page ID from the URL path (e.g. `.../pages/3670278205/...` → `3670278205`).

### Step 2 — Read both pages in parallel

Fetch both pages simultaneously using `mcp__claude_ai_Atlassian__getConfluencePage`:
- Source page: use `contentFormat: "markdown"` for easy reading
- Target page: use `contentFormat: "adf"` to get the full ADF structure needed for the update

### Step 3 — Identify sections to fill

Inspect the target page ADF for empty or placeholder sections. Common sections include:
- **Project Objective** — often an info panel with an empty paragraph
- **Business Objectives** — often a bullet list with one empty list item
- **Success Criteria** — often a bullet list with one empty list item
- **Risk Assessment** — often a bullet list with one empty list item

If the user specified particular sections, focus on those. Otherwise fill all empty sections you find.

### Step 4 — Draft content for each section

Using the source page content, synthesise appropriate content for each section:

**Project Objective**
A concise 2–4 sentence paragraph describing:
- What the feature/project enables or changes
- The current limitation or gap it addresses
- The approach being taken (at a high level)

**Business Objectives**
3–5 bullet points covering:
- Who benefits and how (end users, clients, AppBuilders, etc.)
- What new capability or control is being introduced
- Any UX or quality improvements in scope
- What existing behaviour must be preserved (no regression)

**Success Criteria**
5–8 bullet points that are specific and testable, covering:
- Core functional outcomes (feature works as expected per role/user type)
- Data persistence and API propagation
- UI/UX conditions (labels, flows, states)
- No-regression conditions
- QA/testing pass condition

**Risk Assessment**
4–6 bullet points, each formatted as:
**[Risk name]**: description of the risk and why it matters. Mitigation: specific action to reduce the risk.

Common risk categories to consider: cost implications, technical complexity/new patterns, UX edge cases, auth/session state, unintended side effects of the change.

### Step 5 — Present draft and confirm

Show the drafted content for each section clearly, grouped by section name.

Ask: "Does this look right? Any changes before I update the page?"

Wait for explicit confirmation before proceeding.

### Step 6 — Update the target page

**Do not manually construct ADF JSON.** Use the file-based Python workflow instead:

#### Pre-flight check — paragraph-in-table

Before modifying the ADF, inspect the fetched body for any `paragraph` node that appears directly inside a `tableBody` or `table` content array (alongside `tableRow` nodes). Confluence accepts this on read but rejects it on write (400 error). Fix: move such nodes into the nearest `layoutColumn`'s content array instead.

#### Modification workflow

1. Save the fetched ADF body to `/tmp/target_adf.json` via `json.dump`
2. Load it back and make targeted mutations — find nodes by `type` or `attrs.localId`, replace their `content` arrays in place
3. Empty paragraph: replace content with `[{"text": "...", "type": "text"}]`
4. Empty bullet list: replace the single empty `listItem` with populated `listItem` nodes, each with a fresh unique `localId` (e.g. `"sc-02"`, `"risk-03"`)
5. Risk Assessment items: inline bold for risk name, plain text for description + mitigation — no nested lists
6. Write result to `/tmp/adf_body_fixed.json` using `separators=(',', ':')`
7. Validate: `node -e "JSON.parse(require('fs').readFileSync('/tmp/adf_body_fixed.json','utf8')); console.log('VALID')"`

Preserve all existing `localId` values for unchanged nodes. Never truncate or omit existing sections (tables, RACI, deliverables, layout).

Call `mcp__claude_ai_Atlassian__updateConfluencePage` with:
- `contentFormat: "adf"`
- `body`: contents of `/tmp/adf_body_fixed.json` (read via Bash)
- `versionMessage`: a short description of what was filled in (e.g. `"Filled in Project Objective, Business Objectives, Success Criteria, and Risk Assessment"`)

### Step 7 — Confirm and share link

After a successful update, confirm to the user:
- Which sections were filled
- The version number of the updated page
- A direct link to the page: `https://renoworks.atlassian.net/wiki/spaces/.../pages/{pageId}/...`

## Notes

- Always read the target page in ADF format — the full document must be reconstructed and sent back in its entirety; partial updates are not supported
- Never truncate or omit existing sections (tables, RACI, deliverables, team structure) when reconstructing the ADF body
- If the source page mentions cost implications, UX limitations, or technical caveats, these should be surfaced in the Risk Assessment
- If the source page explicitly names an approved implementation option (e.g. "Option 2"), base the Project Objective and approach description on that option
- If the target page has sections not listed above (e.g. "Assumptions", "Out of Scope"), ask the user whether to fill those too
