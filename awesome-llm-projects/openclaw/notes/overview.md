# OpenClaw 项目架构深度解析

## 版本信息

- **Git Repository**: https://github.com/openclaw/openclaw
- **Commit Hash**: `6af205a13adfec1fd1eb27d2f3d3546b0b4e8f86`
- **Version**: `2026.1.29` (来自 `package.json:3`)
- **Latest Commit**: Peter Steinberger, 2026-01-30 07:26:07 +0000
- **研究日期**: 2026-01-30
- **GitHub Stars**: 109k (截至 2026 年 1 月)
- **注意**: 该分析基于 2026 年 1 月底的代码，项目处于活跃开发状态

## 一、架构理念与核心定位

### 1.1 项目定位

根据 `README.md:21-22`：

> **OpenClaw** is a *personal AI assistant* you run on your own devices.

**关键词解读**：
- **Personal**: 单用户场景，不是多租户 SaaS
- **AI assistant**: Agent 作为核心执行单元，支持工具调用
- **Your own devices**: Local-First，数据隐私和离线可用

### 1.2 核心架构理念

根据 `docs/concepts/architecture.md:12-19`：

```
- 单一长生命周期 Gateway 拥有所有消息表面 (WhatsApp/Telegram/Slack 等)
- 控制平面客户端通过 WebSocket 连接到 Gateway (默认 127.0.0.1:18789)
- Nodes (macOS/iOS/Android) 也通过 WebSocket 连接，声明 role: node
- One Gateway per host: 唯一开启 WhatsApp session 的地方
- Canvas host (默认 18793) 提供 agent 可编辑的 HTML 和 A2UI
```

**三大架构支柱**：

#### 1) Local-First Gateway

**设计理念** (来自 `README.md:120`):
> **Local-first Gateway** — single control plane for sessions, channels, tools, and events.

**架构优势**：
- **数据隐私**: 消息和会话数据不离开本地设备
- **低延迟**: 无需云端往返，响应速度快
- **离线可用**: 基础功能不依赖网络
- **成本控制**: 无需云端基础设施

**技术实现** (来自 `docs/gateway/index.md:11`):
- Gateway 是 always-on 进程，拥有单一 Baileys/Telegram 连接
- 使用 launchd/systemd 守护进程保持运行
- 配置热重载 (hybrid mode)：安全配置立即生效，关键配置触发进程内重启 (SIGUSR1)

#### 2) WebSocket 统一协议

**设计理念** (来自 `docs/concepts/architecture.md:68-79`):

```
Wire protocol:
- Transport: WebSocket, text frames with JSON payloads
- First frame MUST be `connect`
- After handshake:
  - Requests: {type:"req", id, method, params} → {type:"res", id, ok, payload|error}
  - Events: {type:"event", event, payload, seq?, stateVersion?}
```

**架构优势**：
- **双向通信**: Server 可主动推送事件 (presence, agent streaming, health 等)
- **统一接口**: 所有客户端 (macOS app, CLI, WebChat, nodes) 使用相同协议
- **类型安全**: TypeBox schemas 定义协议，生成 JSON Schema 和 Swift 类型
- **Idempotency**: 支持幂等 key，安全重试 side-effecting 操作 (send, agent)

**协议设计细节** (来自 `src/gateway/server.impl.ts:92-94`):

```typescript
export type GatewayServer = {
  close: (opts?: { reason?: string; restartExpectedMs?: number | null }) => Promise<void>;
};
```

Gateway 启动后维护 WebSocket server，客户端连接必须：
1. 首帧发送 `connect` 请求 (包含 client identity, auth, caps)
2. Gateway 返回 `hello-ok` (包含 presence, health snapshot, policy)
3. 之后可发送 RPC 请求 (health, status, send, agent, node.invoke 等) 和订阅事件

#### 3) One Gateway Per Host

**设计理念** (来自 `docs/concepts/architecture.md:19`):
> One Gateway per host; it is the only place that opens a WhatsApp session.

**架构约束**：
- **Baileys session 唯一性**: WhatsApp Web 协议限制同一账号只能有一个活跃连接
- **状态一致性**: 所有消息通过单一 Gateway 路由，避免状态分裂
- **资源效率**: 避免多个进程竞争相同资源 (端口、session 文件)

**多 Gateway 支持** (来自 `docs/gateway/index.md:52-68`):
- 通常不需要：一个 Gateway 可服务多个 channel 和 agent
- 支持多 Gateway：必须隔离 state/config/port (用于 redundancy 或 rescue bot)
- Profile-aware 服务名: macOS `bot.molt.<profile>`, Linux `openclaw-gateway-<profile>.service`

### 1.3 设计哲学对比

| 架构决策 | OpenClaw 选择 | 替代方案 | 理由 |
|---------|--------------|---------|------|
| 部署模式 | Local-First | Cloud SaaS | 数据隐私、低延迟、离线可用 |
| 协议 | WebSocket | HTTP/REST | 双向通信、事件推送、流式响应 |
| Gateway 数量 | One per host | Multiple gateways | Baileys session 唯一性、状态一致性 |
| Agent Runtime | Embedded (Pi) | Microservice | 低延迟、简化部署、统一生命周期 |
| Channel 集成 | Plugin system | Monolithic | 可扩展性、community contributions |

## 二、架构分层与职责划分

### 2.1 四层架构模型

OpenClaw 采用清晰的四层架构，每层职责明确，松耦合设计：

```
┌─────────────────────────────────────────────────────────────────┐
│ Layer 4: Client Layer (操作平面)                                │
│  - macOS app (Swift): 菜单栏控制、Voice Wake、WebChat          │
│  - CLI (TypeScript): 命令行工具 (gateway, agent, send 等)      │
│  - WebChat UI (Lit): 浏览器聊天界面                            │
│  - Nodes (Swift/Kotlin): iOS/Android 设备 node                 │
│  Protocol: WebSocket RPC over ws://127.0.0.1:18789            │
└─────────────────────────────────────────────────────────────────┘
                            ↕ (WebSocket)
┌─────────────────────────────────────────────────────────────────┐
│ Layer 3: Control Plane (控制平面) - Gateway                     │
│  - WebSocket server (ws): 统一 RPC 协议                        │
│  - Session management: 多会话隔离、持久化、compaction          │
│  - Config management: 配置管理、热重载                         │
│  - Service discovery: Bonjour/mDNS, Tailscale                 │
│  - Cron service: 定时任务调度                                  │
│  - Health & Presence: 状态管理和事件推送                       │
│  Location: src/gateway/ (187 files)                           │
└─────────────────────────────────────────────────────────────────┘
                    ↕ (RPC call)        ↕ (inbound msgs)
┌─────────────────────────────────────┬───────────────────────────┐
│ Layer 2: Execution Plane (执行平面) │ Layer 1: Data Plane       │
│  Agent Runtime (Pi-embedded)        │  (数据平面) - Channels    │
│  - Pi agent core (RPC mode)         │  - WhatsApp (Baileys)     │
│  - Tool execution & streaming       │  - Telegram (grammY)      │
│  - Sandbox isolation (Docker)       │  - Slack (Bolt)           │
│  - Model failover                   │  - Discord (discord.js)   │
│  - Skills loading                   │  - Signal (signal-cli)    │
│  - Memory system (sqlite-vec)       │  - iMessage (imsg)        │
│  Location: src/agents/ (436 files)  │  - 10+ other channels     │
│                                      │  Location: src/telegram/, │
│                                      │    src/slack/, etc.       │
└─────────────────────────────────────┴───────────────────────────┘
```

### 2.2 各层职责详解

#### Layer 4: Client Layer (操作平面)

**职责**: 提供用户交互界面，连接到 Gateway 控制平面

**组件**:
1. **macOS app** (`apps/macos/`, 413 个 Swift 文件)
   - 菜单栏控制：启动/停止 Gateway、查看状态
   - Voice Wake: 语音唤醒 Agent
   - WebChat 内嵌：本地聊天界面
   - Node mode: 可作为 node 暴露 system.run、camera 等能力

2. **CLI** (`src/cli/` 157 files + `src/commands/` 223 files)
   - 命令行工具：`openclaw gateway`, `openclaw agent`, `openclaw message send`
   - Onboarding wizard: 引导式配置
   - Doctor command: 配置诊断和修复建议

3. **WebChat UI** (`ui/`, 128 files: Lit components)
   - 浏览器聊天界面
   - 通过 Gateway WebSocket API 获取历史、发送消息
   - 支持远程访问 (SSH tunnel / Tailscale)

4. **Nodes** (`apps/ios/`, `apps/android/`)
   - iOS node (Swift): Canvas, Voice Wake, Talk Mode, camera, screen recording
   - Android node (Kotlin): Canvas, Talk Mode, camera, screen recording, optional SMS
   - 通过 `node.invoke` 执行设备本地命令

**通信方式** (来自 `docs/concepts/architecture.md:49-66`):
```
Client                    Gateway
  |                          |
  |---- req:connect -------->|
  |<------ res (ok) ---------|   (payload=hello-ok with snapshot)
  |<------ event:presence ---|
  |<------ event:tick -------|
  |------- req:agent ------->|
  |<------ res:agent --------|   (ack: {runId,status:"accepted"})
  |<------ event:agent ------|   (streaming tool/assistant deltas)
  |<------ res:agent --------|   (final: {runId,status,summary})
```

#### Layer 3: Control Plane (控制平面) - Gateway

**职责**: 统一控制平面，管理所有客户端、channels、sessions、cron、health

**核心模块** (来自 `src/gateway/`, 187 files):

1. **WebSocket Server** (`server.impl.ts`, `server-ws-runtime.ts`)
   - 监听 `ws://127.0.0.1:18789`
   - 验证每个 inbound frame (AJV + JSON Schema)
   - 路由 RPC 请求到对应 handler
   - 广播 events 到所有连接的客户端

2. **Session Management** (`session-utils.ts`, `server-session-key.ts`)
   - Session 存储: `~/.openclaw/sessions/<sessionKey>/`
   - Session 隔离: main session vs group sessions
   - Session 配置: thinking level, verbose level, model, send policy
   - Session compaction: 会话历史压缩和总结

3. **Config Management** (`config-reload.ts`, `server-runtime-config.ts`)
   - 配置文件: `~/.openclaw/openclaw.json`
   - 热重载: hybrid mode (safe changes hot-apply, critical changes restart)
   - Config patching: 动态更新配置 (via `config.patch` RPC)

4. **Service Discovery** (`server-discovery-runtime.ts`)
   - Bonjour/mDNS: 局域网自动发现 Gateway
   - Tailscale Serve/Funnel: 安全远程访问
   - SSH tunnels: 备选远程访问方案

5. **Cron Service** (`server-cron.ts`)
   - Cron job 调度: `croner` 库
   - Wakeup 机制: 定时唤醒 Agent
   - Cron config: `cron` 配置项

6. **Health & Presence** (`server/health-state.ts`, `server-broadcast.ts`)
   - Health snapshot: Gateway 健康状态 (channels, models, sessions)
   - Presence tracking: 连接的客户端、devices、channels 状态
   - Event broadcasting: 推送 presence、tick、shutdown events 到所有客户端

**Gateway 运行时状态** (来自 `src/gateway/server-runtime-state.ts:45-72`):

```typescript
{
  canvasHost: CanvasHostHandler | null,
  httpServer: HttpServer,
  wss: WebSocketServer,
  clients: Set<GatewayWsClient>,
  broadcast: (event, payload, opts) => void,
  agentRunSeq: Map<string, number>,  // 每个 session 的 agent run 序列号
  dedupe: Map<string, DedupeEntry>,  // 幂等 dedupe cache
  chatRunState: ReturnType<typeof createChatRunState>,
  chatRunBuffers: Map<string, string>,  // Chat 消息缓冲
  chatAbortControllers: Map<string, ChatAbortControllerEntry>,
}
```

#### Layer 2: Execution Plane (执行平面) - Agent Runtime

**职责**: Agent 核心执行引擎，工具调用，流式响应

**核心模块** (来自 `src/agents/`, 436 files):

1. **Pi-embedded Runtime** (`pi-embedded-runner/`, `pi-embedded-subscribe.ts`)
   - 基于 `@mariozechner/pi-agent-core` 的 RPC 模式
   - Agent loop: intake → context assembly → model inference → tool execution → streaming reply
   - 流式处理: tool streaming, block streaming, assistant deltas
   - Lifecycle events: start, end, error

2. **Tool System** (`tools/`, 57+ files; `openclaw-tools.ts`)
   - 内置工具: bash, read, write, edit, browser, canvas, nodes, cron, sessions_*, image, web
   - Tool 注册: `createOpenClawTools` 统一注册入口
   - Tool policy: sandbox allowlist/denylist
   - Plugin tools: 动态加载第三方工具

3. **Sandbox Isolation** (`sandbox/`, 17+ files)
   - Docker 沙箱: 非 main session 运行在 Docker 容器中
   - Allowlist: bash, process, read, write, edit, sessions_*
   - Denylist: browser, canvas, nodes, cron, discord, gateway
   - 安全隔离: 防止 group/channel 消息执行危险操作

4. **Model Failover** (`model-compat.ts`, `failover-error.ts`)
   - 多模型支持: Claude, GPT, Ollama, Bedrock
   - 自动切换: model failover on error
   - Profile rotation: OAuth vs API keys

5. **Skills System** (`skills/`, `agents/skills/`)
   - Bundled skills: 58 个内置 skills (`skills/` 目录)
   - Managed skills: ClawdHub registry 动态拉取
   - Workspace skills: `~/.openclaw/workspace/skills/`
   - Skills injection: 注入到 system prompt 和环境变量

6. **Memory System** (`src/memory/`, 33 files)
   - Long-term memory: sqlite-vec (vector database)
   - Memory retrieval: 语义搜索和上下文增强

**Agent Loop 流程** (来自 `docs/concepts/agent-loop.md:20-39`):

```
1) agent RPC 验证参数 → 返回 {runId, acceptedAt}
2) agentCommand 执行 agent:
   - 解析 model + thinking/verbose defaults
   - 加载 skills snapshot
   - 调用 runEmbeddedPiAgent (pi-agent-core runtime)
   - 发出 lifecycle end/error event
3) runEmbeddedPiAgent:
   - 序列化 runs (per-session + global queue)
   - 解析 model + auth profile，构建 pi session
   - 订阅 pi events，流式传输 assistant/tool deltas
   - 强制 timeout (abort run if exceeded)
4) subscribeEmbeddedPiSession 桥接 pi-agent-core events:
   - tool events => stream: "tool"
   - assistant deltas => stream: "assistant"
   - lifecycle events => stream: "lifecycle" (phase: start/end/error)
5) agent.wait 等待 lifecycle end/error → 返回 {status, startedAt, endedAt}
```

#### Layer 1: Data Plane (数据平面) - Channels

**职责**: 统一消息接口，适配不同消息平台

**Channel 抽象** (来自 `src/channels/plugins/types.plugin.ts:48-84`):

```typescript
export type ChannelPlugin<ResolvedAccount = any> = {
  id: ChannelId;
  meta: ChannelMeta;
  capabilities: ChannelCapabilities;
  config: ChannelConfigAdapter<ResolvedAccount>;
  setup?: ChannelSetupAdapter;
  pairing?: ChannelPairingAdapter;
  security?: ChannelSecurityAdapter<ResolvedAccount>;
  groups?: ChannelGroupAdapter;
  mentions?: ChannelMentionAdapter;
  outbound?: ChannelOutboundAdapter;
  status?: ChannelStatusAdapter<ResolvedAccount>;
  gateway?: ChannelGatewayAdapter<ResolvedAccount>;
  auth?: ChannelAuthAdapter;
  commands?: ChannelCommandAdapter;
  streaming?: ChannelStreamingAdapter;
  threading?: ChannelThreadingAdapter;
  messaging?: ChannelMessagingAdapter;
  actions?: ChannelMessageActionAdapter;
  heartbeat?: ChannelHeartbeatAdapter;
  agentTools?: ChannelAgentToolFactory | ChannelAgentTool[];
};
```

**各平台实现**:
- `src/telegram/` (85 files): grammY bot framework
- `src/slack/` (65 files): Slack Bolt SDK
- `src/discord/` (61 files): discord.js wrapper (@buape/carbon)
- `src/signal/` (24 files): signal-cli
- `src/imessage/` (16 files): imsg (macOS only)
- `src/whatsapp/` (2 files): Baileys (lightweight wrapper)
- `src/line/` (34 files): LINE bot SDK

**Channel 能力矩阵**:
| Channel | Groups | Mentions | Typing | Media | Reactions | Threads |
|---------|--------|----------|--------|-------|-----------|---------|
| Telegram | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| Slack | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Discord | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| WhatsApp | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| Signal | ✅ | ❌ | ❌ | ✅ | ❌ | ❌ |
| iMessage | ✅ | ❌ | ❌ | ✅ | ❌ | ❌ |

### 2.3 代码规模统计

| 层级 | 模块 | 文件数 | 主要语言 | 说明 |
|------|------|--------|----------|------|
| Layer 4 | macOS/iOS app | 413 | Swift | 客户端应用 |
| Layer 4 | Android app | 63 | Kotlin | Android node |
| Layer 4 | CLI + Commands | 380 | TypeScript | 命令行工具 |
| Layer 4 | WebChat UI | 128 | TypeScript (Lit) | 浏览器界面 |
| Layer 3 | Gateway | 187 | TypeScript | 控制平面 |
| Layer 2 | Agent Runtime | 436 | TypeScript | 执行平面 |
| Layer 1 | Channels | 300+ | TypeScript | 数据平面 (10+ channels) |
| - | Config | 130 | TypeScript | 配置管理 |
| - | Infra | 183 | TypeScript | 基础设施 |
| - | Media | 20 | TypeScript | 媒体 pipeline |
| - | Memory | 33 | TypeScript | 记忆系统 |
| - | Plugins | 37 | TypeScript | 插件生态 |
| - | Extensions | 547 | TypeScript | 20+ 扩展插件 |
| - | Docs | 296 | Markdown | 完善的文档 |
| - | Skills | 58 | Markdown/Python | 内置 skills |
| **总计** | - | **2,502 TS + 413 Swift + 63 Kotlin** | - | **3,000+ files** |

**测试覆盖率**: 70% (lines, functions, branches, statements) - 来自 `package.json:262-267`

## 三、核心抽象与接口设计

### 3.1 Channel Plugin 接口 (Plugin System 的精髓)

**设计理念**: 通过统一的 `ChannelPlugin` 接口，将 10+ 个消息平台抽象为可插拔的模块

**核心接口** (来自 `src/channels/plugins/types.plugin.ts:48-84`):

```typescript
export type ChannelPlugin<ResolvedAccount = any> = {
  id: ChannelId;                  // 唯一标识: 'telegram', 'slack', 'discord' 等
  meta: ChannelMeta;              // 显示名称、描述、图标
  capabilities: ChannelCapabilities;  // 能力声明: groups, mentions, typing, media 等
  
  // 核心适配器 (必需)
  config: ChannelConfigAdapter<ResolvedAccount>;  // 配置加载和验证
  
  // 可选适配器 (按需实现)
  setup?: ChannelSetupAdapter;         // CLI setup 流程
  pairing?: ChannelPairingAdapter;     // Pairing 机制 (DM 安全)
  security?: ChannelSecurityAdapter;   // Allowlist, DM policy
  groups?: ChannelGroupAdapter;        // Group 路由规则
  mentions?: ChannelMentionAdapter;    // Mention 解析
  outbound?: ChannelOutboundAdapter;   // 出站消息处理
  status?: ChannelStatusAdapter;       // 状态检查
  gateway?: ChannelGatewayAdapter;     // Gateway lifecycle hooks
  streaming?: ChannelStreamingAdapter; // 流式消息分块
  threading?: ChannelThreadingAdapter; // Thread 支持
  messaging?: ChannelMessagingAdapter; // 消息发送/接收
  actions?: ChannelMessageActionAdapter; // Platform-specific actions
  agentTools?: ChannelAgentToolFactory | ChannelAgentTool[];  // Channel-owned tools
};
```

**设计亮点**:
1. **可选适配器**: 平台只需实现所需能力，避免 boilerplate
2. **类型安全**: 泛型 `<ResolvedAccount>` 保证配置类型安全
3. **能力声明**: `capabilities` 明确声明平台能力，Gateway 据此适配
4. **扩展性**: 新平台只需实现 `ChannelPlugin` 接口，无需修改 Gateway 核心代码

**实际应用示例**: Telegram Plugin 结构 (来自 `src/telegram/` 目录):
```
src/telegram/
  ├── telegram.plugin.ts        # ChannelPlugin 实现入口
  ├── telegram.gateway.ts       # ChannelGatewayAdapter: Gateway 启动/停止 Telegram bot
  ├── telegram.outbound.ts      # ChannelOutboundAdapter: 发送消息到 Telegram
  ├── telegram.security.ts      # ChannelSecurityAdapter: Allowlist 检查
  ├── telegram.groups.ts        # ChannelGroupAdapter: Group mention gating
  ├── telegram.streaming.ts     # ChannelStreamingAdapter: 长消息分块
  └── telegram.actions.ts       # Channel-specific actions (inline keyboards)
```

### 3.2 Session 模型 (多会话隔离)

**设计理念**: 通过 session 模型实现多用户、多 channel、多场景的隔离和协作

**Session 类型** (来自 `docs/concepts/sessions.md` 和代码分析):

1. **Main Session**:
   - 用户与 Agent 的直接对话 (DM)
   - 完整权限：所有 tools (包括 browser, canvas, nodes, exec)
   - Session key 格式: `<channel>:<userId>` (例如: `telegram:123456`)

2. **Group Sessions**:
   - Group 内的对话 (多人参与)
   - 受限权限：默认运行在 Docker sandbox 中
   - Session key 格式: `<channel>:<groupId>` (例如: `slack:C01234567`)
   - Activation mode: mention gating (需要 @bot) 或 always

3. **Agent-Specific Sessions**:
   - 不同 agent 的隔离 session (multi-agent routing)
   - 独立 workspace 和 skills
   - Session key 包含 agent ID

**Session 配置** (来自 `src/sessions/` 目录):
```typescript
{
  thinkingLevel: 'off' | 'minimal' | 'low' | 'medium' | 'high' | 'xhigh',
  verboseLevel: 'on' | 'off',
  model: 'anthropic/claude-opus-4-5' | 'openai/gpt-4' | ...,
  sendPolicy: 'always' | 'auto' | 'never',
  groupActivation: 'mention' | 'always',
  elevated: boolean,  // Per-session elevated bash toggle
}
```

**Session 存储** (来自 `src/gateway/session-utils.ts`):
```
~/.openclaw/sessions/
  ├── telegram:123456/
  │   ├── transcript.jsonl         # 对话历史 (JSONL 格式)
  │   ├── session.json             # Session 元数据
  │   └── compaction-summary.txt   # Compaction 总结
  ├── slack:C01234567/
  │   └── ...
  └── discord:987654321/
      └── ...
```

**Session 生命周期**:
```
创建 (create) → 激活 (activate) → 持久化 (persist) → 压缩 (compact) → 销毁 (destroy)
```

**Agent-to-Agent 通信** (来自 `docs/concepts/agent-loop.md` 和 `src/agents/tools/sessions-*.ts`):
- `sessions_list`: 发现活跃 sessions 和元数据
- `sessions_history`: 获取 session transcript logs
- `sessions_send`: 向另一个 session 发送消息 (可选 reply-back ping-pong)
- `sessions_spawn`: 创建新 session (例如 sub-task delegation)

### 3.3 Tool System (能力扩展)

**设计理念**: 通过统一的 Tool 接口，扩展 Agent 能力 (browser, canvas, nodes, exec 等)

**Tool 注册** (来自 `src/agents/openclaw-tools.ts:150+`):

```typescript
export function createOpenClawTools(opts: CreateOpenClawToolsOptions): AnyAgentTool[] {
  const tools: AnyAgentTool[] = [];
  
  // Core tools (always available)
  tools.push(...coreBashTools);        // bash execution
  tools.push(...coreFileTools);        // read, write, edit
  tools.push(...coreProcessTools);     // process management
  
  // Conditional tools (based on config/session)
  if (browserEnabled) tools.push(browserTool);
  if (canvasEnabled) tools.push(...canvasTools);
  if (nodesEnabled) tools.push(...nodesTools);
  if (cronEnabled) tools.push(...cronTools);
  if (sessionsEnabled) tools.push(...sessionsTools);
  
  // Plugin tools (dynamic)
  tools.push(...pluginTools);
  
  // Channel tools (Discord/Slack actions)
  tools.push(...channelActionTools);
  
  return tools;
}
```

**Tool Policy** (来自 `src/agents/sandbox/` 和配置):
- **Main session**: 所有 tools 可用 (信任用户)
- **Non-main session** (groups/channels): 
  - Allowlist: bash, process, read, write, edit, sessions_*
  - Denylist: browser, canvas, nodes, cron, discord, gateway
  - 运行在 Docker sandbox 中隔离

**Tool 类型分类**:
1. **Exec Tools**: bash, process (命令执行)
2. **File Tools**: read, write, edit (文件操作)
3. **Browser Tools**: browser.navigate, browser.snapshot, browser.action (浏览器控制)
4. **Canvas Tools**: canvas.push, canvas.reset, canvas.eval (Canvas 操作)
5. **Node Tools**: node.invoke (设备本地命令: camera, screen, location)
6. **Session Tools**: sessions_list, sessions_history, sessions_send (Agent 协作)
7. **Cron Tools**: cron.create, cron.list, cron.delete (定时任务)
8. **Channel Tools**: discord.send_message, slack.post (平台特定操作)

### 3.4 Gateway 运行时状态管理

**核心状态** (来自 `src/gateway/server-runtime-state.ts`):

```typescript
type GatewayRuntimeState = {
  // WebSocket 层
  wss: WebSocketServer,              // WebSocket server
  clients: Set<GatewayWsClient>,     // 连接的客户端
  broadcast: (event, payload) => void,  // 事件广播函数
  
  // Agent 执行层
  agentRunSeq: Map<string, number>,  // 每个 session 的 agent run 序列号
  chatRunState: ChatRunState,        // Chat run 状态跟踪
  chatRunBuffers: Map<string, string>,  // Chat 消息缓冲
  chatAbortControllers: Map<string, AbortController>,  // Abort controllers
  
  // Dedupe 层
  dedupe: Map<string, DedupeEntry>,  // 幂等 dedupe cache (send/agent 请求)
  
  // Canvas 层
  canvasHost: CanvasHostHandler | null,  // Canvas file server
};
```

**状态同步机制**:
- **Presence state**: 通过 `stateVersion` 标记版本，客户端检测 gap 后刷新
- **Health state**: 通过 `healthVersion` 标记版本，定期推送更新
- **Event streaming**: seq-tagged events，客户端检测丢失后重新订阅

## 四、数据流向与交互模式

### 4.1 消息流向 (Inbound: User → Agent)

```
┌─────────────┐
│ User        │
│ (WhatsApp)  │
└──────┬──────┘
       │ 1. Message arrives via Baileys WebSocket
       ↓
┌─────────────────────────────────────────────────────────┐
│ Channel Layer (src/telegram/, src/whatsapp/, etc.)     │
│  - Receive message via platform SDK                     │
│  - Normalize to GatewayMessage format                   │
│  - Check allowlist (security adapter)                   │
│  - Extract mentions, thread info                        │
└──────┬──────────────────────────────────────────────────┘
       │ 2. GatewayMessage
       ↓
┌─────────────────────────────────────────────────────────┐
│ Gateway Layer (src/gateway/)                            │
│  - Route to session (session key resolution)            │
│  - Check group activation (mention gating)              │
│  - Queue management (collect/steer/followup)            │
│  - Apply message hooks (plugins)                        │
└──────┬──────────────────────────────────────────────────┘
       │ 3. Enqueue agent run
       ↓
┌─────────────────────────────────────────────────────────┐
│ Agent Runtime (src/agents/)                             │
│  - Load session transcript                              │
│  - Assemble system prompt (AGENTS.md + skills + bootstrap) │
│  - Load skills snapshot                                 │
│  - Call Pi-embedded runtime (RPC mode)                  │
│  - Stream tool events & assistant deltas                │
└──────┬──────────────────────────────────────────────────┘
       │ 4. Tool execution (if needed)
       ↓
┌─────────────────────────────────────────────────────────┐
│ Tool Execution (src/agents/tools/)                      │
│  - Sandbox check (main vs non-main session)            │
│  - Execute tool (bash, browser, canvas, nodes, etc.)   │
│  - Return tool result                                   │
└──────┬──────────────────────────────────────────────────┘
       │ 5. Tool result → Agent
       ↓
┌─────────────────────────────────────────────────────────┐
│ Agent Runtime (continued)                               │
│  - Persist tool result to transcript                    │
│  - Continue agent loop until done                       │
│  - Finalize response (assistant text + tool summaries)  │
└──────┬──────────────────────────────────────────────────┘
       │ 6. Agent response
       ↓
┌─────────────────────────────────────────────────────────┐
│ Gateway Layer (outbound)                                │
│  - Apply send policy                                    │
│  - Chunk long messages (draft-chunking.ts)             │
│  - Add typing indicators                                │
└──────┬──────────────────────────────────────────────────┘
       │ 7. Send to channel
       ↓
┌─────────────────────────────────────────────────────────┐
│ Channel Layer (outbound adapter)                        │
│  - Format message (platform-specific)                   │
│  - Send via platform SDK                                │
│  - Handle rate limits, retries                          │
└──────┬──────────────────────────────────────────────────┘
       │ 8. Message delivered
       ↓
┌─────────────┐
│ User        │
│ (WhatsApp)  │
└─────────────┘
```

**关键环节**:
1. **Normalization**: 将不同平台消息格式统一为 `GatewayMessage`
2. **Session routing**: 根据 channel + user/group ID 解析 session key
3. **Security check**: Allowlist, pairing, mention gating
4. **Queueing**: 序列化同一 session 的 agent runs (避免竞态)
5. **Sandbox isolation**: 非 main session 运行在 Docker 中
6. **Streaming**: 实时流式输出 tool events 和 assistant deltas
7. **Chunking**: 长消息自动分块 (平台消息长度限制)

### 4.2 RPC 调用流向 (Client → Gateway)

```
┌─────────────┐
│ CLI Client  │
│ openclaw    │
│ agent       │
└──────┬──────┘
       │ 1. WS connect: {type:"req", method:"connect", params:{client,auth}}
       ↓
┌─────────────────────────────────────────────────────────┐
│ Gateway WebSocket Server (src/gateway/server-ws-runtime.ts) │
│  - Validate connect params (AJV + JSON Schema)          │
│  - Check auth (token/password or Tailscale identity)    │
│  - Register client, assign device ID                    │
│  - Return hello-ok with snapshot (presence, health)     │
└──────┬──────────────────────────────────────────────────┘
       │ 2. Send RPC request: {type:"req", id, method:"agent", params:{message,sessionKey}}
       ↓
┌─────────────────────────────────────────────────────────┐
│ Gateway RPC Router (src/gateway/server-methods.ts)     │
│  - Validate request schema                              │
│  - Check idempotency key (dedupe cache)                 │
│  - Route to handler: coreGatewayHandlers['agent']       │
└──────┬──────────────────────────────────────────────────┘
       │ 3. Execute handler
       ↓
┌─────────────────────────────────────────────────────────┐
│ Agent Handler (src/gateway/server-chat.ts)             │
│  - Resolve session key                                  │
│  - Enqueue agent run (lane serialization)               │
│  - Return ack: {runId, status:"accepted"}               │
│  - (Continue execution in background)                   │
└──────┬──────────────────────────────────────────────────┘
       │ 4. Stream events: {type:"event", event:"agent", payload:{stream:"tool"|"assistant"|"lifecycle"}}
       ↓
┌─────────────┐
│ CLI Client  │
│ (receives   │
│  streaming  │
│  events)    │
└──────┬──────┘
       │ 5. Final response: {type:"res", id, ok:true, payload:{runId,status:"ok",summary}}
       ↓
┌─────────────┐
│ CLI Client  │
│ (done)      │
└─────────────┘
```

**关键环节**:
1. **Mandatory handshake**: 首帧必须是 `connect`，否则 hard close
2. **Auth check**: Token/password auth 或 Tailscale identity
3. **Schema validation**: 所有 inbound frames 通过 AJV 验证
4. **Idempotency**: Side-effecting 方法 (send, agent) 支持幂等 key，安全重试
5. **Two-stage response**: Agent RPC 先返回 ack，再流式输出，最后 final response
6. **Event broadcasting**: Presence, tick, shutdown events 推送到所有客户端

### 4.3 Service Discovery (Bonjour/mDNS)

```
┌───────────────┐                     ┌───────────────┐
│ macOS App     │                     │ Gateway       │
│ (not connected)│                    │ (running)     │
└───────┬───────┘                     └───────┬───────┘
        │                                     │
        │ 1. Listen for _openclaw._tcp        │ 2. Publish mDNS service
        │    on local network                 │    name: "OpenClaw Gateway"
        │                                     │    type: _openclaw._tcp
        │                                     │    port: 18789
        │<────────────────────────────────────┤    TXT: version, deviceId
        │ 3. Discover Gateway via mDNS        │
        │                                     │
        │ 4. WS connect to discovered host:port
        ├────────────────────────────────────>│
        │                                     │
        │<────────────────────────────────────┤ 5. hello-ok
        │                                     │
```

**技术实现**: 使用 `@homebridge/ciao` 库 (Bonjour/mDNS)

**使用场景**:
- macOS app 自动发现局域网内的 Gateway
- iOS/Android nodes 自动发现并配对
- 多设备协作 (例如: iPhone 作为 camera node，MacBook 作为 Gateway)

## 五、技术栈选型理由

### 5.1 运行时与语言

- **Node.js**: >=22.12.0 (来自 `package.json:152-154`)
- **Package Manager**: pnpm@10.23.0 (来自 `package.json:155`)
- **Language**: TypeScript 5.9.3 (来自 `package.json:240`)

**选型理由**:
- **Node.js 22+**: 
  - Compile cache (`module.enableCompileCache()` in `openclaw.mjs:6-11`): 加速启动
  - 成熟的生态：WebSocket (ws), HTTP (express/hono), 消息平台 SDK
  - 异步 I/O: 适合 I/O 密集型任务 (WebSocket, HTTP, file system)
- **TypeScript**:
  - 类型安全：减少运行时错误
  - IDE 支持：IntelliSense、重构
  - Protocol codegen: TypeBox → JSON Schema → Swift types
- **pnpm**:
  - Monorepo 支持 (`pnpm-workspace.yaml`)
  - 节省磁盘空间：hard links
  - 快速安装：比 npm 快 2x

### 5.2 核心依赖 (package.json:156-209)

#### Agent Runtime (Pi-embedded)
- `@mariozechner/pi-agent-core: 0.49.3` - Pi agent 核心运行时
- `@mariozechner/pi-ai: 0.49.3` - Pi AI 模型接口
- `@mariozechner/pi-coding-agent: 0.49.3` - 代码 agent 支持
- `@mariozechner/pi-tui: 0.49.3` - Terminal UI

**选型理由**:
- **Pi-embedded 架构**: 将 Agent 嵌入 Gateway 进程 (RPC mode)
  - 低延迟：无需跨进程通信
  - 简化部署：单一进程，无需独立 Agent 服务
  - 统一生命周期：Gateway 重启时 Agent 自动重启
- **与 microservice 架构对比**:
  - Microservice: Agent 作为独立服务，通过 HTTP/gRPC 通信
  - Pi-embedded: Agent 作为库嵌入，通过函数调用
  - 权衡：Pi-embedded 牺牲了独立扩展性，换取了低延迟和简单部署

#### WebSocket & Network
- `ws: ^8.19.0` - WebSocket server
- `@homebridge/ciao: ^1.3.4` - Bonjour/mDNS 服务发现
- `undici: ^7.19.0` - HTTP client (Node.js 官方推荐)

**选型理由**:
- **ws**: Node.js 生态最成熟的 WebSocket 库
  - 高性能：C++ 扩展，低内存占用
  - 稳定性：广泛使用，bug 少
  - 兼容性：支持 WebSocket extensions (permessage-deflate)
- **@homebridge/ciao**: Bonjour/mDNS 实现
  - 纯 TypeScript：易于调试
  - 跨平台：macOS, Linux, Windows (通过 Avahi)
  - 服务发现：局域网自动发现 Gateway

#### Messaging Channels
- `@whiskeysockets/baileys: 7.0.0-rc.9` - WhatsApp (Baileys)
- `grammy: ^1.39.3` - Telegram bot framework
- `@grammyjs/runner: ^2.0.3` - Telegram runner
- `@buape/carbon: 0.14.0` - Discord.js wrapper
- `discord-api-types: ^0.38.37` - Discord API types
- `@slack/bolt: ^4.6.0` - Slack bot framework
- `@slack/web-api: ^7.13.0` - Slack API
- `@line/bot-sdk: ^10.6.0` - LINE messaging

#### Browser & Automation
- `playwright-core: 1.58.0` - Browser control (CDP)
- `chromium-bidi: 13.0.1` - Chrome BiDi protocol

#### WebSocket & Network
- `ws: ^8.19.0` - WebSocket server
- `@homebridge/ciao: ^1.3.4` - Bonjour/mDNS 服务发现
- `undici: ^7.19.0` - HTTP client

#### Schema & Validation
- `@sinclair/typebox: 0.34.47` - TypeBox schemas (强制固定版本)
- `ajv: ^8.17.1` - JSON schema validator
- `zod: ^4.3.6` - TypeScript-first schema validation

#### Media Processing
- `sharp: ^0.34.5` - 图片处理
- `@napi-rs/canvas: ^0.1.88` (可选) - Canvas 渲染
- `file-type: ^21.3.0` - 文件类型检测
- `pdfjs-dist: ^5.4.530` - PDF 解析

#### CLI & Terminal
- `commander: ^14.0.2` - CLI framework
- `@clack/prompts: ^0.11.0` - 交互式 CLI prompts
- `@lydell/node-pty: 1.2.0-beta.3` - PTY (pseudo-terminal)
- `cli-highlight: ^2.1.11` - 代码高亮
- `chalk: ^5.6.2` - 终端颜色
- `tslog: ^4.10.2` - 结构化日志

#### Data Storage
- `sqlite-vec: 0.1.7-alpha.2` - Vector database (Memory 系统)
- `proper-lockfile: ^4.1.2` - 文件锁

#### 其他关键依赖
- `@agentclientprotocol/sdk: 0.13.1` - Agent Client Protocol (ACP)
- `croner: ^9.1.0` - Cron jobs
- `node-edge-tts: ^1.2.9` - TTS (Text-to-Speech)
- `express: ^5.2.1` - HTTP server
- `hono: 4.11.4` - 轻量级 web framework
- `chokidar: ^5.0.0` - 文件监控 (hot reload)

### 开发依赖

- **Linting**: `oxlint: ^1.41.0`, `oxlint-tsgolint: ^0.11.1`
- **Formatting**: `oxfmt: 0.26.0`
- **Testing**: `vitest: ^4.0.18`, `@vitest/coverage-v8: ^4.0.18`
- **Build**: `tsx: ^4.21.0`, `rolldown: 1.0.0-rc.1`
- **UI**: `lit: ^3.3.2`, `@lit-labs/signals: ^0.2.0`
- **Swift**: SwiftLint, SwiftFormat (via scripts)

## 项目目录结构

### 顶层目录

```
openclaw/
├── src/                  # TypeScript 核心代码 (2,502 个 TS 文件)
├── apps/                 # 客户端应用
│   ├── macos/           # macOS 菜单栏应用 (Swift)
│   ├── ios/             # iOS node 应用 (Swift)
│   ├── android/         # Android node 应用 (Kotlin)
│   └── shared/          # 跨平台共享代码
├── docs/                 # 文档体系 (296 个 MD 文件)
├── extensions/           # 扩展 channels 和功能 (20+ 插件)
├── skills/               # 内置 skills (58 个)
├── ui/                   # WebChat 和 Control UI (Lit 组件)
├── test/                 # E2E 测试
├── scripts/              # 构建和部署脚本
├── packages/             # Monorepo packages
├── openclaw.mjs          # CLI 启动入口
├── package.json          # 项目配置和依赖
└── tsconfig.json         # TypeScript 配置
```

### 核心模块地图 (src/ 目录)

根据 src/ 目录结构，核心模块按职责划分：

#### 1. Agent Runtime (436 files)
**位置**: `src/agents/`

**职责**: Agent 核心执行引擎，基于 Pi-embedded runtime

**关键子模块**:
- `pi-embedded-runner/` - Pi agent 核心运行时 (30+ 文件)
- `pi-embedded-subscribe.ts` - 订阅和流式处理
- `pi-embedded-helpers/` - 辅助函数 (bootstrap, errors, messaging, thinking)
- `pi-embedded-block-chunker.ts` - 消息分块逻辑
- `sandbox/` - Docker 沙箱实现 (17+ 文件)
- `tools/` - 内置工具实现 (57+ 文件)
- `skills/` - Skills 加载和管理
- `model-compat.ts` - 模型兼容性适配
- `failover-error.ts` - Failover 错误处理
- `openclaw-tools.ts` - 工具注册入口

**核心能力**:
- Tool streaming: 工具调用的流式返回
- Block streaming: 消息分块和实时推送
- Sandbox isolation: Docker 隔离执行
- Model failover: 多模型切换和容错

#### 2. Gateway 控制平面 (187 files)
**位置**: `src/gateway/`

**职责**: WebSocket 控制平面，统一管理 sessions、channels、tools、events

**关键文件**:
- `server.impl.ts` - Gateway 服务核心实现
- `server.ts` - Gateway 启动入口
- `protocol/` - WS 协议定义 (21+ 文件)
- `server-methods/` - RPC 方法实现 (20+ 文件)
- `server-chat.ts` - 聊天消息处理
- `server-channels.ts` - Channel 管理
- `server-cron.ts` - Cron 管理
- `server-discovery-runtime.ts` - 服务发现 (Bonjour/mDNS)
- `server-tailscale.ts` - Tailscale 集成
- `auth.ts` - 身份认证
- `boot.ts` - 启动逻辑
- `session-utils.ts` - Session 存储和管理

**核心能力**:
- WebSocket RPC: 统一的 RPC 协议
- Multi-client support: macOS app, CLI, WebChat, iOS/Android nodes
- Service discovery: Bonjour/mDNS, SSH tunnels, Tailscale Serve/Funnel
- State management: Sessions, config, presence, cron

#### 3. Channel 集成 (101+ files)
**位置**: `src/channels/` + 各平台目录

**职责**: 统一的消息接口，适配不同平台

**各平台实现**:
- `src/telegram/` (85 files) - Telegram (grammY)
- `src/slack/` (65 files) - Slack (Bolt)
- `src/discord/` (61 files) - Discord (discord.js)
- `src/signal/` (24 files) - Signal (signal-cli)
- `src/imessage/` (16 files) - iMessage (imsg)
- `src/whatsapp/` (2 files) - WhatsApp (Baileys)
- `src/line/` (34 files) - LINE

**Channel 抽象层** (`src/channels/`):
- `plugins/catalog.ts` - Channel 插件目录
- `plugins/config-schema.ts` - 配置 schema
- `plugins/normalize/` - 消息格式规范化
- `plugins/outbound/` - 出站消息处理
- `plugins/actions/` - 平台特定 actions (Discord/Telegram/Signal)
- `plugins/pairing.ts` - Pairing 机制
- `registry.ts` - Channel 注册
- `allowlist-match.ts` - Allowlist 匹配
- `mention-gating.ts` - Mention gating
- `typing.ts` - Typing indicators

**核心能力**:
- Unified message interface: `GatewayMessageChannel`
- Platform adaptation: 消息格式转换、媒体处理
- DM security: Pairing mode, allowlist, dm policy
- Group routing: Mention gating, reply tags

#### 4. Session 管理 (7 files)
**位置**: `src/sessions/`

**职责**: 多会话隔离与协作

**关键文件**:
- `send-policy.ts` - 消息发送策略
- `level-overrides.ts` - 级别覆盖 (thinking, verbose)
- `model-overrides.ts` - 模型覆盖
- `session-key-utils.ts` - Session key 生成
- `transcript-events.ts` - 会话事件记录

**核心能力**:
- Session isolation: main session vs group sessions
- Session lifecycle: 创建、激活、持久化、compaction、销毁
- Agent-to-Agent communication: `sessions_*` tools

#### 5. CLI & Commands (380 files)
**位置**: `src/cli/` (157 files) + `src/commands/` (223 files)

**职责**: 命令行工具和命令实现

**核心命令** (根据 README.md 和 docs/cli/):
- `openclaw onboard` - 引导式配置向导
- `openclaw gateway` - 启动 Gateway 服务
- `openclaw agent` - 启动 Agent
- `openclaw message send` - 发送消息
- `openclaw channels login` - Channel 登录
- `openclaw doctor` - 诊断配置问题
- `openclaw pairing approve` - 批准 pairing 请求

#### 6. Configuration (130 files)
**位置**: `src/config/`

**职责**: 配置管理和验证

**核心配置文件**: `~/.openclaw/openclaw.json`

**配置模块**:
- Schema validation (TypeBox)
- Config patching (动态更新)
- Environment variables
- Profile management (dev/prod)

#### 7. Auto-reply (206 files)
**位置**: `src/auto-reply/`

**职责**: 自动回复系统

**核心能力**:
- Auto-reply triggers: 自动回复触发机制
- Rule engine: 规则引擎

#### 8. Browser Control (81 files)
**位置**: `src/browser/`

**职责**: 浏览器控制和自动化

**核心技术**: Playwright (CDP - Chrome DevTools Protocol)

**核心能力**:
- Browser snapshots: 页面快照
- Browser actions: 点击、输入、导航
- File uploads: 文件上传
- Profile management: 浏览器 profile 管理

#### 9. Media Processing (20 files)
**位置**: `src/media/`

**职责**: 统一媒体 pipeline

**核心文件**:
- `fetch.ts` - 媒体下载
- `store.ts` - 媒体存储
- `parse.ts` - 媒体解析
- `image-ops.ts` - 图片操作 (sharp)
- `audio.ts` - 音频处理

#### 10. Media Understanding (36 files)
**位置**: `src/media-understanding/`

**职责**: 图片、音频、视频内容理解

**核心能力**:
- Image understanding: 图片内容识别
- Audio transcription: 音频转文字
- Video understanding: 视频内容分析

#### 11. Memory System (33 files)
**位置**: `src/memory/`

**职责**: 长期记忆存储和检索

**核心技术**: sqlite-vec (Vector database)

#### 12. Plugins (37 files)
**位置**: `src/plugins/`

**职责**: 插件生态系统

**核心模块**:
- `loader.ts` - 插件加载器
- `discovery.ts` - 插件发现
- `install.ts` - 插件安装
- `manifest.ts` - Manifest 管理
- `hooks.ts` - Hook 系统
- `tools.ts` - 插件工具加载
- `runtime/` - 插件运行时

#### 13. Security (9 files)
**位置**: `src/security/`

**职责**: 安全模块

**核心能力**:
- TCC permissions (macOS)
- Elevated bash approval
- Exec approval manager (在 `src/gateway/exec-approval-manager.ts`)

#### 14. Pairing (5 files)
**位置**: `src/pairing/`

**职责**: DM 安全策略

**核心能力**:
- Pairing code generation
- Pairing code verification
- Allowlist management

#### 15. 其他重要模块

- `src/acp/` (13 files) - Agent Client Protocol
- `src/cron/` (30 files) - Cron jobs
- `src/daemon/` (30 files) - Daemon 管理
- `src/infra/` (183 files) - 基础设施
- `src/logging/` (14 files) - 日志系统
- `src/routing/` (4 files) - 消息路由
- `src/terminal/` (11 files) - Terminal 模拟
- `src/tts/` (2 files) - Text-to-Speech
- `src/tui/` (37 files) - Terminal UI
- `src/web/` (77 files) - WebChat 和 Control UI
- `src/wizard/` (9 files) - Onboarding wizard

## 启动流程

### 入口点

根据 `openclaw.mjs:1-14`:

```javascript
#!/usr/bin/env node

import module from "node:module";

// Enable compile cache for faster startup
if (module.enableCompileCache && !process.env.NODE_DISABLE_COMPILE_CACHE) {
  try {
    module.enableCompileCache();
  } catch {
    // Ignore errors
  }
}

await import("./dist/entry.js");
```

### 启动链路

```
openclaw.mjs
    ↓
dist/entry.js (compiled from src/cli/entry.ts)
    ↓
CLI framework (commander.js)
    ↓
Commands dispatch (src/commands/*.ts)
    ↓
Gateway startup (src/gateway/server.ts)
    ↓
Gateway.impl.ts (core implementation)
    ↓
WebSocket server listening on ws://127.0.0.1:18789
```

### Gateway 启动关键步骤

根据 `src/gateway/` 目录分析：

1. **boot.ts**: Gateway 启动逻辑
2. **server-startup.ts**: 启动流程
3. **server-discovery-runtime.ts**: 服务发现 (Bonjour/mDNS)
4. **server-channels.ts**: 启动各个 channels
5. **server-cron.ts**: 启动 cron jobs
6. **server-chat.ts**: 启动聊天消息处理
7. **server-ws-runtime.ts**: WebSocket 运行时

### 配置加载

根据 `package.json:bin:14`:

```json
"bin": {
  "openclaw": "./openclaw.mjs"
}
```

配置文件路径 (来自 README.md):
- `~/.openclaw/openclaw.json` - 主配置文件
- `~/.openclaw/workspace/` - Workspace 根目录
- `~/.openclaw/credentials/` - 凭证存储

## 文档体系 (docs/ 目录)

### 文档分类

根据 `docs/` 目录结构，文档按以下类别组织：

#### 1. 快速开始 (start/)
- `getting-started.md` - 新手指南
- `wizard.md` - Onboarding wizard
- `onboarding.md` - Onboarding 详细流程
- `showcase.md` - 功能展示
- `faq.md` - 常见问题

#### 2. 安装部署 (install/)
- `index.md` - 安装总览
- `docker.md` - Docker 部署
- `nix.md` - Nix 声明式配置
- `updating.md` - 升级指南
- `development-channels.md` - 开发渠道 (stable/beta/dev)

#### 3. Gateway 运维 (gateway/)
- `index.md` - Gateway 运维手册
- `configuration.md` - 配置参考 (所有配置项)
- `security/` - 安全指南
- `sandboxing.md` - 沙箱配置
- `remote.md` - 远程访问
- `tailscale.md` - Tailscale 集成
- `doctor.md` - 诊断工具
- `health.md` - 健康检查

#### 4. Channels 集成 (channels/)
- `whatsapp.md`, `telegram.md`, `slack.md`, `discord.md` 等
- `troubleshooting.md` - Channel 故障排查

#### 5. 核心概念 (concepts/)
- `architecture.md` - 架构概览
- `agent-loop.md` - Agent 循环
- `sessions.md` - Session 管理
- `groups.md` - Group 规则
- `models.md` - 模型配置
- `model-failover.md` - 模型 failover
- `presence.md` - 在线状态
- `typebox.md` - TypeBox schemas

#### 6. 工具和 Skills (tools/)
- `browser.md` - 浏览器控制
- `skills.md` - Skills 平台
- `creating-skills.md` - 创建 skills

#### 7. 平台应用 (platforms/)
- `macos.md` - macOS 应用
- `ios.md` - iOS node
- `android.md` - Android node
- `windows.md` - Windows (WSL2)

#### 8. CLI 参考 (cli/)
- 每个命令的详细文档 (40+ 个 CLI 命令)

#### 9. 自动化 (automation/)
- `cron-jobs.md` - Cron 定时任务
- `webhook.md` - Webhook 触发
- `gmail-pubsub.md` - Gmail Pub/Sub 集成

#### 10. 参考资料 (reference/)
- `templates/` - System Prompt 模板 (AGENTS.md, SOUL.md, TOOLS.md 等)
- `AGENTS.default.md` - 默认 AGENTS 规范
- `rpc.md` - RPC 适配器

## 工程实践亮点

### 1. Monorepo 组织

根据项目结构，采用 **单仓库多包** 的组织方式：
- `pnpm-workspace.yaml` - Workspace 配置
- `packages/` - Monorepo packages
- `extensions/` - 扩展插件
- `apps/` - 客户端应用

### 2. TypeScript 项目

- **TypeScript 5.9.3**: 类型安全
- **TypeBox schemas**: 运行时类型验证 (强制固定版本 0.34.47)
- **Strict types**: 严格的类型检查

### 3. 测试策略

根据 `package.json:80-147` 的 scripts：

**测试类型**:
- Unit tests: `vitest`
- E2E tests: `vitest.e2e.config.ts`
- Live tests: `vitest.live.config.ts`
- Docker tests: `test:docker:*` 系列

**测试覆盖率**: 70% (lines, functions, branches, statements)

**Docker 测试环境**:
- `test:docker:onboard` - Onboarding 测试
- `test:docker:gateway-network` - Gateway 网络测试
- `test:docker:live-models` - Live 模型测试
- `test:docker:plugins` - 插件测试

### 4. 开发工具链

根据 `package.json:114-122`:

**Linting**: 
- `oxlint` - 快速 TypeScript linter
- `oxlint-tsgolint` - TypeScript 专用规则
- `swiftlint` - Swift 代码检查

**Formatting**:
- `oxfmt` - TypeScript 格式化
- `swiftformat` - Swift 格式化

**Hot Reload**:
- `gateway:watch` - Gateway 自动重载 (chokidar)

### 5. CLI 设计

根据 `src/cli/` 和 `src/commands/` 目录：

**CLI Framework**: commander.js
**Interactive prompts**: @clack/prompts
**Wizard system**: 引导式配置 (onboarding wizard)
**Doctor command**: 配置诊断和修复建议

### 6. 插件生态

根据 `src/plugins/` 和 `extensions/` 目录：

**Plugin SDK**: `src/plugin-sdk/`
**Extension channels**: 20+ 扩展插件
**Dynamic loading**: 动态加载和注册
**Hook system**: 插件 hook 机制

### 7. 安全设计

根据 README.md 和 `src/security/`, `src/pairing/`:

**DM Security**:
- Pairing mode: 陌生人需要 pairing code 才能对话
- Allowlist: 显式 allowlist 管理
- DM policy: `pairing` / `open` 模式

**Sandbox Isolation**:
- Docker sandboxes: 非 main session 运行在 Docker 中
- Tool policy: 工具访问控制
- Elevated bash: 需要审批的 elevated 权限

### 8. 模型支持

根据 `package.json:156-214` 和 README.md:

**支持的模型供应商**:
- Anthropic: Claude (推荐 Opus 4.5)
- OpenAI: GPT, Codex
- AWS Bedrock: `@aws-sdk/client-bedrock`
- Ollama: Local models (devDependency)
- Node-llama-cpp: Local LLM (optional)

**模型 Failover**: 自动切换和容错 (src/agents/failover-error.ts)

### 9. 发布流程

根据 `package.json:78-83`:

**Development channels**:
- `stable`: Tagged releases (`vYYYY.M.D`)
- `beta`: Prerelease tags (`vYYYY.M.D-beta.N`)
- `dev`: Moving head of `main`

**Scripts**:
- `release:check` - Release 检查
- `protocol:check` - Protocol 检查
- `plugins:sync` - 同步插件版本

### 10. 跨平台支持

**macOS**: 原生菜单栏应用 (Swift)
**iOS**: Node 应用 (Swift)
**Android**: Node 应用 (Kotlin)
**Linux**: Gateway + CLI
**Windows**: WSL2 支持

## 依赖关系矩阵

### 核心模块依赖

```
Gateway (控制平面)
  ├─→ Agent Runtime (Pi-embedded)
  │     ├─→ Tools (browser, canvas, nodes, sessions, etc.)
  │     ├─→ Sandbox (Docker isolation)
  │     └─→ Model Compat (multi-model support)
  ├─→ Channels (messaging platforms)
  │     ├─→ WhatsApp (Baileys)
  │     ├─→ Telegram (grammY)
  │     ├─→ Slack (Bolt)
  │     ├─→ Discord (discord.js)
  │     └─→ ...
  ├─→ Sessions (multi-session isolation)
  ├─→ Config (configuration management)
  ├─→ Cron (scheduled tasks)
  ├─→ Memory (long-term storage, sqlite-vec)
  ├─→ Media (media pipeline, sharp)
  ├─→ Media Understanding (image/audio/video)
  ├─→ Plugins (extensibility)
  └─→ Security (pairing, sandboxing)
```

### 外部依赖关系

```
Pi agent runtime (@mariozechner/pi-*)
  ├─→ Claude API (Anthropic)
  ├─→ OpenAI API
  └─→ AWS Bedrock

WebSocket (ws)
  ├─→ macOS app
  ├─→ CLI
  ├─→ WebChat
  └─→ iOS/Android nodes

Service Discovery
  ├─→ Bonjour/mDNS (@homebridge/ciao)
  ├─→ SSH tunnels
  └─→ Tailscale Serve/Funnel

Media Processing
  ├─→ sharp (image)
  ├─→ @napi-rs/canvas (optional)
  └─→ pdfjs-dist (PDF)

Browser Control
  └─→ playwright-core (CDP)
```

## 六、架构权衡分析

### 6.1 Local-First vs Cloud-First

| 维度 | Local-First (OpenClaw) | Cloud-First (SaaS) | 权衡分析 |
|------|----------------------|-------------------|---------|
| **数据隐私** | ✅ 数据不离开本地设备 | ❌ 数据存储在云端 | Local-First 更适合隐私敏感场景 |
| **延迟** | ✅ 无网络往返 (<10ms) | ❌ 网络往返 (50-200ms) | Local-First 响应更快 |
| **离线可用** | ✅ 基础功能离线可用 | ❌ 依赖网络 | Local-First 更可靠 |
| **扩展性** | ❌ 单机资源限制 | ✅ 云端弹性扩展 | Cloud-First 更适合高并发场景 |
| **部署复杂度** | ⚠️ 需要本地运行环境 | ✅ 零部署 | Cloud-First 用户体验更好 |
| **成本** | ✅ 无云端基础设施成本 | ❌ 云端资源成本 | Local-First 更经济 |

**OpenClaw 的选择**: Personal AI assistant 场景，数据隐私和低延迟优先级更高，单用户场景不需要云端扩展性，Local-First 更合适。

### 6.2 Embedded Agent vs Microservice Agent

| 维度 | Embedded (OpenClaw) | Microservice | 权衡分析 |
|------|-------------------|--------------|---------|
| **延迟** | ✅ 函数调用 (<1ms) | ❌ 跨进程 RPC (10-50ms) | Embedded 更快 |
| **部署** | ✅ 单一进程 | ❌ 多个服务 | Embedded 更简单 |
| **扩展性** | ❌ 无法独立扩展 | ✅ 独立扩展 Agent | Microservice 更灵活 |
| **语言隔离** | ❌ 必须同一语言 | ✅ 多语言混合 | Microservice 更灵活 |
| **故障隔离** | ❌ Agent 崩溃影响 Gateway | ✅ 故障隔离 | Microservice 更可靠 |

**OpenClaw 的选择**: Personal assistant 场景，单用户单机，不需要独立扩展 Agent，Embedded 架构更简单且延迟更低。

### 6.3 WebSocket vs HTTP/REST

| 维度 | WebSocket (OpenClaw) | HTTP/REST | 权衡分析 |
|------|---------------------|-----------|---------|
| **双向通信** | ✅ Server push 事件 | ❌ 只能 client poll | WebSocket 更高效 |
| **流式响应** | ✅ 原生支持 streaming | ⚠️ 需要 SSE 或 chunked | WebSocket 更简单 |
| **连接开销** | ✅ 长连接 (一次握手) | ❌ 每次请求都握手 | WebSocket 更高效 |
| **负载均衡** | ⚠️ 需要 sticky sessions | ✅ 无状态，易负载均衡 | HTTP 更友好 |
| **缓存** | ❌ 无法缓存 | ✅ HTTP cache | HTTP 更友好 |
| **调试** | ⚠️ 需要专用工具 | ✅ curl/Postman | HTTP 更友好 |

**OpenClaw 的选择**: Gateway 需要主动推送事件 (presence, agent streaming, shutdown)，WebSocket 是天然选择。单机部署不需要负载均衡。

### 6.4 One Gateway vs Multiple Gateways

| 维度 | One Gateway (OpenClaw) | Multiple Gateways | 权衡分析 |
|------|---------------------|-------------------|---------|
| **状态一致性** | ✅ 单一状态源 | ❌ 分布式一致性问题 | One Gateway 更简单 |
| **Baileys session** | ✅ 唯一 session | ❌ 冲突问题 | One Gateway 是必需的 |
| **可用性** | ❌ 单点故障 | ✅ 高可用 | Multiple 更可靠 |
| **扩展性** | ❌ 单机资源限制 | ✅ 水平扩展 | Multiple 更强 |

**OpenClaw 的选择**: WhatsApp Baileys session 限制 (同一账号只能一个连接)，One Gateway 是唯一选择。支持多 profile 隔离 (rescue bot pattern)。

### 6.5 Docker Sandbox vs Process Isolation

| 维度 | Docker Sandbox (OpenClaw) | Process Isolation | 权衡分析 |
|------|-------------------------|-------------------|---------|
| **隔离强度** | ✅ 容器级隔离 | ⚠️ 进程级隔离 | Docker 更安全 |
| **资源限制** | ✅ cgroup 限制 | ⚠️ ulimit 限制 | Docker 更强 |
| **文件系统** | ✅ 独立文件系统 | ❌ 共享文件系统 | Docker 更安全 |
| **启动开销** | ❌ 容器启动 (1-3s) | ✅ 进程启动 (<100ms) | Process 更快 |
| **依赖管理** | ✅ 打包所有依赖 | ❌ 需要主机环境 | Docker 更简单 |

**OpenClaw 的选择**: Group/channel 消息不可信，需要强隔离防止 prompt injection 攻击。Docker 是合理选择，启动开销可接受 (agent run 通常 >5s)。

## 七、可借鉴的架构模式与设计精髓

基于深入的架构分析，以下是 OpenClaw 最值得借鉴的设计模式：

### 7.1 架构模式

#### 1) Local-First Gateway Pattern

**核心思想**: 单一本地控制平面，统一管理所有分布式服务 (channels, clients, agents)

**适用场景**:
- Personal tools (单用户工具)
- 隐私敏感应用 (医疗、金融)
- 低延迟要求 (实时协作)

**实现要点**:
- WebSocket 统一协议
- Service discovery (Bonjour/mDNS)
- 配置热重载 (hybrid mode)

#### 2) Channel Plugin Pattern

**核心思想**: 通过统一接口抽象异构系统 (10+ 消息平台)，可插拔扩展

**适用场景**:
- 多平台集成 (消息、支付、存储)
- 社区生态建设 (扩展 channels)

**实现要点**:
- 定义核心接口 (`ChannelPlugin`)
- 可选适配器 (按需实现)
- 能力声明 (`capabilities`)
- 类型安全 (泛型 `<ResolvedAccount>`)

**代码示例** (简化):
```typescript
// 核心接口
interface ChannelPlugin<Account> {
  id: string;
  config: ChannelConfigAdapter<Account>;
  gateway?: ChannelGatewayAdapter<Account>;  // 可选
  outbound?: ChannelOutboundAdapter;        // 可选
  // ... 其他可选适配器
}

// 平台实现只需实现所需能力
const telegramPlugin: ChannelPlugin<TelegramAccount> = {
  id: 'telegram',
  config: telegramConfigAdapter,
  gateway: telegramGatewayAdapter,  // 启动 bot
  outbound: telegramOutboundAdapter, // 发送消息
  // 不实现 threading (Telegram 不支持)
};
```

#### 3) Session-Based Isolation Pattern

**核心思想**: 通过 session 隔离不同用户、channel、场景，实现安全边界

**适用场景**:
- Multi-tenant 系统
- 权限分级 (admin vs user)
- 安全隔离 (trusted vs untrusted)

**实现要点**:
- Session key 设计: `<channel>:<userId|groupId>`
- Session 类型: main (full access) vs non-main (restricted)
- Sandbox isolation: main → host, non-main → Docker
- Tool policy: 基于 session 类型的工具访问控制

#### 4) Embedded Agent Pattern

**核心思想**: Agent 作为库嵌入主进程 (RPC mode)，而非独立服务

**适用场景**:
- 单用户应用 (不需要独立扩展)
- 低延迟要求 (函数调用 vs RPC)
- 简化部署 (单一进程)

**实现要点**:
- Agent runtime 作为库 (`pi-agent-core`)
- RPC mode: 通过函数调用而非网络
- 生命周期绑定: Gateway 重启 → Agent 重启

**对比**:
- Microservice Agent: Agent 作为独立服务，HTTP/gRPC 通信
- Embedded Agent: Agent 作为库，函数调用
- 权衡: Embedded 牺牲扩展性，换取低延迟和简单部署

### 7.2 设计精髓

#### 1) 类型驱动的协议设计

**核心思想**: TypeBox schemas → JSON Schema → Swift types，端到端类型安全

**价值**:
- 编译时错误检测
- 自动生成文档 (JSON Schema)
- 跨语言类型安全 (TypeScript ↔ Swift)

**实现要点**:
- 使用 TypeBox 定义协议 (运行时类型验证)
- 生成 JSON Schema (协议文档)
- Codegen Swift types (iOS/macOS app)

#### 2) 事件驱动的状态同步

**核心思想**: Server 主动推送 events (presence, health)，客户端根据 `stateVersion` 检测 gap

**价值**:
- 实时性：无需轮询
- 一致性：`stateVersion` 保证
- 可恢复性：gap detection → refresh

**实现要点**:
- Seq-tagged events: `{type:"event", event, payload, seq, stateVersion}`
- Gap detection: 客户端检测 seq 不连续
- Refresh mechanism: `health` + `system-presence` 刷新全量状态

#### 3) 幂等性设计

**核心思想**: Side-effecting 操作 (send, agent) 支持 idempotency key，安全重试

**价值**:
- 网络故障恢复：客户端可安全重试
- 防止重复执行：dedupe cache
- 分布式一致性：跨网络的操作幂等

**实现要点**:
- Idempotency key: 客户端生成 UUID
- Dedupe cache: Gateway 维护短期 cache (5分钟)
- Response caching: 相同 key 返回缓存结果

#### 4) 配置热重载

**核心思想**: 配置变更立即生效 (safe changes) 或触发进程内重启 (critical changes)

**价值**:
- 零停机：safe changes 无需重启
- 快速恢复：in-process restart (SIGUSR1) 避免 systemd 重启
- 自动化：配置文件 watch → 自动 reload

**实现要点**:
- Hybrid mode: 区分 safe changes 和 critical changes
- SIGUSR1 restart: 进程内重启 (保持 PID，避免 systemd 干预)
- Config diff: 对比新旧配置，判断是否需要重启

### 7.3 工程实践亮点

#### 1) Monorepo 组织

- **pnpm workspace**: 管理多包 (gateway, cli, plugins, extensions)
- **Shared types**: 跨包类型共享
- **Unified build**: 统一构建和测试

#### 2) 测试策略

- **70% 覆盖率**: 高质量保证
- **多层次测试**: Unit, E2E, Live, Docker
- **Vitest**: 快速测试框架

#### 3) CLI 设计

- **Onboarding wizard**: 引导式配置 (交互式 prompts)
- **Doctor command**: 自动诊断配置问题
- **命令补全**: shell completion

#### 4) 插件生态

- **Plugin SDK**: 标准化插件接口
- **Hook system**: 灵活的扩展点 (before_agent_start, message_received 等)
- **20+ extensions**: 丰富的扩展 channels (BlueBubbles, Matrix, Zalo 等)

## 八、AI 主动性三大核心：Heartbeat、Memory 自我整理、Skills 持续学习

> **设计哲学突破**: 从被动响应的工具 → 主动察觉的助手

道友观察到的这三个特性，恰恰是 OpenClaw 区别于其他 AI 项目的**灵魂设计**。它们共同构成了 AI 的"主动性"：

### 8.1 Heartbeat: AI 主动"醒来"

**核心理念** (来自 `docs/gateway/heartbeat.md:11-12`):
> Heartbeat runs **periodic agent turns** in the main session so the model can surface anything that needs attention without spamming you.

#### 设计精髓

传统 AI 助手是**被动的**：用户发消息 → AI 响应 → 等待下一条消息

OpenClaw Heartbeat 是**主动的**：AI 定期"醒来" → 检查待办事项 → 主动提醒

**Heartbeat 架构**:

```
┌─────────────────────────────────────────────────────────────────┐
│ Heartbeat Runner (src/infra/heartbeat-runner.ts)               │
│  - 定时器: 每 30 分钟 (可配置)                                   │
│  - Active hours: 08:00-22:00 (可配置)                          │
│  - 触发条件: 时间到 + 在 active hours 内                        │
└──────────┬──────────────────────────────────────────────────────┘
           │ 1. Timer fires
           ↓
┌─────────────────────────────────────────────────────────────────┐
│ Agent Run (Main Session)                                        │
│  - 读取 HEARTBEAT.md checklist                                  │
│  - 执行检查任务 (inbox, calendar, reminders)                   │
│  - 决策: 需要提醒 vs 一切正常                                   │
└──────────┬──────────────────────────────────────────────────────┘
           │ 2. Agent decision
           ↓
┌─────────────────────────────────────────────────────────────────┐
│ Response Handling                                                │
│  - HEARTBEAT_OK → 静默 (不发送消息)                            │
│  - Alert content → 发送到用户                                   │
└─────────────────────────────────────────────────────────────────┘
```

**HEARTBEAT_OK 机制** (来自 `docs/gateway/heartbeat.md:62-71`):

```markdown
Response contract:
- If nothing needs attention, reply with **`HEARTBEAT_OK`**.
- OpenClaw treats `HEARTBEAT_OK` as an ack when it appears at the **start or end** of the reply.
- The token is stripped and the reply is dropped if remaining content ≤ 300 chars.
- For alerts, **do not** include `HEARTBEAT_OK`; return only the alert text.
```

**关键设计**:
- **智能静默**: 没有重要事项时不打扰用户 (`HEARTBEAT_OK`)
- **上下文感知**: Heartbeat 运行在 main session，了解用户最近的对话和工作
- **批量检查**: 一次 heartbeat 可以检查多个事项 (inbox + calendar + tasks)，避免多个 cron 的 API 开销
- **自然时间**: 非精确时间，轻微漂移可接受 (适合监控类任务)

#### Heartbeat vs Cron 对比

| 维度 | Heartbeat | Cron (isolated) | 使用场景 |
|------|-----------|-----------------|---------|
| **时间** | 近似 (30m ±drift) | 精确 (cron expression) | Heartbeat: "每30分钟检查"<br>Cron: "每天9:00整" |
| **上下文** | Full main session context | Clean slate | Heartbeat: 上下文相关决策<br>Cron: 独立任务 |
| **批量** | 可批量多个检查 | 单一任务 | Heartbeat: inbox+calendar+tasks<br>Cron: 单独报告 |
| **成本** | 一次 agent run 处理多项 | 每个任务一次 run | Heartbeat 更经济 |
| **输出** | 智能静默 (HEARTBEAT_OK) | 总是输出 | Heartbeat 避免噪音 |

**HEARTBEAT.md 示例** (来自 `docs/automation/cron-vs-heartbeat.md:159-166`):

```markdown
# Heartbeat checklist
- Scan inbox for urgent emails
- Check calendar for events in next 2h
- Review any pending tasks
- Light check-in if quiet for 8+ hours
```

**配置示例** (来自 `docs/gateway/heartbeat.md:77-93`):

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",           // 间隔
        model: "anthropic/claude-opus-4-5",
        target: "last",         // 发送到最后使用的 channel
        activeHours: { start: "08:00", end: "22:00" },  // 活跃时间
        ackMaxChars: 300       // HEARTBEAT_OK 阈值
      }
    }
  }
}
```

**Agent 可以自我更新 HEARTBEAT.md** (来自 `docs/gateway/heartbeat.md:251-265`):

> Can the agent update HEARTBEAT.md? **Yes — if you ask it to.**
> 
> `HEARTBEAT.md` is just a normal file in the agent workspace, so you can tell the agent (in a normal chat):
> - "Update `HEARTBEAT.md` to add a daily calendar check."
> - "Rewrite `HEARTBEAT.md` so it's shorter and focused on inbox follow-ups."
> 
> If you want this to happen proactively, you can also include an explicit line in your heartbeat prompt like: "If the checklist becomes stale, update HEARTBEAT.md with a better one."

**这是自我进化的关键**：AI 可以根据用户反馈和实际需求，主动优化自己的 heartbeat checklist！

#### 核心突破

**Heartbeat 让 AI 从"等待命令的仆人"变成"主动关心的管家"**:
- ✅ 定期检查重要事项 (inbox, calendar, tasks)
- ✅ 智能决策是否需要提醒 (上下文感知)
- ✅ 批量处理多个检查 (成本优化)
- ✅ 智能静默 (避免打扰)
- ✅ 自我优化 (Agent 可更新 HEARTBEAT.md)

### 8.2 Memory 自我整理: AI 主动管理长期记忆

**核心理念**: AI 不仅存储记忆，还**主动整理和索引**记忆，确保长期记忆可检索和可用

#### 设计精髓

传统 AI 记忆是**被动存储**：对话历史 → 存储到文件 → 超出上下文窗口就丢失

OpenClaw Memory 是**主动管理**：对话历史 → 自动分块 (chunk) → 生成 embeddings → 向量索引 → 可语义搜索

**Memory 架构** (来自 `src/memory/manager.ts:119`):

```typescript
export class MemoryIndexManager {
  // Core components
  private db: DatabaseSync;                    // SQLite database
  private provider: EmbeddingProvider;         // OpenAI/Gemini/Local embeddings
  private sources: Set<MemorySource>;          // "memory" | "sessions"
  
  // Index tables
  // - chunks_vec: Vector embeddings (sqlite-vec)
  // - chunks_fts: Full-text search (FTS5)
  // - embedding_cache: Dedupe cache
  
  // Auto-sync
  private watcher: FSWatcher;                  // chokidar file watcher
  private sessionDirtyTimer: NodeJS.Timeout;   // Debounce timer
}
```

**自动索引流程**:

```
┌─────────────────────────────────────────────────────────────────┐
│ Step 1: File Watcher (chokidar)                                 │
│  - 监控 ~/.openclaw/memory/ (memory files)                      │
│  - 监控 ~/.openclaw/sessions/*/transcript.jsonl (session logs)  │
│  - Debounce: 5s (避免频繁触发)                                   │
└──────────┬──────────────────────────────────────────────────────┘
           │ File changed
           ↓
┌─────────────────────────────────────────────────────────────────┐
│ Step 2: Chunk Generation (src/memory/internal.ts)               │
│  - Markdown chunking: 512 tokens per chunk, 50 tokens overlap   │
│  - Hash-based dedupe: 跳过已索引的 chunks                        │
│  - Extract metadata: path, startLine, endLine, content          │
└──────────┬──────────────────────────────────────────────────────┘
           │ Chunks ready
           ↓
┌─────────────────────────────────────────────────────────────────┐
│ Step 3: Embedding Generation                                    │
│  - Provider: OpenAI (text-embedding-3-small) / Gemini / Local   │
│  - Batch API: OpenAI/Gemini batch endpoint (cost optimization)  │
│  - Concurrency: 4 parallel requests                             │
│  - Cache: embedding_cache table (dedupe by content hash)        │
└──────────┬──────────────────────────────────────────────────────┘
           │ Embeddings ready
           ↓
┌─────────────────────────────────────────────────────────────────┐
│ Step 4: Index Storage (SQLite + sqlite-vec)                     │
│  - chunks_vec table: vector embeddings                          │
│  - chunks_fts table: full-text search (FTS5)                    │
│  - Hybrid search: vector + keyword                              │
└─────────────────────────────────────────────────────────────────┘
```

**Hybrid Search** (来自 `src/memory/hybrid.ts`):

```typescript
// Vector search: 语义相似度
const vectorResults = await searchVector(db, queryEmbedding, topK);

// Keyword search: BM25 全文搜索
const keywordResults = await searchKeyword(db, queryText, topK);

// Merge: 混合排序 (vector score + BM25 score)
const mergedResults = mergeHybridResults(vectorResults, keywordResults);
```

**关键设计常量** (来自 `src/memory/manager.ts:92-110`):

```typescript
const SNIPPET_MAX_CHARS = 700;                    // Snippet 最大字符数
const EMBEDDING_BATCH_MAX_TOKENS = 8000;          // Batch embedding 最大 tokens
const SESSION_DIRTY_DEBOUNCE_MS = 5000;           // Session 变更 debounce
const EMBEDDING_INDEX_CONCURRENCY = 4;            // 并发 embedding 请求
const EMBEDDING_RETRY_MAX_ATTEMPTS = 3;           // 重试次数
const VECTOR_LOAD_TIMEOUT_MS = 30_000;            // Vector extension 加载超时
const EMBEDDING_QUERY_TIMEOUT_REMOTE_MS = 60_000; // Remote embedding 查询超时
const EMBEDDING_BATCH_TIMEOUT_REMOTE_MS = 2 * 60_000; // Batch 处理超时
```

#### 自动整理机制

**1. Session Transcript Auto-Sync** (来自 `src/memory/sync-session-files.ts`):

```typescript
// 自动同步 session transcripts 到 memory index
export function syncSessionFiles(manager: MemoryIndexManager) {
  // 1. 监听 transcript 更新事件
  onSessionTranscriptUpdate((sessionKey) => {
    // 2. Debounce: 5s 后触发索引
    // 3. 读取 transcript.jsonl (delta read: 只读新增部分)
    // 4. 提取对话内容 (user + assistant messages)
    // 5. Chunk + Embedding + Index
  });
}
```

**2. Memory Files Auto-Sync** (来自 `src/memory/sync-memory-files.ts`):

```typescript
// 自动同步 memory files 到 index
export function syncMemoryFiles(manager: MemoryIndexManager) {
  // 1. 监控 ~/.openclaw/memory/ (chokidar)
  // 2. 检测文件变更 (add, change)
  // 3. Debounce: 避免频繁重建
  // 4. 增量索引: 只处理变更的文件
  // 5. Atomic reindex: 使用临时表，完成后原子替换
}
```

**3. Embedding Cache** (避免重复计算):

```typescript
// embedding_cache table
{
  hash: string,      // Content hash (SHA-256)
  embedding: Buffer, // Vector embedding (Float32Array)
  model: string,     // Embedding model
  createdAt: number  // Timestamp
}

// 查询前先检查 cache
if (cache.has(contentHash)) {
  return cache.get(contentHash);
}
```

#### Memory Search API

**Agent 可以主动搜索记忆** (通过 memory_search tool):

```typescript
// Tool call example
{
  "name": "memory_search",
  "params": {
    "query": "What did we discuss about the project deadline?",
    "topK": 5,
    "sources": ["memory", "sessions"]  // 搜索范围
  }
}

// Response
{
  "results": [
    {
      "path": "sessions/telegram:123456/transcript.jsonl",
      "startLine": 145,
      "endLine": 158,
      "score": 0.87,
      "snippet": "You mentioned the project deadline is Feb 1st...",
      "source": "sessions"
    },
    // ...
  ]
}
```

#### 核心突破

**Memory 系统让 AI 从"短期健忘症"变成"长期记忆管理大师"**:
- ✅ 自动索引: 对话和 memory files 自动分块、embedding、索引
- ✅ 混合搜索: Vector (语义) + Keyword (精确匹配)
- ✅ 增量更新: 只处理变更部分，避免全量重建
- ✅ Cache 优化: Embedding dedupe cache，节省 API 成本
- ✅ 可查询: Agent 可以主动搜索历史对话和知识库
- ✅ 跨 session: 所有 sessions 的对话都进入统一 memory index

**与其他项目对比**:

| 项目 | Memory 策略 | 问题 |
|------|------------|------|
| ChatGPT | 对话历史存储在 cloud | 隐私问题，无法本地查询 |
| Langchain Memory | 手动管理 memory stores | 需要开发者显式调用 |
| **OpenClaw** | **自动索引 + 混合搜索** | **零配置，自动整理** |

### 8.3 Skills 持续学习: AI 主动发现和学习新技能

**核心理念**: AI 不仅使用预定义的技能，还可以**主动发现、安装、更新**新技能

#### 设计精髓

传统 AI 工具是**静态的**：开发者预定义工具 → 用户使用固定工具集

OpenClaw Skills 是**动态的**：ClawdHub registry → Agent 主动搜索技能 → 安装 → 学习 → 使用

**Skills 三层加载机制** (来自 `docs/tools/skills.md:11-21`):

```
1) Bundled skills: 内置 58 个 skills (npm package 自带)
2) Managed/local skills: ~/.openclaw/skills (跨 agent 共享)
3) Workspace skills: <workspace>/skills (per-agent 定制)

Precedence (优先级):
<workspace>/skills (highest) → ~/.openclaw/skills → bundled skills (lowest)
```

**Skills 架构**:

```
┌─────────────────────────────────────────────────────────────────┐
│ ClawdHub Registry (https://clawdhub.com)                        │
│  - 公开 skills 仓库                                             │
│  - 搜索、浏览、安装、更新                                       │
│  - 社区贡献的 skills                                            │
└──────────┬──────────────────────────────────────────────────────┘
           │ 1. Agent 搜索 skills
           ↓
┌─────────────────────────────────────────────────────────────────┐
│ ClawdHub CLI (src/infra/skills-remote.ts)                      │
│  - clawdhub search <query>                                      │
│  - clawdhub install <skill-slug>                                │
│  - clawdhub update --all                                        │
│  - clawdhub sync --all                                          │
└──────────┬──────────────────────────────────────────────────────┘
           │ 2. 安装到 workspace
           ↓
┌─────────────────────────────────────────────────────────────────┐
│ Workspace Skills (~/.openclaw/workspace/skills/)                │
│  - <skill-name>/                                                │
│    ├── SKILL.md (frontmatter + instructions)                    │
│    ├── scripts/ (optional)                                      │
│    └── assets/ (optional)                                       │
└──────────┬──────────────────────────────────────────────────────┘
           │ 3. 加载到 agent runtime
           ↓
┌─────────────────────────────────────────────────────────────────┐
│ Skills Loader (src/agents/skills/refresh.ts)                   │
│  - 扫描 bundled + managed + workspace skills                    │
│  - 解析 SKILL.md frontmatter                                    │
│  - 注入到 system prompt                                         │
│  - 注册 slash commands (user-invocable skills)                  │
└──────────┬──────────────────────────────────────────────────────┘
           │ 4. Agent 使用 skills
           ↓
┌─────────────────────────────────────────────────────────────────┐
│ Agent Runtime                                                    │
│  - System prompt 包含所有 skills 的 instructions                │
│  - Agent 根据任务选择合适的 skill                                │
│  - 执行 skill 中的工具调用                                       │
└─────────────────────────────────────────────────────────────────┘
```

**SKILL.md 格式** (来自 `docs/tools/skills.md:77-100`):

```markdown
---
name: nano-banana-pro
description: Generate or edit images via Gemini 3 Pro Image
homepage: https://example.com
user-invocable: true
disable-model-invocation: false
command-dispatch: tool
command-tool: gemini_image
---

# Instructions

Use this skill to generate or edit images.

## Usage

1. Call the `gemini_image` tool with prompt
2. Tool returns image URL
3. Present image to user

## Examples

- User: "Generate a banana"
- Tool call: {"name": "gemini_image", "params": {"prompt": "a yellow banana"}}
```

**Skills 自动刷新** (来自 `src/agents/skills/refresh.ts`):

```typescript
// 注册 skills 变更监听器
export function registerSkillsChangeListener(callback: () => void) {
  // 1. 监控 workspace/skills/ 目录 (chokidar)
  // 2. 监控 ~/.openclaw/skills/ 目录
  // 3. 检测 SKILL.md 变更 (add, change, unlink)
  // 4. Debounce: 避免频繁重新加载
  // 5. 触发回调: 重新加载 skills → 刷新 system prompt
}
```

**ClawdHub 使用示例**:

```bash
# 搜索 skills
clawdhub search "image generation"

# 安装 skill
clawdhub install nano-banana-pro

# 更新所有 skills
clawdhub update --all

# 同步 (发布本地修改到 ClawdHub)
clawdhub sync --all
```

**Agent 可以主动学习新 skills** (通过 skill_search 和 skill_install tools):

```typescript
// Agent 主动搜索 skills
{
  "name": "skill_search",
  "params": {
    "query": "image generation",
    "limit": 5
  }
}

// Agent 主动安装 skill
{
  "name": "skill_install",
  "params": {
    "slug": "nano-banana-pro",
    "workspace": true  // 安装到 workspace
  }
}

// 下次 session: 新 skill 自动加载到 system prompt
```

#### 核心突破

**Skills 系统让 AI 从"固定工具箱"变成"持续学习者"**:
- ✅ 动态发现: Agent 可以主动搜索 ClawdHub registry
- ✅ 自助安装: Agent 可以安装新 skills (经用户同意)
- ✅ 自动刷新: Skills 变更自动重新加载
- ✅ 社区生态: ClawdHub 公开 registry，社区贡献
- ✅ 分层覆盖: Workspace → Managed → Bundled 三层机制
- ✅ 可扩展: Plugin skills (plugins 可以自带 skills)

**与其他项目对比**:

| 项目 | Skills 管理 | 问题 |
|------|------------|------|
| LangChain | 手动注册 Tools | 需要开发者修改代码 |
| OpenAI GPT | 预定义 functions | 无法动态添加 |
| **OpenClaw** | **ClawdHub + 自动加载** | **Agent 可自主学习** |

### 8.4 三大特性的协同效应

这三个特性不是孤立的，而是**协同工作**，形成 AI 的"主动性闭环"：

```
┌────────────────────────────────────────────────────────────────┐
│                    AI 主动性闭环                                │
└────────────────────────────────────────────────────────────────┘

1. Heartbeat: AI 定期醒来
   ↓
2. Memory Search: 检查历史对话和知识库
   - "上次用户提到什么需求？"
   - "有哪些待办事项？"
   ↓
3. Skills Search: 发现需要的新技能
   - "需要图片生成能力？搜索 image generation skill"
   - "需要数据分析？搜索 data analysis skill"
   ↓
4. Skills Install: 安装新技能
   - "安装 nano-banana-pro skill"
   ↓
5. Learn & Use: 学习并使用新技能
   - System prompt 更新，包含新 skill instructions
   - Agent 使用新 skill 完成任务
   ↓
6. Memory Update: 整理新知识
   - 对话记录自动索引到 memory
   - 下次 heartbeat 可以检索
   ↓
7. Heartbeat 继续...
```

**实际场景示例**:

```
Day 1:
- User: "明天帮我生成一张项目封面图"
- Agent: 记录到 memory (TODO: generate project cover image)

Day 2, 9:00 AM (Heartbeat):
- Agent 醒来 → 读取 HEARTBEAT.md checklist
- Memory search: 发现 "TODO: generate project cover image"
- Agent: "需要图片生成能力"
- Skill search: 搜索 ClawdHub
- 发现: "nano-banana-pro" (Gemini 3 Pro Image)
- Skill install: 安装到 workspace
- System prompt 刷新 (包含新 skill)
- Agent: "生成项目封面图 [使用 nano-banana-pro skill]"
- Memory update: 索引这次对话

Day 3, 9:00 AM (Heartbeat):
- Agent 醒来 → 检查
- Memory search: "项目封面图已生成"
- Agent: 一切正常 → 返回 HEARTBEAT_OK (静默)
```

**核心洞察**:

OpenClaw 的三大特性构成了一个**自主学习和主动服务的闭环**：

| 特性 | 作用 | 传统 AI 的问题 | OpenClaw 的突破 |
|------|-----|--------------|----------------|
| **Heartbeat** | 主动察觉 | 被动等待用户消息 | AI 定期"醒来"检查 |
| **Memory 自我整理** | 长期记忆 | 超出上下文窗口就丢失 | 自动索引，可语义搜索 |
| **Skills 持续学习** | 能力扩展 | 固定工具集 | 主动发现、安装、学习新技能 |

**这是 OpenClaw 最核心的价值主张**: 

> 从"被动响应的工具"进化成"主动关心、持续学习、自我优化的助手"

## 九、总结与洞察

### 8.1 架构核心洞察

OpenClaw 项目通过 **Local-First Gateway + WebSocket 统一协议 + Channel Plugin 抽象** 三大支柱，构建了一个优雅的 Personal AI Assistant 架构：

1. **Local-First 理念**: 数据隐私、低延迟、离线可用
2. **WebSocket 统一协议**: 双向通信、事件推送、流式响应
3. **One Gateway Per Host**: Baileys session 唯一性、状态一致性
4. **Channel Plugin 抽象**: 10+ 平台统一接口，可插拔扩展
5. **Session-Based Isolation**: 多会话隔离，安全边界清晰
6. **Embedded Agent**: 低延迟、简化部署、统一生命周期
7. **Docker Sandbox**: 非 main session 强隔离，防止 prompt injection

### 8.2 设计哲学

OpenClaw 的设计哲学可以总结为：

> **Simple, Secure, Self-Hosted**
> 
> - **Simple**: Local-First，单一 Gateway，Embedded Agent
> - **Secure**: Session isolation, Docker sandbox, Tool policy
> - **Self-Hosted**: 数据隐私，用户控制，可审计

这种哲学与当前主流 Cloud-First SaaS 形成鲜明对比，更适合隐私敏感的个人助手场景。

### 8.3 技术选型逻辑

OpenClaw 的技术选型遵循 **"适合场景优先于技术时髦"** 原则：

| 技术选择 | 常见选择 | OpenClaw 理由 |
|---------|---------|--------------|
| Local-First | Cloud SaaS | 个人助手，隐私优先 |
| WebSocket | HTTP/REST | 需要 Server push 和 streaming |
| Embedded Agent | Microservice | 单用户，低延迟优先 |
| Docker Sandbox | Process isolation | Group 消息不可信，需强隔离 |
| One Gateway | Load balanced | Baileys session 唯一性限制 |
| TypeBox | Zod | 运行时验证 + codegen |
| pnpm | npm | Monorepo + 性能 |

### 8.4 可借鉴性评估

**高度可借鉴**:
- ✅ **Channel Plugin Pattern**: 适用于任何多平台集成场景
- ✅ **Session-Based Isolation**: 适用于 Multi-tenant 或权限分级场景
- ✅ **类型驱动协议设计**: 适用于跨语言协作场景

**场景依赖**:
- ⚠️ **Local-First Gateway**: 仅适用于个人工具或隐私敏感场景
- ⚠️ **Embedded Agent**: 仅适用于单用户或低并发场景
- ⚠️ **One Gateway**: 受限于 Baileys session 唯一性

**不建议直接借鉴**:
- ❌ **WhatsApp Baileys 集成**: 技术风险高，官方不支持
- ❌ **macOS TCC 权限管理**: 平台特定，跨平台性差

### 8.5 架构演进推测

基于当前架构和代码活跃度，OpenClaw 可能的演进方向：

1. **多 Agent 协作**: 当前 sessions_* tools 已支持 Agent-to-Agent 通信，未来可能强化
2. **Memory 系统**: 当前 sqlite-vec 支持向量存储，未来可能集成 RAG
3. **Browser Automation 增强**: 当前 Playwright CDP，未来可能集成 Computer Use API
4. **Local LLM 支持**: 当前 optional `node-llama-cpp`，未来可能作为主要选项 (隐私优先)
5. **Plugin 生态**: 当前 20+ extensions，未来可能建立 Plugin marketplace

---

## 下一步研究方向

基于第0层的架构分析，后续研究将深入各个核心模块：

- **第1层**: System Prompt 设计 (模板系统、动态注入、Skills 发现)
  - 重点: AGENTS.md、SOUL.md、TOOLS.md 模板如何组织
  - 重点: Skills 加载和注入机制
  - 文档: `system-prompt-design.md`

- **第2层**: Agent Loop 执行流程 (Pi-embedded runtime、流式处理)
  - 重点: `runEmbeddedPiAgent` 完整执行链路
  - 重点: Tool streaming 和 block streaming 实现
  - 文档: `agent-loop-deep-dive.md`

- **第3层**: Session 管理 (多会话隔离、Agent-to-Agent 通信)
  - 重点: Session 存储和持久化
  - 重点: Compaction 机制
  - 文档: `session-management.md`

- **第4层**: Tool System (工具注册、权限控制、Sandbox)
  - 重点: `createOpenClawTools` 工具注册
  - 重点: Docker sandbox 实现细节
  - 文档: `tool-system-deep-dive.md`

- **第5层**: Gateway Architecture (WebSocket 协议、RPC 实现)
  - 重点: WebSocket frame validation
  - 重点: Event broadcasting 机制
  - 文档: `gateway-architecture.md`

---

**研究方法遵循**: [@awesome-llm-projects/AGENTS.md](../../awesome-llm-projects/AGENTS.md) - 忠于代码，不猜测，引用具体位置

**研究产出规范**:
- 所有结论基于实际代码
- 引用格式: `src/gateway/server.impl.ts:92-94`
- 不确定的地方标注 "待研究"，不编造
- 记录 commit hash 和研究日期
