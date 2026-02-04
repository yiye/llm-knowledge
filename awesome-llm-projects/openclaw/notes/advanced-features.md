# OpenClaw 第8层：高级特性深度研究

## 研究信息

- **研究日期**: 2026-02-04
- **Submodule Path**: `awesome-llm-projects/openclaw/openclaw`
- **研究层次**: 第8层 - 高级特性（Memory、Auto-reply、Media Understanding、ACP）
- **核心发现**: 这一层实现了 OpenClaw 的差异化特性，尤其是 Memory 自我整理和 Heartbeat 主动唤醒

---

## 1. Memory 系统深度剖析 🔥

### 1.1 架构设计

**核心管理器**: `MemoryIndexManager`（`src/memory/manager.ts:117-173`）

```typescript
export class MemoryIndexManager {
  private readonly cacheKey: string;
  private readonly cfg: OpenClawConfig;
  private readonly agentId: string;
  private readonly workspaceDir: string;
  private readonly settings: ResolvedMemorySearchConfig;
  private provider: EmbeddingProvider;
  private db: DatabaseSync;  // SQLite 数据库
  private readonly sources: Set<MemorySource>;  // "memory" | "sessions"
  private watcher: FSWatcher | null = null;  // Chokidar file watcher
  private sessionWatchTimer: NodeJS.Timeout | null = null;
  // ...
}
```

**关键特点**（基于代码分析）：

1. **单例模式** + **缓存**（`manager.ts:112-208`）：
   - 全局 `INDEX_CACHE` 缓存已创建的 Manager
   - Cache key 基于 `agentId + workspaceDir + settings`
   - 避免重复创建，节省资源

2. **两种数据源**（`manager.ts:48`）：
   ```typescript
   type MemorySource = "memory" | "sessions";
   ```
   - `memory`: Workspace 中的 `MEMORY.md` 和 `memory/*.md` 文件
   - `sessions`: Session Transcript（`~/.openclaw/agents/<agentId>/sessions/*.jsonl`）

3. **SQLite + sqlite-vec 存储**（`memory-schema.ts:1-97`）：
   - `meta` 表：存储索引元数据（model、provider、chunkTokens 等）
   - `files` 表：文件元信息（path、hash、mtime、size、source）
   - `chunks` 表：文档分块（id、path、start_line、end_line、text、embedding、model）
   - `chunks_vec` 表：Vector 索引（sqlite-vec 扩展，用于快速相似度搜索）
   - `chunks_fts` 表：FTS5 全文索引（用于关键词搜索）
   - `embedding_cache` 表：Embedding 缓存（provider、model、hash → embedding）

### 1.2 自动索引流程 🔥

**核心流程**（Memory 自我整理的实现）：

1. **File Watcher 监控**（`manager.ts:247`）：
   ```typescript
   private ensureWatcher(): void {
     if (this.watcher || this.closed || !this.sources.has("memory")) return;
     
     const memoryDir = path.join(this.workspaceDir, "memory");
     const memoryMd = path.join(this.workspaceDir, "MEMORY.md");
     const watchPaths = [memoryDir, memoryMd, ...normalizedExtraPaths];
     
     this.watcher = chokidar.watch(watchPaths, {
       ignoreInitial: true,
       // ...
     });
     
     this.watcher.on("change", () => { this.dirty = true; });
     this.watcher.on("add", () => { this.dirty = true; });
     this.watcher.on("unlink", () => { this.dirty = true; });
   }
   ```
   - 监控 `workspace/memory/` 目录和 `MEMORY.md` 文件
   - 文件变更时标记 `dirty = true`
   - **Debounce 策略**：不立即索引，等待稳定

2. **Session Transcript 自动同步**（`sync-session-files.ts:19-131`）：
   ```typescript
   export async function syncSessionFiles(params: {
     agentId: string;
     db: DatabaseSync;
     needsFullReindex: boolean;
     // ...
   }) {
     const files = await listSessionFilesForAgent(params.agentId);
     
     for (const absPath of files) {
       const entry = await buildSessionEntry(absPath);
       const record = db.prepare(`SELECT hash FROM files WHERE path = ? AND source = ?`)
         .get(entry.path, "sessions");
       
       if (!needsFullReindex && record?.hash === entry.hash) {
         continue;  // 跳过未变更的文件
       }
       
       await params.indexFile(entry);  // 重新索引
     }
   }
   ```
   - 扫描 `sessions/*.jsonl` 文件
   - **增量更新策略**：基于文件 hash 判断是否需要重新索引
   - 删除已不存在的文件索引

3. **Session 事件监听**（`manager.ts:248`）：
   ```typescript
   private ensureSessionListener(): void {
     if (this.sessionUnsubscribe || !this.sources.has("sessions")) return;
     
     this.sessionUnsubscribe = onSessionTranscriptUpdate((sessionKey, deltaBytes) => {
       if (sessionKey.startsWith(toAgentStoreSessionKey(this.agentId))) {
         // 标记 dirty，后续定时索引
         this.sessionsDirty = true;
       }
     });
   }
   ```
   - 监听 Session 对话更新事件
   - 新消息产生时自动触发索引

4. **定时索引任务**（`manager.ts:249-270`）：
   ```typescript
   private ensureIntervalSync(): void {
     if (this.intervalTimer) return;
     
     this.intervalTimer = setInterval(() => {
       if (this.dirty || this.sessionsDirty) {
         void this.sync();  // 异步触发索引
       }
     }, SESSION_DIRTY_DEBOUNCE_MS);  // 5000ms
   }
   ```
   - 每 5 秒检查一次 `dirty` 标志
   - 批量处理文件变更，避免频繁索引

### 1.3 Hybrid Search 实现 🔥

**Hybrid Search = Vector Search + Keyword Search**（`hybrid.ts:41-115`）

**Vector Search**（`manager-search.ts:20-94`）：

```typescript
export async function searchVector(params: {
  db: DatabaseSync;
  vectorTable: string;
  queryVec: number[];
  limit: number;
  // ...
}): Promise<SearchRowResult[]> {
  // 使用 sqlite-vec 的余弦距离搜索
  const rows = db.prepare(
    `SELECT c.id, c.path, c.start_line, c.end_line, c.text, c.source,
            vec_distance_cosine(v.embedding, ?) AS dist
       FROM ${vectorTable} v
       JOIN chunks c ON c.id = v.id
      WHERE c.model = ?
      ORDER BY dist ASC
      LIMIT ?`
  ).all(vectorToBlob(queryVec), providerModel, limit);
  
  return rows.map(row => ({
    id: row.id,
    path: row.path,
    startLine: row.start_line,
    endLine: row.end_line,
    score: 1 - row.dist,  // 距离转为相似度分数
    snippet: truncateUtf16Safe(row.text, snippetMaxChars),
    source: row.source,
  }));
}
```

**Keyword Search**（`manager-search.ts:136-187`）：

```typescript
export async function searchKeyword(params: {
  db: DatabaseSync;
  ftsTable: string;
  query: string;
  limit: number;
  buildFtsQuery: (raw: string) => string | null;
  bm25RankToScore: (rank: number) => number;
}): Promise<SearchRowResult[]> {
  const ftsQuery = buildFtsQuery(query);  // 构建 FTS5 查询
  
  const rows = db.prepare(
    `SELECT id, path, source, start_line, end_line, text,
            bm25(${ftsTable}) AS rank
       FROM ${ftsTable}
      WHERE ${ftsTable} MATCH ? AND model = ?
      ORDER BY rank ASC
      LIMIT ?`
  ).all(ftsQuery, providerModel, limit);
  
  return rows.map(row => ({
    // ...
    textScore: bm25RankToScore(row.rank),  // BM25 rank 转为分数
  }));
}
```

**结果合并策略**（`hybrid.ts:41-115`）：

```typescript
export function mergeHybridResults(params: {
  vector: HybridVectorResult[];
  keyword: HybridKeywordResult[];
  vectorWeight: number;  // 默认 0.7
  textWeight: number;    // 默认 0.3
}): Array<{...}> {
  const byId = new Map<string, {
    vectorScore: number;
    textScore: number;
  }>();
  
  // 1. 合并 vector 和 keyword 结果
  for (const r of params.vector) {
    byId.set(r.id, { vectorScore: r.vectorScore, textScore: 0 });
  }
  for (const r of params.keyword) {
    const existing = byId.get(r.id);
    if (existing) {
      existing.textScore = r.textScore;
    } else {
      byId.set(r.id, { vectorScore: 0, textScore: r.textScore });
    }
  }
  
  // 2. 加权计算最终分数
  const merged = Array.from(byId.values()).map(entry => ({
    // ...
    score: params.vectorWeight * entry.vectorScore + params.textWeight * entry.textScore,
  }));
  
  // 3. 按分数排序
  return merged.toSorted((a, b) => b.score - a.score);
}
```

**关键洞察**：
- Vector Search 捕捉**语义相似度**（同义词、概念关联）
- Keyword Search 捕捉**精确匹配**（专有名词、代码片段）
- 加权组合提升召回率和准确率

### 1.4 Embedding Providers 🔥

**多提供商支持**（`embeddings.ts:1-238`）：

```typescript
export type EmbeddingProvider = {
  id: string;
  model: string;
  embedQuery: (text: string) => Promise<number[]>;
  embedBatch: (texts: string[]) => Promise<number[][]>;
};

export async function createEmbeddingProvider(options: {
  provider: "openai" | "local" | "gemini" | "auto";
  fallback: "openai" | "gemini" | "local" | "none";
  // ...
}): Promise<EmbeddingProviderResult> {
  // Auto-select 策略
  if (options.provider === "auto") {
    if (canAutoSelectLocal(options)) {
      return await tryCreateProvider("local", options);
    }
    if (hasOpenAiKey) {
      return await tryCreateProvider("openai", options);
    }
    if (hasGeminiKey) {
      return await tryCreateProvider("gemini", options);
    }
  }
  
  // Failover 机制
  try {
    return await createProviderForType(options.provider, options);
  } catch (err) {
    if (options.fallback !== "none" && isMissingApiKeyError(err)) {
      return await createProviderForType(options.fallback, options);
    }
    throw err;
  }
}
```

**Batch API 优化**（`batch-openai.ts` / `batch-gemini.ts`）：

- **OpenAI Batch API**（`batch-openai.ts:1-200+`）：
  - 批量提交 embedding 请求
  - 轮询检查任务状态
  - 支持大规模索引场景

- **Gemini Batch API**（`batch-gemini.ts:1-150+`）：
  - 类似 OpenAI，但 API 不同
  - 自动处理 rate limiting

**Embedding Cache 去重**（`memory-schema.ts:39-48`）：

```typescript
CREATE TABLE embedding_cache (
  provider TEXT NOT NULL,
  model TEXT NOT NULL,
  provider_key TEXT NOT NULL,
  hash TEXT NOT NULL,         -- 文本内容的 hash
  embedding TEXT NOT NULL,
  dims INTEGER,
  updated_at INTEGER NOT NULL,
  PRIMARY KEY (provider, model, provider_key, hash)
);
```

- 相同文本（hash 相同）复用 embedding
- 跨文件去重，节省 API 调用

### 1.5 Memory Search Tool 🔥

**Tool 定义**（`src/agents/tools/memory-tool.ts:9-72`）：

```typescript
export function createMemorySearchTool(options: {
  config?: OpenClawConfig;
  agentSessionKey?: string;
}): AnyAgentTool | null {
  return {
    label: "Memory Search",
    name: "memory_search",
    description:
      "Mandatory recall step: semantically search MEMORY.md + memory/*.md " +
      "(and optional session transcripts) before answering questions about " +
      "prior work, decisions, dates, people, preferences, or todos; " +
      "returns top snippets with path + lines.",
    parameters: Type.Object({
      query: Type.String(),
      maxResults: Type.Optional(Type.Number()),
      minScore: Type.Optional(Type.Number()),
    }),
    execute: async (_toolCallId, params) => {
      const { manager } = await getMemorySearchManager({ cfg, agentId });
      const results = await manager.search(query, {
        maxResults,
        minScore,
        sessionKey: options.agentSessionKey,
      });
      return jsonResult({
        results,
        provider: status.provider,
        model: status.model,
      });
    },
  };
}
```

**Tool 注入 System Prompt**（据文档 `docs/concepts/memory.md:79-96`）：

- Agent 在回答问题前，会主动调用 `memory_search`
- System prompt 中包含"Mandatory recall step"指令
- 搜索结果自动注入到上下文中

**`memory_get` Tool**（`memory-tool.ts:74-119`）：

```typescript
export function createMemoryGetTool(...): AnyAgentTool | null {
  return {
    name: "memory_get",
    description:
      "Safe snippet read from MEMORY.md, memory/*.md with optional from/lines; " +
      "use after memory_search to pull only the needed lines and keep context small.",
    parameters: Type.Object({
      path: Type.String(),
      from: Type.Optional(Type.Number()),
      lines: Type.Optional(Type.Number()),
    }),
    execute: async (_toolCallId, params) => {
      const result = await manager.readFile({
        relPath,
        from: from ?? undefined,
        lines: lines ?? undefined,
      });
      return jsonResult(result);
    },
  };
}
```

- **两阶段检索**：`memory_search` 找到相关文件 → `memory_get` 精确读取片段
- 避免将整个文件加载到上下文，节省 token

---

## 2. Auto-reply 机制深度剖析

### 2.1 Heartbeat 系统 🔥

**Heartbeat Runner**（`src/infra/heartbeat-runner.ts:1-970`）：

```typescript
export type HeartbeatSummary = {
  enabled: boolean;
  every: string;        // "30m" / "1h" 等
  everyMs: number | null;
  prompt: string;
  target: string;       // "last" / "none" / <channel id>
  model?: string;
  ackMaxChars: number;  // HEARTBEAT_OK 阈值
};

// Heartbeat 执行流程（简化版）
async function runHeartbeat(params: {
  cfg: OpenClawConfig;
  agentId: string;
  deps: HeartbeatDeps;
}) {
  // 1. 检查 Active Hours（时区感知）
  if (!isWithinActiveHours(cfg, agentId)) {
    log.debug("Skipping heartbeat (outside active hours)");
    return;
  }
  
  // 2. 检查系统事件（如 async exec 完成）
  const systemEvents = peekSystemEvents(agentId);
  const prompt = systemEvents.length > 0
    ? EXEC_EVENT_PROMPT  // 覆盖默认 prompt
    : resolveHeartbeatPromptText(cfg, agentId);
  
  // 3. 执行 Agent run
  const reply = await getReplyFromConfig({
    ctx: { text: prompt, mode: "heartbeat" },
    cfg,
    sessionKey: mainSessionKey,
    // ...
  });
  
  // 4. 处理 HEARTBEAT_OK
  const { shouldSkip, text, didStrip } = stripHeartbeatToken(
    reply.text,
    { mode: "heartbeat" }
  );
  
  if (shouldSkip) {
    log.debug("Heartbeat returned HEARTBEAT_OK, not delivering");
    return;
  }
  
  // 5. 发送消息到目标 Channel
  await deliverOutboundPayloads({
    payloads: [{ text, channelId, to }],
    deps,
  });
}
```

**HEARTBEAT_OK 智能静默机制**（`src/auto-reply/heartbeat.test.ts:1-184`）：

```typescript
export function stripHeartbeatToken(
  raw: string | undefined,
  options: { mode: "heartbeat" | "message" }
): {
  shouldSkip: boolean;
  text: string;
  didStrip: boolean;
} {
  // 1. 移除 HEARTBEAT_OK token（开头或结尾）
  const stripped = raw?.replace(/^\s*HEARTBEAT_OK\s*|\s*HEARTBEAT_OK\s*$/gi, "").trim();
  
  // 2. 判断剩余内容是否"空"
  if (options.mode === "heartbeat") {
    if (!stripped || stripped.length <= ackMaxChars) {
      return { shouldSkip: true, text: "", didStrip: true };
    }
  }
  
  return { shouldSkip: false, text: stripped, didStrip: true };
}
```

**关键设计**：
- **Token 位置敏感**：只移除开头/结尾的 `HEARTBEAT_OK`，中间的不动
- **阈值判断**：剩余内容 ≤ 300 字符视为"无重要信息"
- **避免误判**：长消息（如"发现3个重要邮件 HEARTBEAT_OK"）会保留

**Active Hours 时区感知**（`heartbeat-runner.ts:99-160`）：

```typescript
function isWithinActiveHours(
  cfg: OpenClawConfig,
  agentId: string
): boolean {
  const activeHours = resolveActiveHours(cfg, agentId);
  if (!activeHours) return true;  // 未配置则始终执行
  
  const timezone = resolveActiveHoursTimezone(cfg, activeHours.timezone);
  const now = DateTime.now().setZone(timezone);
  const [startHour, startMin] = parseTime(activeHours.start);
  const [endHour, endMin] = parseTime(activeHours.end);
  
  const startTime = now.set({ hour: startHour, minute: startMin });
  const endTime = now.set({ hour: endHour, minute: endMin });
  
  return now >= startTime && now < endTime;
}
```

- 用户配置 `activeHours: { start: "08:00", end: "24:00" }`
- 自动读取用户时区（`resolveUserTimezone`）
- 避免夜间打扰用户

### 2.2 消息分发机制

**Dispatch Pipeline**（`src/auto-reply/dispatch.ts:1-78`）：

```typescript
export async function dispatchInboundMessage(params: {
  ctx: MsgContext | FinalizedMsgContext;
  cfg: OpenClawConfig;
  dispatcher: ReplyDispatcher;
  replyOptions?: Omit<GetReplyOptions, "onToolResult" | "onBlockReply">;
}): Promise<DispatchInboundResult> {
  // 1. Finalize context
  const finalized = finalizeInboundContext(params.ctx);
  
  // 2. Dispatch reply
  return await dispatchReplyFromConfig({
    ctx: finalized,
    cfg: params.cfg,
    dispatcher: params.dispatcher,
    replyOptions: params.replyOptions,
  });
}
```

**Reply Dispatcher**（`src/auto-reply/reply/reply-dispatcher.ts`）：

- 处理消息路由（发送到哪个 Channel）
- 支持 typing indicators（打字中效果）
- Buffering 策略（批量发送）

---

## 3. Media Understanding 系统剖析

### 3.1 多模态支持

**Provider Registry**（`src/media-understanding/runner.ts:50-77`）：

```typescript
const DEFAULT_IMAGE_MODELS: Record<string, string> = {
  openai: "gpt-5-mini",
  anthropic: "claude-opus-4-5",
  google: "gemini-3-flash-preview",
  minimax: "MiniMax-VL-01",
};

const DEFAULT_AUDIO_MODELS = {
  openai: "whisper-1",
  groq: "whisper-large-v3-turbo",
  deepgram: "nova-2-general",
  google: "gemini-2-flash-preview",
};

export function buildProviderRegistry(
  overrides?: Record<string, MediaUnderstandingProvider>
): ProviderRegistry {
  return buildMediaUnderstandingRegistry(overrides);
}
```

**Auto-select 策略**（`runner.ts:51-59`）：

```typescript
const AUTO_AUDIO_KEY_PROVIDERS = ["openai", "groq", "deepgram", "google"];
const AUTO_IMAGE_KEY_PROVIDERS = ["openai", "anthropic", "google", "minimax"];
const AUTO_VIDEO_KEY_PROVIDERS = ["google"];

// Auto-select 流程：
// 1. 检查可用的 API key
// 2. 按优先级选择 provider
// 3. Fallback 到下一个 provider
```

### 3.2 媒体处理 Pipeline

**Attachment 处理**（`src/media-understanding/attachments.ts`）：

- 图片：支持 JPEG、PNG、GIF、WebP
- 音频：支持 MP3、WAV、M4A、OGG
- 视频：支持 MP4、WebM、MOV

**Concurrency Control**（`src/media-understanding/concurrency.ts`）：

- 并发限制（避免同时处理太多媒体）
- 队列管理

---

## 4. ACP 协议（Agent Client Protocol）

### 4.1 协议设计

**ACP Server**（`src/acp/server.ts:1-145`）：

```typescript
export function serveAcpGateway(opts: AcpServerOptions = {}): void {
  const cfg = loadConfig();
  const connection = buildGatewayConnectionDetails({ config: cfg });
  
  // 1. 创建 Gateway Client
  const gateway = new GatewayClient({
    url: connection.url,
    token: opts.gatewayToken,
    password: opts.gatewayPassword,
    clientName: GATEWAY_CLIENT_NAMES.CLI,
    mode: GATEWAY_CLIENT_MODES.CLI,
    onEvent: (evt) => { agent?.handleGatewayEvent(evt); },
  });
  
  // 2. 创建 ACP Connection（NDJSON over stdin/stdout）
  const stream = ndJsonStream(
    Writable.toWeb(process.stdout),
    Readable.toWeb(process.stdin)
  );
  
  // 3. 启动 ACP Agent
  new AgentSideConnection((conn) => {
    agent = new AcpGatewayAgent(conn, gateway, opts);
    agent.start();
    return agent;
  }, stream);
  
  gateway.start();
}
```

**ACP Session 管理**（`src/acp/types.ts:4-11`）：

```typescript
export type AcpSession = {
  sessionId: SessionId;
  sessionKey: string;
  cwd: string;
  createdAt: number;
  abortController: AbortController | null;
  activeRunId: string | null;
};
```

**关键特点**：
- **Gateway-backed**：ACP Server 作为 Gateway 的客户端
- **NDJSON Protocol**：基于换行分隔的 JSON 流
- **Session 映射**：ACP Session ID ↔ OpenClaw Session Key

---

## 5. 与 AI 主动性的关联 🔥

### 5.1 Memory 与主动性

**自动索引 = 主动整理记忆**：

1. **File Watcher**（`manager.ts:247`）：
   - Agent 无需手动触发索引
   - 文件变更自动触发
   - 类似"人类睡觉时大脑整理记忆"

2. **Session Transcript Auto-Sync**（`sync-session-files.ts`）：
   - 对话结束后自动索引到 Memory
   - Agent 下次可以搜索到历史对话
   - 实现"长期记忆"

3. **Hybrid Search**（`hybrid.ts`）：
   - Agent 可以主动调用 `memory_search`
   - 无需用户明确指示
   - System prompt 中的"Mandatory recall step"

### 5.2 Heartbeat 与主动性

**定时唤醒 = AI 主动察觉**：

1. **定时触发**（`heartbeat-runner.ts`）：
   - 不等用户呼唤，AI 自己"醒来"
   - 检查待办事项、邮件、日历等

2. **HEARTBEAT_OK 机制**：
   - "一切正常"时不打扰用户
   - 只有重要信息才发送
   - 类似"人类只在需要时说话"

3. **System Event 驱动**：
   - Async exec 完成自动通知
   - 不需要用户轮询

### 5.3 Memory + Heartbeat 协同

**完整闭环**（概念模型）：

```
1. Heartbeat 定时醒来 → 检查 Gmail 收件箱
2. 发现新邮件 → 使用 memory_search 搜索相关历史对话
3. Memory 返回上下文 → Agent 理解背景
4. Agent 生成回复草稿 → 通过 Channel 发送给用户
5. 对话历史自动索引到 Memory → 下次 Heartbeat 更智能
6. Agent 根据用户反馈优化 HEARTBEAT.md → 闭环完成
```

---

## 6. 可借鉴的设计模式

### 6.1 Memory 系统设计

1. **File Watcher + Debounce**：
   - 监控文件变更
   - 批量处理（5秒 debounce）
   - 避免频繁索引

2. **Incremental Indexing**：
   - 基于文件 hash 判断是否需要重新索引
   - 只处理变更的文件
   - 提升性能

3. **Hybrid Search**：
   - Vector Search（语义相似度）+ Keyword Search（精确匹配）
   - 加权组合（默认 0.7 : 0.3）
   - 提升召回率和准确率

4. **Embedding Cache**：
   - 相同文本复用 embedding
   - 跨文件去重
   - 节省 API 调用

5. **SQLite + sqlite-vec**：
   - 单文件数据库（易于备份和迁移）
   - sqlite-vec 提供高效的向量搜索
   - FTS5 提供全文检索

### 6.2 Heartbeat 系统设计

1. **定时任务 + LLM 推理**：
   - Cron 只负责触发
   - 具体做什么由 LLM 决定
   - 灵活性高

2. **HEARTBEAT_OK 智能静默**：
   - 避免"一切正常"时打扰用户
   - Token 位置敏感
   - 阈值判断

3. **Active Hours 时区感知**：
   - 自动读取用户时区
   - 避免夜间打扰
   - 用户体验友好

4. **System Event 驱动**：
   - Async exec 完成自动通知
   - 覆盖默认 prompt
   - 处理紧急事件

### 6.3 架构设计

1. **单例 + 缓存**：
   - `MemoryIndexManager` 全局缓存
   - 避免重复创建
   - 节省资源

2. **Plugin 化**：
   - Media Understanding Provider Registry
   - ACP Protocol
   - 易于扩展

3. **两阶段检索**：
   - `memory_search` → `memory_get`
   - 先找到相关文件，再精确读取片段
   - 节省 token

---

## 7. 总结

### 核心洞察

1. **Memory 系统 = AI 的长期记忆**：
   - File Watcher 实现"自我整理"
   - Session Transcript 实现"对话记忆"
   - Hybrid Search 实现"智能回忆"

2. **Heartbeat = AI 的主动性**：
   - 定时唤醒，不等用户呼唤
   - HEARTBEAT_OK 机制避免打扰
   - Active Hours 时区感知

3. **Memory + Heartbeat = AI 主动性闭环**：
   - Heartbeat 定时醒来 → Memory Search 获取上下文 → Agent 生成回复 → 对话历史索引到 Memory → 下次更智能

### 关键数据

- **Memory 索引 Debounce**: 5 秒（`SESSION_DIRTY_DEBOUNCE_MS = 5000`）
- **Embedding Batch 最大 Token**: 8000（`EMBEDDING_BATCH_MAX_TOKENS = 8000`）
- **Heartbeat 默认间隔**: 30 分钟（`DEFAULT_HEARTBEAT_EVERY = "30m"`）
- **HEARTBEAT_OK 阈值**: 300 字符（`DEFAULT_HEARTBEAT_ACK_MAX_CHARS = 300`）
- **Vector Search Timeout**: 60 秒（Remote），5 分钟（Local）
- **Hybrid Search 权重**: Vector 0.7, Text 0.3（默认）

### 实现细节

- **Memory DB**: SQLite + sqlite-vec + FTS5
- **File Watcher**: Chokidar
- **Embedding Providers**: OpenAI, Gemini, Local (node-llama-cpp)
- **Protocol**: ACP (NDJSON over stdin/stdout)

---

**研究完成日期**: 2026-02-04  
**下一步**: 第9层 - 工程实践（代码组织与开发体验）
