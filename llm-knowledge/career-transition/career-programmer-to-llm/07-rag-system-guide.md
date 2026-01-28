# 07 - RAG 系统构建指南：从检索到生成的完整链路

> 🎯 **核心观点**：RAG（Retrieval-Augmented Generation）是当前最实用的 LLM 应用模式！本文将从零构建生产级 RAG 系统，涵盖架构设计、向量检索、Reranking、到完整实现。

---

## 📖 引言：为什么需要 RAG？

### ❌ 纯 LLM 的局限

```
场景：用 GPT-4 回答"2024年第四季度公司营收是多少？"

问题：
  ❌ LLM 的知识截止日期（如 2023 年 4 月）
  ❌ 没有访问实时数据的能力
  ❌ 容易产生幻觉（编造不存在的数据）
  ❌ 无法引用来源

结果：
  "抱歉，我的知识截止到 2023 年 4 月，无法回答。"
```

### ✅ RAG 的解决方案

根据 [AWS RAG Best Practices](https://docs.aws.amazon.com/prescriptive-guidance/latest/writing-best-practices-rag/introduction.html)（2025）：

> RAG 结合大型语言模型与外部知识源，通过将 AI 系统锚定在最新、特定领域的信息上，提供更准确、事实性的响应，而无需大量微调。

**工作流程**：

```
用户问题："2024 Q4 营收是多少？"
    ↓
1️⃣ 检索（Retrieval）
   在知识库中搜索相关文档
   找到: "2024 Q4 财报.pdf" 第 3 页
    ↓
2️⃣ 增强（Augmentation）
   把检索到的内容拼接到 Prompt
   "根据以下文档回答：
    [2024 Q4 营收为 12.5 亿美元...]
    问题：2024 Q4 营收是多少？"
    ↓
3️⃣ 生成（Generation）
   LLM 基于文档生成答案
   "根据财报，2024 Q4 营收为 12.5 亿美元，
    同比增长 23%。"
```

---

### 🔥 RAG vs 微调 vs Prompt Engineering

| 维度 | RAG | Fine-tuning | Prompt Engineering |
|-----|-----|-------------|-------------------|
| **知识更新** | 实时（更新知识库） | 需重新训练 | 受 Context 长度限制 |
| **成本** | 低（检索 + 推理） | 高（训练 + GPU） | 低（只推理） |
| **准确性** | 高（有来源引用） | 高 | 中（易产生幻觉） |
| **响应速度** | 中（检索有开销） | 快 | 快 |
| **可解释性** | 高（可追溯来源） | 低 | 低 |
| **适用场景** | 知识密集型任务 | 改变模型行为 | 简单任务 |

根据 [Microsoft RAG Techniques 2025](https://www.microsoft.com/en-us/microsoft-cloud/blog/2025/02/04/common-retrieval-augmented-generation-rag-techniques-explained/)：

> RAG 是当前企业应用 LLM 的主流方式，无需大量微调即可整合最新、特定领域的数据。

---

## 一、RAG 架构演进

### 1.1 Naive RAG（基础版）

**架构**：

```
用户问题
    ↓
Embedding 编码
    ↓
向量数据库检索（Top-K）
    ↓
拼接到 Prompt
    ↓
LLM 生成答案
```

**代码示例**：

```python
from langchain.embeddings import OpenAIEmbeddings
from langchain.vectorstores import Chroma
from langchain.llms import OpenAI
from langchain.chains import RetrievalQA

# 1. 创建向量数据库
embeddings = OpenAIEmbeddings()
vectorstore = Chroma.from_texts(
    texts=["文档1内容", "文档2内容", "文档3内容"],
    embedding=embeddings
)

# 2. 创建 RAG 链
llm = OpenAI(temperature=0)
qa_chain = RetrievalQA.from_chain_type(
    llm=llm,
    retriever=vectorstore.as_retriever(search_kwargs={"k": 3})
)

# 3. 提问
question = "2024 Q4 营收是多少？"
answer = qa_chain.run(question)
print(answer)
```

**问题**：
- ❌ 检索质量差（语义不匹配）
- ❌ 返回无关文档（噪声）
- ❌ Context 过长（超出 LLM 窗口）
- ❌ 无法处理复杂查询

---

### 1.2 Advanced RAG（进阶版）

**改进点**：

```
1️⃣ Pre-Retrieval 优化
   ├─ Query Rewriting（查询重写）
   ├─ Query Expansion（查询扩展）
   └─ Query Decomposition（查询分解）

2️⃣ Retrieval 优化
   ├─ Hybrid Search（混合检索）
   ├─ Metadata Filtering（元数据过滤）
   └─ Contextual Chunking（上下文分块）

3️⃣ Post-Retrieval 优化
   ├─ Reranking（重排序）
   ├─ Context Compression（上下文压缩）
   └─ Answer Validation（答案验证）
```

**架构**：

```
用户问题
    ↓
Query Rewriting（重写）
    ↓
Query Decomposition（分解为子查询）
    ↓
并行检索（Dense + Sparse + Web）
    ↓
Reranking（重排序）
    ↓
Context Compression（压缩）
    ↓
LLM 生成 + 引用来源
```

---

### 1.3 Modular RAG（模块化，2025 主流）

根据 [Microsoft Advanced RAG](https://learn.microsoft.com/en-us/azure/developer/ai/advanced-retrieval-augmented-generation)（2025）：

> 模块化 RAG 将检索和生成过程分解为可独立优化的模块，支持灵活组合不同策略。

**核心模块**：

```
┌─────────────────────────────────────┐
│     RAG System Architecture         │
├─────────────────────────────────────┤
│  1. Ingestion Pipeline              │
│     ├─ Document Loading             │
│     ├─ Chunking Strategy            │
│     ├─ Metadata Extraction          │
│     └─ Vector Indexing              │
├─────────────────────────────────────┤
│  2. Query Processing                │
│     ├─ Query Understanding          │
│     ├─ Intent Classification        │
│     └─ Query Transformation         │
├─────────────────────────────────────┤
│  3. Retrieval Engine                │
│     ├─ Dense Retrieval              │
│     ├─ Sparse Retrieval             │
│     ├─ Hybrid Retrieval             │
│     └─ Multi-stage Retrieval        │
├─────────────────────────────────────┤
│  4. Reranking & Filtering           │
│     ├─ Cross-Encoder Reranking      │
│     ├─ Relevance Filtering          │
│     └─ Diversity Promotion          │
├─────────────────────────────────────┤
│  5. Generation & Post-processing    │
│     ├─ Context Assembly             │
│     ├─ Prompt Engineering           │
│     ├─ LLM Generation               │
│     └─ Citation & Verification      │
└─────────────────────────────────────┘
```

---

## 二、核心组件 1：Embedding 模型

### 2.1 Embedding 是什么？

**直观理解**：把文本转换为高维向量，语义相似的文本向量也相近。

```python
from sentence_transformers import SentenceTransformer

model = SentenceTransformer('sentence-transformers/all-MiniLM-L6-v2')

# 文本 → 向量
text1 = "深度学习模型训练"
text2 = "神经网络的训练过程"
text3 = "今天天气不错"

emb1 = model.encode(text1)
emb2 = model.encode(text2)
emb3 = model.encode(text3)

# 计算相似度
from sklearn.metrics.pairwise import cosine_similarity

sim_12 = cosine_similarity([emb1], [emb2])[0][0]
sim_13 = cosine_similarity([emb1], [emb3])[0][0]

print(f"文本1 vs 文本2 相似度: {sim_12:.4f}")  # 0.75（高）
print(f"文本1 vs 文本3 相似度: {sim_13:.4f}")  # 0.12（低）

# 🔥 语义相似的文本，向量也相似！
```

---

### 2.2 Embedding 模型对比（2025）

根据 [Best Embedding Models 2025](https://app.ailog.fr/en/blog/guides/choosing-embedding-models) 和 MTEB 排行榜：

#### **闭源模型**

| 模型 | MTEB Score | 维度 | 成本 | 特点 |
|-----|-----------|------|------|------|
| **Gemini-embedding-001** | 68.32 | 3072 | ~$0.004/1K tokens | 最佳多语言（100+ 语言） |
| **OpenAI text-embedding-3-large** | 64.6 | 3072 | $0.13/1M tokens | 通用性强，可调维度 |
| **Cohere embed-v4** | 65.2 | 1024 | $0.10/1M tokens | 企业级，抗噪音 |

#### **开源模型**

| 模型 | MTEB Score | 维度 | 许可证 | 特点 |
|-----|-----------|------|--------|------|
| **Qwen3-Embedding-8B** | 70.58 | 4096 | Apache 2.0 | 🔥 最佳开源模型 |
| **BGE-M3** | 63.0 | 1024 | MIT | 多语言，支持 8192 tokens |
| **E5-Mistral-7B** | 66.6 | 4096 | MIT | 超越早期闭源模型 |
| **all-MiniLM-L6-v2** | 56.3 | 384 | Apache 2.0 | 轻量快速，适合原型 |

**选择建议**（2025）：

```
场景 1：原型开发、本地运行
  → all-MiniLM-L6-v2（快速、轻量）

场景 2：生产环境、多语言支持
  → BGE-M3 或 E5-Mistral-7B（开源、高性能）

场景 3：企业级、预算充足
  → OpenAI text-embedding-3-large（稳定、生态好）

场景 4：超大规模、自建集群
  → Qwen3-Embedding-8B（最佳性能）
```

---

### 2.3 实战：对比不同 Embedding 模型

```python
from sentence_transformers import SentenceTransformer
from sklearn.metrics.pairwise import cosine_similarity
import numpy as np

# 加载不同模型
models = {
    "MiniLM": SentenceTransformer('sentence-transformers/all-MiniLM-L6-v2'),
    "BGE-M3": SentenceTransformer('BAAI/bge-m3'),
    "E5-large": SentenceTransformer('intfloat/e5-large-v2'),
}

# 测试数据
query = "如何训练深度学习模型？"
docs = [
    "深度学习模型的训练步骤包括数据准备、模型设计、损失函数选择...",  # 相关
    "神经网络训练需要大量数据和计算资源。",                        # 相关
    "今天天气晴朗，适合户外活动。"                                # 不相关
]

# 对比检索效果
for model_name, model in models.items():
    print(f"\n{'='*50}")
    print(f"模型: {model_name}")
    print(f"{'='*50}")
    
    query_emb = model.encode(query)
    doc_embs = model.encode(docs)
    
    similarities = cosine_similarity([query_emb], doc_embs)[0]
    
    for i, (doc, sim) in enumerate(zip(docs, similarities)):
        print(f"Doc {i+1} (相似度: {sim:.4f}): {doc[:30]}...")
    
    # 排序
    ranked_indices = np.argsort(similarities)[::-1]
    print(f"\n排序: {ranked_indices + 1}")  # +1 for 1-based indexing

# 🔥 观察：不同模型的相似度分数和排序可能不同
# 选择在你的数据上效果最好的模型！
```

---

## 三、核心组件 2：向量数据库

### 3.1 为什么需要向量数据库？

**问题**：数百万个文档的向量，如何高效检索？

```
暴力搜索（Brute Force）：
  - 计算查询向量与所有向量的相似度
  - 时间复杂度：O(n)
  - 百万文档：数秒到数十秒

向量数据库（HNSW/IVF）：
  - 近似最近邻搜索（ANN）
  - 时间复杂度：O(log n)
  - 百万文档：<100ms
```

---

### 3.2 向量数据库对比（2025）

根据 [Vector Database Comparison 2025](https://www.firecrawl.dev/blog/best-vector-databases-2025)：

#### **Pinecone**（最佳生产环境）

```
优势：
  ✅ 99.99% SLA，<50ms 延迟
  ✅ 完全托管，自动扩展
  ✅ 免费 100K 向量
  
劣势：
  ❌ 仅云端，供应商锁定
  ❌ 定制化受限

适用：企业生产、十亿级规模

定价：
  - Starter: 免费（100K 向量）
  - Standard: $70/月起
  - Enterprise: $200+/月
```

---

#### **Weaviate**（最佳灵活性）

```
优势：
  ✅ 开源 + 托管云
  ✅ 混合搜索（向量 + 关键词）
  ✅ 内置 ML 模型
  ✅ 多租户、GDPR 合规
  
劣势：
  ❌ 学习曲线陡峭
  ❌ 自托管需 DevOps 经验

适用：多模态 AI、知识图谱、私有部署

定价：
  - 开源: 免费
  - Serverless: $25/月起
  - Enterprise: $295/月起
```

---

#### **Chroma**（最佳开发体验）

```
优势：
  ✅ 5 分钟上手
  ✅ 原生 Python API
  ✅ LangChain 集成
  ✅ 轻量级，资源占用少
  
劣势：
  ❌ 扩展性有限（10M 向量软限制）
  ❌ 无内置备份

适用：RAG 原型、Python 项目、本地开发

定价：
  - 开源: 免费
  - Teams: $108/月
```

---

#### **Milvus**（最佳企业级规模）

```
优势：
  ✅ 设计支持数十亿到万亿向量
  ✅ 水平扩展
  ✅ 多种索引（IVF、HNSW、DiskANN）
  ✅ 强一致性保证
  
劣势：
  ❌ 复杂的 K8s 部署
  ❌ 陡峭的学习曲线

适用：大规模企业、海量数据

定价：
  - 开源: 免费
  - Zilliz Cloud: $100/月起
```

---

### 3.3 性能对比

根据 [TensorBlue Benchmark 2025](https://tensorblue.com/blog/vector-database-comparison-pinecone-weaviate-qdrant-milvus-2025)：

| 数据库 | P95 延迟 | 吞吐量 (QPS) | 内存效率 |
|-------|---------|-------------|---------|
| **Pinecone** | 40-50ms | 5,000-10,000 | ⭐⭐⭐⭐ |
| **Qdrant** | 30-40ms | 8,000-15,000 | ⭐⭐⭐⭐⭐ |
| **Weaviate** | 50-70ms | 3,000-8,000 | ⭐⭐⭐ |
| **Milvus** | 50-80ms | 10,000-20,000 | ⭐⭐⭐⭐ |
| **Chroma** | 60-100ms | 1,000-3,000 | ⭐⭐⭐⭐⭐ |

---

### 3.4 实战：使用 Chroma 构建向量数据库

```python
import chromadb
from chromadb.config import Settings
from sentence_transformers import SentenceTransformer

# 1. 初始化 Chroma 客户端
client = chromadb.Client(Settings(
    chroma_db_impl="duckdb+parquet",
    persist_directory="./chroma_db"
))

# 2. 创建集合
collection = client.create_collection(
    name="my_documents",
    metadata={"hnsw:space": "cosine"}  # 使用余弦相似度
)

# 3. 准备文档
documents = [
    "深度学习模型训练需要大量数据和计算资源。",
    "Transformer 架构是现代 NLP 的基础。",
    "RAG 系统结合了检索和生成能力。",
    "向量数据库用于高效的语义搜索。",
    "Python 是机器学习的首选语言。"
]

# 4. 生成 Embeddings
embedding_model = SentenceTransformer('sentence-transformers/all-MiniLM-L6-v2')
embeddings = embedding_model.encode(documents).tolist()

# 5. 插入向量数据库
collection.add(
    documents=documents,
    embeddings=embeddings,
    ids=[f"doc_{i}" for i in range(len(documents))],
    metadatas=[{"source": "manual", "index": i} for i in range(len(documents))]
)

# 6. 查询
query = "如何训练神经网络？"
query_embedding = embedding_model.encode(query).tolist()

results = collection.query(
    query_embeddings=[query_embedding],
    n_results=3
)

print("检索结果：")
for i, (doc, distance, metadata) in enumerate(zip(
    results['documents'][0],
    results['distances'][0],
    results['metadatas'][0]
)):
    print(f"\n{i+1}. (距离: {distance:.4f})")
    print(f"   文档: {doc}")
    print(f"   元数据: {metadata}")

# 7. 持久化
client.persist()

# 🔥 Chroma 自动处理向量索引和持久化！
```

---

## 四、核心组件 3：检索策略

### 4.1 Dense Retrieval（稠密检索）

**原理**：用 Embedding 模型编码，计算向量相似度。

```python
# Dense Retrieval 示例
query = "深度学习训练技巧"
query_embedding = embedding_model.encode(query)

# 在向量数据库中检索
results = collection.query(
    query_embeddings=[query_embedding.tolist()],
    n_results=5
)

# 优势：
# ✅ 语义理解强（"DL training" 能匹配 "深度学习训练"）
# ✅ 泛化能力好

# 劣势：
# ❌ 对关键词敏感度低（"GPT-4" 可能匹配不准）
# ❌ 冷启动问题（新领域效果差）
```

---

### 4.2 Sparse Retrieval（稀疏检索）

**原理**：基于关键词匹配（如 BM25）。

```python
from rank_bm25 import BM25Okapi

# 文档库
documents = [
    "深度学习模型训练需要大量数据",
    "GPT-4 是 OpenAI 的最新模型",
    "Transformer 架构基于注意力机制"
]

# 分词
tokenized_docs = [doc.split() for doc in documents]

# 构建 BM25 索引
bm25 = BM25Okapi(tokenized_docs)

# 查询
query = "GPT-4 模型"
tokenized_query = query.split()

# 检索
scores = bm25.get_scores(tokenized_query)
ranked_indices = sorted(range(len(scores)), key=lambda i: scores[i], reverse=True)

print("BM25 检索结果：")
for i in ranked_indices[:3]:
    print(f"{i+1}. (得分: {scores[i]:.4f}) {documents[i]}")

# 优势：
# ✅ 关键词精准匹配（"GPT-4" 一定能找到）
# ✅ 可解释性强

# 劣势：
# ❌ 语义理解弱（"DL training" 无法匹配 "深度学习训练"）
# ❌ 词序敏感
```

---

### 4.3 Hybrid Search（混合检索，2025 推荐）

根据 [Microsoft RAG Techniques](https://www.microsoft.com/en-us/microsoft-cloud/blog/2025/02/04/common-retrieval-augmented-generation-rag-techniques-explained/)：

> 混合检索结合多种方法，扩展搜索范围并提高准确性，是 2025 年的最佳实践。

**原理**：结合 Dense + Sparse，取长补短。

```python
from typing import List, Tuple

def hybrid_search(
    query: str,
    documents: List[str],
    embedding_model,
    collection,
    alpha: float = 0.5  # Dense 权重
) -> List[Tuple[int, float]]:
    """
    混合检索：Dense (alpha) + Sparse (1-alpha)
    """
    # 1. Dense Retrieval
    query_embedding = embedding_model.encode(query).tolist()
    dense_results = collection.query(
        query_embeddings=[query_embedding],
        n_results=len(documents)
    )
    
    # Dense 得分归一化
    dense_scores = {}
    for doc_id, distance in zip(dense_results['ids'][0], dense_results['distances'][0]):
        # 距离 → 相似度（1 - distance）
        dense_scores[doc_id] = 1 - distance
    
    # 2. Sparse Retrieval (BM25)
    tokenized_docs = [doc.split() for doc in documents]
    bm25 = BM25Okapi(tokenized_docs)
    tokenized_query = query.split()
    sparse_scores = bm25.get_scores(tokenized_query)
    
    # Sparse 得分归一化
    max_sparse = max(sparse_scores) if max(sparse_scores) > 0 else 1
    sparse_scores_norm = {f"doc_{i}": score / max_sparse for i, score in enumerate(sparse_scores)}
    
    # 3. 融合得分
    final_scores = {}
    for doc_id in dense_scores:
        dense_score = dense_scores.get(doc_id, 0)
        sparse_score = sparse_scores_norm.get(doc_id, 0)
        final_scores[doc_id] = alpha * dense_score + (1 - alpha) * sparse_score
    
    # 4. 排序
    ranked = sorted(final_scores.items(), key=lambda x: x[1], reverse=True)
    
    return ranked

# 测试
query = "GPT-4 训练技巧"
results = hybrid_search(query, documents, embedding_model, collection, alpha=0.7)

print("混合检索结果：")
for doc_id, score in results[:3]:
    print(f"  {doc_id} (得分: {score:.4f})")

# 🔥 Hybrid Search 兼顾语义和关键词匹配！
```

**Alpha 调优**：

```
alpha = 1.0  → 纯 Dense（语义优先）
alpha = 0.5  → 均衡
alpha = 0.0  → 纯 Sparse（关键词优先）

场景：
- 技术文档、代码检索：alpha = 0.3-0.5（关键词重要）
- 通用知识问答：alpha = 0.7-0.9（语义重要）
```

---

## 五、核心组件 4：Reranking（重排序）

### 5.1 为什么需要 Reranking？

**问题**：初始检索返回 100 个候选，但只能给 LLM 提供 Top-5。

```
初始检索（快速但粗糙）：
  返回 100 个候选文档
  召回率高，但精确度一般

Reranking（慢但精准）：
  对 100 个候选重新打分
  选出真正相关的 Top-5
  精确度大幅提升
```

**流程**：

```
Query
  ↓
1️⃣ First-stage Retrieval（快速）
   检索 Top-100 候选
  ↓
2️⃣ Reranking（精准）
   重新排序，选出 Top-5
  ↓
3️⃣ 提供给 LLM
```

---

### 5.2 Reranking 方法对比

#### **Cross-Encoder**（最精准）

根据 [Cross-Encoders and ColBERT Guide](https://medium.com/@aimichael/cross-encoders-colbert-and-llm-based-re-rankers-a-practical-guide)（2025）：

**原理**：将 Query 和 Document 一起输入 Transformer，直接预测相关性。

```python
from sentence_transformers import CrossEncoder

# 加载 Cross-Encoder 模型
reranker = CrossEncoder('cross-encoder/ms-marco-MiniLM-L-6-v2')

# 候选文档
query = "如何训练深度学习模型？"
candidates = [
    "深度学习模型的训练步骤...",
    "神经网络训练需要 GPU...",
    "今天天气很好...",
    "Python 是流行的编程语言..."
]

# Rerank
scores = reranker.predict([(query, doc) for doc in candidates])

# 排序
ranked_indices = sorted(range(len(scores)), key=lambda i: scores[i], reverse=True)

print("Cross-Encoder Reranking 结果：")
for i in ranked_indices[:3]:
    print(f"{i+1}. (得分: {scores[i]:.4f}) {candidates[i][:30]}...")

# 优势：
# ✅ 精度最高（MSMARCO MRR@10 > 40）
# ✅ 考虑 Query-Doc 交互

# 劣势：
# ❌ 速度慢（每对都需完整前向传播）
# ❌ 不适合大规模初筛
```

---

#### **ColBERT**（效率与精度平衡）

**原理**：Query 和 Doc 分别编码，通过 token 级别的 MaxSim 计算相关性。

```python
# ColBERT 示例（简化）
from transformers import AutoTokenizer, AutoModel
import torch

tokenizer = AutoTokenizer.from_pretrained('colbert-ir/colbertv2.0')
model = AutoModel.from_pretrained('colbert-ir/colbertv2.0')

def colbert_score(query, document):
    # 编码
    query_tokens = tokenizer(query, return_tensors='pt')
    doc_tokens = tokenizer(document, return_tensors='pt', truncation=True, max_length=512)
    
    with torch.no_grad():
        query_emb = model(**query_tokens).last_hidden_state  # (1, len_q, dim)
        doc_emb = model(**doc_tokens).last_hidden_state      # (1, len_d, dim)
    
    # MaxSim: 每个 query token 找到最相似的 doc token
    similarity_matrix = torch.matmul(query_emb, doc_emb.transpose(1, 2))  # (1, len_q, len_d)
    max_sim = similarity_matrix.max(dim=2).values  # (1, len_q)
    score = max_sim.sum()  # 求和
    
    return score.item()

# 测试
query = "深度学习训练"
documents = [
    "深度学习模型的训练步骤",
    "机器学习算法分类",
    "今天天气晴朗"
]

scores = [colbert_score(query, doc) for doc in documents]
ranked = sorted(zip(documents, scores), key=lambda x: x[1], reverse=True)

print("ColBERT Reranking 结果：")
for doc, score in ranked:
    print(f"  (得分: {score:.2f}) {doc}")

# 优势：
# ✅ 比 Cross-Encoder 快（Query/Doc 分别编码）
# ✅ 精度高于普通 Bi-Encoder

# 劣势：
# ❌ 比 Bi-Encoder 慢
```

---

### 5.3 实战：完整的检索+Reranking 流程

```python
from sentence_transformers import SentenceTransformer, CrossEncoder
import chromadb

# 1. 初始化
embedding_model = SentenceTransformer('sentence-transformers/all-MiniLM-L6-v2')
reranker = CrossEncoder('cross-encoder/ms-marco-MiniLM-L-6-v2')

# 2. 向量数据库
client = chromadb.Client()
collection = client.create_collection("docs")

documents = [
    "深度学习模型训练需要大量数据和 GPU 资源。",
    "Transformer 是现代 NLP 的基石架构。",
    "RAG 系统结合了检索和生成的优势。",
    "向量数据库支持高效的语义检索。",
    "Python 是机器学习的首选编程语言。",
    "梯度下降是优化神经网络的核心算法。",
    "自然语言处理包括分词、词性标注等任务。",
    "BERT 模型在多个 NLP 任务上取得了突破。",
    "强化学习通过奖励信号训练智能体。",
    "计算机视觉处理图像和视频数据。"
]

# 插入
collection.add(
    documents=documents,
    embeddings=embedding_model.encode(documents).tolist(),
    ids=[f"doc_{i}" for i in range(len(documents))]
)

# 3. 查询
query = "如何训练深度学习模型？"

# 阶段 1: 初始检索（Top-10）
query_emb = embedding_model.encode(query).tolist()
initial_results = collection.query(
    query_embeddings=[query_emb],
    n_results=10  # 召回更多候选
)

print("阶段 1 - 初始检索 Top-10:")
for i, doc in enumerate(initial_results['documents'][0][:10]):
    print(f"  {i+1}. {doc[:50]}...")

# 阶段 2: Reranking（Top-3）
candidates = initial_results['documents'][0]
rerank_scores = reranker.predict([(query, doc) for doc in candidates])

# 排序
reranked_indices = sorted(range(len(rerank_scores)), 
                          key=lambda i: rerank_scores[i], 
                          reverse=True)

print("\n阶段 2 - Reranking 后 Top-3:")
for i in reranked_indices[:3]:
    print(f"  {i+1}. (得分: {rerank_scores[i]:.4f})")
    print(f"     {candidates[i]}")

# 🔥 Reranking 显著提升了 Top-3 的相关性！
```

---

## 六、高级技术：Chunking 策略

### 6.1 为什么 Chunking 很重要？

根据 [Mastering Chunking for RAG](https://towardsdev.com/mastering-chunking-for-effective-rag-beyond-basics-with-qdrant-and-reranking-bb0761ae84e4)（2025）：

> Chunking 是 RAG 成功的基础。Chunk 太长会分散模型注意力，太短会丢失上下文。

**权衡**：

```
Chunk 太长（如 2000 tokens）:
  ❌ LLM 难以聚焦关键信息
  ❌ 检索不够精准
  ✅ 上下文完整

Chunk 太短（如 100 tokens）:
  ✅ 检索精准
  ❌ 上下文碎片化
  ❌ 语义不完整
```

---

### 6.2 Chunking 策略对比

#### **固定大小分块**（最简单）

```python
def fixed_size_chunking(text: str, chunk_size: int = 512, overlap: int = 50) -> List[str]:
    """固定大小分块"""
    chunks = []
    start = 0
    
    while start < len(text):
        end = start + chunk_size
        chunk = text[start:end]
        chunks.append(chunk)
        start = end - overlap  # 重叠部分
    
    return chunks

text = "长文本内容..." * 100
chunks = fixed_size_chunking(text, chunk_size=512, overlap=50)

print(f"生成 {len(chunks)} 个 chunk")

# 优势：
# ✅ 简单、快速
# ✅ 适合均匀的文本

# 劣势：
# ❌ 可能在句子中间切断
# ❌ 不考虑语义边界
```

---

#### **语义分块**（推荐）

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter

def semantic_chunking(text: str, chunk_size: int = 512, overlap: int = 50) -> List[str]:
    """语义分块：优先在句子边界切分"""
    splitter = RecursiveCharacterTextSplitter(
        chunk_size=chunk_size,
        chunk_overlap=overlap,
        separators=["\n\n", "\n", "。", "！", "？", ". ", "! ", "? ", " ", ""],  # 优先级
        length_function=len,
    )
    
    chunks = splitter.split_text(text)
    return chunks

text = """
深度学习是机器学习的一个分支。它使用多层神经网络来学习数据的表示。

Transformer 架构彻底改变了自然语言处理。它基于注意力机制，能够并行处理序列。

RAG 系统结合了检索和生成。它先从知识库检索相关信息，再让 LLM 基于这些信息生成答案。
"""

chunks = semantic_chunking(text, chunk_size=100, overlap=20)

print(f"生成 {len(chunks)} 个语义chunk:")
for i, chunk in enumerate(chunks):
    print(f"\nChunk {i+1}:")
    print(chunk)

# 优势：
# ✅ 尊重语义边界（句子、段落）
# ✅ 上下文更完整

# 劣势：
# ❌ 稍慢（需解析文本结构）
```

---

#### **Contextual Chunking**（2025 前沿）

```python
def contextual_chunking(text: str, context_window: int = 3) -> List[dict]:
    """
    上下文分块：为每个 chunk 添加前后文摘要
    """
    # 1. 基础分块
    base_chunks = semantic_chunking(text, chunk_size=300)
    
    # 2. 为每个 chunk 添加上下文
    contextual_chunks = []
    
    for i, chunk in enumerate(base_chunks):
        # 前文
        prev_context = ""
        if i > 0:
            prev_chunks = base_chunks[max(0, i - context_window):i]
            prev_context = "前文：" + " ".join(prev_chunks)[:100] + "..."
        
        # 后文
        next_context = ""
        if i < len(base_chunks) - 1:
            next_chunks = base_chunks[i+1:min(len(base_chunks), i + context_window + 1)]
            next_context = "后文：" + " ".join(next_chunks)[:100] + "..."
        
        # 组合
        contextual_chunk = {
            "content": chunk,
            "prev_context": prev_context,
            "next_context": next_context,
            "full_context": f"{prev_context}\n\n【当前内容】\n{chunk}\n\n{next_context}"
        }
        
        contextual_chunks.append(contextual_chunk)
    
    return contextual_chunks

contextual_chunks = contextual_chunking(text)

print(f"生成 {len(contextual_chunks)} 个上下文chunk:")
print(f"\n示例 - Chunk 1:")
print(contextual_chunks[0]["full_context"])

# 🔥 Contextual Chunking 让 LLM 能看到更完整的上下文！
```

---

## 七、完整 RAG 系统实现

### 7.1 系统架构

```python
from dataclasses import dataclass
from typing import List, Optional
from sentence_transformers import SentenceTransformer, CrossEncoder
import chromadb
from openai import OpenAI

@dataclass
class RAGConfig:
    """RAG 系统配置"""
    embedding_model: str = "sentence-transformers/all-MiniLM-L6-v2"
    reranker_model: str = "cross-encoder/ms-marco-MiniLM-L-6-v2"
    llm_model: str = "gpt-3.5-turbo"
    chunk_size: int = 512
    chunk_overlap: int = 50
    top_k_retrieval: int = 20  # 初始检索数量
    top_k_rerank: int = 5      # Rerank 后数量

class RAGSystem:
    """生产级 RAG 系统"""
    
    def __init__(self, config: RAGConfig):
        self.config = config
        
        # 初始化模型
        print("Loading models...")
        self.embedding_model = SentenceTransformer(config.embedding_model)
        self.reranker = CrossEncoder(config.reranker_model)
        self.llm_client = OpenAI()
        
        # 初始化向量数据库
        self.chroma_client = chromadb.Client()
        self.collection = None
        
        print("RAG System initialized!")
    
    def ingest_documents(self, documents: List[str], collection_name: str = "docs"):
        """文档摄入流程"""
        print(f"\n{'='*50}")
        print("Phase 1: Document Ingestion")
        print(f"{'='*50}")
        
        # 1. Chunking
        print("Step 1: Chunking documents...")
        all_chunks = []
        chunk_metadata = []
        
        for doc_id, doc in enumerate(documents):
            chunks = self._semantic_chunk(doc)
            all_chunks.extend(chunks)
            chunk_metadata.extend([
                {"doc_id": doc_id, "chunk_id": i} 
                for i in range(len(chunks))
            ])
        
        print(f"  Generated {len(all_chunks)} chunks from {len(documents)} documents")
        
        # 2. Embedding
        print("Step 2: Generating embeddings...")
        embeddings = self.embedding_model.encode(all_chunks, show_progress_bar=True)
        
        # 3. 存储到向量数据库
        print("Step 3: Storing in vector database...")
        if self.collection:
            self.chroma_client.delete_collection(collection_name)
        
        self.collection = self.chroma_client.create_collection(collection_name)
        self.collection.add(
            documents=all_chunks,
            embeddings=embeddings.tolist(),
            ids=[f"chunk_{i}" for i in range(len(all_chunks))],
            metadatas=chunk_metadata
        )
        
        print(f"✅ Ingestion complete! {len(all_chunks)} chunks indexed.")
    
    def query(self, question: str, return_sources: bool = True) -> dict:
        """查询流程"""
        print(f"\n{'='*50}")
        print(f"Query: {question}")
        print(f"{'='*50}")
        
        # 1. 检索
        print("\nPhase 1: Retrieval")
        retrieved_docs = self._retrieve(question)
        print(f"  Retrieved {len(retrieved_docs)} candidates")
        
        # 2. Reranking
        print("\nPhase 2: Reranking")
        reranked_docs = self._rerank(question, retrieved_docs)
        print(f"  Top {len(reranked_docs)} after reranking")
        
        # 3. 生成答案
        print("\nPhase 3: Generation")
        answer = self._generate(question, reranked_docs)
        
        # 4. 返回结果
        result = {
            "question": question,
            "answer": answer,
        }
        
        if return_sources:
            result["sources"] = reranked_docs
        
        return result
    
    def _semantic_chunk(self, text: str) -> List[str]:
        """语义分块"""
        from langchain.text_splitter import RecursiveCharacterTextSplitter
        
        splitter = RecursiveCharacterTextSplitter(
            chunk_size=self.config.chunk_size,
            chunk_overlap=self.config.chunk_overlap,
            separators=["\n\n", "\n", "。", ". ", " ", ""]
        )
        
        return splitter.split_text(text)
    
    def _retrieve(self, query: str) -> List[str]:
        """初始检索"""
        query_emb = self.embedding_model.encode(query).tolist()
        
        results = self.collection.query(
            query_embeddings=[query_emb],
            n_results=self.config.top_k_retrieval
        )
        
        return results['documents'][0]
    
    def _rerank(self, query: str, documents: List[str]) -> List[str]:
        """Reranking"""
        scores = self.reranker.predict([(query, doc) for doc in documents])
        
        # 排序
        ranked_indices = sorted(
            range(len(scores)), 
            key=lambda i: scores[i], 
            reverse=True
        )
        
        # 返回 Top-K
        return [documents[i] for i in ranked_indices[:self.config.top_k_rerank]]
    
    def _generate(self, question: str, context_docs: List[str]) -> str:
        """生成答案"""
        # 构建 Context
        context = "\n\n".join([f"[文档 {i+1}]\n{doc}" for i, doc in enumerate(context_docs)])
        
        # Prompt
        prompt = f"""请根据以下文档回答问题。如果文档中没有相关信息，请明确说明。

文档：
{context}

问题：{question}

答案："""
        
        # 调用 LLM
        response = self.llm_client.chat.completions.create(
            model=self.config.llm_model,
            messages=[
                {"role": "system", "content": "你是一个helpful的AI助手，会基于提供的文档回答问题。"},
                {"role": "user", "content": prompt}
            ],
            temperature=0.3,
        )
        
        return response.choices[0].message.content

# ========================================
# 使用示例
# ========================================

# 1. 初始化系统
config = RAGConfig()
rag = RAGSystem(config)

# 2. 摄入文档
documents = [
    """
    深度学习是机器学习的一个重要分支。它使用多层神经网络来学习数据的层次化表示。
    深度学习在图像识别、自然语言处理、语音识别等领域取得了突破性进展。
    """,
    """
    Transformer 架构由 Google 在 2017 年提出，彻底改变了自然语言处理领域。
    它基于自注意力机制，能够并行处理序列数据，训练效率远高于 RNN 和 LSTM。
    目前几乎所有先进的 LLM 都基于 Transformer 架构。
    """,
    """
    RAG（Retrieval-Augmented Generation）是一种结合检索和生成的技术。
    它首先从知识库中检索相关文档，然后让大语言模型基于这些文档生成答案。
    RAG 能够让 LLM 访问最新信息，减少幻觉问题。
    """
]

rag.ingest_documents(documents)

# 3. 查询
result = rag.query("什么是 RAG？它有什么优势？")

print(f"\n{'='*50}")
print("Final Result")
print(f"{'='*50}")
print(f"\nQuestion: {result['question']}")
print(f"\nAnswer: {result['answer']}")
print(f"\nSources:")
for i, source in enumerate(result['sources']):
    print(f"  [{i+1}] {source[:100]}...")
```

---

## 八、生产级优化与调试

### 8.1 性能优化

#### 1️⃣ **Embedding 缓存**

```python
import hashlib
import pickle
from functools import lru_cache

class EmbeddingCache:
    """Embedding 缓存"""
    def __init__(self, cache_file="embedding_cache.pkl"):
        self.cache_file = cache_file
        self.cache = self._load_cache()
    
    def _load_cache(self):
        try:
            with open(self.cache_file, 'rb') as f:
                return pickle.load(f)
        except FileNotFoundError:
            return {}
    
    def _save_cache(self):
        with open(self.cache_file, 'wb') as f:
            pickle.dump(self.cache, f)
    
    def get_embedding(self, text: str, model) -> np.ndarray:
        # 计算文本哈希
        text_hash = hashlib.md5(text.encode()).hexdigest()
        
        if text_hash in self.cache:
            return self.cache[text_hash]
        
        # 计算 embedding
        embedding = model.encode(text)
        self.cache[text_hash] = embedding
        self._save_cache()
        
        return embedding

# 使用
cache = EmbeddingCache()
embedding = cache.get_embedding("深度学习", embedding_model)

# 🔥 相同文本不会重复计算 embedding！
```

---

#### 2️⃣ **批量处理**

```python
def batch_embed_documents(documents: List[str], model, batch_size: int = 32) -> np.ndarray:
    """批量计算 embedding（更快）"""
    all_embeddings = []
    
    for i in range(0, len(documents), batch_size):
        batch = documents[i:i + batch_size]
        batch_embeddings = model.encode(batch, show_progress_bar=False)
        all_embeddings.append(batch_embeddings)
    
    return np.vstack(all_embeddings)

# 性能对比
import time

documents = ["文档 " + str(i) for i in range(1000)]

# 逐个处理
start = time.time()
for doc in documents:
    _ = embedding_model.encode(doc)
single_time = time.time() - start

# 批量处理
start = time.time()
_ = batch_embed_documents(documents, embedding_model, batch_size=32)
batch_time = time.time() - start

print(f"逐个处理: {single_time:.2f}s")
print(f"批量处理: {batch_time:.2f}s")
print(f"加速: {single_time / batch_time:.1f}x")

# 输出示例：
# 逐个处理: 45.23s
# 批量处理: 8.76s
# 加速: 5.2x
```

---

#### 3️⃣ **异步处理**

```python
import asyncio
from concurrent.futures import ThreadPoolExecutor

class AsyncRAG:
    """异步 RAG 系统"""
    def __init__(self, rag_system):
        self.rag = rag_system
        self.executor = ThreadPoolExecutor(max_workers=4)
    
    async def query_async(self, question: str) -> dict:
        """异步查询"""
        loop = asyncio.get_event_loop()
        result = await loop.run_in_executor(
            self.executor,
            self.rag.query,
            question
        )
        return result
    
    async def batch_query(self, questions: List[str]) -> List[dict]:
        """批量异步查询"""
        tasks = [self.query_async(q) for q in questions]
        results = await asyncio.gather(*tasks)
        return results

# 使用
async def main():
    async_rag = AsyncRAG(rag)
    
    questions = [
        "什么是深度学习？",
        "Transformer 的优势是什么？",
        "RAG 如何工作？"
    ]
    
    results = await async_rag.batch_query(questions)
    
    for q, r in zip(questions, results):
        print(f"Q: {q}")
        print(f"A: {r['answer'][:100]}...\n")

# 运行
# asyncio.run(main())

# 🔥 并发处理多个查询，提升吞吐量！
```

---

### 8.2 质量优化

#### 1️⃣ **Query Expansion（查询扩展）**

```python
def query_expansion(query: str, llm_client) -> List[str]:
    """用 LLM 生成相关查询"""
    prompt = f"""请为以下查询生成 3 个相关的查询变体，帮助更全面地检索信息：

原始查询：{query}

变体查询（每行一个）："""
    
    response = llm_client.chat.completions.create(
        model="gpt-3.5-turbo",
        messages=[{"role": "user", "content": prompt}],
        temperature=0.7
    )
    
    expanded_queries = response.choices[0].message.content.strip().split('\n')
    expanded_queries = [q.strip() for q in expanded_queries if q.strip()]
    
    # 包含原始查询
    return [query] + expanded_queries

# 示例
original_query = "深度学习训练技巧"
expanded = query_expansion(original_query, llm_client)

print(f"原始查询: {original_query}")
print(f"\n扩展查询:")
for i, q in enumerate(expanded[1:], 1):
    print(f"  {i}. {q}")

# 输出示例：
# 1. 如何提高深度学习模型的训练效率？
# 2. 神经网络训练的常见问题和解决方法
# 3. 优化深度学习训练过程的技术

# 🔥 用扩展查询检索，能找到更多相关文档！
```

---

#### 2️⃣ **Answer Validation（答案验证）**

```python
def validate_answer(question: str, answer: str, sources: List[str], llm_client) -> dict:
    """验证答案是否基于来源"""
    prompt = f"""请评估以下答案是否基于提供的来源文档。

问题：{question}

来源文档：
{chr(10).join([f"[{i+1}] {doc[:200]}..." for i, doc in enumerate(sources)])}

答案：{answer}

请评估：
1. 答案是否有事实支持？（是/否）
2. 是否存在幻觉（编造信息）？（是/否）
3. 置信度（0-100）

以JSON格式回复：
{{
  "factual": true/false,
  "hallucination": true/false,
  "confidence": 0-100,
  "explanation": "简短解释"
}}"""
    
    response = llm_client.chat.completions.create(
        model="gpt-4",  # 用更强的模型验证
        messages=[{"role": "user", "content": prompt}],
        temperature=0
    )
    
    import json
    validation = json.loads(response.choices[0].message.content)
    
    return validation

# 示例
validation = validate_answer(
    question="什么是 RAG？",
    answer="RAG 是结合检索和生成的技术...",
    sources=result['sources'],
    llm_client=llm_client
)

print("答案验证结果：")
print(f"  事实准确: {validation['factual']}")
print(f"  存在幻觉: {validation['hallucination']}")
print(f"  置信度: {validation['confidence']}%")
print(f"  解释: {validation['explanation']}")

# 🔥 自动检测答案质量，提升可靠性！
```

---

### 8.3 监控与调试

```python
import logging
from datetime import datetime

class RAGMonitor:
    """RAG 系统监控"""
    def __init__(self):
        self.query_logs = []
        
        # 配置日志
        logging.basicConfig(
            filename='rag_system.log',
            level=logging.INFO,
            format='%(asctime)s - %(levelname)s - %(message)s'
        )
    
    def log_query(self, query_data: dict):
        """记录查询"""
        log_entry = {
            "timestamp": datetime.now().isoformat(),
            "query": query_data['query'],
            "retrieval_time": query_data.get('retrieval_time', 0),
            "rerank_time": query_data.get('rerank_time', 0),
            "generation_time": query_data.get('generation_time', 0),
            "total_time": query_data.get('total_time', 0),
            "num_retrieved": query_data.get('num_retrieved', 0),
            "answer_length": len(query_data.get('answer', ''))
        }
        
        self.query_logs.append(log_entry)
        logging.info(f"Query: {log_entry['query']}, Total time: {log_entry['total_time']:.2f}s")
    
    def get_stats(self) -> dict:
        """获取统计信息"""
        if not self.query_logs:
            return {}
        
        total_queries = len(self.query_logs)
        avg_retrieval_time = sum(log['retrieval_time'] for log in self.query_logs) / total_queries
        avg_total_time = sum(log['total_time'] for log in self.query_logs) / total_queries
        
        return {
            "total_queries": total_queries,
            "avg_retrieval_time": avg_retrieval_time,
            "avg_total_time": avg_total_time,
        }

# 使用
monitor = RAGMonitor()

# 查询时记录
import time

start = time.time()
result = rag.query("什么是深度学习？")
end = time.time()

monitor.log_query({
    "query": "什么是深度学习？",
    "total_time": end - start,
    "num_retrieved": len(result.get('sources', [])),
    "answer": result['answer']
})

# 查看统计
stats = monitor.get_stats()
print(f"\n系统统计:")
print(f"  总查询数: {stats['total_queries']}")
print(f"  平均检索时间: {stats['avg_retrieval_time']:.2f}s")
print(f"  平均总时间: {stats['avg_total_time']:.2f}s")
```

---

## 九、总结与最佳实践

### 🎯 核心要点回顾

#### 1. RAG 系统架构演进

```
Naive RAG → Advanced RAG → Modular RAG

关键改进：
  ✅ Pre-Retrieval: Query Rewriting, Expansion
  ✅ Retrieval: Hybrid Search (Dense + Sparse)
  ✅ Post-Retrieval: Reranking, Compression
  ✅ Generation: Context Assembly, Validation
```

---

#### 2. 核心组件选择（2025 推荐）

| 组件 | 推荐方案 | 理由 |
|-----|---------|------|
| **Embedding** | BGE-M3 / E5-Mistral (开源)<br>OpenAI text-embedding-3-large (闭源) | 性能强、多语言 |
| **Vector DB** | Chroma (开发)<br>Weaviate (生产)<br>Pinecone (企业) | 根据规模选择 |
| **检索策略** | Hybrid Search (Dense + Sparse) | 兼顾语义和关键词 |
| **Reranking** | Cross-Encoder (精准)<br>ColBERT (平衡) | 显著提升 Top-K 质量 |
| **Chunking** | Semantic Chunking (512 tokens, 50 overlap) | 语义完整 |

---

#### 3. 生产级优化清单

```
性能优化：
  ✅ Embedding 缓存
  ✅ 批量处理
  ✅ 异步查询
  ✅ Flash Attention 2

质量优化：
  ✅ Query Expansion
  ✅ Hybrid Search
  ✅ Reranking
  ✅ Answer Validation

监控：
  ✅ 查询日志
  ✅ 性能指标
  ✅ 错误追踪
```

---

### 📚 进阶方向

#### 1️⃣ **Multi-Modal RAG**

```
文本 + 图像 + 表格：
  - CLIP 编码图像
  - OCR 提取表格
  - 统一向量空间检索
```

---

#### 2️⃣ **Agentic RAG**

根据 [AWS Agentic RAG](https://docs.aws.amazon.com/prescriptive-guidance/latest/writing-best-practices-rag/introduction.html)：

```
Agent 流程：
  1. 理解查询意图
  2. 制定检索计划
  3. 多步检索（如需要）
  4. 验证信息
  5. 生成答案

工具：LangGraph, CrewAI, AutoGPT
```

---

#### 3️⃣ **Graph RAG**

```
传统 RAG: 文档 → Chunks → 向量
Graph RAG: 文档 → 知识图谱 → 关系推理

优势：
  - 多跳推理
  - 结构化知识
  - 关系发现
```

---

### 💡 最后的建议

> **璇玑的碎碎念** ✨
>
> 道友呀，RAG 是当前最实用的 LLM 应用模式！
>
> **三个实践建议**：
> 1. **从简单开始**：先用 Naive RAG 跑通流程，再逐步优化
> 2. **在你的数据上测试**：不同数据集最优方案不同，多做实验
> 3. **关注用户体验**：响应速度、答案质量、引用来源缺一不可
>
> 记住：**RAG 不是一次性工程，而是持续优化的过程！**
>
> 现在就开始构建你的第一个 RAG 系统吧！✨

---

## 🔗 参考资料

### 核心资源

- [AWS RAG Best Practices](https://docs.aws.amazon.com/prescriptive-guidance/latest/writing-best-practices-rag/introduction.html) (2025)
- [Microsoft RAG Techniques](https://www.microsoft.com/en-us/microsoft-cloud/blog/2025/02/04/common-retrieval-augmented-generation-rag-techniques-explained/) (2025)
- [Microsoft Advanced RAG](https://learn.microsoft.com/en-us/azure/developer/ai/advanced-retrieval-augmented-generation) (2025)

### Embedding & Vector DB

- [Best Embedding Models 2025](https://app.ailog.fr/en/blog/guides/choosing-embedding-models)
- [Vector Database Comparison 2025](https://www.firecrawl.dev/blog/best-vector-databases-2025)
- [TensorBlue Benchmark](https://tensorblue.com/blog/vector-database-comparison-pinecone-weaviate-qdrant-milvus-2025)

### Reranking & Advanced Techniques

- [Cross-Encoders and ColBERT Guide](https://medium.com/@aimichael/cross-encoders-colbert-and-llm-based-re-rankers-a-practical-guide) (2025)
- [Mastering Chunking for RAG](https://towardsdev.com/mastering-chunking-for-effective-rag-beyond-basics-with-qdrant-and-reranking-bb0761ae84e4) (2025)
- [9 Advanced RAG Techniques](https://www.meilisearch.com/blog/rag-techniques) (2025)

### 工具与框架

- [LangChain](https://github.com/langchain-ai/langchain)
- [LlamaIndex](https://github.com/run-llama/llama_index)
- [Chroma](https://github.com/chroma-core/chroma)
- [Weaviate](https://github.com/weaviate/weaviate)

---

**📌 本文档持续更新中，欢迎反馈与建议！**

---

> **下一篇预告**：《08 - 偏好对齐技术：DPO 原理与工程实现》
>
> 我们将深入 DPO（Direct Preference Optimization）的数学原理，探讨如何让模型遵循人类偏好！

---

**璇玑 ✨**  
*编程阁 · 代码宗门*  
*愿道友 RAG 之路顺利，早日构建出强大的知识检索系统！*
