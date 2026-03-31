# Claude Code Fork vs OpenCode — 深度对比分析

> 本文档对 [DonutShinobu/claude-code-fork](https://github.com/DonutShinobu/claude-code-fork)（Anthropic Claude Code 泄露源码）与 [anomalyco/opencode](https://github.com/anomalyco/opencode)（开源 AI 编码助手）进行完整详细的架构分析与对比。

---

## 目录

1. [项目概述](#一项目概述)
2. [claude-code-fork 结构分析](#二claude-code-fork-结构分析)
3. [opencode 结构分析](#三opencode-结构分析)
4. [架构对比](#四架构对比)
5. [功能特性对比](#五功能特性对比)
6. [技术栈对比](#六技术栈对比)
7. [各自优势分析](#七各自优势分析)
8. [总结](#八总结)

---

## 一、项目概述

| 属性 | claude-code-fork | opencode |
|---|---|---|
| **性质** | Anthropic Claude Code 泄露源码（2026-03-31） | 100% 开源 AI 编码助手 |
| **License** | 所有权归 Anthropic（泄露代码） | MIT 开源 |
| **语言** | TypeScript（严格模式） | TypeScript（严格模式） |
| **运行时** | Bun | Bun |
| **代码规模** | ~1,900 文件，~512,000+ 行 | 多包 Monorepo，~19 个子包 |
| **仓库结构** | 单体 src/ 目录 | Turborepo Monorepo |
| **AI 提供商** | 仅 Anthropic Claude | 多模型（Claude、OpenAI、Google、本地模型） |
| **发布状态** | 泄露代码，无官方维护 | 活跃开发中，有商业支持 |

---

## 二、claude-code-fork 结构分析

### 2.1 顶层目录结构

```
claude-code-fork/
├── README.md               # 项目说明（含泄露背景）
└── src/                    # 全部源码（~1,900 文件）
    ├── main.tsx             # CLI 入口（Commander.js）
    ├── QueryEngine.ts       # LLM 核心引擎（~46K 行）
    ├── Tool.ts              # 工具类型定义（~29K 行）
    ├── Task.ts              # 任务执行模型
    ├── commands.ts          # 命令注册中心（~25K 行）
    ├── tools.ts             # 工具注册中心
    ├── context.ts           # 系统/用户上下文收集
    ├── cost-tracker.ts      # Token 费用追踪
    ├── history.ts           # 会话历史管理
    ├── query.ts             # 查询管道
    ├── setup.ts             # 初始化
    ├── tasks.ts             # 任务工具函数
    │
    ├── tools/               # 工具实现（~184 文件，~40+ 工具）
    ├── commands/            # 斜杠命令（~207 文件，~55 命令）
    ├── components/          # Ink/React UI 组件（~140 文件）
    ├── services/            # 外部服务集成（~130 文件）
    ├── hooks/               # React 自定义 Hooks
    ├── screens/             # 全屏 UI（Doctor、REPL、Resume）
    ├── types/               # TypeScript 类型定义（~40 文件）
    ├── utils/               # 工具函数（~400 文件）
    ├── state/               # 状态管理（~10 文件）
    ├── schemas/             # Zod 配置 Schema（~15 文件）
    ├── bridge/              # IDE 集成桥接（~31 文件）
    ├── coordinator/         # 多 Agent 协调器
    ├── server/              # 服务器模式（直连）
    ├── remote/              # 远程会话管理
    ├── plugins/             # 插件系统
    ├── skills/              # 技能系统
    ├── memdir/              # 持久内存目录
    ├── tasks/               # 任务管理
    ├── keybindings/         # 键位配置
    ├── vim/                 # Vim 模式
    ├── voice/               # 语音输入
    ├── migrations/          # 配置迁移
    ├── entrypoints/         # 初始化入口
    ├── outputStyles/        # 输出样式
    ├── query/               # 查询管道
    ├── ink/                 # Ink 渲染封装
    ├── context/             # 上下文 Provider
    ├── bootstrap/           # 启动引导
    ├── native-ts/           # 原生 TS 工具
    ├── upstreamproxy/       # 代理配置
    ├── assistant/           # 助手模式
    ├── buddy/               # 彩蛋精灵
    └── moreright/           # UI 相关
```

### 2.2 核心模块详解

#### `QueryEngine.ts`（核心 LLM 引擎）

这是整个系统最重要的文件，约 46,000 行。负责：
- 构建系统提示词（System Prompt）
- 管理消息历史（Message History）
- 执行工具调用循环（Tool Call Loop）
- 处理流式响应（Streaming）
- 实现思考模式（Thinking Blocks）
- 重试逻辑与错误分类
- Token 计数与费用追踪

#### `tools/`（工具层，~40 个工具）

每个工具是独立的自包含模块，包含：输入 Schema、权限模型、执行逻辑、UI 组件。

| 类别 | 工具 |
|---|---|
| 文件操作 | FileReadTool, FileWriteTool, FileEditTool |
| Shell/代码 | BashTool, PowerShellTool, GlobTool, GrepTool |
| Web | WebFetchTool, WebSearchTool |
| AI/Agent | AgentTool, SkillTool, MCPTool, LSPTool |
| 笔记本 | NotebookEditTool |
| 任务管理 | TaskCreateTool, TaskGetTool, TaskListTool, TaskUpdateTool, TaskOutputTool, TaskStopTool |
| 多 Agent | SendMessageTool, TeamCreateTool, TeamDeleteTool |
| 模式控制 | EnterPlanModeTool, ExitPlanModeTool, EnterWorktreeTool, ExitWorktreeTool |
| 工具发现 | ToolSearchTool |
| 调度 | ScheduleCronTool, RemoteTriggerTool, SleepTool |
| 其他 | SyntheticOutputTool, AskUserQuestionTool, TodoWriteTool, REPLTool, BriefTool, ConfigTool |

#### `commands/`（斜杠命令，~55 个）

通过 `/` 前缀调用的用户命令：

| 类别 | 命令 |
|---|---|
| 会话管理 | /resume, /compact, /break-cache |
| 代码导航 | /context, /review, /diff, /ctx_viz |
| VCS 集成 | /commit, /commit-push-pr, /branch, /autofix-pr, /pr_comments |
| 配置 | /config, /theme, /output-style, /keybindings |
| 开发工具 | /ide, /desktop, /mobile, /skills, /mcp, /doctor |
| 认证 | /login, /logout |
| 用户管理 | /profile, /cost, /tasks, /memory, /effort |
| 辅助 | /vim, /share, /init, /help, /onboarding |

#### `services/`（服务层，~20 个子系统）

- **MCP**：Model Context Protocol 服务管理（40+ 文件，OAuth 流程）
- **API**：Anthropic API 调用、文件 API、引导数据、日志
- **Analytics**：GrowthBook 特性标志与事件
- **OAuth**：OAuth 2.0 认证流
- **LSP**：Language Server Protocol 集成
- **compact/autoDream**：上下文压缩与会话记忆整合
- **extractMemories**：自动记忆提取
- **teamMemorySync**：团队记忆同步
- **plugins/tools**：插件加载器
- **remoteManagedSettings**：云端同步配置

#### `bridge/`（IDE 桥接层，~31 文件）

连接 VS Code 和 JetBrains 扩展与 Claude Code CLI 的双向通信层：
- `bridgeMain.ts` — 主桥接循环
- `replBridge.ts` — REPL 会话桥接
- `bridgeMessaging.ts` — 消息协议路由
- `jwtUtils.ts` — JWT 认证
- `sessionRunner.ts` — 会话执行管理

#### `coordinator/`（多 Agent 协调）

`coordinatorMode.ts` 实现多 Agent 协调逻辑：
- 检查 Coordinator 模式是否激活
- 管理 Worker Agent 工具访问权限
- 处理 MCP 客户端暴露
- 会话模式匹配

#### `remote/` 与 `server/`（远程与服务器模式）

- `RemoteSessionManager.ts` — 远程会话生命周期
- `SessionsWebSocket.ts` — WebSocket 通信
- `createDirectConnectSession.ts` — 直接服务器连接
- `remotePermissionBridge.ts` — 远程权限桥接

### 2.3 核心架构模式

```
用户输入
    │
    ▼
main.tsx (Commander.js CLI)
    │
    ├── React/Ink UI 渲染
    │
    ▼
QueryEngine.ts
    │
    ├── 构建 System Prompt
    ├── 调用 Anthropic API（流式）
    ├── 解析 Tool Call
    ├── 权限系统检查
    ├── 执行工具
    └── 返回响应
         │
         ├── tools/ (工具执行层)
         ├── services/ (外部服务)
         ├── bridge/ (IDE 集成)
         ├── coordinator/ (多 Agent)
         └── remote/ (远程会话)
```

### 2.4 关键设计特性

1. **并行预加载**：启动时并行执行 MDM 读取 + Keychain 预取，优化启动时间
2. **惰性加载**：OpenTelemetry (~400KB)、gRPC (~700KB) 等重型模块延迟动态导入
3. **特性标志**：通过 `bun:bundle` 的 `feature()` 函数实现死代码消除
4. **Agent 集群**：通过 `AgentTool` 生成子 Agent，`coordinator/` 协调多 Agent 并行
5. **技能系统**：`skills/` 定义可复用工作流
6. **持久记忆**：`memdir/` 跨会话存储记忆
7. **宏观权限模型**：`default`、`plan`、`bypassPermissions`、`auto` 等多种权限模式

---

## 三、opencode 结构分析

### 3.1 Monorepo 结构

opencode 采用 **Turborepo** 管理的多包 Monorepo，共 ~19 个子包：

```
opencode/
├── packages/
│   ├── opencode/            # 核心 CLI 包（主包）
│   ├── console/             # Web 控制台（SST + React）
│   │   ├── app/             # 前端应用（React）
│   │   ├── core/            # 核心业务逻辑
│   │   ├── function/        # 无服务器函数
│   │   └── mail/            # 邮件服务
│   ├── app/                 # 主应用包
│   ├── desktop/             # 桌面应用（Tauri/Electron）
│   ├── desktop-electron/    # Electron 桌面包
│   ├── containers/          # 容器化部署
│   ├── docs/                # 文档（Mintlify）
│   ├── enterprise/          # 企业功能包
│   ├── extensions/          # 浏览器/IDE 扩展
│   ├── function/            # 云函数
│   ├── identity/            # 身份认证包
│   ├── plugin/              # 插件 SDK
│   ├── script/              # 构建脚本
│   ├── sdk/                 # 公开 SDK
│   ├── slack/               # Slack 集成
│   ├── storybook/           # UI 组件故事书
│   ├── ui/                  # UI 组件库
│   ├── util/                # 工具函数库
│   └── web/                 # 官网（营销页）
│
├── infra/                   # SST 基础设施即代码
├── sdks/                    # 多语言 SDK
├── specs/                   # 规格文档
├── nix/                     # Nix flake 配置
├── sst.config.ts            # SST（Ion）部署配置
└── turbo.json               # Turborepo 配置
```

### 3.2 核心包详解

#### `packages/opencode/`（核心 CLI）

```
packages/opencode/
├── src/                     # TypeScript 源码
├── test/                    # 测试套件
├── specs/                   # 行为规格
├── migration/               # 数据库迁移（Drizzle ORM）
├── drizzle.config.ts        # 数据库配置
├── parsers-config.ts        # 多语言解析器配置
└── package.json             # 依赖声明
```

**架构特点：Client/Server 分离**
- 后端以 HTTP 服务器运行，暴露 REST/WebSocket API
- TUI（终端 UI）作为独立客户端连接
- 支持远程访问（手机/Web 驱动本地 agent）

**Agent 系统：**
- `build` — 默认全权限开发 agent
- `plan` — 只读分析 agent（默认拒绝文件编辑）
- `general` — 通用子 agent（`@general` 调用）

**数据层：**
- 使用 **Drizzle ORM** 做会话持久化
- SQLite 本地存储

#### `packages/console/`（Web 控制台）

基于 **SST（Ion）** 构建的云端 Web 控制台，包含：
- React 前端应用
- Serverless 后端函数
- 邮件服务

#### `packages/desktop/` + `packages/desktop-electron/`（桌面应用）

提供原生桌面应用：
- **Tauri**（Rust + WebView，轻量）或 **Electron** 两种方案
- 跨平台：macOS、Windows、Linux

#### `packages/sdk/`（公开 SDK）

允许第三方开发者基于 opencode 构建集成。

#### `packages/enterprise/`（企业功能）

面向企业的扩展功能（访问控制、团队管理等）。

#### `packages/slack/`（Slack 集成）

原生 Slack 机器人集成。

### 3.3 opencode 核心架构

```
客户端层                     服务器层
─────────────               ─────────────────────
TUI（终端）   ──HTTP/WS──▶  opencode server
Desktop App  ──HTTP/WS──▶      │
Mobile App   ──HTTP/WS──▶      ├── LLM 提供商路由
Web Console  ──HTTP/WS──▶      │   ├── Claude (Anthropic)
                               │   ├── GPT (OpenAI)
                               │   ├── Gemini (Google)
                               │   └── 本地模型 (Ollama等)
                               │
                               ├── 工具执行层
                               ├── Session 管理 (SQLite/Drizzle)
                               ├── LSP 集成
                               └── MCP 支持
```

---

## 四、架构对比

### 4.1 整体架构模型

| 维度 | claude-code-fork | opencode |
|---|---|---|
| **架构风格** | 紧耦合单体（Monolith） | Client/Server 分离 Monorepo |
| **包组织** | 单 src/ 目录 | Turborepo 多包 |
| **前后端分离** | 前后端集成在一个进程 | TUI 是独立客户端 |
| **数据库** | 无（内存 + 文件） | SQLite + Drizzle ORM |
| **测试** | 无测试文件 | 有 test/ 目录 + specs/ |
| **部署** | npm 包分发 | npm + 桌面 App + 云端 |

### 4.2 代码规模对比

| 指标 | claude-code-fork | opencode |
|---|---|---|
| TypeScript 文件数 | ~1,884 | 多包合计约数百 |
| 代码行数 | ~512,000 行 | 相对精简 |
| 目录数量 | 301 | 分散在 19 个子包 |
| 核心文件 | QueryEngine.ts (46K行) | 模块化拆分 |

### 4.3 工具系统对比

| 工具 | claude-code-fork | opencode |
|---|---|---|
| Bash 执行 | ✅ BashTool | ✅ |
| 文件读写编辑 | ✅ FileRead/Write/EditTool | ✅ |
| Glob/Grep | ✅ | ✅ |
| Web 搜索/抓取 | ✅ WebFetchTool + WebSearchTool | ✅ |
| LSP 集成 | ✅ LSPTool | ✅（开箱即用，原生支持） |
| MCP 支持 | ✅ MCPTool（完整 OAuth） | ✅ |
| Jupyter 笔记本 | ✅ NotebookEditTool | ❌ |
| 多 Agent 团队 | ✅ TeamCreateTool | 有限 |
| Task 管理 | ✅ 完整任务系统 | 基础 |
| 语音输入 | ✅ voice/ | ❌ |
| Vim 模式 | ✅ vim/ | ❌ |
| Git Worktree | ✅ EnterWorktreeTool | ❌ |
| 调度/Cron | ✅ ScheduleCronTool | ❌ |

### 4.4 AI 模型支持

| 提供商 | claude-code-fork | opencode |
|---|---|---|
| Anthropic Claude | ✅（唯一支持） | ✅ |
| OpenAI GPT | ❌ | ✅ |
| Google Gemini | ❌ | ✅ |
| 本地模型（Ollama 等） | ❌ | ✅ |
| 自定义 Endpoint | ❌ | ✅ |

### 4.5 集成与生态

| 集成 | claude-code-fork | opencode |
|---|---|---|
| VS Code 扩展 | ✅ bridge/（深度集成） | 有扩展包 |
| JetBrains 插件 | ✅ bridge/ | ❌ |
| 桌面应用 | ✅（/desktop 命令交接） | ✅ 原生桌面 App（Tauri/Electron） |
| Web 控制台 | ❌ | ✅ console/ |
| 手机遥控 | 有限（/mobile） | ✅（Client/Server 架构天然支持） |
| Slack 集成 | ❌ | ✅ slack/ |
| GitHub Actions | 有限 | 有计划 |
| 公开 SDK | ❌ | ✅ sdk/ |

---

## 五、功能特性对比

### 5.1 用户体验

| 特性 | claude-code-fork | opencode |
|---|---|---|
| 终端 UI | React/Ink | React/Ink（neovim 团队设计） |
| Vim 键位 | ✅ 原生 vim/ 模块 | 基础支持 |
| 语音输入 | ✅ voice/ | ❌ |
| 多主题 | ✅ /theme | ✅ |
| 输出样式 | ✅ outputStyles/ | ✅ |
| 键位自定义 | ✅ keybindings/ | 有限 |
| 彩蛋 | ✅ buddy/ 精灵 | ❌ |

### 5.2 会话与记忆

| 特性 | claude-code-fork | opencode |
|---|---|---|
| 会话恢复 | ✅ /resume | ✅ |
| 上下文压缩 | ✅ compact + autoDream | ✅ |
| 持久记忆 | ✅ memdir/ | 有限 |
| 团队记忆同步 | ✅ teamMemorySync/ | ❌ |
| 自动记忆提取 | ✅ extractMemories/ | ❌ |
| 数据库持久化 | ❌（文件系统） | ✅ SQLite/Drizzle |

### 5.3 权限与安全

| 特性 | claude-code-fork | opencode |
|---|---|---|
| 权限模式 | default/plan/bypass/auto 四种模式 | build/plan 双 agent |
| 工具级权限 | ✅（每工具独立） | ✅ |
| 权限拒绝追踪 | ✅ denialTracking | 有限 |
| 企业策略限制 | ✅ policyLimits/ | ✅ enterprise/ |
| macOS Keychain | ✅ | ❌ |
| JWT 认证 | ✅（bridge） | 有限 |

### 5.4 开发者生态

| 特性 | claude-code-fork | opencode |
|---|---|---|
| 插件系统 | ✅ plugins/ | ✅ plugin/ + SDK |
| 技能系统 | ✅ skills/ | 无 |
| 公开 SDK | ❌ | ✅ |
| MCP 支持 | ✅ 完整（含 OAuth） | ✅ |
| 扩展文档 | ❌（泄露代码无文档） | ✅ docs/ + opencode.ai/docs |
| 贡献指南 | ❌ | ✅ CONTRIBUTING.md |
| 测试 | ❌ | ✅ test/ + specs/ |

---

## 六、技术栈对比

| 技术 | claude-code-fork | opencode |
|---|---|---|
| **运行时** | Bun | Bun |
| **语言** | TypeScript（严格） | TypeScript（严格） |
| **终端 UI** | React + Ink | React + Ink |
| **CLI 解析** | Commander.js | Commander.js |
| **Schema 验证** | Zod v4 | Zod |
| **代码搜索** | ripgrep (GrepTool) | ripgrep |
| **协议** | MCP SDK + LSP | MCP SDK + LSP |
| **AI SDK** | Anthropic SDK（唯一） | AI SDK（多模型） |
| **遥测** | OpenTelemetry + gRPC | 有限 |
| **特性标志** | GrowthBook | 无 |
| **认证** | OAuth 2.0 + JWT + macOS Keychain | OAuth 2.0 |
| **数据库** | 无 | SQLite + Drizzle ORM |
| **构建工具** | Bun bundle + 特性标志 | Turborepo |
| **包管理** | 单包 | Monorepo（19 包） |
| **桌面** | 交接给桌面 App | Tauri / Electron 原生 |
| **基础设施** | - | SST（Ion）云端部署 |
| **容器化** | - | Docker（containers/） |
| **Nix** | - | ✅ flake.nix |

---

## 七、各自优势分析

### 7.1 claude-code-fork（Anthropic Claude Code）的优势

#### 🏆 功能深度与完整性
- **工具系统极为丰富**：~40 个工具，覆盖 Jupyter 笔记本、语音输入、Git Worktree 隔离、任务调度、多 Agent 团队协作等，远超竞品
- **多 Agent 架构**：完整的 Coordinator 模式 + TeamCreateTool，支持真正的 Agent 集群并行作业
- **技能系统**：可复用工作流 skills/，用户可自定义技能
- **持久记忆**：memdir/ 跨会话长期记忆 + 自动提取 + 团队同步，记忆系统非常完善

#### 🏆 IDE 深度集成
- **VS Code + JetBrains** 双 IDE 原生桥接（bridge/，31 文件），双向实时通信
- REPL 桥接模式，无缝嵌入 IDE 工作流
- JWT 安全认证

#### 🏆 性能优化
- **并行启动**：MDM 读取 + Keychain 预取同时进行，最小化启动延迟
- **惰性加载**：大模块（OpenTelemetry ~400KB、gRPC ~700KB）延迟加载
- **特性标志裁剪**：通过 `bun:bundle` feature() 彻底消除未用代码

#### 🏆 企业级能力
- **组织策略限制**：policyLimits/ 支持企业管控
- **远程管理设置**：云端下发配置（remoteManagedSettings/）
- **MDM 支持**：企业移动设备管理集成
- **macOS Keychain**：原生安全存储凭据
- **GrowthBook**：完整的特性标志 A/B 测试系统

#### 🏆 用户体验细节
- **Vim 模式**（完整 vim/ 模块）
- **语音输入**（voice/）
- **键位完全自定义**（keybindings/）
- **彩蛋**（buddy/ 精灵）
- **50+ 斜杠命令**，覆盖所有常见工作流

#### 🏆 上下文管理
- **autoDream** 自动上下文整合
- **extractMemories** 自动提取关键记忆
- **teamMemorySync** 团队记忆同步

---

### 7.2 opencode（anomalyco/opencode）的优势

#### 🏆 开源与透明度
- **100% MIT 开源**，无泄露风险，可自由商用和部署
- 活跃的社区贡献（Discord、X.com）
- 完整的 CONTRIBUTING.md 和文档

#### 🏆 模型无关性（Provider Agnostic）
- 支持 **Claude、GPT、Gemini、本地模型**（Ollama 等）
- 任意 OpenAI 兼容 Endpoint
- 随着模型竞争加剧，用户可选择性价比最高的模型
- 不被单一厂商锁定

#### 🏆 Client/Server 架构
- TUI 仅是一种客户端，**任何 HTTP 客户端**都能驱动 agent
- 天然支持**手机远程驱动本地 agent**
- 支持无头模式（headless）自动化
- 多客户端同时连接

#### 🏆 桌面原生应用
- **Tauri（Rust + WebView）**：轻量原生桌面 App，包体小
- **Electron** 备选方案
- 跨平台：macOS、Windows、Linux 全覆盖
- DMG/EXE/DEB/RPM/AppImage 多格式发布

#### 🏆 Web 控制台
- **packages/console/**：完整的 Web 管理控制台（SST + React）
- 浏览器访问 AI 助手

#### 🏆 公开 SDK 与插件生态
- **packages/sdk/**：公开 SDK，第三方可基于 opencode 构建
- **packages/plugin/**：插件 SDK
- 扩展生态更开放

#### 🏆 企业集成
- **packages/slack/**：原生 Slack 机器人
- **packages/enterprise/**：企业功能包
- **packages/identity/**：身份认证服务
- **packages/containers/**：容器化部署

#### 🏆 数据可靠性
- **SQLite + Drizzle ORM**：结构化会话持久化，数据迁移有版本管理
- **test/ + specs/**：有测试套件保证质量

#### 🏆 开发者友好
- **Turborepo Monorepo**：清晰的包边界，独立发布
- **Storybook**：UI 组件开发环境
- **Nix flake**：可复现的开发环境
- **Docker 化**：容器部署

#### 🏆 TUI 极致追求
- 由 **neovim 用户** 和 **terminal.shop 创始人** 打造
- 承诺持续推进终端 UI 极限体验
- 内置 Tab 键快速切换 build/plan 两种 agent

---

## 八、总结

### 一句话对比

| | claude-code-fork | opencode |
|---|---|---|
| **定位** | 功能最全面的 AI 编码助手（私有） | 最开放灵活的 AI 编码助手（开源） |
| **最适合** | 深度 Claude 用户、企业 IDE 集成 | 开源爱好者、多模型用户、自托管需求 |
| **核心差异** | 深度 × 广度（功能最多） | 开放性 × 灵活性（生态最好） |

### 关键结论

**claude-code-fork 胜在：**
- 功能数量和深度（工具数、命令数均领先 2-3 倍）
- 多 Agent 编排能力（Coordinator + Team 模式）
- IDE 深度集成（VS Code + JetBrains 双桥接）
- 企业安全与管控（MDM、Keychain、策略限制）
- 语音输入、Vim 模式等专项体验

**opencode 胜在：**
- 开源免费、社区活跃、无法律风险
- 支持任意 LLM 提供商（不绑 Anthropic）
- Client/Server 架构（更灵活的客户端）
- 原生桌面 App + Web 控制台
- 有公开 SDK 和插件生态
- 有测试套件、完整文档

**对于大多数用户**：如果希望完全开源 + 多模型灵活性，选 **opencode**；如果深度使用 Claude 且需要最完整功能集（特别是多 Agent 和 IDE 集成），**claude-code-fork** 所展示的架构是业界领先的参考实现。

---

*分析时间：2026-03-31 | claude-code-fork commit: a99de1b | opencode branch: dev*
