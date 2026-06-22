# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Workflow

See `KB-Development-Workflow.md` in the Knowledge Base for the full workflow. Summary:

1. Bugs and features are tracked as **GitHub Issues**
2. Claude works on a **feature branch** (worktrees for isolation in local sessions)
3. Claude pushes the branch and opens a **Pull Request**
4. Rick reviews and merges the PR
5. Adding the `claude` label to an issue triggers Claude via GitHub Actions

## Commands

### Development Setup
```bash
# Install dependencies (uses uv package manager)
uv sync

# Run the MCP server (stdio transport, default)
uv run things-mcp

# Run with HTTP transport
THINGS_MCP_TRANSPORT=http uv run things-mcp
```

### Testing

The project includes a comprehensive unit test suite covering URL scheme construction and data formatting functions.

```bash
# Install test dependencies
uv sync --extra test

# Run all tests
uv run pytest

# Run tests with verbose output
uv run pytest -v

# Run specific test file
uv run pytest tests/test_url_scheme.py

# Run tests matching a pattern
uv run pytest -k "test_format"
```

Test coverage includes:
- All URL construction functions (add, update, show, search)
- Authentication token handling
- Data formatting for todos, projects, areas, and tags
- Error handling and edge cases
- Mock external dependencies (Things.py, shell commands)

## Architecture Overview

This is a Model Context Protocol (MCP) server that bridges Claude Desktop with the Things 3 task management app on macOS. The architecture consists of:

1. **src/things_mcp/server.py** - Main MCP server implementation using FastMCP (3.x)
   - Defines all MCP tools for interacting with Things (26 tools)
   - List views (inbox, today, upcoming, etc.)
   - CRUD operations for todos/projects/areas (Areas have create/read/update — no delete by design; see below)
   - Search and tag operations
   - Things URL scheme integration
   - Implements Someday project filtering to match Things UI behavior
   - **Pagination + structured responses**: the 16 list/search read tools accept optional `limit`/`offset` and return a FastMCP `ToolResult` carrying both a human-readable text channel and a `structured_content` envelope `{items, count, total, offset, limit}`. Two helpers drive this: `_paginate_format` (text) and `_paginate_result` (wraps text + JSON-safe structured data); `_error_result` wraps early-return/validation errors. `get_tag_usage` intentionally stays a plain string.

2. **src/things_mcp/url_scheme.py** - Things URL scheme + AppleScript implementation
   - Constructs Things URLs for various operations
   - Uses shell script with `open -g` to execute URLs without bringing Things to foreground
   - Handles authentication tokens for update operations
   - `add_area`/`update_area` use AppleScript (`osascript`) since the Things URL scheme has no area commands; all user-supplied strings are escaped (`\` then `"`) before being embedded in the AppleScript string literal to prevent injection, and `osascript` is invoked with an argv list (no shell)

3. **src/things_mcp/formatters.py** - Data formatting utilities
   - Converts Things database objects to human-readable text
   - Handles nested data (projects within areas, checklist items, etc.)

4. **tests/** - Unit test suite (174 tests)
   - **conftest.py** - Pytest fixtures and mock data
   - **_helpers.py** - `tool_text()` reads the text channel from a `ToolResult` (or a plain-string result) so text assertions work across both return shapes
   - **test_url_scheme.py** - Tests for URL construction
     - Note: `test_construct_url_encodes_slash_in_values` doesn't mock `things.token` (unlike sibling update tests), so it reads the live Things DB and fails with `unable to open database file` in CI/sandboxes lacking DB access — not a real regression.
   - **test_formatters.py** - Tests for data formatting
   - **test_things_server.py** - Tests for server tools (incl. pagination + structured-content shape)
   - **test_things_server_headings.py** - Tests for heading functionality
   - **test_someday_filtering.py** - Tests for Someday project filtering
   - **test_mcp_server_filtering.py** - Integration tests for MCP server filtering

## Key Implementation Details

- Uses things.py library for reading Things SQLite database
- Things DB path: things.py auto-discovers `~/Library/Group Containers/JLMPQHK86H.com.culturedcode.ThingsMac/ThingsData-*/Things Database.thingsdatabase/main.sqlite` (newer Things 3 nests the DB under a per-install `ThingsData-*` dir); override with the `THINGSDB` env var. On macOS the server process needs TCC "data from other apps" access to that Group Container or DB reads fail (`unable to open database file`) — and can *hang* the single FastMCP event loop if the grant is mid-prompt.

### ⚠️ Known failure mode: `brew upgrade uv` silently breaks Things DB reads (diagnosed 2026-06-22)

**Symptom:** Every Things read returns `unable to open database file` (`sqlite3.OperationalError` at `things/database.py:500`). The server still binds `:8100` and answers MCP protocol requests, so `things-mcp-watchdog` sees it as "up" — but Mandy (OpenClaw) and Claude Code both get the error on any actual read. Earlier in the outage the watchdog may restart-loop (`curl exit 7` in `~/scripts/logs/things-mcp-watchdog.log`) because the first DB read hangs startup.

**Root cause:** The launchd chain is `launchd → /bin/bash → ~/scripts/wrappers/uv.sh → exec uv → python`. The `exec` makes TCC attribute the Group-Container read to the **`uv` binary** (responsible process), not to FDA-holding `/bin/bash`. Homebrew binaries are ad-hoc signed, so each `brew upgrade` gives `uv` a new Cellar path/identity and **drops its "data from other apps" (`kTCCServiceSystemPolicyAppData`) grant**. A headless launchd job cannot surface/answer the prompt to restore it. (Triggered 2026-06-12 by uv `0.11.19 → 0.11.21`.)

**What does NOT fix it (verified):**
- Full Disk Access on `uv` (or on `/bin/bash`) — FDA does *not* cover the AppData / Group-Container class.
- `tccutil reset SystemPolicyAppData` + service restart — the headless job still won't re-prompt (instant silent deny).
- Granting the wrong binary — the open() is done by the venv `python3`, but TCC evaluates the *responsible* process (`uv`).

**Fix:**
1. **Stop the churn (done):** `uv` is now `brew pin`ned and on `~/scripts/brew-update-hold.txt`, so upgrades won't silently wipe the grant again. Re-grant FDA/AppData deliberately after any intentional uv upgrade.
2. **Restore the grant:** reboot the mini (clears wedged TCC state — same playbook as the op-daemon hang), then click **Allow** on the *"uv" wants to access data from other apps* prompt on the first read. With uv pinned, that grant persists.

**Robust fallback if the prompt never fires even on a clean boot:** switch reads from direct SQLite to **AppleScript/Automation** (the Automation TCC class persists and *can* be granted to a background app). Precedent: the sibling `things-export-system` repo already exports Things via `osascript` under launchd successfully.

**Diagnosis tools:** `~/scripts/capture-tcc-prompts.sh` (captures tccd prompts — look for `kTCCServiceSystemPolicyAppData` with `Sub:{…/uv}`), things-mcp `logs/stderr.log` (the `OperationalError` traceback), `~/scripts/logs/things-mcp-watchdog.log`.
- Write operations use the Things URL scheme API, except Area create/update which use AppleScript (`osascript`)
- FastMCP (3.x) provides the MCP protocol implementation
- Read tools return a `ToolResult` (human-readable text + `structured_content`); write/report tools and error paths return plain strings
- Error handling for invalid UUIDs and missing parameters; pagination args validated by `_validate_pagination`
- Supports filtering and including nested items via parameters
- **Someday Project Filtering**: Tasks from Someday projects are filtered out of Today, Upcoming, and Anytime views to match the Things UI behavior and reduce clutter
- Unit tests mock all external dependencies (Things.py, shell commands)
- Pytest configuration in pyproject.toml with async support
- Supports both stdio (default) and HTTP transport modes
- HTTP transport always uses Streamable HTTP (FastMCP 2.14+) at `/mcp` endpoint — SSE transport is not supported
- For Docker/remote clients (e.g., OpenClaw), bind to `0.0.0.0` via `THINGS_MCP_HOST=0.0.0.0` and set client transport to `streamable-http`

## Things URL Scheme Authentication

For update operations, the server automatically fetches and includes the auth-token from things.py. This allows updating existing items without user intervention.