# RunnableWithMessageHistory 中 get_session_history 是如何获得 session_id 的？

## 问题

```python
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
```

---

## 回答

这是 Python 里非常重要的概念，叫 **callback（回调函数）** 或者 **higher-order function（高阶函数）**。

## 关键点：传函数 vs 调用函数

你看这一行：

```python
RunnableWithMessageHistory(
    chain,
    get_session_history,   # ← 注意这里没有括号！
    ...
)
```

**`get_session_history`** 和 **`get_session_history()`** 完全不一样：

| 写法 | 含义 |
|---|---|
| `get_session_history` | 函数本身（函数对象的引用，类似 C 里的函数指针） |
| `get_session_history("user_1")` | 调用函数，得到返回值 |
| `get_session_history()` | 调用函数（但参数不够会报错） |

这里传的是**函数本身**，不是函数的返回值。可以理解成：**你把这个函数"交给"了 `RunnableWithMessageHistory`，告诉它："以后你需要历史的时候，自己去调用我给你的这个函数"。**

## 实际执行流程

`session_id` 是在你**真正调用 chain 的时候**传进去的：

```python
chain_with_memory.invoke(
    {"input": "我叫小明"},
    config={"configurable": {"session_id": "user_1"}}    # ← session_id 在这里！
)
```

`RunnableWithMessageHistory` 内部大致是这样工作的（伪代码）：

```python
class RunnableWithMessageHistory:
    def __init__(self, chain, get_history_func, input_messages_key, history_messages_key):
        # 把函数引用存起来，备用
        self.chain = chain
        self.get_history_func = get_history_func   # ← 存的是函数本身
        self.input_key = input_messages_key
        self.history_key = history_messages_key
    
    def invoke(self, input_dict, config):
        # 第 1 步：从 config 里把 session_id 抠出来
        session_id = config["configurable"]["session_id"]
        
        # 第 2 步：现在才真正调用你给我的函数，把 session_id 喂进去
        history = self.get_history_func(session_id)   # ← 这里才是调用！
        # 等价于 history = get_session_history("user_1")
        
        # 第 3 步：把历史消息塞进 prompt 的占位符
        full_input = {
            self.input_key: input_dict[self.input_key],   # "input": "我叫小明"
            self.history_key: history.messages            # "history": [过去的消息...]
        }
        
        # 第 4 步：调用底层 chain
        response = self.chain.invoke(full_input)
        
        # 第 5 步：把这一轮的对话追加进历史
        history.add_user_message(input_dict[self.input_key])
        history.add_ai_message(response)
        
        return response
```

所以整个流程是：

```
你调用 invoke(input, config={"configurable": {"session_id": "user_1"}})
                                                ↓
RunnableWithMessageHistory 从 config 拿到 "user_1"
                                                ↓
内部调用 get_session_history("user_1")        ← 你定义的函数被回调
                                                ↓
get_session_history 查 store 字典，返回 history 对象
                                                ↓
RunnableWithMessageHistory 拿到 history，注入 prompt，调用 LLM
```

## 一个更生活化的类比

想象你去酒店前台办入住：

```python
def 查房卡(房间号):
    return 房卡数据库[房间号]

酒店系统 = 入住流程(
    查房卡,           # ← 你把"查房卡"这个流程交给酒店系统
    房间号key="room"  # ← 告诉系统：客人填表时房间号写在 "room" 那一栏
)

# 后来客人来了
酒店系统.办入住(
    客人信息={"room": "808", "name": "小明"},
    # 系统会自己从客人信息里抠出 "808"，然后调用 查房卡("808")
)
```

`查房卡` 函数你在写的时候并不知道客人房间号是几号——它接收一个参数 `房间号`。把它交给酒店系统时也没指定号码。**真正的房间号是客人入住时才提供的**，系统内部负责把房间号塞进 `查房卡()` 调用里。

## "session_id" 这个 key 是写死的吗？

不是，可以改。如果你想用 `user_id` 而不是 `session_id`，需要这样配置：

```python
from langchain_core.runnables import ConfigurableFieldSpec

chain_with_memory = RunnableWithMessageHistory(
    chain,
    get_session_history,
    input_messages_key="input",
    history_messages_key="history",
    history_factory_config=[
        ConfigurableFieldSpec(
            id="user_id",        # ← 用 user_id 代替 session_id
            annotation=str,
            name="User ID",
            default="",
            is_shared=True,
        ),
    ],
)

# 此时 get_session_history 要改成接收 user_id：
def get_session_history(user_id: str):
    ...

# 调用时：
chain_with_memory.invoke(
    {"input": "你好"},
    config={"configurable": {"user_id": "u_42"}}
)
```

甚至可以**同时**用多个 key（比如 `user_id` + `conversation_id`），这在多用户多对话的产品里非常实用。

## 总结一句话

`get_session_history` 是**你给 LangChain 的电话号码**——LangChain 不会马上打，但等需要的时候（也就是 `.invoke()` 被调用、`session_id` 已知时），它会拨这个号码、把 `session_id` 告诉你，让你返回对应的历史。这就是 callback pattern。
