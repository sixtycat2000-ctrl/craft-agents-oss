# Electron Application — Desktop Platform

> How the Electron app is structured across main, renderer, and preload processes, with window management, auto-update, deep linking, and native integrations.

---

## 1. Process Architecture

```
┌────────────────────────────────────────────────────────────────┐
│                         Electron App                          │
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │  Main Process (Node.js)                                  │ │
│  │                                                          │ │
│  │  index.ts ─── App entry point                            │ │
│  │    ├── Shell env loading                                 │ │
│  │    ├── Sentry init                                       │ │
│  │    ├── Platform services                                 │ │
│  │    ├── Server bootstrap (embedded)                       │ │
│  │    ├── WindowManager creation                            │ │
│  │    ├── BrowserPaneManager creation                       │ │
│  │    ├── Menu system                                       │ │
│  │    └── Deep link handler                                 │ │
│  │                                                          │ │
│  │  window-manager.ts ─── Window lifecycle                  │ │
│  │  browser-pane-manager.ts ─── Browser automation          │ │
│  │  auto-update.ts ─── Update checker + installer           │ │
│  │  deep-link.ts ─── URL protocol handler                   │ │
│  │  menu.ts ─── Native menu bar                             │ │
│  │  notifications.ts ─── System notifications + badge       │ │
│  │  handlers/*.ts ─── RPC handler registration              │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                                │
│  ┌────────────────────┐  ┌──────────────────────────────────┐│
│  │ Preload (Bridge)   │  │ Renderer (Chromium)              ││
│  │                    │  │                                  ││
│  │ bootstrap.ts       │  │ App.tsx ─── React root           ││
│  │ ├─ RoutedClient    │  │ ├─ Theme provider                ││
│  │ ├─ electronAPI     │  │ ├─ Navigation context            ││
│  │ └─ Capabilities    │  │ ├─ AppShell                      ││
│  │                    │  │ │  ├─ TopBar                      ││
│  └────────────────────┘  │ │  ├─ LeftSidebar                 ││
│                           │ │  ├─ PanelStack                  ││
│   context bridge          │ │  └─ ChatDisplay                 ││
│   ◄──────────────────────▶│ └─ Hooks, atoms, contexts        ││
│                           └──────────────────────────────────┘│
└────────────────────────────────────────────────────────────────┘
```

---

## 2. Main Process Startup

### Initialization Sequence

```
app.whenReady()
    │
    ├── 1. loadShellEnv()
    │   Source user's shell profile (.zshrc, .bashrc)
    │   Ensures CLI tools (uv, bun) are in PATH
    │
    ├── 2. Sentry initialization
    │   Error tracking with sensitive data scrubbing
    │   Platform-specific DSN
    │
    ├── 3. Create ElectronPlatform
    │   Wraps native APIs for server-core
    │   Provides logger, error capture, image processing
    │
    ├── 4. Configure bundled tools
    │   Set env vars for bundled CLI tools:
    │   - UV_PATH (Python package manager)
    │   - BUN_PATH (JS runtime)
    │   - SCRIPTS_PATH (built-in scripts)
    │
    ├── 5. Bootstrap embedded server
    │   Create WsRpcServer on localhost
    │   Initialize SessionManager
    │   Register RPC handlers
    │   Set up ConfigWatcher
    │
    ├── 6. Create WindowManager
    │   Set RPC event sink for push events
    │   Create initial windows (restore state)
    │
    ├── 7. Create BrowserPaneManager
    │   Bind to SessionManager
    │   Register browser RPC handlers
    │
    ├── 8. Set up menu system
    │   Build native menu bar (macOS)
    │   Wire menu actions to RPC
    │
    ├── 9. Register deep link handler
    │   craftagents:// protocol
    │
    ├── 10. Auto-update check
    │   Check for updates on launch
    │   Respect dismissal history
    │
    └── 11. Ready
        Windows visible, server running, tools available
```

---

## 3. WindowManager

### Window Creation

```
createWindow(options: { workspaceId?, url?, focused? })
    │
    ├── Determine window features:
    │
    │   Platform-specific:
    │   ├── macOS: titleBarStyle: 'hiddenInset'
    │   │          trafficLightPosition for custom layout
    │   ├── Windows: backgroundMaterial: 'mica' (Win11)
    │   │              or 'acrylic' (Win10)
    │   └── Linux: Standard window frame
    │
    │   Window mode:
    │   ├── Full mode: 1200x800 default
    │   └── Focused mode: 900x700 (single session view)
    │
    ├── Create BrowserWindow:
    │   const win = new BrowserWindow({
    │     width, height,
    │     webPreferences: {
    │       preload: path.join(__dirname, 'preload.js'),
    │       contextIsolation: true,
    │       sandbox: false,
    │     }
    │   })
    │
    ├── Bind to workspace:
    │   Track: webContentsId → workspaceId
    │
    ├── Load renderer:
    │   Development: http://localhost:5173 (Vite dev server)
    │   Production: file://path/to/renderer/index.html
    │
    ├── Restore state:
    │   Previous bounds, maximized state, fullscreen
    │
    └── Track in windows map:
      Map<webContentsId, { window, workspaceId }>
```

### Window Lifecycle

```
Window Events:
┌───────────────────────────────────────────────────┐
│ 'close' event                                     │
│   ├── Intercepted for modal dismissal             │
│   │   (Cmd+W first dismisses modal, then closes)  │
│   ├── If last window: check for active sessions   │
│   └── Save window state (bounds, URL)              │
│                                                   │
│ 'closed' event                                    │
│   ├── Clean up window reference                   │
│   ├── Close browser panes for this window         │
│   └── Disconnect RPC for this webContents         │
│                                                   │
│ 'focus' event                                     │
│   └── Push focus state to renderer                │
│                                                   │
│ 'blur' event                                      │
│   └── Push blur state to renderer                 │
└───────────────────────────────────────────────────┘
```

### Workspace Switching (In-Window)

```
User switches workspace in dropdown
    │
    ▼
1. Renderer calls electronAPI.switchWorkspace(newId)
    │
    ▼
2. Main process: windowManager.updateWindowWorkspace(wcId, newId)
    │
    ├── Update workspace binding
    ├── RPC transport switches workspace client
    │
    ▼
3. Renderer receives new workspace data:
    ├── Clear session atoms
    ├── Load new workspace sessions
    ├── Update navigation
    └── Restore panel state
```

---

## 4. Preload Bridge

### Architecture

```
bootstrap.ts (preload script)
    │
    ├── Determine mode:
    │   ├── Normal: RoutedClient (local + workspace)
    │   └── Thin-client: WsRpcClient (direct to remote)
    │
    ├── Create RPC client:
    │   const localClient = new WsRpcClient({ url: 'ws://localhost:PORT' })
    │   const routedClient = new RoutedClient({ localClient, workspaceClient })
    │
    ├── Build electronAPI from channel map:
    │   const electronAPI = buildClientApi(CHANNEL_MAP, routedClient)
    │
    └── Expose via context bridge:
        contextBridge.exposeInMainWorld('electronAPI', {
          ...electronAPI,
          // Additional capabilities:
          shell: { openExternal, openPath },
          dialog: { showMessageBox, showOpenDialog },
        })
```

### Channel Map

```
CHANNEL_MAP (single source of truth):

// Session operations
getSessions:      { type: 'invoke', channel: 'sessions:GET' }
sendMessage:      { type: 'invoke', channel: 'sessions:SEND_MESSAGE' }
createSession:    { type: 'invoke', channel: 'sessions:CREATE', transform: r => r.sessionId }
deleteSession:    { type: 'invoke', channel: 'sessions:DELETE' }

// Event listeners
onSessionEvent:   { type: 'listener', channel: 'session:event' }
onReconnected:    { type: 'listener', channel: '__transport:reconnected' }

// Workspace
getWorkspaces:    { type: 'invoke', channel: 'server:GET_WORKSPACES' }
switchWorkspace:  { type: 'invoke', channel: 'window:SWITCH_WORKSPACE' }

// Browser
browserCreate:    { type: 'invoke', channel: 'browserPane:CREATE' }
browserDestroy:   { type: 'invoke', channel: 'browserPane:DESTROY' }

// ... 100+ total mappings
```

---

## 5. Auto-Update System

### Update States

```
┌──────────┐     ┌─────────────┐     ┌──────────┐     ┌───────────┐
│   idle   │────▶│ downloading │────▶│   ready  │────▶│ installing│
└──────────┘     └─────────────┘     └──────────┘     └───────────┘

State transitions:
  idle → downloading:  checkForUpdates() finds new version
  downloading → ready: download complete
  ready → installing:  installUpdate() → quitAndInstall
  installing → idle:   app restarts with new version
```

### Update Flow

```
checkForUpdatesOnLaunch()
    │
    ├── Check dismissal history:
    │   User dismissed this version before?
    │   → Skip if recently dismissed
    │
    ├── Query update server:
    │   electron-updater checks for new version
    │   Platform-specific (macOS: zip, Windows: NSIS, Linux: AppImage)
    │
    ├── Update available:
    │   ├── Change menu item: "Install Update v1.2.3"
    │   ├── Push event to renderer:
    │   │   update:available { version, releaseNotes }
    │   └── Download in background:
    │       Push progress: update:download_progress { percent }
    │
    └── Download complete:
        ├── Change menu item: "Install Update v1.2.3"
        └── User clicks → quitAndInstall()
```

### Platform Behavior

```
┌──────────┬─────────────────────────────────────────┐
│ Platform │ Update mechanism                        │
├──────────┼─────────────────────────────────────────┤
│ macOS    │ Zip download → replace app bundle       │
│          │ Code signing required                    │
│          │ Sparkle framework compatible             │
├──────────┼─────────────────────────────────────────┤
│ Windows  │ NSIS installer download → execute        │
│          │ Digital signature verified               │
│          │ Optional differential updates            │
├──────────┼─────────────────────────────────────────┤
│ Linux    │ AppImage download → replace              │
│          │ No auto-install (manual replacement)     │
└──────────┴─────────────────────────────────────────┘
```

---

## 6. Deep Link System

### URL Scheme

```
Protocol: craftagents://

Supported URLs:
  craftagents://allSessions
  craftagents://allSessions/session/{sessionId}
  craftagents://flagged
  craftagents://flagged/session/{sessionId}
  craftagents://state/{stateId}
  craftagents://state/{stateId}/session/{sessionId}
  craftagents://sources
  craftagents://sources/source/{sourceSlug}
  craftagents://settings
  craftagents://settings/{subpage}
  craftagents://action/new-chat
  craftagents://action/new-chat?input=hello&send=true
  craftagents://workspace/{workspaceId}/allSessions/session/{sessionId}
```

### Deep Link Flow

```
External source opens: craftagents://action/new-chat?input=Review%20this&send=true
    │
    ▼
1. macOS: app.on('open-url') / Windows: protocol handler
    │
    ▼
2. parseDeepLink(url):
    {
      action: 'new-chat',
      params: { input: 'Review this', send: true }
    }
    │
    ▼
3. handleDeepLink():
    │
    ├── Find or create window for workspace:
    │   windowManager.getWindowForWorkspace(workspaceId)
    │   OR createWindow({ workspaceId })
    │
    ├── Navigate renderer:
    │   Push deeplink navigation event
    │   Renderer creates new session with input
    │
    └── Query parameters:
        ?input=text     → pre-fill chat input
        &send=true      → auto-send message
        ?window=focused → open in focused mode
        &sidebar=files/path → open right sidebar
```

---

## 7. Menu System

### Menu Structure

```
macOS Native Menu Bar:

┌───────────────────────────────────────────────────────┐
│ Craft Agents │ File │ Edit │ View │ Window │ Help      │
├──────────────┼──────┼──────┼──────┼────────┼──────────┤
│ About        │ New  │ Undo │ Zoom │ Minimize│ Docs     │
│ Check Upd... │ Chat │ Redo │ Side │ Zoom    │ Report   │
│ ──────────── │ Wkspace│ Cut │ Panel│ Close   │          │
│ Settings     │ Close│ Copy │ Theme│         │          │
│ ──────────── │      │ Paste│ Full │         │          │
│ Hide         │      │ Select│ Scrn│         │          │
│ Hide Others  │      │ Find │      │         │          │
│ Quit         │      │      │      │         │          │
└──────────────┴──────┴──────┴──────┴────────┴──────────┘

Windows/Linux:
  Menu hidden → Craft logo dropdown in app
  Same actions available via dropdown
```

### Dynamic Menu Items

```
Menu items that change based on state:

Update:
  No update:  "Check for Updates..."
  Downloading: "Downloading... 45%"
  Ready:      "Install Update v1.2.3"

Sessions:
  "New Chat" → Cmd+N → creates new session
  "Close Window" → Cmd+W → close current window

Actions:
  Keyboard shortcuts handled by action registry
  Accelerators disabled in menu (handled by renderer)
```

---

## 8. Notification System

### System Notifications

```
showNotification(title, body, workspaceId, sessionId)
    │
    ├── Create Notification:
    │   new Notification({ title, body, icon })
    │
    ├── Click handler:
    │   Focus window for workspace
    │   Navigate to session
    │   Push deeplink navigation event
    │
    └── Show notification

Triggers:
  - Agent completes a task (session status → 'completed')
  - Permission request awaiting user action
  - Automation fires with notification action
```

### Badge System

```
updateBadgeCount(count)
    │
    ├── macOS:
    │   Canvas-rendered badge image
    │   app.dock.setIcon(badgeImage)
    │   Shows count on dock icon
    │
    ├── Windows:
    │   Canvas-rendered badge image
    │   mainWindow.setOverlayIcon(badgeImage)
    │   Shows count on taskbar
    │
    └── Linux:
        app.setBadgeCount(count)
        Unity/KDE support
```

---

## 9. Power Management

```
powerBlocker = powerSaveBlocker.start('prevent-display-sleep')

When to block:
  - Agent is processing (active session)
  - Browser automation in progress
  - File download in progress

When to unblock:
  - All sessions idle
  - No active downloads
  - User manually idle > 5 minutes
```

---

## 10. File System Integration

### Thumbnail Protocol

```
Custom protocol: craft-thumbnail://

Renderer requests: craft-thumbnail:///path/to/image.png
    │
    ▼
Main process intercepts:
    protocol.handle('craft-thumbnail', (request) => {
      const path = decodeURIComponent(request.url.replace('craft-thumbnail://', ''))
      // Validate path is within workspace
      // Read and resize image
      // Return as data URL
    })

Benefits:
  - No absolute paths in renderer DOM
  - Path validation on main process
  - Automatic image resizing
```

### File Dialog Operations

```
Client capability: client:openFileDialog

Server requests file dialog:
    invokeClient(clientId, 'client:openFileDialog', [options])
    │
    ▼
Preload handler:
    dialog.showOpenDialog({
      properties: ['openFile', 'multiSelections'],
      filters: [{ name: 'Documents', extensions: ['md', 'txt', 'pdf'] }]
    })
    │
    ▼
Returns: { filePaths: ['/selected/file.md'] }
```

---

## 11. Platform-Specific Features

### macOS

```
┌──────────────────────────────────────────────┐
│ Feature              │ Implementation        │
├──────────────────────┼───────────────────────┤
│ Title bar            │ hiddenInset style     │
│ Traffic lights       │ Custom position       │
│ Dock icon            │ Canvas-rendered badge │
│ Menu bar             │ Full native menu      │
│ App Nap prevention   │ powerSaveBlocker      │
│ Touch Bar            │ Not implemented       │
└──────────────────────┴───────────────────────┘
```

### Windows

```
┌──────────────────────────────────────────────┐
│ Feature              │ Implementation        │
├──────────────────────┼───────────────────────┤
│ Background           │ Mica (Win11)          │
│                      │ Acrylic (Win10)       │
│ Taskbar              │ Overlay icon badge    │
│ Menu                 │ Hidden, logo dropdown │
│ VC++ Redistributable │ Check on startup      │
│ Auto-update          │ NSIS installer        │
└──────────────────────┴───────────────────────┘
```

### Linux

```
┌──────────────────────────────────────────────┐
│ Feature              │ Implementation        │
├──────────────────────┼───────────────────────┤
│ Window frame         │ Standard decorations  │
│ Badge                │ app.setBadgeCount()   │
│ Menu                 │ Hidden, logo dropdown │
│ Auto-update          │ AppImage replacement  │
│ Desktop entry        │ Generated on install  │
└──────────────────────┴───────────────────────┘
```

---

## 12. Error Handling

### Main Process Errors

```
Unhandled exceptions:
    ├── Log to electron-log
    ├── Report to Sentry (production)
    ├── Show error dialog to user
    └── Continue (don't crash)

Uncaught promise rejections:
    ├── Same as above
    └── Prevent default exit behavior

Renderer crashes:
    ├── Detect via 'render-process-gone' event
    ├── Log crash details
    ├── Reload window automatically
    └── Show recovery message to user
```

### Graceful Shutdown

```
app.on('before-quit'):
    │
    ├── Flush all session saves
    ├── Close MCP connections
    ├── Stop file watchers
    ├── Release power save blocker
    ├── Save window state
    └── Stop embedded server (2s drain)

app.on('window-all-closed'):
    ├── macOS: don't quit (stay in dock)
    └── Others: quit
```
