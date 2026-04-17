# OpenClaw Session 管理深度研究

## 版本信息

- **Commit**: `73055728318df378c831950cd01fb7c875a33790`
- **版本**: 2026.3.3
- **研究日期**: 2026-03-06
- **注意**: 该分析基于 2026 年 3 月的代码，可能随版本更新而变化

---

## 1. Session 模型概览

### 1.1 核心设计理念

OpenClaw 的 Session 管理遵循一个核心原则：**Gateway 是 Session 状态的唯一真实来源**（Single Source of Truth）。

在 `docs/concepts/session.md:17-28` 中明确说明：

> Gateway is the source of truth
> All session state is **owned by the gateway** (the "master" OpenClaw). UI clients (macOS app, WebChat, etc.) must query the gateway for session lists and token counts instead of reading local files.

这意味着：
- ✅ 所有 Session 状态由 Gateway 统一管理
- ✅ UI 客户端通过 RPC 调用查询 Session 信息
- ✅ 远程模式下，Session 状态存储在远程 Gateway 主机上

### 1.2 Session 存储结构

Session 状态存储在两个地方：

1. **Session Store** (`sessions.json`)：Session 元数据
   - 位置：`~/.openclaw/agents/<agentId>/sessions/sessions.json`
   - 数据结构：`Record<string, SessionEntry>`（sessionKey → SessionEntry 映射）
   - 包含：sessionId、updatedAt、token 计数、配置覆盖等

2. **Session Transcript** (`.jsonl` 文件)：对话历史
   - 位置：`~/.openclaw/agents/<agentId>/sessions/<sessionId>.jsonl`
   - 格式：每行一个 JSON 对象（JSONL）
   - 内容：完整的对话历史（user/assistant messages、tool calls）

### 1.3 Session Store 实现细节

在 `src/config/sessions/store.ts` 中实现了完整的 Session Store 管理：

**缓存机制** (`store.ts:19-51`)：

```typescript
// 45 秒 TTL 缓存，避免频繁磁盘读取
const DEFAULT_SESSION_STORE_TTL_MS = 45_000;

function isSessionStoreCacheValid(entry: SessionStoreCacheEntry): boolean {
  const now = Date.now();
  const ttl = getSessionStoreTtl();
  return now - entry.loadedAt <= ttl;
}
```

**文件锁机制** (`store.ts:269-337`)：

使用独占文件锁（`lockPath`）保证并发写入安全：

```typescript
async function withSessionStoreLock<T>(
  storePath: string,
  fn: () => Promise<T>,
  opts: SessionStoreLockOptions = {},
): Promise<T> {
  const lockPath = `${storePath}.lock`;
  // 尝试创建独占锁文件（wx 标志）
  const handle = await fs.promises.open(lockPath, "wx");
  // ... 执行操作后释放锁
}
```

**原子写入** (`store.ts:177-239`)：

- Windows 平台：直接写入（依赖文件锁序列化）
- Unix 平台：临时文件 + 原子 rename

---

## 2. Session Key 生成规则

### 2.1 Session Key 格式

OpenClaw 使用结构化的 Session Key 来唯一标识一个会话：

**基本格式**：`agent:<agentId>:<rest>`

在 `src/sessions/session-key-utils.ts:1-18` 中定义了解析规则：

```typescript
export type ParsedAgentSessionKey = {
  agentId: string;
  rest: string;
};

export function parseAgentSessionKey(
  sessionKey: string | undefined | null,
): ParsedAgentSessionKey | null {
  const raw = (sessionKey ?? "").trim();
  if (!raw) return null;
  const parts = raw.split(":").filter(Boolean);
  if (parts.length < 3) return null;
  if (parts[0] !== "agent") return null;
  const agentId = parts[1]?.trim();
  const rest = parts.slice(2).join(":");
  if (!agentId || !rest) return null;
  return { agentId, rest };
}
```

### 2.2 Session Key 类型

根据 `docs/concepts/session.md:42-57`，Session Key 分为以下几类：

**1. Direct Chat Sessions（DM）**：

根据 `session.dmScope` 配置决定：

- `main` (默认)：`agent:<agentId>:<mainKey>` (例如 `agent:main:main`)
  - 所有 DM 共享同一个 Session（跨设备、跨 Channel 连续性）
- `per-peer`：`agent:<agentId>:dm:<peerId>`
  - 按发送者隔离
- `per-channel-peer`：`agent:<agentId>:<channel>:dm:<peerId>`
  - 按 Channel + 发送者隔离（推荐用于多用户收件箱）
- `per-account-channel-peer`：`agent:<agentId>:<channel>:<accountId>:dm:<peerId>`
  - 按账号 + Channel + 发送者隔离（推荐用于多账号收件箱）

**2. Group Chat Sessions**：

- `agent:<agentId>:<channel>:group:<id>`
  - 例如：`agent:main:telegram:group:-1001234567890`
- Telegram Forum Topics：`agent:<agentId>:<channel>:group:<id>:topic:<threadId>`
  - 每个 Topic 独立 Session

**3. Room/Channel Sessions**：

- `agent:<agentId>:<channel>:channel:<id>`
  - 例如 Slack Channel、Discord Channel

**4. 其他特殊 Sessions**：

- Cron Jobs：`cron:<job.id>`
- Webhooks：`hook:<uuid>`
- Node Runs：`node-<nodeId>`

### 2.3 Session Key 生成实现

在 `src/config/sessions/session-key.ts:24-37` 中实现了 Session Key 的生成逻辑：

```typescript
export function resolveSessionKey(scope: SessionScope, ctx: MsgContext, mainKey?: string) {
  // 1. 优先使用显式指定的 SessionKey
  const explicit = ctx.SessionKey?.trim();
  if (explicit) return explicit.toLowerCase();
  
  // 2. 派生 Session Key
  const raw = deriveSessionKey(scope, ctx);
  if (scope === "global") return raw;
  
  // 3. 判断是否为 Group Session
  const isGroup = raw.includes(":group:") || raw.includes(":channel:");
  if (!isGroup) {
    // DM：使用 main session key
    return buildAgentMainSessionKey({ agentId: DEFAULT_AGENT_ID, mainKey: canonicalMainKey });
  }
  
  // Group：保持隔离
  return `agent:${DEFAULT_AGENT_ID}:${raw}`;
}
```

---

## 3. Session 生命周期管理

### 3.1 Session 创建

Session 是按需创建的（Lazy Creation）：

1. 当收到消息时，根据 `ctx` (MsgContext) 生成 Session Key
2. 在 Session Store 中查找对应的 SessionEntry
3. 如果不存在，自动创建新的 SessionEntry（生成 UUID 作为 sessionId）

在 `src/config/sessions/store.ts:339-380` 中的 `recordSessionMetaFromInbound` 实现了这个逻辑。

### 3.2 Session Reset 策略

OpenClaw 支持两种 Reset 模式：**Daily Reset** 和 **Idle Reset**。

在 `src/config/sessions/reset.ts:5-18` 中定义了 Reset Policy：

```typescript
export type SessionResetMode = "daily" | "idle";
export type SessionResetType = "dm" | "group" | "thread";

export type SessionResetPolicy = {
  mode: SessionResetMode;
  atHour: number;           // Daily reset 的时间点（本地时间）
  idleMinutes?: number;     // Idle reset 的超时时间
};
```

**Reset 规则** (`reset.ts:118-133`)：

```typescript
export function evaluateSessionFreshness(params: {
  updatedAt: number;
  now: number;
  policy: SessionResetPolicy;
}): SessionFreshness {
  const dailyResetAt =
    params.policy.mode === "daily"
      ? resolveDailyResetAtMs(params.now, params.policy.atHour)
      : undefined;
  const idleExpiresAt =
    params.policy.idleMinutes != null
      ? params.updatedAt + params.policy.idleMinutes * 60_000
      : undefined;
  const staleDaily = dailyResetAt != null && params.updatedAt < dailyResetAt;
  const staleIdle = idleExpiresAt != null && params.now > idleExpiresAt;
  return {
    fresh: !(staleDaily || staleIdle),  // 任一过期则 stale
    dailyResetAt,
    idleExpiresAt,
  };
}
```

**Daily Reset** (`docs/concepts/session.md:61-62`)：

- 默认：每天凌晨 4:00 AM（Gateway 主机本地时间）
- 如果 Session 的 `updatedAt` 早于最近的 Reset 时间点，则 Session 过期

**Idle Reset**：

- 配置 `idleMinutes`（例如 120 分钟）
- 如果 `now - updatedAt > idleMinutes * 60_000`，则 Session 过期

**混合模式**：

当同时配置 Daily 和 Idle Reset 时，**哪个先过期就用哪个**（任一触发即 Reset）。

### 3.3 Per-Type 和 Per-Channel 覆盖

在 `src/config/sessions/reset.ts:68-98` 中实现了灵活的 Reset 策略覆盖：

```typescript
export function resolveSessionResetPolicy(params: {
  sessionCfg?: SessionConfig;
  resetType: SessionResetType;  // dm / group / thread
  resetOverride?: SessionResetConfig;
}): SessionResetPolicy {
  const baseReset = params.resetOverride ?? sessionCfg?.reset;
  const typeReset = params.resetOverride ? undefined : sessionCfg?.resetByType?.[params.resetType];
  // typeReset > baseReset > 默认
}
```

**配置示例** (`docs/concepts/session.md:94-123`)：

```json5
{
  session: {
    reset: {
      mode: "daily",
      atHour: 4,
      idleMinutes: 120  // 同时启用 Idle Reset（取先到者）
    },
    resetByType: {
      thread: { mode: "daily", atHour: 4 },
      dm: { mode: "idle", idleMinutes: 240 },
      group: { mode: "idle", idleMinutes: 120 }
    },
    resetByChannel: {
      discord: { mode: "idle", idleMinutes: 10080 }  // 7 天
    }
  }
}
```

### 3.4 手动 Reset

用户可以通过以下方式手动重置 Session：

**1. Reset Triggers** (`docs/concepts/session.md:66`)：

- 发送 `/new` 或 `/reset` 命令
- 可选：`/new <model>` 指定新 Session 的模型

**2. 删除 Session 数据**：

- 删除 Session Store 中的 key
- 或删除 JSONL Transcript 文件

### 3.5 Session Compaction

在 `docs/concepts/session.md:32-40` 中提到了 Session Compaction：

- **工具裁剪（Tool Pruning）**：在 LLM 调用前从上下文中移除旧的 tool results
- **不会重写 JSONL 历史**：只影响内存中的上下文
- **Pre-compaction Memory Flush**：接近 Compaction 时触发静默 Memory 写入

### 3.6 Session Startup 日期 grounding（YYYY-MM-DD）

为避免 Agent 根据训练 cutoff 猜测日期，启动和 post-compaction 上下文中会将 `YYYY-MM-DD` 占位符替换为运行时日期，并追加 `Current time:` 行。

**Post-compaction AGENTS 上下文** (`src/auto-reply/reply/post-compaction-context.ts:29-86`)：

- 从 workspace `AGENTS.md` 提取 "Session Startup" 和 "Red Lines" 段落
- `formatDateStamp` 按用户时区生成 `YYYY-MM-DD` 格式日期
- `sections.join("\n\n").replaceAll("YYYY-MM-DD", dateStamp)` 替换占位符
- 追加 `timeLine`（来自 `resolveCronStyleNow`，格式如 `Current time: 2026-03-06 12:00 (Asia/Shanghai)`）

**Memory Flush Prompt** (`src/auto-reply/reply/memory-flush.ts:41-57`)：

- `resolveMemoryFlushPromptForRun` 将 prompt 中的 `YYYY-MM-DD` 替换为 `formatDateStampInTimezone` 的结果
- 若结果不含 `Current time:`，则追加 `timeLine`

**`/new` 和 `/reset` 提示** (`src/auto-reply/reply/session-reset-prompt.ts:4-18`)：

- `buildBareSessionResetPrompt` 构建裸 reset 提示（"A new session was started via /new or /reset..."）
- 通过 `appendCronStyleCurrentTimeLine` 追加 `Current time:` 行
- 实现：`src/agents/current-time.ts:31-37`，`resolveCronStyleNow` 生成 `timeLine`，`appendCronStyleCurrentTimeLine` 在文本末尾追加（若尚未包含 `Current time:`）

这样 Agent 在 Session Startup 时能正确解析 `memory/YYYY-MM-DD.md` 等路径。

---

## 4. Session 隔离与权限控制

### 4.1 Session 隔离模型

OpenClaw 通过 Session Key 实现了严格的隔离：

**1. Main Session vs Non-Main Sessions** (`docs/concepts/groups.md:49-60`)：

- **Main Session**（DM）：`agent:<agentId>:main`
  - 默认运行在 Host 上（非沙箱）
  - 拥有完整工具权限
- **Non-Main Sessions**（Groups/Channels）：`agent:<agentId>:<channel>:group:<id>`
  - 可配置为沙箱模式运行（Docker 隔离）
  - 工具权限可受限

**2. Agent-to-Agent 隔离**：

不同 Agent 的 Sessions 完全隔离（通过 `agentId` 前缀区分）。

### 4.2 Sandbox 模式下的 Session 工具可见性

在 `src/agents/tools/sessions-list-tool.ts:28-30` 中定义了 `sessionToolsVisibility` 配置：

```typescript
function resolveSandboxSessionToolsVisibility(cfg: ReturnType<typeof loadConfig>) {
  return cfg.agents?.defaults?.sandbox?.sessionToolsVisibility ?? "spawned";
}
```

**可见性规则** (`sessions-list-tool.ts:54-58`)：

```typescript
const restrictToSpawned =
  opts?.sandboxed === true &&
  visibility === "spawned" &&
  requesterInternalKey &&
  !isSubagentSessionKey(requesterInternalKey);
```

当 `restrictToSpawned = true` 时：

- `sessions_list`：只能看到当前 Session spawn 的子 Sessions
- `sessions_send`：只能发送到 spawned Sessions
- `sessions_history`：只能读取 spawned Sessions 的历史

**Spawned Session 追踪** (`src/config/sessions/types.ts:37-38`)：

```typescript
export type SessionEntry = {
  /** Parent session key that spawned this session (used for sandbox session-tool scoping). */
  spawnedBy?: string;
  // ...
};
```

### 4.3 Send Policy（发送策略）

在 `src/sessions/send-policy.ts` 中实现了细粒度的发送权限控制：

**配置示例** (`docs/concepts/session.md:72-84`)：

```json5
{
  session: {
    sendPolicy: {
      rules: [
        { action: "deny", match: { channel: "discord", chatType: "group" } },
        { action: "deny", match: { keyPrefix: "cron:" } }
      ],
      default: "allow"
    }
  }
}
```

**决策逻辑** (`send-policy.ts:35-78`)：

```typescript
export function resolveSendPolicy(params: {
  cfg: OpenClawConfig;
  entry?: SessionEntry;
  sessionKey?: string;
  channel?: string;
  chatType?: SessionChatType;
}): SessionSendPolicyDecision {
  // 1. Session 级别覆盖（/send on/off）
  const override = normalizeSendPolicy(params.entry?.sendPolicy);
  if (override) return override;
  
  // 2. 全局 Policy 规则
  for (const rule of policy.rules ?? []) {
    // 匹配 channel / chatType / keyPrefix
    if (matchChannel && matchChannel !== channel) continue;
    if (matchChatType && matchChatType !== chatType) continue;
    if (matchPrefix && !sessionKey.startsWith(matchPrefix)) continue;
    if (action === "deny") return "deny";  // deny 优先
    allowedMatch = true;
  }
  
  // 3. 默认策略
  return fallback ?? "allow";
}
```

**运行时覆盖** (`docs/concepts/session.md:87-91`)：

- `/send on` → 允许当前 Session
- `/send off` → 拒绝当前 Session
- `/send inherit` → 清除覆盖，恢复配置规则

---

## 5. 跨 Session 通信（Agent-to-Agent）

### 5.1 Sessions Tools 概览

OpenClaw 提供了 4 个核心工具来实现跨 Session 通信：

| 工具 | 作用 | 文件 |
|------|------|------|
| `sessions_list` | 列出所有可见的 Sessions | `src/agents/tools/sessions-list-tool.ts` |
| `sessions_send` | 向另一个 Session 发送消息 | `src/agents/tools/sessions-send-tool.ts` |
| `sessions_history` | 读取另一个 Session 的历史 | `src/agents/tools/sessions-history-tool.ts` |
| `sessions_spawn` | 创建子 Agent Session | `src/agents/tools/sessions-spawn-tool.ts` |

### 5.2 sessions_list 实现

在 `src/agents/tools/sessions-list-tool.ts:32-35` 中定义了工具：

```typescript
export function createSessionsListTool(opts?: {
  agentSessionKey?: string;
  sandboxed?: boolean;
}): AnyAgentTool {
  return {
    name: "sessions_list",
    description: "List sessions with optional filters and last messages.",
    parameters: SessionsListToolSchema,
    execute: async (_toolCallId, args) => {
      // ...
    }
  };
}
```

**核心逻辑** (`sessions-list-tool.ts:82-95`)：

1. 调用 Gateway RPC：`sessions.list`
2. 应用可见性过滤（`restrictToSpawned`）
3. 应用 Agent-to-Agent 权限检查
4. 返回 Session 列表（带 displayKey、channel、token 统计等）

**过滤规则**：

- `kinds`：按类型过滤（main / group / cron / hook / node / other）
- `limit`：限制返回数量
- `activeMinutes`：只返回最近活跃的 Sessions
- `messageLimit`：每个 Session 返回最后 N 条消息

### 5.3 sessions_send 实现

**核心流程** (`src/agents/tools/sessions-send-tool.ts:50-198`)：

```typescript
export function createSessionsSendTool(opts?: {
  agentSessionKey?: string;
  agentChannel?: GatewayMessageChannel;
  sandboxed?: boolean;
}): AnyAgentTool {
  return {
    name: "sessions_send",
    description: "Send a message into another session. Use sessionKey or label to identify the target.",
    parameters: SessionsSendToolSchema,
    execute: async (_toolCallId, args) => {
      // 1. 解析目标 Session（sessionKey 或 label）
      const sessionKey = await resolveTargetSessionKey(args);
      
      // 2. 权限检查（Agent-to-Agent Policy）
      if (crossAgent && !a2aPolicy.isAllowed(requesterAgentId, targetAgentId)) {
        return jsonResult({ status: "forbidden", error: "..." });
      }
      
      // 3. 发送消息到目标 Session
      await callGateway({
        method: "chat.send",
        params: {
          sessionKey: resolvedKey,
          message,
          timeoutMs
        }
      });
      
      // 4. 等待响应（可选超时）
      const response = await waitForResponse(timeoutSeconds);
      
      return jsonResult({ status: "ok", response });
    }
  };
}
```

**Label 解析** (`sessions-send-tool.ts:92-137`)：

如果使用 `label` 而非 `sessionKey`：

1. 调用 `sessions.resolve` RPC 查找匹配的 Session
2. 可选指定 `agentId` 限定查找范围
3. 支持跨 Agent 查找（需 Agent-to-Agent 权限）

**Agent-to-Agent Policy** (`sessions-send-tool.ts:70` 和 `sessions-helpers.ts`)：

```typescript
const a2aPolicy = createAgentToAgentPolicy(cfg);
if (!a2aPolicy.isAllowed(requesterAgentId, targetAgentId)) {
  return jsonResult({
    status: "forbidden",
    error: "Agent-to-agent messaging denied by tools.agentToAgent.allow."
  });
}
```

**超时响应机制** (`sessions-send-tool.ts` 和 `sessions-send-helpers.ts`)：

- 支持 `timeoutSeconds` 参数
- 等待目标 Session 回复
- 实现 Ping-Pong 对话（`resolvePingPongTurns`）

### 5.4 sessions_history 实现

在 `src/agents/tools/sessions-history-tool.ts:47-141` 中实现：

**核心逻辑**：

```typescript
export function createSessionsHistoryTool(opts?: {
  agentSessionKey?: string;
  sandboxed?: boolean;
}): AnyAgentTool {
  return {
    name: "sessions_history",
    description: "Fetch message history for a session.",
    execute: async (_toolCallId, args) => {
      // 1. 解析 sessionKey
      const resolvedSession = await resolveSessionReference({
        sessionKey: sessionKeyParam,
        alias,
        mainKey,
        requesterInternalKey,
        restrictToSpawned,
      });
      
      // 2. Spawned Session 检查
      if (restrictToSpawned && !resolvedViaSessionId) {
        const ok = await isSpawnedSessionAllowed({ requesterSessionKey, targetSessionKey });
        if (!ok) return jsonResult({ status: "forbidden", error: "..." });
      }
      
      // 3. Agent-to-Agent 权限检查
      if (isCrossAgent) {
        if (!a2aPolicy.isAllowed(requesterAgentId, targetAgentId)) {
          return jsonResult({ status: "forbidden", error: "..." });
        }
      }
      
      // 4. 读取历史
      const result = await callGateway({
        method: "chat.history",
        params: { sessionKey: resolvedKey, limit }
      });
      
      // 5. 过滤 Tool Messages（可选）
      const messages = includeTools ? rawMessages : stripToolMessages(rawMessages);
      
      return jsonResult({ sessionKey: displayKey, messages });
    }
  };
}
```

**关键特性**：

- 支持 `limit` 限制返回消息数量
- 支持 `includeTools` 控制是否返回 Tool Calls/Results
- 严格的权限检查（Sandbox 可见性 + Agent-to-Agent Policy）

### 5.5 sessions_spawn 实现

`sessions_spawn` 是最复杂的跨 Session 工具，用于创建**子 Agent Session**。

在 `src/agents/tools/sessions-spawn-tool.ts:60-72` 中定义：

```typescript
export function createSessionsSpawnTool(opts?: {
  agentSessionKey?: string;
  agentChannel?: GatewayMessageChannel;
  sandboxed?: boolean;
  requesterAgentIdOverride?: string;
}): AnyAgentTool {
  return {
    name: "sessions_spawn",
    description: "Spawn a background sub-agent run in an isolated session and announce the result back to the requester chat.",
    parameters: SessionsSpawnToolSchema,
    execute: async (_toolCallId, args) => {
      // 1. 安全检查：禁止 sub-agent 再次 spawn
      if (isSubagentSessionKey(requesterSessionKey)) {
        return jsonResult({ status: "forbidden", error: "sessions_spawn is not allowed from sub-agent sessions" });
      }
      
      // 2. 跨 Agent spawn 权限检查
      if (targetAgentId !== requesterAgentId) {
        const allowAgents = resolveAgentConfig(cfg, requesterAgentId)?.subagents?.allowAgents ?? [];
        if (!allowAny && !allowSet.has(normalizedTargetId)) {
          return jsonResult({ status: "forbidden", error: "..." });
        }
      }
      
      // 3. 生成 Sub-agent Session Key
      const subagentSessionKey = `agent:${targetAgentId}:subagent:${crypto.randomUUID()}`;
      
      // 4. 构建 Sub-agent System Prompt
      const systemPrompt = buildSubagentSystemPrompt({
        requesterKey: requesterDisplayKey,
        task,
        label: label || "Background task"
      });
      
      // 5. 启动 Sub-agent Run
      const runId = registerSubagentRun({
        requesterSessionKey: requesterInternalKey,
        subagentSessionKey,
        task,
        cleanup
      });
      
      // 6. 通过 Gateway 发送消息到 Sub-agent Session
      await callGateway({
        method: "chat.send",
        params: {
          sessionKey: subagentSessionKey,
          message: task,
          systemPrompt,
          lane: AGENT_LANE_SUBAGENT,
          deliveryContext: requesterOrigin
        }
      });
      
      // 7. 等待 Sub-agent 完成（可选超时）
      const result = await waitForSubagentCompletion(runId, runTimeoutSeconds);
      
      // 8. 清理 Session（如果 cleanup=delete）
      if (cleanup === "delete") {
        await deleteSession(subagentSessionKey);
      }
      
      return jsonResult({ status: "ok", result, sessionKey: subagentSessionKey });
    }
  };
}
```

**关键特性**：

- **Sub-agent Session Key**：`agent:<agentId>:subagent:<uuid>`
- **Nested Spawn 禁止**：Sub-agent 不能再 spawn（防止无限递归）
- **跨 Agent Spawn**：需要显式配置 `agents.<agentId>.subagents.allowAgents`
- **结果回传**：Sub-agent 完成后自动通知父 Session（Announce Back）
- **Cleanup 策略**：`keep`（保留 Session）或 `delete`（自动删除）

**Sub-agent System Prompt** (`src/agents/subagent-announce.ts`)：

Sub-agent 的 System Prompt 会注入特殊指令：

- 明确告知这是一个后台任务
- 任务完成后自动回传结果到父 Session
- 使用 `AGENT_LANE_SUBAGENT` 标记（优先级处理）

### 5.6 sessions_spawn 内联附件（Attachments）

`sessions_spawn` 支持将文件以内联方式传递给子 Session（仅 `runtime="subagent"`，ACP 不支持）。

**工具 Schema** (`src/agents/tools/sessions-spawn-tool.ts:39-51`)：

```typescript
attachments: Type.Optional(
  Type.Array(
    Type.Object({
      name: Type.String(),
      content: Type.String(),
      encoding: Type.Optional(optionalStringEnum(["utf8", "base64"] as const)),
      mimeType: Type.Optional(Type.String()),
    }),
    { maxItems: 50 },
  ),
),
```

**编码方式** (`src/agents/subagent-spawn.ts:585-620`)：

- `utf8`（默认）：`content` 为 UTF-8 字符串，`Buffer.from(contentVal, "utf8")` 写入
- `base64`：`content` 为 Base64 字符串，经 `decodeStrictBase64` 解码后写入，超出 `maxFileBytes` 会抛出 `attachments_invalid_base64_or_too_large`

**Transcript 内容脱敏** (`src/agents/session-transcript-repair.ts:74-132`)：

- `redactSessionsSpawnAttachmentsArgs` 将 `attachments[].content` 替换为 `__OPENCLAW_REDACTED__`
- 在 `sanitizeToolCallBlock` 中对 `sessions_spawn` 的 `arguments` 和 `input` 应用脱敏，避免敏感附件内容持久化到 JSONL

**生命周期清理** (`src/agents/subagent-registry.ts`)：

- `cleanup="delete"` 或 `retainOnSessionKeep=false` 时，子 Run 结束后调用 `safeRemoveAttachmentsDir` 删除附件目录
- Archive/purge 时同样会移除附件（`subagent-registry.ts:578-579`）
- `retainOnSessionKeep=true` 时，`cleanup="keep"` 会保留附件目录

**配置** `tools.sessions_spawn.attachments` (`src/agents/subagent-spawn.ts:504-523`)：

| 配置项 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `enabled` | boolean | - | 必须为 `true` 才能使用 attachments |
| `maxTotalBytes` | number | 5MB | 所有附件总字节上限 |
| `maxFiles` | number | 50 | 单次 spawn 最大文件数 |
| `maxFileBytes` | number | 1MB | 单文件字节上限 |
| `retainOnSessionKeep` | boolean | false | `cleanup="keep"` 时是否保留附件目录 |

附件写入路径：`<agentWorkspace>/.openclaw/attachments/<attachmentId>/`，子 Session 的 System Prompt 会注入该相对路径提示。

### 5.7 ACP Session Bootstrap 重试

当 `sessions ensure` 未返回有效 Session 标识时，ACP/ACPX 会回退到 `sessions new` 创建新 Session，避免 `NO_SESSION`/`ACP_TURN_FAILED` 失败。

**实现位置**：`extensions/acpx/src/runtime.ts:181-218`

```typescript
// 1. 先执行 sessions ensure
let events = await this.runControlCommand({
  args: this.buildControlArgs({ cwd, command: [agent, "sessions", "ensure", "--name", sessionName] }),
  ...
});
let ensuredEvent = events.find(...);

// 2. 若 ensure 无有效 session ID，则执行 sessions new
if (!ensuredEvent) {
  events = await this.runControlCommand({
    args: this.buildControlArgs({ cwd, command: [agent, "sessions", "new", "--name", sessionName] }),
    ...
  });
  ensuredEvent = events.find(...);
  if (!ensuredEvent) {
    throw new AcpRuntimeError("ACP_SESSION_INIT_FAILED",
      `ACP session init failed: neither 'sessions ensure' nor 'sessions new' returned valid session identifiers for ${sessionName}.`);
  }
}
```

**测试**：`extensions/acpx/src/runtime.test.ts:381-426` 覆盖了 ensure 空时回退到 new、以及两者都空时抛出 `ACP_SESSION_INIT_FAILED` 的场景。

### 5.8 streamTo: "parent"（ACP 运行时）

当 `runtime="acp"` 且 `streamTo: "parent"` 时，子 Run 的进度/完成更新会转发到 requester Session，作为 system events 而非直接投递到子 Session。

**实现位置**：`src/agents/acp-spawn.ts:246-251, 434-440, 485-506`

- 要求存在 `parentSessionKey`（requester session context），否则返回错误：`streamTo="parent" requires an active requester session context`
- 通过 `startAcpSpawnParentStreamRelay` 注册 relay，将子 Run 的 progress/no-output/completion 事件转发到父 Session
- 返回结果中包含 `streamLogPath`（当可用时）

**streamLogPath** (`src/agents/acp-spawn-parent-stream.ts:42-74`)：

- 路径格式：`<sessionsDir>/<sessionId>.acp-stream.jsonl`
- 与子 Session 的 transcript 同目录，按 session 作用域
- 可用于 `tail` 查看 ACP stream relay 的进度历史

**文档**：`docs/tools/index.md:486-500`、`docs/tools/acp-agents.md:243`

---

## 6. Session 配置覆盖机制

### 6.1 配置层级

OpenClaw 支持多层级的 Session 配置覆盖：

**覆盖优先级**（从高到低）：

1. **Session 级别覆盖**（运行时命令）
   - `/thinking high`、`/verbose on`、`/send off` 等
   - 存储在 `SessionEntry.thinkingLevel`、`verboseLevel`、`sendPolicy` 等字段

2. **Per-Session 配置**（静态配置）
   - `resetByType`、`resetByChannel`、`tools.policy`

3. **全局默认配置**
   - `session.reset`、`agents.defaults.*`

### 6.2 Session Entry 字段

在 `src/config/sessions/types.ts:26-97` 中定义了 `SessionEntry` 的完整结构：

```typescript
export type SessionEntry = {
  // === 核心字段 ===
  sessionId: string;                    // UUID
  updatedAt: number;                    // 最后更新时间戳
  sessionFile?: string;                 // Transcript 文件路径
  
  // === Heartbeat 字段 ===
  lastHeartbeatText?: string;           // 最后发送的 Heartbeat 内容
  lastHeartbeatSentAt?: number;         // Heartbeat 发送时间
  
  // === 关系字段 ===
  spawnedBy?: string;                   // 父 Session Key（Sandbox 可见性控制）
  
  // === 配置覆盖字段 ===
  thinkingLevel?: string;               // Thinking 级别覆盖
  verboseLevel?: string;                // Verbose 级别覆盖
  reasoningLevel?: string;              // Reasoning 级别覆盖
  elevatedLevel?: string;               // Elevated 权限级别覆盖
  ttsAuto?: TtsAutoMode;                // TTS 自动播放覆盖
  providerOverride?: string;            // Provider 覆盖
  modelOverride?: string;               // Model 覆盖
  authProfileOverride?: string;         // Auth Profile 覆盖
  sendPolicy?: "allow" | "deny";        // 发送策略覆盖
  groupActivation?: "mention" | "always"; // Group 激活模式覆盖
  
  // === 统计字段 ===
  inputTokens?: number;                 // 输入 Token 统计
  outputTokens?: number;                // 输出 Token 统计
  totalTokens?: number;                 // 总 Token 统计
  contextTokens?: number;               // 上下文 Token 统计
  compactionCount?: number;             // Compaction 次数
  
  // === 元数据字段 ===
  label?: string;                       // 用户自定义标签
  displayName?: string;                 // UI 显示名称
  channel?: string;                     // Channel ID
  subject?: string;                     // Group 主题
  groupChannel?: string;                // Group Channel 名称
  origin?: SessionOrigin;               // Session 来源信息
  
  // === 路由字段 ===
  deliveryContext?: DeliveryContext;    // 投递上下文
  lastChannel?: SessionChannelId;       // 最后使用的 Channel
  lastTo?: string;                      // 最后投递的目标
  lastAccountId?: string;               // 最后使用的账号 ID
  lastThreadId?: string | number;       // 最后使用的 Thread ID
  
  // === 快照字段 ===
  skillsSnapshot?: SessionSkillSnapshot;         // Skills 快照
  systemPromptReport?: SessionSystemPromptReport; // System Prompt 报告
};
```

### 6.3 Level Overrides 实现

在 `src/sessions/level-overrides.ts` 中实现了各种 Level 的覆盖逻辑（例如 `thinkingLevel`、`verboseLevel`）。

**Model Overrides** 在 `src/sessions/model-overrides.ts` 中实现了 Model 覆盖逻辑（`providerOverride`、`modelOverride`）。

---

## 7. 与 AI 主动性的关联

### 7.1 Heartbeat 与 Session

**Heartbeat 执行的 Session**：

根据设计，Heartbeat 在 **Main Session** 中执行（`agent:<agentId>:main`）。

**Heartbeat 状态存储** (`src/config/sessions/types.ts:28-32`)：

```typescript
export type SessionEntry = {
  /**
   * Last delivered heartbeat payload (used to suppress duplicate heartbeat notifications).
   * Stored on the main session entry.
   */
  lastHeartbeatText?: string;
  /** Timestamp (ms) when lastHeartbeatText was delivered. */
  lastHeartbeatSentAt?: number;
  // ...
};
```

**HEARTBEAT_OK 机制**：

- Heartbeat 运行后，如果判定"一切正常"，生成 `HEARTBEAT_OK` 消息
- 与 `lastHeartbeatText` 比较，如果相同则**静默**（不发送通知）
- 避免重复打扰用户

**Group Sessions 跳过 Heartbeat** (`docs/concepts/groups.md:47`)：

> Heartbeats are skipped for group sessions.

这意味着 Heartbeat 只在 Main Session（DM）中执行，不会在 Group Chats 中触发。

### 7.2 Memory 与 Session Transcript

**Session Transcript 自动同步到 Memory**：

在 `src/memory/sync-session-files.ts` 中实现了 Session Transcript 的自动索引：

- **File Watcher** 监控 `~/.openclaw/agents/<agentId>/sessions/*.jsonl` 目录
- 当 Transcript 文件更新时，自动触发 Embedding 生成和索引
- 索引到 SQLite + sqlite-vec 向量数据库

**哪些 Sessions 会被索引？**

根据 Memory 配置，通常是：

- ✅ Main Session（DM）：默认索引
- ✅ 用户配置的特定 Group Sessions
- ❌ 临时 Sub-agent Sessions：通常不索引（除非显式配置）

**Session Transcript 格式** (`src/config/sessions/transcript.ts:54-68`)：

```jsonl
{"type":"session","version":2,"id":"<uuid>","timestamp":"2026-01-30T12:00:00.000Z","cwd":"/path/to/workspace"}
{"role":"user","content":[{"type":"text","text":"Hello"}],"timestamp":1706620800000}
{"role":"assistant","content":[{"type":"text","text":"Hi!"}],"api":"openai-responses","provider":"anthropic","model":"claude-sonnet-4","usage":{...},"timestamp":1706620801000}
```

**Memory Search Tool** (`src/agents/tools/memory-search-tool.ts`)：

- Agent 可以使用 `memory_search` Tool 搜索历史对话
- Hybrid Search：Vector Semantic Search + FTS5 Keyword Search
- 结果按相关性排序返回

### 7.3 Skills 与 Session

**Skills 配置共享**：

Skills 配置在 **Agent 级别**，不是 Session 级别：

- 所有同一 Agent 的 Sessions 共享相同的 Skills
- Skills 存储在 `~/.openclaw/agents/<agentId>/skills/` 目录

**Skills Snapshot** (`src/config/sessions/types.ts:116-121`)：

```typescript
export type SessionSkillSnapshot = {
  prompt: string;                       // Skills 注入的 Prompt 内容
  skills: Array<{ name: string; primaryEnv?: string }>;  // Skills 列表
  resolvedSkills?: Skill[];             // 解析后的 Skills 对象
  version?: number;                     // Snapshot 版本号
};
```

- Session 保存当前使用的 Skills 快照
- 用于诊断和调试（`/context detail` 命令）

**Skills 刷新触发**：

- **File Watcher**（`chokidar`）监控 Skills 目录变更
- 自动刷新 Skills 缓存
- 下一次 Agent Run 时生效

---

## 8. Session 管理最佳实践

### 8.1 Main Session vs Group Sessions 隔离

**推荐模式**（单 Agent + Sandbox）：

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main",          // Group Sessions 沙箱化
        scope: "session",          // 每个 Session 独立容器
        workspaceAccess: "none"    // Group 无法访问 Workspace
      }
    }
  },
  tools: {
    sandbox: {
      tools: {
        allow: ["group:messaging", "group:sessions"],  // 只允许消息和 Sessions Tools
        deny: ["group:runtime", "group:fs", "group:ui", "nodes", "cron", "gateway"]
      }
    }
  }
}
```

这样配置后：

- **DM（Main Session）**：完整权限，运行在 Host
- **Group Sessions**：受限权限，运行在 Docker 沙箱

### 8.2 Session Reset 策略选择

**推荐配置**：

- **DM**：`mode: "idle"`, `idleMinutes: 240`（4 小时）
  - 保持长期对话连续性
- **Group**：`mode: "daily"`, `atHour: 4`
  - 每天清空，避免上下文污染
- **Thread**：`mode: "idle"`, `idleMinutes: 60`（1 小时）
  - 短期讨论，快速过期

### 8.3 Agent-to-Agent 通信权限

**启用 Agent-to-Agent 通信**：

```json5
{
  tools: {
    agentToAgent: {
      enabled: true,
      allow: [
        { from: "main", to: "assistant" },
        { from: "assistant", to: "main" }
      ]
    }
  }
}
```

**注意事项**：

- 默认禁用跨 Agent 通信（安全考虑）
- 需要显式配置 `allow` 规则
- 支持通配符：`{ from: "*", to: "*" }`

---

## 9. 核心设计模式总结

### 9.1 Session Key 层次结构

```
agent:<agentId>:<rest>
       ↓         ↓
       |         └─→ main (DM)
       |         └─→ <channel>:group:<id> (Group)
       |         └─→ <channel>:channel:<id> (Room)
       |         └─→ subagent:<uuid> (Sub-agent)
       └─→ Agent 隔离边界
```

### 9.2 Session Store 读写模式

**读取**：

- 45 秒 TTL 缓存（避免频繁磁盘读取）
- Mtime 检查（Cache Invalidation）

**写入**：

- 文件锁序列化（并发安全）
- 原子 Rename（Unix）或直接写入（Windows）

### 9.3 跨 Session 通信流程

```
Requester Session
     ↓
1. sessions_send("target-session", "message")
     ↓
2. Gateway RPC: chat.send
     ↓
3. Target Session 收到消息
     ↓
4. Target Agent Run 生成回复
     ↓
5. （可选）等待回复（Ping-Pong）
     ↓
6. 返回结果到 Requester
```

### 9.4 Sandbox 可见性控制

```
Main Session (Host)
  ├─ sessions_spawn → Sub-agent Session A (Docker)
  │                     └─ sessions_list → 只能看到自己
  │
  └─ sessions_spawn → Sub-agent Session B (Docker)
                        └─ sessions_list → 只能看到自己

Main Session
  └─ sessions_list → 可以看到 A、B（通过 spawnedBy 追踪）
```

---

## 10. 可借鉴的设计点

### 10.1 Session Key 的结构化设计

**优点**：

- ✅ 统一的 Session 标识格式（`agent:<agentId>:<rest>`）
- ✅ 支持多种 Session 类型（DM / Group / Thread / Sub-agent）
- ✅ 天然支持 Multi-Agent 隔离（通过 `agentId` 前缀）

**可借鉴**：

在设计多租户或多 Agent 系统时，使用结构化的 Key 格式可以简化权限控制和路由逻辑。

### 10.2 Session Store 的缓存 + 锁机制

**优点**：

- ✅ 45 秒 TTL 缓存减少磁盘 I/O
- ✅ Mtime 检查保证缓存一致性
- ✅ 文件锁保证并发写入安全

**可借鉴**：

在设计高频读写的配置/状态存储时，可以采用类似的"短 TTL 缓存 + Mtime 检查 + 文件锁"模式。

### 10.3 Session Reset 的灵活配置

**优点**：

- ✅ 支持多种 Reset 模式（Daily / Idle / 混合）
- ✅ 支持 Per-Type 和 Per-Channel 覆盖
- ✅ 用户可以手动触发 Reset（`/new`）

**可借鉴**：

在设计会话管理系统时，提供灵活的过期策略可以满足不同场景需求（例如客服系统、聊天机器人）。

### 10.4 跨 Session 通信的权限控制

**优点**：

- ✅ Agent-to-Agent Policy（显式配置跨 Agent 通信）
- ✅ Sandbox 可见性控制（`sessionToolsVisibility: "spawned"`）
- ✅ Send Policy（细粒度的发送权限控制）

**可借鉴**：

在设计多 Agent 或多租户系统时，需要考虑：

1. **默认拒绝**：跨边界通信默认禁止，需显式配置
2. **最小权限**：Sandbox 环境只能看到自己创建的资源
3. **运行时覆盖**：支持用户在运行时调整权限（例如 `/send off`）

### 10.5 Sub-agent 的嵌套限制

**优点**：

- ✅ 禁止 Sub-agent 再次 spawn（防止无限递归）
- ✅ `spawnedBy` 字段追踪父子关系
- ✅ Cleanup 策略（`keep` / `delete`）

**可借鉴**：

在设计递归任务系统时，需要设置嵌套深度限制，避免资源耗尽。

---

## 10.5 Multi-Agent 深度更新（⭐ 2026.3.22+ 代码）

### 10.5.1 两种 Spawn 路径（subagent vs ACP）

`sessions_spawn` 工具支持两种运行时路径，通过 `runtime` 参数控制：

```
sessions_spawn(runtime="subagent")   →  spawnSubagentDirect()
    └── src/agents/subagent-spawn.ts
    └── 走 Gateway WebSocket → sessions.create + agent 方法
    └── 在主 Gateway 进程内调度

sessions_spawn(runtime="acp")        →  spawnAcpDirect()
    └── src/agents/acp-spawn.ts
    └── 启动独立子进程（openclaw 进程），NDJSON over stdin/stdout
    └── Codex-style：适合完全隔离的 Agent 运行环境
    └── 支持 streamTo: "parent"（进度实时转发到父 Session）
```

**ACP 路径特点**：
- 创建独立进程，真正的进程级隔离
- 通过 `streamTo: "parent"` 可将子进程输出实时中继到父 Session
- 支持 `streamLogPath` 记录完整日志（可用 `tail` 查看历史）
- 内联 Attachments 不支持（仅 subagent 路径支持）

### 10.5.2 Subagent Registry（子 Agent 状态追踪）

**位置**：`src/agents/subagent-registry.ts`

Registry 负责**全生命周期追踪**所有子 Agent 运行状态：

```typescript
// 状态存储：内存 + 磁盘双写
// 内存：Map<runId, SubagentRunState>（快速查询）
// 磁盘：<workspaceDir>/.openclaw/subagent-runs/<runId>.json（持久化）

type SubagentRunState = {
  runId: string;
  requesterSessionKey: string;
  subagentSessionKey: string;
  status: "running" | "completed" | "failed" | "timeout";
  result?: string;
  error?: string;
  startedAt: number;
  completedAt?: number;
};
```

**核心流程**：
1. `registerSubagentRun()` — 记录 run 开始，双写内存+磁盘
2. 监听 Agent 生命周期事件（通过 `runtime.events.onAgentEvent`）
3. 子 Agent 完成 → `announceSubagentResult()` 将结果发送到父 Session
4. 支持重试：`LIFECYCLE_ERROR_RETRY_GRACE_MS` 容忍瞬态错误

**Announce 机制**：
子 Agent 完成后，Registry 会通过 `sessions.send` 将结果自动注入到父 Session 的消息队列，父 Agent 在下一次 LLM 调用时看到子任务结果。

### 10.5.3 Thread Binding（频道线程绑定）

当 `sessions_spawn` 使用 `thread: true` 时，子 Agent 的消息会绑定到特定的频道线程（如 Slack 线程、Discord 线程）：

```typescript
// 必须有 channel plugin 注册 subagent_spawning hook
if (!hookRunner.hasHooks("subagent_spawning")) {
  return {
    status: "error",
    error: "thread=true is unavailable because no channel plugin registered subagent_spawning hooks.",
  };
}
```

**Thread Binding 流程**：
```
1. 检查 channel plugin 是否注册了 subagent_spawning hook
2. 调用 runSubagentSpawning() → channel plugin 创建线程
3. 获取 threadId，绑定到子 Session
4. 子 Agent 消息发送到该 threadId
5. 子 Agent 结束 → runSubagentEnded() → channel plugin 清理
```

**需要频道支持**：Slack、Discord、Teams 等支持线程的频道可实现此功能；普通频道（Telegram 等）不支持。

### 10.5.4 `subagents` 工具（运行时管理）

`subagents` 工具（`src/agents/tools/subagents-tool.ts`）让父 Agent 可以在运行时管理子 Agent：

| action | 说明 | 参数 |
|--------|------|------|
| `list` | 列出当前所有子 Agent 及状态 | — |
| `kill` | 中止指定子 Agent | `sessionKey` |
| `steer` | 向运行中的子 Agent 发送新指令 | `sessionKey`, `message` |

**典型使用场景**：
```
主 Agent 发现子 Agent 方向跑偏
→ subagents({ action: "steer", sessionKey: "...", message: "请停止并返回已找到的结果" })
→ 子 Agent 收到 system message，调整行为
```

**与 subagent-registry 的关系**：`subagents` 工具通过查询 subagent-registry 获取状态列表，通过 `sessions.send` 实现 steer，通过 `sessions.abort` 实现 kill。

## 11. 未来研究方向

### 11.1 Session Compaction 机制

- **待研究**：`docs/concepts/compaction.md` 和 `docs/concepts/session-pruning.md`
- **关键问题**：
  - 如何决定何时触发 Compaction？
  - Compaction 如何生成摘要？
  - Pre-compaction Memory Flush 的完整流程？

### 11.2 Session Transcript 同步到 Memory

- **待研究**：`src/memory/sync-session-files.ts` 的完整实现
- **关键问题**：
  - File Watcher 的监控粒度？
  - 增量索引策略？
  - Embedding 生成的批处理逻辑？

### 11.3 Multi-Agent Routing

- **待研究**：`docs/concepts/multi-agent.md` 和 `src/routing/` 目录
- **关键问题**：
  - 如何根据消息内容路由到不同 Agent？
  - 多 Agent 之间的 Session 隔离如何实现？
  - Agent Bindings 的配置和使用？

---

## 总结

OpenClaw 的 Session 管理系统是一个**工程化程度极高**的设计，核心特点包括：

1. **结构化 Session Key**：统一格式，支持多种 Session 类型，天然支持 Multi-Agent
2. **灵活的 Reset 策略**：Daily / Idle / 混合模式，Per-Type 和 Per-Channel 覆盖
3. **严格的权限控制**：Agent-to-Agent Policy、Sandbox 可见性、Send Policy
4. **高性能存储**：TTL 缓存 + Mtime 检查 + 文件锁，减少磁盘 I/O
5. **跨 Session 通信**：完善的 Sessions Tools（list / send / history / spawn）

这套设计为 AI Agent 的"主动性"提供了坚实的基础设施：

- **Heartbeat** 在 Main Session 中定期执行
- **Memory** 自动索引 Session Transcript
- **Skills** 在 Agent 级别共享，Session 保存快照

OpenClaw 的 Session 管理不仅是一个"会话管理器"，更是一个**多 Agent 协作的通信层**，值得深入学习和借鉴。

---

**研究完成于**: 2026-03-06
**研究者**: 璇玑 ✨
