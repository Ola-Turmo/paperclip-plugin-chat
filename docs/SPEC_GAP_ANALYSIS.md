# Chat Plugin: Spec Gap Analysis & Recommendations

## Summary

The Paperclip Plugin Spec (v1) is designed for integrations — connectors, dashboards, automation. It assumes request-response communication, key-value state, and sandboxed execution. A real-time chat UI with LLM streaming, conversation persistence, and subprocess management breaks these assumptions at every layer.

This document explains the gaps and proposes spec changes that would make deeply integrated UI plugins like chat viable.

---

## The Three Fundamental Gaps

### 1. No Streaming Support

**What chat needs:** Token-by-token SSE streaming from a long-running Claude CLI subprocess. A single chat response can take 30-120 seconds with continuous output.

**What the spec provides:** `getData()` and `performAction()` — both are request-response JSON-RPC calls. The UI bridge has no concept of streaming, subscriptions, or server-pushed updates.

**What breaks:** The entire chat experience. Polling `getData()` every 100ms would be wasteful, laggy, and produce a terrible UX. Users expect to see tokens appear in real-time, identical to ChatGPT/Claude.ai.

**What the spec would need:**
- A `streamAction(key, params)` RPC method that returns a stream handle
- A corresponding `usePluginStream(key, params)` UI hook that yields chunks
- Worker-to-host streaming protocol — either chunked JSON-RPC notifications or a side-channel (WebSocket/SSE URL the worker can create)
- Backpressure and cancellation semantics (user hits stop → worker kills subprocess)

### 2. No Relational Data

**What chat needs:** Two relational tables with foreign keys, indexes, and ordered queries:
- `threads` — with company scoping, timestamps, session tracking
- `messages` — ordered by `created_at`, with role constraints, JSONB metadata, cascading deletes

**What the spec provides:** `ctx.state` (key-value by scope) and `ctx.entities` (typed records with `data_json`). No foreign keys, no ordered queries, no joins, no cascading deletes.

**What breaks:** Conversation history. Storing messages as serialized JSON blobs in `ctx.state` loses ordering guarantees, makes pagination impossible, and turns "delete thread with all messages" into a manual multi-step operation. `ctx.entities` is closer but still lacks relational integrity.

**What the spec would need:**
- Allow plugins to declare relational schemas (similar to current `db/migrations/` pattern but formalized)
- Plugin-scoped tables with automatic namespacing (e.g., `plugin_<id>_threads`)
- Host runs migrations on install/upgrade, drops on uninstall (with grace period)
- Worker accesses via `ctx.db.query()` / `ctx.db.execute()` — scoped to plugin's own tables only
- OR: Expand `ctx.entities` to support ordered queries, filtering, pagination, and cascading relationships

### 3. No Custom Routes or Subprocess Management

**What chat needs:**
- Custom REST endpoints (`POST /threads/:id/chat`, `POST /threads/:id/stop`)
- Long-running subprocess spawning (Claude CLI, 30-120s lifetime)
- Process lifecycle management (track active processes, kill on stop/disconnect)

**What the spec provides:** All communication goes through JSON-RPC handlers (`getData`, `performAction`). No way to register custom HTTP routes. No explicit subprocess management API.

**What breaks:** The chat endpoint is an SSE stream that spawns a Claude CLI process, pipes stdin/stdout, and streams parsed JSON events to the client. This cannot be modeled as a `performAction()` call — it's a long-lived connection with bidirectional concerns (client disconnect → kill process).

**What the spec would need:**
- `ctx.routes.register(method, path, handler)` — plugin-scoped route registration under `/api/plugins/:pluginId/`
- OR: First-class subprocess API via `ctx.process.spawn()` with lifecycle hooks
- Process registry so host can track and kill plugin subprocesses on shutdown
- Connection-aware handlers that know when the client disconnects

---

## Secondary Gaps

### Environment Variable Injection
Chat needs `PAPERCLIP_API_URL`, `PAPERCLIP_COMPANY_ID`, and a JWT for the spawned Claude process to call back into Paperclip. The spec has `ctx.secrets` and `ctx.config` but no way to generate scoped JWTs or inject env vars into subprocesses.

**Spec would need:** `ctx.auth.createScopedToken(agentId, companyId, capabilities[])` — lets plugins create short-lived tokens for subprocess authentication.

### File System Access
Chat loads skill files from `skills/` directory and a system prompt from `system-prompt.md`. The spec's out-of-process worker has no guaranteed filesystem layout.

**Spec would need:** Plugin assets directory — `ctx.assets.readFile(relativePath)` that resolves against the plugin's installed directory.

### Session Continuity
Chat uses Claude CLI's `--resume` flag with a session ID stored in the thread. This is subprocess-specific state that doesn't map to any spec concept.

**No spec change needed** — this can work through `ctx.state` if the relational data gap is solved.

---

## What CAN Be Aligned Today

Even without spec changes, we can align the non-controversial parts:

| Area | Change |
|------|--------|
| Manifest | Match `PaperclipPluginManifestV1` shape exactly |
| Package.json | Use `paperclipPlugin.worker` key |
| Capabilities | Declare all required capabilities |
| UI slots | Declare page and sidebar slots per spec |
| Plugin ID | Use npm-style `@scope/name` format |

These are cosmetic but establish forward compatibility.

---

## Recommendation

The chat plugin should be classified as a **platform module candidate**, not a standard plugin. The spec explicitly distinguishes these:

> "Platform modules are trusted, in-process, host-integrated, low-level"

Chat needs in-process integration for streaming, DB access, and subprocess management. Trying to force it into the out-of-process sandbox would degrade the UX to the point of being unusable.

**Proposed path:**
1. Ship the current implementation as a "pre-spec" plugin that works with today's in-process loader
2. Align the manifest and package structure with the spec (free compatibility)
3. Propose spec additions (streaming, relational data, scoped routes) to the Paperclip team
4. Once the spec supports these patterns, migrate to full compliance

The alternative — rewriting to use polling + key-value state + no custom routes — would produce a chat experience so degraded that no one would use it.
