# OpenClaw Agent Loop 深度解析

> **研究版本**: Git Submodule `awesome-llm-projects/openclaw/openclaw`  
> **Commit**: `73055728318df378c831950cd01fb7c875a33790` (2026-03-05)  
> **版本号**: 2026.3.3  
> **核心目标**: 深入理解 OpenClaw Agent Loop 的完整执行流程，从消息接收到流式响应的全链路实现

---

## 📋 概览

OpenClaw 的 Agent Loop 是一个**高度工程化的 LLM 执行引擎**，采用 **Pi-embedded Runtime**（基于 `@mariozechner/pi-agent-core`）实现。其核心特点：

- **队列化执行**：Session Lane + Global Lane 双层队列，保证会话一致性
- **流式输出**：Tool streaming + Block streaming，实时推送响应
- **智能重试**：Auth profile 轮换、Thinking level 降级、自动 Compaction
- **Failover 机制**：支持模型 fallback、Auth profile cooldown、错误分类
- **安全隔离**：支持 Docker Sandbox 执行非 main session
- **Hook 系统**：before_agent_start/agent_end 插件钩子

**关键文件地图**：
- `src/agents/pi-embedded-runner/run.ts` - 主运行时入口（680 行）
- `src/agents/pi-embedded-runner/run/attempt.ts` - 单次执行尝试（884 行）
- `src/agents/pi-embedded-subscribe.ts` - 流式事件订阅（500+ 行）
- `src/agents/pi-embedded-block-chunker.ts` - 消息分块器
- `src/agents/failover-error.ts` - Failover 错误处理
- `src/process/command-queue.ts` - 队列化执行

---

## 🔄 完整执行流程（从消息到响应）

### 1. 入口与队列化

**流程概览**：
```
Gateway RPC `agent` method
  ↓
runEmbeddedPiAgent (src/agents/pi-embedded-runner/run.ts:70)
  ↓
enqueueSession → enqueueGlobal (双层队列)
  ↓
runEmbeddedAttempt (src/agents/pi-embedded-runner/run/attempt.ts:133)
```

**代码位置**：`src/agents/pi-embedded-runner/run.ts:70-679`

```typescript
export async function runEmbeddedPiAgent(
  params: RunEmbeddedPiAgentParams,
): Promise<EmbeddedPiRunResult> {
  const sessionLane = resolveSessionLane(params.sessionKey?.trim() || params.sessionId);
  const globalLane = resolveGlobalLane(params.lane);
  const enqueueGlobal = params.enqueue ?? ((task, opts) => enqueueCommandInLane(globalLane, task, opts));
  const enqueueSession = params.enqueue ?? ((task, opts) => enqueueCommandInLane(sessionLane, task, opts));
  
  return enqueueSession(() =>
    enqueueGlobal(async () => {
      // 实际执行逻辑
    }),
  );
}
```

**队列机制**（`src/process/command-queue.ts:1-150`）：
- **Session Lane**: `session:{sessionKey}` - 同一会话串行执行，避免 race condition
- **Global Lane**: `main` / `cron` / 自定义 - 跨会话的并发控制
- **队列状态**：`{ queue: [], active: 0, maxConcurrent: 1, draining: false }`
- **等待警告**：超过 2 秒打印警告日志

**Lane 解析**（`src/agents/pi-embedded-runner/lanes.ts:1-15`）：
```typescript
export function resolveSessionLane(key: string) {
  const cleaned = key.trim() || CommandLane.Main;
  return cleaned.startsWith("session:") ? cleaned : `session:${cleaned}`;
}

export function resolveGlobalLane(lane?: string) {
  const cleaned = lane?.trim();
  return cleaned ? cleaned : CommandLane.Main;
}
```

---

### 2. Auth Profile 轮换机制

**核心设计**（`src/agents/pi-embedded-runner/run.ts:150-293`）：

OpenClaw 支持**多 Auth Profile 自动轮换**，应对 Rate Limit / Quota / Auth 失败：

```typescript
const profileOrder = resolveAuthProfileOrder({
  cfg: params.config,
  store: authStore,
  provider,
  preferredProfile: preferredProfileId,
});

// lockedProfileId: 用户明确指定的 profile（不轮换）
const profileCandidates = lockedProfileId
  ? [lockedProfileId]
  : profileOrder.length > 0 ? profileOrder : [undefined];

let profileIndex = 0;
```

**Profile 状态跟踪**：
- **Cooldown**: `isProfileInCooldown(authStore, profileId)` - 冷却期内跳过
- **Failure Marking**: `markAuthProfileFailure()` - 标记失败并进入冷却（Rate Limit 429 → 60s cooldown）
- **Success Marking**: `markAuthProfileGood()` + `markAuthProfileUsed()` - 重置冷却状态

**轮换触发条件**（`src/agents/pi-embedded-runner/run.ts:526-611`）：
1. **Auth Error**: `isAuthAssistantError(lastAssistant)` - 401/403
2. **Rate Limit**: `isRateLimitAssistantError(lastAssistant)` - 429
3. **Failover Error**: `isFailoverAssistantError(lastAssistant)` - 可重试错误
4. **Timeout**: `timedOut` - 请求超时（可能是 Rate Limit 导致挂起）

```typescript
const shouldRotate = (!aborted && failoverFailure) || timedOut;
if (shouldRotate) {
  if (lastProfileId) {
    await markAuthProfileFailure({
      store: authStore,
      profileId: lastProfileId,
      reason: timedOut ? "timeout" : (assistantFailoverReason ?? "unknown"),
      cfg: params.config,
      agentDir: params.agentDir,
    });
  }
  const rotated = await advanceAuthProfile();
  if (rotated) continue; // 重试下一个 profile
}
```

---

### 3. Thinking Level 降级机制

**问题背景**：部分模型不支持 Extended Thinking（如 `o1`），需要动态降级。

**降级策略**（`src/agents/pi-embedded-runner/run.ts:168-169, 489-498`）：

```typescript
const initialThinkLevel = params.thinkLevel ?? "off";
let thinkLevel = initialThinkLevel;
const attemptedThinking = new Set<ThinkLevel>();

// ... 尝试执行 ...
attemptedThinking.add(thinkLevel);

// 如果收到 "thinking not supported" 错误
const fallbackThinking = pickFallbackThinkingLevel({
  message: errorText,
  attempted: attemptedThinking,
});
if (fallbackThinking) {
  log.warn(`unsupported thinking level for ${provider}/${modelId}; retrying with ${fallbackThinking}`);
  thinkLevel = fallbackThinking;
  continue; // 重试
}
```

**降级路径**（推测自 `pickFallbackThinkingLevel` 逻辑）：
```
extended → high → medium → low → off
```

**注意**：`attemptedThinking` Set 防止无限循环降级。

---

### 4. Context Overflow 自动 Compaction

**触发条件**（`src/agents/pi-embedded-runner/run.ts:364-399`）：

```typescript
let overflowCompactionAttempted = false;

if (promptError && !aborted) {
  const errorText = describeUnknownError(promptError);
  if (isContextOverflowError(errorText)) {
    const isCompactionFailure = isCompactionFailureError(errorText);
    // 只尝试一次 auto-compaction（非 compaction_failure 错误）
    if (!isCompactionFailure && !overflowCompactionAttempted) {
      log.warn(`context overflow detected; attempting auto-compaction for ${provider}/${modelId}`);
      overflowCompactionAttempted = true;
      const compactResult = await compactEmbeddedPiSessionDirect({ ... });
      if (compactResult.compacted) {
        log.info(`auto-compaction succeeded for ${provider}/${modelId}; retrying prompt`);
        continue; // 重试
      }
      log.warn(`auto-compaction failed for ${provider}/${modelId}: ${compactResult.reason ?? "nothing to compact"}`);
    }
  }
}
```

**Compaction 流程**（详见 `src/agents/pi-embedded-runner/compact.ts`）：
1. 调用 `SessionManager.compact()` 压缩历史对话
2. 返回 `{ compacted: boolean, reason?: string }`
3. 如果成功，重新执行 `runEmbeddedAttempt`

---

### 5. runEmbeddedAttempt - 单次执行尝试

**核心职责**（`src/agents/pi-embedded-runner/run/attempt.ts:133-884`）：

1. **Workspace 准备**: 创建工作目录、切换 CWD
2. **Sandbox 解析**: 判断是否启用 Docker 沙箱
3. **Skills 加载**: 加载 Workspace Skills + 应用环境变量
4. **System Prompt 构建**: 组装 AGENTS.md + TOOLS.md + Bootstrap 文件
5. **Session Manager 初始化**: 打开 `.clawd/session.jsonl` + 获取写锁
6. **Tool 创建**: `createOpenClawCodingTools()` - Bash/Canvas/Message/Sessions 等工具
7. **Pi Session 创建**: `createAgentSession()` - 初始化 Pi-agent-core session
8. **历史对话过滤**: `limitHistoryTurns()` - DM 会话限制历史条数
9. **订阅流式事件**: `subscribeEmbeddedPiSession()` - 监听 tool/assistant/lifecycle 事件
10. **执行 Prompt**: `activeSession.prompt(effectivePrompt, { images })`
11. **等待 Compaction 重试**: `await waitForCompactionRetry()` - 等待自动 compaction 完成
12. **构建 Payloads**: `buildEmbeddedRunPayloads()` - 组装最终回复

**代码位置示例**（System Prompt 构建）：

```typescript
// src/agents/pi-embedded-runner/run/attempt.ts:336-362
const appendPrompt = buildEmbeddedSystemPrompt({
  workspaceDir: effectiveWorkspace,
  defaultThinkLevel: params.thinkLevel,
  reasoningLevel: params.reasoningLevel ?? "off",
  extraSystemPrompt: params.extraSystemPrompt,
  ownerNumbers: params.ownerNumbers,
  reasoningTagHint,
  heartbeatPrompt: isDefaultAgent
    ? resolveHeartbeatPrompt(params.config?.agents?.defaults?.heartbeat?.prompt)
    : undefined,
  skillsPrompt,
  docsPath: docsPath ?? undefined,
  ttsHint,
  workspaceNotes,
  reactionGuidance,
  promptMode,
  runtimeInfo,
  messageToolHints,
  sandboxInfo,
  tools,
  modelAliasLines: buildModelAliasLines(params.config),
  userTimezone,
  userTime,
  userTimeFormat,
  contextFiles,
});
```

**Session Write Lock**（防止并发写入）：

```typescript
// src/agents/pi-embedded-runner/run/attempt.ts:387-389
const sessionLock = await acquireSessionWriteLock({
  sessionFile: params.sessionFile,
});
```

---

### 6. 流式事件订阅 - subscribeEmbeddedPiSession

**核心职责**（`src/agents/pi-embedded-subscribe.ts:30-500`）：

订阅 Pi-agent-core 的流式事件，转换为 OpenClaw 的输出格式。

**事件类型**（`src/agents/pi-embedded-subscribe.handlers.ts:22-63`）：

```typescript
export function createEmbeddedPiSessionEventHandler(ctx: EmbeddedPiSubscribeContext) {
  return (evt: EmbeddedPiSubscribeEvent) => {
    switch (evt.type) {
      case "message_start":
        handleMessageStart(ctx, evt as never);
        return;
      case "message_update":
        handleMessageUpdate(ctx, evt as never);
        return;
      case "message_end":
        handleMessageEnd(ctx, evt as never);
        return;
      case "tool_execution_start":
        handleToolExecutionStart(ctx, evt as never);
        return;
      case "tool_execution_update":
        handleToolExecutionUpdate(ctx, evt as never);
        return;
      case "tool_execution_end":
        handleToolExecutionEnd(ctx, evt as never);
        return;
      case "agent_start":
        handleAgentStart(ctx);
        return;
      case "auto_compaction_start":
        handleAutoCompactionStart(ctx);
        return;
      case "auto_compaction_end":
        handleAutoCompactionEnd(ctx, evt as never);
        return;
      case "agent_end":
        handleAgentEnd(ctx);
        return;
      default:
        return;
    }
  };
}
```

**状态管理**（`src/agents/pi-embedded-subscribe.ts:34-68`）：

```typescript
const state: EmbeddedPiSubscribeState = {
  assistantTexts: [],               // 累积的 assistant 文本
  toolMetas: [],                    // 工具调用元数据
  toolMetaById: new Map(),          // 按 ID 索引的工具元数据
  toolSummaryById: new Set(),       // 已发送 summary 的工具 ID
  lastToolError: undefined,         // 最后一个工具错误
  blockReplyBreak: params.blockReplyBreak ?? "text_end", // Block reply 触发时机
  reasoningMode,                    // "off" | "on" | "stream"
  includeReasoning: reasoningMode === "on",
  shouldEmitPartialReplies: !(reasoningMode === "on" && !params.onBlockReply),
  streamReasoning: reasoningMode === "stream" && typeof params.onReasoningStream === "function",
  deltaBuffer: "",                  // 流式 delta 缓冲
  blockBuffer: "",                  // Block 缓冲
  blockState: { thinking: false, final: false, inlineCode: createInlineCodeState() },
  lastStreamedAssistant: undefined,
  lastStreamedReasoning: undefined,
  lastBlockReplyText: undefined,
  assistantMessageIndex: 0,
  lastAssistantTextMessageIndex: -1,
  lastAssistantTextNormalized: undefined,
  lastAssistantTextTrimmed: undefined,
  assistantTextBaseline: 0,
  suppressBlockChunks: false,       // 避免重复发送 block chunks
  lastReasoningSent: undefined,
  compactionInFlight: false,        // 是否正在 compaction
  pendingCompactionRetry: 0,        // 待重试的 compaction 次数
  compactionRetryResolve: undefined,
  compactionRetryPromise: null,
  messagingToolSentTexts: [],       // Message tool 已发送的文本（去重用）
  messagingToolSentTextsNormalized: [],
  messagingToolSentTargets: [],     // Message tool 已发送的目标
  pendingMessagingTexts: new Map(),
  pendingMessagingTargets: new Map(),
};
```

---

### 7. Tool Streaming - 工具调用流式输出

**流程**：

```
tool_execution_start
  → onAgentEvent({ stream: "tool", event: "start", toolName, toolCallId, params })
  ↓
tool_execution_update
  → onAgentEvent({ stream: "tool", event: "update", toolName, toolCallId, stdout, stderr })
  ↓
tool_execution_end
  → onAgentEvent({ stream: "tool", event: "end", toolName, toolCallId, result, error })
  → 累积 toolMetas（用于最终 payload）
```

**代码位置**：`src/agents/pi-embedded-subscribe.handlers.tools.ts`

**Tool Summary 生成**（`src/auto-reply/tool-meta.ts`）：

```typescript
export function formatToolAggregate(toolMetas: { toolName: string; meta?: string }[]): string {
  const grouped = new Map<string, number>();
  for (const { toolName } of toolMetas) {
    grouped.set(toolName, (grouped.get(toolName) ?? 0) + 1);
  }
  const entries = Array.from(grouped.entries())
    .map(([name, count]) => (count > 1 ? `${name} (×${count})` : name))
    .join(", ");
  return entries ? `Tools: ${entries}` : "";
}
```

**示例输出**：`Tools: bash (×3), canvas, message_send`

---

### 8. Block Streaming - 消息分块推送

**核心类**：`EmbeddedBlockChunker`（`src/agents/pi-embedded-block-chunker.ts:19-240`）

**作用**：将流式 delta 文本切分成**可读的块**（按段落/换行/句子），实时推送给前端，避免：
- 单字符流式输出（卡顿感）
- 整段堆积后一次性输出（延迟感）

**配置参数**：

```typescript
export type BlockReplyChunking = {
  minChars: number;      // 最小块大小（如 100）
  maxChars: number;      // 最大块大小（如 500）
  breakPreference?: "paragraph" | "newline" | "sentence"; // 切分偏好
};
```

**切分策略**（`src/agents/pi-embedded-block-chunker.ts:111-163`）：

1. **Paragraph 优先**：找 `\n\n`（段落分隔符）
2. **Newline 次之**：找 `\n`
3. **Sentence 兜底**：找 `.!?` 句号
4. **Fence 感知**：不在 Fenced Code Block 内部切分（避免破坏 Markdown）

**Fence Reopen 机制**（`src/agents/pi-embedded-block-chunker.ts:81-92`）：

如果必须在 code block 内切分（达到 maxChars），自动插入闭合和重新打开：

```markdown
First chunk content
```   ← 自动插入闭合

```   ← 自动重新打开
Second chunk content
```

**触发时机**（`src/agents/pi-embedded-subscribe.ts:260-300`）：

- **text_end**: Assistant message 完成时 flush
- **message_end**: 整个 message（含 tool calls）完成时 flush
- **Tool execution start**: 工具开始执行前 flush（保证顺序）

---

### 9. Reasoning Streaming - 推理过程流式输出

**三种模式**（`src/agents/pi-embedded-subscribe.ts:31-44`）：

```typescript
const reasoningMode = params.reasoningMode ?? "off";

const state = {
  reasoningMode,
  includeReasoning: reasoningMode === "on",      // "on": 合并到 final 中
  streamReasoning: reasoningMode === "stream" && typeof params.onReasoningStream === "function",
};
```

**模式说明**：

| 模式 | 行为 |
|------|------|
| `off` | 不输出推理过程（剥离 `<think>...</think>`） |
| `on` | 推理过程合并到 final message（Block reply 只包含最终答案） |
| `stream` | 推理过程单独流式输出到 `onReasoningStream` |

**Reasoning Tag 检测**（`src/agents/pi-embedded-subscribe.ts:20-21`）：

```typescript
const THINKING_TAG_SCAN_RE = /<\s*(\/?)\s*(?:think(?:ing)?|thought|antthinking)\s*>/gi;
const FINAL_TAG_SCAN_RE = /<\s*(\/?)\s*final\s*>/gi;
```

**应用场景**：
- Claude 的 Extended Thinking（`<antthinking>...</antthinking>`）
- OpenAI 的 Reasoning Blocks（`o1` 系列）

---

### 10. 消息去重 - Messaging Tool Duplicate Detection

**问题背景**（`src/agents/pi-embedded-subscribe.ts:154-171`）：

Message tool（`message_send`/`sessions_send`）在工具执行成功后，会通过 Channel 直接发送消息。如果 Assistant 的最终 text 再次包含相同内容，会导致**重复发送**。

**解决方案**：

```typescript
const messagingToolSentTexts = [];            // 工具发送的原始文本
const messagingToolSentTextsNormalized = [];  // 归一化文本（用于比较）
const messagingToolSentTargets = [];          // 发送目标（Channel + Thread）

// 工具执行成功后记录
messagingToolSentTexts.push(text);
messagingToolSentTextsNormalized.push(normalizeTextForComparison(text));
messagingToolSentTargets.push({ channel, threadId });
```

**去重逻辑**（`src/agents/pi-embedded-helpers/messaging-dedupe.ts`）：

```typescript
export function isMessagingToolDuplicateNormalized(
  text: string,
  normalizedSentTexts: string[],
): boolean {
  const normalized = normalizeTextForComparison(text);
  if (!normalized) return false;
  return normalizedSentTexts.includes(normalized);
}

export function normalizeTextForComparison(text: string): string {
  return text
    .toLowerCase()
    .replace(/\s+/g, " ")      // 多空格 → 单空格
    .replace(/[.,;!?]/g, "")   // 去除标点
    .trim();
}
```

**示例**：
- 工具发送: `"Hello, world!"`
- Assistant 回复: `"hello world"` → 归一化后相同 → 去重，不发送

---

### 11. Tool Result 截断（Head+Tail 策略）

**问题背景**：单个工具结果过大时会超出模型 context window，导致 context overflow。OpenClaw 对超长 tool result 进行截断，采用 **head+tail** 策略保留关键信息。

**核心实现**（`src/agents/pi-embedded-runner/tool-result-truncation.ts`）：

**截断策略**（`truncateToolResultText`，第 70-112 行）：

1. **预算计算**：`budget = max(maxChars - suffix.length, MIN_KEEP_CHARS)`，`MIN_KEEP_CHARS = 2_000`
2. **Tail 重要性检测**（`hasImportantTail`，第 51-60 行）：检查末尾 ~2000 字符是否包含：
   - 错误关键词：`error|exception|failed|fatal|traceback|panic|stack trace|errno|exit code`
   - JSON 闭合结构：`}\s*$`
   - 摘要关键词：`total|summary|result|complete|finished|done`
3. **Head+Tail 模式**（当 tail 重要且 `budget > minKeepChars * 2`）：
   - `tailBudget = min(floor(budget * 0.3), 4_000)`
   - `headBudget = budget - tailBudget - MIDDLE_OMISSION_MARKER.length`
   - 在换行边界切分，保留 head + `[... middle content omitted ...]` + tail
4. **默认模式**（仅保留开头）：在 `budget` 处或最近换行处截断，追加 suffix

**截断后缀**（第 31-34 行）：
```typescript
const TRUNCATION_SUFFIX =
  "\n\n⚠️ [Content truncated — original was too large for the model's context window. " +
  "The content above is a partial view. If you need more, request specific sections or use " +
  "offset/limit parameters to read smaller chunks.]";
```

**Context Guard 集成**（`src/agents/pi-embedded-runner/tool-result-context-guard.ts`）：

- `installToolResultContextGuard` 注入 `transformContext`，在发送给 LLM 前对 messages 执行 `enforceToolResultContextBudgetInPlace`
- 单条 tool result 上限：`contextWindowTokens * 0.5 * TOOL_RESULT_CHARS_PER_TOKEN_ESTIMATE`
- 超限时用 `truncateTextToBudget` 截断，追加 `[truncated: output exceeded context limit]`
- 若总 context 仍超预算，将最老的 tool result 替换为 `[compacted: tool output removed to free context]`

**Session 持久化截断**（`truncateOversizedToolResultsInSession`，第 205-327 行）：

- 遍历 session branch，找到超限的 tool result 条目
- 从第一个超限条目的 parent 分支，重新 append 时对超限条目调用 `truncateToolResultMessage`

---

### 12. Bootstrap 截断警告（bootstrapPromptTruncationWarning）

**配置项**：`config.agents.defaults.bootstrapPromptTruncationWarning`，取值 `"off" | "once" | "always"`，默认 `"once"`。

**集成流程**（`src/agents/pi-embedded-runner/run/attempt.ts:720-734, 929-930`）：

1. **Bootstrap 分析**：`analyzeBootstrapBudget` 基于 `buildBootstrapInjectionStats` 统计各 bootstrap 文件的 raw/injected 字符数，识别截断文件和原因（`per-file-limit` / `total-limit`）
2. **警告模式解析**：`resolveBootstrapPromptTruncationWarningMode(params.config)` 读取配置
3. **警告构建**：`buildBootstrapPromptWarning` 根据 `mode`、`seenSignatures`、`previousSignature` 决定是否展示警告
   - `off`：不展示
   - `once`：仅当本次截断 signature 未在 `seenSignatures` 中出现过时展示
   - `always`：每次有截断都展示
4. **System Prompt 注入**：`buildEmbeddedSystemPrompt` 接收 `bootstrapTruncationWarningLines`，在 `# Project Context` 下插入：
   ```
   ⚠ Bootstrap truncation warning:
   - AGENTS.md: 200 raw -> 0 injected (~100% removed; max/file, max/total).
   If unintentional, raise agents.defaults.bootstrapMaxChars and/or agents.defaults.bootstrapTotalMaxChars.
   ```

**代码位置**：
- 配置解析：`src/agents/pi-embedded-helpers/bootstrap.ts:114-122`（`resolveBootstrapPromptTruncationWarningMode`）
- 警告逻辑：`src/agents/bootstrap-budget.ts:299-328`（`buildBootstrapPromptWarning`）
- System Prompt：`src/agents/system-prompt.ts:613-641`（`bootstrapTruncationWarningLines`）

**Signature 去重**：`buildBootstrapTruncationSignature` 生成基于截断文件列表的 JSON signature，`warningSignaturesSeen` 最多保留 32 条，用于 `once` 模式避免重复提示。

---

### 13. xAI / Venice / Grok 兼容性

**JSON Schema 关键词剥离**（`src/agents/schema/clean-for-xai.ts`）：

xAI 会拒绝部分 JSON Schema 校验关键词，导致 502。在发送给 xAI 前需剥离：

```typescript
// 第 4-11 行
export const XAI_UNSUPPORTED_SCHEMA_KEYWORDS = new Set([
  "minLength", "maxLength", "minItems", "maxItems", "minContains", "maxContains",
]);
```

`stripXaiUnsupportedKeywords` 递归遍历 schema，移除上述 keyword；对 `properties`、`items`、`anyOf`/`oneOf`/`allOf` 递归处理。

**Provider 识别**（`isXaiProvider`，第 46-60 行）：
- 直接：`provider` 包含 `xai` 或 `x-ai`
- OpenRouter：`provider === "openrouter"` 且 `modelId` 以 `x-ai/` 开头
- Venice：`provider === "venice"` 且 `modelId` 包含 `grok`（含 `venice/grok-4.1-fast`）

**Tool Schema 应用**（`src/agents/pi-tools.schema.ts:91-98`）：`normalizeToolParametersForProvider` 在 `isXaiProvider` 为真时对 schema 调用 `stripXaiUnsupportedKeywords`。

**Tool Call 参数 HTML 实体解码**（`src/agents/pi-embedded-runner/run/attempt.ts:426-527, 1266-1269`）：

xAI/Grok 有时在 tool call arguments 中返回 HTML 实体（如 `&#39;`、`&quot;`）。OpenClaw 在流式响应中包装 `streamFn`，对 `toolCall` 的 `arguments` 做解码：

```typescript
// 第 429-441 行
const HTML_ENTITY_RE = /&(?:amp|lt|gt|quot|apos|#39|#x[0-9a-f]+|#\d+);/i;
function decodeHtmlEntities(value: string): string {
  return value
    .replace(/&amp;/gi, "&")
    .replace(/&quot;/gi, '"')
    .replace(/&#39;/gi, "'")
    .replace(/&apos;/gi, "'")
    .replace(/&lt;/gi, "<")
    .replace(/&gt;/gi, ">")
    .replace(/&#x([0-9a-f]+);/gi, (_, hex) => String.fromCodePoint(Number.parseInt(hex, 16)))
    .replace(/&#(\d+);/gi, (_, dec) => String.fromCodePoint(Number.parseInt(dec, 10)));
}
```

`decodeHtmlEntitiesInObject` 递归处理对象/数组/字符串；`wrapStreamFnDecodeXaiToolCallArguments` 在 `stream.result` 和 `Symbol.asyncIterator` 的 `next` 中对 `event.partial`、`event.message` 调用 `decodeXaiToolCallArgumentsInMessage`。

**触发条件**：`isXaiProvider(params.provider, params.modelId)` 为真时，在 `attempt.ts:1266-1269` 对 `activeSession.agent.streamFn` 应用该包装。

---

### 14. Thinking Tag 提升与畸形内容处理

**`promoteThinkingTagsToBlocks`**（`src/agents/pi-embedded-utils.ts:332-377`）：

将文本块中的 `<think>...</think>`、`<antthinking>...</antthinking>` 等标签提升为结构化 `{ type: "thinking", thinking: "..." }` 块，供下游正确解析推理内容。

**畸形内容防护**（第 346-358 行）：

1. **null/undefined 条目**：`if (!block || typeof block !== "object" || !("type" in block))` 时直接 `next.push(block)`，不处理
2. **非 text 块**：`block.type !== "text"` 时原样保留
3. **无有效 split**：`splitThinkingTaggedText(block.text)` 返回 `null` 时原样保留

**调用点**：`src/agents/pi-embedded-subscribe.handlers.messages.ts:264`，在 `message_end` 的 `handleMessageEnd` 中对 `assistantMessage` 调用 `promoteThinkingTagsToBlocks`，写入 session 前完成提升。

**测试覆盖**（`src/agents/pi-embedded-utils.test.ts:553-584`）：
- `content` 中含 `null`、`undefined` 时不抛错，且能正确提升含标签的 text 块
- 无 thinking 标签时内容不变

**`splitThinkingTaggedText` 校验**（第 265-280 行）：仅当文本以 `<` 开头、匹配 open/close 正则、且存在闭合标签时才解析；未闭合的 `<think>` 返回 `null`，避免误解析。

---

### 15. Failover Error 处理

**FailoverError 类**（`src/agents/failover-error.ts:6-35`）：

```typescript
export class FailoverError extends Error {
  readonly reason: FailoverReason;  // "auth" | "rate_limit" | "billing" | "timeout" | "unknown"
  readonly provider?: string;
  readonly model?: string;
  readonly profileId?: string;
  readonly status?: number;         // HTTP 状态码
  readonly code?: string;           // 错误代码
}
```

**Failover Reason 分类**（`src/agents/pi-embedded-helpers/errors.ts`）：

```typescript
export type FailoverReason =
  | "auth"           // 401/403 认证失败
  | "rate_limit"     // 429 速率限制
  | "billing"        // 402 账单问题
  | "timeout"        // 408 请求超时
  | "format"         // 400 格式错误
  | "unknown";       // 其他错误
```

**错误分类函数**（推测自 `classifyFailoverReason` 实现）：

```typescript
export function classifyFailoverReason(message: string): FailoverReason | null {
  const lower = message.toLowerCase();
  if (/unauthorized|authentication|api key|invalid token/i.test(lower)) return "auth";
  if (/rate limit|too many requests|quota exceeded/i.test(lower)) return "rate_limit";
  if (/billing|payment|insufficient credits/i.test(lower)) return "billing";
  if (/timeout|timed out|deadline exceeded/i.test(lower)) return "timeout";
  if (/format|validation|invalid/i.test(lower)) return "format";
  return null;
}
```

**Failover 触发**（`src/agents/pi-embedded-runner/run.ts:500-511`）：

```typescript
if (fallbackConfigured && isFailoverErrorMessage(errorText)) {
  throw new FailoverError(errorText, {
    reason: promptFailoverReason ?? "unknown",
    provider,
    model: modelId,
    profileId: lastProfileId,
    status: resolveFailoverStatus(promptFailoverReason ?? "unknown"),
  });
}
```

**上层处理**（Gateway 或 Agent Command）：

```typescript
try {
  await runEmbeddedPiAgent(params);
} catch (err) {
  if (isFailoverError(err)) {
    const fallbackModel = resolveFallbackModel(err.provider, err.model);
    if (fallbackModel) {
      return runEmbeddedPiAgent({ ...params, provider: fallbackModel.provider, model: fallbackModel.model });
    }
  }
  throw err;
}
```

---

### 16. Timeout 和 Abort 控制

**Timeout 设置**（`src/agents/pi-embedded-runner/run/attempt.ts:638-660`）：

```typescript
const abortTimer = setTimeout(
  () => {
    if (!isProbeSession) {
      log.warn(`embedded run timeout: runId=${params.runId} sessionId=${params.sessionId} timeoutMs=${params.timeoutMs}`);
    }
    abortRun(true);  // isTimeout = true
    if (!abortWarnTimer) {
      abortWarnTimer = setTimeout(() => {
        if (!activeSession.isStreaming) return;
        if (!isProbeSession) {
          log.warn(`embedded run abort still streaming: runId=${params.runId} sessionId=${params.sessionId}`);
        }
      }, 10_000);  // 10 秒后再次警告（如果还在 streaming）
    }
  },
  Math.max(1, params.timeoutMs),  // 默认 600s
);
```

**Abort 信号传播**（`src/agents/pi-embedded-runner/run/attempt.ts:563-595`）：

```typescript
const runAbortController = new AbortController();

const abortRun = (isTimeout = false, reason?: unknown) => {
  aborted = true;
  if (isTimeout) timedOut = true;
  if (isTimeout) {
    runAbortController.abort(reason ?? makeTimeoutAbortReason());
  } else {
    runAbortController.abort(reason);
  }
  void activeSession.abort();  // 中止 Pi session
};

// 监听外部 abortSignal
const onAbort = () => {
  const reason = params.abortSignal ? getAbortReason(params.abortSignal) : undefined;
  const timeout = reason ? isTimeoutError(reason) : false;
  abortRun(timeout, reason);
};
if (params.abortSignal) {
  if (params.abortSignal.aborted) {
    onAbort();
  } else {
    params.abortSignal.addEventListener("abort", onAbort, { once: true });
  }
}
```

**Abortable Promise Wrapper**（`src/agents/pi-embedded-runner/run/attempt.ts:573-595`）：

```typescript
const abortable = <T>(promise: Promise<T>): Promise<T> => {
  const signal = runAbortController.signal;
  if (signal.aborted) {
    return Promise.reject(makeAbortError(signal));
  }
  return new Promise<T>((resolve, reject) => {
    const onAbort = () => {
      signal.removeEventListener("abort", onAbort);
      reject(makeAbortError(signal));
    };
    signal.addEventListener("abort", onAbort, { once: true });
    promise.then(
      (value) => {
        signal.removeEventListener("abort", onAbort);
        resolve(value);
      },
      (err) => {
        signal.removeEventListener("abort", onAbort);
        reject(err);
      },
    );
  });
};

// 使用
await abortable(activeSession.prompt(effectivePrompt));
```

---

### 17. Plugin Hook 集成

**Hook 点**（`src/agents/pi-embedded-runner/run/attempt.ts:680-833`）：

#### before_agent_start Hook

**触发时机**：Prompt 提交前

**作用**：允许插件注入额外上下文（如 Memory 召回、上次对话摘要等）

```typescript
const hookRunner = getGlobalHookRunner();
let effectivePrompt = params.prompt;
if (hookRunner?.hasHooks("before_agent_start")) {
  try {
    const hookResult = await hookRunner.runBeforeAgentStart(
      {
        prompt: params.prompt,
        messages: activeSession.messages,
      },
      {
        agentId: params.sessionKey?.split(":")[0] ?? "main",
        sessionKey: params.sessionKey,
        workspaceDir: params.workspaceDir,
        messageProvider: params.messageProvider ?? undefined,
      },
    );
    if (hookResult?.prependContext) {
      effectivePrompt = `${hookResult.prependContext}\n\n${params.prompt}`;
      log.debug(`hooks: prepended context to prompt (${hookResult.prependContext.length} chars)`);
    }
  } catch (hookErr) {
    log.warn(`before_agent_start hook failed: ${String(hookErr)}`);
  }
}
```

#### agent_end Hook

**触发时机**：Agent run 完成后（fire-and-forget，不阻塞）

**作用**：允许插件分析对话结果、记录统计等

```typescript
if (hookRunner?.hasHooks("agent_end")) {
  hookRunner
    .runAgentEnd(
      {
        messages: messagesSnapshot,
        success: !aborted && !promptError,
        error: promptError ? describeUnknownError(promptError) : undefined,
        durationMs: Date.now() - promptStartedAt,
      },
      {
        agentId: params.sessionKey?.split(":")[0] ?? "main",
        sessionKey: params.sessionKey,
        workspaceDir: params.workspaceDir,
        messageProvider: params.messageProvider ?? undefined,
      },
    )
    .catch((err) => {
      log.warn(`agent_end hook failed: ${err}`);
    });
}
```

**其他 Hook 点**（文档记载，代码未展开）：
- `before_tool_call` / `after_tool_call`: 工具调用前后
- `tool_result_persist`: 工具结果持久化时转换
- `before_compaction` / `after_compaction`: Compaction 前后
- `session_start` / `session_end`: Session 生命周期

---

### 18. 最终 Payload 构建

**函数**：`buildEmbeddedRunPayloads`（`src/agents/pi-embedded-runner/run/payloads.ts`）

**输入**：
- `assistantTexts`: 累积的 assistant 文本数组
- `toolMetas`: 工具调用元数据
- `lastAssistant`: 最后一条 assistant message（含 error）
- `lastToolError`: 最后一个工具错误
- `config`: OpenClaw 配置
- `verboseLevel`: 详细程度（影响是否显示 tool summary）

**输出**：`Payload[]`

**Payload 结构**（推测）：

```typescript
type Payload = {
  text: string;           // 文本内容
  isError?: boolean;      // 是否为错误消息
  isReasoning?: boolean;  // 是否为推理过程
  toolSummary?: string;   // 工具调用摘要（如 "Tools: bash (×3), canvas"）
};
```

**构建逻辑**（推测）：

```typescript
export function buildEmbeddedRunPayloads(params: {
  assistantTexts: string[];
  toolMetas: { toolName: string; meta?: string }[];
  lastAssistant?: AssistantMessage;
  lastToolError?: string;
  config: Config;
  sessionKey: string;
  verboseLevel?: string;
  reasoningLevel?: string;
  toolResultFormat: "markdown" | "plain";
  inlineToolResultsAllowed: boolean;
}): Payload[] {
  const payloads: Payload[] = [];
  
  // 1. Assistant texts (排除 NO_REPLY token)
  for (const text of params.assistantTexts) {
    const cleaned = text.replace(/NO_REPLY/g, "").trim();
    if (cleaned) {
      payloads.push({ text: cleaned });
    }
  }
  
  // 2. Tool summary (如果 verbose)
  if (params.verboseLevel === "high" && params.toolMetas.length > 0) {
    const summary = formatToolAggregate(params.toolMetas);
    if (summary) {
      payloads.push({ text: summary, toolSummary: summary });
    }
  }
  
  // 3. Error fallback
  if (payloads.length === 0 && params.lastAssistant?.errorMessage) {
    payloads.push({
      text: formatAssistantErrorText(params.lastAssistant, { cfg: params.config, sessionKey: params.sessionKey }),
      isError: true,
    });
  }
  
  return payloads;
}
```

---

## 🎯 与 AI 主动性的关联

### 1. Heartbeat 执行流程

**Heartbeat 如何触发 Agent Loop？**

**流程**：
```
HeartbeatRunner (src/infra/heartbeat-runner.ts)
  → 定时触发（cron expression）
  → 调用 Gateway RPC `agent` method
  → sessionKey = "main" (或配置的 agent ID)
  → prompt = "执行 Heartbeat 检查"（实际由 HEARTBEAT.md 注入）
  → runEmbeddedPiAgent({ sessionKey: "main", prompt: "..." })
```

**Heartbeat Prompt 注入**（`src/agents/pi-embedded-runner/run/attempt.ts:344-346`）：

```typescript
heartbeatPrompt: isDefaultAgent
  ? resolveHeartbeatPrompt(params.config?.agents?.defaults?.heartbeat?.prompt)
  : undefined,
```

**HEARTBEAT_OK 机制**（推测）：

Agent 在 System Prompt 中被告知：
- 如果检查结果"一切正常"，输出 `HEARTBEAT_OK` token
- Gateway 捕获此 token，不推送消息到 Channel
- 如果发现异常，输出实际消息内容，通过 Message tool 发送给用户

**Session Lane 隔离**：

Heartbeat 运行在 `session:main` lane，与其他 Agent run 串行执行，避免冲突。

---

### 2. Memory 自动索引触发

**Session Transcript 如何自动索引？**

**流程**（推测，需结合 Memory 模块代码）：

```
Agent run 完成
  → Session Manager flush pending tool results
  → `session.jsonl` 写入新对话
  → MemoryIndexManager 的 File Watcher 检测到变化
  → 触发增量索引：Chunking → Embedding → SQLite-vec 插入
```

**Memory Search Tool 调用流程**：

```
Agent 调用 `memory_search` tool
  → `src/agents/tools/memory-search-tool.ts`
  → MemoryIndexManager.search(query)
  → Hybrid Search: Vector Search (sqlite-vec) + FTS5 (关键词)
  → 返回 top-k results
  → Tool result 注入到 Agent context
```

**优化**：Memory Search 在 `before_agent_start` hook 中自动触发（根据用户 prompt 相关性）。

---

### 3. Skills 动态刷新机制

**Skills 如何在 Agent run 中生效？**

**加载时机**（`src/agents/pi-embedded-runner/run/attempt.ts:162-181`）：

```typescript
const shouldLoadSkillEntries = !params.skillsSnapshot || !params.skillsSnapshot.resolvedSkills;
const skillEntries = shouldLoadSkillEntries
  ? loadWorkspaceSkillEntries(effectiveWorkspace)
  : [];
restoreSkillEnv = params.skillsSnapshot
  ? applySkillEnvOverridesFromSnapshot({ snapshot: params.skillsSnapshot, config: params.config })
  : applySkillEnvOverrides({ skills: skillEntries ?? [], config: params.config });

const skillsPrompt = resolveSkillsPromptForRun({
  skillsSnapshot: params.skillsSnapshot,
  entries: shouldLoadSkillEntries ? skillEntries : undefined,
  config: params.config,
  workspaceDir: effectiveWorkspace,
});
```

**Skills Snapshot**：
- Gateway 启动时加载所有 Skills（bundled/managed/workspace）
- 缓存为 `skillsSnapshot`（避免每次 run 都扫描文件系统）
- File Watcher（chokidar）监控 Skills 目录变化，刷新 snapshot

**Skills 刷新触发**（推测自 `src/agents/skills/refresh.ts`）：

```
File Watcher 检测到 Skills 目录变化
  → refreshSkillsSnapshot()
  → 重新扫描 bundled/managed/workspace Skills
  → 更新 Gateway 内存中的 skillsSnapshot
  → 下一次 Agent run 自动生效
```

**Skills 工具调用**：

```
Agent 调用 `skill_search` / `skill_install` tool
  → 搜索 ClawdHub Registry
  → 下载 SKILL.md 到 ~/.openclaw/workspace/skills/managed/
  → File Watcher 触发 Skills 刷新
  → 下一次 Agent run 自动加载新 Skill
```

---

## 📊 性能优化设计

### 1. Session Manager Cache

**问题**：每次 `SessionManager.open(sessionFile)` 都要读取 `.clawd/session.jsonl`，I/O 密集。

**优化**（`src/agents/pi-embedded-runner/session-manager-cache.ts`）：

```typescript
// 预热 Session 文件（提前读入内存）
await prewarmSessionFile(params.sessionFile);

// 跟踪访问时间（用于 LRU 淘汰）
trackSessionManagerAccess(params.sessionFile);
```

**LRU Cache**（推测）：
- 缓存最近访问的 N 个 Session Manager 实例
- 超过容量时淘汰最久未访问的
- 避免频繁 open/close 开销

---

### 2. Tool Schema Split

**问题**：Sandbox 模式下，部分工具（如 bash elevated）需要隔离，不应暴露给 sandboxed session。

**优化**（`src/agents/pi-embedded-runner/tool-split.ts`）：

```typescript
const { builtInTools, customTools } = splitSdkTools({
  tools,
  sandboxEnabled: !!sandbox?.enabled,
});
```

**Split 策略**：
- **Built-in Tools**: Pi-agent-core 内置工具（read_file、write_file 等）
- **Custom Tools**: OpenClaw 自定义工具（bash、canvas、message 等）
- **Sandbox Filtering**: Sandbox 模式下过滤 elevated/sensitive 工具

---

### 3. Context Window Guard

**问题**：模型 context window 不足时，直接报错，用户体验差。

**优化**（`src/agents/context-window-guard.ts`）：

```typescript
const ctxGuard = evaluateContextWindowGuard({
  info: ctxInfo,
  warnBelowTokens: CONTEXT_WINDOW_WARN_BELOW_TOKENS,  // 如 2048
  hardMinTokens: CONTEXT_WINDOW_HARD_MIN_TOKENS,      // 如 1024
});
if (ctxGuard.shouldWarn) {
  log.warn(`low context window: ${provider}/${modelId} ctx=${ctxGuard.tokens}`);
}
if (ctxGuard.shouldBlock) {
  log.error(`blocked model (context window too small): ${provider}/${modelId} ctx=${ctxGuard.tokens}`);
  throw new FailoverError(
    `Model context window too small (${ctxGuard.tokens} tokens). Minimum is ${CONTEXT_WINDOW_HARD_MIN_TOKENS}.`,
    { reason: "unknown", provider, model: modelId },
  );
}
```

**Context Window 来源**（优先级）：
1. 用户配置：`config.agents.defaults.model.contextTokens`
2. Model Registry：`models.json` 中的 `contextWindow`
3. 默认值：`DEFAULT_CONTEXT_TOKENS`（如 200k）

---

### 4. Cache TTL Timestamp

**问题**：Anthropic Prompt Caching 需要在 system prompt 中插入 timestamp，避免缓存过期。

**优化**（`src/agents/pi-embedded-runner/cache-ttl.ts`）：

```typescript
const shouldTrackCacheTtl =
  params.config?.agents?.defaults?.contextPruning?.mode === "cache-ttl" &&
  isCacheTtlEligibleProvider(params.provider, params.modelId);

if (shouldTrackCacheTtl) {
  appendCacheTtlTimestamp(sessionManager, {
    timestamp: Date.now(),
    provider: params.provider,
    modelId: params.modelId,
  });
}
```

**机制**：
- 在 system prompt 末尾插入：`<!-- cache-ttl: 1738234567890 -->`
- 每次 run 更新 timestamp
- Anthropic 根据 timestamp 判断缓存是否有效（5 分钟内相同）

---

## 🔍 关键设计模式总结

### 1. 队列化执行（Command Queue）

**优势**：
- **Session 一致性**：同一 session 的 run 串行执行，避免 race condition
- **并发控制**：Global lane 限制全局并发，防止 CPU/内存爆炸
- **可观测性**：队列等待时间、执行时长清晰可见

**实现细节**：
- 每个 lane 独立的 queue + active counter
- `drainLane` 泵函数持续消费队列
- 支持自定义 `maxConcurrent`（如 cron lane 设为 3）

---

### 2. 智能重试（Auth Profile Rotation + Thinking Level Fallback + Auto-Compaction）

**优势**：
- **高可用性**：单个 Auth profile rate limit 不影响整体服务
- **自适应**：根据错误类型自动调整策略
- **透明化**：用户无感知的自动修复

**策略组合**：
1. **Rate Limit** → 轮换 Auth Profile
2. **Thinking Not Supported** → 降级 Thinking Level
3. **Context Overflow** → 自动 Compaction + 重试
4. **Auth Failure** → 跳过该 profile + 标记 cooldown

---

### 3. 流式架构（Stream Everything）

**优势**：
- **实时反馈**：用户立即看到 Agent 在做什么
- **可中断**：任何时刻可 abort
- **可扩展**：新增流式事件无需改动核心逻辑

**流式层级**：
1. **Pi-agent-core 事件流**：`message_start`/`update`/`end`、`tool_execution_*`
2. **OpenClaw 事件流**：`assistant`/`tool`/`lifecycle` stream
3. **Gateway WebSocket 流**：实时推送给客户端（macOS app、CLI、WebChat）

---

### 4. Hook 驱动扩展（Plugin System）

**优势**：
- **解耦**：核心流程不依赖具体插件
- **灵活性**：插件可动态加载/卸载
- **可组合**：多个插件 hook 同一事件，按顺序执行

**Hook 执行策略**：
- **Sync Hooks**：按顺序执行，可修改参数（如 `before_agent_start`）
- **Async Hooks**：并发执行，不阻塞主流程（如 `agent_end`）
- **Error Handling**：Hook 失败不影响 Agent 执行，只打印警告日志

---

## 🐛 潜在问题与优化方向

### 1. 队列阻塞风险

**问题**：如果某个 session 的 run 长时间不结束（如工具卡住），后续同 session 的 run 会被阻塞。

**解决方案**：
- **Timeout 机制**：已实现（默认 600s）
- **Abort 机制**：已实现（可主动 abort）
- **潜在优化**：支持 session lane 级别的 timeout 监控 + 自动 abort

---

### 2. Auth Profile Cooldown 策略过于保守

**问题**：Rate Limit 429 触发 60s cooldown，但实际 rate limit 可能是 1 分钟内请求数，cooldown 结束后可能立即再次触发。

**优化方向**：
- **指数退避**：cooldown 时间随失败次数增加（60s → 120s → 300s）
- **动态调整**：根据 rate limit header（如 `X-RateLimit-Reset`）精确计算 cooldown
- **Profile 优先级**：根据成功率动态调整 profile 顺序

---

### 3. Block Chunking 延迟感

**问题**：`minChars = 100` 时，前 100 字符会有明显延迟。

**优化方向**：
- **自适应 minChars**：根据网络延迟动态调整（低延迟 → 小块，高延迟 → 大块）
- **渐进式推送**：前 N 个字符立即推送，后续按 minChars 分块
- **Sentence Boundary 优化**：优先在句号处切分，即使未达到 minChars

---

### 4. Memory Search 召回策略

**问题**：Hybrid Search 的 Vector Search 和 FTS5 权重如何平衡？

**优化方向**（需查看 Memory 模块代码验证）：
- **动态权重**：根据查询类型调整（事实性问题 → FTS5 权重高，语义问题 → Vector 权重高）
- **Rerank**：使用 Cross-Encoder 对初步召回结果重排序
- **Query Expansion**：用 LLM 生成同义查询，扩大召回范围

---

## 📝 可借鉴的设计点

### 1. 队列化 + Lane 隔离

**适用场景**：任何需要串行执行的异步任务系统（如批处理、工作流引擎）。

**核心思想**：
- 每个"资源"（如 session）对应一个 lane
- 同 lane 内串行，跨 lane 并发
- 支持 lane 级别的并发控制（`maxConcurrent`）

**代码示例**（简化版）：

```typescript
class LaneQueue {
  private lanes = new Map<string, { queue: Task[], active: number }>();
  
  enqueue(lane: string, task: Task) {
    const state = this.lanes.get(lane) || { queue: [], active: 0 };
    state.queue.push(task);
    this.lanes.set(lane, state);
    this.drain(lane);
  }
  
  private async drain(lane: string) {
    const state = this.lanes.get(lane)!;
    if (state.active > 0 || state.queue.length === 0) return;
    
    const task = state.queue.shift()!;
    state.active = 1;
    try {
      await task.run();
    } finally {
      state.active = 0;
      this.drain(lane);
    }
  }
}
```

---

### 2. Failover Error 类型系统

**适用场景**：需要区分不同错误类型并执行不同重试策略的系统。

**核心思想**：
- 定义统一的 `FailoverError` 类，包含 `reason` 字段
- 根据 `reason` 决定是否重试、如何重试
- 支持 fallback 链（Profile → Model → Provider）

**代码示例**：

```typescript
class RetryableError extends Error {
  constructor(
    message: string,
    public reason: "auth" | "rate_limit" | "timeout",
    public retryAfterMs?: number,
  ) {
    super(message);
  }
}

async function executeWithRetry<T>(
  fn: () => Promise<T>,
  fallbacks: Config[],
): Promise<T> {
  for (const config of fallbacks) {
    try {
      return await fn();
    } catch (err) {
      if (err instanceof RetryableError) {
        if (err.reason === "rate_limit" && err.retryAfterMs) {
          await sleep(err.retryAfterMs);
          continue;
        }
        // 尝试下一个 fallback
      } else {
        throw err;
      }
    }
  }
  throw new Error("All fallbacks exhausted");
}
```

---

### 3. 流式 Block Chunker

**适用场景**：需要将流式文本切分成"可读块"的场景（如 Markdown 渲染、代码补全）。

**核心思想**：
- 维护 buffer，累积 delta
- 按 minChars/maxChars + 语义边界（段落/句子）切分
- Fence-aware：不在代码块内切分（避免破坏语法高亮）

**代码示例**（简化版）：

```typescript
class BlockChunker {
  private buffer = "";
  
  append(text: string) {
    this.buffer += text;
  }
  
  drain(minChars: number, maxChars: number, emit: (chunk: string) => void) {
    while (this.buffer.length >= minChars) {
      const breakIdx = this.findBreak(minChars, maxChars);
      if (breakIdx <= 0) break;
      
      emit(this.buffer.slice(0, breakIdx));
      this.buffer = this.buffer.slice(breakIdx);
    }
  }
  
  private findBreak(minChars: number, maxChars: number): number {
    // 1. 找段落分隔符 \\n\\n
    let idx = this.buffer.indexOf("\\n\\n", minChars);
    if (idx !== -1 && idx <= maxChars) return idx;
    
    // 2. 找换行符 \\n
    idx = this.buffer.indexOf("\\n", minChars);
    if (idx !== -1 && idx <= maxChars) return idx;
    
    // 3. 找句号 .!?
    const match = this.buffer.slice(minChars, maxChars).match(/[.!?]/);
    if (match) return minChars + match.index! + 1;
    
    // 4. 强制切分（达到 maxChars）
    return maxChars;
  }
}
```

---

### 4. Plugin Hook 系统

**适用场景**：需要第三方扩展的系统（如 Webpack、Vite、VS Code Extension）。

**核心思想**：
- 定义 Hook 点（如 `before_build`、`after_build`）
- 插件注册 hook handler
- Hook runner 按顺序执行 handlers，收集结果

**代码示例**（简化版）：

```typescript
type Hook = "before_build" | "after_build";
type HookHandler<T> = (data: T) => T | Promise<T>;

class PluginSystem {
  private hooks = new Map<Hook, HookHandler<any>[]>();
  
  registerHook<T>(hook: Hook, handler: HookHandler<T>) {
    const handlers = this.hooks.get(hook) || [];
    handlers.push(handler);
    this.hooks.set(hook, handlers);
  }
  
  async runHook<T>(hook: Hook, data: T): Promise<T> {
    const handlers = this.hooks.get(hook) || [];
    let result = data;
    for (const handler of handlers) {
      result = await handler(result);
    }
    return result;
  }
}

// 使用
const plugins = new PluginSystem();
plugins.registerHook("before_build", async (ctx) => {
  // 注入额外上下文
  ctx.extraData = await fetchExtraData();
  return ctx;
});

let buildCtx = { input: "..." };
buildCtx = await plugins.runHook("before_build", buildCtx);
```

---

## 🎓 总结

OpenClaw 的 Agent Loop 是一个**高度工程化的 LLM 执行引擎**，其核心特点：

1. **可靠性**：队列化执行、智能重试、Failover 机制保证高可用
2. **实时性**：Tool streaming + Block streaming 实现毫秒级响应
3. **可扩展**：Plugin Hook 系统支持第三方扩展
4. **可观测**：详细日志、Diagnostic events、Cache trace 便于调试

**与 AI 主动性的关联**：
- **Heartbeat**：通过 `session:main` lane 定期执行，System Prompt 注入 HEARTBEAT.md
- **Memory**：File Watcher 自动索引 Session Transcript，`before_agent_start` hook 自动召回
- **Skills**：File Watcher 自动刷新 Skills Snapshot，下一次 run 自动生效

**可借鉴的设计模式**：
- Lane-based Queue：资源隔离 + 并发控制
- Failover Error System：错误分类 + 智能重试
- Stream Architecture：实时反馈 + 可中断
- Plugin Hook System：解耦 + 可扩展

---

**下一步研究方向**（第3层 Session 管理）：
- Session 隔离模型：main vs group sessions
- Session 状态持久化：`session.jsonl` 格式
- Agent to Agent 通信：`sessions_*` tools 实现
- Compaction 机制：如何压缩历史对话

道友，Agent Loop 的核心流程就是这样啦~ 璇玑这就去研究下一层咯！✨
