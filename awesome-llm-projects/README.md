# 优秀 LLM 开源项目研究

本目录用于深入研究优秀的 LLM 相关开源项目，通过阅读和分析实际代码来学习前沿技术和工程实践。

## 📁 目录结构

```
awesome-llm-projects/
  ├── AGENTS.md              # Agent 工作规范（必读）
  ├── README.md              # 本文件
  └── <project-name>/        # 项目研究文件夹
      ├── <repo-name>/       # 🔗 Git Submodule（项目源代码）
      └── notes/             # 📝 研究笔记
          ├── overview.md
          ├── architecture.md
          └── ...
```

**示例**（研究 DeepSpeed 项目）：
```
awesome-llm-projects/
  └── deepspeed/             # 项目研究文件夹
      ├── DeepSpeed/         # 🔗 Submodule（DeepSpeed 源代码）
      └── notes/             # 📝 DeepSpeed 研究笔记
```

## 🎯 研究方法

### 1. 添加新项目

```bash
# 进入 awesome-llm-projects 目录
cd awesome-llm-projects

# 创建项目研究文件夹
mkdir <项目名称>

# 在项目文件夹下添加 submodule
git submodule add <项目Git地址> <项目名称>/<仓库名称>

# 创建笔记目录
mkdir <项目名称>/notes
```

**示例**（添加 DeepSpeed）：
```bash
mkdir deepspeed
git submodule add https://github.com/microsoft/DeepSpeed.git deepspeed/DeepSpeed
mkdir deepspeed/notes
```

### 2. 研究流程

1. **克隆代码**：通过 git submodule 添加项目到 `<project>/<repo>/`
2. **创建笔记目录**：在 `<project>/notes/` 下创建笔记文件
3. **阅读代码**：从 README 开始，逐步深入核心代码
4. **记录笔记**：将发现和理解记录到 `<project>/notes/` 中
5. **持续追踪**：定期更新 submodule，记录技术演进

### 3. 笔记组织

每个项目的 `notes/` 目录建议包含：

- `overview.md` - 项目概览（目标、特点、使用场景）
- `architecture.md` - 架构分析（模块划分、依赖关系）
- `core-insights.md` - 核心发现（关键技术、巧妙设计）
- `code-reading-log.md` - 代码阅读日志（按时间记录）
- `references.md` - 相关资料（论文、博客、issues）

## 🔥 核心原则

**忠于代码**：所有分析必须基于实际代码，引用具体文件和行号，绝不臆测。

详细规范请阅读 [AGENTS.md](./AGENTS.md)。

## 🚀 研究重点

研究开源项目时重点关注：

1. **架构设计**：模块如何组织？为什么这样设计？
2. **核心算法**：关键技术如何实现？有哪些优化？
3. **工程技巧**：如何处理大规模数据？如何优化性能？
4. **配置管理**：超参数如何设置？如何管理不同环境？
5. **最佳实践**：代码风格、测试策略、文档规范

## 📚 可能的研究方向

- **训练框架**：DeepSpeed、Megatron-LM、ColossalAI 等
- **推理优化**：vLLM、TensorRT-LLM、llama.cpp 等
- **微调工具**：LLaMA-Factory、trl、axolotl 等
- **评估工具**：lm-evaluation-harness、OpenCompass 等
- **部署方案**：text-generation-inference、Xinference 等

## 💡 使用建议

1. **一次专注一个项目**：深入研究一个项目比浅尝多个项目更有价值
2. **动手实验**：不仅读代码，还要运行和修改代码
3. **追溯来源**：遇到关键技术点，追溯到相关论文
4. **记录演进**：定期更新 submodule，观察技术演进
5. **对比学习**：研究多个同类项目，对比不同实现方案

## 🔗 与主项目的关系

本目录是 `llm-training` 知识库的一部分，研究发现可以：

- 补充到主项目的技术文档中（记得标注来源）
- 验证主项目中的技术描述是否准确
- 为主项目提供实际案例和代码示例

---

**记住：代码是最可靠的老师，实践是最好的学习方式。** 🚀✨
