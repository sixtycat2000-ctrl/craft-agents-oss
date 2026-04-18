# Sessions & Storage — Persistence Architecture

> How Craft Agents stores sessions, manages workspace-scoped data, and provides fast access to conversation history with JSONL persistence.

---

## 1. Session Data Model

### Core Types

```
SessionConfig (base metadata)
├── id: string                    # YYMMDD-adjective-noun format
├── workspaceRootPath: string     # Parent workspace directory
├── name: string                  # Auto-generated or user-set
├── createdAt: string             # ISO timestamp
├── lastUsedAt: string            # Updated on every message
├── workingDirectory: string      # CWD for agent operations
├── sdkCwd: string               # SDK transcript directory
├── permissionMode: string        # safe | ask | allow-all
├── enabledSourceSlugs: string[]  # Active MCP/API sources
├── model: string                 # LLM model identifier
├── llmConnection: string         # Connection slug
├── thinkingLevel: string         # none | low | medium | high | max
├── labels: string[]              # Label IDs
├── status: string                # Status ID
├── flag: string | null           # Flag type (null = unflagged)
└── archivedAt: string | null     # Archive timestamp

StoredSession = SessionConfig + messages + tokenUsage

SessionHeader (optimized for listing)
├── All SessionConfig fields
├── messageCount: number          # Pre-computed count
├── preview: string               # Last message preview (100 chars)
├── lastMessageRole: string       # user | assistant
└── lastFinalMessageId: string    # For streaming detection

SessionMetadata (runtime UI display)
├── id, name, createdAt
├── processingStatus: string      # idle | processing | streaming
├── streamingContent: string      # Current streaming text
├── backgroundTasks: Task[]       # Active background tasks
└── tokenUsage: SessionTokenUsage
```

### Message Types

```
StoredMessage
├── type: 'user' | 'assistant' | 'system'
├── content: string
├── toolUseId?: string            # For tool call/result correlation
├── toolName?: string             # Tool name for display
├── toolInput?: any               # Tool call arguments
├── toolResult?: any              # Tool execution result
├── attachments?: Attachment[]    # File attachments
├── thinkingContent?: string      # Extended thinking output
├── timestamp: string
└── metadata?: Record<string, any>
```

---

## 2. Storage Layout

### Directory Structure

```
{workspaceRootPath}/
└── sessions/
    └── {sessionId}/
        ├── session.jsonl          # Main data file
        ├── attachments/           # Uploaded files
        │   ├── image.png
        │   └── document.pdf
        ├── plans/                 # Plan markdown files
        │   └── plan-001.md
        ├── data/                  # transform_data outputs (JSON)
        │   └── result-001.json
        ├── long_responses/        # Summarized tool results
        │   └── lr-001.txt
        └── downloads/             # Files from API sources
            └── export.csv
```

### JSONL Format

```
Line 1: SessionHeader (metadata only, for fast listing)
Line 2+: StoredMessage (one per line)

Example:
────────────────────────────────────────────────────
{"id":"250418-swift-eagle","name":"Code Review","createdAt":"2025-04-18T10:00:00Z",...}
{"type":"user","content":"Review this code","timestamp":"2025-04-18T10:01:00Z"}
{"type":"assistant","content":"I'll review the code...","timestamp":"2025-04-18T10:01:05Z"}
{"type":"assistant","toolUseId":"tu-1","toolName":"Read","toolInput":{"path":"src/main.ts"},"timestamp":"..."}
{"type":"assistant","toolUseId":"tu-1","toolResult":{"output":"file contents..."},"timestamp":"..."}
{"type":"assistant","content":"The code looks good overall...","timestamp":"2025-04-18T10:01:10Z"}
────────────────────────────────────────────────────
```

---

## 3. Session ID Generation

### Format

```
YYMMDD-adjective-noun

Examples:
  250418-swift-eagle
  250418-calm-river
  250418-bold-fox
  250418-quiet-deer
```

### Word Lists

```
Adjectives (pool of ~50):
  swift, calm, bold, quiet, bright, dark, warm, cool,
  sharp, smooth, rapid, gentle, fierce, brave, keen,
  clever, nimble, steady, eager, vivid, ...

Nouns (pool of ~50):
  eagle, river, fox, deer, wolf, bear, hawk, lion,
  owl, crane, lynx, orca, puma, raven, tiger, falcon,
  panther, otter, bison, cobra, ...
```

### Collision Handling

```
generateSessionId():
  1. date = YYMMDD
  2. adj = random from adjectives
  3. noun = random from nouns
  4. id = "${date}-${adj}-${noun}"
  5. if exists: retry with different adj/noun (max 10 attempts)
  6. if all taken: append random 4-digit suffix
```

---

## 4. CRUD Operations

### Create

```
createSession(workspaceId, options)
    │
    ├── Generate session ID (YYMMDD-adjective-noun)
    │
    ├── Create directory structure:
    │   sessions/{id}/
    │   sessions/{id}/attachments/
    │   sessions/{id}/plans/
    │   sessions/{id}/data/
    │   sessions/{id}/downloads/
    │
    ├── Build SessionHeader:
    │   - Merge defaults from workspace config
    │   - Set permissionMode from workspace default
    │   - Set model from workspace default
    │   - Set enabledSourceSlugs from workspace default
    │
    ├── Write initial JSONL (header line only):
    │   atomic: session.jsonl.tmp → session.jsonl
    │
    ├── Register in session index:
    │   Update in-memory map + notify watchers
    │
    └── Return SessionConfig
```

### Read (Fast Path)

```
readSessionHeader(sessionId)
    │
    ├── Open session.jsonl
    ├── Read first 8KB (buffer limit)
    ├── Parse first line as SessionHeader
    └── Return header (no messages loaded)

Performance: <1ms per session, used for sidebar listing
```

### Read (Full)

```
loadSession(sessionId)
    │
    ├── Read entire session.jsonl
    │
    ├── Parse line 1 as SessionHeader
    │
    ├── Parse lines 2+ as StoredMessage[]
    │   - Handle multiline content (escaped newlines)
    │   - Handle long responses (reference files in long_responses/)
    │   - Expand {{SESSION_PATH}} tokens
    │
    ├── Compute derived fields:
    │   - messageCount
    │   - preview (last user message)
    │   - tokenUsage totals
    │
    └── Return StoredSession
```

### Update (Debounced)

```
saveSession(session)
    │
    ├── Queue for async persistence:
    │   Debounce: 100ms (300ms on Windows)
    │
    ├── Serialize session:
    │   Line 1: SessionHeader (recomputed)
    │   Lines 2+: StoredMessage[] (each on one line)
    │
    ├── Portable path conversion:
    │   Absolute paths → ~/ prefix where possible
    │   Session directory → {{SESSION_PATH}} token
    │
    └── Atomic write:
        1. Write to session.jsonl.tmp
        2. fs.rename(session.jsonl.tmp, session.jsonl)
```

### Delete

```
deleteSession(sessionId)
    │
    ├── Cancel pending saves
    ├── Unregister session-scoped callbacks
    ├── Remove entire session directory (recursive)
    └── Update session index
```

---

## 5. Session Index & Listing

### In-Memory Index

```
SessionIndex
├── Map<sessionId, SessionHeader>     # Fast lookup by ID
├── orderedIds: string[]              # Sorted by lastUsedAt
├── byDate: Map<date, string[]>       # Grouped by creation date
└── byLabel: Map<labelId, string[]>   # Grouped by label
```

### Listing Operations

```
listSessions(workspaceId, options)
    │
    ├── Filter by options:
    │   - status: show only sessions with given status
    │   - labels: show only sessions with given labels
    │   - flag: show only flagged sessions
    │   - archived: include/exclude archived
    │   - search: text search in name and messages
    │
    ├── Sort:
    │   Default: lastUsedAt (newest first)
    │   Alternatives: createdAt, name
    │
    └── Return SessionHeader[] (no messages)
```

### Lazy Loading

```
UI requests:
  Session list → readSessionHeader() for each → metadata only
  Open session → loadSession() → full messages
  Scroll up   → messages already loaded (from initial load)
```

Memory pattern: 300 sessions × 1KB header = ~300KB (vs 300 sessions × 500KB full = ~150MB)

---

## 6. Persistence Queue

### Write Debouncing

```
Session updated (streaming delta, metadata change, etc.)
    │
    ▼
Persistence Queue
    │
    ├── Check if session already queued:
    │   Yes → update queued data (merge)
    │   No  → add to queue with debounce timer
    │
    ├── Debounce timer fires:
    │   Take all queued sessions
    │   Serialize and write each
    │   Use atomic writes (tmp + rename)
    │
    └── Flush on:
        - App quit (flush all)
        - Session switch (flush that session)
        - Timer (every 5 seconds, flush oldest)
```

### Crash Safety

```
Scenario: App crashes mid-write

Protection:
  1. Atomic writes via tmp + rename
     → session.jsonl.tmp exists, session.jsonl is intact
     → On next load, .tmp file cleaned up

  2. Header always in sync
     → Header recomputed from full data before write
     → If header says 10 messages but file has 9:
       → Recover by re-counting messages

  3. Append-friendly format
     → JSONL is append-friendly
     → Partial line at end → detect and truncate
```

---

## 7. ConfigWatcher Integration

### File Watching

```
ConfigWatcher monitors:
  sessions/ directory (recursive)
    │
    ├── New file created → index session
    ├── File modified → reload session header
    ├── File deleted → remove from index
    └── Directory created → new session directory
```

### Callbacks

```
onSessionCreate(sessionId: string)
onSessionUpdate(sessionId: string)
onSessionDelete(sessionId: string)
onSessionListChange()  → triggers sidebar refresh
```

### Cross-Process Sync

```
Scenario: External editor modifies session file

1. ConfigWatcher detects change
2. Reloads session header
3. Fires onSessionUpdate callback
4. SessionManager updates in-memory state
5. Push event to renderer → atom update → UI refresh
```

---

## 8. Session Status Workflow

### Status System

```
Statuses are workspace-scoped, user-defined:

Default statuses:
  ┌──────────┐    ┌───────────┐    ┌──────────────┐
  │  Inbox   │───▶│  Active   │───▶│  Completed   │
  └──────────┘    └─────┬─────┘    └──────────────┘
                        │
                        ▼
                  ┌───────────┐
                  │  Blocked  │
                  └───────────┘

Custom statuses:
  Workspace admins define their own workflow
  Example: Triage → In Progress → Review → Done
```

### Status Operations

```
setSessionStatus(sessionId, statusId)
    │
    ├── Validate status exists in workspace
    ├── Update session config
    ├── Log status change event
    │   → Triggers automations matching status change
    └── Debounced save
```

---

## 9. Label System

### Labels

```
Labels are workspace-scoped tags:

Label definition:
├── id: string           # Unique identifier
├── name: string         # Display name
├── color: string        # Hex color
└── createdAt: string

Session can have multiple labels:
  session.labels = ["urgent", "frontend", "bug"]
```

### Label Operations

```
addLabel(sessionId, labelId)
removeLabel(sessionId, labelId)
setLabels(sessionId, labelIds[])

Automation triggers:
  onLabelAdd(sessionId, labelId)
  onLabelRemove(sessionId, labelId)
```

---

## 10. Token Usage Tracking

### Per-Session Tracking

```
SessionTokenUsage
├── inputTokens: number
├── outputTokens: number
├── totalTokens: number
├── contextTokens: number     # Current context window usage
├── costUsd: number          # Estimated cost
└── byModel: Map<string, {   # Breakdown by model
      inputTokens: number
      outputTokens: number
      costUsd: number
    }>
```

### Cost Estimation

```
Model pricing (per million tokens):
  claude-opus-4-6:    $15 input, $75 output
  claude-sonnet-4-6:  $3 input, $15 output
  claude-haiku-4-5:   $0.80 input, $4 output

Usage tracking:
  Each agent turn updates tokenUsage
  Cumulative across session lifetime
  Displayed in session header and settings
```

---

## 11. Session Export & Import

### Export Format

```
Session Bundle (for sharing):
├── session.jsonl          # Full session data
├── metadata.json          # Session config
├── attachments/           # All attached files
│   └── ...
└── manifest.json          # Bundle version, checksums

Export flow:
  1. Serialize session to JSONL
  2. Copy attachments
  3. Generate manifest with SHA-256 checksums
  4. Zip bundle
  5. Return path for download
```

### Remote Transfer

```
For workspace-to-workspace transfer:

Export:
  1. Summarize session (for manifest)
  2. Chunk session data (for large sessions)
  3. Return chunks via RPC

Import:
  1. Receive chunks
  2. Reassemble session data
  3. Create session in target workspace
  4. Expand portable paths for new workspace
  5. Verify integrity via checksums
```

---

## 12. Sharing (Viewer)

### Share Flow

```
shareToViewer(sessionId)
    │
    ├── Generate share token (UUID)
    ├── Create static HTML bundle:
    │   - Session transcript rendered as HTML
    │   - Markdown → HTML
    │   - Code → syntax-highlighted HTML
    │   - Images embedded as data URLs
    │
    ├── Upload to viewer service
    │
    └── Return share URL:
        https://viewer.craft.do/share/{token}
```

---

## 13. Portable Paths

### Cross-Machine Compatibility

```
Path encoding on save:
  /Users/yul/.craft-agent/workspaces/my-ws/sessions/250418-swift-eagle
  → ~/.craft-agent/workspaces/my-ws/sessions/250418-swift-eagle

  /Users/yul/.craft-agent/workspaces/my-ws/sessions/250418-swift-eagle/data/result.json
  → {{SESSION_PATH}}/data/result.json

Path decoding on load:
  ~/... → expand to current user's home directory
  {{SESSION_PATH}} → replace with actual session directory
```

This allows session data to be portable across machines with different home directories.

---

## 14. Session Search

### Full-Text Search

```
searchSessions(workspaceId, query)
    │
    ├── Parse query:
    │   - Plain text: search name + message content
    │   - "label:urgent": filter by label
    │   - "status:active": filter by status
    │   - "after:2025-04-01": date filter
    │
    ├── Search execution:
    │   - Headers first (fast name match)
    │   - Then full content (if needed)
    │   - Rank by relevance
    │
    └── Return matching session IDs + preview matches
```

---

## 15. Cleanup & Retention

### Archive Policy

```
archiveSession(sessionId):
  ├── Set archivedAt timestamp
  ├── Remove from active session index
  ├── Move to archive list
  └── Data retained until manual delete or retention policy

deleteOldArchivedSessions(retentionDays):
  ├── Find sessions archived > retentionDays ago
  ├── Delete session directories
  └── Clean up from index
```

### Storage Estimation

```
Per-session storage:
  JSONL file:          10-500 KB (depends on message count)
  Attachments:         0-50 MB (depends on files)
  Plans:               1-50 KB
  Data outputs:        1-100 KB

Typical session (50 messages): ~100 KB
Heavy session (500 messages + attachments): ~10 MB

300 sessions average: ~30-50 MB
```
