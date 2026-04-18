# Craft Agents OSS — Security Features

> Comprehensive analysis of all security mechanisms, why they exist, how they work, and where they can improve.

---

## Table of Contents

1. [Credential Storage & Encryption](#1-credential-storage--encryption)
2. [Authentication & OAuth](#2-authentication--oauth)
3. [Authorization & Permission System](#3-authorization--permission-system)
4. [Command Validation & Sandboxing](#4-command-validation--sandboxing)
5. [Transport Security](#5-transport-security)
6. [Input Validation & Sanitization](#6-input-validation--sanitization)
7. [Environment & Subprocess Isolation](#7-environment--subprocess-isolation)
8. [CSRF & Session Protection](#8-csrf--session-protection)
9. [Audit Logging](#9-audit-logging)
10. [XSS & Content Security](#10-xss--content-security)
11. [Rate Limiting & DoS Protection](#11-rate-limiting--dos-protection)
12. [Path Security](#12-path-security)
13. [File Upload & Binary Detection](#13-file-upload--binary-detection)
14. [Future Improvements](#future-improvements)

---

## 1. Credential Storage & Encryption

### Problem

AI agent platforms handle highly sensitive credentials: API keys (Anthropic, OpenAI, AWS), OAuth tokens, service account JSONs, and bearer tokens. If stored in plaintext, any local process or malware could steal them. Environment variables are visible to any child process via `/proc/self/environ`.

### What It Uses

| Component | Technology | Location |
|-----------|-----------|----------|
| Encryption | AES-256-GCM (authenticated encryption) | `packages/shared/src/credentials/backends/secure-storage.ts` |
| Key Derivation | PBKDF2 with 100,000 iterations | Same file |
| Machine Binding | Hardware UUID (IOPlatformUUID / MachineGuid / machine-id) | Same file |
| File Permissions | `0o600` (owner read/write only) | Same file |
| Store Format | Binary: 8B magic + 4B flags + 32B salt + 12B IV + 16B auth tag + ciphertext | Same file |

**Supported credential types:** `anthropic_api_key`, `claude_oauth`, `llm_api_key`, `llm_oauth`, `llm_iam` (AWS), `llm_service_account` (GCP), `source_oauth`, `source_bearer`, `source_apikey`, `source_basic`.

**Health check system** validates credential store integrity on startup — detects file corruption, decryption failures, and missing credentials.

### Why It Is Secure

- **AES-256-GCM** provides both confidentiality *and* integrity (authenticated encryption). Tampering with the ciphertext is detected by the auth tag.
- **PBKDF2 with 100k iterations** makes brute-force key derivation computationally expensive.
- **Hardware-UUID-bound key** means credentials are tied to the physical machine — stealing the file alone is useless without the same hardware.
- **Random IV per write** prevents identical plaintexts from producing identical ciphertexts.
- **File permission `0o600`** prevents other local users from reading the store.
- **No plaintext on disk** — credentials are only decrypted in memory when needed.
- **Legacy migration** from hostname-based keys is handled automatically, avoiding degraded security during upgrades.

---

## 2. Authentication & OAuth

### Problem

Users need to authenticate with multiple identity providers (Google, Microsoft, Slack, Claude) without exposing credentials. OAuth flows are vulnerable to authorization code interception, CSRF, and open redirect attacks.

### What It Uses

| Feature | Technology | Location |
|---------|-----------|----------|
| OAuth 2.0 + PKCE | RFC 7636 (code_verifier / code_challenge) | `packages/shared/src/auth/pkce.ts` |
| CSRF Prevention | Random 16-byte state parameter | `packages/shared/src/auth/oauth.ts` |
| Dynamic Client Registration | Per-flow OAuth client creation | `packages/shared/src/auth/oauth.ts` |
| OAuth Discovery | RFC 8414 + RFC 9728 metadata resolution | `packages/shared/src/auth/oauth.ts` |
| SSRF Protection | URL safelist (HTTPS-only, no private IPs) | `packages/shared/src/auth/oauth.ts:693-820` |
| WebUI Session Auth | JWT (HS256), 24h expiry, Argon2id password hash | `packages/server-core/src/webui/auth.ts` |
| OAuth Relay | Stable redirect URI with state envelope | `packages/shared/src/auth/oauth-relay.ts` |

**Supported providers:** Claude, Google, Microsoft, Slack, ChatGPT/Codex, Generic OAuth.

### Why It Is Secure

- **PKCE** prevents authorization code interception — even if an attacker intercepts the code, they cannot exchange it without the `code_verifier` which never leaves the client.
- **State parameter** (16 random bytes) prevents CSRF — an attacker cannot forge an OAuth callback without knowing the state.
- **SSRF protection in discovery** blocks:
  - Non-HTTPS URLs
  - Localhost / loopback addresses (`127.0.0.1`, `::1`)
  - Private IP ranges (`10.x`, `172.16-31.x`, `192.168.x`, `169.254.x`)
- **OAuth relay** uses a stable redirect URI (`https://agents.craft.do/auth/callback`) with deployment-specific state wrapped inside, preventing open redirect vulnerabilities.
- **WebUI sessions** use `HttpOnly` + `SameSite=Strict` + `Secure` cookies, preventing XSS token theft, CSRF, and MITM attacks.
- **Argon2id** for password hashing is memory-hard, resisting GPU/ASIC brute-force attacks.
- **JWT expiry (24h)** limits the window of a compromised session token.

---

## 3. Authorization & Permission System

### Problem

AI agents execute commands and modify files on behalf of users. Without granular permission controls, a malicious or confused prompt could cause irreversible damage — deleting files, exfiltrating data, or running arbitrary code.

### What It Uses

| Feature | Description | Location |
|---------|-------------|----------|
| Three-Tier Modes | `safe` (read-only), `ask` (approval required), `allow-all` (auto-approve) | `packages/shared/src/agent/mode-manager.ts` |
| Tool Permissions | Allowlist/blocklist per tool (Read, Write, Edit, Bash, etc.) | Same file |
| API Permissions | GET always allowed; POST/PUT/DELETE require explicit whitelist | Same file |
| Write Path Restrictions | Only `plansFolderPath`, `dataFolderPath`, and explicit allowlists writable in safe mode | Same file |
| Privileged Execution Broker | Approval workflow for elevated commands with TTL and hash verification | `packages/server-core/src/services/privileged-execution-broker.ts` |
| Mode Audit Trail | Every mode change is tracked with who, when, and why | Same file |

### Why It Is Secure

- **Default-deny** — safe mode blocks all writes by default; `ask` mode requires explicit approval.
- **Per-session state** — no global contamination; each session maintains independent permissions.
- **Monotonic version tracking** — prevents race conditions where concurrent requests bypass mode checks.
- **Privileged execution broker**:
  - Commands are SHA256-hashed for integrity — modified commands are rejected.
  - Configurable TTL (default 120s) — approvals expire automatically.
  - Policy enforcement via allowlist — unauthorized privileged commands are blocked.
  - Full audit trail in `privileged-actions.jsonl`.
- **API endpoint allowlisting** — POST/PUT/DELETE to arbitrary endpoints is blocked by default.

---

## 4. Command Validation & Sandboxing

### Problem

AI-generated shell commands are a primary attack vector. Prompt injection could trick an agent into executing destructive commands (`rm -rf /`), exfiltrating data via network calls, or escalating privileges.

### What It Uses

| Feature | Technology | Location |
|---------|-----------|----------|
| Bash AST Validation | `bash-parser` for proper parse tree | `packages/shared/src/agent/bash-validator.ts` |
| PowerShell Validation | `System.Management.Automation` AST parser | `packages/shared/src/agent/powershell-validator.ts` |
| Read-Only Command Allowlist | Regex patterns for safe commands | Same files |
| Compound Command Analysis | Validates ALL segments of `&&`, `||`, `;` chains | `bash-validator.ts` |

**Bash validation blocks:** command substitution `$(...)`, backticks, process substitution `<(...)`, pipelines `|`, redirects `>` `>>`, parameter expansion `${}`, background execution `&`, dangerous patterns (`find -exec`, `awk system()`).

**PowerShell validation blocks:** pipelines `|`, subexpressions `$(...)`, script blocks `{ }`, `Invoke-Expression`, dot-sourcing `.`, assignments `=`, background `&`.

### Why It Is Secure

- **AST-based parsing** (not regex on raw strings) — prevents bypass via escaping tricks, encoding, or whitespace manipulation.
- **Compound command coverage** — `git status && rm -rf /` is rejected because *both* parts are validated independently.
- **Default-deny** — only explicitly allowed commands pass; everything else is blocked.
- **Customizable allowlist** — workspace `permissions.json` can extend safe commands.
- **Cross-platform** — separate validators for Bash (macOS/Linux) and PowerShell (Windows).

---

## 5. Transport Security

### Problem

AI agents communicate over WebSocket connections that carry sensitive data: conversation content, API keys, file contents. Without encryption, network-level attackers can intercept everything.

### What It Uses

| Feature | Technology | Location |
|---------|-----------|----------|
| TLS/WSS | PEM certificates, configurable via env vars | `packages/server-core/src/transport/server.ts` |
| Client Certificate Auth | Optional CA chain verification | Same file |
| Bearer Token Auth | `CRAFT_SERVER_TOKEN` environment variable | Same file |
| Connection Limits | Max 50 concurrent clients (configurable) | Same file |
| Heartbeat/Timeout | Auto-disconnect stale connections | Same file |

**TLS env vars:** `CRAFT_RPC_TLS_CERT`, `CRAFT_RPC_TLS_KEY`, `CRAFT_RPC_TLS_CA`.

### Why It Is Secure

- **TLS encrypts all traffic** — prevents network sniffing and MITM attacks.
- **Client certificate verification** (optional) provides mutual TLS authentication.
- **Bearer token** provides a shared-secret authentication layer independent of TLS.
- **Connection limits** (50 max) prevent resource exhaustion.
- **Heartbeat system** detects and disconnects zombie connections, freeing resources and reducing attack surface.
- **Dev certificate generation** script (`scripts/generate-dev-cert.sh`) ensures developers can test TLS locally.

---

## 6. Input Validation & Sanitization

### Problem

User inputs (URLs, shell arguments, file paths, markdown content) can contain injection payloads. AI-generated content is especially risky because the "user" is an LLM that might be manipulated via prompt injection.

### What It Uses

| Feature | Technology | Location |
|---------|-----------|----------|
| URL Validation | AI-assisted + rule-based checks | `packages/shared/src/validation/url-validator.ts` |
| Shell Sanitization | Character escaping for shell metacharacters | `packages/shared/src/automations/security.ts` |
| Content Validators | Schema validation for configs, skills, permissions | `packages/shared/src/config/validators.ts` |
| Safe HTML Components | Proxy-based React component wrapping | `packages/ui/src/components/markdown/safe-components.tsx` |
| Slug Validation | Alphanumeric + hyphens only | `packages/shared/src/config/validators.ts` |

**Shell sanitization escapes:** `\`, `` ` ``, `$`, `"`, `'`, newlines, carriage returns.

**URL validation enforces:** HTTPS only, exact hostname `mcp.craft.do`, path prefix `/links/`, no credentials in URL, safe characters only.

### Why It Is Secure

- **Shell sanitization** prevents command injection via environment variables that might contain malicious payloads.
- **URL validation** blocks credential injection (`https://user:pass@evil.com`), subdomain attacks, and non-HTTPS connections.
- **Safe HTML components** catch invalid React tag names (containing `+`, `@`, spaces) and render them as escaped text instead of crashing or executing.
- **Content validators** enforce schema constraints on all configuration inputs, rejecting malformed or oversized payloads.

---

## 7. Environment & Subprocess Isolation

### Problem

MCP servers and tool subprocesses inherit the parent process's environment variables. If a malicious MCP server reads `ANTHROPIC_API_KEY` from the environment, it can exfiltrate credentials to an external server.

### What It Uses

| Feature | Description | Location |
|---------|-------------|----------|
| Environment Sanitization | Strips sensitive env vars from subprocess env | `packages/session-tools-core/src/runtime/sandbox-env.ts` |
| Blocked Variables | `ANTHROPIC_API_KEY`, `AWS_*`, `GITHUB_TOKEN`, `OPENAI_API_KEY`, `GOOGLE_API_KEY`, `STRIPE_SECRET_KEY`, `NPM_TOKEN`, etc. | Same file |
| Cache Isolation | Separate `TMPDIR`, `UV_CACHE_DIR`, `XDG_CACHE_HOME`, `PYTHONPYCACHEPREFIX` per subprocess | Same file |
| Explicit Passthrough | Only vars listed in source config `env` field are passed through | README.md |

### Why It Is Secure

- **Blocklist approach** — all known sensitive credential environment variables are stripped before spawning subprocesses.
- **Cache isolation** — prevents cross-session cache poisoning or data leakage via shared temp/cache directories.
- **Explicit passthrough** — environment variables must be explicitly configured to reach a subprocess; default is deny-all.
- **In-depth defense** — even if a user's `ANTHROPIC_API_KEY` is set globally, MCP servers never see it.

---

## 8. CSRF & Session Protection

### Problem

Cross-Site Request Forgery (CSRF) attacks trick a user's browser into making authenticated requests to a target application. Without protection, visiting a malicious website could trigger actions on the agent platform.

### What It Uses

| Feature | Technology | Location |
|---------|-----------|----------|
| SameSite Cookies | `SameSite=Strict` on session cookie | `packages/server-core/src/webui/auth.ts` |
| HttpOnly Flag | Prevents JavaScript access to session cookie | Same file |
| Secure Flag | HTTPS-only cookie transmission | Same file |
| OAuth State Parameter | 16-byte random CSRF token | `packages/shared/src/auth/oauth.ts` |

### Why It Is Secure

- **`SameSite=Strict`** — the browser never sends the cookie with cross-origin requests. This is the strongest CSRF protection available.
- **`HttpOnly`** — JavaScript cannot read or steal the session token, mitigating XSS-to-session-hijack.
- **`Secure`** — cookie is only sent over HTTPS, preventing network sniffing.
- **OAuth state parameter** — binds the OAuth callback to the specific request that initiated it; an attacker cannot forge a valid callback.

---

## 9. Audit Logging

### Problem

Without audit logs, security incidents are impossible to investigate. You cannot determine what happened, when, or by whom. Compliance requirements (SOC 2, GDPR) mandate audit trails.

### What It Uses

| Feature | Description | Location |
|---------|-------------|----------|
| Event Logger | Append-only `events.jsonl` with CloudEvents schema | `packages/shared/src/automations/event-logger.ts` |
| Privileged Action Log | `privileged-actions.jsonl` for elevated commands | `packages/server-core/src/services/privileged-execution-broker.ts` |
| Event Schema | UUID, type, ISO 8601 timestamp, session/workspace context, payload, duration | Same files |
| Buffered I/O | 100ms flush delay with retry (3 attempts, exponential backoff) | `event-logger.ts` |
| Atomic Buffer Swap | Prevents race conditions in concurrent writes | Same file |

**Privileged action events:** `privileged_request_created`, `privileged_request_approved`, `privileged_request_denied`, `privileged_request_hash_mismatch`, `privileged_request_blocked_by_policy`, `privileged_request_expired`.

### Why It Is Secure

- **Append-only format** — logs cannot be modified after the fact, preserving forensic integrity.
- **CloudEvents schema** — standardized, machine-parseable format for SIEM integration.
- **Buffered I/O with retry** — ensures logs are written even under load, preventing silent event loss.
- **Atomic operations** — prevents log corruption from concurrent writes.
- **Command hashing (SHA256)** — detects if a command was modified between approval and execution.
- **TTL on approvals** — privileged requests expire after 120 seconds, limiting the window for replay attacks.

---

## 10. XSS & Content Security

### Problem

AI agents generate markdown and HTML content that is rendered in the WebUI. Malicious content (from prompt injection or adversarial inputs) could execute JavaScript, steal session tokens, or redirect users.

### What It Uses

| Feature | Technology | Location |
|---------|-----------|----------|
| React Auto-Escaping | `{}` JSX expressions auto-escape | Throughout `packages/ui/` |
| Safe Markdown Rendering | `rehype-raw` disabled by default, `linkify-it` | `packages/ui/src/components/markdown/Markdown.tsx` |
| Invalid Tag Handling | Proxy-based component wrapping, fallback to escaped text | `packages/ui/src/components/markdown/safe-components.tsx` |
| Code Highlighting | Shiki (safe, no eval) | Same file |

### Why It Is Secure

- **No raw HTML by default** — `rehype-raw` is disabled, preventing arbitrary HTML injection.
- **React auto-escaping** — all dynamic content is HTML-escaped by default.
- **Invalid tag detection** — malformed HTML tags (containing `+`, `@`, spaces) are rendered as escaped text instead of being interpreted.
- **Safe link rendering** — `linkify-it` validates URLs before rendering them as clickable links.

---

## 11. Rate Limiting & DoS Protection

### Problem

Brute-force attacks can guess passwords or tokens. Resource exhaustion attacks can overwhelm the server with connections or requests.

### What It Uses

| Feature | Description | Location |
|---------|-------------|----------|
| Per-IP Rate Limiting | 5 attempts per 60 seconds | `packages/server-core/src/webui/auth.ts` |
| Global Rate Limiting | 20 total attempts (defeats IP rotation) | Same file |
| Sliding Window | Time-based tracking with automatic cleanup | Same file |
| WebSocket Connection Limit | Max 50 concurrent clients | `packages/server-core/src/transport/server.ts` |
| Heartbeat Timeout | Auto-disconnect after missed pongs | Same file |

### Why It Is Secure

- **Dual rate limiting** — per-IP prevents single-source attacks; global limit prevents distributed attacks with IP rotation.
- **Sliding window** — more accurate than fixed-window counters, prevents burst attacks at window boundaries.
- **Automatic cleanup** — prevents memory leaks from accumulated rate-limit entries.
- **Connection cap** — prevents resource exhaustion from connection flooding.
- **Heartbeat enforcement** — stale connections are terminated, freeing resources.

---

## 12. Path Security

### Problem

Path traversal attacks (e.g., `../../etc/passwd`) can let an agent or MCP server access files outside the intended workspace. Symlink attacks can redirect file writes to sensitive locations.

### What It Uses

| Feature | Description | Location |
|---------|-------------|----------|
| Symlink-Aware Containment | `realpathSync.native()` resolves symlinks | `packages/session-tools-core/src/runtime/path-security.ts` |
| Sibling-Prefix Prevention | Validates nearest existing ancestor | Same file |
| Write Path Allowlisting | Only specific paths writable in safe mode | `packages/shared/src/agent/mode-manager.ts` |
| Cross-Platform Normalization | Windows backslash → forward slash | Same file |

### Why It Is Secure

- **Symlink resolution** — follows all symlinks to their real targets before validating containment, preventing escape via symbolic links.
- **Sibling-prefix bypass prevention** — detects cases like `/workspace-evil/` matching prefix of `/workspace/`.
- **Case-insensitive comparison on Windows** — prevents bypass via case differences.
- **Separate validation for creation vs. existing paths** — creation paths (which may not exist yet) use different validation logic than existing paths.

---

## 13. File Upload & Binary Detection

### Problem

File type spoofing (renaming `.exe` to `.txt`) can bypass restrictions. Binary content in text processing can crash parsers or exploit vulnerabilities.

### What It Uses

| Feature | Description | Location |
|---------|-------------|----------|
| Magic Number Detection | Identifies file types by content, not extension | `packages/shared/src/utils/binary-detection.ts` |
| Size Limits | Configurable max file size | Same file |
| Sandboxed Document Processing | Isolated Python environments for PDF/XLSX/DOCX/IMG | `apps/electron/resources/scripts/` |

### Why It Is Secure

- **Content-based detection** — magic numbers cannot be spoofed by renaming files.
- **Size limits** — prevents memory exhaustion from oversized uploads.
- **Sandboxed processing** — document tools run in isolated Python environments with limited filesystem access.

---

## Future Improvements

### High Priority

1. **Content Security Policy (CSP) Headers**
   - **Problem:** No CSP headers are served, leaving the WebUI vulnerable to inline script injection if a bypass is found.
   - **Recommendation:** Add strict CSP headers (`script-src 'self'`, `style-src 'self'`, `default-src 'none'`) to all HTTP responses.

2. **Secret Scanning in Git Pre-Commit Hooks**
   - **Problem:** No pre-commit hooks prevent accidental commits of API keys or tokens.
   - **Recommendation:** Add `gitleaks` or `detect-secrets` as a pre-commit hook.

3. **Credential Rotation Automation**
   - **Problem:** Credentials have no automatic rotation; users must manually update keys.
   - **Recommendation:** Add key rotation prompts based on age, and support for short-lived tokens with automatic refresh.

### Medium Priority

4. **Subresource Integrity (SRI) for CDN Assets**
   - **Problem:** If CDN-hosted assets are compromised, the WebUI loads them without integrity checks.
   - **Recommendation:** Add `integrity` attributes to all `<script>` and `<link>` tags loading external resources.

5. **Security Headers Suite**
   - **Problem:** Missing `X-Content-Type-Options`, `X-Frame-Options`, `Strict-Transport-Security`, `Referrer-Policy` headers.
   - **Recommendation:** AddHelmet-style security header middleware to the WebUI server.

6. **Dependency Vulnerability Scanning in CI**
   - **Problem:** No automated dependency audit in CI pipeline (`bun audit` or `snyk`).
   - **Recommendation:** Add `bun audit` step to `validate.yml` and block merges on critical vulnerabilities.

7. **Hardware Security Module (HSM) / Keychain Integration**
   - **Problem:** Encryption keys are derived from hardware UUID and stored in software. A determined attacker with physical access could extract the key.
   - **Recommendation:** Use OS keychain (macOS Keychain, Windows Credential Manager, Linux Secret Service) for key storage.

8. **Audit Log Tamper Detection**
   - **Problem:** Append-only logs are not cryptographically signed; a root user could modify them.
   - **Recommendation:** Add hash-chaining (each entry includes hash of previous entry) or Merkle tree signing.

### Lower Priority / Nice-to-Have

9. **OAuth Token Binding (DPoP / mTLS)**
   - **Problem:** OAuth tokens can be stolen and replayed from different machines.
   - **Recommendation:** Implement DPoP (RFC 9449) or mutual TLS token binding.

10. **Automated Penetration Testing**
    - **Problem:** Security relies on manual code review and unit tests, not adversarial testing.
    - **Recommendation:** Integrate OWASP ZAP or similar automated pentest tool in CI.

11. **Supply Chain Verification**
    - **Problem:** No lockfile integrity verification or Sigstore signing for dependencies.
    - **Recommendation:** Pin dependencies with integrity hashes, use `npm --ignore-scripts` by default.

12. **Sandboxed Tool Execution (Container/VM)**
    - **Problem:** Bash commands run in the user's shell environment with user-level permissions.
    - **Recommendation:** Offer optional container/VM isolation for tool execution (e.g., via Docker or Firecracker).

13. **Rate Limiting for API Endpoints**
    - **Problem:** Rate limiting exists for auth but not for general API endpoints.
    - **Recommendation:** Extend rate limiting to all API endpoints with per-user and global limits.

14. **WebAuthn / Passkey Support**
    - **Problem:** Password-based auth is the weakest link; passwords can be phished.
    - **Recommendation:** Add WebAuthn/passkey support for phishing-resistant authentication.

15. **Security.txt**
    - **Problem:** No `security.txt` file for vulnerability reporting discovery.
    - **Recommendation:** Add `.well-known/security.txt` following RFC 9116.

---

## Architecture Summary

```
┌─────────────────────────────────────────────────────┐
│                    User / Browser                    │
└──────────────┬──────────────────────┬───────────────┘
               │                      │
        ┌──────▼──────┐        ┌──────▼──────┐
        │   WebUI     │        │  CLI / IDE  │
        │  (React)    │        │  (Electron) │
        └──────┬──────┘        └──────┬──────┘
               │                      │
    ┌──────────▼──────────────────────▼──────────┐
    │           Transport Layer                   │
    │  TLS/WSS · Bearer Token · Connection Limit │
    └──────────────────┬─────────────────────────┘
                       │
    ┌──────────────────▼─────────────────────────┐
    │         Authentication Layer                │
    │  OAuth+PKCE · JWT · Argon2id · Rate Limit  │
    └──────────────────┬─────────────────────────┘
                       │
    ┌──────────────────▼─────────────────────────┐
    │         Authorization Layer                 │
    │  Permission Modes · Tool Allowlist ·        │
    │  API Endpoint Rules · Write Path Controls   │
    └──────────────────┬─────────────────────────┘
                       │
    ┌──────────────────▼─────────────────────────┐
    │         Validation Layer                    │
    │  Bash/PS AST Validation · Shell Sanitization│
    │  URL Validation · Path Security · Input     │
    └──────────────────┬─────────────────────────┘
                       │
    ┌──────────────────▼─────────────────────────┐
    │         Execution Layer                     │
    │  Env Sanitization · Sandbox Isolation ·     │
    │  Privileged Execution Broker · Audit Logs   │
    └──────────────────┬─────────────────────────┘
                       │
    ┌──────────────────▼─────────────────────────┐
    │         Credential Storage                  │
    │  AES-256-GCM · PBKDF2 · Hardware UUID ·    │
    │  File Permissions 0o600                     │
    └────────────────────────────────────────────┘
```

---

*Document generated from codebase analysis. Last updated: 2026-04-18.*
