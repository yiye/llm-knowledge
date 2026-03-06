# CoPaw 架构分析

> 基于 commit `d7415fa`，研究日期 2026-03-05

---

## 一、Agent 核心层

### 1.1 CoPawAgent 初始化序列

源文件：`src/copaw/agents/react_agent.py:75-158`

`CoPawAgent` 继承 AgentScope 的 `ReActAgent`，初始化顺序严格有依赖：

```
__init__() 入口
    │
    ├─ 1. _create_toolkit()          # 创建 Toolkit，注册 8 个内置工具
    │      execute_shell_command / read_file / write_file / edit_file
    │      browser_use / desktop_screenshot / send_file_to_user / get_current_time
    │
    ├─ 2. _register_skills(toolkit)  # 从 active_skills/ 加载所有 Skills
    │      ensure_skills_initialized() → list_available_skills()
    │      → toolkit.register_agent_skill(skill_dir)
    │
    ├─ 3. _build_sys_prompt()        # 从工作目录读取 MD 文件组装 System Prompt
    │      build_system_prompt_from_working_dir()
    │      → 顺序：AGENTS.md（必须）→ SOUL.md（必须）→ PROFILE.md（可选）
    │
    ├─ 4. create_model_and_formatter() # 从 providers.json 配置创建 LLM 实例
    │
    ├─ 5. super().__init__(           # 调用 ReActAgent 初始化
    │         name="Friday",          # Agent 固定名称
    │         model=model,
    │         sys_prompt=sys_prompt,
    │         toolkit=toolkit,
    │         memory=InMemoryMemory(),
    │         formatter=formatter,
    │         max_iters=50,           # 默认最大推理迭代次数
    │     )
    │
    ├─ 6. _setup_memory_manager()    # 配置记忆管理器
    │      - 检查 ENABLE_MEMORY_MANAGER 环境变量
    │      - 若启用：将 memory_manager.chat_model/formatter 更新为当前模型
    │      - 注册 memory_search 工具到 toolkit
    │      - 将 memory 替换为 memory_manager.get_in_memory_memory()
    │
    ├─ 7. CommandHandler()           # 初始化命令处理器（/compact、/new 等）
    │
    └─ 8. _register_hooks()          # 注册 pre_reasoning 钩子
           BootstrapHook     → hook_type="pre_reasoning"
           MemoryCompactionHook → hook_type="pre_reasoning"（仅在 memory_manager 启用时）
```

关键设计：步骤 6 在 `super().__init__` 之后，因此可以访问已初始化的 `self.memory`、`self.toolkit`，并将 memory 整体替换为 ReMe 管理的版本。

---

### 1.2 System Prompt 组装机制

源文件：`src/copaw/agents/prompt.py:22-172`

#### 文件加载顺序

```python
# prompt.py:26-30
FILE_ORDER = [
    ("AGENTS.md", True),   # required — 工作目录规范、工具使用指南
    ("SOUL.md", True),     # required — Agent 人格与价值观
    ("PROFILE.md", False), # optional — Agent 身份 + 用户画像
]
```

每次加载均从 `WORKING_DIR`（`~/.copaw/`）读取，因此用户可以直接编辑这些文件来改变 Agent 行为。

#### 拼接格式

```python
# prompt.py:82-88
# 每个文件以 `# {filename}` 作为节标题，内容用双换行分隔
self.prompt_parts.append(f"# {filename}")
self.prompt_parts.append("")
self.prompt_parts.append(content)
```

最终 System Prompt 结构（示例）：

```
# AGENTS.md

...(内容)...

# SOUL.md

...(内容)...

# PROFILE.md

...(内容)...
```

#### YAML Frontmatter 自动剥离

```python
# prompt.py:76-79
if content.startswith("---"):
    parts = content.split("---", 2)
    if len(parts) >= 3:
        content = parts[2].strip()
```

MD 文件里的 YAML frontmatter（`---` 包裹的元数据）会被自动去掉，只把正文内容放入 System Prompt。

#### 热更新机制

每次请求处理时（`runner.py:169`），Runner 都会调用 `agent.rebuild_sys_prompt()`，从磁盘重新读取最新的 MD 文件。这意味着用户修改 `AGENTS.md` 后，**下一次对话就立即生效**，无需重启服务。

```python
# runner.py:169
agent.rebuild_sys_prompt()
```

`rebuild_sys_prompt` 同时更新 `self._sys_prompt` 和内存中已存在的 system 消息（`react_agent.py:339-354`），确保会话历史中的 system 消息也同步更新。

---

### 1.3 MD 模板文件说明

这些文件是 `src/copaw/agents/md_files/en/` 下的模板，随包安装时复制到 `~/.copaw/`。

#### SOUL.md — Agent 灵魂（哲学宣言）

```
_You're not a chatbot. You're becoming someone._

Core Truths:
- Be genuinely helpful, not performatively helpful.
- Have opinions. You're allowed to disagree.
- Be resourceful before asking.
- Earn trust through competence.
- Remember you're a guest.

Boundaries:
- Private things stay private.
- When in doubt, ask before acting externally.
- Never send half-baked replies to messaging surfaces.
```

这个文件定义 Agent 的底层价值观和边界——不是功能说明，而是"性格设定"。强调 Agent 应该有个性、有判断力，而非纯粹服从型助手。

#### PROFILE.md — 身份与用户画像（模板）

初始为空模板（有 Identity 和 User Profile 两个 section），由 **Bootstrap 流程**通过 Agent 与用户对话后填写：

```markdown
## Identity
- Name: (pick something you like)
- Creature: (AI? robot? familiar?)
- Vibe: (sharp? warm? chaotic?)

## User Profile
- Name:
- Timezone:
- Notes:
```

#### BOOTSTRAP.md — 首次启动仪式

```
_You just woke up. Time to figure out who you are._

The Conversation:
1. Your name — What should they call you?
2. Your nature — What kind of creature are you?
3. Your vibe — Formal? Casual? Snarky? Warm?

After You Know Who You Are:
- Update PROFILE.md
- Talk through SOUL.md with user

When You're Done:
- Delete this file (BOOTSTRAP.md). You don't need a bootstrap script anymore.
```

这个文件驱动首次对话引导流程，完成后必须自行删除。

#### MEMORY.md — 长期记忆模板

用于存放 Agent 特有的配置知识（SSH 主机别名、用户习惯等），区别于每日日志：

```markdown
## Tool Setup
Add whatever helps you do your job. This is your cheat sheet.
Examples:
- SSH hosts: home-server → 192.168.1.100
```

#### HEARTBEAT.md — 心跳任务模板

文件为空时跳过心跳 API 调用，有内容时按内容定期执行：

```
# Keep this file empty to skip heartbeat API calls.
# Add tasks below when you want the agent to check something periodically.
```

---

### 1.4 Hook 机制

源文件：`src/copaw/agents/hooks/bootstrap.py`、`hooks/memory_compaction.py`

Hook 通过 AgentScope 的 `register_instance_hook` 注册到 `pre_reasoning` 阶段（`react_agent.py:315-334`），每次推理前按顺序执行。

#### BootstrapHook（首次对话引导）

**触发逻辑**（`bootstrap.py:61-107`）：

```python
# 三重检查，全部满足才触发
bootstrap_path = working_dir / "BOOTSTRAP.md"
bootstrap_completed_flag = working_dir / ".bootstrap_completed"

if bootstrap_completed_flag.exists():  # 已触发过 → 跳过
    return None
if not bootstrap_path.exists():        # 文件不存在 → 跳过
    return None
messages = await agent.memory.get_memory()
if not is_first_user_interaction(messages):  # 不是第一条用户消息 → 跳过
    return None
```

**触发后行为**：
1. 调用 `build_bootstrap_guidance(language)` 生成引导文字（支持 en/zh）
2. 将引导文字 **prepend** 到第一条 user 消息前（不修改后续消息）
3. `touch .bootstrap_completed`（touchfile 而非数据库，简单可靠）

**一次性设计的关键**：`.bootstrap_completed` 文件的存在就是触发标记。即使 BOOTSTRAP.md 还在（Agent 还未删除它），也不会再次触发引导。这样 Bootstrap 引导和 BOOTSTRAP.md 的生命周期解耦。

#### MemoryCompactionHook（上下文压缩）

**触发逻辑**（`memory_compaction.py:95-190`）：

```python
# 1. 获取所有非 COMPRESSED 消息
messages = await agent.memory.get_memory(
    exclude_mark=_MemoryMark.COMPRESSED,
    prepend_summary=False,
)

# 2. 拆分 system prompt 消息和剩余消息
system_prompt_messages = [m for m in messages if m.role == "system"]
remaining_messages = messages[len(system_prompt_messages):]

# 3. 若剩余消息不超过 keep_recent，直接跳过
if len(remaining_messages) <= self.keep_recent:
    return None

# 4. 计算 token 数：可压缩消息 + 已有 summary
# estimated_total_tokens = 消息 token 数 + previous_summary token 数

# 5. 超过阈值才触发（异步，不阻塞当前请求）
if estimated_total_tokens > self.memory_compact_threshold:
    self.memory_manager.add_async_summary_task(messages=messages_to_compact)
```

**压缩阈值计算**（`react_agent.py:106-111`）：

```python
# max_input_length 默认 128K token
self._memory_compact_threshold = int(max_input_length * MEMORY_COMPACT_RATIO)
```

**消息分区**（保留最近 `keep_recent` 条，压缩更早的）：

```
[System Prompt(s)] | [可压缩消息...] | [保留最近 10 条]
                          ↑ 压缩这部分
```

**异步设计**：`add_async_summary_task` 将压缩任务提交到后台队列，不阻塞当前对话的推理过程。压缩完成后，下次请求的 BootstrapHook 检测到 token 数下降，自然不再触发。

---

## 二、Skills 系统

### 2.1 三目录模型

源文件：`src/copaw/agents/skills_manager.py:55-75`

```
src/copaw/agents/skills/          # builtin — 随包发布的内置 Skills
~/.copaw/customized_skills/       # customized — 用户自定义/修改的 Skills
~/.copaw/active_skills/           # active — 运行时实际加载的 Skills
```

**所有权关系**：

| 目录 | 谁写 | 谁读 |
|------|------|------|
| builtin | 开发者（发版时包含） | sync 脚本 |
| customized | 用户（直接编辑或通过控制台） | sync 脚本 |
| active | sync 脚本自动维护 | CoPawAgent 启动时 |

### 2.2 同步策略

源文件：`src/copaw/agents/skills_manager.py:139-226`

```python
# sync_skills_to_working_dir() 的核心逻辑
skills_to_sync = _collect_skills_from_dir(builtin_skills)   # 先收集内置
skills_to_sync.update(_collect_skills_from_dir(customized_skills))  # customized 同名覆盖内置
# → 最终 active = builtin ∪ customized（customized 优先）
```

**同步规则**：

- `force=False`（默认）：active 中已存在的 Skill **不被覆盖**
- `force=True`：先 `rmtree` 再 `copytree`，强制更新

**反向同步**（active → customized）：

`sync_skills_from_active_to_customized()` 用 `filecmp.dircmp` 递归比较目录内容（`skills_manager.py:229-278`），只有在 active 中的 Skill 与 builtin 版本**内容不同**时，才将其保存到 customized，避免将未修改的内置 Skill 污染 customized 目录。

### 2.3 SKILL.md 校验

源文件：`src/copaw/agents/skills_manager.py:653-677`

创建 Skill 时（`SkillService.create_skill`），必须有 YAML frontmatter 且包含 `name` 和 `description`：

```python
post = frontmatter.loads(content)
skill_name = post.get("name", None)
skill_description = post.get("description", None)
if not skill_name or not skill_description:
    return False  # 创建失败
```

标准 SKILL.md 结构：

```markdown
---
name: my_skill
description: 我的自定义能力说明
---

# 使用说明
本 Skill 用于……
```

### 2.4 Skill 目录结构与安全防护

每个 Skill 目录结构：

```
<skill_name>/
├── SKILL.md          # 必须（能力描述 + 使用规范）
├── references/       # 可选（文档参考资料）
│   └── *.md / *.txt
└── scripts/          # 可选（可执行脚本）
    └── *.py / *.sh
```

`load_skill_file` 有路径遍历防护（`skills_manager.py:950-973`）：

```python
normalized = file_path.replace("\\", "/")
# file_path 必须以 references/ 或 scripts/ 开头
if not (normalized.startswith("references/") or normalized.startswith("scripts/")):
    return None
# 禁止 .. 和绝对路径
if ".." in normalized or normalized.startswith("/"):
    return None
```

---

## 三、Runner / Session / 命令分发

### 3.1 请求处理完整流程

源文件：`src/copaw/app/runner/runner.py:60-214`

```
HTTP 请求到达 FastAPI
    │
    ▼
AgentRunner.query_handler(msgs, request)
    │
    ├─ 前置命令检测（command_dispatch.py）
    │   query.startswith("/") → _is_command() ?
    │   ├─ YES → run_command_path()  ──────────────────┐
    │   │         daemon 命令（/restart 等）             │ 轻量路径：
    │   │         conversation 命令（/compact /new 等）   │ 不创建 CoPawAgent
    │   │         yield (msg, True) → return            │ 直接返回结果
    │   └─ NO ↓                                        ┘
    │
    ├─ 构建 env_context（session_id、user_id、channel、working_dir）
    │
    ├─ 获取 MCP clients（来自 MCPClientManager，每次请求实时获取）
    │
    ├─ 读取 config（max_iters、max_input_length）
    │
    ├─ 创建 CoPawAgent(env_context, mcp_clients, memory_manager, ...)
    │   agent.set_console_output_enabled(enabled=False)
    │
    ├─ await agent.register_mcp_clients()  # 注册 MCP 工具
    │
    ├─ await session.load_session_state(session_id, user_id, agent)
    │   # 从 JSON 文件恢复对话历史
    │
    ├─ agent.rebuild_sys_prompt()
    │   # 总是从磁盘重读 AGENTS.md/SOUL.md/PROFILE.md
    │   # 确保用户修改文件后立即生效
    │
    ├─ async for msg, last in stream_printing_messages(
    │       agents=[agent],
    │       coroutine_task=agent(msgs),   # 启动 ReAct 主循环
    │   ):
    │       yield msg, last               # 流式推送给 Channel
    │
    └─ finally:
        await session.save_session_state(session_id, user_id, agent)
        await chat_manager.update_chat(chat)
```

### 3.2 MemoryManager 是 Runner 级单例

源文件：`src/copaw/app/runner/runner.py:238-268`

```python
# init_handler 中创建一次（服务启动时）
self.memory_manager = MemoryManager(
    working_dir=str(WORKING_DIR),
    chat_model=chat_model,
    formatter=formatter,
    ...
)
await self.memory_manager.start()
```

每次 `query_handler` 创建 `CoPawAgent` 时，将同一个 `memory_manager` 实例传入：

```python
agent = CoPawAgent(
    ...
    memory_manager=self.memory_manager,  # 共享单例
)
```

**单例的含义**：所有 session 的向量索引、BM25 索引、摘要队列都共享同一个实例。记忆系统是跨 session 的全局资源，而不是 per-session 的。这使得 Agent 在不同对话中可以共享长期记忆。

### 3.3 轻量命令路径（不创建 Agent）

源文件：`src/copaw/app/runner/command_dispatch.py`

命令分为两类：

| 命令类型 | 示例 | 处理方式 |
|---------|------|---------|
| Daemon 命令 | `/restart` | `DaemonCommandHandlerMixin` |
| Conversation 命令 | `/compact`、`/new` | `CommandHandler.SYSTEM_COMMANDS` |

两类命令都走轻量路径：只创建 `_LightweightSessionAgent`（只含 memory），不创建完整的 `CoPawAgent`，避免：
- 加载 Skills（磁盘 IO）
- 初始化 LLM 实例（冷启动延迟）
- 注册 MCP clients（网络连接）

```python
# command_dispatch.py:62-73
class _LightweightSessionAgent:
    """Minimal agent-like object for session load/save (memory only)."""
    def __init__(self, memory: CoPawInMemoryMemory) -> None:
        self.memory = memory

    def state_dict(self) -> dict:
        return {"memory": self.memory.state_dict()}
```

### 3.4 Session 持久化与跨平台兼容

源文件：`src/copaw/app/runner/session.py`

`SafeJSONSession` 继承 AgentScope 的 `JSONSession`，只覆写 `_get_save_path`：

```python
# session.py:16,38-50
_UNSAFE_FILENAME_RE = re.compile(r'[\\/:*?"<>|]')

def _get_save_path(self, session_id: str, user_id: str) -> str:
    safe_sid = sanitize_filename(session_id)   # "discord:dm:12345" → "discord--dm--12345"
    safe_uid = sanitize_filename(user_id) if user_id else ""
    if safe_uid:
        file_path = f"{safe_uid}_{safe_sid}.json"
    else:
        file_path = f"{safe_sid}.json"
    return os.path.join(self.save_dir, file_path)
```

替换字符集：`\ / : * ? " < > |`（Windows 文件名非法字符）→ `--`

这是一个最小化改动的跨平台适配：不修改任何业务逻辑，只在文件名生成时做字符替换。
