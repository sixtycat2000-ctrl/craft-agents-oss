# Craft Agents — Browser Tool Lessons & Learnings for Your App

> Extracted from release notes (v0.2.30 → v0.8.9) and commit history.
> Organized by theme, with actionable takeaways for building a similar Electron chat AI agent with embedded browser.

---

## Timeline: Browser Tool Evolution

The browser feature landed in **v0.6.0** and has been refined across 9+ releases:

| Version | What Changed | Lesson |
|---------|-------------|--------|
| **v0.6.0** | Initial integrated browser with in-app panes, multi-panel view (browser + chat side-by-side), session branching | Start with multi-panel layout from day one — users need to see browser AND chat simultaneously |
| **v0.7.0** | WebSocket RPC replaced Electron IPC, headless server support | Decouple browser tool from Electron IPC immediately — you'll want remote/headless later |
| **v0.7.5** | Webhook actions for automations | Browser actions can trigger external notifications — useful for "agent finished browsing" alerts |
| **v0.7.7** | Server-side directory browsing, 5-level thinking system, parallel call_llm | Agent needs thinking controls when doing complex browser tasks; parallel sub-models speed up multi-step browsing |
| **v0.8.0** | Hybrid local/remote transport, multiple remote workspaces, browser-accessible WebUI | Users want to access browser sessions from multiple devices |
| **v0.8.2** | **Built-in browser toggle** — settings option to disable built-in browser_tool | Some users already have Playwright/Puppeteer MCP servers — make your browser tool optional |
| **v0.8.3** | Session self-management tools — agents can set labels, status, query other sessions | Agents should be able to report what they did in the browser (e.g., auto-label "browsed") |
| **v0.8.6** | Chunked session transfers, image input for custom endpoints | Browser screenshots can be large — plan for chunked data transfer from the start |
| **v0.8.8** | Inter-session messaging (send_agent_message) | One agent controlling the browser can notify other sessions of results |

---

## Key Learnings by Theme

### 1. Architecture: Decouple Early

**What Craft did:** Browser tool interface (`BrowserPaneFns`) is in the shared package with zero Electron imports. The Electron-specific `BrowserPaneManager` is behind a callback registry.

**Why it matters:** When v0.7.0 added headless server support, the browser tool couldn't run on headless (no `BrowserWindow`). Because the interface was decoupled, they could gracefully fall back to `"Browser window controls are not available"` without touching the tool logic.

**Your takeaway:**
- Define your browser abstraction as an interface, not an implementation
- Make the tool gracefully degrade when no browser backend is available
- Don't couple browser logic to Electron main process — use a callback registry or dependency injection

```
DO:  tool handler → getBrowserPaneFns() → browser backend
DON'T: tool handler → electron.BrowserWindow directly
```

### 2. UX: Live Visual Feedback is Critical

**What Craft did:** Three stacked `BrowserView`s per instance:
- **Toolbar** (48px) — URL bar, navigation, loading spinner, theme color extraction
- **Page content** — the actual web page the agent is browsing
- **Native overlay** — cursor animation, "Craft Agents are working..." badge, agent control indicator

The overlay provides visual cues that the AI is actively controlling the browser — animated cursor, pulsing border, status label. This was refined across multiple releases.

**Why it matters:** Without visual feedback, users don't know if the agent is working or stuck. The overlay turns an opaque background process into something the user can watch and understand.

**Your takeaway:**
- Show the browser window while the agent is working (don't hide it)
- Animate cursor movements so users can see what the agent is doing
- Show a status badge ("Agent is browsing...", "Waiting for page load...")
- Auto-release control when done (the `release` command pattern)
- Extract and display the page's theme color in your toolbar — it makes the browser feel native

### 3. Security: CAPTCHA Detection & Handoff

**What Craft did:** The runtime auto-detects security challenges (Cloudflare, reCAPTCHA) after `navigate`, `click`, and `snapshot`. When detected:
1. Agent releases control to the user
2. Browser window is shown (not hidden)
3. User sees a message: "Please complete the verification check"
4. After verification, agent resumes with `snapshot`

```
// From browser-tool-runtime.ts
if (challenge.detected) {
  await fns.releaseControl();
  return {
    output: 'Security verification detected. Browser shown — please complete the check.',
    appendReleaseHint: false,
  };
}
```

**Why it matters:** CAPTCHAs are the #1 failure mode for browser automation. Auto-detecting them and handing off to the user (instead of failing silently) turns a hard error into a recoverable pause.

**Your takeaway:**
- Implement challenge detection on every navigation and click
- Build a release/resume mechanism — agent pauses, user completes challenge, agent continues
- Don't try to solve CAPTCHAs automatically — it's fragile and ethically questionable

### 4. Session-to-Browser Binding

**What Craft did:** `BrowserPaneManager.createForSession(sessionId)` implements lazy 1:1 binding:
- First `browser_tool` call → creates BrowserWindow, binds to session
- Subsequent calls → reuses the same window
- Session ends → window persists but unbinds (can be reused by another session)
- Multiple sessions can have their own windows simultaneously

This was critical for multi-session inbox — each chat session gets its own browser context.

**Why it matters:** Shared browser state between sessions is confusing and dangerous (cookies, auth tokens, navigation history leak between sessions).

**Your takeaway:**
- One browser instance per chat session (lazy, on first use)
- Use separate cookie partitions (Electron `session.fromPartition`)
- Allow users to see and switch between session-bound browser windows
- Persist windows across sessions (don't destroy on session end)

### 5. Token Efficiency: Accessibility Tree Over HTML

**What Craft did:** The `snapshot` command returns an accessibility tree with `@eN` refs, not raw HTML. Typical output:

```
URL: https://example.com
Title: Example
Elements: 45 (button:8, link:12, textbox:3)

  @e1 [link] "Home"
  @e2 [textbox] "Search" value=""
  @e3 [button] "Submit"
```

This is dramatically more token-efficient than dumping HTML. A typical page snapshot uses ~500-2000 tokens vs 10,000+ for full HTML.

**Why it matters:** Browser interactions are already token-heavy (each action is a tool call + response). Using accessibility trees keeps costs manageable.

**Your takeaway:**
- Use accessibility tree snapshots, not HTML scraping
- Use `@eN` element refs for interaction (click `@e3`, fill `@e5`) — deterministic, no CSS selector fragility
- Consider offering a `find <query>` command for keyword-based element search (token-efficient way to find elements without reading the full snapshot)

### 6. The Single-Tool CLI Pattern

**What Craft did:** Instead of exposing 30+ separate tools (`browser_navigate`, `browser_click`, `browser_fill`...), everything is one `browser_tool` with a CLI-like command string:

```
browser_tool({ command: "navigate https://example.com" })
browser_tool({ command: "fill @e5 user@example.com; click @e3" })  // batch!
```

**Why it matters:** LLMs handle fewer tools better. 30 browser tools pollute the tool namespace and increase discovery cost. The CLI pattern keeps the schema trivial while supporting 40+ commands.

**Your takeaway:**
- Use one tool with a command parameter, not many tools
- Support batch commands with semicolons (fill multiple fields then click submit in one call)
- Stop batch execution after navigation commands (page state changes unpredictably)
- Support array mode for commands containing semicolons: `["evaluate", "var x = 1; x + 2"]`

### 7. Multi-Provider Support Was a Game Changer

**Release history shows:** Adding Google AI Studio, ChatGPT Plus, GitHub Copilot, OpenRouter, Ollama, Amazon Bedrock across v0.4.0–v0.7.8 transformed the product from a Claude-only tool to a universal agent platform.

**Why it matters for your browser tool:** Different models have different capabilities with browser automation:
- Claude models handle complex multi-step browsing well
- GPT-4o may be better at visual reasoning from screenshots
- Local models (Ollama) are free but slower
- Users should be able to choose which model drives the browser

**Your takeaway:**
- Don't couple browser tool to a specific LLM provider
- Let users configure which model handles browser sessions
- Consider using a cheaper/faster model for simple browsing and a powerful model for complex tasks

### 8. Feature Gate: Make It Optional

**What Craft did (v0.8.2):** Added a settings toggle to disable the built-in browser_tool. Users with external browser tools (Playwright MCP, Puppeteer) can turn off the built-in one to avoid conflicts.

**Your takeaway:**
- Not every user wants embedded browser control
- Make it opt-in or easily disableable
- Don't conflict with MCP-based browser tools the user might already have
- Cache invalidation: when the toggle changes, rebuild the tool list for all active sessions

### 9. Automations + Browser = Powerful Workflows

**What Craft did (v0.4.3 → v0.5.1):** The hooks/automations system evolved from simple event triggers to a full automation engine with conditions, cron scheduling, and webhook actions.

**Combined with browser:** Users can set up automations like:
- "Every weekday at 9am, open Linear and summarize my assigned issues"
- "When a session gets the 'urgent' label, take a screenshot of the relevant GitHub PR"
- "After every browser session, send the summary to Slack"

**Your takeaway:**
- Browser actions should emit events (session started, page navigated, element clicked)
- Allow automations to trigger browser-based workflows
- Support cron scheduling for recurring browsing tasks
- Webhook actions enable "browse → notify" patterns

### 10. Internationalization Impacts Browser Tool

**What Craft did (v0.8.5–v0.8.7):** Added i18n for English, Spanish, Chinese, Japanese, Hungarian, German, Polish.

**Impact on browser tool:** The browser tool description and help text are in English only (LLM-facing, not user-facing). But UI elements around the browser (toolbar, empty state, status badges) need translation.

**Your takeaway:**
- Keep the tool description in English (LLMs work best in English)
- Translate browser toolbar UI, empty state, and status messages
- Consider that users may browse non-English websites — the accessibility tree `name` field will contain non-ASCII text

---

## Lessons from Bugs & Fixes

These issues from release notes are worth knowing about:

### v0.7.7: Parallel call_llm Execution
- **Issue:** Sequential LLM calls for browser subtasks were slow
- **Fix:** Parallel execution support
- **Lesson:** Browser workflows often need multiple independent evaluations (e.g., "check all 5 links on this page"). Design for parallelism from the start.

### v0.8.1: Remote Workspace Recovery
- **Issue:** If the remote server disconnected, browser windows were lost
- **Fix:** Reconnection flow that recovers workspace state
- **Lesson:** Browser instances should survive transient disconnections. Keep a registry of active instances, not just in-memory references.

### v0.8.5–v0.8.6: Message Loss After Sleep/Wake
- **Issue:** Session messages (including browser tool results) were lost when the machine slept/woke
- **Fix:** Persistence queue improvements
- **Lesson:** Browser tool results (especially screenshots) are expensive to reproduce. Persist them immediately, don't buffer.

### v0.8.3–v0.8.7: Stale Server Lock
- **Issue:** `.server.lock` files left behind after crashes prevented restart
- **Fix:** Stale lock detection and cleanup
- **Lesson:** Browser CDP connections can leave stale state. Always clean up on process exit, and detect/handle stale state on startup.

---

## Feature Prioritization for Your App

Based on the Craft Agents evolution, here's the recommended implementation order for browser tool features:

### Phase 1: Core (Must-Have)
1. **Single `browser_tool` with CLI commands** — navigate, snapshot, click, fill, type, screenshot
2. **Accessibility tree snapshots** with `@eN` refs (not HTML)
3. **Session-bound browser instances** — one BrowserWindow per chat session
4. **Live browser view** — user sees the page in real-time while agent works
5. **Agent control overlay** — cursor animation + status badge
6. **CAPTCHA detection** with user handoff

### Phase 2: Polish (Important)
7. **Batch commands** with semicolons
8. **Find command** — keyword search over elements
9. **Console/network logs** — debugging support
10. **Clipboard integration** — set-clipboard, get-clipboard, paste (crucial for spreadsheet workflows)
11. **Window management** — focus, release, close, list windows
12. **Feature toggle** — let users disable built-in browser

### Phase 3: Advanced (Nice-to-Have)
13. **Sub-agent browser loop** — use a fast model for autonomous multi-step browsing
14. **Automation triggers** — "browse this URL every morning"
15. **Cross-session browser coordination** — one agent can hand off browser state to another
16. **Canvas interaction** — click-at, drag for canvas-based UIs (Google Sheets, maps)
17. **File upload** — upload local files to web forms
18. **Download tracking** — monitor and retrieve downloaded files

### Phase 4: Enterprise (Later)
19. **Remote browser** — browser runs on server, user views via WebUI
20. **Cookie/partition isolation** — separate browser profiles per workspace
21. **Network proxy support** — configure HTTP/HTTPS proxies for the browser
22. **Theme color extraction** — match browser toolbar to website theme

---

## Key Architectural Decisions to Make Now

| Decision | Craft's Choice | Recommendation for Your App |
|----------|---------------|---------------------------|
| Browser backend | Electron `BrowserView` + CDP | Start with Playwright for faster implementation, migrate to native CDP for polish |
| Tool interface | Single tool, CLI commands | Same — proven pattern |
| Element addressing | Accessibility tree `@eN` refs | Same — token-efficient and deterministic |
| Session binding | 1:1, lazy, persistent | Same — prevents state leakage |
| Visual feedback | 3-layer BrowserView stack | Start simpler (screencast or CDP), add overlay later |
| Graceful degradation | Fallback message when no browser | Same — essential for remote/headless |
| Feature gating | Config toggle, disabled by default | Make opt-in initially, enable by default once stable |
| State persistence | JSONL with atomic writes | Persist browser tool results immediately — screenshots are expensive to regenerate |
