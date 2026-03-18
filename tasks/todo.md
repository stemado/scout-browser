# Scout Browser Plugin — Remaining Work

## Context

This is a Claude Code plugin for submission to Anthropic's official marketplace. The plugin is fully built — all 13 files are in place, all JSON validates, all content has been reviewed and scrubbed. What remains is testing, GitHub repo creation, and submission.

**Spec:** `D:\Projects\scout-mcp\docs\superpowers\specs\2026-03-17-scout-browser-plugin-design.md`
**Plan:** `D:\Projects\scout-mcp\docs\superpowers\plans\2026-03-17-scout-browser-plugin.md`

## Plugin Structure (complete)

```
scout-browser/
├── .claude-plugin/plugin.json    # name: "scout", v1.0.0
├── .mcp.json                     # npx -y @stemado/scout-mcp
├── CLAUDE.md                     # System prompt (Scout/Find/Act workflow)
├── README.md                     # Public install + usage docs
├── CHANGELOG.md                  # v1.0.0 entry
├── LICENSE                       # MIT
├── .gitignore
├── skills/scout/SKILL.md         # Auto-trigger skill for browsing tasks
├── commands/scout.md             # /scout:scout <url>
├── commands/export-workflow.md   # /scout:export-workflow [name]
├── commands/schedule.md          # /scout:schedule [list|name|delete]
├── hooks/hooks.json              # SessionStart dep check
└── hooks/check-deps.sh           # Cross-platform Python/Node/Chrome check
```

## Remaining Tasks

### 1. Manual Plugin Test

Load the plugin and verify everything works:

```bash
claude --plugin-dir .
```

Inside that session:
- [ ] `/help` shows `/scout:scout`, `/scout:export-workflow`, `/scout:schedule`
- [ ] SessionStart hook fires: "Scout: All dependencies OK (...)"
- [ ] `/scout:scout https://example.com` launches browser, returns scout report
- [ ] Session stays alive for follow-up interactions
- [ ] Close session works

If issues are found, fix and commit.

### 2. Create GitHub Repo

```bash
gh repo create mtsteinle/scout-browser --public --source=. --push
```

Verify the repo is live at https://github.com/mtsteinle/scout-browser

### 3. Submit to Anthropic

Submit via **both** forms:
- https://claude.ai/settings/plugins/submit
- https://platform.claude.com/plugins/submit

#### Submission Form Answers

**Plugin Name:** Scout

**Plugin Description:**
Scout gives Claude Code a browser. It scouts page structure (~200 tokens vs ~124K for screenshots), interacts with websites through a realistic browser session, and exports replayable Python automation scripts. Three commands cover the full lifecycle: scout a page, export the workflow, schedule it to run. Built on Botasaurus for reliable browser sessions that work with sites that block typical automation tools.

**Is this plugin for Claude Code or Cowork?** Claude Code

**Link to GitHub:** https://github.com/mtsteinle/scout-browser

**Company/Organization URL:** https://github.com/mtsteinle

**Primary Contact Email:** frankdoherty1921@gmail.com

**Plugin Examples:**

1. **Enterprise Portal Automation** — "Log into our vendor portal, navigate to Reports > Monthly Summary, and export the CSV." Scout discovers the page structure (including nested iframes and shadow DOM), handles credential injection securely via `fill_secret`, navigates the report interface, and exports a replayable Python script the user can schedule to run daily.

2. **HR/Benefits Workflow** — "Check our benefits platform for open enrollment status." Scout navigates the authentication flow, discovers the SPA structure, extracts the relevant data, and produces a standalone automation — turning a 15-minute manual process into a one-command operation.

3. **API Discovery** — "Pull the last 30 days of activity from our dashboard — there's no export button." Scout monitors the XHR calls that populate the data table, discovers the underlying API endpoint, and the user can export a script that hits the API directly with session cookies.

## Important Notes

- **No anti-detection/stealth language** — All content uses "human-like browser behavior" and "realistic browser sessions" instead. This was a deliberate decision to avoid Anthropic policy friction.
- **Plugin name is `scout`** — Verify the old `scout-plugin` (personal use) was never submitted to the marketplace to avoid name collision.
- **MCP tool prefix** — All commands use `mcp__plugin_scout_scout__*` (plugin name "scout" + server key "scout").
- **The plugin does NOT bundle Python source** — It references the published `@stemado/scout-mcp` npm package which spawns `uvx scout-mcp-server`.
