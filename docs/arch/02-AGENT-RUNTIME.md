# Agent Runtime — Dual-Backend Architecture

> How Craft Agents runs AI conversations using two interchangeable backends (Claude SDK and Pi SDK), manages permissions, and orchestrates tool execution.

---

## 1. Architecture Overview

```
┌────────────────────────────────────────────────────────────────┐
│                      SessionManager                            │
│  (owns session lifecycle, routes to agent backend)             │
└───────────┬───────────────────────────┬────────────────────────┘
            │                           │
    ┌───────▼────────┐         ┌───────▼────────┐
    │  ClaudeAgent   │         │    PiAgent     │
    │  (in-process)  │         │  (subprocess)  │
    │                │         │                │
    │ Claude Agent   │         │ Pi SDK         │
    │ SDK            │         │ (JSONL/stdio)  │
    └───────┬────────┘         └───────┬────────┘
            │                          │
            │   ┌──────────────────┐   │
            └──▶│  BaseAgent       │◀──┘
                │  (shared logic)  │
                └──────┬───────────┘
                       │
          ┌────────────┼────────────┐
          │            │            │
    ┌─────▼─────┐ ┌───▼────┐ ┌────▼──────┐
    │ Permission │ │ Mode   │ │ Source    │
    │ Manager   │ │ Manager│ │ Manager  │
    └───────────┘ └────────┘ └───────────┘
```

### Design Principle

The agent runtime follows a **Strategy pattern** with a shared base. `BaseAgent` contains all common logic (session loading, source resolution, permission checks, streaming). `ClaudeAgent` and `PiAgent` implement the `chatImpl()` template method with their respective SDK integrations.

---

## 2. BaseAgent (1,225 lines)

### Core Responsibilities

```
BaseAgent
├── Session Management
│   ├── loadSession(), saveSession()
│   ├── appendMessage(), updateStreamingTurn()
│   └── getConversationHistory()
├── Source Resolution
│   ├── resolveEnabledSources()
│   ├── buildMcpTools() → tool[]
│   └── buildApiTools() → tool[]
├── Permission Checks
│   ├── checkPermission(command) → allow/deny
│   ├── getPermissionMode() → safe/ask/allow-all
│   └── requestToolPermission() → user approval
├── Prompt Construction
│   ├── buildSystemPrompt()
│   ├── injectWorkspaceContext()
│   └── injectUserPreferences()
├── Streaming
│   ├── emitEvent(type, data)
│   ├── onTurnStart(), onTurnDelta(), onTurnComplete()
│   └── onToolCall(), onToolResult()
└── Error Handling
    ├── classifyError() → retryable/fatal
    ├── retryWithBackoff()
    └── emitErrorEvent()
```

### Template Method Pattern

```typescript
abstract class BaseAgent {
  // Template method — calls chatImpl()
  async chat(sessionId: string, message: string): Promise<void> {
    const session = this.loadSession(sessionId)
    this.emitEvent('turn_start', { sessionId })

    try {
      await this.chatImpl(session, message)  // ← subclass implements
    } catch (error) {
      this.handleError(sessionId, error)
    }

    this.emitEvent('turn_complete', { sessionId })
    this.saveSession(session)
  }

  // Abstract — each backend implements differently
  protected abstract chatImpl(
    session: StoredSession,
    message: string
  ): Promise<void>
}
```

### Shared Module Delegation

BaseAgent delegates to specialized modules rather than implementing everything:

| Module | Lines | Purpose |
|--------|-------|---------|
| `PermissionManager` | ~600 | Command validation, bash/PowerShell checking |
| `SourceManager` | ~400 | MCP pool management, API tool generation |
| `PromptBuilder` | ~500 | System prompt construction from config |
| `ModeManager` | ~2,170 | Permission mode transitions and enforcement |
| `ThinkingLevels` | ~127 | Model thinking configuration |
| `BashValidator` | ~627 | Bash command safety analysis |
| `PowerShellValidator` | ~1,095 | PowerShell command safety analysis |

---

## 3. ClaudeAgent (2,683 lines)

### In-Process Agent

ClaudeAgent runs the Claude Agent SDK directly in the server process:

```
SessionManager
    │
    ▼
ClaudeAgent.chatImpl()
    │
    ├── 1. Build tools array
    │   ├── Session-scoped tools (browser_tool, call_llm, SubmitPlan, ...)
    │   ├── MCP tools from sources (via MCP pool)
    │   └── Built-in tools (Read, Write, Bash, ...)
    │
    ├── 2. Create SDK MCP server
    │   └── createSdkMcpServer({ name: 'session', tools })
    │
    ├── 3. Initialize SDK Agent
    │   └── new Agent({ model, mcpServers, systemPrompt })
    │
    ├── 4. Stream conversation
    │   ├── agent.stream(messages)
    │   ├── Route events → emitEvent()
    │   ├── Handle tool calls → execute via MCP server
    │   └── Handle permission requests → prompt user
    │
    └── 5. Process results
        ├── Append assistant message
        ├── Update token usage
        └── Trigger persistence
```

### SDK Integration Details

```typescript
class ClaudeAgent extends BaseAgent {
  private sdk: Agent | null = null

  protected async chatImpl(session: StoredSession, message: string) {
    // Get or create SDK agent
    this.sdk = new Agent({
      model: session.model || this.defaultModel,
      mcpServers: {
        session: this.createSessionMcpServer(session),
        // Additional MCP servers from sources
        ...this.createSourceMcpServers(session),
      },
    })

    // Stream with event routing
    const stream = this.sdk.stream([
      ...session.messages,
      { role: 'user', content: message }
    ])

    for await (const event of stream) {
      switch (event.type) {
        case 'content_block_delta':
          this.emitTextDelta(event.delta.text)
          break
        case 'tool_use':
          await this.handleToolUse(event)
          break
        case 'permission_request':
          await this.handlePermissionRequest(event)
          break
      }
    }
  }
}
```

### MCP Server Creation

ClaudeAgent creates an **in-process MCP server** wrapping session-scoped tools:

```
createSdkMcpServer({
  name: 'session',
  tools: [
    browser_tool,       // Browser automation
    call_llm,           // Secondary LLM calls
    SubmitPlan,         // Plan approval workflow
    spawn_session,      // Independent session creation
    set_session_labels, // Label management
    set_session_status, // Status management
    get_session_info,   // Session metadata
    list_sessions,      // Session listing
    resolve_labels,     // Label resolution
    resolve_status,     // Status resolution
    send_agent_message, // Inter-session messaging
  ]
})
```

This MCP server runs **in the same process** as the agent — no subprocess overhead.

### Permission Flow

```
Agent wants to execute: Bash("rm -rf /tmp/test")

1. SDK emits permission_request event
   │
   ▼
2. ClaudeAgent permission handler
   │  Checks ModeManager:
   │  - "safe" mode → deny immediately
   │  - "allow-all" mode → approve immediately
   │  - "ask" mode → prompt user
   ▼
3. Push event to renderer: permission_request
   │  RPC push → renderer → PermissionRequestDialog
   ▼
4. User clicks Allow/Deny
   │  RPC invoke → SessionManager → ClaudeAgent
   ▼
5. Permission response forwarded to SDK
   │  Agent continues or aborts tool call
```

### Event Adapter

ClaudeAgent uses an event adapter to translate SDK events into the common event format:

```
Claude SDK Events              Common Agent Events
─────────────────              ───────────────────
content_block_start     →      turn_start
content_block_delta     →      text_delta
content_block_stop      →      text_complete
tool_use                →      tool_call
tool_result             →      tool_result
permission_request      →      permission_request
message_stop            →      turn_complete
error                   →      error
```

### Environment Sanitization

ClaudeAgent strips Claude-specific environment variables before spawning SDK:

```typescript
// Removed from subprocess env:
// - CLAUDE_CODE_USE_BEDROCK
// - AWS_BEARER_TOKEN_BEDROCK
// - ANTHROPIC_BEDROCK_BASE_URL
// Reason: Pi Bedrock uses its own AWS env path
```

---

## 4. PiAgent (2,247 lines)

### Subprocess Agent

PiAgent runs the Pi SDK as an out-of-process subprocess:

```
SessionManager
    │
    ▼
PiAgent.chatImpl()
    │
    ├── 1. Spawn or reuse pi-agent-server subprocess
    │   └── ChildProcess ( Bun.spawn('pi-agent-server') )
    │
    ├── 2. Send configuration via JSONL
    │   ├── { type: 'init', model, sources, tools }
    │   └── { type: 'register_tools', tools: proxyDefs }
    │
    ├── 3. Stream conversation via JSONL
    │   ├── → { type: 'message', content: "user text" }
    │   ├── ← { type: 'text_delta', content: "response..." }
    │   ├── ← { type: 'tool_call', name, args }
    │   ├── → { type: 'tool_result', id, result }
    │   └── ← { type: 'complete', usage }
    │
    └── 4. Handle tool execution
        ├── Session tools → direct execution
        │   (browser_tool, call_llm, etc.)
        └── Source MCP tools → MCP pool
```

### JSONL Communication Protocol

Messages are newline-delimited JSON over stdio:

```
Parent → Child (stdin):
{ "type": "init", "model": "claude-sonnet-4-6", "sources": [...] }
{ "type": "message", "content": "Hello, how are you?" }
{ "type": "tool_result", "toolUseId": "xyz", "result": "..." }
{ "type": "permission_response", "requestId": "abc", "approved": true }

Child → Parent (stdout):
{ "type": "ready" }
{ "type": "text_delta", "content": "I'm doing " }
{ "type": "text_delta", "content": "well, thanks!" }
{ "type": "tool_call", "toolUseId": "xyz", "name": "Read", "args": {...} }
{ "type": "permission_request", "requestId": "abc", "command": "rm ..." }
{ "type": "complete", "usage": { "inputTokens": 1000, "outputTokens": 500 } }
{ "type": "error", "message": "Rate limited" }
```

### Tool Proxy Pattern

PiAgent executes session-scoped tools directly in the parent process:

```
Pi subprocess calls tool "browser_tool"
    │
    ▼
JSONL message: { type: "tool_call", name: "browser_tool", args: { command: "navigate ..." } }
    │
    ▼
PiAgent.executeSessionTool()
    │  const fns = getSessionScopedToolCallbacks(sessionId)?.browserPaneFns
    │  executeBrowserToolCommand({ command, fns, sessionId })
    ▼
Result sent back via JSONL: { type: "tool_result", ... }
```

This keeps browser tools, LLM calls, and session management in the parent process where the MCP pool and BrowserPaneManager live.

### Event Adapter

```
Pi SDK Events                Common Agent Events
────────────────             ───────────────────
text_delta            →      text_delta
tool_call             →      tool_call
tool_result           →      tool_result
permission_request    →      permission_request
complete              →      turn_complete
error                 →      error
```

---

## 5. Tool System Architecture

### Tool Registration Flow

```
                    ┌─────────────────────────────┐
                    │    SESSION_TOOL_REGISTRY     │
                    │  (global tool definitions)   │
                    └──────────────┬──────────────┘
                                   │
                    ┌──────────────▼──────────────┐
                    │  getSessionScopedTools()     │
                    │                              │
                    │  1. Read registry             │
                    │  2. Create tool instances     │
                    │  3. Bind session callbacks    │
                    │  4. Wrap in MCP server        │
                    └──────────────┬──────────────┘
                                   │
                 ┌─────────────────┼─────────────────┐
                 │                 │                 │
        ┌────────▼───────┐ ┌─────▼────────┐ ┌──────▼───────┐
        │ Claude SDK     │ │ Pi Agent     │ │ MCP Sources  │
        │ MCP Server     │ │ Proxy Tools  │ │ (via pool)   │
        │ (in-process)   │ │ (parent exec)│ │ (subprocess) │
        └────────────────┘ └──────────────┘ └──────────────┘
```

### Tool Categories

| Category | Source | Examples |
|----------|--------|---------|
| Session-scoped | `session-scoped-tools.ts` | browser_tool, call_llm, SubmitPlan |
| Source MCP | MCP pool | Linear tools, GitHub tools, Craft docs |
| API source | `api-tools.ts` | Gmail listMessages, Calendar events |
| SDK built-in | Claude SDK | Read, Write, Bash, Glob, Grep |

### Session Tool Registration (step-by-step)

```
Step 1: Agent backend registers core callbacks
  registerSessionScopedToolCallbacks(sessionId, {
    queryFn: (req) => this.queryLlm(req),
    onPlanSubmitted: (path) => this.handlePlan(path),
    onAuthRequest: (req) => this.handleAuth(req),
  })

Step 2: Electron adds browser callbacks
  mergeSessionScopedToolCallbacks(sessionId, {
    browserPaneFns: {
      navigate: (url) => bpm.navigate(id, url),
      click: (ref) => bpm.click(id, ref),
      // ...30 methods
    }
  })

Step 3: Tool creation time
  getSessionScopedTools(sessionId, workspaceRoot) → {
    // Each tool gets a lazy getter:
    getBrowserPaneFns: () => getSessionScopedToolCallbacks(sessionId)?.browserPaneFns
    getQueryFn: () => getSessionScopedToolCallbacks(sessionId)?.queryFn
  }

Step 4: Tools wrapped in MCP server
  Claude path: createSdkMcpServer({ name: 'session', tools })
  Pi path: getSessionToolProxyDefs() → send to subprocess

Step 5: Tools available to LLM
  LLM sees: browser_tool, call_llm, SubmitPlan, spawn_session, ...
```

---

## 6. Call LLM Tool (719 lines)

### Purpose

`call_llm` lets the agent invoke a secondary LLM for subtasks:

```
Primary Agent (Claude Sonnet)
    │
    │ "Analyze these 5 files and summarize each"
    │
    ▼
call_llm({
  prompt: "Summarize this file: ...",
  model: "claude-haiku-4-5",   // faster/cheaper model
  attachments: [file1, file2]
})
    │
    ▼
Secondary Model returns structured result
    │
    ▼
Primary Agent continues with summaries
```

### Parallel Execution

Multiple `call_llm` invocations can run in parallel:

```typescript
// Agent prompt: "Check all 5 links"
// → 5 parallel call_llm invocations
// → Results aggregated before continuing
```

### Request Pipeline

```
call_llm({ prompt, model, attachments })
    │
    ├── 1. Validate model and connection
    ├── 2. Process attachments (read files, resize images)
    ├── 3. Build LLM request with system prompt
    ├── 4. Query via session callback:
    │       queryFn({ messages, model, options })
    ├── 5. Parse response (text or structured JSON)
    └── 6. Return result to agent
```

### Output Formats

```typescript
const OUTPUT_FORMATS = {
  text: z.string(),
  json: z.record(z.any()),
  markdown: z.string(),
  // Custom schemas via Zod
}
```

---

## 7. Permission System

### Three Modes

```
┌──────────┐  ┌──────────┐  ┌──────────┐
│   Safe   │  │   Ask    │  │ Allow-All│
│(Explore) │  │(Ask Edit)│  │  (Auto)  │
│          │  │          │  │          │
│ Read-only│  │ Prompt   │  │ Auto-    │
│ Blocks   │  │ user for │  │ approve  │
│ all      │  │ write    │  │ all      │
│ writes   │  │ ops      │  │ commands │
└──────────┘  └──────────┘  └──────────┘
```

### Mode Manager (2,170 lines)

```
ModeManager
├── Mode Resolution
│   ├── Workspace default mode
│   ├── Session override
│   └── Cyclable modes (user can cycle through allowed modes)
├── Command Classification
│   ├── Safe commands (Read, Glob, Grep) → always allowed
│   ├── Dangerous commands (Bash, Write, file ops) → check mode
│   └── Browser commands → check mode + browser enabled
├── Validation Pipeline
│   ├── BashValidator (627 lines) → classify bash commands
│   ├── PowerShellValidator (1,095 lines) → classify PS commands
│   └── Custom rules per permission mode
└── Permission Request Flow
    ├── Build permission prompt for user
    ├── Track pending requests per session
    └── Handle allow/deny/allow-always responses
```

### Command Validation Pipeline

```
Agent calls: Bash("curl -X POST https://api.example.com")
    │
    ▼
1. BashValidator.analyze("curl -X POST https://api.example.com")
    │  Classifies command:
    │  - Network operation (POST)
    │  - External URL
    │  - Risk: MEDIUM
    ▼
2. ModeManager.check(mode, command, classification)
    │
    │  safe mode → DENY ("Network operations blocked in safe mode")
    │  ask mode  → PROMPT (show dialog with command details)
    │  allow-all → ALLOW
    ▼
3. If ask mode:
    │  Push permission_request event to renderer
    │  User sees: "Allow this command?"
    │  Response → permission_response event
    ▼
4. Execute or abort
```

---

## 8. Thinking Levels (127 lines)

### Configuration

Models can use extended thinking at configurable levels:

```
┌───────────────────┬──────────────┬─────────────────────┐
│ Level             │ Budget       │ When to Use          │
├───────────────────┼──────────────┼─────────────────────┤
│ none              │ 0 tokens     │ Simple tasks         │
│ low               │ 10K tokens   │ Standard tasks       │
│ medium            │ 30K tokens   │ Complex reasoning    │
│ high              │ 100K tokens  │ Deep analysis        │
│ max               │ Full context │ Critical reasoning   │
└───────────────────┴──────────────┴─────────────────────┘
```

### Resolution

```
resolveClaudeThinkingOptions({
  level: 'high',
  model: 'claude-sonnet-4-6',
})
→ { thinking: { type: 'enabled', budget_tokens: 100000 } }
```

---

## 9. Session Lifecycle

### Full Lifecycle Diagram

```
┌─────────────────────────────────────────────────────┐
│                 Session Lifecycle                    │
├─────────────────────────────────────────────────────┤
│                                                     │
│  1. CREATE                                         │
│     createSession(workspaceId, options)             │
│     → Generate ID (YYMMDD-adjective-noun)          │
│     → Create directory structure                    │
│     → Write initial header to session.jsonl         │
│     → Register in session index                     │
│                                                     │
│  2. ACTIVATE                                       │
│     User navigates to session                       │
│     → Load messages from JSONL (lazy)               │
│     → Bind agent backend                            │
│     → Register session-scoped callbacks             │
│     → Create MCP server for tools                   │
│                                                     │
│  3. PROCESS                                        │
│     sendMessage(sessionId, text)                    │
│     → Append user message                           │
│     → Invoke agent.chatImpl()                       │
│     → Stream events to renderer                     │
│     → Execute tool calls                            │
│     → Handle permissions                            │
│     → Append assistant message                      │
│     → Debounced save to JSONL                       │
│                                                     │
│  4. IDLE                                           │
│     Processing complete                             │
│     → Session persisted to disk                     │
│     → MCP connections maintained                    │
│     → Browser window persists                       │
│                                                     │
│  5. ARCHIVE / DELETE                               │
│     User archives or deletes                        │
│     → Update metadata                               │
│     → Or remove directory entirely                  │
│     → Unregister callbacks                          │
│                                                     │
└─────────────────────────────────────────────────────┘
```

### Concurrent Sessions

Multiple sessions can be active simultaneously:

```
SessionManager
├── Session "250418-swift-eagle"  → ClaudeAgent → Claude API
├── Session "250418-calm-river"   → PiAgent → Pi subprocess
├── Session "250418-bold-fox"     → ClaudeAgent → Claude API
└── Session "250418-quiet-deer"   → Idle (no processing)
```

Each session has:
- Independent agent backend instance
- Separate MCP tool set
- Separate browser window (if used)
- Separate permission queue
- Separate streaming state

---

## 10. Mini Agent Mode

For simple operations (session labeling, status changes), the system uses a lightweight "mini agent":

```
MINI_AGENT_TOOLS = [
  'set_session_labels',
  'set_session_status',
  'get_session_info',
  'list_sessions',
  'resolve_labels',
  'resolve_status',
  'send_agent_message',
]
```

Mini agents skip full tool loading, source resolution, and MCP connection setup. Used by automations and background tasks.

---

## 11. Key Design Decisions

### Why Dual Backend?

| Aspect | Claude SDK | Pi SDK |
|--------|-----------|--------|
| Execution | In-process | Subprocess |
| Latency | Lower (no IPC) | Higher (JSONL/stdio) |
| Isolation | Shared process | Separate process |
| Memory | Shared heap | Independent heap |
| Crash impact | Can take down server | Isolated failure |
| Model support | Anthropic only | Multi-provider |
| Tool execution | Direct MCP server | Proxy via parent |

### Why In-Process MCP for Claude?

Creating an MCP server in-process avoids subprocess overhead for tool execution. The Claude SDK's `Agent` class accepts `mcpServers` directly — wrapping tools as an MCP server gives the SDK a consistent interface.

### Why JSONL for Pi?

Pi SDK uses stdio for communication. JSONL (one JSON object per line) is simple, parseable, and works with Bun's readline. No framing protocol needed.

---

## 12. Error Handling

### Error Classification

```
Agent Errors
├── Retryable
│   ├── Rate limit (429) → exponential backoff + retry
│   ├── Timeout → retry with increased timeout
│   └── Network error → reconnect + retry
├── Fatal
│   ├── Auth failure (401/403) → emit auth_request event
│   ├── Context window overflow → suggest context clear
│   └── Model not found → emit error event
└── Recoverable
    ├── Tool execution failure → report to agent, continue
    ├── Permission denied → report to agent, try alternative
    └── Source connection lost → reconnect source, retry
```

### Crash Recovery

```
Pi subprocess crashes:
    │
    ▼
1. Detect exit (non-zero code or signal)
2. Log error with subprocess stderr
3. Emit error event to renderer
4. Mark session as "processing_failed"
5. Session persists — user can retry
6. Next sendMessage spawns fresh subprocess
```

---

## 13. Performance Characteristics

| Metric | ClaudeAgent | PiAgent |
|--------|------------|---------|
| Tool call latency | ~5ms (in-process) | ~50ms (JSONL roundtrip) |
| Memory per session | Shared heap | +50-100MB subprocess |
| Startup time | ~100ms (SDK init) | ~2s (subprocess spawn) |
| Max concurrent | Limited by API rate | Limited by system memory |
| Crash isolation | None (shared process) | Full process isolation |
