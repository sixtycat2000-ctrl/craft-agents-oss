# Deployment & Operations — Server, CLI, WebUI, and Monitoring

> How Craft Agents runs as a headless server, serves the WebUI, manages the CLI client, handles health monitoring, and supports Docker deployments.

---

## 1. Deployment Modes

```
┌───────────────────────────────────────────────────────────────┐
│                   Deployment Options                          │
│                                                               │
│  ┌──────────────────┐  ┌──────────────────┐  ┌────────────┐ │
│  │  Desktop App     │  │  Headless Server │  │  CLI Client│ │
│  │  (Electron)      │  │  (Bun binary)    │  │  (Terminal)│ │
│  │                  │  │                  │  │            │ │
│  │  GUI + embedded  │  │  WebSocket RPC   │  │  JSON out  │ │
│  │  server          │  │  + WebUI         │  │  streaming │ │
│  │                  │  │  + health check  │  │            │ │
│  │  Single user     │  │  Multi-client    │  │  Single    │ │
│  │  Local only      │  │  Remote capable  │  │  session   │ │
│  └──────────────────┘  └──────────────────┘  └────────────┘ │
│                                                               │
│  ┌──────────────────┐  ┌──────────────────────────────────┐  │
│  │  Docker          │  │  Thin Client (Electron + remote) │  │
│  │  (containerized) │  │  (GUI connects to headless)      │  │
│  └──────────────────┘  └──────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────┘
```

---

## 2. Headless Server

### Architecture

```
packages/server/src/index.ts
    │
    ├── Parse environment variables
    ├── Validate configuration
    ├── Bootstrap server
    │   ├── Create WsRpcServer
    │   ├── Initialize SessionManager
    │   ├── Register RPC handlers
    │   ├── Set up ConfigWatcher
    │   ├── Start model refresh service
    │   └── Optional: HTTP handler for WebUI
    │
    ├── Signal handling:
    │   SIGINT, SIGTERM → graceful shutdown
    │
    └── Process lifecycle:
        Start → serve → drain → cleanup → exit
```

### Environment Configuration

```
Required:
  CRAFT_SERVER_TOKEN          # Bearer token for auth
                              # Min 16 chars, 192-bit recommended

Network:
  CRAFT_RPC_HOST=127.0.0.1   # Bind address
  CRAFT_RPC_PORT=9100        # Bind port

TLS:
  CRAFT_RPC_TLS_CERT         # Certificate file path
  CRAFT_RPC_TLS_KEY          # Private key file path
  CRAFT_RPC_TLS_CA           # CA certificate (optional)
  CRAFT_RPC_TLS_PASSPHRASE   # Key passphrase (optional)

WebUI:
  CRAFT_WEBUI_DIR            # Static assets directory
  CRAFT_WEBUI_PASSWORD       # Shorter password for web access
  CRAFT_HEALTH_PORT          # HTTP health endpoint port

General:
  CRAFT_DEBUG                # Enable debug logging
  CRAFT_LOG_LEVEL            # Log level (debug/info/warn/error)
```

### Startup Sequence

```
Server starts:
    │
    ├── 1. Validate token:
    │   Min 16 characters
    │   Entropy check (reject single-char repeats)
    │   Warn on low uniqueness
    │
    ├── 2. Check bind address:
    │   Non-localhost without TLS?
    │   → Refuse (unless --allow-insecure-bind)
    │
    ├── 3. Acquire lock file:
    │   ~/.craft-agent/.server.lock
    │   Check for stale lock (PID + startedAt)
    │   Clean up stale locks from crashes
    │
    ├── 4. Create WebSocket server:
    │   Configure TLS if cert/key provided
    │   Set max clients (default: 50)
    │   Register message handlers
    │
    ├── 5. Initialize SessionManager:
    │   Load workspace configs
    │   Initialize MCP pool
    │   Set up ConfigWatcher
    │
    ├── 6. Register RPC handlers:
    │   Sessions, sources, credentials, settings, etc.
    │
    ├── 7. Start HTTP handler (if WebUI configured):
    │   Serve static assets
    │   Handle OAuth callbacks
    │   Health check endpoint
    │
    ├── 8. Start model refresh service:
    │   Periodic model list refresh for LLM connections
    │
    └── 9. Listen:
        Log: "Server listening on {host}:{port}"
        Ready for connections
```

### Graceful Shutdown

```
SIGINT/SIGTERM received:
    │
    ├── 1. Stop accepting new connections
    │
    ├── 2. Drain period (2 seconds):
    │   Let pending requests complete
    │   Flush session saves
    │
    ├── 3. Cleanup:
    │   Close MCP connections
    │   Stop file watchers
    │   Flush persistence queue
    │   Close WebSocket connections
    │
    ├── 4. Release lock file:
    │   Delete ~/.craft-agent/.server.lock
    │
    └── 5. Exit process
```

---

## 3. Lock File Management

### Server Lock

```
~/.craft-agent/.server.lock:
    {
      "pid": 12345,
      "startedAt": "2025-04-18T10:00:00Z"
    }

Purpose:
  Prevent multiple servers from running simultaneously

Stale detection:
  1. Read lock file
  2. Check if PID is running (process.kill(pid, 0))
  3. If PID not running → stale lock → clean up
  4. If PID running but startedAt is old → warn

Legacy format:
  Some old versions wrote just the PID (plain number)
  Detected and handled for backward compatibility
```

---

## 4. WebUI

### Architecture

```
WebUI serves a browser-based interface from the headless server:

                    Browser
                      │
                      ▼
              ┌───────────────┐
              │ HTTP Handler  │
              │ (same port)   │
              └───────┬───────┘
                      │
            ┌─────────┼─────────┐
            │         │         │
      ┌─────▼──┐ ┌───▼────┐ ┌──▼──────────┐
      │ Static │ │ OAuth  │ │ Health      │
      │ Assets │ │Callback│ │ Check       │
      │ (SPA)  │ │ Handler│ │ /health     │
      └────────┘ └────────┘ └─────────────┘
            │
            │ WebSocket upgrade
            ▼
      ┌───────────────┐
      │ WsRpcServer   │
      │ (same port)   │
      └───────────────┘
```

### HTTP Handler

```
HTTP requests on the same port as WebSocket:

GET /                         → Serve index.html (SPA)
GET /assets/*                 → Serve static assets
GET /health                   → Health check JSON
GET /auth/callback*           → OAuth callback handler

WebSocket upgrade:
  Upgrade: websocket header → delegate to WsRpcServer
  JWT cookie validation on upgrade request
```

### WebUI Authentication

```
Two auth paths for WebUI:

Path 1: WebUI password
  CRAFT_WEBUI_PASSWORD="short-password"
  POST /auth/login { password }
  → Set JWT session cookie
  → Redirect to /

Path 2: Server token
  Use CRAFT_SERVER_TOKEN as password
  Same flow

JWT cookie:
  Signed with HMAC-SHA256
  Contains: { sessionId, expiresAt }
  Expires after configurable time
  Validated on every WebSocket upgrade
```

### WebUI Assets

```
apps/webui/:
    Built React app (Vite production build)
    Contains:
      index.html
      assets/*.js
      assets/*.css

Build:
  cd apps/webui && bun run build
  Output: dist/

Serve:
  CRAFT_WEBUI_DIR=path/to/dist
  Server serves these assets via HTTP handler
```

---

## 5. CLI Client

### Architecture

```
apps/cli/src/index.ts:
    Terminal client for server interaction

Modes:
  1. Connect to existing server:
     craft-agent --server wss://host:port --token xxx

  2. Spawn embedded server:
     craft-agent run
     → Starts headless server in background
     → Connects to it via WebSocket

  3. Single-shot command:
     craft-agent "Summarize this file" --file readme.md
```

### CLI Commands

```
┌──────────────────────────────────────────────────────────┐
│ Command          │ Description                           │
├──────────────────┼───────────────────────────────────────┤
│ run              │ Start embedded server + interactive   │
│ chat             │ Connect to running server             │
│ sessions list    │ List all sessions                     │
│ sessions get     │ Get session details                   │
│ sessions create  │ Create new session                    │
│ sessions delete  │ Delete session                        │
│ send             │ Send message to session               │
│ sources list     │ List configured sources               │
│ workspaces list  │ List workspaces                       │
│ health           │ Check server health                   │
└──────────────────┴───────────────────────────────────────┘
```

### Output Formats

```
Default: Human-readable formatted output
  ┌──────────────────────────────────────────────┐
  │ Session: 250418-swift-eagle                  │
  │ Status:  active                              │
  │ Model:   claude-sonnet-4-6                   │
  │ Messages: 12                                 │
  │ Created: 2025-04-18 10:00:00                 │
  └──────────────────────────────────────────────┘

JSON: --json flag for scripting
  {
    "id": "250418-swift-eagle",
    "status": "active",
    "model": "claude-sonnet-4-6",
    "messageCount": 12
  }

Streaming: --stream flag for real-time output
  [user]     Summarize this code
  [assistant] I'll analyze the code...
  [tool]     Read src/main.ts ✓
  [assistant] The code implements...
```

---

## 6. Docker Deployment

### Dockerfile Pattern

```dockerfile
FROM oven/bun:1

WORKDIR /app

# Copy package files
COPY package.json bun.lock ./
COPY packages/ packages/

# Install dependencies
RUN bun install --frozen-lockfile

# Build server
RUN cd packages/server && bun build src/index.ts --outdir dist --target bun

# Build WebUI
RUN cd apps/webui && bun run build

# Environment defaults
ENV CRAFT_RPC_HOST=0.0.0.0
ENV CRAFT_RPC_PORT=9100

# Expose port
EXPOSE 9100

# Run server
CMD ["bun", "run", "packages/server/dist/index.js"]
```

### Docker Compose

```yaml
version: '3.8'
services:
  craft-agent:
    build: .
    ports:
      - "9100:9100"
    environment:
      CRAFT_SERVER_TOKEN: ${SERVER_TOKEN}
      CRAFT_RPC_HOST: 0.0.0.0
      CRAFT_RPC_PORT: 9100
      CRAFT_WEBUI_DIR: /app/apps/webui/dist
    volumes:
      - craft-data:/root/.craft-agent

volumes:
  craft-data:
```

---

## 7. Health Monitoring

### Health Endpoint

```
GET /health HTTP/1.1

Response:
{
  "status": "ok",
  "uptime_seconds": 3600,
  "version": "0.8.9",
  "checks": [
    {
      "name": "session_manager",
      "status": "ok",
      "details": {
        "active_sessions": 5,
        "workspaces": 2
      }
    },
    {
      "name": "memory",
      "status": "ok",
      "details": {
        "heap_used_mb": 150,
        "heap_total_mb": 256,
        "rss_mb": 300
      }
    },
    {
      "name": "connections",
      "status": "ok",
      "details": {
        "active_clients": 3,
        "max_clients": 50
      }
    }
  ]
}

Status values:
  ok       → All checks passing
  degraded → Some checks degraded (high memory, many clients)
  unhealthy → Critical checks failing
```

### Monitoring Integration

```
Health check for load balancers:
  Interval: 10 seconds
  Timeout: 5 seconds
  Healthy: status == "ok" or status == "degraded"
  Unhealthy: status == "unhealthy" or connection refused

Prometheus metrics (future):
  craft_active_sessions
  craft_active_clients
  craft_memory_heap_used_bytes
  craft_rpc_requests_total
  craft_rpc_request_duration_seconds
```

---

## 8. Model Refresh Service

### Purpose

```
LLM providers periodically update their model catalogs.
The model refresh service keeps local model lists current.

Operation:
  Every 5 minutes:
    For each LLM connection:
      ├── Query provider API for model list
      ├── Compare with cached list
      ├── Update cache if changed
      └── Emit event if models changed

  Benefits:
    - New models appear in UI automatically
    - Deprecated models removed
    - Pricing updates reflected
```

### Implementation

```
initModelRefreshService(credentialGetter):
    │
    ├── Schedule periodic refresh:
    │   setInterval(refreshAll, 5 * 60 * 1000)
    │
    ├── For each connection:
    │   ├── Get credentials:
    │   │   apiKey = await credentialGetter(slug)
    │   ├── Query models:
    │   │   GET /v1/models (provider-specific)
    │   ├── Parse response:
    │   │   Extract model IDs, names, pricing
    │   └── Update cache:
    │       modelsCache.set(slug, modelList)
    │
    └── Error handling:
        Network error → keep cached list
        Auth error → log warning
        Rate limit → backoff and retry
```

---

## 9. Remote Workspace Configuration

### Remote Server Setup

```
Connect Electron to remote server:

1. User adds remote workspace:
   URL: wss://craft.example.com:9100
   Token: xxxxxx
   Remote workspace ID: abc-123

2. Electron stores connection config:
   {
     localWorkspaceId: "local-xyz",
     remoteServer: {
       url: "wss://craft.example.com:9100",
       token: "xxxxxx",
       remoteWorkspaceId: "abc-123"
     }
   }

3. On workspace switch:
   RoutedClient creates WsRpcClient for remote
   Workspace ID mapped: local → remote
   Traffic routed: REMOTE_ELIGIBLE → remote server
```

### Multi-Remote Support

```
User can connect to multiple remote servers:

  Workspace A → Local embedded server
  Workspace B → Remote server (staging)
  Workspace C → Remote server (production)

Each workspace has independent:
  Sessions, sources, credentials, automations
  Only one active at a time per window
  Multiple windows can use different remotes
```

---

## 10. Token Generation

### Server Token Generation

```
generateServerToken():
    crypto.randomBytes(24).toString('hex')
    → 48-character hex string (192 bits of entropy)

Usage:
  $ craft-agent --generate-token
  Generated token: a1b2c3d4e5f6... (48 chars)

  $ CRAFT_SERVER_TOKEN=a1b2c3d4... craft-agent serve

Security:
  Sufficient entropy for production use
  Timing-safe comparison on validation
  No rate limiting on auth (single comparison per connection)
```

---

## 11. Operational Procedures

### Server Startup Check

```
Before starting server:
  1. Check lock file → clean if stale
  2. Check port availability
  3. Validate TLS certs (if configured)
  4. Validate token strength
  5. Check disk space for sessions
  6. Load workspace configs
  7. Start listening
```

### Session Recovery

```
After server restart:
  1. Scan workspace session directories
  2. Load session headers (fast)
  3. Find sessions mid-processing:
     processingStatus: 'processing' or 'streaming'
  4. Mark as 'interrupted':
     Agent was stopped mid-turn
  5. User sees interrupted state → can retry
```

### Log Management

```
Log levels:
  debug   → All operations, tool calls, events
  info    → Server lifecycle, connections, sessions
  warn    → Non-critical errors, retries, deprecated usage
  error   → Failures, crashes, credential errors

Log output:
  Desktop: electron-log (file + console)
  Server:  stdout + stderr
  CLI:     stderr (JSON mode: structured logs)

Log rotation (desktop):
  Max file size: 5MB
  Max files: 5
  Location: ~/Library/Logs/Craft Agents/ (macOS)
```

---

## 12. Build & Distribution

### Electron Build

```
Build targets:
  macOS:  .dmg (Intel + Apple Silicon)
  Windows: .exe (NSIS installer)
  Linux:  .AppImage

Build pipeline:
  1. esbuild: main process + preload
  2. Vite: renderer (React app)
  3. electron-builder: package + sign + notarize

Code signing:
  macOS: Apple Developer ID certificate
  Windows: Code signing certificate
  Auto-update requires valid signatures
```

### Server Build

```
Build pipeline:
  1. TypeScript compile (tsc)
  2. Bun bundle (optional, for single-binary distribution)

Output:
  Single executable or directory with all dependencies
```

---

## 13. Performance Baselines

```
┌──────────────────────────────────┬──────────────────────────────┐
│ Metric                           │ Value                        │
├──────────────────────────────────┼──────────────────────────────┤
│ Server startup time              │ 2-5 seconds                  │
│ Session creation                 │ 50-100ms                     │
│ Session listing (300 sessions)   │ 100-200ms                    │
│ Session full load                │ 200-500ms                    │
│ RPC round-trip (local)           │ 5-15ms                       │
│ RPC round-trip (remote, same DC) │ 20-50ms                      │
│ WebSocket message throughput     │ 10,000+ msg/sec              │
│ Event buffer memory (per client) │ ~100KB                       │
│ Health check response time       │ <10ms                        │
│ Graceful shutdown time           │ 2-3 seconds                  │
└──────────────────────────────────┴──────────────────────────────┘
```
