# Session History Archive — Design Spec

## Problem

Users naturally want to export a workflow **after** completing their browsing session. The current flow forces them to export **before** closing the session, because `close_session` destroys all in-memory history by deleting the `BrowserSession` object from `AppContext.sessions`.

When `/scout:export-workflow` is called after the session is closed, `get_session_history` fails with:

```
Error: No active session with id '<id>'. Use launch_session first.
```

This is a recurring pain point. The natural user flow is: browse → close → export. The system requires: browse → export → close.

## Solution

**In-memory archive** — when a session is closed, serialize its history and store it in a lightweight archive dict on `AppContext`. Modify `get_session_history` to fall back to the archive when the session is no longer active.

### Why this approach

- Same-conversation-only scope (no disk persistence needed)
- Minimal change: 3 code edits + 1 docstring update to one file (`server.py` in Scout-mcp)
- Zero changes to the `export-workflow` command — it already calls `get_session_history` with the session_id
- No risk to other tools — only `get_session_history` gets the fallback; all other tools still require active sessions via `_get_session()`

## Design

### Change 1: Add archive storage to `AppContext`

**File:** `Scout-mcp/src/scout/server.py`, `AppContext` dataclass (line 65)

Add two fields:

```python
_closed_histories: dict[str, str] = field(default_factory=dict)
_max_closed_histories: int = 5
```

- Keys: session_id strings
- Values: sanitized history strings (output of `sanitize_response`)
- Cap: 5 entries, FIFO eviction (oldest removed when exceeded)
- Python 3.7+ dict insertion order guarantees correct eviction via `next(iter(dict))`

### Change 2: Archive history before deleting session in `close_session`

**File:** `Scout-mcp/src/scout/server.py`, `close_session` handler (line 1209)

After `session.close()` returns but **before** `del app_ctx.sessions[session_id]`, insert:

```python
# Archive history for post-close retrieval (e.g., export-workflow)
history = session.history.get_full_history()
archived = history.model_dump(exclude_none=True)
archived["security_summary"] = session.security_counter.summary()
app_ctx._closed_histories[session_id] = sanitize_response(
    archived, secrets=session._secret_values
)
# Evict oldest if over limit
if len(app_ctx._closed_histories) > app_ctx._max_closed_histories:
    oldest = next(iter(app_ctx._closed_histories))
    del app_ctx._closed_histories[oldest]
```

**Why after `session.close()` but before `del`:**
- `session.close()` stops the network monitor first (no race condition on event appends)
- `session.close()` does NOT clear `history`, `_secret_values`, or `security_counter` — all remain available
- `sanitize_response` scrubs secrets and strips invisible Unicode before storage, so the archive is safe to return directly

### Change 3: Add fallback to `get_session_history`

**File:** `Scout-mcp/src/scout/server.py`, `get_session_history` handler (line 955)

Replace the direct `_get_session` call with a try/except that falls back to the archive:

```python
app_ctx = _get_ctx(ctx)

# Validate format before attempting lookup (preserves specific error message)
if not _SESSION_ID_RE.match(session_id):
    raise ValueError("Invalid session ID format: expected 12 hex characters.")

# Try active session first
session = app_ctx.sessions.get(session_id)
if session is None or not session.is_active:
    # Fall back to closed session archive
    archived = app_ctx._closed_histories.get(session_id)
    if archived is not None:
        await ctx.info("Returning history from closed session archive.")
        return archived
    raise ValueError(
        f"No session with id '{session_id}' (active or recently closed). "
        "Use launch_session first."
    )

history: SessionHistory = session.history.get_full_history()
# ... rest unchanged ...
```

**Why this structure:**

- Format validation happens first, preserving the specific "expected 12 hex characters" error message
- Active sessions take the fast path (no overhead)
- Only when the session is missing/inactive do we check the archive
- If neither active nor archived, raise with an updated error message that mentions "recently closed"
- The archived value is already sanitized, so it's returned directly

### Docstring update

Update the `get_session_history` docstring `Args` from:

```
session_id: Active session ID.
```

To:

```
session_id: Session ID (active or recently closed).
```

## What does NOT change

- **`export-workflow` command** — works unchanged. It calls `get_session_history` with the session_id Claude remembers from the conversation.
- **All other tools** — unchanged. They use `_get_session()` which enforces active sessions. No risk of accidentally operating on dead sessions.
- **`_get_session` helper** — unchanged. It still enforces active sessions for all tools except `get_session_history`.
- **Session cleanup on shutdown** — `_closed_histories` is garbage collected with `AppContext`. No file cleanup needed.
- **`SessionHistoryTracker`** — unchanged. No modifications to `history.py`.

## Edge cases

| Scenario | Behavior |
|----------|----------|
| Export immediately after close (same conversation) | Archive hit, history returned |
| Export from a session that was never closed (still active) | Active session path, no change |
| Export after server restart | Archive lost (acceptable — same-conversation scope only) |
| 6th session closed (exceeds cap) | Oldest archive evicted, most recent 5 retained |
| Invalid session_id format | Inline `_SESSION_ID_RE` check raises before archive lookup (format validation preserved) |
| Session closed, then same ID relaunched | New active session takes precedence (dict lookup finds active session on the fast path) |

## Testing

1. Launch session, perform actions, close session, call `get_session_history` — should return archived history
2. Launch and close 6 sessions, verify only last 5 are archived
3. Verify archived history has secrets scrubbed (boundary markers present, no raw credential values)
4. Verify active session path is unaffected (no performance regression)
5. Verify other tools still reject closed session IDs
