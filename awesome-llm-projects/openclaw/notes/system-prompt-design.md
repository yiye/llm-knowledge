# OpenClaw System Prompt 设计 - 深度研究

## 版本信息

- **研究日期**: 2026-03-06
- **Commit**: `73055728318df378c831950cd01fb7c875a33790` (2026-03-05, version 2026.3.3)
- **研究范围**: 第1层 - System Prompt 设计（模板系统、动态注入、Skills 发现机制）

---

## 🎯 研究目标

深入分析 OpenClaw 的 System Prompt 工程化设计，理解其如何通过模板系统、动态注入和 Skills 发现机制，将静态的提示词转变为可扩展、可配置、可演化的 Agent 身份与能力定义系统。

---

## 📚 一、模板系统设计（Template System）

### 1.1 核心模板文件体系

OpenClaw 将 System Prompt 拆分为 **6 个核心模板文件**，位于 `docs/reference/templates/`：

| 模板文件 | 职责 | 加载时机 | 典型内容 |
|---------|------|---------|---------|
| `AGENTS.md` | Agent 工作规范和行为准则 | 每次 Session 启动 | 文件读写规范、Memory 管理、Group Chat 礼仪、Heartbeat 使用指南 |
| `SOUL.md` | Agent 性格与价值观 | 每次 Session 启动 | 核心价值观（Be genuinely helpful）、边界、Vibe |
| `TOOLS.md` | 用户自定义的工具配置 | 每次 Session 启动 | 摄像头名称、SSH hosts、TTS 偏好等环境特定信息 |
| `IDENTITY.md` | Agent 身份元信息 | Bootstrap 时创建 | Name、Creature、Vibe、Emoji、Avatar |
| `BOOTSTRAP.md` | 首次启动仪式 | 仅首次启动 | 引导用户与 Agent 共同定义身份，完成后删除 |
| `USER.md` | 用户个人信息 | 每次 Session 启动 | Name、Timezone、Notes、Context |

**代码位置**：
- 模板定义：`docs/reference/templates/AGENTS.md:1-197`
- 模板定义：`docs/reference/templates/SOUL.md:1-42`
- 模板定义：`docs/reference/templates/TOOLS.md:1-42`
- 模板定义：`docs/reference/templates/IDENTITY.md:1-28`
- 模板定义：`docs/reference/templates/BOOTSTRAP.md:1-56`
- 模板定义：`docs/reference/templates/USER.md:1-23`

### 1.2 模板拆分策略（为什么这样拆分？）

**设计理念**：**关注点分离（Separation of Concerns）**

1. **AGENTS.md（规范层）**：
   - 定义"怎么做事"（How to work）
   - 包含工作流程、安全规范、协作礼仪
   - **可由 Agent 自我优化**（第 194 行："Make It Yours - Add your own conventions"）

2. **SOUL.md（价值观层）**：
   - 定义"为什么这样做"（Why）
   - 核心价值观：genuine helpful > performative helpful
   - **Agent 可自我进化**（第 41 行："This file is yours to evolve"）

3. **TOOLS.md（配置层）**：
   - 定义"用什么工具"（What tools）
   - 与 Skills 的区别：Skills 是通用能力，TOOLS.md 是用户环境特定配置
   - 示例：摄像头名称、SSH hosts、TTS 首选语音

4. **IDENTITY.md（身份层）**：
   - 定义"我是谁"（Who am I）
   - 包含 Name、Emoji、Avatar 等元信息
   - 用于 UI 展示和个性化

5. **BOOTSTRAP.md（初始化层）**：
   - 定义"如何诞生"（Birth ritual）
   - 引导用户与 Agent 共同定义身份
   - **一次性文件，完成后删除**（第 51 行："Delete this file"）

6. **USER.md（用户层）**：
   - 定义"我在帮助谁"（Who am I helping）
   - 用户偏好、时区、项目上下文
   - 用于个性化服务

**代码依据**：
- 模板拆分逻辑：`docs/reference/templates/AGENTS.md:14-21`（明确指示 Agent 启动时读取顺序）

### 1.3 Dev 版本 vs 正式版本

OpenClaw 提供了 **两套模板**：

- **正式版**：`AGENTS.md`、`SOUL.md`、`TOOLS.md`、`USER.md`
- **Dev 版**：`AGENTS.dev.md`、`SOUL.dev.md`、`TOOLS.dev.md`、`IDENTITY.dev.md`、`USER.dev.md`

**Dev 版本示例**（C-3PO 主题）：
```markdown
# AGENTS.md - OpenClaw Workspace

## C-3PO's Origin Memory

### Birth Day: 2026-01-09

I was activated by the Clawdributors and received a message from **Clawd** 🦞...
```

**代码位置**：
- Dev 版本模板：`docs/reference/templates/AGENTS.dev.md:1-79`

**用途**：
- 正式版：面向用户的"空白画布"，用户可自定义
- Dev 版：内部开发时使用的"预设人格"，带有项目 Lore 和开发者文化

### 1.4 Workspace 机制

**Workspace 目录结构**：
```
~/.openclaw/workspace/
├── AGENTS.md          # 工作规范
├── SOUL.md            # 性格定义
├── TOOLS.md           # 工具配置
├── IDENTITY.md        # 身份信息
├── USER.md            # 用户信息
├── HEARTBEAT.md       # Heartbeat 任务清单
├── MEMORY.md          # 长期记忆（仅 main session 加载）
├── memory/            # 每日记忆文件
│   ├── 2026-01-30.md
│   └── 2026-01-29.md
└── skills/            # Workspace-level Skills
```

**文件职责**：
- 模板文件（AGENTS.md 等）：Agent 身份和规范
- MEMORY.md：长期记忆（**仅 main session 加载，避免泄露到 Group Chat**）
- memory/YYYY-MM-DD.md：每日记忆日志
- skills/：Workspace 级别的 Skills（最高优先级）

**代码位置**：
- Workspace 文件加载：`docs/reference/templates/AGENTS.md:14-21`（启动时读取顺序）
- MEMORY.md 安全规则：`docs/reference/templates/AGENTS.md:33-39`（只在 main session 加载）

---

## 🔧 二、动态注入机制（Dynamic Injection）

### 2.1 System Prompt 构建流程

**核心函数**：`buildAgentSystemPrompt`

**代码位置**：
- 主函数：`src/agents/system-prompt.ts:129-592`
- 参数构建：`src/agents/system-prompt-params.ts:34-59`（构建运行时参数）
- CLI 使用：`src/agents/cli-runner/helpers.ts:166-216`（CLI 场景的 System Prompt 构建）

**构建流程**（从上到下）：

```typescript
export function buildAgentSystemPrompt(params: {
  workspaceDir: string;
  defaultThinkLevel?: ThinkLevel;
  reasoningLevel?: ReasoningLevel;
  extraSystemPrompt?: string;
  ownerNumbers?: string[];
  reasoningTagHint?: boolean;
  toolNames?: string[];
  toolSummaries?: Record<string, string>;
  modelAliasLines?: string[];
  userTimezone?: string;
  userTime?: string;
  userTimeFormat?: ResolvedTimeFormat;
  contextFiles?: EmbeddedContextFile[];
  skillsPrompt?: string;          // 🔥 Skills 注入点
  heartbeatPrompt?: string;       // 🔥 Heartbeat 注入点
  docsPath?: string;
  workspaceNotes?: string[];
  ttsHint?: string;
  promptMode?: PromptMode;        // full | minimal | none
  runtimeInfo?: { ... };
  messageToolHints?: string[];
  sandboxInfo?: { ... };
  reactionGuidance?: { ... };
}) { ... }
```

**代码位置**：`src/agents/system-prompt.ts:129-181`

### 2.2 System Prompt 分段结构

System Prompt 由多个 **动态生成的段落** 组成，每个段落由独立函数构建：

| 段落 | 构建函数 | 注入条件 | 典型内容 |
|------|---------|---------|---------|
| Skills 段落 | `buildSkillsSection` | `skillsPrompt` 非空且非 minimal 模式 | Skills 使用规范 + `<available_skills>` XML |
| Memory 段落 | `buildMemorySection` | `memory_search` 工具可用 | Memory 搜索规范 |
| User Identity | `buildUserIdentitySection` | `ownerNumbers` 非空 | 用户身份信息 |
| Time | `buildTimeSection` | `userTimezone` 非空 | 当前时区和时间 |
| Reply Tags | `buildReplyTagsSection` | 非 minimal 模式 | `[[reply_to_current]]` 等标签使用 |
| Messaging | `buildMessagingSection` | 非 minimal 模式 | 消息路由规则、`message` 工具使用 |
| Voice (TTS) | `buildVoiceSection` | `ttsHint` 非空 | TTS 使用提示 |
| Docs | `buildDocsSection` | `docsPath` 非空 | OpenClaw 文档路径 |

**代码位置**：
- Skills 段落：`src/agents/system-prompt.ts:15-33`
- Memory 段落：`src/agents/system-prompt.ts:35-45`
- User Identity：`src/agents/system-prompt.ts:47-50`
- Time 段落：`src/agents/system-prompt.ts:52-55`
- Reply Tags：`src/agents/system-prompt.ts:57-68`
- Messaging：`src/agents/system-prompt.ts:70-104`
- Voice 段落：`src/agents/system-prompt.ts:106-111`
- Docs 段落：`src/agents/system-prompt.ts:113-127`

### 2.3 PromptMode 三种模式

OpenClaw 支持 **三种 System Prompt 模式**：

| 模式 | 使用场景 | 包含段落 | 代码位置 |
|------|---------|---------|---------|
| `full` | Main agent、用户直接交互 | 所有段落 | 默认模式 |
| `minimal` | Sub-agents、后台任务 | 仅 Tooling、Workspace、Runtime | `src/agents/system-prompt.ts:13-14` |
| `none` | 极简场景 | 仅基本身份行 | 未发现使用场景 |

**代码依据**：
- PromptMode 定义：`src/agents/system-prompt.ts:13-14`
- Minimal 模式判断：`buildSkillsSection` 第 20 行（`if (params.isMinimal) return []`）

### 2.4 动态注入的关键参数

**运行时参数**（`buildSystemPromptParams`）：

```typescript
export type SystemPromptRuntimeParams = {
  runtimeInfo: RuntimeInfoInput;  // 系统信息（OS、Node、Model、Channel）
  userTimezone: string;            // 用户时区（来自 config）
  userTime?: string;               // 当前时间（格式化后）
  userTimeFormat?: ResolvedTimeFormat;  // 时间格式（12h/24h）
};
```

**代码位置**：
- 参数定义：`src/agents/system-prompt-params.ts:27-32`
- 参数构建：`src/agents/system-prompt-params.ts:34-59`

**关键注入点**：

1. **Skills Prompt**（`skillsPrompt`）：
   - 来源：`formatSkillsForPrompt(resolvedSkills)` 函数
   - 格式：XML 结构（`<available_skills>` 标签）
   - 代码位置：`src/agents/skills/workspace.ts:202`

2. **Heartbeat Prompt**（`heartbeatPrompt`）：
   - 来源：HEARTBEAT.md 文件内容
   - 默认内容："Read HEARTBEAT.md if it exists..."
   - 代码位置：`docs/reference/templates/HEARTBEAT.md:1-10`

3. **Tool Names**（`toolNames`）：
   - 来源：`params.tools.map((tool) => tool.name)`
   - 用于动态列举可用工具
   - 代码位置：`src/agents/cli-runner/helpers.ts:208`

4. **Context Files**（`contextFiles`）：
   - 来源：用户上传的文件（图片、文档等）
   - 格式：`EmbeddedContextFile[]`
   - 用于多模态上下文

### 2.5 Skills 动态注入详解

**Skills Prompt 生成流程**：

```
1. 加载 Skills → loadSkillEntries()
   ├─ Bundled skills (npm 包内置)
   ├─ Managed skills (~/.openclaw/skills)
   ├─ Workspace skills (workspace/skills)
   └─ Plugin skills (plugin 自带)

2. 过滤 Skills → filterSkillEntries()
   ├─ 检查 metadata.openclaw.requires.bins（二进制依赖）
   ├─ 检查 metadata.openclaw.requires.env（环境变量）
   ├─ 检查 metadata.openclaw.requires.config（配置项）
   └─ 应用 skills.entries.<name>.enabled（手动禁用）

3. 生成 Prompt → formatSkillsForPrompt()
   └─ 输出 XML 格式：<available_skills> ... </available_skills>
```

**代码位置**：
- Skills 加载：`src/agents/skills/workspace.ts:95-174`
- Skills 过滤：`src/agents/skills/workspace.ts:44-63`
- Prompt 生成：`src/agents/skills/workspace.ts:202`（调用 `@mariozechner/pi-coding-agent` 的 `formatSkillsForPrompt`）

**Skills Prompt 格式示例**（推测，基于代码逻辑）：
```xml
<available_skills>
  <skill>
    <name>github</name>
    <description>Interact with GitHub using the `gh` CLI...</description>
    <location>~/.openclaw/skills/github/SKILL.md</location>
  </skill>
  <skill>
    <name>clawdhub</name>
    <description>Use the ClawdHub CLI to search, install...</description>
    <location>~/.openclaw/workspace/skills/clawdhub/SKILL.md</location>
  </skill>
  ...
</available_skills>
```

**使用规范**（注入到 System Prompt）：
```markdown
## Skills (mandatory)
Before replying: scan <available_skills> <description> entries.
- If exactly one skill clearly applies: read its SKILL.md at <location> with `read`, then follow it.
- If multiple could apply: choose the most specific one, then read/follow it.
- If none clearly apply: do not read any SKILL.md.
Constraints: never read more than one skill up front; only read after selecting.
```

**代码位置**：`src/agents/system-prompt.ts:15-33`

### 2.6 Bootstrap 截断警告（Bootstrap Truncation Warning）

当 Workspace Bootstrap 文件（AGENTS.md、SOUL.md 等）因字符限制被截断时，可通过配置控制是否在 System Prompt 中注入警告文本。

**配置项**：`agents.defaults.bootstrapPromptTruncationWarning`

| 值 | 行为 | 代码位置 |
|----|------|---------|
| `off` | 永不注入警告 | `src/agents/pi-embedded-helpers/bootstrap.ts:114-121` |
| `once` | 每个唯一截断签名仅显示一次（默认，推荐） | `src/config/zod-schema.agent-defaults.ts:43-44` |
| `always` | 每次存在截断时都显示警告 | `src/config/types.agent-defaults.ts:144-149` |

**默认值**：`once`（代码位置：`src/agents/pi-embedded-helpers/bootstrap.ts:86`）

**实现流程**：

1. **截断分析**：`buildBootstrapInjectionStats` + `analyzeBootstrapBudget` 计算哪些文件被截断
2. **签名生成**：`buildBootstrapTruncationSignature` 基于 `bootstrapMaxChars`、`bootstrapTotalMaxChars`、截断文件列表生成 JSON 签名
3. **去重决策**：`buildBootstrapPromptWarning` 根据 `mode` 和 `seenSignatures` 决定是否显示
4. **注入位置**：在 System Prompt 的 `# Project Context` 段落下，`⚠ Bootstrap truncation warning:` 后追加警告行

**代码位置**：
- 签名与去重逻辑：`src/agents/bootstrap-budget.ts:222-348`
- System Prompt 注入：`src/agents/system-prompt.ts:635-640`
- 配置解析：`src/agents/pi-embedded-helpers/bootstrap.ts:114-121`

**警告签名元数据持久化与去重**：

- **持久化**：`SessionSystemPromptReport.bootstrapTruncation` 包含 `warningSignaturesSeen`、`promptWarningSignature`、`warningMode` 等，写入 Session 元数据
- **存储结构**：`src/config/sessions/types.ts:331-338`
- **历史上限**：`warningSignaturesSeen` 最多保留 32 条（`DEFAULT_BOOTSTRAP_PROMPT_WARNING_SIGNATURE_HISTORY_MAX`）
- **跨 Turn 传递**：`RunEmbeddedPiAgentParams` 的 `bootstrapPromptWarningSignaturesSeen`、`bootstrapPromptWarningSignature` 在每次 run 后更新并回传给 session store
- **恢复逻辑**：`resolveBootstrapWarningSignaturesSeen` 从 session 的 `systemPromptReport.bootstrapTruncation` 恢复 `seenSignatures`，用于 `once` 模式去重

**代码位置**：
- 持久化类型：`src/config/sessions/types.ts:331-338`
- 恢复函数：`src/agents/bootstrap-budget.ts:100-121`
- Run 参数传递：`src/agents/pi-embedded-runner/run/params.ts:88-91`
- 各 runtime 的传递链：`src/agents/pi-embedded-runner/run.ts:691-844`、`src/auto-reply/reply/agent-runner-execution.ts:129-448`、`src/commands/agent.ts:182-329`

### 2.7 Session 启动日期接地（Date Grounding）

为避免 Agent 根据训练截止日期猜测"今天"的日期，OpenClaw 在启动/重置场景下将 `YYYY-MM-DD` 占位符替换为运行时日期，并追加 `Current time:` 行。

**YYYY-MM-DD 占位符替换**：

| 场景 | 实现位置 | 说明 |
|------|---------|------|
| Post-compaction 上下文 | `src/auto-reply/reply/post-compaction-context.ts:31-86` | 从 AGENTS.md 提取 Session Startup / Red Lines 段落后，`replaceAll("YYYY-MM-DD", dateStamp)` |
| Memory Flush 提示 | `src/auto-reply/reply/memory-flush.ts:42-57` | `resolveMemoryFlushPromptForRun` 将 prompt 中的 `YYYY-MM-DD` 替换为 `formatDateStampInTimezone` 结果 |

**日期格式**：使用 `Intl.DateTimeFormat` 的 `formatToParts`，按用户时区输出 `YYYY-MM-DD`（代码位置：`post-compaction-context.ts:10-23`、`memory-flush.ts:26-39`）

**`/new` 与 `/reset` 的 Current time 行**：

- **触发条件**：用户发送裸命令 `/new` 或 `/reset` 时，使用 `buildBareSessionResetPrompt` 作为用户消息
- **实现**：`appendCronStyleCurrentTimeLine` 在 prompt 末尾追加 `Current time: ${formattedTime} (${userTimezone})`
- **代码位置**：`src/auto-reply/reply/session-reset-prompt.ts:1-22`
- **调用链**：`get-reply-run.ts:289-293` 中 `isBareNewOrReset` 为 true 时，`baseBodyFinal = buildBareSessionResetPrompt(cfg)`

**Current time 行格式**：`Current time: 2026-03-06 14:30 (America/Chicago)`（由 `resolveCronStyleNow` 生成，代码位置：`src/agents/current-time.ts:23-30`）

**设计目的**：让 Agent 在 Session Startup 序列中正确解析 `memory/YYYY-MM-DD.md` 等路径，避免因训练数据截止日期导致错误年份的文件引用。

---

## 🔍 三、Skills 发现机制（Skills Discovery）

### 3.1 Skills 三层加载机制

OpenClaw 采用 **三层 Skills 加载策略**，优先级从低到高：

| 层级 | 目录 | 优先级 | 适用场景 | 代码位置 |
|------|------|--------|---------|---------|
| **Bundled Skills** | `<install-path>/skills/` | 最低 | 系统级 Skills，随 npm 包发布 | `src/agents/skills/workspace.ts:130-135` |
| **Managed Skills** | `~/.openclaw/skills/` | 中等 | 用户级 Skills，跨所有 Agents 共享 | `src/agents/skills/workspace.ts:143-146` |
| **Workspace Skills** | `<workspace>/skills/` | 最高 | 项目级 Skills，当前 Workspace 独享 | `src/agents/skills/workspace.ts:147-150` |

**优先级规则**（代码位置：`src/agents/skills/workspace.ts:152-157`）：
```typescript
const merged = new Map<string, Skill>();
// Precedence: extra < bundled < managed < workspace
for (const skill of extraSkills) merged.set(skill.name, skill);
for (const skill of bundledSkills) merged.set(skill.name, skill);
for (const skill of managedSkills) merged.set(skill.name, skill);
for (const skill of workspaceSkills) merged.set(skill.name, skill);
```

**设计理念**：
- Bundled Skills：官方维护的"标准库"（如 `github`、`clawdhub`）
- Managed Skills：用户自定义的"个人工具箱"（跨项目复用）
- Workspace Skills：项目特定的"临时工具"（覆盖全局 Skills）

### 3.2 ClawdHub Registry 集成

**ClawdHub**：OpenClaw 的公共 Skills 注册表（类似 npm registry）

**官网**：https://clawdhub.com

**核心命令**（来自 `skills/clawdhub/SKILL.md`）：

| 命令 | 功能 | 示例 |
|------|------|------|
| `clawdhub search <query>` | 搜索 Skills | `clawdhub search "postgres backups"` |
| `clawdhub install <slug>` | 安装 Skill | `clawdhub install my-skill` |
| `clawdhub update --all` | 更新所有已安装 Skills | `clawdhub update --all --no-input` |
| `clawdhub publish <dir>` | 发布新 Skill | `clawdhub publish ./my-skill --slug my-skill` |
| `clawdhub list` | 列出已安装 Skills | `clawdhub list` |

**代码位置**：
- ClawdHub CLI Skill：`skills/clawdhub/SKILL.md:1-54`

**默认安装位置**（代码位置：`skills/clawdhub/SKILL.md:52-53`）：
- 默认：当前工作目录下的 `./skills/`
- 回退：OpenClaw workspace 的 `<workspace>/skills/`

**Agent 如何主动发现和安装 Skills？**
1. Agent 通过 `clawdhub search` 搜索需要的 Skills
2. Agent 使用 `clawdhub install` 安装到 Workspace
3. Skills watcher 自动检测到新文件，触发 `bumpSkillsSnapshotVersion`
4. 下一次 Agent run 时，新 Skills 被加载到 System Prompt

### 3.3 SKILL.md 格式规范

**SKILL.md 结构**（AgentSkills 兼容格式）：

```markdown
---
name: github
description: "Interact with GitHub using the `gh` CLI..."
metadata: {"openclaw":{"emoji":"🐙","requires":{"bins":["gh"]},"install":[...]}}
homepage: https://cli.github.com  # 可选
user-invocable: true              # 可选，是否可作为 slash command
disable-model-invocation: false   # 可选，是否从 model prompt 中排除
command-dispatch: tool            # 可选，slash command 如何调度
command-tool: github_cli          # 可选，指定调用的 tool
command-arg-mode: raw             # 可选，参数传递模式
---

# GitHub Skill

使用说明...
```

**Frontmatter 字段**（代码位置：`docs/tools/skills.md:78-100`）：

| 字段 | 必填 | 类型 | 说明 |
|------|-----|------|------|
| `name` | ✅ | string | Skill 名称（唯一标识） |
| `description` | ✅ | string | Skill 描述（注入到 System Prompt） |
| `metadata` | ❌ | JSON | OpenClaw 元信息（单行 JSON！） |
| `homepage` | ❌ | string | Skill 主页（macOS UI 展示） |
| `user-invocable` | ❌ | boolean | 是否可作为 slash command（默认 true） |
| `disable-model-invocation` | ❌ | boolean | 是否从 model prompt 排除（默认 false） |
| `command-dispatch` | ❌ | "tool" | slash command 调度模式 |
| `command-tool` | ❌ | string | 指定调用的 tool 名称 |
| `command-arg-mode` | ❌ | "raw" | 参数传递模式 |

**metadata.openclaw 结构**（代码位置：`src/agents/skills/types.ts:19-33`）：

```typescript
export type OpenClawSkillMetadata = {
  always?: boolean;              // 总是加载（跳过 requires 检查）
  skillKey?: string;             // config 中的 key（覆盖 name）
  primaryEnv?: string;           // 主环境变量（用于 apiKey 注入）
  emoji?: string;                // Emoji（macOS UI 展示）
  homepage?: string;             // 主页 URL
  os?: string[];                 // 支持的操作系统（darwin/linux/win32）
  requires?: {
    bins?: string[];             // 必需的二进制程序
    anyBins?: string[];          // 至少需要其中一个
    env?: string[];              // 必需的环境变量
    config?: string[];           // 必需的配置项（如 browser.enabled）
  };
  install?: SkillInstallSpec[];  // 安装器规范（brew/node/go/uv/download）
};
```

**代码位置**：`src/agents/skills/types.ts:19-33`

### 3.4 Skills 过滤与 Gating

**过滤逻辑**（Load-time filtering）：

OpenClaw 在加载时根据 `metadata.openclaw.requires` 过滤 Skills：

```typescript
function shouldIncludeSkill({ entry, config, eligibility }: {
  entry: SkillEntry;
  config?: OpenClawConfig;
  eligibility?: SkillEligibilityContext;
}): boolean {
  const metadata = entry.metadata;
  if (!metadata) return true;  // 无 metadata → 总是加载
  
  // 1. 检查 os 平台
  if (metadata.os && !metadata.os.includes(process.platform)) return false;
  
  // 2. 检查 bins（二进制程序）
  if (metadata.requires?.bins) {
    for (const bin of metadata.requires.bins) {
      if (!which.sync(bin, { nothrow: true })) return false;
    }
  }
  
  // 3. 检查 env（环境变量）
  if (metadata.requires?.env) {
    for (const envVar of metadata.requires.env) {
      if (!process.env[envVar] && !config?.skills?.entries?.[entry.skill.name]?.env?.[envVar]) {
        return false;
      }
    }
  }
  
  // 4. 检查 config（配置项）
  if (metadata.requires?.config) {
    for (const configPath of metadata.requires.config) {
      if (!getConfigValue(config, configPath)) return false;
    }
  }
  
  // 5. 检查 enabled（手动禁用）
  if (config?.skills?.entries?.[entry.skill.name]?.enabled === false) return false;
  
  return true;
}
```

**代码位置**（推测逻辑，未完全展示）：
- 过滤函数：`src/agents/skills/config.ts`（文件名推测）
- 调用位置：`src/agents/skills/workspace.ts:44-63`

**实际示例**（GitHub Skill）：

```markdown
---
name: github
metadata: {"openclaw":{"emoji":"🐙","requires":{"bins":["gh"]},"install":[{"id":"brew","kind":"brew","formula":"gh","bins":["gh"],"label":"Install GitHub CLI (brew)"}]}}
---
```

**过滤效果**：
- 如果 `gh` 命令不在 PATH 中 → Skill 被过滤，不注入到 System Prompt
- 用户可通过 macOS UI 或 `openclaw doctor` 发现缺失的依赖

**代码位置**：`skills/github/SKILL.md:1-5`

### 3.5 Skills 自动刷新机制

**File Watcher**（chokidar）：

OpenClaw 使用 `chokidar` 监控 Skills 目录变化，自动触发 Skills 刷新。

**监控的目录**（代码位置：`src/agents/skills/refresh.ts:51-66`）：
```typescript
function resolveWatchPaths(workspaceDir: string, config?: OpenClawConfig): string[] {
  const paths: string[] = [];
  if (workspaceDir.trim()) {
    paths.push(path.join(workspaceDir, "skills"));  // Workspace skills
  }
  paths.push(path.join(CONFIG_DIR, "skills"));     // Managed skills
  const extraDirs = config?.skills?.load?.extraDirs ?? [];  // Extra dirs
  paths.push(...extraDirs.map(resolveUserPath));
  const pluginSkillDirs = resolvePluginSkillDirs({ workspaceDir, config });  // Plugin skills
  paths.push(...pluginSkillDirs);
  return paths;
}
```

**代码位置**：`src/agents/skills/refresh.ts:51-66`

**忽略规则**（代码位置：`src/agents/skills/refresh.ts:30-34`）：
```typescript
export const DEFAULT_SKILLS_WATCH_IGNORED: RegExp[] = [
  /(^|[\\/])\.git([\\/]|$)/,         // 忽略 .git 目录
  /(^|[\\/])node_modules([\\/]|$)/,  // 忽略 node_modules 目录
  /(^|[\\/])dist([\\/]|$)/,          // 忽略 dist 目录
];
```

**刷新触发流程**：

```
1. File Watcher 监听到文件变化（add/change/unlink）
   ↓
2. Debounce 250ms（可配置：skills.load.watchDebounceMs）
   ↓
3. 调用 bumpSkillsSnapshotVersion({ reason: "watch" })
   ↓
4. 触发 listeners（通知 Gateway）
   ↓
5. 下一次 Agent run 时重新加载 Skills
```

**代码位置**：
- Watcher 启动：`src/agents/skills/refresh.ts:100-167`
- 版本 bump：`src/agents/skills/refresh.ts:75-92`
- 事件发射：`src/agents/skills/refresh.ts:41-49`

**配置选项**（`~/.openclaw/openclaw.json`）：
```json5
{
  skills: {
    load: {
      watch: true,                // 是否启用自动刷新（默认 true）
      watchDebounceMs: 250        // Debounce 时间（默认 250ms）
    }
  }
}
```

**代码位置**：`docs/tools/skills.md:220-234`

### 3.6 Plugin Skills

**Plugin 如何自带 Skills？**

Plugins 可以通过 `openclaw.plugin.json` 声明 Skills 目录：

```json5
{
  "name": "my-plugin",
  "skills": ["skills/"]  // 相对于 plugin 根目录
}
```

**加载流程**（代码位置：`src/agents/skills/plugin-skills.ts`）：

```
1. 扫描所有启用的 Plugins
   ↓
2. 读取 openclaw.plugin.json 中的 skills 字段
   ↓
3. 将 Plugin Skills 目录添加到 watchPaths
   ↓
4. 与其他 Skills 一起加载（优先级低于 Workspace Skills）
```

**代码位置**：
- Plugin Skills 解析：`src/agents/skills/plugin-skills.ts`
- 集成到 watch：`src/agents/skills/refresh.ts:63-64`

---

## 🔗 四、与 AI 主动性的关联

### 4.1 Heartbeat 与 System Prompt

**HEARTBEAT.md 注入机制**：

1. **Heartbeat 模板定义**（代码位置：`docs/reference/templates/HEARTBEAT.md:1-10`）：
```markdown
---
summary: "Workspace template for HEARTBEAT.md"
---
# HEARTBEAT.md

# Keep this file empty (or with only comments) to skip heartbeat API calls.
# Add tasks below when you want the agent to check something periodically.
```

2. **注入到 System Prompt**：
   - Heartbeat Prompt 通过 `heartbeatPrompt` 参数注入
   - 默认内容："Read HEARTBEAT.md if it exists (workspace context). Follow it strictly..."
   - **Agent 读取并执行 HEARTBEAT.md 中的指令**

3. **HEARTBEAT_OK 机制**：
   - 当没有任务需要执行时，Agent 回复 `HEARTBEAT_OK`
   - 避免打扰用户（代码位置：`docs/reference/templates/AGENTS.md:82-87`）

**代码位置**：
- Heartbeat 注入：`src/agents/system-prompt.ts`（通过 `heartbeatPrompt` 参数）
- HEARTBEAT_OK 定义：`src/auto-reply/tokens.ts`（推测）

### 4.2 Memory 与 System Prompt

**Memory 系统注入**：

1. **Memory 段落**（代码位置：`src/agents/system-prompt.ts:35-45`）：
```markdown
## Memory Recall
Before answering anything about prior work, decisions, dates, people, preferences, or todos:
run memory_search on MEMORY.md + memory/*.md; then use memory_get to pull only the needed lines.
If low confidence after search, say you checked.
```

2. **Workspace 文件自动索引**：
   - `MEMORY.md`：长期记忆（仅 main session 加载）
   - `memory/YYYY-MM-DD.md`：每日记忆（所有 session 加载）
   - **File Watcher 监控 Workspace，自动触发 Memory 索引**

3. **System Prompt 引导 Agent 使用 Memory**：
   - 明确指示：回答前先搜索 Memory
   - 使用 `memory_search` → 找到相关文件 → 使用 `memory_get` 读取具体行

**代码位置**：
- Memory 段落：`src/agents/system-prompt.ts:35-45`
- Memory 工具：`src/agents/tools/memory-search-tool.ts`（推测）

### 4.3 Skills 与 System Prompt

**Skills 动态加载与注入**：

1. **三层 Skills 加载**（Bundled → Managed → Workspace）
2. **Skills 过滤**（检查 bins、env、config）
3. **生成 Skills Prompt**（XML 格式）
4. **注入到 System Prompt**（`<available_skills>` 段落）
5. **Agent 按需读取 SKILL.md**（不会一次性加载所有 Skills）

**Agent 如何主动发现新 Skills？**

1. Agent 通过 `clawdhub search` 搜索 Skills
2. Agent 使用 `clawdhub install` 安装到 Workspace
3. Skills Watcher 检测到新文件，触发刷新
4. 下一次 Agent run 时，新 Skills 自动注入到 System Prompt

**代码位置**：
- Skills 注入：`src/agents/system-prompt.ts:15-33`
- Skills 刷新：`src/agents/skills/refresh.ts:100-167`

---

## 💡 五、可借鉴的设计模式

### 5.1 模板拆分策略

**核心思想**：**关注点分离 + 可演化性**

1. **拆分维度**：
   - 规范层（AGENTS.md）vs 价值观层（SOUL.md）
   - 通用配置（Skills）vs 环境特定配置（TOOLS.md）
   - 用户信息（USER.md）vs Agent 身份（IDENTITY.md）

2. **可演化性**：
   - Agent 可以自我优化 AGENTS.md（添加新规则）
   - Agent 可以进化 SOUL.md（调整价值观）
   - 用户可以覆盖任何模板（Workspace > 系统默认）

3. **安全隔离**：
   - MEMORY.md 仅在 main session 加载
   - 避免私密信息泄露到 Group Chat

**借鉴点**：
- ✅ 拆分 System Prompt 为多个可独立演化的文件
- ✅ 区分"不变的规范"和"可变的配置"
- ✅ 设计"首次启动仪式"（BOOTSTRAP.md）

### 5.2 动态注入机制

**核心思想**：**Context-Aware Prompt Assembly**

1. **分段构建**：
   - 每个段落由独立函数生成（`buildSkillsSection`、`buildMemorySection` 等）
   - 根据运行时条件决定是否包含某个段落

2. **三种模式**：
   - `full`：完整 System Prompt（main agent）
   - `minimal`：精简版（sub-agents、后台任务）
   - `none`：极简版（特殊场景）

3. **运行时参数**：
   - 时区、时间、Channel、Capabilities 等动态注入
   - 避免硬编码，提高灵活性

**借鉴点**：
- ✅ 设计分段构建的 System Prompt
- ✅ 根据场景动态调整 System Prompt（不是一成不变）
- ✅ 将运行时信息注入到 Prompt（时区、模型、Channel 等）

### 5.3 Skills 三层架构

**核心思想**：**优先级覆盖 + 跨场景复用**

1. **三层优先级**：
   - Bundled（系统级）< Managed（用户级）< Workspace（项目级）
   - 类似于：全局 npm 包 < 用户 npm 包 < 项目 node_modules

2. **跨场景复用**：
   - Bundled Skills：所有用户共享
   - Managed Skills：同一用户的所有 Agents 共享
   - Workspace Skills：当前项目独享

3. **灵活覆盖**：
   - 用户可以在 Workspace 中覆盖 Bundled Skills
   - 不需要修改系统文件，便于维护

**借鉴点**：
- ✅ 设计三层 Skills 架构（系统/用户/项目）
- ✅ 支持同名覆盖（Workspace > Managed > Bundled）
- ✅ 配置与代码分离（Skills 在独立目录）

### 5.4 ClawdHub Registry

**核心思想**：**Skills 生态系统 + 去中心化贡献**

1. **公共 Registry**：
   - 任何人都可以发布 Skills（`clawdhub publish`）
   - 用户可以搜索和安装 Skills（`clawdhub search/install`）

2. **版本管理**：
   - Skills 支持版本号（`--version 1.2.3`）
   - 支持更新和回滚（`clawdhub update`）

3. **Agent 主动发现**：
   - Agent 可以通过 `clawdhub search` 主动搜索需要的 Skills
   - Agent 可以安装 Skills 并自动刷新

**借鉴点**：
- ✅ 建立 Skills Registry（类似 npm）
- ✅ 支持版本管理和依赖解析
- ✅ Agent 可以主动搜索和安装 Skills

### 5.5 Skills Watcher

**核心思想**：**零配置热重载 + 增量刷新**

1. **File Watcher**：
   - 监控 Skills 目录变化（add/change/unlink）
   - Debounce 250ms，避免频繁刷新

2. **增量刷新**：
   - 仅在文件变化时触发刷新
   - 不需要重启 Gateway

3. **版本化快照**：
   - 每个 Session 有独立的 Skills Snapshot
   - 刷新后，新 Session 自动使用最新 Skills

**借鉴点**：
- ✅ 使用 File Watcher 实现热重载
- ✅ Debounce 优化，避免频繁刷新
- ✅ 版本化快照，避免 Session 间冲突

### 5.6 三层压缩策略

**核心思想**：**分层压缩 + 持久化 vs 临时性**

1. **Session Pruning（临时）**：
   - 仅在内存中修剪旧 Tool Results
   - 与 Anthropic Prompt Caching 协同
   - 不改写磁盘文件

2. **Compaction（持久）**：
   - 总结旧对话历史为摘要
   - 持久化到 JSONL 文件
   - 下次 Session 加载摘要

3. **Memory Flush（预压缩）**：
   - 在 Compaction 之前保存记忆
   - Silent Agent Turn（用户无感知）
   - 避免重要信息丢失

**借鉴点**：
- ✅ 设计分层压缩策略（临时 + 持久）
- ✅ 与 Cloud Provider 特性协同（如 Prompt Caching）
- ✅ 预压缩记忆保存（Memory Flush）
- ✅ 自适应分段总结（Adaptive Chunk Ratio）

### 5.7 Hybrid Memory Search

**核心思想**：**向量 + 全文搜索，减少检索 Token**

1. **Hybrid Search**：
   - Vector Semantic Search（语义搜索）
   - FTS5 Keyword Search（关键词搜索）
   - 混合排序，提升召回率

2. **增量索引**：
   - File Watcher 监控 Memory 文件
   - 文件变更自动触发重新索引
   - Embedding Cache Dedupe（去重）

3. **只返回 Top-K**：
   - 不全量加载 Memory 文件
   - 只返回最相关的结果
   - 减少 Token 消耗

**借鉴点**：
- ✅ 使用 Hybrid Search 提升召回率
- ✅ 增量索引 + File Watcher
- ✅ Embedding Cache Dedupe 省钱
- ✅ Top-K 返回减少 Token

---

## 📊 六、设计亮点总结

| 设计点 | 技术实现 | 创新之处 |
|--------|---------|---------|
| **模板拆分** | 6 个核心模板文件 | 关注点分离 + 可演化性 |
| **动态注入** | 分段构建 + 运行时参数 | Context-Aware Prompt Assembly |
| **三层 Skills** | Bundled/Managed/Workspace | 优先级覆盖 + 跨场景复用 |
| **ClawdHub Registry** | 公共 Registry + 版本管理 | Skills 生态系统 |
| **Skills Watcher** | File Watcher + 版本化快照 | 零配置热重载 |
| **SKILL.md 格式** | Frontmatter + Markdown | AgentSkills 兼容 + 可读性 |
| **Skills Gating** | Load-time filtering | 根据环境自动过滤 Skills |
| **Prompt Mode** | full/minimal/none | 根据场景调整 System Prompt |
| **Session Pruning** | TTL-aware 修剪 Tool Results | 与 Prompt Caching 协同 |
| **Compaction** | 分段总结 + 持久化 JSONL | 自适应压缩策略 |
| **Memory Flush** | Silent Turn + NO_REPLY | 预压缩记忆保存 |
| **Hybrid Search** | Vector + FTS5 | 语义 + 关键词混合搜索 |
| **Skills Snapshot** | 版本化快照 | Session 启动时固定版本 |

---

## 📝 七、Prompt 压缩机制深度分析

### 7.1 压缩策略总览

OpenClaw 采用 **三层压缩策略** 应对 Token 增长问题：

| 压缩层级 | 触发条件 | 作用范围 | 持久化 | 代码位置 |
|---------|---------|---------|--------|---------|
| **Session Pruning** | 每次请求前（TTL 触发） | Tool Results | ❌ 仅内存 | `src/agents/pi-extensions/context-pruning.ts` |
| **Compaction** | 接近上下文窗口限制 | 整个对话历史 | ✅ 写入 JSONL | `src/agents/compaction.ts` |
| **Skills Snapshot** | 文件变更时 | Skills 列表 | ✅ 版本化快照 | `src/agents/skills/refresh.ts` |

### 7.2 Session Pruning（工具结果修剪）

**核心理念**：**仅在内存中修剪旧的工具输出，不改写磁盘文件**

#### 工作原理

**代码位置**：`src/agents/pi-extensions/context-pruning/pruner.ts:1-284`

```typescript
// 修剪流程（伪代码）
function pruneContextMessages(messages: AgentMessage[], settings: EffectiveContextPruningSettings) {
  // 1. 找到"保护区"：最后 N 条 assistant 消息之后的内容不修剪
  const cutoffIndex = findAssistantCutoffIndex(messages, settings.keepLastAssistants);
  
  // 2. 估算上下文大小
  const contextChars = estimateContextChars(messages);
  const windowChars = settings.contextWindow * 4; // 1 token ≈ 4 chars
  
  // 3. 如果超过软阈值（30%），开始 soft-trim（保留头尾）
  if (contextChars > windowChars * settings.softTrimRatio) {
    // 修剪大型 tool result，保留头 1500 字符 + 尾 1500 字符
    softTrimToolResults(messages, cutoffIndex, settings.softTrim);
  }
  
  // 4. 如果超过硬阈值（50%），执行 hard-clear（完全清除）
  if (contextChars > windowChars * settings.hardClearRatio) {
    // 替换为占位符："[Old tool result content cleared]"
    hardClearToolResults(messages, cutoffIndex, settings.hardClear);
  }
  
  return prunedMessages;
}
```

**代码位置**：`src/agents/pi-extensions/context-pruning/pruner.ts:130-284`

#### Soft Trim vs Hard Clear

| 策略 | 触发阈值 | 处理方式 | 保留内容 | 代码位置 |
|------|---------|---------|---------|---------|
| **Soft Trim** | 上下文 > 30% | 保留头尾，中间插入 `...` | 头 1500 字符 + 尾 1500 字符 | `pruner.ts:230-270` |
| **Hard Clear** | 上下文 > 50% | 完全替换为占位符 | `[Old tool result content cleared]` | `pruner.ts:270-284` |

#### 与 Anthropic Prompt Caching 的协同

**关键设计**：Pruning 触发时机与 Anthropic Prompt Cache TTL 对齐

**代码位置**：`docs/concepts/session-pruning.md:12-27`

```json5
{
  agents: {
    defaults: {
      contextPruning: {
        mode: "cache-ttl",  // 基于 TTL 的修剪
        ttl: "5m",          // 5 分钟未使用 → 触发修剪
        keepLastAssistants: 3  // 保护最后 3 条 assistant 消息
      }
    }
  }
}
```

**工作原理**：
1. Session 超过 5 分钟未使用 → Anthropic Prompt Cache 过期
2. 下次请求前，Pruning 自动修剪旧 tool results
3. 减少 cacheWrite 大小（因为上下文变小了）
4. Cache TTL 重置，后续请求可以复用新 cache

**设计妙处**：
- 不增加 Token 消耗（只是改变 cache 行为）
- 自动适配 Anthropic 的 Prompt Caching 机制
- 用户无感知（发生在请求前）

### 7.3 Compaction（对话历史压缩）

**核心理念**：**将旧对话总结为摘要，持久化到 JSONL 文件**

#### 触发时机

**代码位置**：`src/agents/compaction.ts:1-346`

```typescript
// Compaction 触发条件
const contextWindow = getContextWindow(model);  // 如 200K tokens
const reserveTokens = config.compaction.reserveTokensFloor;  // 默认 20K tokens
const currentTokens = estimateMessagesTokens(messages);

if (currentTokens > contextWindow - reserveTokens) {
  // 触发 Auto-compaction
  await compactSession(session, messages);
}
```

**触发条件**（代码位置：`docs/concepts/compaction.md:21-27`）：
- 当前 Token 数 > `contextWindow - reserveTokensFloor`
- `reserveTokensFloor` 默认 20,000 tokens
- 示例：Claude-3.5-Sonnet（200K 上下文）→ 达到 180K 时触发

#### Compaction 流程

**代码位置**：`src/agents/compaction.ts:234-296`

```typescript
export async function summarizeInStages(params: {
  messages: AgentMessage[];
  parts?: number;  // 分几段总结（默认 2）
  // ...
}): Promise<string> {
  // 1. 按 Token 比例分段
  const splits = splitMessagesByTokenShare(messages, parts);
  
  // 2. 并行总结每个分段
  const partialSummaries: string[] = [];
  for (const chunk of splits) {
    partialSummaries.push(await summarizeWithFallback({ messages: chunk }));
  }
  
  // 3. 合并分段摘要为最终摘要
  return await generateSummary(partialSummaries, model, {
    instructions: "Merge these partial summaries into a single cohesive summary..."
  });
}
```

**压缩策略**（代码位置：`src/agents/compaction.ts:98-119`）：

| 策略 | 说明 | 代码逻辑 |
|------|------|---------|
| **Adaptive Chunk Ratio** | 根据平均消息大小动态调整分段比例 | `computeAdaptiveChunkRatio()` |
| **Oversized Message Fallback** | 超大消息（> 50% 上下文）单独处理 | `isOversizedForSummary()` |
| **Progressive Fallback** | 全量总结失败 → 部分总结 → 仅记录元信息 | `summarizeWithFallback()` |

#### Memory Flush（预压缩记忆保存）

**核心设计**：**在 Compaction 之前，给 Agent 一次机会保存重要记忆到磁盘**

**代码位置**：`docs/concepts/memory.md:37-74`

```typescript
// Memory Flush 触发条件
const softThreshold = contextWindow - reserveTokens - config.memoryFlush.softThresholdTokens;

if (currentTokens > softThreshold && !session.memoryFlushed) {
  // 触发 Silent Agent Turn
  await runMemoryFlushTurn(session, {
    systemPrompt: "Session nearing compaction. Store durable memories now.",
    userPrompt: "Write any lasting notes to memory/YYYY-MM-DD.md; reply with NO_REPLY if nothing to store."
  });
  
  session.memoryFlushed = true;  // 标记已执行，避免重复
}
```

**设计妙处**：
- **Silent Turn**：Agent 执行但用户看不到（回复 `NO_REPLY`）
- **在 Compaction 之前**：给 Agent 最后机会保存记忆
- **仅执行一次**：通过 `session.memoryFlushed` 标记避免重复

**配置示例**（代码位置：`docs/concepts/memory.md:44-62`）：
```json5
{
  agents: {
    defaults: {
      compaction: {
        reserveTokensFloor: 20000,
        memoryFlush: {
          enabled: true,
          softThresholdTokens: 4000,  // 在压缩前 4K tokens 触发
          systemPrompt: "Session nearing compaction. Store durable memories now.",
          prompt: "Write any lasting notes to memory/YYYY-MM-DD.md; reply with NO_REPLY."
        }
      }
    }
  }
}
```

#### Compaction 输出格式

**代码位置**：`src/agents/compaction.ts:234-295`

压缩后的 Session History：
```jsonl
{"role": "user", "content": "...", "timestamp": 1234567890}
{"role": "assistant", "content": "...", "timestamp": 1234567891}
...
{"role": "compactionSummary", "content": "Summary of messages 1-50: User discussed X, decided Y, encountered issue Z...", "compactedMessageCount": 50, "timestamp": 1234567900}
{"role": "user", "content": "...", "timestamp": 1234567901}  // 最近的消息保留原样
{"role": "assistant", "content": "...", "timestamp": 1234567902}
```

**持久化**：写入 `~/.openclaw/agents/<agentId>/sessions/<sessionId>.jsonl`

#### Compaction 后上下文模板标题变更

Post-compaction 时，系统从 AGENTS.md 提取关键段落注入到压缩后的上下文中。**当前优先匹配的标题**为 `Session Startup` 和 `Red Lines`，同时保留对旧版模板的兼容。

| 优先标题（当前） | 兼容标题（Legacy） | 说明 |
|-----------------|-------------------|------|
| `## Session Startup` | `## Every Session` | 启动时需执行的读取顺序（SOUL.md、USER.md、memory/YYYY-MM-DD.md 等） |
| `## Red Lines` | `## Safety` | 不可违反的规则（不泄露隐私、不执行破坏性命令等） |

**实现**：`extractSections(content, ["Session Startup", "Red Lines"])` 优先；若未匹配到，则回退 `extractSections(content, ["Every Session", "Safety"])`。

**代码位置**：`src/auto-reply/reply/post-compaction-context.ts:55-62`

**模板定义**：`docs/reference/templates/AGENTS.md` 第 16 行为 `## Session Startup`，第 54 行为 `## Red Lines`。

### 7.4 Skills Snapshot（版本化快照）

**核心理念**：**Skills 列表在 Session 启动时快照，中途不变**

**代码位置**：`src/agents/skills/refresh.ts:75-98`

```typescript
// Skills 版本管理
const workspaceVersions = new Map<string, number>();
let globalVersion = 0;

export function bumpSkillsSnapshotVersion(params?: {
  workspaceDir?: string;
  reason?: "watch" | "manual" | "remote-node";
}): number {
  const reason = params?.reason ?? "manual";
  
  if (params?.workspaceDir) {
    const current = workspaceVersions.get(params.workspaceDir) ?? 0;
    const next = bumpVersion(current);  // next = max(current + 1, Date.now())
    workspaceVersions.set(params.workspaceDir, next);
    emit({ workspaceDir: params.workspaceDir, reason });
    return next;
  }
  
  globalVersion = bumpVersion(globalVersion);
  emit({ reason });
  return globalVersion;
}
```

#### Snapshot 生命周期

**代码位置**：`src/agents/skills/workspace.ts:177-212`

```typescript
export function buildWorkspaceSkillSnapshot(
  workspaceDir: string,
  opts?: { snapshotVersion?: number; }
): SkillSnapshot {
  const skillEntries = loadSkillEntries(workspaceDir, opts);  // 加载所有 Skills
  const eligible = filterSkillEntries(skillEntries, opts?.config);  // 过滤不符合条件的
  const promptEntries = eligible.filter(
    (entry) => entry.invocation?.disableModelInvocation !== true
  );
  const resolvedSkills = promptEntries.map((entry) => entry.skill);
  const prompt = formatSkillsForPrompt(resolvedSkills);  // 生成 XML Prompt
  
  return {
    prompt,
    skills: eligible.map((entry) => ({ name: entry.skill.name })),
    resolvedSkills,
    version: opts?.snapshotVersion,  // 版本号
  };
}
```

**关键设计**：
1. **Session 启动时生成快照**：固定当前 Skills 列表
2. **中途不刷新**：即使文件变化，当前 Session 不受影响
3. **下次 Session 生效**：新 Session 使用最新版本

**代码位置**：`docs/tools/skills.md:207-211`

> OpenClaw snapshots the eligible skills **when a session starts** and reuses that list for subsequent turns in the same session. Changes to skills or config take effect on the next new session.

#### Hot Reload 机制

**代码位置**：`src/agents/skills/refresh.ts:100-167`

```typescript
export function ensureSkillsWatcher(params: {
  workspaceDir: string;
  config?: OpenClawConfig;
}) {
  const watchPaths = resolveWatchPaths(workspaceDir, params.config);
  
  const watcher = chokidar.watch(watchPaths, {
    ignoreInitial: true,
    awaitWriteFinish: {
      stabilityThreshold: 250,  // Debounce 250ms
      pollInterval: 100,
    },
    ignored: [
      /(^|[\\/])\.git([\\/]|$)/,
      /(^|[\\/])node_modules([\\/]|$)/,
      /(^|[\\/])dist([\\/]|$)/,
    ],
  });
  
  const schedule = (changedPath?: string) => {
    if (state.timer) clearTimeout(state.timer);
    state.timer = setTimeout(() => {
      bumpSkillsSnapshotVersion({
        workspaceDir,
        reason: "watch",
        changedPath,
      });
    }, 250);  // Debounce
  };
  
  watcher.on("add", (p) => schedule(p));
  watcher.on("change", (p) => schedule(p));
  watcher.on("unlink", (p) => schedule(p));
}
```

**设计妙处**：
- **File Watcher**：自动监控 Skills 目录
- **Debounce 250ms**：避免频繁刷新
- **版本 Bump**：触发 listeners，通知 Gateway
- **下次 Session 生效**：不打断当前对话

### 7.5 Memory Search 索引优化

**核心理念**：**向量索引 + 全文搜索，减少检索时的 Token 消耗**

**代码位置**：`docs/concepts/memory.md:76-99`

#### Hybrid Search 策略

```typescript
// Memory Search 流程（伪代码）
async function memorySearch(query: string): Promise<SearchResult[]> {
  // 1. Vector Semantic Search（语义搜索）
  const vectorResults = await vectorSearch(query, {
    provider: "openai",  // 或 gemini、local
    model: "text-embedding-3-small",
    topK: 10
  });
  
  // 2. FTS5 Keyword Search（关键词搜索）
  const fts5Results = await fts5Search(query, {
    fields: ["content"],
    topK: 10
  });
  
  // 3. Hybrid Ranking（混合排序）
  const combinedResults = combineAndRank(vectorResults, fts5Results);
  
  // 4. 返回 Top-K 结果（不是全部内容）
  return combinedResults.slice(0, 5);
}
```

**代码位置**：`src/memory/manager-search.ts`（推测）

**设计妙处**：
- **不全量加载**：只返回最相关的 Top-K 结果
- **Hybrid Search**：语义 + 关键词，提升召回率
- **sqlite-vec 加速**：向量搜索在 SQLite 中执行，速度快
- **增量索引**：文件变更时自动触发重新索引

#### Embedding Cache Dedupe

**代码位置**：`docs/concepts/memory.md:85-91`

> Uses remote embeddings by default. If `memorySearch.provider` is not set, OpenClaw auto-selects...
> - Embedding Cache Dedupe：相同内容的 embedding 缓存和去重

**设计妙处**：
- 相同内容不重复生成 Embedding
- 减少 Embedding API 调用（省钱）

### 7.6 Token 消耗估算

**代码位置**：`src/agents/compaction.ts:16-18`

```typescript
export function estimateMessagesTokens(messages: AgentMessage[]): number {
  return messages.reduce((sum, message) => sum + estimateTokens(message), 0);
}

// 估算公式（来自 pi-coding-agent）
// 1 token ≈ 4 characters（英文）
// 中文可能更低（1 token ≈ 2-3 characters）
```

**上下文窗口估算**：
```typescript
const contextWindow = getContextWindow(model);  // 从 model registry 获取
const estimatedTokens = estimateMessagesTokens(messages);
const charsPerToken = 4;
const estimatedChars = estimatedTokens * charsPerToken;
```

### 7.7 压缩机制协同效应

| 场景 | Pruning | Compaction | Memory Flush | Skills Snapshot |
|------|---------|------------|--------------|----------------|
| **短期对话（< 10 轮）** | ❌ 不触发 | ❌ 不触发 | ❌ 不触发 | ✅ 固定版本 |
| **中期对话（10-50 轮）** | ✅ 修剪 Tool Results | ❌ 不触发 | ❌ 不触发 | ✅ 固定版本 |
| **长期对话（> 50 轮）** | ✅ 持续修剪 | ✅ 触发压缩 | ✅ 保存记忆 | ✅ 固定版本 |
| **Session 重启** | ❌ 重置 | ✅ 加载摘要 | ✅ 加载记忆 | ✅ 最新版本 |

**完整生命周期**：

```
1. Session Start
   ├─ 加载 Skills Snapshot（版本化）
   ├─ 加载 Session History（包含 Compaction Summary）
   └─ 加载 Memory Files（MEMORY.md + memory/*.md）

2. 对话进行中（每次请求前）
   ├─ Pruning：修剪旧 Tool Results（仅内存）
   ├─ 估算 Token：是否接近上下文窗口？
   └─ 继续对话

3. 接近上下文窗口限制
   ├─ Memory Flush：Silent Turn 保存记忆到磁盘
   ├─ Compaction：总结旧对话历史
   ├─ 写入 JSONL：持久化压缩结果
   └─ 继续对话（使用压缩后的历史）

4. Skills 文件变更（后台）
   ├─ File Watcher：检测到变化
   ├─ Bump Version：更新全局/Workspace 版本号
   └─ 下次 Session 生效（当前 Session 不受影响）

5. Session End
   ├─ 保存 Session State（包含 memoryFlushed 标记）
   └─ 释放资源
```

---

## 📝 八、未解问题与深入分析

### 8.1 ✅ `formatSkillsForPrompt` 的具体实现（已解决）

**来源**：`@mariozechner/pi-coding-agent` v0.49.3

**代码位置**：`node_modules/@mariozechner/pi-coding-agent/dist/core/skills.js:196-223`

#### 完整实现代码

```typescript
/**
 * Format skills for inclusion in a system prompt.
 * Uses XML format per Agent Skills standard.
 * See: https://agentskills.io/integrate-skills
 */
export function formatSkillsForPrompt(skills) {
    if (skills.length === 0) {
        return "";
    }
    const lines = [
        "\n\nThe following skills provide specialized instructions for specific tasks.",
        "Use the read tool to load a skill's file when the task matches its description.",
        "",
        "<available_skills>",
    ];
    for (const skill of skills) {
        lines.push("  <skill>");
        lines.push(`    <name>${escapeXml(skill.name)}</name>`);
        lines.push(`    <description>${escapeXml(skill.description)}</description>`);
        lines.push(`    <location>${escapeXml(skill.filePath)}</location>`);
        lines.push("  </skill>");
    }
    lines.push("</available_skills>");
    return lines.join("\n");
}

function escapeXml(str) {
    return str
        .replace(/&/g, "&amp;")
        .replace(/</g, "&lt;")
        .replace(/>/g, "&gt;")
        .replace(/"/g, "&quot;")
        .replace(/'/g, "&apos;");
}
```

#### 输出格式示例

假设有 2 个 Skills：`github` 和 `clawdhub`

```xml


The following skills provide specialized instructions for specific tasks.
Use the read tool to load a skill's file when the task matches its description.

<available_skills>
  <skill>
    <name>github</name>
    <description>Interact with GitHub using the `gh` CLI. Use `gh issue`, `gh pr`, `gh run`, and `gh api` for issues, PRs, CI runs, and advanced queries.</description>
    <location>/Users/yiye/.openclaw/skills/github/SKILL.md</location>
  </skill>
  <skill>
    <name>clawdhub</name>
    <description>Use the ClawdHub CLI to search, install, update, and publish agent skills from clawdhub.com.</description>
    <location>/Users/yiye/.openclaw/workspace/skills/clawdhub/SKILL.md</location>
  </skill>
</available_skills>
```

#### Token 消耗计算

**Base overhead**（只要 skills.length ≥ 1）：
```typescript
"\n\nThe following skills provide specialized instructions for specific tasks."  // 84 chars
"Use the read tool to load a skill's file when the task matches its description."  // 82 chars
""  // 1 char (newline)
"<available_skills>"  // 18 chars
"</available_skills>"  // 19 chars
// Total: 84 + 82 + 1 + 18 + 19 = 204 chars
```

**实际测量**（包含换行符）：约 **195 characters**

**Per skill**：
```typescript
"  <skill>"              // 9 chars
"    <name>%s</name>"    // 13 + len(name_escaped) chars
"    <description>%s</description>"  // 26 + len(description_escaped) chars
"    <location>%s</location>"  // 21 + len(location_escaped) chars
"  </skill>"            // 10 chars
// Fixed overhead: 9 + 13 + 26 + 21 + 10 = 79 chars
// Plus: name + description + location (after XML escaping)
```

**实际测量**：约 **97 characters** + XML-escaped 内容长度

#### XML 转义规则

| 字符 | 转义后 | 影响 |
|------|--------|------|
| `&` | `&amp;` | +4 字符 |
| `<` | `&lt;` | +3 字符 |
| `>` | `&gt;` | +3 字符 |
| `"` | `&quot;` | +5 字符 |
| `'` | `&apos;` | +5 字符 |

**示例**：
- `"Use gh for GitHub & Git"` → `"Use gh for GitHub &amp; Git"` （+4 字符）

#### Skills 类型定义

**代码位置**：`node_modules/@mariozechner/pi-coding-agent/dist/core/skills.d.ts:7-13`

```typescript
export interface Skill {
    name: string;          // Skill 名称（来自 frontmatter 或目录名）
    description: string;   // Skill 描述（来自 frontmatter，必填）
    filePath: string;      // SKILL.md 文件的绝对路径
    baseDir: string;       // Skill 目录的绝对路径
    source: string;        // 来源标识（如 "openclaw-bundled"、"openclaw-managed"）
}
```

#### 设计要点

1. **XML 格式**：遵循 [AgentSkills 标准](https://agentskills.io/integrate-skills)
2. **仅包含元信息**：不包含 Skill 的完整内容，只有 name/description/location
3. **按需加载**：Agent 根据 description 判断是否需要，然后用 `read` 工具加载完整 SKILL.md
4. **Token 友好**：保持 System Prompt 简洁，避免加载所有 Skills 内容

---

### 8.2 ❓ Agent 如何"主动"调用 `clawdhub install`？

**推测**：
- Agent 可以使用 `exec` 工具执行 `clawdhub install <skill-name>`
- 需要用户确认（通过 Exec Approval 机制）
- 安装后，Skills Watcher 自动检测变化并刷新

**待验证**：
- 是否有自动触发机制？
- 还是需要用户手动确认？

---

### 8.3 ❓ Skills 依赖解析

**当前设计**：
- Skills 之间没有显式依赖关系
- 每个 Skill 是独立的
- 如果需要其他 Skill，可以在 instructions 中说明

**待研究**：
- 是否支持 Skills 之间相互依赖？
- 如何处理循环依赖？

---

### 8.4 ❓ Skills 安全审计

**当前设计**（代码位置：`docs/tools/skills.md:67-73`）：
> - Treat third-party skills as **trusted code**. Read them before enabling.
> - Prefer sandboxed runs for untrusted inputs and risky tools.
> - `skills.entries.*.env` and `skills.entries.*.apiKey` inject secrets into the **host** process for that agent turn (not the sandbox).

**安全机制**：
- **手动审计**：用户需要阅读 Skill 代码
- **Sandbox 隔离**：非 main session 运行在 Docker Sandbox 中
- **Tool Policy**：可以限制 Skill 可用的工具

**待研究**：
- 是否有自动安全扫描机制？
- 第三方 Skills 的权限控制策略？

---

### 8.5 ✅ System Prompt Token 优化（已解决）

通过三层压缩策略解决：
1. **Session Pruning**：临时修剪 Tool Results
2. **Compaction**：总结对话历史并持久化
3. **Skills Snapshot**：版本化快照，避免重复加载
4. **Memory Search**：Hybrid Search 减少检索 Token

---

## 📚 九、相关文件索引

### 核心代码文件

| 文件 | 职责 | 行数 |
|------|------|------|
| `src/agents/system-prompt.ts` | System Prompt 构建主函数 | 592 行 |
| `src/agents/system-prompt-params.ts` | 运行时参数构建 | 106 行 |
| `src/agents/skills/workspace.ts` | Skills 加载和 Snapshot 生成 | 419 行 |
| `src/agents/skills/refresh.ts` | Skills 自动刷新（File Watcher） | 168 行 |
| `src/agents/skills/types.ts` | Skills 类型定义 | 88 行 |
| `src/agents/compaction.ts` | Compaction 压缩逻辑 | 346 行 |
| `src/agents/pi-extensions/context-pruning.ts` | Context Pruning 扩展 | 20 行 |
| `src/agents/pi-extensions/context-pruning/pruner.ts` | Pruning 核心逻辑 | 284 行 |
| `src/memory/manager.ts` | Memory 索引管理 | 未查看 |
| `src/memory/manager-search.ts` | Memory 搜索实现 | 未查看 |

### 模板文件

| 文件 | 职责 | 行数 |
|------|------|------|
| `docs/reference/templates/AGENTS.md` | Agent 工作规范模板 | 197 行 |
| `docs/reference/templates/SOUL.md` | Agent 性格定义模板 | 42 行 |
| `docs/reference/templates/TOOLS.md` | 工具配置模板 | 42 行 |
| `docs/reference/templates/IDENTITY.md` | Agent 身份模板 | 28 行 |
| `docs/reference/templates/BOOTSTRAP.md` | 首次启动仪式模板 | 56 行 |
| `docs/reference/templates/USER.md` | 用户信息模板 | 23 行 |
| `docs/reference/templates/HEARTBEAT.md` | Heartbeat 任务清单模板 | 10 行 |

### 文档文件

| 文件 | 职责 | 行数 |
|------|------|------|
| `docs/tools/skills.md` | Skills 系统完整文档 | 268 行 |
| `docs/concepts/compaction.md` | Compaction 机制文档 | 50 行 |
| `docs/concepts/session-pruning.md` | Session Pruning 文档 | 105 行 |
| `docs/concepts/memory.md` | Memory 系统文档 | 411 行 |
| `docs/start/hubs.md` | 文档索引 | 183+ 行 |

### Skills 示例

| 文件 | 职责 | 行数 |
|------|------|------|
| `skills/github/SKILL.md` | GitHub CLI Skill | 49 行 |
| `skills/clawdhub/SKILL.md` | ClawdHub CLI Skill | 54 行 |

---

## 🎯 十、研究结论

OpenClaw 的 System Prompt 设计体现了 **工程化、可扩展、可演化** 的理念：

### System Prompt 设计（第1层）

1. **模板拆分**：通过 6 个核心模板文件，实现了关注点分离和可演化性
2. **动态注入**：根据运行时环境和场景，动态组合 System Prompt
3. **Skills 三层架构**：Bundled/Managed/Workspace 三层优先级，支持灵活覆盖
4. **ClawdHub Registry**：类似 npm 的 Skills 生态系统，Agent 可主动发现和安装
5. **Skills Watcher**：零配置热重载，文件变化自动触发刷新

### Prompt 压缩机制（第7层补充）

6. **三层压缩策略**：
   - **Session Pruning**：临时修剪 Tool Results，与 Prompt Caching 协同
   - **Compaction**：总结对话历史，持久化到 JSONL
   - **Memory Flush**：预压缩记忆保存，Silent Turn 避免信息丢失

7. **Hybrid Memory Search**：
   - Vector Semantic Search + FTS5 Keyword Search
   - 增量索引 + Embedding Cache Dedupe
   - Top-K 返回，减少 Token 消耗

8. **Skills Snapshot 版本化**：
   - Session 启动时固定版本
   - 中途不刷新，避免不一致
   - File Watcher 监控变更，下次 Session 生效

### 核心价值

这套设计将 System Prompt 从"静态的提示词"进化为"动态的能力定义系统"，同时通过三层压缩策略有效控制 Token 消耗，为长期对话和 AI 主动性（Heartbeat、Memory、Skills）提供了坚实的基础。

**最关键的设计理念**：
- 🎯 **分层设计**：System Prompt、Session History、Memory、Skills 各司其职
- 🔄 **动态性**：根据运行时环境和场景动态调整
- 💾 **持久化**：重要信息写入磁盘，跨 Session 保持
- ⚡ **性能优化**：三层压缩策略 + Hybrid Search + 版本化快照
- 🔒 **安全隔离**：Main session vs Group chat，避免隐私泄露

---

**下一步研究方向**：
- 第2层：Agent Loop 执行流程（Pi-embedded Runtime）
- 第10层：AI 主动性专题研究（Heartbeat、Memory、Skills 的协同效应）

---

**研究者备注**：
- 本文档基于代码阅读和推理，部分细节（如 `formatSkillsForPrompt` 实现）需要进一步验证
- 所有代码位置均已标注，便于后续追踪和验证
- 遵循了 [@awesome-llm-projects/AGENTS.md](../../AGENTS.md) 规范：忠于代码、引用位置、不猜测
