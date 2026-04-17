# OpenClaw 安全与沙箱设计深度研究

## 📋 版本信息

- **研究日期**: 2026-03-05
- **项目**: OpenClaw (https://github.com/openclaw/openclaw)
- **Commit**: 73055728318df378c831950cd01fb7c875a33790 (2026-03-05, version 2026.3.3)
- **研究者**: 璇玑

---

## 🎯 研究目标

深度剖析 OpenClaw 的安全与沙箱设计，理解其如何在"让 AI 拥有 shell 访问权限"这一高风险场景下，通过多层防御机制实现威胁控制。

**核心问题**：
- 如何隔离非可信 session 的执行环境？（Docker 沙箱）
- 如何防止未授权的 DM 访问？（Pairing 机制）
- 如何细粒度控制工具权限？（Tool Policy）
- 如何记录和审计安全事件？（审计日志）
- 如何保障 Heartbeat/Memory/Skills 三大 AI 主动性特性的安全执行？

---

## 🏗️ 安全架构总览

OpenClaw 采用**多层纵深防御**策略，从外到内依次为：

```
┌─────────────────────────────────────────────────────────────┐
│ 第1层：身份认证 (Identity & Access Control)                 │
│  - DM Pairing (pairing code + TTL)                         │
│  - Group Allowlist (mention gating)                        │
│  - Gateway Auth (token/password)                           │
│  - Device Pairing (WebSocket clients)                      │
└─────────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────┐
│ 第2层：工具权限控制 (Tool Policy)                            │
│  - Global Tool Policy (tools.allow/deny)                   │
│  - Agent Tool Policy (agents.list[].tools)                 │
│  - Sandbox Tool Policy (tools.sandbox.tools)               │
│  - Tool Groups (group:runtime, group:fs, etc.)             │
└─────────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────┐
│ 第3层：执行环境隔离 (Sandboxing)                             │
│  - Docker 沙箱 (network=none, read-only root)              │
│  - Workspace 隔离 (none/ro/rw)                             │
│  - Elevated Exec 逃逸控制 (explicit approval)              │
└─────────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────┐
│ 第4层：审计与监控 (Audit & Monitoring)                       │
│  - Security Audit (`openclaw security audit`)              │
│  - Exec Approval Manager (高风险操作审批)                   │
│  - Session Transcript (JSONL 日志)                         │
│  - 形式化验证 (TLA+ 模型)                                    │
└─────────────────────────────────────────────────────────────┘
```

---

## 🔐 第1层：身份认证 (Pairing 机制)

### 1.1 DM Pairing 核心实现

**文件**: `src/pairing/pairing-store.ts` (497 行)

**设计目标**: 防止陌生人通过 DM 触发 AI，同时保持用户友好的审批流程。

**Pairing Code 生成**:
```typescript
// src/pairing/pairing-store.ts:186-194
function randomCode(): string {
  // Human-friendly: 8 chars, upper, no ambiguous chars (0O1I).
  let out = "";
  for (let i = 0; i < PAIRING_CODE_LENGTH; i++) {
    const idx = crypto.randomInt(0, PAIRING_CODE_ALPHABET.length);
    out += PAIRING_CODE_ALPHABET[idx];
  }
  return out;
}

// 常量定义
const PAIRING_CODE_LENGTH = 8;
const PAIRING_CODE_ALPHABET = "ABCDEFGHJKLMNPQRSTUVWXYZ23456789"; // 排除 0O1I
```

**关键特性**:
- **8位大写字母**: 易于口头传达，避免混淆字符
- **1小时 TTL**: `PAIRING_PENDING_TTL_MS = 60 * 60 * 1000` (pairing-store.ts:12)
- **最多3个 Pending**: `PAIRING_PENDING_MAX = 3` (pairing-store.ts:13)，防止 DoS 攻击
- **文件锁机制**: 使用 `proper-lockfile` 保证并发安全 (pairing-store.ts:14-23)

**存储位置**:
- Pairing 请求: `~/.openclaw/credentials/<channel>-pairing.json`
- Allow List: `~/.openclaw/credentials/<channel>-allowFrom.json`

**Pairing 流程**:
```typescript
// 1. 陌生人发送 DM → 生成 pairing code (upsertChannelPairingRequest)
// 2. AI 回复 8 位 code，等待批准
// 3. 用户执行 `openclaw pairing approve <channel> <code>`
// 4. 批准后，sender ID 写入 allowFrom.json (approveChannelPairingCode)
// 5. 下次同一 sender 发送 DM，直接通过
```

**并发安全**:
```typescript
// src/pairing/pairing-store.ts:121-140
async function withFileLock<T>(
  filePath: string,
  fallback: unknown,
  fn: () => Promise<T>,
): Promise<T> {
  await ensureJsonFile(filePath, fallback);
  let release: (() => Promise<void>) | undefined;
  try {
    release = await lockfile.lock(filePath, PAIRING_STORE_LOCK_OPTIONS);
    return await fn();
  } finally {
    if (release) {
      try {
        await release();
      } catch {
        // ignore unlock errors
      }
    }
  }
}
```

**TTL 和容量控制**:
```typescript
// src/pairing/pairing-store.ts:153-184
function isExpired(entry: PairingRequest, nowMs: number): boolean {
  const createdAt = parseTimestamp(entry.createdAt);
  if (!createdAt) {
    return true;
  }
  return nowMs - createdAt > PAIRING_PENDING_TTL_MS; // 1小时过期
}

function pruneExcessRequests(reqs: PairingRequest[], maxPending: number) {
  if (maxPending <= 0 || reqs.length <= maxPending) {
    return { requests: reqs, removed: false };
  }
  // 按 lastSeenAt 排序，保留最新的 maxPending 个
  const sorted = reqs.slice().toSorted((a, b) => resolveLastSeenAt(a) - resolveLastSeenAt(b));
  return { requests: sorted.slice(-maxPending), removed: true };
}
```

### 1.2 四种 DM 策略

**配置路径**: `channels.<channel>.dmPolicy` (或 `channels.<channel>.dm.policy`)

| 策略 | 行为 | 安全等级 | 适用场景 |
|------|------|----------|----------|
| `pairing` | 陌生人收到 8 位 code，需批准 | ✅ 高 | **默认推荐**，个人助手 |
| `allowlist` | 陌生人直接被拒，无 pairing | ✅ 最高 | 封闭系统，已知用户列表 |
| `open` | 任何人都可以 DM | ⚠️ 危险 | 公开 bot（需显式 `allowFrom: ["*"]`） |
| `disabled` | 完全忽略 DM | ✅ 安全 | 只需 group chat |

**源码位置**: `docs/gateway/security/index.md:180-197`

### 1.3 Gateway 认证

**文件**: `src/gateway/auth.ts`, `src/gateway/device-auth.ts`

**三种认证方式**:
1. **Token Auth**: `gateway.auth.mode: "token"`, 共享 Bearer Token
2. **Password Auth**: `gateway.auth.mode: "password"`, 密码认证
3. **Tailscale Identity**: `gateway.auth.allowTailscale: true`, Tailscale Serve 身份头

**Device Pairing Payload**:
```typescript
// src/gateway/device-auth.ts:13-31
export function buildDeviceAuthPayload(params: DeviceAuthPayloadParams): string {
  const version = params.version ?? (params.nonce ? "v2" : "v1");
  const scopes = params.scopes.join(",");
  const token = params.token ?? "";
  const base = [
    version,
    params.deviceId,
    params.clientId,
    params.clientMode,
    params.role,
    scopes,
    String(params.signedAtMs),
    token,
  ];
  if (version === "v2") {
    base.push(params.nonce ?? "");
  }
  return base.join("|");
}
```

**本地设备自动批准**:
- Loopback 连接自动信任
- 同一 Tailnet 地址的 Gateway host 自动信任
- 其他 Tailnet peer 仍需手动批准

### 1.4 Gateway 默认 HTTP 安全头

**文件**: `src/gateway/http-common.ts`

Gateway 对所有 HTTP 响应应用基线安全头（API JSON、HTML 页面、静态资源、SSE 流均适用）：

```typescript
// src/gateway/http-common.ts:11-21
export function setDefaultSecurityHeaders(res, opts?) {
  res.setHeader("X-Content-Type-Options", "nosniff");
  res.setHeader("Referrer-Policy", "no-referrer");
  res.setHeader("Permissions-Policy", "camera=(), microphone=(), geolocation=()");
  // ...
}
```

**Permissions-Policy** 显式禁用摄像头、麦克风、地理定位，降低 Web 页面被滥用访问设备传感器的风险。

---

## 🛠️ 第2层：工具权限控制 (Tool Policy)

### 2.1 三层 Tool Policy 架构

**文件**: `src/agents/sandbox/tool-policy.ts` (143 行)

**权限计算顺序**:
```
1. Agent-specific sandbox policy (agents.list[].tools.sandbox.tools)
   ↓ (如果未定义)
2. Global sandbox policy (tools.sandbox.tools)
   ↓ (如果未定义)
3. Default sandbox policy (DEFAULT_TOOL_ALLOW / DEFAULT_TOOL_DENY)
```

**核心规则**:
- **Deny 优先**: `deny` 列表中的工具直接拒绝，不可覆盖
- **Allow Whitelist**: 如果 `allow` 非空，则只允许列表中的工具
- **Allow Empty = 全部允许**: `allow` 为空时，除 `deny` 外的工具全部允许

**实现代码**:
```typescript
// src/agents/sandbox/tool-policy.ts:58-69
export function isToolAllowed(policy: SandboxToolPolicy, name: string) {
  const normalized = name.trim().toLowerCase();
  const deny = compilePatterns(policy.deny);
  if (matchesAny(normalized, deny)) {
    return false; // Deny 优先
  }
  const allow = compilePatterns(policy.allow);
  if (allow.length === 0) {
    return true; // Allow 为空 = 全部允许
  }
  return matchesAny(normalized, allow);
}
```

### 2.2 Tool Groups (快捷方式)

**支持的 Tool Groups**:
```typescript
// 根据 docs/gateway/sandbox-vs-tool-policy-vs-elevated.md:86-97
- group:runtime      → exec, bash, process
- group:fs           → read, write, edit, apply_patch
- group:sessions     → sessions_list, sessions_history, sessions_send, sessions_spawn, session_status
- group:memory       → memory_search, memory_get
- group:ui           → browser, canvas
- group:automation   → cron, gateway
- group:messaging    → message
- group:nodes        → nodes
- group:openclaw     → 所有内置 OpenClaw 工具（不含 provider 插件）
```

**使用示例**:
```json5
{
  tools: {
    sandbox: {
      tools: {
        allow: ["group:runtime", "group:fs", "group:sessions", "group:memory"],
        deny: ["exec", "bash"] // 即使在 group:runtime 中，也会被拒绝
      }
    }
  }
}
```

### 2.3 Tool Policy 解析流程

**文件**: `src/agents/sandbox/tool-policy.ts:71-142`

```typescript
export function resolveSandboxToolPolicyForAgent(
  cfg?: OpenClawConfig,
  agentId?: string,
): SandboxToolPolicyResolved {
  // 1. 读取 agent-specific 配置
  const agentConfig = cfg && agentId ? resolveAgentConfig(cfg, agentId) : undefined;
  const agentAllow = agentConfig?.tools?.sandbox?.tools?.allow;
  const agentDeny = agentConfig?.tools?.sandbox?.tools?.deny;
  
  // 2. 读取 global 配置
  const globalAllow = cfg?.tools?.sandbox?.tools?.allow;
  const globalDeny = cfg?.tools?.sandbox?.tools?.deny;

  // 3. 决定 allow 来源 (agent > global > default)
  const allow = Array.isArray(agentAllow)
    ? agentAllow
    : Array.isArray(globalAllow)
      ? globalAllow
      : [...DEFAULT_TOOL_ALLOW];

  // 4. 决定 deny 来源 (agent > global > default)
  const deny = Array.isArray(agentDeny)
    ? agentDeny
    : Array.isArray(globalDeny)
      ? globalDeny
      : [...DEFAULT_TOOL_DENY];

  // 5. 展开 Tool Groups
  const expandedDeny = expandToolGroups(deny);
  let expandedAllow = expandToolGroups(allow);

  // 6. 强制包含 image tool (多模态工作流必需)
  if (
    !expandedDeny.map((v) => v.toLowerCase()).includes("image") &&
    !expandedAllow.map((v) => v.toLowerCase()).includes("image")
  ) {
    expandedAllow = [...expandedAllow, "image"];
  }

  return {
    allow: expandedAllow,
    deny: expandedDeny,
    sources: { allow: allowSource, deny: denySource }
  };
}
```

**默认值**:
```typescript
// src/agents/sandbox/constants.ts (推测)
export const DEFAULT_TOOL_ALLOW = []; // 空 = 全部允许
export const DEFAULT_TOOL_DENY = [];  // 空 = 不拒绝任何工具
```

---

## 🐳 第3层：Docker 沙箱隔离

### 3.1 沙箱类型定义

**文件**: `src/agents/sandbox/types.ts` (86 行)

**核心类型**:
```typescript
export type SandboxConfig = {
  mode: "off" | "non-main" | "all";      // 何时启用沙箱
  scope: "session" | "agent" | "shared"; // 容器作用域
  workspaceAccess: "none" | "ro" | "rw"; // Workspace 挂载模式
  workspaceRoot: string;                  // 沙箱 workspace 根目录
  docker: SandboxDockerConfig;            // Docker 容器配置
  browser: SandboxBrowserConfig;          // 沙箱浏览器配置
  tools: SandboxToolPolicy;               // 沙箱工具策略
  prune: SandboxPruneConfig;              // 容器清理策略
};
```

### 3.2 三种沙箱模式

| 模式 | 行为 | 适用场景 |
|------|------|----------|
| `off` | 关闭沙箱，所有工具在 host 执行 | 个人助手，完全信任 |
| `non-main` | 只沙箱非 main session | **默认推荐**，保护 group chat |
| `all` | 所有 session 都沙箱 | 公开 bot，最高安全 |

**源码位置**: `docs/gateway/sandboxing.md:35-44`

### 3.3 三种沙箱作用域

| 作用域 | 容器数量 | 隔离级别 | 适用场景 |
|--------|----------|----------|----------|
| `session` | 每个 session 一个容器 | 最高 | 多用户场景，防止跨 session 数据泄露 |
| `agent` | 每个 agent 一个容器 | 中等 | **默认推荐**，平衡性能和隔离 |
| `shared` | 所有 session 共享一个容器 | 最低 | 单用户，性能优先 |

**源码位置**: `docs/gateway/sandboxing.md:45-52`

### 3.4 Workspace 挂载模式

| 模式 | 行为 | 可用工具 | 适用场景 |
|------|------|----------|----------|
| `none` | 不挂载 agent workspace，工具操作 `~/.openclaw/sandboxes` | 全部 | **默认推荐**，完全隔离 |
| `ro` | 只读挂载到 `/agent` | read only (write/edit/apply_patch 禁用) | 只读助手 |
| `rw` | 读写挂载到 `/workspace` | 全部 | 需要修改 workspace |

**源码位置**: `docs/gateway/sandboxing.md:53-66`

### 3.5 Docker 容器创建参数

**文件**: `src/agents/sandbox/docker.ts:125-231`

**关键安全参数**:
```typescript
// src/agents/sandbox/docker.ts:125-231
export function buildSandboxCreateArgs(params: {
  name: string;
  cfg: SandboxDockerConfig;
  scopeKey: string;
  createdAtMs?: number;
  labels?: Record<string, string>;
  configHash?: string;
}) {
  const args = ["create", "--name", params.name];
  
  // 标签（用于容器管理和清理）
  args.push("--label", "openclaw.sandbox=1");
  args.push("--label", `openclaw.sessionKey=${params.scopeKey}`);
  args.push("--label", `openclaw.createdAtMs=${createdAtMs}`);
  
  // 只读根文件系统 (防止持久化恶意代码)
  if (params.cfg.readOnlyRoot) {
    args.push("--read-only");
  }
  
  // tmpfs 挂载 (临时文件系统，容器销毁后清空)
  for (const entry of params.cfg.tmpfs) {
    args.push("--tmpfs", entry); // 默认: ["/tmp", "/var/tmp", "/run"]
  }
  
  // 网络隔离 (默认: "none")
  if (params.cfg.network) {
    args.push("--network", params.cfg.network);
  }
  
  // 用户身份 (默认: 非 root)
  if (params.cfg.user) {
    args.push("--user", params.cfg.user);
  }
  
  // 去除所有 capabilities (最小权限原则)
  for (const cap of params.cfg.capDrop) {
    args.push("--cap-drop", cap); // 默认: ["ALL"]
  }
  
  // 禁止提升特权
  args.push("--security-opt", "no-new-privileges");
  
  // Seccomp/AppArmor 配置
  if (params.cfg.seccompProfile) {
    args.push("--security-opt", `seccomp=${params.cfg.seccompProfile}`);
  }
  if (params.cfg.apparmorProfile) {
    args.push("--security-opt", `apparmor=${params.cfg.apparmorProfile}`);
  }
  
  // 资源限制
  if (typeof params.cfg.pidsLimit === "number" && params.cfg.pidsLimit > 0) {
    args.push("--pids-limit", String(params.cfg.pidsLimit));
  }
  if (params.cfg.memory) {
    args.push("--memory", normalizeDockerLimit(params.cfg.memory));
  }
  if (params.cfg.cpus) {
    args.push("--cpus", String(params.cfg.cpus));
  }
  
  // Bind mounts (需谨慎使用，会绕过沙箱隔离)
  if (params.cfg.binds?.length) {
    for (const bind of params.cfg.binds) {
      args.push("--mount", `type=bind,${bind}`);
    }
  }
  
  // 镜像
  args.push(params.cfg.image);
  
  // 工作目录和启动命令
  args.push("sleep", "infinity");
  
  return args;
}
```

**默认安全配置**:
```typescript
// src/agents/sandbox/config.ts:39-82
export function resolveSandboxDockerConfig(params: {
  scope: SandboxScope;
  globalDocker?: Partial<SandboxDockerConfig>;
  agentDocker?: Partial<SandboxDockerConfig>;
}): SandboxDockerConfig {
  return {
    image: DEFAULT_SANDBOX_IMAGE, // "openclaw-sandbox:bookworm-slim"
    containerPrefix: DEFAULT_SANDBOX_CONTAINER_PREFIX, // "openclaw-sandbox"
    workdir: DEFAULT_SANDBOX_WORKDIR, // "/workspace"
    readOnlyRoot: true,               // ✅ 只读根文件系统
    tmpfs: ["/tmp", "/var/tmp", "/run"], // ✅ 临时文件系统
    network: "none",                  // ✅ 无网络访问
    user: undefined,                  // 默认非 root
    capDrop: ["ALL"],                 // ✅ 去除所有 capabilities
    env: { LANG: "C.UTF-8" },
    setupCommand: undefined,
    pidsLimit: undefined,
    memory: undefined,
    memorySwap: undefined,
    cpus: undefined,
    ulimits: undefined,
    seccompProfile: undefined,
    apparmorProfile: undefined,
    dns: undefined,
    extraHosts: undefined,
    binds: undefined,
  };
}
```

### 3.6 Elevated Exec 逃逸机制

**设计目标**: 允许受信任的操作在 host 上执行高权限操作，同时保持审批控制。

**三个控制点**:
1. **Enablement**: `tools.elevated.enabled` (global) + `agents.list[].tools.elevated.enabled` (per-agent)
2. **Sender Allowlist**: `tools.elevated.allowFrom.<provider>` (仅允许特定 sender 使用)
3. **Exec Approval**: 可选的"Ask"模式，每次执行前需人工批准

**工作流程**:
```
1. 用户在 DM 中发送 `/elevated on`
2. Agent 在 sandboxed session 中尝试执行 `exec` with `elevated: true`
3. Gateway 检查:
   - elevated 是否启用？(tools.elevated.enabled)
   - sender 是否在 allowlist？(tools.elevated.allowFrom)
   - 是否需要 approval？(security/ask 配置)
4. 如果通过，exec 在 **host** 上执行，绕过沙箱
5. 结果返回给 agent
```

**⚠️ 重要限制**:
- Elevated 只影响 `exec` 工具，不影响其他工具
- `/exec` 指令只改变 session 默认设置，不能覆盖 tool policy deny
- 如果 `exec` 在 global/agent tool policy 中被 deny，elevated 也无法执行

**源码位置**: `docs/gateway/sandbox-vs-tool-policy-vs-elevated.md:98-114`

---

## 🔍 第4层：审计与监控

### 4.1 Security Audit 工具

**命令**: `openclaw security audit [--deep] [--fix]`

**文件**: `src/security/audit.ts` (986+ 行)

**审计维度**:
```typescript
// src/security/audit.ts:33-62
export type SecurityAuditReport = {
  ts: number;
  summary: SecurityAuditSummary;  // { critical, warn, info }
  findings: SecurityAuditFinding[];
  deep?: {
    gateway?: {
      attempted: boolean;
      url: string | null;
      ok: boolean;
      error: string | null;
    };
  };
};
```

**检查项分类**:
1. **Inbound Access** (DM/group policies)
   - `dmPolicy="open"` without explicit `allowFrom: ["*"]`
   - `groupPolicy="open"` in channels with tools enabled
   - Group allowlists missing or too broad

2. **Tool Blast Radius**
   - Elevated tools + open rooms
   - Sandbox mode off + untrusted access

3. **Network Exposure**
   - Gateway bind beyond loopback without auth
   - Tailscale Serve/Funnel + weak auth
   - Short/weak auth tokens

4. **Browser Control Exposure**
   - Remote nodes without pairing
   - Relay ports exposed to LAN/public

5. **Filesystem Hygiene**
   - `~/.openclaw` permissions (should be 700)
   - `openclaw.json` permissions (should be 600)
   - State files world-readable/writable
   - Symlinks in critical paths

6. **Plugins**
   - Extensions exist without explicit allowlist

7. **gateway.nodes.denyCommands 无效条目**
   - 仅支持**精确命令名匹配**（如 `system.run`），不支持通配符（如 `system.*`）或 shell 文本过滤
   - 审计会对未知/无效条目**建议可能的精确 node command ID**（如 `system.run.prep` → `did you mean: system.run`），便于用户修正
   - **源码**: `src/security/audit-extra.sync.ts:970-1024` (`collectNodeDenyCommandPatternFindings`)，`suggestKnownNodeCommands` 提供前缀匹配和模糊建议

8. **Model Hygiene**
   - Configured models look legacy (< 2024)

**关键检查逻辑**:
```typescript
// src/security/audit.ts:126-192 (Filesystem Findings)
async function collectFilesystemFindings(params: {
  stateDir: string;
  configPath: string;
  env?: NodeJS.ProcessEnv;
  platform?: NodeJS.Platform;
}): Promise<SecurityAuditFinding[]> {
  const findings: SecurityAuditFinding[] = [];

  // 检查 state dir 权限
  const stateDirPerms = await inspectPathPermissions(params.stateDir, {...});
  if (stateDirPerms.ok) {
    if (stateDirPerms.worldWritable) {
      findings.push({
        checkId: "fs.state_dir.perms_world_writable",
        severity: "critical", // 🔥 严重
        title: "State dir is world-writable",
        detail: `${params.stateDir} is world-writable; other users can write into your OpenClaw state.`,
        remediation: `chmod 700 ${params.stateDir}`,
      });
    } else if (stateDirPerms.groupWritable) {
      findings.push({
        checkId: "fs.state_dir.perms_group_writable",
        severity: "warn", // ⚠️ 警告
        title: "State dir is group-writable",
        detail: `${params.stateDir} is group-writable; group users can write into your OpenClaw state.`,
        remediation: `chmod 700 ${params.stateDir}`,
      });
    }
  }

  // 检查 config 文件权限
  const configPerms = await inspectPathPermissions(params.configPath, {...});
  if (configPerms.ok) {
    if (configPerms.worldWritable || configPerms.groupWritable) {
      findings.push({
        checkId: "fs.config.perms_writable",
        severity: "critical",
        title: "Config file is writable by others",
        detail: `${params.configPath} is writable by others; anyone can modify your OpenClaw config.`,
        remediation: `chmod 600 ${params.configPath}`,
      });
    }
  }

  return findings;
}
```

### 4.2 Exec Approval Manager

**文件**: `src/gateway/exec-approval-manager.ts` (83 行)

**设计目标**: 为高风险 exec 操作提供人工审批机制。

**核心数据结构**:
```typescript
// src/gateway/exec-approval-manager.ts:4-23
export type ExecApprovalRequestPayload = {
  command: string;         // 待执行的命令
  cwd?: string | null;     // 工作目录
  host?: string | null;    // 执行节点
  security?: string | null; // 安全策略
  ask?: string | null;     // Ask 模式配置
  agentId?: string | null;
  resolvedPath?: string | null;
  sessionKey?: string | null;
};

export type ExecApprovalRecord = {
  id: string;              // UUID
  request: ExecApprovalRequestPayload;
  createdAtMs: number;
  expiresAtMs: number;
  resolvedAtMs?: number;
  decision?: ExecApprovalDecision; // "approve" | "deny"
  resolvedBy?: string | null;      // 批准人
};
```

**工作流程**:
```typescript
// src/gateway/exec-approval-manager.ts:32-82
export class ExecApprovalManager {
  private pending = new Map<string, PendingEntry>();

  // 1. 创建审批请求
  create(request: ExecApprovalRequestPayload, timeoutMs: number, id?: string | null): ExecApprovalRecord {
    const now = Date.now();
    const resolvedId = id && id.trim().length > 0 ? id.trim() : randomUUID();
    return {
      id: resolvedId,
      request,
      createdAtMs: now,
      expiresAtMs: now + timeoutMs,
    };
  }

  // 2. 等待决策 (异步阻塞)
  async waitForDecision(record: ExecApprovalRecord, timeoutMs: number): Promise<ExecApprovalDecision | null> {
    return await new Promise<ExecApprovalDecision | null>((resolve, reject) => {
      const timer = setTimeout(() => {
        this.pending.delete(record.id);
        resolve(null); // 超时返回 null
      }, timeoutMs);
      this.pending.set(record.id, { record, resolve, reject, timer });
    });
  }

  // 3. 批准/拒绝 (由用户通过 CLI/UI 触发)
  resolve(recordId: string, decision: ExecApprovalDecision, resolvedBy?: string | null): boolean {
    const pending = this.pending.get(recordId);
    if (!pending) {
      return false;
    }
    clearTimeout(pending.timer);
    pending.record.resolvedAtMs = Date.now();
    pending.record.decision = decision;
    pending.record.resolvedBy = resolvedBy ?? null;
    this.pending.delete(recordId);
    pending.resolve(decision); // 唤醒等待的 agent
    return true;
  }
}
```

**使用场景**:
- macOS node 的 `system.run` (远程代码执行)
- Elevated exec on host (需要 `ask` 模式)
- 任何标记为 "需要批准" 的命令

### 4.3 Session Transcript 审计日志

**存储位置**: `~/.openclaw/agents/<agentId>/sessions/*.jsonl`

**格式**: JSONL (每行一个 JSON 对象)

**包含信息**:
- 用户消息
- Agent 回复
- Tool 调用（参数、返回值）
- Thinking 内容（推理过程）
- 错误和异常

**安全考虑**:
- **敏感信息泄露**: Transcript 可能包含 API key、密码、私密对话
- **权限控制**: 文件权限应为 `600`，只有 owner 可读写
- **Redaction**: `logging.redactSensitive: "tools"` 可自动脱敏 tool 输出

**源码位置**: `docs/gateway/security/index.md:495-507`

### 4.4 形式化验证 (TLA+ 模型)

**文档**: `docs/gateway/security/formal-verification.md`

**目标**: 用 TLA+/TLC 模型检查器验证核心安全属性。

**已验证的模型**:
1. **Gateway Exposure**: bind 模式 + auth 配置的安全性
2. **Nodes.run Pipeline**: 命令 allowlist + approval 机制
3. **Pairing Store**: TTL + pending cap + 并发安全
4. **Ingress Gating**: Mention gating + 控制命令绕过检查
5. **Routing Isolation**: DM session 隔离 + identityLinks
6. **Pairing Concurrency**: 并发 pairing 请求的 idempotency
7. **Ingress Trace Correlation**: 消息去重和 trace ID 传播

**运行方式**:
```bash
git clone https://github.com/vignesh07/openclaw-formal-models
cd openclaw-formal-models

# 验证 Gateway 暴露模型
make gateway-exposure-v2
make gateway-exposure-v2-protected

# 验证 Pairing store 并发安全
make pairing-race
make pairing-idempotency

# 负面测试（预期失败，验证模型有效）
make pairing-race-negative
```

**关键发现**:
- ✅ Pairing store 的 pending cap 在并发场景下可能被突破（非原子 check-then-write）
- ✅ 需要文件锁保证原子性（已实现：`proper-lockfile`）
- ✅ Gateway 在 `bind="lan"` + `auth.mode="off"` 时可被远程攻击（模型验证通过）

**源码位置**: `docs/gateway/security/formal-verification.md`

---

## 🔥 与 AI 主动性的关联分析

### 5.1 Heartbeat 执行时的安全控制

**问题**: Heartbeat 定时"醒来"执行检查时，如何保证不会执行恶意操作？

**答案**:
1. **Session 隔离**: Heartbeat 在 main session 中执行，可以配置为 `sandbox.mode: "off"` (完全信任)
2. **Tool Policy**: Heartbeat 可以访问哪些工具，由 `agents.list[].tools` 控制
3. **Sensitive Operations**: 即使是 main session，执行 `exec` 等敏感操作时，仍可启用 approval

**配置示例**:
```json5
{
  agents: {
    list: [
      {
        id: "main",
        sandbox: { mode: "off" }, // Heartbeat 不沙箱
        tools: {
          allow: ["read", "web_fetch", "message", "memory_search"], // 只允许安全工具
          deny: ["exec", "bash", "write", "edit"] // 禁止高风险操作
        }
      }
    ]
  }
}
```

**HEARTBEAT_OK 静默机制**:
- Heartbeat 生成的消息如果内容是 "HEARTBEAT_OK"，不会发送给用户
- 防止"一切正常"时频繁打扰用户
- 但仍会记录到 Session Transcript

### 5.2 Memory 文件的隔离与权限控制

**问题**: Memory 索引的文件是否跨 Agent 共享？SQLite 文件权限如何控制？

**答案**:
1. **Per-Agent Memory**: 每个 Agent 有独立的 Memory DB (`~/.openclaw/agents/<agentId>/memory/`)
2. **文件权限**: SQLite 文件应为 `600`，只有 owner 可读写
3. **Sandbox 访问**: Sandboxed session 如何访问 Memory？
   - `memory_search` tool 由 Gateway 实现，不在沙箱内执行
   - Sandboxed session 通过 RPC 调用 Gateway 的 `memory_search`
   - Gateway 根据 session 的 agent ID 路由到对应的 Memory DB

**存储结构**:
```
~/.openclaw/
  └── agents/
      ├── main/
      │   └── memory/
      │       ├── embeddings.db       # SQLite + sqlite-vec
      │       └── chunks/              # 原始文本块
      └── family/
          └── memory/
              ├── embeddings.db
              └── chunks/
```

**跨 Agent 隔离**:
- Agent `main` 的 session 无法访问 Agent `family` 的 Memory
- 即使是同一用户，不同 Agent 的 Memory 完全隔离

### 5.3 第三方 Skills 的安全审计

**问题**: 如何防止恶意 Skills？Skills 的沙箱隔离策略？

**答案**:
1. **Skills 不在沙箱内执行**: Skills 是 Markdown 文档（SKILL.md），注入到 system prompt 中
2. **Skills 只影响 Agent 行为**: 不直接执行代码，只提供指令和示例
3. **Tool Policy 仍然生效**: 即使 Skill 建议使用某个 tool，Tool Policy 仍会拒绝未授权的工具
4. **ClawdHub Registry**: 公开 Skills 注册表，社区审查
5. **Plugin Skills**: Plugin 自带的 Skills，需要信任 Plugin 本身

**安全建议**:
- 只安装来自可信来源的 Skills（官方、知名作者）
- 审查 Skills 内容（SKILL.md 是纯文本，可人工审查）
- 使用 `tools.deny` 禁止 Skills 可能建议的危险工具

**例外**: 如果 Plugin 提供的是 **Tool**（而非 Skill），那么该 Tool 在 sandboxed session 中执行时会受沙箱隔离。

---

## 🎯 可借鉴的设计模式

### 1. 多层纵深防御

**核心理念**: "安全不是单点防御，而是多层栅栏"

**借鉴点**:
- 身份认证（Pairing）→ 工具权限（Tool Policy）→ 执行隔离（Sandbox）→ 审计监控（Audit）
- 即使一层被突破，后续层仍能限制损害
- 对于 AI Agent，这尤其重要，因为 prompt injection 无法完全防御

### 2. Pairing 机制的人机友好设计

**核心理念**: "安全审批不应成为用户负担"

**借鉴点**:
- 8 位大写字母（易口头传达，避免混淆）
- 1 小时 TTL（平衡安全和便利）
- 最多 3 个 pending（防止 DoS，同时不影响正常使用）
- 文件锁保证并发安全（proper-lockfile）

### 3. Tool Policy 的分层覆盖

**核心理念**: "全局默认 + Per-Agent 覆盖 + Sandbox 额外限制"

**借鉴点**:
- Global → Agent → Sandbox 三层策略
- Deny 优先，Allow Whitelist
- Tool Groups 快捷方式（`group:runtime`）
- 强制包含必需工具（`image` for 多模态）

### 4. Docker 沙箱的最小权限配置

**核心理念**: "默认拒绝一切，只开放必需的"

**借鉴点**:
- `network: "none"` (无网络)
- `readOnlyRoot: true` (只读根文件系统)
- `capDrop: ["ALL"]` (去除所有 capabilities)
- `no-new-privileges` (禁止提升特权)
- `tmpfs` 挂载（临时文件系统，容器销毁后清空）

### 5. Elevated Exec 的显式逃逸

**核心理念**: "逃逸必须显式、有审计、可撤销"

**借鉴点**:
- Elevated 是 opt-in，不是默认
- Sender allowlist（只有特定用户可用）
- Optional approval（可选的人工审批）
- 只影响 `exec`，不影响其他工具
- 无法覆盖 Tool Policy deny

### 6. Security Audit 工具

**核心理念**: "配置错误比代码漏洞更常见"

**借鉴点**:
- `--fix` 自动修复（chmod 600/700）
- `--deep` 实时探测（probeGateway）
- 清晰的 Severity 分类（critical/warn/info）
- 具体的 Remediation 建议（如何修复）

### 7. 形式化验证

**核心理念**: "关键安全属性应该被证明，而非测试"

**借鉴点**:
- TLA+ 模型验证核心逻辑（Pairing store, Routing isolation）
- Negative models（预期失败的测试，验证模型有效）
- 有界模型检查（bounded model checking）
- 持续集成（CI 自动运行模型检查）

---

## 📊 安全威胁模型总结

### 威胁分类

| 威胁类型 | 攻击面 | 防御机制 | 剩余风险 |
|---------|-------|---------|---------|
| **Prompt Injection** | 用户消息、Web fetch 结果、文件内容 | Tool Policy + Sandbox + Model hardening | ⚠️ 中等（无法完全防御） |
| **未授权 DM 访问** | 陌生人发送 DM | Pairing code (8 位 + 1h TTL + cap 3) | ✅ 低 |
| **Group Chat 滥用** | 公开群组中的恶意成员 | Mention gating + Group allowlist | ✅ 低 |
| **工具滥用** | Agent 执行高风险操作 | Tool Policy (deny > allow) + Sandbox | ✅ 低 |
| **沙箱逃逸** | 容器漏洞、Bind mounts | Docker 安全配置 + no-new-privileges | ⚠️ 中等（依赖 Docker） |
| **Elevated Exec 滥用** | 受信用户滥用 elevated | Sender allowlist + Approval | ⚠️ 中等（需人工审批） |
| **Gateway 远程访问** | LAN/公网暴露 | Auth (token/password) + Tailscale | ✅ 低 |
| **文件权限泄露** | State dir 可读 | Security audit (`chmod 600/700`) | ⚠️ 中等（需用户执行） |
| **Session Transcript 泄露** | 日志包含敏感信息 | Redaction + File permissions | ⚠️ 中等（需配置） |

### 配置安全等级

**Level 0 - 默认配置（中等安全）**:
```json5
{
  channels: { whatsapp: { dmPolicy: "pairing" } },
  agents: { defaults: { sandbox: { mode: "off" } } },
  gateway: { bind: "loopback", auth: { mode: "token" } }
}
```
- ✅ 防止陌生人 DM
- ❌ Main session 不沙箱，高风险操作直接在 host 执行

**Level 1 - 推荐配置（高安全）**:
```json5
{
  channels: {
    whatsapp: {
      dmPolicy: "pairing",
      groups: { "*": { requireMention: true } }
    }
  },
  agents: {
    defaults: {
      sandbox: { mode: "non-main", scope: "agent", workspaceAccess: "none" }
    }
  },
  tools: {
    sandbox: {
      tools: {
        allow: ["group:fs", "group:sessions", "group:memory"],
        deny: ["exec", "bash", "process"]
      }
    }
  },
  gateway: { bind: "loopback", auth: { mode: "token" } }
}
```
- ✅ 防止陌生人 DM
- ✅ Group 需 mention
- ✅ 非 main session 沙箱隔离
- ✅ 沙箱禁止 exec/bash

**Level 2 - 最高安全（paranoid）**:
```json5
{
  channels: {
    whatsapp: {
      dmPolicy: "allowlist",
      groups: { "*": { requireMention: true } }
    }
  },
  agents: {
    defaults: {
      sandbox: { mode: "all", scope: "session", workspaceAccess: "ro" }
    }
  },
  tools: {
    allow: ["read", "memory_search", "sessions_list"],
    deny: ["exec", "bash", "write", "edit", "browser", "web_fetch"]
  },
  gateway: { bind: "loopback", auth: { mode: "password" } }
}
```
- ✅ 无 pairing，只允许已知用户
- ✅ 所有 session 沙箱隔离
- ✅ 只读 workspace
- ✅ 只允许最低风险工具

---

## 📦 依赖安全更新 (2026.3.3)

**`package.json` 与 pnpm overrides**:

| 依赖 | 版本 | 说明 |
|------|------|------|
| `tar` | 7.5.10 | 从 7.5.9 升级，修复 GHSA-qffp-2rhf-9h96 |
| `hono` | 4.12.5 | 通过 pnpm override 固定版本 |

**源码位置**: `package.json:334` (dependencies.tar)、`package.json:364-368` (pnpm.overrides)

---

## 🔗 关键文件索引

### 核心安全实现

| 文件 | 行数 | 核心功能 | 关键位置 |
|------|------|----------|----------|
| `src/pairing/pairing-store.ts` | 497 | Pairing 机制实现 | 186-194 (code gen), 349-444 (upsert), 446-496 (approve) |
| `src/agents/sandbox/types.ts` | 86 | 沙箱类型定义 | 5-60 (SandboxConfig) |
| `src/agents/sandbox/config.ts` | 173 | 沙箱配置解析 | 126-172 (resolve for agent) |
| `src/agents/sandbox/docker.ts` | 352 | Docker 容器管理 | 125-231 (create args) |
| `src/agents/sandbox/tool-policy.ts` | 143 | Tool Policy 解析 | 58-69 (isToolAllowed), 71-142 (resolve) |
| `src/security/audit.ts` | 986+ | 安全审计工具 | 126-192 (filesystem), 85-99 (count severity) |
| `src/gateway/exec-approval-manager.ts` | 83 | Exec 审批管理 | 32-82 (class impl) |
| `src/gateway/device-auth.ts` | 31 | 设备认证 | 13-31 (buildDeviceAuthPayload) |
| `src/gateway/http-common.ts` | 108 | HTTP 安全头 | 11-21 (setDefaultSecurityHeaders, Permissions-Policy) |

### 文档

| 文档 | 核心内容 |
|------|----------|
| `docs/gateway/security/index.md` | 完整安全指南（817 行） |
| `docs/gateway/sandboxing.md` | 沙箱配置详解（194 行） |
| `docs/gateway/sandbox-vs-tool-policy-vs-elevated.md` | 三层权限对比（129 行） |
| `docs/gateway/security/formal-verification.md` | 形式化验证说明（165 行） |

---

## ⭐ 可插拔沙箱后端（2026.3.22+ 新增）

原有的沙箱系统只支持 Docker 容器，2026.3.22 版本引入了**插件化沙箱后端**架构：

### 支持的沙箱后端

| 后端 | 插件/来源 | 说明 | 适用场景 |
|------|----------|------|----------|
| Docker | 内置（默认） | 本地 Docker 容器隔离 | 本地部署，完整隔离 |
| OpenShell | `extensions/openshell/` | 云端受控 Shell 服务 | 云端部署、无 Docker 环境 |
| SSH | 内置/配置 | 远程服务器 SSH 执行 | 远程机器、专用服务器 |

### OpenShell 插件

**位置**：`extensions/openshell/`

```json
{
  "sandbox": {
    "backend": "openshell",
    "openshell": {
      "endpoint": "https://your-openshell-instance.com",
      "apiKey": "${OPENSHELL_API_KEY}"
    }
  }
}
```

**特点**：
- 云端托管的 shell 执行环境，不依赖本地 Docker
- 适合无法安装 Docker 的部署环境（如某些 VPS/容器托管平台）
- 通过 `extensions/openshell/` 插件实现，与 Docker 后端共享相同的 tool policy 和权限模型

### SSH 后端

```json
{
  "sandbox": {
    "backend": "ssh",
    "ssh": {
      "host": "sandbox.internal",
      "user": "runner",
      "keyPath": "~/.ssh/sandbox_key"
    }
  }
}
```

**适用场景**：远程专用沙箱服务器，通过 SSH 隔离执行环境。

### 后端兼容性

所有沙箱后端共享相同的：
- `SandboxConfig` 接口（`mode`、`scope`、`workspaceAccess`）
- Tool Policy 过滤（工具允许/拒绝列表）
- Exec Approval 审批流程

只有底层执行机制不同，对 Agent 层完全透明。

## 🚀 后续研究方向

1. **实际沙箱逃逸测试**: 尝试从 sandboxed session 中逃逸到 host
2. **Elevated Exec 滥用场景**: 研究 elevated 的实际使用模式和风险
3. **Prompt Injection 防御**: 研究如何在 system prompt 中加强防御
4. **Memory 跨 Agent 共享**: 研究是否需要 shared memory 场景
5. **Browser 沙箱**: 深入研究 sandboxed browser 的实现和安全性
6. **OpenShell 安全模型**: 云端 shell 服务的威胁面分析

---

**研究完成时间**: 2026-03-05（2026-03-24 更新可插拔沙箱后端部分）
**代码引用**: 基于 OpenClaw 2026.3.3 → 2026.3.22+
