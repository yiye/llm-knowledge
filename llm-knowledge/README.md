# LLM 知识体系

> 系统性、体系化的 LLM 知识库，从基础到进阶的完整学习路径

## 📚 知识结构

本知识体系采用**分层递进**的组织方式：

```
foundations/      → 基础层：数学、概念
core-concepts/    → 核心层：LLM 本质
advanced/         → 进阶层：训练、对齐、推理
career-transition/ → 学习路径：转型指南
```

---

## 🏗️ 基础层（Foundations）

**目标**：为理解 LLM 打下坚实的数学和概念基础

### 📐 数学基础
- [L-1：数学基础](./foundations/L-1-math-foundations.md)
  - 线性代数、概率论、微积分

### 📊 核心概念
- [L-2：点积与相似度](./foundations/L-2-dot-product-similarity.md)
  - 向量、点积、余弦相似度
- [L-3：表示学习](./foundations/L-3-representation-learning.md)
  - Embedding、语义空间

**推荐学习顺序**：L-1 → L-2 → L-3

---

## 🧠 核心层（Core Concepts）

**目标**：深入理解 LLM 的本质和工作原理

- [L0：LLM 模型本质](./core-concepts/L0-llm-model-essence.md)
  - 什么是大语言模型
  - Token、概率分布、自回归生成

- [L0.1：LLM 与人类心智](./core-concepts/L0.1-llm-vs-human-mind.md)
  - LLM 与人类思维的异同
  - 理解 LLM 的认知边界

- [L0.2：稀疏架构演进](./core-concepts/L0.2-sparse-architecture-evolution.md)
  - MoE（Mixture of Experts）
  - 从稠密到稀疏的架构演进

**推荐学习顺序**：L0 → L0.1 → L0.2

---

## 🚀 进阶层（Advanced）

**目标**：掌握 LLM 的训练、优化和应用

### 训练与微调
- [L+1：预训练与微调](./advanced/L+1-pretraining-and-finetuning.md)
  - 预训练 vs 微调
  - LoRA、QLoRA 等高效微调方法

### 对齐技术
- [L+2：对齐 - RLHF 与 DPO](./advanced/L+2-alignment-rlhf-dpo.md)
  - 偏好对齐的必要性
  - RLHF、DPO 技术演进

- [L+2：DPO 实践](./advanced/L+2-dpo-practice.md)
  - DPO 的实际应用和案例

- [L+2：推理能力](./advanced/L+2-reasoning.md)
  - Chain-of-Thought、Tree-of-Thought
  - 推理能力的提升方法

### 行业实践
- [L+3：里程碑事件](./advanced/L+3-milestones.md)
  - ChatGPT、GPT-4、Claude、DeepSeek-V3 等重要节点

- [L+3：实践指南](./advanced/L+3-practical-guide.md)
  - 从理论到实践的落地指南

- [L+3：训练差异](./advanced/L+3-training-differences.md)
  - 不同规模模型的训练差异

**推荐学习顺序**：L+1 → L+2系列 → L+3系列

---

## 🎓 学习路径（Career Transition）

**面向程序员的系统性转型指南**

完整的 15 章学习路径，从编程思维到 LLM 思维的系统转变：

### 第一部分：思维转变与基础准备（01-04章）
- [01：思维转变](./career-transition/career-programmer-to-llm/01-mindset-shift.md)
- [02：技能迁移地图](./career-transition/career-programmer-to-llm/02-skill-migration-map.md)
- [03：学习路径设计](./career-transition/career-programmer-to-llm/03-learning-path-design.md)
- [04：数学基础补充](./career-transition/career-programmer-to-llm/04-math-fundamentals.md)

### 第二部分：核心技术掌握（05-10章）
- [05：Transformer 架构](./career-transition/career-programmer-to-llm/05-transformer-architecture.md)
- [06：LoRA/QLoRA 实践](./career-transition/career-programmer-to-llm/06-lora-qlora-practice.md)
- [07：RAG 系统指南](./career-transition/career-programmer-to-llm/07-rag-system-guide.md)
- [08：DPO 对齐](./career-transition/career-programmer-to-llm/08-dpo-alignment.md)
- [09：推理优化](./career-transition/career-programmer-to-llm/09-inference-optimization.md)
- [10：分布式训练](./career-transition/career-programmer-to-llm/10-distributed-training.md)

### 第三部分：工程实践与职业发展（11-15章）
- [11：MLOps 实践](./career-transition/career-programmer-to-llm/11-mlops-practices.md)
- [12：训练基础设施](./career-transition/career-programmer-to-llm/12-training-infrastructure.md)
- [13：生产环境部署](./career-transition/career-programmer-to-llm/13-production-deployment.md)
- [14：职业路径](./career-transition/career-programmer-to-llm/14-career-paths.md)
- [15：行业案例](./career-transition/career-programmer-to-llm/15-industry-cases.md)

**推荐学习方式**：按章节顺序学习，结合主体知识库的对应章节深入

---

## 🎯 推荐学习路径

### 快速入门（1-2周）
```
基础层 → 核心层 → 进阶层（L+2 对齐）
L-1 → L-2 → L0 → L+2（RLHF/DPO）
```

### 系统学习（2-3个月）
```
完整路径：基础 → 核心 → 进阶 → 学习路径（career-transition）
foundations → core-concepts → advanced → career-transition
```

### 专项深入
- **关注训练**：L+1、L+3-training-differences、career-transition (10-12章)
- **关注对齐**：L+2系列、career-transition (08章)
- **关注应用**：L+2-reasoning、career-transition (07, 13章)

---

## 📖 撰写规范

撰写或更新知识体系文档时，请参考 [AGENTS.md](./AGENTS.md)。

核心要求：
- ✅ 分层清晰：遵循 foundations → core → advanced 的递进关系
- ✅ 系统完整：文档之间互相引用，形成知识网络
- ✅ 理论+实践：既有原理，也有代码和案例
- ✅ 有据可查：所有技术内容必须有可靠来源

---

## 🔗 相关资源

- **开源项目研究**：[awesome-llm-projects/](../awesome-llm-projects/)
- **零散知识片段**：[quick-tips/](../quick-tips/)
- **产品创意探索**：[llm-ideas/](../llm-ideas/)
- **主项目规范**：[AGENTS.md](../AGENTS.md)

---

**欢迎探索 LLM 知识的奇妙世界！** 🚀✨
