# mem0 图记忆(Graph Memory)整理

## 一、背景:mem0、向量记忆与知识图谱

**mem0** 是给 AI agent 提供"长期记忆"的记忆层。它的核心做法是把对话里的事实抽取出来、做成向量嵌入存进**向量库**(vector store),检索时靠语义相似度找回相关记忆。

这种纯向量做法擅长"找出意思相近的内容",但有个固有弱点:它丢失了事实之间的**显式关系**。

mem0 历史上曾支持**图数据库**(graph database)来补这个短板。它和**知识图谱**(knowledge graph)的关系非常直接——本质上就是从对话中动态构建一个个性化的知识图谱:实体(Entity)抽取 + 关系(Relationship)抽取 + 三元组存储(Triple),正是知识图谱的标准构造方式。区别在于传统知识图谱往往是预先构建的静态领域知识,而 mem0 的图是随对话增量演化、按用户隔离的"记忆图谱"。底层可接 Neo4j、Memgraph、FalkorDB、Kuzu、Apache AGE、Neptune 等。

**两套存储的分工:**

- 向量库回答"语义上相近的内容是什么"
- 图库回答"实体之间是怎么关联的、谁连着谁"

---

## 二、核心概念与术语

### 向量嵌入(vector embedding,简称 embedding)

把一段文本交给**嵌入模型**,模型输出一个**定长的浮点数数组**。例如 OpenAI 的 `text-embedding-3-small` 输出 1536 维,就是 1536 个浮点数。

这个数组的意义在于:它把文本映射到高维空间里的一个点,而空间被训练成"语义相近的文本,点也靠得近"。判断两段文本意思像不像,就变成算两个向量的距离(通常用 cosine similarity)。

**在图节点里的具体实现:** 抽取出实体(如 "Alice")后,把实体文本喂给嵌入模型拿到数组,作为一个属性(property)挂在图节点上(节点除了 name,还多一个 embedding 属性)。主要用途是**实体消解**(entity resolution):下次又抽到 "Alice" 或 "Alice Chen" 时,不直接新建节点,而是把新实体的嵌入和已有节点的嵌入逐个算相似度,够近就认为是同一个人并合并,避免同一实体在图里裂成多个节点。

### 三元组(triple,也作 triplet)

知识图谱/RDF 的标准术语是 **triple**;mem0 文档和口语里也用 **triplet**,指同一个东西——即 `(主语, 谓语, 宾语)`(subject–predicate–object)结构。例:`(Alice, ALLERGIC_TO, TreeNuts)`。

### 实体(entity)

图里的节点。指人、公司、地点、偏好、项目等具体对象。

### 时序推理(temporal reasoning)

系统能处理"事实随时间变化"这件事:知道哪条信息是旧的、被哪条新信息取代了,从而既能反映现状,又能保留历史轨迹。

---

## 三、`enable_graph=True` 时的工作流程

**关键前提:图记忆是叠加在向量记忆之上的,不是替代关系。** 开启后,mem0 同时维护向量库和图库,`add()` 和 `search()` 每次都**并行**操作两边。

配置上需要在 config 里写一个 `graph_store` 块(指定 provider 和连接信息),再把 `enable_graph` 设为 `True`。`add()` 里先做 `_add_to_vector_store`,如果 `enable_graph` 为真再做 `_add_to_graph_store`,最后返回 {"results": 向量结果, "relations": 图结果}<br>
e.g. 
```python
add("Hi, I'm Alice. I'm allergic to tree nuts and I work at TechCorp.", user_id="alice")
```
开了 `enable_graph=True`<br>

Response:
```python
{
  "results": [
    {
      "id": "0b3e2f1a-7c44-4d9e-9a21-1f6b8c2d4e55",
      "memory": "Name is Alice",
      "event": "ADD"
    },
    {
      "id": "9d7c1a83-2b6e-4f50-8a13-3e9f7c1b2a64",
      "memory": "Allergic to tree nuts",
      "event": "ADD"
    },
    {
      "id": "4f8a6d22-1e3c-49b7-bb05-7c2d9e6f1a38",
      "memory": "Works at TechCorp",
      "event": "ADD"
    }
  ],
  "relations": {
    "deleted_entities": [],
    "added_entities": [
      [
        {
          "source": "alice",
          "relationship": "allergic_to",
          "destination": "tree_nuts"
        }
      ],
      [
        {
          "source": "alice",
          "relationship": "works_at",
          "destination": "techcorp"
        }
      ]
    ]
  }
}
```

### 写入阶段(add)

向量侧走常规流程(见第四节)。图侧(`_add_to_graph_store`)是独立管线:

1. **实体与关系抽取 (Entity & Relationship extraction)** —— 用一个 LLM(配合专门 prompt / function calling)从对话抽出实体和关系,产出一批三元组。
2. **实体节点处理** —— 每个实体成为一个节点,节点带有向量嵌入(vector embedding)作为属性(property)。这一步包含实体消解(entity resolution),靠嵌入相似度判断是否为已有实体。
3. **冲突检测与更新解析** —— 新关系不直接插入。整合新信息时检查它与现有关系是否冲突;若冲突,一个基于 LLM 的更新解析器(update resolver)判断旧关系是否应被标记为过时。
4. **时序式写入** —— 处理冲突的方式不是删除,而是把过时关系标记为被取代。这样既反映现状又保留历史(即时序推理 temporal reasoning)。e.g. Alice 原来是素食者,后来不是了,旧的 `IS_VEGAN` 边会被标记为被取代/失效,而不是物理删掉。
5. **写入图库** —— 新三元组作为节点和边落盘。

### 检索阶段(search)

向量侧:嵌入 query,算 cosine 相似度,取 top-k。

- 你这一侧 —— 只传自然语言。search("programming preferences", user_id="user_123"),你给的是普通字符串,从头到尾不碰 Cypher。
- LLM 这一侧 —— 负责的是语义部分,不是查询语言。它从你的自然语言 query 里抽取实体(entity extraction),把 "programming preferences" 这种话变成一组要去图里找的实体节点。LLM 的产物是"实体列表 + 它们的嵌入",不是一段 Cypher。
- mem0 这一侧 —— Cypher 是写死在 mem0 源码里的,具体在 mem0/graph_stores/neo4j.py 这个模块。它是固定的查询模板,运行时只把参数(实体名、嵌入向量、user_id 过滤条件等)填进去。从一个 GitHub issue 能直接看到这个结构——有人报告 Neo4j 后端按 user_id 过滤搜不到结果,定位到的原因就是 mem0/graph_stores/neo4j.py 里的图搜索查询在构建 Cypher 时没有正确地传递或应用 user_id 过滤约束。"构建 Cypher"这个动作发生在 mem0 的代码里,印证了模板是 mem0 内置的。

那这个 hardcode 的 Cypher 大致干两件事(mem0 用的是结合向量相似度搜索与直接图遍历的混合检索方式):
- 节点相似度匹配 —— 拿 query/实体的嵌入,跟图里节点上作为属性存的嵌入算相似度,找出最相关的起始节点。
- 关系遍历 —— 从这些节点出发,用 MATCH (n)-[r]->(m) 这类固定 pattern 把连着的关系捞出来。

Cypher源码：
- 检索(实体中心遍历): https://github.com/mem0ai/mem0/blob/v1.0.11/mem0/memory/graph_memory.py#L309-L342
- 写入(MERGE 节点+关系): https://github.com/mem0ai/mem0/blob/v1.0.11/mem0/memory/graph_memory.py#L613-L641
- 软删除(标记 valid=false): https://github.com/mem0ai/mem0/blob/v1.0.11/mem0/memory/graph_memory.py#L422-L434

图侧用两种互补策略:

- **实体中心法(entity-centric)** —— 聚焦查询里的关键实体,在图中定位对应节点,然后遍历这些节点连接的关系。
- **语义三元组法(semantic triplet)** —— 把整个查询编码成稠密嵌入(dense embedding),拿去和图里的关系三元组做匹配。

两侧结果汇总后一起返回,`relations` 字段暴露图遍历出的关系。

---

## 四、图记忆 vs 纯向量库:价值与代价

### 先澄清一个常见误解:图里的节点是 entity,不是 memory

图**不是一张 memory 之间互相连的图**。节点是**实体**,边是**实体与实体之间的关系**,这些关系从一条条 memory 里抽出来。

例:`(Alice)-[ALLERGIC_TO]->(TreeNuts)`、`(Alice)-[WORKS_AT]->(TechCorp)`、`(Alice)-[COLLEAGUE_OF]->(Bob)`、`(Bob)-[LIVES_IN]->(Tokyo)`。十次对话里关于 Alice 的零散信息,全都挂到同一个 Alice 节点上。

> 注意:这与 GraphRAG 把文档切 chunk、用 next/parent 关系连 chunk 是**两条不同的技术路线**。mem0 做的是实体知识图谱,与文档结构无关;而且 mem0 本来就不 chunk,它抽的是原子事实,每条 memory 已是最小事实单元。

### 图相对纯向量库,真正的帮助(主要是检索的"正确性 / 完整性completeness",不是速度)

1. **跨 memory 的聚合** —— 纯向量库按相似度返回 top-k 条离散 memory。关于某实体的事实若散落在大量对话中、措辞各异,向量检索可能漏掉用了不同说法的那条。图不靠措辞,直接收集挂在该节点上的所有边,给出完整画像(这是 recall 召回完整性问题)。

2. **多跳的关系查询(join)** —— 深的多跳(5、6 跳)确实少,但 **2 跳极常见且够用**。<br>
    e.g. :"我能带同事去吃哪家餐厅?" 需要 <br>
    `(我)-[同事]->(Bob)` → `(Bob)-[过敏]->(seafood)` → 再过滤餐厅。<br>
    纯向量库做不了这种 join,除非库里恰好存了拼好的整条记录。

3. **结构化的冲突检测与时序处理** —— 见下一节。

### 矛盾消解:向量侧 vs 图侧的准确对比

**纠正一个易犯的错误说法:** 对于mem0，不能说"向量库无法处理矛盾、矛盾事实都堆在库里互相污染"。

因为 v3 之前的 `add()` **本来就有 resolution 步骤**:抽取候选事实后,检索出相似的已有 memory,再让 LLM 逐条判定 ADD / UPDATE / DELETE / NOOP。所以"Alice 不再吃素"进来时,"Alice 吃素"那条会被 UPDATE(改写并重新嵌入)或 DELETE。**向量侧本来就有矛盾消解机制。**

那图在时序这件事上真正的增量价值，准确说是两点:

- **冲突检测更结构化、更可靠** —— 向量侧的 resolution 是"尽力而为、受检索门控的":UPDATE/DELETE 能否触发,取决于旧 memory 在 add 时有没有被召回进 top-k 上下文;若没召回,消解就不发生。而图的冲突检测在**带类型的关系**上做——"同一主语 + 同一关系类型 `DIET`,宾语不同"是一个清晰得多、不那么依赖召回运气的冲突信号。

- **保留版本历史** —— 这是更本质的区别。向量侧的 UPDATE/DELETE 是"覆盖/销毁",消解后库里只剩当前状态,"Alice 曾经吃素"这个历史没了。图不删旧边,只标记为被取代。所以严格意义的时序推理("Alice 的饮食以前是什么"、"变化何时发生")只有图答得了;向量侧消解后只能查"最新状态",查不了"历史轨迹"。

> 一个反讽点:v3 把 UPDATE/DELETE 这一步整个砍了,改成 ADD-only。所以 **v3 的向量路径确实会让矛盾事实堆积**,然后靠检索时排序(让最新最相关的浮上来)兜底。"矛盾堆在库里"用来描述 v3 是对的,用来描述 `enable_graph` 那个与 UPDATE/DELETE 共存的时代则是错的。

### 代价(overhead)

- 每次 `add()` 开图,要额外的 LLM 调用做实体抽取、关系抽取、冲突检测
- 每次 `search()` 也要额外 LLM 调用从 query 抽实体 + 做图遍历
- 需要部署、运维一个独立的图数据库
- 实现还有粗糙处(如曾有 bug:删 memory 时图里留下孤儿节点,向量库清了图库没清)

### 结论:大多数场景下,图的收益抵不过代价

最有力的证据是 **mem0 团队自己把图功能砍掉了**。他们用一个轻得多的方案(entity linking,见第五节)拿到了图的大部分实际收益,而新算法在基准测试上的分数反而大幅上涨——且这些分数是在没有图的情况下拿到的。

**什么时候图仍真的值得:** 当应用核心就是关系 join——比如 CRM agent 要天天回答"跟这个客户有业务往来、风险画像相似的其他客户有哪些",这种查询本质是图遍历,实体加权那种"软提升"替代不了。若只是要个"记住用户偏好、跨会话别忘事"的记忆层,纯向量库 + 实体加权基本够,图属于过度工程。

---

## 五、v2.0.0 更新:图存储被移除

### 发布时间
https://docs.mem0.ai/migration/oss-v2-to-v3#graph-memory-%E2%86%92-entity-linking

图存储移除发生在 **mem0 v2.0.0** 大版本(Python SDK 标记为 v2.0.0,迁移文档 URL 为 `oss-v2-to-v3`)。具体日期约在 **2026 年 3 月下旬**,无法精确到某一天——精确时间戳建议直接看 GitHub 上 `mem0ai/mem0` 的 v2.0.0 release 页面。后续有 v2.0.2、TypeScript 端 ts-v3.0.3 等小版本跟进。

### 被移除的内容

- `enable_graph` / `enableGraph` 配置标志
- `graph_store` / `graphStore` 配置块(Neo4j、Memgraph、Kuzu、Apache AGE、Neptune)
- 全部图记忆代码路径(约 4000 行)
- 搜索结果里的 `relations` 字段不再有数据

### 替代方案:实体链接(entity linking)

新机制仍然抽取实体,但**不建独立图、不做图遍历**:在 `add()` 阶段从每条记忆抽取实体(专有名词、引号内文本、复合名词短语),存进一个与现有向量库平行的集合 `{collection}_entities`;检索时把查询中的实体与该集合匹配,作为一个**加权信号**提升相关记忆的排名。

差别在于:实体关系现在通过检索排序**间接**被利用,不再暴露成可查询的图结构。如果原应用依赖直接遍历图关系,需要重新设计这部分。

### 同期的其他变化(v2.0.0 整体重设计)

- **抽取**:单趟 ADD-only(一次 LLM 调用,无 UPDATE/DELETE),延迟约减半
- **检索**:多信号混合检索(语义 + BM25 关键词 + 实体匹配,融合成一个分数)
- **SDK 清理**:接口与 Platform API 对齐

> 注意:被移除的是**开源 SDK** 里的图功能。mem0 的托管平台(Platform)情况可能不同——若使用托管服务,建议单独确认图记忆目前是否仍在、以何种形式存在。
