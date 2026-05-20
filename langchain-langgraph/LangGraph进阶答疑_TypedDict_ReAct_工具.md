# LangGraph 进阶答疑：TypedDict、Reducer、ReAct 可视化、create_react_agent、pipe operator、缺失工具

> 本文档回答 5 个问题：TypedDict 与 Reducer 是什么、TypedDict vs Pydantic、ReAct agent 的可视化图、`create_react_agent` 的具体实现、`|` operator 的英文叫法、以及当用户请求的工具未定义时 LLM 的行为。

---

## 1. TypedDict 和 Reducer 是什么？TypedDict vs Pydantic

### 1.1 TypedDict 是什么

`TypedDict` 是 Python 标准库 `typing` 模块里的东西（Python 3.8+），用来**给字典标注每个 key 的类型**。

```python
from typing import TypedDict

class State(TypedDict):
    messages: list
    user_id: str
```

**最关键的一点**：`TypedDict` 在运行时**就是一个普通的 dict**，它不做任何检查。

```python
s = State(messages=[], user_id="abc")
print(type(s))   # → <class 'dict'>  ← 它就是个 dict！
print(s)         # → {'messages': [], 'user_id': 'abc'}

# 你甚至可以放错类型，Python 运行时完全不报错：
bad = State(messages="这不是list", user_id=12345)  # ✅ 不报错！
```

`TypedDict` 的类型标注只有 **IDE 和静态类型检查工具（mypy、pyright）** 会读，运行时完全无视。它的作用是：写代码时 IDE 能自动补全、能提示你拼错了 key、能在 CI 里做静态检查。**零运行时开销**。

### 1.2 Reducer 是什么

Reducer 是 **LangGraph 特有的概念**。它回答一个问题：

> 当一个 node 返回 `{"messages": [新消息]}` 时，这个新值要怎么和 state 里**已有的**值合并？

```python
# 默认行为（没有 reducer）：覆盖
class State(TypedDict):
    count: int          # node 返回 {"count": 5} → 直接把旧值替换成 5

# 有 reducer：自定义合并逻辑
from typing import Annotated
from langgraph.graph.message import add_messages
import operator

class State(TypedDict):
    messages: Annotated[list, add_messages]   # 用 add_messages 合并 → 追加
    logs:     Annotated[list, operator.add]   # 用 operator.add 合并 → 列表拼接
    count:    int                             # 没标 reducer → 覆盖
```

`Annotated[类型, reducer函数]` 这个语法的意思是：「这个字段是 `list` 类型，更新它的时候用 `add_messages` 这个函数来合并」。

Reducer 本质就是一个函数，签名是 `(旧值, 新值) -> 合并后的值`：

```python
# operator.add 对 list 来说就是：
旧值 = [1, 2]
新值 = [3, 4]
operator.add(旧值, 新值)   # → [1, 2, 3, 4]   拼接

# 如果没有 reducer（默认覆盖），相当于：
def 默认reducer(旧值, 新值):
    return 新值   # 直接丢掉旧值
```

**为什么 agent 必须用 reducer**：在 ReAct 循环里，每个 node 只返回「自己新增的那条消息」。如果用默认的「覆盖」，第二轮 agent 一返回，整个对话历史就被一条消息覆盖没了。`add_messages` 保证每次都是**追加**。

### 1.3 TypedDict vs Pydantic

两者都能用来定义 LangGraph 的 State，区别很大：

| 维度 | TypedDict | Pydantic BaseModel |
|---|---|---|
| 本质 | 带类型标注的 dict | 一个类/对象 |
| 运行时验证 | ❌ 无，放错类型不报错 | ✅ 有，放错类型抛 `ValidationError` |
| 类型自动转换 | ❌ 无 | ✅ 有（如字符串 `"5"` 自动转成 int `5`） |
| 默认值 | 弱（靠 `total=False` / `NotRequired`） | ✅ 强，直接 `field: int = 0` |
| 自定义校验器 | ❌ 无 | ✅ 有（`@field_validator` 等） |
| 性能 | 最快（就是个 dict） | 较慢（每次都要验证） |
| 取值写法 | `s["messages"]` | `s.messages` |
| 序列化 | 普通 dict 操作 | 内置 `.model_dump()` / `.model_dump_json()` |

```python
# TypedDict —— 不验证
from typing import TypedDict
class StateA(TypedDict):
    age: int
StateA(age="不是数字")        # ✅ 静默通过，运行时不报错

# Pydantic —— 验证 + 转换
from pydantic import BaseModel
class StateB(BaseModel):
    age: int
StateB(age="不是数字")        # ❌ 抛 ValidationError
StateB(age="25")              # ✅ 自动转成 int 25
```

**怎么选**：LangGraph 里 `TypedDict` 是默认、最常用的选择，简单轻量。如果你的 state 需要**输入校验**（比如确保某字段一定是合法 email、数字在某个范围内），用 Pydantic。一句话：**TypedDict 是"约定"，Pydantic 是"强制"。**

---

## 2. Code 3.5.1（ReAct Agent）的可视化图

3.5.1 那个图编译出来长这样：

```
                    ┌─────────┐
                    │  START  │
                    └────┬────┘
                         │
                         ▼
                  ┌─────────────┐
          ┌──────▶│    agent    │   ← 调用 LLM 思考
          │       │  (LLM node) │
          │       └──────┬──────┘
          │              │
          │              ▼
          │      ◇ should_continue ◇   ← 条件边：检查 LLM 是否想调用工具
          │         ╱            ╲
          │   有tool_calls      没有tool_calls
          │       ╱                ╲
          │      ▼                  ▼
          │ ┌─────────┐        ┌─────────┐
          │ │  tools  │        │   END   │
          │ │ 执行工具 │        └─────────┘
          │ └────┬────┘
          │      │
          └──────┘
       tools → agent 的边
       构成循环（loop）
```

**执行轨迹举例**（用户问「北京和上海的天气和人口分别是多少？」）：

```
START
  │
  ▼
agent ────────▶ LLM 说："我要查 4 个东西" → 产生 4 个 tool_calls
  │
  ▼
should_continue ──▶ 有 tool_calls → 去 tools
  │
  ▼
tools ────────▶ 并发执行 get_weather×2 + get_population×2 → 4 条 ToolMessage
  │
  ▼ (回到 agent —— 这就是循环)
agent ────────▶ LLM 看到 4 个结果 → 这次直接给自然语言总结，无 tool_calls
  │
  ▼
should_continue ──▶ 没有 tool_calls → END
  │
  ▼
END
```

关键就是 `tools → agent` 这条边形成了**循环**：工具执行完不是结束，而是回去让 agent 再判断「信息够了吗？还要继续查吗？」

---

## 3. 如果用户问天气，但代码里没定义 `get_weather` tool，LLM 会怎么做？

先说结论：**LLM 不会、也不可能调用一个没绑定给它的工具。它只会用自然语言回答。**

### 3.1 原理

`llm.bind_tools(tools)` 这一步，只会把 `tools` 列表里那些工具的 **schema（名称、参数、描述）** 放进发给 OpenAI/Anthropic 的 API 请求里。LLM 在那次请求中**只能看到这些工具**。

如果 `tools` 里没有 `get_weather`，那么对 LLM 来说，`get_weather` 这个工具**根本不存在**。API 层面也不允许它返回一个未注册的工具调用。所以它**物理上无法**发起 `get_weather` 的 tool call。

### 3.2 实际会发生什么

假设你的 agent 只绑定了 `get_population`，用户却问天气：

```python
tools = [get_population]   # 注意：没有 get_weather
llm = ChatOpenAI(model="gpt-4o-mini").bind_tools(tools)

result = graph.invoke({"messages": [HumanMessage("北京今天天气怎么样？")]})
```

LLM 会发现「我手上的工具只有 `get_population`，跟天气无关」，于是它**不会产生任何 tool_calls**，直接返回一条普通文本 `AIMessage`。在 ReAct 图里：

```
START → agent → should_continue 检查到【没有 tool_calls】→ 直接 END
```

注意：**`tools` 节点根本不会被执行**，循环也不会发生。一轮就结束了。

LLM 返回的文本内容，通常是下面几种之一（取决于模型和 prompt）：

1. **诚实地说做不到**（gpt-4o 这类好模型的典型反应）：
   > "抱歉，我目前没有获取实时天气的能力。我可以帮你查询城市人口等信息。"

2. **用训练知识泛泛而谈**：
   > "北京春季气候通常温和……" —— 但这不是实时数据，可能过时。

3. **幻觉（hallucination）**——较差的模型或 prompt 不严谨时，可能**编造**一个看似具体的天气（"北京今天 22 度晴" 但其实是瞎说）。这是要警惕的。

### 3.3 工程上的应对

为了避免第 3 种「编造」，常见做法：

```python
# 方法 1：在 system prompt 里明确约束
system = """你只能使用提供的工具回答。
如果用户的问题需要你没有的工具（如实时天气），
就明确告诉用户你无法获取该信息，不要编造。"""

# 方法 2：补上缺失的工具（治本）
@tool
def get_weather(city: str) -> str:
    """获取指定城市的实时天气"""
    return call_real_weather_api(city)

tools = [get_population, get_weather]   # 把工具补全
```

### 3.4 一个边界情况

极少数情况下，LLM 可能"幻觉出"一个不存在的工具调用（比如它脑补出 `get_weather` 并尝试调用）。如果真发生了，`ToolNode` 找不到对应工具会**报错**。官方 `create_react_agent` 和 `ToolNode` 对此有容错处理（可以把错误作为 `ToolMessage` 返回给 agent，让它重新决策），但这属于异常路径，正常情况下不会触发。
