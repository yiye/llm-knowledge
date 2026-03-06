# NanoClaw 核心发现与工程亮点

- **版本**：v1.1.6，commit `62c25b1`
- **研究日期**：2026-03-02

---

## 1. 双游标防止消息丢失

**位置**：`src/index.ts:48-52`、`src/db.ts:292-341`

NanoClaw 维护两个独立游标解决"处理中崩溃"问题：

```
lastTimestamp          → 已"看到"的最新消息时间戳（全局）
lastAgentTimestamp[jid] → 各 Group 最后成功处理的时间戳
```

工作流（`src/index.ts:346-421`）：
1. 发现新消息 → 立即推进 `lastTimestamp`（防止重读）
2. 推进 `lastAgentTimestamp[jid]`（乐观更新）
3. 若 Agent 报错且未发送任何输出 → 回滚 `lastAgentTimestamp[jid]`（`src/index.ts:235`）
4. 启动时 `recoverPendingMessages()` 检测两游标差值，重新入队

**意义**：实现了"至少一次"（at-least-once）交付语义，在系统崩溃或容器出错时不丢消息。输出已发送时不回滚，防止重复发送。

---

## 2. 文件系统 IPC：简单但强大的跨边界通信

**位置**：`src/ipc.ts`、`container/agent-runner/src/ipc-mcp-stdio.ts`

跨越容器边界传递数据，NanoClaw 选择文件系统而非 socket：

```
容器写文件 → 宿主轮询读文件 → 删除文件（消费语义）
```

三个目录各有职责（`src/ipc.ts:62-63`）：
- `ipc/{group}/messages/` ← 容器 → 宿主：发给用户的消息
- `ipc/{group}/tasks/` ← 容器 → 宿主：任务创建/修改指令
- `ipc/{group}/input/` ← 宿主 → 容器：新用户消息、`_close` 关闭信号

**授权模型**：
- 源 Group 的身份由**目录路径**决定，不由容器自申报
- 宿主侧（`src/ipc.ts:77-382`）对每个操作独立校验权限
- 即使容器被攻破修改 env vars，宿主校验仍生效

**原子写入**（`ipc-mcp-stdio.ts:30-33`）：写临时文件再 rename，防止宿主读到半写文件。

---

## 3. MessageStream：保活 SDK 会话的精妙设计

**位置**：`container/agent-runner/src/index.ts:66-96`

Claude Agent SDK 的 `query()` 接受 AsyncIterable 作为 prompt，当 iterable 结束时 SDK 进入单轮（`isSingleUserTurn=true`）模式，Agent Swarms 子 agent 无法完成任务。

`MessageStream` 的设计：保持 AsyncIterable 不结束，直到收到 `_close` 信号：

```typescript
class MessageStream {
  push(text: string): void { /* 队列中加消息，唤醒等待者 */ }
  end(): void { /* 结束 iterable */ }
  async *[Symbol.asyncIterator](): AsyncGenerator<...> {
    while (true) {
      while (this.queue.length > 0) yield this.queue.shift()!;
      if (this.done) return;
      await new Promise<void>(r => { this.waiting = r; });  // 等待 push() 或 end()
    }
  }
}
```

同时在查询运行期间轮询 IPC input 目录（`index.ts:371-387`），将新消息 `push()` 进 stream，实现"热路径"消息注入（不需要重启容器）。

---

## 4. Secrets 的三重防护

**位置**：`src/container-runner.ts:301-306`、`container/agent-runner/src/index.ts:511-516`、`index.ts:193-210`

API 密钥的传递路径设计精细：

**第一重：不写磁盘，不 mount**（`container-runner.ts:302-305`）：
```typescript
container.stdin.write(JSON.stringify(input));  // secrets 在 input 中
container.stdin.end();
delete input.secrets;  // 立即清除，不进日志
```

**第二重：不进 process.env**（`agent-runner/index.ts:513-516`）：
```typescript
const sdkEnv: Record<string, string | undefined> = { ...process.env };
for (const [key, value] of Object.entries(containerInput.secrets || {})) {
  sdkEnv[key] = value;  // 只在 SDK 的 env 参数中，不写入 process.env
}
```

**第三重：Bash 子进程剥离**（`agent-runner/index.ts:199`）：
```typescript
const unsetPrefix = `unset ANTHROPIC_API_KEY CLAUDE_CODE_OAUTH_TOKEN 2>/dev/null; `;
// 每次 Bash 调用前 prepend 这段，确保子进程看不到 secrets
```

---

## 5. 容器 agent-runner 的每 Group 可定制性

**位置**：`src/container-runner.ts:168-187`

每个 Group 有自己的 `agent-runner-src` 副本（挂载在容器 `/app/src`）：

```typescript
// 首次运行时从 container/agent-runner/src/ 复制到 data/sessions/{folder}/agent-runner-src/
if (!fs.existsSync(groupAgentRunnerDir) && fs.existsSync(agentRunnerSrc)) {
  fs.cpSync(agentRunnerSrc, groupAgentRunnerDir, { recursive: true });
}
```

这意味着：
- 不同 Group 可以有不同的 agent-runner 实现（不同工具集、不同行为）
- 主 Group 可以修改其 agent-runner 而不影响其他 Group
- 容器启动时重新编译 TypeScript（通过 `entrypoint.sh`）

---

## 6. Skills Engine 三路合并：安全地修改 fork

**位置**：`.claude/skills/add-telegram/manifest.yaml`、`skills-engine/`

传统的"插件系统"面临的问题：用户修改了代码后，插件更新破坏用户的改动。Skills Engine 的解法：

1. `add/`：新文件，直接添加，无冲突风险
2. `modify/`：包含改动的**基准版本**（Skill 编写时的原始代码）和**目标版本**（Skill 要改成什么）
3. 三路合并：base（基准）+ ours（用户当前代码）+ theirs（Skill 目标）= result

`modify/src/index.ts.intent.md` 额外提供语义描述，供 Claude Code 在合并冲突时理解改动意图。

这是一种"AI-native 的插件系统"——不依赖固定的钩子点，而是教会 AI 如何安全地理解和修改代码。

---

## 7. 调度任务的 context_mode：一个被精心设计的参数

**位置**：`src/types.ts:62`、`src/task-scheduler.ts:117-119`

```typescript
context_mode: 'group' | 'isolated'
```

- `group`：任务使用该 Group 的现有 Claude 会话 session ID，能访问聊天历史和 memory
- `isolated`：`sessionId=undefined`，开新会话，完全独立

MCP 工具的文档（`ipc-mcp-stdio.ts:70-79`）明确指导 Agent 如何选择：
- 需要历史上下文 → `group`
- 独立任务（prompt 中自带所有信息）→ `isolated`

这个设计让定时任务既可以是真正的定时问答（有上下文），也可以是无状态的机器人任务（干净独立）。

---

## 8. 主 Group 的特权边界

NanoClaw 有一个特殊的"主 Group"（默认 folder 为 `main`），相当于管理员控制台：

| 能力 | 主 Group | 普通 Group |
|------|---------|-----------|
| 注册新 Group | ✅ | ❌ |
| 读取项目根目录 | ✅（只读） | ❌ |
| 刷新 Group 列表 | ✅ | ❌ |
| 为其他 Group 创建任务 | ✅ | ❌ |
| 查看所有 Group 的任务 | ✅ | 仅自己 |
| 向其他 Group 发消息 | ✅ | ❌ |
| 查看可用 Group 列表 | ✅ | 空列表 |

这种设计让主频道（自我聊天）成为安全的管理入口，而不需要单独的 web UI 或命令行工具。

---

## 9. 待研究的问题

以下问题在当前代码阅读中未找到明确答案：

1. **`skills-engine/migrate.ts`** 的三路合并具体实现——是否用了 `diff3`、`immer` 或自研算法？（待读 `skills-engine/` 目录）
2. **`container/Dockerfile`** 的具体内容——容器基础镜像、Claude Code 的安装方式、`entrypoint.sh` 如何编译 TypeScript？（待读）
3. **Agent Swarms 的消息路由**——当 `TeamCreate` 创建子 agent 时，子 agent 也需要通过 MCP 发消息吗？MCP 进程是否被子 agent 继承？（`ipc-mcp-stdio.ts` 注释中提到"Standalone process that agent teams subagents can inherit"，但具体机制待验证）
