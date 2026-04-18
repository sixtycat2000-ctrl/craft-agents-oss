# Craft Agents OSS — System Architecture Overview

> Canonical architecture reference. Covers the monorepo layout, package dependency graph, technology stack, and cross-cutting patterns shared by every subsystem.

---

## 1. Product Summary

Craft Agents is a multi-session AI agent platform that runs as a desktop Electron app, a headless server, or a CLI client. Users create *workspaces* containing *sessions* (chat threads), *sources* (MCP servers, REST APIs, local files), and *skills* (reusable prompt templates). A dual-backend agent runtime supports Claude SDK and Pi SDK, with browser automation, credential management, and event-driven automations.

---

## 2. Repository Layout

```
craft-agents-oss/
├── apps/
│   ├── electron/          # Desktop GUI (Electron 33+)
│   ├── cli/               # Terminal client
│   ├── viewer/            # Static session web viewer
│   └── webui/             # Built assets for headless server
├── packages/
│   ├── core/              # Type definitions, zero-logic
│   ├── shared/            # Business logic (agent, sources, sessions, config)
│   ├── ui/                # React component library (shared)
│   ├── server-core/       # Headless-agnostic server infrastructure
│   ├── server/            # Standalone headless server binary
│   ├── session-tools-core/# Plan/template utilities
│   ├── session-mcp-server/# MCP server for session tools (stdio)
│   └── pi-agent-server/   # Pi agent subprocess (JSONL/stdio)
├── docs/                  # Architecture docs (this directory)
└── scripts/               # Build scripts, release tooling
```

### Package Roles

| Package | Lines (approx) | Role | External deps |
|---------|----------------|------|---------------|
| `core` | ~2,000 | Pure types and debug util | None |
| `shared` | ~35,000 | All business logic | Claude SDK, Pi SDK, Zod |
| `ui` | ~15,000 | React components, markdown, code viewer | React, Shiki, Framer Motion |
| `server-core` | ~12,000 | WebSocket RPC server, handlers, bootstrap | ws, Bun |
| `server` | ~500 | Headless binary, env config | server-core, shared |
| `session-tools-core` | ~1,500 | Plan file management, template processing | Zod, gray-matter |
| `session-mcp-server` | ~800 | MCP stdio bridge for plan tools | MCP SDK |
| `pi-agent-server` | ~600 | Pi subprocess wrapper | Pi SDK |

---

## 3. Package Dependency Graph

```
                        ┌──────────┐
                        │  core    │  (types only)
                        └────┬─────┘
                             │
              ┌──────────────┼──────────────────┐
              │              │                  │
        ┌─────▼─────┐  ┌────▼─────┐    ┌───────▼───────┐
        │   shared   │  │   ui     │    │ session-tools │
        │            │  │          │    │     core      │
        └─────┬──────┘  └────┬─────┘    └───────┬───────┘
              │              │                  │
    ┌─────────┼──────────────┤          ┌───────▼────────┐
    │         │              │          │ session-mcp    │
    │   ┌─────▼──────────┐  │          │   server       │
    │   │  server-core   │  │          └────────────────┘
    │   └─────┬──────────┘  │
    │         │              │
    │   ┌─────▼──┐     ┌────▼─────┐
    │   │ server │     │ electron │
    │   └────────┘     └────┬─────┘
    │                      │
    │              ┌───────▼────────┐
    │              │ pi-agent-server│
    │              └────────────────┘
    │
    └──▶ cli (depends on shared + server-core)
```

**Key constraint:** `core` has zero runtime dependencies. `shared` depends on `core`. `ui` depends on `core`. Everything else depends on at least `shared`.

---

## 4. Technology Stack

### Runtime & Build

| Layer | Technology | Purpose |
|-------|-----------|---------|
| Runtime | Bun 1.1+ | Primary JS runtime, also package manager |
| Desktop | Electron 33+ | Main process, BrowserWindow, native APIs |
| Build (main) | esbuild | Bundle Electron main + preload |
| Build (renderer) | Vite 6 | React HMR, code splitting, CSS |
| Build (server) | Bun native | No bundling needed |
| Language | TypeScript 5.7 (strict) | All source code |

### Frontend

| Layer | Technology | Purpose |
|-------|-----------|---------|
| UI Framework | React 19 | Component model |
| State | Jotai | Atomic state with per-session isolation |
| Styling | Tailwind CSS v4 | Utility classes |
| Components | shadcn/ui | Base primitives |
| Animation | Framer Motion | Transitions, gestures |
| Syntax | Shiki | Code highlighting |
| Markdown | react-markdown | Chat content rendering |
| Diagrams | Mermaid | Diagram blocks in chat |
| Math | KaTeX | Equation rendering |

### Backend & Protocol

| Layer | Technology | Purpose |
|-------|-----------|---------|
| Transport | WebSocket (ws) | RPC + push events |
| Codec | JSON (+ base64 for binary) | Wire format |
| Auth | Bearer token / JWT cookie | Connection authentication |
| TLS | Node.js crypto | Optional wss:// |
| MCP | Model Context Protocol | Tool integration |
| CDP | Chrome DevTools Protocol | Browser automation |
| LLM SDKs | Claude Agent SDK, Pi SDK | Dual agent backend |

### Storage

| Layer | Technology | Purpose |
|-------|-----------|---------|
| Sessions | JSONL files | Append-only message persistence |
| Config | JSON files | Settings, preferences, themes |
| Credentials | AES-256-GCM encrypted file | Secure secret storage |
| Automations | JSON + JSONL history | Rules + execution log |
| Sources | JSON per source | MCP/API/local config |

---

## 5. Process Architecture

The system runs as multiple cooperating processes:

```
┌─────────────────────────────────────────────────────────────────┐
│                    Electron Desktop App                         │
│                                                                 │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────┐ │
│  │  Main Process    │  │  Renderer (x N)  │  │  Preload     │ │
│  │  (Node.js)       │  │  (Chromium)      │  │  (Bridge)    │ │
│  │                  │  │                  │  │              │ │
│  │  - WindowManager │  │  - React App     │  │  - RoutedCli │ │
│  │  - BrowserPane   │  │  - Jotai atoms   │  │  - electronAPI│ │
│  │  - Menu/Tray     │  │  - Components    │  │              │ │
│  │  - Auto-update   │  │  - Navigation    │  │              │ │
│  └────────┬─────────┘  └────────┬─────────┘  └──────┬───────┘ │
│           │ IPC (context bridge)│                    │         │
│           └────────────────────┼────────────────────┘         │
│                                │                               │
└────────────────────────────────┼───────────────────────────────┘
                                 │
                          WebSocket RPC
                                 │
         ┌───────────────────────┼───────────────────────┐
         │                       │                       │
┌────────▼────────┐   ┌──────────▼──────────┐   ┌───────▼───────┐
│  Embedded       │   │  Headless Server    │   │  Pi Subprocess│
│  Server         │   │  (standalone)       │   │  (stdio)      │
│  (in Electron)  │   │                     │   │               │
│                 │   │  - WebSocket server  │   │  - Pi SDK     │
│  - SessionMgr   │   │  - SessionMgr       │   │  - Tools      │
│  - RPC handlers │   │  - RPC handlers     │   │  - JSONL comm │
│  - ConfigWatch  │   │  - WebUI HTTP       │   └───────┬───────┘
│  - MCP pool     │   │  - Health endpoint  │           │
└─────────────────┘   └─────────────────────┘    subprocess
```

---

## 6. Deployment Modes

### Mode 1: Desktop (Primary)

```
User Machine
┌─────────────────────────────────┐
│  Electron App                   │
│  ┌───────────┐  ┌─────────────┐ │
│  │ Renderer  │  │ Embedded    │ │
│  │ (React)   │──│ Server      │ │
│  └───────────┘  └─────────────┘ │
│         Localhost WebSocket      │
└─────────────────────────────────┘
```

- All-in-one: renderer + server + agent in one process group
- Server listens on 127.0.0.1:random
- Browser automation via Electron BrowserWindow
- File access via local filesystem

### Mode 2: Thin Client (Remote)

```
User Machine                    Remote Server
┌──────────────────┐           ┌──────────────────┐
│  Electron App    │    WS     │  Headless Server │
│  ┌────────────┐  │──────────│  ┌──────────────┐ │
│  │ Renderer   │  │  wss://  │  │ SessionMgr   │ │
│  │ (React)    │  │          │  │ Agent Runtime│ │
│  └────────────┘  │          │  └──────────────┘ │
└──────────────────┘           └──────────────────┘
```

- Renderer connects to remote headless server
- Local-only operations (window management, native dialogs) stay local
- Remote-eligible operations (sessions, sources) route to server
- RoutedClient transparently splits traffic

### Mode 3: CLI

```
Terminal                      Server
┌──────────────┐             ┌──────────────┐
│  CLI Client  │───── WS ───│  Headless or │
│              │             │  Embedded    │
└──────────────┘             └──────────────┘
```

- Text-based interaction with any running server
- JSON output mode for scripting
- Can spawn embedded server for standalone use

### Mode 4: WebUI

```
Browser                       Server
┌──────────────┐             ┌──────────────┐
│  Web Browser │───── WS ───│  Headless    │
│  (static)    │             │  Server      │
└──────────────┘             └──────────────┘
```

- Static assets served from headless server
- JWT session cookie for auth
- Limited capabilities (no native APIs)

---

## 7. Data Flow: User Message to Agent Response

This is the most common interaction pattern in the system:

```
1. User types message in FreeFormInput
   │
   ▼
2. electronAPI.sendMessage(sessionId, text)
   │  (IPC via preload bridge)
   ▼
3. RoutedClient.invoke('sessions:sendMessage', sessionId, text)
   │  (WebSocket RPC to server)
   ▼
4. SessionsHandler → SessionManager.sendMessage()
   │  (returns {started: true} immediately — fire-and-forget)
   ▼
5. Agent Backend selected (ClaudeAgent or PiAgent)
   │
   ├─ ClaudeAgent path:
   │  │
   │  ▼
   │  Claude SDK streams events → tool calls → more events
   │
   ├─ PiAgent path:
   │  │
   │  ▼
   │  Pi subprocess receives prompt → JSONL events → tool results
   │
   ▼
6. Each event pushed via EventSink
   │  sessionManager.setEventSink((channel, target, ...args) => {
   │    server.push(channel, target, ...args)
   │  })
   ▼
7. WsRpcServer.push('session:event', {to:'workspace'}, event)
   │  (per-client sequence numbering + buffer)
   ▼
8. RoutedClient receives event → dispatches to listener
   │
   ▼
9. processAgentEvent() in App.tsx
   │  updates Jotai atom for that specific session
   ▼
10. React re-renders ChatDisplay with new turn data
    │  (streaming text, tool cards, diff viewers)
    ▼
11. Session persisted to JSONL via debounced save queue
```

---

## 8. Cross-Cutting Patterns

### 8.1 Interface Segregation + Lazy Getters

Every subsystem uses interfaces to decouple logic from implementation:

```typescript
// Tool layer knows nothing about Electron
interface BrowserPaneFns {
  navigate(url: string): Promise<...>
  snapshot(): Promise<...>
  // ...30 methods
}

// Factory receives a lazy getter
createBrowserTools({
  sessionId,
  getBrowserPaneFns: () => registry.get(sessionId)?.browserPaneFns
})
```

This pattern repeats for sources, credentials, permissions — each behind a getter that resolves at execution time, allowing registration to happen in any order.

### 8.2 Atomic File Writes

All persistence uses atomic writes to prevent corruption:

```
1. Write to .tmp file
2. fs.rename(.tmp, target)  // atomic on all platforms
```

Used by: session JSONL, config JSON, workspace JSON, credential store.

### 8.3 Event-Driven Architecture

The system uses push events extensively:

```
File change → ConfigWatcher → callback → RPC push → renderer atom update
Agent event → SessionManager → EventSink → RPC push → renderer atom update
Automation trigger → EventLogger → prompt enqueue → session processing
```

All event paths use typed channels with compile-time validation via `BroadcastEventMap`.

### 8.4 Session-Scoped Callbacks

Tools and callbacks are scoped to individual sessions via a registry:

```typescript
Map<sessionId, {
  browserPaneFns?: BrowserPaneFns
  queryFn?: (req) => Promise<result>
  spawnSessionFn?: (req) => Promise<result>
  onPlanSubmitted?: (path) => void
  onAuthRequest?: (req) => void
  // ...session management callbacks
}>
```

This enables multi-session inbox where each session has independent agent state, browser windows, and tool bindings.

### 8.5 Debounced Persistence

Writes are debounced to batch rapid updates:

| Subsystem | Debounce | Reason |
|-----------|----------|--------|
| Session metadata | 100ms (300ms on Windows) | Rapid streaming updates |
| Config changes | 100ms | Multiple field updates |
| File watcher events | 100ms | Directory scan batching |
| Token refresh cooldown | 5 minutes | Rate limiting |

---

## 9. Workspace Data Model

Everything is scoped to a workspace:

```
~/.craft-agent/
├── config.json                    # Global config
├── preferences.json               # User preferences
├── theme.json                     # App-level theme
├── themes/                        # Preset themes (*.json)
├── credentials.enc                # Encrypted credential store
└── workspaces/
    └── {workspace-slug}/
        ├── config.json            # Workspace settings
        ├── sources/               # MCP/API/local sources
        │   └── {source-slug}/
        │       ├── config.json
        │       ├── guide.md
        │       └── icon.*
        ├── sessions/              # Chat sessions
        │   └── {session-id}/
        │       ├── session.jsonl
        │       ├── attachments/
        │       ├── plans/
        │       ├── data/
        │       └── downloads/
        ├── skills/                # Prompt templates
        ├── statuses/              # Status workflow
        ├── labels/                # Label system
        ├── automations.json       # Automation rules
        └── permissions.json       # Workspace permissions
```

---

## 10. API Surface Area

### RPC Channels

~370 channels organized by domain:

| Domain | Channel Count | Local/Remote |
|--------|--------------|-------------|
| sessions | ~25 | Remote-eligible |
| sources | ~20 | Remote-eligible |
| workspaces | ~8 | Local-only |
| files | ~12 | Mixed |
| credentials | ~10 | Remote-eligible |
| browser pane | ~15 | Local-only |
| llm connections | ~12 | Remote-eligible |
| automations | ~8 | Remote-eligible |
| skills | ~10 | Mixed |
| labels/statuses | ~8 each | Remote-eligible |
| settings | ~15 | Remote-eligible |
| system/update | ~10 | Local-only |
| theme | ~5 | Local-only |
| auth/oauth | ~15 | Mixed |
| window | ~8 | Local-only |

### MCP Tools (Session-Scoped)

~15 tools registered per session:

| Tool | Purpose |
|------|---------|
| `browser_tool` | Browser automation (40+ CLI commands) |
| `call_llm` | Secondary LLM invocation |
| `SubmitPlan` | Submit plan for approval |
| `spawn_session` | Create independent session |
| `set_session_labels` | Label management |
| `set_session_status` | Status management |
| `get_session_info` | Session metadata query |
| `list_sessions` | Session listing |
| `resolve_labels` | Label name→ID resolution |
| `resolve_status` | Status name→ID resolution |
| `send_agent_message` | Inter-session messaging |

Plus all MCP server tools from sources (~32+ per Craft document source).

---

## 11. Key Numbers

| Metric | Value |
|--------|-------|
| Total TypeScript lines | ~70,000 |
| Packages | 9 |
| Apps | 4 |
| RPC channels | ~370 |
| Browser tool commands | 40+ |
| Supported LLM providers | 10+ |
| Credential types | 9 |
| Source types | 3 (MCP, API, local) |
| Permission modes | 3 (safe, ask, allow-all) |
| i18n locales | 3 (en, es, zh-Hans) |
| Max WebSocket clients | 50 |
| Event buffer per client | 100 events / 5 min TTL |
| Session ID format | YYMMDD-adjective-noun |
