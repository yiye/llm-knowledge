# OpenClaw 第8层：高级特性深度研究

## 研究信息

- **研究日期**: 2026-03-06
- **Submodule Path**: `awesome-llm-projects/openclaw/openclaw`
- **研究版本**: 2026.3.3（commit 73055728318df378c831950cd01fb7c875a33790）
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

3. **Session 事件监听**（`manager-sync-ops.ts:407-420`）：
   ```typescript
   protected ensureSessionListener() {
     if (!this.sources.has("sessions") || this.sessionUnsubscribe) return;
     
     this.sessionUnsubscribe = onSessionTranscriptUpdate((update) => {
       const sessionFile = update.sessionFile;
       if (!this.isSessionFileForAgent(sessionFile)) return;
       this.scheduleSessionDirty(sessionFile);
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

// ⭐ 2026.3.22 更新：支持 6 种 provider（原来只有 3 种）
export async function createEmbeddingProvider(options: {
  provider: "openai" | "local" | "gemini" | "voyage" | "mistral" | "ollama" | "auto";
  fallback: "openai" | "gemini" | "local" | "voyage" | "mistral" | "ollama" | "none";
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

### 1.4.1 Ollama Embedding 支持（2026.3.3 新增）🔥

**配置**（`src/config/types.tools.ts:327,346`）：

- `memorySearch.provider = "ollama"`：使用本地 Ollama 实例做 embedding
- `memorySearch.fallback = "ollama"`：主 provider 失败时回退到 Ollama

**实现**（`src/memory/embeddings-ollama.ts:1-136`）：

```typescript
export async function createOllamaEmbeddingProvider(
  options: EmbeddingProviderOptions,
): Promise<{ provider: EmbeddingProvider; client: OllamaEmbeddingClient }> {
  const client = resolveOllamaEmbeddingClient(options);
  const embedUrl = `${client.baseUrl.replace(/\/$/, "")}/api/embeddings`;
  // ...
}
```

**`models.providers.ollama` 配置复用**（`embeddings-ollama.ts:47-83`）：

- `baseUrl`：从 `options.config.models?.providers?.ollama?.baseUrl` 或 `options.remote?.baseUrl` 解析，默认 `http://127.0.0.1:11434`
- `apiKey`：从 `models.providers.ollama.apiKey`、`remote.apiKey` 或 `OLLAMA_API_KEY` 环境变量解析
- `headers`：支持 `providerConfig?.headers` 覆盖
- URL 中的 `/v1` 会被自动剥离，确保指向 Ollama 原生 API 根路径

**默认模型**（`embeddings-ollama.ts:17`）：`nomic-embed-text`

### 1.4.2 Voyage / Mistral Embedding 支持（⭐ 新增）

**Voyage AI**（`src/memory/embeddings-voyage.ts`）：

- `memorySearch.provider = "voyage"` 使用 Voyage AI 的 embedding API
- 支持批处理：`batch-voyage.ts` 实现 batch embedding 提交
- 默认模型：`voyage-3-large`

**Mistral AI**（`src/memory/embeddings-mistral.ts`）：

- `memorySearch.provider = "mistral"` 使用 Mistral embedding API
- 支持批处理：`batch-mistral.ts`（与 voyage/openai/gemini 并列）
- 默认模型：`mistral-embed`

**完整 Embedding Provider 矩阵**：

| Provider | 模块 | 批处理 | 需要 API Key |
|----------|------|--------|------------|
| `openai` | `embeddings-openai.ts` | `batch-openai.ts` | OPENAI_API_KEY |
| `gemini` | `embeddings-gemini.ts` | `batch-gemini.ts` | GEMINI_API_KEY |
| `local` | `node-llama.ts` | — | 无（本地模型） |
| `ollama` | `embeddings-ollama.ts` | — | 可选 |
| `voyage` | `embeddings-voyage.ts` | `batch-voyage.ts` | VOYAGE_API_KEY |
| `mistral` | `embeddings-mistral.ts` | `batch-mistral.ts` | MISTRAL_API_KEY |

### 1.4.3 QMD 外部 Memory 后端（⭐ 全新功能）

**QMD（Quick Memory Database）** 是一个可选的外部 Memory CLI 后端，通过 `search-manager.ts` 接入：

```typescript
// src/memory/search-manager.ts
export async function getMemorySearchManager(
  config: OpenClawConfig,
  agentId: string,
  workspaceDir: string,
): Promise<MemorySearchManager> {
  if (config.memorySearch?.backend === "qmd") {
    const qmd = new QmdMemoryManager(config, agentId, workspaceDir);
    // 如果 QMD 不可用，自动降级到内置 SQLite 后端
    return new FallbackMemoryManager(qmd, builtinManager);
  }
  return MemoryIndexManager.get(config, agentId, workspaceDir);
}
```

**`QmdMemoryManager`**（`src/memory/qmd-manager.ts`）：
- 通过调用外部 `qmd` CLI 进程执行 memory 操作
- 进程通信：stdin/stdout JSON 协议
- 支持 `search`、`add`、`delete`、`list` 操作

**`FallbackMemoryManager`**：
- 主 backend（QMD）失败时自动降级到 `MemoryIndexManager`（内置 SQLite）
- 透明降级：对 Agent 层无感知

**使用场景**：需要共享 memory 数据库、或使用自定义 memory 存储格式时。

### 1.4.4 Hybrid Search 增强：MMR + 时间衰减（⭐ 新增）

**`src/memory/hybrid.ts`** 实现了增强版混合搜索：

**MMR（Maximum Marginal Relevance）**：
- 避免搜索结果冗余：不只选相关度最高的，而是在相关性和结果多样性之间取平衡
- 算法：迭代选择与 query 相关但与已选结果不重复的 chunk
- 参数：`mmrLambda`（相关性权重，0=纯多样性，1=纯相关性，默认 0.7）

**Temporal Decay（时间衰减）**：
- 近期 chunk 权重更高，旧 chunk 权重递减
- 基于 `chunks.updated_at` 时间戳计算衰减系数
- 参数：`temporalDecayDays`（衰减半衰期，默认 30 天）

**完整 hybrid 搜索流程**：
```
1. Vector Search（sqlite-vec cosine） → 候选集
2. Keyword Search（FTS5 BM25）      → 候选集
3. Score Merge（加权融合）           → 合并候选
4. Temporal Decay（时间加权）        → 调整分数
5. MMR Re-rank（多样性过滤）         → 最终结果
```

### 1.4.5 Local Embedding 初始化加固（回归覆盖）🔥

**瞬态初始化重试**（`src/memory/embeddings.ts:114-141`）：

- `ensureContext` 在初始化失败时将 `initPromise = null`，允许后续请求重试
- 测试覆盖：`embeddings.test.ts:541-594` — `"retries initialization after a transient ensureContext failure"`
- 首次 `embedBatch` 失败后，第二次调用会重新执行 `getLlama` 并成功

**embedQuery + embedBatch 并发启动共享**（`embeddings.test.ts:595-650`）：

- 测试：`"shares initialization when embedQuery and embedBatch start concurrently"`
- `embedQuery` 与 `embedBatch` 同时调用时，共享同一个 `initPromise`，只加载一次模型
- `getLlamaSpy`、`loadModelSpy`、`createContextSpy` 各调用 1 次

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

### 2.2 Hooks 生命周期事件（2026.3.3 新增）🔥

**内部 Hook 事件**（`src/config/types.hooks.ts:76`、`src/hooks/internal-hooks.ts`）：

- **`message:transcribed`**：音频转录完成后触发，`context` 含 `transcript`、`bodyForAgent`、`channelId` 等（`internal-hooks.ts:99-138`）
- **`message:preprocessed`**：媒体理解、链接摘要等全部完成后触发，`context` 含 `bodyForAgent`（完整富文本）、`transcript`（若有音频）（`internal-hooks.ts:140-184`）

**`message:sent` 扩展上下文**（`src/hooks/internal-hooks.ts:70-96`）：

```typescript
export type MessageSentHookContext = {
  to: string;
  content: string;
  success: boolean;
  channelId: string;
  isGroup?: boolean;   // 是否群组/频道上下文
  groupId?: string;    // 群组/频道 ID，便于与 message:received 关联
  // ...
};
```

**`session_start` / `session_end` 中的 `sessionKey`**（`src/plugins/types.ts:573-585`）：

```typescript
export type PluginHookSessionStartEvent = {
  sessionId: string;
  sessionKey?: string;
  resumedFrom?: string;
};

export type PluginHookSessionEndEvent = {
  sessionId: string;
  sessionKey?: string;
  messageCount: number;
  durationMs?: number;
};
```

- Hook 上下文 `PluginHookSessionContext`（`types.ts:566-570`）同样包含 `sessionKey`
- 文档：`docs/automation/hooks.md:262-325`

### 2.2.1 压缩 Hooks（⭐ 新增）

**插件压缩钩子**（`src/plugins/hooks.ts`）：

- **`before_compaction`**：会话上下文压缩前触发，插件可读取完整历史
- **`after_compaction`**：压缩完成后触发，插件可查看压缩摘要

**内部压缩钩子**（`src/hooks/internal-hooks.ts`）：

- **`session:compact:before`**：内部压缩前事件（与插件钩子并行的独立路径）
- **`session:compact:after`**：内部压缩后事件

**两路触发逻辑**（`src/agents/pi-embedded-runner/compact.ts`）：

```
压缩触发
├── triggerInternalHook("session:compact:before")  — 内部订阅者
├── hookRunner.runBeforeCompaction()               — 插件 before_compaction
│
│   [执行压缩]
│
├── triggerInternalHook("session:compact:after")   — 内部订阅者
└── hookRunner.runAfterCompaction()                — 插件 after_compaction
```

> **注意**：当 context engine 控制压缩时（engine-owned compaction），会从 `compactEmbeddedPiSession` 触发插件钩子，传入 `messageCount: -1` 作为标识。

### 2.2.2 子 Agent 生命周期 Hooks（⭐ 新增）

**`subagent_spawning`**（spawn 前）：
- 触发时机：`sessions_spawn` 工具调用且 `thread: true` 时必须有此 hook
- 用途：channel plugin 创建线程绑定（Slack thread、Discord thread 等）
- 若没有注册此 hook，`thread: true` 会直接返回错误

**`subagent_spawned`**（spawn 成功后）：
- 触发时机：子 Agent 会话成功创建并开始运行后
- 用途：通知 channel plugin 子任务开始

**`subagent_ended`**（子 Agent 结束后）：
- 触发时机：子 Agent 运行完成（成功或失败）
- 用途：清理线程绑定、通知用户结果

**实现位置**：`src/agents/subagent-spawn.ts` 中的 `ensureThreadBindingForSubagentSpawn`、`runSubagentSpawning`、`runSubagentSpawned`、`runSubagentEnded`。

### 2.3 消息分发机制

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

### 3.2 音频 Echo Transcript（2026.3.3 新增）🔥

**配置**（`src/config/zod-schema.core.ts:684-685`、`src/config/types.tools.ts:99-104`）：

- `tools.media.audio.echoTranscript`（boolean，默认 `false`）：是否在 Agent 处理前将转录文本回显到聊天
- `tools.media.audio.echoFormat`（string，可选）：回显格式，支持 `{transcript}` 占位符

**实现**（`src/media-understanding/apply.ts:532-541`）：

```typescript
const audioCfg = cfg.tools?.media?.audio;
if (audioCfg?.echoTranscript && transcript) {
  await sendTranscriptEcho({
    ctx,
    cfg,
    transcript,
    format: audioCfg.echoFormat ?? DEFAULT_ECHO_TRANSCRIPT_FORMAT,
  });
}
```

**默认格式**（`src/media-understanding/echo-transcript.ts:14`）：`'📝 "{transcript}"'`

**文档**：`docs/nodes/audio.md:120-143`、`docs/nodes/media-understanding.md:43,61-62`

### 3.3 媒体处理 Pipeline

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

### 4.2 ACP 改进（2026.3.3）🔥

**持久化 Channel Bindings**（`src/acp/persistent-bindings.*`）：

- `persistent-bindings.lifecycle.ts`：`ensureConfiguredAcpBindingSession` 确保配置的 binding 对应 session 存在
- `persistent-bindings.resolve.ts`：支持 `parentConversationId` 继承父频道 binding（Discord thread → 父 channel）
- `persistent-bindings.route.ts`：路由 API 暴露 `parentConversationId`
- Discord thread 场景：当 `conversationId` 为 thread 时，可回退到 `parentConversationId` 的 channel binding

**Session Bootstrap 重试**（CHANGELOG #28786, #31338, #34055）：

- 当 `sessions ensure` 返回空 session 标识时，自动用 `sessions new` 重试
- 避免 ACP spawn 出现 `NO_SESSION` / `ACP_TURN_FAILED` 失败

**`streamTo: "parent"`**（`src/agents/acp-spawn.ts:246-251,430-485`、`sessions-spawn-tool.ts:37,101,123-126`）：

- `sessions_spawn` 工具参数：`streamTo: "parent"` 仅当 `runtime: "acp"` 时有效
- 将子 run 的进度摘要转发到 requester session 作为 system events，而非直接投递到子 session
- 需要活跃的 requester session 上下文，否则报错：`streamTo="parent" requires an active requester session context`
- 可生成 session-scoped relay log（`<sessionId>.acp-stream.jsonl`），通过 `streamLogPath` 返回
- 文档：`docs/tools/acp-agents.md:243`、`docs/tools/index.md:475-500`

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

## 7. Plugin Runtime 新 API（2026.3.3）🔥

**`runtime.system.requestHeartbeatNow`**（`src/plugins/runtime/runtime-system.ts:1-15`、`src/infra/heartbeat-wake.ts`）：

- 插件可立即唤醒指定 session 的 Heartbeat，常用于 `enqueueSystemEvent` 之后
- 签名：`requestHeartbeatNow(opts?: { reason?: string; agentId?: string; sessionKey?: string; coalesceMs?: number })`
- 典型用法：异步 exec 完成后调用，触发 Agent 处理 system event

**`runtime.events.onAgentEvent`**（`src/plugins/runtime/runtime-events.ts:1-10`、`src/infra/agent-events.ts:79-84`）：

- 订阅 Agent 运行事件流
- 回调参数：`AgentEventPayload`（含 `runId`、`stream`、`data`、`sessionKey` 等）
- `stream` 类型：`"lifecycle" | "tool" | "assistant" | "error"`

**`runtime.events.onSessionTranscriptUpdate`**（`src/sessions/transcript-events.ts:9-14`）：

- 订阅 session transcript 更新
- 回调参数：`{ sessionFile: string }`（session JSONL 文件路径）
- 返回 unsubscribe 函数
- Memory 模块用此监听 session 变更并触发增量索引（`manager-sync-ops.ts:411-420`）

**类型定义**：`src/plugins/runtime/types-core.ts:40-43`

---

## 8. 总结

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

- **Memory DB**: SQLite + sqlite-vec + FTS5（内置），或 QMD 外部 CLI（可选）
- **File Watcher**: Chokidar
- **Embedding Providers**: OpenAI, Gemini, Local (node-llama-cpp), Ollama, Voyage, Mistral（6 种）
- **Hybrid Search**: Vector + BM25 + MMR 多样性 + Temporal Decay 时间衰减
- **Protocol**: ACP (NDJSON over stdin/stdout)

---

**研究完成日期**: 2026-03-06  
**下一步**: 第9层 - 工程实践（代码组织与开发体验）
