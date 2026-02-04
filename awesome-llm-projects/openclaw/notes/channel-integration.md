# OpenClaw Channel 集成层深度研究

## 版本信息

- **Commit**: `b9910ab03713d2598a5d4fcbf95c9dc935064b68`
- **研究日期**: 2026-02-04
- **文件范围**: `src/channels/`、`src/routing/`、`src/media/`、`src/telegram/`、`src/discord/`、`src/slack/` 等 Channel 相关模块
- **注意**: 该分析基于 2026 年 2 月的代码，可能随版本更新而变化

---

## 🎯 核心发现：统一抽象 + 动态扩展的 Channel 架构

OpenClaw 的 Channel 集成层是整个系统的"多平台消息枢纽"，通过**三层抽象**（Registry → Dock → Plugin）实现了对 20+ 个消息平台的统一管理，同时保持了高度的可扩展性。

**核心设计理念**：
- **统一抽象**：所有 Channel 遵循相同的接口协议（`ChannelPlugin`）
- **轻量 Dock**：共享代码路径使用轻量的 `ChannelDock` 避免加载重型实现
- **动态路由**：消息路由基于 channel、accountId、peer、guild、team 等多维度信息
- **安全隔离**：DM pairing、allowlist 匹配、沙箱权限等多层安全策略
- **媒体统一**：统一的媒体下载、存储、清理 pipeline

---

## 1. 三层抽象架构

### 1.1 Registry 层：Channel 元数据注册表

**位置**: `src/channels/registry.ts`

定义了 7 个**核心 Channel** 的元数据和顺序：

```typescript:39:101:src/channels/registry.ts
export const CHAT_CHANNEL_ORDER = [
  "telegram",
  "whatsapp",
  "discord",
  "googlechat",
  "slack",
  "signal",
  "imessage",
] as const;

const CHAT_CHANNEL_META: Record<ChatChannelId, ChannelMeta> = {
  telegram: {
    id: "telegram",
    label: "Telegram",
    selectionLabel: "Telegram (Bot API)",
    detailLabel: "Telegram Bot",
    docsPath: "/channels/telegram",
    docsLabel: "telegram",
    blurb: "simplest way to get started — register a bot with @BotFather and get going.",
    systemImage: "paperplane",
    selectionDocsPrefix: "",
    selectionDocsOmitLabel: true,
    selectionExtras: [WEBSITE_URL],
  },
  // ... 其他 channels
};
```

**核心功能**：
- 定义 Channel 的显示顺序（Telegram 优先，最简单）
- 提供 Channel 的元数据（label、docsPath、blurb 等）
- 支持别名（`imsg` → `imessage`、`gchat` → `googlechat`）
- 提供统一的 ID 规范化函数（`normalizeChannelId`）

---

### 1.2 Dock 层：Channel 停靠点（轻量抽象）

**位置**: `src/channels/dock.ts`

`ChannelDock` 是**轻量级的 Channel 行为描述**，供共享代码路径使用（避免加载重型的 Channel 实现）。

**Dock 结构**：

```typescript:40:64:src/channels/dock.ts
export type ChannelDock = {
  id: ChannelId;
  capabilities: ChannelCapabilities;
  commands?: ChannelCommandAdapter;
  outbound?: {
    textChunkLimit?: number;
  };
  streaming?: ChannelDockStreaming;
  elevated?: ChannelElevatedAdapter;
  config?: {
    resolveAllowFrom?: (params: {
      cfg: OpenClawConfig;
      accountId?: string | null;
    }) => Array<string | number> | undefined;
    formatAllowFrom?: (params: {
      cfg: OpenClawConfig;
      accountId?: string | null;
      allowFrom: Array<string | number>;
    }) => string[];
  };
  groups?: ChannelGroupAdapter;
  mentions?: ChannelMentionAdapter;
  threading?: ChannelThreadingAdapter;
  agentPrompt?: ChannelAgentPromptAdapter;
};
```

**示例：Telegram Dock 配置**：

```typescript:92:127:src/channels/dock.ts
telegram: {
  id: "telegram",
  capabilities: {
    chatTypes: ["direct", "group", "channel", "thread"],
    nativeCommands: true,
    blockStreaming: true,
  },
  outbound: { textChunkLimit: 4000 },
  config: {
    resolveAllowFrom: ({ cfg, accountId }) =>
      (resolveTelegramAccount({ cfg, accountId }).config.allowFrom ?? []).map((entry) =>
        String(entry),
      ),
    formatAllowFrom: ({ allowFrom }) =>
      allowFrom
        .map((entry) => String(entry).trim())
        .filter(Boolean)
        .map((entry) => entry.replace(/^(telegram|tg):/i, ""))
        .map((entry) => entry.toLowerCase()),
  },
  groups: {
    resolveRequireMention: resolveTelegramGroupRequireMention,
    resolveToolPolicy: resolveTelegramGroupToolPolicy,
  },
  threading: {
    resolveReplyToMode: ({ cfg }) => cfg.channels?.telegram?.replyToMode ?? "first",
    buildToolContext: ({ context, hasRepliedRef }) => {
      const threadId = context.MessageThreadId ?? context.ReplyToId;
      return {
        currentChannelId: context.To?.trim() || undefined,
        currentThreadTs: threadId != null ? String(threadId) : undefined,
        hasRepliedRef,
      };
    },
  },
},
```

**关键差异**：
- **Dock** 是轻量的、只包含必要的配置和适配器函数
- **Plugin** 是完整的、包含重型实现（monitor、web login 等）
- 共享代码路径应优先使用 `getChannelDock()` 而非 `getChannelPlugin()`

**各 Channel 的 Text Chunk Limit**：
- Telegram: 4000 字符
- WhatsApp: 4000 字符
- Discord: 2000 字符
- Google Chat: 4000 字符
- Slack: 4000 字符
- Signal: 4000 字符
- iMessage: 4000 字符

---

### 1.3 Plugin 层：完整的 Channel 实现

**位置**: `src/channels/plugins/types.plugin.ts`

`ChannelPlugin` 是完整的 Channel 实现接口，包含 **20+ 个适配器**：

```typescript:48:85:src/channels/plugins/types.plugin.ts
export type ChannelPlugin<ResolvedAccount = any> = {
  id: ChannelId;
  meta: ChannelMeta;
  capabilities: ChannelCapabilities;
  defaults?: {
    queue?: {
      debounceMs?: number;
    };
  };
  reload?: { configPrefixes: string[]; noopPrefixes?: string[] };
  // CLI onboarding wizard hooks for this channel.
  onboarding?: ChannelOnboardingAdapter;
  config: ChannelConfigAdapter<ResolvedAccount>;
  configSchema?: ChannelConfigSchema;
  setup?: ChannelSetupAdapter;
  pairing?: ChannelPairingAdapter;
  security?: ChannelSecurityAdapter<ResolvedAccount>;
  groups?: ChannelGroupAdapter;
  mentions?: ChannelMentionAdapter;
  outbound?: ChannelOutboundAdapter;
  status?: ChannelStatusAdapter<ResolvedAccount>;
  gatewayMethods?: string[];
  gateway?: ChannelGatewayAdapter<ResolvedAccount>;
  auth?: ChannelAuthAdapter;
  elevated?: ChannelElevatedAdapter;
  commands?: ChannelCommandAdapter;
  streaming?: ChannelStreamingAdapter;
  threading?: ChannelThreadingAdapter;
  messaging?: ChannelMessagingAdapter;
  agentPrompt?: ChannelAgentPromptAdapter;
  directory?: ChannelDirectoryAdapter;
  resolver?: ChannelResolverAdapter;
  actions?: ChannelMessageActionAdapter;
  heartbeat?: ChannelHeartbeatAdapter;
  // Channel-owned agent tools (login flows, etc.).
  agentTools?: ChannelAgentToolFactory | ChannelAgentTool[];
};
```

**关键适配器说明**：

| 适配器 | 作用 | 示例 |
|--------|------|------|
| `onboarding` | CLI 引导式配置 | 询问 bot token、配置 allowFrom |
| `config` | 配置解析和验证 | 解析 `openclaw.json` 中的 channel 配置 |
| `pairing` | DM 配对机制 | 生成 pairing code、验证配对 |
| `security` | 安全策略 | allowlist 匹配、DM 策略检查 |
| `groups` | 群组行为 | requireMention、toolPolicy |
| `mentions` | 提及解析 | 提取 @mention、strip patterns |
| `outbound` | 消息发送 | 发送文本、图片、文件 |
| `status` | 状态检查 | 检查连接状态、账号信息 |
| `gateway` | Gateway 集成 | 启动 monitor、处理 WebSocket 连接 |
| `streaming` | 流式输出 | 支持 block streaming 的配置 |
| `threading` | 线程支持 | 解析 replyTo、构建 thread context |
| `messaging` | 消息处理 | 接收消息、解析上下文 |
| `directory` | 联系人/群组目录 | 列出可用的 peers 和 groups |
| `resolver` | 目标解析 | 将用户输入解析为 peer ID |
| `actions` | 消息操作 | Discord actions、Slack commands |
| `heartbeat` | Heartbeat 适配器 | 支持主动消息推送 |
| `agentTools` | Channel 专属工具 | WhatsApp 登录工具 |

**Plugin 注册机制**：

`src/channels/plugins/index.ts` 提供了统一的 Plugin 注册和查询接口：

```typescript:31:57:src/channels/plugins/index.ts
export function listChannelPlugins(): ChannelPlugin[] {
  const combined = dedupeChannels(listPluginChannels());
  return combined.toSorted((a, b) => {
    const indexA = CHAT_CHANNEL_ORDER.indexOf(a.id as ChatChannelId);
    const indexB = CHAT_CHANNEL_ORDER.indexOf(b.id as ChatChannelId);
    const orderA = a.meta.order ?? (indexA === -1 ? 999 : indexA);
    const orderB = b.meta.order ?? (indexB === -1 ? 999 : indexB);
    if (orderA !== orderB) {
      return orderA - orderB;
    }
    return a.id.localeCompare(b.id);
  });
}

export function getChannelPlugin(id: ChannelId): ChannelPlugin | undefined {
  const resolvedId = String(id).trim();
  if (!resolvedId) {
    return undefined;
  }
  return listChannelPlugins().find((plugin) => plugin.id === resolvedId);
}

export function normalizeChannelId(raw?: string | null): ChannelId | null {
  // Channel docking: keep input normalization centralized in src/channels/registry.ts.
  // Plugin registry must be initialized before calling.
  return normalizeAnyChannelId(raw);
}
```

**插件去重和排序**：
1. 从 Plugin Registry 中获取所有 Channel plugins
2. 去重（相同 ID 只保留一个）
3. 按 `meta.order` 或 `CHAT_CHANNEL_ORDER` 索引排序
4. 同优先级按 ID 字母顺序排序

---

## 2. 消息路由机制

### 2.1 Route 解析流程

**位置**: `src/routing/resolve-route.ts`

消息路由的核心是 `resolveAgentRoute()` 函数，它根据多维度信息将消息路由到对应的 Agent 和 Session。

**路由输入**：

```typescript:20:29:src/routing/resolve-route.ts
export type ResolveAgentRouteInput = {
  cfg: OpenClawConfig;
  channel: string;
  accountId?: string | null;
  peer?: RoutePeer | null;
  /** Parent peer for threads — used for binding inheritance when peer doesn't match directly. */
  parentPeer?: RoutePeer | null;
  guildId?: string | null;
  teamId?: string | null;
};
```

**路由输出**：

```typescript:31:48:src/routing/resolve-route.ts
export type ResolvedAgentRoute = {
  agentId: string;
  channel: string;
  accountId: string;
  /** Internal session key used for persistence + concurrency. */
  sessionKey: string;
  /** Convenience alias for direct-chat collapse. */
  mainSessionKey: string;
  /** Match description for debugging/logging. */
  matchedBy:
    | "binding.peer"
    | "binding.peer.parent"
    | "binding.guild"
    | "binding.team"
    | "binding.account"
    | "binding.channel"
    | "default";
};
```

**路由匹配优先级**（从高到低）：

```typescript:211:259:src/routing/resolve-route.ts
// 1. Peer binding (最高优先级)
if (peer) {
  const peerMatch = bindings.find((b) => matchesPeer(b.match, peer));
  if (peerMatch) {
    return choose(peerMatch.agentId, "binding.peer");
  }
}

// 2. Parent peer binding (Thread 继承)
const parentPeer = input.parentPeer
  ? { kind: input.parentPeer.kind, id: normalizeId(input.parentPeer.id) }
  : null;
if (parentPeer && parentPeer.id) {
  const parentPeerMatch = bindings.find((b) => matchesPeer(b.match, parentPeer));
  if (parentPeerMatch) {
    return choose(parentPeerMatch.agentId, "binding.peer.parent");
  }
}

// 3. Guild binding (Discord servers)
if (guildId) {
  const guildMatch = bindings.find((b) => matchesGuild(b.match, guildId));
  if (guildMatch) {
    return choose(guildMatch.agentId, "binding.guild");
  }
}

// 4. Team binding (Slack workspaces)
if (teamId) {
  const teamMatch = bindings.find((b) => matchesTeam(b.match, teamId));
  if (teamMatch) {
    return choose(teamMatch.agentId, "binding.team");
  }
}

// 5. Account binding (特定 accountId)
const accountMatch = bindings.find(
  (b) =>
    b.match?.accountId?.trim() !== "*" && !b.match?.peer && !b.match?.guildId && !b.match?.teamId,
);
if (accountMatch) {
  return choose(accountMatch.agentId, "binding.account");
}

// 6. Channel binding (任意 accountId: *)
const anyAccountMatch = bindings.find(
  (b) =>
    b.match?.accountId?.trim() === "*" && !b.match?.peer && !b.match?.guildId && !b.match?.teamId,
);
if (anyAccountMatch) {
  return choose(anyAccountMatch.agentId, "binding.channel");
}

// 7. Default (最低优先级)
return choose(resolveDefaultAgentId(input.cfg), "default");
```

**核心设计**：
- **Peer-based routing** 是最精细的控制（可以为每个 DM/群组分配不同的 Agent）
- **Thread 继承** 允许 Thread 从 Parent channel 继承 Agent binding
- **Guild/Team binding** 支持 Discord server 和 Slack workspace 级别的路由
- **Wildcard `*`** 可以匹配所有 accountId

---

### 2.2 Session Key 构建算法

**位置**: `src/routing/session-key.ts`

Session Key 是 OpenClaw 中用于标识和隔离会话的核心机制。

**Session Key 格式**：

```
agent:<agentId>:<mainKey>                              # main session
agent:<agentId>:dm:<peerId>                            # per-peer DM
agent:<agentId>:<channel>:dm:<peerId>                  # per-channel-peer DM
agent:<agentId>:<channel>:<accountId>:dm:<peerId>      # per-account-channel-peer DM
agent:<agentId>:<channel>:<peerKind>:<peerId>          # group/channel session
agent:<agentId>:<channel>:<peerKind>:<peerId>:thread:<threadId>  # thread session
```

**DM Scope 配置**：

```typescript:126:173:src/routing/session-key.ts
export function buildAgentPeerSessionKey(params: {
  agentId: string;
  mainKey?: string | undefined;
  channel: string;
  accountId?: string | null;
  peerKind?: "dm" | "group" | "channel" | null;
  peerId?: string | null;
  identityLinks?: Record<string, string[]>;
  /** DM session scope. */
  dmScope?: "main" | "per-peer" | "per-channel-peer" | "per-account-channel-peer";
}): string {
  const peerKind = params.peerKind ?? "dm";
  if (peerKind === "dm") {
    const dmScope = params.dmScope ?? "main";
    let peerId = (params.peerId ?? "").trim();
    const linkedPeerId =
      dmScope === "main"
        ? null
        : resolveLinkedPeerId({
            identityLinks: params.identityLinks,
            channel: params.channel,
            peerId,
          });
    if (linkedPeerId) {
      peerId = linkedPeerId;
    }
    peerId = peerId.toLowerCase();
    if (dmScope === "per-account-channel-peer" && peerId) {
      const channel = (params.channel ?? "").trim().toLowerCase() || "unknown";
      const accountId = normalizeAccountId(params.accountId);
      return `agent:${normalizeAgentId(params.agentId)}:${channel}:${accountId}:dm:${peerId}`;
    }
    if (dmScope === "per-channel-peer" && peerId) {
      const channel = (params.channel ?? "").trim().toLowerCase() || "unknown";
      return `agent:${normalizeAgentId(params.agentId)}:${channel}:dm:${peerId}`;
    }
    if (dmScope === "per-peer" && peerId) {
      return `agent:${normalizeAgentId(params.agentId)}:dm:${peerId}`;
    }
    return buildAgentMainSessionKey({
      agentId: params.agentId,
      mainKey: params.mainKey,
    });
  }
  const channel = (params.channel ?? "").trim().toLowerCase() || "unknown";
  const peerId = ((params.peerId ?? "").trim() || "unknown").toLowerCase();
  return `agent:${normalizeAgentId(params.agentId)}:${channel}:${peerKind}:${peerId}`;
}
```

**DM Scope 选项**：

| Scope | 行为 | 示例 Session Key |
|-------|------|------------------|
| `main` (默认) | 所有 DM 共享同一个 main session | `agent:main:main` |
| `per-peer` | 每个联系人独立 session（跨 channel 共享） | `agent:main:dm:user123` |
| `per-channel-peer` | 每个 channel 的联系人独立 session | `agent:main:telegram:dm:user123` |
| `per-account-channel-peer` | 每个 account + channel 的联系人独立 session | `agent:main:telegram:bot1:dm:user123` |

**Identity Links 机制**：

允许将不同 channel 的用户身份关联到同一个"canonical name"，从而实现跨 channel 的会话合并。

```json
{
  "session": {
    "dmScope": "per-peer",
    "identityLinks": {
      "alice": ["telegram:123456789", "discord:987654321"]
    }
  }
}
```

在这个配置下，Telegram 和 Discord 的 Alice 会共享同一个 session：`agent:main:dm:alice`。

---

### 2.3 Thread Session 机制

**位置**: `src/routing/session-key.ts`

OpenClaw 支持 Thread-based sessions（Discord threads、Telegram topics、Slack threads 等）：

```typescript:233:250:src/routing/session-key.ts
export function resolveThreadSessionKeys(params: {
  baseSessionKey: string;
  threadId?: string | null;
  parentSessionKey?: string;
  useSuffix?: boolean;
}): { sessionKey: string; parentSessionKey?: string } {
  const threadId = (params.threadId ?? "").trim();
  if (!threadId) {
    return { sessionKey: params.baseSessionKey, parentSessionKey: undefined };
  }
  const normalizedThreadId = threadId.toLowerCase();
  const useSuffix = params.useSuffix ?? true;
  const sessionKey = useSuffix
    ? `${params.baseSessionKey}:thread:${normalizedThreadId}`
    : params.baseSessionKey;
  return { sessionKey, parentSessionKey: params.parentSessionKey };
}
```

**Thread Session Key 格式**：

```
agent:main:discord:channel:123456:thread:789012
```

**useSuffix 选项**：
- `true` (默认)：Thread 使用独立的 session (`:thread:<id>` 后缀)
- `false`：Thread 与 parent channel 共享 session

---

## 3. 媒体处理 Pipeline

### 3.1 Media Store 架构

**位置**: `src/media/store.ts`

OpenClaw 提供了统一的媒体文件存储和管理机制。

**核心功能**：

```typescript:170:209:src/media/store.ts
export async function saveMediaSource(
  source: string,
  headers?: Record<string, string>,
  subdir = "",
): Promise<SavedMedia> {
  const baseDir = resolveMediaDir();
  const dir = subdir ? path.join(baseDir, subdir) : baseDir;
  await fs.mkdir(dir, { recursive: true, mode: 0o700 });
  await cleanOldMedia();
  const baseId = crypto.randomUUID();
  if (looksLikeUrl(source)) {
    const tempDest = path.join(dir, `${baseId}.tmp`);
    const { headerMime, sniffBuffer, size } = await downloadToFile(source, tempDest, headers);
    const mime = await detectMime({
      buffer: sniffBuffer,
      headerMime,
      filePath: source,
    });
    const ext = extensionForMime(mime) ?? path.extname(new URL(source).pathname);
    const id = ext ? `${baseId}${ext}` : baseId;
    const finalDest = path.join(dir, id);
    await fs.rename(tempDest, finalDest);
    return { id, path: finalDest, size, contentType: mime };
  }
  // local path
  const stat = await fs.stat(source);
  if (!stat.isFile()) {
    throw new Error("Media path is not a file");
  }
  if (stat.size > MAX_BYTES) {
    throw new Error("Media exceeds 5MB limit");
  }
  const buffer = await fs.readFile(source);
  const mime = await detectMime({ buffer, filePath: source });
  const ext = extensionForMime(mime) ?? path.extname(source);
  const id = ext ? `${baseId}${ext}` : baseId;
  const dest = path.join(dir, id);
  await fs.writeFile(dest, buffer, { mode: 0o600 });
  return { id, path: dest, size: stat.size, contentType: mime };
}
```

**存储目录**：`~/.openclaw/media/`

**文件命名策略**：

1. **标准格式**：`{uuid}.{ext}` (如 `a1b2c3d4-e5f6-7890-1234-567890abcdef.jpg`)
2. **嵌入原始文件名格式**：`{sanitized-name}---{uuid}.{ext}` (如 `photo---a1b2c3d4-e5f6-7890-1234-567890abcdef.jpg`)

**文件命名清理规则**：

```typescript:22:30:src/media/store.ts
function sanitizeFilename(name: string): string {
  const trimmed = name.trim();
  if (!trimmed) {
    return "";
  }
  const sanitized = trimmed.replace(/[^\p{L}\p{N}._-]+/gu, "_");
  // Collapse multiple underscores, trim leading/trailing, limit length
  return sanitized.replace(/_+/g, "_").replace(/^_|_$/g, "").slice(0, 60);
}
```

**原始文件名提取**：

```typescript:37:55:src/media/store.ts
export function extractOriginalFilename(filePath: string): string {
  const basename = path.basename(filePath);
  if (!basename) {
    return "file.bin";
  } // Fallback for empty input

  const ext = path.extname(basename);
  const nameWithoutExt = path.basename(basename, ext);

  // Check for ---{uuid} pattern (36 chars: 8-4-4-4-12 with hyphens)
  const match = nameWithoutExt.match(
    /^(.+)---[a-f0-9]{8}-[a-f0-9]{4}-[a-f0-9]{4}-[a-f0-9]{4}-[a-f0-9]{12}$/i,
  );
  if (match?.[1]) {
    return `${match[1]}${ext}`;
  }

  return basename; // Fallback: use as-is
}
```

**存储限制**：

```typescript:13:15:src/media/store.ts
const MEDIA_MAX_BYTES = 5 * 1024 * 1024; // 5MB default
const MAX_BYTES = MEDIA_MAX_BYTES;
const DEFAULT_TTL_MS = 2 * 60 * 1000; // 2 minutes
```

**自动清理机制**：

```typescript:67:83:src/media/store.ts
export async function cleanOldMedia(ttlMs = DEFAULT_TTL_MS) {
  const mediaDir = await ensureMediaDir();
  const entries = await fs.readdir(mediaDir).catch(() => []);
  const now = Date.now();
  await Promise.all(
    entries.map(async (file) => {
      const full = path.join(mediaDir, file);
      const stat = await fs.stat(full).catch(() => null);
      if (!stat) {
        return;
      }
      if (now - stat.mtimeMs > ttlMs) {
        await fs.rm(full).catch(() => {});
      }
    }),
  );
}
```

**清理策略**：
- 每次保存媒体时触发清理
- 删除超过 TTL (默认 2 分钟) 的文件
- 基于文件的 mtime (modification time)

---

### 3.2 MIME 检测机制

OpenClaw 使用 **Buffer Sniffing + Header MIME** 的组合策略来检测媒体类型。

**下载流程**（`downloadToFile` 函数）：

```typescript:92:161:src/media/store.ts
async function downloadToFile(
  url: string,
  dest: string,
  headers?: Record<string, string>,
  maxRedirects = 5,
): Promise<{ headerMime?: string; sniffBuffer: Buffer; size: number }> {
  return await new Promise((resolve, reject) => {
    // ...
    const req = requestImpl(parsedUrl, { headers, lookup: pinned.lookup }, (res) => {
      // Follow redirects
      if (res.statusCode && res.statusCode >= 300 && res.statusCode < 400) {
        const location = res.headers.location;
        if (!location || maxRedirects <= 0) {
          reject(new Error(`Redirect loop or missing Location header`));
          return;
        }
        const redirectUrl = new URL(location, url).href;
        resolve(downloadToFile(redirectUrl, dest, headers, maxRedirects - 1));
        return;
      }
      if (!res.statusCode || res.statusCode >= 400) {
        reject(new Error(`HTTP ${res.statusCode ?? "?"} downloading media`));
        return;
      }
      let total = 0;
      const sniffChunks: Buffer[] = [];
      let sniffLen = 0;
      const out = createWriteStream(dest, { mode: 0o600 });
      res.on("data", (chunk) => {
        total += chunk.length;
        if (sniffLen < 16384) {
          sniffChunks.push(chunk);
          sniffLen += chunk.length;
        }
        if (total > MAX_BYTES) {
          req.destroy(new Error("Media exceeds 5MB limit"));
        }
      });
      pipeline(res, out)
        .then(() => {
          const sniffBuffer = Buffer.concat(sniffChunks, Math.min(sniffLen, 16384));
          const rawHeader = res.headers["content-type"];
          const headerMime = Array.isArray(rawHeader) ? rawHeader[0] : rawHeader;
          resolve({
            headerMime,
            sniffBuffer,
            size: total,
          });
        })
        .catch(reject);
    });
    // ...
  });
}
```

**关键细节**：
- **Sniff Buffer 大小**：16KB (前 16384 字节)
- **重定向支持**：最多 5 次重定向
- **流式下载**：边下载边保存到磁盘，避免内存占用过大
- **尺寸限制检查**：下载过程中实时检查，超过 5MB 立即中止

**MIME 检测**：

调用 `detectMime()` 函数（在 `src/media/mime.ts` 中实现）结合 buffer sniffing 和 header MIME 来确定最终的 MIME 类型。

---

### 3.3 媒体 Buffer 保存

**位置**: `src/media/store.ts:211-242`

用于保存已在内存中的媒体 buffer（如 Channel 收到的图片/文件）：

```typescript:211:242:src/media/store.ts
export async function saveMediaBuffer(
  buffer: Buffer,
  contentType?: string,
  subdir = "inbound",
  maxBytes = MAX_BYTES,
  originalFilename?: string,
): Promise<SavedMedia> {
  if (buffer.byteLength > maxBytes) {
    throw new Error(`Media exceeds ${(maxBytes / (1024 * 1024)).toFixed(0)}MB limit`);
  }
  const dir = path.join(resolveMediaDir(), subdir);
  await fs.mkdir(dir, { recursive: true, mode: 0o700 });
  const uuid = crypto.randomUUID();
  const headerExt = extensionForMime(contentType?.split(";")[0]?.trim() ?? undefined);
  const mime = await detectMime({ buffer, headerMime: contentType });
  const ext = headerExt ?? extensionForMime(mime) ?? "";

  let id: string;
  if (originalFilename) {
    // Embed original name: {sanitized}---{uuid}.ext
    const base = path.parse(originalFilename).name;
    const sanitized = sanitizeFilename(base);
    id = sanitized ? `${sanitized}---${uuid}${ext}` : `${uuid}${ext}`;
  } else {
    // Legacy: just UUID
    id = ext ? `${uuid}${ext}` : uuid;
  }

  const dest = path.join(dir, id);
  await fs.writeFile(dest, buffer, { mode: 0o600 });
  return { id, path: dest, size: buffer.byteLength, contentType: mime };
}
```

**默认 subdir**：`inbound` (入站媒体)

**文件权限**：`0o600` (仅所有者可读写)

---

## 4. DM 安全策略

### 4.1 Pairing 机制

**位置**: `src/channels/plugins/pairing.ts`

Pairing 是 OpenClaw 的 DM 安全核心机制：用户需要通过 **pairing code** 来授权 AI 与自己建立 DM 连接。

**Pairing 适配器接口**：

```typescript
export type ChannelPairingAdapter = {
  generateCode?: (params: { cfg: OpenClawConfig }) => Promise<string> | string;
  notifyApproval?: (params: {
    cfg: OpenClawConfig;
    id: string;
    runtime?: RuntimeEnv;
  }) => Promise<void> | void;
};
```

**Pairing 流程**：

1. **生成 Pairing Code**：`cli pairing generate --channel telegram`
2. **用户发送 Code**：用户在 Telegram 中向 bot 发送 pairing code
3. **验证并授权**：Gateway 验证 code，将用户 ID 添加到 `allowFrom` 配置
4. **通知用户**：Channel 的 `notifyApproval` 回调发送确认消息

**支持 Pairing 的 Channels**：

```typescript:11:16:src/channels/plugins/pairing.ts
export function listPairingChannels(): ChannelId[] {
  // Channel docking: pairing support is declared via plugin.pairing.
  return listChannelPlugins()
    .filter((plugin) => plugin.pairing)
    .map((plugin) => plugin.id);
}
```

---

### 4.2 Allowlist 匹配机制

**位置**: `src/channels/allowlist-match.ts`

Allowlist 是 OpenClaw 的 DM 白名单机制，只有在 `allowFrom` 列表中的用户才能与 AI 交互。

**匹配源类型**：

```typescript:1:11:src/channels/allowlist-match.ts
export type AllowlistMatchSource =
  | "wildcard"
  | "id"
  | "name"
  | "tag"
  | "username"
  | "prefixed-id"
  | "prefixed-user"
  | "prefixed-name"
  | "slug"
  | "localpart";
```

**匹配结果**：

```typescript:13:17:src/channels/allowlist-match.ts
export type AllowlistMatch<TSource extends string = AllowlistMatchSource> = {
  allowed: boolean;
  matchKey?: string;
  matchSource?: TSource;
};
```

**匹配逻辑示例**（Telegram）：

`src/channels/dock.ts:100-111` 中定义了 Telegram 的 allowFrom 格式化规则：

```typescript:100:111:src/channels/dock.ts
config: {
  resolveAllowFrom: ({ cfg, accountId }) =>
    (resolveTelegramAccount({ cfg, accountId }).config.allowFrom ?? []).map((entry) =>
      String(entry),
    ),
  formatAllowFrom: ({ allowFrom }) =>
    allowFrom
      .map((entry) => String(entry).trim())
      .filter(Boolean)
      .map((entry) => entry.replace(/^(telegram|tg):/i, ""))
      .map((entry) => entry.toLowerCase()),
},
```

**支持的 Allowlist 格式**：

| Channel | 示例格式 | 说明 |
|---------|---------|------|
| Telegram | `123456789`、`telegram:123456789`、`tg:123456789` | User ID (数字) |
| Discord | `987654321`、`discord:username#1234` | User ID 或 username#discriminator |
| WhatsApp | `+1234567890`、`whatsapp:+1234567890` | E.164 格式电话号码 |
| Slack | `U01234ABCD`、`slack:U01234ABCD` | User ID |
| Signal | `+1234567890`、`signal:+1234567890` | E.164 格式电话号码 |
| iMessage | `+1234567890`、`email@example.com` | 电话号码或 Apple ID |

**Wildcard 支持**：

所有 Channels 都支持 `*` 作为 wildcard，允许所有用户访问（**不推荐用于生产环境**）。

---

### 4.3 Group 安全策略

**Group 策略配置**：

1. **requireMention** (默认 `true`)：群组消息必须 @mention bot 才会响应
2. **toolPolicy**：控制群组中 Agent 可以使用的工具权限（如禁用 `bash`、`exec` 等敏感工具）
3. **groupAllowFrom**：群组白名单（只响应特定群组）

**Group 策略适配器**：

```typescript
export type ChannelGroupAdapter = {
  resolveRequireMention?: (params: {
    cfg: OpenClawConfig;
    accountId?: string | null;
    peerId: string;
  }) => boolean;
  resolveToolPolicy?: (params: {
    cfg: OpenClawConfig;
    accountId?: string | null;
    peerId: string;
  }) => string | undefined;
  resolveGroupIntroHint?: (params: {
    cfg: OpenClawConfig;
    accountId?: string | null;
  }) => string | undefined;
};
```

**示例：Telegram Group 策略**（`src/channels/dock.ts:112-115`）：

```typescript:112:115:src/channels/dock.ts
groups: {
  resolveRequireMention: resolveTelegramGroupRequireMention,
  resolveToolPolicy: resolveTelegramGroupToolPolicy,
},
```

---

## 5. 消息分块机制

### 5.1 Text Chunk Limit

每个 Channel 都有自己的消息长度限制，OpenClaw 通过 **Draft Chunking** 机制自动分块发送长消息。

**Chunk Limit 配置**（`src/channels/dock.ts`）：

```typescript
telegram: { outbound: { textChunkLimit: 4000 } },
whatsapp: { outbound: { textChunkLimit: 4000 } },
discord: { outbound: { textChunkLimit: 2000 } },
googlechat: { outbound: { textChunkLimit: 4000 } },
slack: { outbound: { textChunkLimit: 4000 } },
signal: { outbound: { textChunkLimit: 4000 } },
imessage: { outbound: { textChunkLimit: 4000 } },
```

---

### 5.2 Draft Chunking 算法

**位置**: `src/telegram/draft-chunking.ts`

Draft Chunking 是 OpenClaw 的智能分块算法，支持按**段落**、**换行**、**句子**优先断开。

**分块配置**：

```typescript:9:41:src/telegram/draft-chunking.ts
export function resolveTelegramDraftStreamingChunking(
  cfg: OpenClawConfig | undefined,
  accountId?: string | null,
): {
  minChars: number;
  maxChars: number;
  breakPreference: "paragraph" | "newline" | "sentence";
} {
  const providerChunkLimit = getChannelDock("telegram")?.outbound?.textChunkLimit;
  const textLimit = resolveTextChunkLimit(cfg, "telegram", accountId, {
    fallbackLimit: providerChunkLimit,
  });
  const normalizedAccountId = normalizeAccountId(accountId);
  const draftCfg =
    cfg?.channels?.telegram?.accounts?.[normalizedAccountId]?.draftChunk ??
    cfg?.channels?.telegram?.draftChunk;

  const maxRequested = Math.max(
    1,
    Math.floor(draftCfg?.maxChars ?? DEFAULT_TELEGRAM_DRAFT_STREAM_MAX),
  );
  const maxChars = Math.max(1, Math.min(maxRequested, textLimit));
  const minRequested = Math.max(
    1,
    Math.floor(draftCfg?.minChars ?? DEFAULT_TELEGRAM_DRAFT_STREAM_MIN),
  );
  const minChars = Math.min(minRequested, maxChars);
  const breakPreference =
    draftCfg?.breakPreference === "newline" || draftCfg?.breakPreference === "sentence"
      ? draftCfg.breakPreference
      : "paragraph";
  return { minChars, maxChars, breakPreference };
}
```

**默认配置**：

```typescript:6:7:src/telegram/draft-chunking.ts
const DEFAULT_TELEGRAM_DRAFT_STREAM_MIN = 200;
const DEFAULT_TELEGRAM_DRAFT_STREAM_MAX = 800;
```

**分块策略**：
- **minChars**：最小分块长度（默认 200）
- **maxChars**：最大分块长度（默认 800，不超过 textChunkLimit）
- **breakPreference**：断开优先级（默认 `paragraph`）
  - `paragraph`：优先在双换行处断开
  - `newline`：优先在单换行处断开
  - `sentence`：优先在句子结尾（`.`、`!`、`?`）断开

**分块算法流程**：

1. 累积字符直到达到 `minChars`
2. 继续累积直到找到合适的断点或达到 `maxChars`
3. 根据 `breakPreference` 寻找最优断点
4. 如果找不到断点且超过 `maxChars`，强制断开
5. 将分块后的消息通过 Channel 发送

---

### 5.3 Block Streaming Coalesce

**位置**: `src/channels/dock.ts:66-71`

某些 Channels 支持 **Block Streaming**（流式输出），但需要控制分块合并的策略以优化用户体验。

**Streaming Coalesce 配置**：

```typescript:66:71:src/channels/dock.ts
type ChannelDockStreaming = {
  blockStreamingCoalesceDefaults?: {
    minChars?: number;
    idleMs?: number;
  };
};
```

**示例：Discord Streaming 配置**（`src/channels/dock.ts:189-191`）：

```typescript:189:191:src/channels/dock.ts
streaming: {
  blockStreamingCoalesceDefaults: { minChars: 1500, idleMs: 1000 },
},
```

**参数说明**：
- **minChars**：最小字符数，达到后才发送 block（避免频繁更新）
- **idleMs**：空闲时间，超过后即使未达到 minChars 也发送 block（避免长时间无响应）

**支持 Block Streaming 的 Channels**：
- Telegram: `blockStreaming: true`
- Discord: `blockStreaming: false`（但支持 coalesce 配置）
- Google Chat: `blockStreaming: true`
- Slack: `blockStreaming: false`（但支持 coalesce 配置）

---

## 6. 具体 Channel 实现

### 6.1 Telegram 实现

**核心文件数量**: 80+ 文件

**关键模块**：

| 模块 | 文件 | 作用 |
|------|------|------|
| Bot 创建 | `src/telegram/bot.ts` | 基于 grammY 创建 Telegram bot |
| 消息处理 | `src/telegram/bot-message.ts` | 解析和处理 Telegram 消息 |
| 消息分发 | `src/telegram/bot-message-dispatch.ts` | 将消息分发到 Agent |
| 原生命令 | `src/telegram/bot-native-commands.ts` | 注册 `/help`、`/start` 等命令 |
| 消息发送 | `src/telegram/send.ts` | 发送文本、图片、文件等 |
| Draft Streaming | `src/telegram/draft-stream.ts` | 流式输出实现 |
| Draft Chunking | `src/telegram/draft-chunking.ts` | 分块策略 |
| 账号管理 | `src/telegram/accounts.ts` | 多账号配置解析 |
| Webhook | `src/telegram/webhook.ts` | Webhook 模式支持 |
| Monitor | `src/telegram/monitor.ts` | 长轮询 monitor |
| 格式化 | `src/telegram/format.ts` | Markdown/HTML 格式化 |
| 媒体下载 | `src/telegram/download.ts` | 下载 Telegram 媒体文件 |

**Bot 创建流程**（`src/telegram/bot.ts:112-150`）：

```typescript:112:150:src/telegram/bot.ts
export function createTelegramBot(opts: TelegramBotOptions) {
  const runtime: RuntimeEnv = opts.runtime ?? {
    log: console.log,
    error: console.error,
    exit: (code: number): never => {
      throw new Error(`exit ${code}`);
    },
  };
  const cfg = opts.config ?? loadConfig();
  const account = resolveTelegramAccount({
    cfg,
    accountId: opts.accountId,
  });
  const telegramCfg = account.config;

  const fetchImpl = resolveTelegramFetch(opts.proxyFetch, {
    network: telegramCfg.network,
  });
  const shouldProvideFetch = Boolean(fetchImpl);
  const timeoutSeconds =
    typeof telegramCfg?.timeoutSeconds === "number" && Number.isFinite(telegramCfg.timeoutSeconds)
      ? Math.max(1, Math.floor(telegramCfg.timeoutSeconds))
      : undefined;
  const client: ApiClientOptions | undefined =
    shouldProvideFetch || timeoutSeconds
      ? {
          ...(shouldProvideFetch && fetchImpl
            ? { fetch: fetchImpl as unknown as ApiClientOptions["fetch"] }
            : {}),
          ...(timeoutSeconds ? { timeoutSeconds } : {}),
        }
      : undefined;

  const bot = new Bot(opts.token, client ? { client } : undefined);
  bot.api.config.use(apiThrottler());
  bot.use(sequentialize(getTelegramSequentialKey));
  bot.catch((err) => {
    runtime.error?.(danger(`telegram bot error: ${formatUncaughtError(err)}`));
  });
  // ...
}
```

**关键特性**：
- **API Throttler**：使用 `@grammyjs/transformer-throttler` 限流，避免触发 Telegram API 限制
- **Sequentialize**：使用 `@grammyjs/runner` 的 `sequentialize` 确保同一个 chat 的消息按顺序处理
- **错误捕获**：统一的错误处理机制

**Sequential Key 生成**（`src/telegram/bot.ts:68-110`）：

```typescript:68:110:src/telegram/bot.ts
export function getTelegramSequentialKey(ctx: {
  chat?: { id?: number };
  message?: TelegramMessage;
  update?: {
    message?: TelegramMessage;
    edited_message?: TelegramMessage;
    callback_query?: { message?: TelegramMessage };
    message_reaction?: { chat?: { id?: number } };
  };
}): string {
  // Handle reaction updates
  const reaction = ctx.update?.message_reaction;
  if (reaction?.chat?.id) {
    return `telegram:${reaction.chat.id}`;
  }
  const msg =
    ctx.message ??
    ctx.update?.message ??
    ctx.update?.edited_message ??
    ctx.update?.callback_query?.message;
  const chatId = msg?.chat?.id ?? ctx.chat?.id;
  const rawText = msg?.text ?? msg?.caption;
  const botUsername = (ctx as { me?: { username?: string } }).me?.username;
  if (
    rawText &&
    isControlCommandMessage(rawText, undefined, botUsername ? { botUsername } : undefined)
  ) {
    if (typeof chatId === "number") {
      return `telegram:${chatId}:control`;
    }
    return "telegram:control";
  }
  const isGroup = msg?.chat?.type === "group" || msg?.chat?.type === "supergroup";
  const messageThreadId = msg?.message_thread_id;
  const isForum = (msg?.chat as { is_forum?: boolean } | undefined)?.is_forum;
  const threadId = isGroup
    ? resolveTelegramForumThreadId({ isForum, messageThreadId })
    : messageThreadId;
  if (typeof chatId === "number") {
    return threadId != null ? `telegram:${chatId}:topic:${threadId}` : `telegram:${chatId}`;
  }
  return "telegram:unknown";
}
```

**Sequential Key 格式**：
- `telegram:{chatId}` - 普通 DM/群组
- `telegram:{chatId}:control` - 控制命令（优先处理）
- `telegram:{chatId}:topic:{threadId}` - Forum topics
- `telegram:unknown` - 未知来源

---

### 6.2 Discord 实现

**核心文件数量**: 41 文件

**关键模块**：

| 模块 | 文件 | 作用 |
|------|------|------|
| Monitor | `src/discord/monitor.ts` | Gateway 连接和事件监听 |
| 消息发送 | `src/discord/send.ts` | 发送文本、embed、文件等 |
| 消息分块 | `src/discord/chunk.ts` | 消息分块算法 |
| 账号管理 | `src/discord/accounts.ts` | 多账号配置解析 |
| 用户解析 | `src/discord/resolve-users.ts` | 解析用户 ID/username |
| Channel 解析 | `src/discord/resolve-channels.ts` | 解析 channel ID/名称 |
| Emoji/Sticker | `src/discord/send.emojis-stickers.ts` | 自定义 emoji 和 sticker |
| PluralKit | `src/discord/pluralkit.ts` | PluralKit 集成 |
| Audit | `src/discord/audit.ts` | 审计日志 |

**Discord 的特殊之处**：
- **Gateway 连接**：使用 WebSocket Gateway 而非 REST API 长轮询
- **Server/Channel 层级**：支持 guild (server) → channel → thread 的三层结构
- **PluralKit 支持**：可以识别和处理 PluralKit 代理消息
- **Slash Commands**：支持 Discord 的原生 slash commands

---

### 6.3 WhatsApp 实现

**核心库**: Baileys (WhatsApp Web Multi-Device API)

**关键特性**：
- **QR 配对**：需要扫描 QR code 配对
- **本地状态**：存储大量本地状态（auth state、pre-keys 等）
- **端到端加密**：支持 Signal Protocol 加密
- **群组支持**：支持群组消息和群组管理

**WhatsApp 的复杂性**：
- 状态管理较重（需要存储大量 auth 信息）
- 配对过程需要物理手机扫码
- 连接不稳定需要处理重连逻辑

---

### 6.4 Slack 实现

**核心库**: Bolt SDK

**关键特性**：
- **Socket Mode**：使用 WebSocket 连接而非 Webhook
- **Workspace Apps**：每个 workspace 需要单独配置 app
- **Rich Messaging**：支持 Block Kit 丰富消息格式
- **Thread 支持**：原生支持 thread (回复链)

---

### 6.5 Signal 实现

**核心依赖**: signal-cli REST API

**关键特性**：
- **隐私优先**：端到端加密
- **Linked Device**：需要将 signal-cli 作为 linked device 配对
- **REST API**：通过 HTTP REST API 与 signal-cli 交互

---

### 6.6 iMessage 实现

**核心库**: imsg (macOS native integration)

**关键特性**：
- **macOS Only**：只能在 macOS 上运行
- **Native Integration**：直接与 macOS Messages.app 集成
- **工作进行中**：文档标注为 "work in progress"

---

## 7. Plugin 扩展生态

### 7.1 Extension Channels

**位置**: `extensions/` 目录

OpenClaw 支持通过 **Extension Channels** 动态扩展到更多消息平台。

**已实现的 Extension Channels**：
- Microsoft Teams (`extensions/msteams`)
- Matrix (`extensions/matrix`)
- LINE (`extensions/line`)
- Zalo (`extensions/zalo`)
- Zalo Personal (`extensions/zalouser`)
- Nextcloud Talk (`extensions/nextcloud-talk`)
- Nostr (`extensions/nostr`)
- Tlon (`extensions/tlon`)
- Twitch (`extensions/twitch`)
- Mattermost (`extensions/mattermost`)
- BlueBubbles (`extensions/bluebubbles`)

**Extension 安装方式**：

```bash
openclaw extensions install <extension-name>
```

或在 `openclaw.json` 中配置：

```json
{
  "extensions": {
    "paths": ["./extensions/msteams"]
  }
}
```

---

### 7.2 Plugin 动态加载机制

**位置**: `src/plugins/runtime.ts`（未在本次研究中详细分析，但在 Registry 中引用）

Plugin Registry 负责：
1. 扫描配置的 extension paths
2. 加载 extension 的 `channel.ts` 文件
3. 注册 ChannelPlugin 到 registry
4. 提供统一的查询接口（`listChannelPlugins()`、`getChannelPlugin()`）

---

## 8. 与 AI 主动性的关联

### 8.1 Heartbeat 与 Channel 的关联

**核心问题**：Heartbeat 生成的消息如何路由到正确的 Channel？

**答案**：通过 **ChannelHeartbeatAdapter**（在 `ChannelPlugin` 中定义）。

```typescript
export type ChannelHeartbeatAdapter = {
  sendMessage: (params: {
    cfg: OpenClawConfig;
    accountId?: string | null;
    target: string; // peer ID or channel ID
    text: string;
  }) => Promise<void>;
};
```

**Heartbeat 消息路由流程**：

1. **Heartbeat Runner** 定时触发（`src/infra/heartbeat-runner.ts`）
2. **Agent 执行 Heartbeat run**，生成回复消息
3. **消息路由**：根据 HEARTBEAT.md 中配置的 target 解析到对应的 Channel 和 peer
4. **Channel Adapter 发送**：调用 `ChannelHeartbeatAdapter.sendMessage()` 发送消息

**HEARTBEAT_OK 机制**：

如果 Agent 判断"一切正常"，可以返回 `HEARTBEAT_OK` 标记，Gateway 会**静默处理**（不发送消息），避免打扰用户。

**Channel Heartbeat Adapter 位置**：

根据代码结构推测，Heartbeat Adapter 应该在各 Channel 的 Plugin 实现中定义（如 `src/telegram/heartbeat.ts`、`src/discord/heartbeat.ts` 等），但在本次研究中未找到具体的 `features/heartbeat.ts` 文件（可能尚未实现或在其他位置）。

---

### 8.2 Memory 与 Channel 的关联

**核心问题**：Channel 中的媒体消息（图片、文件）如何自动索引到 Memory？

**答案**：通过 **Session Transcript 自动同步**（`src/memory/sync-session-files.ts`）。

**Memory 索引流程**：

1. **消息接收**：Channel 接收到消息（包含媒体）
2. **媒体保存**：调用 `saveMediaBuffer()` 或 `saveMediaSource()` 保存到 `~/.openclaw/media/`
3. **Transcript 记录**：消息内容（包含媒体路径）记录到 Session Transcript (`~/.openclaw/sessions/<sessionKey>/transcript.jsonl`)
4. **File Watcher 触发**：MemoryIndexManager 的 File Watcher 检测到 Transcript 变化
5. **自动索引**：提取媒体路径，读取文件内容，生成 embedding，索引到 SQLite + sqlite-vec

**媒体消息的 Memory 检索**：

Agent 可以通过 `memory_search` tool 搜索相关的媒体消息：

```
memory_search(query="用户发送的猫咪图片")
```

返回结果中会包含媒体文件的路径和上下文信息。

---

### 8.3 Skills 与 Channel 的关联

**核心问题**：Channel-specific tools（Discord actions、Slack commands）是否可以作为 Skills 动态加载？

**答案**：**可以！** 通过 `ChannelPlugin.agentTools`。

**Channel Agent Tools 定义**：

```typescript:82:83:src/channels/plugins/types.plugin.ts
// Channel-owned agent tools (login flows, etc.).
agentTools?: ChannelAgentToolFactory | ChannelAgentTool[];
```

**示例：WhatsApp Login Tool**：

`src/channels/plugins/agent-tools/whatsapp-login.ts` 定义了 WhatsApp 登录相关的 Agent Tools（如 `whatsapp_qr_login`），这些 tools 会在 WhatsApp Channel Plugin 中注册，Agent 可以动态调用。

**Channel Actions 作为 Skills**：

- **Discord Actions**（`src/channels/plugins/actions/discord.ts`）：如创建 channel、管理权限等
- **Slack Commands**（`src/channels/plugins/slack.actions.ts`）：如发送消息到特定 channel、更新 message 等

这些 Actions 都可以作为 Skills 注册到 Agent，Agent 可以根据需要动态调用。

**Skills 刷新机制与 Channel 的关联**：

当 Channel Plugin 的配置变化（如添加新的 agentTools）时，Skills 系统会**自动刷新**，Agent 在下一次 run 时会加载新的 tools。

---

## 9. 架构设计亮点

### 9.1 三层抽象的优雅分层

**Registry → Dock → Plugin** 的三层设计实现了：
- **轻量查询**：共享代码路径使用 Dock（不加载重型实现）
- **完整功能**：执行边界使用 Plugin（加载完整实现）
- **动态扩展**：Extension Channels 可以无缝集成到 Registry

---

### 9.2 消息路由的多维度匹配

**Peer → Guild → Team → Account → Channel → Default** 的优先级设计允许：
- **精细控制**：可以为每个 DM/群组分配不同的 Agent
- **层级继承**：Thread 可以从 Parent channel 继承 Agent binding
- **灵活配置**：支持 wildcard `*` 匹配所有 account

---

### 9.3 DM Scope 的灵活隔离策略

**main → per-peer → per-channel-peer → per-account-channel-peer** 的四级 scope 设计允许：
- **会话合并**：跨 channel 共享同一个联系人的对话历史（per-peer + identityLinks）
- **会话隔离**：每个 channel 的联系人独立 session（per-channel-peer）
- **账号隔离**：多 account 场景下每个 account 的联系人独立 session（per-account-channel-peer）

---

### 9.4 媒体处理的统一 Pipeline

**Media Store** 提供了统一的媒体下载、存储、清理机制，所有 Channels 共享同一套 API，避免了各 Channel 重复实现媒体处理逻辑。

**关键特性**：
- **SSRF 防护**：`resolvePinnedHostname()` 防止 SSRF 攻击
- **MIME 检测**：Buffer sniffing + Header MIME 组合策略
- **自动清理**：TTL 过期自动清理，避免磁盘占用

---

### 9.5 Draft Chunking 的智能分块

**段落 → 换行 → 句子** 的三级断点优先级设计避免了：
- **词语截断**：不会在单词中间断开
- **语义破坏**：优先在段落/句子边界断开
- **阅读体验差**：用户收到的消息分块自然流畅

---

## 10. 可借鉴的设计模式

### 10.1 轻量 Dock + 重型 Plugin 的分层策略

**适用场景**：需要在共享代码路径中查询模块元数据，但不希望加载完整实现（避免性能损耗）。

**实现要点**：
- Dock 只包含配置和适配器函数引用
- Plugin 包含完整实现（monitors、web login 等）
- 共享代码优先使用 `getChannelDock()`，执行边界使用 `getChannelPlugin()`

---

### 10.2 多维度消息路由 + 优先级匹配

**适用场景**：消息需要根据多个维度（channel、account、peer、guild 等）路由到不同的处理器。

**实现要点**：
- 定义清晰的优先级顺序（Peer > Guild > Team > Account > Channel > Default）
- 使用 Binding 配置表达路由规则
- 支持 Wildcard 匹配和继承（Thread 继承 Parent）

---

### 10.3 Session Key 的分级隔离策略

**适用场景**：需要灵活控制会话隔离粒度（全局共享 vs 每用户独立 vs 每 channel 独立）。

**实现要点**：
- 定义多级 Scope（main、per-peer、per-channel-peer、per-account-channel-peer）
- 使用 Identity Links 实现跨 channel 身份关联
- Session Key 格式清晰可读（`agent:<agentId>:<channel>:<peerKind>:<peerId>`）

---

### 10.4 媒体处理的统一 Pipeline

**适用场景**：多个模块需要下载、存储、管理媒体文件。

**实现要点**：
- 统一的 `saveMediaSource()` 和 `saveMediaBuffer()` API
- MIME 检测使用 Buffer sniffing + Header MIME 组合策略
- 自动清理机制基于 TTL 和 mtime

---

### 10.5 Plugin 动态扩展机制

**适用场景**：需要支持第三方模块动态注册和加载。

**实现要点**：
- 定义统一的 Plugin 接口（ChannelPlugin）
- Plugin Registry 负责扫描、加载、去重、排序
- Extension 通过配置文件声明路径，无需修改核心代码

---

## 11. 工程实践总结

### 11.1 代码组织

**优秀之处**：
- 每个 Channel 独立目录（`src/telegram/`、`src/discord/` 等）
- 共享逻辑抽取到 `src/channels/`、`src/routing/`、`src/media/`
- Plugin 相关代码统一在 `src/channels/plugins/`
- Extension Channels 独立在 `extensions/` 目录

---

### 11.2 类型安全

**优秀之处**：
- 大量使用 TypeScript 类型定义（`ChannelPlugin`、`ChannelDock`、`ResolvedAgentRoute` 等）
- 适配器接口清晰（`ChannelPairingAdapter`、`ChannelGroupAdapter` 等）
- 类型推导友好（`ChatChannelId` 使用 `as const` 生成联合类型）

---

### 11.3 安全性

**优秀之处**：
- **DM Pairing**：防止未授权 DM 访问
- **Allowlist 匹配**：白名单机制
- **SSRF 防护**：`resolvePinnedHostname()` 防止恶意 URL
- **文件权限**：媒体文件使用 `0o600` 权限（仅所有者可读写）
- **尺寸限制**：5MB 媒体尺寸限制，避免 DoS

---

### 11.4 测试覆盖

**测试文件**：
- 大量 `*.test.ts` 文件（如 `bot.test.ts`、`send.test.ts`、`chunk.test.ts` 等）
- 测试覆盖关键功能（消息发送、分块、路由、allowlist 匹配等）

---

### 11.5 文档质量

**优秀之处**：
- 每个 Channel 都有独立的文档（`docs/channels/telegram.md` 等）
- 文档结构清晰（概览、配置、故障排查等）
- 示例丰富（配置示例、命令示例等）

---

## 12. 待研究的问题

1. **Heartbeat Adapter 的具体实现**：各 Channel 的 Heartbeat Adapter 代码位置？如何发送定时消息？
2. **Memory 索引的触发时机**：Transcript 文件变化后多久触发 Memory 索引？是否有批量索引优化？
3. **Plugin 热重载机制**：Extension Channels 的配置变化如何触发热重载？是否支持无缝切换？
4. **消息去重机制**：Telegram 的 `createTelegramUpdateDedupe()` 如何工作？去重窗口多大？
5. **Channel 状态同步**：多客户端场景下，Channel 的状态（如 Telegram offset）如何同步？

---

## 13. 总结

OpenClaw 的 Channel 集成层通过**三层抽象**（Registry → Dock → Plugin）、**多维度路由**（Peer → Guild → Team → Account → Channel → Default）、**统一媒体 Pipeline**、**智能分块机制**、**DM 安全策略**等设计，实现了对 20+ 个消息平台的统一管理和动态扩展。

**核心设计理念**：
- **统一抽象**：所有 Channel 遵循相同的 ChannelPlugin 接口
- **轻量查询**：共享代码路径使用轻量 Dock 避免加载重型实现
- **精细路由**：支持多维度消息路由和会话隔离
- **安全优先**：Pairing、Allowlist、SSRF 防护等多层安全策略
- **动态扩展**：Extension Channels 可以无缝集成到系统

**可借鉴的设计模式**：
- 轻量 Dock + 重型 Plugin 的分层策略
- 多维度消息路由 + 优先级匹配
- Session Key 的分级隔离策略
- 媒体处理的统一 Pipeline
- Plugin 动态扩展机制

OpenClaw 的 Channel 集成层是整个系统的"消息枢纽"，为上层的 Agent、Session、Tool 系统提供了坚实的多平台消息基础设施。通过统一的抽象和灵活的扩展机制，OpenClaw 能够轻松支持新的消息平台，同时保持代码的简洁和可维护性。

---

**研究完成日期**: 2026-02-04  
**研究者**: 璇玑 ✨
