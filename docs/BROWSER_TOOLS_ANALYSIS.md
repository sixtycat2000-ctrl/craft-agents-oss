# Browser Tools — Deep Analysis

## 1. Architecture Overview

Browser Tools is a **3-layer system** that bridges the AI agent to a Chromium browser window:

```
┌──────────────────────────────────────────────────────────────────┐
│  Layer 1: Tool Interface (shared package — portable)             │
│                                                                  │
│  browser-tools.ts         → BrowserPaneFns interface + factory   │
│  browser-tool-runtime.ts  → CLI parser + command router          │
│  session-scoped-tools.ts  → registers browser_tool into MCP      │
│  session-scoped-tool-callback-registry.ts → Map<id, callbacks>   │
└──────────────────────┬───────────────────────────────────────────┘
                       │ getBrowserPaneFns() (lazy resolve)
┌──────────────────────▼───────────────────────────────────────────┐
│  Layer 2: Wiring Layer (server-core — headless agnostic)         │
│                                                                  │
│  SessionManager.ts                                               │
│    mergeSessionScopedToolCallbacks(sessionId, {                  │
│      browserPaneFns: {                                           │
│        navigate: (url) => bpm.navigate(instanceId, url),         │
│        click: (ref) => bpm.clickElement(instanceId, ref),        │
│        ...30+ method bindings                                     │
│      }                                                           │
│    })                                                            │
└──────────────────────┬───────────────────────────────────────────┘
                       │ delegates to
┌──────────────────────▼───────────────────────────────────────────┐
│  Layer 3: Electron Backend (apps/electron — desktop only)        │
│                                                                  │
│  BrowserPaneManager.ts                                           │
│    implements IBrowserPaneManager                                │
│    owns instances: Map<id, BrowserInstance>                      │
│    binds sessions → browser windows (1:1, lazy, reusable)        │
│    calls CDP (Chrome DevTools Protocol) under the hood           │
└──────────────────────────────────────────────────────────────────┘
```

**Key insight**: Layers 1 and 2 have **zero Electron imports**. The entire browser tool system in `packages/shared/` is platform-agnostic. Only Layer 3 (the `BrowserPaneManager`) is Electron-specific.

---

## 2. How `browser_tool` Gets Registered into the Agent Runtime

### Registration Chain (full path)

```
Step 1: Electron Main Process
  BrowserPaneManager created, bound to SessionManager

Step 2: SessionManager creates agent session
  → mergeSessionScopedToolCallbacks(sessionId, {
      browserPaneFns: { navigate: (url) => bpm.navigate(id, url), ... }
    })
  → stored in Map<sessionId, SessionScopedToolCallbacks>

Step 3: Agent backend calls getSessionScopedTools(sessionId, workspaceRoot)
  → creates tools array from SESSION_TOOL_REGISTRY
  → if getBrowserToolEnabled():
      tools.push(...createBrowserTools({
        sessionId,
        getBrowserPaneFns: () => {
          return getSessionScopedToolCallbacks(sessionId)?.browserPaneFns
        }
      }))

Step 4: createBrowserTools() returns [tool('browser_tool', ...)]
  → single SDK tool object with schema { command: string | string[] }
  → handler calls executeBrowserToolCommand({ command, fns, sessionId })

Step 5: Tools wrapped in MCP server
  → createSdkMcpServer({ name: 'session', tools })
  → returned as MCP server config

Step 6a: ClaudeAgent path
  → this.sdk = new Agent({ mcpServers: { session: getSessionScopedTools(...) } })
  → SDK auto-discovers browser_tool from MCP server

Step 6b: PiAgent path
  → getSessionToolProxyDefs() returns tool metadata
  → sent to Pi subprocess via { type: 'register_tools' }
  → PiAgent.executeSessionTool() handles browser_tool directly:
      const fns = getSessionScopedToolCallbacks(sessionId)?.browserPaneFns
      executeBrowserToolCommand({ command, fns, sessionId })
```

### Execution Flow (when LLM calls `browser_tool`)

```
LLM output: browser_tool({ command: "navigate https://example.com" })
  │
  ▼
Claude SDK routes to 'session' MCP server → finds browser_tool
  │
  ▼
browser-tools.ts handler:
  const fns = options.getBrowserPaneFns()  ← lazy resolve from registry
  │
  ▼
browser-tool-runtime.ts:
  tokenizeCommand("navigate https://example.com") → ["navigate", "https://example.com"]
  cmd === 'navigate' branch
  │
  ▼
fns.navigate("https://example.com")  ← BrowserPaneFns method
  │
  ▼
SessionManager-wired closure:
  bpm.navigate(instanceId, "https://example.com")
  │
  ▼
BrowserPaneManager → CDP → BrowserWindow
  │
  ▼
Result flows back: { url, title } → formatted output → tool_result event → UI
```

---

## 3. Can Browser Tools Be a Reusable Library?

### Short Answer: **Yes, it already is ~80% of the way there.**

### What's Already Portable

| Component | Location | Electron Dependency |
|-----------|----------|-------------------|
| `BrowserPaneFns` interface | `packages/shared/src/agent/browser-tools.ts` | **None** |
| `createBrowserTools()` factory | `packages/shared/src/agent/browser-tools.ts` | **None** |
| `executeBrowserToolCommand()` runtime | `packages/shared/src/agent/browser-tool-runtime.ts` | **None** |
| CLI parser (`tokenizeCommand`, `splitBatchCommands`) | `packages/shared/src/agent/browser-tool-runtime.ts` | **None** |
| All 30+ command handlers | `packages/shared/src/agent/browser-tool-runtime.ts` | **None** |
| Tool description & help text | `packages/shared/src/agent/browser-tools.ts` | **None** |
| Zod schema | `packages/shared/src/agent/browser-tools.ts` | **None** |
| Callback registry | `packages/shared/src/agent/session-scoped-tool-callback-registry.ts` | **None** |
| MCP server registration | `packages/shared/src/agent/session-scoped-tools.ts` | **None** |

The **only** Electron-dependent code is:
- `BrowserPaneManager` class (`apps/electron/src/main/browser-pane-manager.ts`) — creates `BrowserWindow`, calls CDP
- SessionManager wiring (`packages/server-core/src/sessions/SessionManager.ts`) — binds `bpm.*` methods to `BrowserPaneFns`

### What a Standalone Library Would Look Like

```
@craft-agent/browser-tools
├── src/
│   ├── types.ts              ← BrowserPaneFns, all arg/result interfaces
│   ├── factory.ts            ← createBrowserTools() → SDK tool[]
│   ├── runtime.ts            ← executeBrowserToolCommand(), all command handlers
│   ├── parser.ts             ← tokenizeCommand(), splitBatchCommands(), decodeEscapes()
│   ├── format.ts             ← formatNodeLine(), formatBytes(), etc.
│   └── index.ts              ← public API
└── package.json
```

**Consumers would only need to implement `BrowserPaneFns`** — a 30-method interface:

```typescript
// Consumer implements this — could be Playwright, Puppeteer, or raw CDP
const browserPaneFns: BrowserPaneFns = {
  navigate: (url) => myBrowser.goto(url),
  snapshot: () => myBrowser.accessibilityTree(),
  click: (ref) => myBrowser.click(ref),
  // ... 27 more methods
};

// Plug into any agent runtime
const tools = createBrowserTools({
  sessionId: 'my-session',
  getBrowserPaneFns: () => browserPaneFns,
});
```

### What Would Need to Change

1. **Extract `BrowserPaneFns` and all types** into a standalone types file
2. **Remove `import { tool } from '@anthropic-ai/claude-agent-sdk'`** — make the factory accept a tool creator function, or export a generic handler and let consumers wrap it
3. **Remove `import { z } from 'zod'`** for schema — or keep it as a peer dependency
4. **Bundle `browser-tool-runtime.ts`** — it currently imports types from `browser-tools.ts` (circular-avoidable)
5. **Remove `process.env.CRAFT_BROWSER_OPEN_SETTLE_TIMEOUT_MS`** — make timeouts configurable via options
6. **Remove `process.platform`** reference in `paste` command — pass via options

### Estimated Effort

| Task | Effort |
|------|--------|
| Extract types + runtime into standalone package | 1-2 days |
| Remove SDK/zod hard dependency (make injectable) | 1 day |
| Add Playwright adapter implementing `BrowserPaneFns` | 2-3 days |
| Add Puppeteer adapter implementing `BrowserPaneFns` | 2-3 days |
| Tests + docs | 1-2 days |
| **Total** | **~1-1.5 weeks** |

---

## 4. Can Browser Tools Be a Submodel (Sub-Agent)?

### Current Design: Single Tool, Not a Sub-Agent

`browser_tool` is registered as **one MCP tool** in the session-scoped MCP server. The LLM sees it as a single callable tool with a `command` parameter. All routing happens in `executeBrowserToolCommand()`.

### Why a Sub-Agent/Model Approach Could Work

The browser tool has a natural "observe → think → act" loop:

```
snapshot → find element → click → wait → snapshot → verify
```

Currently the primary LLM orchestrates this loop via multiple `browser_tool` calls. Each call:
1. Consumes context window space
2. Requires the LLM to reason about state between calls
3. Adds latency (round-trip through the primary model)

A sub-agent could run this loop autonomously:

```
User: "Fill out the form at example.com/form"

Primary Agent → delegates to BrowserSubAgent:
  BrowserSubAgent:
    1. navigate https://example.com/form
    2. snapshot → find form fields
    3. fill @e1 "John"
    4. fill @e2 "john@example.com"
    5. click @submit
    6. snapshot → verify success
    7. return result to primary agent
```

### Implementation Path: Using the Existing LLM Tool

Craft Agents already has `call_llm` — a secondary model invocation tool. A browser sub-agent could be implemented as:

```typescript
// Option A: Dedicated browser sub-agent tool
function createBrowserSubAgent(options: {
  browserPaneFns: BrowserPaneFns;
  sessionId: string;
}) {
  return tool('browser_subagent', 'Execute a multi-step browser task autonomously', {
    task: z.string().describe('What to accomplish in the browser'),
    url: z.string().optional().describe('Starting URL'),
    maxSteps: z.number().default(15),
  }, async (args) => {
    // Loop: executeBrowserToolCommand → feed to fast model → repeat
    // until task complete or maxSteps exhausted
  });
}
```

### Tradeoffs

| Aspect | Current (single tool) | Sub-agent |
|--------|----------------------|-----------|
| **Token cost** | High (primary model reasons each step) | Low (fast model handles loop) |
| **Latency** | Higher (round-trip per step) | Lower (sub-agent loops locally) |
| **Reliability** | Primary model maintains full context | Sub-agent may lose nuance |
| **Debuggability** | Every step visible in conversation | Black box unless logged |
| **Flexibility** | User can interject mid-task | Fire-and-forget |
| **Complexity** | Simple (one tool, one handler) | Complex (loop, state, termination) |

### Recommended Approach

**Hybrid**: Keep the current `browser_tool` for simple operations (navigate, click, fill). Add an optional `browser_task` tool for multi-step autonomous workflows that uses a faster/cheaper model internally. This matches the pattern already established by `call_llm`.

---

## 5. Design Patterns Used

### 5.1 Interface Segregation

`BrowserPaneFns` is the **sole contract** between the tool layer and the browser backend. The runtime knows nothing about Electron, CDP, or BrowserWindow. This is classic Dependency Inversion — the runtime depends on an abstraction, not a concrete implementation.

```
browser-tool-runtime.ts → depends on BrowserPaneFns (interface)
                              ↑ implemented by
                    SessionManager closure → BrowserPaneManager → CDP
```

### 5.2 Lazy Resolution via Getter

The `getBrowserPaneFns` function is a **lazy getter**, not a direct reference:

```typescript
getBrowserPaneFns: () => {
  const callbacks = getSessionScopedToolCallbacks(sessionId);
  return callbacks?.browserPaneFns;
}
```

This is critical because:
- Browser callbacks are registered **after** tools are created (temporal decoupling)
- Callbacks can change during a session's lifetime
- The tool cache stores tools but always resolves the current callbacks at execution time

### 5.3 CLI-in-a-Tool Pattern

Instead of 30+ separate MCP tools (`browser_navigate`, `browser_click`, etc.), everything is unified into one `browser_tool` with a CLI-like command syntax. This:

- Reduces tool discovery cost for the LLM (1 tool vs 30)
- Enables batch commands with semicolons
- Keeps the tool schema trivial (`{ command: string | string[] }`)
- Routes internally via a command string parser

### 5.4 Session-Bound Instance Management

Browser instances are bound to sessions via `BrowserPaneManager.createForSession()`:

```
First call:  createForSession("sess-1") → creates BrowserWindow, binds to sess-1
Second call: createForSession("sess-1") → returns existing bound window (reuse)
Third call:  createForSession("sess-2") → unbinds from sess-1, binds to sess-2 (window persists)
```

This means:
- **1:1 session-to-window mapping** at any given time
- **Window reuse** across sessions (no create/destroy churn)
- **Lazy creation** (window created on first browser_tool call, not on session start)

---

## 6. Command Parser Design

The runtime includes a sophisticated CLI parser that handles:

- **Tokenization** with quote handling: `"hello world"` and `'hello world'`
- **Escape sequences**: `\n`, `\t`, `\\`, `\"`
- **Batch splitting** on semicolons (respecting quotes): `fill @e1 val1; fill @e2 val2`
- **Navigation stop** — batch stops after `navigate`, `click`, `back`, `forward`
- **Array mode** — `["evaluate", "var x = 1; x + 2"]` bypasses all parsing

This parser is ~200 lines of hand-written code (no external dependency) and is itself reusable.

---

## 7. Security Challenge Detection

The runtime includes automatic detection of CAPTCHA/security challenges:

```typescript
const challenge = await fns.detectChallenge();
if (challenge.detected) {
  await fns.releaseControl();  // Give control back to user
  return { output: 'Security verification detected...', appendReleaseHint: false };
}
```

This fires after:
- `navigate` — catches Cloudflare, reCAPTCHA on page load
- `snapshot` — catches challenges on near-empty pages (0-2 actionable elements)
- `click` — catches challenges triggered by clicking

When detected, the agent **releases control** to the user (shows the browser), and instructs the user to complete verification manually before continuing.

---

## 8. Feature Gate

Browser tools are gated behind a config setting:

```typescript
// In session-scoped-tools.ts
if (getBrowserToolEnabled()) {
  tools.push(...createBrowserTools({ ... }));
}
```

This lets users with external browser tools (Playwright MCP, Puppeteer MCP) disable the built-in one to avoid conflicts. The setting is checked at tool creation time, not runtime, so the cache invalidation function `invalidateAllSessionToolsCaches()` is called when the setting changes.

---

## 9. Summary: Reusability Scorecard

| Aspect | Score | Notes |
|--------|-------|-------|
| **Interface portability** | 9/10 | `BrowserPaneFns` is clean, no Electron leaks |
| **Runtime portability** | 9/10 | `executeBrowserToolCommand` is pure TypeScript |
| **Parser portability** | 10/10 | Zero dependencies |
| **SDK coupling** | 6/10 | Depends on `tool()` from `@anthropic-ai/claude-agent-sdk` and `z` from `zod` |
| **Sub-agent potential** | 8/10 | Natural fit — clear observe→act loop |
| **Adapter ecosystem** | 7/10 | `BrowserPaneFns` is well-defined; Playwright/Puppeteer adapters are straightforward |

### Bottom Line

Browser Tools is **already architected as a library** — it just lives inside the monorepo. The separation between interface (`BrowserPaneFns`), runtime (command parser + router), and backend (`BrowserPaneManager`) is clean. Extracting it into a standalone package requires mainly removing the Claude SDK `tool()` wrapper and making it adapter-agnostic. The sub-agent pattern is a natural next step that leverages the existing `call_llm` infrastructure.
