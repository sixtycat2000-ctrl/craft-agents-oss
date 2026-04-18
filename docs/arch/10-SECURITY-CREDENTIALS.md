# Security & Credentials — Encryption, Auth, and Permissions

> How Craft Agents encrypts credentials, manages authentication flows, enforces permission modes, and isolates sessions.

---

## 1. Security Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    Security Layers                           │
│                                                             │
│  Layer 1: Transport Security                                │
│  ├── WebSocket TLS (wss://)                                 │
│  ├── Bearer token authentication                            │
│  └── JWT session cookies (WebUI)                            │
│                                                             │
│  Layer 2: Credential Security                               │
│  ├── AES-256-GCM encrypted store                            │
│  ├── Machine-bound encryption keys                          │
│  └── OAuth token auto-refresh                               │
│                                                             │
│  Layer 3: Permission System                                 │
│  ├── Three permission modes (safe/ask/allow-all)            │
│  ├── Command validation (Bash, PowerShell)                  │
│  └── Tool-level access control                              │
│                                                             │
│  Layer 4: Session Isolation                                 │
│  ├── Per-session cookie partitions                          │
│  ├── Separate browser contexts                              │
│  └── Credential scope boundaries                            │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Credential Encryption

### Storage Backend

```
SecureStorageBackend (primary):
    File: ~/.craft-agent/credentials.enc
    Format: AES-256-GCM encrypted JSON
    Key: Derived from machine-specific identifier

Encryption pipeline:
    ┌─────────────────────────────────────────────────────┐
    │ 1. Collect all credentials into JSON object          │
    │    {                                                │
    │      "llm_api_key::anthropic-api": {...},           │
    │      "source_oauth::ws-123::gmail": {...},          │
    │      ...                                            │
    │    }                                                │
    │                                                     │
    │ 2. Serialize to JSON string                         │
    │                                                     │
    │ 3. Encrypt:                                         │
    │    key = deriveKey(machineId)                       │
    │    iv = crypto.randomBytes(12)                      │
    │    encrypted = AES-256-GCM(json, key, iv)           │
    │    tag = authentication tag (16 bytes)               │
    │                                                     │
    │ 4. Write file:                                      │
    │    {                                                │
    │      "iv": base64(iv),                              │
    │      "tag": base64(tag),                            │
    │      "data": base64(encrypted)                      │
    │    }                                                │
    └─────────────────────────────────────────────────────┘
```

### Key Derivation

```
Machine key derivation:

    machineId → getMachineIdentifier()
    │
    ├── macOS: IOPlatformSerialNumber (hardware UUID)
    ├── Windows: MachineGuid from registry
    └── Linux: /etc/machine-id

    key = PBKDF2(machineId, salt, 100000 iterations, 256 bits)

    Properties:
    - Key bound to physical machine
    - Cannot transfer credentials.enc to another machine
    - No user password required (hardware-bound)
```

### Credential Types

```
┌──────────────────────────┬───────────────────────────────────────┐
│ Type                     │ Stored Fields                         │
├──────────────────────────┼───────────────────────────────────────┤
│ anthropic_api_key        │ { value: "sk-ant-..." }               │
│ claude_oauth             │ { value, refreshToken, expiresAt,     │
│                          │   clientId, clientSecret }            │
│ llm_api_key              │ { value: "sk-..." }                   │
│ llm_oauth                │ { value, refreshToken, expiresAt,     │
│                          │   clientId, clientSecret }            │
│ llm_iam                  │ { value, region, accessKeyId,         │
│                          │   secretAccessKey }                   │
│ llm_service_account      │ { value, projectId, clientEmail,     │
│                          │   privateKey }                        │
│ workspace_oauth          │ { value, refreshToken, expiresAt }    │
│ source_oauth             │ { value, refreshToken, expiresAt,     │
│                          │   clientId, clientSecret }            │
│ source_bearer            │ { value: "token" }                    │
│ source_apikey            │ { value: "key" }                      │
│ source_basic             │ { value: "user:pass" }                │
└──────────────────────────┴───────────────────────────────────────┘
```

### Credential Operations

```
get(credentialId):
    │
    ├── Read credentials.enc from disk
    ├── Decrypt with machine key
    ├── Parse JSON
    ├── Find by credentialId
    └── Return StoredCredential | null

set(credentialId, credential):
    │
    ├── Read + decrypt existing store
    ├── Upsert credential
    ├── Encrypt with machine key
    └── Atomic write to credentials.enc

delete(credentialId):
    │
    ├── Read + decrypt
    ├── Remove entry
    ├── Encrypt + write
    └── Return success

list(filter?):
    │
    ├── Read + decrypt
    ├── Filter by type, connectionSlug, workspaceId
    └── Return matching entries (values masked)
```

### Health Checking

```
checkHealth():
    │
    ├── File exists?
    │   No → return { status: "not_found" }
    │
    ├── Can read file?
    │   No → return { status: "permission_error" }
    │
    ├── Can decrypt?
    │   No → return { status: "corrupted" }
    │
    ├── Parse JSON?
    │   No → return { status: "invalid_format" }
    │
    └── All OK → return { status: "healthy", count: N }
```

---

## 3. Authentication Flows

### OAuth 2.0 Flow

```
┌──────────┐          ┌──────────┐          ┌──────────┐
│ Renderer │          │ Main Proc│          │ Provider │
└────┬─────┘          └────┬─────┘          └────┬─────┘
     │                     │                     │
     │ startOAuth(slug)    │                     │
     │────────────────────▶│                     │
     │                     │                     │
     │                     │ Generate state      │
     │                     │ Store in OAuthStore │
     │                     │                     │
     │                     │ Open browser:       │
     │                     │ authorization_url   │
     │                     │────────────────────▶│
     │                     │                     │
     │                     │     User authenticates
     │                     │                     │
     │                     │◀── callback URL ────│
     │                     │  with auth code     │
     │                     │                     │
     │                     │ Exchange code:      │
     │                     │ POST /token         │
     │                     │────────────────────▶│
     │                     │                     │
     │                     │◀── access_token ────│
     │                     │    refresh_token    │
     │                     │                     │
     │                     │ Store in credential │
     │                     │ manager             │
     │                     │                     │
     │ oauthComplete       │                     │
     │◀────────────────────│                     │
     │                     │                     │
```

### PKCE Flow (Google, Microsoft)

```
Enhanced OAuth with Proof Key for Code Exchange:

1. Generate code_verifier (random string)
2. code_challenge = SHA256(code_verifier) → base64url
3. Authorization URL includes code_challenge
4. Token exchange includes code_verifier
5. Server verifies code_verifier matches challenge

Used by: Google, Microsoft, Claude
Provides: Protection against authorization code interception
```

### Device Code Flow (ChatGPT, GitHub)

```
For providers without redirect URI support:

1. POST /device/code → get user_code + verification_uri
2. Show user_code to user
3. Open verification_uri in browser
4. Poll /device/token until user completes auth
5. Receive access_token + refresh_token

User experience:
    "Go to github.com/login/device"
    "Enter code: XXXX-XXXX"
    [Polling every 5 seconds...]
    "Authentication complete!"
```

### Claude OAuth

```
Claude Max/Pro subscription authentication:

1. Open Claude auth URL with PKCE
2. User logs in with Claude credentials
3. Callback with authorization code
4. Exchange for access + refresh tokens
5. Tokens scoped to Claude Max subscription
6. Auto-refresh when token expires

Token storage:
    Type: claude_oauth
    Key: claude_oauth::
    Fields: value (access_token), refreshToken, expiresAt
```

---

## 4. Permission Modes

### Three-Mode System

```
┌──────────────────────────────────────────────────────────┐
│                                                          │
│  SAFE MODE (Explore)                                     │
│  ─────────────────────                                   │
│  Allowed:  Read, Glob, Grep, browser snapshot            │
│  Blocked:  Write, Bash, file ops, browser fill/click     │
│  Use case: Read-only exploration of codebases            │
│                                                          │
│  ASK MODE (Ask to Edit)  ← default                      │
│  ─────────────────────────────                           │
│  Allowed:  All tools with user approval                  │
│  Behavior: Show permission dialog for each write op      │
│  Use case: General development work                      │
│                                                          │
│  ALLOW-ALL MODE (Auto)                                   │
│  ─────────────────────                                   │
│  Allowed:  All tools, auto-approved                      │
│  Behavior: Execute everything without prompting          │
│  Use case: Trusted automation, CI/CD                     │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### Mode Resolution

```
Effective mode determined by:

  1. Workspace default (from workspace config.json)
  2. Session override (from session config)
  3. Cyclable modes (restricted set user can cycle through)

  Priority: session override > workspace default

  Cyclable modes example:
    workspace default: "ask"
    cyclable: ["safe", "ask", "allow-all"]
    User can cycle: safe → ask → allow-all → safe → ...
```

### Command Validation

```
Bash Command Validation Pipeline (BashValidator, 627 lines):

  "curl -X POST https://api.example.com -d @file.txt"
      │
      ▼
  1. Parse command into tokens
      │
      ▼
  2. Classify command:
      │  Command: curl
      │  Risk: NETWORK_WRITE
      │  Operations: [HTTP_POST, FILE_READ, NETWORK_EXTERNAL]
      │
      ▼
  3. Check against mode rules:
      │
      │  safe mode:
      │    NETWORK_WRITE → BLOCKED
      │
      │  ask mode:
      │    Show dialog:
      │    "Allow curl to POST to api.example.com?"
      │    [Allow] [Deny] [Allow Always for curl]
      │
      │  allow-all mode:
      │    NETWORK_WRITE → ALLOWED
      │
      ▼
  4. Return permission decision
```

### Dangerous Command Patterns

```
Blocked in safe mode, prompted in ask mode:

  Bash:
    ├── Network: curl, wget, nc, ssh, scp, rsync
    ├── File write: rm, mv, cp (overwrite), tee, chmod, chown
    ├── System: sudo, su, kill, reboot, shutdown
    ├── Package: npm install, pip install, brew install
    └── Process: exec, eval, source with redirection

  PowerShell:
    ├── Network: Invoke-WebRequest, Invoke-RestMethod
    ├── File write: Set-Content, Remove-Item, Move-Item
    ├── System: Start-Process, Stop-Process
    └── Execution: Invoke-Expression, & operator
```

---

## 5. Session Isolation

### Browser Cookie Partition

```
Each browser instance uses separate Electron session:
  session.fromPartition('persist:browser-pane')

  Session A's browser:
    Cookies: session_A_domain.com_cookies
    localStorage: isolated
    Cache: isolated

  Session B's browser:
    Cookies: session_B_domain.com_cookies
    localStorage: isolated
    Cache: isolated

  No cross-session cookie/token leakage
```

### Credential Scope

```
Credentials scoped by workspace and source:

  Credential key format:
    {type}::{scope}

  Examples:
    llm_api_key::anthropic-api           → Global LLM key
    llm_oauth::google-ai-studio          → Global LLM OAuth
    source_oauth::ws-123::gmail-api      → Workspace+Source OAuth
    workspace_oauth::ws-123::slack        → Workspace OAuth

  Sessions can only access credentials for their workspace
```

### Agent Isolation

```
Each session has independent:
  ├── Agent backend instance (ClaudeAgent or PiAgent)
  ├── MCP tool set (different sources per session)
  ├── Browser window (1:1 binding)
  ├── Permission queue (separate approval dialogs)
  ├── Streaming state (no cross-session streaming)
  └── Working directory (session-specific CWD)
```

---

## 6. Transport Security

### TLS Configuration

```
Server TLS (headless server):
    CRAFT_RPC_TLS_CERT=/path/to/cert.pem
    CRAFT_RPC_TLS_KEY=/path/to/key.pem
    CRAFT_RPC_TLS_CA=/path/to/ca.pem       (optional)

    Result: wss:// connections encrypted in transit
    Client validates cert chain

Client TLS:
    Node.js: rejectUnauthorized option
    Browser: Native WebSocket (no self-signed support)

Non-local bind protection:
    Binding to 0.0.0.0 without TLS → REFUSED
    --allow-insecure-bind override available
    Reason: prevent cleartext token exposure on network
```

### Authentication

```
Connection auth methods:

  1. Bearer token (CLI, Electron):
     Generate: crypto.randomBytes(24).toString('hex')  // 48 chars, 192 bits
     Validate: timing-safe string comparison
     Minimum: 16 characters
     Entropy check: reject single-char repeats, warn on low uniqueness

  2. JWT session cookie (WebUI):
     Sign: HMAC-SHA256 with server secret
     Payload: { sessionId, expiresAt }
     Validate: verify signature + check expiry
     Set on: successful WebUI login

  3. Local bypass (127.0.0.1):
     Connections from localhost skip auth
     Reason: embedded server in Electron, single user
```

---

## 7. Path Security

### Path Traversal Prevention

```
All file operations validate paths:

  validatePath(requestedPath, workspaceRoot):
    │
    ├── Resolve to absolute path
    ├── Check starts with workspaceRoot
    ├── Check no traversal sequences (../)
    ├── Check no null bytes
    └── Return sanitized absolute path

  Session IDs:
    ├── Format validation: YYMMDD-[a-z]+-[a-z]+
    ├── No path separators allowed
    ├── No null bytes
    └── Must match existing session directory
```

### Source Permission Boundaries

```
Source permissions restrict file access:

  Local source:
    allowedPaths: ["/data/project"]
    → Agent can only read files under /data/project

  MCP source:
    allowedTools: ["search", "read"]
    → Agent can only use listed tools

  API source:
    rateLimit: { requestsPerMinute: 60 }
    → Throttled API access
```

---

## 8. Error Handling Security

### Sensitive Data Scrubbing

```
Sentry error reporting scrubs:
    ├── API keys (sk-*, key-*)
    ├── Bearer tokens
    ├── OAuth tokens
    ├── Email addresses
    ├── IP addresses
    └── File paths beyond workspace root

Log output sanitization:
    ├── Credentials never logged
    ├── Tokens never logged
    ├── Only credential IDs (not values) in debug output
```

### Subprocess Security

```
Pi subprocess:
    ├── No network access to localhost services
    ├── Sandboxed working directory
    ├── No access to credential store
    ├── Tool execution proxied through parent
    └── Killed on parent process exit

MCP subprocess:
    ├── Environment variables controlled
    ├── Credentials injected via env (not CLI args)
    ├── Process killed after idle timeout
    └── Stderr captured and logged
```

---

## 9. Auto-Update Security

```
Update verification:
    ├── Code signing verification (macOS, Windows)
    ├── HTTPS for update metadata
    ├── HTTPS for update download
    ├── Hash verification before install
    └── No auto-install without user approval
```

---

## 10. Security Checklist for Deployment

```
┌──────────────────────────────────────────────────────────┐
│ Production Deployment Checklist                           │
├──────────────────────────────────────────────────────────┤
│                                                          │
│ Transport                                                │
│   [ ] TLS enabled for non-localhost binds                │
│   [ ] Strong bearer token (48+ chars, high entropy)     │
│   [ ] JWT secret rotated regularly                      │
│   [ ] Rate limiting on authentication attempts           │
│                                                          │
│ Credentials                                              │
│   [ ] credentials.enc file permissions restricted        │
│   [ ] Machine key not exportable                         │
│   [ ] OAuth tokens have reasonable expiry                │
│   [ ] Refresh tokens stored securely                    │
│                                                          │
│ Permissions                                              │
│   [ ] Default mode set to "ask"                          │
│   [ ] "allow-all" mode restricted to trusted users       │
│   [ ] Dangerous commands blocked in safe mode            │
│   [ ] Permission audit logging enabled                   │
│                                                          │
│ Isolation                                                │
│   [ ] Browser sessions use separate partitions           │
│   [ ] MCP subprocesses have no host network access       │
│   [ ] File paths validated against workspace root        │
│   [ ] Cross-session credential access prevented          │
│                                                          │
│ Monitoring                                               │
│   [ ] Error reporting with data scrubbing                │
│   [ ] Credential health checks on startup                │
│   [ ] Audit log for permission decisions                 │
│   [ ] Webhook retry limits enforced                      │
└──────────────────────────────────────────────────────────┘
```
