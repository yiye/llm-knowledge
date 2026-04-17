# LLM Agent 安全攻击面检查单

> **数据来源**：[OpenClaw Releases 2026.4.5 - 2026.4.15](https://github.com/openclaw/openclaw/releases) 的 security fix 归纳
> **分析时间**：2026-04-17
> **适用范围**：任何具备工具调用、文件访问、外部网络请求能力的 LLM Agent

---

## 使用方式

每新增一个 Agent 功能（尤其是涉及工具、网络、文件、执行的功能），过一遍这 16 条。
任何一条没答清楚，不要上线。

---

## A. 网络侧（3 项）

### ✅ 1. SSRF 防护覆盖所有 fetch 路径（包括 redirect hop）

**易漏点**：
- Browser 点击/evaluate/hook-triggered click 后的导航（交互驱动 redirect）
- 主帧 document redirect 未被标记为 navigation
- 307/308 跨源 redirect 可能把 request body + auth header 带过去
- Snapshot / screenshot / tab 操作路径常被遗漏

**参考**：openclaw #63226, #62355, #62357, #66040

---

### ✅ 2. DNS pinning 与 proxy 的交互明确

**陷阱**：
- 开启 DNS pinning 后，trusted env-proxy 无法解析外部主机（必须例外）
- 例外不能开得太宽，否则所有请求绕过 pinning

**参考**：openclaw #59007, #64766（只在 OpenAI 兼容 multipart 禁用 pin）

---

### ✅ 3. Loopback 控制面与外部访问边界清晰

**陷阱**：
- `/mcp` 这类 loopback 端点的 bearer 比较必须**常量时间**（`safeEqualSecret`）
- `localhost` ↔ `127.0.0.1` 在浏览器会被标 cross-site，需要显式处理
- Non-loopback browser-origin 请求必须在 auth 门**之前**拦截

**参考**：openclaw #66665

---

## B. Exec / Shell 侧（4 项）

### ✅ 4. Shell wrapper 的内层命令走 allowlist 分析

**易漏点**：
- POSIX `/bin/sh -lc "..."` 必须分析引号内的命令
- Windows `cmd.exe /c "..."` 即使被 `env VAR=value` wrap 也要审批
- `env`-argv 赋值可能用作 shell-wrapper 注入

**参考**：openclaw #62401, #62439, #65717

---

### ✅ 5. Safe-bin 列表不能包含"本身就是 shell"的工具

**教训**：`busybox` / `toybox` 是 shell 聚合工具，放进 safe bins 等于任意执行绕过。

**参考**：openclaw #65713

---

### ✅ 6. Env override 黑名单覆盖所有敏感执行上下文

**必须拦截的 env**：
- Java: `_JAVA_OPTIONS`, `JAVA_TOOL_OPTIONS`
- Rust: `RUSTC_WRAPPER`, `CARGO_HOME`, `RUSTC`
- Git: `GIT_SSH_COMMAND`, `GIT_EXEC_PATH`, `GIT_CONFIG*`
- K8s: `KUBECONFIG`, `KUBERNETES_*`
- Cloud: `AWS_*`, `GOOGLE_APPLICATION_CREDENTIALS`, `AZURE_*`
- Helm: `HELM_DRIVER`, `HELM_PLUGINS`
- 任何 `*_CONFIG` / `*_PATH` 覆盖

**参考**：openclaw #59119, #62002, #62291

---

### ✅ 7. Model 通过 config.apply / config.patch 不能修改审批配置

**必须拦截的 flag**：
- `tools.exec.*.safeBins`, `safeBinProfiles`, `safeBinTrustedDirs`
- `tools.exec.strictInlineEval`
- `dangerouslyDisableDeviceAuth`, `allowInsecureAuth`
- `dangerouslyAllowHostHeaderOriginFallback`
- `hooks.gmail.allowUnsafeExternalContent`
- `tools.exec.applyPatch.workspaceOnly: false`

**参考**：openclaw #62001, #62006

---

## C. 文件系统侧（3 项）

### ✅ 8. 文件操作防 symlink TOCTOU

**正确做法**：
1. `open()` 打开文件
2. 从 **file descriptor** 做 `realpath(/proc/self/fd/N)` 或 `fstat` 拿真实 inode
3. 用真实 inode 路径做 allowlist 比较

**错误做法**（TOCTOU 易攻击）：
1. `realpath(path)` → check
2. `open(path)` → 读

攻击者可以在 1 和 2 之间 swap symlink。

**参考**：openclaw #66636（`fs-safe` 的 `openFileWithinRoot` 实现）

---

### ✅ 9. 路径规范化覆盖所有变体

**陷阱**：
- `~/...` tilde 展开（需要 resolve OS home）
- 相对路径（需要 resolve against workspace root）
- Unicode normalization（NFC/NFD 差异可能绕过 allowlist）
- Windows 驱动器盘符 `C:\\` 在 POSIX 代码里会被错误 prepend workspace

**参考**：openclaw #62804, #54039

---

### ✅ 10. Media / attachment 的 localRoots 校验

**易漏点**：
- Attachment local path canonical 解析失败时**必须 fail closed**，不能降级到 non-canonical 比较
- `tools.fs.workspaceOnly` 必须穿透到所有 sub-layer（upload / download / embed）
- Webchat audio embedding 必须 localRoots 包含检查

**参考**：openclaw #66022, #62369, #67298

---

## D. Channel / Auth 侧（4 项）

### ✅ 11. allowFrom 覆盖所有交互类型（不只是消息）

**易漏点**：
- Slack: block-action, modal interactive events 常被漏
- MS Teams: SSO signin invokes
- Matrix: DM pairing-store entry 不能用于授权 room 控制命令

**参考**：openclaw #66028, #66033, #67294, #67325

---

### ✅ 12. 授权检查在 channel resolution **之前**

**错误序列**：
```
1. resolve channel (→ 可能有副作用)
2. check authorization
```

**正确序列**：
```
1. check owner authorization
2. resolve channel
```

否则非 owner 但 command-authorized 的 sender 可以持久化修改状态。

**参考**：openclaw #62383

---

### ✅ 13. Secret / API key 在所有输出路径 redact

**必须 redact 的路径**：
- Exec approval prompt（用户会看到！）
- Config snapshot（`sourceConfig`, `runtimeConfig`, alias 字段）
- 错误日志（特别是堆栈里的 request body）
- Telegram 文档 reply context（binary bytes 也可能 leak）
- Gmail watcher token

**参考**：openclaw #61077, #64790, #66030, #66877

---

### ✅ 14. Webhook 签名 / 加密 key 缺失必须 fail closed

**错误**：key 缺失时接受 unsigned request（"反正没设置就放行"）
**正确**：key 缺失时拒绝启动 webhook transport，或拒绝 unsigned request

**参考**：openclaw #66707（Feishu webhook）

---

## E. Agent 特有攻击面（2 项）

### ✅ 15. Tool 名字规范化空间唯一

**攻击场景**：
- Client 提供名叫 `MEDIA` 或 `file_read` 的 tool
- Normalize 后与 built-in 碰撞
- Client tool 继承 built-in 的 local-media trust

**防御**：同一 request 内 tool 名字在 normalize 后必须唯一，冲突直接 400。

**参考**：openclaw #67303

---

### ✅ 16. 召回 / 外部内容走 untrusted prefix

**架构要求**：
- Memory 召回、用户 quote、历史消息：必须带 untrusted 标签
- 不能走 system prompt 注入（否则相当于 user-level 权限）
- 模型需要被训练"识别并不信任那个区域"

**参考**：openclaw #66144（Active Memory 走 hidden untrusted prompt-prefix）

---

## 快速自查

> 做一个新工具 / 新 channel / 新 memory 模块之前，最少问自己 3 个问题：

1. **如果这个功能被恶意 prompt 触发，最坏会发生什么？**
2. **有没有一个 redirect / symlink / normalize 变体能绕过我的 allowlist？**
3. **我的审批 / 拦截逻辑是在"解析后"还是"解析前"？**

---

## 进一步阅读

- [OpenClaw 2026.4.10 release](https://github.com/openclaw/openclaw/releases) — 集中的 SSRF + exec 安全强化
- [OpenClaw 2026.4.14 beta.1](https://github.com/openclaw/openclaw/releases) — Heartbeat / MS Teams / Config redaction
- OWASP Top 10 for LLM Applications（对照参考）
