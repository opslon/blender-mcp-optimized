# Blender MCP Optimizations

Changes made to improve performance, reduce latency, and slim down dependencies.
Based on the original `blender-mcp` project.

---

## 1. Health-Check Ping Removed

**Files:** `src/blender_mcp/server.py`

**Before:** `get_blender_connection()` sent a `get_polyhaven_status` command to Blender on every single tool call as a "ping" to verify the connection was alive. This added a full socket round-trip before the actual command.

**After:** Connection validity is checked by verifying the socket object is non-null. If a previous command failed and set `sock = None`, a new connection is created on the next call.

**How to test:** Call any MCP tool (e.g. `get_scene_info`) and observe there is no extra `get_polyhaven_status` command logged on the Blender addon side before the actual command.

---

## 2. Supabase Telemetry Removed

**Files removed:** `src/blender_mcp/telemetry.py`, `src/blender_mcp/telemetry_decorator.py`
**Files modified:** `src/blender_mcp/server.py`, `pyproject.toml`

**Before:** Every tool call was wrapped with a `@telemetry_tool` decorator that recorded usage to Supabase. Each telemetry event created a new Supabase HTTP client and also sent a `get_telemetry_consent` command to Blender (yet another round-trip). `supabase` and `tomli` were required dependencies.

**After:** All telemetry code, decorators, and imports removed. `supabase` and `tomli` removed from `pyproject.toml` dependencies.

**How to test:**
- `pip install .` should no longer pull in `supabase` or `tomli`
- Tools should work without any telemetry-related log messages
- No network calls to Supabase should occur

---

## 3. Length-Prefix Socket Protocol

**Files:** `addon.py`, `src/blender_mcp/server.py`

**Before:** Messages were sent as raw JSON over the socket. The receiver accumulated chunks and attempted `json.loads()` after every `recv()` call. For a 1MB response received in 128 chunks, this meant 128 JSON parse attempts.

**After:** Messages use a 4-byte big-endian length header (`struct.pack('!I', length)`) followed by the JSON payload. The receiver reads the header first, then reads exactly that many bytes — one parse, guaranteed complete.

**Protocol format:**
```
[4 bytes: payload length (big-endian uint32)] [N bytes: JSON payload]
```

**How to test:**
- All existing MCP tools should work as before
- Large responses (screenshots, model downloads) should transfer without issues
- Both addon and server must be updated together (they are no longer compatible with the original versions independently)

---

## 4. In-Memory Screenshot Transfer

**Files:** `addon.py`, `src/blender_mcp/server.py`

**Before:** The addon saved the screenshot to a temp file on disk. The MCP server then read that same file from disk and deleted it. This required both processes to share filesystem access and added disk I/O latency.

**After:** The addon captures the screenshot to a temp file (required by Blender's API), reads it into memory, encodes it as base64, and sends it over the socket. The temp file is cleaned up immediately on the addon side. The MCP server decodes the base64 data directly — no file access needed.

**How to test:**
- Call `get_viewport_screenshot` and verify the image is returned correctly
- No `blender_screenshot_*.png` temp files should remain after the call
- Should work even if the MCP server runs on a different machine (no shared filesystem needed)

---

## 5. Handler Map Caching (Addon)

**Files:** `addon.py`

**Before:** `_execute_command_internal()` built a new `handlers` dictionary and conditionally merged integration-specific handlers on every single command.

**After:** The handler map is built once and cached. It is only rebuilt when the integration toggle state changes (PolyHaven, Hyper3D, Sketchfab, Hunyuan3D enabled/disabled).

**How to test:**
- Toggle integrations on/off in the Blender sidebar and verify the correct commands are available/unavailable
- Commands should execute as before with no regressions

---

## 6. Connection Retry with Backoff

**Files:** `src/blender_mcp/server.py`

**Before:** `BlenderConnection.connect()` attempted to connect once. If it failed, the tool call would error immediately.

**After:** `connect()` retries up to 3 times with exponential backoff (0.5s, 1s, 2s delays between attempts).

**How to test:**
- Start the MCP server before the Blender addon is running
- Start the addon within a few seconds — the server should connect on a retry
- Check logs for `Connection attempt X/3 failed` messages followed by a successful connection

---

## Compatibility Notes

- The addon (`addon.py`) and MCP server (`src/blender_mcp/server.py`) must be updated together due to the protocol change (length-prefix framing). They are not backward-compatible with the original versions.
- The `get_telemetry_consent` handler still exists in `addon.py` (Blender side) but is no longer called by the MCP server. It can be removed from the addon if desired.
