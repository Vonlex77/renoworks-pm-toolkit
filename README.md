# Renoworks PM Toolkit

A Claude Code plugin for the Renoworks Project Management team. Bundles three skills for Jira and Confluence workflows.

## Included Skills

| Skill | Trigger phrase |
|---|---|
| **Jira Project Setup** | "Set up a new Jira project from this Confluence page" |
| **Jira Ticket Creator** | "Create a few Jira tickets for this Epic" |
| **Project Charter** | "Fill in the project charter from this discovery page" |

---

## One-Time Setup (per person, ~5 minutes)

### Step 1 — Enable the Atlassian connector

These skills connect to Jira and Confluence via the Atlassian integration built into Claude.

1. Go to [claude.ai/settings/connectors](https://claude.ai/settings/connectors)
2. Find **Atlassian** and click **Connect**
3. Sign in with your `@renoworks.com` Atlassian account and grant access

You only need to do this once. The connection persists across sessions.

### Step 2 — Add the marketplace

In Claude Code, run:

```
/plugin marketplace add https://github.com/Vonlex77/renoworks-pm-toolkit
```

### Step 3 — Install the plugin

```
/plugin install renoworks-pm-toolkit@renoworks-marketplace
```

### Step 4 — Reload plugins

```
/reload-plugins
```

That's it. The skills are now available in every Claude Code session.

---

## Using the Skills

Just describe what you want in plain language — Claude will pick up the right skill automatically.

**Jira Project Setup** — use when starting a new initiative from a Confluence project charter:
> "Set up the Jira project from this charter: https://renoworks.atlassian.net/wiki/..."

**Jira Ticket Creator** — use when adding a handful of tickets to an existing Epic:
> "Create two backend tickets and one QA ticket under the S4 Roles & Permissions epic"

**Project Charter** — use when you have a discovery doc and want to populate a charter template:
> "Fill in the project charter at [target URL] using the discovery doc at [source URL]"

---

## Getting Updates

When Chad publishes a new version, run:

```
/plugin update renoworks-pm-toolkit
/reload-plugins
```

---

## Troubleshooting

**Skills aren't appearing** — run `/reload-plugins` and try again.

**Atlassian tools not working** — check that your connector is still connected at [claude.ai/settings/connectors](https://claude.ai/settings/connectors). Re-authenticate if it shows as disconnected.

**Wrong Jira account** — the connector uses whichever Atlassian account you authenticated with. Make sure it's your `@renoworks.com` account.
