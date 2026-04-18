# Craft Agents vs Goose — 横向架构比较

> 基于 Craft Agents OSS (TypeScript/Electron, v0.8.9) 与 Goose (Rust/Tauri, v1.31.0) 的设计文档和源码进行的架构对比评定。

---

## 1. 项目基本面对照

| 维度 | Craft Agents | Goose |
|------|-------------|-------|
| 语言 | TypeScript (strict) | Rust |
| 运行时 | Bun + Node.js + Electron | Tokio async + Tauri v2 |
| 桌面框架 | Electron 33+ | Tauri v2 (从 Electron 迁移) |
| UI 框架 | React 19 + Jotai + Tailwind | React + TypeScript |
| 代码规模 | ~70,000 行 TS | ~95,000 行 Rust |
| 包结构 | Monorepo (9 packages + 4 apps) | Cargo workspace (多个 crates) |
| 许可证 | 开源 | Apache 2.0 |
| 发布节奏 | 按功能发布 (v0.2.30 → v0.8.9) | 周发 (v1.8.0 → v1.31.0) |

---

## 2. 核心架构对比

### Craft Agents: 多进程 WebSocket RPC

```
┌─────────────┐    WebSocket     ┌──────────────┐
│ Electron    │◀────────────────▶│ SessionMgr   │
│ Renderer    │   JSON codec     │ AgentBackend │
│ (React)     │                  │ MCP Pool     │
└─────────────┘                  └──────┬───────┘
       │                                │
  RoutedClient                    ┌─────▼─────┐
  (local/remote split)            │ PiAgent   │
                                  │ (subproc) │
                                  └───────────┘
```

特点: 渲染器和服务器通过 WebSocket RPC 通信，即使在同一进程内也是如此。RoutedClient 透明地将流量分流到本地或远程服务器。

### Goose: Agent Client Protocol (ACP)

```
┌─────────────┐    ACP/localhost   ┌──────────────┐
│ Tauri/CLI   │◀─────────────────▶│ Agent::reply │
│ React/ratatui│   SSE + JSON      │ Provider     │
│              │                    │ Extensions   │
└─────────────┘                    └──────────────┘
```

特点: 单一 `Agent::reply()` 入口，所有 UI 表面 (Desktop/CLI/Server) 共享同一个 Rust core。通过 ACP 协议标准化通信。

### 评定

**Craft 优势**: 原生支持远程服务器 (thin client 模式)，workspace 级别的流量路由是内置的。

**Goose 优势**: 单一二进制部署更简洁；Rust 的内存安全和零成本抽象在并发场景下有性能优势。

---

## 3. Agent Runtime 对比

### Craft: 双后端策略

```
BaseAgent (abstract)
├── ClaudeAgent  → Claude Agent SDK (in-process)
└── PiAgent      → Pi SDK subprocess (JSONL/stdio)
```

Claude SDK 的工具通过 in-process MCP server 注册；Pi SDK 的工具通过 JSONL proxy 回到父进程执行。两者共享 `BaseAgent` 的 permission、source resolution、streaming 逻辑。

### Goose: Provider Trait 单一抽象

```rust
trait Provider {
    fn stream(&self, messages, tools) -> MessageStream;
}
```

所有 25+ provider 共享一个 trait。ToolShim 机制让不支持原生 tool calling 的模型也能调用工具 (通过文本标记解析)。Agent loop 是一个无限循环，streaming 输出 `BoxStream<AgentEvent>`。

### 评定

| 维度 | Craft Agents | Goose |
|------|-------------|-------|
| Provider 数量 | ~10 | 25+ |
| 工具调用兼容性 | 仅支持原生 tool calling 的模型 | ToolShim 兼容非原生模型 |
| Agent 后端数量 | 2 (Claude SDK + Pi SDK) | 1 (统一 Provider trait) |
| 进程模型 | 混合 (in-process + subprocess) | 统一 (单进程) |
| 复杂度来源 | 双后端维护成本 | ToolShim 的额外 round-trip |

**Goose 优势**: Provider 生态更广 (28+)，ToolShim 让几乎所有模型都能使用工具。

**Craft 优势**: Claude SDK in-process 路径延迟更低 (工具调用 ~5ms vs ~50ms JSONL)；Pi subprocess 提供了进程级隔离。

---

## 4. Session 存储对比

### Craft: JSONL 文件

```
sessions/{id}/session.jsonl
  Line 1: SessionHeader (metadata)
  Line 2+: StoredMessage (one per line)
```

- 原子写入 (tmp + rename)
- 可移植路径 (~/ 和 {{SESSION_PATH}})
- Debounced 持久化 (100ms)
- Header-only 快速列表 (<1ms per session)
- 配置文件变更通过 ConfigWatcher (chokidar) 监控

### Goose: SQLite

```sql
sessions table (20+ columns)
messages table (JSON content)
threads table (session grouping)
```

- WAL mode 并发读写
- 30 秒 busy timeout
- Schema 迁移 (目前 version 10)
- `BEGIN IMMEDIATE` 防死锁
- SessionManager 全局单例

### 评定

| 维度 | Craft Agents (JSONL) | Goose (SQLite) |
|------|---------------------|----------------|
| 查询能力 | 有限 (全扫描或内存索引) | SQL 全功能 |
| 并发安全 | 文件锁 + debounce | SQLite WAL |
| 可移植性 | 极好 (纯文本，跨机器) | 需导出/导入 |
| 复杂度 | 低 | 中 (schema 迁移，连接池) |
| 备份友好性 | 高 (文本文件) | 低 (需 SQLite 工具) |
| 聚合查询 | 需加载全部数据 | SQL 原生支持 |

**Craft 优势**: JSONL 的可读性和可移植性更好。可以直接用文本编辑器查看 session 内容。跨机器同步只需文件复制。

**Goose 优势**: SQLite 在大量 session (500+) 的查询和过滤场景下性能更好。Thread (session 分组) 功能在 SQL 中实现更自然。

---

## 5. MCP / 工具系统对比

### Craft: 单工具 CLI 模式

```
browser_tool({ command: "navigate https://example.com" })
browser_tool({ command: "fill @e3 hello; click @submit" })  // batch
```

- 单一 `browser_tool` 包含 40+ 命令
- Session-scoped 工具通过 callback registry 绑定
- MCP pool 管理连接池
- API source 自动生成工具 (Google, Slack, Microsoft)
- 浏览器工具用 accessibility tree + @eN ref (token 高效)

### Goose: 四层工具架构

```
Layer 1: Tool Resolution (name → extension mapping)
Layer 2: Inspection Pipeline (5 sequential inspectors)
Layer 3: Execution (MCP clients, platform extensions)
Layer 4: Response Processing (large response handling)
```

- 12 个内置 platform extensions (developer, analyze, orchestrator, summon, todo, apps...)
- 8 种 extension transport (stdio, HTTP, builtin, platform, frontend, inline Python, SSE, Unix socket)
- Inspector pipeline: Security → Egress → Adversary → Permission → Repetition
- 大响应 (>200K chars) 自动重定向到临时文件

### 评定

| 维度 | Craft Agents | Goose |
|------|-------------|-------|
| 工具暴露方式 | 1 个 browser_tool + MCP tools | 每个 MCP tool 独立暴露 |
| LLM 工具发现成本 | 极低 (1 tool + CLI commands) | 随工具数量线性增长 |
| 安全检查层 | Permission mode + bash validator | 5 层 inspector pipeline |
| Extension transport | 3 (stdio, SSE, HTTP) | 8 (含 builtin, platform, frontend, Python, Unix socket) |
| 浏览器自动化 | 内置 (CDP + accessibility tree) | 无内置，依赖外部 MCP server |
| API source 自动工具生成 | 有 (Google/Slack/Microsoft) | 无 |
| 批量命令 | 有 (分号分隔) | 无 |

**Craft 优势**: 内置浏览器自动化是差异化竞争力。API source 自动工具生成让用户无需编码即可接入 REST API。单工具 CLI 模式对 LLM 更友好。

**Goose 优势**: Inspector pipeline 的安全纵深更强。Platform extensions 直接访问 agent 进程内部能力，无需子进程开销。Extension transport 选择更丰富。

---

## 6. 安全与权限对比

### Craft: 三模式权限

```
safe (Explore)  →  read-only
ask (Ask Edit)  →  user approval per action
allow-all (Auto)→  auto-approve
```

- BashValidator (627 行) 和 PowerShellValidator (1,095 行) 做命令分类
- Credential: AES-256-GCM + machine-bound key
- Browser cookie partition per session
- CAPTCHA 自动检测 + 用户交接

### Goose: 五层 Inspector Pipeline

```
SecurityInspector  → pattern + ML threat detection
EgressInspector    → outbound network logging
AdversaryInspector → LLM-based adversarial review
PermissionInspector→ user-configured rules
RepetitionInspector→ infinite loop detection
```

- OSV 恶意包检查
- 容器隔离 (optional sandboxing)
- SmartApprove 模式: LLM 判断是否为只读操作
- Platform keychain 集成 (macOS Keychain, Linux Secret Service, Windows Credential Manager)

### 评定

| 维度 | Craft Agents | Goose |
|------|-------------|-------|
| 安全纵深层数 | 3 (mode + validator + credential) | 5 (inspectors) + OSV + sandbox |
| LLM 辅助安全 | 无 | 有 (AdversaryInspector + SmartApprove) |
| 凭据存储 | 自管 AES-256-GCM | OS keychain + file fallback |
| 循环检测 | 无内置 | RepetitionInspector |
| 出站流量监控 | 无 | EgressInspector (passive logging) |
| 恶意扩展检查 | 无 | OSV database lookup |
| CAPTCHA 处理 | 自动检测 + 用户交接 | 无 |

**Goose 优势**: 安全纵深明显更强。五层 inspector + OSV + sandbox 形成多重防线。LLM 辅助判断 (AdversaryInspector, SmartApprove) 是创新点。

**Craft 优势**: CAPTCHA 检测和用户交接是实用功能。三模式设计更简单直观。

---

## 7. 上下文管理对比

### Craft: Debounce + Save

- Session 通过 JSONL 持久化，debounced 100ms
- ConfigWatcher 监控文件变更
- 无内置 context compaction 机制
- 依赖 Claude SDK / Pi SDK 自身的 context 管理

### Goose: 三层 Context 管理

```
Layer 1: Tool Pair Summarization (background task)
  → 后台 tokio task，batch 10 tool calls per summarization

Layer 2: Auto-Compaction (80% threshold trigger)
  → Progressive tool response removal: 0% → 10% → 20% → 50% → 100%

Layer 3: Recovery Compaction (on ContextLengthExceeded)
  → Summary message replaces old conversation
  → Original messages kept as agent_invisible
```

- TokenCounter: o200k_base tokenizer, DashMap cache (10,000 entries)
- Prompt caching optimization: hourly timestamps, alphabetical tool sorting
- MOIM (Message-Optimized Injection): runtime context injection without persistence

### 评定

**Goose 优势显著**: 三层 context 管理系统是 Goose 的核心差异化能力。Progressive compaction 策略确保长对话不会因 context 溢出而失败。Tool pair summarization 在后台异步运行，不阻塞主循环。

**Craft**: 依赖外部 SDK 的 context 管理。对于长对话 (50+ turns)，可能出现 context 溢出问题，需要用户手动清理或重新开始 session。

---

## 8. 浏览器工具对比

### Craft: 内置浏览器自动化

```
3-layer architecture:
  Layer 1: BrowserPaneFns interface (portable)
  Layer 2: SessionManager wiring
  Layer 3: BrowserPaneManager (Electron CDP)

Features:
  - Accessibility tree snapshots (@eN refs)
  - 40+ CLI commands (navigate, click, fill, type, scroll, ...)
  - Batch commands (semicolon-separated)
  - CAPTCHA detection + user handoff
  - Visual feedback (cursor animation, agent badge)
  - Session-bound instances (1:1 lazy binding)
  - Cookie partition per session
```

### Goose: 无内置浏览器

- 依赖外部 MCP browser server (如 Playwright MCP)
- 无 CAPTCHA 检测
- 无视觉反馈
- 无 session-bound 浏览器实例

### 评定

**Craft 的内置浏览器工具是显著的差异化优势**: 从工具层到 UI 层完整集成的浏览器自动化能力，包括 CDP 集成、accessibility tree、CAPTCHA 处理、视觉反馈、session 隔离。这在桌面 AI agent 产品中是独特的。

---

## 9. 远程部署对比

### Craft: 原生支持远程

```
RoutedClient:
  LOCAL_ONLY channels → embedded server
  REMOTE_ELIGIBLE channels → remote headless server

Features:
  - Headless server binary
  - WebSocket RPC with TLS
  - Event replay on reconnection
  - RoutedClient transparent traffic split
  - WebUI on same port
  - JWT session cookies
```

### Goose: `goose serve` (较新)

```
HTTP/WebSocket API with SSE streaming
SessionEventBus for multi-client
OpenAPI spec generation
Local-only bind by default
```

### 评定

**Craft 优势**: 远程部署是 Craft 的核心设计之一。RoutedClient 的透明路由、event buffer + replay、workspace switching 都是生产级特性。

**Goose**: `goose serve` 是近期才添加的功能 (v1.29+)，成熟度较低。

---

## 10. 通知和自动化对比

### Craft: 事件驱动自动化引擎

```
Events → Matcher Engine → Actions (Prompt / Webhook)

Features:
  - 10+ event types (agent + app)
  - Regex matcher + cron scheduler
  - Condition system (time, state, logical AND/OR/NOT)
  - Prompt action (auto-create session + send)
  - Webhook action (HTTP request with template variables)
  - History logging (JSONL)
  - Retry queue for failed webhooks
```

### Goose: Recipes + Scheduling

```
Features:
  - YAML-based prompt templates (recipes)
  - MiniJinja template rendering
  - Scheduled sessions (cron-like)
  - .goosehints for project context
```

### 评定

**Craft 优势**: 自动化引擎更完整。条件系统 + webhook + retry queue 形成了完整的自动化闭环。

**Goose 优势**: Recipe 系统 (YAML + MiniJinja) 更适合开发者创建可复用的 prompt 模板。.goosehints 的文件级上下文注入很实用。

---

## 11. 多 Provider 支持

| Provider | Craft Agents | Goose |
|----------|:----------:|:-----:|
| Anthropic (API key) | ✓ | ✓ |
| Claude (OAuth) | ✓ | ✓ |
| OpenAI | ✓ | ✓ |
| Google AI Studio | ✓ | ✓ |
| ChatGPT Plus | ✓ | ✓ (CLI provider) |
| GitHub Copilot | ✓ | ✓ |
| OpenRouter | ✓ | ✓ |
| Ollama | ✓ | ✓ |
| Amazon Bedrock | ✓ | ✓ |
| Google Vertex | — | ✓ |
| Azure OpenAI | — | ✓ |
| Databricks | — | ✓ |
| IBM watsonx | — | ✓ |
| DeepSeek | — | ✓ |
| Mistral | — | ✓ |
| Cohere | — | ✓ |
| Novita AI | — | ✓ |
| KimiCode | — | ✓ |
| Llama-swap | — | ✓ |
| 自定义 endpoint | ✓ | ✓ (declarative YAML) |

**Goose 覆盖 28+ providers，远超 Craft 的 ~10 个。** Declarative provider system 让添加新 provider 只需 YAML 配置，无需写 Rust 代码。

---

## 12. 可观测性对比

### Craft: 基础日志

- electron-log (file + console)
- Debug flag for verbose output
- Health check HTTP endpoint
- Sentry error reporting (with data scrubbing)

### Goose: 生产级可观测性

- OpenTelemetry 全链路追踪 (session.id propagation)
- PostHog analytics
- Langfuse LLM observability
- tracing crate (structured logging)
- Feature-gated telemetry (隐私控制)
- OpenSSF Scorecard

### 评定

**Goose 优势**: OpenTelemetry + Langfuse 的可观测性栈在调试 LLM 交互和分析 token 使用方面远优于 Craft 的基础日志。

---

## 13. 综合评分

| 维度 (1-10) | Craft Agents | Goose | 说明 |
|-------------|:----------:|:-----:|------|
| **架构清晰度** | 9 | 8 | Craft 的 3-layer 浏览器 + callback registry 更干净；Goose 的 singleton 全局状态增加测试复杂度 |
| **Provider 生态** | 6 | 9 | Goose 28+ providers + declarative YAML |
| **浏览器自动化** | 9 | 3 | Craft 内置完整方案；Goose 依赖外部 MCP |
| **安全纵深** | 6 | 9 | Goose 5 层 inspector + OSV + sandbox |
| **上下文管理** | 5 | 9 | Goose 三层 compaction 系统 |
| **远程部署** | 9 | 5 | Craft 原生支持；Goose serve 较新 |
| **自动化能力** | 8 | 6 | Craft 完整引擎；Goose recipes + scheduling |
| **存储设计** | 7 | 8 | Goose SQLite 查询更强；Craft JSONL 可移植性更好 |
| **可观测性** | 5 | 8 | Goose OTEL + Langfuse |
| **CI/测试** | 6 | 9 | Goose 40 workflows + record/replay testing |
| **API 集成** | 8 | 5 | Craft 自动工具生成；Goose 需 MCP server |
| **桌面体验** | 8 | 7 | Craft 多 panel + browser 视图；Goose Tauri 更轻量 |
| **LLM token 效率** | 8 | 7 | Craft 单工具 CLI 模式 + accessibility tree |
| **加权总分** | **7.5** | **7.3** | |

---

## 14. 核心差异化

### Craft Agents 的独特优势

1. **内置浏览器自动化**: 从 CDP 到 UI 的完整集成，accessibility tree + @eN ref + CAPTCHA 处理 + 视觉反馈
2. **原生远程部署**: RoutedClient 透明路由 + event replay + workspace switching
3. **API source 自动工具生成**: Google/Slack/Microsoft REST API 自动变成可调用工具
4. **单工具 CLI 模式**: 降低 LLM 工具发现成本，batch commands 提高效率

### Goose 的独特优势

1. **Provider 生态**: 28+ providers + declarative YAML + ToolShim 兼容非原生模型
2. **安全纵深**: 5 层 inspector pipeline + OSV + sandbox + LLM 辅助判断
3. **上下文管理**: 三层 compaction 系统，长对话不因 context 溢出而失败
4. **可观测性**: OpenTelemetry + Langfuse 全链路追踪
5. **Rust 性能**: 内存安全 + 零成本抽象 + 单一二进制部署 (50MB vs Electron 300MB)
6. **CI 成熟度**: 40 workflows + record/replay testing + 5 平台构建

---

## 15. 针对你们的 Electron 桌面 AI Agent 的建议

基于两个项目的对比，建议在以下方面参考各项目的优势：

### 从 Craft 学习
- **浏览器自动化**: 3-layer architecture + BrowserPaneFns interface + accessibility tree
- **API source 自动工具生成**: 低代码接入 REST API 的模式
- **自动化引擎**: 事件匹配 + 条件系统 + webhook + retry queue
- **远程部署**: RoutedClient 透明路由设计

### 从 Goose 学习
- **上下文管理**: 三层 compaction 系统 (背景总结 + 自动压缩 + 紧急恢复)
- **安全纵深**: Inspector pipeline 模式，每层独立，fail-open 设计
- **Provider 扩展**: Declarative provider system 降低新 provider 接入成本
- **可观测性**: OpenTelemetry 集成对调试 LLM 交互至关重要
- **ToolShim**: 兼容不支持原生 tool calling 的模型的能力

### 避免两个项目的共同短板
- Craft 缺少 Goose 的 context compaction 和安全纵深
- Goose 缺少 Craft 的浏览器自动化和 API source 工具生成
- 两者都缺少对非技术用户的引导 (Goose 有 recipe 但门槛仍然较高)
