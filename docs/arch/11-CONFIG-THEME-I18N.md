# Configuration, Themes & Internationalization

> How Craft Agents manages configuration files, user preferences, the theme engine, and multi-language support across the application.

---

## 1. Configuration System

### Config File Locations

```
~/.craft-agent/
├── config.json                # Main application config
├── preferences.json           # User preferences
├── theme.json                 # App-level theme overrides
├── themes/                    # Preset theme files
│   ├── light.json
│   ├── dark.json
│   └── scenic.json
├── credentials.enc            # Encrypted credential store
├── config-defaults.json       # Bundled defaults (read-only)
└── workspaces/                # Workspace directories
    └── {slug}/
        ├── config.json        # Workspace settings
        ├── sources/           # Source configurations
        ├── sessions/          # Session data
        ├── skills/            # Skill configurations
        ├── statuses/          # Status workflow config
        ├── labels/            # Label system config
        ├── automations.json   # Automation rules
        └── permissions.json   # Workspace permissions
```

### Config.json Schema

```typescript
interface StoredConfig {
  // LLM Configuration
  llmConnections: LlmConnection[]
  defaultLlmConnection: string
  defaultThinkingLevel: 'none' | 'low' | 'medium' | 'high' | 'max'

  // Workspace Management
  workspaces: WorkspaceEntry[]
  activeWorkspaceId: string
  activeSessionId: string | null

  // Application Settings
  settings: {
    notifications: boolean
    theme: 'light' | 'dark' | 'system'
    inputSettings: {
      sendOnEnter: boolean
      spellCheck: boolean
    }
    proxy: {
      type: 'none' | 'http' | 'https' | 'socks5'
      host: string
      port: number
      auth?: { username: string; password: string }
    } | null
    browserToolEnabled: boolean
    powerManagement: {
      preventIdle: boolean
    }
  }

  // Server Configuration (embedded mode)
  serverConfig?: {
    port: number
    host: string
  }
}
```

### Config Operations

```
loadStoredConfig():
    │
    ├── Read ~/.craft-agent/config.json
    │   File missing? → return defaults
    │
    ├── Parse JSON
    │
    ├── Validate against schema:
    │   LlmConnections valid?
    │   Workspace entries exist?
    │   Settings values in range?
    │
    ├── Apply migrations:
    │   Legacy format → current format
    │   Missing fields → defaults
    │
    ├── Convert portable paths:
    │   ~/ → expand to home directory
    │
    └── Return validated config

saveConfig(config):
    │
    ├── Validate before write
    │
    ├── Convert paths to portable format:
    │   /Users/yul/... → ~/...
    │
    ├── Atomic write:
    │   config.json.tmp → config.json
    │
    └── Trigger change callbacks
```

### ConfigWatcher

```
ConfigWatcher monitors all configuration files:

Monitors:
  ├── ~/.craft-agent/config.json        → onConfigChange
  ├── ~/.craft-agent/preferences.json   → onPreferencesChange
  ├── ~/.craft-agent/theme.json         → onAppThemeChange
  ├── ~/.craft-agent/themes/*.json      → onPresetThemeChange
  ├── workspaces/*/config.json          → onWorkspaceConfigChange
  ├── workspaces/*/sources/*/config.json → onSourceChange
  ├── workspaces/*/sessions/*/session.jsonl → onSessionChange
  └── workspaces/*/automations.json     → onAutomationsChange

Implementation:
  chokidar.watch(paths, { ignoreInitial: true })
  Debounce: 100ms (300ms for session files on Windows)

  On file change:
    1. Read new content
    2. Validate
    3. Fire registered callbacks
    4. Push events to renderer if needed
```

---

## 2. LLM Connections

### Connection Management

```
LlmConnection:
├── slug: string              // Unique identifier
├── name: string              // Display name
├── provider: string          // anthropic, google, openai, etc.
├── model: string             // Default model for this connection
├── apiBaseUrl: string        // API endpoint URL
├── authType: string          // api_key, oauth, iam, service_account
├── createdAt: string
└── updatedAt: string

Supported Providers:
┌──────────────────┬──────────────────────────────────────────┐
│ Provider         │ Auth Type                                │
├──────────────────┼──────────────────────────────────────────┤
│ Anthropic        │ API key, Claude OAuth                    │
│ Google AI Studio │ API key, OAuth                           │
│ OpenAI           │ API key                                  │
│ ChatGPT Plus     │ OAuth (device code flow)                 │
│ GitHub Copilot   │ OAuth (device code flow)                 │
│ OpenRouter       │ API key                                  │
│ Ollama           │ None (local)                             │
│ Amazon Bedrock   │ IAM credentials                          │
│ Google Vertex    │ Service account                          │
│ Custom           │ API key, bearer, OAuth                   │
└──────────────────┴──────────────────────────────────────────┘
```

### Connection Operations

```
addLlmConnection(config):
    │
    ├── Generate slug from name
    ├── Validate provider and auth type
    ├── Store credentials in credential manager
    ├── Add to config.llmConnections
    └── Save config

testLlmConnection(slug):
    │
    ├── Load connection config
    ├── Load credentials
    ├── Send test request to API
    │   POST /v1/messages { model, max_tokens: 10, messages: [...] }
    │
    ├── Success → return { success: true, model }
    └── Failure → return { success: false, error }

listModels(slug):
    │
    ├── Query provider API for available models
    ├── Cache results (5-minute TTL)
    └── Return model list
```

---

## 3. User Preferences

### Preferences Schema

```typescript
interface UserPreferences {
  // Profile
  name: string
  timezone: string
  location: string
  language: string               // en, es, zh-Hans
  notes: string                  // Custom instructions for agent

  // Diff Viewer
  diffViewer: {
    defaultView: 'unified' | 'split'
    showLineNumbers: boolean
    contextLines: number
  }

  // Co-authorship
  coAuthoredBy: {
    enabled: boolean
    name: string
    email: string
  }

  updatedAt: string
}
```

### Preference Injection into Prompts

```
formatPreferencesForPrompt(preferences):
    │
    ├── Build context string:
    │   "User name: Yul"
    │   "Timezone: America/New_York"
    │   "Location: New York"
    │   "Language: English"
    │   "Custom notes: prefers TypeScript over JavaScript"
    │
    └── Inject into system prompt:
        The agent uses this context for personalized responses
```

---

## 4. Theme Engine

### Theme Architecture

```
Theme sources (merged in priority order):

  1. System theme (macOS dark/light mode)
     Auto-detected via Electron nativeTheme

  2. Preset theme (built-in)
     ~/.craft-agent/themes/{light,dark,scenic}.json

  3. App-level overrides
     ~/.craft-agent/theme.json

  4. Workspace-level theme
     workspace config.json: defaults.colorTheme

Final theme = merge(system, preset, appOverrides, workspace)
```

### Theme Structure

```typescript
interface ThemeFile {
  id: string
  name: string
  type: 'light' | 'dark' | 'scenic'
  colors: {
    // Semantic colors
    primary: string
    primaryHover: string
    primaryForeground: string

    // Surface colors
    background: string
    surface: string
    surfaceHover: string
    surfaceActive: string

    // Text colors
    text: string
    textMuted: string
    textInverse: string

    // Border colors
    border: string
    borderHover: string

    // Status colors
    success: string
    warning: string
    error: string
    info: string

    // Chat-specific
    userBubble: string
    assistantBubble: string
    toolCard: string

    // Sidebar
    sidebarBackground: string
    sidebarItemHover: string
    sidebarItemActive: string

    // 20+ more tokens
  }
  radius?: {
    sm: number
    md: number
    lg: number
  }
}
```

### Scenic Theme

```
Scenic theme uses gradient backgrounds:

  background: "linear-gradient(135deg, #667eea 0%, #764ba2 100%)"
  surface: "rgba(255, 255, 255, 0.1)"

  Glassmorphism effect:
    backdrop-filter: blur(10px)
    border: 1px solid rgba(255, 255, 255, 0.2)

  Applied to:
    Sidebar, panels, cards, inputs
    NOT applied to code blocks (need opaque bg for readability)
```

### Theme Application

```
Theme → CSS variables → all components:

:root {
  --color-primary: #6366f1;
  --color-background: #ffffff;
  --color-surface: #f8f9fa;
  --color-text: #1a1a1a;
  --color-border: #e5e7eb;
  --radius-sm: 4px;
  --radius-md: 8px;
  --radius-lg: 12px;
}

.dark {
  --color-primary: #818cf8;
  --color-background: #0f0f0f;
  --color-surface: #1a1a1a;
  --color-text: #f5f5f5;
  --color-border: #2a2a2a;
}

Components use CSS variables:
  background: var(--color-surface);
  color: var(--color-text);
  border: 1px solid var(--color-border);
```

### Shiki Theme Sync

```
Code highlighting themes synced with UI theme:

  UI Light → Shiki "github-light"
  UI Dark → Shiki "github-dark"
  UI Scenic → Shiki "github-dark" (dark bg works better)

Custom Shiki themes:
  Load from theme file's shikiTheme field
  Support custom token colors
```

### Theme Lifecycle

```
1. App startup:
   ├── Load system theme preference
   ├── Load preset themes from ~/.craft-agent/themes/
   ├── Load app overrides from ~/.craft-agent/theme.json
   └── Apply merged theme

2. System theme change:
   ├── macOS: nativeTheme.on('updated')
   ├── Push event: theme:system_changed { isDark }
   └── Re-apply merged theme

3. User changes theme:
   ├── Save to config.json: settings.theme
   ├── Push event: theme:app_changed { theme }
   └── All windows re-render

4. Workspace theme:
   ├── Loaded from workspace config.json
   ├── Applied per-window
   └── Different windows can have different themes
```

---

## 5. Internationalization (i18n)

### Architecture

```
i18n System:
├── registry.ts              # Single source of truth for locales
├── locales/
│   ├── en.json              # English (base)
│   ├── es.json              # Spanish
│   └── zh-Hans.json         # Simplified Chinese
└── i18n.ts                  # i18next configuration

Framework: i18next + react-i18next
```

### Locale Registry

```typescript
// registry.ts — single source of truth
const LOCALE_REGISTRY = {
  en: {
    name: 'English',
    nativeName: 'English',
    messages: () => import('./locales/en.json'),
    dateLocale: () => import('date-fns/locale/en-US'),
  },
  es: {
    name: 'Spanish',
    nativeName: 'Español',
    messages: () => import('./locales/es.json'),
    dateLocale: () => import('date-fns/locale/es'),
  },
  'zh-Hans': {
    name: 'Chinese Simplified',
    nativeName: '简体中文',
    messages: () => import('./locales/zh-Hans.json'),
    dateLocale: () => import('date-fns/locale/zh-CN'),
  },
}

// Derived automatically:
export const SUPPORTED_LANGUAGE_CODES = Object.keys(LOCALE_REGISTRY)
export const LANGUAGES = Object.entries(LOCALE_REGISTRY).map(...)
export function getDateLocale(code: string) { ... }
```

### Key Naming Convention

```
Flat dot-notation with category prefix:

┌──────────────┬──────────────────────────────────────────┐
│ Prefix       │ Scope                                    │
├──────────────┼──────────────────────────────────────────┤
│ common.*     │ Shared labels (Cancel, Save, Close)      │
│ menu.*       │ App menu items                            │
│ sidebar.*    │ Left sidebar navigation                   │
│ session.*    │ Session list UI                           │
│ chat.*       │ Chat input, session viewer                │
│ settings.*   │ Settings pages                            │
│ toast.*      │ Toast/notification messages               │
│ errors.*     │ Error screens                             │
│ browser.*    │ Browser empty state                       │
│ status.*     │ Status names                              │
│ mode.*       │ Permission mode names                     │
│ hints.*      │ Empty state suggestions                   │
│ shortcuts.*  │ Keyboard shortcuts descriptions           │
│ auth.*       │ Auth banner/prompts                       │
│ workspace.*  │ Workspace management                      │
│ onboarding.* │ Onboarding flow                           │
│ dialog.*     │ Modal dialogs                             │
│ table.*      │ Data table headers                        │
│ time.*       │ Relative time strings                     │
└──────────────┴──────────────────────────────────────────┘
```

### Translation Rules

```
1. Never call i18n.t() at module level
   Store labelKey strings, resolve in components

2. Use i18next pluralization:
   "time.minutesAgo_one": "{{count}} minute ago"
   "time.minutesAgo_other": "{{count}} minutes ago"

3. Keep brand names in English:
   Craft, Craft Agents, Agents, Workspace, Claude, Anthropic, OpenAI, MCP, API, SDK

4. Include ... in translation value if UI needs ellipsis:
   "common.loading": "Loading..."
   (Don't append ... in JSX)

5. Use <Trans> component for HTML in translations:
   <Trans>Read our <a href="/docs">documentation</a></Trans>

6. Use i18n.resolvedLanguage (not i18n.language):
   resolvedLanguage falls back correctly

7. All keys must exist in all locale files (en, es, zh-Hans)
   Alphabetically sorted within each file
```

### Adding a New Locale

```
1. Create src/i18n/locales/{code}.json with all keys from en.json

2. Add entry to LOCALE_REGISTRY in registry.ts:
   {
     code: {
       name: 'Full Name',
       nativeName: 'Native Name',
       messages: () => import('./locales/code.json'),
       dateLocale: () => import('date-fns/locale/code'),
     }
   }

3. Run tests — registry tests validate completeness

4. Done — no other files need to change
   SUPPORTED_LANGUAGE_CODES, LANGUAGES, etc. derived automatically
```

### Translation Length Warnings

```
Translations can be 20-100%+ longer than English.

High-risk areas:
┌──────────────────────┬───────────────────────────────────────┐
│ Element              │ Max Translation Length                 │
├──────────────────────┼───────────────────────────────────────┤
│ Permission badges    │ 3-5 characters                        │
│ Settings tab labels  │ ≤10 characters ideal                  │
│ Button labels        │ Avoid 2x English length               │
│ Menu items           │ Flexible, avoid 3x+ growth            │
└──────────────────────┴───────────────────────────────────────┘

Mitigation: Use shorter synonyms in constrained UI elements
```

---

## 6. Workspace Configuration

### Workspace Config Schema

```typescript
interface WorkspaceConfig {
  id: string
  name: string
  slug: string

  defaults: {
    model: string
    defaultLlmConnection: string
    enabledSourceSlugs: string[]
    permissionMode: 'safe' | 'ask' | 'allow-all'
    cyclablePermissionModes: string[]
    workingDirectory: string
    thinkingLevel: string
    colorTheme: string | null
  }

  localMcpServers: {
    enabled: boolean
  }

  createdAt: string
  updatedAt: string
}
```

### Workspace CRUD

```
createWorkspace(name):
    │
    ├── Generate slug: "My Workspace" → "my-workspace"
    ├── Generate unique path: ~/.craft-agent/workspaces/my-workspace
    ├── Create directory structure:
    │   sources/, sessions/, skills/, statuses/, labels/
    ├── Initialize default configs:
    │   statuses/default-statuses.json
    │   labels/default-labels.json
    │   .claude-plugin/manifest.json
    │
    └── Register in global config

deleteWorkspace(slug):
    │
    ├── Check for active sessions
    ├── Close MCP connections
    ├── Delete credentials for workspace
    ├── Remove directory (recursive)
    └── Update global config

switchWorkspace(slug):
    │
    ├── Save current workspace state
    ├── Load new workspace config
    ├── Update active workspace in config
    ├── Trigger workspace change events
    └── Renderer reloads sessions
```

---

## 7. Skills System

### Skill Configuration

```typescript
interface SkillConfig {
  id: string
  name: string
  slug: string
  description: string
  prompt: string               // Template with {{variables}}
  variables?: SkillVariable[]
  enabledSourceSlugs?: string[]
  permissionMode?: string
  model?: string
  iconPath?: string
  createdAt: string
  updatedAt: string
}

interface SkillVariable {
  name: string
  type: 'text' | 'select' | 'file'
  required: boolean
  defaultValue?: string
  options?: string[]           // For select type
}
```

### Skill Storage

```
{workspaceRootPath}/skills/{skill-slug}/
├── config.json               # Skill configuration
├── files/                    # Skill template files
│   └── template.md
└── icon.*                    # Skill icon
```

### Skill Execution

```
User triggers skill:
    │
    ├── Load skill config
    ├── Resolve variables:
    │   Prompt user for missing required variables
    │   Apply defaults for optional variables
    │
    ├── Expand template:
    │   "Summarize {{document}} focusing on {{topic}}"
    │   → "Summarize report.pdf focusing on revenue"
    │
    ├── Create or reuse session:
    │   Apply skill's model, permission mode, sources
    │
    └── Send expanded prompt to agent
```

---

## 8. Status Workflow Configuration

### Status System

```
{workspaceRootPath}/statuses/statuses.json:

{
  "statuses": [
    { "id": "inbox", "name": "Inbox", "color": "#6b7280" },
    { "id": "active", "name": "Active", "color": "#3b82f6" },
    { "id": "completed", "name": "Completed", "color": "#22c55e" },
    { "id": "blocked", "name": "Blocked", "color": "#ef4444" }
  ],
  "order": ["inbox", "active", "completed", "blocked"]
}

Custom workflows:
  Users define their own statuses
  Drag-and-drop reordering
  Colors customizable
  Automation triggers on status transitions
```

### Label System

```
{workspaceRootPath}/labels/labels.json:

{
  "labels": [
    { "id": "urgent", "name": "Urgent", "color": "#ef4444" },
    { "id": "frontend", "name": "Frontend", "color": "#3b82f6" },
    { "id": "bug", "name": "Bug", "color": "#f97316" },
    { "id": "feature", "name": "Feature", "color": "#22c55e" }
  ]
}

Multi-label:
  Sessions can have multiple labels
  Filter sidebar by label
  Automations trigger on label changes
```
