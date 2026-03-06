# NanoClaw 项目概览

## 基本信息

- **仓库地址**：https://github.com/qwibitai/nanoclaw
- **版本**：v1.1.6
- **Commit**：`62c25b1`（研究时 HEAD）
- **研究日期**：2026-03-02
- **主要语言**：TypeScript 99.2%，Node.js 20+
- **代码规模**：约 7,600 行（含测试），纯业务逻辑约 4,400 行
- **协议**：MIT

## 项目定位

来自 `package.json:4`：

```
"description": "Personal Claude assistant. Lightweight, secure, customizable."
```

来自 `README.md`：NanoClaw 是 OpenClaw 的轻量级替代，同样的核心功能（WhatsApp AI 助手、记忆、定时任务），但代码库小到可以完整理解。OpenClaw 接近 50 万行代码，NanoClaw 总计 7,600 行。

**核心设计哲学**（来自 README）：

1. **Small enough to understand**：单进程，几个源文件，无微服务
2. **Secure by isolation**：Agent 跑在 Linux 容器内，OS 级别隔离而非应用层权限检查
3. **Skills over features**：不堆功能到代码库，用 Skill 文件教 Claude Code 改造 fork
4. **AI-native**：无安装向导，无监控面板，一切交给 Claude Code

## 核心功能

- **WhatsApp 集成**：通过 `@whiskeysockets/baileys` 库连接 WhatsApp Web 协议
- **容器化 Agent**：每个 Group 的 Claude Agent 跑在独立 Docker/Apple Container 容器内
- **持久记忆**：每个 Group 有独立 `CLAUDE.md` 文件和会话 session ID
- **定时任务**：支持 cron、interval、once 三种调度模式
- **Agent Swarms**：通过 `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` 支持多 agent 协同（第一个支持该特性的个人 AI 助手）
- **Skills 扩展**：通过代码转换 Skill 添加 Telegram、Discord、Gmail 等集成

## 关键依赖

| 依赖 | 版本 | 用途 |
|------|------|------|
| `@whiskeysockets/baileys` | ^7.0.0-rc.9 | WhatsApp Web 协议 |
| `better-sqlite3` | ^11.8.1 | 本地 SQLite 存储 |
| `cron-parser` | ^5.5.0 | Cron 表达式解析 |
| `@anthropic-ai/claude-agent-sdk` | (容器内) | Claude Code Agent 调用 |
| `@modelcontextprotocol/sdk` | (容器内) | MCP 服务器实现 |
| `zod` | ^4.3.6 | MCP 工具参数校验 |
| `pino` | ^9.6.0 | 结构化日志 |

## 目录结构

```
nanoclaw/
├── src/                      # 宿主进程（Node.js 主进程）
│   ├── index.ts              # 主编排器（532行）
│   ├── channels/
│   │   └── whatsapp.ts       # WhatsApp 通道（384行）
│   ├── container-runner.ts   # 容器启动与 IPC（691行）
│   ├── container-runtime.ts  # 容器运行时抽象（87行）
│   ├── mount-security.ts     # 挂载安全校验（419行）
│   ├── db.ts                 # SQLite 操作（669行）
│   ├── ipc.ts                # IPC 文件监听（387行）
│   ├── task-scheduler.ts     # 定时任务调度（256行）
│   ├── group-queue.ts        # 并发控制队列（357行）
│   ├── router.ts             # 消息格式化与路由
│   ├── config.ts             # 配置常量
│   └── types.ts              # 类型定义（104行）
├── container/
│   ├── Dockerfile            # Agent 容器镜像
│   ├── agent-runner/
│   │   └── src/
│   │       ├── index.ts      # 容器内 Agent 入口（588行）
│   │       └── ipc-mcp-stdio.ts  # MCP 工具服务器（285行）
│   └── skills/               # 容器技能文件
├── .claude/skills/           # Claude Code Skills（15个）
├── groups/                   # 各 Group 数据目录
│   ├── main/                 # 主控频道
│   └── global/               # 跨 Group 共享记忆
├── store/                    # SQLite 数据库（messages.db）
└── data/                     # 运行时数据
    ├── ipc/                  # IPC 文件（按 Group 隔离）
    └── sessions/             # 每 Group 的 .claude/ 和 agent-runner-src/
```
