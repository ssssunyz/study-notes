# LangChain 和 LangGraph 完整指南

> 本文档涵盖 LangChain 和 LangGraph 的原理、使用案例、代码示例（每段代码都有简洁版和详细注释版）、FAISS 详解、Embedding API vs LLM API 的区分，以及 10 道高频面试题（英文）。

---

## 目录

- [一、Term 和原理的解释](#一term-和原理的解释)
- [二、实际运用案例](#二实际运用案例)
- [三、Code Examples](#三code-examples)
  - [3.1 LangChain 基础：用 LCEL 写一个简单 Chain](#31-langchain-基础用-lcel-写一个简单-chain)
  - [3.2 LangChain RAG（Retrieval-Augmented Generation）](#32-langchain-ragretrieval-augmented-generation)
  - [3.3 LangChain 加 Memory（对话记忆）](#33-langchain-加-memory对话记忆)
  - [3.4 LangGraph 基础：StateGraph](#34-langgraph-基础stategraph)
  - [3.5 LangGraph 实现 ReAct Agent（带 Tools 和循环）](#35-langgraph-实现-react-agent带-tools-和循环)
  - [3.6 LangGraph 加 Checkpointer（持久化 + Human-in-the-loop）](#36-langgraph-加-checkpointer持久化--human-in-the-loop)
- [四、补充：FAISS 详解](#四补充faiss-详解)
- [五、补充：Embedding API vs LLM API](#五补充embedding-api-vs-llm-api)
- [六、Top 10 Interview Questions for AI Engineers (English)](#六top-10-interview-questions-for-ai-engineers-english)

---

## 一、Term 和原理的解释

### 1.1 为什么会出现 LangChain？

在 ChatGPT 爆火之后（2022年底），开发者们发现一个问题：**直接调用 LLM 的 API 只能解决简单问题**。如果想构建真正有用的 AI 应用，你会遇到很多麻烦：

- **Prompt 管理混乱**：prompt 散落在代码各处，难以维护和复用
- **LLM 是无状态的（stateless）**：每次对话不会记得上一次说了什么
- **LLM 的知识有截止日期**：无法回答你公司内部文档的问题
- **LLM 不能执行操作**：它只会生成文本，不能查数据库、调用 API
- **不同 LLM 提供商接口不统一**：OpenAI、Anthropic、Google 的 API 各不相同
- **多步骤任务难以组织**：比如「先翻译，再总结，再生成邮件」这种流程

**LangChain** 由 Harrison Chase 在 2022 年 10 月创建，本质上是一个 **LLM 应用开发框架（framework）**，它通过提供一系列抽象（abstractions）来解决上面这些问题。

### 1.2 LangChain 的核心 Components

| Component | 作用 | 类比 |
|---|---|---|
| **Models** | 统一不同 LLM 的接口（OpenAI、Anthropic 等） | 数据库的 ORM |
| **Prompts** | Prompt 模板化、参数化 | HTML 模板引擎 |
| **Chains** | 把多个步骤串联起来 | Unix pipeline |
| **Memory** | 让 LLM 记住对话历史 | Session/Cookie |
| **Retrievers** | 从外部数据源检索信息 | 搜索引擎 |
| **Tools** | 让 LLM 调用外部函数 | Function call |
| **Agents** | LLM 自主决定调用哪些 tools | 自动驾驶 |
| **Document Loaders** | 加载 PDF、网页、数据库等数据 | ETL |
| **Text Splitters** | 把长文档切分成小块 | 切菜板 |
| **Vector Stores** | 存储向量化的文档 | 专用数据库 |

### 1.3 LCEL（LangChain Expression Language）

这是 LangChain 后期引入的**新写法**，用 `|`（pipe 操作符）来组合组件，灵感来自 Unix pipeline 和函数式编程：

```python
chain = prompt | llm | output_parser
```

LCEL 出现的原因是早期的 `LLMChain`、`SequentialChain` 等类太重、太死板，不灵活。LCEL 让组合变得简洁，并且自动支持 **streaming、async、batch、并行执行**。

### 1.4 为什么又出现了 LangGraph？

LangChain 用着用着，大家发现一个根本性问题：**LangChain 的 Chain 本质是 DAG（有向无环图），不能循环**。

但是 **Agent 天然就需要循环**：
- LLM 想一想 → 调用工具 → 看结果 → 再想一想 → 再调用工具 → ... → 得出答案

LangChain 早期的 `AgentExecutor` 内部用了一个 while 循环硬实现了这个能力，但是有很多痛点：

1. **控制流不透明（black box）**：你无法精细控制 agent 每一步的行为
2. **难以调试**：agent 卡住了你不知道为什么
3. **难以加自定义逻辑**：比如「调用工具 3 次失败就停止」这种规则很难加
4. **状态管理混乱**：中间状态散落在各处
5. **不支持 human-in-the-loop**：人工审核某一步无法实现
6. **多 agent 协作困难**：让多个 agent 互相对话很难写
7. **持久化（persistence）差**：agent 跑到一半挂了无法续跑

**LangGraph** 在 2024 年初推出，专门解决这些问题。它的核心思想是：**把 LLM 应用建模成一个状态机（state machine）/ 图（graph）**。

### 1.5 LangGraph 的核心概念

| 概念 | 解释 |
|---|---|
| **State** | 全局共享的状态字典，所有节点都能读写 |
| **Node** | 一个函数，接收 state，返回对 state 的更新 |
| **Edge** | 节点之间的连接，决定下一步去哪里 |
| **Conditional Edge** | 根据 state 的内容动态决定下一步 |
| **StateGraph** | 整个图的容器 |
| **Checkpointer** | 状态持久化机制，支持中断和恢复 |
| **Reducer** | 定义 state 字段如何被更新（比如 append 而不是覆盖） |

**关键区别**：LangGraph 显式支持**循环**和**条件分支**，让你像写流程图一样写 agent。

### 1.6 LangChain vs LangGraph 的关系

很多人搞混它们的关系。简单说：

- **LangChain** 提供基础组件（LLM 接口、prompt、tools、retriever 等）
- **LangGraph** 提供**编排（orchestration）层**，用来组织 LangChain 的组件实现复杂流程
- LangGraph **可以独立使用**，但通常和 LangChain 一起用
- 现在的官方推荐：**简单线性流程用 LCEL，复杂 agent 用 LangGraph**

---

## 二、实际运用案例

### 案例 1：企业内部知识库问答（RAG）
比如 Notion AI、内部 wiki 助手。用户问「我们公司的报销政策是什么」，系统从公司文档库里检索相关内容，再让 LLM 基于这些内容回答。**主要用 LangChain。**

### 案例 2：客服 Chatbot
带记忆的多轮对话，能查订单、查物流、退款。需要调用多个 API。**LangChain（基础）+ LangGraph（复杂流程）。**

### 案例 3：代码助手 / Coding Agent
比如类似 Cursor、Cline 的工具。Agent 需要：读文件 → 思考 → 写代码 → 运行测试 → 看结果 → 修改 → 再测试，**典型的循环结构，必须用 LangGraph**。

### 案例 4：研究报告自动生成
输入一个主题，agent 自动：搜索网页 → 阅读结果 → 提取要点 → 综合 → 写报告 → 自检 → 修改。**LangGraph 的典型场景。**

### 案例 5：多 Agent 系统（Multi-Agent System）
比如「产品经理 agent + 工程师 agent + 测试 agent」相互协作。**只有 LangGraph 能优雅实现。**

### 案例 6：审批工作流（Human-in-the-loop）
比如 AI 起草合同 → 人工审核 → AI 修改 → 人工最终确认。**LangGraph 的 checkpoint 机制是核心。**

---

## 三、Code Examples

> **阅读建议**：如果你不熟悉 FAISS，可以先跳到 [第四章](#四补充faiss-详解) 了解 FAISS 是什么。如果你不清楚 embedding API 和 LLM API 的区别，可以先读 [第五章](#五补充embedding-api-vs-llm-api)。

### 3.1 LangChain 基础：用 LCEL 写一个简单 Chain

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

# 1. 定义 prompt 模板
prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个专业翻译，把用户输入翻译成{language}。"),
    ("user", "{text}")
])

# 2. 定义 LLM
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

# 3. 输出解析器（把 AIMessage 转成 string）
parser = StrOutputParser()

# 4. 用 LCEL 的 | 组合起来
chain = prompt | llm | parser

# 5. 调用
result = chain.invoke({"language": "法语", "text": "今天天气真好"})
print(result)  # → "Il fait beau aujourd'hui"

# 也支持 streaming
for chunk in chain.stream({"language": "法语", "text": "今天天气真好"}):
    print(chunk, end="", flush=True)
```

---

### 3.2 LangChain RAG（Retrieval-Augmented Generation）

#### 3.2.1 简洁版

```python
from langchain_community.document_loaders import PyPDFLoader
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langchain_openai import OpenAIEmbeddings, ChatOpenAI
from langchain_community.vectorstores import FAISS
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.runnables import RunnablePassthrough
from langchain_core.output_parsers import StrOutputParser

# 1. 加载文档
loader = PyPDFLoader("company_handbook.pdf")
docs = loader.load()

# 2. 切分文档
splitter = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=200)
chunks = splitter.split_documents(docs)

# 3. 向量化并存入 vector store
embeddings = OpenAIEmbeddings()
vectorstore = FAISS.from_documents(chunks, embeddings)
retriever = vectorstore.as_retriever(search_kwargs={"k": 4})

# 4. 定义 prompt
prompt = ChatPromptTemplate.from_template("""
基于以下 context 回答问题。如果答案不在 context 中，说"我不知道"。

Context: {context}

Question: {question}
""")

# 5. 组合 RAG chain
llm = ChatOpenAI(model="gpt-4o-mini")

def format_docs(docs):
    return "\n\n".join(d.page_content for d in docs)

rag_chain = (
    {"context": retriever | format_docs, "question": RunnablePassthrough()}
    | prompt
    | llm
    | StrOutputParser()
)

# 6. 提问
answer = rag_chain.invoke("公司的年假政策是怎样的？")
print(answer)
```

#### 3.2.2 详细注释版

```python
# ============================================================
# 第 0 步：导入所有需要的模块
# ============================================================

# DocumentLoader：负责把原始文件（PDF/HTML/CSV...）读成 LangChain 的 Document 对象
# PyPDFLoader 专门用于读 PDF，底层用 pypdf 库
from langchain_community.document_loaders import PyPDFLoader

# TextSplitter：把长文档切分成小 chunk。
# RecursiveCharacterTextSplitter 是最常用的：它会尝试按 ["\n\n", "\n", " ", ""] 
# 的顺序递归切分，尽量保持段落、句子的完整性
from langchain_text_splitters import RecursiveCharacterTextSplitter

# Embeddings：把文本转成向量的模型
# ChatOpenAI：调用 OpenAI 的 chat completion API
from langchain_openai import OpenAIEmbeddings, ChatOpenAI

# FAISS：向量数据库（vector store），用于存向量并做相似度检索
from langchain_community.vectorstores import FAISS

# ChatPromptTemplate：构造可参数化的 prompt 模板
from langchain_core.prompts import ChatPromptTemplate

# RunnablePassthrough：LCEL 中的"原样传递"组件，
# 类似于函数式编程里的 identity function，把输入原封不动地传出去
from langchain_core.runnables import RunnablePassthrough

# StrOutputParser：把 LLM 返回的 AIMessage 对象解析成纯 string
from langchain_core.output_parsers import StrOutputParser


# ============================================================
# 第 1 步：加载文档（Document Loading）
# ============================================================
# 把 PDF 读进来。
# PyPDFLoader 默认会按"页"切分，每一页变成一个 Document 对象。
# 一个 Document 包含两个字段：
#   - page_content (str): 这一页的文本内容
#   - metadata (dict): 元数据，比如 {"source": "...", "page": 0}
loader = PyPDFLoader("company_handbook.pdf")
docs = loader.load()  # 返回 List[Document]，长度 = PDF 页数

# 注意：此时每个 Document 可能很大（一整页文本），
# 直接塞给 embedding 模型会超出 token 限制，所以下一步要切分。


# ============================================================
# 第 2 步：切分文档（Text Splitting / Chunking）
# ============================================================
# chunk_size=1000：每个 chunk 最多 1000 个字符（不是 token，注意区别）
# chunk_overlap=200：相邻 chunk 之间重叠 200 个字符
# 
# 为什么要 overlap？因为如果一个句子被切在两个 chunk 的边界上，
# 单独看任何一个 chunk 都缺少上下文。重叠可以缓解这个问题。
# 
# 经验值：
#   - chunk_size 太小（如 200）→ 检索精确但缺乏上下文，LLM 难以理解
#   - chunk_size 太大（如 4000）→ 检索不精确，且会浪费 token、稀释相关性
#   - 1000 + 200 overlap 是大多数英文/中文场景的合理起点
splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,
    chunk_overlap=200
)
chunks = splitter.split_documents(docs)
# 切完之后，chunks 的数量通常远大于原 docs 的数量。
# 比如一份 50 页的 PDF 可能产生 200~400 个 chunks。


# ============================================================
# 第 3 步：向量化 + 存入向量数据库（Embedding & Indexing）
# ============================================================
# OpenAIEmbeddings 默认使用 text-embedding-3-small 模型
# （维度 1536，比老的 ada-002 更便宜且更好）
embeddings = OpenAIEmbeddings()

# FAISS.from_documents 做了三件事：
#   1. 对每个 chunk 调用 embeddings.embed_documents()，得到向量
#   2. 创建一个 FAISS 索引（默认是 IndexFlatL2 - 暴力 L2 距离）
#   3. 把向量 + 原始 chunk 文本 + metadata 一起存进去
# 
# ⚠️ 这一步会调用 Embedding API（不是 LLM API，便宜很多，详见第五章）。
# LangChain 内部会自动 batch，几百个 chunk 通常 1~2 次请求就搞定。
# 生产环境通常做一次后把索引序列化到磁盘，下次直接加载（见下方持久化代码）。
vectorstore = FAISS.from_documents(chunks, embeddings)

# 把 vectorstore 包装成 Retriever。
# Retriever 是一个抽象接口，统一了"给定 query 返回相关文档"的行为。
# search_kwargs={"k": 4} 表示每次检索返回 top-4 个最相似的 chunks。
# 
# k 的取值经验：
#   - k 太小（如 1）→ 容易漏掉相关信息
#   - k 太大（如 20）→ 噪声多，prompt 太长，成本高
#   - 4~6 是常见选择
retriever = vectorstore.as_retriever(search_kwargs={"k": 4})


# ============================================================
# 第 4 步：构造 Prompt 模板
# ============================================================
# 这是 RAG 的灵魂：明确告诉 LLM "只基于 context 回答"，
# 否则 LLM 会"自信地"编造（hallucination）。
# 
# {context} 和 {question} 是占位符，运行时会被替换。
prompt = ChatPromptTemplate.from_template("""
基于以下 context 回答问题。如果答案不在 context 中，说"我不知道"。

Context: {context}

Question: {question}
""")
# 工程实践提示：
#   - 应当加入"请引用来源"的指令，让 LLM 输出 [Source: page 3] 之类的标记
#   - 应当限制回答长度
#   - 复杂场景可以加入 few-shot 例子


# ============================================================
# 第 5 步：用 LCEL 把所有组件组装成一个 chain
# ============================================================
llm = ChatOpenAI(model="gpt-4o-mini")

# format_docs：把 retriever 返回的 List[Document] 拼成一个大字符串。
# 因为 prompt 里的 {context} 需要的是 string，不是 Document 对象列表。
def format_docs(docs):
    # 用两个换行分隔每个 chunk，让 LLM 容易区分不同片段
    return "\n\n".join(d.page_content for d in docs)

# 下面这段是整个 RAG pipeline 的核心，逐字解释：
rag_chain = (
    # 第一阶段：构造一个 dict，准备喂给 prompt 模板
    # 这个 dict 有两个 key，对应 prompt 里的 {context} 和 {question}：
    #
    #   "context": retriever | format_docs
    #     ↑ 把输入（用户问题）送进 retriever，得到相关 chunks，再用 format_docs 拼成字符串
    #
    #   "question": RunnablePassthrough()
    #     ↑ 把输入（用户问题）原样传过来
    #
    # 关键点：LCEL 看到一个 dict 时，会把 dict 的所有 value **并行执行**，
    # 然后把结果重新组合成一个 dict。所以 retriever 和 passthrough 是并行跑的。
    {
        "context": retriever | format_docs,
        "question": RunnablePassthrough()
    }
    # 第二阶段：把上一步的 dict {"context": ..., "question": ...} 喂给 prompt 模板
    # prompt 会做字符串替换，输出一个 ChatPromptValue 对象
    | prompt
    # 第三阶段：把 prompt 喂给 LLM，得到 AIMessage 对象
    | llm
    # 第四阶段：把 AIMessage 解析成纯字符串
    | StrOutputParser()
)


# ============================================================
# 第 6 步：调用 chain 提问
# ============================================================
# 整个流程：
#   "公司的年假政策是怎样的？"
#     → 同时进入 retriever（检索）和 passthrough（透传）
#     → retriever 返回 4 个最相关的 chunks
#     → format_docs 拼成 context 字符串
#     → 组合成 {"context": "...", "question": "公司的年假政策是怎样的？"}
#     → prompt 模板填充
#     → LLM 生成回答
#     → StrOutputParser 提取文本
answer = rag_chain.invoke("公司的年假政策是怎样的？")
print(answer)

# 顺便提一句：因为是 LCEL 写的，你也可以：
#   - rag_chain.stream(...)     # 流式输出
#   - rag_chain.ainvoke(...)    # 异步
#   - rag_chain.batch([...])    # 批量
# 而且 LangSmith 会自动 trace 每一步，调试非常方便。
```

**RAG pipeline 数据流可视化**：

```
 用户问题 "公司年假政策？"
        │
        ├──────────────────────┐
        ▼                      ▼
   [retriever]          [RunnablePassthrough]
        │                      │
   找出 top-4 相关 chunks       │
        │                      │
   [format_docs] 拼成字符串      │
        │                      │
        ▼                      ▼
  {"context": "...年假规定...", "question": "公司年假政策？"}
        │
        ▼
   [prompt] 填充模板
        │
        ▼
   [llm] gpt-4o-mini 生成
        │
        ▼
   [StrOutputParser] 解析
        │
        ▼
   "根据公司手册，员工每年享有..."
```

---

### 3.3 LangChain 加 Memory（对话记忆）

#### 3.3.1 简洁版

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain_core.messages import HumanMessage
from langchain_core.runnables.history import RunnableWithMessageHistory
from langchain_community.chat_message_histories import ChatMessageHistory

llm = ChatOpenAI(model="gpt-4o-mini")

prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个友好的助手。"),
    MessagesPlaceholder(variable_name="history"),
    ("human", "{input}")
])

chain = prompt | llm

# 用 dict 保存每个 session 的历史
store = {}
def get_session_history(session_id: str):
    if session_id not in store:
        store[session_id] = ChatMessageHistory()
    return store[session_id]

chain_with_memory = RunnableWithMessageHistory(
    chain,
    get_session_history,
    input_messages_key="input",
    history_messages_key="history"
)

# 第一轮
chain_with_memory.invoke(
    {"input": "我叫小明"},
    config={"configurable": {"session_id": "user_1"}}
)

# 第二轮（会记得名字）
response = chain_with_memory.invoke(
    {"input": "我叫什么名字？"},
    config={"configurable": {"session_id": "user_1"}}
)
print(response.content)  # → "你叫小明"
```

#### 3.3.2 详细注释版

```python
# ============================================================
# 背景：为什么需要 Memory？
# ============================================================
# LLM 本身是无状态的（stateless）—— 每次 API 调用是独立的，
# OpenAI 服务器不会记得你上一次说了什么。
# 如果想让 LLM "记得" 历史对话，必须每次都把所有历史消息
# 一起塞回 prompt 里。Memory 模块帮你自动管理这件事。

from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain_core.messages import HumanMessage
# RunnableWithMessageHistory 是现代推荐方式（旧的 ConversationBufferMemory 已 deprecated）
from langchain_core.runnables.history import RunnableWithMessageHistory
# ChatMessageHistory 就是一个 List[BaseMessage] 的薄封装
from langchain_community.chat_message_histories import ChatMessageHistory


# ============================================================
# 第 1 步：定义 LLM 和带历史占位符的 prompt
# ============================================================
llm = ChatOpenAI(model="gpt-4o-mini")

# 关键点：MessagesPlaceholder("history") 
# 这是一个"占位符"，运行时会被替换成历史消息列表（HumanMessage/AIMessage 交替）
# 完整结构：
#   [SystemMessage("你是一个友好的助手"),
#    HumanMessage("我叫小明"),     ← 来自 history
#    AIMessage("你好小明"),        ← 来自 history
#    HumanMessage("我叫什么？")]   ← 来自当前 input
prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个友好的助手。"),
    MessagesPlaceholder(variable_name="history"),   # ← 历史会插到这里
    ("human", "{input}")                            # ← 当前用户输入
])

# 基础 chain：先填模板再调 LLM。注意此时还没有记忆能力
chain = prompt | llm


# ============================================================
# 第 2 步：定义 session 存储
# ============================================================
# store 是一个内存字典，key 是 session_id，value 是 ChatMessageHistory 对象。
# 
# ⚠️ 生产环境千万不要用这个 dict —— 进程重启就丢了。
# 生产环境应该用：
#   - Redis: RedisChatMessageHistory
#   - PostgreSQL: PostgresChatMessageHistory  
#   - MongoDB: MongoDBChatMessageHistory
# LangChain 都提供了对应实现，只要替换 ChatMessageHistory 即可。
store = {}

def get_session_history(session_id: str):
    """
    给定 session_id，返回（或创建）该 session 的历史对象。
    这个函数会被 RunnableWithMessageHistory 在每次调用时执行。
    """
    if session_id not in store:
        # 第一次见到这个 session_id，创建一个空的 history
        store[session_id] = ChatMessageHistory()
    return store[session_id]


# ============================================================
# 第 3 步：用 RunnableWithMessageHistory 包装 chain
# ============================================================
# 这个包装器做的事情：
#   1. 调用前：用 session_id 查询历史，把它注入到 prompt 的 "history" 占位符
#   2. 调用 LLM 得到回答
#   3. 调用后：把 (用户输入, LLM 回答) 这对消息追加进历史里
chain_with_memory = RunnableWithMessageHistory(
    chain,
    get_session_history,
    # 告诉包装器：用户输入在 input dict 的哪个 key 下面
    # 比如 {"input": "你好"} 中的 "input"
    input_messages_key="input",
    # 告诉包装器：要把历史注入到 prompt 的哪个变量
    # 必须与上面 MessagesPlaceholder(variable_name="history") 一致
    history_messages_key="history"
)


# ============================================================
# 第 4 步：调用 —— 注意 configurable 的传法
# ============================================================
# 关键：config={"configurable": {"session_id": "user_1"}}
# session_id 在 config 而不是 input 里。
# 这样设计是因为 session_id 是"运行时配置"，不是模型输入。
# 同一个 chain 可以同时服务多个用户，只要传不同的 session_id。

# 第一轮对话
chain_with_memory.invoke(
    {"input": "我叫小明"},
    config={"configurable": {"session_id": "user_1"}}
)
# 此时 store["user_1"] 里多了两条消息：
#   [HumanMessage("我叫小明"), AIMessage("你好小明，很高兴认识你！")]

# 第二轮对话（同一个 session_id）
response = chain_with_memory.invoke(
    {"input": "我叫什么名字？"},
    config={"configurable": {"session_id": "user_1"}}
)
# 这次调用前，history 占位符会被替换成上一轮的两条消息，
# 所以 LLM 能"看到"你之前说过"我叫小明"
print(response.content)  # → "你叫小明"

# 如果换一个 session_id，就是另一个用户，互不影响
chain_with_memory.invoke(
    {"input": "我叫什么名字？"},
    config={"configurable": {"session_id": "user_2"}}
)
# → "抱歉，我不知道你的名字"  ← user_2 没说过名字
```

[get_session_history](./get_session_history_回调函数详解.md)

---

### 3.4 LangGraph 基础：StateGraph

#### 3.4.1 简洁版

```python
from typing import TypedDict, Annotated
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langchain_core.messages import HumanMessage, AIMessage
from langchain_openai import ChatOpenAI

# 1. 定义 State（用 TypedDict）
class State(TypedDict):
    # add_messages 是 reducer：自动 append 而不是覆盖
    messages: Annotated[list, add_messages]

# 2. 定义节点（就是普通函数）
llm = ChatOpenAI(model="gpt-4o-mini")

def chatbot_node(state: State):
    response = llm.invoke(state["messages"])
    return {"messages": [response]}  # 返回的 dict 会和 state 合并

# 3. 构建 Graph
graph_builder = StateGraph(State)
graph_builder.add_node("chatbot", chatbot_node)
graph_builder.add_edge(START, "chatbot")
graph_builder.add_edge("chatbot", END)

graph = graph_builder.compile()

# 4. 调用
result = graph.invoke({"messages": [HumanMessage(content="你好！")]})
print(result["messages"][-1].content)
```

#### 3.4.2 详细注释版

```python
# ============================================================
# 背景：StateGraph 是 LangGraph 的核心抽象
# ============================================================
# 一个 StateGraph 由三要素组成：
#   - State: 一个 TypedDict，描述全局共享的状态结构
#   - Node:  一个函数，接收 state 返回更新
#   - Edge:  连接节点的有向边（普通 / 条件 / 并行）

from typing import TypedDict, Annotated
# StateGraph: 图的 builder
# START, END: 两个特殊节点，表示流程起点和终点
from langgraph.graph import StateGraph, START, END
# add_messages: 内置的 reducer，专门用于消息列表的追加合并
from langgraph.graph.message import add_messages
from langchain_core.messages import HumanMessage, AIMessage
from langchain_openai import ChatOpenAI


# ============================================================
# 第 1 步：定义 State Schema
# ============================================================
# TypedDict 描述 state 的字段结构。LangGraph 用这个做类型检查和自动文档。
# 
# 关键概念 - Reducer：
# 当一个 node 返回 {"messages": [new_msg]} 时，
# LangGraph 需要决定：把新值"覆盖"旧值，还是"合并"？
# 
# - 默认行为：覆盖（赋值）
# - 使用 Annotated[T, reducer_func] 可指定合并逻辑
# 
# add_messages 是一个特殊的 reducer：
#   - 新消息会被 append 到现有列表
#   - 如果新消息有相同 id，会替换而不是追加（支持消息编辑）
#   - 自动给没有 id 的消息分配 uuid
class State(TypedDict):
    # 等价于说："messages 是 list 类型，更新时用 add_messages 合并"
    messages: Annotated[list, add_messages]


# ============================================================
# 第 2 步：定义节点（Node）
# ============================================================
# 节点就是普通 Python 函数（也可以是 async 函数）。
# 签名约定：
#   - 输入：当前 state（你可以读取 state 里的任何字段）
#   - 输出：dict，只需包含要更新的字段（不需要返回完整 state）
# 
# 返回的 dict 会通过 reducer 合并进 state。
llm = ChatOpenAI(model="gpt-4o-mini")

def chatbot_node(state: State):
    # 读取 state 里的全部历史消息，喂给 LLM
    response = llm.invoke(state["messages"])
    # 返回：要把 response 加进 messages 列表
    # 因为 messages 字段用了 add_messages reducer，
    # 所以返回 [response] 会被 append，而不是覆盖整个列表
    return {"messages": [response]}


# ============================================================
# 第 3 步：构建图（Build the Graph）
# ============================================================
# StateGraph(State) 创建一个 builder，泛型参数是 state schema
graph_builder = StateGraph(State)

# 添加节点：第一个参数是节点名（字符串，用于路由引用），第二个是函数
graph_builder.add_node("chatbot", chatbot_node)

# 添加边：
# START 是特殊的"入口"节点，所有流程从这里开始
# END 是特殊的"出口"节点，到这里图就停止
graph_builder.add_edge(START, "chatbot")   # START → chatbot
graph_builder.add_edge("chatbot", END)     # chatbot → END

# compile() 把 builder 变成可执行的图。
# 编译时会做：
#   - 验证图的合法性（没有死节点、没有循环引用问题等）
#   - 优化执行计划
#   - 准备 checkpointer 等基础设施
graph = graph_builder.compile()


# ============================================================
# 第 4 步：调用图
# ============================================================
# invoke 的输入是初始 state（可以只传部分字段，没传的字段为 None / 默认值）
result = graph.invoke({"messages": [HumanMessage(content="你好！")]})

# result 是终态 state（最后一个节点执行完之后的完整 state）
# result["messages"] 包含了：[原始 HumanMessage, LLM 返回的 AIMessage]
print(result["messages"][-1].content)  # → "你好！很高兴见到你"


# ============================================================
# 补充：其他调用方式
# ============================================================
# 流式输出每个节点的执行结果：
# for chunk in graph.stream({"messages": [HumanMessage("你好")]}):
#     print(chunk)  # 每个节点执行完输出一次

# 流式输出 LLM 生成的 token（更细粒度）：
# for chunk in graph.stream(..., stream_mode="messages"):
#     print(chunk)
```

---

### 3.5 LangGraph 实现 ReAct Agent（带 Tools 和循环）

#### 3.5.1 简洁版

```python
from typing import TypedDict, Annotated, Literal
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode
from langchain_core.messages import HumanMessage
from langchain_core.tools import tool
from langchain_openai import ChatOpenAI

# 1. 定义 tools
@tool
def get_weather(city: str) -> str:
    """获取指定城市的天气"""
    return f"{city}今天晴天，25度"

@tool
def get_population(city: str) -> str:
    """获取指定城市的人口"""
    return f"{city}人口约2400万"

tools = [get_weather, get_population]

# 2. LLM 绑定 tools
llm = ChatOpenAI(model="gpt-4o-mini").bind_tools(tools)

# 3. 定义 State
class State(TypedDict):
    messages: Annotated[list, add_messages]

# 4. 定义节点
def agent_node(state: State):
    response = llm.invoke(state["messages"])
    return {"messages": [response]}

tool_node = ToolNode(tools)  # LangGraph 内置的工具执行节点

# 5. 定义条件边（关键！）
def should_continue(state: State) -> Literal["tools", END]:
    last_message = state["messages"][-1]
    # 如果 LLM 想调用工具，去 tools 节点；否则结束
    if last_message.tool_calls:
        return "tools"
    return END

# 6. 构建图
builder = StateGraph(State)
builder.add_node("agent", agent_node)
builder.add_node("tools", tool_node)

builder.add_edge(START, "agent")
builder.add_conditional_edges("agent", should_continue)
builder.add_edge("tools", "agent")  # ← 关键的循环！tools 执行完回到 agent

graph = builder.compile()

# 7. 调用
result = graph.invoke({
    "messages": [HumanMessage(content="北京和上海的天气和人口分别是多少？")]
})
for m in result["messages"]:
    m.pretty_print()
```

#### 3.5.2 详细注释版

```python
# ============================================================
# 背景：ReAct = Reasoning + Acting
# ============================================================
# ReAct 是 2022 年提出的 agent 范式，让 LLM 交替进行：
#   Thought（思考）→ Action（行动/调用工具）→ Observation（看结果）→ ...
# 直到得出最终答案。
# 
# 在 LangGraph 里，ReAct 就是这样一个图：
#   START → agent → [条件分支]
#                       ├──→ tools → agent (循环)
#                       └──→ END

from typing import TypedDict, Annotated, Literal
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
# ToolNode：LangGraph 预置的节点，自动执行 LLM 请求的工具调用
from langgraph.prebuilt import ToolNode
from langchain_core.messages import HumanMessage
# @tool 装饰器：把普通函数变成 LLM 可调用的工具
from langchain_core.tools import tool
from langchain_openai import ChatOpenAI


# ============================================================
# 第 1 步：定义 tools（工具）
# ============================================================
# @tool 装饰器做的事：
#   1. 读取函数签名（参数名、类型注解）→ 生成 JSON Schema
#   2. 读取 docstring → 作为工具描述给 LLM 看
#   3. 包装成 BaseTool 对象，可以被 LLM 调用
# 
# ⚠️ docstring 极其重要！LLM 是靠它决定何时调用此工具的。
@tool
def get_weather(city: str) -> str:
    """获取指定城市的天气"""  # ← 这个 docstring 会进入 LLM 的 prompt
    # 实际场景这里会调真实的天气 API
    return f"{city}今天晴天，25度"

@tool
def get_population(city: str) -> str:
    """获取指定城市的人口"""
    return f"{city}人口约2400万"

tools = [get_weather, get_population]


# ============================================================
# 第 2 步：让 LLM 知道有哪些 tools
# ============================================================
# bind_tools 把 tools 的 schema 注入 LLM 的请求，
# OpenAI/Anthropic 的 API 原生支持 function/tool calling。
# 
# 调用 llm.invoke(...) 时，LLM 可能返回：
#   选项 A: 普通文本回答（AIMessage.content 有值，tool_calls 为空）
#   选项 B: 想调用工具（AIMessage.tool_calls 包含工具名和参数）
llm = ChatOpenAI(model="gpt-4o-mini").bind_tools(tools)


# ============================================================
# 第 3 步：定义 State
# ============================================================
class State(TypedDict):
    # 所有消息（人类、AI、工具结果）都按顺序存在这里
    messages: Annotated[list, add_messages]


# ============================================================
# 第 4 步：定义节点
# ============================================================
# agent 节点：就是调用 LLM，让它决定下一步做什么
def agent_node(state: State):
    response = llm.invoke(state["messages"])
    # response 可能是普通回答，也可能包含 tool_calls
    return {"messages": [response]}

# tool 节点：LangGraph 预置的 ToolNode 
# 它会：
#   1. 读取 state 里最后一条 AIMessage 的 tool_calls
#   2. 并发执行所有被请求的工具
#   3. 把结果包装成 ToolMessage 加进 state
# 
# 你也可以自己实现这个节点，但用预置的更省事。
tool_node = ToolNode(tools)


# ============================================================
# 第 5 步：定义条件边（routing function）
# ============================================================
# 这是 ReAct 循环的关键！
# 每次 agent 执行完，决定下一步去哪：
#   - 如果 LLM 想调用工具 → 去 tools 节点
#   - 否则（LLM 给了最终答案）→ 结束

# 返回值的类型注解 Literal["tools", END] 不是强制的，
# 但写上之后 IDE 和 mypy 会帮你检查。
def should_continue(state: State) -> Literal["tools", END]:
    # 看最新一条消息（一定是 agent 刚生成的 AIMessage）
    last_message = state["messages"][-1]
    
    # 如果这条消息包含 tool_calls，说明 LLM 想调用工具
    # tool_calls 是一个 list，每个元素长这样：
    #   {"name": "get_weather", "args": {"city": "北京"}, "id": "call_abc"}
    if last_message.tool_calls:
        return "tools"
    
    # 否则 LLM 给的是最终答案（自然语言），结束
    return END


# ============================================================
# 第 6 步：构建图
# ============================================================
builder = StateGraph(State)

# 添加两个节点
builder.add_node("agent", agent_node)
builder.add_node("tools", tool_node)

# START → agent：流程从 agent 开始
builder.add_edge(START, "agent")

# agent → [条件边]：根据 should_continue 的返回值动态决定
# 这是和普通 add_edge 的核心区别：目的地不是固定的
builder.add_conditional_edges("agent", should_continue)

# tools → agent：这条边创建了循环！
# 工具执行完，回到 agent 让它看结果决定下一步
# （继续调工具？还是已经够信息可以总结答案了？）
builder.add_edge("tools", "agent")

graph = builder.compile()


# ============================================================
# 第 7 步：调用
# ============================================================
result = graph.invoke({
    "messages": [HumanMessage(content="北京和上海的天气和人口分别是多少？")]
})

# 这次调用会经历多轮循环：
# 
# Round 1:
#   agent: "我需要查北京和上海的天气和人口" → 请求 4 个 tool calls
#   should_continue: 有 tool_calls → 去 tools
#   tools: 并发执行 4 个工具，得到 4 条 ToolMessage
# 
# Round 2:
#   agent: 看到所有工具结果 → 生成最终自然语言回答
#   should_continue: 没有 tool_calls → END
# 
# 打印所有消息看完整过程：
for m in result["messages"]:
    m.pretty_print()
# 你会看到：
#   HumanMessage(用户问题)
#   AIMessage(想调用 4 个工具)
#   ToolMessage × 4 (工具结果)
#   AIMessage(最终回答)


# ============================================================
# 提示：还有更短的写法
# ============================================================
# 上面这一切，LangGraph 提供了一个 prebuilt 直接用：
# 
# from langgraph.prebuilt import create_react_agent
# graph = create_react_agent(llm, tools)
# 
# 适合快速搭建。但理解上面的"手写版"对调试和定制非常重要。
```

---

### 3.6 LangGraph 加 Checkpointer（持久化 + Human-in-the-loop）

#### 3.6.1 简洁版

```python
from langgraph.checkpoint.memory import MemorySaver
# 生产环境用 PostgresSaver 或 SqliteSaver

memory = MemorySaver()
graph = builder.compile(
    checkpointer=memory,
    interrupt_before=["tools"]  # 在执行 tools 前暂停，等人工确认
)

config = {"configurable": {"thread_id": "user_123"}}

# 第一次调用 - 会停在 tools 节点之前
graph.invoke({"messages": [HumanMessage(content="北京天气如何？")]}, config)

# 查看当前状态
state = graph.get_state(config)
print("即将执行：", state.next)  # → ('tools',)

# 人工审核后，继续执行
graph.invoke(None, config)  # 传 None 表示继续
```

#### 3.6.2 详细注释版

```python
# ============================================================
# 背景：Checkpointer 是什么、解决什么问题？
# ============================================================
# Checkpointer = 在图的每个节点执行后自动保存 state 的机制。
# 
# 它带来的能力：
#   1. 长期记忆：同一个 thread_id 再次调用，会自动加载上次的 state
#   2. Time travel：可以回到任意历史 checkpoint 重新执行
#   3. Human-in-the-loop：在指定节点暂停，等人工审核后再继续
#   4. 容错：节点崩溃后可以从上一个 checkpoint 恢复
#   5. 调试：完整记录每一步，对照 LangSmith 很方便定位问题

# MemorySaver：内存版的 checkpointer
# ⚠️ 进程重启就丢，仅适合开发/演示
from langgraph.checkpoint.memory import MemorySaver

# 生产环境的选项：
# from langgraph.checkpoint.sqlite import SqliteSaver       # 单机持久化
# from langgraph.checkpoint.postgres import PostgresSaver   # 生产首选
# from langgraph.checkpoint.mongodb import MongoDBSaver

memory = MemorySaver()


# ============================================================
# 第 1 步：编译图时传入 checkpointer 和 interrupt 配置
# ============================================================
# 假设 builder 是 3.5 节里的 ReAct agent 的 StateGraph
graph = builder.compile(
    checkpointer=memory,
    # interrupt_before：在指定节点执行**之前**暂停
    # 这里的意思是：每次轮到 tools 节点要执行时，先停下来等指令
    # 用途：人工审核 LLM 决定调用什么工具，避免危险操作
    interrupt_before=["tools"]
    
    # 还有：
    # interrupt_after=["..."]   # 在节点执行**之后**暂停
)


# ============================================================
# 第 2 步：thread_id —— 标识一个对话/会话
# ============================================================
# thread_id 是 checkpointer 的"主键"。
# - 同一个 thread_id 的多次调用，共享同一份 state
# - 不同 thread_id 互不影响
# - 类比：thread_id ≈ chat 应用里的"对话 ID"
config = {"configurable": {"thread_id": "user_123"}}


# ============================================================
# 第 3 步：第一次调用 —— 会在 tools 节点前暂停
# ============================================================
graph.invoke(
    {"messages": [HumanMessage(content="北京天气如何？")]},
    config  # ← 别忘了传 config，否则没有持久化
)

# 此时图的执行流是：
#   START → agent (LLM 决定调用 get_weather) 
#   → ⏸ 暂停（因为下一步是 tools，被 interrupt_before 拦下）
# 
# graph.invoke() 不抛异常，正常返回，但流程并没真正走完


# ============================================================
# 第 4 步：检查当前状态，决定是否放行
# ============================================================
# get_state 返回当前 thread 的状态快照
state = graph.get_state(config)

# state 的核心字段：
#   - values: 完整的 state dict（messages 等所有字段）
#   - next:   一个 tuple，里面是"接下来要执行的节点名"
#             如果是 () 说明流程已结束
#             这里应该是 ('tools',)
print("即将执行：", state.next)

# 看 LLM 想调用什么工具（这是人工审核的重点）：
last_msg = state.values["messages"][-1]
print("LLM 想调用的工具：", last_msg.tool_calls)
# → [{"name": "get_weather", "args": {"city": "北京"}, ...}]

# 这时候你可以：
#   选项 A: 同意 → 继续执行（见第 5 步）
#   选项 B: 拒绝 → 不再 invoke，或修改 state 后再继续
#   选项 C: 修改参数 → graph.update_state(config, {"messages": [...]})


# ============================================================
# 第 5 步：继续执行（resume）
# ============================================================
# 关键技巧：invoke 时传 None 作为 input，表示"从 checkpoint 继续，不要新输入"
graph.invoke(None, config)

# 这次会从上次暂停的地方继续：
#   tools (执行 get_weather) → agent (生成自然语言回答) → END


# ============================================================
# 补充：修改 state 后再继续
# ============================================================
# 如果人工不满意 LLM 的工具调用，可以改写：
# 
# graph.update_state(
#     config,
#     {"messages": [edited_ai_message]},
#     as_node="agent"  # 假装这条更新是从 agent 节点输出的
# )
# graph.invoke(None, config)


# ============================================================
# 补充：查看历史 checkpoints（time travel）
# ============================================================
# for snapshot in graph.get_state_history(config):
#     print(snapshot.config, snapshot.next, snapshot.values)
# 
# 拿到任意一个历史 snapshot 的 config，传给 invoke 就能从那里重新执行
# graph.invoke(None, historical_snapshot.config)
```

---

## 四、补充：FAISS 详解

### 4.1 FAISS 是什么？

**FAISS** = **Fa**cebook **AI** **S**imilarity **S**earch，是 Meta（原 Facebook）AI Research 团队在 2017 年开源的**向量相似度搜索库（vector similarity search library）**。

### 4.2 发音

读作 **"face"**（/feɪs/），就是英文「脸」那个词的发音。Meta 团队自己就是这么读的。不是「fais」也不是「F-A-I-S-S」分开念。

### 4.3 它属于哪种检索？

简短回答：**FAISS 是纯粹的 dense vector top-k 检索，既不是 BM25，也不是 hybrid。**

来梳理一下这几个容易混淆的概念：

| 概念 | 解释 | 关系 |
|---|---|---|
| **Top-k** | 一种**操作**：返回最相似的 k 个结果 | 是 *动作*，不是 *算法* |
| **BM25** | 一种**稀疏检索算法（sparse retrieval）**，基于关键词词频统计 | 是 *具体算法* |
| **Dense Vector Search** | 一种**稠密检索算法（dense retrieval）**，基于 embedding 向量的相似度 | 是 *具体算法* |
| **Hybrid Search** | 把 BM25 + Dense Vector 的结果融合（通常用 RRF 或加权） | 是 *策略* |

FAISS 做的是**「在大规模 dense vector 集合中高效找出 top-k 个最相似的向量」**。它不懂关键词，不懂 BM25，它只懂向量。

### 4.4 FAISS 的核心能力

它提供多种索引算法，让你在**搜索精度**和**速度/内存**之间权衡：

| 索引类型 | 原理 | 特点 |
|---|---|---|
| `IndexFlatL2` / `IndexFlatIP` | 暴力计算（exact search） | 精度 100%，但慢，适合 < 100 万条 |
| `IndexIVF` | Inverted File，先聚类再搜索 | 快，精度略降 |
| `IndexHNSW` | Hierarchical Navigable Small World（图算法） | 高速高精度，内存大 |
| `IndexPQ` | Product Quantization，向量压缩 | 省内存，精度有损 |
| `IndexIVFPQ` | IVF + PQ 组合 | 大规模数据的标配 |

### 4.5 如果要做 Hybrid Search 怎么办？

FAISS 本身不行，需要组合：

```python
# 方案：BM25 (用 rank_bm25) + FAISS (向量) → EnsembleRetriever 融合
from langchain.retrievers import EnsembleRetriever
from langchain_community.retrievers import BM25Retriever

bm25_retriever = BM25Retriever.from_documents(chunks)        # 稀疏
faiss_retriever = FAISS.from_documents(chunks, emb).as_retriever()  # 稠密

hybrid = EnsembleRetriever(
    retrievers=[bm25_retriever, faiss_retriever],
    weights=[0.4, 0.6]  # 加权融合
)
```

生产环境更常用的向量库还有 **Pinecone、Weaviate、Qdrant、Milvus、Chroma、pgvector**，其中一些（如 Weaviate、Qdrant）原生支持 hybrid search，FAISS 不行。

### 4.6 FAISS 索引持久化（save_local / load_local）

3.2 的 RAG 示例里有这一行：

```python
vectorstore = FAISS.from_documents(chunks, embeddings)
```

这一步会调用 Embedding API 把所有 chunks 向量化，是有成本的。**生产环境绝对不能每次启动都重跑**，应该把索引序列化到磁盘：

```python
import os
from langchain_community.vectorstores import FAISS

INDEX_PATH = "faiss_index"

if os.path.exists(INDEX_PATH):
    # 第二次及以后：直接从磁盘加载，不调用任何 API，0 成本
    # allow_dangerous_deserialization=True 是因为 FAISS 索引用 pickle 存储，
    # LangChain 强制你明确知道在反序列化（防止恶意 pickle 攻击）
    vectorstore = FAISS.load_local(
        INDEX_PATH,
        embeddings,
        allow_dangerous_deserialization=True
    )
else:
    # 第一次：调用 embedding API，建索引
    vectorstore = FAISS.from_documents(chunks, embeddings)
    # 存到磁盘，下次启动就不用重新算了
    vectorstore.save_local(INDEX_PATH)
```

`save_local` 会在磁盘上生成两个文件：
- `index.faiss`：FAISS 的二进制向量索引
- `index.pkl`：原始文本和 metadata（pickle 格式）

---

## 五、补充：Embedding API vs LLM API

这是 RAG 学习中一个非常容易混淆的点。

### 5.1 它们是两个不同的 API

OpenAI 的 API 有两大类，性质完全不同：

| | **Chat API**（LLM call） | **Embedding API** |
|---|---|---|
| 模型 | gpt-4o, gpt-4o-mini, o1... | text-embedding-3-small/large |
| 输入 | 文本（prompt） | 文本 |
| 输出 | 文本（生成的回答） | 一个向量（一串数字，如 [0.12, -0.34, ...]，长度 1536） |
| 用途 | 生成、对话、推理 | 把文本变成向量用于相似度计算 |
| 速度 | 慢（几秒~几十秒） | 快（几百毫秒） |
| 价格（每 1M tokens） | gpt-4o-mini 约 $0.15 输入 | text-embedding-3-small 约 **$0.02** |
| 是否支持批量 | 不能（每次一个对话） | **可以（一次发几百~几千条文本）** |

所以 `FAISS.from_documents(chunks, embeddings)` 调用的是 **Embedding API，不是 LLM call**。它们便宜大约 7~8 倍，而且速度快很多。

### 5.2 那到底会调用几次 API？

不是 "有几个 chunk 就调几次"。OpenAI 的 Embedding API 支持**批量（batch）**：一次请求可以塞进去最多 2048 条文本。

LangChain 的 `OpenAIEmbeddings` 内部有个参数叫 `chunk_size`（**注意：这个 chunk_size 是 batch size，不是文档切分的 chunk_size，名字撞车了**），默认是 **1000**。意思是：每次 API 请求最多打包 1000 条文本一起发。

举个具体例子，假设你有 400 个 chunks：

```
400 chunks → 1 次 embedding API 请求（因为 400 < 1000）
3500 chunks → 4 次 embedding API 请求（每次最多 1000 条）
50000 chunks → 50 次 embedding API 请求
```

### 5.3 实际成本估算

假设你有一份 50 页的 PDF，按 chunk_size=1000 字符切分，大约产生 400 个 chunks。

- 每个 chunk 约 250 tokens（中文每字 1~1.5 token，1000 字符大概 200~300 tokens）
- 总 tokens：400 × 250 = 100,000 tokens = 0.1M tokens
- 用 text-embedding-3-small：0.1 × $0.02 = **$0.002**（不到 1 分钱）
- API 请求次数：**1 次**（因为 400 < 1000）

所以小项目几乎免费。但**生产环境**有几种情况会让成本上来：

1. **大语料库**：比如把整个公司的 10 万份文档索引化，可能就是几十美元。
2. **重复索引**：每次部署重新跑一次，浪费钱。
3. **embedding 大模型**：text-embedding-3-large 比 small 贵 6 倍多。

→ 解决办法见 [4.6 节的持久化](#46-faiss-索引持久化save_local--load_local)。

### 5.4 检索阶段（retrieve）也会调用 embedding API

补充一个容易忽略的点：**用户每问一个问题，也会触发 1 次 embedding API 调用**，因为要把问题向量化才能去 FAISS 里搜索。

```python
rag_chain.invoke("公司年假政策？")
# ↑ 这一次调用内部会发生：
#   1 次 embedding API call（把问题变成向量）  ← 便宜
#   1 次 FAISS 本地搜索（不走网络）           ← 免费
#   1 次 chat completion API call（LLM 生成回答） ← 这才是真正的 LLM call，最贵
```

所以一次 RAG 问答的成本几乎全在最后那个 chat completion 上。embedding 部分可以忽略不计。

### 5.5 一句话总结

`FAISS.from_documents(chunks, embeddings)` 调用的是**便宜且批量的 embedding API**，不是 LLM call。LangChain 会自动 batch，几百个 chunk 通常 1~2 次请求就搞定。但**索引化是一次性成本**，所以"建一次就持久化"是好习惯，省钱也省时间。

---

## 六、Top 10 Interview Questions for AI Engineers (English)

### Q1: What is the difference between LangChain and LangGraph, and when would you use each?

**Answer:**
LangChain is a framework providing abstractions (models, prompts, retrievers, tools, memory) for building LLM applications. Its core composition primitive, LCEL, builds **DAGs (Directed Acyclic Graphs)** — linear or branching pipelines that flow in one direction.

LangGraph is built on top of LangChain primitives but adds a **stateful, cyclic graph orchestration layer**. It models applications as a state machine with nodes (functions) and edges (transitions), including conditional and cyclic edges.

**When to use LangChain (specifically LCEL):**
- Linear pipelines: RAG, prompt → LLM → parse
- Stateless transformations
- Simple chatbots with basic memory

**When to use LangGraph:**
- Agents with tool-use loops (the LLM decides to call tools repeatedly)
- Multi-agent systems
- Workflows requiring human-in-the-loop
- Anything requiring durable state, branching, retries, or cycles
- Production agents needing checkpointing, streaming intermediate steps, and observability

In short: **LCEL for pipelines, LangGraph for agents and workflows**. The LangChain team officially recommends building new agent applications with LangGraph instead of the legacy `AgentExecutor`.

---

### Q2: Explain LCEL (LangChain Expression Language). Why was it introduced and what does it give you for free?

**Answer:**
LCEL is a declarative composition language using the `|` operator to chain `Runnable` objects:

```python
chain = prompt | llm | parser
```

It was introduced to replace heavy, rigid classes like `LLMChain` and `SequentialChain`, which made it hard to customize behavior and didn't compose well.

Every LCEL chain is a `Runnable`, which gives you these capabilities **for free**:

1. **Unified interface**: `.invoke()`, `.batch()`, `.stream()`, `.ainvoke()`, `.abatch()`, `.astream()` — sync, async, and streaming, all without rewriting code.
2. **Automatic parallelization**: When you pass a dict like `{"a": chain_a, "b": chain_b}`, both run in parallel.
3. **Streaming support**: Tokens stream through the entire chain end-to-end.
4. **First-class observability**: Every step is automatically traced in LangSmith.
5. **Fallbacks and retries**: `.with_fallbacks()`, `.with_retry()` work on any Runnable.
6. **Schema introspection**: `.input_schema` and `.output_schema` for validation.

Under the hood, the `|` operator is just `__or__` returning a `RunnableSequence`. The dict syntax becomes `RunnableParallel`.

---

### Q3: What is RAG (Retrieval-Augmented Generation) and how do you implement it with LangChain? What are common pitfalls?

**Answer:**
RAG augments LLM responses with retrieved external knowledge so the model can answer questions about data it wasn't trained on (private docs, recent info).

**Standard pipeline:**
1. **Ingestion**: Load documents (`DocumentLoader`) → Split into chunks (`TextSplitter`) → Embed (`Embeddings`) → Store (`VectorStore`).
2. **Retrieval**: At query time, embed the user's question, find top-k similar chunks via vector similarity.
3. **Generation**: Stuff retrieved chunks into the prompt as context and let the LLM answer.

**Common pitfalls and mitigations:**

| Pitfall | Mitigation |
|---|---|
| Chunk size too small → loses context | Use `RecursiveCharacterTextSplitter` with overlap (e.g. 1000/200) |
| Chunk size too large → noisy retrieval, hits token limit | Tune based on document type |
| Semantic search misses keyword matches | **Hybrid search** (BM25 + vector) |
| Top-k retrieval returns near-duplicates | Use **MMR** (Maximal Marginal Relevance) |
| User query is too vague for good retrieval | **Query rewriting** / **HyDE** (Hypothetical Document Embeddings) |
| Wrong chunks retrieved due to embedding limitations | Add a **re-ranker** (e.g. Cohere Rerank) |
| LLM hallucinates beyond context | Strict prompt: "If not in context, say you don't know" + cite sources |
| Stale data in vector store | Implement re-indexing pipeline |
| No multi-hop reasoning | Use **agentic RAG** with LangGraph — let an agent iteratively retrieve |

---

### Q4: Explain how agents work in LangChain. What is the ReAct pattern?

**Answer:**
An agent is an LLM that **decides which actions to take** rather than following a fixed pipeline. The LLM is given a set of tools (functions) and chooses which to call based on the input.

**ReAct (Reasoning + Acting)** is the foundational agent pattern from a 2022 paper. The LLM alternates between:
- **Thought**: "I need to find X, so I should use tool Y"
- **Action**: Call tool Y with arguments
- **Observation**: Tool returns result
- **Thought**: "Now I know X, but I also need Z..."
- ...
- **Final Answer**

Modern implementations use **native function/tool calling** from the LLM provider (OpenAI, Anthropic) instead of parsing free-form text, which is much more reliable.

In LangChain, you can use `create_react_agent` from `langgraph.prebuilt` (the modern way) or the legacy `AgentExecutor`. The legacy approach hides the loop behind a black box; LangGraph exposes it as an explicit graph: `agent_node → tool_node → agent_node → ...` with a conditional edge that exits when no more tool calls are requested.

**Key challenges with agents in production:**
- **Infinite loops** — set `recursion_limit`
- **Cost explosion** — agents can rack up many LLM calls
- **Unreliable tool selection** — improve tool descriptions, use few-shot examples
- **Hallucinated tool arguments** — use structured outputs / strict schemas
- **No determinism** — log every step with LangSmith

---

### Q5: What is the StateGraph in LangGraph? Explain State, Nodes, Edges, and Reducers.

**Answer:**
`StateGraph` is the core abstraction in LangGraph for building stateful, multi-step LLM workflows.

**State**: A typed dictionary (usually `TypedDict` or Pydantic model) that flows through the graph. It's the single source of truth — every node reads from and writes to it.

```python
class State(TypedDict):
    messages: Annotated[list, add_messages]
    user_id: str
    retrieved_docs: list
```

**Node**: A Python function (sync or async) that takes the current state and returns a partial state update (a dict). The framework merges the returned dict into the state.

```python
def my_node(state: State) -> dict:
    return {"retrieved_docs": [...]}
```

**Edge**: Connects nodes. Two types:
- **Normal edge**: `graph.add_edge("a", "b")` — always go from a to b.
- **Conditional edge**: `graph.add_conditional_edges("a", routing_fn)` — `routing_fn(state)` returns the name of the next node.

**Reducer**: Defines **how** an updated field is merged into state. By default, returning `{"x": 5}` overwrites `state["x"]`. With a reducer (via `Annotated[list, add_messages]`), updates can be appended, merged, or transformed.

This matters because in agents, you want messages to **accumulate** across nodes, not be replaced. Without `add_messages`, every node would wipe the conversation history.

Special nodes `START` and `END` mark the entry and exit. The graph is compiled with `.compile()` before use.

---

### Q6: How does checkpointing work in LangGraph, and what does it enable?

**Answer:**
A **Checkpointer** is a persistence layer that saves the state of the graph after every node execution. Each saved snapshot is a **checkpoint**, indexed by a `thread_id` (one thread per conversation/session).

```python
from langgraph.checkpoint.postgres import PostgresSaver
graph = builder.compile(checkpointer=PostgresSaver(conn))

config = {"configurable": {"thread_id": "user_42"}}
graph.invoke(input, config)
```

**What it enables:**

1. **Memory across invocations**: Re-invoke with the same `thread_id` and the previous state is loaded — built-in long-term memory for chatbots.

2. **Time travel**: Replay history. `graph.get_state_history(config)` returns all past checkpoints; you can resume from any one to explore alternate timelines.

3. **Human-in-the-loop**: Combined with `interrupt_before` or `interrupt_after`, execution pauses at chosen nodes. A human can inspect state, modify it via `graph.update_state()`, then resume by invoking with `None` as input.

4. **Fault tolerance**: If a node crashes, state up to the last checkpoint is preserved — you can resume after fixing the bug.

5. **Streaming UX patterns**: Long-running agents can persist progress and the frontend can poll state.

**Available implementations**: `MemorySaver` (in-memory, dev only), `SqliteSaver`, `PostgresSaver`, `MongoDBSaver`, plus async versions. For production, always use a durable backend.

---

### Q7: How do you implement Human-in-the-Loop (HITL) workflows in LangGraph?

**Answer:**
LangGraph supports HITL through three mechanisms:

**1. Static interrupts** — declared at compile time:
```python
graph = builder.compile(
    checkpointer=checkpointer,
    interrupt_before=["execute_payment"],  # pause before this node
    interrupt_after=["draft_email"]         # pause after this node
)
```
Execution stops, returns control to the application, and waits.

**2. Dynamic interrupts** — `interrupt()` function (modern API, v0.2+):
```python
from langgraph.types import interrupt, Command

def approval_node(state):
    decision = interrupt({"draft": state["draft"]})  # pauses here
    return {"approved": decision}

# Resume from the application:
graph.invoke(Command(resume="approved"), config)
```
This is more flexible — you can interrupt conditionally, mid-node.

**3. State editing** — between interrupts, you can modify state directly:
```python
graph.update_state(config, {"draft": edited_version})
graph.invoke(None, config)  # continue
```

**Typical HITL patterns:**
- **Approval**: Pause before destructive actions (sending email, executing trade).
- **Editing**: Let humans edit LLM outputs before they're used downstream.
- **Tool-call review**: Pause before tool execution so a human can vet what the agent is about to do.
- **Multi-turn elicitation**: Pause to ask the user for clarification mid-workflow.

Checkpointing is **required** for HITL — without it, there's nowhere to durably pause.

---

### Q8: What are conditional edges in LangGraph, and how do you handle multiple possible paths?

**Answer:**
A **conditional edge** is an edge where the destination is determined dynamically at runtime by a routing function that inspects the current state.

```python
def route(state: State) -> str:
    if state["sentiment"] == "negative":
        return "escalate_to_human"
    elif len(state["messages"]) > 10:
        return "summarize"
    else:
        return "respond"

graph.add_conditional_edges(
    "classifier",
    route,
    {
        "escalate_to_human": "human_node",
        "summarize": "summary_node",
        "respond": "response_node",
    }  # optional mapping from return value → node name
)
```

**Key points:**

1. The routing function returns either a node name (string) or `END`. With the optional mapping dict, it can return arbitrary keys mapped to nodes.

2. **Multiple parallel paths**: A routing function can return a **list** of node names — they execute in parallel. The graph then waits for all branches before continuing (a "fan-out / fan-in" pattern).

```python
def route(state) -> list[str]:
    return ["search_web", "search_docs", "search_db"]  # all run in parallel
```

3. **Use the `Send` API** for dynamic parallelization where each branch gets different state:
```python
from langgraph.types import Send
def route(state):
    return [Send("worker", {"item": x}) for x in state["items"]]
```
This is how you implement **map-reduce** patterns over a dynamic number of items.

4. **Common patterns powered by conditional edges**:
   - Agent loop (continue if tool calls exist, else END)
   - Validation loop (retry if output fails check)
   - Router pattern (classify intent → dispatch to specialist sub-graph)
   - Self-correction (critic node → revise or accept)

---

### Q9: What memory strategies are available in LangChain/LangGraph, and what are their tradeoffs?

**Answer:**
Memory means giving the LLM context about prior turns. Several strategies exist:

| Strategy | How it works | Pros | Cons |
|---|---|---|---|
| **Buffer Memory** | Keep entire conversation in prompt | Lossless, simple | Hits context limit; expensive |
| **Window Memory** | Keep only last N messages | Cheap, simple | Forgets older context |
| **Token-Limited Buffer** | Keep messages until token threshold | Predictable cost | Same forgetting issue |
| **Summary Memory** | Periodically summarize old turns into a running summary | Compresses arbitrarily long histories | Lossy; summary errors compound; extra LLM calls |
| **Summary Buffer Hybrid** | Recent N messages verbatim + summary of older | Best of both worlds | Implementation complexity |
| **Vector Store Memory** | Embed past messages, retrieve relevant ones per turn | Scales to unbounded history | Retrieval misses; loses temporal flow |
| **Entity Memory** | Track facts about entities (people, places) in structured form | Persistent semantic knowledge | Requires extraction logic |

**In modern LangChain/LangGraph:**
- The legacy `ConversationBufferMemory` etc. classes are deprecated.
- The recommended pattern in LangChain is `RunnableWithMessageHistory`.
- In LangGraph, **memory is just state managed by the checkpointer**. You decide what to keep in `state["messages"]` — you can write a node that trims, summarizes, or rewrites messages before they go to the LLM.
- **Short-term memory** = thread state in checkpointer; **long-term memory** = the LangGraph **Store** API (key-value with optional vector search, queryable across threads).

**Production tip**: Often the right pattern is **summary + last-N-turns + retrieval over an episodic memory store** — combining all three strategies.

---

### Q10: What are the main limitations and criticisms of LangChain, and how does LangGraph address them?

**Answer:**
LangChain has been heavily criticized in the community. The main complaints:

**1. Over-abstraction**
*Critique*: Layers of wrappers (LLMChain wrapping a prompt wrapping a template wrapping a string) obscure what's actually happening. Debugging requires diving through call stacks.
*LangGraph's answer*: Nodes are just functions. State is just a dict. You write plain Python; the graph only handles orchestration.

**2. Leaky abstractions**
*Critique*: Once you need behavior outside the prescribed pattern, the abstractions fight you. Custom logic forces you to subclass or patch internals.
*LangGraph's answer*: Conditional edges and custom node functions cover essentially any control flow — no inheritance needed.

**3. Hidden control flow in `AgentExecutor`**
*Critique*: The agent loop is a black box. You can't easily add custom termination, retries, or per-step logic.
*LangGraph's answer*: The loop is **explicit** as a graph. Every step is a node you control. Add a "validator" node, a "guardrail" node, retries — trivially.

**4. State management was an afterthought**
*Critique*: Memory and intermediate state in LangChain were bolted on with different abstractions for different cases.
*LangGraph's answer*: First-class typed state with reducers. One unified mental model.

**5. Production concerns**
*Critique*: Hard to checkpoint, hard to make resumable, hard to do streaming-of-intermediate-steps, hard to debug at scale.
*LangGraph's answer*: Built-in checkpointers (Postgres, SQLite), built-in streaming modes (`values`, `updates`, `messages`, `custom`), native LangSmith integration, durable execution.

**6. Rapidly changing API**
*Critique*: Code written 6 months ago often breaks. Multiple parallel ways to do the same thing (legacy chains vs LCEL vs LangGraph).
*LangGraph's answer*: Stable, smaller surface area. Still a real concern overall — pin versions in production.

**7. Bloated dependencies**
*Critique*: `langchain` historically pulled in huge transitive deps.
*Both addressed by*: The split into `langchain-core`, `langchain-openai`, `langchain-community`, etc. Install only what you need.

**What LangGraph does NOT solve:**
- LLM hallucinations and reliability — that's a model problem.
- Cost — agents are expensive by nature.
- The fundamental fact that many LLM apps don't need a framework at all; sometimes direct API calls + a few utility functions are clearer.

**Mature take**: Use LangChain components selectively (loaders, retrievers, tool integrations are genuinely useful), use LangGraph for orchestration when complexity warrants it, and don't be afraid to drop down to raw SDK calls for simple cases.
