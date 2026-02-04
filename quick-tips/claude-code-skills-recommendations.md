# Claude Code Skills 实践推荐 - 8个高频使用工具

## 💡 核心观点

来自一线开发者的实践经验，推荐 8 个真正能转化为生产力的 Claude Code Skills。涵盖项目管理、开发流程、前端设计、知识管理、视频制作、科研写作等场景。

## 🔑 核心内容

### 1. planning-with-files：加强版 Plan 模式 ⭐

**核心思路**：把 Context Window 当成 RAM，把文件系统当成磁盘。

**工作机制**：
- 自动创建三个 Markdown 文件：
  - `task_plan.md`（任务规划）
  - `findings.md`（发现记录）
  - `progress.md`（进度日志）
- 每做两步操作自动保存一次
- 每次重要决策前重新读一遍计划文件

**适用场景**：复杂项目、长周期开发任务

**项目地址**：https://github.com/OthmanAdi/planning-with-files

---

### 2. superpowers：规范驱动开发（SDD）的 Skills 版 ⭐

**核心理念**：不要让 AI 上来就写代码。

**三板斧工作流**：
1. `/superpowers:brainstorm` - 需求探索和设计
2. `/superpowers:write-plan` - 拆分实现计划
3. `/superpowers:execute-plan` - 分批执行

**特性**：
- 强制执行 TDD（测试驱动开发）
- 先写测试 → 再写代码 → 最后重构
- 内置代码审查、系统化调试、Git Worktree 管理等 10+ 子 Skills
- 支持上下文自动触发（无需手动调用）

**适用人群**：不喜欢 Claude Code 过于"莽撞"的开发者

**项目地址**：https://github.com/obra/superpowers

---

### 3. frontend-design：告别千篇一律的 AI UI

**问题痛点**：AI 生成的前端页面都长得一样（Inter 字体 + 紫色渐变 + 圆角卡片）

**改进方向**：
- 选择独特的字体组合（不再默认 Arial 和 Inter）
- 使用更有视觉冲击力的配色方案
- 根据设计风格匹配对应的实现复杂度
  - 极简风格 = 精准的留白和排版（而非简单地少放元素）

**最佳搭配**：React + Tailwind

**注意**：主要改善视觉设计层面，交互逻辑和业务功能仍需人工把关

**项目地址**：https://github.com/anthropics/skills（Anthropic 官方出品）

---

### 4. NotebookLM Skill：查询个人知识库

**核心能力**：让 Claude Code 直接和 Google NotebookLM 对话，查询上传到 NotebookLM 的文档，获取带引用来源的回答。

**使用场景**：
- 上传项目的技术文档、设计规范、API 文档到 NotebookLM
- Claude Code 写代码时直接查询这些文档（而非靠猜测）
- 大幅降低幻觉概率（答案来自用户自己上传的资料）

**特性**：
- 支持自动追问（挖掘细节信息、边界情况、最佳实践）
- 认证持久化（登录一次即可）

**项目地址**：https://github.com/PleasePrompto/notebooklm-skill

---

### 5. Obsidian Skills：Obsidian 官方认证

**作者**：kepano（Obsidian 创始人 Steph Ango）

**支持格式**：
- Obsidian Flavored Markdown
- Obsidian Bases
- JSON Canvas

**安装方式**：放到 Obsidian vault 的 `.claude` 目录下，兼容 Claude Code 和 Codex CLI

**使用场景**：
- 整理会议记录
- 生成项目文档
- 创建 Canvas 看板

**社区认可度**：GitHub 8.7k ⭐

**生态扩展**：
- `obsidian-claude-code`（原生插件）
- `obsidian-plugin-skill`（Obsidian 插件开发专用）

**项目地址**：https://github.com/kepano/obsidian-skills

---

### 6. Remotion Skills：用代码做视频

**技术框架**：Remotion（用 React 做视频，视频的每一帧都是一个 React 组件）

**工作流程**：
1. 用自然语言描述视频效果
2. Claude 生成对应的 React 代码
3. 实时预览
4. 渲染成 MP4

**社区增强功能**：
- TTS 语音生成
- 预置的标题幻灯片
- 代码块展示
- 图表动画

**适用场景**：技术演示视频、产品介绍短片

**优势**：精确控制动画细节，所有内容都是代码定义（修改比传统视频剪辑方便）

**使用建议**：不要一次描述整个视频，一个 composition 一个 composition 地迭代

**项目地址**：
- 官方文档：https://www.remotion.dev/docs/ai/skills
- 社区增强版：https://github.com/wshuyi/remotion-video-skill

---

### 7. Skill Creator：快速创建自定义 Skill ⭐

**官方出品**：Anthropic 官方 Skill

**功能**：通过问答方式引导创建新的 Skill，无需写代码。

**创建过程**：
- 定义 Skill 的名称、描述、触发条件、具体指令
- 最终生成规范的 `SKILL.md` 文件

**Skill 结构**：
- YAML 元数据（名称和描述）
- Markdown 格式的指令
- 加载时仅消耗 ~100 tokens（元数据扫描）
- 激活后 < 5k tokens（非常轻量）

**适用场景**：
- 特定框架的最佳实践
- 团队编码规范
- 特定领域的专业知识

**学习价值**：入门 Skills 开发的最快路径

**项目地址**：https://github.com/anthropics/skills

---

### 8. research-skills：科研写作工具包

**作者**：鲁工（文章作者）

**面向场景**：医学影像 AI 方向的综述论文和研究计划

**三个 Skill**：

1. **Medical Imaging Review**：医学影像 AI 综述写作
   - 结构化的 7 阶段写作流程
   - 覆盖 CCTA、肺部、脑部、心脏、病理、眼底等细分领域
   - 支持 Zotero 文献管理集成

2. **Paper Slide Deck**：论文自动转 PPT
   - 从 PDF 论文中自动检测和提取图片
   - 提供 17 种视觉风格模板
   - 集成 Nano Banana API 生成 slide
   - 导出 PPTX 和 PDF

3. **Research Proposal**：博士研究计划撰写
   - 遵循 Nature Reviews 风格的学术写作规范
   - 支持中英双语输出
   - 最终产出 2000-4000 词的研究计划
   - 包含 40+ 参考文献

**项目地址**：https://github.com/luwill/research-skills

---

## 📌 场景速查表

| 场景 | 推荐 Skill |
|------|-----------|
| 项目管理 | planning-with-files |
| 开发流程 | superpowers |
| 前端设计 | frontend-design |
| 知识查询 | NotebookLM Skill |
| 笔记管理 | Obsidian Skills |
| 视频制作 | Remotion Skills |
| Skills 开发 | Skill Creator |
| 科研写作 | research-skills |

---

## 🛠️ 安装方式

**三种方式**：
1. `/plugin install` 命令一键安装（已上架 marketplace 的 Skills）
2. 手动 clone 到 `~/.claude/skills/` 目录
3. 通过 vercel 的 `npx skill` 命令安装

具体安装方式见各 Skills 的 GitHub README。

---

## ⚠️ 使用建议

- **宜精不宜多**：跟 MCP 一样，不要装太多 Skills
- **定期清理**：及时移除不用的 Skills
- **关注更新**：Skills 生态正在快速发展（2026 年初）

---

## 📚 参考来源

- [推荐我日常高频使用的8个Skills，产出效率翻一倍 - AI编程实验室](https://mp.weixin.qq.com/s/oaE-__gKDQdNzUdKSYdJhA) - 2026年2月

---

> 💭 **启示**：
> 
> **planning-with-files**、**superpowers**、**Skill Creator** 三个工具的启发性在于：
> - **planning-with-files**：通过文件系统持久化思考过程，突破 Context Window 限制
> - **superpowers**：规范驱动开发（SDD），强制执行 TDD，避免 AI 写代码过于莽撞
> - **Skill Creator**：降低自定义 Skill 的门槛，可以快速沉淀团队/个人的工作流和最佳实践
> 
> Skills 生态正处于爆发期（2026 年初），值得持续关注。
