---
name: scout
description: "Use when the user asks to automate a website, scout a page, explore page structure, build an automation script, navigate a portal, log into a site, inspect iframes or shadow DOM, download files from a web application, fetch page content, read a URL, check what a page says, get content from a site, look up a page, scrape a page, or do a quick page lookup. Trigger phrases include scout, automate this site, explore this page, figure out how this works, what does this page look like, navigate to, open a browser, browse this site, find elements, find the button, fetch this page, read this URL, what does this page say, browse this URL, get the content, look up, pull up, scrape this."
argument-hint: "<url>"
allowed-tools:
  - "mcp__plugin_scout_scout__*"
---

You have access to a live browser via the `scout` MCP server. It launches a browser session (Botasaurus) and lets you interactively explore websites through Chrome DevTools Protocol.

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

## When invoked with a URL

If the user provides a URL (via `/scout:scout <url>` or naturally in conversation):

1. Call `launch_session` with the URL. Use headed mode (headless=false) so the user can observe.

   **Localhost detection:** If the URL targets localhost, 127.0.0.1, [::1], or any other loopback address, extract the port number from the URL and pass it as `allow_localhost_port=<port>` to `launch_session`. If no explicit port is in the URL, use 80 for http or 443 for https.

2. Call `scout_page_tool` with the session_id to get a compact page overview.

3. Present the scout report to the user:
   - Page title and URL
   - Number of iframes (noting any cross-origin)
   - Shadow DOM boundaries found
   - Element counts by type (buttons, inputs, links, etc.)

4. Ask the user what they'd like to do next:
   - Find specific elements (use `find_elements` with a query)
   - Explore further (click, type, navigate)
   - Export the session as an automation script
   - Close the session

Keep the session alive for follow-up interactions. Do NOT close it unless the user asks.

If no URL is provided and the context doesn't imply one, ask the user what site they want to scout.

## Core Workflow: Scout -> Find -> Act

1. **Launch** — `launch_session` with the target URL (headless=false so the user can observe)
2. **Scout** — `scout_page_tool` for a compact page overview (~200 tokens): metadata, iframes, shadow DOM, element counts
3. **Find** — `find_elements` to search by text, type, or CSS selector
4. **Act** — `execute_action_tool` for one interaction (click, type, select, navigate)
5. **Scout again** — See what changed after the action
6. **Repeat** until the workflow is complete

## Key Tools

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

## When Clicks Fail

If `execute_action_tool` returns a `warning` on a click:
1. Use `inspect_element_tool` to check visibility, overlays, and shadow DOM context
2. Use `take_screenshot_tool` for visual confirmation
3. Try `execute_javascript` to dispatch a JS click: `document.querySelector('selector').click()`

## Session Rules

- **Login once.** Repeated logins trigger CAPTCHAs and account locks.
- **Scout after every action.** The DOM may have changed entirely.
- **Never guess structure.** Always scout before generating automation code.

## Script Generation

When composing botasaurus-driver scripts:
- Use **exact selectors** from `find_elements`
- Include **iframe context switching** where elements live in iframes
- **Parameterize credentials** — use env vars, never hardcode
- Realistic browser behavior is automatic — `Driver()` handles this out of the box
