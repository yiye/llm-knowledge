# Moltbot 项目概览

## 基本信息

- **项目地址**：https://github.com/moltbot/moltbot
- **当前版本**：v2026.1.24-1
- **研究日期**：2026-01-28
- **主要语言**：TypeScript (82.6%), Swift (13.4%), Kotlin (1.9%)
- **Star 数量**：79.1k stars
- **许可证**：MIT

## 项目定位

Moltbot 是一个**个人 AI 助手平台**，核心特点是：

> Your own personal AI assistant. Any OS. Any Platform. The lobster way. 🦞

- **本地优先**：运行在用户自己的设备上
- **多渠道接入**：WhatsApp、Telegram、Slack、Discord、Google Chat、Signal、iMessage 等
- **跨平台**：支持 macOS、Linux、Windows (WSL2)、iOS、Android
- **语音交互**：支持语音唤醒和对话模式
- **Live Canvas**：Agent 驱动的可视化工作空间

## 核心特点

### 1. Gateway 控制平面 ✅

**架构设计**：单一 WebSocket 控制平面，所有组件通过此进行通信

**代码位置**：`src/gateway/server.impl.ts`

根据 `src/gateway/server.impl.ts:147-200`，Gateway 核心功能：
- WebSocket Server (`ws://127.0.0.1:18789`)
- 三种绑定模式：loopback (默认) / lan / tailnet
- HTTP 端点（可选）：
  - `/v1/chat/completions` - OpenAI 兼容接口
  - `/v1/responses` - OpenResponses API
  - Control UI (Web 界面)
- 子系统协调：
  - Channel Manager - 渠道管理
  - Cron Service - 定时任务
  - Node Registry - 节点注册
  - Model Catalog - 模型目录
  - Exec Approval - 执行审批

**协议定义**：`src/gateway/protocol/index.ts` - 使用 JSON Schema + Ajv 验证

### 2. 多渠道支持 ✅

**架构设计**：插件化架构，每个消息平台都是独立的 Channel Plugin

**代码位置**：`src/channels/plugins/`

支持的渠道及技术栈（从 `package.json` 依赖）：
- **WhatsApp**: `@whiskeysockets/baileys` v7.0.0-rc.9
- **Telegram**: `grammy` v1.39.3 + `@grammyjs/runner`
- **Slack**: `@slack/bolt` v4.6.0
- **Discord**: `discord-api-types` v0.38.37
- **Google Chat**: 官方 API
- **Signal**: signal-cli
- **iMessage**: imsg
- **LINE**: `@line/bot-sdk` v10.6.0
- 扩展渠道：BlueBubbles, Matrix, Zalo, Teams 等

**关键组件**：
- `src/channels/plugins/normalize/` - 消息归一化（10+ 个文件）
- `src/channels/plugins/outbound/` - 出站消息处理
- `src/channels/plugins/types.core.ts` - Channel 插件接口定义

### 3. 多 Agent 路由 ✅

**代码位置**：`src/routing/`, `src/sessions/`

**路由机制**：
- `src/routing/resolve-route.ts` - 路由解析逻辑
- `src/routing/session-key.ts` - 会话键管理
- `src/routing/bindings.ts` - 路由绑定配置

**Session 模型**：
- `main` Session - 直接对话
- Group Session - 群组隔离
- Per-Agent Session - 每个 Agent 独立会话

可以将不同渠道/账号/对话路由到隔离的 Agents 和工作空间。

### 4. 语音能力 ✅

**代码位置**：
- macOS: `apps/macos/` (Swift)
- iOS: `apps/ios/` (412 个 Swift 文件)
- Android: `apps/android/` (63 个 Kotlin 文件)

**功能**（根据 README 和代码结构）：
- Voice Wake - 语音唤醒
- Talk Mode - 连续对话模式
- PTT (Push-to-Talk) - 按键通话
- TTS: `node-edge-tts` v1.2.9 (包依赖)

### 5. Skills 系统 ✅

**代码位置**：`skills/` (根目录 40+ 个 Skill), `src/agents/skills/` (11 个文件)

**Skills 类型**：
- Bundled Skills - 内置技能
- Managed Skills - 托管技能
- Workspace Skills - 工作空间特定技能

**可用 Skills**（部分列举）：
- `1password/` - 1Password 集成
- `github/` - GitHub 集成
- `notion/`, `obsidian/` - 笔记工具
- `discord/`, `slack/` - 消息平台操作
- `spotify-player/` - Spotify 播放器
- `tmux/` - Tmux 管理
- `weather/` - 天气查询
- `voice-call/` - 语音通话
- `coding-agent/` - 编码助手
- 等 40+ 个

每个 Skill 包含 `SKILL.md` 或 `*.md` 描述文件。

## 技术栈（初步）

- **运行时**：Node.js ≥22
- **包管理**：pnpm（推荐）、npm、bun
- **前端**：TypeScript
- **iOS**：Swift
- **Android**：Kotlin
- **数据库**：（待确认）
- **消息通信**：WebSocket

## 项目结构（初步）

根据 GitHub 文件列表：

```
moltbot/
  ├── .agent/workflows/      # Agent 工作流
  ├── apps/                  # 应用程序
  ├── packages/clawdbot/     # Clawdbot 包
  ├── src/                   # 源代码
  ├── ui/                    # UI 相关
  ├── skills/                # Skills 系统
  ├── docs/                  # 文档
  ├── extensions/            # 扩展
  ├── Swabble/               # （待了解）
  ├── AGENTS.md              # Agent 规范
  ├── package.json
  └── ...
```

**详细分析**：（待补充）

## 使用场景

1. **个人 AI 助手**：在自己的设备上运行，数据完全掌控
2. **多渠道统一接入**：通过一个助手处理多个消息平台
3. **开发者工具**：支持自定义 skills 和扩展
4. **隐私优先**：不依赖云服务，本地运行

## 安装和运行（官方方式）

```bash
# 安装
npm install -g moltbot@latest

# 运行 onboarding wizard
moltbot onboard --install-daemon

# 启动 Gateway
moltbot gateway --port 18789 --verbose
```

## 研究重点

- [ ] Gateway 架构和 WebSocket 协议设计
- [ ] 多渠道接入的实现方式
- [ ] Session 管理和多 Agent 路由
- [ ] Skills 系统的设计和实现
- [ ] 语音交互的实现
- [ ] Canvas (A2UI) 的实现
- [ ] 安全模型（sandbox、权限控制）
- [ ] 配置管理系统

## 疑问

- Moltbot 和 Clawdbot 的关系？（文档中提到 "clawdbot remains available as a compatibility shim"）
- Swabble 是什么？
- 具体的架构设计理念？
- 如何处理多个 LLM 模型的切换和 fallback？

## 相关资料

- 官方文档：molt.bot（文档链接）
- GitHub：https://github.com/moltbot/moltbot
- Discord 社区：（README 中有链接）

---

**研究状态**：初始化，需要深入阅读代码
