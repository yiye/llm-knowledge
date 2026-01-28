# Moltbot 代码阅读日志

## 2026-01-28 - 项目初始化

### 基本信息获取

- **克隆完成**：Git submodule 添加成功
- **版本**：v2026.1.24-1
- **文件统计**：（待补充）

### 初步观察

从 README 和文件列表可以看出：

1. **多语言项目**：
   - TypeScript 为主 (82.6%)
   - Swift 用于 iOS/macOS app
   - Kotlin 用于 Android

2. **关键文件**：
   - `AGENTS.md` - Agent 工作规范
   - `package.json` - 依赖管理
   - `moltbot.mjs` - 入口文件？
   - `docker-compose.yml` - Docker 支持

3. **目录结构**：
   - `src/` - 核心源代码
   - `apps/` - 应用程序
   - `packages/clawdbot/` - Clawdbot 相关
   - `ui/` - UI 实现
   - `skills/` - Skills 系统
   - `docs/` - 文档

### 下一步计划

1. [x] 阅读 `package.json` 了解依赖和脚本
2. [x] 阅读 `AGENTS.md` 了解项目的 Agent 规范
3. [x] 查看 `src/` 目录结构
4. [x] 研究 Gateway 实现（WebSocket 控制平面）
5. [ ] 了解 Skills 系统设计（待深入）

---

## 2026-01-28 下午 - 架构分析完成 ✅

### 阅读的文件

**核心入口和配置**：
1. `moltbot.mjs:1-14` - 入口文件（启用 Node.js 编译缓存）
2. `package.json:1-287` - 依赖和脚本配置
3. `src/index.ts:1-95` - 主模块入口
4. `README.md:1-200` - 项目概览和功能介绍

**架构核心**：
5. `src/gateway/server.impl.ts:1-200` - Gateway 服务器实现
6. `src/gateway/protocol/index.ts:1-100` - WebSocket 协议定义
7. `src/cli/program/build-program.ts:1-18` - CLI 程序构建
8. `src/cli/program/command-registry.ts:1-150` - 命令注册表

**Channel 系统**：
9. `src/channels/plugins/types.core.ts:1-150` - Channel 插件接口

**目录结构探索**：
- `src/` - 主要包含 2401 个 TypeScript 文件
- `src/gateway/` - 187 个文件
- `src/agents/` - 436 个文件
- `src/channels/` - 大量子模块

### 架构核心发现

#### 1. Gateway 控制平面设计

**位置**: `src/gateway/server.impl.ts:147-200`

Gateway 是整个系统的神经中枢：
```typescript
export async function startGatewayServer(
  port = 18789,
  opts: GatewayServerOptions = {},
): Promise<GatewayServer>
```

启动流程包括：
1. 配置加载和验证（支持 Legacy Config 自动迁移）
2. 插件自动启用（`applyPluginAutoEnable`）
3. Channel Manager 创建
4. Agent 事件处理器注册
5. Cron Service 初始化
6. Node Registry 设置
7. 健康检查系统启动

**巧妙设计**：使用统一的 WebSocket 控制平面，所有组件（CLI、UI、Mobile App）都通过同一接口通信。

#### 2. 插件化 Channel 架构

**位置**: `src/channels/plugins/`

每个消息平台都是一个独立的 Channel Plugin，遵循统一接口：
- `ChannelMeta` - 元数据定义
- `normalize/` - 消息归一化（将各平台消息转为内部格式）
- `outbound/` - 出站处理（内部格式转为平台格式）
- `onboarding/` - 配置引导

**好处**：
- 易于添加新平台支持
- 各渠道相互隔离
- 上层 Agent 无需关心平台差异

#### 3. Pi Agent 集成

**位置**: `src/agents/`, 依赖 `@mariozechner/pi-agent-core` v0.49.3

使用成熟的 Pi Agent 运行时：
- `pi-embedded-runner/` - 嵌入式运行器（30 个文件）
- `pi-embedded-helpers/` - 辅助工具
- `tools/` - 57 个工具定义文件
- `sandbox/` - 沙箱执行环境（17 个文件）

支持工具流式输出和块流式输出，RPC 模式运行。

#### 4. 协议验证机制

**位置**: `src/gateway/protocol/index.ts`

使用 JSON Schema + Ajv 进行严格的协议验证：
- `AgentEvent`, `ChatEvent`, `ConfigApplyParams` 等
- 每个消息类型都有对应的 Schema
- 运行时类型安全保障

#### 5. 分层架构

```
表现层: UI (Lit), CLI (Commander), Mobile (Swift/Kotlin)
    ↓
控制层: Gateway (WebSocket Server)
    ↓
业务层: Agents (Pi Agent), Channels (Plugins)
    ↓
基础设施层: Config, Logging, Infra
```

每一层职责清晰，解耦良好。

### 技术栈总结

**后端核心**：
- Node.js ≥22 (TypeScript)
- WebSocket (`ws` v8.19.0)
- Express v5.2.1 + Hono v4.11.4

**Agent 运行时**：
- Pi Agent Core v0.49.3
- Pi AI v0.49.3
- Pi Coding Agent v0.49.3

**消息渠道**：
- WhatsApp: Baileys v7.0.0-rc.9
- Telegram: grammY v1.39.3
- Slack: @slack/bolt v4.6.0
- Discord: discord-api-types v0.38.37
- LINE: @line/bot-sdk v10.6.0

**浏览器控制**：
- Playwright Core v1.58.0

**前端**：
- Lit v3.3.2 (Web Components)
- Lucide v0.563.0 (图标)

**移动端**：
- iOS: Swift (412 个文件)
- Android: Kotlin (63 个文件)

### 工程亮点

1. **严格的类型系统**：使用 JSON Schema 和 TypeScript 双重保障
2. **完善的测试**：单元测试、E2E 测试、Live 测试、Docker 测试
3. **代码质量工具**：oxlint, oxfmt, swiftlint, swiftformat
4. **热重载支持**：配置文件可热重载，无需重启
5. **文档完善**：每个 Skill 都有详细的 SKILL.md 说明

### 待深入研究的点

- [ ] A2UI (Agent-to-UI) 的具体实现机制
- [ ] Pi Agent 的工具流式输出原理
- [ ] Session 路由的详细决策逻辑
- [ ] Sandbox 沙箱的安全隔离机制
- [ ] Skills 的动态加载和执行流程
- [ ] Mobile App 与 Gateway 的 Bonjour 配对过程

---

## 2026-01-28 晚上 - Agent 系统深度分析 ✅

### 探索范围

深入研究了 `src/agents/` 目录（436 个 TypeScript 文件）：

**核心模块探索**：
1. **Pi Agent 运行时** (`pi-embedded-runner/` - 30 文件)
   - `run.ts` - 主执行流程
   - `lanes.ts` - 并发控制（Session Lane + Global Lane）
   - `model.ts` - 模型选择
   - `compact.ts` - 上下文压缩
   - `run/attempt.ts` - 单次执行尝试

2. **工具系统** (`tools/` - 57 文件)
   - 消息工具：message, telegram-actions, discord-actions, slack-actions
   - Session 工具：sessions-send, sessions-list, sessions-spawn
   - 浏览器工具：browser-tool (Playwright)
   - Canvas 工具：canvas-tool (A2UI)
   - Cron 工具：cron-tool
   - Web 工具：web-search, web-fetch
   - 图像工具：image-tool
   - Memory 工具：memory-tool

3. **Skills 系统** (`skills/` - 11 文件)
   - 类型定义、配置、加载、热重载
   - 三层加载：Bundled → Managed → Workspace

4. **沙箱系统** (`sandbox/` - 17 文件)
   - Docker 容器隔离
   - 工作空间权限控制
   - 浏览器沙箱

5. **认证系统** (`auth-profiles/` - 15 文件)
   - API Key / Token / OAuth
   - Auth Profile 管理
   - Failover 和 Cooldown 机制

### 关键发现

#### 1. Pi Agent 的轻量级集成

Moltbot 只使用了 Pi Agent Core 的**模型和工具部分**：
- ✅ 使用：`@mariozechner/pi-agent-core` 的 Agent Loop
- ✅ 使用：`@mariozechner/pi-ai` 的统一 LLM API
- ❌ 不使用：Pi Coding Agent 的 Session 管理
- ❌ 不使用：`~/.pi/` 配置

Session 管理、服务发现、工具绑定都是 **Moltbot 自己实现的**。

#### 2. 执行流程核心

**位置**: `src/agents/pi-embedded-runner/run.ts:70-680`

```
用户消息
  ↓
进入 Session Lane (串行队列)
  ↓
runEmbeddedPiAgent():
  1. 模型选择和认证
  2. 工作空间准备
  3. Skills 加载
  4. 系统提示词构建
  5. 历史消息加载
  6. 上下文检查 (需要压缩?)
  7. runEmbeddedAttempt() - 调用 LLM
  8. 工具调用循环
  9. 错误处理和重试
  10. 保存会话记录
  ↓
返回结果到 Gateway
```

#### 3. Lane 并发控制

**位置**: `src/agents/pi-embedded-runner/lanes.ts`

巧妙的两层队列设计：
- **Session Lane**: 每个 Session 独立队列（同一会话串行）
- **Global Lane**: 全局队列（可选，进一步限制并发）

避免了同一 Session 的竞态条件！

#### 4. 工具系统的丰富性

57 个工具分为多个类别：
- 消息工具（10+ 个）
- Session 管理（6 个）
- 浏览器控制（Playwright）
- Canvas（A2UI）
- Cron 调度
- Node 控制（iOS/Android）
- Web 搜索和抓取
- 图像生成
- Memory（Vector 搜索）
- Gateway 管理

工具创建工厂：`src/agents/moltbot-tools.ts:22-162`

#### 5. 错误分类和智能重试

**位置**: `src/agents/pi-embedded-helpers/errors.ts`

错误分类非常细致：
- `auth` → 切换 Auth Profile
- `rate_limit` → Cooldown (60 分钟)
- `context_overflow` → 压缩上下文重试
- `billing` → Cooldown (24 小时)
- `timeout` → 短 Cooldown
- `unknown` → Model Failover

#### 6. Skills 的三层加载

```
1. Bundled Skills (内置)
   ↓
2. Managed Skills (~/.clawdbot/skills)
   ↓
3. Workspace Skills (<workspace>/skills)
   ↓ (优先级最高，可覆盖前两层)
```

Skills 可以自动安装依赖（brew, npm, go, uv, download）。

#### 7. 沙箱隔离设计

三种模式：
- `mode: "off"` - 无沙箱
- `mode: "non-main"` - 仅非主 Session 沙箱化
- `mode: "all"` - 全部沙箱化

工作空间权限：
- `none` - 完全隔离
- `ro` - 只读挂载
- `rw` - 读写挂载

#### 8. 系统提示词构建

**位置**: `src/agents/system-prompt.ts`

动态构建，包含多个部分：
- Agent Identity
- Skills (mandatory)
- Memory Recall
- User Identity
- Messaging
- Sessions
- Cron
- Canvas
- Browser
- Nodes
- Tooling
- Workspace
- Runtime
- Bootstrap Files (AGENTS.md, SOUL.md, etc.)

### 架构亮点

1. **插件化设计**: Tools, Skills, Channels 都是插件
2. **分层清晰**: Tools → Runner → Pi Agent Core → Infrastructure
3. **错误处理完善**: 分类错误 + 智能重试
4. **并发控制巧妙**: Lane 系统避免竞态
5. **沙箱设计周到**: Docker + 权限控制
6. **认证系统强大**: 多 Profile + OAuth + Failover

### 生成的文档

创建了 **`agent-system-deep-dive.md`** (约 800 行)，包含：
- 10 个核心模块的详细解析
- 完整执行流程图
- 关键数据结构
- 设计模式和架构亮点
- 扩展点说明
- 调试技巧

### 待深入研究

- [ ] 具体的工具实现细节（如 Browser Tool 的 CDP 通信）
- [ ] Skills 的动态加载和卸载机制
- [ ] Memory 系统的 Vector 搜索实现
- [ ] Canvas A2UI 的渲染流程
- [ ] Mobile Nodes 的 Bonjour 配对过程

---

**当前状态**：Agent 系统深度分析完成，架构理解透彻 ✅
