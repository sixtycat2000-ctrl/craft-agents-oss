# Transport & RPC — WebSocket Communication Layer

> How Craft Agents connects renderer to server via WebSocket RPC, routes local vs remote traffic, handles reconnection with event replay, and delivers push events to the UI.

---

## 1. Transport Architecture

```
┌──────────────────────────────────────────────────────────┐
│                    Electron Renderer                      │
│                                                          │
│  React App                                               │
│    │                                                     │
│    ▼                                                     │
│  electronAPI (preload bridge)                            │
│    │                                                     │
│    ▼                                                     │
│  ┌──────────────────────────────────────────────┐        │
│  │           RoutedClient                       │        │
│  │                                              │        │
│  │  ┌────────────┐      ┌───────────────────┐   │        │
│  │  │ localClient│      │ workspaceClient   │   │        │
│  │  │ (embedded) │      │ (local or remote) │   │        │
│  │  └─────┬──────┘      └────────┬──────────┘   │        │
│  └────────┼──────────────────────┼───────────────┘        │
│           │                      │                        │
└───────────┼──────────────────────┼────────────────────────┘
            │ WebSocket            │ WebSocket
            │ localhost:port       │ remote:port / localhost:port
            │                      │
┌───────────▼──────────┐  ┌───────▼──────────────────────┐
│  Embedded Server     │  │  Remote Headless Server      │
│  (in Electron)       │  │  (standalone)                │
│                      │  │                              │
│  ┌────────────────┐  │  │  ┌────────────────┐         │
│  │ WsRpcServer    │  │  │  │ WsRpcServer    │         │
│  │ - Handlers     │  │  │  │ - Handlers     │         │
│  │ - Event buffer │  │  │  │ - Event buffer │         │
│  │ - Heartbeat    │  │  │  │ - Heartbeat    │         │
│  └────────────────┘  │  │  └────────────────┘         │
│  ┌────────────────┐  │  │  ┌────────────────┐         │
│  │ SessionManager │  │  │  │ SessionManager │         │
│  │ BrowserPaneMgr │  │  │  │ (headless)     │         │
│  └────────────────┘  │  │  └────────────────┘         │
└──────────────────────┘  └──────────────────────────────┘
```

---

## 2. Wire Protocol

### Message Envelope

Every message on the wire shares a common envelope:

```typescript
interface MessageEnvelope {
  id: string              // UUID for request/response correlation
  type: MessageType       // Discriminator
  channel?: string        // RPC channel name
  args?: any[]            // Request or event arguments
  result?: any            // Response result
  error?: {               // Error payload
    code: string
    message: string
  }
  seq?: number            // Per-client sequence for reliable delivery
  serverId?: string       // Server identity stamp
  clientId?: string       // Assigned client ID
  protocolVersion?: string // Protocol version (major.minor)
  clientCapabilities?: string[] // Client capability advertisements
}
```

### Message Types

```
┌────────────────────┬───────────────────────────────────────┐
│ Type               │ Purpose                               │
├────────────────────┼───────────────────────────────────────┤
│ handshake          │ Client → Server: init connection       │
│ handshake_ack      │ Server → Client: accept + assign ID   │
│ request            │ Client → Server: invoke RPC            │
│ response           │ Server → Client: RPC result            │
│ event              │ Server → Client: push notification     │
│ error              │ Server → Client: protocol error        │
│ sequence_ack       │ Client → Server: acknowledge events    │
└────────────────────┴───────────────────────────────────────┘
```

### Codec

The codec handles JSON serialization with special treatment for binary data:

```typescript
// Encoding: Uint8Array → base64 with type marker
{
  data: "SGVsbG8gV29ybGQ=",
  __craftRpcType: "u8"
}

// All other types: standard JSON.stringify
```

---

## 3. Connection Lifecycle

### Handshake Sequence

```
Client                              Server
  │                                   │
  │──── WebSocket CONNECT ──────────▶│
  │                                   │
  │──── handshake ──────────────────▶│
  │     {                             │
  │       protocolVersion: "1.0",     │
  │       token: "bearer-xxx",        │
  │       workspaceId: "abc",         │
  │       webContentsId: 123,         │
  │       clientCapabilities: [...],  │
  │       reconnectClientId: null,    │
  │       lastSeq: null               │
  │     }                             │
  │                                   │
  │◀─── handshake_ack ───────────────│
  │     {                             │
  │       protocolVersion: "1.0",     │
  │       serverVersion: "0.8.9",     │
  │       clientId: "assigned-id",    │
  │       registeredChannels: [...],  │
  │       reconnected: false,         │
  │       stale: false                │
  │     }                             │
  │                                   │
  │     ─── Ready for RPC ───         │
  │                                   │
```

### Handshake Validation

```
Server receives handshake
    │
    ├── Check protocol version (major must match)
    │   Mismatch → close 4004 (PROTOCOL_VERSION_UNSUPPORTED)
    │
    ├── Check authentication
    │   Bearer token OR JWT session cookie
    │   Invalid → close 4005 (AUTH_FAILED)
    │
    ├── Check reconnection
    │   reconnectClientId present?
    │   → Look up disconnected client
    │   → Validate workspace + webContents match
    │   → Replay buffered events
    │
    ├── Assign clientId
    │   Generate unique ID
    │   Register in clients map
    │
    └── Send handshake_ack
```

### Close Codes

```
┌──────┬───────────────────────────────────────────────┐
│ Code │ Reason                                        │
├──────┼───────────────────────────────────────────────┤
│ 4001 │ Handshake timeout (5s)                        │
│ 4002 │ Invalid JSON in message                       │
│ 4003 │ Expected handshake, got other type            │
│ 4004 │ Protocol version unsupported                  │
│ 4005 │ Authentication failed                         │
│ 4006 │ Unknown client (reconnect ID not found)       │
│ 4008 │ Server at capacity (max 50 clients)           │
└──────┴───────────────────────────────────────────────┘
```

---

## 4. Request/Response Pattern

### Client Invocation

```
Client: electronAPI.sendMessage(sessionId, "Hello")
    │
    ▼
RoutedClient.invoke('sessions:sendMessage', sessionId, "Hello")
    │
    ├── Determine route:
    │   isLocalOnly('sessions:sendMessage') → false
    │   → Route to workspaceClient
    │
    ├── Generate UUID for request
    ├── Start 30-second timeout
    ├── Send request envelope:
    │   {
    │     id: "uuid-123",
    │     type: "request",
    │     channel: "sessions:sendMessage",
    │     args: [sessionId, "Hello"]
    │   }
    │
    ▼
Server receives request
    │
    ├── Look up handler for 'sessions:sendMessage'
    │   Not found → send error response (CHANNEL_NOT_FOUND)
    │
    ├── Create RequestContext { clientId, workspaceId, webContentsId }
    │
    ├── Execute handler with 60-second timeout
    │   handler(ctx, sessionId, "Hello")
    │   → Returns { started: true }
    │
    └── Send response envelope:
        {
          id: "uuid-123",
          type: "response",
          result: { started: true }
        }
    │
    ▼
Client receives response
    │
    ├── Match by UUID
    ├── Clear timeout
    └── Resolve promise with result
```

### Fire-and-Forget for Long Operations

`sendMessage` returns immediately. Results stream via push events:

```
Client → Server: request { channel: "sessions:sendMessage", args: [...] }
Server → Client: response { result: { started: true } }  // immediate

// Then, streaming results via push:
Server → Client: event { channel: "session:event", args: [{ type: "text_delta", ... }] }
Server → Client: event { channel: "session:event", args: [{ type: "tool_call", ... }] }
Server → Client: event { channel: "session:event", args: [{ type: "complete", ... }] }
```

---

## 5. Push Event System

### Event Routing

```
Server pushes event:
    server.push(channel, target, ...args)

Target options:
┌───────────────────────────────────────────────────┐
│ { to: 'all' }                                     │
│   → Every connected client                        │
│                                                   │
│ { to: 'workspace', workspaceId: 'abc' }           │
│   → Clients with matching workspaceId             │
│                                                   │
│ { to: 'client', clientId: 'xyz' }                │
│   → Single client                                 │
│                                                   │
│ { to: 'workspace', exclude: 'clientId' }          │
│   → Workspace clients except one                  │
└───────────────────────────────────────────────────┘
```

### Per-Client Sequence Numbers

```
Server maintains per-client sequence counter:

Client A connects:
  event seq=1: { channel: "session:event", args: [...] }
  event seq=2: { channel: "session:event", args: [...] }
  event seq=3: { channel: "session:event", args: [...] }

Client B connects:
  event seq=1: { channel: "sources:changed", args: [...] }
  event seq=2: { channel: "session:event", args: [...] }
```

### Event Buffer

Each client has a ring buffer for event replay:

```
┌───────────────────────────────────────────────────┐
│ Per-Client Event Buffer                            │
│                                                   │
│ Max size: 100 events                              │
│ TTL: 5 minutes                                    │
│                                                   │
│ Eviction policy:                                  │
│   1. Events older than TTL → drop                 │
│   2. Buffer exceeds max size → drop oldest        │
│                                                   │
│ Acknowledgment:                                   │
│   Client sends sequence_ack every 30s             │
│   Server evicts acknowledged events               │
└───────────────────────────────────────────────────┘
```

### Typed Events

```typescript
// Compile-time validation via BroadcastEventMap
interface BroadcastEventMap {
  'session:event': { sessionId: string; type: string; /* ... */ }
  'session:unread_summary_changed': { /* ... */ }
  'session:files_changed': { files: string[] }
  'sources:changed': { workspaceId: string }
  'labels:changed': { workspaceId: string }
  'statuses:changed': { workspaceId: string }
  'automations:changed': { workspaceId: string }
  'skills:changed': { workspaceId: string }
  'theme:app_changed': { theme: ThemeOverrides }
  'theme:system_changed': { isDark: boolean }
  'update:available': { version: string }
  'update:download_progress': { percent: number }
  'browserPane:state_changed': { instanceId: string; /* ... */ }
  'notification:navigate': { workspaceId: string; sessionId: string }
  // ... 20+ more
}

// pushTyped() validates at compile time
pushTyped(server, 'session:event', { to: 'workspace', workspaceId }, {
  sessionId: 'abc',
  type: 'text_delta',
  // Type error if fields mismatch
})
```

---

## 6. Reconnection

### Client Reconnection Flow

```
Client disconnects (network drop, server restart, sleep/wake)
    │
    ▼
1. Connection state → 'disconnected'
    │
    ▼
2. Exponential backoff reconnection:
    │  Attempt 1: wait 1s
    │  Attempt 2: wait 2s
    │  Attempt 3: wait 4s
    │  ...
    │  Max delay: 30s
    │  Reset after 10s stable connection
    ▼
3. Reconnect with identity:
    │  handshake {
    │    reconnectClientId: "previous-id",
    │    lastSeq: 42,
    │    ...
    │  }
    ▼
4. Server validates:
    │  Look up disconnected client (TTL: 60s)
    │  Match workspaceId + webContentsId
    │
    ├── Client found, buffer available:
    │   → Replay events from lastSeq+1
    │   → handshake_ack { reconnected: true, stale: false }
    │
    ├── Client found, buffer evicted:
    │   → Cannot replay
    │   → handshake_ack { reconnected: true, stale: true }
    │
    └── Client not found (expired):
        → Fresh connection
        → handshake_ack { reconnected: false }
    │
    ▼
5. Client handles result:
    │
    ├── Fresh connection:
    │   → Reload all sessions
    │   → Re-subscribe to events
    │
    ├── Reconnected, not stale:
    │   → Resume streaming
    │   → No data loss
    │
    └── Reconnected, stale:
        → Emit __transport:reconnected(true)
        → Reload sessions (may have missed events)
```

### Heartbeat

```
Server ──── ping ────▶ Client     (every 30s)
Client ◀─── pong ──── Server

Missed pongs tracked:
  1 missed: OK (network jitter)
  2 missed: Warning
  3 missed: Terminate connection → trigger reconnect
```

---

## 7. Channel Routing

### RoutedClient Architecture

```
Application code calls:
  electronAPI.sendMessage(sessionId, text)
      │
      ▼
  RoutedClient.invoke('sessions:sendMessage', sessionId, text)
      │
      ├── Check routing table:
      │   isLocalOnly('sessions:sendMessage')?
      │
      ├── YES → localClient.invoke(...)
      │   (embedded server at localhost)
      │
      └── NO  → workspaceClient.invoke(...)
          (local or remote, depending on active workspace)
```

### Channel Classification

```
LOCAL_ONLY (200+ channels):
┌───────────────────────────────────────────────────┐
│ Category            │ Examples                     │
├─────────────────────┼─────────────────────────────┤
│ Remote connectivity │ remote.TEST_CONNECTION       │
│ Workspace CRUD      │ workspaces.CREATE, .DELETE   │
│ Window management   │ window.GET_WORKSPACE, .SWITCH│
│ Native dialogs      │ file.OPEN_DIALOG             │
│ Browser pane        │ browserPane.CREATE, .NAVIGATE│
│ Auth/OAuth          │ auth.START, .COMPLETE        │
│ Theme               │ theme.GET, .ON_CHANGE        │
│ System              │ system.VERSIONS              │
│ Auto-update         │ update.CHECK, .INSTALL       │
│ Menu                │ menu.NEW_CHAT                │
│ Notifications       │ notification.SHOW            │
│ Power mgmt          │ power.PREVENT_IDLE           │
│ Badge               │ badge.DRAW                   │
│ Debug               │ debug.LOG                    │
└─────────────────────┴─────────────────────────────┘

REMOTE_ELIGIBLE (170+ channels):
┌───────────────────────────────────────────────────┐
│ Category            │ Examples                     │
├─────────────────────┼─────────────────────────────┤
│ Sessions            │ sessions.CREATE, .SEND_MESSAGE│
│ Sources             │ sources.CREATE, .GET_TOOLS   │
│ Credentials         │ credentials.GET, .SET        │
│ LLM connections     │ llmConnections.CREATE        │
│ Skills              │ skills.GET, .SAVE            │
│ Labels/Statuses     │ labels.LIST, statuses.REORDER│
│ Automations         │ automations.SAVE, .TEST      │
│ Files               │ file.READ, .WRITE            │
│ Git                 │ git.STATUS, .LOG             │
│ Settings            │ settings.SET_LLM, .SET_PROXY │
│ Resources           │ resources.EXPORT, .IMPORT    │
│ Transfer            │ transfer.EXPORT_CHUNKS       │
│ OAuth (source)      │ oauth.START, .COMPLETE       │
│ Server status       │ server.STATUS, .HEALTH       │
└─────────────────────┴─────────────────────────────┘
```

### Workspace Switching (Remote)

```
User switches to remote workspace
    │
    ▼
1. RoutedClient.invoke('window:switchWorkspace', workspaceId)
    │
    ▼
2. Handler returns remote server details:
    │  { workspaceId, remoteServer: { url, token, remoteWorkspaceId } }
    │
    ▼
3. RoutedClient performs client swap:
    │
    ├── Create new WsRpcClient for remote URL
    ├── Connect with bearer token
    ├── Map local workspace ID → remote workspace ID
    │
    ├── Make-before-break listener swap:
    │   1. Subscribe all REMOTE_ELIGIBLE listeners on new client
    │   2. Unsubscribe from old workspace client
    │   3. Swap workspaceClient reference
    │
    └── Emit synthetic __transport:reconnected event
```

---

## 8. Client Capabilities (Server → Client)

The server can request operations from the client:

```
Server needs to open a URL
    │
    ▼
1. Check client has capability: 'client:openExternal'
    │
    ▼
2. server.invokeClient(clientId, 'client:openExternal', [url])
    │
    ▼
3. Client receives request envelope
    │  { type: "request", channel: "client:openExternal", args: ["https://..."] }
    │
    ▼
4. Preload handler executes:
    │  shell.openExternal(url)
    │
    ▼
5. Client sends response:
    { type: "response", result: true }
```

### Available Capabilities

```
┌────────────────────────┬──────────────────────────────────┐
│ Capability             │ Action                           │
├────────────────────────┼──────────────────────────────────┤
│ client:openExternal    │ Open URL in default browser      │
│ client:openPath        │ Open file with default app       │
│ client:showItemInFolder│ Reveal file in Finder/Explorer   │
│ client:confirmDialog   │ Show native confirm dialog       │
│ client:openFileDialog  │ Show file/folder picker          │
└────────────────────────┴──────────────────────────────────┘
```

---

## 9. API Builder Pattern

### From Channels to electronAPI

The preload bridge auto-generates the `electronAPI` from a channel map:

```typescript
// Channel map (single source of truth)
const CHANNEL_MAP = {
  getSessions: { type: 'invoke', channel: 'sessions:GET' },
  sendMessage: { type: 'invoke', channel: 'sessions:SEND_MESSAGE' },
  onSessionEvent: { type: 'listener', channel: 'session:event' },
  getWorkspaces: { type: 'invoke', channel: 'server:GET_WORKSPACES' },
  // ... 100+ mappings
}

// Build typed API proxy
const electronAPI = buildClientApi(CHANNEL_MAP, routedClient)

// Usage in renderer:
electronAPI.getSessions(workspaceId)       // → invoke('sessions:GET', workspaceId)
electronAPI.sendMessage(sessionId, text)   // → invoke('sessions:SEND_MESSAGE', ...)
electronAPI.onSessionEvent(callback)       // → on('session:event', callback)
```

### Invoke Transform

Some channels need result transformation:

```typescript
createSession: {
  type: 'invoke',
  channel: 'sessions:CREATE',
  transform: (result) => result.sessionId  // Extract ID from response
}
```

---

## 10. RPC Handler Registration

### Handler Categories

```
registerCoreRpcHandlers(server, deps)
    │
    ├── Session handlers
    │   sessions:GET          → list sessions
    │   sessions:GET_MESSAGES → load session with messages
    │   sessions:CREATE       → create new session
    │   sessions:DELETE       → delete session
    │   sessions:SEND_MESSAGE → start agent processing
    │   sessions:EXPORT       → export as bundle
    │   sessions:IMPORT       → import bundle
    │
    ├── Source handlers
    │   sources:CREATE        → create source
    │   sources:DELETE        → delete source
    │   sources:GET_TOOLS     → get MCP tools for source
    │   sources:OAUTH_START   → start OAuth flow
    │
    ├── LLM connection handlers
    │   llmConnections:CREATE → add LLM connection
    │   llmConnections:TEST   → test connection
    │   llmConnections:LIST   → list connections
    │
    ├── Workspace handlers (local only)
    │   server:GET_WORKSPACES → list workspaces
    │   server:CREATE_WORKSPACE → create workspace
    │
    ├── File handlers
    │   file:READ             → read file content
    │   file:READ_DATA_URL    → read as data URL
    │   file:OPEN_DIALOG      → native file picker
    │
    ├── Credential handlers
    │   credentials:GET       → get credential
    │   credentials:SET       → store credential
    │
    ├── Settings handlers
    │   settings:SET_LLM      → set default LLM
    │   settings:SET_PROXY    → configure proxy
    │
    └── ... 15+ more categories
```

### Dependency Injection

```typescript
// Handler deps are injected, shared by Electron and headless
interface HandlerDeps {
  sessionManager: SessionManagerInterface
  platform: {
    logger: Logger
    captureError?: (error: Error) => void
  }
  windowManager?: WindowManagerInterface      // Electron only
  browserPaneManager?: BrowserPaneManagerInterface  // Electron only
  oauthFlowStore: OAuthFlowStore
}
```

---

## 11. Performance & Reliability

### Throughput Characteristics

```
┌───────────────────────────┬──────────────────────────┐
│ Metric                    │ Value                    │
├───────────────────────────┼──────────────────────────┤
│ Max concurrent clients    │ 50                       │
│ Request timeout           │ 60s (server-side)        │
│ Client timeout            │ 30s (client-side)        │
│ Handshake timeout         │ 5s                       │
│ Heartbeat interval        │ 30s                      │
│ Max missed heartbeats     │ 3 (90s total)            │
│ Event buffer per client   │ 100 events               │
│ Event buffer TTL          │ 5 minutes                │
│ Disconnected client TTL   │ 60 seconds               │
│ Max disconnected clients  │ 50                       │
│ Sequence ack interval     │ 30 seconds               │
│ Reconnect backoff         │ 1s → 2s → 4s → ... 30s  │
└───────────────────────────┴──────────────────────────┘
```

### Memory Management

```
Per-client memory:
  Connection state:      ~1 KB
  Event buffer (100):    ~100 KB (depends on event size)
  Pending requests:      ~1 KB
  ─────────────────────────────
  Total per client:      ~102 KB

50 clients:              ~5 MB
50 disconnected clients: ~5 MB (buffered events)
─────────────────────────────────
Total server overhead:   ~10 MB
```

---

## 12. Security

### Authentication

```
Connection auth:
┌─────────────────────┬─────────────────────────────────┐
│ Mode                │ Validation                      │
├─────────────────────┼─────────────────────────────────┤
│ Bearer token        │ Constant-time string compare    │
│ JWT session cookie  │ Verify signature + expiry       │
│ Local (127.0.0.1)   │ No auth required                │
└─────────────────────┴─────────────────────────────────┘
```

### TLS

```
Server TLS configuration:
  cert: fs.readFileSync('cert.pem')
  key: fs.readFileSync('key.pem')
  ca: fs.readFileSync('ca.pem')        // optional
  passphrase: 'secret'                  // optional

Client TLS:
  Node.js: rejectUnauthorized option
  Browser: Native WebSocket (no self-signed support)
```

### Non-Local Bind Protection

```
Server binds to 0.0.0.0 without TLS:
  → Refuses to start
  → --allow-insecure-bind override for trusted networks

Reason: Prevents cleartext token transmission on network interfaces
```

---

## 13. Health Checks

### HTTP Health Endpoint

```
GET /health HTTP/1.1

Response:
{
  "status": "ok" | "degraded" | "unhealthy",
  "checks": [
    { "name": "session_manager", "status": "ok" },
    { "name": "memory", "status": "ok", "used_mb": 150 }
  ],
  "uptime_seconds": 3600
}
```

### Diagnostic State

```
Client connection state:
  idle → connecting → connected → disconnected
                                    ↓
                                 reconnecting
                                    ↓
                              connected (resume)

Error classification:
  auth | protocol | timeout | network | server | unknown
```
