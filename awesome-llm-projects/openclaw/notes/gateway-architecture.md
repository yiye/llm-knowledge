# OpenClaw Gateway 架构深度研究

## 版本信息

- **Commit**: `7305572` → 最新代码分析 (版本 ≥ 2026.3.22)
- **研究日期**: 2026-03-06（2026-03-24 依据最新代码更新）
- **研究重点**: Gateway 控制平面架构、WebSocket 协议、服务发现、与 AI 主动性的关联

---

## 一、Gateway 核心定位

**Gateway 是 OpenClaw 的唯一控制平面和节点传输层**，统一管理所有客户端连接和 Agent 运行时。

### 1.1 架构特点

在 `src/gateway/server.impl.ts:92-94` 中定义了 Gateway Server 的类型：

```typescript
export type GatewayServer = {
  close: (opts?: { reason?: string; restartExpectedMs?: number | null }) => Promise<void>;
};
```

**核心特性**：

1. **WebSocket 控制平面**：所有客户端（CLI、macOS app、iOS/Android nodes）通过 WebSocket 连接
2. **统一 RPC 协议**：基于 JSON-RPC 风格的 Request/Response/Event 三层帧结构
3. **Local-First 设计**：默认绑定 `127.0.0.1:18789`，可配置 LAN/Tailnet 暴露
4. **插件化架构**：Channel、Plugin、Gateway Methods 均支持动态加载

### 1.2 支持的客户端类型

根据 `docs/gateway/protocol.md:137-141` 定义的角色系统：

- **Operator**（控制平面客户端）：
  - CLI (`openclaw`)
  - macOS menubar app
  - Web Control UI
  - 自动化脚本
  
- **Node**（能力宿主）：
  - iOS nodes（Camera、Canvas、Screen、Location）
  - Android nodes
  - Headless nodes（远程执行环境）

---

## 二、WebSocket 协议设计

### 2.1 帧结构（Framing）

在 `docs/gateway/protocol.md:129-133` 中定义了三种帧类型：

```typescript
// Request 帧
{type:"req", id, method, params}

// Response 帧
{type:"res", id, ok, payload|error}

// Event 帧（单向广播）
{type:"event", event, payload, seq?, stateVersion?}
```

**协议特点**：

- **文本帧 + JSON**：使用 WebSocket 文本帧传输 JSON 序列化的消息
- **请求-响应模式**：通过 `id` 字段关联请求和响应
- **事件流**：服务端可主动推送事件（agent 进度、heartbeat、presence 更新等）

### 2.2 握手流程（Handshake）

根据 `src/gateway/server/ws-connection.ts:120-125` 和 `docs/gateway/protocol.md:22-78`：

```
1. Gateway → Client: connect.challenge (nonce)
   {type:"event", event:"connect.challenge", payload:{nonce:"...", ts:...}}

2. Client → Gateway: connect request（带签名）
   {type:"req", method:"connect", params:{auth, device, role, scopes, ...}}

3. Gateway → Client: hello-ok（可能包含 device token）
   {type:"res", ok:true, payload:{protocol:3, auth:{deviceToken:"..."}}}
```

**握手关键逻辑**（`src/gateway/server/ws-connection.ts:61-126`）：

- 连接建立时立即发送 `connect.challenge`（防止重放攻击）
- 客户端必须在 30 秒内完成握手（`getHandshakeTimeoutMs()`）
- 设备身份认证：非本地连接必须签名 nonce（`device.signature`）
- Gateway 颁发 device token 用于后续连接

### 2.3 RPC 方法清单

在 `src/gateway/server-methods-list.ts` 中定义了 **100+ Gateway Methods**（`BASE_METHODS` + channel plugin 动态追加）：

> **注**：最终有效方法集 = `listGatewayMethods()` = `BASE_METHODS` ∪ 各 channel plugin 的 `gatewayMethods`（去重）

**核心方法分类**：

| 类别 | 方法示例 | 说明 |
|------|---------|------|
| **Agent 控制** | `agent`, `agent.wait`, `agent.identity.get` | Agent 执行、身份管理、等待 |
| **Session 管理** | `sessions.list/create/send/abort/patch/reset/delete/compact` | 会话完整生命周期 |
| **Session 订阅** | `sessions.subscribe/unsubscribe`, `sessions.messages.subscribe/unsubscribe` | 实时事件订阅 |
| **Channel 管理** | `channels.status`, `channels.logout` | 消息渠道控制 |
| **Cron 任务** | `cron.list/status/add/update/remove/run/runs` | 定时任务完整 CRUD |
| **Node 管理** | `node.list`, `node.invoke`, `node.pair.approve`, `node.pending.*` | 节点注册和 RPC |
| **配置管理** | `config.get/set/apply/patch/schema/schema.lookup` | 运行时配置热更新 |
| **Agent CRUD** | `agents.list/create/update/delete`, `agents.files.*` | Agent 文件管理 |
| **Skills 管理** | `skills.status/bins/install/update` | Skills 动态加载 |
| **执行审批** | `exec.approvals.*`（6 个）, `exec.approval.request/waitDecision/resolve` | Exec 安全审批系统 |
| **向导配置** | `wizard.start/next/cancel/status` | 引导式配置流程 |
| **语音对话** | `talk.config`, `talk.speak`, `talk.mode` | TTS/语音交互 |
| **TTS** | `tts.status/providers/enable/disable/convert/setProvider` | 文本转语音 |
| **聊天控制** | `chat.send`, `chat.abort`, `chat.history` | WebChat WebSocket-native 方法 |
| **用量统计** | `usage.status`, `usage.cost` | 用量和费用查询 |
| **Secrets** | `secrets.reload`, `secrets.resolve` | 敏感信息管理 |
| **Browser** | `browser.request` | 浏览器控制 |
| **工具目录** | `tools.catalog` | 工具目录查询 |
| **健康状态** | `health`, `doctor.memory.status`, `system-presence`, `last-heartbeat` | 监控和诊断 |

**完整方法列表**（`src/gateway/server-methods-list.ts`）：
```typescript
const BASE_METHODS = [
  // 健康与诊断
  "health", "doctor.memory.status", "logs.tail",
  "status", "usage.status", "usage.cost",
  // 渠道管理
  "channels.status", "channels.logout",
  // TTS
  "tts.status", "tts.providers", "tts.enable", "tts.disable", "tts.convert", "tts.setProvider",
  // 配置管理
  "config.get", "config.set", "config.apply", "config.patch",
  "config.schema", "config.schema.lookup",
  // 执行审批（安全系统）
  "exec.approvals.get", "exec.approvals.set",
  "exec.approvals.node.get", "exec.approvals.node.set",
  "exec.approval.request", "exec.approval.waitDecision", "exec.approval.resolve",
  // 向导配置
  "wizard.start", "wizard.next", "wizard.cancel", "wizard.status",
  // 语音对话
  "talk.config", "talk.speak", "talk.mode",
  // 模型与工具
  "models.list", "tools.catalog",
  // Agent CRUD + 文件管理
  "agents.list", "agents.create", "agents.update", "agents.delete",
  "agents.files.list", "agents.files.get", "agents.files.set",
  // Skills
  "skills.status", "skills.bins", "skills.install", "skills.update",
  // 更新
  "update.run",
  // 语音唤醒
  "voicewake.get", "voicewake.set",
  // Secrets
  "secrets.reload", "secrets.resolve",
  // Session 完整生命周期
  "sessions.list", "sessions.subscribe", "sessions.unsubscribe",
  "sessions.messages.subscribe", "sessions.messages.unsubscribe",
  "sessions.preview", "sessions.create", "sessions.send", "sessions.abort",
  "sessions.patch", "sessions.reset", "sessions.delete", "sessions.compact",
  // Heartbeat & Cron
  "last-heartbeat", "set-heartbeats", "wake",
  "cron.list", "cron.status", "cron.add", "cron.update",
  "cron.remove", "cron.run", "cron.runs",
  // Node 配对
  "node.pair.request", "node.pair.list",
  "node.pair.approve", "node.pair.reject", "node.pair.verify",
  // 设备配对
  "device.pair.list", "device.pair.approve",
  "device.pair.reject", "device.pair.remove",
  "device.token.rotate", "device.token.revoke",
  // Node 管理 & 调用
  "node.rename", "node.list", "node.describe",
  "node.pending.drain", "node.pending.enqueue",
  "node.invoke", "node.pending.pull", "node.pending.ack",
  "node.invoke.result", "node.event",
  "node.canvas.capability.refresh",
  // 系统 & 网关
  "gateway.identity.get",
  "system-presence", "system-event", "send",
  // Agent 执行
  "agent", "agent.identity.get", "agent.wait",
  // Browser & WebChat
  "browser.request",
  "chat.history", "chat.abort", "chat.send",
];
```

### 2.4 事件广播机制

在 `src/gateway/server-methods-list.ts` 中定义了 **22 种事件类型**（含 `GATEWAY_EVENT_UPDATE_AVAILABLE` 常量）：

```typescript
export const GATEWAY_EVENTS = [
  "connect.challenge",        // 握手挑战（连接建立时推送 nonce）
  "agent",                    // Agent 执行进度（流式 token/tool）
  "chat",                     // 聊天消息（WebChat 客户端专用）
  "session.message",          // 会话消息事件
  "session.tool",             // 会话工具调用事件
  "sessions.changed",         // 会话列表变更通知
  "presence",                 // 客户端在线状态变更
  "tick",                     // 心跳 tick（15s 周期）
  "talk.mode",                // 语音模式切换
  "shutdown",                 // Gateway 关闭通知
  "health",                   // 健康状态更新
  "heartbeat",                // Heartbeat 执行事件
  "cron",                     // Cron 任务执行事件
  "node.pair.requested",      // 节点配对请求（需 PAIRING_SCOPE）
  "node.pair.resolved",       // 节点配对结果（需 PAIRING_SCOPE）
  "node.invoke.request",      // 节点 RPC 请求
  "device.pair.requested",    // 设备配对请求（需 PAIRING_SCOPE）
  "device.pair.resolved",     // 设备配对结果（需 PAIRING_SCOPE）
  "voicewake.changed",        // 语音唤醒配置变更
  "exec.approval.requested",  // Exec 审批请求（需 APPROVALS_SCOPE）
  "exec.approval.resolved",   // Exec 审批结果（需 APPROVALS_SCOPE）
  GATEWAY_EVENT_UPDATE_AVAILABLE, // 版本更新可用通知
];
```

> 部分事件受 Scope 访问控制（见 2.5 节），如 `exec.approval.*`、`device.pair.*`、`node.pair.*`。

**Broadcast 实现**（`src/gateway/server-broadcast.ts:34-88`）：

```typescript
export function createGatewayBroadcaster(params: { clients: Set<GatewayWsClient> }) {
  let seq = 0;
  const broadcast = (event: string, payload: unknown, opts?: {...}) => {
    const eventSeq = ++seq;
    const frame = JSON.stringify({
      type: "event", event, payload,
      seq: eventSeq,
      stateVersion: opts?.stateVersion,  // presence/health 版本号
    });
    
    for (const c of params.clients) {
      // 🔥 权限检查：某些事件只推送给有对应 scope 的客户端
      if (!hasEventScope(c, event)) continue;
      
      // 🔥 慢消费者保护：bufferedAmount 超过阈值时断开连接
      const slow = c.socket.bufferedAmount > MAX_BUFFERED_BYTES;
      if (slow && opts?.dropIfSlow) continue;
      if (slow) {
        c.socket.close(1008, "slow consumer");
        continue;
      }
      
      c.socket.send(frame);
    }
  };
  return { broadcast };
}
```

**慢消费者保护机制**：
- `MAX_BUFFERED_BYTES = 5MB`（`src/gateway/server-constants.ts`）
- `dropIfSlow` 选项：对于高频事件（如 agent 流式输出），可选择跳过慢客户端
- 缓冲区超限时直接断开连接（避免内存泄漏）

### 2.5 权限和作用域系统

根据 `docs/gateway/protocol.md:137-161` 和 `src/gateway/server-broadcast.ts:5-32`：

**Operator Scopes**：
```typescript
const ADMIN_SCOPE = "operator.admin";       // 管理员权限（绕过所有检查）
const APPROVALS_SCOPE = "operator.approvals"; // Exec 审批权限
const PAIRING_SCOPE = "operator.pairing";   // 设备配对权限

const EVENT_SCOPE_GUARDS: Record<string, string[]> = {
  "exec.approval.requested": [APPROVALS_SCOPE],
  "exec.approval.resolved": [APPROVALS_SCOPE],
  "device.pair.requested": [PAIRING_SCOPE],
  "device.pair.resolved": [PAIRING_SCOPE],
  "node.pair.requested": [PAIRING_SCOPE],
  "node.pair.resolved": [PAIRING_SCOPE],
};
```

**Node Capabilities**（`docs/gateway/protocol.md:152-160`）：
- `caps`: 高层能力类别（camera、canvas、screen、location、voice）
- `commands`: 命令白名单（`camera.snap`、`canvas.navigate` 等）
- `permissions`: 细粒度开关（`screen.record`、`camera.capture`）

### 2.6 HTTP 安全响应头（Baseline Security Headers）

Gateway 对所有 HTTP 响应应用基线安全头，实现在 `src/gateway/http-common.ts:11-22`：

```typescript
export function setDefaultSecurityHeaders(
  res: ServerResponse,
  opts?: { strictTransportSecurity?: string },
) {
  res.setHeader("X-Content-Type-Options", "nosniff");
  res.setHeader("Referrer-Policy", "no-referrer");
  res.setHeader("Permissions-Policy", "camera=(), microphone=(), geolocation=()");
  // ... HSTS 可选
}
```

**头列表**（`src/gateway/http-common.ts:15-17`）：

| Header | 值 | 说明 |
|--------|-----|------|
| `X-Content-Type-Options` | `nosniff` | 禁止 MIME 类型嗅探 |
| `Referrer-Policy` | `no-referrer` | 不发送 Referer |
| `Permissions-Policy` | `camera=(), microphone=(), geolocation=()` | 禁用摄像头、麦克风、定位（2026.3.3 新增） |

**设计说明**（`src/gateway/http-common.ts:5-10` 注释）：不在此处设置 `X-Frame-Options` 或 `Content-Security-Policy`，因 canvas host、A2UI 等可能被嵌入 iframe。

---

## 三、Gateway 启动流程

### 3.1 完整启动链路

在 `src/gateway/server.impl.ts:147-591` 中实现了完整的 Gateway 启动流程：

```
1. 配置加载和迁移
   ├─ readConfigFileSnapshot() - 读取 ~/.openclaw/openclaw.json
   ├─ migrateLegacyConfig() - 自动迁移旧版配置
   └─ applyPluginAutoEnable() - 插件自动启用

2. 插件和 Channel 初始化
   ├─ loadGatewayPlugins() - 加载插件系统
   ├─ createChannelManager() - 初始化 Channel 管理器
   └─ loadGatewayModelCatalog() - 加载模型目录

3. Skills 系统初始化
   ├─ initSubagentRegistry() - 初始化子 Agent 注册表
   ├─ registerSkillsChangeListener() - 注册 Skills 变更监听器
   └─ primeRemoteSkillsCache() - 预加载远程 Skills 缓存

4. Heartbeat 启动
   └─ startHeartbeatRunner({ cfg }) - 启动 Heartbeat 调度器

5. Cron 服务启动
   └─ cron.start() - 启动 Cron 任务调度

6. WebSocket 服务器启动
   ├─ createGatewayRuntimeState() - 创建运行时状态
   ├─ attachGatewayWsHandlers() - 注册 RPC 方法
   └─ listenGatewayHttpServer() - 监听端口

7. 服务发现启动
   ├─ startGatewayDiscovery() - 启动 Bonjour/mDNS 广播
   └─ startGatewayTailscaleExposure() - Tailscale Serve/Funnel

8. 维护定时器启动
   └─ startGatewayMaintenanceTimers() - tick/health/dedupe 定时器
```

### 3.2 配置加载和迁移

在 `src/gateway/server.impl.ts:162-210` 中实现了配置加载和自动迁移：

```typescript
// 1. 读取配置快照
let configSnapshot = await readConfigFileSnapshot();

// 2. 检测并迁移旧版配置
if (configSnapshot.legacyIssues.length > 0) {
  const { config: migrated, changes } = migrateLegacyConfig(configSnapshot.parsed);
  await writeConfigFile(migrated);
  log.info(`gateway: migrated legacy config entries:\n${changes.join("\n")}`);
}

// 3. 验证配置有效性
if (configSnapshot.exists && !configSnapshot.valid) {
  throw new Error(`Invalid config at ${configSnapshot.path}.\n${issues}`);
}

// 4. 自动启用插件
const autoEnable = applyPluginAutoEnable({ config: configSnapshot.config, env: process.env });
if (autoEnable.changes.length > 0) {
  await writeConfigFile(autoEnable.config);
  log.info(`gateway: auto-enabled plugins:\n${autoEnable.changes.join("\n")}`);
}
```

**配置热重载**（`src/gateway/config-reload.ts`）：
- 监听 `~/.openclaw/openclaw.json` 文件变更
- 自动重载 Channel、Plugin、Heartbeat 配置
- 无需重启 Gateway 即可生效

### 3.3 Skills 系统初始化

在 `src/gateway/server.impl.ts:218-380` 中初始化 Skills 系统：

```typescript
// 1. 初始化子 Agent 注册表
initSubagentRegistry();

// 2. 注册 Skills 变更监听器（文件监控）
const skillsChangeUnsub = registerSkillsChangeListener((event) => {
  if (event.reason === "remote-node") return;
  
  // 🔥 防抖处理：30 秒延迟批量刷新（避免反馈循环）
  if (skillsRefreshTimer) clearTimeout(skillsRefreshTimer);
  skillsRefreshTimer = setTimeout(() => {
    skillsRefreshTimer = null;
    const latest = loadConfig();
    void refreshRemoteBinsForConnectedNodes(latest);
  }, 30_000); // 30 秒防抖
});

// 3. 预加载远程 Skills 缓存
void primeRemoteSkillsCache();
```

**Skills 变更监听实现**（`src/agents/skills/refresh.ts:66-90`）：

```typescript
export function registerSkillsChangeListener(listener: (event: SkillsChangeEvent) => void) {
  listeners.add(listener);
  return () => listeners.delete(listener);  // 返回取消订阅函数
}

export function bumpSkillsSnapshotVersion(params?: {...}): number {
  const reason = params?.reason ?? "manual";
  
  // 更新版本号（使用时间戳或递增）
  if (params?.workspaceDir) {
    const current = workspaceVersions.get(params.workspaceDir) ?? 0;
    const next = bumpVersion(current);
    workspaceVersions.set(params.workspaceDir, next);
    emit({ workspaceDir: params.workspaceDir, reason, changedPath });
    return next;
  }
  
  globalVersion = bumpVersion(globalVersion);
  emit({ reason, changedPath });
  return globalVersion;
}
```

**Skills 文件监控**（`src/agents/skills/refresh.ts:100-176`）：
- 使用 `chokidar` 监控 Skills 目录（workspace/skills、~/.openclaw/skills、插件 skills）
- 防抖处理：1000ms 延迟批量触发刷新（避免频繁触发）
- 忽略模式：`.git/`、`node_modules/`、`dist/`

### 3.4 Heartbeat Runner 启动

在 `src/gateway/server.impl.ts:414` 中启动 Heartbeat 调度器：

```typescript
let heartbeatRunner = startHeartbeatRunner({ cfg: cfgAtStart });
```

**Heartbeat Runner 实现**（`src/infra/heartbeat-runner.ts:1-970`）：

- **定时器管理**：使用 Node.js `setInterval` 调度 Heartbeat 执行
- **Active Hours 控制**：根据用户时区检查活跃时间窗口（避免夜间打扰）
- **HEARTBEAT_OK 智能静默**：当回复只包含 `HEARTBEAT_OK` 且内容≤300字符时，不发送消息
- **队列管理**：检查主队列（main lane）是否繁忙，避免阻塞用户操作

**Heartbeat 事件订阅**（`src/gateway/server.impl.ts:410-412`）：

```typescript
const heartbeatUnsub = onHeartbeatEvent((evt) => {
  broadcast("heartbeat", evt, { dropIfSlow: true });
});
```

### 3.5 WebSocket 服务器启动

在 `src/gateway/server.impl.ts:426-457` 中注册 WebSocket 处理器：

```typescript
attachGatewayWsHandlers({
  wss,                      // WebSocketServer 实例
  clients,                  // Set<GatewayWsClient> 客户端池
  port,
  gatewayHost: bindHost,
  canvasHostEnabled,
  canvasHostServerPort,
  resolvedAuth,             // Token/Password 认证配置
  gatewayMethods,           // 100+ RPC 方法列表（BASE_METHODS + plugin methods）
  events: GATEWAY_EVENTS,   // 22 种事件类型
  logGateway: log,
  logHealth,
  logWsControl,
  extraHandlers: {          // 插件和 Exec 审批的额外 RPC 方法
    ...pluginRegistry.gatewayHandlers,
    ...execApprovalHandlers,
  },
  broadcast,                // 事件广播函数
  context: {                // RPC 方法执行上下文
    deps, cron, nodeRegistry, channelManager, execApprovalManager, ...
  },
});
```

**WS 连接处理器**（`src/gateway/server-ws-runtime.ts:8-49`）：
- 调用 `attachGatewayWsConnectionHandler` 注册 `connection` 事件监听器
- 处理握手、消息路由、错误处理、连接关闭

**WS 消息路由**（`src/gateway/server/ws-connection/message-handler.ts`）：
- 解析 JSON 帧（Request/Response/Event）
- 路由到对应的 RPC 方法（`server-methods/` 目录）
- 返回响应或推送事件

---

## 四、服务发现机制

### 4.1 三种发现模式

根据 `docs/gateway/discovery.md:44-101` 和 `src/gateway/server-discovery-runtime.ts:10-100`：

| 发现模式 | 适用场景 | 实现方式 | 配置项 |
|---------|---------|---------|--------|
| **Bonjour/mDNS** | 同局域网自动发现 | `_openclaw-gw._tcp` 服务广播 | `discovery.mdns.mode` |
| **Tailscale** | 跨网络（tailnet）访问 | MagicDNS 域名或 tailnet IP | `gateway.bind: "tailnet"` |
| **SSH Tunnel** | 通用回退方案 | 端口转发 `127.0.0.1:18789` | 手动配置 |

### 4.2 Bonjour/mDNS 实现

在 `src/gateway/server-discovery-runtime.ts:41-58` 中启动 Bonjour 广播：

```typescript
if (bonjourEnabled) {
  try {
    const bonjour = await startGatewayBonjourAdvertiser({
      instanceName: formatBonjourInstanceName(params.machineDisplayName),
      gatewayPort: params.port,
      gatewayTlsEnabled: params.gatewayTls?.enabled ?? false,
      gatewayTlsFingerprintSha256: params.gatewayTls?.fingerprintSha256,
      canvasPort: params.canvasPort,
      sshPort,                // SSH 端口（默认 22）
      tailnetDns,             // Tailscale MagicDNS 提示
      cliPath,                // openclaw CLI 路径
      minimal: mdnsMinimal,   // 最小模式（只广播核心字段）
    });
    bonjourStop = bonjour.stop;
  } catch (err) {
    params.logDiscovery.warn(`bonjour advertising failed: ${String(err)}`);
  }
}
```

**Bonjour 服务类型**（`docs/gateway/discovery.md:58-70`）：
- 服务类型：`_openclaw-gw._tcp`
- TXT 记录字段：
  - `role=gateway`
  - `lanHost=<hostname>.local`
  - `gatewayPort=18789`
  - `gatewayTls=1`（可选）
  - `gatewayTlsSha256=<sha256>`（可选）
  - `canvasPort=18793`
  - `sshPort=22`
  - `tailnetDns=<magicdns>`（可选）
  - `cliPath=<path>`（可选）

**Bonjour 禁用方式**：
- 环境变量：`OPENCLAW_DISABLE_BONJOUR=1`
- 配置项：`discovery.mdns.mode: "off"`
- 测试环境：`NODE_ENV=test` 或 `VITEST` 自动禁用

### 4.3 Tailscale 集成

在 `src/gateway/server-discovery-runtime.ts:32-34` 中检测 Tailscale：

```typescript
const tailscaleEnabled = params.tailscaleMode !== "off";
const needsTailnetDns = bonjourEnabled || params.wideAreaDiscoveryEnabled;
const tailnetDns = needsTailnetDns
  ? await resolveTailnetDnsHint({ enabled: tailscaleEnabled })
  : undefined;
```

**Tailscale 暴露模式**（`src/gateway/server-tailscale.ts`）：
- **Serve 模式**：仅 Tailnet 内访问（推荐）
- **Funnel 模式**：公网可访问（需谨慎使用）

**Tailscale IP 选择**（`src/infra/tailnet.ts`）：
- 优先选择 IPv4（`100.64.0.0/10` 地址段）
- 备用 IPv6（`fd7a:115c:a1e0::/48` 地址段）

### 4.4 Wide-Area DNS-SD

在 `src/gateway/server-discovery-runtime.ts:60-97` 中实现广域发现：

```typescript
if (params.wideAreaDiscoveryEnabled) {
  const wideAreaDomain = resolveWideAreaDiscoveryDomain({
    configDomain: params.wideAreaDiscoveryDomain ?? undefined,
  });
  
  const tailnetIPv4 = pickPrimaryTailnetIPv4();
  if (!tailnetIPv4) {
    params.logDiscovery.warn("no Tailscale IPv4 address found; skipping unicast DNS-SD");
  } else {
    const result = await writeWideAreaGatewayZone({
      domain: wideAreaDomain,
      gatewayPort: params.port,
      displayName: formatBonjourInstanceName(params.machineDisplayName),
      tailnetIPv4,
      tailnetIPv6: pickPrimaryTailnetIPv6() ?? undefined,
      gatewayTlsEnabled: params.gatewayTls?.enabled ?? false,
      gatewayTlsFingerprintSha256: params.gatewayTls?.fingerprintSha256,
      tailnetDns,
      sshPort,
      cliPath: resolveBonjourCliPath(),
    });
    params.logDiscovery.info(
      `wide-area DNS-SD ${result.changed ? "updated" : "unchanged"} (${wideAreaDomain} → ${result.zonePath})`
    );
  }
}
```

**Wide-Area DNS-SD 特点**：
- 基于 Unicast DNS-SD（RFC 6763）
- 需要 Tailscale IPv4 地址
- 写入本地 DNS zone 文件（供 DNS 服务器读取）
- 支持跨网络发现（不依赖 mDNS 多播）

---

## 五、Gateway 运行时状态管理

### 5.1 核心状态结构

在 `src/gateway/server-runtime-state.ts:24-72` 中定义了 Gateway 运行时状态：

```typescript
export async function createGatewayRuntimeState(params: {...}): Promise<{
  canvasHost: CanvasHostHandler | null;      // Canvas Host（A2UI）
  httpServer: HttpServer;                    // HTTP 服务器（主）
  httpServers: HttpServer[];                 // 多监听地址支持
  httpBindHosts: string[];                   // 绑定地址列表
  wss: WebSocketServer;                      // WebSocket 服务器
  clients: Set<GatewayWsClient>;             // 客户端连接池
  broadcast: (event, payload, opts?) => void; // 事件广播函数
  agentRunSeq: Map<string, number>;          // Agent 运行序列号
  dedupe: Map<string, DedupeEntry>;          // 消息去重表
  chatRunState: ReturnType<typeof createChatRunState>; // Chat 运行状态
  chatRunBuffers: Map<string, string>;       // Chat 输出缓冲区
  chatDeltaSentAt: Map<string, number>;      // Chat 增量发送时间戳
  addChatRun: (sessionId, entry) => void;    // 添加 Chat Run
  removeChatRun: (sessionId, clientRunId, sessionKey?) => ChatRunEntry | undefined; // 移除 Chat Run
  chatAbortControllers: Map<string, ChatAbortControllerEntry>; // Chat 中止控制器
}> {
  // 实现省略...
}
```

**关键状态说明**：

1. **客户端连接池**（`clients: Set<GatewayWsClient>`）：
   - 存储所有活跃的 WebSocket 连接
   - 每个客户端包含：`socket`、`connect`（握手信息）、`deviceId`、`role`、`scopes`
   
2. **Broadcast 函数**（`src/gateway/server-broadcast.ts:34-88`）：
   - 统一事件广播入口
   - 自动处理权限检查（`hasEventScope`）
   - 慢消费者保护（`bufferedAmount` 检查）

3. **Agent 运行序列号**（`agentRunSeq: Map<string, number>`）：
   - 跟踪每个 Session 的 Agent 运行次数
   - 用于生成唯一的 Run ID（`${sessionKey}:${seq}`）

4. **消息去重表**（`dedupe: Map<string, DedupeEntry>`）：
   - 防止重复消息投递（基于消息哈希）
   - 定期清理过期条目（维护定时器）

### 5.2 维护定时器

在 `src/gateway/server-maintenance.ts` 中实现了三个维护定时器：

```typescript
export function startGatewayMaintenanceTimers(params: {...}) {
  // 1. Tick 定时器（15 秒）- 推送 presence 和 health 更新
  const tickInterval = setInterval(() => {
    const presenceVersion = params.getPresenceVersion();
    const healthVersion = params.getHealthVersion();
    params.broadcast("tick", { ts: Date.now() }, {
      dropIfSlow: true,
      stateVersion: { presence: presenceVersion, health: healthVersion },
    });
    params.nodeSendToAllSubscribed("tick", { ts: Date.now() });
  }, 15_000);

  // 2. Health 定时器（30 秒）- 刷新健康状态快照
  const healthInterval = setInterval(() => {
    params.refreshGatewayHealthSnapshot();
    const healthVersion = params.getHealthVersion();
    params.logHealth.trace(`health snapshot refreshed (v${healthVersion})`);
  }, 30_000);

  // 3. Dedupe 清理定时器（60 秒）- 清理过期的去重条目
  const dedupeCleanup = setInterval(() => {
    const now = Date.now();
    for (const [key, entry] of params.dedupe.entries()) {
      if (now - entry.seenAt > 5 * 60 * 1000) {  // 5 分钟过期
        params.dedupe.delete(key);
      }
    }
  }, 60_000);

  return { tickInterval, healthInterval, dedupeCleanup };
}
```

**定时器职责**：
- **tick**：维持客户端心跳，推送 presence/health 版本号
- **health**：定期刷新系统健康状态（队列长度、内存使用等）
- **dedupe**：清理过期的消息去重表（防止内存泄漏）

### 5.3 状态版本控制

在 `src/gateway/server/health-state.ts` 中实现了版本号管理：

```typescript
let presenceVersion = 0;
let healthVersion = 0;

export function getPresenceVersion(): number {
  return presenceVersion;
}

export function incrementPresenceVersion(): number {
  return ++presenceVersion;
}

export function getHealthVersion(): number {
  return healthVersion;
}

export function refreshGatewayHealthSnapshot() {
  healthVersion++;
  // 刷新健康状态快照...
}
```

**版本号用途**：
- **Presence Version**：客户端在线状态变更时递增（连接/断开/角色变更）
- **Health Version**：系统健康状态更新时递增（队列、内存、Channel 状态）
- 客户端收到 `tick` 事件时，根据版本号判断是否需要重新拉取完整状态

---

## 六、与 AI 主动性的关联

### 6.1 Heartbeat 调度集成

**Heartbeat 在 Gateway 中的完整生命周期**：

```
1. Gateway 启动时初始化 Heartbeat Runner
   └─ src/gateway/server.impl.ts:414
      heartbeatRunner = startHeartbeatRunner({ cfg: cfgAtStart })

2. Heartbeat Runner 启动定时器
   └─ src/infra/heartbeat-runner.ts:800-970
      - 解析 heartbeat.every 配置（默认 30m）
      - 检查 Active Hours（时区感知）
      - 创建 setInterval 定时器

3. 定时触发 Heartbeat 执行
   └─ src/infra/heartbeat-runner.ts:600-750
      - 检查主队列是否繁忙（避免阻塞用户操作）
      - 读取 HEARTBEAT.md（如果存在）
      - 构造 Agent 运行请求（注入 heartbeat prompt）
      - 调用 Agent Run（reuse session context）

4. Agent 执行 Heartbeat 任务
   └─ src/agents/pi-embedded-runner/
      - 流式执行 Agent（Claude API）
      - 检测 HEARTBEAT_OK token
      - 返回结果（可能包含 thinking）

5. 判断是否发送消息
   └─ src/infra/heartbeat-runner.ts:650-700
      - 如果回复包含 HEARTBEAT_OK 且内容 ≤ ackMaxChars（默认 300 字符）：静默
      - 否则：通过 Channel 发送消息到目标（last/telegram/whatsapp/...）

6. 广播 Heartbeat 事件
   └─ src/gateway/server.impl.ts:410-412
      broadcast("heartbeat", evt, { dropIfSlow: true })

7. 客户端接收 Heartbeat 事件
   └─ macOS app/CLI/Web UI 显示 Heartbeat 状态指示器
```

**Heartbeat 配置热重载**（`src/gateway/config-reload.ts`）：

```typescript
// 配置文件变更时自动重启 Heartbeat Runner
const reloadHeartbeat = () => {
  if (heartbeatRunner) {
    heartbeatRunner.stop();
  }
  const latest = loadConfig();
  heartbeatRunner = startHeartbeatRunner({ cfg: latest });
};
```

**Heartbeat 权限控制**：
- Heartbeat 在 **main session** 中执行（agent 默认会话）
- 拥有与用户交互相同的工具权限（bash、memory_search、skill_install 等）
- 可以通过 `heartbeat.session` 配置指定其他 Session

**Heartbeat 队列管理**（`src/infra/heartbeat-runner.ts:620-650`）：

```typescript
// 检查主队列是否繁忙
const queueSize = getQueueSize ? getQueueSize(CommandLane.Main) : 0;
if (queueSize > 0) {
  log.debug(`heartbeat skipped (main queue busy: ${queueSize} pending)`);
  emitHeartbeatEvent({
    type: "skipped",
    reason: "queue-busy",
    agentId,
    sessionKey,
    queueSize,
  });
  return { skipped: true, reason: "queue-busy" };
}
```

### 6.2 Memory 初始化（推测）

**注意**：璇玑在 Gateway 代码中**没有找到直接的 `MemoryIndexManager` 初始化代码**，但从架构推测：

1. **Memory 初始化位置（推测）**：
   - 可能在 Plugin 系统中初始化（`extensions/memory-*` 插件）
   - 或在 Agent 启动时延迟初始化（首次使用 `memory_search` 时）

2. **Memory 与 Gateway 的集成点**：
   - **Session Transcript 监听**：`src/memory/manager.ts:13` 导入了 `onSessionTranscriptUpdate`
   - **文件监控**：Gateway 的文件监控系统（`chokidar`）可能触发 Memory 索引更新
   - **配置管理**：Memory 配置（embedding provider、vector search 等）在 `openclaw.json` 中

3. **Memory 自动索引流程**（`src/memory/manager.ts:1-2397`）：
   ```typescript
   // 监听 Session Transcript 更新
   onSessionTranscriptUpdate((evt) => {
     // 触发增量索引
     syncSessionFiles(evt.sessionKey);
   });

   // 监听 Workspace 文件变更
   const watcher = chokidar.watch(workspaceDir, {
     ignored: /(^|[\\/])\.|node_modules/,
     persistent: true,
   });
   watcher.on("change", (path) => {
     // 触发增量索引
     syncMemoryFiles(path);
   });
   ```

4. **Memory Search Tool 调用链**：
   ```
   Agent → memory_search tool → MemoryIndexManager.search()
   → Hybrid Search (Vector + FTS5) → 返回结果
   ```

**待研究**：
- [ ] Memory 初始化的具体位置和时机
- [ ] Memory 插件（`extensions/memory-*`）的加载机制
- [ ] Memory 配置的热重载是否支持

### 6.3 Skills 刷新机制

**Skills 在 Gateway 中的完整刷新流程**：

```
1. Gateway 启动时注册 Skills 变更监听器
   └─ src/gateway/server.impl.ts:368-380
      registerSkillsChangeListener((event) => {
        // 30 秒防抖后刷新远程 Skills
        refreshRemoteBinsForConnectedNodes(latest);
      })

2. Skills 文件监控启动
   └─ src/agents/skills/refresh.ts:100-176
      ensureSkillsWatcher({ workspaceDir, config })
      - 监控目录：workspace/skills、~/.openclaw/skills、plugins/skills
      - 使用 chokidar 监听文件变更

3. 文件变更触发版本更新
   └─ src/agents/skills/refresh.ts:73-90
      bumpSkillsSnapshotVersion({ workspaceDir, reason: "watch", changedPath })
      - 更新 globalVersion 或 workspaceVersions
      - 触发所有注册的监听器

4. Gateway 监听器收到通知
   └─ src/gateway/server.impl.ts:368-380
      - 30 秒防抖延迟（避免频繁触发）
      - 刷新远程节点的 Skills 列表

5. Agent 下次运行时读取最新 Skills
   └─ src/agents/skills/
      - 根据 snapshotVersion 判断是否需要重新扫描
      - 三层加载：Bundled → Managed → Workspace
      - 动态注入到 System Prompt
```

**Skills 三层加载优先级**（`src/agents/skills/`）：

```typescript
// 1. Bundled Skills（内置 53+ skills）
const bundledSkills = loadBundledSkills();  // skills/ 目录

// 2. Managed Skills（用户安装，来自 ClawdHub）
const managedSkills = loadManagedSkills();  // ~/.openclaw/skills/ 目录

// 3. Workspace Skills（项目级，优先级最高）
const workspaceSkills = loadWorkspaceSkills(workspaceDir);  // <workspace>/skills/ 目录

// 合并（Workspace 覆盖 Managed 覆盖 Bundled）
const allSkills = { ...bundledSkills, ...managedSkills, ...workspaceSkills };
```

**Skills 动态安装流程**：

```
1. Agent 调用 skill_search tool
   └─ src/agents/tools/skill-search-tool.ts
      - 搜索本地 Skills（fuzzy match）
      - 调用 ClawdHub Registry API 搜索远程 Skills

2. Agent 调用 skill_install tool
   └─ src/agents/tools/skill-install-tool.ts
      - 从 ClawdHub 下载 SKILL.md
      - 保存到 ~/.openclaw/skills/<name>/
      - 触发 Skills 变更事件

3. Skills 变更事件触发刷新
   └─ registerSkillsChangeListener 回调
      - Gateway 更新 Skills 快照版本
      - 远程节点同步 Skills 列表

4. Agent 下次运行时自动加载新 Skills
   └─ System Prompt 动态注入新 Skill
```

**Skills 配置热重载**（`src/gateway/config-reload.ts`）：
- 配置文件中的 `skills.load.extraDirs` 变更时自动刷新
- 无需重启 Gateway

---

## 七、Gateway 关闭流程

### 7.1 优雅关闭实现

在 `src/gateway/server-close.ts` 中实现了优雅关闭处理器：

```typescript
export function createGatewayCloseHandler(params: {
  httpServers: HttpServer[];
  wss: WebSocketServer;
  clients: Set<GatewayWsClient>;
  broadcast: (event, payload, opts?) => void;
  canvasHostServer: CanvasHostServer | null;
  cronStop: () => Promise<void>;
  heartbeatStop: () => void;
  // ... 其他清理资源
}) {
  return async (opts?: { reason?: string; restartExpectedMs?: number }) => {
    const reason = truncateCloseReason(opts?.reason ?? "gateway closed");
    
    // 1. 广播 shutdown 事件（提前通知客户端）
    params.broadcast("shutdown", {
      reason,
      restartExpectedMs: opts?.restartExpectedMs ?? null,
    });

    // 2. 停止 Heartbeat Runner
    params.heartbeatStop();

    // 3. 停止 Cron 服务
    await params.cronStop();

    // 4. 关闭所有 WebSocket 连接
    for (const client of params.clients) {
      try {
        client.socket.close(1001, reason);
      } catch {
        /* ignore */
      }
    }

    // 5. 关闭 WebSocket 服务器
    await new Promise<void>((resolve) => {
      params.wss.close(() => resolve());
    });

    // 6. 关闭 HTTP 服务器
    await Promise.all(
      params.httpServers.map((server) =>
        new Promise<void>((resolve) => {
          server.close(() => resolve());
        })
      )
    );

    // 7. 停止 Canvas Host
    if (params.canvasHostServer) {
      await params.canvasHostServer.stop();
    }

    // 8. 停止 Bonjour 广播
    if (params.bonjourStop) {
      await params.bonjourStop();
    }

    // 9. 清理监听器
    params.agentUnsub();
    params.heartbeatUnsub();
    params.skillsChangeUnsub();
    params.transcriptUnsub();

    // 10. 清理定时器
    clearInterval(params.tickInterval);
    clearInterval(params.healthInterval);
    clearInterval(params.dedupeCleanup);
  };
}
```

**关闭流程关键点**：

1. **提前通知**：广播 `shutdown` 事件（包含 `restartExpectedMs`）
2. **停止调度**：先停止 Heartbeat 和 Cron，避免新任务启动
3. **关闭连接**：优雅关闭所有 WebSocket 连接（1001 状态码）
4. **清理资源**：停止服务器、定时器、文件监听器、事件监听器
5. **顺序保证**：先停止业务逻辑，再关闭网络连接，最后清理资源

### 7.2 重启机制

在 `src/infra/restart.ts` 中实现了 SIGUSR1 重启支持：

```typescript
export function setGatewaySigusr1RestartPolicy(opts: { allowExternal: boolean }) {
  if (opts.allowExternal) {
    process.on("SIGUSR1", () => {
      log.info("received SIGUSR1, restarting gateway");
      // 触发 Gateway 重启
      restartGateway();
    });
  }
}
```

**重启触发方式**：
- `openclaw gateway restart`（CLI 命令）
- `kill -USR1 <gateway-pid>`（信号）
- macOS app 菜单中的 "Restart Gateway"

**重启流程**：
1. 调用 `close({ reason: "restart", restartExpectedMs: 5000 })`
2. Gateway 优雅关闭（广播 shutdown 事件）
3. 进程退出或重新启动

---

## 八、可借鉴的设计模式

### 8.1 控制平面统一化

**设计理念**：
- **单一控制平面**：所有客户端（CLI、GUI、Mobile）通过同一 WebSocket 协议连接
- **角色+权限系统**：Operator vs Node，Scopes 细粒度控制
- **协议版本控制**：客户端和服务端协商协议版本，避免不兼容

**可借鉴点**：
- 避免为每种客户端设计独立的 API（HTTP REST、gRPC、WebSocket...）
- 统一的事件广播机制（服务端主动推送）
- 设备身份认证+配对（Device Token + Public Key Signature）

### 8.2 配置热重载机制

**设计理念**：
- **文件监控**：使用 `chokidar` 监听配置文件变更
- **增量更新**：只重载变更的部分（Channel、Plugin、Heartbeat）
- **无缝切换**：无需重启 Gateway，用户无感知

**实现要点**：
```typescript
// 1. 监听配置文件
const watcher = chokidar.watch(CONFIG_PATH, { persistent: true });

// 2. 防抖处理（1 秒延迟）
let reloadTimer: ReturnType<typeof setTimeout> | null = null;
watcher.on("change", () => {
  if (reloadTimer) clearTimeout(reloadTimer);
  reloadTimer = setTimeout(() => {
    reloadConfig();
  }, 1000);
});

// 3. 增量重载
function reloadConfig() {
  const latest = loadConfig();
  
  // 重载 Channels
  channelManager.reload(latest.channels);
  
  // 重载 Heartbeat
  heartbeatRunner.stop();
  heartbeatRunner = startHeartbeatRunner({ cfg: latest });
  
  // 重载 Skills
  bumpSkillsSnapshotVersion({ reason: "manual" });
}
```

### 8.3 慢消费者保护

**设计理念**：
- **缓冲区监控**：检查 `socket.bufferedAmount`（未发送的字节数）
- **分级处理**：
  - `dropIfSlow`: 跳过非关键事件（tick、presence）
  - 超过阈值：直接断开连接（避免阻塞其他客户端）

**实现要点**（`src/gateway/server-broadcast.ts:68-78`）：
```typescript
for (const c of params.clients) {
  const slow = c.socket.bufferedAmount > MAX_BUFFERED_BYTES; // 5MB
  
  if (slow && opts?.dropIfSlow) {
    continue;  // 跳过非关键事件
  }
  
  if (slow) {
    c.socket.close(1008, "slow consumer");  // 断开连接
    continue;
  }
  
  c.socket.send(frame);
}
```

### 8.4 防抖批量处理

**设计理念**：
- **Skills 刷新**：30 秒防抖（避免文件监控的频繁触发）
- **配置重载**：1 秒防抖（避免编辑器保存时的多次写入）
- **批量处理**：合并短时间内的多个变更事件

**实现模式**：
```typescript
let timer: ReturnType<typeof setTimeout> | null = null;
const DEBOUNCE_MS = 30_000;

function onFileChange(path: string) {
  if (timer) clearTimeout(timer);
  
  timer = setTimeout(() => {
    timer = null;
    // 执行批量处理
    refreshAll();
  }, DEBOUNCE_MS);
}
```

### 8.5 状态版本控制

**设计理念**：
- **增量更新**：客户端根据版本号判断是否需要重新拉取状态
- **版本递增**：每次状态变更时递增版本号（`++version`）
- **事件携带版本**：`tick` 事件携带 `stateVersion: { presence, health }`

**可借鉴点**：
- 避免每次 tick 都推送完整状态（节省带宽）
- 客户端可以缓存状态，只在版本变更时更新
- 适用于高频推送场景（presence、health、metrics）

### 8.6 服务发现多层次设计

**设计理念**：
- **局域网**：Bonjour/mDNS 自动发现（零配置）
- **跨网络**：Tailscale MagicDNS（私有网络）
- **通用回退**：SSH Tunnel（任何 SSH 可达的环境）

**可借鉴点**：
- 不依赖单一发现机制（避免单点故障）
- 优先选择零配置方案（用户体验友好）
- 保留手动配置通道（专家用户灵活性）

### 8.7 插件化架构

**设计理念**：
- **Gateway Methods 扩展**：插件可注册新的 RPC 方法
- **Channel 扩展**：插件可添加新的消息渠道（MS Teams、Matrix、Zalo）
- **动态加载**：插件在 Gateway 启动时自动发现和加载

**实现要点**（`src/gateway/server.impl.ts:222-236`）：
```typescript
const { pluginRegistry, gatewayMethods: baseGatewayMethods } = loadGatewayPlugins({
  cfg: cfgAtStart,
  workspaceDir: defaultWorkspaceDir,
  log,
  coreGatewayHandlers,
  baseMethods,
});

// 合并核心方法和插件方法
const channelMethods = listChannelPlugins().flatMap((plugin) => plugin.gatewayMethods ?? []);
const gatewayMethods = Array.from(new Set([...baseGatewayMethods, ...channelMethods]));
```

---

## 九、CLI 与配置增强（2026.3.3）

### 9.1 `openclaw config validate` 命令

在 Gateway 启动前校验配置文件，无需启动 Gateway 即可发现配置错误。

**命令注册**（`src/cli/config-cli.ts:354-360`）：

```typescript
cmd
  .command("validate")
  .description("Validate the current config against the schema without starting the gateway")
  .option("--json", "Output validation result as JSON", false)
  .action(async (opts) => {
    await runConfigValidate({ json: Boolean(opts.json) });
  });
```

**核心实现**（`src/cli/config-cli.ts:341-389`）：

- 调用 `readConfigFileSnapshot()` 读取配置
- 校验逻辑复用 Gateway 启动时的 schema 验证（Zod + 插件 AJV）
- 成功：`Config valid: <path>`，退出码 0
- 失败：输出 `path`、`message`，并包含 `allowedValues` 等元数据（Zod 枚举提示）
- `--json`：机器可读输出，格式为 `{ valid, path, issues?, error? }`

**`--json` 输出结构**（`src/cli/config-cli.test.ts:257-295`）：

```json
{
  "valid": false,
  "path": "/tmp/custom-openclaw.json",
  "issues": [
    {
      "path": "update.channel",
      "message": "Invalid input (allowed: \"stable\", \"beta\", \"dev\")",
      "allowedValues": ["stable", "beta", "dev"]
    }
  ]
}
```

**Config Guard 豁免**（`src/cli/program/preaction.ts:50`）：`config validate` 绕过配置就绪检查，即使配置无效也可执行（用于诊断）。

### 9.2 CLI Banner Tagline 模式：`cli.banner.taglineMode`

控制 CLI 启动时 banner 标语（tagline）的显示方式。

**配置项**（`src/config/zod-schema.ts:254-256`）：

```typescript
taglineMode: z
  .union([z.literal("random"), z.literal("default"), z.literal("off")])
  .optional(),
```

**模式说明**（`src/cli/tagline.ts:254-272`）：

| 值 | 行为 |
|----|------|
| `random` | 从 TAGLINES 池中随机选取（含节日标语），默认行为 |
| `default` | 固定显示 `"All your chats, one OpenClaw."` |
| `off` | 不显示 tagline |

**解析逻辑**（`src/cli/banner.ts:38-56`）：优先使用 `options.mode`（如 `--no-banner` 等），否则从 `loadConfig().cli?.banner?.taglineMode` 读取。

**配置示例**（`docs/gateway/configuration-reference.md:2789`）：

```json
{
  "cli": {
    "banner": {
      "taglineMode": "off"
    }
  }
}
```

### 9.3 版本上报改进（serverVersion / bootstrapVersion）

Gateway 与 Control UI 的版本信息传递流程已统一，避免 `dev` 占位符泄露。

**Bootstrap 端点**（`src/gateway/control-ui.ts:349-355`）：

Control UI 通过 `GET /__openclaw/control-ui-config.json` 获取 `serverVersion`：

```typescript
sendJson(res, 200, {
  basePath,
  assistantName: identity.name,
  // ...
  serverVersion: resolveRuntimeServiceVersion(process.env),
} satisfies ControlUiBootstrapConfig);
```

**版本解析**（`src/version.ts:105-119`）：

```typescript
export function resolveRuntimeServiceVersion(
  env: RuntimeVersionEnv = process.env,
  fallback = RUNTIME_SERVICE_VERSION_FALLBACK,
): string {
  return (
    firstNonEmpty(
      env["OPENCLAW_VERSION"],
      resolveUsableRuntimeVersion(VERSION),
      env["OPENCLAW_SERVICE_VERSION"],
      env["npm_package_version"],
    ) ?? fallback
  );
}
```

**同源策略**（`ui/src/ui/app-gateway.ts:88-114`）：`resolveControlUiClientVersion()` 仅在 **同源**（`page.host === gateway.host`）时向前端传递 `serverVersion`，跨域目标不传递，防止版本信息泄露到其他站点。

**WebSocket hello**（`src/gateway/server/ws-connection/message-handler.ts:1035`）：握手响应中也包含 `version: resolveRuntimeServiceVersion(process.env)`，供客户端显示。

---

## 十、总结

### 10.1 Gateway 架构亮点

1. **统一控制平面**：
   - 所有客户端通过同一 WebSocket 协议连接
   - 角色+权限系统（Operator vs Node，Scopes 控制）
   - 协议版本控制（避免客户端/服务端不兼容）

2. **服务发现多层次**：
   - Bonjour/mDNS（局域网零配置）
   - Tailscale（跨网络私有连接）
   - SSH Tunnel（通用回退方案）

3. **配置热重载**：
   - 文件监控+防抖处理
   - 增量更新（Channel、Plugin、Heartbeat、Skills）
   - 无需重启 Gateway

4. **慢消费者保护**：
   - 缓冲区监控（`bufferedAmount`）
   - 分级处理（dropIfSlow、断开连接）
   - 避免阻塞其他客户端

5. **插件化架构**：
   - Gateway Methods 扩展
   - Channel 扩展
   - 动态加载+自动注册

### 10.2 与 AI 主动性的关联

| AI 主动性特性 | Gateway 集成方式 | 关键代码位置 |
|-------------|----------------|------------|
| **Heartbeat** | 启动时初始化 Runner，定时触发 Agent 执行 | `src/gateway/server.impl.ts:414` |
| **Memory** | （推测）插件系统初始化，监听 Session Transcript 更新 | `extensions/memory-*`（待研究） |
| **Skills** | 注册文件监听器，30 秒防抖刷新远程节点 | `src/gateway/server.impl.ts:368-380` |

**Heartbeat 生命周期**：
```
Gateway 启动 → startHeartbeatRunner
→ 定时器触发 → 检查队列
→ 读取 HEARTBEAT.md → 构造 Agent 请求
→ Agent 执行 → 检测 HEARTBEAT_OK
→ 判断是否发送 → 广播事件
```

**Skills 刷新机制**：
```
Gateway 启动 → registerSkillsChangeListener
→ chokidar 监控文件 → 防抖 30 秒
→ bumpSkillsSnapshotVersion → 触发监听器
→ 刷新远程节点 Skills → Agent 下次运行时加载
```

### 10.3 新增系统（2026.3.22+ 代码更新）

#### Exec Approval 执行审批系统

通过独立的 `createExecApprovalHandlers` / `createSecretsHandlers` 注入到 `extraHandlers`，不在 `coreGatewayHandlers` 中：

```typescript
extraHandlers: {
  ...pluginRegistry.gatewayHandlers,
  ...execApprovalHandlers,   // exec.approval.request/waitDecision/resolve
  ...secretsHandlers,        // secrets.reload/resolve
}
```

审批流程：Agent 调用 exec → 发出 `exec.approval.requested` 事件 → Operator 客户端弹出审批 → 调用 `exec.approval.resolve` → Agent 继续。

#### WebChat WebSocket-native 方法

`chat.send`, `chat.abort`, `chat.history` 是为 WebChat 前端专门实现的 Gateway-native 方法，与通过 `sessions.*` 的普通会话通信并列，提供更低延迟的浏览器聊天体验。

#### Agent CRUD 管理

`agents.create/update/delete` 和 `agents.files.list/get/set` 实现对 Agent 配置文件（`IDENTITY.md`、`SOUL.md` 等）的远程读写，供 Control UI 和 CLI 管理 Agent 身份。

#### ClawHub 插件市场集成（2026.3.22）

`skills.install` / `skills.update` RPC 方法支持从 ClawHub 市场安装和更新插件包：

```bash
# 通过 CLI 路由到 Gateway
openclaw plugins install <name>
openclaw skills install <name>
```

底层流程：CLI → Gateway `skills.install` RPC → 从 ClawHub 拉取包 → 安装到本地 skills 目录 → 触发 `bumpSkillsSnapshotVersion`。

### 10.4 待深入研究的问题

- [ ] Memory 初始化的具体位置和时机（可能在插件系统中）
- [ ] Memory 与 Gateway 生命周期的集成方式
- [ ] Wide-Area DNS-SD 的完整实现细节
- [ ] Gateway 多实例部署的协调机制

---

**研究小结**：

璇玑通过深入研究 Gateway 的代码实现和官方文档，发现 Gateway 是 OpenClaw 架构的核心枢纽，统一管理所有客户端连接、Agent 运行、Heartbeat 调度、Skills 刷新等功能。它的设计理念（统一控制平面、配置热重载、慢消费者保护、插件化架构）非常值得学习和借鉴！✨

尤其是 Heartbeat 和 Skills 与 Gateway 的深度集成，展现了如何在 Gateway 层面支撑 AI 主动性特性，让 AI 从"被动响应工具"进化为"主动察觉助手"的关键基础设施设计。👏
