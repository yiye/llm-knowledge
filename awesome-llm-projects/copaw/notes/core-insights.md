# CoPaw 核心工程亮点

> 基于 commit `d7415fa`，研究日期 2026-03-05
> 本文聚焦于代码中值得学习和借鉴的工程实践。

---

## 1. System Prompt 即文件（可热更新的 Agent 配置）

**位置**：`src/copaw/agents/prompt.py`、`src/copaw/app/runner/runner.py:169`

**设计思路**：System Prompt 不是硬编码字符串，而是从 `~/.copaw/` 目录下的三个 MD 文件动态拼接而来：

```python
FILE_ORDER = [
    ("AGENTS.md", True),   # 工作流程和规范
    ("SOUL.md", True),     # 人格和价值观
    ("PROFILE.md", False), # 身份和用户画像（用户自定义）
]
```

每次请求时调用 `agent.rebuild_sys_prompt()` 重新从磁盘读取，**用户修改文件后下一条消息即生效**，无需重启。

**可借鉴点**：
- Agent 配置文件化（配置与代码分离）
- 分层设计：系统级（AGENTS.md）+ 价值观层（SOUL.md）+ 个性化层（PROFILE.md）
- 可选文件（optional）不阻断 prompt 构建，降低配置门槛

---

## 2. Bootstrap 首次引导的一次性触发设计

**位置**：`src/copaw/agents/hooks/bootstrap.py:61-107`

**问题**：如何确保首次引导只触发一次，且与 BOOTSTRAP.md 文件的生命周期解耦？

**方案**：使用 touchfile（`.bootstrap_completed`）而非数据库标记：

```python
bootstrap_completed_flag = working_dir / ".bootstrap_completed"
if bootstrap_completed_flag.exists():
    return None  # 已触发过，跳过
# ... 触发引导 ...
bootstrap_completed_flag.touch()  # 标记为已完成
```

**优点**：
- 无需数据库或 KV 存储
- 文件系统的存在/不存在本身就是 boolean 状态
- 删除文件可以手动重置引导（容易 debug）
- `.bootstrap_completed` 与 `BOOTSTRAP.md` 生命周期独立：Agent 删除 BOOTSTRAP.md 后不会再次触发，标记文件持久存在

---

## 3. 异步上下文压缩（不阻塞当前对话）

**位置**：`src/copaw/agents/hooks/memory_compaction.py:188`

**问题**：上下文窗口满时，压缩操作（LLM 调用生成摘要）耗时较长，若同步执行会阻塞用户响应。

**方案**：只检测，不等待：

```python
if estimated_total_tokens > self.memory_compact_threshold:
    self.memory_manager.add_async_summary_task(
        messages=messages_to_compact,
    )
    # 注意：这里直接 return None，不 await
```

将压缩任务提交到后台队列，当前请求继续处理。压缩结果写入每日日志，下次请求时被纳入 summary token 计数。

**可借鉴点**：
- 检测与执行解耦
- 上下文管理不阻塞主流程
- "最终一致性"而非"强一致性"——当前对话可能略微超出预期 token，但几轮后自然收敛

---

## 4. OpenAI Stream 工具调用的防御性解析

**位置**：`src/copaw/providers/openai_chat_model_compat.py`

**问题**：不同 LLM Provider（Qwen、Ollama、DeepSeek 等）通过 OpenAI 兼容接口返回的流式 tool_call chunk 格式不完全一致，部分 Provider 会在某些 chunk 中返回 `name=None`、`arguments=None` 或非字符串类型，导致下游解析崩溃。

**方案**：`_SanitizedStream` 代理流，逐 chunk 修复：

```python
class _SanitizedStream:
    """Proxy OpenAI async stream that sanitizes each emitted item."""
    async def __anext__(self) -> Any:
        item = await self._ctx_stream.__anext__()
        return _sanitize_stream_item(item)  # 每个 chunk 都过一遍
```

`_sanitize_tool_call` 的修复逻辑：

```python
# None → 空字符串
safe_name = raw_name if isinstance(raw_name, str) else str(raw_name or "")
safe_arguments = raw_arguments if isinstance(raw_arguments, str) else json.dumps(raw_arguments)
# 重建 function 对象
safe_function = SimpleNamespace(name=safe_name, arguments=safe_arguments)
```

**可借鉴点**：
- 在适配层（而非业务层）做格式修复，业务代码保持干净
- 用 `SimpleNamespace` 轻量克隆对象，避免深拷贝开销
- 只修改有问题的部分（`if sanitized is not tool_call: changed = True`），减少不必要的对象创建

---

## 5. Provider 注册表的统一抽象

**位置**：`src/copaw/providers/registry.py`

**设计**：9 个内置 Provider 都以 `ProviderDefinition` 结构描述，通过全局 `PROVIDERS` 字典管理：

```python
PROVIDERS: dict[str, ProviderDefinition] = {
    "modelscope": PROVIDER_MODELSCOPE,
    "dashscope": PROVIDER_DASHSCOPE,
    "openai": PROVIDER_OPENAI,
    "anthropic": PROVIDER_ANTHROPIC,
    "ollama": PROVIDER_OLLAMA,
    "llamacpp": PROVIDER_LLAMACPP,
    "mlx": PROVIDER_MLX,
    # ...
}
```

所有 Provider 默认使用 `OpenAIChatModelCompat`（OpenAI 兼容接口），只有 Anthropic 使用专用的 `AnthropicChatModel`：

```python
_CHAT_MODEL_MAP: dict[str, Type[ChatModelBase]] = {
    "OpenAIChatModel": OpenAIChatModelCompat,
}
if AnthropicChatModel is not None:
    _CHAT_MODEL_MAP["AnthropicChatModel"] = AnthropicChatModel
```

**自定义 Provider 支持**（`registry.py:257-284`）：
- `validate_custom_provider_id`：ID 格式校验（小写字母开头，只含字母/数字/连字符/下划线）
- `register_custom_provider` / `unregister_custom_provider`：运行时动态注入
- `sync_custom_providers`：从持久化配置同步到内存注册表

**可借鉴点**：
- 内置与自定义 Provider 统一管理接口
- ID 格式防护（防止与内置 ID 冲突）
- 运行时动态注册，无需重启

---

## 6. MCP 热更新的双阶段锁设计

**位置**：`src/copaw/app/mcp/manager.py:77-134`

**问题**：替换 MCP client 时，连接新 client 是耗时网络操作；若全程持锁，会阻塞所有并发请求的 `get_clients()` 调用。

**方案**：两阶段操作：

```python
async def replace_client(self, key, client_config, timeout=60.0):
    # Phase 1: 在锁外连接新 client（可能耗时 60s）
    new_client = self._build_client(client_config)
    await asyncio.wait_for(new_client.connect(), timeout=timeout)

    # Phase 2: 在锁内完成 swap（极短时间）
    async with self._lock:
        old_client = self._clients.get(key)
        self._clients[key] = new_client   # 原子替换

    # 锁外关闭旧 client（不阻塞读取）
    if old_client is not None:
        await old_client.close()
```

**可借鉴点**：
- "连接在锁外，swap 在锁内"——最小化锁的持有时间
- 读写锁的异步实现（`asyncio.Lock`）
- 关闭旧连接也在锁外，避免死锁

---

## 7. MCP Client 重建信息的附加元数据

**位置**：`src/copaw/app/mcp/manager.py:199-230`、`src/copaw/agents/react_agent.py:525-556`

**问题**：Agent 在会话期间若 MCP client 断线（网络抖动），如何自动重连？

**方案**：创建 client 时，将构建参数存到 client 对象的私有属性上：

```python
# mcp/manager.py:219
setattr(client, "_copaw_rebuild_info", rebuild_info)
# rebuild_info 包含：transport、name、url/command/args/env/cwd
```

Agent 检测到 `ClosedResourceError` 时（`react_agent.py:425-558`），调用 `_recover_mcp_client`：

```python
def _rebuild_mcp_client(client) -> Any | None:
    rebuild_info = getattr(client, "_copaw_rebuild_info", None)
    if not isinstance(rebuild_info, dict):
        return None
    # 根据 transport 类型重建
    if rebuild_info["transport"] == "stdio":
        return StdIOStatefulClient(...)
    return HttpStatefulClient(...)
```

重建后还有一个巧妙的引用稳定性处理：

```python
@staticmethod
def _reuse_shared_client_reference(original_client, rebuilt_client):
    """Keep manager-shared client reference stable after rebuild."""
    # 将 rebuilt_client 的 __dict__ 更新到 original_client 上
    # 这样 manager 持有的引用不变，但内部状态已更新
    original_dict.update(rebuilt_dict)
    return original_client
```

**可借鉴点**：
- 把重建信息附加在对象本身上（self-describing objects）
- 引用稳定性：in-place 更新而非替换引用，避免需要通知所有持有者

---

## 8. 本地模型后端的消息归一化

**位置**：`src/copaw/local_models/backends/llamacpp_backend.py:16-44`、`mlx_backend.py:18-37`

**问题**：AgentScope formatter 生成的消息中，`content` 可能是 list（多模态块）或 `None`，但 llama.cpp 和 MLX 的 chat template 只接受字符串类型的 content。

**方案**：两个后端都实现了 `_normalize_messages`：

```python
def _normalize_messages(messages: list[dict]) -> list[dict]:
    for msg in messages:
        content = msg.get("content")
        if isinstance(content, list):
            # [{"type": "text", "text": "..."}, ...] → 拼接为字符串
            parts = [block.get("text", "") for block in content if isinstance(block, dict)]
            msg = {**msg, "content": "\n".join(parts)}
        elif content is None:
            msg = {**msg, "content": ""}
    return out
```

llama.cpp 还额外处理了 `tool_calls=None` 的问题（Jinja 模板对 None 迭代会崩溃）：

```python
if "tool_calls" in msg and not msg["tool_calls"]:
    msg = {k: v for k, v in msg.items() if k != "tool_calls"}  # 删除字段
```

**可借鉴点**：
- 适配层在最靠近第三方库的地方做格式转换，业务层不感知
- 使用字典推导式安全删除字段（不 mutate 原始 dict）

---

## 9. MLX 结构化输出的 Prompt 注入

**位置**：`src/copaw/local_models/backends/mlx_backend.py:135-143`

**问题**：MLX（Apple Silicon 本地模型）没有原生的结构化输出（JSON mode）支持。

**方案**：Prompt 注入 JSON schema 约束：

```python
if structured_model:
    schema = structured_model.model_json_schema()
    prompt += (
        f"\nRespond with a JSON object matching this schema: "
        f"{schema}\n"
    )
```

**局限**：这是尽力而为（best-effort）的方案，模型不保证遵守。适合对格式要求不严格的场景。

**可借鉴点**：降级策略——无法原生支持时，用 prompt engineering 兜底，而不是直接报错。

---

## 10. Skills 同名覆盖的变更检测优化

**位置**：`src/copaw/agents/skills_manager.py:229-278`

**问题**：active → customized 的反向同步时，若内置 Skill 未被修改就不应存到 customized（会污染用户目录）。

**方案**：用 `filecmp.dircmp` 递归比较目录内容：

```python
def _is_directory_same(dir1: Path, dir2: Path) -> bool:
    dcmp = filecmp.dircmp(dir1, dir2)
    if dcmp.left_only or dcmp.right_only or dcmp.funny_files:
        return False
    if dcmp.diff_files:
        return False
    for sub_dcmp in dcmp.subdirs.values():
        if not _compare_dircmp(sub_dcmp):
            return False
    return True
```

只有在 active 中的 Skill 内容与内置版本不同时，才保存到 customized。

**可借鉴点**：
- `filecmp` 比逐文件 hash 更高效（利用 OS 缓存的 stat 信息）
- 避免用户目录被默认内容污染

---

## 11. 命令路径避免 Agent 冷启动开销

**位置**：`src/copaw/app/runner/command_dispatch.py`

**设计原则**：凡是 `/` 开头的命令，都走轻量路径——只创建含 memory 的极简对象，不创建完整的 `CoPawAgent`。

**节省的开销**：

| 步骤 | 正常路径 | 命令路径 |
|------|---------|---------|
| Skills 加载 | 磁盘 IO，读取所有 SKILL.md | 跳过 |
| LLM 模型初始化 | 可能有冷启动延迟 | 跳过 |
| MCP clients 注册 | 网络连接 | 跳过 |
| System Prompt 构建 | 读取 3 个 MD 文件 | 跳过 |
| ReActAgent 初始化 | InMemoryMemory + Hook 注册 | 跳过 |

**可借鉴点**：将"轻操作"（如清除对话、重启服务）与"重操作"（完整 Agent 推理）分离，针对性优化响应延迟。

---

## 12. normalize_reasoning_tool_choice（跨 Provider 行为标准化）

**位置**：`src/copaw/agents/react_agent.py:55-62`

```python
def normalize_reasoning_tool_choice(
    tool_choice: Literal["auto", "none", "required"] | None,
    has_tools: bool,
) -> Literal["auto", "none", "required"] | None:
    if tool_choice is None and has_tools:
        return "auto"
    return tool_choice
```

不同 Provider 对 `tool_choice=None`（未指定）的处理方式不同——有些默认不调用工具，有些随机。显式设置为 `"auto"` 可以确保"有工具就能用工具"的行为在所有 Provider 上一致。

**可借鉴点**：在 Agent 框架层面统一处理 Provider 差异，上层业务代码不需要关心 Provider 特性。

---

## 总结：最值得借鉴的 5 个设计模式

| 模式 | 体现位置 | 核心价值 |
|------|---------|---------|
| **配置即文件** | AGENTS.md/SOUL.md/PROFILE.md → System Prompt | Agent 行为可热更新，无需重启 |
| **Touchfile 状态标记** | `.bootstrap_completed` | 比数据库更简单，比内存更持久，可手动重置 |
| **双阶段锁**（连接外，swap 内） | MCPClientManager.replace_client | 最小化锁持有时间，提升并发性能 |
| **Self-describing objects** | `_copaw_rebuild_info` 附加到 client | 对象携带自己的重建信息，解耦管理器 |
| **轻量命令路径** | command_dispatch → _LightweightSessionAgent | 针对不同请求类型分级处理，节省冷启动开销 |
