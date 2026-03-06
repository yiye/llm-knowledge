# CoPaw 代码阅读日志

> 研究日期：2026-03-05
> Commit：`d7415fa3c2fa5fb34628a96397f8250057d6c792`（v0.0.5-beta.2）

---

## 2026-03-05

### 第一轮：项目整体结构扫描

**阅读内容**：目录结构探索（`ls` + `find`）

**主要内容**：
- `src/copaw/` 下 11 个顶层模块
- 核心模块：`agents/`（557行 react_agent + 885行 skills_manager）、`app/runner/`（266行）
- `agents/md_files/en/` 下有 6 个 MD 文件模板（SOUL/PROFILE/BOOTSTRAP/MEMORY/HEARTBEAT + AGENTS.md）
- `local_models/backends/` 下有 llama.cpp 和 MLX 两个后端
- 测试文件：`tests/` 目录，11 个 test_*.py 文件

**关键发现**：
- Agent 名字硬编码为 `"Friday"`（`react_agent.py:128`）
- MemoryManager 实际是 `ReMeCopaw` 的薄封装，核心逻辑在外部 `reme` 包中

---

### 第二轮：Agent 核心层精读

**阅读文件**：

| 文件 | 行数 | 关键行号 |
|------|------|---------|
| `src/copaw/agents/react_agent.py` | 557 | 55-62（normalize_tool_choice）、75-158（__init__）、308-335（_register_hooks）、339-354（rebuild_sys_prompt）、565-616（reply/override） |
| `src/copaw/agents/prompt.py` | 173 | 22-30（PromptConfig.FILE_ORDER）、45-110（_load_file）、114-141（build）、145-172（build_system_prompt_from_working_dir） |
| `src/copaw/agents/hooks/bootstrap.py` | 104 | 44-125（__call__，触发逻辑） |
| `src/copaw/agents/hooks/memory_compaction.py` | 195 | 33-48（__init__，参数）、67-190（__call__，压缩判断） |
| `src/copaw/agents/md_files/en/SOUL.md` | 41 | 全文（人格宣言） |
| `src/copaw/agents/md_files/en/PROFILE.md` | 32 | 全文（身份模板） |
| `src/copaw/agents/md_files/en/BOOTSTRAP.md` | 48 | 全文（首次引导仪式） |
| `src/copaw/agents/md_files/en/MEMORY.md` | 26 | 全文（长期记忆模板） |
| `src/copaw/agents/md_files/en/HEARTBEAT.md` | 12 | 全文（心跳任务模板） |

**关键发现**：

1. `normalize_reasoning_tool_choice`（`react_agent.py:55-62`）：标准化 `tool_choice` 为 `"auto"`，统一不同 Provider 的默认行为
2. `_build_sys_prompt` 调用在 `super().__init__` **之前**（`react_agent.py:116-120`），但 `_setup_memory_manager` 调用在之后（`react_agent.py:133-138`），依赖顺序严格
3. Bootstrap hook 用 `is_first_user_interaction(messages)` 判断（待研究：这个函数定义在 `agents/utils.py`，未完整阅读）
4. BootstrapHook 的 `prepend_to_message_content(msg, bootstrap_guidance)` 将引导文字插入到 user 消息内容前面，而非作为独立消息
5. MemoryCompaction 中的 `_MemoryMark.COMPRESSED` 来自 `agentscope.agent._react_agent`，说明 AgentScope 内部已有记忆压缩标记机制，CoPaw 在此基础上做上层控制
6. `check_valid_messages`（`memory_compaction.py:122-125`）调整 keep_recent 数量，确保保留消息集合是有效的（待研究：具体校验规则）

**遗留问题**：
- [ ] `agents/utils.py` 中 `is_first_user_interaction` 和 `check_valid_messages` 的具体实现
- [ ] `agents/command_handler.py` 中 `SYSTEM_COMMANDS` 的完整列表

---

### 第三轮：Skills 系统精读

**阅读文件**：

| 文件 | 行数 | 关键行号 |
|------|------|---------|
| `src/copaw/agents/skills_manager.py` | 885 | 55-75（目录路径函数）、120-136（_collect_skills_from_dir）、139-226（sync_skills_to_working_dir）、229-278（变更检测）、529-578（SkillService.list_all_skills）、595-749（SkillService.create_skill）、752-873（enable/disable/delete）、901-1050（load_skill_file + 安全防护） |

**关键发现**：

1. `list_available_skills`（`skills_manager.py:352-370`）只看 `active/` 目录：遍历子目录，只要有 `SKILL.md` 就算一个 Skill，**不读文件内容**，只用文件名。
2. `_collect_skills_from_dir` 同样只检查 `SKILL.md` 是否存在（`skills_manager.py:134`），不解析内容。
3. 创建 Skill（`create_skill`，`skills_manager.py:595-749`）时才用 `frontmatter.loads` 解析内容并校验 frontmatter。运行时加载（`toolkit.register_agent_skill`）由 AgentScope 负责解析。
4. `SkillService.list_all_skills` 触发了一次反向同步（`active → customized`，`skills_manager.py:549-562`），每次列出所有 Skill 时都会尝试将用户的修改持久化到 customized。
5. `extra_files` 参数（`create_skill`，`skills_manager.py:601`）：支持在 Skill 根目录放额外文件，主要用于从 Hub 导入含 runtime assets 的 Skill。

**遗留问题**：
- [ ] `toolkit.register_agent_skill(str(skill_dir))` 的实现在 AgentScope 中，未阅读其如何解析 SKILL.md
- [ ] `skills_hub.py` 中 Hub 导入的具体实现（URL 下载 + 解析流程）

---

### 第四轮：Runner / Session / 命令分发精读

**阅读文件**：

| 文件 | 行数 | 关键行号 |
|------|------|---------|
| `src/copaw/app/runner/runner.py` | 266 | 36-50（AgentRunner.__init__）、60-214（query_handler）、217-231（init_handler）、237-281（shutdown_handler） |
| `src/copaw/app/runner/session.py` | 51 | 15-17（正则）、19-27（sanitize_filename）、30-50（SafeJSONSession） |
| `src/copaw/app/runner/command_dispatch.py` | 173 | 28-43（_get_last_user_text）、46-59（_is_command）、62-73（_LightweightSessionAgent）、75-173（run_command_path） |

**关键发现**：

1. `query_handler`（`runner.py:60`）是 async generator，用 `yield msg, last` 流式推送结果。外层 FastAPI 路由将其包装为 SSE 或 WebSocket 流。
2. `env_context`（`runner.py:98-103`）由 `build_env_context` 构建，包含 session_id、user_id、channel、working_dir，作为 System Prompt 的前缀注入（`react_agent.py:101,244-246`），让 Agent 知道自己在哪个 Channel 被调用。
3. `session.load_session_state` 可能因 schema 不匹配抛 `KeyError`（`runner.py:157-162`），被 `except` 静默处理并继续。这是一个向后兼容处理——老 session 文件可能缺字段。
4. Daemon 命令（`/restart`）先 `yield` 一个提示消息（`"Restart in progress..."`），再执行重启回调。这样用户不会看到"服务无响应"的假死状态。
5. `_LightweightSessionAgent.state_dict` 和 `load_state_dict` 只序列化/反序列化 `memory`，与完整 `CoPawAgent` 的 state_dict 不兼容（后者还包含 toolkit 状态等）。因此命令路径不能复用正常路径的 session 文件结构。

**遗留问题**：
- [ ] `daemon_commands.py` 中 `parse_daemon_query` 的完整命令集合
- [ ] `build_env_context`（`runner/utils.py`）的具体实现，以及 env_context 的格式

---

### 第五轮：Provider / 兼容层 / 本地模型 / MCP 精读

**阅读文件**：

| 文件 | 行数 | 关键行号 |
|------|------|---------|
| `src/copaw/providers/registry.py` | 371 | 23-195（Provider 定义）、185-195（PROVIDERS dict）、198（ID 校验正则）、245-300（custom provider CRUD）、303-348（sync_local/ollama_models） |
| `src/copaw/providers/openai_chat_model_compat.py` | 177 | 16-19（_clone_with_overrides）、22-69（_sanitize_tool_call）、72-120（_sanitize_chunk）、136-167（_SanitizedStream）、170-186（OpenAIChatModelCompat） |
| `src/copaw/local_models/backends/llamacpp_backend.py` | 149 | 16-44（_normalize_messages）、47-89（__init__）、92-120（chat_completion）、123-140（chat_completion_stream） |
| `src/copaw/local_models/backends/mlx_backend.py` | 234 | 18-37（_normalize_messages）、40-50（_resolve_model_dir）、53-81（__init__）、84-96（_build_prompt）、99-184（chat_completion）、187-243（chat_completion_stream） |
| `src/copaw/app/mcp/manager.py` | 217 | 22-37（MCPClientManager.__init__）、38-57（init_from_config）、60-74（get_clients）、77-134（replace_client）、136-170（close_all）、197-230（_build_client） |

**关键发现**：

1. `PROVIDER_ALIYUN_CODINGPLAN`（`registry.py:94-100`）：专门的阿里云 Coding Plan Provider，base_url 为 `coding.dashscope.aliyuncs.com`，这是 CoPaw 面向国内用户的差异化配置。
2. `sync_local_models`（`registry.py:303-322`）：动态从本地模型 manifest 更新 llamacpp/mlx 的模型列表，每次有新模型下载时刷新。
3. `_clone_with_overrides`（`openai_chat_model_compat.py:16-19`）：用 `SimpleNamespace` + `__dict__` 做轻量对象克隆，是这个文件中的核心工具函数，被大量复用。
4. MLX 的 `_resolve_model_dir`（`mlx_backend.py:40-50`）：MLX 模型是目录格式（safetensors + config），但 `LocalModelInfo.local_path` 可能指向目录内的某个文件（如权重文件），需要转换为父目录。
5. MLX `_build_prompt`（`mlx_backend.py:84-96`）：用 `tokenizer.apply_chat_template` 生成 prompt 字符串，工具调用通过 `getattr(self._tokenizer, "has_tool_calling", False)` 检测 tokenizer 是否支持。
6. MCP manager 在 `_build_client` 中给每个 client 设置 `_copaw_rebuild_info`（私有约定属性），Agent 的 `_rebuild_mcp_client` 通过 `getattr(client, "_copaw_rebuild_info", None)` 读取。这是一个协议约定，不依赖接口定义。

**遗留问题**：
- [ ] `app/mcp/watcher.py`：MCP 配置变更监听的具体实现（如何触发 `replace_client`）
- [ ] `src/copaw/app/channels/base.py`：Channel 基类接口定义，各 Channel 如何实现消息收发
- [ ] `src/copaw/app/channels/renderer.py`：消息渲染逻辑，Markdown 如何适配各平台格式
- [ ] `src/copaw/tunnel/cloudflare.py`：Cloudflare tunnel 自动管理的具体实现
- [ ] `src/copaw/config/watcher.py`：配置热监听的实现

---

## 待研究清单

### 高优先级

| 文件 | 原因 |
|------|------|
| `src/copaw/app/channels/base.py` | Channel 抽象接口，了解多通道架构的核心设计 |
| `src/copaw/app/channels/renderer.py` | 消息渲染适配，影响各平台用户体验 |
| `src/copaw/agents/utils.py` | `is_first_user_interaction`、`check_valid_messages` 等工具函数 |
| `src/copaw/agents/command_handler.py` | `SYSTEM_COMMANDS` 完整列表和处理逻辑 |

### 中优先级

| 文件 | 原因 |
|------|------|
| `src/copaw/app/mcp/watcher.py` | MCP 热更新的触发机制 |
| `src/copaw/app/runner/daemon_commands.py` | Daemon 命令集合 |
| `src/copaw/agents/skills_hub.py` | Hub 远程导入实现 |
| `src/copaw/config/watcher.py` | 配置热监听 |

### 低优先级

| 文件 | 原因 |
|------|------|
| `src/copaw/tunnel/cloudflare.py` | Cloudflare tunnel 管理 |
| `src/copaw/app/crons/` | 定时任务实现 |
| `tests/` | 测试用例，了解边界条件 |
