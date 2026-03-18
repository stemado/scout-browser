# Session History Archive Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Preserve session history after `close_session` so `/scout:export-workflow` works when called after a session is closed.

**Architecture:** Add a `_closed_histories` dict to `AppContext` that archives sanitized history strings on close. `get_session_history` falls back to the archive when the session is no longer active. All changes are in one file: `D:\Projects\Scout-mcp\src\scout\server.py`.

**Tech Stack:** Python 3.11+, Pydantic, FastMCP

**Spec:** `d:\Projects\scout-browser\docs\superpowers\specs\2026-03-17-session-history-archive-design.md`

---

### Task 1: Add archive fields to `AppContext`

**Files:**
- Modify: `D:\Projects\Scout-mcp\src\scout\server.py:65-73`

- [ ] **Step 1: Add `_closed_histories` and `_max_closed_histories` fields**

At `server.py:73`, after the `_extension_relay` field, add:

```python
    _closed_histories: dict[str, str] = field(default_factory=dict)
    _max_closed_histories: int = 5
```

The full `AppContext` should read:

```python
@dataclass
class AppContext:
    """Holds all browser sessions across the server's lifetime."""

    sessions: dict[str, BrowserSession] = field(default_factory=dict)
    max_sessions: int = 1
    _launch_lock: asyncio.Lock = field(default_factory=asyncio.Lock)
    _env_vars: dict[str, str] | None = field(default=None)
    _extension_relay: object | None = field(default=None)  # ExtensionRelay when active
    _closed_histories: dict[str, str] = field(default_factory=dict)
    _max_closed_histories: int = 5
```

- [ ] **Step 2: Verify no syntax errors**

Run: `cd D:/Projects/Scout-mcp && uv run python -c "from scout.server import AppContext; print('OK')"`
Expected: `OK`

- [ ] **Step 3: Commit**

```bash
cd D:/Projects/Scout-mcp
git add src/scout/server.py
git commit -m "feat: add closed session history archive to AppContext"
```

---

### Task 2: Archive history in `close_session`

**Files:**
- Modify: `D:\Projects\Scout-mcp\src\scout\server.py:1209-1210`
- Test: `D:\Projects\Scout-mcp\tests\test_history_archive.py`

- [ ] **Step 1: Write failing test for archive-on-close**

Create `D:\Projects\Scout-mcp\tests\test_history_archive.py`:

```python
"""Tests for session history archive — post-close history retrieval."""

from dataclasses import dataclass, field
from unittest.mock import MagicMock

from scout.history import SessionHistoryTracker
from scout.sanitize import _BOUNDARY_START, _BOUNDARY_END
from scout.security.audit_log import SecurityCounter
from scout.server import AppContext


def _make_session_stub(session_id: str = "aabbccddeeff") -> MagicMock:
    """Create a minimal BrowserSession-like stub with real history/security."""
    stub = MagicMock()
    stub.session_id = session_id
    stub.is_active = True
    stub.history = SessionHistoryTracker(session_id)
    stub.security_counter = SecurityCounter()
    stub._secret_values = set()
    # Simulate session.close() returning a result
    close_result = MagicMock()
    close_result.session_duration_seconds = 10.0
    close_result.total_actions_performed = 3
    close_result.total_scouts_performed = 1
    close_result.model_dump.return_value = {
        "closed": True,
        "session_duration_seconds": 10.0,
        "total_actions_performed": 3,
        "total_scouts_performed": 1,
    }
    stub.close.return_value = close_result
    return stub


class TestArchiveOnClose:
    """Verify that close_session preserves history in _closed_histories."""

    def test_history_archived_after_close(self):
        """After close, _closed_histories should contain the session's history."""
        ctx = AppContext()
        session = _make_session_stub("aabbccddeeff")
        session.history.record_navigation("https://example.com")
        ctx.sessions["aabbccddeeff"] = session

        # Simulate what close_session does: close, archive, delete
        session.close()

        history = session.history.get_full_history()
        archived = history.model_dump(exclude_none=True)
        archived["security_summary"] = session.security_counter.summary()
        from scout.sanitize import sanitize_response
        ctx._closed_histories["aabbccddeeff"] = sanitize_response(
            archived, secrets=session._secret_values
        )
        del ctx.sessions["aabbccddeeff"]

        assert "aabbccddeeff" in ctx._closed_histories
        archived_str = ctx._closed_histories["aabbccddeeff"]
        assert "example.com" in archived_str
        assert _BOUNDARY_START in archived_str

    def test_fifo_eviction_at_cap(self):
        """When more than _max_closed_histories sessions are archived, oldest is evicted."""
        ctx = AppContext()
        ctx._max_closed_histories = 3

        for i in range(4):
            sid = f"{i:012x}"
            ctx._closed_histories[sid] = f"history-{i}"
            if len(ctx._closed_histories) > ctx._max_closed_histories:
                oldest = next(iter(ctx._closed_histories))
                del ctx._closed_histories[oldest]

        assert len(ctx._closed_histories) == 3
        # First session (000000000000) should be evicted
        assert "000000000000" not in ctx._closed_histories
        # Last three should remain
        assert "000000000001" in ctx._closed_histories
        assert "000000000002" in ctx._closed_histories
        assert "000000000003" in ctx._closed_histories

    def test_secrets_scrubbed_in_archive(self):
        """Archived history should have secret values replaced with [REDACTED]."""
        ctx = AppContext()
        session = _make_session_stub("aabbccddeeff")
        session._secret_values = {"SuperSecret123"}
        # Record an action that contains the secret in its value
        from scout.models import ActionRecord
        session.history.record_action(ActionRecord(
            action="type",
            selector="#password",
            value="SuperSecret123",
            success=True,
        ))
        ctx.sessions["aabbccddeeff"] = session

        session.close()
        history = session.history.get_full_history()
        archived = history.model_dump(exclude_none=True)
        archived["security_summary"] = session.security_counter.summary()
        from scout.sanitize import sanitize_response
        ctx._closed_histories["aabbccddeeff"] = sanitize_response(
            archived, secrets=session._secret_values
        )

        archived_str = ctx._closed_histories["aabbccddeeff"]
        assert "SuperSecret123" not in archived_str
        assert "[REDACTED]" in archived_str
```

- [ ] **Step 2: Run tests to verify the archive mechanism works in isolation**

Run: `cd D:/Projects/Scout-mcp && uv run pytest tests/test_history_archive.py -v`
Expected: All 3 tests PASS — these validate the archive mechanism (serialization, FIFO eviction, secret scrubbing) before we wire it into `close_session`

- [ ] **Step 3: Add archive logic to `close_session`**

In `server.py`, replace lines 1209-1210:

```python
    result: SessionCloseResult = await asyncio.to_thread(session.close)
    del app_ctx.sessions[session_id]
```

With:

```python
    result: SessionCloseResult = await asyncio.to_thread(session.close)

    # Archive history for post-close retrieval (e.g., export-workflow)
    history = session.history.get_full_history()
    archived = history.model_dump(exclude_none=True)
    archived["security_summary"] = session.security_counter.summary()
    app_ctx._closed_histories[session_id] = sanitize_response(
        archived, secrets=session._secret_values
    )
    if len(app_ctx._closed_histories) > app_ctx._max_closed_histories:
        oldest = next(iter(app_ctx._closed_histories))
        del app_ctx._closed_histories[oldest]

    del app_ctx.sessions[session_id]
```

- [ ] **Step 4: Run tests to verify they still pass**

Run: `cd D:/Projects/Scout-mcp && uv run pytest tests/test_history_archive.py -v`
Expected: All 3 tests PASS

- [ ] **Step 5: Run full non-integration test suite to check for regressions**

Run: `cd D:/Projects/Scout-mcp && uv run pytest -m "not integration" -v`
Expected: All tests PASS

- [ ] **Step 6: Commit**

```bash
cd D:/Projects/Scout-mcp
git add src/scout/server.py tests/test_history_archive.py
git commit -m "feat: archive session history on close for post-close retrieval"
```

---

### Task 3: Add fallback to `get_session_history`

**Files:**
- Modify: `D:\Projects\Scout-mcp\src\scout\server.py:942-965`
- Test: `D:\Projects\Scout-mcp\tests\test_history_archive.py` (add tests)

- [ ] **Step 1: Write tests for the lookup fallback logic**

Append to `D:\Projects\Scout-mcp\tests\test_history_archive.py`:

```python
import re

import pytest

from scout.server import _SESSION_ID_RE


def _resolve_history(app_ctx: AppContext, session_id: str) -> str | None:
    """Mirror the lookup logic from get_session_history (without MCP Context).

    Returns the history string if found (active or archived), raises ValueError
    if invalid format or not found.
    """
    if not _SESSION_ID_RE.match(session_id):
        raise ValueError("Invalid session ID format: expected 12 hex characters.")

    session = app_ctx.sessions.get(session_id)
    if session is None or not session.is_active:
        archived = app_ctx._closed_histories.get(session_id)
        if archived is not None:
            return archived
        raise ValueError(
            f"No session with id '{session_id}' (active or recently closed). "
            "Use launch_session first."
        )
    # Active session found — return sentinel to distinguish from archive path
    return "ACTIVE_SESSION"


class TestGetSessionHistoryFallback:
    """Verify the lookup logic used by get_session_history."""

    def test_active_session_takes_precedence_over_archive(self):
        """When session is active, archive is not returned."""
        ctx = AppContext()
        session = _make_session_stub("aabbccddeeff")
        ctx.sessions["aabbccddeeff"] = session
        ctx._closed_histories["aabbccddeeff"] = "stale-archive-data"

        result = _resolve_history(ctx, "aabbccddeeff")
        assert result == "ACTIVE_SESSION"  # Not the archive

    def test_closed_session_returns_archive(self):
        """When session is not active, archived history is returned."""
        ctx = AppContext()
        ctx._closed_histories["aabbccddeeff"] = "archived-history-string"

        result = _resolve_history(ctx, "aabbccddeeff")
        assert result == "archived-history-string"

    def test_unknown_session_raises_valueerror(self):
        """When session is neither active nor archived, ValueError is raised."""
        ctx = AppContext()

        with pytest.raises(ValueError, match="active or recently closed"):
            _resolve_history(ctx, "aabbccddeeff")

    def test_invalid_format_raises_before_archive_check(self):
        """Invalid session_id format raises ValueError immediately."""
        ctx = AppContext()
        ctx._closed_histories["aabbccddeeff"] = "data"

        with pytest.raises(ValueError, match="12 hex characters"):
            _resolve_history(ctx, "!!!bogus!!!")

    def test_inactive_session_in_dict_falls_back_to_archive(self):
        """A session in the dict but marked inactive should fall back to archive."""
        ctx = AppContext()
        session = _make_session_stub("aabbccddeeff")
        session.is_active = False
        ctx.sessions["aabbccddeeff"] = session
        ctx._closed_histories["aabbccddeeff"] = "archived-data"

        result = _resolve_history(ctx, "aabbccddeeff")
        assert result == "archived-data"
```

- [ ] **Step 2: Run tests to verify they pass**

Run: `cd D:/Projects/Scout-mcp && uv run pytest tests/test_history_archive.py -v`
Expected: All tests PASS

- [ ] **Step 3: Modify `get_session_history` with archive fallback**

In `server.py`, replace lines 942-965 (the entire `get_session_history` function):

```python
@mcp.tool()
async def get_session_history(
    session_id: str,
    ctx: Context[ServerSession, AppContext] = None,
) -> dict:
    """Return the complete structured history of a browser session.

    Includes every action taken, every scout report (summarized), every network event
    captured, and the sequence of URLs visited. Use this data to compose botasaurus-driver scripts.

    Args:
        session_id: Session ID (active or recently closed).
    """
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
    await ctx.info(
        f"Session history: {len(history.actions)} actions, "
        f"{len(history.scouts)} scouts, {len(history.network_events)} network events"
    )
    data = history.model_dump(exclude_none=True)
    data["security_summary"] = session.security_counter.summary()
    return sanitize_response(data, secrets=session._secret_values)
```

- [ ] **Step 4: Run full non-integration test suite**

Run: `cd D:/Projects/Scout-mcp && uv run pytest -m "not integration" -v`
Expected: All tests PASS

- [ ] **Step 5: Commit**

```bash
cd D:/Projects/Scout-mcp
git add src/scout/server.py tests/test_history_archive.py
git commit -m "feat: get_session_history falls back to closed session archive"
```

---

### Task 4: Manual verification

- [ ] **Step 1: Start the MCP server and verify the change works end-to-end**

Use Scout to:
1. Launch a session
2. Navigate to a page
3. Close the session
4. Call `get_session_history` with the closed session's ID
5. Verify history is returned (should contain the navigation)

- [ ] **Step 2: Verify active session path is unaffected**

1. Launch a session
2. Navigate to a page
3. Call `get_session_history` while session is still active
4. Verify history is returned normally

- [ ] **Step 3: Verify other tools still reject closed session IDs**

After closing the session in Step 1, try calling another tool (e.g., `scout_page_tool` or `find_elements`) with the closed session's ID. Verify it raises the standard `"No active session"` error — confirming only `get_session_history` gets the archive fallback.

- [ ] **Step 4: Final commit with any fixes**

If any issues are found during manual verification, fix and commit.
