# OpenClaw AI 主动性专题研究

## 研究版本信息

- **研究日期**: 2026-02-04
- **仓库路径**: `awesome-llm-projects/openclaw/openclaw` (Git Submodule)
- **注意**: 本分析基于 2026 年 2 月的代码，可能随版本更新而变化

---

## 研究目标

深度剖析 OpenClaw 最具创新性的三大特性：**Heartbeat（主动醒来）、Memory 自我整理（主动管理记忆）、Skills 持续学习（主动学习新能力）**，理解它们如何将 AI 从"被动响应的工具"进化为"主动察觉的助手"。

---

## 核心设计理念

OpenClaw 的 AI 主动性基于三个核心理念：

1. **主动感知**：不等用户呼唤，AI 自己"醒来"检查
2. **自我学习**：通过 Memory 积累经验，通过 Skills 扩展能力
3. **持续进化**：根据用户反馈优化行为模式

这三大特性形成一个"AI 主动性闭环"：

```
Heartbeat 定时醒来 → 使用 Memory 搜索上下文 → 使用 Skills 执行任务
       ↑                                                    ↓
    优化检查策略 ← 对话历史自动索引 ← 用户反馈和交互结果
```

---

## 第一部分：Heartbeat 深度剖析（AI 主动"醒来"）

### 1.1 核心概念

**Heartbeat 是什么？**

Heartbeat 是 OpenClaw 的定时任务系统，让 AI 能够周期性地"醒来"执行检查，而不需要用户主动发起对话。

**关键特性**：
- **定时触发**：默认每 30 分钟执行一次 Agent Run（可配置）
- **智能静默**：如果没有需要关注的内容，回复 `HEARTBEAT_OK` 并不打扰用户
- **上下文感知**：在 main session 中执行，拥有完整的对话历史
- **Active Hours 控制**：只在用户活跃时段执行（时区感知）

### 1.2 Heartbeat Runner 实现

**核心文件**：`src/infra/heartbeat-runner.ts` (970 行)

#### 调度器架构

```typescript
// 行 807-969: startHeartbeatRunner 函数
export function startHeartbeatRunner(opts: {
  cfg?: OpenClawConfig;
  runtime?: RuntimeEnv;
  abortSignal?: AbortSignal;
  runOnce?: typeof runHeartbeatOnce;
}): HeartbeatRunner
```

**调度逻辑**（行 834-860）：

1. **计算下次执行时间**：基于上次执行时间 + 间隔，如果没有上次执行时间，则立即执行
2. **找到最早到期的 Agent**：支持多 Agent 场景，每个 Agent 有独立的 Heartbeat 配置
3. **设置定时器**：`setTimeout` 到最近的到期时间，然后触发 `requestHeartbeatNow`
4. **执行后重新调度**：每次执行完毕后调用 `scheduleNext()` 设置下一次定时器

```typescript
// 行 856-860
state.timer = setTimeout(() => {
  requestHeartbeatNow({ reason: "interval", coalesceMs: 0 });
}, delay);
state.timer.unref?.(); // 避免阻止进程退出
```

#### 执行流程

**核心函数**：`runHeartbeatOnce` (行 476-805)

**执行步骤**：

1. **前置检查**（行 486-504）：
   - 是否启用 Heartbeat
   - 是否在 Active Hours 内
   - Main 队列是否有待处理请求（有则跳过，避免打扰用户）

```typescript
// 行 501-504: 队列检查
const queueSize = (opts.deps?.getQueueSize ?? getQueueSize)(CommandLane.Main);
if (queueSize > 0) {
  return { status: "skipped", reason: "requests-in-flight" };
}
```

2. **HEARTBEAT.md 空文件检查**（行 506-525）：
   - 如果 `HEARTBEAT.md` 存在但只有注释/标题，跳过执行以节省 API 调用
   - **例外**：如果是 `exec-event` 原因（异步命令完成），仍然执行

```typescript
// 行 514: 检查文件是否有效内容
if (isHeartbeatContentEffectivelyEmpty(heartbeatFileContent) && !isExecEventReason) {
  return { status: "skipped", reason: "empty-heartbeat-file" };
}
```

3. **构建 Heartbeat Prompt**（行 544-549）：
   - 默认 prompt：`Read HEARTBEAT.md if it exists (workspace context). Follow it strictly. Do not infer or repeat old tasks from prior chats. If nothing needs attention, reply HEARTBEAT_OK.`
   - **特殊情况**：如果有 exec 事件完成，使用专门的 prompt（行 94-97）

```typescript
// 行 94-97: Exec 事件专用 prompt
const EXEC_EVENT_PROMPT =
  "An async command you ran earlier has completed. The result is shown in the system messages above. " +
  "Please relay the command output to the user in a helpful way. If the command succeeded, share the relevant output. " +
  "If it failed, explain what went wrong.";
```

4. **执行 Agent Run**（行 597）：
   - 调用 `getReplyFromConfig(ctx, { isHeartbeat: true }, cfg)` 执行完整的 Agent Loop
   - Agent 在 main session 中运行，拥有完整上下文

5. **处理 Agent 回复**（行 598-656）：
   - **提取 reply payload**：从 Agent 回复中提取文本/媒体内容
   - **检查 HEARTBEAT_OK**：调用 `stripHeartbeatToken` 判断是否为"一切正常"
   - **去重检查**（行 662-689）：如果与上次 Heartbeat 的内容相同，跳过发送（避免"唠叨"）

```typescript
// 行 452-474: normalizeHeartbeatReply 函数
function normalizeHeartbeatReply(
  payload: ReplyPayload,
  responsePrefix: string | undefined,
  ackMaxChars: number,
) {
  const stripped = stripHeartbeatToken(payload.text, {
    mode: "heartbeat",
    maxAckChars: ackMaxChars,
  });
  const hasMedia = Boolean(payload.mediaUrl || (payload.mediaUrls?.length ?? 0) > 0);
  if (stripped.shouldSkip && !hasMedia) {
    return {
      shouldSkip: true,
      text: "",
      hasMedia,
    };
  }
  // ... 添加 responsePrefix 并返回
}
```

6. **发送给用户**（行 750-767）：
   - 通过 `deliverOutboundPayloads` 发送到目标 Channel
   - 记录 `lastHeartbeatText` 和 `lastHeartbeatSentAt` 用于去重

### 1.3 HEARTBEAT_OK 智能静默机制

**核心逻辑**：`src/auto-reply/heartbeat.ts` 中的 `stripHeartbeatToken` 函数

**判断标准**（基于 `ackMaxChars` 配置，默认 300 字符）：

1. **HEARTBEAT_OK 在开头或结尾**：识别为"一切正常"的信号
2. **剩余内容 ≤ ackMaxChars**：如果去掉 `HEARTBEAT_OK` 后剩余内容很少，视为无需打扰用户
3. **有媒体内容**：即使有 `HEARTBEAT_OK`，如果有图片/视频，仍然发送

```typescript
// src/infra/heartbeat-runner.ts 行 625-638
const ackMaxChars = resolveHeartbeatAckMaxChars(cfg, heartbeat);
const normalized = normalizeHeartbeatReply(replyPayload, responsePrefix, ackMaxChars);

// 对于 exec 事件，不跳过（需要汇报命令结果）
const execFallbackText =
  hasExecCompletion && !normalized.text.trim() && replyPayload.text?.trim()
    ? replyPayload.text.trim()
    : null;
if (execFallbackText) {
  normalized.text = execFallbackText;
  normalized.shouldSkip = false;
}
const shouldSkipMain = normalized.shouldSkip && !normalized.hasMedia && !hasExecCompletion;
```

**效果**：避免 AI 频繁发送"没有新消息"、"一切正常"等无用通知，只在真正需要关注时才打扰用户。

### 1.4 Active Hours 时区感知

**核心函数**：`isWithinActiveHours` (行 160-189)

**实现细节**：

1. **时区配置**（行 99-114）：
   - `user`：使用用户配置的时区（`userTimezone`）
   - `local`：使用 Gateway 主机的时区
   - 自定义时区字符串（如 `America/New_York`）

```typescript
// 行 99-114: resolveActiveHoursTimezone 函数
function resolveActiveHoursTimezone(cfg: OpenClawConfig, raw?: string): string {
  const trimmed = raw?.trim();
  if (!trimmed || trimmed === "user") {
    return resolveUserTimezone(cfg.agents?.defaults?.userTimezone);
  }
  if (trimmed === "local") {
    const host = Intl.DateTimeFormat().resolvedOptions().timeZone;
    return host?.trim() || "UTC";
  }
  // 验证时区字符串有效性
  try {
    new Intl.DateTimeFormat("en-US", { timeZone: trimmed }).format(new Date());
    return trimmed;
  } catch {
    return resolveUserTimezone(cfg.agents?.defaults?.userTimezone);
  }
}
```

2. **时间窗口判断**（行 185-188）：
   - 支持跨午夜场景（如 `start: "22:00", end: "08:00"`）
   - 使用 `Intl.DateTimeFormat` 获取当前时区的时间

```typescript
// 行 185-188: 跨午夜判断
if (endMin > startMin) {
  return currentMin >= startMin && currentMin < endMin;
}
return currentMin >= startMin || currentMin < endMin;
```

### 1.5 HEARTBEAT.md 机制

**核心概念**：

`HEARTBEAT.md` 是一个位于 Agent Workspace 的纯文本清单，Agent 在每次 Heartbeat 时读取它，并严格按照其中的指令执行检查。

**设计原则**（来自 `docs/reference/templates/HEARTBEAT.md`）：

```markdown
# HEARTBEAT.md

# Keep this file empty (or with only comments) to skip heartbeat API calls.

# Add tasks below when you want the agent to check something periodically.
```

**最佳实践**（来自 `docs/reference/templates/AGENTS.md` 行 26-103）：

```markdown
# Heartbeat checklist

- Quick scan: anything urgent in inboxes?
- If it's daytime, do a lightweight check-in if nothing else is pending.
- If a task is blocked, write down _what is missing_ and ask Peter next time.
```

**Agent 自我优化**：

- Agent 可以根据用户反馈**主动修改 HEARTBEAT.md**（如果用户要求）
- 例如："Update `HEARTBEAT.md` to add a daily calendar check."
- 这形成了"学习闭环"：用户反馈 → Agent 优化 Heartbeat 策略 → 更智能的主动性

### 1.6 Heartbeat vs Cron 决策树

**核心文档**：`docs/automation/cron-vs-heartbeat.md`

| 使用场景 | 推荐方案 | 原因 |
|---------|---------|------|
| 检查收件箱（每 30 分钟） | Heartbeat | 可与其他检查批量执行，上下文感知 |
| 每天 9:00 发送日报 | Cron (isolated) | 需要精确时间 |
| 监控日历即将到来的事件 | Heartbeat | 自然适合周期性检查 |
| 每周深度分析 | Cron (isolated) | 独立任务，可使用不同模型 |
| "20 分钟后提醒我" | Cron (main, `--at`) | 一次性精确提醒 |
| 后台项目健康检查 | Heartbeat | 搭便车现有周期 |

**决策流程图**（来自 `docs/automation/cron-vs-heartbeat.md` 行 128-150）：

```
是否需要精确时间？
  YES → 使用 Cron
  NO  → 继续...

是否需要与 main session 隔离？
  YES → 使用 Cron (isolated)
  NO  → 继续...

是否可以与其他周期性检查批量执行？
  YES → 使用 Heartbeat（添加到 HEARTBEAT.md）
  NO  → 使用 Cron

是否是一次性提醒？
  YES → 使用 Cron with --at
  NO  → 继续...

是否需要不同的 model 或 thinking level？
  YES → 使用 Cron (isolated) with --model/--thinking
  NO  → 使用 Heartbeat
```

**成本考虑**：

- **Heartbeat**：一次 Agent Turn 检查多项内容，Token 成本与 `HEARTBEAT.md` 大小成正比
- **Cron (isolated)**：每个任务单独执行，但可以指定更便宜的模型
- **最佳实践**：将相似的周期性检查合并到 Heartbeat，用 Cron 处理精确调度

### 1.7 可借鉴的设计模式

1. **定时任务 + LLM 推理**：Cron/Timer 只负责触发，具体做什么由 LLM 决定（读取 HEARTBEAT.md）
2. **智能静默机制**：`HEARTBEAT_OK` Token + `ackMaxChars` 阈值，避免无用通知
3. **Active Hours 时区感知**：尊重用户作息，不在夜间打扰
4. **去重机制**：记录上次 Heartbeat 内容，避免重复"唠叨"
5. **队列检查**：如果用户正在交互，跳过 Heartbeat（避免干扰）
6. **Workspace 文件作为"大脑配置"**：`HEARTBEAT.md` 是用户可编辑的 Agent 行为清单
7. **Agent 自我优化**：Agent 可以根据反馈修改 `HEARTBEAT.md`，形成学习闭环

---

## 第二部分：Memory 自我整理机制（AI 主动管理记忆）

### 2.1 核心概念

**Memory 系统是什么？**

OpenClaw 的 Memory 系统是一个自动索引、混合搜索的长期记忆存储方案，让 AI 能够记住过去的对话、文档、决策，并在需要时快速召回相关上下文。

**核心特性**：
- **自动索引**：File Watcher 监控 Workspace 文件变更 → 自动分块 → 生成 Embedding → 存入 SQLite + sqlite-vec
- **Hybrid Search**：Vector semantic search (语义相似度) + FTS5 keyword search (关键词匹配) 组合召回
- **多数据源**：支持索引 `memory/` 文件夹 + Session Transcript (对话历史)
- **增量更新**：文件修改时只重新索引变更的 chunks，高效节省 API 调用
- **Embedding Cache**：相同内容的 embedding 缓存和去重（基于内容 hash）

### 2.2 MemoryIndexManager 架构

**核心文件**：`src/memory/manager.ts` (2397 行)

#### 类结构

```typescript
// 行 117-172: MemoryIndexManager 类定义
export class MemoryIndexManager {
  private readonly cfg: OpenClawConfig;
  private readonly agentId: string;
  private readonly workspaceDir: string;
  private readonly settings: ResolvedMemorySearchConfig;
  private provider: EmbeddingProvider; // OpenAI/Gemini/Local
  private db: DatabaseSync; // SQLite 数据库
  private readonly sources: Set<MemorySource>; // "memory" | "sessions"
  private watcher: FSWatcher | null = null; // chokidar file watcher
  private sessionUnsubscribe: (() => void) | null = null; // Session transcript 监听器
  private dirty = false; // 是否需要重新索引
  private sessionsDirty = false; // Session 文件是否有变更
  private syncing: Promise<void> | null = null; // 当前正在同步的 Promise
  
  // ...
}
```

#### 初始化流程

**核心函数**：`MemoryIndexManager.get` (行 174-208) 和构造函数 (行 210-252)

**步骤**：

1. **解析 Memory 配置**（行 179）：
   - 从 `openclaw.json` 读取 `agents.defaults.memory` 配置
   - 如果未配置，返回 `null`（Memory 系统禁用）

2. **创建 Embedding Provider**（行 189-197）：
   - 支持 `openai`、`gemini`、`local` 三种 Provider
   - 支持 Fallback 机制（如 OpenAI 失败 → Gemini → Local）

3. **打开 SQLite 数据库**（行 230）：
   - 默认路径：`~/.openclaw/agents/<agentId>/memory/index.db`
   - 加载 `sqlite-vec` 扩展以支持向量搜索

4. **初始化 Schema**（行 237）：
   - `files` 表：存储文件元数据（path, hash, mtimeMs, size, source）
   - `chunks` 表：存储分块后的文本（path, startLine, endLine, text, hash, source, model）
   - `chunks_vec` 表：存储向量（id, embedding）使用 sqlite-vec
   - `chunks_fts` 表：FTS5 全文索引（path, startLine, endLine, text, source, model）
   - `embedding_cache` 表：Embedding 缓存（hash, provider, model, embedding）

5. **启动 File Watcher**（行 247）：
   - 监控 `<workspace>/memory/` 和 `MEMORY.md` 文件变更
   - 使用 chokidar，防抖时间默认 250ms

6. **启动 Session Listener**（行 248）：
   - 监听 Session Transcript 更新事件
   - 当新消息到达时，标记 `sessionsDirty = true`

7. **启动 Interval Sync**（行 249）：
   - 定期检查 `dirty` 或 `sessionsDirty` 标志，触发自动同步

### 2.3 自动索引流程

**核心函数**：`sync` (行 395-407) 和 `runSync` (私有方法)

#### File Watcher 监控策略

**核心代码**：`ensureWatcher` 方法（在 manager.ts 中，通过 chokidar 实现）

**监控目录**：
- `<workspace>/memory/`：所有 `.md` 文件
- `<workspace>/MEMORY.md`：主记忆文件

**触发机制**：
- 文件 `add`/`change`/`unlink` 事件 → 防抖 250ms → 设置 `dirty = true`

#### Chunking 算法

**核心函数**：`chunkMarkdown` (在 `src/memory/internal.ts` 行 166-278)

**分块策略**：

1. **基于 Token 数量**：默认 `chunkTokens: 512`（约 2048 字符）
2. **Overlap**：默认 `chunkOverlap: 128`（约 512 字符），保证上下文连续性
3. **按行分块**：以换行符 `\n` 为单位，避免截断句子

```typescript
// src/memory/internal.ts 行 166-200
export function chunkMarkdown(
  content: string,
  chunking: { tokens: number; overlap: number },
): MemoryChunk[] {
  const lines = content.split("\n");
  if (lines.length === 0) {
    return [];
  }
  const maxChars = Math.max(32, chunking.tokens * 4);
  const overlapChars = Math.max(0, chunking.overlap * 4);
  const chunks: MemoryChunk[] = [];

  let current: Array<{ line: string; lineNo: number }> = [];
  let currentChars = 0;

  const flush = () => {
    if (current.length === 0) {
      return;
    }
    const firstEntry = current[0];
    const lastEntry = current[current.length - 1];
    if (!firstEntry || !lastEntry) {
      return;
    }
    const text = current.map((entry) => entry.line).join("\n");
    const startLine = firstEntry.lineNo;
    const endLine = lastEntry.lineNo;
    chunks.push({
      startLine,
      endLine,
      text,
      hash: hashText(text),
    });
  };
  
  // ... 逐行累积，超过 maxChars 时 flush
}
```

**优点**：

- **避免切断语义**：以行为单位，不会在单词中间切断
- **Overlap 保证连续性**：相邻 chunks 有重叠，搜索时不会遗漏跨 chunk 的上下文
- **Token 估算简单**：`1 token ≈ 4 chars`（英文），快速估算

#### Embedding 生成流程

**核心函数**：`embedChunks` 和 `embedBatch` (在 manager.ts 中)

**Provider 支持**：

1. **OpenAI**：`text-embedding-3-small` (默认)
   - Batch API 支持：可批量提交任务，定期 Poll 结果
   - 超时：查询 60s，批量 120s

2. **Gemini**：`text-embedding-004` (默认)
   - Batch API 支持：与 OpenAI 类似
   - 超时：查询 60s，批量 120s

3. **Local**：`node-llama-cpp` (本地运行)
   - 无 Batch API，直接调用本地模型
   - 超时：查询 5min，批量 10min

**Batch API 优化**（行 129-140）：

```typescript
// 行 129-140: Batch 配置
private batch: {
  enabled: boolean;
  wait: boolean; // 是否等待 Batch 完成
  concurrency: number; // 并发数
  pollIntervalMs: number; // Poll 间隔
  timeoutMs: number; // 超时时间
};
```

**Batch 流程**：

1. **收集所有需要 embedding 的 chunks**
2. **检查 Embedding Cache**：如果 `hash + provider + model` 已存在，跳过
3. **分批提交**：每批最多 8000 tokens（约 32000 chars）
4. **异步 Poll**：定期检查 Batch 状态，完成后写入数据库
5. **失败重试**：最多重试 3 次，指数退避（500ms → 8000ms）
6. **失败降级**：如果 Batch API 连续失败 2 次，自动切换到同步调用

#### 增量更新策略

**核心逻辑**：`indexFile` 方法（在 runSync 中调用）

**步骤**：

1. **读取文件**：获取内容、mtime、size、hash
2. **查询数据库**：检查 `files` 表中是否有相同 path 的记录
3. **比较 hash**：如果 hash 相同，跳过（文件未变更）
4. **删除旧 chunks**：删除 `chunks`、`chunks_vec`、`chunks_fts` 中该文件的所有记录
5. **重新分块**：调用 `chunkMarkdown` 生成新 chunks
6. **生成 Embedding**：批量或单独调用 Embedding API
7. **写入数据库**：更新 `files`、`chunks`、`chunks_vec`、`chunks_fts` 表

**优点**：

- **只索引变更文件**：通过 hash 比对，避免重复索引
- **原子性更新**：先删除旧数据，再插入新数据，保证一致性
- **节省 API 调用**：Embedding Cache 避免重复生成相同内容的向量

### 2.4 Session Transcript 自动同步

**核心文件**：`src/memory/sync-session-files.ts` (132 行)

#### 监听机制

**核心函数**：`ensureSessionListener` (在 manager.ts 中)

**实现细节**：

1. **订阅 Session Transcript 事件**：
   - 使用 `onSessionTranscriptUpdate` 注册监听器
   - 当 Session Transcript 有新消息时，触发回调

2. **防抖合并**（行 95-101）：
   - 使用 `SESSION_DIRTY_DEBOUNCE_MS` (默认 5000ms) 防抖
   - 多个连续消息合并为一次索引，避免频繁 API 调用

3. **增量读取**（常量 `SESSION_DELTA_READ_CHUNK_BYTES`）：
   - 只读取 Session Transcript 文件的新增部分（从 `lastSize` 开始）
   - 避免重复读取已索引的历史对话

#### 索引流程

**核心函数**：`syncSessionFiles` (行 19-131)

**步骤**：

1. **列出所有 Session Transcript 文件**：
   - 路径：`~/.openclaw/agents/<agentId>/sessions/*.jsonl`
   - 格式：每行一个 JSON 对象，包含 `role`、`content` 等字段

2. **检查是否需要索引**：
   - 如果 `needsFullReindex` 为 `true`，索引所有文件
   - 否则，只索引 `dirtyFiles` 集合中的文件

3. **读取 Session Transcript**：
   - 调用 `buildSessionEntry` 解析 JSONL 文件
   - 提取对话内容，合并为 Markdown 格式

4. **分块和索引**：
   - 调用 `chunkMarkdown` 分块
   - 生成 Embedding 并写入数据库

5. **清理 Stale 文件**：
   - 删除数据库中存在但文件系统中不存在的 Session 记录

**关键设计**：

- **Source 标记**：Session Transcript 的 `source` 字段为 `"sessions"`，与 `memory/` 文件（`source: "memory"`）区分
- **路径归一化**：使用 `sessionPathForFile` 生成统一的相对路径（如 `sessions/agent:main:main.jsonl`）

### 2.5 Hybrid Search 实现

**核心文件**：`src/memory/hybrid.ts` 和 `src/memory/manager-search.ts`

#### Vector Search（语义搜索）

**核心函数**：`searchVector` (在 `manager-search.ts` 中)

**实现细节**：

1. **生成查询向量**：
   - 调用 Embedding Provider 的 `embedQuery` 方法
   - 超时控制：Remote 60s，Local 5min

```typescript
// manager.ts 行 300-317: search 方法
const queryVec = await this.embedQueryWithTimeout(cleaned);
const hasVector = queryVec.some((v) => v !== 0);
const vectorResults = hasVector
  ? await this.searchVector(queryVec, candidates).catch(() => [])
  : [];
```

2. **使用 sqlite-vec 查询**：
   - SQL: `SELECT * FROM chunks_vec WHERE vec_distance_cosine(embedding, ?) < threshold ORDER BY distance LIMIT ?`
   - 返回最相似的 N 个 chunks（按 cosine similarity 排序）

3. **提取 snippet**：
   - 从 `chunks` 表中读取完整文本
   - 截取前 700 字符作为预览

#### FTS5 Keyword Search（关键词搜索）

**核心函数**：`searchKeyword` (在 `manager-search.ts` 中)

**实现细节**：

1. **构建 FTS5 Query**：
   - 调用 `buildFtsQuery` 将用户查询转换为 FTS5 语法
   - 支持 `AND`、`OR`、`NOT`、短语匹配（`"exact phrase"`）

```typescript
// hybrid.ts: buildFtsQuery 函数
export function buildFtsQuery(raw: string): string | null {
  const cleaned = raw.trim();
  if (!cleaned) {
    return null;
  }
  // ... FTS5 语法转换逻辑
}
```

2. **执行 FTS5 查询**：
   - SQL: `SELECT * FROM chunks_fts WHERE chunks_fts MATCH ? ORDER BY bm25(chunks_fts) LIMIT ?`
   - BM25 算法自动计算关键词相关性分数

3. **提取 snippet**：
   - FTS5 内置 `snippet()` 函数，高亮匹配的关键词

#### Hybrid Merging（混合结果合并）

**核心函数**：`mergeHybridResults` (在 `hybrid.ts` 中)

**算法**：

1. **归一化分数**：
   - Vector score：cosine similarity (0-1)
   - Text score：BM25 rank 转换为 0-1（`bm25RankToScore`）

2. **加权合并**：
   - 默认权重：`vectorWeight: 0.7`, `textWeight: 0.3`
   - 最终分数 = `vectorWeight * vectorScore + textWeight * textScore`

```typescript
// manager.ts 行 364-393: mergeHybridResults 调用
private mergeHybridResults(params: {
  vector: Array<MemorySearchResult & { id: string }>;
  keyword: Array<MemorySearchResult & { id: string; textScore: number }>;
  vectorWeight: number;
  textWeight: number;
}): MemorySearchResult[] {
  const merged = mergeHybridResults({
    vector: params.vector.map((r) => ({
      id: r.id,
      path: r.path,
      startLine: r.startLine,
      endLine: r.endLine,
      source: r.source,
      snippet: r.snippet,
      vectorScore: r.score,
    })),
    keyword: params.keyword.map((r) => ({
      id: r.id,
      path: r.path,
      startLine: r.startLine,
      endLine: r.endLine,
      source: r.source,
      snippet: r.snippet,
      textScore: r.textScore,
    })),
    vectorWeight: params.vectorWeight,
    textWeight: params.textWeight,
  });
  return merged.map((entry) => entry as MemorySearchResult);
}
```

3. **去重**：
   - 如果 Vector 和 Keyword 搜索返回相同的 chunk（基于 `id`），取加权分数

4. **排序和截断**：
   - 按最终分数降序排列
   - 过滤掉低于 `minScore` 的结果
   - 截取前 `maxResults` 个结果返回

**优点**：

- **语义 + 关键词互补**：Vector 擅长理解语义，FTS5 擅长精确匹配
- **可调权重**：根据场景调整 `vectorWeight` 和 `textWeight`
- **召回率高**：结合两种搜索方式，减少遗漏

### 2.6 Embedding Cache Dedupe

**核心逻辑**：Embedding Cache 表（`embedding_cache`）

**Schema**：

```sql
CREATE TABLE IF NOT EXISTS embedding_cache (
  hash TEXT NOT NULL,        -- 内容 hash (sha256)
  provider TEXT NOT NULL,    -- "openai" | "gemini" | "local"
  model TEXT NOT NULL,       -- "text-embedding-3-small" 等
  embedding BLOB NOT NULL,   -- Float32Array 序列化
  PRIMARY KEY (hash, provider, model)
);
```

**工作流程**：

1. **生成 Embedding 前**：查询 `embedding_cache` 表，检查 `(hash, provider, model)` 是否存在
2. **缓存命中**：直接返回缓存的 Embedding，跳过 API 调用
3. **缓存未命中**：调用 Embedding API，生成后写入 `embedding_cache` 表

**缓存 Key 计算**（在 `internal.ts` 行 146-148）：

```typescript
// 行 146-148: hashText 函数
export function hashText(value: string): string {
  return crypto.createHash("sha256").update(value).digest("hex");
}
```

**优点**：

- **节省 API 调用**：相同内容（如多个文件重复提到同一概念）只调用一次 Embedding API
- **跨文件去重**：不同文件的相同 chunk 共享 Embedding
- **提升索引速度**：缓存命中时无需等待 API 响应

**清理策略**（可选，配置 `cache.maxEntries`）：

- 如果缓存条目超过 `maxEntries`，自动清理最旧的条目（LRU 策略）
- 默认无限制

### 2.7 Memory Search Tool 实现

**核心文件**：`src/agents/tools/memory-search-tool.ts` (推测，实际代码未提供，但可从 `manager.ts` 的 `search` 方法推断)

**Tool 接口**：

```typescript
{
  name: "memory_search",
  description: "Search long-term memory for relevant context",
  parameters: {
    query: string,      // 用户查询
    maxResults?: number, // 最多返回多少结果（默认 10）
    minScore?: number,  // 最低分数阈值（默认 0.7）
  },
  returns: {
    results: Array<{
      path: string,
      startLine: number,
      endLine: number,
      score: number,
      snippet: string,
      source: "memory" | "sessions",
    }>
  }
}
```

**执行流程**：

1. **Agent 调用 Tool**：在 Agent Loop 中，Agent 决定需要搜索 Memory 时调用 `memory_search` Tool
2. **执行 Hybrid Search**：调用 `MemoryIndexManager.search(query, opts)`
3. **格式化结果**：返回给 Agent，Agent 根据结果生成回复
4. **触发同步（可选）**：如果配置了 `sync.onSearch`，在搜索前先执行一次 `sync()`

**Agent 使用示例**（推测的 Agent 输出）：

```xml
<thinking>
User asked about our last discussion on project deadlines. I need to search memory for relevant context.
</thinking>

<tool_use>
  <tool_name>memory_search</tool_name>
  <parameters>
    <query>project deadlines discussion</query>
    <maxResults>5</maxResults>
  </parameters>
</tool_use>
```

**Tool 返回示例**：

```json
{
  "results": [
    {
      "path": "memory/2026-01-28.md",
      "startLine": 45,
      "endLine": 67,
      "score": 0.89,
      "snippet": "## Project Deadlines\n\nWe discussed the Q1 deadlines:\n- Feature freeze: Feb 15\n- Code review: Feb 20\n- Release: Feb 28\n\nPeter mentioned he needs more time for testing...",
      "source": "memory"
    },
    {
      "path": "sessions/agent:main:main.jsonl",
      "startLine": 234,
      "endLine": 245,
      "score": 0.82,
      "snippet": "Peter: Can we push the deadline to March?\nAssistant: Let me check the project plan...",
      "source": "sessions"
    }
  ]
}
```

### 2.8 可借鉴的设计模式

1. **File Watcher 驱动的自我整理**：无需手动触发，文件变更自动索引
2. **Hybrid Search 策略**：Vector + FTS5 结合，提升召回率和准确率
3. **Embedding Cache Dedupe**：基于内容 hash 的缓存，节省 API 调用
4. **增量更新**：通过 hash 比对，只索引变更的 chunks
5. **Batch API 优化**：批量生成 Embedding，降低延迟和成本
6. **Multi-Provider Fallback**：OpenAI → Gemini → Local，提升可靠性
7. **Session Transcript 自动同步**：对话历史自动索引，形成"经验积累"
8. **Source 标记**：区分 `memory` 和 `sessions` 数据源，支持灵活过滤
9. **防抖合并**：连续文件变更合并为一次索引，避免频繁触发
10. **SQLite + sqlite-vec 一体化**：向量存储和关系数据在同一数据库，简化架构

---

## 第三部分：Skills 持续学习机制（AI 主动学习新能力）

### 3.1 核心概念

**Skills 系统是什么？**

OpenClaw 的 Skills 系统是一个可扩展的能力加载框架，让 AI 能够动态学习新工具的使用方法，而不需要修改核心代码。

**核心特性**：
- **三层加载机制**：Bundled → Managed → Workspace，支持覆盖和优先级
- **SKILL.md 格式**：Frontmatter + Instructions，清晰定义能力
- **自动刷新**：chokidar watcher 监控 Skills 目录变更，自动重新加载
- **Gating 机制**：基于 OS、环境变量、二进制依赖、配置项的加载条件
- **Plugin Skills**：Plugins 可以自带 Skills，动态加载
- **ClawHub Registry**：公开 Skills 注册表（类似 npm），支持搜索、安装、更新

### 3.2 Skills 三层加载机制

**核心文件**：`src/agents/skills/workspace.ts` (441 行)

#### 加载顺序和优先级

**核心函数**：`loadSkillEntries` (行 99-189)

**三层定义**：

1. **Bundled Skills**：打包在 OpenClaw 安装包中的内置 Skills
   - 路径：`skills/` 目录（相对于 npm package 或 OpenClaw.app 根目录）
   - Source: `"openclaw-bundled"`
   - 例如：`browser`、`peekaboo`、`sag` (ElevenLabs TTS)、`nano-banana-pro` (Gemini 图片生成) 等

2. **Managed Skills**：用户全局 Skills，所有 Agent 共享
   - 路径：`~/.openclaw/skills/`
   - Source: `"openclaw-managed"`
   - 用途：覆盖 Bundled Skills，或添加用户自定义 Skills

3. **Workspace Skills**：特定 Agent 的 Skills，只对该 Agent 可见
   - 路径：`<workspace>/skills/`
   - Source: `"openclaw-workspace"`
   - 用途：为不同 Agent 提供不同能力（如项目特定工具）

**优先级规则**（行 158-171）：

```typescript
// 行 158-171: 合并策略
const merged = new Map<string, Skill>();
// Precedence: extra < bundled < managed < workspace
for (const skill of extraSkills) {
  merged.set(skill.name, skill);
}
for (const skill of bundledSkills) {
  merged.set(skill.name, skill); // 覆盖 extraSkills
}
for (const skill of managedSkills) {
  merged.set(skill.name, skill); // 覆盖 bundledSkills
}
for (const skill of workspaceSkills) {
  merged.set(skill.name, skill); // 覆盖 managedSkills
}
```

**优点**：

- **按需覆盖**：用户可以在 `~/.openclaw/skills/` 中放置同名 Skill，覆盖 Bundled 版本
- **多 Agent 隔离**：不同 Agent 的 Workspace Skills 相互独立
- **全局共享**：Managed Skills 对所有 Agent 可见，避免重复配置

#### Extra Dirs 和 Plugin Skills

**Extra Dirs**（行 126-129）：

- 通过 `skills.load.extraDirs` 配置额外的 Skills 目录
- 优先级最低（低于 Bundled）

**Plugin Skills**（行 130-133）：

- 调用 `resolvePluginSkillDirs` 从 Plugin Manifest 中提取 Skills 路径
- Plugin 可以在 `openclaw.plugin.json` 中声明 `skills: ["path/to/skills"]`

```typescript
// 行 130-133
const pluginSkillDirs = resolvePluginSkillDirs({
  workspaceDir,
  config: opts?.config,
});
const mergedExtraDirs = [...extraDirs, ...pluginSkillDirs];
```

### 3.3 SKILL.md 格式和 Frontmatter

**核心文件**：`docs/tools/skills.md` (301 行)

#### SKILL.md 基本结构

```markdown
---
name: nano-banana-pro
description: Generate or edit images via Gemini 3 Pro Image
metadata: { "openclaw": { "requires": { "bins": ["uv"], "env": ["GEMINI_API_KEY"] }, "primaryEnv": "GEMINI_API_KEY" } }
---

# Instructions

Use the `uv` command to generate images:

```bash
uv run gemini-image --prompt "{prompt}" --output {baseDir}/output.png
```

The image will be saved to {baseDir}/output.png.
```

**Frontmatter 字段**（行 79-101）：

- **必填**：
  - `name`：Skill 名称（唯一标识）
  - `description`：功能描述
- **可选**：
  - `homepage`：官方网站（显示在 macOS Skills UI）
  - `user-invocable`：是否可作为 Slash Command 调用（默认 `true`）
  - `disable-model-invocation`：是否排除在 Model Prompt 之外（默认 `false`）
  - `command-dispatch`：Slash Command 是否直接调用 Tool（`tool`）
  - `command-tool`：Tool 名称（配合 `command-dispatch: tool`）
  - `command-arg-mode`：参数模式（默认 `raw`）
  - `metadata`：单行 JSON，包含 OpenClaw 特定的元数据

#### Metadata.openclaw 字段

**核心逻辑**：`src/agents/skills/frontmatter.ts` 中的 `resolveOpenClawMetadata` 函数

**字段定义**（行 109-136）：

```typescript
// 示例 metadata
{
  "openclaw": {
    "always": true,  // 总是包含该 Skill（跳过其他 gating 检查）
    "emoji": "♊️",  // macOS Skills UI 中显示的 emoji
    "homepage": "https://gemini-cli.com",  // 官方网站
    "os": ["darwin", "linux"],  // 支持的操作系统
    "requires": {
      "bins": ["gemini"],  // 必须存在的二进制（在 PATH 中）
      "anyBins": ["node", "bun"],  // 至少存在一个二进制
      "env": ["GEMINI_API_KEY"],  // 必须存在的环境变量（或在 config 中提供）
      "config": ["browser.enabled"]  // 必须在 openclaw.json 中为 truthy
    },
    "primaryEnv": "GEMINI_API_KEY",  // 关联的主要环境变量（与 skills.entries.<name>.apiKey 配合）
    "install": [  // 安装器规范（由 macOS Skills UI 使用）
      {
        "id": "brew",
        "kind": "brew",
        "formula": "gemini-cli",
        "bins": ["gemini"],
        "label": "Install Gemini CLI (brew)"
      }
    ]
  }
}
```

**Gating 机制**（行 109-146）：

- **OS 检查**：如果 `os` 字段存在，只在匹配的 OS 上加载
- **Bins 检查**：如果 `requires.bins` 存在，必须所有二进制都在 `PATH` 中
- **AnyBins 检查**：如果 `requires.anyBins` 存在，至少一个二进制在 `PATH` 中
- **Env 检查**：如果 `requires.env` 存在，环境变量必须存在或在 `skills.entries.<name>.env` 中提供
- **Config 检查**：如果 `requires.config` 存在，配置项必须在 `openclaw.json` 中为 truthy

**核心函数**：`shouldIncludeSkill` (在 `src/agents/skills/config.ts` 中)

### 3.4 Skills 自动刷新机制

**核心文件**：`src/agents/skills/refresh.ts` (176 行)

#### File Watcher 实现

**核心函数**：`ensureSkillsWatcher` (行 100-175)

**监控路径**（行 49-63）：

```typescript
// 行 49-63: resolveWatchPaths 函数
function resolveWatchPaths(workspaceDir: string, config?: OpenClawConfig): string[] {
  const paths: string[] = [];
  if (workspaceDir.trim()) {
    paths.push(path.join(workspaceDir, "skills"));  // Workspace Skills
  }
  paths.push(path.join(CONFIG_DIR, "skills"));  // Managed Skills (~/.openclaw/skills)
  const extraDirs = config?.skills?.load?.extraDirs ?? [];
  paths.push(...extraDirs.map(resolveUserPath));
  const pluginSkillDirs = resolvePluginSkillDirs({ workspaceDir, config });
  paths.push(...pluginSkillDirs);  // Plugin Skills
  return paths;
}
```

**Chokidar 配置**（行 137-146）：

```typescript
// 行 137-146
const watcher = chokidar.watch(watchPaths, {
  ignoreInitial: true,  // 不触发初始文件的事件
  awaitWriteFinish: {
    stabilityThreshold: debounceMs,  // 默认 250ms
    pollInterval: 100,
  },
  ignored: DEFAULT_SKILLS_WATCH_IGNORED,  // 忽略 .git, node_modules, dist
});
```

**触发机制**（行 167-172）：

```typescript
// 行 167-172
watcher.on("add", (p) => schedule(p));     // 新增文件
watcher.on("change", (p) => schedule(p));  // 文件变更
watcher.on("unlink", (p) => schedule(p));  // 删除文件
watcher.on("error", (err) => {
  log.warn(`skills watcher error (${workspaceDir}): ${String(err)}`);
});
```

**防抖和版本管理**（行 150-165）：

```typescript
// 行 150-165: schedule 函数
const schedule = (changedPath?: string) => {
  state.pendingPath = changedPath ?? state.pendingPath;
  if (state.timer) {
    clearTimeout(state.timer);
  }
  state.timer = setTimeout(() => {
    const pendingPath = state.pendingPath;
    state.pendingPath = undefined;
    state.timer = undefined;
    bumpSkillsSnapshotVersion({
      workspaceDir,
      reason: "watch",
      changedPath: pendingPath,
    });
  }, debounceMs);
};
```

**版本管理**（行 34-98）：

- **全局版本**：`globalVersion`，所有 Agent 共享（Managed Skills 变更时增加）
- **Workspace 版本**：`workspaceVersions.get(workspaceDir)`，特定 Agent 的版本（Workspace Skills 变更时增加）
- **最终版本**：`Math.max(globalVersion, workspaceVersion)`

**优点**：

- **热重载**：Skills 变更后，下一次 Agent Run 自动使用新 Skills
- **防抖**：避免频繁刷新（250ms 内的连续变更合并为一次）
- **版本号**：使用时间戳作为版本号，保证单调递增

#### Session Snapshot 机制

**核心概念**（来自 `docs/tools/skills.md` 行 241-244）：

> OpenClaw snapshots the eligible skills **when a session starts** and reuses that list for subsequent turns in the same session. Changes to skills or config take effect on the next new session.

**实现细节**：

1. **Session 开始时**：调用 `buildWorkspaceSkillSnapshot` 生成 Skills 快照
2. **快照包含**：
   - `prompt`：格式化后的 Skills XML（注入到 System Prompt）
   - `skills`：Eligible Skills 列表（name + primaryEnv）
   - `resolvedSkills`：完整的 Skill 对象
   - `version`：Snapshot 版本号

3. **后续 Agent Runs**：复用快照，避免重复扫描文件系统

4. **刷新触发**：
   - Skills Watcher 触发版本增加 → 下一个 Session 开始时重新生成快照
   - 配置热重载 → 立即刷新快照
   - Remote Node 上线 → 检查新 Eligible Skills

**优点**：

- **性能优化**：避免每次 Agent Run 都扫描文件系统
- **一致性**：同一 Session 中 Skills 列表保持不变
- **热重载支持**：版本号机制支持自动刷新

### 3.5 Agent 主动搜索和安装（推测）

> **注意**：相关文件 `skill-search-tool.ts` 和 `skill-install-tool.ts` 在代码库中未找到（可能路径不同或未实现），以下基于项目结构和文档推测。

**预期 Tool 接口**：

#### `skill_search` Tool

```typescript
{
  name: "skill_search",
  description: "Search ClawHub registry for skills",
  parameters: {
    query: string,      // 搜索关键词
    tags?: string[],    // 标签过滤
    maxResults?: number, // 最多返回多少结果
  },
  returns: {
    results: Array<{
      name: string,
      description: string,
      author: string,
      tags: string[],
      homepage: string,
      downloads: number,
    }>
  }
}
```

**实现推测**：

1. **调用 ClawHub API**：`https://clawhub.com/api/search?q={query}`
2. **本地 Fuzzy Match**：对已安装的 Skills 进行本地搜索
3. **合并结果**：ClawHub 结果 + 本地结果

#### `skill_install` Tool

```typescript
{
  name: "skill_install",
  description: "Install a skill from ClawHub",
  parameters: {
    skillSlug: string,  // Skill 的唯一标识（如 "nano-banana-pro"）
    location?: "workspace" | "managed", // 安装位置（默认 "workspace"）
  },
  returns: {
    success: boolean,
    installedPath: string,
    message: string,
  }
}
```

**实现推测**：

1. **下载 Skill**：从 ClawHub 下载 Skill 的 `.tar.gz` 包
2. **解压到目标目录**：
   - `location: "workspace"` → `<workspace>/skills/{skillName}/`
   - `location: "managed"` → `~/.openclaw/skills/{skillName}/`
3. **验证 SKILL.md**：检查 Frontmatter 格式是否正确
4. **触发刷新**：调用 `bumpSkillsSnapshotVersion` 更新版本号

**Agent 使用示例**（推测）：

```xml
<thinking>
User asked how to generate images. I should search for image generation skills.
</thinking>

<tool_use>
  <tool_name>skill_search</tool_name>
  <parameters>
    <query>image generation</query>
  </parameters>
</tool_use>

<!-- Tool 返回结果 -->

<thinking>
Found "nano-banana-pro" skill which supports Gemini 3 Pro Image generation. I'll install it.
</thinking>

<tool_use>
  <tool_name>skill_install</tool_name>
  <parameters>
    <skillSlug>nano-banana-pro</skillSlug>
    <location>workspace</location>
  </parameters>
</tool_use>

<!-- 安装成功后，下一次 Agent Run 就可以使用该 Skill -->
```

### 3.6 Plugin Skills 机制

**核心文件**：`src/agents/skills/plugin-skills.ts` (75 行)

#### Plugin Manifest 声明

**openclaw.plugin.json 示例**（推测）：

```json
{
  "id": "memory-pinecone",
  "kind": "memory",
  "skills": ["skills/pinecone"]
}
```

**加载逻辑**：`resolvePluginSkillDirs` (行 14-74)

**步骤**：

1. **加载 Plugin Manifest Registry**（行 22-25）：
   - 扫描 `extensions/*` 目录
   - 解析每个 Plugin 的 `openclaw.plugin.json`

2. **检查 Plugin 是否启用**（行 36-42）：
   - 调用 `resolveEnableState` 检查 `plugins.entries.<id>.enabled`
   - 调用 `resolveMemorySlotDecision` 处理 Memory Plugin 互斥（只能启用一个 Memory Plugin）

3. **收集 Skills 路径**（行 55-70）：
   - 从 `record.skills` 中读取相对路径
   - 解析为绝对路径：`path.resolve(record.rootDir, trimmed)`
   - 检查路径是否存在，去重后返回

**优点**：

- **插件自带 Skills**：Plugin 可以扩展 Agent 能力
- **动态加载**：Plugin 启用/禁用时，Skills 自动加载/卸载
- **隔离管理**：Plugin Skills 独立于 Bundled/Managed/Workspace Skills

### 3.7 ClawHub Registry（推测）

**ClawHub 是什么？**

ClawHub 是 OpenClaw 的公开 Skills 注册表（类似 npm、PyPI），用户可以浏览、搜索、安装、发布 Skills。

**预期功能**（基于文档 `docs/tools/skills.md` 行 50-67）：

#### 浏览和搜索

- **Web 界面**：https://clawhub.com
- **分类浏览**：按 Category、Tags、Popular、New 等维度浏览
- **搜索功能**：按名称、描述、作者搜索

#### 安装和更新

- **CLI 命令**（推测）：
  - `clawhub install <skill-slug>` - 安装 Skill 到当前目录或 Workspace
  - `clawhub update --all` - 更新所有已安装的 Skills
  - `clawhub sync --all` - 扫描本地 Skills 并发布更新

#### 发布和版本管理

- **发布流程**（推测）：
  1. 开发者在本地创建 `SKILL.md` 和相关文件
  2. 运行 `clawhub publish`，打包并上传到 ClawHub
  3. ClawHub 验证 SKILL.md 格式，生成唯一 slug
  4. 用户可以通过 `clawhub install <slug>` 安装

- **版本管理**（推测）：
  - 支持 Semantic Versioning (semver)
  - 安装时可以指定版本：`clawhub install <slug>@<version>`
  - 更新时检查新版本，提示用户

**API 设计**（推测）：

```typescript
// GET https://clawhub.com/api/search?q=image
{
  "results": [
    {
      "slug": "nano-banana-pro",
      "name": "nano-banana-pro",
      "description": "Generate or edit images via Gemini 3 Pro Image",
      "author": "openclaw",
      "version": "1.0.0",
      "tags": ["image", "generation", "gemini"],
      "homepage": "https://github.com/openclaw/nano-banana-pro",
      "downloads": 12345,
      "createdAt": "2025-12-01T00:00:00Z",
      "updatedAt": "2026-01-15T00:00:00Z"
    }
  ]
}

// GET https://clawhub.com/api/skills/<slug>
{
  "slug": "nano-banana-pro",
  "name": "nano-banana-pro",
  "description": "Generate or edit images via Gemini 3 Pro Image",
  "author": "openclaw",
  "version": "1.0.0",
  "tags": ["image", "generation", "gemini"],
  "homepage": "https://github.com/openclaw/nano-banana-pro",
  "readme": "# nano-banana-pro\n\n...",
  "downloadUrl": "https://clawhub.com/downloads/nano-banana-pro-1.0.0.tar.gz",
  "versions": ["1.0.0", "0.9.0", "0.8.0"]
}

// POST https://clawhub.com/api/skills (需要认证)
{
  "name": "my-skill",
  "description": "My custom skill",
  "version": "1.0.0",
  "files": {
    "SKILL.md": "...",
    "README.md": "..."
  }
}
```

### 3.8 可借鉴的设计模式

1. **三层 Skills 架构**：Bundled（系统级）→ Managed（用户级）→ Workspace（项目级），支持覆盖和优先级
2. **SKILL.md 格式标准化**：Frontmatter + Instructions，清晰定义能力和依赖
3. **Gating 机制**：基于 OS、二进制、环境变量、配置项的加载条件，避免加载无法使用的 Skills
4. **File Watcher 自动刷新**：Skills 变更后自动重新加载，无需手动重启
5. **Session Snapshot 优化**：Session 开始时生成快照，后续复用，平衡性能和一致性
6. **Plugin Skills 扩展**：插件可以自带 Skills，动态加载和卸载
7. **ClawHub Registry**：公开注册表 + CLI 工具，类似 npm 的生态系统
8. **Agent 主动安装**：通过 `skill_search` 和 `skill_install` Tools，Agent 可以自己学习新能力
9. **版本号机制**：使用时间戳作为版本号，触发热重载

---

## 第四部分：AI 主动性闭环分析

### 4.1 完整闭环流程

**场景示例**：AI 主动管理收件箱并生成邮件草稿

```
1. [Heartbeat 定时醒来]
   - 时间：每 30 分钟（或用户配置的间隔）
   - 触发：`runHeartbeatOnce({ reason: "interval" })`
   - 执行：读取 HEARTBEAT.md，其中包含"检查 Gmail 收件箱"

2. [读取 HEARTBEAT.md 指令]
   - 内容："- Scan Gmail inbox for urgent emails"
   - Agent 理解：需要检查 Gmail

3. [使用 Memory 搜索历史上下文]
   - Agent 调用：`memory_search({ query: "Gmail important contacts" })`
   - 返回：上次标记的 VIP 联系人列表（从 memory/contacts.md）

4. [使用 Skill 执行任务]
   - Agent 调用：`gmail.list({ query: "is:unread from:vip@company.com" })`
   - 返回：2 封未读邮件

5. [生成邮件草稿]
   - Agent 调用：`gmail.draft({ to: "vip@company.com", body: "..." })`
   - 保存草稿到 Gmail

6. [通过 Channel 发送给用户]
   - Agent 生成回复："发现 VIP 邮件，已生成草稿回复，请查看 Gmail"
   - 通过 WhatsApp/Telegram 发送给用户

7. [对话历史自动索引到 Memory]
   - Session Transcript 更新：记录本次对话
   - Memory Watcher 触发：5 秒后索引新对话到 SQLite
   - 结果：下次 Heartbeat 可以搜索到"VIP 邮件草稿"的历史记录

8. [用户反馈优化 HEARTBEAT.md]
   - 用户回复："以后只提醒紧急邮件，不要每封都通知"
   - Agent 修改 HEARTBEAT.md：添加"only urgent emails (marked with ⭐)"
   - 下次 Heartbeat：按新规则执行
```

### 4.2 协同效应分析

**Heartbeat + Memory**：

- **上下文感知决策**：Heartbeat 运行在 main session，拥有完整对话历史
- **历史经验召回**：通过 Memory Search 召回过去的决策和用户偏好
- **避免重复提醒**：Memory 记录上次 Heartbeat 内容，去重机制避免"唠叨"

**Heartbeat + Skills**：

- **动态能力扩展**：Heartbeat 可以触发 Skills 搜索和安装
- **批量检查**：一次 Heartbeat 可以使用多个 Skills（Gmail、Calendar、GitHub、Weather 等）
- **上下文组合**：Skills 提供工具，Heartbeat 提供调度，结合起来实现"主动执行任务"

**Memory + Skills**：

- **Skills 使用效果记录**：Memory 记录哪些 Skills 成功、哪些失败，Agent 学习优化
- **Skills 文档索引**：Skills 的 SKILL.md 可以索引到 Memory，Agent 可以搜索"如何使用某个 Skill"
- **Plugin 记忆**：Plugin Skills 的使用历史也进入 Memory，形成"插件学习曲线"

### 4.3 核心设计理念回顾

**主动感知**（Heartbeat）：

- **定时醒来**：不等用户呼唤，AI 自己"醒来"检查
- **智能静默**：只在有需要时打扰用户，避免噪音
- **时区感知**：尊重用户作息，不在夜间打扰

**自我学习**（Memory + Skills）：

- **Memory 积累经验**：对话历史和文档自动索引，形成"长期记忆"
- **Skills 扩展能力**：通过 ClawHub 安装新 Skills，Agent 学会新工具
- **Hybrid Search 智能召回**：语义 + 关键词结合，提升召回率

**持续进化**（闭环优化）：

- **Agent 自我优化 HEARTBEAT.md**：根据用户反馈调整检查策略
- **Memory 记录 Skills 效果**：哪些 Skills 有用、哪些无效，形成"工具使用经验"
- **版本号驱动刷新**：Skills 变更后自动重新加载，无需手动干预

### 4.4 与传统 AI 助手的对比

| 维度 | 传统 AI 助手 | OpenClaw AI 主动性 |
|------|-------------|-------------------|
| **触发方式** | 用户主动发起对话 | Heartbeat 定时醒来 + 用户对话 |
| **记忆机制** | Session Context（短期） | Memory 系统（长期） + Session Context |
| **能力扩展** | 开发者添加代码 | Skills 动态加载 + ClawHub 生态 |
| **学习方式** | 模型训练（固定） | Memory 积累 + Skills 安装 + Agent 自我优化 |
| **智能度** | 被动响应 | 主动感知 + 上下文决策 + 自我进化 |

**OpenClaw 的创新点**：

1. **AI "醒来"概念**：Heartbeat 让 AI 主动检查，而不是被动等待
2. **自我整理的记忆**：Memory Watcher 自动索引，无需手动管理
3. **持续学习能力**：Skills 系统 + ClawHub 生态，Agent 可以"学会"新工具
4. **闭环优化**：Agent 可以修改 HEARTBEAT.md，根据反馈优化行为

---

## 总结：OpenClaw AI 主动性的工程价值

### 设计亮点

1. **Workspace 文件系统作为"AI 大脑"**：
   - `HEARTBEAT.md`：Agent 行为清单
   - `memory/`：长期记忆存储
   - `skills/`：能力库
   - 所有配置文件化，用户可编辑，Agent 可自我修改

2. **File Watcher 驱动的自我整理**：
   - Memory Watcher：文件变更 → 自动索引
   - Skills Watcher：Skills 变更 → 自动刷新
   - 无需手动触发，自动保持最新

3. **Hybrid Search 策略**：
   - Vector（语义） + FTS5（关键词） = 高召回率 + 高准确率
   - 可调权重，适应不同场景

4. **三层 Skills 架构**：
   - Bundled → Managed → Workspace
   - 支持覆盖、优先级、隔离

5. **定时任务 + LLM 推理结合**：
   - Cron/Timer 只负责触发
   - 具体做什么由 LLM 读取 HEARTBEAT.md 决定
   - 灵活性 + 可编程性

### 可借鉴的实践

**对于 AI Agent 开发者**：

1. **实现 Heartbeat 机制**：让 AI 定时"醒来"，而不是被动等待用户
2. **智能静默**：`HEARTBEAT_OK` Token + `ackMaxChars` 阈值，避免无用通知
3. **File Watcher 自动化**：监控文件变更，自动触发索引/刷新
4. **Hybrid Search**：Vector + FTS5 结合，提升召回率
5. **三层配置架构**：System → User → Project，支持覆盖和隔离

**对于产品设计者**：

1. **主动性 = 智能的关键**：从"被动工具"到"主动助手"的核心区别
2. **用户反馈闭环**：让 AI 根据反馈自我优化（如修改 HEARTBEAT.md）
3. **生态系统思维**：Skills + ClawHub，类似 npm 的扩展生态
4. **可控的主动性**：Active Hours、智能静默、去重机制，避免打扰用户

### 未来演进方向（推测）

1. **更智能的 Heartbeat 调度**：
   - 基于用户活跃度动态调整间隔（忙碌时减少，空闲时增加）
   - 基于历史数据预测"什么时候用户最需要提醒"

2. **Memory 语义理解升级**：
   - 从 Chunk-level 到 Entity-level（识别人名、公司名、项目名等实体）
   - Knowledge Graph 构建（实体关系图谱）

3. **Skills 自动推荐**：
   - Agent 分析用户需求，主动推荐 ClawHub 上的 Skills
   - 基于使用频率和成功率，优化 Skills 加载优先级

4. **多 Agent 协作**：
   - Heartbeat 触发多个 Agent 协同工作
   - Memory 跨 Agent 共享（需要权限控制）

---

## 参考文件清单

### Heartbeat 相关

- `src/infra/heartbeat-runner.ts` (970 行) - Heartbeat 调度器核心实现
- `docs/automation/cron-vs-heartbeat.md` - Heartbeat vs Cron 决策指南
- `docs/gateway/heartbeat.md` - Heartbeat 配置文档
- `docs/reference/templates/HEARTBEAT.md` - HEARTBEAT.md 模板
- `docs/reference/templates/AGENTS.md` - Workspace AGENTS.md 模板（包含 Heartbeat 使用指南）

### Memory 相关

- `src/memory/manager.ts` (2397 行) - MemoryIndexManager 核心类
- `src/memory/sync-session-files.ts` (132 行) - Session Transcript 自动同步
- `src/memory/internal.ts` (278 行) - 分块、hash、文件列表等工具函数
- `src/memory/hybrid.ts` - Hybrid Search 合并算法
- `src/memory/manager-search.ts` - Vector 和 Keyword 搜索实现
- `docs/concepts/memory.md` (推测) - Memory 系统概念文档

### Skills 相关

- `src/agents/skills/workspace.ts` (441 行) - Skills 加载和快照生成
- `src/agents/skills/refresh.ts` (176 行) - Skills Watcher 和版本管理
- `src/agents/skills/plugin-skills.ts` (75 行) - Plugin Skills 解析
- `src/agents/skills/frontmatter.ts` (推测) - SKILL.md Frontmatter 解析
- `docs/tools/skills.md` (301 行) - Skills 系统完整文档
- `docs/start/hubs.md` - 文档导航（包含 ClawHub 链接）

---

**研究完成时间**: 2026-02-04 ✨
