# UI Components — Renderer Architecture

> How the React renderer is structured with Jotai atoms for state, URL-driven navigation, per-session isolation, and a shared component library.

---

## 1. Component Hierarchy

```
App
├── ThemeProvider
│   └── Loads theme, applies CSS variables
├── NavigationProvider
│   ├── URL sync (pushState/popState)
│   ├── History management
│   └── Route reconciliation
├── AppShellProvider
│   └── Shell-level context (shortcuts, focus)
└── AppShell
    ├── TopBar
    │   ├── WorkspaceSwitcher
    │   ├── BreadcrumbNav
    │   ├── PermissionModeSelector
    │   └── TransportConnectionBanner
    ├── LeftSidebar
    │   ├── SessionFilterTabs (All/Flagged/Archived)
    │   ├── SessionList (virtualized)
    │   ├── SourcesListPanel
    │   ├── SkillsListPanel
    │   └── NewChatButton
    ├── PanelStackContainer
    │   ├── Panel[0] (always present)
    │   │   ├── PanelHeader
    │   │   └── MainContentPanel
    │   │       ├── ChatDisplay
    │   │       │   ├── TurnCard[] (assistant messages)
    │   │       │   ├── UserMessageBubble[] (user messages)
    │   │       │   └── ChatInputZone
    │   │       │       └── FreeFormInput
    │   │       └── NavigatorPanel (file tree)
    │   ├── PanelResizeSash
    │   └── Panel[1] (optional, side-by-side)
    │       └── ...same structure
    └── RightSidebar (collapsible)
        ├── FilesPanel
        ├── SourcesPanel
        └── SkillsPanel
```

---

## 2. State Management (Jotai)

### Session Atoms

```typescript
// Per-session atom family — isolated state
sessionAtomFamily = atomFamily((sessionId: string) =>
  atom<Session | null>(null)
)

// Lightweight metadata map for sidebar
sessionMetaMapAtom = atom<Map<string, SessionMeta>>(new Map())

// Ordered session IDs for rendering
sessionIdsAtom = atom<string[]>([])

// Track which sessions have full messages loaded
loadedSessionsAtom = atom<Set<string>>(new Set())

// Active session in focused panel
activeSessionIdAtom = atom<string | null>(null)
```

### Why Atom Family

```
Problem: Streaming text in Session A re-renders Session B

Solution: Per-session atoms with atomFamily

┌─────────────────────────────────────────────────────┐
│ Session A atom ← streaming update → only A re-renders│
│ Session B atom ← unchanged → B stays still           │
│ Session C atom ← unchanged → C stays still           │
└─────────────────────────────────────────────────────┘

Memory: Each atom created on demand, garbage collected
        when session removed from view
```

### Panel Stack Atoms

```typescript
// Panel management
panelStackAtom = atom<PanelStackEntry[]>([])
focusedPanelIdAtom = atom<string | null>(null)
focusedPanelIndexAtom = atom<number>(0)
focusedPanelRouteAtom = atom<ViewRoute>('allSessions')

// Actions
pushPanelAtom = atom(null, (get, set, route: ViewRoute) => { ... })
closePanelAtom = atom(null, (get, set, panelId: string) => { ... })
resizePanelsAtom = atom(null, (get, set, proportions: number[]) => { ... })
```

### Browser Instance Atoms

```typescript
browserInstancesMapAtom = atom<Map<string, BrowserInstanceInfo>>(new Map())
activeBrowserInstanceIdAtom = atom<string | null>(null)
removedBrowserInstanceIdsAtom = atom<Set<string>>(new Set())
```

---

## 3. Navigation System

### URL-Driven Architecture

```
URL is the single source of truth for navigation state.

URL format:
  ?ws={workspaceSlug}
  &route={focusedRoute}
  &panels={route1:prop1,route2:prop2}
  &fi={focusedPanelIndex}
  &sidebar={type:id}

Examples:
  ?ws=my-workspace&route=allSessions/session/abc123
  ?ws=my-workspace&panels=allSessions:0.6,flagged:0.4&fi=1
  ?ws=my-workspace&route=settings/shortcuts&sidebar=files/docs
```

### Navigation Flow

```
User clicks session in sidebar
    │
    ▼
1. navigate(routes.view.allSessions(sessionId))
    │
    ▼
2. Route parsed to NavigationState:
    {
      panels: [
        { route: 'allSessions/session/abc123', proportion: 1.0 }
      ],
      focusedIndex: 0,
      sidebar: null
    }
    │
    ▼
3. Atoms updated:
    set(panelStackAtom, newPanels)
    set(focusedPanelRouteAtom, 'allSessions/session/abc123')
    │
    ▼
4. URL updated:
    window.history.pushState(state, '', newUrl)
    │
    ▼
5. Components re-render based on atom changes
```

### History Management

```
Browser back/forward:
    │
    ▼
1. popstate event fires
    │
    ▼
2. Parse URL params to NavigationState
    │
    ▼
3. Panel reconciliation:
    │   Match panels by route
    │   Preserve React keys (avoid unmount/remount)
    │   Keep scroll position, streaming state, input text
    │
    ▼
4. Update atoms
    │
    ▼
5. Components re-render without losing state
```

### Route Types

```
View routes (navigation state):
  allSessions                              → Session list + no session selected
  allSessions/session/{id}                 → Session list + specific session
  flagged                                  → Flagged sessions list
  flagged/session/{id}                     → Flagged list + session
  state/{statusId}/session/{id}            → Filtered by status + session
  label/{labelId}/session/{id}             → Filtered by label + session
  sources/source/{slug}                    → Source detail
  settings/{subpage}                       → Settings page

Action routes (side effects, then redirect):
  action/newSession?input=text&send=true   → Create session, send message
  action/rename-session/{id}?name=New      → Rename session
  action/delete-session/{id}               → Delete session
  action/flag-session/{id}                 → Toggle flag
```

---

## 4. Event Processing Pipeline

### How Agent Events Reach the UI

```
Server pushes event:
    WsRpcServer.push('session:event', target, { type: 'text_delta', ... })
    │
    ▼
RoutedClient dispatches to listener:
    electronAPI.onSessionEvent((event) => processAgentEvent(event))
    │
    ▼
processAgentEvent(event) in App.tsx:
    │
    ├── switch (event.type):
    │
    │   case 'text_delta':
    │     set(updateStreamingContentAtom, sessionId, event.content)
    │     → Only that session's atom updates
    │     → ChatDisplay shows streaming text
    │
    │   case 'tool_call':
    │     set(appendMessageAtom, sessionId, { type: 'tool_use', ... })
    │     → TurnCard renders tool card
    │
    │   case 'tool_result':
    │     set(updateToolResultAtom, sessionId, toolUseId, result)
    │     → Tool card updates with result
    │
    │   case 'permission_request':
    │     set(addPendingPermission, sessionId, request)
    │     → PermissionRequestDialog renders
    │
    │   case 'complete':
    │     set(finalizeTurnAtom, sessionId)
    │     set(clearStreamingAtom, sessionId)
    │     → Session marked as idle
    │
    │   case 'error':
    │     set(setSessionErrorAtom, sessionId, error)
    │     → Error banner shown
    │
    └── Debounced metadata update:
        set(updateSessionMetaAtom, sessionId, newMeta)
        → Sidebar re-sorts
```

---

## 5. Chat Display

### TurnCard Component

```
TurnCard renders assistant messages with rich content:

┌───────────────────────────────────────────────────┐
│ TurnCard                                           │
│ ┌─────────────────────────────────────────────────┤
│ │ Markdown content (react-markdown)               │
│ │ - Bold, italic, links                           │
│ │ - Code blocks (Shiki syntax highlighting)       │
│ │ - Mermaid diagrams (rendered SVG)               │
│ │ - Math equations (KaTeX)                        │
│ │ - Data tables (interactive)                     │
│ │ - Collapsible sections                          │
│ │                                                 │
│ │ Tool calls (inline):                            │
│ │ ┌──────────────────────────────────────────┐   │
│ │ │ 🔧 Read src/main.ts           ✓ Success  │   │
│ │ │ ─────────────────────────────────────── │   │
│ │ │ (expandable: shows file content)          │   │
│ │ └──────────────────────────────────────────┘   │
│ │                                                 │
│ │ ┌──────────────────────────────────────────┐   │
│ │ │ 🔧 Bash npm test               ✗ Failed  │   │
│ │ │ ─────────────────────────────────────── │   │
│ │ │ (expandable: shows terminal output)       │   │
│ │ └──────────────────────────────────────────┘   │
│ │                                                 │
│ │ Diff viewer (multi-file):                        │
│ │ ┌──────────────────────────────────────────┐   │
│ │ │ Unified/Split diff view                   │   │
│ │ │ with syntax highlighting                  │   │
│ │ └──────────────────────────────────────────┘   │
│ │                                                 │
│ │ Browser screenshots:                             │
│ │ ┌──────────────────────────────────────────┐   │
│ │ │ [Screenshot of page]                      │   │
│ │ └──────────────────────────────────────────┘   │
│ └─────────────────────────────────────────────────┤
│ [Copy] [Retry] [Edit] [Branch]                     │
└───────────────────────────────────────────────────┘
```

### Markdown Rendering Modes

```
Three rendering modes:

1. terminal — plain text, no formatting
   Used for: code terminal output, simple text

2. minimal — basic markdown (bold, links, code)
   Used for: tool results, status messages

3. full — complete markdown with extensions
   Used for: assistant responses
   Features:
   - Syntax highlighting (Shiki, 50+ languages)
   - Mermaid diagrams (rendered client-side)
   - Math equations (KaTeX)
   - Data tables (interactive sort/filter)
   - Diff blocks (unified/split view)
   - Image preview (inline + fullscreen)
   - PDF preview (embedded viewer)
   - Collapsible sections
   - Code annotations (inline comments)
```

---

## 6. Chat Input System

### FreeFormInput

```
┌─────────────────────────────────────────────────────────┐
│ FreeFormInput (104KB component)                          │
│                                                          │
│ ┌─────────────────────────────────────────────────────┐ │
│ │ Text input area (auto-resize, max 20 lines)         │ │
│ │ - Markdown preview on right                         │ │
│ │ - @-mention for sources                             │ │
│ │ - File drag & drop support                          │ │
│ └─────────────────────────────────────────────────────┘ │
│ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌───────────────┐ │
│ │Attach│ │Model │ │ Mode │ │Think │ │    Send  ▶    │ │
│ └──────┘ └──────┘ └──────┘ └──────┘ └───────────────┘ │
└─────────────────────────────────────────────────────────┘

Features:
  - Multi-line input with auto-resize
  - File attachment (drag & drop, file picker)
  - Model selector dropdown
  - Permission mode selector
  - Thinking level selector
  - @-mention for source tools
  - Draft persistence (debounced save)
  - Input history (up arrow to recall)
```

### Draft Management

```
User types in session A → switches to session B → back to session A:
    │
    ▼
1. Leaving session A:
    Save draft to sessionDraftsRef: Map<sessionId, string>
    Debounced 300ms
    │
    ▼
2. Session B active:
    Load session B's draft (if any)
    │
    ▼
3. Return to session A:
    Restore draft from ref
    Cursor position preserved
```

---

## 7. Shared UI Package

### Package Structure

```
packages/ui/src/
├── components/
│   ├── chat/                    # Chat-specific components
│   │   ├── TurnCard.tsx         # Assistant message card
│   │   ├── UserMessageBubble.tsx# User message bubble
│   │   ├── SessionViewer.tsx    # Full session transcript
│   │   ├── InlineExecution.tsx  # Inline command display
│   │   └── AcceptPlanDropdown.tsx# Plan approval UI
│   ├── markdown/                # Markdown rendering
│   │   ├── Markdown.tsx         # Main renderer (3 modes)
│   │   ├── CodeBlock.tsx        # Syntax-highlighted code
│   │   ├── MarkdownDiffBlock.tsx# Diff visualization
│   │   ├── MarkdownMermaidBlock.tsx # Mermaid diagrams
│   │   ├── MarkdownDatatableBlock.tsx # Data tables
│   │   └── MarkdownImageBlock.tsx # Image handling
│   ├── code-viewer/             # Code display
│   │   ├── ShikiCodeViewer.tsx  # Syntax-highlighted viewer
│   │   └── ShikiDiffViewer.tsx  # Diff viewer
│   ├── overlay/                 # Fullscreen overlays
│   │   ├── CodePreviewOverlay.tsx
│   │   ├── MultiDiffPreviewOverlay.tsx
│   │   ├── TerminalPreviewOverlay.tsx
│   │   ├── ImagePreviewOverlay.tsx
│   │   └── PDFPreviewOverlay.tsx
│   ├── ui/                      # Base primitives
│   │   ├── Island.tsx           # Animated popover
│   │   ├── FilterableSelectPopover.tsx
│   │   └── LoadingIndicator.tsx
│   └── annotations/             # Code annotation system
│       ├── AnnotationOverlayLayer.tsx
│       └── interaction-state-machine.ts
├── context/
│   ├── PlatformContext.tsx       # Platform abstraction
│   └── ShikiThemeContext.tsx     # Syntax highlighting
└── styles/
    └── globals.css               # Tailwind base styles
```

### Platform Context

```
PlatformContext abstracts platform-specific operations:

interface PlatformActions {
  onOpenFile?: (path: string) => void
  onOpenUrl?: (url: string) => void
  onOpenCodePreview?: (sessionId, toolUseId) => void
  onRevealInFinder?: (path: string) => void
  onCopyToClipboard?: (text: string) => Promise<void>
  // ... more actions
}

Usage:
  // Electron: delegates to electronAPI
  <PlatformProvider actions={{
    onOpenFile: (path) => window.electronAPI.openFile(path)
  }}>

  // Viewer: delegates to browser APIs
  <PlatformProvider actions={{
    onOpenUrl: (url) => window.open(url, '_blank')
  }}>
```

---

## 8. Overlay System

### Overlay Architecture

```
Overlays are fullscreen modals for content preview:

┌───────────────────────────────────────────────────┐
│ Fullscreen Overlay                                 │
│ ┌─────────────────────────────────────────────────┤
│ │ Header [filename.ts] [× Close] [◀ ▶ Navigate]  │
│ ├─────────────────────────────────────────────────┤
│ │                                                 │
│ │ Code Preview Overlay                            │
│ │ ┌─────────────────────────────────────────────┐│
│ │ │ Syntax-highlighted code                     ││
│ │ │ with annotations                            ││
│ │ │ and line numbers                            ││
│ │ └─────────────────────────────────────────────┘│
│ │                                                 │
│ ├─────────────────────────────────────────────────┤
│ │ [Zoom +] [Zoom -] [Copy] [Open in Editor]      │
│ └─────────────────────────────────────────────────┘
```

### Overlay Types

```
┌──────────────────────┬──────────────────────────────┐
│ Overlay              │ Content                      │
├──────────────────────┼──────────────────────────────┤
│ CodePreviewOverlay   │ Syntax-highlighted source    │
│ MultiDiffPreview     │ Side-by-side diff view       │
│ TerminalPreview      │ ANSI-colored terminal output │
│ ImagePreview         │ Image with zoom/pan          │
│ PDFPreview           │ PDF with page navigation     │
│ JSONPreview          │ Formatted JSON tree          │
│ ActivityCardsOverlay │ Card-based activity view     │
└──────────────────────┴──────────────────────────────┘
```

---

## 9. Theme System

### Theme Architecture

```
Theme sources (merged in order):

1. Preset themes (built-in)
   ├── Light (default)
   ├── Dark
   └── Scenic (gradient-based)

2. App-level overrides (~/.craft-agent/theme.json)
   Custom color overrides

3. Workspace-level theme
   Per-workspace color scheme

4. System theme (macOS dark/light)
   Auto-detected, synced with OS
```

### Theme Structure

```typescript
interface ThemeOverrides {
  colors?: {
    primary?: string         // Primary accent color
    background?: string      // Main background
    surface?: string         // Card/panel background
    text?: string            // Primary text
    textMuted?: string       // Secondary text
    border?: string          // Border color
    // ... 20+ more color tokens
  }
  typography?: {
    fontFamily?: string
    fontSize?: { base, sm, lg }
  }
  radius?: {
    default?: number         // Border radius
  }
}
```

### CSS Variable Generation

```
Theme → CSS variables:
  --color-primary: #6366f1
  --color-background: #ffffff
  --color-surface: #f8f9fa
  --color-text: #1a1a1a
  --radius-default: 8px

Applied at root level → all components inherit
Shiki themes synced for code highlighting
```

---

## 10. Keyboard Shortcuts

### Action Registry

```
Shortcuts registered globally, handled by renderer:

┌──────────────────┬──────────────┬─────────────────────────┐
│ Shortcut         │ Action       │ Description             │
├──────────────────┼──────────────┼─────────────────────────┤
│ Cmd+N            │ newChat      │ Create new session      │
│ Cmd+K            │ commandPalette│ Open search/command    │
│ Cmd+Enter        │ sendMessage  │ Send current message    │
│ Cmd+Shift+Enter  │ sendMessage  │ Send with attachments   │
│ Cmd+W            │ closePanel   │ Close focused panel     │
│ Cmd+\            │ toggleSidebar│ Toggle left sidebar     │
│ Cmd+]            │ nextPanel    │ Focus next panel        │
│ Cmd+[            │ prevPanel    │ Focus previous panel    │
│ Cmd+1-9          │ switchSession│ Switch to session N     │
│ Escape           │ cancel       │ Cancel current action   │
│ Cmd+Shift+A      │ toggleArchived│ Toggle archived view  │
│ Cmd+F            │ search       │ Focus search input      │
└──────────────────┴──────────────┴─────────────────────────┘
```

### Focus Zone Management

```
Focus zones for keyboard navigation:

Zone 1: Session list (sidebar)
  ↑/↓: Move selection
  Enter: Open session
  Delete: Delete session

Zone 2: Chat display (main panel)
  ↑: Scroll up
  ↓: Scroll down

Zone 3: Input area
  Enter: Send message
  Shift+Enter: New line
  ↑: Recall previous input

Tab cycles between zones
```

---

## 11. Performance Optimizations

### Virtual Scrolling

```
SessionList with 300+ sessions:
    │
    ▼
Virtual scrolling renders only visible items:
    - Buffer of 5 items above/below viewport
    - Fixed item height (64px) for calculation
    - Scroll position maintained during updates

Result: 300 sessions → ~10 DOM nodes rendered
```

### Lazy Loading

```
Session data loading strategy:

Initial load:
    getSessions(workspaceId)
    → Returns SessionMeta[] (id, name, preview, timestamps)
    → ~1KB per session, ~300KB total for 300 sessions

On session open:
    getSessionMessages(sessionId)
    → Returns full StoredSession with messages
    → ~100KB per session
    → Added to loadedSessionsAtom

Memory-efficient:
    Only 1-3 sessions fully loaded at a time
    Unloaded sessions have metadata only
```

### Code Splitting

```
Lazy-loaded components:
    const SettingsPanel = lazy(() => import('./SettingsPanel'))
    const OnboardingFlow = lazy(() => import('./OnboardingFlow'))
    const AutomationEditor = lazy(() => import('./AutomationEditor'))

Reduces initial bundle by ~200KB
```

---

## 12. Context System

### React Contexts

```
┌──────────────────────────────────────────────────────────┐
│ Context             │ Purpose                           │
├─────────────────────┼──────────────────────────────────┤
│ ThemeContext        │ Theme colors, dark mode, CSS vars │
│ NavigationContext   │ Routes, history, panel management │
│ PlatformContext     │ Platform-specific operations      │
│ SessionContext      │ Active session data, streaming    │
│ AppShellContext     │ Shortcuts, focus, global actions  │
│ ShikiThemeContext   │ Syntax highlighting configuration │
└─────────────────────┴──────────────────────────────────┘
```

---

## 13. Accessibility

### Keyboard Navigation

```
Roving tab index for list navigation:
    - Arrow keys move focus between items
    - Enter/Space activates focused item
    - Tab moves between focus zones

Focus zones:
    - Session list
    - Chat display
    - Input area
    - Panel stack
```

### Screen Reader Support

```
ARIA labels on interactive elements:
    - Session items: aria-label="Session: {name}, {status}"
    - Tool cards: aria-label="Tool: {name}, {status}"
    - Buttons: aria-label describes action

Semantic HTML:
    - <nav> for sidebar
    - <main> for content
    - <aside> for secondary panels
    - <article> for session messages
```
