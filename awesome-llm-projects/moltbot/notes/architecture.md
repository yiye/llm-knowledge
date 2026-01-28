# Moltbot 架构分析

> **研究版本**: v2026.1.27-beta.1  
> **分析日期**: 2026-01-28  
> **代码来源**: [moltbot/moltbot](https://github.com/moltbot/moltbot)

---

## 🎯 架构核心理念

Moltbot 的架构设计围绕**单一 WebSocket 控制平面 (Gateway)** 展开，所有组件（渠道、客户端、Agent、节点）都通过 Gateway 进行通信和协调。

**核心设计原则**：
- **本地优先**：运行在用户自己的设备上，数据完全掌控
- **控制平面分离**：Gateway 负责协调，各组件独立运行
- **多渠道统一**：抽象统一的消息接口，支持 10+ 消息平台
- **可扩展性**：通过 Plugin SDK、Skills 系统支持功能扩展

---

## 📐 整体架构

### 系统架构图

```
┌─────────────────── 消息渠道层 ────────────────────┐
│ WhatsApp │ Telegram │ Slack │ Discord │ Signal │
│  iMessage │ BlueBubbles │ Google Chat │ Teams  │
└──────────────────────┬────────────────────────────┘
                       │ 归一化消息
                       ▼
┌────────────────── Gateway (控制平面) ──────────────┐
│ WebSocket Server: ws://127.0.0.1:18789            │
│ ┌─────────────┐  ┌──────────┐  ┌───────────────┐ │
│ │ Session 管理│  │ 路由系统 │  │ 配置管理      │ │
│ └─────────────┘  └──────────┘  └───────────────┘ │
│ ┌─────────────┐  ┌──────────┐  ┌───────────────┐ │
│ │ Channel 管理│  │ Cron 调度│  │ 事件总线      │ │
│ └─────────────┘  └──────────┘  └───────────────┘ │
└──────────────────────┬────────────────────────────┘
                       │ WebSocket/RPC
                       ▼
┌─────────────────── 执行层 ─────────────────────────┐
│                                                    │
│  ┌──────────────┐    ┌─────────────┐            │
│  │ Pi Agent     │    │ Browser     │            │
│  │ (RPC Mode)   │    │ Controller  │            │
│  └──────────────┘    └─────────────┘            │
│                                                    │
│  ┌──────────────┐    ┌─────────────┐            │
│  │ Canvas Host  │    │ Skills      │            │
│  │ (A2UI)       │    │ Runtime     │            │
│  └──────────────┘    └─────────────┘            │
│                                                    │
└────────────────────────────────────────────────────┘
         │                    │
         ▼                    ▼
┌──────────────┐      ┌──────────────┐
│ macOS App    │      │ Mobile Nodes │
│ (Menu Bar)   │      │ iOS/Android  │
└──────────────┘      └──────────────┘
```

---

## 🏗️ 核心模块详解

### 1. Gateway (控制平面)

**位置**: `src/gateway/`

Gateway 是整个系统的神经中枢，负责协调所有组件的通信。

#### 核心职责

根据 `src/gateway/server.impl.ts:1-200`：

1. **启动和初始化**（`startGatewayServer` 函数）：
   - 配置加载和验证
   - 插件自动启用
   - Channel 管理器创建
   - Agent 事件处理器注册
   - 定时任务调度
   - 健康检查系统

2. **WebSocket 服务**：
   - 默认端口：`18789`
   - 支持三种绑定模式（`GatewayServerOptions.bind`）：
     - `loopback`: 127.0.0.1（默认）
     - `lan`: 0.0.0.0
     - `tailnet`: Tailscale 地址
     - `auto`: 自动选择

3. **HTTP 端点**（可选）：
   - `/v1/chat/completions` - OpenAI 兼容接口
   - `/v1/responses` - OpenResponses API
   - Control UI (Web 界面)

#### 关键组件

根据 `src/gateway/server.impl.ts:44-72`：

```typescript
// Gateway 启动时初始化的子系统
- startGatewayDiscovery()         // 服务发现（Bonjour/mDNS）
- createChannelManager()           // 渠道管理
- createAgentEventHandler()        // Agent 事件处理
- buildGatewayCronService()        // Cron 调度
- loadGatewayModelCatalog()        // 模型目录
- NodeRegistry                     // 节点注册表
- createNodeSubscriptionManager()  // 节点订阅管理
- ExecApprovalManager              // 执行权限审批
- startGatewayTailscaleExposure()  // Tailscale 暴露
```

#### 协议定义

**位置**: `src/gateway/protocol/index.ts`

使用 JSON Schema + Ajv 进行协议验证，支持的消息类型包括：

- **Agent 相关**：`AgentEvent`, `AgentSummary`, `AgentsListParams`
- **Chat 相关**：`ChatEvent`, `ChatSendParams`, `ChatAbortParams`, `ChatInjectParams`
- **Config 相关**：`ConfigApplyParams`, `ConfigPatchParams`, `ConfigGetParams`
- **Cron 相关**：`CronAddParams`, `CronListParams`, `CronRunParams`
- **Channels 相关**：`ChannelsStatusParams`, `ChannelsLogoutParams`
- **Node 相关**：`NodesListParams`, `NodesInvokeParams`
- **Session 相关**：`SessionsSendParams`, `SessionsResetParams`

---

### 2. Channels (渠道系统)

**位置**: `src/channels/`

#### 插件架构

根据 `package.json:17-79` 和 `src/channels/plugins/types.core.ts`：

Channels 采用**插件化架构**，每个消息平台都是一个独立的 Channel Plugin。

**支持的渠道**（`dist/` 目录结构）：
- `whatsapp/` - 使用 `@whiskeysockets/baileys` (7.0.0-rc.9)
- `telegram/` - 使用 `grammy` (1.39.3)
- `slack/` - 使用 `@slack/bolt` (4.6.0)
- `discord/` - 使用 `discord-api-types` (0.38.37)
- `signal/` - 使用 signal-cli
- `imessage/` - 使用 imsg
- `line/` - 使用 `@line/bot-sdk` (10.6.0)
- 扩展渠道：BlueBubbles, Matrix, Zalo, Teams 等

#### Channel Plugin 接口

根据 `src/channels/plugins/types.core.ts:1-150`：

```typescript
// Channel 元数据
type ChannelMeta = {
  id: ChannelId;              // 渠道 ID
  label: string;              // 显示名称
  selectionLabel: string;     // 选择器标签
  docsPath: string;           // 文档路径
  blurb: string;              // 简介
  order?: number;             // 排序
  aliases?: string[];         // 别名
};

// Channel 账号状态
type ChannelAccountSnapshot = {
  accountId: string;
  enabled?: boolean;
  configured?: boolean;
  linked?: boolean;
  running?: boolean;
  connected?: boolean;
  lastConnectedAt?: number | null;
  lastMessageAt?: number | null;
  // ... 更多状态字段
};
```

#### 消息归一化

**位置**: `src/channels/plugins/normalize/`

每个渠道都有专门的消息归一化逻辑：
- `whatsapp.ts` - WhatsApp 消息归一化
- `telegram.ts` - Telegram 消息归一化
- `discord.ts` - Discord 消息归一化
- `signal.ts` - Signal 消息归一化
- `slack.ts` - Slack 消息归一化
- 等等

**目的**：将不同平台的消息格式统一为内部标准格式。

#### 出站消息处理

**位置**: `src/channels/plugins/outbound/`

负责将内部消息格式转换为各渠道的原生格式并发送。

---

### 3. Agents (智能体系统)

**位置**: `src/agents/`

#### 核心技术栈

根据 `package.json:166-169`：

```json
"@mariozechner/pi-agent-core": "0.49.3",
"@mariozechner/pi-ai": "0.49.3",
"@mariozechner/pi-coding-agent": "0.49.3",
"@mariozechner/pi-tui": "0.49.3",
```

Moltbot 使用 **Pi Agent** 作为底层 Agent 运行时，支持：
- 工具流式输出 (Tool Streaming)
- 块流式输出 (Block Streaming)
- RPC 模式运行

#### Agent 架构

**关键目录**：
- `pi-embedded-runner/` - Pi Agent 嵌入式运行器 (30 个文件)
- `pi-embedded-helpers/` - Pi Agent 辅助工具
- `sandbox/` - 沙箱执行环境 (17 个文件)
- `tools/` - Agent 工具定义 (57 个文件)
- `skills/` - Skills 系统 (11 个文件)

#### 工具系统

根据 `src/agents/moltbot-tools.ts` 和 `src/channels/plugins/types.core.ts:15-17`：

```typescript
type ChannelAgentTool = AgentTool<TSchema, unknown>;
type ChannelAgentToolFactory = (params: { cfg?: MoltbotConfig }) => ChannelAgentTool[];
```

每个 Channel 可以提供自己的 Agent Tools，实现特定平台的功能。

#### 模型支持

根据 `src/agents/model-compat.ts` 和相关文件：

**支持的模型提供商**：
- **Anthropic**: Claude (Pro/Max 订阅 + Opus 4.5)
- **OpenAI**: GPT 系列
- **Google**: Gemini 系列
- **AWS Bedrock**
- **本地模型**: 通过 `node-llama-cpp` (可选依赖)

**模型认证系统**（`src/agents/auth-profiles/`）：
- OAuth 认证流程
- API Key 管理
- 认证配置文件（Auth Profile）管理
- 模型 Failover 支持

---

### 4. Session (会话管理)

**位置**: `src/sessions/`, `src/routing/`

#### Session 模型

根据 README.md:138：

> Session model: `main` for direct chats, group isolation, activation modes, queue modes, reply-back.

**会话类型**：
- **`main` Session**: 用于直接对话
- **Group Session**: 群组会话隔离
- **Per-Agent Session**: 每个 Agent 独立的会话

#### 路由系统

**位置**: `src/routing/`

根据文件列表：
- `bindings.ts` - 路由绑定
- `resolve-route.ts` - 路由解析
- `session-key.ts` - 会话键管理

**路由功能**：将不同的渠道/账号/对话路由到隔离的 Agents。

---

### 5. Browser (浏览器控制)

**位置**: `src/browser/` (81 个文件)

#### 技术实现

根据 `package.json:199`：

```json
"playwright-core": "1.58.0",
```

使用 **Playwright** 实现浏览器自动化控制。

#### 功能特性（README.md:152）

- 专用的 moltbot Chrome/Chromium
- 页面快照 (Snapshots)
- 页面操作 (Actions)
- 文件上传
- 浏览器配置文件 (Profiles)

---

### 6. Canvas Host (可视化工作空间)

**位置**: `src/canvas-host/`

#### A2UI 技术

根据 README.md:123：

> Live Canvas — agent-driven visual workspace with A2UI.

**A2UI** (Agent-to-UI)：Agent 驱动的 UI 渲染系统，Agent 可以直接控制界面显示。

#### 构建流程

根据 `package.json:146`：

```bash
canvas:a2ui:bundle - bash scripts/bundle-a2ui.sh
```

Canvas 有独立的构建流程，打包为可嵌入的组件。

---

### 7. Skills (技能系统)

**位置**: `skills/` (根目录), `src/agents/skills/`

#### Skills 类型

根据 README.md:126：

- **Bundled Skills**: 内置技能
- **Managed Skills**: 托管技能
- **Workspace Skills**: 工作空间特定技能

#### 可用 Skills（`skills/` 目录）

从目录结构可以看到 40+ 个 Skill：
- `1password/` - 1Password 集成
- `apple-notes/` - Apple Notes
- `apple-reminders/` - Apple Reminders
- `discord/` - Discord 操作
- `github/` - GitHub 集成
- `notion/` - Notion 集成
- `obsidian/` - Obsidian 集成
- `slack/` - Slack 操作
- `spotify-player/` - Spotify 播放器
- `tmux/` - Tmux 管理
- `weather/` - 天气查询
- `voice-call/` - 语音通话
- ... 等等

每个 Skill 包含 `SKILL.md` 描述文件。

---

### 8. Control UI (Web 控制界面)

**位置**: `ui/` (124 个文件)

#### 技术栈

根据 `package.json` devDependencies：

```json
"lit": "^3.3.2",                    // Web Components
"@lit-labs/signals": "^0.2.0",
"@lit/context": "^1.1.6",
"@mariozechner/mini-lit": "0.2.1",
"lucide": "^0.563.0",               // 图标库
```

使用 **Lit** (Web Components) 构建轻量级的 Web UI。

#### 构建流程

```bash
ui:install - 安装 UI 依赖
ui:dev - 开发模式
ui:build - 生产构建
```

---

### 9. Mobile Apps (移动端)

#### iOS App

**位置**: `apps/ios/` (412 个 Swift 文件)

- **语言**: Swift
- **构建工具**: XcodeGen
- **功能**: 
  - Canvas 显示
  - Voice Wake
  - Talk Mode
  - 摄像头
  - 屏幕录制
  - Bonjour 配对

#### Android App

**位置**: `apps/android/` (63 个 Kotlin 文件)

- **语言**: Kotlin
- **构建工具**: Gradle
- **功能**:
  - Canvas 显示
  - Talk Mode
  - 摄像头
  - 屏幕录制
  - 可选 SMS

#### macOS App

**位置**: `apps/macos/`, `src/macos/`

- **语言**: Swift
- **功能**:
  - 菜单栏应用
  - Voice Wake/PTT (Push-to-Talk)
  - Talk Mode 覆层
  - WebChat
  - 调试工具
  - 远程 Gateway 控制

---

### 10. Plugin SDK

**位置**: `src/plugin-sdk/`, `src/plugins/`

#### 导出接口

根据 `package.json:9-10`：

```json
"./plugin-sdk": "./dist/plugin-sdk/index.js",
"./plugin-sdk/*": "./dist/plugin-sdk/*",
```

提供标准的插件开发接口，允许第三方扩展功能。

---

## 🔄 数据流分析

### 典型消息流程

```
1. 用户在 WhatsApp 发送消息
   ↓
2. WhatsApp Channel Plugin (src/whatsapp/) 接收
   ↓
3. 消息归一化 (src/channels/plugins/normalize/whatsapp.ts)
   ↓
4. Gateway 路由 (src/routing/resolve-route.ts)
   ↓
5. Session 分配 (src/sessions/)
   ↓
6. Agent 处理 (src/agents/pi-embedded-runner/)
   ↓
7. 工具调用 (src/agents/tools/)
   ↓
8. 响应生成
   ↓
9. 消息格式化 (src/channels/plugins/outbound/whatsapp.ts)
   ↓
10. 通过 WhatsApp 发送回复
```

---

## 🔐 安全模型

### DM 访问控制

根据 README.md:110-115：

**默认策略**：`dmPolicy="pairing"`

- 未知发送者需要配对码
- 使用 `moltbot pairing approve <channel> <code>` 批准
- 公开 DM 需要显式设置 `dmPolicy="open"` + `"*"` 在 allowlist

### Sandbox 执行

**位置**: `src/agents/sandbox/` (17 个文件)

提供隔离的执行环境，防止恶意代码执行。

### 执行权限审批

**位置**: `src/gateway/exec-approval-manager.ts`

Agent 执行敏感操作时需要用户审批。

---

## 📦 构建和部署

### 构建流程

根据 `package.json:89`：

```bash
build:
  1. pnpm canvas:a2ui:bundle        # 打包 Canvas A2UI
  2. tsc -p tsconfig.json            # TypeScript 编译
  3. scripts/canvas-a2ui-copy.ts     # 复制 Canvas 资源
  4. scripts/copy-hook-metadata.ts   # 复制 Hook 元数据
  5. scripts/write-build-info.ts     # 写入构建信息
```

### 平台构建

**iOS**:
```bash
ios:gen - XcodeGen 生成项目
ios:build - Xcodebuild 构建
ios:run - 运行在模拟器
```

**Android**:
```bash
android:assemble - Gradle 构建 APK
android:install - 安装到设备
android:run - 启动应用
```

**macOS**:
```bash
mac:package - 打包 .app
mac:restart - 重启应用
```

---

## 🧪 测试体系

### 测试类型

根据 `package.json:122-142`：

1. **单元测试**: `vitest` (覆盖率要求 70%)
2. **E2E 测试**: `vitest.e2e.config.ts`
3. **Live 测试**: `vitest.live.config.ts` (真实模型调用)
4. **Docker 测试**: 多个 Docker E2E 场景
5. **UI 测试**: `ui/test/`

### 测试覆盖率配置

```json
"thresholds": {
  "lines": 70,
  "functions": 70,
  "branches": 70,
  "statements": 70
}
```

---

## 🔧 配置系统

**位置**: `src/config/` (130 个文件)

### 配置文件

根据 `src/config/config.ts`:

- **主配置**: `CONFIG_PATH` (可通过环境变量覆盖)
- **格式**: JSON5
- **验证**: 使用 JSON Schema
- **迁移**: 自动 Legacy Config 迁移

### 配置热重载

**位置**: `src/gateway/config-reload.ts`

Gateway 支持配置热重载，无需重启服务。

---

## 📊 监控和日志

### 日志系统

**位置**: `src/logging/` (14 个文件)

根据 `package.json:205`：

```json
"tslog": "^4.10.2",
```

使用 `tslog` 提供结构化日志。

### 子系统日志

根据 `src/gateway/server.impl.ts:78-89`：

```typescript
const log = createSubsystemLogger("gateway");
const logCanvas = log.child("canvas");
const logDiscovery = log.child("discovery");
const logTailscale = log.child("tailscale");
const logChannels = log.child("channels");
const logBrowser = log.child("browser");
const logHealth = log.child("health");
const logCron = log.child("cron");
const logReload = log.child("reload");
const logHooks = log.child("hooks");
const logPlugins = log.child("plugins");
const logWsControl = log.child("ws");
```

每个子系统都有独立的日志通道。

### 健康检查

**位置**: `src/gateway/server/health-state.ts`

提供 Gateway 健康状态监控。

---

## 🌐 网络暴露

### Tailscale 集成

**位置**: `src/gateway/server-tailscale.ts`

支持两种模式：
- **Serve**: Tailnet 内部访问
- **Funnel**: 公网访问

### SSH Tunnels

支持通过 SSH 隧道远程访问 Gateway。

---

## 📚 依赖关系图

### 核心依赖

```
moltbot
├── @mariozechner/pi-agent-core (Agent 运行时)
├── @whiskeysockets/baileys (WhatsApp)
├── grammy (Telegram)
├── @slack/bolt (Slack)
├── discord-api-types (Discord)
├── playwright-core (浏览器控制)
├── ws (WebSocket)
├── express (HTTP 服务)
├── hono (轻量 HTTP 框架)
└── sharp (图像处理)
```

### 平台特定

```
iOS/macOS
└── Swift (原生应用)

Android
└── Kotlin (原生应用)

Web UI
└── Lit (Web Components)
```

---

## 🎨 设计模式

### 1. 插件化架构

所有 Channel 都是插件，遵循统一接口：
- `ChannelMeta` - 元数据
- `normalize` - 消息归一化
- `outbound` - 出站处理
- `onboarding` - 配置引导

### 2. 事件驱动

Gateway 作为事件总线，组件通过事件通信：
- `AgentEvent` - Agent 事件
- `ChatEvent` - 聊天事件
- `GATEWAY_EVENTS` - Gateway 事件列表

### 3. RPC 模式

Agent 通过 RPC 与 Gateway 通信，支持：
- 同步调用
- 流式响应
- 工具回调

### 4. 分层架构

```
表现层 (UI/CLI)
    ↓
控制层 (Gateway)
    ↓
业务层 (Agents/Channels)
    ↓
基础设施层 (Infra/Logging/Config)
```

---

## 🔍 关键技术决策

### 1. 为什么选择 WebSocket?

- **实时性**: 双向通信，低延迟
- **统一接口**: 所有客户端通过同一协议连接
- **推送能力**: 服务端主动推送事件

### 2. 为什么采用插件化 Channels?

- **可扩展性**: 易于添加新的消息平台
- **隔离性**: 各渠道独立，互不影响
- **统一抽象**: 简化上层 Agent 逻辑

### 3. 为什么使用 Pi Agent?

- **成熟**: 经过验证的 Agent 运行时
- **流式支持**: 原生支持流式输出
- **工具系统**: 完善的工具调用机制

### 4. 为什么本地优先?

- **隐私**: 数据不离开用户设备
- **控制**: 用户完全掌控
- **成本**: 无需云服务费用

---

## 📖 相关资源

- **官方文档**: https://docs.molt.bot
- **GitHub**: https://github.com/moltbot/moltbot
- **Discord 社区**: https://discord.gg/clawd

---

**下一步研究方向**：
- [ ] 深入研究 Pi Agent 的工具流式机制
- [ ] 分析 A2UI 的具体实现
- [ ] 理解 Session 路由的详细逻辑
- [ ] 研究 Sandbox 沙箱的安全机制
- [ ] 探索 Skills 系统的加载和执行流程
