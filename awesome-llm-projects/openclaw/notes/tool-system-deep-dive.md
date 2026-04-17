# OpenClaw Tool 系统深度剖析

## 📌 版本信息

- **Commit**: `73055728318df378c831950cd01fb7c875a33790` → 最新代码分析（≥ 2026.3.22）
- **版本**: 2026.3.3（2026-03-24 更新）
- **研究日期**: 2026-03-05（2026-03-24 依据最新代码更新）
- **注意**: 该分析基于 2026 年 3 月最新代码

---

## 🎯 核心设计理念

OpenClaw 的 Tool 系统是一个**高度模块化、多层权限控制、支持跨设备调用**的工具框架，核心目标：

1. **统一抽象**：所有工具实现统一的 `AnyAgentTool` 接口（`AgentTool<any, unknown>` 类型别名）
2. **细粒度权限**：通过多层 Policy（Global → Agent → Provider → Sandbox → Group）精确控制工具访问
3. **安全隔离**：Docker Sandbox 隔离非 main session 的工具执行
4. **跨设备调用**：通过 `node.invoke` 实现 macOS/iOS/Android 远程工具调用
5. **插件扩展**：支持第三方插件动态注册工具
6. **Provider 适配**：自动处理不同 LLM Provider（OpenAI/Claude/Gemini）的 Schema 兼容性

---

## 1. 工具注册机制（统一入口）

### 1.1 核心注册函数：`createOpenClawTools`

**位置**：`src/agents/openclaw-tools.ts:22-162`

```typescript
export function createOpenClawTools(options?: {
  sandboxBrowserBridgeUrl?: string;
  allowHostBrowserControl?: boolean;
  agentSessionKey?: string;
  agentChannel?: GatewayMessageChannel;
  agentAccountId?: string;
  agentTo?: string;
  agentThreadId?: string | number;
  agentGroupId?: string | null;
  agentGroupChannel?: string | null;
  agentGroupSpace?: string | null;
  agentDir?: string;
  sandboxRoot?: string;
  workspaceDir?: string;
  sandboxed?: boolean;
  config?: OpenClawConfig;
  pluginToolAllowlist?: string[];
  currentChannelId?: string;
  currentThreadTs?: string;
  replyToMode?: "off" | "first" | "all";
  hasRepliedRef?: { value: boolean };
  modelHasVision?: boolean;
  requesterAgentIdOverride?: string;
}): AnyAgentTool[]
```

**注册流程**：

1. **内置工具注册**（按固定顺序）：
   - `createBrowserTool()` - Browser 控制（CDP）
   - `createCanvasTool()` - Canvas + A2UI
   - `createNodesTool()` - macOS/iOS/Android nodes
   - `createCronTool()` - Cron 任务管理
   - `createMessageTool()` - 消息发送（多平台，可选）
   - `createTtsTool()` - TTS 语音合成
   - `createImageGenerateTool()` - 图片生成（可选，`image_generate`）⭐ 新增
   - `createGatewayTool()` - Gateway 管理（restart/config/update）
   - `createAgentsListTool()` - Agent 列表（`agents_list`）
   - `createSessionsListTool()` - Session 列表
   - `createSessionsHistoryTool()` - Session 历史
   - `createSessionsSendTool()` - Agent 间通信（`sessions_send`）
   - `createSessionsYieldTool()` - 长等待让步信号（`sessions_yield`）⭐ 新增
   - `createSessionsSpawnTool()` - Subagent 生成（`sessions_spawn`）
   - `createSubagentsTool()` - Subagent 管理（`subagents`，list/steer/kill）⭐ 新增
   - `createSessionStatusTool()` - Session 状态（`session_status`）
   - `createWebSearchTool()` / `createWebFetchTool()` - Web 工具（可选）
   - `createImageTool()` - 图片分析（可选，依赖 imageModel 配置）
   - `createPdfTool()` - PDF 分析（可选，依赖 agentDir + pdfModel 配置）

2. **插件工具加载**：
   ```typescript
   const pluginTools = resolvePluginTools({
     context: { config, workspaceDir, agentDir, agentId, sessionKey, messageChannel, ... },
     existingToolNames: new Set(tools.map((tool) => tool.name)),
     toolAllowlist: options?.pluginToolAllowlist,
   });
   ```

3. **返回合并列表**：
   ```typescript
   return [...tools, ...pluginTools];
   ```

### 1.2 两阶段注册架构（⭐ 重要更新）

工具注册分为**两个独立阶段**，顺序固定：

```
阶段 1: createOpenClawCodingTools (pi-tools.ts)
├── Pi base tools (read/write/edit/grep/find/ls)
├── exec, process, apply_patch
├── listChannelAgentTools() — channel 插件工具
└── createOpenClawTools()  — OpenClaw 核心工具
    ├── 固定顺序内置工具（见 1.1 节列表）
    └── resolvePluginTools() — 插件动态工具（末尾追加）

阶段 2: 最终合并
└── [...phase1Tools]  （所有工具的有序列表）
```

**关键约束**：
- 插件工具（`resolvePluginTools`）**永远在最末尾**，避免覆盖内置工具的名称
- `existingToolNames` 去重：插件工具注册时会检查是否与已有工具名冲突
- Channel 工具（`listChannelAgentTools`）在 OpenClaw 核心工具之前注册

### 1.3 Coding Tools 注册：`createOpenClawCodingTools`

**位置**：`src/agents/pi-tools.ts:114-442`

除了 OpenClaw 特有工具外，还注册了文件系统工具（来自 `@mariozechner/pi-coding-agent`）：

- `read` - 文件读取
- `write` - 文件写入
- `edit` - 文件编辑（行范围替换）
- `apply_patch` - 多文件 patch（实验性，OpenAI 模型专属）
- `exec` - 命令执行
- `process` - 后台进程管理

**关键特性**：

1. **Sandbox 适配**：沙箱环境下使用 `createSandboxedReadTool()` / `createSandboxedWriteTool()` / `createSandboxedEditTool()` 包装工具，限制路径访问范围
2. **Provider Schema 适配**：自动调用 `normalizeToolParameters()` 处理不同 Provider 的 Schema 差异
3. **Claude 兼容性**：为 Claude 模型注入 `CLAUDE_PARAM_GROUPS` 参数组（`path_and_range`、`insert_opts`、`command_opts`）
4. **Channel-specific tools**：根据 Channel 动态加载 Discord/Slack/Telegram/WhatsApp 特定工具

### 1.3 PDF 工具：`createPdfTool`

**位置**：`src/agents/tools/pdf-tool.ts:294-426`

**工厂函数**：

```typescript
export function createPdfTool(options?: {
  config?: OpenClawConfig;
  agentDir?: string;
  workspaceDir?: string;
  sandbox?: PdfSandboxConfig;
  fsPolicy?: ToolFsPolicy;
}): AnyAgentTool | null
```

**核心能力**：

1. **Native PDF 支持**（`src/agents/tools/pdf-native-providers.ts`）：
   - **Anthropic**：通过 Messages API 直接发送 `type: "document"` + `media_type: "application/pdf"`（`anthropicAnalyzePdf`）
   - **Google**：通过 Gemini `generateContent` API 发送 `inline_data` PDF（`geminiAnalyzePdf`）
   - Provider 检测：`NATIVE_PDF_PROVIDERS = new Set(["anthropic", "google"])`（`pdf-tool.helpers.ts:17`）

2. **Extraction 回退**：非 Native Provider（如 OpenAI）时，使用 `extractPdfContent` 提取文本+图片，再送入多模态模型分析

3. **配置项**（`agents.defaults`）：
   - `pdfModel`：主模型 + fallbacks（`{ primary, fallbacks }`），未配置时回退到 `imageModel` 再回退到 provider 默认
   - `pdfMaxBytesMb`：单 PDF 最大字节数（默认 10 MB，`pdf-tool.ts:44`）
   - `pdfMaxPages`：提取模式下最大页数（默认 20，`pdf-tool.ts:45`）

4. **工具参数**：`pdf` / `pdfs`（路径或 URL）、`prompt`、`pages`（页范围，仅 extraction 模式支持）、`model`、`maxBytesMb`

**注册条件**：需 `agentDir` 且 `resolvePdfModelConfigForTool` 返回有效配置（`openclaw-tools.ts:92-93`）

---

## 2. 工具抽象层设计

### 2.1 统一接口：`AnyAgentTool`

**位置**：`src/agents/tools/common.ts:7`

```typescript
import type { AgentTool, AgentToolResult } from "@mariozechner/pi-agent-core";

// 类型别名：any = parameters schema type, unknown = execution result details
export type AnyAgentTool = AgentTool<any, unknown>;
```

**`AgentTool` 接口结构**（来自 pi-agent-core）：

```typescript
interface AgentTool<P, D> {
  label: string;              // 工具显示名称（UI 用）
  name: string;               // 工具名称（LLM 调用时使用）
  description: string;        // 工具功能描述（注入到 System Prompt）
  parameters: P;              // TypeBox schema（定义输入参数结构）
  execute: (toolCallId: string, params: Record<string, unknown>) => Promise<AgentToolResult<D>>;
}

interface AgentToolResult<D> {
  content: Array<{ type: "text"; text: string } | { type: "image"; data: string; mimeType: string }>;
  details?: D;                // 结构化数据（可选，供后续处理使用）
}
```

### 2.2 工具辅助函数：参数读取与结果构造

**位置**：`src/agents/tools/common.ts:33-244`

**参数读取**（类型安全 + 自动校验）：

```typescript
// 读取字符串参数（支持 required 重载）
readStringParam(params, key, { required: true }): string
readStringParam(params, key, options?: StringParamOptions): string | undefined

// 读取数字参数
readNumberParam(params, key, { required, label, integer }): number | undefined

// 读取字符串数组
readStringArrayParam(params, key, options): string[] | undefined

// 读取 Reaction 参数（emoji + remove flag）
readReactionParams(params, { emojiKey, removeKey, removeErrorMessage }): ReactionParams
```

**结果构造**（自动格式化）：

```typescript
// JSON 结果（自动序列化）
jsonResult(payload: unknown): AgentToolResult<unknown>

// 图片结果（自动处理 base64 + MIME type）
imageResult({ label, path, base64, mimeType, extraText, details }): Promise<AgentToolResult>

// 从文件路径读取图片结果
imageResultFromFile({ label, path, extraText, details }): Promise<AgentToolResult>
```

**关键设计**：

1. **类型安全**：通过 TypeScript 重载实现 `required` 参数时返回非 undefined 类型
2. **自动 trim**：字符串参数默认 trim，避免空格干扰
3. **图片自动压缩**：`sanitizeToolResultImages()` 自动处理图片大小（避免 Token 超限）
4. **MEDIA 标记**：图片返回时自动添加 `MEDIA:<path>` 文本标记，便于 Agent 引用

---

## 3. 多层权限控制体系 🔥

### 3.1 权限层级（优先级从高到低）

OpenClaw 实现了**6 层工具权限控制**，层层过滤：

```
┌─────────────────────────────────────────────────────────┐
│ 1. Sandbox Tool Policy (Deny wins)                     │
│    └─ tools.sandbox.tools.allow / deny                 │
│       - 沙箱环境额外限制（防止敏感工具泄露）            │
├─────────────────────────────────────────────────────────┤
│ 2. Subagent Tool Policy (Deny wins)                    │
│    └─ tools.subagents.tools.deny (default hardcoded)   │
│       - 防止 subagent 调用 session 管理工具            │
├─────────────────────────────────────────────────────────┤
│ 3. Group Tool Policy (Deny wins)                       │
│    └─ groups[id].tools.allow / deny                    │
│       - 基于 Channel Group（Discord/Slack）限制权限    │
├─────────────────────────────────────────────────────────┤
│ 4. Agent Provider Policy (Deny wins)                   │
│    └─ agents.list[].tools.byProvider[provider]         │
│       - 针对特定 Provider/Model 限制工具（如 Gemini）  │
├─────────────────────────────────────────────────────────┤
│ 5. Global Provider Policy (Deny wins)                  │
│    └─ tools.byProvider[provider]                       │
│       - 全局 Provider 特定限制                         │
├─────────────────────────────────────────────────────────┤
│ 6. Agent Tool Policy (Deny wins)                       │
│    └─ agents.list[].tools.allow / deny                 │
│       - Agent 级别工具限制                             │
├─────────────────────────────────────────────────────────┤
│ 7. Global Tool Policy (Deny wins)                      │
│    └─ tools.allow / deny                               │
│       - 全局工具限制                                   │
├─────────────────────────────────────────────────────────┤
│ 8. Tool Profile (Base allowlist)                       │
│    └─ tools.profile / agents.list[].tools.profile      │
│       - minimal / coding / messaging / full            │
└─────────────────────────────────────────────────────────┘
```

**判断逻辑**（`src/agents/pi-tools.policy.ts:334-339`）：

```typescript
export function isToolAllowedByPolicies(
  name: string,
  policies: Array<SandboxToolPolicy | undefined>,
) {
  // 所有 policy 都必须允许，才能通过（AND 逻辑）
  return policies.every((policy) => isToolAllowedByPolicyName(name, policy));
}
```

### 3.2 Policy 匹配规则

**位置**：`src/agents/pi-tools.policy.ts:11-77`

**支持三种模式**：

1. **精确匹配**：`"browser"` → 只匹配 `browser` 工具
2. **通配符匹配**：`"sessions_*"` → 匹配 `sessions_list`、`sessions_send` 等
3. **全部匹配**：`"*"` → 匹配所有工具

**判断逻辑**：

```typescript
function makeToolPolicyMatcher(policy: SandboxToolPolicy) {
  const deny = compilePatterns(policy.deny);   // 编译 deny 规则
  const allow = compilePatterns(policy.allow); // 编译 allow 规则
  
  return (name: string) => {
    const normalized = normalizeToolName(name); // 标准化工具名（小写、别名替换）
    
    // 1. Deny 优先（黑名单）
    if (matchesAny(normalized, deny)) {
      return false;
    }
    
    // 2. Allow 为空 = 允许所有（白名单模式禁用）
    if (allow.length === 0) {
      return true;
    }
    
    // 3. Allow 匹配 = 允许
    if (matchesAny(normalized, allow)) {
      return true;
    }
    
    // 4. apply_patch 特殊规则：exec 允许时自动允许 apply_patch
    if (normalized === "apply_patch" && matchesAny("exec", allow)) {
      return true;
    }
    
    return false;
  };
}
```

### 3.3 Tool Groups（工具组简写）

**位置**：`src/agents/tool-policy.ts:13-57`

为了简化配置，OpenClaw 定义了多个工具组：

```typescript
export const TOOL_GROUPS: Record<string, string[]> = {
  "group:memory": ["memory_search", "memory_get"],
  "group:web": ["web_search", "web_fetch"],
  "group:fs": ["read", "write", "edit", "apply_patch"],
  "group:runtime": ["exec", "process"],
  "group:sessions": ["sessions_list", "sessions_history", "sessions_send", "sessions_spawn", "session_status"],
  "group:ui": ["browser", "canvas"],
  "group:automation": ["cron", "gateway"],
  "group:messaging": ["message"],
  "group:nodes": ["nodes"],
  "group:openclaw": [/* 所有内置工具 */],
};
```

**配置示例**：

```json5
{
  "tools": {
    "allow": ["group:fs", "group:runtime", "browser"],
    "deny": ["exec"]  // deny 优先，即使 group:runtime 包含 exec 也会被拒绝
  }
}
```

### 3.4 Tool Profiles（预设工具集）

**位置**：`src/agents/tool-policy.ts:59-76`

```typescript
const TOOL_PROFILES: Record<ToolProfileId, ToolProfilePolicy> = {
  minimal: {
    allow: ["session_status"],  // 仅状态查询
  },
  coding: {
    allow: ["group:fs", "group:runtime", "group:sessions", "group:memory", "image"],
  },
  messaging: {
    allow: ["group:messaging", "sessions_list", "sessions_history", "sessions_send", "session_status"],
  },
  full: {},  // 无限制（等同于不设置）
};
```

**BREAKING：Onboarding 默认 profile**（`src/commands/onboard-config.ts:6,29-31`）：

- 新安装（交互式 + 非交互式）默认 `tools.profile: "messaging"`
- `applyOnboardingLocalWorkspaceConfig` 在未配置时设置 `ONBOARDING_DEFAULT_TOOLS_PROFILE = "messaging"`
- 已有显式 `tools.profile` 的配置会被保留，不会被覆盖

**配置示例**：

```json5
{
  "tools": {
    "profile": "coding",  // 基础工具集
    "deny": ["exec"]      // 在 profile 基础上额外拒绝 exec
  }
}
```

### 3.5 Provider-Specific Policy（模型特定限制）

**位置**：`src/agents/pi-tools.policy.ts:190-273`

**使用场景**：某些 Provider（如 Google Gemini）的工具调用能力较弱，需要限制工具集

**配置示例**：

```json5
{
  "tools": {
    "profile": "coding",  // 全局使用 coding profile
    "byProvider": {
      "google-antigravity": {
        "profile": "minimal"  // Gemini 模型仅使用 minimal profile
      },
      "openai/gpt-5.2": {
        "allow": ["group:fs", "sessions_list"]  // 特定模型的白名单
      }
    }
  }
}
```

**解析优先级**：`provider/model` > `provider`

```typescript
function resolveProviderToolPolicy(params: {
  byProvider?: Record<string, ToolPolicyConfig>;
  modelProvider?: string;
  modelId?: string;
}): ToolPolicyConfig | undefined {
  // 1. 尝试匹配 "provider/model"（如 "openai/gpt-5.2"）
  // 2. 回退到 "provider"（如 "openai"）
  const candidates = [...(fullModelId ? [fullModelId] : []), normalizedProvider];
  for (const key of candidates) {
    const match = lookup.get(key);
    if (match) return match;
  }
  return undefined;
}
```

### 3.6 Subagent Tool Policy（Subagent 特殊限制）

**位置**：`src/agents/pi-tools.policy.ts:79-106`

**默认拒绝列表**（防止 Subagent 干扰主会话）：

```typescript
const DEFAULT_SUBAGENT_TOOL_DENY = [
  // Session 管理 - 由主 agent 协调
  "sessions_list", "sessions_history", "sessions_send", "sessions_spawn",
  // 系统管理 - 危险操作
  "gateway", "agents_list",
  // 交互式设置 - 非任务类工具
  "whatsapp_login",
  // 状态/调度 - 主 agent 协调
  "session_status", "cron",
  // Memory - 通过 spawn prompt 传递相关信息即可
  "memory_search", "memory_get",
];
```

**可配置覆盖**：

```json5
{
  "tools": {
    "subagents": {
      "tools": {
        "deny": ["gateway", "cron"],  // 额外拒绝
        "allow": ["read", "exec"]     // 白名单模式（仅允许这些工具）
      }
    }
  }
}
```

### 3.7 Sandbox Tool Policy（沙箱额外限制）

**位置**：`src/agents/sandbox/tool-policy.ts`

**默认限制**（`src/agents/sandbox/constants.ts`）：

```typescript
export const DEFAULT_TOOL_DENY: string[] = [];  // 默认无拒绝

export const DEFAULT_TOOL_ALLOW: string[] = [
  "*",  // 默认允许所有工具（沙箱环境已提供基础隔离）
];
```

**配置路径**：

- 全局：`tools.sandbox.tools.allow / deny`
- Agent 级别：`agents.list[].tools.sandbox.tools.allow / deny`

**特殊规则**：`image` 工具自动添加到 allowlist（`src/agents/sandbox/tool-policy.ts:125-132`）

```typescript
// image 是多模态工作流必需的，除非显式拒绝，否则自动允许
if (
  !expandedDeny.includes("image") &&
  !expandedAllow.includes("image")
) {
  expandedAllow = [...expandedAllow, "image"];
}
```

---

## 4. Docker Sandbox 安全隔离 🔥

### 4.1 Sandbox 工作模式

**位置**：`docs/gateway/sandboxing.md`

**三种模式**（`agents.defaults.sandbox.mode`）：

1. **`off`**：所有工具在 Host 运行（无隔离）
2. **`non-main`**（默认推荐）：仅隔离非 main session（直接聊天在 Host，Group/Subagent 在 Sandbox）
3. **`all`**：所有 session 都在 Sandbox 运行（最安全）

### 4.2 Sandbox 作用域

**三种作用域**（`agents.defaults.sandbox.scope`）：

1. **`session`**（默认）：每个 session 独立容器（最强隔离）
2. **`agent`**：每个 agent 共享一个容器（节省资源）
3. **`shared`**：所有 session 共享一个容器（资源最优）

### 4.3 Workspace 访问模式

**三种模式**（`agents.defaults.sandbox.workspaceAccess`）：

1. **`none`**（默认）：
   - 沙箱看到独立 workspace：`~/.openclaw/sandboxes/<sandbox-id>/workspace`
   - 工具（`read`/`write`/`edit`）路径自动 root 到沙箱 workspace
   - Skills 自动镜像到沙箱：`~/.openclaw/sandboxes/<sandbox-id>/skills`

2. **`ro`**（只读挂载）：
   - Agent workspace 只读挂载到 `/agent`
   - 禁用 `write`/`edit`/`apply_patch` 工具
   - 适合只读查询场景

3. **`rw`**（读写挂载）：
   - Agent workspace 读写挂载到 `/workspace`
   - 所有文件操作直接作用于 Host workspace
   - **风险较高**：需要配合严格的 Tool Policy

### 4.4 Sandbox 配置示例

```json5
{
  "agents": {
    "defaults": {
      "sandbox": {
        "mode": "non-main",         // 仅隔离非 main session
        "scope": "session",         // 每 session 独立容器
        "workspaceAccess": "none",  // 沙箱独立 workspace
        "docker": {
          "image": "openclaw-sandbox:bookworm-slim",
          "network": "none",        // 无网络访问（默认）
          "readOnlyRoot": true,     // 容器根目录只读
          "user": "1000:1000",      // 非 root 用户
          "binds": [                // 自定义挂载点
            "/home/user/source:/source:ro",
            "/var/run/docker.sock:/var/run/docker.sock:ro"
          ],
          "setupCommand": "apt-get update && apt-get install -y nodejs"  // 一次性初始化
        },
        "browser": {
          "autoStart": true,        // 自动启动沙箱浏览器
          "allowHostControl": false // 禁止访问 Host 浏览器
        }
      }
    }
  }
}
```

### 4.5 Sandbox 与 Tool Policy 的交互

**逃逸路径**（`docs/gateway/sandboxing.md:152-159`）：

1. **`tools.elevated`**：显式逃逸沙箱，在 Host 执行 `exec`
   - 需要同时启用全局 + Agent 级别配置
   - 仅在 **非 main session 已沙箱化** 时生效（main session 本来就在 Host）

2. **`docker.binds`**：通过自定义挂载点绕过沙箱文件系统隔离
   - `:ro` 模式相对安全
   - `:rw` 模式需谨慎（等同于 Host 直接访问）

3. **`docker.network`**：开启网络访问
   - 默认 `"none"`（无网络）
   - 设置为 `"bridge"` 或 `"host"` 后可访问外部网络

---

## 5. 跨设备工具调用（Node 系统）🔥

### 5.1 Node 工具：统一设备管理入口

**位置**：`src/agents/tools/nodes-tool.ts:88-460`

**支持设备类型**：

- **macOS**：通过 OpenClaw Mac App（Companion）
- **iOS**：通过 OpenClaw iOS App
- **Android**：通过 OpenClaw Android App
- **Headless Node**：通过 `openclaw node run` 命令运行的独立进程

**核心功能**：

```typescript
const NODES_TOOL_ACTIONS = [
  "status",         // 列出所有已连接 nodes
  "describe",       // 查询 node 详细信息（caps、commands、versions）
  "pending",        // 查询待审批的 pairing 请求
  "approve",        // 批准 pairing 请求
  "reject",         // 拒绝 pairing 请求
  "notify",         // 发送系统通知（macOS system.notify）
  "camera_snap",    // 拍照（前置/后置/双摄）
  "camera_list",    // 列出可用摄像头
  "camera_clip",    // 录制视频片段
  "photos_latest",  // 获取最新照片（iOS/Android photos.latest）
  "screen_record",  // 屏幕录制
  "location_get",   // 获取 GPS 位置
  "run",            // 执行命令（macOS/iOS/Android system.run）
  "invoke",         // 通用 node.invoke（受限）
] as const;
```

**Nodes 更新（2026.3）**（`src/agents/tools/nodes-tool.ts`）：

1. **`photos_latest` action**（行 333-428）：专用 action 调用 `photos.latest` 命令，支持 `limit`、`maxWidth`、`quality`，返回图片 + MEDIA 标记，空结果统一返回 `{ content: [], details: [] }`

2. **Media-returning invoke 命令被阻止**（行 756-761）：`camera.snap`、`camera.clip`、`photos.latest`、`screen.record` 等返回媒体 payload 的 invoke 命令在 Agent 上下文中被阻止，需使用对应专用 action（`camera_snap`、`photos_latest` 等），避免 base64 上下文膨胀；仅当 `allowMediaInvokeCommands` 为 true 时允许（`openclaw-tools.ts:63`）

3. **`camera.list` metadata-only 仍允许**：`camera.list` 不在 `MEDIA_INVOKE_ACTIONS` 中（行 59-65），仅返回元数据（摄像头列表）不返回媒体，故 `action=invoke` + `invokeCommand=camera.list` 仍可用；专用 action `camera_list` 也通过 `NODE_READ_ACTION_COMMANDS` 调用同一命令（行 66-72）

### 5.2 跨设备调用机制：`node.invoke`

**位置**：`src/agents/tools/gateway.ts` + Gateway RPC `node.invoke`

**调用流程**：

```
Agent Tool Call
    ↓
nodes_tool.execute()
    ↓
callGatewayTool("node.invoke", { nodeId, command, params })
    ↓
Gateway WebSocket RPC
    ↓
Node App (macOS/iOS/Android)
    ↓
Execute Command (camera.snap / system.run / etc.)
    ↓
Return Payload (base64 image / JSON result)
    ↓
Gateway → Agent Tool Result
```

**示例：拍照工具调用**（`src/agents/tools/nodes-tool.ts:166-246`）：

```typescript
case "camera_snap": {
  const nodeId = await resolveNodeId(gatewayOpts, node);
  
  // 通过 Gateway RPC 调用 Node 的 camera.snap 命令
  const raw = await callGatewayTool<{ payload: unknown }>("node.invoke", gatewayOpts, {
    nodeId,
    command: "camera.snap",
    params: {
      facing: "front",
      maxWidth: 1920,
      quality: 0.9,
      format: "jpg",
    },
    idempotencyKey: crypto.randomUUID(),
  });
  
  // 解析返回的 base64 图片数据
  const payload = parseCameraSnapPayload(raw?.payload);
  
  // 保存到临时文件
  const filePath = cameraTempPath({ kind: "snap", facing: "front", ext: "jpg" });
  await writeBase64ToFile(filePath, payload.base64);
  
  // 返回图片 + MEDIA 标记
  return {
    content: [
      { type: "text", text: `MEDIA:${filePath}` },
      { type: "image", data: payload.base64, mimeType: "image/jpeg" }
    ],
    details: { facing, path: filePath, width: payload.width, height: payload.height }
  };
}
```

### 5.3 Browser 工具的跨设备适配

**位置**：`src/agents/tools/browser-tool.ts:42-114`

**Browser 工具支持三种运行模式**：

1. **`target: "sandbox"`**：沙箱浏览器（Docker 容器内的 CDP）
2. **`target: "host"`**：Host 浏览器（Gateway 所在机器的 CDP）
3. **`target: "node"`**：Node 浏览器（远程设备的浏览器）

**Node Browser 自动路由**：

```typescript
async function resolveBrowserNodeTarget(params: {
  requestedNode?: string;
  target?: "sandbox" | "host" | "node";
  sandboxBridgeUrl?: string;
}): Promise<BrowserNodeTarget | null> {
  const policy = cfg.gateway?.nodes?.browser;
  const mode = policy?.mode ?? "auto";  // auto / manual / off
  
  // 1. 如果 mode=off，禁止 Node browser
  if (mode === "off") {
    if (params.target === "node" || params.requestedNode) {
      throw new Error("Node browser proxy is disabled.");
    }
    return null;
  }
  
  // 2. 查询已连接的支持 browser 的 nodes
  const nodes = await listNodes({});
  const browserNodes = nodes.filter((node) => node.connected && isBrowserNode(node));
  
  // 3. 如果只有一个 browser node，自动路由（mode=auto）
  if (mode === "auto" && browserNodes.length === 1) {
    return { nodeId: browserNodes[0].nodeId, label: browserNodes[0].displayName };
  }
  
  // 4. 否则需要手动指定 node 或 target
  return null;
}
```

**配置示例**：

```json5
{
  "gateway": {
    "nodes": {
      "browser": {
        "mode": "auto",     // 自动路由到唯一的 browser node
        "node": "macbook"   // 或手动指定默认 node
      }
    }
  }
}
```

---

## 6. 插件工具系统 🔥

### 6.1 插件工具注册流程

**位置**：`src/plugins/tools.ts:43-129`

```typescript
export function resolvePluginTools(params: {
  context: OpenClawPluginToolContext;  // 运行时上下文
  existingToolNames?: Set<string>;     // 已注册工具名（避免冲突）
  toolAllowlist?: string[];            // 可选工具白名单
}): AnyAgentTool[] {
  // 1. 加载所有插件（从 extensions/ 和 ~/.openclaw/plugins/）
  const registry = loadOpenClawPlugins({ config, workspaceDir, logger });
  
  // 2. 遍历插件的 tool factories
  for (const entry of registry.tools) {
    // 3. 调用 factory 函数生成工具实例
    let resolved = entry.factory(params.context);
    
    // 4. 检查插件 ID 是否与核心工具冲突
    if (existingNormalized.has(normalizeToolName(entry.pluginId))) {
      log.error(`plugin id conflicts with core tool name (${entry.pluginId})`);
      continue;
    }
    
    // 5. 可选工具需要白名单允许
    if (entry.optional) {
      resolved = resolved.filter((tool) =>
        isOptionalToolAllowed({ toolName: tool.name, pluginId: entry.pluginId, allowlist })
      );
    }
    
    // 6. 检查工具名冲突
    for (const tool of resolved) {
      if (existing.has(tool.name)) {
        log.error(`plugin tool name conflict: ${tool.name}`);
        continue;
      }
      
      // 7. 记录插件元数据（用于后续权限判断）
      pluginToolMeta.set(tool, { pluginId: entry.pluginId, optional: entry.optional });
      tools.push(tool);
    }
  }
  
  return tools;
}
```

### 6.2 插件工具上下文

**位置**：`src/plugins/types.ts`

```typescript
export type OpenClawPluginToolContext = {
  config?: OpenClawConfig;
  workspaceDir?: string;
  agentDir?: string;
  agentId?: string;
  sessionKey?: string;
  messageChannel?: GatewayMessageChannel;
  agentAccountId?: string;
  sandboxed?: boolean;
};
```

### 6.3 插件工具示例：Lobster

**Lobster** 是一个可选插件工具，提供类型化工作流运行时（支持可恢复审批）

**注册方式**（假设 `extensions/lobster/src/tools.ts`）：

```typescript
export function createLobsterTool(context: OpenClawPluginToolContext): AnyAgentTool | null {
  // 1. 检查 Lobster CLI 是否可用
  if (!isLobsterInstalled()) {
    return null;  // 返回 null 表示工具不可用
  }
  
  // 2. 创建工具实例
  return {
    label: "Lobster",
    name: "lobster",
    description: "Run typed workflows with resumable approvals",
    parameters: LobsterToolSchema,
    execute: async (toolCallId, params) => {
      // 调用 Lobster CLI
      const result = await execLobster(params);
      return jsonResult(result);
    },
  };
}
```

**插件 manifest**（`extensions/lobster/plugin.json`）：

```json5
{
  "id": "lobster",
  "optional": true,  // 可选插件（需要白名单）
  "tools": ["lobster"]
}
```

**启用配置**：

```json5
{
  "tools": {
    "allow": ["group:fs", "group:runtime", "lobster"]  // 显式允许 lobster 工具
  }
}
```

---

## 7. Provider Schema 适配层 🔥

### 7.1 Provider 差异问题

不同 LLM Provider 对 Tool Schema 的支持存在差异：

| Provider | 问题 |
|----------|------|
| **OpenAI** | 要求 top-level 必须是 `type: "object"`（拒绝 `anyOf` 等 union schema） |
| **Gemini** | 拒绝多个 JSON Schema 关键字：`const`、`examples`、`title`、`$schema`、`$comment`、`dependencies` 等 |
| **Claude** | 支持较好，但需要特殊的参数组（`path_and_range`、`insert_opts`、`command_opts`）提升性能 |

### 7.2 统一 Schema 标准化：`normalizeToolParameters`

**位置**：`src/agents/pi-tools.schema.ts:65-175`

**处理逻辑**：

```typescript
export function normalizeToolParameters(tool: AnyAgentTool): AnyAgentTool {
  const schema = tool.parameters as Record<string, unknown>;
  
  // 1. 如果已经是 type: "object" + properties，仅清理 Gemini 不兼容字段
  if ("type" in schema && "properties" in schema && !schema.anyOf) {
    return { ...tool, parameters: cleanSchemaForGemini(schema) };
  }
  
  // 2. 如果缺少 type 但有 properties，强制添加 type: "object"（OpenAI 需要）
  if (!("type" in schema) && (schema.properties || schema.required) && !schema.anyOf && !schema.oneOf) {
    return { ...tool, parameters: cleanSchemaForGemini({ ...schema, type: "object" }) };
  }
  
  // 3. 如果是 Union schema（anyOf/oneOf），合并成单一 object schema
  const variantKey = Array.isArray(schema.anyOf) ? "anyOf" : Array.isArray(schema.oneOf) ? "oneOf" : null;
  if (variantKey) {
    const variants = schema[variantKey] as unknown[];
    const mergedProperties: Record<string, unknown> = {};
    const requiredCounts = new Map<string, number>();
    
    // 遍历所有 variant，合并 properties
    for (const entry of variants) {
      const props = (entry as { properties?: unknown }).properties;
      if (props && typeof props === "object") {
        for (const [key, value] of Object.entries(props)) {
          mergedProperties[key] = mergePropertySchemas(mergedProperties[key], value);
        }
      }
      
      // 统计 required 字段出现次数
      const required = (entry as { required?: unknown[] }).required || [];
      for (const key of required) {
        requiredCounts.set(key as string, (requiredCounts.get(key as string) ?? 0) + 1);
      }
    }
    
    // 只有在所有 variant 中都 required 的字段才标记为 required
    const mergedRequired = Array.from(requiredCounts.entries())
      .filter(([, count]) => count === variants.length)
      .map(([key]) => key);
    
    // 返回扁平化的 object schema
    return {
      ...tool,
      parameters: cleanSchemaForGemini({
        type: "object",
        properties: mergedProperties,
        required: mergedRequired.length > 0 ? mergedRequired : undefined,
      }),
    };
  }
  
  return tool;
}
```

### 7.3 Gemini Schema 清理：`cleanSchemaForGemini`

**位置**：`src/agents/schema/clean-for-gemini.ts`

**删除 Gemini 不支持的字段**：

```typescript
export function cleanSchemaForGemini(schema: Record<string, unknown>): unknown {
  const GEMINI_UNSUPPORTED_KEYS = [
    "const",
    "examples",
    "title",
    "$schema",
    "$comment",
    "dependencies",
    "propertyNames",
    "contentMediaType",
    "contentEncoding",
  ];
  
  function cleanObject(obj: Record<string, unknown>): Record<string, unknown> {
    const cleaned: Record<string, unknown> = {};
    for (const [key, value] of Object.entries(obj)) {
      if (GEMINI_UNSUPPORTED_KEYS.includes(key)) {
        continue;  // 跳过不支持的字段
      }
      
      if (typeof value === "object" && value !== null) {
        if (Array.isArray(value)) {
          cleaned[key] = value.map((item) => cleanObject(item));
        } else {
          cleaned[key] = cleanObject(value as Record<string, unknown>);
        }
      } else {
        cleaned[key] = value;
      }
    }
    return cleaned;
  }
  
  return cleanObject(schema);
}
```

### 7.4 Claude 参数组注入

**位置**：`src/agents/pi-tools.read.ts`

**为 Claude 注入特殊参数组**（提升文件编辑性能）：

```typescript
export const CLAUDE_PARAM_GROUPS = [
  {
    name: "path_and_range",
    parameters: ["path", "from", "to"],
  },
  {
    name: "insert_opts",
    parameters: ["after", "before"],
  },
  {
    name: "command_opts",
    parameters: ["command", "background", "yieldMs", "timeout", "elevated"],
  },
] as const;

export function patchToolSchemaForClaudeCompatibility(
  tool: AnyAgentTool,
  modelAuthMode?: ModelAuthMode,
): AnyAgentTool {
  // 1. Claude OAuth 模式禁止下划线开头的工具名
  if (modelAuthMode === "oauth" && tool.name.startsWith("_")) {
    const normalizedName = tool.name.replace(/^_+/, "");
    return { ...tool, name: normalizedName };
  }
  
  // 2. 为 read/write/edit/exec 工具注入参数组
  if (["read", "write", "edit", "exec"].includes(tool.name)) {
    return {
      ...tool,
      parameters: {
        ...tool.parameters,
        parameterGroups: CLAUDE_PARAM_GROUPS,
      },
    };
  }
  
  return tool;
}
```

---

## 8. 与 AI 主动性的关联 🔥

### 8.1 Heartbeat 相关工具：`cron` Tool

**位置**：`src/agents/tools/cron-tool.ts`

**核心功能**：

```typescript
const CRON_ACTIONS = [
  "status",    // 查询 cron scheduler 状态
  "list",      // 列出所有 cron jobs
  "add",       // 创建新 job
  "update",    // 更新 job 配置
  "remove",    // 删除 job
  "run",       // 立即触发 job
  "runs",      // 查询 job 运行历史
  "wake",      // 发送 wake 事件（触发 Heartbeat）
] as const;
```

**Heartbeat vs Cron 决策树**（`docs/automation/cron-vs-heartbeat.md`）：

| 场景 | 推荐方案 | 原因 |
|------|---------|------|
| 定期检查 Gmail 收件箱 | **Heartbeat** | AI 需要推理（哪些邮件需要关注） |
| 每天 9:00 发送 "早安" 消息 | **Cron** | 固定任务，无需 AI 推理 |
| 每周一生成周报 | **Heartbeat** | AI 需要总结和生成内容 |
| 定期备份数据库 | **Cron** | 机械操作，无需 AI 介入 |

**`wake` action 的作用**：

```typescript
case "wake": {
  const text = readStringParam(params, "text", { required: true });
  const mode = params.mode === "now" ? "now" : "next-heartbeat";
  
  // 向 Gateway 发送 wake 事件（注入 system event 到 main session）
  return jsonResult(
    await callGatewayTool("wake", gatewayOpts, { mode, text }, { expectFinal: false })
  );
}
```

**Heartbeat 完整流程**：

```
1. Heartbeat Runner (src/infra/heartbeat-runner.ts)
    ↓ 定时触发（基于 cron expression / interval）
2. 向 main session 注入 system event（包含 HEARTBEAT.md 中的检查指令）
    ↓
3. Agent 读取 HEARTBEAT.md（包含 inbox/calendar/GitHub 等检查列表）
    ↓
4. Agent 使用 memory_search 搜索相关历史
    ↓
5. Agent 调用相关工具（gmail_check / calendar_list / github_notifications）
    ↓
6. Agent 生成回复（如果有需要关注的内容）或返回 HEARTBEAT_OK（静默）
```

### 8.2 Memory 相关工具：`memory_search` + `memory_get`

**位置**：`src/agents/tools/memory-tool.ts`

**核心实现**：

```typescript
export function createMemorySearchTool(options: {
  config?: OpenClawConfig;
  agentSessionKey?: string;
}): AnyAgentTool | null {
  // 1. 检查 Memory 是否启用（需要配置 embedding provider）
  const agentId = resolveSessionAgentId({ sessionKey, config });
  if (!resolveMemorySearchConfig(config, agentId)) {
    return null;  // Memory 未配置，不注册工具
  }
  
  return {
    label: "Memory Search",
    name: "memory_search",
    description: "Mandatory recall step: semantically search MEMORY.md + memory/*.md (and optional session transcripts) before answering questions about prior work, decisions, dates, people, preferences, or todos; returns top snippets with path + lines.",
    parameters: MemorySearchSchema,
    execute: async (_toolCallId, params) => {
      const query = readStringParam(params, "query", { required: true });
      const maxResults = readNumberParam(params, "maxResults");
      const minScore = readNumberParam(params, "minScore");
      
      // 2. 获取 MemoryIndexManager 实例
      const { manager, error } = await getMemorySearchManager({ cfg, agentId });
      if (!manager) {
        return jsonResult({ results: [], disabled: true, error });
      }
      
      // 3. 执行 Hybrid Search（Vector + FTS5）
      const results = await manager.search(query, {
        maxResults,
        minScore,
        sessionKey: options.agentSessionKey,
      });
      
      // 4. 返回搜索结果（包含 path、lines、snippet）
      return jsonResult({
        results,
        provider: manager.status().provider,
        model: manager.status().model,
      });
    },
  };
}
```

**Memory 工具的作用**（第8层 Memory 自我整理机制）：

1. **自动索引**：File Watcher 监控 `~/.openclaw/workspace/<agentId>/memory/` 变更 → 自动 Chunking + Embedding → 存入 SQLite + sqlite-vec
2. **Hybrid Search**：`memory_search` 执行 **Vector Semantic Search** + **FTS5 Keyword Search** → 按相似度排序返回 top-k 片段
3. **上下文注入**：Agent 通过 `memory_search` 获取相关历史信息 → 注入到当前对话上下文
4. **Session Transcript 自动同步**：对话历史 (`~/.openclaw/agents/<agentId>/sessions/*.jsonl`) 自动索引到 Memory（可选）

### 8.3 Media 音频回显：`echoTranscript` + `echoFormat`

**位置**：`src/media-understanding/echo-transcript.ts`、`src/media-understanding/apply.ts:534-539`、`src/config/zod-schema.core.ts:684-685`

**配置**（`tools.media.audio`）：

- `echoTranscript`（默认 `false`）：是否在 Agent 处理前将转录文本回显到发起聊天的 Channel
- `echoFormat`：回显格式字符串，支持 `{transcript}` 占位符，默认 `'📝 "{transcript}"'`

**流程**：语音消息 → 转录 → 若 `echoTranscript=true` 则 `sendTranscriptEcho` 发送到 `ctx.OriginatingTo` → 再进入 Agent 处理

**类型定义**：`src/config/types.tools.ts:99-104`

### 8.4 Skills 相关工具：`skill_search` + `skill_install`（待实现）

**预期位置**：`src/agents/tools/skill-*-tool.ts`（计划中，第10层专题研究时详细分析）

**核心功能**：

1. **`skill_search`**：搜索 ClawdHub Registry 中的可用 Skills
   - 本地 fuzzy match（从 bundled/managed/workspace Skills）
   - 远程 API 搜索（ClawdHub Registry）
   - 返回 Skills 列表（name、description、author、tags）

2. **`skill_install`**：安装新 Skill
   - 从 ClawdHub 下载 Skill（SKILL.md 文件）
   - 验证 Skill 格式（frontmatter + instructions）
   - 保存到 `~/.openclaw/workspace/<agentId>/skills/`
   - 触发 Skills 刷新（chokidar watcher 自动检测）

**Agent 主动学习流程**（计划）：

```
1. Agent 遇到新需求（如 "生成 PDF 报告"）
    ↓
2. Agent 调用 skill_search({ query: "pdf generation" })
    ↓
3. 找到 "pdf-generator" Skill（包含 pdf_create 工具使用指南）
    ↓
4. Agent 调用 skill_install({ skillId: "pdf-generator" })
    ↓
5. Skill 安装到 workspace → Skills 刷新 → 下次 Agent run 自动加载
    ↓
6. Agent 使用新 Skill 完成任务
```

---

## 9. 关键技术细节总结

### 9.1 工具调用流程（完整链路）

```
User Message
    ↓
1. Gateway 接收消息 (WebSocket / HTTP)
    ↓
2. 路由到 Agent Session (main / group / subagent)
    ↓
3. 创建 Agent Tools:
   - createOpenClawCodingTools() → 文件系统工具
   - createOpenClawTools() → OpenClaw 特有工具
   - resolvePluginTools() → 插件工具
    ↓
4. 多层权限过滤:
   - Tool Profile → Global Policy → Agent Policy → Provider Policy → Group Policy → Subagent Policy → Sandbox Policy
    ↓
5. Schema 标准化:
   - normalizeToolParameters() → OpenAI/Gemini 兼容性
   - patchToolSchemaForClaudeCompatibility() → Claude 参数组
    ↓
6. 注入到 System Prompt (TOOLS.md)
    ↓
7. LLM 生成 Tool Call
    ↓
8. Tool Execute:
   - 沙箱环境：Docker 容器执行
   - Host 环境：直接执行
   - Node 环境：通过 node.invoke RPC 调用远程设备
    ↓
9. Tool Result 返回 → LLM 生成最终回复
    ↓
10. 消息发送到 Channel (Telegram / Discord / Slack / etc.)
```

### 9.2 核心设计模式

1. **Factory Pattern**：所有工具通过 `create*Tool()` 工厂函数创建，支持运行时配置注入
2. **Strategy Pattern**：工具权限通过可插拔的 Policy 策略控制，支持多层组合
3. **Adapter Pattern**：Schema 标准化层适配不同 Provider 的差异
4. **Proxy Pattern**：跨设备工具调用通过 `node.invoke` 代理到远程设备
5. **Observer Pattern**：Skills 自动刷新通过 chokidar file watcher 监听文件变更

### 9.3 可借鉴的优秀实践

1. **多层权限控制**：通过 Profile + Global + Agent + Provider + Group + Sandbox + Subagent 7 层 Policy，实现细粒度权限管理
2. **Provider Schema 适配层**：统一 Schema 标准化入口（`normalizeToolParameters`），支持多 Provider 兼容
3. **跨设备工具调用**：通过 WebSocket RPC（`node.invoke`）实现统一的远程工具调用接口
4. **Docker Sandbox 隔离**：通过 Sandbox mode（off/non-main/all）+ scope（session/agent/shared）+ workspaceAccess（none/ro/rw）实现灵活的隔离策略
5. **插件工具系统**：通过 Plugin SDK（`OpenClawPluginToolContext`）支持第三方工具动态注册
6. **Tool Groups 简化配置**：通过 `group:fs`、`group:runtime` 等工具组简化配置，提升可读性
7. **Subagent 特殊限制**：通过硬编码 `DEFAULT_SUBAGENT_TOOL_DENY` 防止 Subagent 干扰主会话，同时支持配置覆盖

---

## 10. 新增工具详解（⭐ 2026.3.22+ 更新）

### 10.1 `sessions_yield` — 长等待让步信号

**位置**：`src/agents/tools/sessions-yield-tool.ts`

`sessions_yield` 用于 Agent 在需要等待外部事件（如人工审核、长时间任务）时，主动让步控制权而不阻塞整个 Agent Lane。

**设计意图**：
- 与 `sessions_spawn` 配合实现"发出任务 → 挂起等待 → 恢复执行"模式
- 允许 Gateway 在等待期间处理其他会话请求
- 让子 Agent 或外部系统完成后通过 `sessions.send` 唤醒

### 10.2 `subagents` — 子 Agent 管理工具

**位置**：`src/agents/tools/subagents-tool.ts`

`subagents` 工具让主 Agent 可以管理其生成的子 Agent 运行状态：

```typescript
// 列出所有子 Agent 及其状态
subagents({ action: "list" })

// 中止子 Agent
subagents({ action: "kill", sessionKey: "agent:xxx:subagent:uuid" })

// 向运行中的子 Agent 发送指令调整
subagents({ action: "steer", sessionKey: "...", message: "更改方向" })
```

**与 `sessions_spawn` 的关系**：`sessions_spawn` 创建子 Agent，`subagents` 管理已创建的子 Agent。

### 10.3 `image_generate` — 标准图片生成工具

**位置**：`src/agents/tools/image-generate-tool.ts`（推测）

`image_generate` 是 2026.3.22 新增的标准图片生成工具，取代之前非标准的实现路径：

- 通过插件系统支持多个图片生成后端（如 `fal` 插件）
- 与 `createImageTool()`（图片分析）区分：`image_generate` 是生成，`image` 是分析

**可用性控制**：依赖图片生成插件（如 `extensions/fal/`）启用。

### 10.4 插件工具系统（Plugin Tools）

通过 `resolvePluginTools` 动态加载插件提供的工具，目前已有的插件工具：

| 插件 | 工具名 | 说明 |
|------|--------|------|
| `tavily` | `tavily_search`, `tavily_extract` | Tavily AI 搜索和内容提取 |
| `firecrawl` | `firecrawl_search`, `firecrawl_scrape` | Firecrawl 网页爬取和搜索 |
| `exa` | `exa_search`, `exa_find_similar` | Exa 语义搜索 |
| `brave` | `brave_search` | Brave 搜索 |
| `duckduckgo` | `duckduckgo_search` | DuckDuckGo 搜索 |
| `perplexity` | `perplexity_search` | Perplexity AI 搜索 |

**安装方式**（2026.3.22 ClawHub 市场）：
```bash
openclaw plugins install tavily
openclaw plugins install firecrawl
```

**工具可用性**：插件工具受 `pluginToolAllowlist` 参数过滤，可按 Agent 或全局配置白名单。

## 11. 待深入研究的方向

1. **Skills 工具系统**：`skill_search` / `skill_install` 的完整实现（第1层 System Prompt 设计中详细分析）
2. **Browser Tool CDP 实现**：`browserSnapshot()` 的 AI snapshot 算法（第8层高级特性）
3. **Memory 工具与 MemoryIndexManager 的集成**：Hybrid Search 的评分算法（第10层 AI 主动性专题）
4. **Canvas Tool 的 A2UI 实现**：A2UI v0.8 的协议细节（第8层高级特性）
5. **Tool Policy 与 Group Policy 的交互**：Channel-specific policy 的解析逻辑（第6层 Channel 集成）
6. **Elevated Exec 的权限模型**：如何在沙箱环境下安全逃逸（第7层安全与沙箱）

---

## 参考文献

- **工具注册**：`src/agents/openclaw-tools.ts`, `src/agents/pi-tools.ts`
- **工具抽象**：`src/agents/tools/common.ts`, `src/agents/pi-tools.types.ts`
- **权限控制**：`src/agents/pi-tools.policy.ts`, `src/agents/tool-policy.ts`
- **沙箱隔离**：`src/agents/sandbox/`, `docs/gateway/sandboxing.md`
- **跨设备调用**：`src/agents/tools/nodes-tool.ts`, `src/agents/tools/browser-tool.ts`
- **插件系统**：`src/plugins/tools.ts`, `src/plugins/types.ts`
- **Schema 适配**：`src/agents/pi-tools.schema.ts`, `src/agents/schema/clean-for-gemini.ts`
- **Memory 工具**：`src/agents/tools/memory-tool.ts`
- **Cron 工具**：`src/agents/tools/cron-tool.ts`
- **Subagents 工具**：`src/agents/tools/subagents-tool.ts`
- **Sessions Yield**：`src/agents/tools/sessions-yield-tool.ts`
- **Image Generate**：`src/agents/tools/image-generate-tool.ts`
- **插件工具**：`extensions/tavily/`, `extensions/firecrawl/`, `extensions/exa/`, `extensions/brave/`
- **PDF 工具**：`src/agents/tools/pdf-tool.ts`, `src/agents/tools/pdf-native-providers.ts`, `src/agents/tools/pdf-tool.helpers.ts`
- **Web Search**：`src/agents/tools/web-search.ts`
- **Onboarding**：`src/commands/onboard-config.ts`
- **Diffs 插件**：`extensions/diffs/src/tool.ts`, `extensions/diffs/src/config.ts`, `extensions/diffs/src/types.ts`
- **Media 回显**：`src/media-understanding/echo-transcript.ts`, `src/media-understanding/apply.ts`
- **文档**：`docs/tools/index.md`, `docs/tools/subagents.md`, `docs/tools/pdf`, `docs/nodes/audio.md`

---

璇玑 2026-03-05
