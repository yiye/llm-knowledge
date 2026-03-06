# CoPaw 项目概览

## 版本信息

| 字段 | 值 |
|------|---|
| 仓库地址 | https://github.com/agentscope-ai/CoPaw |
| Commit | `d7415fa3c2fa5fb34628a96397f8250057d6c792` |
| 版本 Tag | v0.0.5-beta.2 |
| 研究日期 | 2026-03-05 |
| 主要语言 | Python 73.1% / TypeScript 21.2% |
| 开源协议 | Apache-2.0 |

## 项目定位

**个人助理产品**，而非开发者工具。目标是「Works for you, grows with you」：

- 数据完全本地，不依赖第三方托管
- 通过 Skills 扩展能力（无限可能）
- 支持定时自动执行任务
- 多通道对话（钉钉、飞书、QQ、Discord、iMessage、Telegram）

与同类项目的定位差异：
- **OpenClaw**（同组织，TypeScript）：面向开发者的 AI 助手，功能更重
- **CoPaw**（Python）：面向普通用户的个人助理，强调易用性和数据主权

## 模块地图

```
src/copaw/
├── agents/                  # Agent 核心（最重要）
│   ├── react_agent.py       # CoPawAgent 主类（557行，extends ReActAgent）
│   ├── skills_manager.py    # Skills 三目录管理（885行）
│   ├── skills_hub.py        # Skills Hub 远程导入
│   ├── prompt.py            # System Prompt 组装
│   ├── schema.py            # 数据模型
│   ├── model_factory.py     # 模型工厂（从 providers 配置创建模型）
│   ├── command_handler.py   # /compact /new 等对话命令
│   ├── md_files/            # System Prompt 模板文件
│   │   ├── en/              # SOUL.md / PROFILE.md / BOOTSTRAP.md
│   │   └── zh/              # MEMORY.md / HEARTBEAT.md / AGENTS.md
│   ├── hooks/               # pre_reasoning 生命周期钩子
│   │   ├── bootstrap.py     # 首次对话引导钩子
│   │   └── memory_compaction.py  # 上下文压缩钩子
│   ├── memory/              # 记忆系统
│   │   ├── memory_manager.py     # 封装 ReMeCopaw（171行）
│   │   └── agent_md_manager.py   # MD 文件读写管理（128行）
│   ├── tools/               # 内置工具函数
│   │   ├── file_io.py / file_search.py
│   │   ├── memory_search.py / shell.py
│   │   └── browser_*.py / desktop_screenshot.py / send_file.py
│   └── skills/              # 内置 Skills 源码（跟随包发布）
│       ├── cron / file_reader / news / dingtalk_channel
│       ├── pdf / docx / pptx / xlsx / browser_visible / himalaya
│
├── app/                     # 应用服务层（FastAPI）
│   ├── _app.py              # FastAPI 应用入口
│   ├── channels/            # 多通道抽象层
│   │   ├── base.py / manager.py / registry.py
│   │   ├── renderer.py      # 消息渲染（Markdown → 各平台格式）
│   │   └── dingtalk / feishu / discord_ / telegram / qq / imessage / console / voice
│   ├── runner/              # 请求执行层
│   │   ├── runner.py        # AgentRunner（266行，extends AgentScope Runner）
│   │   ├── session.py       # SafeJSONSession（跨平台文件名兼容）
│   │   ├── command_dispatch.py  # 命令前置分发
│   │   └── repo/            # 会话持久化（json_repo.py）
│   ├── mcp/                 # MCP 集成
│   │   ├── manager.py       # MCPClientManager（热更新）
│   │   └── watcher.py       # 配置变更监听
│   ├── crons/               # 定时任务
│   └── routers/             # FastAPI 路由（RESTful API）
│
├── providers/               # 模型 Provider 层
│   ├── models.py            # Provider 数据模型（Pydantic）
│   ├── registry.py          # 9 个内置 Provider + 自定义 Provider 注册
│   ├── store.py             # providers.json 持久化
│   ├── ollama_manager.py    # Ollama 进程管理
│   └── openai_chat_model_compat.py  # OpenAI stream 兼容层
│
├── local_models/            # 本地模型后端
│   ├── backends/
│   │   ├── base.py          # LocalBackend 抽象类
│   │   ├── llamacpp_backend.py   # llama.cpp（全平台）
│   │   └── mlx_backend.py       # MLX（Apple Silicon）
│   ├── manager.py / factory.py / schema.py
│
├── config/                  # 配置管理
│   ├── config.py            # 配置数据结构（Pydantic）
│   └── watcher.py           # 配置文件热监听
│
├── cli/                     # CLI 命令实现
│   ├── main.py / init_cmd.py / app_cmd.py
│   ├── cron_cmd.py / skills_cmd.py / channels_cmd.py
│   └── providers_cmd.py / daemon_cmd.py / clean_cmd.py
│
├── tunnel/                  # 隧道管理
│   ├── cloudflare.py        # Cloudflare tunnel 自动管理
│   └── binary_manager.py    # cloudflared 二进制下载管理
│
├── tokenizer/               # Token 计数
├── envs/                    # 环境变量读取
├── constant.py              # 全局常量（WORKING_DIR、MEMORY_COMPACT_RATIO 等）
└── __main__.py              # 入口
```

## 关键依赖关系

```
CoPaw（产品层）
    ├── AgentScope                   # Alibaba 开源 Agent 框架（提供 ReActAgent / Toolkit / Msg 等基础设施）
    ├── AgentScope Runtime           # 沙盒 / 请求调度运行时（提供 Runner / AgentRequest 等）
    └── ReMe                        # 记忆系统（提供 ReMeCopaw / 向量+BM25 混合检索，独立 pip 包）
```

## 关键常量（`constant.py`）

| 常量 | 作用 |
|------|------|
| `WORKING_DIR` | `~/.copaw`，所有数据的根目录 |
| `ACTIVE_SKILLS_DIR` | `~/.copaw/active_skills/`，运行时加载的 Skills |
| `CUSTOMIZED_SKILLS_DIR` | `~/.copaw/customized_skills/`，用户自定义 Skills |
| `MEMORY_COMPACT_RATIO` | 上下文压缩触发阈值比例（默认值在此定义） |
| `MEMORY_COMPACT_KEEP_RECENT` | 压缩时保留的最近消息数（默认 10） |

## 技术选型亮点

- **FastAPI** 作为后端服务框架，与 AgentScope Runtime 集成
- **Pydantic** 用于所有配置和数据模型的校验
- **anyio / asyncio** 全异步设计，支持流式输出
- **python-frontmatter** 解析 SKILL.md 的 YAML frontmatter
- **filecmp** 用于 Skills 目录变更检测
- **Cloudflare Tunnel** 自动管理，实现内网穿透接入外部 Channel
