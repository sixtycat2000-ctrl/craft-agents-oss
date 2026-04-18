# Craft Agents — Non-Security Features Reference

> This document catalogues every non-security feature in Craft Agents, describing the problem it addresses, its design, why it matters for product value, and potential future improvements.

---

## Table of Contents

1. [Agent System (Dual Backend)](#1-agent-system-dual-backend)
2. [Sources Integration](#2-sources-integration)
3. [Skills System](#3-skills-system)
4. [Session Management](#4-session-management)
5. [Automations](#5-automations)
6. [Permission Modes](#6-permission-modes)
7. [Labels System](#7-labels-system)
8. [Status System](#8-status-system)
9. [Theme System](#9-theme-system)
10. [Browser Tools](#10-browser-tools)
11. [Data Tables & transform_data](#11-data-tables--transform_data)
12. [Mermaid Diagrams](#12-mermaid-diagrams)
13. [LLM Tool (Secondary Model)](#13-llm-tool-secondary-model)
14. [Multi-File Diff Viewer](#14-multi-file-diff-viewer)
15. [File Attachments & Preview](#15-file-attachments--preview)
16. [Internationalization (i18n)](#16-internationalization-i18n)
17. [Headless Server & Remote Access](#17-headless-server--remote-access)
18. [CLI Client](#18-cli-client)
19. [Web UI](#19-web-ui)
20. [MCP Integration & Client Pool](#20-mcp-integration--client-pool)
21. [Workspace Management](#21-workspace-management)
22. [Search](#22-search)
23. [Scheduler](#23-scheduler)
24. [Deep Linking](#24-deep-linking)
25. [Tool Icons](#25-tool-icons)
26. [Large Response Handling](#26-large-response-handling)
27. [Background Tasks](#27-background-tasks)
28. [Configuration System](#28-configuration-system)

---

## 1. Agent System (Dual Backend)

### Problem It Addresses
Users want access to the best AI models from multiple providers — Anthropic, Google, OpenAI, GitHub Copilot — but each has different APIs, auth flows, and capabilities. A single-backend approach locks users into one provider.

### Design
- **Claude Agent Backend** — uses the `@anthropic-ai/claude-agent-sdk` for Anthropic API key, Claude Max/Pro OAuth, and all OpenAI-compatible endpoints (OpenRouter, Ollama, Vercel AI Gateway, custom).
- **Pi Agent Backend** — uses the Pi SDK for Google AI Studio, ChatGPT Plus (Codex OAuth), GitHub Copilot OAuth, and OpenAI API keys.
- **Backend Factory** (`backend/factory.ts`) selects the appropriate backend based on connection config.
- Both backends share a common `BaseAgent` abstraction (`base-agent.ts`) for streaming, tool visualization, and context management.
- Per-workspace default model selection with model fallback chains.

### Why It Matters (Product Value)
- Users can bring their own subscriptions (Copilot, ChatGPT Plus, Claude Max) — no need to buy new API keys.
- Multi-provider support is a key differentiator vs single-provider tools like Claude Code CLI.
- Model fallback chains ensure sessions don't stall when a provider is down.
- Custom endpoints (Ollama, OpenRouter) let power users access hundreds of models.

### Future Improvements
- **Model routing rules** — automatically select model based on task complexity (use Haiku for simple queries, Opus for complex).
- **A/B testing between providers** — run the same prompt on two models and compare.
- **Cost tracking per provider** — show token usage and estimated costs across connections.
- **Provider health monitoring** — dashboard showing latency, error rates, and availability per provider.

---

## 2. Sources Integration

### Problem It Addresses
Agents are only useful if they can act on real data. Users need to connect to external services (Gmail, Slack, Linear, GitHub, databases) and local files (Obsidian vaults, Git repos) without writing config files or understanding API specs.

### Design
- **Three source types**: MCP servers (HTTP/SSE or stdio), REST APIs (direct HTTP), and local filesystems.
- **Agent-driven setup**: The agent reads public API docs, configures auth, writes `guide.md`, and validates the connection. Users just say "add Linear as a source."
- **Credential Manager** (`credential-manager.ts`) — unified credential operations with strategy pattern for different OAuth providers (Google, Slack, Microsoft, GitHub, generic RFC 9728).
- **Token Refresh Manager** (`token-refresh-manager.ts`) — proactive token refresh with rate limiting and deduplication to prevent race conditions.
- **API Tools Factory** (`api-tools.ts`) — dynamically creates tools from source config with multi-header auth, binary file detection, and OOM-safe response handling.
- **Server Builder** (`server-builder.ts`) — builds MCP/API server configs with layered header merging (static -> credential-store -> bearer token).
- **Auto-icon fetching** — downloads and caches service icons from URLs.
- **Source-scoped `guide.md`** — agent-written documentation injected into context for future sessions.

### Why It Matters (Product Value)
- Zero-config source setup is the flagship "hard to believe it just works" feature — differentiated by agent-driven configuration vs manual setup wizards.
- Supports any API, not just MCP — broader integration surface than competitors.
- OAuth providers (Google, Slack, Microsoft) cover the majority of workplace tools.
- Local source support (Obsidian, Git repos) appeals to power users and developers.
- Auto-refreshing tokens eliminates auth-related session interruptions.

### Future Improvements
- **Source templates gallery** — pre-built configs for popular services with one-click setup.
- **Source health dashboard** — show connection status, last sync, and error rates per source.
- **Source sharing** — export/import source configs between workspaces and users.
- **Rate limit awareness** — per-source rate limit tracking with automatic backoff.
- **Webhook sources** — inbound webhooks that trigger sessions when external events occur.
- **Source dependency graph** — visualize which sessions and automations depend on which sources.

---

## 3. Skills System

### Problem It Addresses
Users want to teach the agent specialized behaviors (code review, commit messages, API documentation) without repeating instructions every session. They also want Claude Code SDK compatibility for ecosystem reuse.

### Design
- **SKILL.md format** — identical to Claude Code SDK skills, using markdown frontmatter for metadata (`name`, `description`, `globs`, `alwaysAllow`, `requiredSources`).
- **Three-tier priority**: global (`~/.agents/skills/`) < workspace (`workspaces/{id}/skills/`) < project (`{projectRoot}/.agents/skills/`).
- **Auto-triggering** — `globs` patterns suggest skills when matching files are discussed.
- **Tool pre-approval** — `alwaysAllow` tools bypass permission prompts.
- **Source auto-enablement** — `requiredSources` automatically activates named sources.
- **Slash commands** — invoke via `/skill-name` in the chat input.
- **Validation** — `skill_validate` tool checks structure and frontmatter.
- **Custom icons** — auto-discovered `icon.svg`, `icon.png`, etc. per skill directory.
- **Cache with 5-minute TTL** — balances freshness with filesystem performance.

### Why It Matters (Product Value)
- Skills are the primary customization mechanism — users can tailor agent behavior without code changes.
- Claude Code SDK compatibility means existing skills from the ecosystem work immediately.
- Project-level skills enable team sharing via version control (`.agents/skills/` in repo root).
- Auto-triggering reduces friction — the right skill activates at the right time.

### Future Improvements
- **Skill marketplace** — share and discover community skills.
- **Skill versioning** — track skill changes over time with diffing.
- **Skill testing framework** — automated testing of skill behaviors with expected outputs.
- **Skill composition** — skills that chain or combine other skills.
- **Skill analytics** — track which skills are used most, success rates, and token costs.

---

## 4. Session Management

### Problem It Addresses
Conversations with agents are long-running and valuable. Users need to create, persist, search, organize, and resume sessions without losing context. The system must handle concurrent saves reliably and provide human-readable identifiers.

### Design
- **JSONL format** — header line (metadata) + message lines (one per JSON). Atomic writes via temp file + rename.
- **Human-readable session IDs** — format `YYMMDD-adjective-noun` (e.g., `260111-swift-river`), ~20,000 combinations/day.
- **Persistence Queue** (`persistence-queue.ts`) — debounced (500ms) async writes with coalescing, serialized per-session to prevent race conditions.
- **Portable paths** — `{{SESSION_PATH}}` token enables cross-platform path references.
- **Sub-directory structure**: `plans/`, `attachments/`, `data/`, `long_responses/`, `downloads/` — each session is self-contained.
- **AI-generated titles** — automatic session naming based on conversation content.
- **Resilient parsing** — corrupted lines are skipped, not fatal.
- **Header-only reads** — fast session listing without loading all messages.
- **Metadata signature tracking** — prevents clobbering external edits.

### Why It Matters (Product Value)
- Sessions are the core unit of work — reliable persistence directly impacts user trust.
- Human-readable IDs improve discoverability and communication (e.g., "check session 260418-swift-river").
- Sub-directories keep all session artifacts (files, plans, data) co-located for easy cleanup.
- JSONL format enables streaming reads and simple tooling integration.

### Future Improvements
- **Session branching** — fork a session at any point to explore alternative paths.
- **Session templates** — create new sessions from pre-configured templates (sources, skills, prompts).
- **Session export formats** — export to Markdown, PDF, or HTML for sharing outside the app.
- **Session analytics** — token usage, tool calls, and time metrics per session.
- **Session archiving policies** — auto-archive sessions older than N days.
- **Cross-workspace session search** — unified search across all workspaces.

---

## 5. Automations

### Problem It Addresses
Users need to react to events in real-time (label changes, tool completions, schedule triggers) and automate repetitive workflows without writing code. Manual monitoring of sessions and labels doesn't scale.

### Design
- **Event-driven architecture** — typed `WorkspaceEventBus` (observer pattern) with per-workspace isolation.
- **Event types**: App events (`LabelAdd`, `LabelRemove`, `PermissionModeChange`, `FlagChange`, `SessionStatusChange`, `SchedulerTick`) and Agent events (`PreToolUse`, `PostToolUse`, `UserPromptSubmit`, `SessionStart`, `SessionEnd`).
- **Action types**: Prompt (creates new session) and Webhook (sends HTTP request).
- **Cron scheduling** — `SchedulerService` ticks every minute, matching automations via cron expressions with timezone support.
- **Conditions system** — time, state, and logical composition (AND/OR/NOT) for fine-grained control.
- **Regex matchers** — filter events by pattern (e.g., match specific label names).
- **Environment variable expansion** — `$CRAFT_LABEL`, `$CRAFT_SESSION_ID`, `$CRAFT_WH_*` secrets injected into prompts/webhooks.
- **Rate limiting** — 10 events/min (60 for SchedulerTick) to prevent runaway automation.
- **Webhook retry** — exponential backoff for failed HTTP requests.
- **History tracking** — `automations-history.jsonl` records every fire.
- **Facade pattern** — `AutomationSystem` is the single entry point, no global state.

### Why It Matters (Product Value)
- Automations turn a reactive chat tool into a proactive workflow engine.
- Cron-based scheduling enables recurring tasks (daily standups, weekly summaries) without manual intervention.
- Webhook actions integrate with external tools (Slack notifications, CI/CD triggers).
- Event-driven prompts create context-aware agent sessions automatically.
- The natural language interface ("set up a daily standup every weekday at 9am") makes automation accessible to non-technical users.

### Future Improvements
- **Visual automation builder** — drag-and-drop UI for creating automation flows.
- **Automation templates** — pre-built automation patterns for common workflows.
- **Conditional branching** — if/else logic within a single automation.
- **Multi-step automations** — chain actions (prompt -> wait for response -> webhook).
- **Automation metrics** — success rates, execution times, and failure alerts.
- **Slack/Discord bot integration** — built-in webhook templates for popular services.

---

## 6. Permission Modes

### Problem It Addresses
Users need to control what the agent can do — from safe exploration to full auto-pilot. A binary allow/block is too coarse; users want graduated trust levels and fine-grained control over specific tools and commands.

### Design
- **Three modes**: Explore (safe, read-only), Ask to Edit (prompt for approval, default), Auto (allow all).
- **AST-based command validation** — bash and PowerShell commands are parsed into ASTs, not just regex-matched. Compound commands (`&&`, `||`, `|`) are validated per-segment.
- **Cascading permission rules** — workspace-level `permissions.json` extended by source-level configs.
- **Auto-scoped MCP patterns** — source permissions are automatically scoped (e.g., `list` becomes `mcp__linear__.*list`).
- **Write path restrictions** — glob patterns for allowed write directories with symlink escape prevention.
- **Blocked command hints** — contextual guidance when commands are rejected.
- **Incremental regex matching** — provides helpful mismatch diagnostics (shows how far the pattern matched).
- **Per-session mode** — each session has its own mode, changeable via SHIFT+TAB.

### Why It Matters (Product Value)
- Graduated trust makes the agent usable in production environments — users can start in Explore mode and escalate as needed.
- AST-based validation is more robust than regex-only approaches — handles compound commands, pipes, and subshells correctly.
- Source-scoped permissions enable multi-tenant use — different services get different access levels.
- The mode system is a key differentiator from raw API tools that have no execution boundary.

### Future Improvements
- **Custom permission presets** — user-defined modes beyond the three defaults.
- **Time-based mode restrictions** — auto-switch to Explore mode outside business hours.
- **Audit log** — detailed log of every permission decision for compliance.
- **Role-based permissions** — different users in a team get different mode capabilities.
- **Permission profiles** — save and share permission configurations.

---

## 7. Labels System

### Problem It Addresses
Sessions accumulate quickly. Users need a flexible, hierarchical categorization system to organize and filter sessions — flat tags aren't expressive enough, and folders are too rigid.

### Design
- **Hierarchical JSON tree** — labels can be nested up to 5 levels with `children` arrays.
- **Multi-select** — sessions can have many labels (additive, not exclusive).
- **Color-only visuals** — no icons, just colors for clean UI.
- **Dual format support** — system colors (`accent`, `info`, etc.) or custom hex with light/dark variants.
- **Auto-labeling rules** — regex patterns with capture groups that automatically label sessions based on user messages.
- **Value types** — `string`, `number`, `date` hints for structured labels (e.g., `priority::3`, `due::2026-01-30`).
- **Evaluation safety** — auto-rules only run on user messages, strip code blocks, limit 10 matches per message.
- **Array-position ordering** — label order in config equals display order.
- **Globally unique IDs** — slug-based, no duplicates.

### Why It Matters (Product Value)
- Hierarchical labels enable multi-dimensional session organization (e.g., `Engineering > Frontend > Bug` vs `Priority > High`).
- Auto-labeling reduces manual tagging — the system categorizes sessions as they happen.
- Multi-select labels are more flexible than single-select folders or statuses.
- Label-based filtering combined with automations enables powerful workflows (auto-escalate `urgent` sessions).

### Future Improvements
- **Label-based analytics** — aggregate metrics by label (time spent, token usage, completion rate).
- **Smart label suggestions** — AI-suggested labels based on conversation content.
- **Label colors from workspace theme** — auto-generate harmonious colors from the theme palette.
- **Cross-workspace label taxonomy** — shared label hierarchies across workspaces.
- **Label-based batch operations** — archive, export, or delete all sessions with a given label.

---

## 8. Status System

### Problem It Addresses
Sessions need a workflow state (backlog -> todo -> in-progress -> done) to manage work like a kanban board. Without statuses, users can't distinguish active work from completed work at a glance.

### Design
- **Exclusive, single-select** — each session has exactly one status (contrast with multi-label).
- **Two categories**: `open` (inbox/active list) and `closed` (archive/completed list).
- **Fixed vs default vs custom types** — `todo`, `done`, `cancelled` are fixed (cannot delete/rename); `backlog`, `needs-review` are default (modifiable); users can create custom statuses.
- **Color + icon** — system colors with optional opacity, or custom hex. Icons from local file, emoji, or URL.
- **Self-healing configuration** — if required statuses are missing, they're re-created on load.
- **Icon resolution chain** — local file -> emoji -> fallback.
- **Status-based filtering** — inbox shows `open`, archive shows `closed`.

### Why It Matters (Product Value)
- The inbox/archive split mirrors how people actually work — active vs done.
- Fixed statuses prevent accidental deletion of critical workflow states.
- Custom statuses adapt to any workflow (e.g., `awaiting-deploy`, `in-review`, `blocked`).
- Combined with automations, status changes can trigger actions (notify Slack when status becomes `needs-review`).

### Future Improvements
- **Custom workflow rules** — define allowed status transitions (e.g., can't go from `done` to `backlog`).
- **Status duration tracking** — show how long sessions stay in each status.
- **SLA timers** — alert when sessions exceed time thresholds in certain statuses.
- **Status-based sorting** — sort inbox by status priority, not just date.
- **Bulk status updates** — select multiple sessions and change status at once.

---

## 9. Theme System

### Problem It Addresses
Users spend hours in the app. Visual customization reduces eye strain, expresses personal or brand identity, and differentiates workspaces. A hardcoded theme is a usability and accessibility issue.

### Design
- **6-color system**: `background`, `foreground`, `accent`, `info`, `success`, `destructive`.
- **Cascading hierarchy**: App default (Settings -> Appearance) < Workspace override < Preset themes < Theme overrides.
- **Multiple color formats**: Hex, RGB, HSL, OKLCH (recommended for perceptual uniformity), named colors.
- **Dark mode** — optional `dark` object for per-color overrides.
- **Scenic mode** — full-window background image with glass panels.
- **Live updates** — changes apply immediately, no restart required.
- **Preset themes** — `~/.craft-agent/themes/{name}.json` packages for sharing.
- **CSS variable integration** — themes map directly to CSS custom properties.

### Why It Matters (Product Value)
- Theme customization is a table-stakes feature for desktop apps — its absence signals a lack of polish.
- Workspace-level themes help users visually distinguish contexts (work vs personal vs side-project).
- OKLCH color format ensures perceptual uniformity — colors look equally vibrant across the palette.
- Dark mode is essential for developer productivity and accessibility.

### Future Improvements
- **Theme marketplace** — share and discover community themes.
- **Auto-theme from wallpaper** — extract colors from the desktop wallpaper.
- **Time-based themes** — switch between light and dark based on time of day.
- **High-contrast accessibility themes** — WCAG AA/AAA compliant presets.
- **Theme preview** — live preview before applying changes.

---

## 10. Browser Tools

### Problem It Addresses
Many tasks require web interaction — filling forms, reading dashboards, scraping data, testing UIs. Without built-in browser control, users must switch to a separate browser automation tool and manually transfer context.

### Design
- **Single unified tool** — `browser_tool` command with CLI-like syntax: `open`, `navigate`, `snapshot`, `find`, `click`, `fill`, `select`, `type`, `screenshot`, `drag`, `click-at`.
- **Accessibility tree** — `snapshot` returns element refs (`@e1`, `@e2`) instead of raw HTML, reducing token usage.
- **Batch commands** — semicolon-separated, stops after navigation for safety.
- **Canvas UI support** — `click-at` and `drag` for coordinate-based interaction (Google Sheets, maps).
- **Clipboard integration** — `set-clipboard`, `get-clipboard`, `paste` with TSV support for spreadsheet workflows.
- **Annotated screenshots** — element refs overlaid on the image for visual reference.
- **Security challenge detection** — recognizes Cloudflare and similar challenges.
- **Allowed in Explore mode** — browser tools are safe (no file writes), so they work in read-only mode.
- **Viewport settle polling** — waits for page stability after interactions.

### Why It Matters (Product Value)
- Built-in browser control eliminates context-switching — the agent sees, navigates, and extracts from the web in the same conversation.
- Accessibility tree snapshots are dramatically more token-efficient than full HTML dumps.
- Canvas interaction support handles modern web apps that other tools can't.
- Allowed in Explore mode means users can browse safely without worrying about accidental writes.

### Future Improvements
- **Multi-tab management** — open and switch between multiple tabs.
- **Network interception** — mock API responses for testing.
- **PDF printing** — save pages as PDF from the browser.
- **Video/canvas recording** — record interactions for replay or documentation.
- **Browser profiles** — separate cookie jars and sessions for different accounts.

---

## 11. Data Tables & transform_data

### Problem It Addresses
Agents frequently produce structured data (comparisons, reports, query results). Plain markdown tables are limited — no sorting, filtering, or interactivity. Large datasets blow up token budgets when rendered inline.

### Design
- **Three formats**: Markdown tables (< 20 rows), `datatable` blocks (interactive with sort/filter/group-by/search), `spreadsheet` blocks (Excel/CSV export).
- **Column types**: `text`, `number`, `currency`, `percent`, `boolean`, `date`, `badge`.
- **`transform_data` tool** — runs Python/Node/Bun scripts in an isolated subprocess (30s timeout, no network, path sandboxing). Input files relative to session dir, output to `data/`.
- **File-backed tables** — datasets with 20+ rows are written to JSON and referenced via `src`, keeping inline content small.
- **Merge semantics** — inline `columns` override file-backed values.
- **Security** — blocked env vars, no network access, isolated subprocess.

### Why It Matters (Product Value)
- Interactive tables are a major UX upgrade over static markdown — users can explore data without asking the agent to re-query.
- `transform_data` enables complex data processing (CSV parsing, multi-source joins, filtering) without bloating the conversation.
- Spreadsheet export makes agent output immediately usable in Excel/Google Sheets.
- File-backed large datasets keep token usage manageable even with hundreds of rows.

### Future Improvements
- **Chart generation** — auto-generate charts from table data (bar, line, pie).
- **Pivot tables** — interactive pivot table support.
- **Live data connections** — tables that auto-refresh from source data.
- **Collaborative editing** — multiple users edit the same table.
- **SQL query mode** — run SQL queries against datatable blocks.

---

## 12. Mermaid Diagrams

### Problem It Addresses
Agents need to communicate complex relationships (architecture, flows, timelines) visually. Text descriptions are insufficient for diagrams, and external diagramming tools break the conversation flow.

### Design
- **Native Mermaid rendering** — supports flowcharts (LR/RL/TD/BT), sequence diagrams, Gantt charts, pie charts, mind maps, ER diagrams, class diagrams, and user journeys.
- **Themed SVG output** — diagrams use the workspace theme colors.
- **Custom styling** — subgraphs, various node shapes and arrow types.
- **Horizontal preference** — LR layouts recommended for better viewing in chat panels.

### Why It Matters (Product Value)
- Diagrams make agent output significantly more comprehensible — a flowchart beats a paragraph of text for system design.
- Native rendering keeps everything in-context — no external tools or image uploads.
- Themed output maintains visual consistency with the rest of the app.

### Future Improvements
- **Interactive diagrams** — clickable nodes that navigate to sessions or sources.
- **Diagram versioning** — track diagram changes across conversation turns.
- **Export formats** — PNG, SVG, or PDF download.
- **More chart types** — Kanban boards, org charts, network graphs.

---

## 13. LLM Tool (Secondary Model)

### Problem It Addresses
Some subtasks are better handled by a different model — summarization with a fast/cheap model, complex analysis with a powerful model, or classification with a specialized fine-tuned model. Using the primary agent for everything wastes tokens and time.

### Design
- **`createLLMTool()` factory** — produces a tool that invokes a secondary LLM with custom parameters.
- **Attachments support** — text files (UTF-8, with line ranges for >2000 lines) and images (5MB limit).
- **Structured output** — `outputFormat` and `outputSchema` for guaranteed JSON responses (API key only; prompt-based for OAuth).
- **Thinking support** — extended thinking with configurable budget (API key only).
- **Constraint limits** — max 20 attachments, 2000 lines/500KB per file, 5MB per image, 2MB total.
- **Authentication-aware** — different capabilities for API key vs OAuth auth paths.
- **Timeout wrapper** — prevents runaway sub-agent calls.

### Why It Matters (Product Value)
- Enables model orchestration — use the right model for the right subtask.
- Structured output is critical for programmatic consumption (other tools, automations).
- Parallel LLM calls (via tool use) speed up multi-step analysis.
- Keeps the primary agent's context window focused on the main task.

### Future Improvements
- **Model chaining** — automatic model selection based on task type.
- **Cost-aware routing** — balance quality vs cost for secondary calls.
- **Streaming sub-agent responses** — show partial results as they arrive.
- **Tool-use in sub-agents** — allow the secondary model to call tools.

---

## 14. Multi-File Diff Viewer

### Problem It Addresses
When agents modify multiple files in a single turn, users need to review all changes before accepting. Line-by-line terminal diffs don't scale to multi-file changes with hundreds of lines.

### Design
- **VS Code-style side-by-side diff** — familiar UX for developers.
- **Multi-file navigation** — browse all changed files in a turn.
- **Syntax highlighting** — language-aware coloring.
- **Change navigation** — jump between additions, deletions, and modifications.

### Why It Matters (Product Value)
- Code review is the primary trust-building moment — users must see exactly what changed before accepting.
- Multi-file view is essential for refactors and features that touch multiple files.
- VS Code-style UX reduces the learning curve for developer users.

### Future Improvements
- **Inline commenting** — add comments to specific diff lines.
- **Partial acceptance** — accept some changes and reject others.
- **Diff between turns** — compare changes across conversation turns.
- **Three-way merge** — show base, agent, and user versions for conflict resolution.

---

## 15. File Attachments & Preview

### Problem It Addresses
Users need to provide context from diverse file types — screenshots, PDFs, Office documents, code files — to the agent. Manual conversion to text is tedious and loses formatting.

### Design
- **Drag-and-drop interface** — attach files directly to messages.
- **Supported formats**: Images (PNG, JPG, GIF, WebP, SVG, BMP, ICO, AVIF), PDFs (with text extraction), Office documents (DOCX, XLSX, PPTX), code files.
- **Auto-conversion** — files are converted to markdown/text for agent consumption.
- **Preview rendering**: inline (400px max-height with fade) and fullscreen (full navigation, page counter, text selection).
- **HTML preview** — sandboxed iframe for email bodies, newsletters, and HTML reports (JS blocked, forms blocked, navigation blocked).
- **PDF preview** — first page inline, full navigation in fullscreen.
- **Image preview** — object-contain with item navigation.

### Why It Matters (Product Value)
- File attachments make the agent useful for real-world tasks involving documents, screenshots, and data files.
- Auto-conversion eliminates format barriers — the agent can work with any file type.
- Preview rendering keeps users in-context — no need to open external viewers.
- HTML sandboxing enables safe rendering of email content and web reports.

### Future Improvements
- **Audio/video attachments** — transcription and analysis of multimedia files.
- **Batch attachment processing** — attach and process multiple files in one message.
- **Attachment search** — full-text search across all attachments in a workspace.
- **OCR for images** — extract text from screenshots and photos.

---

## 16. Internationalization (i18n)

### Problem It Addresses
The product serves a global audience. A UI only in English excludes non-English-speaking users and limits market reach. Hard-coded strings make translation impossible.

### Design
- **Flat dot-notation keys** — `common.cancel`, `menu.toggleSidebar` — simple and tooling-friendly.
- **Registry pattern** — `LOCALE_REGISTRY` as single source of truth for locale metadata.
- **Lazy locale loading** — translations loaded on demand, not bundled upfront.
- **date-fns locale support** — date formatting respects locale conventions.
- **Brand name preservation** — product names kept in English across all locales.
- **Developer tooling** — pre-commit hooks for detecting hardcoded strings.
- **Supported languages**: English (en), Spanish (es), Simplified Chinese (zh-Hans), Japanese (ja), Hungarian (hu), German (de), Polish (pl).

### Why It Matters (Product Value)
- i18n is essential for global adoption — the product is immediately useful to millions of non-English speakers.
- Community translations scale organically — users can contribute translations for their languages.
- Developer tooling (hardcoded string detection) prevents i18n regressions.

### Future Improvements
- **RTL support** — Arabic, Hebrew, and other right-to-left languages.
- **Crowdsourced translations** — platform for community translation contributions.
- **Automatic translation** — AI-powered translation of new strings.
- **Locale-specific number/currency formatting** — beyond date formatting.
- **More languages** — prioritize based on user geography data.

---

## 17. Headless Server & Remote Access

### Problem It Addresses
Users want to run agents on powerful remote machines (GPUs, large memory) and access them from lightweight clients. They also need long-running sessions that survive client disconnections, and team access from multiple machines.

### Design
- **Headless Bun server** — runs without Electron, deployable on any machine (Linux VPS, Docker).
- **WebSocket RPC** — replaced Electron IPC with WS-based communication for local/remote transparency.
- **Hybrid transport** — unified workspace connection state for local and remote workspaces.
- **TLS support** — `wss://` required for remote connections, self-signed cert support for development.
- **Docker support** — multi-platform build, `docker-compose.yml` with WebUI assets baked in.
- **Multiple remote workspaces** — connect to multiple servers simultaneously.
- **Chunked session transfers** — large sessions transferred via chunked WebSocket with base64 encoding.
- **Session export/import** — "Send to Workspace" transfers sessions between servers.
- **Remote workspace recovery** — reconnect disconnected workspaces from the switcher.
- **Server lock management** — stale `.server.lock` file handling on restart.

### Why It Matters (Product Value)
- Remote server support is critical for enterprise and power-user adoption — it enables GPU workloads, persistent sessions, and team access.
- Docker support simplifies deployment to any cloud provider.
- "Send to Workspace" enables cross-machine workflows — start on desktop, continue on laptop.
- Hybrid transport means the same codebase handles local and remote transparently.

### Future Improvements
- **Kubernetes operator** — deploy and manage Craft Agent servers on K8s.
- **Server clustering** — load-balance sessions across multiple servers.
- **Server health monitoring** — dashboard for resource usage, session counts, and uptime.
- **Automatic session failover** — migrate sessions to healthy servers on failure.
- **SSH tunnel support** — connect to servers behind firewalls without exposing ports.

---

## 18. CLI Client

### Problem It Addresses
Developers and CI/CD pipelines need programmatic access to the agent. A GUI-only tool can't be scripted, automated, or integrated into build systems.

### Design
- **Full command set**: `ping`, `health`, `workspaces`, `sessions`, `connections`, `sources`, `session create/delete/messages`, `send`, `cancel`, `invoke` (raw RPC), `listen` (push events), `run` (self-contained).
- **Entity management**: labels, sources, skills, automations, permissions, themes — full CRUD via CLI.
- **Input modes**: flat flags, `--json` for structured data, `--stdin` for piped input.
- **Output contract**: JSON envelope with `ok`, `data`/`error`, `warnings`.
- **`run` command** — fully self-contained: spawns server, creates session, sends prompt, streams response, exits.
- **Multi-provider** — `--provider` and `--model` flags for any LLM backend.
- **Validation mode** — `--validate-server` runs 21-step integration test.
- **JSON output** — `--json` flag for scripting and piping.

### Why It Matters (Product Value)
- CLI access is essential for developer adoption — it integrates with existing workflows (terminal, scripts, CI/CD).
- The `run` command is the simplest possible entry point — one command, no setup.
- Entity management via CLI enables automation of configuration (bootstrap workspaces in CI).
- Validation mode provides a built-in smoke test for deployments.

### Future Improvements
- **Watch mode** — monitor sessions and stream new messages.
- **Interactive mode** — REPL-like interface for multi-turn CLI conversations.
- **Output templating** — format output with custom templates (e.g., Markdown report).
- **CLI completions** — shell auto-completion for commands and flags.
- **Session replay** — replay a session from the CLI for debugging.

---

## 19. Web UI

### Problem It Addresses
Not all users have or want the desktop app. Mobile users, team members on shared machines, and remote access scenarios need a browser-based interface.

### Design
- **Browser-based interface** — runs on the same port as the headless server.
- **Mobile responsive** — 120% zoom, 1.3x touch targets, iOS Safari optimizations.
- **OAuth relay** — unified browser-based authentication for sources via WebUI.
- **Progressive Web App** — favicon, manifest, and Apple touch icon.
- **Login/authentication** — JWT-based with `argon2id` password hashing and global rate limiting.
- **Full feature parity** — session management, settings, source and skill management.

### Why It Matters (Product Value)
- Web access dramatically expands the addressable user base — no install required.
- Mobile responsive design enables on-the-go agent access.
- PWA support means the Web UI can be installed on mobile home screens.
- Team access — multiple users can connect to the same remote server.

### Future Improvements
- **Offline mode** — service worker caching for offline session viewing.
- **Push notifications** — browser notifications for automation events.
- **Collaborative sessions** — multiple users in the same session (real-time cursors).
- **Mobile-native features** — camera integration for image attachments, share sheet support.

---

## 20. MCP Integration & Client Pool

### Problem It Addresses
Model Context Protocol (MCP) is the emerging standard for connecting AI agents to external tools and data. Without MCP support, the agent is limited to built-in tools. Without connection pooling, MCP server connections are inefficient and unstable.

### Design
- **Dual transport** — HTTP/SSE for remote MCP servers, stdio for local subprocesses.
- **Connection pooling** (`McpClientPool`) — centralized pool with health checks, reconnection, and lifecycle management.
- **Proxy tool naming** — `mcp__{slug}__{toolName}` format prevents naming collisions between sources.
- **Config change detection** — pool reconciles active sources on config changes.
- **Binary file handling** — images, audio, and other binary responses saved to `downloads/`.
- **Large response summarization** — tool responses exceeding ~60KB are auto-summarized with intent-aware context.
- **Intent injection** — `_intent` field added to MCP tool schemas to preserve summarization focus.
- **Local MCP isolation** — sensitive environment variables filtered from stdio subprocesses.
- **In-process API sources** — API sources masquerade as MCP clients via `ApiSourcePoolClient`.

### Why It Matters (Product Value)
- MCP is becoming the standard integration layer — supporting it means access to a growing ecosystem of tools.
- Connection pooling ensures stability — reconnections, health checks, and lifecycle management prevent stale connections.
- Proxy naming prevents collisions when multiple MCP servers provide tools with the same name.
- Large response summarization keeps token budgets manageable without losing key information.

### Future Improvements
- **MCP resource subscriptions** — real-time updates when MCP resources change.
- **MCP sampling support** — allow MCP servers to request LLM completions.
- **MCP server marketplace** — browse and install MCP servers from a registry.
- **Connection analytics** — latency, error rates, and tool usage per MCP server.
- **Offline MCP support** — cache MCP tool results for offline access.

---

## 21. Workspace Management

### Problem It Addresses
Users work on multiple projects with different sources, skills, and settings. Mixing everything in one space creates confusion, security risks, and context pollution.

### Design
- **Workspace-scoped data**: sessions, sources, skills, labels, statuses, automations, themes — all isolated per workspace.
- **Directory structure** — `~/.craft-agent/workspaces/{id}/` with dedicated sub-directories.
- **Workspace config** — `config.json` per workspace with default theme, status, and settings.
- **CI workspace support** — programmatic workspace creation for automation.
- **Workspace switching** — switch between workspaces without restarting.
- **Cross-workspace sharing** — "Send to Workspace" for sources, skills, automations, and sessions.
- **Per-workspace default model** — different workspaces can use different LLM providers.

### Why It Matters (Product Value)
- Workspace isolation prevents cross-contamination — work sources don't leak into personal sessions.
- Per-workspace settings (model, theme, permissions) adapt the tool to the context.
- CI workspace support enables programmatic agent usage in pipelines.
- Cross-workspace sharing reduces duplication — configure once, share everywhere.

### Future Improvements
- **Workspace templates** — pre-configured workspaces for common use cases (software development, content creation, data analysis).
- **Team workspaces** — shared workspaces with role-based access control.
- **Workspace analytics** — aggregate metrics across sessions within a workspace.
- **Workspace backup/restore** — export and import entire workspaces.
- **Workspace cloning** — duplicate a workspace for experimentation.

---

## 22. Search

### Problem It Addresses
As sessions accumulate, finding relevant past conversations becomes critical. Users need fuzzy search across session titles, messages, and metadata without exact-match requirements.

### Design
- **Sublime Text-style fuzzy matching** — scoring algorithm with position weighting.
- **Ranked results** — search results ordered by relevance score with match highlighting.
- **Multi-field search** — session titles, message content, labels, and statuses.
- **Filter composition** — combine search with label and status filters.
- **Cross-node matching** — search works within code blocks and formatted content.
- **CSS Custom Highlight API** — modern browser highlighting for search results.

### Why It Matters (Product Value)
- Search is the primary navigation mechanism for historical sessions — without it, old work is effectively lost.
- Fuzzy matching handles typos and partial queries — users don't need exact phrasing.
- Filter composition enables precise targeting (e.g., "urgent label + status:todo + 'deploy'").

### Future Improvements
- **Semantic search** — vector-based similarity search for finding conceptually related sessions.
- **Search suggestions** — auto-complete based on past queries and popular terms.
- **Saved searches** — persistent search filters accessible from the sidebar.
- **Search analytics** — track what users search for to identify knowledge gaps.
- **Full-text search across attachments** — search within PDF, DOCX, and other attached files.

---

## 23. Scheduler

### Problem It Addresses
Automations need time-based triggers (cron schedules). Without a built-in scheduler, recurring tasks require external cron daemons or manual execution.

### Design
- **Cron expression parsing** — standard 5-field cron syntax.
- **Timezone support** — per-automation timezone for correct scheduling across regions.
- **Event emission** — `SchedulerTick` events emitted to the automation system every minute.
- **Service lifecycle** — `start()`, `stop()`, `setSchedule()` for runtime control.
- **Integration with automations** — tick events are matched against automation cron expressions.

### Why It Matters (Product Value)
- Cron-based scheduling enables recurring workflows (daily standups, weekly reports, hourly checks) without external infrastructure.
- Timezone support prevents scheduling errors for distributed teams.
- Built-in scheduling keeps the system self-contained — no external cron daemon needed.

### Future Improvements
- **Calendar-based scheduling** — schedule based on calendar events (e.g., "before every meeting").
- **Adaptive scheduling** — adjust frequency based on previous run results (e.g., check more often if issues found).
- **Schedule previews** — show next N execution times when creating an automation.
- **Schedule conflict detection** — warn when multiple automations would fire simultaneously.

---

## 24. Deep Linking

### Problem It Addresses
External apps (browsers, email clients, scripts) need to navigate directly to specific sessions, settings, or actions within the desktop app. Without deep links, users must manually navigate to the destination.

### Design
- **`craftagents://` URL scheme** — registered with the OS for cross-app navigation.
- **Supported URLs**: `allSessions`, `allSessions/session/{id}`, `settings`, `sources/source/{slug}`, `action/new-chat`.
- **OS-level registration** — protocol handler registered during app installation.

### Why It Matters (Product Value)
- Deep links enable integration with external tools — browsers, email, and scripts can open specific sessions.
- `action/new-chat` enables "open in Craft Agents" buttons in web tools.
- Essential for notification workflows — click a Slack notification to jump directly to the relevant session.

### Future Improvements
- **Deep link with pre-filled prompt** — open a new session with a pre-loaded message.
- **Deep link with source/skill activation** — open a session with specific sources enabled.
- **Deep link search** — open the app with a pre-filled search query.
- **Universal links (macOS/iOS)** — HTTP-based links that work across platforms.

---

## 25. Tool Icons

### Problem It Addresses
CLI commands shown in the UI are visually monotonous and hard to distinguish. Users need visual cues to quickly identify which tools and commands were executed in a session.

### Design
- **Custom icon mapping** — `~/.craft-agent/tool-icons/tool-icons.json` maps commands to icons.
- **Bundled defaults** — ~57 built-in tool icons (git, docker, npm, etc.) that are never overwritten.
- **Complex bash matching** — handles command chains, prefixes, and pipes.
- **Icon resolution** — `id` + `displayName` + `icon` file + `commands` array for matching.

### Why It Matters (Product Value)
- Visual tool identification improves session readability — users can scan a conversation and quickly see which tools were used.
- Custom icons allow users to add icons for their own tools and scripts.
- Bundled defaults provide a polished out-of-the-box experience.

### Future Improvements
- **Auto-icon fetching** — automatically download icons for detected CLI tools.
- **Icon themes** — switch between icon sets (outlined, filled, monochrome).
- **Tool usage visualization** — show tool frequency and patterns across sessions.

---

## 26. Large Response Handling

### Problem It Addresses
Tool responses (file contents, API responses, database query results) can exceed 60KB, consuming excessive tokens and slowing the conversation. Raw large responses overwhelm the context window.

### Design
- **Automatic summarization** — responses exceeding ~60KB are summarized using Claude Haiku.
- **Intent-aware context** — `_intent` field injected into MCP tool schemas preserves summarization focus.
- **Long response storage** — full responses saved to `long_responses/` directory for reference.
- **Section breakdown** — error messages include section breakdown for large responses that can't be processed.

### Why It Matters (Product Value)
- Token efficiency — summarization keeps context windows manageable without losing key information.
- Cost reduction — summarized responses use fewer tokens in subsequent turns.
- Full responses are preserved — users can access the complete output when needed.
- Intent-aware summarization ensures the summary focuses on what matters for the current task.

### Future Improvements
- **Configurable summarization thresholds** — let users set their own size limits.
- **Progressive loading** — load summarized version first, expand on demand.
- **Smart extraction** — extract only the relevant sections based on the query intent.
- **Response caching** — cache summarized responses to avoid re-processing.

---

## 27. Background Tasks

### Problem It Addresses
Some agent operations (file processing, API calls, long-running scripts) take significant time. Blocking the UI during execution is unacceptable — users need to continue working while tasks run.

### Design
- **Progress tracking** — real-time status updates during execution.
- **Task cancellation** — abort long-running operations.
- **Non-blocking execution** — tasks run in background threads/processes.
- **Status notifications** — users are notified when tasks complete or fail.

### Why It Matters (Product Value)
- Background execution is essential for usability — users can start a task and continue other work.
- Cancellation prevents runaway tasks from consuming resources indefinitely.
- Progress tracking reduces uncertainty — users know how long to expect.

### Future Improvements
- **Task queue management** — prioritize, reorder, and schedule background tasks.
- **Task dependencies** — chain tasks that depend on each other.
- **Task history** — log of all background tasks with results and timing.
- **Resource limits** — per-task CPU and memory limits.

---

## 28. Configuration System

### Problem It Addresses
The application has many configurable aspects (workspaces, connections, preferences, themes) that need consistent validation, migration, and persistence. Ad-hoc config handling leads to corruption and user confusion.

### Design
- **Layered storage**: `~/.craft-agent/config.json` (global), `workspaces/{id}/config.json` (workspace), `preferences.json` (user prefs), `theme.json` (app theme).
- **Zod schema validation** — runtime validation for all config files with clear error messages.
- **Migration support** — automatic schema migration when config format changes between versions.
- **Watch/reload** — config changes detected and applied in real-time during development.
- **Bundled defaults** — `config-defaults.json` synced from app bundle on every launch.

### Why It Matters (Product Value)
- Reliable configuration is foundational — every other feature depends on it.
- Schema validation prevents corruption — malformed configs are caught early with helpful error messages.
- Auto-migration means users don't need to manually update configs when upgrading.
- Bundled defaults ensure a consistent starting point across installations.

### Future Improvements
- **Config UI** — visual settings interface for all configuration options.
- **Config profiles** — save and switch between configuration profiles.
- **Config sync** — synchronize configuration across devices.
- **Config diffing** — show what changed when config is modified.
- **Config backup** — automatic backups before destructive changes.

---

## Cross-Cutting Observations

### Design Patterns Used Throughout
- **Repository pattern** — storage layers for sessions, sources, skills, labels, statuses.
- **Factory pattern** — agent backends, API tools, LLM tools.
- **Observer/Event pattern** — event bus, automation triggers, config watchers.
- **Builder pattern** — server configs, LLM requests.
- **Pool pattern** — MCP connections.
- **Facade pattern** — automation system, mode manager.
- **Strategy pattern** — OAuth providers, credential managers.

### Storage Formats
- **JSONL** for session data (streaming-friendly, line-oriented).
- **JSON** for configuration (human-readable, tooling-friendly).
- **Markdown** for skills and guides (Git-friendly, AI-friendly).
- **AES-256-GCM encrypted files** for credentials.

### Key Architectural Decisions
1. **Workspace isolation** — all data scoped to workspace, not global.
2. **Agent-driven configuration** — users describe intent, agent handles setup.
3. **Dual transport (local + remote)** — same codebase handles both transparently.
4. **JSONL over SQLite** — simpler, streaming-friendly, no database dependency.
5. **OKLCH for colors** — perceptually uniform color system.
6. **Accessibility tree over raw HTML** — massive token savings for browser tools.
7. **Proxy tool naming** — prevents MCP tool name collisions.
8. **Atomic file writes** — temp file + rename prevents corruption.
