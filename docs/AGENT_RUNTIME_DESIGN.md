# Craft Agents OSS — Dual Agent Runtime Design

> How the platform supports both Pi-Code and Claude Code agent runtimes through a provider-agnostic abstraction layer.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [The Problem: Two SDKs, One Platform](#the-problem-two-sdks-one-platform)
3. [Backend Abstraction Layer](#backend-abstraction-layer)
4. [Backend Factory & Driver Registry](#backend-factory--driver-registry)
5. [Claude Agent Runtime](#claude-agent-runtime)
6. [Pi Agent Runtime](#pi-agent-runtime)
7. [Event Adapter Pattern](#event-adapter-pattern)
8. [Tool System & Parity](#tool-system--parity)
9. [Session Lifecycle & Connection Resolution](#session-lifecycle--connection-resolution)
10. [Transport & Streaming](#transport--streaming)
11. [How to Add a New Provider](#how-to-add-a-new-provider)
12. [Design Trade-offs](#design-trade-offs)

---

## Architecture Overview

```
┌──────────────────────────────────────────────────────────────────┐
│                     UI Layer (CLI / IDE / WebUI)                 │
│                  Sees only AgentEvent[] — provider-agnostic      │
└───────────────────────────┬──────────────────────────────────────┘
                            │
┌───────────────────────────▼──────────────────────────────────────┐
│                    Session Manager                                │
│         Session CRUD, connection resolution, event dispatch      │
└───────────────────────────┬──────────────────────────────────────┘
                            │
┌───────────────────────────▼──────────────────────────────────────┐
│                AgentBackend Interface (Facade)                    │
│    chat() · abort() · setModel() · setSourceServers() · ...     │
└────────────┬──────────────────────────────────┬──────────────────┘
             │                                  │
   ┌─────────▼──────────┐            ┌──────────▼──────────┐
   │   ClaudeAgent      │            │     PiAgent          │
   │   (In-process)     │            │   (Subprocess)       │
   │                    │            │                      │
   │ Claude Agent SDK   │            │ Pi Agent Server      │
   │ @anthropic-ai/     │            │ @mariozechner/       │
   │ claude-agent-sdk   │            │ pi-coding-agent      │
   └─────────┬──────────┘            └──────────┬───────────┘
             │                                  │
   ┌─────────▼──────────┐            ┌──────────▼───────────┐
   │ ClaudeEventAdapter │            │  PiEventAdapter       │
   │ SDKMessage →       │            │  Pi Core Event →     │
   │ AgentEvent         │            │  AgentEvent           │
   └────────────────────┘            └──────────────────────┘
```

---

## The Problem: Two SDKs, One Platform

Craft Agents needs to support two fundamentally different agent SDKs:

| Dimension | Claude Code SDK | Pi-Code SDK |
|-----------|----------------|-------------|
| **Package** | `@anthropic-ai/claude-agent-sdk` | `@mariozechner/pi-coding-agent` |
| **Scope** | Anthropic models only | 20+ LLM providers (OpenAI, Google, local, etc.) |
| **Execution** | In-process `query()` function | Spawned subprocess (JSONL over stdio) |
| **Tool format** | Zod schemas via `tool()` helper | JSON Schema with `mcp__session__` prefix |
| **Streaming** | Async generator of `SDKMessage` | JSONL events on subprocess stdout |
| **Standard tools** | Built-in (Read, Write, Edit, Bash, Grep, Glob) | Built-in (lowercase equivalents) |
| **Permission model** | Native SDK integration | Via tool name mapping + config |
| **Dependency type** | Peer dependency (runtime) | Direct dependency (bundled) |

The platform must make both runtimes **interchangeable** — the UI, session manager, and business logic never know which backend is running.

---

## Backend Abstraction Layer

### The `AgentBackend` Interface

**File:** `packages/shared/src/agent/backend/types.ts`

Every provider must implement this interface:

```typescript
interface AgentBackend {
  // Lifecycle
  chat(message, attachments?, options?): AsyncGenerator<AgentEvent>;
  abort(reason?: string): Promise<void>;
  forceAbort(reason: string): void;
  interruptForHandoff(reason: string): void;
  destroy(): void;
  dispose(): void;
  postInit(): Promise<PostInitResult>;

  // Configuration
  getModel(): string;
  setModel(model: string): void;
  getThinkingLevel(): ThinkingLevel;
  setThinkingLevel(level: ThinkingLevel): void;

  // Permissions
  getPermissionMode(): PermissionMode;
  setPermissionMode(mode: PermissionMode): void;
  cyclePermissionMode(): PermissionMode;

  // Sources (MCP / API)
  setSourceServers(mcpServers, apiServers, intendedSlugs): void;
  getActiveSourceSlugs(): string[];
  markSourceUnseen(slug: string): void;
}
```

**Key design decision:** `chat()` returns `AsyncGenerator<AgentEvent>` — a unified event stream that both backends produce regardless of their internal streaming mechanism.

### Shared Base Class

**File:** `packages/shared/src/agent/base-agent.ts`

`BaseAgent` provides common functionality used by both runtimes:
- Model/thinking level management
- Permission mode management with mode version tracking
- Config watchers for hot-reloading
- Tool allowlist/blocklist enforcement
- Write path extraction from shell commands
- Write target validation with symlink-aware containment

Both `ClaudeAgent` and `PiAgent` extend `BaseAgent`.

---

## Backend Factory & Driver Registry

### Factory

**File:** `packages/shared/src/agent/backend/factory.ts`

The factory resolves which backend to create based on the LLM connection configuration:

```
User creates session
  → Session has LLM connection (or falls back to workspace/global default)
    → Connection has providerType: "anthropic" | "pi" | "pi_compat"
      → providerTypeToAgentProvider() maps to AgentProvider
        → createBackendFromResolvedContext() instantiates the backend
```

### Driver Registry

**File:** `packages/shared/src/agent/backend/internal/drivers/`

Each provider registers a `ProviderDriver` with hooks for every stage:

```typescript
interface ProviderDriver {
  provider: AgentProvider;

  // Path resolution for SDK binaries
  initializeHostRuntime(context): void;  // Opportunistic (no throw)
  prepareRuntime(context): void;          // Strict (throws on missing)

  // Build payload for agent constructor
  buildRuntime(context): ProviderRuntimePayload;

  // Model listing
  fetchModels(context): Promise<ModelInfo[]>;

  // Connection validation
  validateStoredConnection?(context): Promise<ValidationResult>;

  // Lightweight connectivity test
  testConnection?(context): Promise<TestResult>;
}
```

**Registered drivers:**

```typescript
const DRIVER_REGISTRY: Record<AgentProvider, ProviderDriver> = {
  anthropic: anthropicDriver,  // packages/shared/src/agent/backend/internal/drivers/anthropic.ts
  pi:         piDriver,        // packages/shared/src/agent/backend/internal/drivers/pi.ts
};
```

### Runtime Path Resolution

**File:** `packages/shared/src/agent/backend/internal/runtime-resolver.ts`

The resolver locates SDK executables using a priority chain:

```
1. Packaged app paths   (production: Electron resources/)
2. Monorepo node_modules (development: ../../node_modules/)
3. System PATH           (fallback for non-packaged installs)
```

Resolved paths include:
- `claudeCliPath` — Claude Agent SDK CLI
- `piServerPath` — Pi agent server entry point
- `interceptorBundlePath` — Network interceptor for both backends
- `sessionServerPath` — Session MCP server
- `bridgeServerPath` — Bridge MCP server
- `nodeRuntimePath` — Node.js/Bun executable

Platform-specific binary resolution handles macOS, Windows, and Linux with architecture detection (x64/arm64).

---

## Claude Agent Runtime

**File:** `packages/shared/src/agent/claude-agent.ts`

### How It Works

The Claude backend runs **in-process** — it calls the Claude Agent SDK's `query()` function directly:

```typescript
class ClaudeAgent extends BaseAgent {
  async *chatImpl(message, attachments, options): AsyncGenerator<AgentEvent> {
    // 1. Build SDK options (model, thinking, system prompt, tools)
    const sdkOptions = this.buildSdkOptions();

    // 2. Create MCP servers for sources and session tools
    const servers = this.createMcpServers();

    // 3. Call SDK query() — runs in-process
    this.currentQuery = query({
      model: this.getModel(),
      messages: conversationHistory,
      tools: servers,
      options: sdkOptions,
    });

    // 4. Stream events through adapter
    for await (const message of this.currentQuery) {
      const events = await this.eventAdapter.adapt(message);
      for (const event of events) {
        yield event;
      }
    }
  }
}
```

### Key Characteristics

| Aspect | Implementation |
|--------|---------------|
| **Execution model** | In-process async generator |
| **Standard tools** | Provided by SDK (Read, Write, Edit, Bash, Grep, Glob) |
| **Session tools** | Created via `tool()` helper with Zod schemas |
| **MCP sources** | Wrapped as SDK MCP servers via `createSourceProxyServers()` |
| **Tool names** | PascalCase (`Read`, `Edit`, `Bash`) |
| **Permissions** | Native SDK `permissionMode` + `PreToolUse` pipeline |
| **Auth** | API key or OAuth token injected into SDK environment |
| **Network interception** | `unified-network-interceptor.ts` for request/response tracking |
| **Abort** | `AbortController` passed to SDK |

### MCP Source Proxy

MCP sources are wrapped as Claude SDK MCP servers with automatic tool name conversion:

```typescript
// MCP tool: {"name": "createIssue", "inputSchema": {...}}
// Becomes Claude SDK tool: mcp__linear__createIssue
function createSourceProxyServers(pool: McpClientPool) {
  for (const slug of pool.getConnectedSlugs()) {
    const mcpTools = pool.getTools(slug);
    const proxyTools = mcpTools.map(mcpTool => {
      return tool(
        mcpTool.name,
        mcpTool.description,
        jsonSchemaToZodShape(mcpTool.inputSchema),  // JSON Schema → Zod
        async (args) => {
          const result = await pool.callTool(proxyName, args);
          return { content: [{ type: 'text', text: result.content }] };
        }
      );
    });
    servers[slug] = createSdkMcpServer({ name: `source-proxy-${slug}`, tools: proxyTools });
  }
}
```

---

## Pi Agent Runtime

**File:** `packages/shared/src/agent/pi-agent.ts`

### How It Works

The Pi backend runs as a **spawned subprocess** — it communicates via JSONL over stdin/stdout:

```typescript
class PiAgent extends BaseAgent {
  async *chatImpl(message, attachments, options): AsyncGenerator<AgentEvent> {
    // 1. Spawn pi-agent-server subprocess (if not already running)
    if (!this.subprocess) {
      this.subprocess = spawn(this.resolvedPaths.piServerPath, [...args]);
      this.setupStdioHandlers();
    }

    // 2. Send message to subprocess via stdin (JSONL)
    this.subprocess.stdin.write(JSON.stringify({ type: 'chat', message, ... }));

    // 3. Read events from stdout via event queue
    while (true) {
      const event = await this.eventQueue.pop();
      if (event.type === 'complete') break;
      yield event;
    }
  }
}
```

### Subprocess Protocol

```
┌────────────┐   stdin (JSONL)    ┌──────────────────┐
│            │ ──────────────────> │                  │
│  PiAgent   │                    │  pi-agent-server  │
│  (parent)  │ <────────────────  │  (child process)  │
│            │   stdout (JSONL)   │                  │
└────────────┘                    └──────────────────┘
```

**Messages sent to subprocess:**
- `{ type: 'chat', message, attachments, options }` — start a turn
- `{ type: 'abort', reason }` — cancel current turn
- `{ type: 'set_model', model }` — change model
- `{ type: 'set_sources', ... }` — update MCP sources

**Events received from subprocess:**
- `{ type: 'ready' }` — subprocess initialized
- `{ type: 'text_delta', text }` — streaming text
- `{ type: 'tool_start', name, input }` — tool call beginning
- `{ type: 'tool_result', ... }` — tool call result
- `{ type: 'complete', reason }` — turn finished
- `{ type: 'error', message }` — error occurred

### Key Characteristics

| Aspect | Implementation |
|--------|---------------|
| **Execution model** | Spawned subprocess (JSONL over stdio) |
| **Standard tools** | Built into Pi SDK (lowercase: `read`, `write`, `edit`, `bash`) |
| **Session tools** | Proxied via `getSessionToolProxyDefs()` with JSON Schema format |
| **MCP sources** | Passed as tool definitions with `mcp__session__` prefix |
| **Tool names** | lowercase → PascalCase via `PI_TOOL_NAME_MAP` |
| **Permissions** | Enforced by mapping tool names + Pi SDK config |
| **Auth** | Credentials passed in subprocess environment |
| **Abort** | `turnInterrupt()` sent to subprocess client |

### Tool Name Mapping

Pi SDK uses lowercase tool names. The permission system and UI expect PascalCase. The mapping bridges this:

```typescript
const PI_TOOL_NAME_MAP: Record<string, string> = {
  bash:  'Bash',
  read:  'Read',
  write: 'Write',
  edit:  'Edit',
  grep:  'Grep',
  find:  'Find',
  ls:    'Ls',
  // ... more tools
};
```

---

## Event Adapter Pattern

Both backends produce provider-specific event types that must be normalized into a unified `AgentEvent` stream.

### Base Event Adapter

**File:** `packages/shared/src/agent/backend/base-event-adapter.ts`

```typescript
abstract class BaseEventAdapter<TProviderMessage> {
  abstract adapt(message: TProviderMessage): Promise<AgentEvent[]>;

  // Shared utilities
  startTurn(): void;
  createToolStart(name, input, parentId?): AgentEvent;
  createToolResult(id, output, isError?): AgentEvent;
  createTextDelta(text): AgentEvent;
  createTextComplete(text): AgentEvent;
  createComplete(reason, usage?): AgentEvent;
}
```

Shared state managed in the base:
- Command output accumulation (for classifying read commands)
- Block reason tracking (end_turn, tool_use, max_tokens)
- Read command classification (bash `cat` → Read tool mapping)

### Claude Event Adapter

**File:** `packages/shared/src/agent/backend/claude/event-adapter.ts`

Maps `SDKMessage` → `AgentEvent`:

| SDK Message Type | AgentEvent(s) |
|-----------------|---------------|
| `assistant` | `text_complete`, `tool_start` (from content blocks) |
| `stream_event (content_block_start)` | `tool_start` (if tool_use block) |
| `stream_event (content_block_delta)` | `text_delta` (text) or `tool_start` progress |
| `stream_event (content_block_stop)` | `text_complete` or tool completion |
| `stream_event (message_delta)` | `complete` (with usage, stop reason) |
| `tool_progress` | `tool_progress` |
| `result` | `complete` (final result) |
| `system` | Status notifications |
| `auth_status` | Token expiry notifications |

State tracking: `ToolIndex` (match tool_use IDs to results), `emittedToolStarts` (dedup), `activeParentTools` (nested tools), `pendingText` (text buffering).

### Pi Event Adapter

**File:** `packages/shared/src/agent/backend/pi/event-adapter.ts`

Maps Pi Core Events → `AgentEvent`:

| Pi Event | AgentEvent |
|----------|-----------|
| Text delta | `text_delta` + `text_complete` |
| Tool call start | `tool_start` (with PascalCase name via `PI_TOOL_NAME_MAP`) |
| Tool call result | `tool_result` |
| Compaction | Translated to status event |
| Auto-retry | Translated to status event |

### Unified AgentEvent Type

```typescript
type AgentEvent =
  | { type: 'text_delta'; text: string }
  | { type: 'text_complete'; text: string }
  | { type: 'tool_start'; id: string; name: string; input: Record<string, any> }
  | { type: 'tool_progress'; id: string; text: string }
  | { type: 'tool_result'; id: string; output: string; isError?: boolean }
  | { type: 'status'; message: string }
  | { type: 'error'; error: Error }
  | { type: 'complete'; reason: string; usage?: TokenUsage }
  | { type: 'source_activated'; slug: string }
  | { type: 'permission_request'; ... }
  | { type: 'auth_request'; ... };
```

The UI and session manager consume **only** `AgentEvent` — they never see provider-specific types.

---

## Tool System & Parity

### Canonical Tool Registry

**File:** `packages/session-tools-core/src/tool-defs.ts`

A single source of truth for all session-scoped tools:

```typescript
interface SessionToolDef {
  name: string;
  description: string;
  inputSchema: ZodSchema;          // Zod for type safety
  executionMode: 'registry' | 'backend';
  safeMode: 'allow' | 'block';
  readOnly: boolean;
  handler?: (ctx, args) => Promise<ToolResult>;
}
```

- **`registry` tools** — have concrete handlers, executed directly in the main process
- **`backend` tools** — require provider-specific implementation (`call_llm`, `spawn_session`, `browser_tool`)

### Format Conversion Pipeline

```
                    ┌──────────────────────┐
                    │  Zod Schema (source)  │
                    └──────┬───────────────┘
                           │
              ┌────────────┼────────────────┐
              │            │                │
    ┌─────────▼─────┐  ┌───▼──────┐  ┌─────▼──────────┐
    │ Claude SDK     │  │ JSON     │  │ Pi SDK          │
    │ tool() with    │  │ Schema   │  │ mcp__session__  │
    │ Zod .shape     │  │ (MCP)    │  │ prefix + JSON   │
    └───────────────┘  └──────────┘  └─────────────────┘
```

- **Claude SDK:** Uses Zod schemas directly via `tool()` helper
- **MCP Protocol:** Converted via `zodToJsonSchema()` for JSON Schema format
- **Pi SDK:** Converted to JSON Schema + `mcp__session__` prefix via `getSessionToolProxyDefs()`

### Tool Parity Tests

**Files:**
- `packages/shared/src/agent/backend/claude/session-tool-parity.test.ts`
- `packages/shared/src/agent/backend/pi/session-tool-parity.test.ts`

These tests verify that each backend implements all required `backend`-mode tools:

```typescript
const SESSION_BACKEND_TOOL_NAMES = new Set(['call_llm', 'spawn_session', 'browser_tool']);

// Claude parity test
it('implements all backend-mode session tools from core registry', () => {
  const missing = [...SESSION_BACKEND_TOOL_NAMES].filter(
    (name) => !CLAUDE_BACKEND_SESSION_TOOL_NAMES.has(name)
  );
  expect(missing).toEqual([]);
});
```

### MCP Source Tools

External MCP sources (Linear, GitHub, Slack, etc.) are integrated differently per backend:

**Claude:** MCP tools wrapped as SDK MCP servers with `jsonSchemaToZodShape()` conversion → tool names like `mcp__linear__createIssue`

**Pi:** MCP tools passed as JSON Schema definitions to subprocess → prefixed with `mcp__session__` → tool names like `mcp__session__mcp__linear__createIssue`

---

## Session Lifecycle & Connection Resolution

### Connection Resolution Priority

```
Session connection (locked after first message)
  ↓ fallback
Workspace default connection
  ↓ fallback
Global default connection
```

### Provider Type → Backend Mapping

```typescript
type LlmProviderType = 'anthropic' | 'pi' | 'pi_compat';

function providerTypeToAgentProvider(type: LlmProviderType): AgentProvider {
  switch (type) {
    case 'anthropic':  return 'anthropic';  // → ClaudeAgent
    case 'pi':         return 'pi';          // → PiAgent (cloud providers)
    case 'pi_compat':  return 'pi';          // → PiAgent (self-hosted/Ollama)
  }
}
```

### Authentication Per Provider

| Auth Type | Anthropic | Pi | Pi Compat |
|-----------|-----------|----|-----------|
| API Key | Yes | Yes | Yes |
| API Key + Endpoint | Yes | Yes | Yes |
| OAuth | Yes | Yes | — |
| Bearer Token | Yes | — | — |
| IAM Credentials (AWS) | Yes | — | — |
| Service Account (GCP) | Yes | — | — |
| Environment | Yes | — | — |
| None | Yes | Yes | Yes |

### Session Lifecycle

```
1. CREATE
   → Generate session ID (YYMMDD-adjective-noun)
   → Create directory: sessions/{id}/ (plans/, attachments/, data/, downloads/)
   → Initialize session.jsonl with header

2. SEND MESSAGE
   → Resolve LLM connection (session → workspace → global)
   → createBackendFromConnection() → ClaudeAgent or PiAgent
   → Load MCP sources → setSourceServers()
   → agent.chat(message) → AsyncGenerator<AgentEvent>
   → Process events → push to UI via WebSocket
   → Save session after each turn

3. TOOL EXECUTION
   → Permission check (mode + tool allowlist + path validation)
   → Registry tools: execute handler directly
   → Backend tools: delegate to provider
   → MCP tools: call via MCP client pool

4. COMPLETE
   → Emit 'complete' event with usage stats
   → Persist session to disk
   → Clean up resources

5. ABORT / INTERRUPT
   → ClaudeAgent: AbortController
   → PiAgent: turnInterrupt() to subprocess
   → Queue redirect message if mid-stream
```

### Session Branching

Sessions can fork from any message:

```typescript
interface BranchSeed {
  branchFromMessageId: string;       // Hard context cutoff
  branchFromSdkSessionId: string;    // Provider session ID
  branchFromSdkTurnId: string;       // Provider turn anchor
}
```

- **Claude:** Uses SDK `resumeSessionAt` + `forkSession` for native branching
- **Pi:** Summarizes parent context → injects as system message (fallback)

---

## Transport & Streaming

### WebSocket RPC Protocol

**File:** `packages/server-core/src/transport/server.ts`

All clients (CLI, IDE, WebUI) communicate via WebSocket RPC:

```
┌────────┐     WSS/WS      ┌──────────────┐
│  CLI   │ ◄─────────────► │              │
├────────┤                 │  WsRpcServer  │
│  IDE   │ ◄─────────────► │              │
│(Electron)│                │  (Node/Bun)  │
├────────┤                 │              │
│ WebUI  │ ◄─────────────► │              │
└────────┘                 └──────────────┘
```

**Message envelope:**
```typescript
{
  id: string;              // Unique message ID
  type: 'handshake' | 'request' | 'response' | 'event' | 'error';
  channel?: string;        // RPC channel (e.g., 'sessions')
  args?: any[];            // Request arguments
  seq?: number;            // Event sequence number
}
```

### Streaming Flow

```
LLM Provider
  → Claude SDK / Pi Subprocess
    → Provider-specific events (SDKMessage / Pi Core Event)
      → Event Adapter (ClaudeEventAdapter / PiEventAdapter)
        → AgentEvent (unified)
          → Session Manager
            → WebSocket push to client
              → UI renders incrementally
```

**Key properties:**
- Streaming is chunked — `text_delta` events arrive continuously
- Tool calls emit `tool_start` → `tool_progress` → `tool_result`
- Binary data (images, files) encoded as base64 in the JSON transport
- Event buffer (100 events, 5 min TTL) supports reconnection replay

### Handshake & Reconnection

1. Client connects with `protocolVersion`, `clientCapabilities`, `workspaceId`
2. Server assigns `clientId`, registers channels
3. On disconnect, client reconnects with `reconnectClientId` + `lastSeq`
4. Server replays buffered events if within TTL; otherwise sends `stale` flag

---

## How to Add a New Provider

Adding a new agent runtime (e.g., OpenAI Agent SDK) requires these steps:

### 1. Create a Driver

**File:** `packages/shared/src/agent/backend/internal/drivers/openai.ts`

```typescript
export const openaiDriver: ProviderDriver = {
  provider: 'openai',
  initializeHostRuntime: ({ hostRuntime, resolvedPaths }) => { /* resolve SDK paths */ },
  prepareRuntime: ({ hostRuntime, resolvedPaths }) => { /* strict validation */ },
  buildRuntime: ({ context }) => ({ /* provider-specific payload */ }),
  fetchModels: async ({ connection, credentials }) => { /* list models */ },
  testConnection: async (args) => { /* connectivity check */ },
};
```

### 2. Register in Driver Registry

**File:** `packages/shared/src/agent/backend/factory.ts`

```typescript
import { openaiDriver } from './internal/drivers/openai';

const DRIVER_REGISTRY = {
  anthropic: anthropicDriver,
  pi:        piDriver,
  openai:    openaiDriver,  // ← Add here
};
```

### 3. Implement the Agent Backend

**File:** `packages/shared/src/agent/openai-agent.ts`

```typescript
export class OpenAIAgent extends BaseAgent {
  async *chatImpl(message, attachments, options): AsyncGenerator<AgentEvent> {
    // Call OpenAI SDK
    // Normalize events
    // Yield AgentEvent
  }
}
```

### 4. Implement the Event Adapter

**File:** `packages/shared/src/agent/backend/openai/event-adapter.ts`

```typescript
export class OpenAIEventAdapter extends BaseEventAdapter<OpenAIMessage> {
  async adapt(message: OpenAIMessage): Promise<AgentEvent[]> {
    // Map OpenAI events → AgentEvent
  }
}
```

### 5. Add Tool Parity Tests

**File:** `packages/shared/src/agent/backend/openai/session-tool-parity.test.ts`

Verify all `backend`-mode tools are implemented: `call_llm`, `spawn_session`, `browser_tool`.

### 6. Add Runtime Path Resolution

Update `runtime-resolver.ts` to locate the new SDK binaries.

### Summary: Files to Create/Modify

| File | Action |
|------|--------|
| `backend/internal/drivers/openai.ts` | Create |
| `backend/openai/event-adapter.ts` | Create |
| `agent/openai-agent.ts` | Create |
| `backend/openai/session-tool-parity.test.ts` | Create |
| `backend/factory.ts` | Modify (add to registry) |
| `backend/internal/runtime-resolver.ts` | Modify (add paths) |
| `config/llm-connections.ts` | Modify (add provider type) |

No changes needed in: UI, session manager, transport layer, or business logic.

---

## Design Trade-offs

### In-Process vs Subprocess

| Trade-off | Claude (in-process) | Pi (subprocess) |
|-----------|-------------------|-----------------|
| **Latency** | Lower — no IPC overhead | Higher — JSONL serialization + IPC |
| **Isolation** | Lower — crash takes down host | Higher — crash isolated to subprocess |
| **Resource control** | Lower — shares host memory | Higher — OS-level resource limits |
| **Debugging** | Easier — same process | Harder — must attach to subprocess |
| **Startup** | Faster — no spawn | Slower — process creation + init |

### Zod vs JSON Schema

| Trade-off | Zod (Claude) | JSON Schema (Pi/MCP) |
|-----------|-------------|---------------------|
| **Type safety** | Full TypeScript inference | Manual casting required |
| **Runtime validation** | Built-in | Requires separate validator |
| **Interop** | Must convert for non-SDK consumers | Universal standard |
| **Ecosystem** | TypeScript-native | Language-agnostic |

### AsyncGenerator vs Callbacks

The `AsyncGenerator<AgentEvent>` pattern was chosen for `chat()` because:
- **Backpressure** — consumers control the pace (no buffer overflow)
- **Cancellation** — breaking the loop naturally stops production
- **Composability** — `yield*` chains adapters without callbacks
- **Familiarity** — `for await` is idiomatic in modern TypeScript

### Peer Dependency for Claude SDK

Claude Agent SDK is a `peerDependency` rather than a direct dependency because:
- It may not be installed in all environments (Pi-only deployments)
- Version flexibility — different host apps may pin different SDK versions
- Legal/licensing — SDK has specific terms separate from the platform

---

*Document generated from codebase analysis. Last updated: 2026-04-18.*
