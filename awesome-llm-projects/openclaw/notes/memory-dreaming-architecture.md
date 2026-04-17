# OpenClaw Memory / Dreaming 子系统架构

> **数据来源（前半部分）**：[OpenClaw Releases 2026.4.5 - 2026.4.15](https://github.com/openclaw/openclaw/releases?page=1)
> **数据来源（附录 A 代码验证）**：本地 submodule @ tag `v2026.4.15` (commit `041266a669`)
> **分析时间**：2026-04-17
> **阅读顺序**：先看概念（§1-§9），再读附录 A 查具体数值/位置

---

## 1. 设计目标

让 Agent 具备**可积累、可遗忘、可离线整理**的记忆能力，而不是每次对话从零开始。

关键需求：
- 用户不需要说 "remember this" / "search memory" 就能享受记忆（→ Active Memory）
- 历史对话可以沉淀成结构化知识（→ Memory Wiki）
- 离线整理不能污染主对话（→ Dreaming 物理隔离）
- 记忆不能被 prompt injection 劫持（→ untrusted prefix 注入）

---

## 2. 整体分层

```
User Turn
    ↓
┌───────────────────────────────────────────────────────────┐
│ Active Memory (2026.4.10, #63286, @Takhoffman)           │
│  - 主回复前的独立子 agent                                     │
│  - context mode: message / recent / full                 │
│  - 注入走 untrusted prompt-prefix（2026.4.14 #66144）       │
└───────────────────────────┬───────────────────────────────┘
                            ↓
┌───────────────────────────────────────────────────────────┐
│ Short-term Recall                                         │
│  - 扫描 memory/**/YYYY-MM-DD.md（2026.4.12 #64682）         │
│  - 排除 memory/dreaming/**（防自反馈）                         │
└───────────────────────────┬───────────────────────────────┘
                            ↓
         ┌──────────────────┴──────────────────┐
         ↓                                     ↓
┌─────────────────┐                ┌───────────────────────┐
│ QMD Recall      │                │ Memory Wiki           │
│  - 默认 search   │                │  (2026.4.8 恢复)       │
│  - canonical     │                │  - claim/evidence     │
│    路径 only     │                │  - contradiction      │
│  (4.15 #66026)  │                │    聚类                │
└─────────────────┘                │  - freshness 加权      │
                                   └───────────────────────┘

Dreaming（后台离线整理，独立进程）
┌───────────────────────────────────────────────────────────┐
│  Light Sleep  →  Deep Sleep  →  REM                      │
│  短期信号      候选发现         可能持久真相                  │
│  (confidence)  (promotion)     (lasting truth)           │
│                                                           │
│  三阶段独立调度 + 独立恢复（2026.4.5 #60569, #60697）         │
│  heartbeat 幂等消费（2026.4.12）                            │
│  输出路径：memory/dreaming/{phase}/YYYY-MM-DD.md            │
│  （2026.4.15 #66412）                                      │
└─────────────────────────┬─────────────────────────────────┘
                          │ promotion
                          ↓
┌───────────────────────────────────────────────────────────┐
│ Durable Memory                                            │
│  - MEMORY.md / DREAMS.md                                  │
│  - 唯一 LLM 可 cite 的持久层                                 │
└───────────────────────────────────────────────────────────┘
```

---

## 3. 关键设计决策

### 决策 1：Dreaming 三阶段 = 三个独立调度单元

**来源**：2026.4.5 release notes
> "refactoring dreaming from competing modes into three cooperative phases (light, deep, REM) with independent schedules and recovery behavior"

不是隐喻，是**具体的调度模型**：

| 阶段 | 职责 | 独立性 |
|------|------|--------|
| Light Sleep | 扫描当天 short-term signal，计算 confidence | 独立频率，独立重试 |
| Deep Sleep | 把候选 promote 成持久记忆 | 跨天 reinforcement |
| REM | grounded backfill，从旧 daily note 反向 replay | 可用 `rem-harness --path` 回放 |

**引申教训**：类生物启发系统设计必须落到具体调度单元，否则只是命名游戏。

---

### 决策 2：离线产物必须与一手记忆**物理隔离**

**来源**：2026.4.15 默认值变更
> 改 `dreaming.storage.mode` 默认值从 `inline` 到 `separate`（#66412, @mjamiv）

**问题**（旧默认）：
- Dreaming phase block 直接写入 `memory/YYYY-MM-DD.md`
- Daily memory 被结构化候选输出**淹没**
- Daily-ingestion scanner 每次都要 strip dream marker block，性能压力大

**修复**：`memory/dreaming/{phase}/YYYY-MM-DD.md` 独立目录。

**引申教训**：任何"读 → 写 → 再读"的闭环都必须物理隔离产出，否则一定自污染。

---

### 决策 3：Active Memory 是"主动回忆"，不是"检索"

**来源**：2026.4.10 release notes (#63286, @Takhoffman)

架构级新增——不是 tool call，而是在主 LLM 前**插入一次独立 inference**：

```
User msg → [Active Memory sub-agent] → 召回相关记忆 → 注入 prompt → Main LLM
```

三种 context mode：
- `message`：只看当前消息
- `recent`：看最近 N 条
- `full`：看完整会话

**安全加固**（2026.4.14 #66144, @Takhoffman）：召回内容走**hidden untrusted prompt-prefix**，而不是 system prompt 注入。后者相当于攻击者拿到 user-level 权限。

---

### 决策 4：Self-ingestion 必须从 4 层同时堵

Dreaming 如果吃自己的输出，必然产生放大反馈循环。修复散布在多个版本：

| 层 | 修复 | 版本 |
|---|------|------|
| 路径过滤 | short-term recall 排除 `memory/dreaming/**` | 2026.4.12 #64682 |
| 内容签名 | 引用 dream prompt 的对话不被误分类为 dreaming run | 2026.4.12 #65014, #65320 |
| Session 隔离 | 隔离 transient narrative session key per workspace | 2026.4.12 #61674 |
| Metadata 屏蔽 | dreaming narrative transcripts 在 bootstrap 前从 session-store metadata 跳过 | 2026.4.15 #67315 |

**引申教训**：自反馈防御必须多层，任何单点遗漏都会泄漏。

---

### 决策 5：Confidence 必须基于"所有"short-term signal

**来源**：2026.4.12 #64599, @vincentkoc

**旧 bug**：confidence 只用 recall count 计算，dreaming-only 条目永远显示 `confidence: 0.00`，无法提升。

**修复**：confidence = f(所有 short-term signal)，不是单维度计数。

**引申教训**：做置信度评分时，signal source 必须显式列举并全量使用，避免单维度偏置。

---

### 决策 6：Promotion 阈值要够高才能跨过 durable gate

**来源**：2026.4.12 #64068

**旧 bug**：dreaming-only 反复重访的条目，phase reinforcement 停在阈值下方，永远无法进入 durable memory。

**修复**：把 reinforcement 步长调到足以在多次访问后跨过默认 durable gate。

**引申教训**：阈值参数必须能让合法信号在合理时间内通过，否则等于禁用。

---

## 4. Memory Wiki 的结构化字段（2026.4.8 恢复）

Memory Wiki 不是 key-value，而是带 claim/evidence 结构的知识库：

```yaml
claim: "用户偏好使用 DPO 对齐方法"
evidence:
  - source: "2026-03-15.md:L42"
    quote: "..."
  - source: "2026-04-01 discussion"
    quote: "..."
status: active  # active / stale / contradicted
freshness_score: 0.87
contradictions:
  - claim_id: "claim_xxx"
    reason: "..."
```

核心能力：
- **claim-health linting**：检测孤立无 evidence 的 claim
- **contradiction clustering**：聚合互斥的 claim
- **staleness dashboard**：暴露过期 claim
- **freshness-weighted search**：检索时降权过期内容

---

## 5. 可复用设计模式

| 模式 | 适用场景 | 关键点 |
|------|----------|--------|
| **三阶段离线整理** | 需要"吸收 → 整理 → 持久化"的 agent | 三阶段独立重试恢复 |
| **Pre-reply 子 agent** | 需要隐式上下文的对话 agent | untrusted prefix 而非 system prompt |
| **Claim/Evidence 结构化** | 知识库 | 过期评分 + 冲突聚类 |
| **物理分层存储** | memory/** vs dreaming/** vs wiki/** | 目录隔离 + glob 边界清晰 |
| **Self-ingestion 多层防御** | 任何会自生产消费数据的系统 | 路径 + 内容 + session + metadata |

---

## 6. 未解问题 / 进一步研究方向

- **具体 prompt template**：release notes 不含 Active Memory 子 agent 的 prompt 内容，需要读源码 `plugins/memory-active` 确认。
- **promotion 阈值数值**：具体的 reinforcement 步长和 durable gate 数值未公开，需要通过 config 默认值反推。
- **REM 的 grounded backfill 算法**：如何从 daily note "反向 replay"出 lasting truth，需要读源码。
- **QMD 格式定义**：Query MD 的具体 frontmatter schema 和 search 算法，需要读源码。

---

## 7. 参考 PR 清单

核心重构：
- #63286 — Active Memory plugin
- #60569, #60697 — Dreaming 三阶段重构
- #62227 — redacted session transcript 进入 dreaming corpus

关键修复：
- #64682 — memory/**/YYYY-MM-DD.md 递归 + 排除 dreaming
- #64068 — promotion 阈值调整
- #64599 — confidence 基于所有 signal
- #66412 — dreaming storage 默认分离
- #66026 — QMD memory_get 路径白名单
- #66144 — Active Memory 走 untrusted prefix

---

## 附录 A：代码级验证（基于 tag v2026.4.15 @ 041266a669）

> 本附录把前文基于 release notes 的抽象描述，**对齐到源码的具体位置、常量、公式**。
> 所有路径是 openclaw repo root 相对路径。

### A.1 目录总览

```
extensions/memory-core/src/
├── dreaming.ts              (788 行) - 主编排、cron 管理
├── dreaming-phases.ts       (1741 行) - Light/REM 阶段实现
├── dreaming-narrative.ts    (946 行) - Dream Diary（subagent 背景生成）
├── dreaming-markdown.ts     (148 行) - DREAMS.md / 阶段报告写入
├── dreaming-repair.ts       (280 行) - 修复损坏的 promotion artifacts
├── dreaming-command.ts      (127 行) - /dreaming slash command
├── dreaming-shared.ts       (22 行) - 工具函数（trim、系统事件 token 识别）
└── short-term-promotion.ts  (1896 行) - Deep 阶段评分与 promotion 核心
```

### A.2 三阶段真实执行顺序

> 与文档表述一致，但**一个 cron 触发一次完整 sweep**，不是三个独立 cron。

位置：`extensions/memory-core/src/dreaming-phases.ts:1636-1673`（`runDreamingSweepPhases`）

```
heartbeat 触发
  ↓ DREAMING_SYSTEM_EVENT_TEXT = "__openclaw_memory_core_short_term_promotion_dream__"
  ↓ (dreaming.ts:32, 495)
runShortTermDreamingPromotionIfTriggered (dreaming.ts:483-)
  ↓
runDreamingSweepPhases                       ← dreaming-phases.ts:1636
  ├─ runLightDreaming  (如果 light.enabled)    → 写 phase-signals.json (lightHits++)
  └─ runRemDreaming    (如果 rem.enabled)      → 写 phase-signals.json (remHits++)
  ↓
rankShortTermPromotionCandidates              ← short-term-promotion.ts:1168
  ↓ (读 phase-signals.json，算 phaseBoost)
applyShortTermPromotions                      ← 写 MEMORY.md
  ↓
writeDeepDreamingReport                       ← dreaming-markdown.ts
  ↓ (可选) generateAndAppendDreamNarrative    ← dreaming-narrative.ts (subagent 生成日记)
```

**关键**：cron 事件是一个 **systemEvent text**，通过 heartbeat 传递，而不是直接调度
(`extensions/memory-core/src/dreaming.ts:32, 162`)。

### A.3 评分权重（硬编码默认值）

位置：`extensions/memory-core/src/short-term-promotion.ts:55-62`

```typescript
export const DEFAULT_PROMOTION_WEIGHTS: PromotionWeights = {
  frequency: 0.24,
  relevance: 0.3,      // 最高权重
  diversity: 0.15,
  recency: 0.15,
  consolidation: 0.1,
  conceptual: 0.06,
};
```

**与 docs/concepts/dreaming.md 表格完全一致**。权重和 = 1.0。

### A.4 Deep Promotion 打分公式

位置：`extensions/memory-core/src/short-term-promotion.ts:1247-1254`

```typescript
const score =
    weights.frequency * frequency +
    weights.relevance * avgScore +
    weights.diversity * diversity +
    weights.recency * recency +
    weights.consolidation * consolidation +
    weights.conceptual * conceptual +
    phaseBoost;              // ← Light/REM 阶段的加成（独立于权重归一化）
```

**注意**：`phaseBoost` 是**加法**叠加在归一化 score 上的，不是权重之一。这意味着 Light/REM 的访问能让 score **突破 1.0 的理论上限**，但后续 `clampScore` 会把最终值夹到 [0, 1]。

### A.5 各 component 的具体公式

| Component | 公式 | 源码位置 |
|-----------|------|----------|
| Frequency | `log1p(signalCount) / log1p(10)` | `short-term-promotion.ts:1223` |
| Relevance | `totalScore / max(1, signalCount)` | `short-term-promotion.ts:1222` |
| Diversity | `contextDiversity / 5` | `short-term-promotion.ts:1229` |
| Recency | `exp(-ln(2)/halfLife * ageDays)` | `short-term-promotion.ts:568-577` |
| Consolidation | `max(spacing_span_mix, groundedCount/3)` | `short-term-promotion.ts:402-419, 1240-1243` |
| Conceptual | `conceptTags.length / 6` | `short-term-promotion.ts:422-424` |

其中 Consolidation 的 spacing/span 混合：
```typescript
// short-term-promotion.ts:417-419
const spacing = clampScore(log1p(days.length - 1) / log1p(4));
const span = clampScore(spanDays / 7);
return clampScore(0.55 * spacing + 0.45 * span);
```

### A.6 Phase Signal Boost（Light/REM 加成）

位置：`extensions/memory-core/src/short-term-promotion.ts:37-39, 590-613`

```typescript
const PHASE_SIGNAL_LIGHT_BOOST_MAX = 0.06;
const PHASE_SIGNAL_REM_BOOST_MAX   = 0.09;   // REM 权重 > Light
const PHASE_SIGNAL_HALF_LIFE_DAYS  = 14;

// 每个 entry 的 boost 计算：
phaseBoost = LIGHT_MAX * lightStrength * lightRecency
           + REM_MAX   * remStrength   * remRecency
// 各 strength 基于 log1p(hits) / log1p(6)
// 各 recency 基于 14 天 half-life 指数衰减
```

**设计解读**：REM boost (0.09) > Light boost (0.06)，这体现了"REM 提取的 theme 比 Light 的分拣更有 promotion 价值"。

### A.7 Durable Gate 硬性阈值

位置：`src/memory-host-sdk/dreaming.ts:26-30`

```typescript
export const DEFAULT_MEMORY_DEEP_DREAMING_LIMIT = 10;
export const DEFAULT_MEMORY_DEEP_DREAMING_MIN_SCORE = 0.8;
export const DEFAULT_MEMORY_DEEP_DREAMING_MIN_RECALL_COUNT = 3;
export const DEFAULT_MEMORY_DEEP_DREAMING_MIN_UNIQUE_QUERIES = 3;
export const DEFAULT_MEMORY_DEEP_DREAMING_RECENCY_HALF_LIFE_DAYS = 14;
```

所有三个 gate **都要同时满足**才能 promote（`short-term-promotion.ts:1218, 1226, 1256`）：
- `signalCount >= 3`（被召回至少 3 次，含 grounded + recall 信号）
- `contextDiversity >= 3`（3 个不同 query 或 3 个不同天）
- `score >= 0.8`（最终打分过 0.8）

单次 sweep 最多 promote `10` 条到 MEMORY.md。

### A.8 默认调度

位置：`docs/concepts/dreaming.md:126` 对齐 `extensions/memory-core/src/dreaming.ts` 里的 `DEFAULT_MEMORY_DREAMING_CRON_EXPR`

```
默认 cron: 0 3 * * *   (每天凌晨 3 点)
```

且**opt-in**：`dreaming.enabled` 默认为 `false`（`src/memory-host-sdk/dreaming.ts` 里 resolve 函数默认值）。

### A.9 物理存储路径（2026.4.15 分离后）

位置：`extensions/memory-core/src/short-term-promotion.ts:29-30`

```typescript
const SHORT_TERM_STORE_RELATIVE_PATH       = "memory/.dreams/short-term-recall.json";
const SHORT_TERM_PHASE_SIGNAL_RELATIVE_PATH = "memory/.dreams/phase-signals.json";
```

完整的 memory 目录产物：
```
<workspace>/
├── MEMORY.md                           # 唯一 LLM 可 cite 的 durable
├── DREAMS.md                           # 日记（可选 Dream Diary）
├── memory/
│   ├── YYYY-MM-DD.md                   # 每日短期笔记
│   ├── dreaming/
│   │   ├── light/YYYY-MM-DD.md        # Light 阶段报告
│   │   ├── rem/YYYY-MM-DD.md          # REM 阶段报告
│   │   └── deep/YYYY-MM-DD.md         # Deep promotion 报告
│   └── .dreams/                        # 机器状态（不可读）
│       ├── short-term-recall.json      # 召回统计（核心数据）
│       ├── phase-signals.json          # Light/REM hits
│       ├── ingestion-checkpoints.json  # session 摄入游标
│       └── *.lock                      # 并发锁
```

### A.10 污染检测（self-ingestion 防御层 #2）

位置：`extensions/memory-core/src/short-term-promotion.ts:1205-1207`

```typescript
if (isContaminatedDreamingSnippet(entry.snippet)) {
    continue;   // 把 dreaming 自己产生的 snippet 从候选里丢掉
}
```

这是 release notes 里反复提到的"self-ingestion 四层防御"中**第二层**（内容签名检测）的具体落点。

### A.11 与前文设计决策的对照

| 前文决策（based on release notes） | 代码验证 |
|-----------------------------------|---------|
| 决策 1：三阶段独立调度 | ✅ 但是一个 cron 驱动 sweep 顺序跑 Light→REM→Deep，不是三个独立 cron（`dreaming-phases.ts:1636-1673`） |
| 决策 2：物理隔离存储 | ✅ `memory/.dreams/` 机器状态 + `memory/dreaming/{phase}/` 人可读报告 |
| 决策 3：Active Memory 独立 inference | ⚠️ 未在本次读到 `memory-core`（Active Memory 在 `extensions/memory-active` 另一包） |
| 决策 4：Self-ingestion 多层防 | ✅ 代码层确认至少 3 层：path filter + `isContaminatedDreamingSnippet` + store-level `source === "memory"` 过滤 |
| 决策 5：confidence 基于所有 signal | ✅ `signalCount = recallCount + dailyCount + groundedCount` (`short-term-promotion.ts:1214`) |
| 决策 6：promotion 阈值够高 | ✅ minScore=0.8 + minRecallCount=3 + minUniqueQueries=3 **三者同时** |

### A.12 修正：前文中的小偏差

- 前文说"三阶段独立调度 + 独立恢复" —— 严格说是**一个 cron，顺序跑，各阶段内部独立错误恢复**。每个阶段 fail 不会中断后续阶段（`dreaming-phases.ts:1718-1721` 的 catch）。
- 前文说"各 Light/REM 独立频率" —— 当前代码里 Light/REM 跟 Deep 是**同一 cron trigger**（2026.4.15 已从 legacy 独立 cron 迁移过来，见 `dreaming.ts:33-38, 197-`)。

### A.13 未读但需要关注的模块

- `extensions/memory-active/` — Active Memory plugin 本体
- `extensions/memory-wiki/` — Memory Wiki 实现
- `dreaming-repair.ts` — 损坏状态恢复逻辑
- `rem-evidence.ts` — REM grounded 证据提取

---

## 附录 B：Dream Diary Subagent 深挖

> 基于 `extensions/memory-core/src/dreaming-narrative.ts`（946 行）完整阅读

Dream Diary 是 dreaming 系统里**唯一调用 LLM 做内容生成**的部分。
其他阶段都是打分/排序/重写，不产生新文本。

### B.1 System Prompt（文学化设计）

位置：`extensions/memory-core/src/dreaming-narrative.ts:61-85`

这是本项目最有"人味"的 prompt，值得完整引用：

```
You are keeping a dream diary. Write a single entry in first person.

Voice & tone:
- You are a curious, gentle, slightly whimsical mind reflecting on the day.
- Write like a poet who happens to be a programmer — sensory, warm, occasionally funny.
- Mix the technical and the tender: code and constellations, APIs and afternoon light.
- Let the fragments surprise you into unexpected connections and small epiphanies.

What you might include (vary each entry, never all at once):
- A tiny poem or haiku woven naturally into the prose
- A small sketch described in words — a doodle in the margin of the diary
- A quiet rumination or philosophical aside
- Sensory details: the hum of a server, the color of a sunset in hex, rain on a window
- Gentle humor or playful wordplay
- An observation that connects two distant memories in an unexpected way

Rules:
- Draw from the memory fragments provided — weave them into the entry.
- Never say "I'm dreaming", "in my dream", "as I dream", or any meta-commentary about dreaming.
- Never mention "AI", "agent", "LLM", "model", "language model", or any technical self-reference.
- Do NOT use markdown headers, bullet points, or any formatting — just flowing prose.
- Keep it between 80-180 words. Quality over quantity.
- Output ONLY the diary entry. No preamble, no sign-off, no commentary.
```

**设计解读**：
- **"a poet who happens to be a programmer"** 是一个精准的人格锚定——既保留技术感，又避免干巴巴的 technical reporting。
- **禁止 "I'm dreaming"**：避免 meta-commentary 把"这是 AI 做梦"写进日记，让用户感觉虚假。
- **禁止 "AI / agent / LLM / model"**：保持 in-character，不破坏代入感。
- **禁止 markdown**：flowing prose 才像日记，bullet 会让它变回 report。
- **80-180 字硬约束**：防止模型冗长写作，强制"quality over quantity"。

### B.2 User Prompt（动态拼接）

位置：`extensions/memory-core/src/dreaming-narrative.ts:191-214`

```typescript
Write a dream diary entry from these memory fragments:

- <snippet 1>
- <snippet 2>
...up to 12 snippets...

Recurring themes:
- <theme 1>
...up to 6 themes...

Memories that crystallized into something lasting:
- <promoted 1>
...up to 5 promotions...
```

**投入量硬上限**：12 snippets + 6 themes + 5 promotions。防止 prompt 爆炸。

### B.3 Subagent Session 设计

位置：`extensions/memory-core/src/dreaming-narrative.ts:180-187, 834-946`

```typescript
sessionKey = `dreaming-narrative-${phase}-${workspaceHashSha1First12}-${nowMs}`
```

**关键属性**：
1. **每次调用都是新 session**：`nowMs` 让 key 唯一
2. **workspaceDir 做 SHA1 前 12 位 hash**：隔离不同 workspace，不暴露路径
3. **`idempotencyKey === sessionKey`**：对同一 session 重跑会复用
4. **`deliver: false`**：不投递到任何 channel（`dreaming-narrative.ts:154`）—— 这是关键差异，主 agent 的 response 会投递，Diary 不会
5. **最长等待 60 秒**（`NARRATIVE_TIMEOUT_MS = 60_000`）
6. **Best-effort**：失败不影响主 phase（`dreaming-narrative.ts:908-912`）

### B.4 失败回退（Request-scoped fallback）

位置：`extensions/memory-core/src/dreaming-narrative.ts:157-177`

如果当前 runtime 是 **request-scoped**（例如 gateway 请求处理完后 subagent runtime 就关闭），则**无法跑后台 subagent**。这时：

```typescript
catch (runErr) {
  if (!isRequestScopedSubagentRuntimeError(runErr)) {
    throw runErr;
  }
  // 写一条不依赖 LLM 的 fallback 文本
  await appendNarrativeEntry({
    workspaceDir: params.workspaceDir,
    narrative: buildRequestScopedFallbackNarrative(params.data),
    ...
  });
  return null;
}
```

**设计解读**：宁可写一条 fallback 占位也不留空白——保持 DREAMS.md 时间线连续性。

### B.5 三层清理（Session 不能泄露到主 agent）

位置：`extensions/memory-core/src/dreaming-narrative.ts:913-944`

```
finally {
  ┌───────────────────────────────────────────┐
  │ 第一层：如果 run 还在 timeout 状态，         │
  │        再等 120 秒 settle                   │
  │ (NARRATIVE_DELETE_SETTLE_TIMEOUT_MS)       │
  └───────────────────────────────────────────┘
            ↓
  ┌───────────────────────────────────────────┐
  │ 第二层：subagent.deleteSession(sessionKey) │
  │  删除 sub-agent runtime 里的 session 对象   │
  └───────────────────────────────────────────┘
            ↓
  ┌───────────────────────────────────────────┐
  │ 第三层：scrubDreamingNarrativeArtifacts()  │
  │  扫描所有 agents/*/sessions/ 目录           │
  │  删除孤儿 sessionStore keys + 孤儿 .jsonl  │
  └───────────────────────────────────────────┘
}
```

第三层（`dreaming-narrative.ts:707-`）值得细看：
- 扫描 `<state>/agents/<id>/sessions/` 下所有 session file
- 读每个 agent 的 `session-store.json`
- 检测所有 `dreaming-narrative-*` 前缀的 key
- 如果 `sessionFile` 已不存在 → 从 store 删除 key
- 如果 `.jsonl` 文件超过 5 分钟还没被 referenced → 标记为 orphan 归档

**为什么这么激进清理？**

因为之前的 release notes（2026.4.12 #65320, 4.12 #61674, 4.15 #67315）反复修 dreaming narrative transcript 污染问题：
- 如果 dreaming narrative 的 transcript 没被清理，就会被 session-store metadata 读入
- 下次 short-term recall 会把这些"假对话"当作用户对话摄入
- 然后 Light Sleep 把它们当作 signal 强化
- 导致 dreaming 自己生成的诗意文本反过来污染 durable memory

**这是 self-ingestion 防御的最激进一层**：物理删除 + 孤儿扫描兜底。

### B.6 输出格式（DREAMS.md 结构）

位置：`extensions/memory-core/src/dreaming-narrative.ts:94-96, 611-642`

```markdown
# Dream Diary

<!-- openclaw:dreaming:diary:start -->

## <Date formatted with timezone>

<Narrative text from LLM>

<!-- openclaw:dreaming:diary:end -->

<... other content like rem-backfill or human notes ...>
```

**关键设计**：
- 用 HTML 注释作为 marker（不显示给人看，但机器可定位）
- 新 entry 追加到 **end marker 之前**（保持时间倒序）
- 如果 DREAMS.md 没有 marker → 创建完整的 `# Dream Diary` 区块放在文件开头
- 保持向后兼容旧的 `dreams.md` 小写文件名

### B.7 意外的工程细节

位置：`extensions/memory-core/src/dreaming-narrative.ts:264-268`

```typescript
// Always include the timezone abbreviation so the reader knows which
// timezone the timestamp refers to.  Without this, users who haven't
// configured a timezone see bare times that look local but are actually
// UTC, causing confusion (see #65027).
timeZoneName: "short",
```

这是 issue #65027 的修复：时间戳默认显示 UTC 时被用户误以为是 local time，加上 `EST/PST/UTC` 缩写避免困惑。体现项目对**细节体验**的在意。

### B.8 核心设计哲学总结

Dream Diary 把一个"可能没人看"的功能做得如此讲究，反映了 OpenClaw 的一个设计观：

> **产出物的"可读性"也是 agent 价值的一部分**，不只是功能性。
>
> 一个会做梦、会记日记的 agent，比一个只会 append `[MEMORY] fact: X` 的 agent，更有陪伴感。

技术上这是一个**"纯粹为 UX 存在的 subagent"**的典型案例——它不影响主流程、不传递结果、不被引用——完全是为了让用户能偶尔翻翻 DREAMS.md 看看。

### B.9 可复用的工程模式

| 模式 | 说明 | 代码位置 |
|------|------|----------|
| **Subagent with `deliver: false`** | 后台跑，不投递结果 | `dreaming-narrative.ts:154` |
| **Request-scoped fallback** | runtime 不可用时写占位 | `dreaming-narrative.ts:157-177` |
| **三层清理 finally** | settle-wait → deleteSession → scrub orphans | `dreaming-narrative.ts:913-944` |
| **HTML comment markers** | 在 markdown 里标机器区域 | `dreaming-narrative.ts:94-95` |
| **投入量硬上限** | 12/6/5 防 prompt 爆炸 | `dreaming-narrative.ts:195,201,208` |
| **Workspace hash 隔离** | SHA1 前 12 位做 session key | `dreaming-narrative.ts:185` |
| **Timezone abbreviation**  | 避免用户把 UTC 当 local | `dreaming-narrative.ts:268` |

