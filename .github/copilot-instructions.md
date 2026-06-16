# Coding Agent Instructions for chrome-devtools-mcp

These instructions help AI coding agents quickly become productive in this codebase. They capture the architecture, critical workflows, and project-specific patterns.

## Big Picture

**Purpose:** An MCP server that lets AI agents control and inspect a live Chrome browser via Puppeteer and Chrome DevTools Protocol.

**Key Components:**
- **Entry Point** (`src/index.ts`): Enforces Node version (`^20.19.0` LTS or newer); critical for cross-platform compatibility.
- **Server Setup** (`src/main.ts`): Initializes MCP server, registers tools from `src/tools/`, manages stdio transport.
- **Browser Control** (`src/browser.ts`): Two modes—launch new Chrome or connect to existing (remote debugging). Selected by CLI flags; single browser instance managed globally.
- **Shared Context** (`src/McpContext.ts`): Maintains browser state—pages, selected page, accessibility snapshot, network/console event collectors, element/node lookups. Shared across all tool handlers.
- **Tool Registry** (`src/tools/`): Modular tool files (pages, input, network, performance, console, etc.) each export `ToolDefinition` objects. Auto-discovered and registered in `src/tools/tools.ts`.

**Data Flow:** CLI args → browser setup → `McpContext` → tool handlers → `McpResponse` (structured output) → client.

## Developer Workflows

**Build & Run:**
- `npm run build`: TypeScript→JavaScript in `build/`, runs post-build script for license headers.
- `npm run bundle`: Creates optimized bundle via rollup (for distribution).
- `npm start`: Build then run server locally.
- `npm run start-debug`: Build with `DEBUG=mcp:*` for detailed trace logging.

**Testing:**
- `npm run test`: Full build + run all tests (Node test runner, no Jest/Vitest).
- `npm run test:only`: Run `.only` tests (fast iteration).
- `npm run test:update-snapshots`: Refresh snapshot files in `tests/**.snapshot`.
- Snapshots stored as `.snapshot` files (serialized as strings, not JSON).

**Quality & Docs:**
- `npm run format` / `npm run check-format`: ESLint + Prettier for code style.
- `npm run docs`: Regenerate tool reference doc from source (reads schema/descriptions from tool definitions).

## CLI & Configuration

**Key Command-Line Options** (see `src/cli.ts`):
- `--browserUrl` or `-u`: Connect to debuggable Chrome at HTTP endpoint (e.g., `http://127.0.0.1:9222`).
- `--wsEndpoint`: WebSocket endpoint for remote browser.
- `--autoConnect`: Auto-connect to Chrome in user data dir (Chrome 145+).
- `--headless`, `--channel`, `--viewport`, `--userDataDir`: Launch-mode configuration.
- `--logFile`: Write debug logs to file.
- `--chrome-arg=...`: Pass raw Chrome flags.
- `--no-category-*` (emulation, performance, network): Disable tool groups at runtime.

**Remote Debugging:** Start Chrome with `--remote-debugging-port=9222`, then pass `--browserUrl=http://127.0.0.1:9222`.

## Tool Definition Patterns

**Anatomy of a Tool** (e.g., `src/tools/input.ts`):
```typescript
export const click = defineTool({
  name: 'click',
  description: 'Clicks on element (must match tool reference exactly)',
  annotations: {
    category: ToolCategory.INPUT,
    readOnlyHint: false,  // true if no side effects
  },
  schema: { uid: zod.string().describe('Element UID from snapshot') },
  handler: async (request, response, context) => {
    const handle = await context.getElementByUid(request.params.uid);
    try {
      await context.waitForEventsAfterAction(async () => {
        await handle.asLocator().click();
      });
      response.appendResponseLine('Successfully clicked');
      response.includeSnapshot();  // Always include post-action snapshot
    } finally {
      void handle.dispose();
    }
  },
});
```

**Key Conventions:**
- Always wrap browser actions in `context.waitForEventsAfterAction()` to capture network, navigation, and console events deterministically.
- Call `response.includeSnapshot()` after state-changing actions so agents see results.
- Use `response.appendResponseLine()` for human-readable summaries, not raw JSON.
- Dispose element handles in `finally` blocks.
- Use `ToolCategory` enum (not string literals) for category annotation.
- Use `timeoutSchema` helper from `ToolDefinition.ts` for consistent timeout params.

**Response Shaping:**
- `setIncludePages()`: List open pages.
- `setIncludeNetworkRequests()` / `setIncludeConsoleData()`: Include filtered request/message history.
- `includeSnapshot({verbose?, filePath?})`: Include accessibility tree (optional verbose/file output).
- `attachImage()`, `attachNetworkRequest()`, `attachConsoleMessage()`: Attach binary or specific message data.

## State & Element Lookups

**Page Selection:** `context.getSelectedPage()` is the default target for all subsequent operations. Use `select_page` tool to change.

**Element UIDs:** Elements in snapshot get unique UIDs (`uid` field in `TextSnapshotNode`). Tools look them up via `context.getElementByUid(uid)` → `ElementHandle` → `asLocator()` for robust Puppeteer patterns.

**Accessibility Tree:** `context.getAXNodeByUid(uid)` → `TextSnapshotNode` for reading properties (role, name, value, children).

## Testing Patterns

**Setup:** `tests/setup.ts` configures snapshot serialization (readable `.snapshot` files, not JSON).

**Running Tests:**
- Tests compile to `build/tests/**/*.test.js`.
- Use `npm run test:only` with `it.only()` to focus on specific test.
- Snapshot files auto-generated; update with `npm run test:update-snapshots`.

**Example Test Structure:**
```typescript
import {describe, it} from 'node:test';
describe('my tool', () => {
  it('should do X', async (t) => {
    // snapshot comparison
    t.assert.match(result, 'expected text');
  });
});
```

## Design Principles

- **Token-Optimized:** Return summaries ("LCP 3.2s") not raw data dumps; large outputs go to files.
- **Deterministic Blocks:** Compose small tools (click, screenshot) instead of magic buttons.
- **Self-Healing Errors:** Include context and potential fixes in error messages.
- **Agent-Agnostic:** Use MCP standards; don't lock in to one LLM.

## Common Task Examples

**Add a new tool:**
1. Create handler in appropriate `src/tools/*.ts` file (e.g., `src/tools/network.ts` for network tools).
2. Export via `defineTool()` with all required fields.
3. Tools auto-registered by `src/tools/tools.ts` (no manual registration needed).
4. Run `npm run docs` to sync `docs/tool-reference.md`.

**Debug a failing tool:**
1. Run with `npm run start-debug` and check `DEBUG=mcp:*` logs.
2. Add `--logFile=log.txt` to write logs to file.
3. Use `npm run test:only` to isolate the test and examine snapshots.

**Modify snapshot format:**
- Edit `src/formatters/snapshotFormatter.ts` for accessibility tree presentation.
- Regenerate snapshots: `npm run test:update-snapshots`.

## External Dependencies

- **Puppeteer:** Browser automation via `Browser`, `Page`, `Locator` APIs.
- **Chrome DevTools Frontend:** Protocol parsing and trace processing (`src/trace-processing/parse.ts`).
- **@modelcontextprotocol/sdk:** MCP server framework and type definitions.

## Integration Notes

- **Tool Discovery:** Auto-registered by name from `src/tools/tools.ts`; keep names stable (docs reference them).
- **DevTools IDs:** Use `context.resolveCdpRequestId()` / `resolveCdpElementId()` to map protocol IDs to UI-visible IDs.
- **Inspector Testing:** Test with `npx @modelcontextprotocol/inspector node build/src/index.js` for interactive debugging.
