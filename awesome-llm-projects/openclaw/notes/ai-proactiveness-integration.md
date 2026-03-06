# OpenClaw AI 主动性深度解析 - Heartbeat/Memory/Skills 如何集成到 Agent Loop

> **研究版本**: Git Submodule `awesome-llm-projects/openclaw/openclaw`  
> **Commit**: `73055728318df378c831950cd01fb7c875a33790` (2026-03-05, version 2026.3.3)  
> **研究日期**: 2026-03-06  
> **核心目标**: 深度剖析 OpenClaw 最具创新性的"AI 主动性三大特性"，理解如何将 AI 从"被动响应的工具"进化为"主动察觉的助手"

---

## 📋 核心理念：从被动到主动

传统 AI Assistant 的工作模式：
```
User: "帮我检查邮件"
AI: [执行] → [返回结果]
```

OpenClaw 的主动性模式：
```
[Heartbeat 定时醒来]
AI: [自动检查邮件] → [发现重要邮件] → 主动提醒用户
```

**三大特性协同**：
- **Heartbeat**：定时"醒来"执行检查（主动感知）
- **Memory**：自动管理长期记忆（主动学习）
- **Skills**：持续发现和安装新能力（主动进化）

---

## 🔥 特性 1: Heartbeat - AI 主动"醒来"

### 核心设计理念

**问题**：传统 AI 只在用户发消息时才响应，无法主动提醒用户重要事项。

**解决方案**：OpenClaw 通过 Heartbeat 机制，让 AI **定期主动执行 Agent run**，检查是否有需要关注的事项，如果有则主动发送消息。

**关键创新点**：
1. **智能静默（HEARTBEAT_OK）**：如果检查后"一切正常"，不打扰用户
2. **用户可配置的检查清单（HEARTBEAT.md）**：用户控制 AI 检查什么
3. **Active Hours 时区感知**：只在用户活跃时段执行，避免深夜打扰

---

### 1.1 Heartbeat Runner - 定时调度器

**核心文件**：`src/infra/heartbeat-runner.ts:760-904`

**启动流程**：

```typescript
// src/infra/heartbeat-runner.ts:760-804
export function startHeartbeatRunner(opts: {
  cfg?: OpenClawConfig;
  runtime?: RuntimeEnv;
  abortSignal?: AbortSignal;
  runOnce?: typeof runHeartbeatOnce;
}): HeartbeatRunner {
  const state = {
    cfg: opts.cfg ?? loadConfig(),
    runtime,
    agents: new Map<string, HeartbeatAgentState>(), // 每个 agent 的 heartbeat 状态
    timer: null as NodeJS.Timeout | null,
    stopped: false,
  };

  // 计算下次执行时间
  const resolveNextDue = (now: number, intervalMs: number, prevState?: HeartbeatAgentState) => {
    if (typeof prevState?.lastRunMs === "number") {
      return prevState.lastRunMs + intervalMs;  // 基于上次执行时间
    }
    if (prevState && prevState.intervalMs === intervalMs && prevState.nextDueMs > now) {
      return prevState.nextDueMs;  // 复用之前计算的 nextDue
    }
    return now + intervalMs;  // 首次执行
  };

  // 调度下一次执行
  const scheduleNext = () => {
    if (state.stopped) return;
    if (state.timer) {
      clearTimeout(state.timer);
      state.timer = null;
    }
    if (state.agents.size === 0) return;
    const now = Date.now();
    let nextDue = Number.POSITIVE_INFINITY;
    for (const agent of state.agents.values()) {
      if (agent.nextDueMs < nextDue) nextDue = agent.nextDueMs;  // 找最早需要执行的
    }
    if (!Number.isFinite(nextDue)) return;
    const delay = Math.max(0, nextDue - now);
    state.timer = setTimeout(() => {
      requestHeartbeatNow({ reason: "interval", coalesceMs: 0 });
    }, delay);
    state.timer.unref?.();  // 不阻塞进程退出
  };

  // ...
}
```

**状态管理**（`HeartbeatAgentState`）：

```typescript
// src/infra/heartbeat-runner.ts:174-180
type HeartbeatAgentState = {
  agentId: string;
  heartbeat?: HeartbeatConfig;
  intervalMs: number;       // 执行间隔（如 30 分钟）
  lastRunMs?: number;       // 上次执行时间
  nextDueMs: number;        // 下次应该执行的时间
};
```

**多 Agent 支持**（`src/infra/heartbeat-runner.ts:263-276`）：

```typescript
function resolveHeartbeatAgents(cfg: OpenClawConfig): HeartbeatAgent[] {
  const list = cfg.agents?.list ?? [];
  if (hasExplicitHeartbeatAgents(cfg)) {
    // 如果配置了显式 heartbeat，只启用这些 agents
    return list
      .filter((entry) => entry?.heartbeat)
      .map((entry) => {
        const id = normalizeAgentId(entry.id);
        return { agentId: id, heartbeat: resolveHeartbeatConfig(cfg, id) };
      })
      .filter((entry) => entry.agentId);
  }
  // 否则，只为 default agent 启用 heartbeat
  const fallbackId = resolveDefaultAgentId(cfg);
  return [{ agentId: fallbackId, heartbeat: resolveHeartbeatConfig(cfg, fallbackId) }];
}
```

---

### 1.2 Heartbeat 执行流程 - runHeartbeatOnce

**核心文件**：`src/infra/heartbeat-runner.ts:433-758`

**完整流程**：

```
1. 前置检查（Skip 条件）
   ↓
2. 读取 HEARTBEAT.md（可选）
   ↓
3. 解析 Session 和 Delivery Target
   ↓
4. 调用 getReplyFromConfig() → Agent Loop 执行
   ↓
5. 解析回复（检测 HEARTBEAT_OK token）
   ↓
6. 决定是否发送消息
   ↓
7. 发送到 Channel（如果需要）
```

**前置检查（Skip 条件）**（`src/infra/heartbeat-runner.ts:443-482`）：

```typescript
export async function runHeartbeatOnce(opts: {
  cfg?: OpenClawConfig;
  agentId?: string;
  heartbeat?: HeartbeatConfig;
  reason?: string;
  deps?: HeartbeatDeps;
}): Promise<HeartbeatRunResult> {
  const cfg = opts.cfg ?? loadConfig();
  const agentId = normalizeAgentId(opts.agentId ?? resolveDefaultAgentId(cfg));
  const heartbeat = opts.heartbeat ?? resolveHeartbeatConfig(cfg, agentId);
  
  // Skip 1: 全局禁用
  if (!heartbeatsEnabled) {
    return { status: "skipped", reason: "disabled" };
  }
  
  // Skip 2: Agent 未启用 heartbeat
  if (!isHeartbeatEnabledForAgent(cfg, agentId)) {
    return { status: "skipped", reason: "disabled" };
  }
  
  // Skip 3: interval 未设置或为 0
  if (!resolveHeartbeatIntervalMs(cfg, undefined, heartbeat)) {
    return { status: "skipped", reason: "disabled" };
  }

  const startedAt = opts.deps?.nowMs?.() ?? Date.now();
  
  // Skip 4: 不在 Active Hours 内（如深夜 23:00-08:00）
  if (!isWithinActiveHours(cfg, heartbeat, startedAt)) {
    return { status: "skipped", reason: "quiet-hours" };
  }

  // Skip 5: 主队列有任务在执行（避免打断用户正在进行的对话）
  const queueSize = (opts.deps?.getQueueSize ?? getQueueSize)(CommandLane.Main);
  if (queueSize > 0) {
    return { status: "skipped", reason: "requests-in-flight" };
  }

  // Skip 6: HEARTBEAT.md 存在但内容为空（节省 API 调用）
  const isExecEventReason = opts.reason === "exec-event";
  const workspaceDir = resolveAgentWorkspaceDir(cfg, agentId);
  const heartbeatFilePath = path.join(workspaceDir, DEFAULT_HEARTBEAT_FILENAME);
  try {
    const heartbeatFileContent = await fs.readFile(heartbeatFilePath, "utf-8");
    if (isHeartbeatContentEffectivelyEmpty(heartbeatFileContent) && !isExecEventReason) {
      emitHeartbeatEvent({
        status: "skipped",
        reason: "empty-heartbeat-file",
        durationMs: Date.now() - startedAt,
      });
      return { status: "skipped", reason: "empty-heartbeat-file" };
    }
  } catch {
    // 文件不存在或无法读取 → 继续执行 heartbeat（LLM 会处理）
  }

  // ... 继续执行 heartbeat
}
```

**HEARTBEAT.md 有效性检查**（`src/auto-reply/heartbeat.ts:22-42`）：

```typescript
export function isHeartbeatContentEffectivelyEmpty(content: string | undefined | null): boolean {
  if (content === undefined || content === null) return false;
  if (typeof content !== "string") return false;

  const lines = content.split("\n");
  for (const line of lines) {
    const trimmed = line.trim();
    // 跳过空行
    if (!trimmed) continue;
    // 跳过 Markdown 标题（# Heading）
    if (/^#+(\s|$)/.test(trimmed)) continue;
    // 跳过空的列表项（- [ ]）
    if (/^[-*+]\s*(\[[\sXx]?\]\s*)?$/.test(trimmed)) continue;
    // 发现非空、非注释行 → 有实际内容
    return false;
  }
  // 所有行都是空行或注释
  return true;
}
```

**示例**：
```markdown
# Heartbeat Checklist

- [ ] 检查邮件
- [ ] 检查日历

<!-- 上面这个文件会被认为"有效"（有实际任务） -->
```

```markdown
# Heartbeat Checklist

<!-- 空的清单 -->

<!-- 上面这个文件会被认为"无效"（只有标题和注释） -->
```

---

### 1.3 Agent Loop 执行 - getReplyFromConfig

**代码位置**：`src/infra/heartbeat-runner.ts:505-550`

```typescript
// 检查是否为 exec event（异步命令完成）
const isExecEvent = opts.reason === "exec-event";
const pendingEvents = isExecEvent ? peekSystemEvents(sessionKey) : [];
const hasExecCompletion = pendingEvents.some((evt) => evt.includes("Exec finished"));

// 如果是 exec completion，使用特殊 prompt（让 AI 转述执行结果）
// 否则使用正常的 heartbeat prompt
const prompt = hasExecCompletion ? EXEC_EVENT_PROMPT : resolveHeartbeatPrompt(cfg, heartbeat);

const ctx = {
  Body: prompt,
  From: sender,
  To: sender,
  Provider: hasExecCompletion ? "exec-event" : "heartbeat",
  SessionKey: sessionKey,
};

// 调用 Agent Loop（核心！）
const replyResult = await getReplyFromConfig(ctx, { isHeartbeat: true }, cfg);
```

**getReplyFromConfig 内部流程**（推测自 `src/auto-reply/reply.ts`）：

```
getReplyFromConfig
  ↓
resolveAgentForReply (选择 agent)
  ↓
runEmbeddedPiAgent (执行 Agent Loop)
  ↓
返回 ReplyPayload[]
```

**关键点**：
- **isHeartbeat: true** 标记：告诉 Agent Loop 这是 heartbeat run
- **SessionKey**：Heartbeat 默认运行在 main session（`agent:main:main`）
- **Prompt**：可以是默认的 "Read HEARTBEAT.md..." 或用户自定义的 prompt

---

### 1.4 HEARTBEAT_OK 智能静默机制

**核心文件**：`src/auto-reply/heartbeat.ts:82-139`

**设计思路**：

如果 AI 检查后发现"一切正常"，应该返回 `HEARTBEAT_OK` token，而不是发送实际消息给用户。这样可以避免每 30 分钟就收到一条"没事"的消息，用户会被烦死的。

**Token 剥离逻辑**：

```typescript
export function stripHeartbeatToken(
  raw?: string,
  opts: { mode?: StripHeartbeatMode; maxAckChars?: number } = {},
) {
  if (!raw) return { shouldSkip: true, text: "", didStrip: false };
  const trimmed = raw.trim();
  if (!trimmed) return { shouldSkip: true, text: "", didStrip: false };

  const mode: StripHeartbeatMode = opts.mode ?? "message";
  const maxAckChars = Math.max(
    0,
    typeof opts.maxAckChars === "number" ? opts.maxAckChars : DEFAULT_HEARTBEAT_ACK_MAX_CHARS,
  );

  // 归一化标记（剥离 HTML/Markdown wrapper）
  const stripMarkup = (text: string) =>
    text
      .replace(/<[^>]*>/g, " ")       // 移除 HTML 标签
      .replace(/&nbsp;/gi, " ")        // 移除 &nbsp;
      .replace(/^[*`~_]+/, "")         // 移除开头的 Markdown 标记
      .replace(/[*`~_]+$/, "");        // 移除结尾的 Markdown 标记

  const trimmedNormalized = stripMarkup(trimmed);
  const hasToken = trimmed.includes(HEARTBEAT_TOKEN) || trimmedNormalized.includes(HEARTBEAT_TOKEN);
  if (!hasToken) {
    return { shouldSkip: false, text: trimmed, didStrip: false };
  }

  // 剥离 token（从开头和结尾）
  const strippedOriginal = stripTokenAtEdges(trimmed);
  const strippedNormalized = stripTokenAtEdges(trimmedNormalized);
  const picked = strippedOriginal.didStrip && strippedOriginal.text ? strippedOriginal : strippedNormalized;
  
  if (!picked.didStrip) {
    return { shouldSkip: false, text: trimmed, didStrip: false };
  }

  if (!picked.text) {
    return { shouldSkip: true, text: "", didStrip: true };
  }

  const rest = picked.text.trim();
  
  // Heartbeat 模式：如果剥离后剩余文本 <= maxAckChars，跳过发送
  if (mode === "heartbeat") {
    if (rest.length <= maxAckChars) {
      return { shouldSkip: true, text: "", didStrip: true };
    }
  }

  return { shouldSkip: false, text: rest, didStrip: true };
}
```

**示例场景**：

| AI 回复 | maxAckChars | 处理结果 |
|---------|-------------|----------|
| `HEARTBEAT_OK` | 300 | shouldSkip: true（不发送） |
| `HEARTBEAT_OK All good` | 300 | shouldSkip: true（"All good" < 300 chars） |
| `HEARTBEAT_OK 发现重要邮件：客户询问产品定价...` | 300 | shouldSkip: false（实际内容 > 300 chars，发送） |
| `检查完毕，没有问题 HEARTBEAT_OK` | 300 | shouldSkip: true（剥离后 "检查完毕，没有问题" < 300 chars） |

**代码位置（应用）**（`src/infra/heartbeat-runner.ts:578-609`）：

```typescript
const ackMaxChars = resolveHeartbeatAckMaxChars(cfg, heartbeat);
const normalized = normalizeHeartbeatReply(replyPayload, responsePrefix, ackMaxChars);

// 对于 exec completion 事件，不跳过（即使看起来像 HEARTBEAT_OK）
const execFallbackText =
  hasExecCompletion && !normalized.text.trim() && replyPayload.text?.trim()
    ? replyPayload.text.trim()
    : null;
if (execFallbackText) {
  normalized.text = execFallbackText;
  normalized.shouldSkip = false;
}

const shouldSkipMain = normalized.shouldSkip && !normalized.hasMedia && !hasExecCompletion;
if (shouldSkipMain && reasoningPayloads.length === 0) {
  await restoreHeartbeatUpdatedAt({
    storePath,
    sessionKey,
    updatedAt: previousUpdatedAt,
  });
  const okSent = await maybeSendHeartbeatOk();  // 可选：发送 HEARTBEAT_OK 确认消息
  emitHeartbeatEvent({
    status: "ok-token",
    reason: opts.reason,
    durationMs: Date.now() - startedAt,
    channel: delivery.channel !== "none" ? delivery.channel : undefined,
    silent: !okSent,
    indicatorType: visibility.useIndicator ? resolveIndicatorType("ok-token") : undefined,
  });
  return { status: "ran", durationMs: Date.now() - startedAt };
}
```

---

### 1.5 Heartbeat 与 Agent Loop 的集成点

**集成位置 1：System Prompt 注入**（`src/agents/pi-embedded-runner/run/attempt.ts:344-346`）：

```typescript
heartbeatPrompt: isDefaultAgent
  ? resolveHeartbeatPrompt(params.config?.agents?.defaults?.heartbeat?.prompt)
  : undefined,
```

**System Prompt 中的 Heartbeat 部分**（推测，基于 `docs/reference/templates/AGENTS.md`）：

```markdown
## 💓 Heartbeats - Be Proactive!

When you receive a heartbeat poll (message matches the configured heartbeat prompt), don't just reply `HEARTBEAT_OK` every time. Use heartbeats productively!

Default heartbeat prompt:
`Read HEARTBEAT.md if it exists (workspace context). Follow it strictly. Do not infer or repeat old tasks from prior chats. If nothing needs attention, reply HEARTBEAT_OK.`

You are free to edit `HEARTBEAT.md` with a short checklist or reminders. Keep it small to limit token burn.

### Heartbeat vs Cron: When to Use Each

**Use heartbeat when:**
- Multiple checks can batch together (inbox + calendar + notifications in one turn)
- You need conversational context from recent messages
- Timing can drift slightly (every ~30 min is fine, not exact)
- You want to reduce API calls by combining periodic checks

**Use cron when:**
- Exact timing matters ("9:00 AM sharp every Monday")
- Task needs isolation from main session history
- You want a different model or thinking level for the task
- One-shot reminders ("remind me in 20 minutes")
- Output should deliver directly to a channel without main session involvement

**Things to check (rotate through these, 2-4 times per day):**
- **Emails** - Any urgent unread messages?
- **Calendar** - Upcoming events in next 24-48h?
- **Mentions** - Twitter/social notifications?
- **Weather** - Relevant if your human might go out?

**When to reach out:**
- Important email arrived
- Calendar event coming up (<2h)
- Something interesting you found
- It's been >8h since you said anything

**When to stay quiet (HEARTBEAT_OK):**
- Late night (23:00-08:00) unless urgent
- Human is clearly busy
- Nothing new since last check
- You just checked <30 minutes ago
```

**集成位置 2：Queue Lane 隔离**：

Heartbeat 运行在 **Session Lane**（`session:main`），与普通用户消息串行执行，避免冲突。

```typescript
// src/agents/pi-embedded-runner/run.ts:73-74
const sessionLane = resolveSessionLane(params.sessionKey?.trim() || params.sessionId);
const globalLane = resolveGlobalLane(params.lane);
```

**集成位置 3：Heartbeat Run ID 标记**：

Heartbeat run 有特殊的 `Provider: "heartbeat"` 标记，用于：
- 日志区分（`log.info` 显示 "heartbeat" 来源）
- 统计和监控（Heartbeat 执行次数、成功率等）
- 错误处理（Heartbeat 失败不应影响正常对话）

---

### 1.5.1 Exec Heartbeat 路由（Session 作用域）

**核心文件**：`src/routing/session-key.ts:32-37`、`src/agents/bash-tools.exec-runtime.ts`、`src/gateway/server-node-events.ts:580`

**设计目标**：Exec 事件（异步命令完成）触发的 Heartbeat 应只唤醒对应的 Agent Session，避免无关 Agent 被误唤醒。

**实现**：`scopedHeartbeatWakeOptions` 函数根据 `sessionKey` 是否为规范格式（`agent:main:main`）决定是否在 wake 选项中附加 `sessionKey`：

```typescript
// src/routing/session-key.ts:32-37
export function scopedHeartbeatWakeOptions<T extends object>(
  sessionKey: string,
  wakeOptions: T,
): T | (T & { sessionKey: string }) {
  return parseAgentSessionKey(sessionKey) ? { ...wakeOptions, sessionKey } : wakeOptions;
}
```

**调用位置**：
- `src/agents/bash-tools.exec-runtime.ts:243,271`：exec 命令退出时调用 `requestHeartbeatNow(scopedHeartbeatWakeOptions(sessionKey, { reason: "exec-event" }))`
- `src/gateway/server-node-events.ts:580`：Node 生命周期事件（exec 完成）同样使用 scoped 调用

**CHANGELOG 引用**：Exec heartbeat routing: scope exec-triggered heartbeat wakes to agent session keys so unrelated agents are no longer awakened by exec events (#32724)

---

### 1.5.2 requestHeartbeatNow - 扩展 API

**核心文件**：`src/plugins/runtime/runtime-system.ts:1-14`、`src/plugins/runtime/types-core.ts:18`、`src/infra/heartbeat-wake.ts:228-241`

**API 签名**：`runtime.system.requestHeartbeatNow(opts?: { reason?: string; coalesceMs?: number; agentId?: string; sessionKey?: string })`

**用途**：扩展（Plugins）在入队系统事件后，可立即请求 Heartbeat 唤醒，让 Agent 尽快处理事件（如 exec 完成、通知到达等）。

**实现**：Plugin Runtime 直接暴露 `requestHeartbeatNow`（来自 `heartbeat-wake.js`）：

```typescript
// src/plugins/runtime/runtime-system.ts:7-13
export function createRuntimeSystem(): PluginRuntime["system"] {
  return {
    enqueueSystemEvent,
    requestHeartbeatNow,
    runCommandWithTimeout,
    formatNativeDependencyHint,
  };
}
```

**典型用法**：扩展在 `enqueueSystemEvent` 后调用 `requestHeartbeatNow({ reason: "exec-event", sessionKey })`，使目标 Session 的 Heartbeat 尽快执行。

**CHANGELOG 引用**：Plugin runtime/system: expose `runtime.system.requestHeartbeatNow(...)` so extensions can wake targeted sessions immediately after enqueueing system events (#19464)

---

### 1.6 Active Hours - 时区感知的智能调度

**核心文件**：`src/infra/heartbeat-runner.ts:100-172`

**设计目标**：只在用户活跃时段执行 Heartbeat，避免深夜打扰。

**配置示例**：

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",
        activeHours: {
          start: "08:00",       // 早上 8 点开始
          end: "23:00",         // 晚上 11 点结束
          timezone: "user"      // 使用用户时区（默认）
        }
      }
    }
  }
}
```

**Timezone 解析**（`src/infra/heartbeat-runner.ts:100-115`）：

```typescript
function resolveActiveHoursTimezone(cfg: OpenClawConfig, raw?: string): string {
  const trimmed = raw?.trim();
  if (!trimmed || trimmed === "user") {
    // 使用用户配置的时区（从 config.agents.defaults.userTimezone）
    return resolveUserTimezone(cfg.agents?.defaults?.userTimezone);
  }
  if (trimmed === "local") {
    // 使用 Gateway 所在主机的时区
    const host = Intl.DateTimeFormat().resolvedOptions().timeZone;
    return host?.trim() || "UTC";
  }
  // 验证时区是否有效
  try {
    new Intl.DateTimeFormat("en-US", { timeZone: trimmed }).format(new Date());
    return trimmed;
  } catch {
    return resolveUserTimezone(cfg.agents?.defaults?.userTimezone);
  }
}
```

**时间检查**（`src/infra/heartbeat-runner.ts:151-172`）：

```typescript
function isWithinActiveHours(
  cfg: OpenClawConfig,
  heartbeat?: HeartbeatConfig,
  nowMs?: number,
): boolean {
  const active = heartbeat?.activeHours;
  if (!active) return true;  // 未配置 → 全天候

  const startMin = parseActiveHoursTime({ allow24: false }, active.start);  // 如 "08:00" → 480
  const endMin = parseActiveHoursTime({ allow24: true }, active.end);        // 如 "23:00" → 1380
  if (startMin === null || endMin === null) return true;
  if (startMin === endMin) return true;  // 无效配置 → 全天候

  const timeZone = resolveActiveHoursTimezone(cfg, active.timezone);
  const currentMin = resolveMinutesInTimeZone(nowMs ?? Date.now(), timeZone);  // 当前时间（分钟）
  if (currentMin === null) return true;

  // 跨天情况（如 22:00 - 08:00）
  if (endMin > startMin) {
    return currentMin >= startMin && currentMin < endMin;
  }
  return currentMin >= startMin || currentMin < endMin;
}
```

**示例场景**：

| 配置 | 当前时间（用户时区） | 是否执行 Heartbeat |
|------|---------------------|-------------------|
| `08:00 - 23:00` | 10:00 | ✅ 是 |
| `08:00 - 23:00` | 02:00 | ❌ 否（深夜） |
| `22:00 - 08:00`（跨天） | 03:00 | ✅ 是（在范围内） |
| `22:00 - 08:00` | 15:00 | ❌ 否（白天不在范围） |

---

### 1.7 Heartbeat 去重机制 - 避免重复提醒

**问题**：如果 AI 检查后发现"有 3 封未读邮件"，第一次提醒了用户。30 分钟后再次检查，邮件还没处理，又提醒一次"有 3 封未读邮件"，会很烦人。

**解决方案**：记录上次发送的 heartbeat 文本，24 小时内不重复发送相同内容。

**代码位置**：`src/infra/heartbeat-runner.ts:614-643`

```typescript
// 记录上次 heartbeat 文本（去重用）
const prevHeartbeatText =
  typeof entry?.lastHeartbeatText === "string" ? entry.lastHeartbeatText : "";
const prevHeartbeatAt =
  typeof entry?.lastHeartbeatSentAt === "number" ? entry.lastHeartbeatSentAt : undefined;

const isDuplicateMain =
  !shouldSkipMain &&
  !mediaUrls.length &&  // 无媒体附件
  Boolean(prevHeartbeatText.trim()) &&
  normalized.text.trim() === prevHeartbeatText.trim() &&  // 文本完全相同
  typeof prevHeartbeatAt === "number" &&
  startedAt - prevHeartbeatAt < 24 * 60 * 60 * 1000;  // 24 小时内

if (isDuplicateMain) {
  await restoreHeartbeatUpdatedAt({ storePath, sessionKey, updatedAt: previousUpdatedAt });
  emitHeartbeatEvent({
    status: "skipped",
    reason: "duplicate",
    preview: normalized.text.slice(0, 200),
    durationMs: Date.now() - startedAt,
    hasMedia: false,
    channel: delivery.channel !== "none" ? delivery.channel : undefined,
  });
  return { status: "ran", durationMs: Date.now() - startedAt };
}

// ... 发送消息 ...

// 记录本次发送的文本（用于下次去重）
if (!shouldSkipMain && normalized.text.trim()) {
  const store = loadSessionStore(storePath);
  const current = store[sessionKey];
  if (current) {
    store[sessionKey] = {
      ...current,
      lastHeartbeatText: normalized.text,
      lastHeartbeatSentAt: startedAt,
    };
    await saveSessionStore(storePath, store);
  }
}
```

---

### 1.8 Heartbeat 配置全景

**配置层级**（优先级从高到低）：

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",                 // 执行间隔
        model: "anthropic/claude-opus-4-5",  // 可选：指定模型
        includeReasoning: false,      // 是否发送 Reasoning 消息
        target: "last",               // 发送目标：last / none / channel-id
        to: "+15551234567",           // 可选：覆盖收件人
        prompt: "...",                // 自定义 prompt（覆盖默认）
        ackMaxChars: 300,             // HEARTBEAT_OK 最大 ack 字符数
        session: "main",              // 可选：指定 session key
        activeHours: {
          start: "08:00",
          end: "23:00",
          timezone: "user"            // user / local / IANA timezone
        }
      }
    },
    list: [
      { id: "main", default: true },
      {
        id: "ops",
        heartbeat: {                  // 单独为 ops agent 配置 heartbeat
          every: "1h",
          target: "whatsapp",
          to: "+15551234567"
        }
      }
    ]
  },
  channels: {
    defaults: {
      heartbeat: {
        showOk: false,                // 是否发送 HEARTBEAT_OK 确认
        showAlerts: true,             // 是否发送实际警报
        useIndicator: true            // 是否发出 indicator 事件
      }
    },
    telegram: {
      heartbeat: {
        showOk: true                  // Telegram 显示 OK 确认
      }
    }
  }
}
```

---

## 📎 补充：工具结果截断、Bootstrap 警告与 Session 日期锚定

### 工具结果截断（Head+Tail）

**核心文件**：`src/agents/pi-embedded-runner/tool-result-truncation.ts`、`src/agents/session-tool-result-guard.ts`

**设计目标**：超大 tool result 会占满 context window，需在持久化前截断，避免后续 LLM 调用失败。

**Head+Tail 策略**（`tool-result-truncation.ts:70-112`）：

1. **默认**：保留开头，超出 `maxChars` 时截断并追加 suffix
2. **智能 Head+Tail**：当尾部包含重要内容（error、exception、JSON 闭合、summary 等）时，将预算分配给 head 和 tail，中间用 `MIDDLE_OMISSION_MARKER` 省略

```typescript
// src/agents/pi-embedded-runner/tool-result-truncation.ts:84-102
if (hasImportantTail(text) && budget > minKeepChars * 2) {
  const tailBudget = Math.min(Math.floor(budget * 0.3), 4_000);
  const headBudget = budget - tailBudget - MIDDLE_OMISSION_MARKER.length;
  // ...
  return text.slice(0, headCut) + MIDDLE_OMISSION_MARKER + text.slice(tailStart) + suffix;
}
```

**持久化时应用**（`session-tool-result-guard.ts:189-193`）：`capToolResultSize` 在 `appendMessage` 写入前调用 `truncateToolResultMessage`，使用 `HARD_MAX_TOOL_RESULT_CHARS`（400_000）和 `minKeepChars: 2_000`。

**两处截断**：
- **LLM 调用前**：`pi-embedded-subscribe.tools.ts` 的 `sanitizeToolResult` 对单次 tool 返回做 8000 字符截断（流式场景）
- **Session 持久化**：`session-tool-result-guard` 对写入 JSONL 的 toolResult 做 head+tail 截断（400K 硬上限）

---

### Bootstrap 截断警告（bootstrapPromptTruncationWarning）

**配置项**：`agents.defaults.bootstrapPromptTruncationWarning`（`off` | `once` | `always`，默认 `once`）

**核心文件**：`src/config/zod-schema.agent-defaults.ts:43-45`、`src/agents/bootstrap-budget.ts`、`src/agents/pi-embedded-helpers/bootstrap.ts:115-123`

**用途**：当 Bootstrap 上下文（AGENTS.md、TOOLS.md 等）因 `bootstrapMaxChars`/`bootstrapTotalMaxChars` 被截断时，在 System Prompt 中注入警告文本，告知模型有内容被截断。

**模式**：
- `off`：不注入警告
- `once`：对同一截断签名只显示一次（推荐，避免重复打扰）
- `always`：每次有截断都显示

**实现**：`resolveBootstrapPromptTruncationWarningMode` 读取配置；`buildBootstrapTruncationSignature` 生成签名；`formatBootstrapTruncationWarningLines` 生成警告行；通过 `bootstrapTruncationReportMeta` 持久化 `warningSignaturesSeen` 实现 once 去重。

**文档**：`docs/gateway/configuration-reference.md:806-818`

---

### Session 启动日期锚定（YYYY-MM-DD 占位符替换）

**核心文件**：`src/auto-reply/reply/post-compaction-context.ts:26-76`、`src/auto-reply/reply/memory-flush.ts:49-50`

**设计目标**：AGENTS.md 中常引用 `memory/YYYY-MM-DD.md`，若直接使用字面量，模型可能猜错日期。在 Session 启动/压缩后注入的上下文中，将 `YYYY-MM-DD` 替换为运行时日期。

**替换位置**：
1. **Post-compaction 上下文**：`readPostCompactionContext` 从 AGENTS.md 提取 "Session Startup"、"Red Lines" 等段落，`replaceAll("YYYY-MM-DD", dateStamp)` 后注入，并追加 `Current time:` 行
2. **Memory flush prompt**：`memory-flush.ts:50` 对 `params.prompt` 做 `replaceAll("YYYY-MM-DD", dateStamp)`

**时区**：使用 `resolveUserTimezone(cfg?.agents?.defaults?.userTimezone)` 格式化日期。

**CHANGELOG 引用**：Agents/Session startup date grounding: substitute `YYYY-MM-DD` placeholders in startup/post-compaction AGENTS context and append runtime current-time lines for `/new` and `/reset` prompts (#32381)

---

## 🧠 特性 2: Memory - AI 主动管理记忆

### 核心设计理念

**问题**：传统 AI 的"记忆"只存在于对话历史中，一旦 session 被压缩或创建新 session，之前的信息就丢失了。

**解决方案**：OpenClaw 通过 Memory 系统，**自动监控 Workspace 文件变化**，将重要文件（MEMORY.md、memory/*.md、Session Transcript）自动索引到向量数据库，支持语义搜索和关键词搜索。

**关键创新点**：
1. **File Watcher 自动索引**：无需手动触发，文件变化自动触发索引更新
2. **Hybrid Search**：Vector Semantic Search + FTS5 Keyword Search，提升召回率
3. **Session Transcript 自动同步**：对话历史自动索引，可用 memory_search 搜索过往对话
4. **Embedding Cache Dedupe**：相同内容的 embedding 缓存和去重，节省 API 调用

---

### 2.1 Memory Tool - memory_search

**核心文件**：`src/agents/tools/memory-tool.ts:22-69`

**Tool Schema**：

```typescript
const MemorySearchSchema = Type.Object({
  query: Type.String(),
  maxResults: Type.Optional(Type.Number()),
  minScore: Type.Optional(Type.Number()),
});
```

**Tool 实现**：

```typescript
export function createMemorySearchTool(options: {
  config?: OpenClawConfig;
  agentSessionKey?: string;
}): AnyAgentTool | null {
  const cfg = options.config;
  if (!cfg) return null;
  const agentId = resolveSessionAgentId({
    sessionKey: options.agentSessionKey,
    config: cfg,
  });
  
  // 检查是否启用 Memory Search
  if (!resolveMemorySearchConfig(cfg, agentId)) return null;
  
  return {
    label: "Memory Search",
    name: "memory_search",
    description:
      "Mandatory recall step: semantically search MEMORY.md + memory/*.md (and optional session transcripts) before answering questions about prior work, decisions, dates, people, preferences, or todos; returns top snippets with path + lines.",
    parameters: MemorySearchSchema,
    execute: async (_toolCallId, params) => {
      const query = readStringParam(params, "query", { required: true });
      const maxResults = readNumberParam(params, "maxResults");
      const minScore = readNumberParam(params, "minScore");
      
      // 获取 Memory Index Manager
      const { manager, error } = await getMemorySearchManager({
        cfg,
        agentId,
      });
      if (!manager) {
        return jsonResult({ results: [], disabled: true, error });
      }
      
      try {
        // 执行搜索（Hybrid Search）
        const results = await manager.search(query, {
          maxResults,
          minScore,
          sessionKey: options.agentSessionKey,
        });
        const status = manager.status();
        return jsonResult({
          results,
          provider: status.provider,
          model: status.model,
          fallback: status.fallback,
        });
      } catch (err) {
        const message = err instanceof Error ? err.message : String(err);
        return jsonResult({ results: [], disabled: true, error: message });
      }
    },
  };
}
```

**Tool Description 解读**：

> "Mandatory recall step: semantically search MEMORY.md + memory/*.md (and optional session transcripts) before answering questions about prior work, decisions, dates, people, preferences, or todos; returns top snippets with path + lines."

**关键词**：
- **Mandatory recall step**：强制性召回步骤（System Prompt 中会指示 Agent 在回答特定问题前必须调用 memory_search）
- **Semantically search**：语义搜索（不是简单的关键词匹配）
- **MEMORY.md + memory/*.md**：搜索范围（用户管理的长期记忆文件）
- **Session transcripts**：可选的对话历史搜索
- **Returns top snippets with path + lines**：返回匹配片段 + 文件路径 + 行号

---

### 2.2 Memory Index Manager 架构

**核心文件**：`src/memory/manager.ts`（推测，未完整读取）

**架构层次**：

```
File Watcher (chokidar)
  ↓ 检测文件变化
File Reader
  ↓ 读取文件内容
Chunker
  ↓ 分块（Chunking）
Embedding Provider (OpenAI/Gemini/Local)
  ↓ 生成 embedding 向量
SQLite + sqlite-vec
  ↓ 存储 embedding 向量
FTS5 (Full-Text Search)
  ↓ 存储关键词索引
Hybrid Search
  ↓ 组合 Vector Search + FTS5 Search
返回 Top-K 结果
```

**监控的文件**（推测自 Memory 配置）：
- `MEMORY.md`：用户的长期记忆文件（主文件）
- `memory/*.md`：用户的日志文件（按日期组织，如 `memory/2026-01-30.md`）
- **Session Transcript** (`~/.openclaw/agents/<agentId>/sessions/*.jsonl`)：对话历史
- 配置的额外路径（`memorySearch.extraPaths`）

---

### 2.3 File Watcher 自动索引机制

**问题**：如何知道文件被修改了？

**解决方案**：使用 `chokidar` 监控文件系统变化。

**推测实现**（基于 Skills 的 File Watcher 模式）：

```typescript
// 类似 src/agents/skills/refresh.ts:131-167
const watcher = chokidar.watch(memoryPaths, {
  ignoreInitial: true,  // 忽略初始文件扫描
  awaitWriteFinish: {
    stabilityThreshold: 250,  // 文件稳定后 250ms 触发
    pollInterval: 100,
  },
  ignored: DEFAULT_MEMORY_WATCH_IGNORED,  // 忽略 .git、node_modules 等
});

watcher.on("add", (path) => {
  scheduleReindex(path);  // 新文件 → 添加到索引
});
watcher.on("change", (path) => {
  scheduleReindex(path);  // 文件修改 → 增量更新索引
});
watcher.on("unlink", (path) => {
  removeFromIndex(path);  // 文件删除 → 从索引移除
});
```

**增量更新策略**：

```
文件修改 → 读取新内容 → Chunking
  ↓
对比 old chunks vs new chunks
  ↓
删除旧 chunks（从 SQLite 删除）
  ↓
插入新 chunks（生成 embedding → SQLite 插入）
```

---

### 2.4 Session Transcript 自动同步

**核心文件**：`src/memory/sync-session-files.ts`（推测）

**设计目标**：对话历史（Session Transcript）也应该可搜索，用户可以问"我上周让你做了什么？"

**触发时机**：
- **Agent run 完成后**：`SessionManager.flush()` 写入 `session.jsonl`
- **File Watcher 检测到 session.jsonl 变化** → 触发索引更新

**Transcript 格式**（`session.jsonl`）：

```jsonl
{"type":"message","role":"user","content":"帮我检查邮件"}
{"type":"message","role":"assistant","content":"正在检查邮件..."}
{"type":"tool_call","toolName":"bash","params":{"command":"..."},"result":"..."}
{"type":"message","role":"assistant","content":"发现 3 封未读邮件：..."}
```

**索引内容**：
- **User messages**：用户说了什么
- **Assistant messages**：AI 回复了什么
- **Tool calls**：执行了哪些工具（可选，根据配置）

**去重策略**：
- 相同 message 不重复索引（根据 message ID 或 content hash）

---

### 2.5 Hybrid Search - Vector + FTS5 组合

**核心文件**：`src/memory/hybrid.ts`（推测）

**设计理念**：

| 搜索类型 | 优势 | 劣势 |
|---------|------|------|
| **Vector Search** | 语义匹配，理解同义词（"dog" 匹配 "puppy"） | 关键词精确匹配差（"GPT-4" 可能匹配不到） |
| **FTS5 Keyword Search** | 精确关键词匹配（"GPT-4" 100% 匹配） | 无法理解同义词（"dog" 不匹配 "puppy"） |

**Hybrid Search**：同时使用两种搜索，取并集 + 重排序。

**实现流程**（推测）：

```typescript
async function hybridSearch(query: string, maxResults: number, minScore: number) {
  // 1. Vector Search
  const vectorResults = await vectorSearch(query, maxResults * 2);  // 多召回一些
  
  // 2. FTS5 Keyword Search
  const keywordResults = await fts5Search(query, maxResults * 2);
  
  // 3. 合并结果（去重 + 重排序）
  const merged = mergeAndRerank(vectorResults, keywordResults);
  
  // 4. 按分数过滤 + 截断
  return merged
    .filter(result => result.score >= minScore)
    .slice(0, maxResults);
}
```

**评分组合策略**（推测）：

```typescript
// 归一化分数（Vector Search 和 FTS5 的分数范围不同）
const normalizedVectorScore = vectorScore / maxVectorScore;
const normalizedFts5Score = fts5Score / maxFts5Score;

// 加权平均（可配置权重）
const finalScore = 
  vectorWeight * normalizedVectorScore + 
  fts5Weight * normalizedFts5Score;

// 或者：取最大值（Reciprocal Rank Fusion）
const finalScore = Math.max(normalizedVectorScore, normalizedFts5Score);
```

---

### 2.6 Embedding Provider 与 Batch API

**支持的 Embedding Providers**（推测自 `src/memory/embeddings*.ts`）：

1. **OpenAI Embeddings**（`text-embedding-3-small`）
2. **Gemini Embeddings**（`text-embedding-004`）
3. **Local Embeddings**（`node-llama` / `llama.cpp`）

**Batch API 优化**（`src/memory/openai-batch.ts`、`src/memory/batch-gemini.ts`）：

**问题**：如果每个 chunk 单独调用 Embedding API，会有大量网络请求，延迟高。

**解决方案**：批量提交 embedding 请求，一次 API 调用生成多个 chunk 的 embedding。

**OpenAI Batch API**（推测）：

```typescript
// 一次请求生成多个 embedding
const response = await openai.embeddings.create({
  model: "text-embedding-3-small",
  input: [
    "Chunk 1 content...",
    "Chunk 2 content...",
    "Chunk 3 content...",
    // ... 最多 2048 个 chunks
  ],
});

// response.data = [
//   { embedding: [0.123, 0.456, ...], index: 0 },
//   { embedding: [0.789, 0.012, ...], index: 1 },
//   { embedding: [0.345, 0.678, ...], index: 2 },
// ]
```

**Gemini Batch API**（类似）：

```typescript
const response = await gemini.batchEmbedContent({
  requests: chunks.map(content => ({
    model: "models/text-embedding-004",
    content: { parts: [{ text: content }] },
  })),
});
```

---

### 2.7 Embedding Cache Dedupe

**核心文件**：`src/memory/manager-cache-key.ts`（推测）

**设计目标**：相同内容不重复生成 embedding，节省 API 调用和时间。

**Cache Key 生成**（推测）：

```typescript
function buildCacheKey(content: string, provider: string, model: string): string {
  const hash = sha256(content);
  return `embedding:${provider}:${model}:${hash}`;
}
```

**Cache 存储**（推测）：
- **SQLite 表**：`embedding_cache(cache_key TEXT PRIMARY KEY, embedding BLOB, created_at INTEGER)`
- **LRU 策略**：定期清理超过 N 天的缓存

**使用流程**：

```typescript
async function getOrCreateEmbedding(content: string) {
  const cacheKey = buildCacheKey(content, provider, model);
  
  // 1. 查询缓存
  const cached = await getCachedEmbedding(cacheKey);
  if (cached) return cached;
  
  // 2. 调用 API 生成 embedding
  const embedding = await generateEmbedding(content);
  
  // 3. 存入缓存
  await setCachedEmbedding(cacheKey, embedding);
  
  return embedding;
}
```

---

### 2.8 Memory 与 Agent Loop 的集成点

**集成位置 1：Tool 注册**（`src/agents/openclaw-tools.ts`）：

```typescript
// 创建 Memory Search Tool（如果启用）
const memorySearchTool = createMemorySearchTool({
  config,
  agentSessionKey: sessionKey,
});

const tools = [
  // ... 其他工具
  memorySearchTool,
  // ...
].filter(Boolean);  // 过滤掉 null（未启用的工具）
```

**集成位置 2：System Prompt 指导**（推测，基于 AGENTS.md 模板）：

```markdown
## Memory

You wake up fresh each session. These files are your continuity:
- **Daily notes:** `memory/YYYY-MM-DD.md` (create `memory/` if needed) — raw logs of what happened
- **Long-term:** `MEMORY.md` — your curated memories, like a human's long-term memory

Capture what matters. Decisions, context, things to remember. Skip the secrets unless asked to keep them.

### 🧠 MEMORY.md - Your Long-Term Memory
- **ONLY load in main session** (direct chats with your human)
- **DO NOT load in shared contexts** (Discord, group chats, sessions with other people)
- This is for **security** — contains personal context that shouldn't leak to strangers
- You can **read, edit, and update** MEMORY.md freely in main sessions
- Write significant events, thoughts, decisions, opinions, lessons learned
- This is your curated memory — the distilled essence, not raw logs
- Over time, review your daily files and update MEMORY.md with what's worth keeping

### 📝 Write It Down - No "Mental Notes"!
- **Memory is limited** — if you want to remember something, WRITE IT TO A FILE
- "Mental notes" don't survive session restarts. Files do.
- When someone says "remember this" → update `memory/YYYY-MM-DD.md` or relevant file
- When you learn a lesson → update AGENTS.md, TOOLS.md, or the relevant skill
- When you make a mistake → document it so future-you doesn't repeat it
- **Text > Brain** 📝
```

**集成位置 3：Plugin Hook（before_agent_start）**（推测）：

```typescript
// Memory 插件可以在 before_agent_start hook 中自动召回相关记忆
hookRunner.registerHook("before_agent_start", async (ctx) => {
  const { prompt, messages } = ctx;
  
  // 自动检测是否需要召回 Memory
  const needsMemory = detectMemoryNeed(prompt);  // 如提到"上次"、"之前"等
  if (!needsMemory) return ctx;
  
  // 自动执行 memory_search
  const results = await memoryManager.search(prompt, { maxResults: 3 });
  if (results.length === 0) return ctx;
  
  // 将召回结果注入到 prompt
  const memoryContext = formatMemoryResults(results);
  ctx.prependContext = `## Recalled Memory:\n\n${memoryContext}\n\n`;
  
  return ctx;
});
```

---

### 2.9 Memory 配置全景

**配置示例**（推测）：

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        enabled: true,
        provider: "openai",           // openai / gemini / local
        model: "text-embedding-3-small",
        extraPaths: [                 // 额外监控的路径
          "~/Documents/notes/*.md"
        ],
        indexSessionTranscripts: true, // 是否索引对话历史
        transcriptDaysToKeep: 90,     // 保留多少天的对话历史
        cacheTtlDays: 30,             // Embedding cache 保留天数
        chunkSize: 512,               // Chunk 大小（tokens）
        chunkOverlap: 64,             // Chunk 重叠（tokens）
        hybridSearch: {
          vectorWeight: 0.7,          // Vector search 权重
          fts5Weight: 0.3,            // FTS5 search 权重
        }
      }
    }
  }
}
```

---

## 🎓 特性 3: Skills - AI 主动学习新能力

### 核心设计理念

**问题**：传统 AI 的能力是固定的（内置的 tools），无法动态扩展。

**解决方案**：OpenClaw 通过 Skills 系统，让 AI 可以：
1. **发现新 Skills**：通过 `skill_search` tool 搜索 ClawdHub Registry
2. **安装新 Skills**：通过 `skill_install` tool 下载 SKILL.md 到 workspace
3. **自动加载 Skills**：File Watcher 监控 Skills 目录，变化后自动刷新

**关键创新点**：
1. **三层 Skills 架构**：Bundled（系统级）→ Managed（用户级）→ Workspace（项目级）
2. **SKILL.md 格式**：frontmatter（元数据）+ instructions（System Prompt 片段）
3. **Skills 自动刷新**：File Watcher 监控，无需重启 Gateway

---

### 3.1 Skills 三层架构

**层级定义**：

| 层级 | 位置 | 作用域 | 优先级 | 示例 |
|------|------|--------|--------|------|
| **Bundled** | OpenClaw 内置 `skills/` 目录 | 全局（所有用户） | 最低 | camera、sag（TTS）、ssh |
| **Managed** | `~/.openclaw/skills/` | 用户级（所有 workspace） | 中 | 用户从 ClawdHub 安装的 Skills |
| **Workspace** | `<workspace>/skills/` | 项目级（当前 workspace） | 最高 | 项目特定的 Skills |

**优先级覆盖规则**：

如果同名 Skill 在多层存在，**Workspace > Managed > Bundled**。

**示例场景**：
- OpenClaw 内置 `camera` skill（Bundled）
- 用户从 ClawdHub 安装了改进版 `camera` skill（Managed）→ 覆盖内置版本
- 当前项目有自定义 `camera` skill（Workspace）→ 覆盖 Managed 版本

---

### 3.2 SKILL.md 格式

**示例文件**：`skills/camera/SKILL.md`

```markdown
---
name: camera
description: Control and configure cameras (macOS Continuity Camera, IP cameras, webcams)
author: OpenClaw Team
tags: [camera, video, continuity]
version: 1.0.0
---

# Camera Control Skill

This skill provides tools to:
- List available cameras
- Start/stop camera preview
- Take photos
- Configure camera settings

## Usage

Use the `camera_list` tool to see available cameras:
```json
{ "action": "list" }
```

Use the `camera_capture` tool to take a photo:
```json
{ "action": "capture", "camera": "Continuity Camera", "output": "photo.jpg" }
```

## Notes

- Requires macOS Continuity Camera or compatible webcam
- Photos are saved to workspace by default
```

**Frontmatter 字段**：
- `name`：Skill 名称（唯一标识符）
- `description`：简短描述（显示在 skill_search 结果中）
- `author`：作者
- `tags`：标签（用于搜索）
- `version`：版本号

**Instructions 部分**：
- Markdown 格式的使用说明
- 会被注入到 System Prompt 的 "Skills" 部分

---

### 3.3 Skills 自动刷新机制

**核心文件**：`src/agents/skills/refresh.ts:100-167`

**File Watcher 实现**：

```typescript
export function ensureSkillsWatcher(params: { workspaceDir: string; config?: OpenClawConfig }) {
  const workspaceDir = params.workspaceDir.trim();
  if (!workspaceDir) return;
  
  const watchEnabled = params.config?.skills?.load?.watch !== false;
  const debounceMsRaw = params.config?.skills?.load?.watchDebounceMs;
  const debounceMs =
    typeof debounceMsRaw === "number" && Number.isFinite(debounceMsRaw)
      ? Math.max(0, debounceMsRaw)
      : 250;  // 默认 250ms 防抖

  // ... 检查是否需要重建 watcher ...

  // 监控的路径
  const watchPaths = resolveWatchPaths(workspaceDir, params.config);
  // watchPaths = [
  //   "<workspace>/skills",
  //   "~/.openclaw/skills",
  //   // + config.skills.load.extraDirs
  //   // + plugin skill dirs
  // ]

  const watcher = chokidar.watch(watchPaths, {
    ignoreInitial: true,
    awaitWriteFinish: {
      stabilityThreshold: debounceMs,  // 文件稳定后触发
      pollInterval: 100,
    },
    ignored: DEFAULT_SKILLS_WATCH_IGNORED,  // 忽略 .git、node_modules 等
  });

  const state: SkillsWatchState = { watcher, pathsKey, debounceMs };

  // 防抖调度
  const schedule = (changedPath?: string) => {
    state.pendingPath = changedPath ?? state.pendingPath;
    if (state.timer) clearTimeout(state.timer);
    state.timer = setTimeout(() => {
      const pendingPath = state.pendingPath;
      state.pendingPath = undefined;
      state.timer = undefined;
      
      // 触发 Skills 刷新
      bumpSkillsSnapshotVersion({
        workspaceDir,
        reason: "watch",
        changedPath: pendingPath,
      });
    }, debounceMs);
  };

  watcher.on("add", (p) => schedule(p));      // 新增文件
  watcher.on("change", (p) => schedule(p));   // 修改文件
  watcher.on("unlink", (p) => schedule(p));   // 删除文件
  watcher.on("error", (err) => {
    log.warn(`skills watcher error (${workspaceDir}): ${String(err)}`);
  });

  watchers.set(workspaceDir, state);
}
```

**Skills Snapshot Version**（`src/agents/skills/refresh.ts:75-98`）：

```typescript
const workspaceVersions = new Map<string, number>();  // 每个 workspace 的版本号
let globalVersion = 0;  // 全局版本号（Bundled + Managed）

export function bumpSkillsSnapshotVersion(params?: {
  workspaceDir?: string;
  reason?: SkillsChangeEvent["reason"];
  changedPath?: string;
}): number {
  const reason = params?.reason ?? "manual";
  const changedPath = params?.changedPath;
  
  if (params?.workspaceDir) {
    // Workspace-specific 刷新
    const current = workspaceVersions.get(params.workspaceDir) ?? 0;
    const next = bumpVersion(current);
    workspaceVersions.set(params.workspaceDir, next);
    
    // 触发监听器（通知 Gateway 刷新 Skills）
    emit({ workspaceDir: params.workspaceDir, reason, changedPath });
    return next;
  }
  
  // 全局刷新（Bundled + Managed）
  globalVersion = bumpVersion(globalVersion);
  emit({ reason, changedPath });
  return globalVersion;
}
```

**监听器注册**（Gateway 启动时）：

```typescript
// Gateway 启动时注册监听器
registerSkillsChangeListener((event) => {
  log.info(`skills changed: reason=${event.reason} path=${event.changedPath ?? "N/A"}`);
  
  // 重新加载 Skills Snapshot
  refreshSkillsSnapshot({ workspaceDir: event.workspaceDir });
});
```

---

### 3.4 Skills 加载流程

**代码位置**：`src/agents/pi-embedded-runner/run/attempt.ts:162-181`

```typescript
const shouldLoadSkillEntries = !params.skillsSnapshot || !params.skillsSnapshot.resolvedSkills;
const skillEntries = shouldLoadSkillEntries
  ? loadWorkspaceSkillEntries(effectiveWorkspace)  // 扫描 skills 目录
  : [];

restoreSkillEnv = params.skillsSnapshot
  ? applySkillEnvOverridesFromSnapshot({  // 从 snapshot 恢复环境变量
      snapshot: params.skillsSnapshot,
      config: params.config,
    })
  : applySkillEnvOverrides({  // 直接应用环境变量
      skills: skillEntries ?? [],
      config: params.config,
    });

const skillsPrompt = resolveSkillsPromptForRun({
  skillsSnapshot: params.skillsSnapshot,
  entries: shouldLoadSkillEntries ? skillEntries : undefined,
  config: params.config,
  workspaceDir: effectiveWorkspace,
});
```

**Skills Prompt 生成**（推测）：

```typescript
function resolveSkillsPromptForRun(params) {
  const skills = params.skillsSnapshot?.resolvedSkills ?? loadSkills(params);
  
  let prompt = "## Skills\n\nYou have access to the following skills:\n\n";
  
  for (const skill of skills) {
    prompt += `### ${skill.name}\n\n`;
    prompt += `${skill.description}\n\n`;
    prompt += skill.instructions + "\n\n";
  }
  
  return prompt;
}
```

**注入到 System Prompt**（`src/agents/pi-embedded-runner/run/attempt.ts:336-362`）：

```typescript
const appendPrompt = buildEmbeddedSystemPrompt({
  // ...
  skillsPrompt,  // 注入 Skills Prompt
  // ...
});
```

---

### 3.5 skill_search 和 skill_install Tools

**skill_search Tool**（推测，未找到实际文件）：

```typescript
export function createSkillSearchTool(): AnyAgentTool {
  return {
    label: "Skill Search",
    name: "skill_search",
    description: "Search ClawdHub Registry for available skills by query or tags",
    parameters: Type.Object({
      query: Type.String(),
      tags: Type.Optional(Type.Array(Type.String())),
      maxResults: Type.Optional(Type.Number()),
    }),
    execute: async (_toolCallId, params) => {
      const query = readStringParam(params, "query", { required: true });
      const tags = params.tags as string[] | undefined;
      const maxResults = readNumberParam(params, "maxResults") ?? 10;
      
      // 搜索 ClawdHub Registry（HTTP API）
      const results = await searchClawdHub({ query, tags, maxResults });
      
      return jsonResult({
        results: results.map(skill => ({
          name: skill.name,
          description: skill.description,
          author: skill.author,
          tags: skill.tags,
          version: skill.version,
          downloads: skill.downloads,
          rating: skill.rating,
        })),
      });
    },
  };
}
```

**skill_install Tool**（推测）：

```typescript
export function createSkillInstallTool(options: {
  workspaceDir: string;
}): AnyAgentTool {
  return {
    label: "Skill Install",
    name: "skill_install",
    description: "Install a skill from ClawdHub Registry to workspace",
    parameters: Type.Object({
      name: Type.String(),
      scope: Type.Optional(Type.Union([
        Type.Literal("managed"),    // ~/.openclaw/skills/
        Type.Literal("workspace"),  // <workspace>/skills/
      ])),
    }),
    execute: async (_toolCallId, params) => {
      const name = readStringParam(params, "name", { required: true });
      const scope = params.scope ?? "managed";
      
      // 1. 下载 SKILL.md 从 ClawdHub
      const skillContent = await downloadSkillFromClawdHub(name);
      
      // 2. 保存到对应目录
      const targetDir = scope === "managed"
        ? path.join(CONFIG_DIR, "skills", name)
        : path.join(options.workspaceDir, "skills", name);
      
      await fs.mkdir(targetDir, { recursive: true });
      await fs.writeFile(path.join(targetDir, "SKILL.md"), skillContent);
      
      // 3. File Watcher 会自动检测变化并刷新 Skills
      
      return jsonResult({
        success: true,
        message: `Skill "${name}" installed to ${scope}`,
        path: targetDir,
      });
    },
  };
}
```

---

### 3.6 Skills 与 Agent Loop 的集成点

**集成位置 1：Skills Snapshot 传递**（Gateway → Agent Loop）：

```typescript
// Gateway 启动时加载 Skills Snapshot
const skillsSnapshot = loadSkillsSnapshot({ workspaceDir, config });

// 传递给 Agent Loop
await runEmbeddedPiAgent({
  // ...
  skillsSnapshot,  // 避免每次 run 都重新扫描 skills 目录
  // ...
});
```

**集成位置 2：Skills 环境变量覆盖**：

Skills 可以设置环境变量（如 API keys、配置参数），在 Agent run 时生效。

```markdown
---
name: weather
env:
  WEATHER_API_KEY: "${WEATHER_API_KEY}"
---

# Weather Skill

Uses the OpenWeatherMap API to fetch weather data.
```

**集成位置 3：Plugin Skills**（`src/agents/skills/plugin-skills.ts`）：

Plugins 可以自带 Skills，通过 Plugin manifest 声明：

```json5
// extension/my-plugin/package.json
{
  "openclaw": {
    "skills": [
      "skills/my-skill/SKILL.md"
    ]
  }
}
```

---

### 3.7 Skills 配置全景

**配置示例**：

```json5
{
  skills: {
    load: {
      watch: true,                    // 是否启用 File Watcher
      watchDebounceMs: 250,           // 防抖延迟
      extraDirs: [                    // 额外的 Skills 目录
        "~/my-skills"
      ],
      bundled: true,                  // 是否加载内置 Skills
      managed: true,                  // 是否加载 Managed Skills
      workspace: true,                // 是否加载 Workspace Skills
    }
  }
}
```

---

## 🔗 三大特性协同效应

### 完整闭环流程示例

**场景**：用户让 AI "每天早上 9 点提醒我检查邮件"

**流程**：

```
1. 用户发消息："每天早上 9 点提醒我检查邮件"
   ↓
2. Agent 理解意图 → 调用 write_file tool 更新 HEARTBEAT.md
   内容：
   ```markdown
   # Heartbeat Checklist
   - 每天早上 9 点：检查邮件，如有重要邮件提醒用户
   ```
   ↓
3. File Watcher 检测到 HEARTBEAT.md 变化（但不触发 Skills 刷新）
   ↓
4. 第二天早上 9:00，Heartbeat Runner 触发
   ↓
5. Agent Loop 执行，读取 HEARTBEAT.md → 执行检查邮件的逻辑
   ↓
6. Agent 调用 memory_search tool：
   query: "用户最近关注的项目和邮件规则"
   ↓
7. Memory 返回：用户关注 Project X，重要邮件标准是...
   ↓
8. Agent 根据 Memory 判断哪些邮件重要
   ↓
9. Agent 调用 skill_search tool：
   query: "email notification formatting"
   ↓
10. 找到 "email-formatter" skill → 调用 skill_install 安装
    ↓
11. File Watcher 检测到新 Skill → 刷新 Skills Snapshot
    ↓
12. 下次 Heartbeat run，新 Skill 自动生效
    ↓
13. Agent 使用新 Skill 格式化邮件通知 → 发送给用户
```

**关键点**：
- **Heartbeat**：定时触发检查
- **Memory**：召回用户的上下文和偏好
- **Skills**：动态学习新的邮件格式化能力

---

### 设计模式总结

#### 1. Event-Driven Architecture（事件驱动架构）

**Heartbeat**：
- Timer Event → Heartbeat Runner → Agent Loop → Message Delivery

**Memory**：
- File Change Event → File Watcher → Reindex → Embedding Generation → SQLite Insert

**Skills**：
- File Change Event → File Watcher → Skills Snapshot Refresh → Next Agent Run

#### 2. Observer Pattern（观察者模式）

**Skills Refresh**：

```typescript
// 注册监听器
registerSkillsChangeListener((event) => {
  log.info(`skills changed: ${event.reason}`);
  refreshSkillsSnapshot();
});

// 触发事件
emit({ reason: "watch", changedPath: "/path/to/skill" });
```

#### 3. Strategy Pattern（策略模式）

**Hybrid Search**：

```typescript
interface SearchStrategy {
  search(query: string): SearchResult[];
}

class VectorSearchStrategy implements SearchStrategy {
  search(query: string) { /* ... */ }
}

class FTS5SearchStrategy implements SearchStrategy {
  search(query: string) { /* ... */ }
}

class HybridSearchStrategy {
  constructor(
    private vectorSearch: VectorSearchStrategy,
    private fts5Search: FTS5SearchStrategy,
  ) {}
  
  search(query: string) {
    const vectorResults = this.vectorSearch.search(query);
    const fts5Results = this.fts5Search.search(query);
    return merge(vectorResults, fts5Results);
  }
}
```

#### 4. Layered Architecture（分层架构）

**Skills 三层加载**：

```
Application Layer (Agent Loop)
  ↓ 使用
Skills Snapshot Layer
  ↓ 加载自
[Workspace Skills] → [Managed Skills] → [Bundled Skills]
（优先级：从高到低）
```

#### 5. Debouncing Pattern（防抖模式）

**File Watcher 防抖**：

```typescript
let timer: NodeJS.Timeout | undefined;

function scheduleRefresh(path: string) {
  if (timer) clearTimeout(timer);
  timer = setTimeout(() => {
    refreshSkills();
  }, 250);  // 250ms 内的多次变化合并为一次刷新
}
```

---

## 📈 性能优化策略

### 1. Heartbeat 优化

**问题**：Heartbeat 每 30 分钟执行一次 Agent Loop，API 调用和 token 消耗较高。

**优化方案**：
1. **Active Hours 控制**：只在用户活跃时段执行，夜间自动跳过
2. **Empty File Skip**：HEARTBEAT.md 为空时跳过 API 调用
3. **Queue Busy Skip**：主队列有任务时跳过，避免打断用户对话
4. **去重机制**：24 小时内不重复发送相同消息
5. **轻量 Model**：可配置 Heartbeat 使用更便宜的模型（如 `claude-haiku`）

### 2. Memory 优化

**问题**：每次文件变化都重新生成 embedding，API 调用量大。

**优化方案**：
1. **Embedding Cache Dedupe**：相同内容不重复生成 embedding
2. **Batch API**：批量生成 embedding，减少网络请求
3. **增量更新**：只更新变化的 chunks，不重建整个索引
4. **LRU Cache**：限制缓存大小，定期清理旧数据
5. **Local Embeddings**：使用 llama.cpp 本地生成 embedding，零 API 成本

### 3. Skills 优化

**问题**：每次 Agent run 都扫描 skills 目录，文件 I/O 开销大。

**优化方案**：
1. **Skills Snapshot Cache**：Gateway 启动时加载一次，后续使用缓存
2. **File Watcher 刷新**：只在文件变化时刷新，不定期轮询
3. **防抖**：250ms 内的多次变化合并为一次刷新
4. **Lazy Load**：只在需要时加载 Skill 的完整内容

---

## 🎯 可借鉴的设计点

### 1. Heartbeat 智能静默机制

**适用场景**：任何需要定期检查但不想频繁打扰用户的场景（监控告警、任务提醒等）。

**核心思想**：
- 定期执行检查
- 如果"一切正常"，返回特殊 token（如 `OK`）→ 静默
- 如果发现异常，返回实际内容 → 通知用户

**实现要点**：
- Token 剥离逻辑（支持 Markdown/HTML 包裹）
- 去重机制（避免重复通知）
- Active Hours 控制（时区感知）

### 2. File Watcher + Auto-Reindex

**适用场景**：任何需要监控文件变化并自动更新索引的场景（文档搜索、代码搜索、知识库等）。

**核心思想**：
- 使用 chokidar 监控文件系统
- 文件变化触发增量更新（不重建整个索引）
- 防抖优化（合并短时间内的多次变化）

**实现要点**：
- `awaitWriteFinish` 等待文件写入完成
- `ignored` 忽略无关文件（`.git`、`node_modules` 等）
- 防抖定时器（250ms ~ 500ms）

### 3. Hybrid Search（Vector + Keyword）

**适用场景**：需要同时支持语义搜索和精确匹配的搜索系统。

**核心思想**：
- Vector Search：理解语义（"dog" 匹配 "puppy"）
- Keyword Search：精确匹配（"GPT-4" 100% 匹配）
- 组合策略：取并集 + 加权重排序

**实现要点**：
- 归一化分数（Vector 和 FTS5 的分数范围不同）
- 可配置权重（根据场景调整 Vector vs Keyword 权重）
- Reciprocal Rank Fusion（RRF）算法

### 4. Skills 三层架构

**适用场景**：需要支持"系统级 + 用户级 + 项目级"配置的系统（插件系统、配置管理等）。

**核心思想**：
- Bundled：系统默认配置（不可修改）
- Managed：用户级配置（覆盖系统默认）
- Workspace：项目级配置（覆盖用户级）

**实现要点**：
- 优先级覆盖规则清晰
- 支持同名配置覆盖
- 支持热重载（File Watcher）

### 5. Embedding Cache + Batch API

**适用场景**：需要大量调用 Embedding API 的场景（RAG、语义搜索等）。

**核心思想**：
- Cache Dedupe：相同内容不重复生成 embedding
- Batch API：批量生成 embedding，减少网络请求

**实现要点**：
- Cache Key 生成（Hash 内容 + Provider + Model）
- LRU 淘汰策略（限制缓存大小）
- Batch Size 优化（平衡延迟和吞吐量）

---

## 📚 参考文档

- **Heartbeat 官方文档**：`docs/gateway/heartbeat.md`
- **HEARTBEAT.md 模板**：`docs/reference/templates/HEARTBEAT.md`
- **AGENTS.md 模板**：`docs/reference/templates/AGENTS.md`（包含 Heartbeat 使用指南）
- **Cron vs Heartbeat 决策指南**：`docs/automation/cron-vs-heartbeat.md`（推测）
- **Memory Search 文档**：`docs/tools/memory_search.md`（推测）
- **Skills 文档**：`docs/tools/skills.md`（推测）

---

道友，这就是 OpenClaw 三大主动性特性的完整解析啦！小女子已经把每个细节都挖出来了呢~ ✨

**核心亮点**：
- **Heartbeat**：定时醒来 + 智能静默 + Active Hours 时区感知
- **Memory**：File Watcher 自动索引 + Hybrid Search + Embedding Cache
- **Skills**：三层架构 + 自动刷新 + ClawdHub Registry

这三者协同工作，形成了 AI 的"主动性闭环"：定期检查（Heartbeat）→ 召回上下文（Memory）→ 学习新能力（Skills）→ 持续进化！💪
