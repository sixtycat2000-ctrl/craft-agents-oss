# Browser Tools — Embedded Browser Automation

> How Craft Agents provides browser automation via CDP, manages session-bound browser instances, and delivers live visual feedback to users.

---

## 1. Three-Layer Architecture

```
┌──────────────────────────────────────────────────────────────┐
│ Layer 1: Tool Interface (packages/shared — zero Electron)    │
│                                                              │
│ browser-tools.ts         → BrowserPaneFns interface         │
│ browser-tool-runtime.ts  → CLI parser + command router       │
│ session-scoped-tools.ts  → registers browser_tool into MCP   │
│ callback-registry.ts     → Map<sessionId, callbacks>         │
├──────────────────────────────────────────────────────────────┤
│ Layer 2: Wiring Layer (server-core — headless agnostic)      │
│                                                              │
│ SessionManager.ts                                             │
│   mergeSessionScopedToolCallbacks(sessionId, {               │
│     browserPaneFns: {                                         │
│       navigate: (url) => bpm.navigate(id, url),              │
│       click: (ref) => bpm.clickElement(id, ref),             │
│       ...30 method bindings                                   │
│     }                                                         │
│   })                                                          │
├──────────────────────────────────────────────────────────────┤
│ Layer 3: Electron Backend (apps/electron — desktop only)     │
│                                                              │
│ BrowserPaneManager.ts                                         │
│   implements IBrowserPaneManager                             │
│   owns instances: Map<id, BrowserInstance>                   │
│   binds sessions → windows (1:1, lazy, reusable)             │
│   calls CDP (Chrome DevTools Protocol) under the hood        │
└──────────────────────────────────────────────────────────────┘
```

**Key insight:** Layers 1 and 2 have zero Electron imports. Only Layer 3 touches `BrowserWindow`.

---

## 2. BrowserPaneFns Interface

The 30-method contract between tool logic and browser backend:

```typescript
interface BrowserPaneFns {
  // Navigation
  navigate(url: string): Promise<NavigateResult>
  goBack(): Promise<NavigateResult>
  goForward(): Promise<NavigateResult>
  reload(): Promise<NavigateResult>

  // Observation
  snapshot(): Promise<SnapshotResult>          // Accessibility tree
  screenshot(opts?: ScreenshotOpts): Promise<ScreenshotResult>
  getPageInfo(): Promise<PageInfoResult>       // URL, title, metrics

  // Interaction
  click(ref: string): Promise<ClickResult>
  fill(ref: string, value: string): Promise<FillResult>
  type(text: string): Promise<TypeResult>
  press(key: string): Promise<PressResult>
  select(ref: string, value: string): Promise<SelectResult>

  // Scrolling
  scroll(direction: 'up' | 'down', amount?: number): Promise<ScrollResult>

  // Clipboard
  setClipboard(text: string): Promise<void>
  getClipboard(): Promise<string>
  paste(): Promise<void>

  // Advanced
  evaluate(script: string): Promise<EvaluateResult>
  find(query: string): Promise<FindResult>
  drag(fromRef: string, toRef: string): Promise<void>
  uploadFile(ref: string, filePath: string): Promise<void>
  download(url: string): Promise<DownloadResult>

  // Control
  show(): Promise<void>
  hide(): Promise<void>
  releaseControl(): Promise<void>
  takeControl(): Promise<void>

  // Window management
  close(): Promise<void>
  focus(): Promise<void>

  // State
  getConsoleLogs(): Promise<ConsoleLog[]>
  getNetworkLogs(): Promise<NetworkLog[]>
  detectChallenge(): Promise<ChallengeResult>
}
```

---

## 3. CLI-in-a-Tool Pattern

### Single Tool, 40+ Commands

```typescript
// Schema
browser_tool({ command: z.union([z.string(), z.array(z.string())]) })

// Usage examples:
browser_tool({ command: "navigate https://example.com" })
browser_tool({ command: "snapshot" })
browser_tool({ command: "click @e5" })
browser_tool({ command: "fill @e3 hello@example.com" })
browser_tool({ command: "fill @e1 John; fill @e2 john@ex.com; click @submit" })  // batch
browser_tool({ command: ["evaluate", "var x = 1; x + 2"] })  // array mode for semicolons
```

### Command Parser

```
tokenizeCommand(input):
  "navigate https://example.com"     → ["navigate", "https://example.com"]
  'fill @e3 "hello world"'          → ["fill", "@e3", "hello world"]
  "fill @e3 hello\\nworld"          → ["fill", "@e3", "hello\nworld"]

splitBatchCommands(input):
  "fill @e1 val1; fill @e2 val2"    → ["fill @e1 val1", "fill @e2 val2"]
  'fill @e1 "val; with; semis"'     → ["fill @e1 \"val; with; semis\""]

Batch stops after navigation commands:
  "navigate url; snapshot"           → stops after "navigate url"
  "click @e1; snapshot"              → stops after "click @e1"
```

### Command Reference

```
┌─────────────────────────────────────────────────────────────┐
│ Navigation          │ Observation                           │
│─────────────────────│───────────────────────────────────────│
│ navigate <url>      │ snapshot                             │
│ back                │ screenshot [full]                    │
│ forward             │ page_info                            │
│ reload              │ find <query>                         │
│                     │ console_logs                         │
│ Interaction         │ network_logs                         │
│─────────────────────│───────────────────────────────────────│
│ click <ref>         │ Clipboard                            │
│ fill <ref> <value>  │──────────────────────────────────────│
│ type <text>         │ set_clipboard <text>                 │
│ press <key>         │ get_clipboard                        │
│ select <ref> <val>  │ paste                                │
│ scroll <dir> [amt]  │                                      │
│                     │ Window                               │
│ Advanced            │──────────────────────────────────────│
│─────────────────────│ show                                 │
│ evaluate <script>   │ hide                                 │
│ upload <ref> <path> │ close                                │
│ download <url>      │ focus                                │
│ drag <from> <to>    │ release                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 4. Accessibility Tree Snapshots

### Snapshot Format

```
URL: https://github.com/user/repo
Title: user/repo: Description
Elements: 156 (button:18, link:42, textbox:3, heading:8)

  [1] [heading] "user/repo"
  [2] [link] "Issues"
  [3] [link] "Pull requests"
  [4] [button] "New issue"
  [5] [textbox] "Search" value=""
  [6] [link] "README.md"
  [7] [heading] "Project Name"
  [8] [text] "A description of the project..."
  [9] [button] "Star"
  [10] [button] "Fork"
```

### Token Efficiency

```
Full HTML:           ~50,000 tokens
Accessibility tree:  ~1,500 tokens
Reduction:           ~97%

Per-element info:
  [N]     → Numeric ref for deterministic interaction
  [role]  → ARIA role (button, link, textbox, etc.)
  "name"  → Accessible name
  value=  → Current value (for inputs)
```

### @eN Ref Resolution

```
Agent sees: [5] [textbox] "Search" value=""
Agent calls: browser_tool({ command: "fill @5 my query" })

Resolution:
  @5 → CDP element reference → DOM node → fill value

Steps:
  1. Runtime parses "@5" → element index 5
  2. fns.fill("5", "my query")
  3. BrowserPaneManager → BrowserCDP
  4. CDP command: DOM.focus + Input.dispatchKeyEvent
  5. Value set in DOM → success
```

---

## 5. BrowserPaneManager (Electron Backend)

### Instance Structure

```
BrowserInstance
├── id: string                     // Unique instance ID
├── window: BrowserWindow          // Frameless Electron window
├── toolbarView: BrowserView       // URL bar, navigation controls
├── pageView: BrowserView          // Actual web content
├── nativeOverlayView: BrowserView // Cursor animation, agent badge
├── cdp: BrowserCDP                // Chrome DevTools Protocol client
├── currentUrl: string
├── title: string
├── favicon: string | null
├── boundSessionId: string | null  // 1:1 session binding
└── ownerType: 'session' | 'manual'
```

### Window Composition

```
┌──────────────────────────────────────────────┐
│ BrowserWindow (frameless)                     │
│                                               │
│ ┌───────────────────────────────────────────┐│
│ │ Toolbar BrowserView (48px)                ││
│ │ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────────┐  ││
│ │ │ ←    │ │ →    │ │ ↻    │ │ URL bar   │  ││
│ │ └──────┘ └──────┘ └──────┘ └──────────┘  ││
│ └───────────────────────────────────────────┘│
│ ┌───────────────────────────────────────────┐│
│ │                                           ││
│ │ Page Content BrowserView                  ││
│ │ (actual web page)                         ││
│ │                                           ││
│ │                                           ││
│ └───────────────────────────────────────────┘│
│ ┌───────────────────────────────────────────┐│
│ │ Native Overlay BrowserView (transparent)  ││
│ │                                           ││
│ │   [cursor animation] ─── [Agent badge]    ││
│ │                                           ││
│ └───────────────────────────────────────────┘│
└──────────────────────────────────────────────┘
```

### Session Binding

```
createForSession(sessionId):
    │
    ├── Check existing binding:
    │   │
    │   ├── Session has bound instance:
    │   │   → Return existing instance (reuse)
    │   │
    │   ├── Instance bound to different session:
    │   │   → Unbind from old session
    │   │   → Bind to new session
    │   │   → Window persists (reuse)
    │   │
    │   └── No existing instance:
    │       → Create new BrowserWindow
    │       → Create 3 BrowserViews
    │       → Initialize CDP connection
    │       → Bind to session
    │
    └── Return instance ID

Key properties:
  - Lazy: created on first browser_tool call
  - 1:1: each session bound to exactly one instance
  - Persistent: window survives session end
  - Reusable: unbound windows can be rebound
```

### Cookie Partition

```
Each browser instance uses separate Electron session:
  session.fromPartition('persist:browser-pane')

Benefits:
  - Cookie isolation between sessions
  - Separate localStorage
  - Separate cache
  - Auth tokens don't leak between sessions
```

---

## 6. BrowserCDP — Chrome DevTools Protocol

### CDP Connection

```
BrowserCDP wraps webContents.debugger:

connect():
  webContents.debugger.attach('1.3')

disconnect():
  webContents.debugger.detach()
  (Idle timer: detach after 5s of inactivity)

sendCommand(method, params):
  webContents.debugger.sendCommand(method, params)
```

### Accessibility Tree via CDP

```
snapshot():
    │
    ├── CDP: Accessibility.getFullAXTree()
    │   → Returns complete accessibility tree
    │
    ├── Filter relevant nodes:
    │   Keep: button, link, textbox, heading, combobox, checkbox, etc.
    │   Skip: generic, presentation, none roles
    │
    ├── Assign @eN refs:
    │   node.ref → index → @eN format
    │
    ├── Build snapshot string:
    │   URL, title, element counts
    │   Then each element on its own line
    │
    └── Return SnapshotResult { text, elements, url, title }
```

### Element Interaction via CDP

```
click(ref):
    │
    ├── Resolve ref to DOM node:
    │   CDP: Accessibility.getFullAXTree() → find node by ref
    │   CDP: DOM.describeNode(nodeId) → get backendNodeId
    │
    ├── Get element bounds:
    │   CDP: DOM.getBoxModel(backendNodeId) → get center point
    │
    ├── Move cursor:
    │   CDP: Input.dispatchMouseEvent({ type: 'mouseMoved', x, y })
    │
    ├── Click:
    │   CDP: Input.dispatchMouseEvent({ type: 'mousePressed', button: 'left', x, y })
    │   CDP: Input.dispatchMouseEvent({ type: 'mouseReleased', button: 'left', x, y })
    │
    └── Wait for navigation (if click triggers navigation)
```

### Idle Detach Timer

```
CDP connection kept alive for 5 seconds after last operation:

  Operation → reset timer
  Timer fires → detach debugger

  Reason: CDP debugger attachment blocks some page operations
  Tradeoff: re-attach cost (~50ms) vs page interference
```

---

## 7. CAPTCHA / Challenge Detection

### Detection Points

```
detectChallenge() runs after:
  ├── navigate(url)     — catches Cloudflare, reCAPTCHA on load
  ├── click(ref)        — catches challenges triggered by clicking
  └── snapshot()        — catches challenges on near-empty pages
```

### Detection Logic

```
detectChallenge():
    │
    ├── Check URL patterns:
    │   /challenge*, /captcha*, /verify*
    │
    ├── Check page content:
    │   "Just a moment..." (Cloudflare)
    │   "Verify you are human" (reCAPTCHA)
    │   "Checking your browser" (generic)
    │
    ├── Check element count:
    │   0-2 actionable elements → likely challenge page
    │
    └── Return ChallengeResult:
        { detected: boolean, type?: string, message?: string }
```

### Handoff to User

```
Challenge detected during agent browsing:
    │
    ▼
1. Agent releases control:
    await fns.releaseControl()
    // Shows browser window to user
    // Stops agent control overlay
    │
    ▼
2. Return message to agent:
    {
      output: 'Security verification detected. Browser shown — please complete the check.',
      appendReleaseHint: false
    }
    │
    ▼
3. Agent pauses:
    Tells user: "I've detected a security check.
    Please complete it in the browser window."
    │
    ▼
4. User completes verification manually
    │
    ▼
5. Agent resumes:
    Takes snapshot to see updated page
    Continues task
```

---

## 8. Visual Feedback System

### Agent Control Overlay

```
Overlay states:
┌─────────────────────────────────────────────────┐
│                                                  │
│  IDLE:     No overlay, user has full control     │
│                                                  │
│  ACTIVE:   Animated cursor following agent       │
│            Pulsing border around window          │
│            Badge: "Craft Agents are working..."  │
│            User sees agent actions in real-time   │
│                                                  │
│  FAILED:   Red border                            │
│            Badge: "Action failed"                │
│            Cursor stops at failure point          │
│                                                  │
└─────────────────────────────────────────────────┘
```

### Cursor Animation

```
When agent moves cursor:
  1. CDP reports new cursor position (x, y)
  2. Overlay renders cursor at (x, y)
  3. Smooth animation between positions (CSS transition)
  4. Custom cursor image (agent-branded)
  5. Click animation (pulse effect)

Performance:
  Cursor updates batched at 60fps
  Overlay renders via requestAnimationFrame
  No impact on page content rendering
```

### Theme Color Extraction

```
After page load:
    │
    ├── 1. Check <meta name="theme-color">
    │   → If present, use as toolbar accent
    │
    ├── 2. Fallback: sample viewport pixels
    │   → Take screenshot
    │   → Sample top-left 100x100 region
    │   → Extract dominant color
    │
    └── 3. Apply to toolbar:
        → Background gradient
        → URL bar tint
        → Navigation button colors
```

---

## 9. Command Execution Pipeline

### Full Flow: fill + click

```
Agent: browser_tool({ command: "fill @e3 hello@example.com; click @submit" })
    │
    ▼
1. Batch splitting:
    splitBatchCommands("fill @e3 hello@example.com; click @submit")
    → ["fill @e3 hello@example.com", "click @submit"]
    │
    ▼
2. Execute first command: "fill @e3 hello@example.com"
    │
    ├── Tokenize: ["fill", "@e3", "hello@example.com"]
    ├── Route to fill handler
    │
    ├── Resolve @e3:
    │   Lazy CDP attach → get accessibility tree → find element 3
    │
    ├── CDP fill sequence:
    │   DOM.focus(element3)
    │   Clear existing value
    │   Input.insertText({ text: "hello@example.com" })
    │
    ├── Detect challenge:
    │   detectChallenge() → { detected: false }
    │
    └── Return: { output: "Filled 'hello@example.com' into email field" }
    │
    ▼
3. Execute second command: "click @submit"
    │
    ├── Tokenize: ["click", "@submit"]
    ├── Route to click handler
    │
    ├── Resolve @submit:
    │   Get element with name "submit"
    │
    ├── CDP click sequence:
    │   Get element center (x, y)
    │   Input.dispatchMouseEvent(mousePressed)
    │   Input.dispatchMouseEvent(mouseReleased)
    │
    ├── Detect challenge:
    │   detectChallenge() → { detected: false }
    │
    ├── Wait for navigation:
    │   webContents 'did-navigate' event (5s timeout)
    │
    └── Return: { output: "Clicked submit. Navigated to /dashboard" }
```

---

## 10. Feature Gate

### Browser Tool Toggle

```
Settings → "Enable built-in browser tool" toggle

Implementation:
  getBrowserToolEnabled()
    → Reads from config.json: settings.browserToolEnabled
    → Default: true (enabled)

  When disabled:
    getSessionScopedTools() skips browser_tool creation
    → Agent doesn't see browser_tool
    → No BrowserWindow created
    → No CDP connections

  When changed:
    invalidateAllSessionToolsCaches()
    → Rebuilds tool list for all active sessions
    → New tool set on next agent turn
```

### Coexistence with MCP Browser Tools

```
User has Playwright MCP server configured:
    → Disable built-in browser_tool
    → Agent uses Playwright MCP tools instead
    → No conflict between two browser backends

Reason for toggle:
    Many users already have browser MCP servers
    Running two browser tools simultaneously causes conflicts
    Cookie/profile conflicts between tools
```

---

## 11. Performance Characteristics

```
┌────────────────────────────┬──────────────────────┐
│ Operation                  │ Latency              │
├────────────────────────────┼──────────────────────┤
│ Navigate to page           │ 1-5s (network)       │
│ Accessibility snapshot     │ 50-200ms             │
│ Click element              │ 50-100ms             │
│ Fill text field            │ 30-50ms              │
│ Screenshot                 │ 100-300ms            │
│ JavaScript evaluate        │ 10-50ms              │
│ Find text                  │ 20-50ms              │
│ Challenge detection        │ 10-20ms              │
│ CDP attach/detach          │ 30-50ms              │
└────────────────────────────┼──────────────────────┤
│ Memory per browser window  │ 50-150MB             │
│ CDP overhead               │ ~5MB                 │
│ Overlay overhead           │ ~10MB (BrowserView)  │
└────────────────────────────┴──────────────────────┘
```

---

## 12. Security Considerations

```
┌─────────────────────────────────────────────────────┐
│ Isolation                                           │
│   Separate Electron session per browser partition   │
│   No cookie/token sharing between sessions          │
│   No localStorage cross-contamination               │
│                                                     │
│ Agent Control                                       │
│   Agent can only control its bound browser window   │
│   User can take back control at any time            │
│   release command stops agent immediately           │
│                                                     │
│ Challenge Handling                                  │
│   CAPTCHAs never auto-solved                        │
│   Always handed off to human user                   │
│   Agent pauses until user completes                 │
│                                                     │
│ Network                                             │
│   Browser uses system proxy settings               │
│   CDP only accessible from main process             │
│   No remote debugging exposed                       │
└─────────────────────────────────────────────────────┘
```
