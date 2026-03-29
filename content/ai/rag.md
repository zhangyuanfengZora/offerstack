---
title: RAG 原理与实践
weight: 1
---

## 什么是 RAG？

**RAG（Retrieval-Augmented Generation）** 检索增强生成，是解决 LLM 知识局限性的核心技术。

```
用户问题
    ↓
Embedding 向量化
    ↓
向量数据库检索（Top-K 相关文档）
    ↓
将检索结果 + 原始问题组合为 Prompt
    ↓
LLM 生成回答
```

## Q: RAG 的核心组件有哪些？

| 组件 | 说明 | 常用工具 |
|------|------|---------|
| 文档加载 | 解析 PDF/Word/网页等 | LangChain Loaders |
| 文档切分 | 按语义/大小切分 Chunk | RecursiveCharacterSplitter |
| Embedding | 将文本转为向量 | OpenAI/BGE/M3E |
| 向量数据库 | 存储和检索向量 | Milvus/Qdrant/Chroma |
| LLM | 生成最终回答 | GPT/Claude/Llama |

## Q: Chunk 切分策略有哪些？

```python
# 1. 固定大小切分
text_splitter = CharacterTextSplitter(
    chunk_size=500,
    chunk_overlap=50  # 重叠部分保证上下文连贯
)

# 2. 递归字符切分（推荐）
text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=500,
    chunk_overlap=50,
    separators=["\n\n", "\n", "。", "！", "？", " ", ""]
)

# 3. 语义切分（更智能）
# 基于语义相似度进行切分
```

## Q: 如何评估 RAG 的效果？

| 指标 | 说明 |
|------|------|
| **Faithfulness** | 回答是否忠实于检索内容 |
| **Answer Relevancy** | 回答是否切题 |
| **Context Precision** | 检索内容是否精准 |
| **Context Recall** | 相关内容是否被检索到 |

常用评估框架：**RAGAS**

## Q: RAG 的常见优化手段？

1. **查询重写**：将用户问题重写为更适合检索的形式
2. **HyDE**：先让 LLM 生成假设答案，再用答案检索
3. **重排序（Rerank）**：用 Cross-Encoder 对检索结果重新排序
4. **多路召回**：同时使用向量检索 + 关键词检索（BM25）
5. **父子文档检索**：小 chunk 检索，大 chunk 送给 LLM

{{% callout type="info" %}}
**面试 Tips**：如果你的项目用了 RAG，一定要能说清楚 Chunk 大小的选择依据、Embedding 模型的选型理由、以及如何解决幻觉问题。
{{% /callout %}}
