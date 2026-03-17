# Scout — Browser Automation for Claude Code

Give Claude Code a browser. Scout page structure, interact with websites, and export replayable automations.

Scout reports cost **~200 tokens** vs ~124,000 for screenshot-based approaches — making browsing cheap enough to be casual.

## Prerequisites

- **Node.js** (for MCP server transport)
- **Google Chrome** (for browser automation)
- **Python 3.11+** (for the MCP server runtime)

## Install

From the Claude Code plugin marketplace:

```
/plugin install scout
```

Or load manually for testing:

```bash
claude --plugin-dir ./scout-browser
```

## Commands

### `/scout:scout <url>`

Open a browser and scout a website's page structure. Returns a compact overview of metadata, iframes, shadow DOM, and interactive elements. Keeps the session alive for follow-up interactions.

### `/scout:export-workflow [name]`

Export the current browser session as a replayable automation package:
- **Python script** — standalone botasaurus-driver script with realistic timing
- **Workflow JSON** — portable machine-readable workflow definition
- **requirements.txt** — pip dependencies
- **.env.example** — credential template (only when credentials are detected)

### `/scout:schedule [list | workflow-name <when> | delete <name>]`

Schedule exported workflows to run automatically using your OS task manager (Windows Task Scheduler / macOS launchd / Linux cron).

## The Lifecycle

```
Scout → Interact → Export → Schedule
```

1. **Scout** a page to understand its structure
2. **Interact** — click, type, navigate through the workflow
3. **Export** the session as a standalone Python script
4. **Schedule** it to run on a recurring basis

## MCP Server

This plugin uses [scout-mcp-server](https://pypi.org/project/scout-mcp-server/) (`@stemado/scout-mcp` on npm) for browser automation. The MCP server is installed automatically when the plugin loads.

## License

MIT
