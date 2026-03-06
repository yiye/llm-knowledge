# NanoClaw 架构分析

- **版本**：v1.1.6，commit `62c25b1`
- **研究日期**：2026-03-02

---

## 1. 整体架构

NanoClaw 是单进程 Node.js 应用，主进程负责 WhatsApp 收发消息、状态管理、调度；AI 推理完全在容器内运行。

```
┌─────────────────────────────────────────────────────────────┐
│  宿主进程 (Node.js)                                          │
│                                                             │
│  WhatsApp ──► SQLite ──► 轮询循环(2s) ──► GroupQueue        │
│                                                ↓            │
│  IPC Watcher(1s) ◄── /data/ipc/ ◄── ContainerRunner        │
│  TaskScheduler(60s) ────────────────────────── ↑            │
└─────────────────────────────────────────────────────────────┘
                              │ docker run -i
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  容器 (每 Group 独立)                                        │
│                                                             │
│  agent-runner/index.ts                                      │
│    ├─ 读 stdin (ContainerInput + secrets)                   │
│    ├─ 调用 Claude Agent SDK query()                         │
│    ├─ MCP Server (ipc-mcp-stdio.ts)                         │
│    │    └─ 写 /workspace/ipc/{messages,tasks}/              │
│    └─ 写 stdout (OUTPUT_START/END 标记)                     │
└─────────────────────────────────────────────────────────────┘
```

## 2. 数据库 Schema（store/messages.db）

来自 `src/db.ts:17-85`：

### 表结构

| 表名 | 主键 | 核心字段 | 用途 |
|------|------|---------|------|
| `chats` | `jid` | name, last_message_time, channel, is_group | 已知对话列表，用于 Group 发现 |
| `messages` | `(id, chat_jid)` | sender, content, timestamp, is_bot_message | 消息历史，不存储 bot 自身回复 |
| `scheduled_tasks` | `id` | group_folder, chat_jid, prompt, schedule_type, schedule_value, context_mode, next_run, status | 定时任务定义 |
| `task_run_logs` | `AUTOINCREMENT` | task_id FK, run_at, duration_ms, status, result, error | 任务执行历史 |
| `router_state` | `key` | value | KV 状态持久化（游标等） |
| `sessions` | `group_folder` | session_id | Claude 会话 ID（跨启动恢复） |
| `registered_groups` | `jid` | name, folder UNIQUE, trigger_pattern, container_config, requires_trigger | 已注册的 Group 配置 |

### 关键索引

- `messages(timestamp)` —— 消息游标查询的核心索引（`src/db.ts:38`）
- `scheduled_tasks(next_run)` —— 调度器扫描到期任务（`src/db.ts:53`）
- `scheduled_tasks(status)` —— 过滤 active 任务（`src/db.ts:54`）

### 双游标设计

在 `router_state` 表中维护两个关键游标：

```
last_timestamp             → 所有已"看到"的消息时间戳
last_agent_timestamp[jid]  → 各 Group 最后一次被 Agent 实际处理的消息时间戳
```

两者差值即"已看到但未处理"的消息，在崩溃恢复时重新入队（`src/index.ts:429-441`）。

## 3. 消息处理流程（专题1）

来自 `src/index.ts`：

```
startMessageLoop() —— 每 2s 轮询
  │
  ├─ getNewMessages(registeredJids, lastTimestamp)
  │    └─ SELECT ... WHERE timestamp > lastTimestamp AND jid IN (...)
  │         AND is_bot_message=0
  │
  ├─ 立即推进 lastTimestamp（防止重读）
  │
  ├─ 按 Group 分组消息
  │
  ├─ 触发词过滤（非主 Group 必须包含 @AssistantName）
  │
  ├─ 若 Group 容器活跃 → queue.sendMessage() 通过 stdin pipe 传入
  │                       （piping path，lastAgentTimestamp 同步推进）
  │
  └─ 否则 → queue.enqueueMessageCheck(chatJid)
              └─ processGroupMessages()
                   ├─ getMessagesSince(chatJid, lastAgentTimestamp[jid])
                   ├─ formatMessages() 格式化为文本
                   ├─ 推进 lastAgentTimestamp[jid]（乐观更新）
                   ├─ runAgent() → runContainerAgent()
                   │    └─ 失败 → 回滚 lastAgentTimestamp（src/index.ts:235）
                   └─ 成功 → 保持推进
```

**崩溃恢复**（`src/index.ts:429-441`）：

启动时 `recoverPendingMessages()` 扫描所有 registered groups，若 `lastTimestamp > lastAgentTimestamp[jid]`，则将该 Group 重新入队。

## 4. 容器安全隔离（专题2）

来自 `src/container-runner.ts` 和 `src/mount-security.ts`。

### 挂载策略（buildVolumeMounts）

`src/container-runner.ts:57-200`：

| 挂载 | 主 Group | 非主 Group | 权限 |
|------|---------|-----------|------|
| 项目根目录 `/workspace/project` | ✅ | ❌ | 只读 |
| 自己的 `groups/{folder}/` → `/workspace/group` | ✅ | ✅ | 读写 |
| `groups/global/` → `/workspace/global` | ✅ | ✅ | 只读 |
| `data/sessions/{folder}/.claude/` → `/home/node/.claude` | ✅ | ✅ | 读写（per-group 隔离） |
| `data/ipc/{folder}/` → `/workspace/ipc` | ✅ | ✅ | 读写（per-group 命名空间） |
| `data/sessions/{folder}/agent-runner-src/` → `/app/src` | ✅ | ✅ | 读写（per-group 可定制） |
| `/workspace/extra/*` | 可选 | 可选 | 按 allowlist |

**关键安全设计**：
- 主 Group 项目根目录只读，防止 Agent 修改宿主代码（绕过沙盒）
- 每 Group 的 `.claude/` 目录完全隔离，防止跨 Group session 访问
- IPC 目录 per-group 命名空间，防止跨 Group 权限提升

### Secrets 安全传输

来自 `src/container-runner.ts:301-306`：

```typescript
// Secrets 只通过 stdin 传入容器，从不写磁盘，从不 mount
container.stdin.write(JSON.stringify(input));  // input.secrets 包含 API Key
container.stdin.end();
delete input.secrets;  // 立即从内存中删除，不出现在日志
```

容器内（`container/agent-runner/src/index.ts:513-516`）将 secrets 合并进 SDK 专用的 env 副本，不写入 `process.env` 本身，这样 Bash 子进程无法访问 secrets。

### mount-security.ts 额外防线

allowlist 存储在 `~/.config/nanoclaw/mount-allowlist.json`（项目根目录**外**），容器无法 mount 该文件并修改，从而无法扩大自身权限。默认屏蔽模式包含 `.ssh`、`.gnupg`、`.aws`、`.kube`、`.docker` 等（`src/mount-security.ts:29-47`）。

### 流式输出协议

```
---NANOCLAW_OUTPUT_START---
{"status":"success","result":"...","newSessionId":"..."}
---NANOCLAW_OUTPUT_END---
```

宿主进程（`src/container-runner.ts:309-363`）边接收 stdout 边解析标记对，实现流式传输，每个标记对解析后立即触发 `onOutput` 回调发送消息给用户。

## 5. Agent Runner 内部（专题3）

来自 `container/agent-runner/src/index.ts`。

### MessageStream：保活 AsyncIterable

`index.ts:66-96`：实现了一个 push-based AsyncIterable，解决了 Claude Agent SDK 的 `isSingleUserTurn` 问题。只要 stream 不调用 `end()`，SDK 就认为会话仍在等待用户输入，可以接收后续 IPC 消息。

### Claude SDK 调用参数

`index.ts:417-457`：

- `cwd: '/workspace/group'`——Agent 工作目录，读取该目录的 CLAUDE.md
- `additionalDirectories`——扫描 `/workspace/extra/*/` 的 CLAUDE.md
- `resume`——session ID，恢复历史会话
- `allowedTools`——明确白名单：Bash、文件操作、Web、Agent Swarms、MCP 工具
- `permissionMode: 'bypassPermissions'`——跳过权限弹窗（安全由容器隔离保证）
- `mcpServers.nanoclaw`——为每次 query 启动一个 stdio MCP 进程

### 两个 Hooks

1. `PreCompact`：在 Claude Code 压缩会话前，将完整对话归档到 `/workspace/group/conversations/` 目录（`index.ts:146-186`）
2. `PreToolUse(Bash)`：在每次 Bash 调用前，prepend `unset ANTHROPIC_API_KEY CLAUDE_CODE_OAUTH_TOKEN`，防止 secrets 泄漏到子进程（`index.ts:193-210`）

### 查询循环

`index.ts:539-575`：

```
while (true):
  runQuery(prompt, sessionId)     ← 含 IPC polling
  writeOutput(sessionUpdateMarker)
  nextMessage = waitForIpcMessage()  ← 轮询 _close 或新消息文件
  if nextMessage == null: break   ← _close 信号，退出容器
  prompt = nextMessage
  continue
```

## 6. IPC 通信机制（专题4）

**设计原则**：用文件系统替代 socket/pipe，天然支持容器边界穿越，且可审计。

### 目录结构

```
data/ipc/
  {groupFolder}/
    messages/    ← 容器写入消息文件 → 宿主读取并发送给用户
    tasks/       ← 容器写入任务操作文件 → 宿主读取并执行
    input/       ← 宿主写入新消息文件 → 容器读取并处理
    current_tasks.json      ← 宿主写入任务快照 → 容器读取
    available_groups.json   ← 宿主写入 Group 列表 → 容器读取
```

### 权限边界（src/ipc.ts）

`src/ipc.ts` 宿主侧在处理 IPC 文件时严格校验：
- **发送消息**：非主 Group 只能发给自己的 `chatJid`（`ipc.ts:77-90`）
- **schedule_task**：非主 Group 只能为自己创建任务（`ipc.ts:203-209`）
- **pause/resume/cancel_task**：只能操作自己 Group 的任务（`ipc.ts:276-325`）
- **register_group/refresh_groups**：仅主 Group 可操作（`ipc.ts:329-382`）

容器内 MCP 服务器（`ipc-mcp-stdio.ts`）的授权是通过 env vars（`NANOCLAW_IS_MAIN`、`NANOCLAW_GROUP_FOLDER`）实现初步校验，但最终授权发生在宿主侧（防止容器篡改环境变量）。

### 原子写入

`container/agent-runner/src/ipc-mcp-stdio.ts:30-33`：

```typescript
const tempPath = `${filepath}.tmp`;
fs.writeFileSync(tempPath, JSON.stringify(data, null, 2));
fs.renameSync(tempPath, filepath);  // 原子操作，防止宿主读到半写文件
```

## 7. 定时任务调度（专题6）

来自 `src/task-scheduler.ts`。

### 调度流程

```
startSchedulerLoop() —— 每 60s 轮询
  ├─ getDueTasks()：SELECT WHERE status='active' AND next_run <= now()
  ├─ 再次 getTaskById() 确认未被并发修改
  └─ queue.enqueueTask() → runTask()
       ├─ 找到该 task 的 RegisteredGroup 对象
       ├─ context_mode='group' → 使用现有 sessionId
       ├─ context_mode='isolated' → sessionId=undefined（新会话）
       ├─ runContainerAgent()（与消息处理相同路径）
       ├─ TASK_CLOSE_DELAY_MS=10s 后关闭容器（src/task-scheduler.ts:124）
       └─ updateTaskAfterRun()：计算 nextRun，写 task_run_logs
```

### nextRun 计算

```typescript
// cron：解析表达式取下一个触发时间
const interval = CronExpressionParser.parse(task.schedule_value, { tz: TIMEZONE });
nextRun = interval.next().toISOString();

// interval：当前时间 + ms
nextRun = new Date(Date.now() + ms).toISOString();

// once：nextRun=null，updateTaskAfterRun 中 status 变为 'completed'
```

## 8. Skills 扩展体系（专题7）

来自 `.claude/skills/` 目录和 `skills-engine/`。

### Skill 文件结构

```
.claude/skills/{name}/
  SKILL.md         ← Claude Code 读取的指令文件（如何使用这个 Skill）
  manifest.yaml    ← 机器可读的 Skill 元数据
  add/             ← 新增文件（直接复制到项目）
    src/channels/telegram.ts
    src/channels/telegram.test.ts
  modify/          ← 被修改文件的基准版本（用于三路合并）
    src/index.ts
    src/index.ts.intent.md  ← 描述改动意图和不变量
    src/config.ts
  tests/           ← Skill 测试文件
```

### manifest.yaml 示例（add-telegram）

`manifest.yaml` 字段：
```yaml
skill: telegram
version: 1.0.0
adds:
  - src/channels/telegram.ts
  - src/channels/telegram.test.ts
modifies:
  - src/index.ts
  - src/config.ts
structured:
  npm_dependencies:
    grammy: "^1.39.3"
  env_additions:
    - TELEGRAM_BOT_TOKEN
test: "npx vitest run src/channels/telegram.test.ts"
```

### 三路合并机制

Skills Engine（`skills-engine/migrate.ts`）使用三路合并确保 Skill 可以应用于用户已修改的代码库：
- **base**：`modify/` 中的基准版本（Skill 编写时的原始代码）
- **theirs**：Skill 的修改版本（Skill 想要的结果）
- **ours**：用户的当前代码（可能已经有其他修改）
- **result**：合并结果，保留用户的改动

`.nanoclaw/state.yaml` 记录已应用的 Skills，防止重复应用。
