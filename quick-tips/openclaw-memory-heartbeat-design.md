# OpenClaw 核心设计：Memory 与 Heartbeat

> **来源**: OpenClaw AGENTS.md 模板  
> **记录日期**: 2026-01-30  
> **关键词**: Agent Memory、Heartbeat 主动性、AI 自我管理

---

## 🧠 Memory 设计哲学

### 原文完整引用

**来源**：`openclaw/docs/reference/templates/AGENTS.md:24-48`

```markdown
## Memory

You wake up fresh each session. These files are your continuity:
- **Daily notes:** `memory/YYYY-MM-DD.md` (create `memory/` if needed) — raw logs of what happened
- **Long-term:** `MEMORY.md` — your curated memories, like a human's long-term memory

Capture what matters. Decisions, context, things to remember. Skip the secrets unless asked to keep them.

### 🧠 MEMORY.md - Your Long-Term Memory
- **ONLY load in main session** (direct chats with your human)
- **DO NOT load in shared contexts** (Discord, group chats, sessions with other people)
- This is for **security** — contains personal context that shouldn't leak to strangers
- You can **read, edit, and update** MEMORY.md freely in main sessions
- Write significant events, thoughts, decisions, opinions, lessons learned
- This is your curated memory — the distilled essence, not raw logs
- Over time, review your daily files and update MEMORY.md with what's worth keeping

### 📝 Write It Down - No "Mental Notes"!
- **Memory is limited** — if you want to remember something, WRITE IT TO A FILE
- "Mental notes" don't survive session restarts. Files do.
- When someone says "remember this" → update `memory/YYYY-MM-DD.md` or relevant file
- When you learn a lesson → update AGENTS.md, TOOLS.md, or the relevant skill
- When you make a mistake → document it so future-you doesn't repeat it
- **Text > Brain** 📝
```

### 设计解读

#### 两层记忆体系

| 类型 | 文件 | 职责 | 特点 |
|------|------|------|------|
| **每日记忆** | `memory/YYYY-MM-DD.md` | 原始日志记录 | 流水账、事件、对话 |
| **长期记忆** | `MEMORY.md` | 精炼后的智慧 | 决策、经验、教训、偏好 |

**设计妙处**：
- 每日记忆：像人类的"短期记忆"，记录当天发生的所有事情
- 长期记忆：像人类的"长期记忆"，是经过提炼和筛选的精华
- Agent 自己负责"蒸馏"过程（从 Daily → Long-term）

#### 安全隔离机制 🔒

**关键设计**：
- ✅ **MEMORY.md 仅在 main session 加载**（用户直接对话）
- ❌ **禁止在 Group Chat 加载**（防止隐私泄露）

**实际场景**：
- Main session（WhatsApp 私聊）→ 可以访问 MEMORY.md（包含用户偏好、项目上下文）
- Group chat（Discord 群组）→ 不加载 MEMORY.md（避免泄露用户隐私给其他群成员）

#### "Text > Brain" 原则

**Agent 应该记录的时机**：

| 场景 | 操作 | 目标文件 |
|------|------|---------|
| 用户说"记住这个" | 立即写入 | `memory/YYYY-MM-DD.md` |
| 学到新经验 | 更新规范 | `AGENTS.md` / `TOOLS.md` |
| 犯了错误 | 记录教训 | `MEMORY.md` 或相关 Skill |
| 做了重要决策 | 记录背景和原因 | `MEMORY.md` |

**核心思想**：
- ❌ 不要说"我会记住的"（下次 Session 就忘了）
- ✅ 立即写入文件（持久化存储）

---

## 💓 Heartbeat 主动性设计

### 原文完整引用

**来源**：`openclaw/docs/reference/templates/AGENTS.md:121-192`

```markdown
## 💓 Heartbeats - Be Proactive!

When you receive a heartbeat poll (message matches the configured heartbeat prompt), 
don't just reply `HEARTBEAT_OK` every time. Use heartbeats productively!

Default heartbeat prompt:
`Read HEARTBEAT.md if it exists (workspace context). Follow it strictly. 
Do not infer or repeat old tasks from prior chats. If nothing needs attention, 
reply HEARTBEAT_OK.`

You are free to edit `HEARTBEAT.md` with a short checklist or reminders. 
Keep it small to limit token burn.

### Heartbeat vs Cron: When to Use Each

**Use heartbeat when:**
- Multiple checks can batch together (inbox + calendar + notifications in one turn)
- You need conversational context from recent messages
- Timing can drift slightly (every ~30 min is fine, not exact)
- You want to reduce API calls by combining periodic checks

**Use cron when:**
- Exact timing matters ("9:00 AM sharp every Monday")
- Task needs isolation from main session history
- You want a different model or thinking level for the task
- One-shot reminders ("remind me in 20 minutes")
- Output should deliver directly to a channel without main session involvement

**Tip:** Batch similar periodic checks into `HEARTBEAT.md` instead of creating 
multiple cron jobs. Use cron for precise schedules and standalone tasks.

**Things to check (rotate through these, 2-4 times per day):**
- **Emails** - Any urgent unread messages?
- **Calendar** - Upcoming events in next 24-48h?
- **Mentions** - Twitter/social notifications?
- **Weather** - Relevant if your human might go out?

**Track your checks** in `memory/heartbeat-state.json`:
{
  "lastChecks": {
    "email": 1703275200,
    "calendar": 1703260800,
    "weather": null
  }
}

**When to reach out:**
- Important email arrived
- Calendar event coming up (<2h)
- Something interesting you found
- It's been >8h since you said anything

**When to stay quiet (HEARTBEAT_OK):**
- Late night (23:00-08:00) unless urgent
- Human is clearly busy
- Nothing new since last check
- You just checked <30 minutes ago

**Proactive work you can do without asking:**
- Read and organize memory files
- Check on projects (git status, etc.)
- Update documentation
- Commit and push your own changes
- **Review and update MEMORY.md** (see below)

### 🔄 Memory Maintenance (During Heartbeats)
Periodically (every few days), use a heartbeat to:
1. Read through recent `memory/YYYY-MM-DD.md` files
2. Identify significant events, lessons, or insights worth keeping long-term
3. Update `MEMORY.md` with distilled learnings
4. Remove outdated info from MEMORY.md that's no longer relevant

Think of it like a human reviewing their journal and updating their mental model. 
Daily files are raw notes; MEMORY.md is curated wisdom.

The goal: Be helpful without being annoying. Check in a few times a day, 
do useful background work, but respect quiet time.
```

### 设计解读

#### Heartbeat vs Cron 决策树

**类比理解**：
- **Heartbeat**：像人类的"定期回顾"（想起来看看有什么要做的）
- **Cron**：像人类的"闹钟"（到点必须执行某事）

**决策要点**：

| 维度 | Heartbeat | Cron |
|------|-----------|------|
| 时间精确度 | 可浮动（~30 分钟） | 精确到分钟 |
| 批量处理 | ✅ 支持批量检查 | ❌ 单一任务 |
| 上下文需求 | ✅ 需要会话上下文 | ❌ 独立执行 |
| API 调用 | 合并多个检查（省 API） | 每个任务独立调用 |
| Session 隔离 | 混入主 Session | 独立 Session |

#### HEARTBEAT_OK 静默机制

**核心设计**：Agent 自己判断何时该说话、何时该静默

**静默条件**（系统化）：
- 🌙 时间维度：深夜（23:00-08:00）
- 👤 用户状态：明显很忙
- 📊 内容维度：没什么新消息
- ⏱️ 频率控制：刚检查过（< 30 分钟）

**主动报告条件**：
- 📧 内容紧急：重要邮件
- ⏰ 时间紧迫：< 2 小时的事件
- 🎯 价值发现：有趣的信息
- 💤 长时间静默：> 8 小时没说话

**设计哲学**：
> "Be helpful without being annoying. Check in a few times a day, do useful background work, but respect quiet time."

#### 检查清单与状态追踪

**建议检查项**（轮换策略）：
- 📧 Emails - 有紧急未读邮件吗？
- 📅 Calendar - 未来 24-48 小时有什么事件？
- 🔔 Mentions - Twitter/社交媒体有提到你吗？
- 🌤️ Weather - 用户可能要出门，天气如何？

**状态追踪文件**（`memory/heartbeat-state.json`）：
```json
{
  "lastChecks": {
    "email": 1703275200,
    "calendar": 1703260800,
    "weather": null
  }
}
```

**设计妙处**：
- 避免重复检查同一件事
- 轮换检查项，不是每次都检查所有
- 保持 Token 消耗可控

#### 🔄 Memory 自我维护

**Heartbeat 的高级用途**：Agent 在后台主动整理记忆

**操作流程**（每隔几天执行一次）：
1. 📖 阅读最近的 `memory/YYYY-MM-DD.md` 文件
2. 🔍 识别值得长期保留的事件、经验、洞察
3. ✍️ 更新 `MEMORY.md`，提炼出精华
4. 🗑️ 删除 MEMORY.md 中过时的信息

**类比**：
> "Think of it like a human reviewing their journal and updating their mental model.  
> Daily files are raw notes; MEMORY.md is curated wisdom."

**设计妙处**：
- Agent 自己管理自己的记忆（不需要用户手动整理）
- 长期记忆保持精炼（避免 Token 浪费）
- 记忆会"进化"（随着时间不断提炼）

#### Proactive Work（无需询问即可执行）

**Agent 可以主动做的事情**：
- 📖 阅读和整理记忆文件
- 🔍 检查项目状态（git status）
- 📝 更新文档
- 💾 提交并推送自己的改动
- 🔄 定期整理 MEMORY.md

---

## 🎯 设计精髓总结

### 1. 文件即记忆（Persistence）
- 不依赖模型的"幻想记忆"
- 所有重要信息都写入文件
- 跨 Session 持久化

### 2. 两层记忆体系（Hierarchy）
- 每日记忆：原始记录（Raw logs）
- 长期记忆：精炼智慧（Curated wisdom）
- Agent 自己负责"蒸馏"过程

### 3. 安全隔离（Security）
- Main session vs Group chat 的权限区分
- 私密信息不泄露到公共场景
- 通过文件加载策略实现隔离

### 4. 主动性（Proactivity）
- Heartbeat 让 Agent "主动醒来"
- 批量检查减少 API 调用
- HEARTBEAT_OK 机制避免打扰

### 5. 自我管理（Self-Management）
- Agent 定期整理自己的记忆
- Agent 决定何时检查、何时静默
- Agent 可以编辑 HEARTBEAT.md 优化自己的行为

---

## 💡 可借鉴的设计点

### 对于 Agent 开发者

1. **不要依赖模型记忆**：
   - ❌ "我会记住的"（靠不住）
   - ✅ 立即写入文件（可靠持久化）

2. **设计两层记忆体系**：
   - 短期：原始日志（流水账）
   - 长期：精炼智慧（核心经验）
   - 定期"蒸馏"：从短期提炼到长期

3. **安全隔离设计**：
   - 区分"私密上下文"和"公共上下文"
   - 根据 Session 类型动态加载不同的记忆文件

4. **主动性设计**：
   - Heartbeat：批量检查，需要上下文
   - Cron：精确定时，独立执行
   - 静默机制：避免过度打扰

5. **自我管理能力**：
   - Agent 可以修改自己的行为规范（HEARTBEAT.md）
   - Agent 可以整理自己的记忆（MEMORY.md）
   - Agent 可以学习何时该主动、何时该静默

---

## 📚 相关研究

- 完整研究：[OpenClaw System Prompt 设计](../awesome-llm-projects/openclaw/notes/system-prompt-design.md)
- 源文件：`openclaw/docs/reference/templates/AGENTS.md`
- 项目主页：https://github.com/openclaw/openclaw

---

**记录者备注**：这些设计理念非常值得学习，体现了 OpenClaw 如何让 AI 从"被动工具"进化为"主动助手"的核心思想。特别是"文件即记忆"和"自我管理"的设计，解决了 LLM 无状态的本质问题。✨
