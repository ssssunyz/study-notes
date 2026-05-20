# MCP 中的 Meta-Tool 概念详解

---

## 一、问题背景

### 传统方式（不用 MCP）

```python
# ❌ 所有 tools 都要传给 Claude API
response = client.messages.create(
    model="claude-3-5-sonnet-20241022",
    tools=[
        search_user_memory,      # 150 tokens
        add_user_memory,         # 150 tokens
        read_inventory,          # 120 tokens
        update_inventory,        # 150 tokens
        read_rating,             # 120 tokens
        update_rating,           # 150 tokens
        # ... 如果有 50 个工具
        tool_7, tool_8, ..., tool_50
    ],  # ← 整个 tools 列表占用 5000+ tokens
    messages=messages
)

# 问题：
# - Tools 定义很长
# - 占用很多 context
# - 每次都要发送完整列表
```

---

## 二、Meta-Tool 方式（使用 MCP）

### 核心思想

**不是把所有 tools 传给 Claude，而是：**
1. 只传 **1 个虚拟工具** 给 Claude（叫 meta-tool）
2. 这个工具的作用是"调用 MCP Server"
3. **真正的 tools 都在 MCP Server 中**

### 可视化对比

```
┌─────────────────────────────────────────────┐
│ 传统方式（不用MCP）                          │
├─────────────────────────────────────────────┤
│                                             │
│  Claude API 需要知道：                      │
│  ├─ search_user_memory                      │
│  ├─ add_user_memory                         │
│  ├─ read_inventory                          │
│  ├─ update_inventory                        │
│  ├─ read_rating                             │
│  ├─ update_rating                           │
│  ├─ ... (50 个工具)                        │
│  └─ ... 占用 5000+ tokens ❌               │
│                                             │
└─────────────────────────────────────────────┘


┌─────────────────────────────────────────────┐
│ MCP 方式（使用Meta-Tool）                    │
├─────────────────────────────────────────────┤
│                                             │
│  Claude API 只需要知道：                    │
│  └─ call_mcp(tool_name, arguments)         │
│     └─ 占用仅 200 tokens ✅                │
│                                             │
│  MCP Server 知道：                          │
│  ├─ search_user_memory                     │
│  ├─ add_user_memory                        │
│  ├─ read_inventory                         │
│  ├─ update_inventory                       │
│  ├─ read_rating                            │
│  ├─ update_rating                          │
│  ├─ ... (50 个工具)                        │
│  └─ ... (存在Server中，不占用Claude context) │
│                                             │
└─────────────────────────────────────────────┘
```

---

## 三、实现细节

### 步骤 1：定义 Meta-Tool

```python
# ✅ 传给 Claude 的工具定义（只有 1 个）
meta_tool = {
    "name": "call_mcp",
    "description": "Call a tool from the MCP Server",
    "input_schema": {
        "type": "object",
        "properties": {
            "tool_name": {
                "type": "string",
                "description": "The name of the tool to call (e.g., 'search_user_memory', 'read_inventory')"
            },
            "arguments": {
                "type": "object",
                "description": "The arguments to pass to the tool"
            }
        },
        "required": ["tool_name", "arguments"]
    }
}

# 这个 meta_tool 只占用 ~ 200 tokens！
```

### 步骤 2：在 Claude API 中使用

```python
from anthropic import Anthropic

client = Anthropic()

response = client.messages.create(
    model="claude-3-5-sonnet-20241022",
    max_tokens=1024,
    tools=[meta_tool],  # ← 只传 1 个工具！
    messages=[
        {"role": "user", "content": "帮我查一下北京天气，然后发邮件给friend@example.com"}
    ]
)

print(response.stop_reason)  # "tool_use"
print(response.content[0].name)  # "call_mcp"
print(response.content[0].input)
# {
#   "tool_name": "search_user_memory",
#   "arguments": {"query": "dietary preferences"}
# }
```

### 步骤 3：MCP Server 维护所有真实工具

```python
# mcp_server.py
class MCPServer:
    def __init__(self):
        # ✅ 所有真实工具都在这里
        self.tools = {
            "search_user_memory": self.search_user_memory,
            "add_user_memory": self.add_user_memory,
            "read_inventory": self.read_inventory,
            "update_inventory": self.update_inventory,
            "read_rating": self.read_rating,
            "update_rating": self.update_rating,
            # ... 可以有 50 个、100 个、1000 个工具！
        }
    
    def search_user_memory(self, query):
        """查询用户偏好"""
        return f"Found preferences: {query}"
    
    def add_user_memory(self, key, value):
        """添加用户偏好"""
        return f"Saved: {key} = {value}"
    
    def read_inventory(self, food_type):
        """查询库存"""
        return f"Inventory: {food_type} available"
    
    # ... 其他工具
    
    def execute_tool(self, tool_name, arguments):
        """执行工具"""
        if tool_name in self.tools:
            return self.tools[tool_name](**arguments)
        return f"Tool {tool_name} not found"
```

### 步骤 4：Agent 处理 Meta-Tool 调用

```python
# agent.py
import json
import subprocess
import sys

# 启动 MCP Server
mcp_process = subprocess.Popen(
    [sys.executable, "mcp_server.py"],
    stdin=subprocess.PIPE,
    stdout=subprocess.PIPE,
    text=True
)

class MCPClient:
    def __init__(self, process):
        self.process = process
        self.request_id = 0
    
    def call_tool(self, tool_name, arguments):
        """通过 MCP 调用真实工具"""
        self.request_id += 1
        
        request = {
            "jsonrpc": "2.0",
            "id": self.request_id,
            "method": "tools/execute",  # MCP 协议
            "params": {
                "name": tool_name,
                "arguments": arguments
            }
        }
        
        # 发送请求到 MCP Server
        self.process.stdin.write(json.dumps(request) + "\n")
        self.process.stdin.flush()
        
        # 接收响应
        response = self.process.stdout.readline()
        return json.loads(response)

mcp_client = MCPClient(mcp_process)

# Agent 主循环
messages = [
    {"role": "user", "content": "帮我查一下北京天气"}
]

response = client.messages.create(
    model="claude-3-5-sonnet-20241022",
    max_tokens=1024,
    tools=[meta_tool],  # ← 只有 1 个 meta-tool
    messages=messages
)

# 处理 meta-tool 调用
while response.stop_reason == "tool_use":
    tool_use = response.content[0]
    
    # Claude 调用了 "call_mcp"
    if tool_use.name == "call_mcp":
        tool_name = tool_use.input["tool_name"]
        arguments = tool_use.input["arguments"]
        
        # 通过 MCP Client 调用真实工具
        print(f"Calling MCP tool: {tool_name} with args: {arguments}")
        mcp_result = mcp_client.call_tool(tool_name, arguments)
        
        # 继续对话
        messages.append({"role": "assistant", "content": response.content})
        messages.append({
            "role": "user",
            "content": [
                {
                    "type": "tool_result",
                    "tool_use_id": tool_use.id,
                    "content": json.dumps(mcp_result)
                }
            ]
        })
        
        response = client.messages.create(
            model="claude-3-5-sonnet-20241022",
            max_tokens=1024,
            tools=[meta_tool],  # ← 仍然只有 1 个
            messages=messages
        )

print(response.content[0].text)
```

---

## 四、执行流程详解

### 用户请求："帮我查北京天气，发邮件给friend@example.com"

```
┌──────────────────────────────────────────────────────────┐
│ Step 1: 用户发送请求                                      │
└──────────────────────────────────────────────────────────┘
  User: "帮我查北京天气，发邮件给friend@example.com"
         ↓
┌──────────────────────────────────────────────────────────┐
│ Step 2: Claude API 处理                                   │
│ (只知道有 1 个 meta-tool: call_mcp)                       │
└──────────────────────────────────────────────────────────┘
  Claude 思考：
  "我需要：
    1. 查用户偏好 → 调用 call_mcp(search_user_memory, ...)
    2. 查库存 → 调用 call_mcp(read_inventory, ...)
    3. 发邮件 → 调用 call_mcp(send_email, ...)"
  
  response = {
    "stop_reason": "tool_use",
    "content": [
      {
        "type": "tool_use",
        "id": "abc123",
        "name": "call_mcp",
        "input": {
          "tool_name": "search_user_memory",
          "arguments": {"query": "dietary preferences"}
        }
      }
    ]
  }
         ↓
┌──────────────────────────────────────────────────────────┐
│ Step 3: Agent 处理 meta-tool 调用                        │
└──────────────────────────────────────────────────────────┘
  Agent 看到：Claude 要调用 "call_mcp"
  - tool_name = "search_user_memory"
  - arguments = {"query": "dietary preferences"}
  
  Agent 转发给 MCP Server：
  MCPClient.call_tool("search_user_memory", {"query": "..."})
         ↓
┌──────────────────────────────────────────────────────────┐
│ Step 4: MCP Server 执行真实工具                          │
└──────────────────────────────────────────────────────────┘
  MCPServer.execute_tool("search_user_memory", {...})
  
  MCPServer 的 tools 字典中找到这个工具：
  self.tools["search_user_memory"]({...})
  
  返回结果：
  {
    "result": "Found: vegetarian, no dairy, allergic to seafood"
  }
         ↓
┌──────────────────────────────────────────────────────────┐
│ Step 5: Agent 把结果回送给 Claude                        │
└──────────────────────────────────────────────────────────┘
  messages.append({
    "role": "user",
    "content": [{
      "type": "tool_result",
      "tool_use_id": "abc123",
      "content": "Found: vegetarian, no dairy, allergic to seafood"
    }]
  })
         ↓
┌──────────────────────────────────────────────────────────┐
│ Step 6: Claude 继续推理                                   │
│ (需要调用第 2 个工具: read_inventory)                     │
└──────────────────────────────────────────────────────────┘
  Claude 又输出：
  {
    "stop_reason": "tool_use",
    "content": [{
      "type": "tool_use",
      "id": "def456",
      "name": "call_mcp",
      "input": {
        "tool_name": "read_inventory",
        "arguments": {"exclude": ["seafood"]}
      }
    }]
  }
         ↓
  ... (重复 Step 3-6) ...
         ↓
┌──────────────────────────────────────────────────────────┐
│ Step 7: Claude 输出最终回复                              │
└──────────────────────────────────────────────────────────┘
  {
    "stop_reason": "end_turn",
    "content": [{
      "type": "text",
      "text": "根据您的偏好，推荐以下菜品..."
    }]
  }
         ↓
  返回给用户
```

---

## 五、Context 占用对比

### 方案 A：传统方式（不用 MCP）

```python
# ❌ 所有 tools 定义传给 Claude
tools = [
    search_user_memory,      # 150 tokens
    add_user_memory,         # 150 tokens
    read_inventory,          # 120 tokens
    update_inventory,        # 150 tokens
    read_rating,             # 120 tokens
    update_rating,           # 150 tokens
    # ... 如果有 50 个
]

# Context 占用：
# ┌────────────────────────────┐
# │ System Prompt:   500       │
# │ All Tools:     5,000       │ ← 很多！
# │ Conversation: 194,500      │ ← 只剩这么多
# └────────────────────────────┘
# 对话进行到 50 轮时就接近 limit
```

### 方案 B：使用 Meta-Tool（使用 MCP）

```python
# ✅ 只有 1 个 meta-tool 传给 Claude
tools = [
    {
        "name": "call_mcp",
        "description": "Call a tool from MCP Server",
        "input_schema": {...}
    }
]

# Context 占用：
# ┌────────────────────────────┐
# │ System Prompt:    500      │
# │ Meta-Tool:        200      │ ← 很少！
# │ Conversation: 199,300      │ ← 充足！
# └────────────────────────────┘
# 对话可以进行 200+ 轮而不担心 context
```

### 数字对比

| 配置 | Tools 占用 | 对话 Tokens | 能对话的轮数 |
|------|-----------|-----------|-----------|
| 传统 6 tools | 790 | 198,210 | 100+ |
| 传统 50 tools | 5,000 | 194,500 | 20-30 |
| **MCP Meta-Tool** | **200** | **199,300** | **200+** |

---

## 六、为什么这样设计很聪明？

### 1️⃣ Context 节省

```
不用 MCP：Tools 定义占用 5000+ tokens
用 MCP：Tools 定义占用 200 tokens
节省：4800 tokens！这些都可以用于对话。
```

### 2️⃣ 工具无限扩展

```python
# MCP Server 中可以有无限个工具
class MCPServer:
    def __init__(self):
        self.tools = {
            "search_user_memory": ...,
            "add_user_memory": ...,
            "read_inventory": ...,
            # ... 50 个
            # ... 100 个
            # ... 1000 个
            # Claude 都不需要知道，context 不会增加！
        }
```

### 3️⃣ Claude 聚焦推理

```
不用 MCP：Claude 要读完 50 个 tools 定义，才能推理
        → 分心，降低推理质量

用 MCP：Claude 只需理解 1 个 meta-tool
      → 专注推理，质量更好
```

### 4️⃣ 灵活的工具管理

```python
# MCP Server 可以动态注册/注销工具，无需重新部署 Claude

class MCPServer:
    def register_tool(self, name, func):
        self.tools[name] = func
    
    def unregister_tool(self, name):
        del self.tools[name]

# Agent 代码完全不变！
# Claude 的 meta-tool 定义也不变！
```

---

## 七、为什么要用"Meta-Tool"这个名字？

```
"Meta" = 超越、更高层级

Meta-Tool 的工作：
  ← 代理层 →
  "我是中介，帮你调用真实工具"
  
类比：
  不用 Meta-Tool：Claude 直接打电话给 50 个人
  用 Meta-Tool：Claude 只打电话给 1 个秘书，秘书代理转接
```

---

## 八、MCP Server 响应格式

### MCP Server 如何告诉 Agent 有哪些工具？

```python
# Agent 可以先问 MCP Server：你有哪些工具？

class MCPServer:
    def list_tools(self):
        """列出所有可用工具"""
        return {
            "tools": [
                {
                    "name": "search_user_memory",
                    "description": "Search user dietary preferences",
                    "schema": {...}
                },
                {
                    "name": "add_user_memory",
                    "description": "Save user preference",
                    "schema": {...}
                },
                # ... 更多工具
            ]
        }

# Agent 可以用这个列表来告诉 Claude（通过 system prompt）
# "MCP Server 有以下工具可用：..." 
# 但这个信息只在 system prompt 中，不占用 tools 参数
```

---

## 九、完整代码示例

### 使用 MCP Meta-Tool 的完整实现

```python
# mcp_server.py
import json
import sys

class MCPServer:
    def __init__(self):
        self.tools = {
            "search_user_memory": self.search_user_memory,
            "add_user_memory": self.add_user_memory,
            "read_inventory": self.read_inventory,
            "update_inventory": self.update_inventory,
            "read_rating": self.read_rating,
            "update_rating": self.update_rating,
        }
    
    def search_user_memory(self, query):
        return f"Found user preference: {query}"
    
    def add_user_memory(self, key, value):
        return f"Saved {key}={value}"
    
    def read_inventory(self, food_type):
        return f"{food_type} is available"
    
    def update_inventory(self, food_type, quantity):
        return f"Updated {food_type} to {quantity}"
    
    def read_rating(self, food_type):
        return f"{food_type} rating: 4.5/5"
    
    def update_rating(self, food_type, rating):
        return f"Updated {food_type} rating to {rating}"
    
    def execute_tool(self, tool_name, arguments):
        if tool_name in self.tools:
            return self.tools[tool_name](**arguments)
        return f"Tool {tool_name} not found"
    
    def handle_request(self, request):
        method = request.get("method")
        
        if method == "execute_tool":
            tool_name = request["params"]["name"]
            arguments = request["params"]["arguments"]
            result = self.execute_tool(tool_name, arguments)
            
            return {
                "jsonrpc": "2.0",
                "id": request["id"],
                "result": {"content": result}
            }

def main():
    server = MCPServer()
    
    while True:
        try:
            line = sys.stdin.readline()
            if not line:
                break
            
            request = json.loads(line)
            response = server.handle_request(request)
            print(json.dumps(response))
            sys.stdout.flush()
        except:
            pass

if __name__ == "__main__":
    main()
```

```python
# agent.py
from anthropic import Anthropic
import json
import subprocess
import sys

client = Anthropic()

# 启动 MCP Server
mcp_process = subprocess.Popen(
    [sys.executable, "mcp_server.py"],
    stdin=subprocess.PIPE,
    stdout=subprocess.PIPE,
    text=True
)

class MCPClient:
    def __init__(self, process):
        self.process = process
        self.request_id = 0
    
    def execute_tool(self, tool_name, arguments):
        self.request_id += 1
        
        request = {
            "jsonrpc": "2.0",
            "id": self.request_id,
            "method": "execute_tool",
            "params": {
                "name": tool_name,
                "arguments": arguments
            }
        }
        
        self.process.stdin.write(json.dumps(request) + "\n")
        self.process.stdin.flush()
        
        response = self.process.stdout.readline()
        return json.loads(response)

mcp_client = MCPClient(mcp_process)

# ✅ 定义 Meta-Tool（只有 1 个！）
meta_tool = {
    "name": "call_mcp",
    "description": "Call a tool from the MCP Server. Available tools: search_user_memory, add_user_memory, read_inventory, update_inventory, read_rating, update_rating",
    "input_schema": {
        "type": "object",
        "properties": {
            "tool_name": {
                "type": "string",
                "description": "The name of the tool to call"
            },
            "arguments": {
                "type": "object",
                "description": "The arguments for the tool"
            }
        },
        "required": ["tool_name", "arguments"]
    }
}

# Agent 主循环
messages = [
    {
        "role": "user",
        "content": "帮我查一下有什么素食，然后告诉我评分"
    }
]

response = client.messages.create(
    model="claude-3-5-sonnet-20241022",
    max_tokens=1024,
    tools=[meta_tool],  # ← 只传 1 个工具
    messages=messages
)

# 处理 tool_use
while response.stop_reason == "tool_use":
    tool_use = response.content[0]
    
    tool_name = tool_use.input["tool_name"]
    arguments = tool_use.input["arguments"]
    
    print(f"🔧 Calling MCP tool: {tool_name} with {arguments}")
    
    # 调用 MCP Server
    mcp_response = mcp_client.execute_tool(tool_name, arguments)
    tool_result = mcp_response["result"]["content"]
    
    print(f"✅ Result: {tool_result}")
    
    # 继续对话
    messages.append({"role": "assistant", "content": response.content})
    messages.append({
        "role": "user",
        "content": [
            {
                "type": "tool_result",
                "tool_use_id": tool_use.id,
                "content": tool_result
            }
        ]
    })
    
    response = client.messages.create(
        model="claude-3-5-sonnet-20241022",
        max_tokens=1024,
        tools=[meta_tool],  # ← 仍然只有 1 个
        messages=messages
    )

# 输出最终结果
print("\n最终回复：")
print(response.content[0].text)

mcp_process.terminate()
```

---

## 十、总结

### Meta-Tool 概念

```
传统方式：
  Claude API 知道：[tool_1, tool_2, ..., tool_50]
              ↓
            执行

MCP Meta-Tool：
  Claude API 知道：[call_mcp]
              ↓
            执行
              ↓
         MCP Server (知道真实的 50 个工具)
              ↓
            执行真实工具
```

### 关键优势

| 维度 | 传统 | MCP |
|------|------|-----|
| **Context 占用** | 5000+ tokens | 200 tokens |
| **工具数量** | 受 context 限制 | 无限 |
| **Claude 负担** | 高（要理解所有工具） | 低（只理解 1 个工具） |
| **易维护** | 困难 | 容易 |

### 为什么是"Meta"

```
Meta-Tool 不是真实工具，而是"工具的代理"
它的作用：转发 Claude 的请求到 MCP Server
```

这就像：
```
不用 Meta-Tool：Claude 是公司，要管理 50 个员工
             → 工作量很大，context 占用很多

用 Meta-Tool：Claude 是公司老板，只管 1 个秘书
             → 秘书代理联系 50 个员工
             → 老板专心做决策
```
