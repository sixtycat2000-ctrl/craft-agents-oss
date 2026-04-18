# Automations & Events — Event-Driven Workflow Engine

> How Craft Agents implements event-driven automations with condition matching, scheduled triggers, prompt and webhook actions, and integration with the agent runtime.

---

## 1. Automation Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Event Bus                                 │
│                                                              │
│  Agent Events        App Events         Scheduled Events    │
│  ┌─────────────┐    ┌──────────────┐   ┌────────────────┐  │
│  │ PostToolUse │    │ LabelAdd     │   │ Cron tick      │  │
│  │ PreToolUse  │    │ LabelRemove  │   │ (5 levels)     │  │
│  │ SessionStart│    │ StatusChange │   │                │  │
│  │ SessionEnd  │    │ FlagChange   │   │                │  │
│  │ Notification│    │ ModeChange   │   │                │  │
│  └──────┬──────┘    └──────┬───────┘   └───────┬────────┘  │
│         │                  │                    │           │
│         └──────────────────┼────────────────────┘           │
│                            │                                │
│                            ▼                                │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              Matcher Engine                           │   │
│  │                                                       │   │
│  │  For each automation:                                 │   │
│  │  1. Check if enabled                                  │   │
│  │  2. Match event type (regex or cron)                  │   │
│  │  3. Evaluate conditions                               │   │
│  │  4. Execute actions                                   │   │
│  └──────────────────────────────────────────────────────┘   │
│                            │                                │
│              ┌─────────────┼──────────────┐                 │
│              ▼             ▼              ▼                 │
│     ┌──────────────┐ ┌──────────┐ ┌──────────────┐        │
│     │PromptAction  │ │Webhook   │ │EventLogAction│        │
│     │(send to agent│ │(HTTP req)│ │(history log) │        │
│     └──────────────┘ └──────────┘ └──────────────┘        │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Event Types

### Agent Events

```
Agent events emitted during session processing:

┌─────────────────────┬───────────────────────────────────────┐
│ Event               │ When emitted                          │
├─────────────────────┼───────────────────────────────────────┤
│ SessionStart        │ Agent begins processing message       │
│ SessionEnd          │ Agent finishes processing             │
│ PreToolUse          │ Before tool execution (with args)     │
│ PostToolUse         │ After tool execution (with result)    │
│ Notification        │ Agent emits notification message      │
│ Stop                │ Agent stops (user or error)           │
│ Error               │ Agent encounters error                │
│ TextDelta           │ Streaming text content                │
│ ToolCall            │ Tool invocation                       │
│ ToolResult          │ Tool execution result                 │
└─────────────────────┴───────────────────────────────────────┘
```

### App Events

```
App events from UI and system operations:

┌─────────────────────┬───────────────────────────────────────┐
│ Event               │ When emitted                          │
├─────────────────────┼───────────────────────────────────────┤
│ LabelAdd            │ Label added to session                │
│ LabelRemove         │ Label removed from session            │
│ StatusChange        │ Session status changed                │
│ FlagChange          │ Session flagged/unflagged             │
│ PermissionModeChange│ Permission mode changed               │
│ SchedulerTick       │ Cron tick at configured level         │
│ SourceChange        │ Source added/removed/updated          │
│ SkillChange         │ Skill added/removed/updated           │
│ WorkspaceChange     │ Workspace settings changed            │
└─────────────────────┴───────────────────────────────────────┘
```

### SDK Event Bridge

```
Agent SDK hooks → Automation events:

SDK HookCallback:
  ├── PreToolUse   → emit('PreToolUse', { sessionId, tool, args })
  ├── PostToolUse  → emit('PostToolUse', { sessionId, tool, result })
  ├── Notification → emit('Notification', { sessionId, message })
  ├── Stop         → emit('Stop', { sessionId, reason })
  └── Error        → emit('Error', { sessionId, error })

Bridge: sdk-bridge.ts maps SDK hooks to automation event format
```

---

## 3. Automation Configuration

### Matcher Definition

```typescript
interface AutomationMatcher {
  id: string
  name: string
  enabled: boolean

  // Trigger: regex match on event name OR cron schedule
  matcher?: string          // Regex: "SessionEnd|PostToolUse"
  cron?: string             // Schedule: "0 9 * * 1-5" (weekdays 9am)

  // Conditions: additional filters
  conditions?: Condition[]

  // Actions: what to do when matched
  actions: Action[]
}
```

### Condition Types

```
┌─────────────────────────────────────────────────────────────┐
│ TimeCondition                                               │
│   timeOfDay: { after: "09:00", before: "17:00" }           │
│   dayOfWeek: ["monday", "tuesday", "wednesday", ...]       │
│   timezone: "America/New_York"                              │
│                                                             │
│ StateCondition                                              │
│   field: "status"                                          │
│   value: "completed"                                       │
│   from: "active"           // State transition: active→completed │
│   contains: "review"       // Value contains substring     │
│                                                             │
│ LogicalCondition                                            │
│   operator: "and" | "or" | "not"                           │
│   conditions: Condition[]   // Nested conditions           │
└─────────────────────────────────────────────────────────────┘
```

### Action Types

```
┌─────────────────────────────────────────────────────────────┐
│ PromptAction                                                │
│   type: "prompt"                                            │
│   prompt: "Summarize the completed work and send to Slack"  │
│   llmConnection: "anthropic-api"        // Optional override│
│   model: "claude-sonnet-4-6"            // Optional override│
│   permissionMode: "ask"                 // Optional override│
│   labels: ["automated"]                 // Optional labels  │
│                                                             │
│ WebhookAction                                               │
│   type: "webhook"                                           │
│   url: "https://hooks.slack.com/services/..."               │
│   method: "POST"                        // HTTP method      │
│   headers: { "Authorization": "Bearer ..." }                │
│   body: "{\"text\": \"Session {{sessionId}} completed\"}"   │
│   auth: { type: "bearer", token: "..." }                    │
└─────────────────────────────────────────────────────────────┘
```

---

## 4. Matcher Engine

### Matching Pipeline

```
Event arrives → Matcher Engine:
    │
    ├── For each enabled automation:
    │
    │   1. Event type match:
    │      matcher regex test against event name
    │      OR cron schedule matches current time
    │
    │   2. Condition evaluation:
    │      For each condition:
    │        TimeCondition → check time/day/timezone
    │        StateCondition → check event field values
    │        LogicalCondition → recursively evaluate children
    │      All conditions must pass (AND logic)
    │
    │   3. Action execution:
    │      For each action:
    │        PromptAction → enqueue prompt for new session
    │        WebhookAction → send HTTP request
    │
    └── Log result to history
```

### Condition Evaluation

```
evaluateCondition(condition, event):

  TimeCondition:
    ├── Check current time against timeOfDay range
    ├── Check current day against dayOfWeek list
    └── Convert to specified timezone before comparison

  StateCondition:
    ├── field: check event.data[field]
    ├── value: exact match
    ├── from/to: check transition (previous → current)
    ├── contains: substring match
    └── not_value: field must not equal value

  LogicalCondition:
    ├── AND: all children must pass
    ├── OR: any child must pass
    └── NOT: negate child result

Short-circuit evaluation for performance
```

### Template Variables

```
Webhook and prompt bodies support template variables:

  {{sessionId}}      → current session ID
  {{sessionName}}    → session display name
  {{workspaceId}}    → workspace ID
  {{eventType}}      → event type name
  {{timestamp}}      → ISO timestamp
  {{data.fieldName}} → event data field
  {{from}}           → previous state value
  {{to}}             → new state value

Example webhook body:
  {
    "text": "Session {{sessionName}} ({{sessionId}}) changed to {{to}}",
    "channel": "#agents"
  }
```

---

## 5. Prompt Action Execution

### Prompt Flow

```
PromptAction matched:
    │
    ▼
1. Create pending prompt:
    {
      automationId: "auto-123",
      prompt: "Summarize the completed work",
      sessionId: null,         // New session
      llmConnection: override, // Optional
      model: override,         // Optional
      permissionMode: "ask",   // Optional
      labels: ["automated"]    // Optional
    }
    │
    ▼
2. Enqueue for execution:
    Added to pending prompts queue
    Processed by SessionManager
    │
    ▼
3. Create or reuse session:
    ├── New session if no sessionId
    │   createSession() with configured options
    │
    └── Existing session if sessionId provided
        Resume in existing session
    │
    ▼
4. Send prompt to agent:
    sendMessage(sessionId, expandedPrompt)
    │
    ▼
5. Agent processes:
    Uses configured model and permission mode
    Labels applied to session
    Results streamed normally
```

### Prompt Expansion

```
Template variables expanded before sending:

  "Summarize the work done in session {{data.sourceSessionId}}"
  → "Summarize the work done in session 250418-swift-eagle"

  "Check status of {{data.toolName}} execution"
  → "Check status of browser_tool execution"
```

---

## 6. Webhook Action Execution

### Webhook Flow

```
WebhookAction matched:
    │
    ▼
1. Build HTTP request:
    ├── URL: action.url
    ├── Method: action.method (POST, GET, PUT, etc.)
    ├── Headers: action.headers + auth headers
    └── Body: template-expanded action.body
    │
    ▼
2. Send request:
    fetch(url, { method, headers, body })
    │
    ├── Timeout: 10 seconds
    │
    ├── Success (2xx):
    │   Log response to history
    │   Return WebhookActionResult
    │
    ├── Retryable error (5xx, network):
    │   Add to retry queue
    │   Exponential backoff (1s, 2s, 4s, max 30s)
    │   Max 5 retries
    │
    └── Permanent error (4xx):
        Log error to history
        No retry
```

### Retry Queue

```
automations-retry-queue.jsonl:
    Persistent retry queue for failed webhooks

  Entry format:
    {
      automationId: "auto-123",
      url: "https://hooks.slack.com/...",
      method: "POST",
      headers: {...},
      body: "...",
      attempt: 2,
      nextAttemptAt: "2025-04-18T10:05:00Z",
      error: "Connection refused"
    }

  Processing:
    - Checked every minute
    - Attempts after nextAttemptAt
    - Removed on success or max retries
```

---

## 7. Cron Scheduler

### Five-Level Thinking System

```
The scheduler integrates with the agent's thinking levels:

┌──────────────────────────────────────────────────────────┐
│ Level │ Frequency       │ Use Case                      │
├───────┼─────────────────┼───────────────────────────────┤
│ 1     │ Every 1 minute  │ Real-time monitoring           │
│ 2     │ Every 5 minutes │ Periodic checks                │
│ 3     │ Every 15 minutes│ Background tasks               │
│ 4     │ Every hour      │ Daily summaries                │
│ 5     │ Every day       │ Scheduled reports              │
└───────┴─────────────────┴───────────────────────────────┘

Cron expressions:
  "*/5 * * * *"  → every 5 minutes (level 2)
  "0 9 * * 1-5"  → weekdays at 9am (level 5)
  "0 * * * *"    → every hour (level 4)
```

### Cron Matcher

```
cron-matcher.ts evaluates cron expressions:

  matchCron(expression, date):
    ├── Parse cron fields: minute hour dayOfMonth month dayOfWeek
    ├── Match each field against current date
    ├── Support ranges: "1-5" → Monday through Friday
    ├── Support steps: "*/5" → every 5th
    └── Return boolean match

  Scheduler tick:
    Every minute, emit SchedulerTick event with current time
    Match against all cron-enabled automations
    Execute matched automations
```

---

## 8. Automation History

### Logging

```
automations-history.jsonl:
    One entry per automation execution

  Entry format:
    {
      automationId: "auto-123",
      automationName: "Daily Summary",
      event: "SchedulerTick",
      timestamp: "2025-04-18T09:00:00Z",
      conditionsMet: true,
      actions: [
        { type: "prompt", result: { sessionId: "abc" } },
        { type: "webhook", result: { status: 200 } }
      ],
      error: null,
      duration_ms: 150
    }

  Retention limits:
    - 20 entries per automation (rolling)
    - 1000 entries total
    - Field values truncated at 2000 chars
```

### History Queries

```
getAutomationHistory(automationId, options):
    │
    ├── Filter by automationId
    ├── Filter by date range
    ├── Filter by success/failure
    └── Return with pagination

getRecentHistory(limit):
    │
    └── Return last N entries across all automations
```

---

## 9. Automation CRUD

### Storage

```
automations.json (per workspace):
    {
      "automations": [
        {
          "id": "auto-123",
          "name": "Morning Briefing",
          "enabled": true,
          "matcher": null,
          "cron": "0 9 * * 1-5",
          "conditions": [],
          "actions": [
            {
              "type": "prompt",
              "prompt": "Check my Linear issues and summarize the urgent ones"
            }
          ],
          "createdAt": "2025-04-18T10:00:00Z",
          "updatedAt": "2025-04-18T10:00:00Z"
        }
      ]
    }
```

### Operations

```
saveAutomationsConfig(workspaceId, config)
    │
    ├── Validate via Zod schema
    ├── Atomic write to automations.json
    └── ConfigWatcher triggers reload

loadAutomationsConfig(workspaceId)
    │
    ├── Read automations.json
    ├── Validate schema
    ├── Start/stop cron timers as needed
    └── Return config

testAutomation(workspaceId, automationId, testEvent)
    │
    ├── Build synthetic event from testEvent
    ├── Run through matcher engine
    ├── Execute actions in dry-run mode (no actual sends)
    └── Return match result + expanded actions
```

---

## 10. Canonical Matcher Adapters

### Unified Matching

```
matcherMatches* adapters in utils.ts:
    Provides consistent matching across app and agent events

    matcherMatchesEvent(matcher, event):
        ├── Check event type match
        ├── Evaluate all conditions
        └── Return boolean

    matcherMatchesCron(matcher, date):
        ├── Evaluate cron expression
        └── Return boolean

Avoid direct primitive-only matcher checks in feature code.
All condition gating goes through these adapters for consistency.
```

---

## 11. Integration Points

### Session Integration

```
SessionManager → Automation triggers:

  Session created → emit('SessionStart', { sessionId })
  Session completed → emit('SessionEnd', { sessionId })
  Session labeled → emit('LabelAdd', { sessionId, labelId })
  Session status changed → emit('StatusChange', { sessionId, from, to })
  Session flagged → emit('FlagChange', { sessionId, flagged })
```

### Agent Integration

```
Agent hooks → Automation triggers:

  PreToolUse:
    emit('PreToolUse', {
      sessionId, tool: 'browser_tool',
      args: { command: 'navigate https://...' }
    })

  PostToolUse:
    emit('PostToolUse', {
      sessionId, tool: 'browser_tool',
      result: { success: true, url: '...' }
    })
```

### Source Integration

```
Source changes → Automation triggers:

  Source added → emit('SourceChange', { type: 'added', sourceSlug })
  Source removed → emit('SourceChange', { type: 'removed', sourceSlug })
  Source updated → emit('SourceChange', { type: 'updated', sourceSlug })
```

---

## 12. Example Automations

### Example 1: Daily Standup Summary

```json
{
  "name": "Daily Standup",
  "enabled": true,
  "cron": "57 8 * * 1-5",
  "conditions": [
    { "type": "time", "timeOfDay": { "after": "08:00", "before": "09:00" } }
  ],
  "actions": [
    {
      "type": "prompt",
      "prompt": "Check Linear for my assigned issues due this week. Summarize progress and blockers."
    }
  ]
}
```

### Example 2: Urgent Issue Alert

```json
{
  "name": "Urgent Issue Alert",
  "enabled": true,
  "matcher": "LabelAdd",
  "conditions": [
    {
      "type": "state",
      "field": "labelId",
      "value": "urgent"
    }
  ],
  "actions": [
    {
      "type": "webhook",
      "url": "https://hooks.slack.com/services/...",
      "method": "POST",
      "body": "{\"text\": \"Urgent label added to session {{sessionName}}\"}"
    }
  ]
}
```

### Example 3: Post-Browse Summary

```json
{
  "name": "Browse Complete Summary",
  "enabled": true,
  "matcher": "PostToolUse",
  "conditions": [
    {
      "type": "state",
      "field": "tool",
      "value": "browser_tool"
    }
  ],
  "actions": [
    {
      "type": "webhook",
      "url": "https://api.example.com/log",
      "method": "POST",
      "body": "{\"event\": \"browser_tool_used\", \"session\": \"{{sessionId}}\"}"
    }
  ]
}
```

---

## 13. Performance Considerations

```
┌────────────────────────────────┬──────────────────────────┐
│ Metric                         │ Value                    │
├────────────────────────────────┼──────────────────────────┤
│ Event processing latency       │ <1ms per automation      │
│ Condition evaluation           │ <100μs per condition     │
│ Cron check frequency           │ Every 60 seconds         │
│ History max size               │ 1000 entries             │
│ History per automation         │ 20 entries (rolling)     │
│ Webhook timeout                │ 10 seconds               │
│ Webhook max retries            │ 5                        │
│ Retry backoff                  │ 1s → 2s → 4s → 8s → 16s │
│ Field truncation               │ 2000 characters          │
└────────────────────────────────┴──────────────────────────┘
```
