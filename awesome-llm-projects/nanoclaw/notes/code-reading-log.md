# NanoClaw 代码阅读日志

---

## 2026-03-02

**Commit**: `62c25b1` | **版本**: v1.1.6

---

### 会话 1：添加 Submodule + 初步了解

**操作**：
- `git submodule add https://github.com/qwibitai/nanoclaw.git` 添加到 `awesome-llm-projects/nanoclaw/nanoclaw/`
- 查看 README.md（通过 GitHub 页面）、package.json、src/ 目录结构

**主要收获**：
- 项目定位：OpenClaw 的轻量替代，~7,600 行 TypeScript vs OpenClaw ~50 万行
- 核心架构：单进程 Node.js + Docker 容器化 Agent
- 支持 Agent Swarms（Anthropic 实验性特性）
- Skills 体系：15 个 Skills 覆盖 Telegram、Discord、Gmail、Slack 等集成

---

### 会话 2：深度研究（系统性阅读所有核心文件）

按计划阅读顺序：专题5→1→2→3→4→6→7

---

**专题5：数据层（src/db.ts + src/types.ts）**

读取文件：`src/db.ts`（669行）、`src/types.ts`（104行）

关键发现：

1. **Schema 设计**（`db.ts:17-85`）：7 张表，分工清晰。`messages` 表存储完整消息内容，`chats` 表只存元数据（用于 Group 发现，不含敏感内容），两者分开的设计值得注意。

2. **is_bot_message 字段**（`db.ts:87-107`）：通过在线 ALTER TABLE 实现了 schema 迁移。bot 回复消息存入 DB（`is_bot_message=1`），在查询时过滤掉，这样 bot 的输出也被记录（可供 Agent 回顾历史），但不会触发新的处理。

3. **JSON 迁移**（`db.ts:611-669`）：项目曾用 JSON 文件存状态，现已迁移到 SQLite，迁移完成后将原文件重命名为 `.migrated` 保留备份。

4. **RegisteredGroup 类型**（`types.ts:35-42`）：`requiresTrigger` 字段控制是否需要触发词，默认 `true`，solo chat 可以设为 `false`（任何消息都触发）。

---

**专题1：消息流（src/index.ts）**

（前一会话已读，此处补充细节）

关键发现：

1. **Piping 路径**（`index.ts:398-415`）：若目标 Group 已有活跃容器，通过 `queue.sendMessage()` 直接将消息写入容器的 stdin IPC 目录，此时不需要启动新容器，响应更快。

2. **GroupQueue 并发控制**：全局最多 `MAX_CONCURRENT_CONTAINERS`（默认5）个容器并发运行。每个 Group 串行处理（相同 Group 的消息不并发）。

3. **触发词 Pattern**（`config.ts:56-59`）：使用正则 `^@{AssistantName}\b`（`i` flag），大小写不敏感，单词边界匹配（`@AndyBob` 不触发）。

---

**专题2：容器安全（container-runner.ts + container-runtime.ts + mount-security.ts）**

读取文件：`container-runner.ts`（691行）、`container-runtime.ts`（87行）、`mount-security.ts`（419行）

关键发现：

1. **runtime 抽象**（`container-runtime.ts`）：只有 87 行，将 Docker 命令封装为函数（`readonlyMountArgs`、`stopContainer`、`ensureContainerRuntimeRunning`、`cleanupOrphans`）。要切换到 Apple Container 只需修改这一个文件（`CONTAINER_RUNTIME_BIN = 'container'` 即可）。

2. **孤儿容器清理**（`container-runtime.ts:64-87`）：启动时通过 `docker ps --filter name=nanoclaw-` 找到上次崩溃残留的容器并清理。

3. **agent-runner-src 的 per-group 副本**（`container-runner.ts:168-187`）：首次运行时从 `container/agent-runner/src/` 复制到 `data/sessions/{folder}/agent-runner-src/`，容器挂载该目录。这意味着不同 Group 可以有不同版本的 agent-runner，主 Group 可以给自己的 agent-runner 加工具。

4. **Allowlist 安全**（`mount-security.ts`）：allowlist 路径被硬编码到 `~/.config/nanoclaw/` 且不被 mount 进容器，这是关键安全不变量——容器无法读取或修改自己的 mount 权限配置。

5. **OUTPUT_START/END 标记**（`container-runner.ts:29-31`）：为什么不直接用 JSONL？因为 Claude Code 的 stderr 会混入 stdout，哨标记使得解析器可以在混杂输出中精确定位 JSON 数据。

---

**专题3：Agent Runner（container/agent-runner/src/）**

读取文件：`index.ts`（588行）、`ipc-mcp-stdio.ts`（285行）

关键发现：

1. **Skill 同步**（`container-runner.ts:136-146`，配合 `index.ts` 中 Skill 路径）：宿主在每次启动容器前，将 `container/skills/` 目录的 Skills 同步到各 Group 的 `.claude/skills/`，确保容器内 Claude Code 能访问最新的 Skills。

2. **Settings.json 配置**（`container-runner.ts:113-134`）：宿主为每个 Group 创建 `settings.json`，开启：
   - `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`（Agent Swarms）
   - `CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD=1`（多目录 CLAUDE.md 加载）
   - `CLAUDE_CODE_DISABLE_AUTO_MEMORY=0`（启用 memory 功能）

3. **PreCompact Hook**（`agent-runner/index.ts:146-186`）：Claude Code 在压缩对话时触发，将完整对话转成 Markdown 文件存到 `conversations/` 目录，这是 NanoClaw 对话归档的实现方式。

4. **MCP 工具列表**（`ipc-mcp-stdio.ts`）：7个工具：
   - `send_message`：立即发消息给用户（定时任务中必须用这个）
   - `schedule_task`：创建定时任务（含 cron/interval/once 校验）
   - `list_tasks`、`pause_task`、`resume_task`、`cancel_task`：任务管理
   - `register_group`：注册新 WhatsApp Group（仅主 Group）

5. **全局 CLAUDE.md**（`agent-runner/index.ts:394-399`）：非主 Group 的 Agent 会加载 `/workspace/global/CLAUDE.md`（映射到宿主 `groups/global/CLAUDE.md`）作为附加 system prompt，实现跨 Group 共享的全局指令。

---

**专题4：IPC（src/ipc.ts）**

读取文件：`ipc.ts`（387行）

关键发现：

1. **IPC 类型系统**（`ipc.ts:155-387`）：`processTaskIpc()` 用 switch-case 处理 6 种 IPC 类型，每种都有独立的授权检查。`sourceGroup` 参数来自目录路径（不可伪造），`isMain` 也由目录路径推断。

2. **refresh_groups 机制**（`ipc.ts:327-349`）：Agent 可通过 IPC 请求宿主刷新 WhatsApp Group 列表，宿主执行后立即更新 `available_groups.json` 快照，让容器内的 Agent 能看到最新的可注册 Group。

3. **错误处理**（`ipc.ts:95-113`）：处理失败的 IPC 文件移到 `data/ipc/errors/` 目录，不会丢失，可事后检查。

---

**专题6：定时任务（src/task-scheduler.ts）**

读取文件：`task-scheduler.ts`（256行）

关键发现：

1. **容器快速关闭**（`task-scheduler.ts:123-133`）：定时任务是单轮交互，完成后写入 `_close` 信号（`TASK_CLOSE_DELAY_MS=10000`，10s 后），而不是等默认的 30 分钟 IDLE_TIMEOUT。这避免了容器长期占用资源。

2. **nextRun 时机**（`task-scheduler.ts:195-212`）：nextRun 在**任务运行后立即计算**（而非任务开始前），cron 任务的下次运行时间是以**任务完成时**为基准，不累积偏移。`once` 任务没有 nextRun，status 变为 `completed`。

3. **防抖机制**（`task-scheduler.ts:233-237`）：`getTaskById()` 再次检查状态，防止在调度器轮询期间任务被用户取消或暂停但仍被执行。

---

**专题7：Skills 体系**

读取文件：`setup/SKILL.md`（部分）、`add-telegram/SKILL.md`（部分）、`add-telegram/manifest.yaml`

关键发现：

1. **Skills 是"可执行的文档"**：SKILL.md 不是普通文档，是给 Claude Code 的精确指令，规定了每个决策点的行为、用户交互方式（`AskUserQuestion`）、错误处理策略。

2. **manifest.yaml 的机器可读部分**：`structured` 字段包含 npm 依赖、env 变量等，供 `apply-skill.ts` 脚本自动处理，不依赖 LLM 来安装依赖。

3. **`add-telegram` 的"Replace or Alongside"设计**：Telegram 可以替换 WhatsApp 或同时运行，由 `TELEGRAM_ONLY` 环境变量控制，这个选项在 SKILL.md 的 Phase 1 阶段通过 `AskUserQuestion` 收集。

4. **`.nanoclaw/state.yaml` 幂等性**：记录已应用的 Skills，`apply-skill.ts` 读取此文件跳过已应用的，保证幂等。

---

### 待阅读

- `container/Dockerfile` + `container/agent-runner/` 的 entrypoint.sh（了解容器构建细节）
- `skills-engine/migrate.ts`（三路合并的具体实现）
- `src/group-queue.ts`（并发控制的详细逻辑）
- `src/channels/whatsapp.ts`（WhatsApp Baileys 集成细节）
- `docs/SDK_DEEP_DIVE.md`（官方对 SDK 使用的深入说明）
