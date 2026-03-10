# Standalone repo, Tailwind refactor, SSE integration, security analysis

**Date:** 2026-03-10
**Type:** Refactor | Enhancement
**Scope:** Plugin repo structure, UI, docs

## Summary

Extracted the chat plugin from the Paperclip monorepo into a standalone repo (`webprismdevin/paperclip-plugin-chat`), refactored the entire UI from inline styles to Tailwind CSS classes, confirmed SSE streaming is fully wired end-to-end, and added a plugin security model analysis to the comparison doc.

## Changes

### Standalone repo extraction

Moved the plugin out of `packages/plugins/plugin-chat/` in the Paperclip monorepo into its own repo. Updated `package.json` to replace `"@paperclipai/plugin-sdk": "workspace:*"` with `"@paperclipai/plugin-sdk": "^1.0.0"` (assumes SDK gets published to npm as part of core). Removed `"private": true`. Added standalone `tsconfig.json` (no longer extends monorepo root). Fixed all doc links from `../../docs/specs/` to `docs/`. Moved spec docs into `docs/` at repo root.

### Tailwind CSS refactor (`src/ui/index.tsx`)

Replaced ~400 lines of inline `style={{}}` with Tailwind utility classes, following patterns from the official file browser example plugin in PR #432. Net reduction of 374 lines (181 added, 555 removed).

Key conversions:
- `style={{ color: "var(--foreground, #1e293b)" }}` → `className="text-foreground"`
- `style={{ background: "var(--card, #fff)", border: "1px solid var(--border)" }}` → `className="bg-card border border-border"`
- Layout styles → `flex`, `gap-3`, `py-4`, `items-center`, etc.

Only 2 inline styles remain: spinner `@keyframes` animation and dynamic sidebar width toggle — both require runtime values.

### Utility hooks from file browser example

Added three hooks adopted from `plugin-file-browser-example`:

- `useIsDarkMode()` — watches `document.documentElement.classList.contains("dark")` via MutationObserver
- `useIsMobile(breakpointPx)` — responsive breakpoint detection via `matchMedia`
- `useAvailableHeight(ref, options)` — dynamic fill-height calculation with ResizeObserver, replacing hardcoded `calc(100vh - 8rem)`

### SSE streaming confirmed resolved

Verified that PR #432 (`feature/plugins-clean`) ships a complete `usePluginStream()` implementation:
- SDK hook: `packages/plugins/sdk/src/ui/hooks.ts`
- Bridge: `EventSource`-based SSE in `ui/src/plugins/bridge.ts`
- Registration: `usePluginStream` in `bridge-init.ts`
- Server: `plugin-stream-bus.ts` + SSE route at `/api/plugins/:pluginId/bridge/stream/:channel`

Our plugin UI already integrates this — subscribes to `chat:${threadId}` and processes `text`, `thinking`, `tool_use`, `error`, `done` events in real time.

Updated comparison doc to mark streaming as resolved and remove polling references.

### Plugin security model analysis (`docs/plugin-vs-core-chat.md`)

Added new "Plugin Security Model" section documenting 9 enforcement layers:
- Process isolation (child process per plugin)
- Capability-gated RPC (44 operation-to-capability mappings)
- State isolation (`pluginId` in every query)
- Session ownership (`taskKey` pattern matching)
- Company boundaries (`inCompany()` checks)
- Module sandboxing (allow-listed imports)
- Network protection (SSRF/DNS rebinding prevention)
- Environment isolation (no secrets in env)
- Resource limits (5-min RPC timeout, crash backoff)

Added table mapping each chat-required capability to the sandbox boundary it breaks:
- `ctx.chat.stream()` → bypasses agent mediation
- `ctx.agents.create()` → creates autonomous entities with host permissions
- `ctx.exec()` → escapes process sandbox entirely

Updated side-by-side comparison table with security boundary, LLM cost control, and data access scope rows. Updated recommendation to include the security argument.

## Files Created

### `changelog/` (this directory)
- First changelog entry for the standalone repo

## Files Modified

### `src/ui/index.tsx`
- All inline styles → Tailwind classes
- Added `useIsDarkMode`, `useIsMobile`, `useAvailableHeight` hooks
- Removed hover rules from `CHAT_STYLES` (now Tailwind `hover:` classes)
- Container height: `calc(100vh - 8rem)` → `useAvailableHeight(containerRef)`

### `docs/plugin-vs-core-chat.md`
- Added "Plugin Security Model" section with enforcement layer table
- Expanded items 9-12 with sandbox boundary breakdown table
- Added security rows to side-by-side comparison
- Updated recommendation with security rationale
- Marked SSE streaming as resolved (PR #432)
- Updated workarounds table and "What Would Need to Change" section

### `package.json`
- Removed `"private": true`
- Changed SDK dep from `workspace:*` to `^1.0.0`
- Added `"files"`, `"license"`, `"repository"` fields
- Removed `prebuild` script (monorepo-specific)

### `tsconfig.json`
- Standalone config (no longer `extends` monorepo root)
- Added `"lib": ["ES2023", "DOM"]` and `"jsx": "react-jsx"` inline

### `README.md`
- Fixed doc links from `../../docs/specs/` to `docs/`
- Updated build instructions (no `cd packages/...` prefix)

## Testing

1. Build from the monorepo: `cd packages/plugins/plugin-chat && npm run build`
2. Deploy to Docker: `docker cp dist/ui/index.js paperclip-plugin-chat-server-1:/app/packages/plugins/plugin-chat/dist/ui/index.js`
3. Hard refresh browser at the plugin URL
4. Verify: Tailwind classes render correctly (text colors, borders, backgrounds match host theme), sidebar toggle works, dynamic height fills available space, dark mode detected correctly

## Notes

- The monorepo copy at `packages/plugins/plugin-chat/src/ui/index.tsx` was also updated so builds work against the running Docker instance
- The standalone repo can't `npm install` or `npm run build` yet — `@paperclipai/plugin-sdk` isn't published to npm. Building requires the monorepo until the SDK ships.
