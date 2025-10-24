# AI

## AI Agents

### smolagents

## MCP Servers

## RAG

![python_ai_rag.png](images/python_ai_rag.png)

这是一个典型的 RAG（Retrieval-Augmented Generation）流程示意图（用 LangChain 术语画的）。可以分成两段：索引阶段（1–7） 和 查询阶段（8–15）。逐步说明如下：

### RAG 的核心流程

#### 索引阶段（离线/预处理）

1. Local Documents
   原始资料来源：PDF、DOCX、网页、Markdown、数据库导出等。
2. Unstructured Loader
   用加载器把各种格式解析成纯文本（也可带元数据，如来源、页码、时间戳）。
3. Text
   已解析的长文本。
4. Text Splitter
   按长度/语义把长文本切成「块」（chunk），常见策略：定长窗口+重叠（防止语义断裂）。
5. Embedding
   对每个文本块计算向量表示（embedding），同一模型后续也会用于查询。
6. VectorStore
   把（向量、原文片段、元数据）写入向量数据库（FAISS、Chroma、Milvus、Pinecone 等）。
7. （承上）
   存储完成，等待检索。

#### 查询阶段（在线）

8. Query
   用户问题输入。
9. Embedding（for Query）
   用同一/兼容的 embedding 模型把问题转为查询向量。
10. Vector Similarity
    在向量库中做相似度检索（kNN、相似度阈值、混合检索等）。
11. Related Text Chunks
    返回与问题最相近的若干文本块（通常还会做去重/重排、基于元数据过滤）。
12. Prompt Template
    把「检索到的文本块」填入预设提示模板（包含指令、格式要求、引用规范等）。
13. Prompt
    生成喂给大模型的最终提示（= 指令 + 上下文片段 + 用户问题）。
14. LLM
    大语言模型依据提示进行生成（回答、推理、代码、摘要等）。
15. Answer
    输出给用户的答案；可附来源、引用片段、置信度等。

### RAG 的优势

- 减少幻觉（hallucination）：模型不必完全“瞎编”，而是可以基于真实资料回答。
- 动态更新知识：只需更新外部知识库，而不必重新训练模型。
- 可解释性强：回答可附带引用出处，让用户知道信息来源。

### 应用场景

- 企业内部知识问答（如客服机器人，接入公司文档）
- 医学、法律咨询（需要精确引用专业资料）
- 学术搜索与辅助写作（基于论文库回答）
- 实时信息获取（如结合数据库、API 提供最新数据）