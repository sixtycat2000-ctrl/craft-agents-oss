# Sources & MCP — External Tool Integration

> How Craft Agents connects to external data sources (MCP servers, REST APIs, local files), manages credentials, and exposes tools to the agent runtime.

---

## 1. Source Types

```
┌──────────────────────────────────────────────────────────┐
│                    Source System                          │
│                                                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │ MCP Server   │  │  REST API    │  │ Local Files  │  │
│  │              │  │              │  │              │  │
│  │ stdio/SSE/   │  │ Google APIs  │  │ Directories  │  │
│  │ HTTP         │  │ Slack, MSFT  │  │ Obsidian     │  │
│  │              │  │ Custom REST  │  │ Git repos    │  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  │
│         │                 │                  │          │
│         ▼                 ▼                  ▼          │
│  ┌──────────────────────────────────────────────────┐  │
│  │              MCP Client Pool                     │  │
│  │  (connection pooling + health monitoring)        │  │
│  └──────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

---

## 2. MCP Source Configuration

### Transport Types

```
MCP Server Transport:

┌─────────────────────────────────────────────────────┐
│ stdio                                                │
│   command: "bun"                                     │
│   args: ["run", "craft-document-tools"]              │
│   env: { API_KEY: "..." }                            │
│                                                      │
│   Process lifecycle:                                 │
│   - Spawned on first tool call                       │
│   - Kept alive via MCP pool                          │
│   - Killed after idle timeout                        │
├─────────────────────────────────────────────────────┤
│ sse (Server-Sent Events)                             │
│   url: "https://mcp.example.com/events"              │
│   headers: { Authorization: "Bearer ..." }           │
│                                                      │
│   Connection lifecycle:                              │
│   - Connect on first tool call                       │
│   - Reconnect on disconnect                          │
├─────────────────────────────────────────────────────┤
│ http (Streamable HTTP)                               │
│   url: "https://mcp.example.com/mcp"                 │
│   authType: "bearer"                                 │
│   headers: { ... }                                   │
│                                                      │
│   Request/response per tool call                     │
└─────────────────────────────────────────────────────┘
```

### Source Directory Layout

```
{workspaceRootPath}/sources/{source-slug}/
├── config.json          # Source configuration
├── guide.md             # Usage guide (markdown with embedded cache)
├── icon.svg             # Source icon (svg, png, jpg, or jpeg)
└── permissions.json     # Source-specific permissions
```

### Config.json Schema

```json
{
  "id": "source-uuid-123",
  "slug": "linear-api",
  "name": "Linear",
  "type": "mcp",
  "transport": {
    "type": "stdio",
    "command": "bun",
    "args": ["run", "@anthropic/mcp-linear"],
    "env": {}
  },
  "enabled": true,
  "createdAt": "2025-04-18T10:00:00Z",
  "guideCache": {
    "sections": [
      { "title": "Overview", "content": "..." },
      { "title": "Available Tools", "content": "..." }
    ],
    "cachedAt": "2025-04-18T10:00:00Z"
  }
}
```

---

## 3. REST API Source Configuration

### API Source Types

```
┌───────────────────────────────────────────────────────────┐
│ API Source Configuration                                   │
│                                                           │
│ authType:                                                 │
│   none          → No authentication                       │
│   bearer        → Bearer token in Authorization header    │
│   apikey        → API key in header or query parameter    │
│   basic         → HTTP Basic auth                         │
│   oauth         → OAuth 2.0 flow                         │
│                                                           │
│ OAuth providers:                                           │
│   google        → Gmail, Calendar, Drive, YouTube, etc.  │
│   slack         → Slack Web API                          │
│   microsoft     → Microsoft Graph API                    │
│   generic       → Custom OAuth 2.0 server                │
│                                                           │
│ Tool generation:                                          │
│   Auto-generates tools from API spec                     │
│   Each endpoint → one tool                                │
│   Parameters from spec → tool arguments                   │
└───────────────────────────────────────────────────────────┘
```

### API Config Example

```json
{
  "id": "api-source-456",
  "slug": "gmail-api",
  "name": "Gmail",
  "type": "api",
  "api": {
    "baseUrl": "https://gmail.googleapis.com/gmail/v1",
    "authType": "oauth",
    "oauth": {
      "provider": "google",
      "scopes": ["https://www.googleapis.com/auth/gmail.readonly"]
    },
    "testEndpoint": "/users/me/profile",
    "renewEndpoint": null
  },
  "enabled": true
}
```

### API Tool Generation

```
API source "Gmail" loaded
    │
    ▼
api-tools.ts generates tools:
    │
    ├── gmail_listMessages(userId, maxResults, query)
    │   → GET /users/{userId}/messages?maxResults=...&q=...
    │
    ├── gmail_getMessage(userId, id)
    │   → GET /users/{userId}/messages/{id}
    │
    ├── gmail_listLabels(userId)
    │   → GET /users/{userId}/labels
    │
    └── gmail_getAttachment(userId, messageId, id)
        → GET /users/{userId}/messages/{messageId}/attachments/{id}
```

---

## 4. Local File Source

### Configuration

```json
{
  "id": "local-source-789",
  "slug": "my-documents",
  "name": "My Documents",
  "type": "local",
  "local": {
    "path": "/Users/yul/Documents",
    "watchForChanges": true,
    "recursive": true,
    "fileExtensions": [".md", ".txt", ".pdf", ".docx"]
  }
}
```

### Local Source Features

```
Local file sources provide:
├── File listing (recursive, filtered by extensions)
├── File content reading (text extraction)
├── File watching (notify agent on changes)
├── Search (full-text search within directory)
└── Metadata (size, modified date, type)
```

---

## 5. MCP Client Pool

### Architecture

```
SessionManager
    │
    ▼
McpPool
├── Map<poolKey, PoolClient>
│   ├── "linear-api:session-1" → McpClient (connected)
│   ├── "github-api:session-1" → McpClient (connected)
│   ├── "craft-docs:session-2" → McpClient (connected)
│   └── ...
│
├── getClient(sourceSlug, sessionId)
│   ├── Check pool for existing client
│   ├── If not found → create and connect
│   ├── If disconnected → reconnect
│   └── Return client
│
├── executeTool(sourceSlug, sessionId, toolName, args)
│   ├── Get client from pool
│   ├── Call client.callTool(toolName, args)
│   └── Return result
│
└── cleanup(sessionId?)
    ├── Close all clients for session
    └── Or close all clients (shutdown)
```

### Pool Client Types

```
┌─────────────────────────────────────────────────────┐
│ PoolClient                                           │
│                                                      │
│ McpPoolClient (MCP sources)                          │
│   ├── sourceConfig: SourceConfig                     │
│   ├── client: CraftMcpClient                        │
│   ├── connected: boolean                            │
│   ├── toolDefinitions: Tool[]                       │
│   └── lastActivity: number                          │
│                                                      │
│ ApiSourcePoolClient (API sources)                    │
│   ├── sourceConfig: SourceConfig                     │
│   ├── tools: ApiTool[]                              │
│   ├── credentialManager: SourceCredentialManager    │
│   └── tokenRefreshManager: TokenRefreshManager      │
└─────────────────────────────────────────────────────┘
```

### Connection Lifecycle

```
Source enabled for session
    │
    ▼
1. McpPool.getClient(sourceSlug, sessionId)
    │
    ├── Pool hit: return existing client
    │   └── Reconnect if disconnected
    │
    └── Pool miss: create new client
        │
        ▼
2. Create client based on source type:
    │
    ├── MCP (stdio):
    │   ├── Spawn subprocess: Bun.spawn(command, args, env)
    │   ├── Wait for initialization
    │   ├── List tools via MCP protocol
    │   └── Store in pool
    │
    ├── MCP (sse/http):
    │   ├── Connect to URL
    │   ├── Authenticate with headers
    │   ├── List tools via MCP protocol
    │   └── Store in pool
    │
    └── API:
        ├── Generate tools from spec
        ├── Load credentials
        ├── Set up token refresh
        └── Store in pool (no connection needed)
    │
    ▼
3. Client available for tool execution
    │
    ├── Agent calls tool → pool routes to correct client
    │
    └── Session ends → cleanup clients for that session
```

---

## 6. Credential Management

### Source Credential Manager

```
SourceCredentialManager
├── Credential Types:
│   ├── oauth          → OAuth access/refresh tokens
│   ├── bearer         → Bearer token
│   ├── apikey         → API key
│   ├── basic          → Username + password
│   └── header         → Custom header value
│
├── Operations:
│   ├── getCredential(sourceId) → Credential
│   ├── setCredential(sourceId, credential) → void
│   ├── deleteCredential(sourceId) → void
│   └── refreshToken(sourceId) → Credential
│
└── Storage:
    All credentials in ~/.craft-agent/credentials.enc
    Key format: source_{type}::{workspaceId}::{sourceId}
```

### Credential Flow for API Sources

```
Agent calls gmail_listMessages()
    │
    ▼
1. ApiSourcePoolClient intercepts
    │
    ▼
2. Look up credential for "gmail-api" source
    │  SourceCredentialManager.getCredential(sourceId)
    │  → Encrypted store → decrypt → return OAuth token
    │
    ▼
3. Check token expiry:
    │
    ├── Token valid → use it
    │
    └── Token expired:
        │
        ▼
        TokenRefreshManager.refresh(sourceId)
        │
        ├── Google: call token endpoint with refresh_token
        │   → New access_token + optional new refresh_token
        │
        ├── Slack: same pattern
        │
        └── Generic: call renewEndpoint
            → Extract new token from response
        │
        ▼
        Save refreshed credential → continue request
```

---

## 7. OAuth Token Refresh

### Token Refresh Manager

```
TokenRefreshManager
├── Tracks refreshable sources
│   isRefreshableSource(source):
│     - Has OAuth credentials with refresh_token, OR
│     - Has api.renewEndpoint configured
│
├── Auto-refresh logic:
│   - Check on every tool call
│   - Refresh if expired or within 5 min of expiry
│   - 5-minute cooldown between refresh attempts
│   - Queue concurrent refreshes (dedup)
│
└── Refresh methods:
    ├── refreshOAuth(source) → standard OAuth2 refresh
    │   POST /token { grant_type: refresh_token, ... }
    │
    └── refreshRenew(source) → custom renew endpoint
        POST renewEndpoint with current token
```

### OAuth Providers

```
┌──────────────┬────────────────────────────┬──────────────────┐
│ Provider     │ Auth Flow                  │ Token Refresh    │
├──────────────┼────────────────────────────┼──────────────────┤
│ Google       │ OAuth 2.0 + PKCE           │ Standard refresh │
│ Slack        │ OAuth 2.0 (code flow)      │ Standard refresh │
│ Microsoft    │ OAuth 2.0 + PKCE           │ Standard refresh │
│ Generic      │ Configurable               │ Custom endpoint  │
│ Claude       │ OAuth 2.0 + PKCE           │ Standard refresh │
│ ChatGPT      │ Device code flow           │ Manual re-auth   │
│ GitHub       │ OAuth 2.0 (device code)    │ Standard refresh │
└──────────────┴────────────────────────────┴──────────────────┘
```

---

## 8. Source Server Builder

### MCP Server Spawning

```
server-builder.ts creates MCP servers from source config:

buildMcpServer(sourceConfig)
    │
    ├── stdio transport:
    │   ├── Build command: [config.command, ...config.args]
    │   ├── Build env: { ...process.env, ...config.env }
    │   ├── Spawn child process
    │   ├── Create MCP client over stdio
    │   └── Return client
    │
    ├── sse transport:
    │   ├── Build URL: config.url
    │   ├── Build headers: config.headers + auth
    │   ├── Create EventSource connection
    │   ├── Create MCP client over SSE
    │   └── Return client
    │
    └── http transport:
        ├── Build URL: config.url
        ├── Build headers: config.headers + auth
        ├── Create HTTP MCP client
        └── Return client
```

### Environment Variable Injection

```
Source config:
  env: {
    LINEAR_API_KEY: "{{credential}}",    // Resolved from credential store
    WORKSPACE_ROOT: "{{workspacePath}}",  // Resolved from workspace
    NODE_ENV: "production"                // Literal value
  }

Resolution:
  {{credential}} → look up source credential → inject actual key
  {{workspacePath}} → workspace root directory
  Everything else → pass through as-is
```

---

## 9. Built-in Sources

### Craft Document Tools

```
Craft Agents includes a built-in MCP source for Craft documents:

Source: craft-document-tools
Type: MCP (stdio)
Command: Built-in Craft document tools binary

Tools provided (32+):
├── Document operations
│   ├── craft_search_documents(query, spaceId?)
│   ├── craft_get_document(docId)
│   ├── craft_create_document(title, content, spaceId?)
│   └── craft_update_document(docId, content)
│
├── Block operations
│   ├── craft_add_block(docId, blockType, content, afterBlockId?)
│   ├── craft_update_block(docId, blockId, content)
│   └── craft_delete_block(docId, blockId)
│
├── Media operations
│   ├── craft_upload_image(filePath, docId?)
│   └── craft_embed_url(url, docId, blockId)
│
├── Space operations
│   ├── craft_list_spaces()
│   ├── craft_get_space(spaceId)
│   └── craft_create_space(name)
│
└── Export operations
    ├── craft_export_document(docId, format)
    └── craft_export_to_markdown(docId)
```

---

## 10. Source Guide System

### Guide Format

```
guide.md contains usage instructions for the agent:

---
sections:
  - title: Overview
    content: |
      This source connects to Linear for project management.
      Use it to create issues, track progress, and manage sprints.
  - title: Available Tools
    content: |
      - linear_create_issue: Create a new issue
      - linear_update_issue: Update an existing issue
      - linear_search_issues: Search issues by query
  - title: Best Practices
    content: |
      Always specify a team when creating issues.
      Use search before creating to avoid duplicates.
---

# Linear Integration Guide

This source connects to Linear...
```

### Guide Caching

```
Guide is cached in config.json:
  guideCache: {
    sections: [
      { title: "Overview", content: "..." },
      { title: "Tools", content: "..." }
    ],
    cachedAt: "2025-04-18T10:00:00Z"
  }

Cache invalidation:
  - On guide.md file change (ConfigWatcher)
  - On source config update
  - Manual refresh via UI
```

---

## 11. Tool Registration to Agent

### From Source to Agent Tool

```
Source enabled for session
    │
    ▼
1. SessionManager reads enabledSourceSlugs from session config
    │
    ▼
2. For each source slug:
    │
    ├── Load source config
    ├── Check if source is enabled
    │
    ▼
3. Get tools from pool client:
    │
    ├── MCP source:
    │   client.listTools() → ToolDefinition[]
    │   Each tool wrapped as SDK tool
    │
    ├── API source:
    │   api-tools.ts generates tools from spec
    │   Each endpoint → one tool
    │
    └── Local source:
        File operations → tools
    │
    ▼
4. Register tools in agent's MCP server
    │  For ClaudeAgent: added to session MCP server
    │  For PiAgent: sent as proxy definitions
    │
    ▼
5. Agent can call source tools
    │  tool_call("linear_create_issue", { title: "Bug fix" })
    │
    ▼
6. Tool execution routed through pool:
    │  McpPool.executeTool("linear-api", sessionId, ...)
    │  → McpClient.callTool("linear_create_issue", args)
    │  → Subprocess/stdio communication
    │  → Result returned to agent
```

---

## 12. Source Permissions

### Permission Model

```
permissions.json per source:
{
  "allowedTools": ["*"],          // or specific tool names
  "deniedTools": [],
  "allowedPaths": ["/data"],     // For local sources
  "maxTokens": 10000,            // Response token limit
  "rateLimit": {
    "requestsPerMinute": 60
  }
}
```

### Permission Enforcement

```
Agent calls source tool:
    │
    ▼
1. Check tool permission:
    │   allowedTools includes tool name? OR "*"?
    │   deniedTools includes tool name?
    │
    ▼
2. Check rate limit:
    │   Requests in last minute < rateLimit.requestsPerMinute?
    │
    ▼
3. Execute if allowed:
    │   Call tool via pool
    │   Truncate response if exceeds maxTokens
    │
    └── Deny if not allowed:
        Return error to agent
```

---

## 13. Source CRUD Operations

### Create Source

```
createSource(workspaceId, config)
    │
    ├── Generate unique slug from name:
    │   "Linear API" → "linear-api"
    │   Collision → "linear-api-2"
    │
    ├── Create source directory:
    │   sources/linear-api/
    │
    ├── Validate config via Zod schema:
    │   Transport config valid?
    │   Auth config complete?
    │   URL format correct?
    │
    ├── Save config.json (atomic write)
    │
    ├── Download icon if URL provided:
    │   Fetch → save as icon.svg/png/jpg
    │
    └── Return LoadedSource
```

### Delete Source

```
deleteSource(workspaceId, sourceSlug)
    │
    ├── Close MCP connections in pool
    ├── Delete credentials from store
    ├── Remove source directory (recursive)
    │
    └── Notify active sessions:
        Sessions using this source lose access
        Agent sees tool_removed event
```

---

## 14. Source Health Monitoring

### Health Check

```
checkSourceHealth(sourceSlug, sessionId)
    │
    ├── MCP source:
    │   ├── Try ping/heartbeat
    │   ├── Check subprocess alive (stdio)
    │   └── Check connection state (sse/http)
    │
    ├── API source:
    │   ├── Call testEndpoint with credentials
    │   └── Check response status
    │
    └── Local source:
        ├── Check directory exists
        └── Check read permissions
```

### Reconnection

```
MCP source disconnects:
    │
    ▼
1. Mark client as disconnected
    │
    ▼
2. Next tool call triggers reconnect:
    │   stdio: spawn new subprocess
    │   sse: reconnect EventSource
    │   http: new HTTP client
    │
    ▼
3. Re-list tools after reconnect
    │
    ▼
4. Resume tool execution
```

---

## 15. Source Events

### Event Triggers

```
Source events that trigger automations:

onSourceCreated(sourceSlug)      → source.added
onSourceDeleted(sourceSlug)      → source.removed
onSourceConfigChanged(source)    → source.updated
onSourceAuthenticated(source)    → source.authenticated
onSourceConnectionLost(source)   → source.connection_lost
onSourceConnectionRestored(source) → source.connection_restored
```

### ConfigWatcher for Sources

```
ConfigWatcher monitors:
  sources/ directory
    │
    ├── New directory → source created
    ├── config.json modified → source updated
    ├── guide.md modified → guide cache invalidated
    ├── icon.* modified → icon cache invalidated
    └── Directory removed → source deleted
```
