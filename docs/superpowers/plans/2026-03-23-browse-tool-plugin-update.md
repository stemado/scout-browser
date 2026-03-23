# Browse Tool Plugin Update — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Update the scout-browser plugin so Claude Code knows when and how to use the new `browse` MCP tool — a lightweight, stateless alternative to the full Scout session flow.

**Architecture:** Four files need updating: the scout skill (primary — routing logic and tool docs), the plugin's CLAUDE.md (system prompt — high-level awareness), the user's global CLAUDE.md (always-on awareness across all projects), and plugin.json (description update). The `allowed-tools` wildcard `mcp__plugin_scout_scout__*` already covers the new `browse` tool.

**Tech Stack:** Markdown (skill files, CLAUDE.md)

**Context:** The `browse` tool is being added to scout-mcp (see `D:\Projects\scout-mcp\docs\superpowers\plans\2026-03-22-browse-tool.md`). It provides a single-call page fetch with content extraction — HTTP-first with stealth browser fallback, trafilatura extraction, BM25 keyword scoring, and optional LLM query extraction. The plugin must teach Claude the decision boundary between `browse` (read content) and the full session flow (interact with pages).

---

## File Structure

| File | Responsibility |
|------|---------------|
| **Modify:** `skills/scout/SKILL.md` | Add browse tool routing logic, tool reference, usage guidance |
| **Modify:** `CLAUDE.md` | Add browse tool as a lightweight browsing option in the plugin system prompt |
| **Modify:** `C:\Users\sdoherty\.claude\CLAUDE.md` | Add browse tool to the global "Browser & Web Capabilities" section |
| **Modify:** `.claude-plugin/plugin.json` | Update description to mention content fetching |

---

### Task 1: Update Scout Skill — Add Browse Tool Routing and Documentation

**Files:**
- Modify: `skills/scout/SKILL.md`

The scout skill is the primary place Claude learns how to use Scout tools. We need to:
1. Update the trigger description to include browse-appropriate phrases
2. Add a decision framework: browse vs session
3. Add the browse tool to the tool table
4. Add a browse usage section with parameter docs

- [ ] **Step 1: Update the skill trigger description**

In `skills/scout/SKILL.md`, replace the existing `description` field in frontmatter with a version that includes browse-related trigger phrases. The new description should add: "fetch page content", "read a URL", "what does this page say", "get content from", "browse a URL", "quick page lookup".

Replace the current description line:

```yaml
description: "Use when the user asks to automate a website, scout a page, explore page structure, build an automation script, navigate a portal, log into a site, inspect iframes or shadow DOM, or download files from a web application. Trigger phrases include scout, automate this site, explore this page, figure out how this works, what does this page look like, navigate to, open a browser, browse this site, find elements, find the button."
```

With:

```yaml
description: "Use when the user asks to automate a website, scout a page, explore page structure, build an automation script, navigate a portal, log into a site, inspect iframes or shadow DOM, download files from a web application, fetch page content, read a URL, check what a page says, get content from a site, look up a page, scrape a page, or do a quick page lookup. Trigger phrases include scout, automate this site, explore this page, figure out how this works, what does this page look like, navigate to, open a browser, browse this site, find elements, find the button, fetch this page, read this URL, what does this page say, browse this URL, get the content, look up, pull up, scrape this."
```

- [ ] **Step 2: Add the browse vs session decision framework**

After the opening paragraph (line 9: "You have access to a live browser...") and before "## When invoked with a URL", insert a new section:

```markdown
## Choosing the Right Tool: `browse` vs Session

Scout provides two modes of operation. **Choose based on what the user needs:**

| Need | Tool | Why |
|------|------|-----|
| **Read** page content, fetch docs, look up information | `browse` | One call. No session overhead. Returns clean markdown. |
| **Interact** with a page — click, type, navigate, fill forms | Full session (`launch_session` → ...) | Stateful browser needed for multi-step interaction. |
| Fetch content from a **bot-protected** site | `browse` | Has automatic stealth browser fallback built in. |
| **Automate** a workflow or generate a script | Full session | Need to explore page structure interactively first. |
| Quick **API/JSON** endpoint check | `browse` | Handles JSON responses natively (pretty-prints them). |
| **Login** to a portal or fill credentials | Full session | Need `fill_secret` and interactive form submission. |

**Default to `browse`** when the user just wants to read or fetch content. It's faster, cheaper, and stateless.

**Default to a session** when the user needs to interact with the page, see it visually, or build automation.
```

- [ ] **Step 3: Add browse tool to the Key Tools table**

In the "## Key Tools" table, add `browse` as the **first row** (since it's the lightest-weight tool and the first one Claude should consider):

Add this row after the table header:

```markdown
| `browse` | Fetch a URL and extract clean content — one call, no session needed |
```

The updated table should read:

```markdown
| Tool | Purpose |
|------|---------|
| `browse` | Fetch a URL and extract clean content — one call, no session needed |
| `launch_session` | Open browser at a URL |
| `scout_page_tool` | Compact page structure report |
| `find_elements` | Search for interactive elements |
| `execute_action_tool` | Click, type, select, navigate, scroll |
| `fill_secret` | Type credentials from .env (never exposed to AI) |
| `execute_javascript` | Run arbitrary JS in page context |
| `take_screenshot_tool` | Visual capture for debugging |
| `inspect_element_tool` | Deep element inspection (visibility, shadow DOM, overlays) |
| `monitor_network` | Intercept XHR/fetch calls to discover APIs |
| `close_session` | Release the browser |
```

- [ ] **Step 4: Add browse usage section**

After the "## Key Tools" table and before "## When Clicks Fail", insert a new section:

```markdown
## Using `browse` — Quick Content Fetch

`browse` fetches a URL, extracts clean content, and returns it in one call. No session needed.

**Parameters:**
- `url` (required) — The page URL to fetch
- `query` (optional) — Extract only content relevant to this query. Uses BM25 keyword scoring to return the most relevant passages.
- `max_length` (optional) — Cap response length in characters. `0` = unlimited. Default: 5000.

**Examples:**

```
# Fetch and read a page
browse(url="https://docs.example.com/api-reference")

# Fetch and extract only relevant content
browse(url="https://en.wikipedia.org/wiki/Rust_(programming_language)", query="memory safety")

# Fetch with no length cap
browse(url="https://example.com/changelog", max_length=0)
```

**What it returns:** `BrowseResult` with fields:
- `success` — whether the fetch succeeded
- `url` — final URL after redirects
- `title` — page title
- `content` — clean markdown content (full page or query-extracted passages)
- `extraction_mode` — `"full"` or `"extracted"` (when query was used)
- `fetch_method` — `"http"` or `"browser"` (if bot protection triggered fallback)
- `error` — error message if `success` is false

**How it works under the hood:**
1. HTTP fetch first (fast path, ~200ms)
2. If bot-blocked (Cloudflare, CAPTCHAs), falls back to a transient stealth browser
3. Content extraction via trafilatura (strips nav, ads, footers — keeps article content)
4. If `query` provided: BM25 keyword scoring selects the most relevant passages (or LLM extraction if `SCOUT_BROWSE_MODEL` env var is configured, e.g. `anthropic:claude-sonnet-4-20250514`)
5. Truncation at paragraph boundaries to respect `max_length`
```

- [ ] **Step 5: Verify the allowed-tools wildcard covers browse**

The current `allowed-tools` in frontmatter is:
```yaml
allowed-tools:
  - "mcp__plugin_scout_scout__*"
```

This wildcard already matches `mcp__plugin_scout_scout__browse`. **No change needed.** Just verify this is still present and correct.

- [ ] **Step 6: Commit**

```bash
git add skills/scout/SKILL.md
git commit -m "feat: add browse tool routing and documentation to scout skill"
```

---

### Task 2: Update CLAUDE.md — Add Browse Tool Awareness

**Files:**
- Modify: `CLAUDE.md`

The CLAUDE.md is injected into every conversation as system prompt context. It needs to mention the browse tool so Claude is aware of it even before the skill is invoked.

- [ ] **Step 1: Add browse tool section to CLAUDE.md**

After the "#### Philosophy" section (ends with the "Do not guess at page structure" paragraph) and before the "#### Session Lifecycle" section, insert:

```markdown
#### Two Modes of Browsing

Scout has two modes. Pick the right one:

- **`browse`** — One tool call. Fetches a URL, extracts clean markdown content. Use for reading docs, checking pages, looking up information. Supports query filtering to extract only relevant passages. No session, no cleanup.
- **Full session** (`launch_session` → `scout_page_tool` → `find_elements` → `execute_action_tool` → `close_session`) — Stateful interactive browser. Use for clicking, typing, logging in, navigating multi-page flows, or building automation scripts.

**Default to `browse` for read-only tasks.** It's faster and requires no session management.
```

- [ ] **Step 2: Add browse to the Workflow Pattern section**

At the top of the "#### Workflow Pattern" section, add a note before the numbered list:

```markdown
> **For read-only tasks:** Skip the workflow below — use `browse(url, query?)` instead. One call, clean content out.
```

- [ ] **Step 3: Commit**

```bash
git add CLAUDE.md
git commit -m "feat: add browse tool awareness to CLAUDE.md system prompt"
```

---

### Task 3: Update Global CLAUDE.md — Always-On Browse Awareness

**Files:**
- Modify: `C:\Users\sdoherty\.claude\CLAUDE.md`

The global CLAUDE.md is loaded in every conversation across all projects. It has a "Browser & Web Capabilities (Scout MCP)" section that lists all Scout tools. The `browse` tool must be added here so Claude discovers it even when the scout skill hasn't been triggered.

- [ ] **Step 1: Add browse to the "When to Use Scout" list**

In the "### When to Use Scout" section, add a new bullet at the top of the list:

```markdown
- Need to quickly read a page, fetch docs, or check a changelog — use `browse` (one tool call, no session)
```

- [ ] **Step 2: Add browse as a fast path before the Scout/Find/Act Cycle**

Before the "### The Scout/Find/Act Cycle" section, insert:

```markdown
### Quick Fetch with `browse`

For **read-only** tasks — checking docs, reading changelogs, looking up information — use `browse` instead of the full session cycle below. One tool call, clean markdown content out.

- `browse(url)` — Fetch page, extract clean content
- `browse(url, query="specific topic")` — Fetch and extract only relevant passages
- `browse(url, max_length=0)` — Fetch with no length cap (default: 5000 chars)

Falls back to a stealth browser automatically if the site blocks HTTP requests.
```

- [ ] **Step 3: Add browse to the "Browsing is Cheap" section**

In the "### Browsing is Cheap — Use It Liberally" section, add:

```markdown
- `browse` is even cheaper — no session overhead, no cleanup, just content
```

- [ ] **Step 4: Commit**

```bash
git add "C:\Users\sdoherty\.claude\CLAUDE.md"
git commit -m "feat: add browse tool to global CLAUDE.md for cross-project awareness"
```

Note: This file is outside the plugin repo. Commit separately or skip if the user prefers to manage it manually.

---

### Task 4: Update plugin.json Description

**Files:**
- Modify: `.claude-plugin/plugin.json`

- [ ] **Step 1: Update the description field**

In `.claude-plugin/plugin.json`, replace the `description` value:

Current:
```json
"description": "Give Claude Code a browser. Scout page structure, interact with websites, and export replayable automations."
```

New:
```json
"description": "Give Claude Code a browser. Fetch page content, scout page structure, interact with websites, and export replayable automations."
```

- [ ] **Step 2: Commit**

```bash
git add .claude-plugin/plugin.json
git commit -m "chore: update plugin description to mention content fetching"
```

---

### Task 5: Verification

- [ ] **Step 1: Read the updated SKILL.md end-to-end**

Read `skills/scout/SKILL.md` from top to bottom. Verify:
- The decision framework is clear and actionable
- The browse tool is the first row in the tools table
- The browse usage section has correct parameters and examples
- The allowed-tools wildcard is present and unchanged
- The existing session workflow documentation is untouched

- [ ] **Step 2: Read the updated CLAUDE.md end-to-end**

Read `CLAUDE.md` from top to bottom. Verify:
- The "Two Modes" section is positioned before session-specific content
- The workflow pattern note correctly references browse
- All existing content is preserved

- [ ] **Step 3: Read the updated global CLAUDE.md**

Read `C:\Users\sdoherty\.claude\CLAUDE.md` and verify:
- The "Quick Fetch with `browse`" section is positioned before the Scout/Find/Act Cycle
- The "When to Use Scout" list includes the browse bullet
- The "Browsing is Cheap" section mentions browse
- Parameter names are consistent with other files

- [ ] **Step 4: Check for consistency across all files**

Verify that:
- All three files (SKILL.md, plugin CLAUDE.md, global CLAUDE.md) describe browse the same way
- No contradictions between the decision framework in the skill and the "Two Modes" section
- The parameter names match what the browse tool actually accepts: `url`, `query`, `max_length`
- The plugin.json description reads naturally

- [ ] **Step 5: Commit (if any fixes needed)**

```bash
git add skills/scout/SKILL.md CLAUDE.md .claude-plugin/plugin.json
git commit -m "fix: consistency pass on browse tool documentation"
```
