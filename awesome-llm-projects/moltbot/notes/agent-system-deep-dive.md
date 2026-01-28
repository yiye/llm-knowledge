# Moltbot Agent 系统深度解析

> **研究版本**: v2026.1.27-beta.1  
> **分析日期**: 2026-01-28  
> **代码位置**: `src/agents/` (436 个 TypeScript 文件)

---

## 🎯 Agent 系统概览

Moltbot 的 Agent 系统是整个项目的**核心大脑**，负责：
- 与大语言模型（LLM）交互
- 工具调用和执行
- 会话管理和持久化
- 多模型支持和认证
- 沙箱环境隔离
- Skills 系统集成

**代码规模**: 436 个 TypeScript 文件，是项目中最复杂的模块。

---

## 📂 目录结构详解

### 整体目录树

```
src/agents/
├── pi-embedded-runner/     (30 文件) - Pi Agent 运行时核心
├── pi-embedded-helpers/    (9 文件)  - Pi Agent 辅助函数
├── pi-extensions/          (10 文件) - Pi Agent 扩展
├── tools/                  (57 文件) - Agent 工具集
├── skills/                 (11 文件) - Skills 系统
├── sandbox/                (17 文件) - 沙箱环境
├── auth-profiles/          (15 文件) - 认证配置文件
├── cli-runner/             (1 文件)  - CLI 运行器
├── schema/                 (2 文件)  - Schema 定义
├── test-helpers/           (2 文件)  - 测试辅助
└── 根目录                  (270+ 文件) - 核心逻辑和配置
```

---

## 🏗️ 核心模块详解

### 1. Pi Agent 运行时 (`pi-embedded-runner/`)

这是 Agent 的**执行引擎**，负责与 Pi Agent Core 的集成。

#### 文件结构

```
pi-embedded-runner/
├── run.ts                    - 主入口：runEmbeddedPiAgent
├── runs.ts                   - 运行管理
├── lanes.ts                  - 并发控制（Session Lane + Global Lane）
├── model.ts                  - 模型解析和选择
├── compact.ts                - 上下文压缩
├── history.ts                - 历史消息管理
├── system-prompt.ts          - 系统提示词构建
├── abort.ts                  - 中止控制
├── extensions.ts             - Pi Agent 扩展
├── logger.ts                 - 日志记录
├── run/                      - 运行时子模块
│   ├── attempt.ts            - 单次尝试执行
│   ├── params.ts             - 运行参数定义
│   ├── payloads.ts           - 负载构建
│   ├── types.ts              - 类型定义
│   └── images.ts             - 图像处理
└── 其他辅助文件
```

#### 核心函数：`runEmbeddedPiAgent`

**位置**: `src/agents/pi-embedded-runner/run.ts:70-680`

**职责**：
1. **队列管理**：通过 Session Lane 和 Global Lane 实现并发控制
2. **模型选择**：解析 provider/model，验证上下文窗口
3. **认证处理**：Auth Profile 选择和 Failover
4. **工作空间准备**：解析工作空间路径，初始化环境
5. **执行循环**：调用 `runEmbeddedAttempt` 执行实际推理
6. **错误处理**：分类错误（auth/rate_limit/context_overflow 等）
7. **重试逻辑**：支持 Model Failover 和上下文压缩重试

**关键代码片段**：

```typescript
// src/agents/pi-embedded-runner/run.ts:70-150
export async function runEmbeddedPiAgent(
  params: RunEmbeddedPiAgentParams,
): Promise<EmbeddedPiRunResult> {
  const sessionLane = resolveSessionLane(params.sessionKey?.trim() || params.sessionId);
  const globalLane = resolveGlobalLane(params.lane);
  
  return enqueueSession(() =>
    enqueueGlobal(async () => {
      // 1. 工作空间解析
      const resolvedWorkspace = resolveUserPath(params.workspaceDir);
      
      // 2. 模型选择
      const { model, authStorage, modelRegistry } = resolveModel(
        provider, modelId, agentDir, params.config
      );
      
      // 3. 上下文窗口检查
      const ctxInfo = resolveContextWindowInfo({ ... });
      
      // 4. Auth Profile 选择
      const authStore = ensureAuthProfileStore(agentDir, { ... });
      
      // 5. 执行 Agent 运行
      const result = await runEmbeddedAttempt({ ... });
      
      return result;
    })
  );
}
```

#### Lane 系统（并发控制）

**位置**: `src/agents/pi-embedded-runner/lanes.ts`

**设计目的**：防止同一 Session 的并发执行，避免竞态条件。

```
用户消息 A (Session X) ─┐
                        ├─→ Session Lane X (串行) ─┐
用户消息 B (Session X) ─┘                        ├─→ Global Lane (可选) ─→ 执行
                                                  │
用户消息 C (Session Y) ───→ Session Lane Y (串行) ─┘
```

**两层队列**：
1. **Session Lane**: 每个 Session 独立队列，保证同一会话的消息串行处理
2. **Global Lane**: 全局队列（可选），进一步限制并发

---

### 2. Pi Agent 辅助函数 (`pi-embedded-helpers/`)

提供 Pi Agent 运行时的辅助功能。

#### 文件列表

```
pi-embedded-helpers/
├── bootstrap.ts       - Bootstrap 文件处理（AGENTS.md, SOUL.md 等）
├── errors.ts          - 错误分类和处理
├── google.ts          - Google Gemini 特定逻辑
├── openai.ts          - OpenAI 特定逻辑
├── images.ts          - 图像处理（尺寸、格式等）
├── messaging-dedupe.ts - 消息去重
├── thinking.ts        - Thinking/Reasoning 处理
├── turns.ts           - Turn 管理
└── types.ts           - 类型定义
```

#### 核心功能：错误分类

**位置**: `src/agents/pi-embedded-helpers/errors.ts`

Moltbot 对 LLM 错误进行了细致的分类，以支持智能 Failover：

```typescript
type FailoverReason =
  | "auth"              // 认证错误
  | "rate_limit"        // 速率限制
  | "billing"           // 账单问题
  | "context_overflow"  // 上下文溢出
  | "timeout"           // 超时
  | "format"            // 格式错误
  | "unknown";          // 未知错误
```

**错误处理流程**：

```
LLM 错误
  ↓
classifyFailoverReason() - 错误分类
  ↓
┌─────────────┬─────────────┬─────────────┐
│ auth        │ rate_limit  │ context_overflow │
│ → 切换 Auth │ → 切换 Profile │ → 压缩上下文 │
│   Profile   │   (Cooldown) │   重试        │
└─────────────┴─────────────┴─────────────┘
```

---

### 3. Tools 系统 (`tools/`)

Moltbot 提供了 **57 个工具**，是 Agent 能力的基础。

#### 工具分类

##### (1) 消息和渠道工具

```
tools/
├── message-tool.ts           - 统一消息发送工具
├── telegram-actions.ts       - Telegram 特定操作
├── discord-actions*.ts       - Discord 操作（3 个文件）
├── slack-actions.ts          - Slack 操作
├── whatsapp-actions.ts       - WhatsApp 操作
└── tts-tool.ts               - TTS 语音合成
```

**`message` 工具**（位置：`src/agents/tools/message-tool.ts:326`）：

功能强大的统一消息工具，支持：
- 跨渠道消息发送
- Polls（投票）
- Reactions（反应）
- Inline Buttons（内联按钮）
- 文件附件

**示例调用**：

```json
{
  "action": "send",
  "channel": "telegram",
  "to": "+1234567890",
  "message": "Hello from Agent!",
  "buttons": [[{"text": "Yes", "callback_data": "yes"}]]
}
```

##### (2) Session 管理工具

```
tools/
├── sessions-send-tool.ts     - 跨 Session 消息发送
├── sessions-list-tool.ts     - Session 列表查询
├── sessions-history-tool.ts  - Session 历史查看
├── sessions-spawn-tool.ts    - 生成 Sub-Agent
├── session-status-tool.ts    - Session 状态查询
└── sessions-helpers.ts       - Session 辅助函数
```

**Agent-to-Agent (A2A) 通信**：

位置：`src/agents/tools/sessions-send-tool.a2a.ts`

Agent 之间可以相互发送消息，实现协作：

```
Agent A (main)
  ↓ sessions_send(sessionKey="agent-b", message="请帮我...")
Agent B (sub-agent)
  ↓ 处理任务
  ↓ sessions_send(sessionKey="main", message="完成了！")
Agent A (main)
```

##### (3) 浏览器控制工具

**位置**: `src/agents/tools/browser-tool.ts:217`

使用 Playwright 实现的浏览器控制：

```typescript
createBrowserTool({
  sandboxBridgeUrl?: string,      // 沙箱浏览器桥接 URL
  allowHostControl?: boolean,     // 是否允许控制宿主浏览器
})
```

**支持的操作**（`browser-tool.schema.ts:84`）：
- `navigate` - 导航到 URL
- `click` - 点击元素
- `type` - 输入文本
- `screenshot` - 截图
- `scroll` - 滚动
- `wait` - 等待
- `eval` - 执行 JavaScript

##### (4) Canvas 工具

**位置**: `src/agents/tools/canvas-tool.ts:52`

Agent 驱动的可视化工作空间（A2UI）：

```typescript
createCanvasTool()
```

操作包括：
- `push` - 推送 UI 内容
- `reset` - 重置 Canvas
- `snapshot` - 获取快照
- `eval` - 执行代码

##### (5) Cron 调度工具

**位置**: `src/agents/tools/cron-tool.ts:132`

Agent 可以管理定时任务：

```json
{
  "action": "add",
  "schedule": "0 9 * * *",
  "message": "每天早上 9 点提醒我"
}
```

##### (6) Node 控制工具

**位置**: `src/agents/tools/nodes-tool.ts:91`

控制移动端 Nodes（iOS/Android）：

- `camera.snap` - 拍照
- `camera.clip` - 录制视频
- `screen.record` - 屏幕录制
- `location.get` - 获取位置
- `notify` - 发送通知

##### (7) Web 工具

```
tools/
├── web-search.ts       - Web 搜索
├── web-fetch.ts        - HTTP 请求
├── web-tools.ts        - Web 工具集成
└── web-fetch-utils.ts  - 辅助函数
```

**`web.search` 工具**（位置：`src/agents/tools/web-search.ts:409`）：

集成搜索引擎，支持实时信息获取。

**`web.fetch` 工具**（位置：`src/agents/tools/web-fetch.ts:572`）：

HTTP 请求工具，带 SSRF 防护。

##### (8) 图像生成工具

**位置**: `src/agents/tools/image-tool.ts:295`

支持多种图像生成模型：
- DALL-E (OpenAI)
- Imagen (Google)
- Stable Diffusion
- Midjourney (通过扩展)

##### (9) Memory 工具

**位置**: `src/agents/tools/memory-tool.ts`

Vector Memory 系统：

```typescript
createMemorySearchTool()  // 搜索记忆
createMemoryGetTool()     // 获取记忆片段
```

使用 `sqlite-vec` 实现向量搜索。

##### (10) Gateway 管理工具

**位置**: `src/agents/tools/gateway-tool.ts:61`

Agent 可以管理 Gateway 本身：

```json
{
  "action": "reload",  // 重新加载配置
  "action": "status",  // 查看状态
  "action": "restart"  // 重启 Gateway
}
```

#### 工具创建工厂

**位置**: `src/agents/moltbot-tools.ts:22-162`

所有工具通过 `createMoltbotTools()` 函数统一创建：

```typescript
export function createMoltbotTools(options?: {
  sandboxBrowserBridgeUrl?: string;
  allowHostBrowserControl?: boolean;
  agentSessionKey?: string;
  agentChannel?: GatewayMessageChannel;
  workspaceDir?: string;
  sandboxed?: boolean;
  config?: MoltbotConfig;
  // ... 更多选项
}): AnyAgentTool[]
```

**返回的工具列表包括**：
- Browser Tool
- Canvas Tool
- Nodes Tool
- Cron Tool
- Message Tool
- TTS Tool
- Gateway Tool
- Sessions Tools
- Web Tools
- Image Tool
- Memory Tools
- 等等

#### 工具策略（Tool Policy）

**位置**: `src/agents/tool-policy.ts`

工具可以通过策略进行控制：

```typescript
type ToolProfileId = "minimal" | "coding" | "messaging" | "full";
```

- **minimal**: 只读工具（read, grep, find, ls）
- **coding**: 编码工具（+ write, edit, bash）
- **messaging**: 消息工具（+ message, sessions_send）
- **full**: 所有工具

---

### 4. Skills 系统 (`skills/`)

Skills 是 Moltbot 的**插件系统**，允许用户扩展 Agent 能力。

#### 文件结构

```
skills/
├── types.ts           - Skills 类型定义
├── config.ts          - Skills 配置
├── workspace.ts       - Workspace Skills 加载
├── bundled-dir.ts     - Bundled Skills 目录
├── plugin-skills.ts   - Plugin Skills
├── refresh.ts         - Skills 热重载
├── serialize.ts       - Skills 序列化
├── frontmatter.ts     - Frontmatter 解析
├── env-overrides.ts   - 环境变量覆盖
└── 相关测试文件
```

#### Skill 类型

根据 `src/agents/skills/types.ts:66-88`：

```typescript
type SkillEntry = {
  skill: Skill;                          // Skill 定义（来自 pi-coding-agent）
  frontmatter: ParsedSkillFrontmatter;   // YAML Frontmatter
  metadata?: MoltbotSkillMetadata;       // Moltbot 扩展元数据
  invocation?: SkillInvocationPolicy;    // 调用策略
};

type MoltbotSkillMetadata = {
  always?: boolean;           // 总是加载
  skillKey?: string;          // Skill 标识
  primaryEnv?: string;        // 主要环境变量
  emoji?: string;             // 图标
  homepage?: string;          // 主页
  os?: string[];              // 支持的操作系统
  requires?: {                // 依赖要求
    bins?: string[];          // 需要的二进制文件
    anyBins?: string[];       // 任一二进制文件
    env?: string[];           // 环境变量
    config?: string[];        // 配置项
  };
  install?: SkillInstallSpec[]; // 安装规范
};
```

#### Skill 加载顺序

根据 `docs/concepts/agent.md:54-60`：

```
1. Bundled Skills     (内置，随安装包)
   ↓
2. Managed Skills     (~/.clawdbot/skills)
   ↓
3. Workspace Skills   (<workspace>/skills)
   ↓ (优先级最高，可覆盖)
最终加载的 Skills
```

#### Skill 安装

Skills 可以自动安装依赖：

```yaml
---
install:
  - kind: brew
    formula: tmux
  - kind: node
    package: typescript
  - kind: go
    module: github.com/example/tool
---
```

支持的安装方式：
- `brew` - Homebrew
- `node` - npm/pnpm/yarn/bun
- `go` - Go modules
- `uv` - Python uv
- `download` - 直接下载

#### Skills 命令

Skills 可以注册自定义斜杠命令：

```typescript
type SkillCommandSpec = {
  name: string;           // 命令名称（如 "/review"）
  skillName: string;      // 所属 Skill
  description: string;    // 描述
  dispatch?: {            // 调度方式
    kind: "tool";
    toolName: string;
    argMode?: "raw";
  };
};
```

---

### 5. 沙箱系统 (`sandbox/`)

沙箱提供**隔离的执行环境**，防止恶意代码影响宿主系统。

#### 文件结构

```
sandbox/
├── types.ts           - 沙箱类型定义
├── types.docker.ts    - Docker 配置
├── config.ts          - 沙箱配置
├── context.ts         - 沙箱上下文
├── docker.ts          - Docker 容器管理
├── workspace.ts       - 工作空间管理
├── tool-policy.ts     - 工具策略
├── browser.ts         - 浏览器沙箱
├── browser-bridges.ts - 浏览器桥接
├── runtime-status.ts  - 运行时状态
├── registry.ts        - 沙箱注册表
├── prune.ts           - 容器清理
├── manage.ts          - 沙箱管理
└── 其他文件
```

#### 沙箱类型

根据 `src/agents/sandbox/types.ts:49-60`：

```typescript
type SandboxScope = "session" | "agent" | "shared";

type SandboxConfig = {
  mode: "off" | "non-main" | "all";  // 启用模式
  scope: SandboxScope;                // 沙箱范围
  workspaceAccess: "none" | "ro" | "rw"; // 工作空间访问权限
  workspaceRoot: string;              // 沙箱工作空间根目录
  docker: SandboxDockerConfig;        // Docker 配置
  browser: SandboxBrowserConfig;      // 浏览器配置
  tools: SandboxToolPolicy;           // 工具策略
  prune: SandboxPruneConfig;          // 清理配置
};
```

#### 沙箱模式

1. **`mode: "off"`**: 完全关闭沙箱（直接在宿主运行）
2. **`mode: "non-main"`**: 仅非主 Session 使用沙箱
3. **`mode: "all"`**: 所有 Session 都使用沙箱

#### 沙箱隔离

```
Host System
  │
  ├─ Docker Container (沙箱)
  │   ├─ Agent Workspace (独立文件系统)
  │   ├─ Tool Execution (受限工具集)
  │   ├─ Browser (Chromium in Docker)
  │   └─ Network (可限制)
  │
  └─ Gateway (控制平面)
```

#### 工作空间访问

- **`none`**: 沙箱完全隔离，无法访问宿主工作空间
- **`ro`**: 只读挂载宿主工作空间
- **`rw`**: 读写挂载宿主工作空间

#### 浏览器沙箱

根据 `src/agents/sandbox/types.ts:30-42`：

```typescript
type SandboxBrowserConfig = {
  enabled: boolean;
  image: string;                // Docker 镜像
  containerPrefix: string;      // 容器名前缀
  cdpPort: number;              // Chrome DevTools Protocol 端口
  vncPort: number;              // VNC 端口（远程查看）
  noVncPort: number;            // NoVNC Web 端口
  headless: boolean;            // 无头模式
  enableNoVnc: boolean;         // 启用 NoVNC
  allowHostControl: boolean;    // 允许控制宿主浏览器
  autoStart: boolean;           // 自动启动
  autoStartTimeoutMs: number;   // 启动超时
};
```

---

### 6. 认证系统 (`auth-profiles/`)

Moltbot 支持多个 LLM 提供商的认证，并实现了智能 Failover。

#### 文件结构

```
auth-profiles/
├── types.ts          - 认证类型定义
├── oauth.ts          - OAuth 认证流程
├── order.ts          - Auth Profile 排序
├── cooldown.ts       - Cooldown 机制
├── failover.ts       - Failover 逻辑
├── store.ts          - Auth Profile 存储
└── 相关测试文件
```

#### 认证类型

根据 `src/agents/auth-profiles/types.ts:5-32`：

```typescript
type ApiKeyCredential = {
  type: "api_key";
  provider: string;
  key: string;
  email?: string;
};

type TokenCredential = {
  type: "token";
  provider: string;
  token: string;
  expires?: number;
  email?: string;
};

type OAuthCredential = OAuthCredentials & {
  type: "oauth";
  provider: string;
  clientId?: string;
  email?: string;
};

type AuthProfileCredential = 
  | ApiKeyCredential 
  | TokenCredential 
  | OAuthCredential;
```

#### OAuth 认证流程

**位置**: `src/agents/auth-profiles/oauth.ts`

Moltbot 支持 Anthropic 和 OpenAI 的 OAuth 认证（Pro/Max 订阅）：

```
1. 用户启动 OAuth 流程
   ↓
2. 本地启动临时 HTTP 服务器监听回调
   ↓
3. 打开浏览器进行 OAuth 授权
   ↓
4. 获取 Access Token 和 Refresh Token
   ↓
5. 保存到 Auth Profile Store
   ↓
6. 自动刷新 Token（到期前）
```

#### Auth Profile 排序和选择

**位置**: `src/agents/auth-profiles/order.ts`

Moltbot 支持多个 Auth Profile，并智能选择：

```typescript
function resolveAuthProfileOrder(params: {
  provider: string;
  configOrder?: string[];      // 配置指定的顺序
  storeOrder?: string[];       // 存储中的顺序
  lastGood?: string;           // 上次成功的 Profile
  usageStats?: ProfileUsageStats[]; // 使用统计
}): string[]
```

**选择策略**：
1. 优先使用配置指定的顺序
2. 其次使用存储中的顺序
3. 考虑上次成功的 Profile
4. 考虑使用频率（Round-Robin）
5. 跳过 Cooldown 中的 Profile

#### Failover 机制

当一个 Auth Profile 失败时：

```
Profile A (失败: auth error)
  ↓
标记失败 (cooldown 5 分钟)
  ↓
尝试 Profile B
  ↓
成功 → 标记 Profile B 为 lastGood
失败 → 继续尝试 Profile C
```

#### Cooldown 机制

根据失败原因，设置不同的 Cooldown 时间：

```typescript
type AuthProfileFailureReason =
  | "auth"        // → Cooldown 10 分钟
  | "rate_limit"  // → Cooldown 60 分钟
  | "billing"     // → Cooldown 24 小时
  | "timeout"     // → Cooldown 5 分钟
  | "format"      // → Cooldown 1 分钟
  | "unknown";    // → Cooldown 5 分钟
```

---

### 7. 系统提示词（System Prompt）

**位置**: `src/agents/system-prompt.ts`

Moltbot 动态构建系统提示词，包含多个部分。

#### 提示词模式

根据 `src/agents/system-prompt.ts:7-13`：

```typescript
type PromptMode = 
  | "full"     // 完整提示词（主 Agent）
  | "minimal"  // 精简提示词（Sub-Agent）
  | "none";    // 极简提示词
```

#### 系统提示词结构

```
## Agent Identity
[Agent 身份和角色]

## Skills (mandatory)
[可用 Skills 列表和使用指南]

## Memory Recall
[Memory 系统使用说明]

## User Identity
[用户身份信息]

## Current Date & Time
[当前时间和时区]

## Reply Tags
[回复标签说明]

## Messaging
[消息发送指南]

## Multi-Session Delivery
[跨 Session 消息]

## Sessions
[Session 管理]

## Cron
[定时任务]

## Canvas
[Canvas 使用]

## Browser
[浏览器控制]

## Nodes
[移动端 Node 控制]

## Tooling
[工具使用指南]

## Workspace
[工作空间规则]

## Runtime
[运行时信息]

## Bootstrap Files
[AGENTS.md, SOUL.md 等注入的文件内容]
```

#### Bootstrap 文件

**位置**: `docs/concepts/agent.md:22-38`

系统会自动注入以下文件到上下文：

1. **`AGENTS.md`** - 操作指令和"记忆"
2. **`SOUL.md`** - 人格、边界、语气
3. **`TOOLS.md`** - 工具使用说明
4. **`BOOTSTRAP.md`** - 首次运行仪式（完成后删除）
5. **`IDENTITY.md`** - Agent 名称/风格/emoji
6. **`USER.md`** - 用户资料和偏好称呼

---

### 8. 模型管理

#### 模型选择

**位置**: `src/agents/model-selection.ts`, `src/agents/model-compat.ts`

Moltbot 维护了一个模型注册表，包含：
- Token 成本
- 上下文窗口大小
- 支持的能力（vision, thinking, tool calling）

#### 模型配置

**位置**: `src/agents/models-config.ts`

支持自定义模型配置：

```json5
{
  "models": {
    "definitions": [
      {
        "id": "my-local-llama",
        "provider": "openai-compatible",
        "baseUrl": "http://localhost:8000/v1",
        "contextWindow": 32000,
        "cost": { "input": 0, "output": 0 }
      }
    ]
  }
}
```

#### Failover 配置

**位置**: `src/agents/failover-error.ts`

支持模型 Failover：

```json5
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "anthropic/claude-sonnet-4-5",
        "fallbacks": [
          "openai/gpt-5.1-codex",
          "google/gemini-2.5-flash"
        ]
      }
    }
  }
}
```

当主模型失败时，自动切换到备用模型。

---

### 9. 会话管理

#### Session 类型

根据 `src/agents/tools/sessions-helpers.ts:11-32`：

```typescript
type SessionKind = 
  | "main"    // 主会话（直接对话）
  | "group"   // 群组会话
  | "cron"    // Cron 触发的会话
  | "hook"    // Hook 触发的会话
  | "node"    // Node 触发的会话
  | "other";  // 其他
```

#### Session 存储

会话记录存储在：

```
~/.clawdbot/agents/<agentId>/sessions/<SessionId>.jsonl
```

使用 JSONL 格式，每行一条消息。

#### Session 写锁

**位置**: `src/agents/session-write-lock.ts`

防止并发写入导致数据损坏：

```typescript
class SessionWriteLock {
  async acquire(sessionId: string): Promise<void>;
  async release(sessionId: string): Promise<void>;
}
```

---

### 10. 上下文压缩（Compaction）

**位置**: `src/agents/pi-embedded-runner/compact.ts`

当上下文超过限制时，Moltbot 会自动压缩历史消息。

#### 压缩策略

```
完整历史: [Msg1, Msg2, Msg3, ..., Msg100]
  ↓
保留最近 N 条
  ↓
[Summary of Msg1-80] + [Msg81, Msg82, ..., Msg100]
```

#### 压缩触发

1. **主动压缩**: 上下文即将溢出时
2. **被动压缩**: 收到 context_overflow 错误后

---

## 🔄 Agent 执行流程

### 完整执行流程图

```
用户消息
  ↓
Gateway 接收 (WebSocket)
  ↓
路由到 Agent Session
  ↓
进入 Session Lane (队列)
  ↓
┌─────────────────────────────────────┐
│ runEmbeddedPiAgent                  │
│                                     │
│ 1. 解析模型和认证                   │
│ 2. 准备工作空间                     │
│ 3. 加载 Skills                      │
│ 4. 构建系统提示词                   │
│ 5. 加载历史消息                     │
│ 6. 检查上下文窗口                   │
│    ├─ 需要压缩? → compactSession    │
│    └─ OK                            │
│ 7. 调用 runEmbeddedAttempt          │
│    ├─ 构建请求                      │
│    ├─ 调用 LLM API                  │
│    ├─ 流式接收响应                  │
│    ├─ 工具调用?                     │
│    │   ├─ 执行工具                  │
│    │   └─ 继续 LLM 推理             │
│    └─ 返回最终响应                  │
│ 8. 处理错误                         │
│    ├─ Auth 错误? → 切换 Profile     │
│    ├─ Rate Limit? → Cooldown        │
│    ├─ Context Overflow? → 压缩重试  │
│    └─ 其他错误? → Failover 或报错   │
│ 9. 保存会话记录                     │
│ 10. 返回结果                        │
└─────────────────────────────────────┘
  ↓
格式化响应
  ↓
通过 Gateway 发送到用户
  ↓
渠道 Plugin 转换格式
  ↓
发送到消息平台 (WhatsApp/Telegram/etc.)
```

---

## 🎨 设计模式和架构亮点

### 1. 插件化设计

**工具、Skills、Channels** 都采用插件模式：
- 统一接口
- 动态加载
- 可扩展

### 2. 分层架构

```
表现层: Tools (57 个工具)
  ↓
业务层: Pi Agent Runner (执行引擎)
  ↓
集成层: Pi Agent Core (LLM API)
  ↓
基础设施层: Auth, Sandbox, Config
```

### 3. 错误处理和重试

**分类错误 → 智能重试**：
- Auth 错误 → 切换 Profile
- Rate Limit → Cooldown
- Context Overflow → 压缩
- Unknown → Failover

### 4. 并发控制

**Lane 系统**：
- Session Lane: 会话级串行
- Global Lane: 全局并发控制

### 5. 沙箱隔离

**Docker 容器**：
- 文件系统隔离
- 网络隔离
- 工具权限控制

---

## 📊 关键数据结构

### Agent 运行参数

```typescript
type RunEmbeddedPiAgentParams = {
  sessionId: string;
  sessionKey?: string;
  provider: string;
  model: string;
  message?: string;
  workspaceDir: string;
  agentDir?: string;
  config?: MoltbotConfig;
  authProfileId?: string;
  skillsPrompt?: string;
  tools: AnyAgentTool[];
  messageChannel?: string;
  lane?: string;
  timeoutSeconds?: number;
  // ... 更多参数
};
```

### Agent 运行结果

```typescript
type EmbeddedPiRunResult = {
  payloads: Payload[];          // 响应内容
  usage: UsageLike;             // Token 使用量
  cost?: number;                // 成本
  model: string;                // 使用的模型
  provider: string;             // 提供商
  stopReason: string;           // 停止原因
  authProfileId?: string;       // 使用的 Auth Profile
  compactionCount?: number;     // 压缩次数
  // ... 更多字段
};
```

---

## 🔧 扩展点

### 1. 添加新工具

在 `src/agents/tools/` 创建新文件：

```typescript
import { Type } from "@sinclair/typebox";
import type { AnyAgentTool } from "./common.js";

export function createMyNewTool(): AnyAgentTool {
  return {
    name: "my_tool",
    description: "My awesome tool",
    parameters: Type.Object({
      param1: Type.String(),
    }),
    execute: async (toolCallId, args) => {
      // 实现逻辑
      return {
        output: "Result",
        details: { ... },
      };
    },
  };
}
```

然后在 `moltbot-tools.ts` 中注册。

### 2. 添加新 Skill

在 `skills/my-skill/` 创建 `SKILL.md`：

```markdown
---
description: My skill description
emoji: 🎯
---

# My Skill

This skill does...
```

### 3. 添加新的 LLM Provider

在 `src/agents/pi-ai/` 扩展 Pi AI：

```typescript
import { getModel } from "@mariozechner/pi-ai";

const myModel = getModel("my-provider", "my-model");
```

或自定义模型定义：

```typescript
const customModel: Model<'openai-completions'> = {
  id: "my-model",
  name: "My Model",
  api: "openai-completions",
  provider: "my-provider",
  baseUrl: "https://api.my-provider.com/v1",
  // ...
};
```

---

## 📈 性能考虑

### 1. Session Lane 队列

避免同一 Session 的并发执行，但不同 Session 可并行。

### 2. Skills 缓存

Skills 加载后会缓存，避免重复解析。

### 3. 模型注册表缓存

模型信息本地缓存，减少网络请求。

### 4. Auth Profile Cooldown

失败的 Profile 进入 Cooldown，避免重复尝试。

---

## 🐛 调试和日志

### 日志系统

**位置**: `src/agents/pi-embedded-runner/logger.ts`

使用 `tslog` 提供结构化日志：

```typescript
const log = createSubsystemLogger("agent");
log.info("Agent started");
log.error("Failed to execute tool", { error });
```

### 调试技巧

1. **查看 Session JSONL**:
   ```bash
   cat ~/.clawdbot/agents/default/sessions/<sessionId>.jsonl
   ```

2. **启用详细日志**:
   ```bash
   CLAWDBOT_VERBOSE=1 moltbot gateway
   ```

3. **检查 Auth Profile**:
   ```bash
   cat ~/.clawdbot/agents/default/auth-profiles.json
   ```

---

## 📚 相关文档

- **Agent Runtime**: `docs/concepts/agent.md`
- **Agent Loop**: `docs/concepts/agent-loop.md`
- **System Prompt**: `docs/concepts/system-prompt.md`
- **Tools**: `docs/tools/`
- **Skills**: `docs/tools/skills.md`

---

## 💡 核心洞察

1. **Pi Agent 集成是轻量级的**：Moltbot 只使用了 Pi Agent Core 的模型和工具部分，Session 管理完全自己实现。

2. **工具系统是核心竞争力**：57 个工具覆盖了消息、浏览器、Canvas、Cron、Nodes 等各个方面。

3. **认证系统很强大**：支持多 Profile、OAuth、智能 Failover、Cooldown 机制。

4. **沙箱设计很周到**：Docker 隔离 + 工具策略 + 工作空间权限控制。

5. **Skills 系统很灵活**：三层加载（Bundled/Managed/Workspace）+ 自动安装依赖。

6. **错误处理很细致**：分类错误、智能重试、上下文压缩、模型 Failover。

7. **并发控制很巧妙**：Lane 系统保证 Session 串行，避免竞态条件。

---

**总结**：Moltbot 的 Agent 系统是一个**设计精良、功能丰富、可扩展性强**的 AI Agent 运行时。它不仅集成了 Pi Agent Core，还在此基础上构建了完善的工具系统、Skills 系统、沙箱系统和认证系统，是一个非常值得学习的工程实践案例。✨
