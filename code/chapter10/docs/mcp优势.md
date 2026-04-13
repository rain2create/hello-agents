# MCP 相比传统 Tool Call 的优势

MCP 和 Function Calling（Tool Call）**不是竞争关系，而是互补**。Function Calling 是模型的大脑，负责"决定什么时候调用工具"；MCP 是工程层协议，负责**把这些工具统一、标准地接到模型上**。

但传统 Tool Call 在工程落地时，有**三个很痛的缺点**。下面先说痛点，再看 MCP 怎么解决。

---

## 传统 Tool Call 的三大痛点

### 1. 每接入一个新服务，就要重复写适配器

无论接 GitHub、接数据库、接文件系统，你都要自己写 HTTP 请求、处理异常、封装参数。来一个服务写一遍，来十个写十遍。

### 2. 不同模型的接口格式不一样，切换模型成本高

OpenAI 的 Tool Call 格式长这样：
```json
{
    "type": "function",
    "function": {"name": "xxx", "parameters": {...}}
}
```

Claude 的长这样：
```json
{
    "name": "xxx",
    "input_schema": {...}
}
```

模型返回的调用结果格式也不同。你要为**每个模型**写不同的解析代码，换模型等于重写一半工具层。

### 3. 工具无法复用，生态割裂

你写的 GitHub 适配器只能在你的项目里用，同事想用得拷过去改；社区里有人写了个更好的文件系统工具，你也得先读懂他的代码再手动集成。**没有统一标准，大家都在造互不兼容的轮子。**

---

## MCP 的三大优势

| 痛点 | MCP 的解法 |
|------|-------------|
| **重复写适配器** | 直接连接现成的 MCP Server，零代码接入新服务 |
| **模型格式不统一** | MCP 提供标准化中间层，**一次接入，OpenAI/Claude/Llama 通用** |
| **工具无法复用** | 社区 MCP Server 即插即用，你写的 Server 别人也能直接连 |

---

## 具体例子：搜索 GitHub + 读取本地文件

### 传统 Tool Call（痛苦版）

```python
import requests, json

# 1. 手动为每个服务写适配器
def search_github(query):
    resp = requests.get(
        "https://api.github.com/search/repositories",
        params={"q": query}
    )
    return resp.json()

def read_file(path):
    with open(path, "r", encoding="utf-8") as f:
        return f.read()

# 2. 还要为不同模型写不同格式的工具定义
openai_tools = [{
    "type": "function",
    "function": {"name": "search_github", "parameters": {...}}
}]

claude_tools = [{
    "name": "search_github",
    "input_schema": {...}  # 注意：不是 parameters
}]

# 3. 解析调用结果时，OpenAI 和 Claude 的返回结构也不一样
# ... 大量格式兼容的胶水代码
```

**问题总结**：
- 每新增一个外部服务，就要写一遍 HTTP + 错误处理 + 参数校验
- 换模型供应商时，工具定义和结果解析代码都要改
- 自己写的工具别人很难直接用，社区的也接不进来

---

### MCP 方式（简洁版） 直接连接到server 调server的工具

```python
from hello_agents.tools import MCPTool

# 直接连接两个社区 MCP Server，无需手写任何 API 调用
github_tool = MCPTool(
    name="gh",
    server_command=["npx", "-y", "@modelcontextprotocol/server-github"]
)
fs_tool = MCPTool(
    name="fs",
    server_command=["npx", "-y", "@modelcontextprotocol/server-filesystem", "."]
)

agent.add_tool(github_tool)
agent.add_tool(fs_tool)

# 一句话完事
agent.run("搜索 GitHub 上关于 AI agent 的项目，然后读取本地 README.md")
```

**好处总结**：
- ✅ 不用手写 GitHub API 和文件 IO 逻辑
- ✅ 不用管底层是 OpenAI 还是 Claude，MCP 帮你统一了格式
- ✅ 工具来自社区，开箱即用；你也可以把自己写的 MCP Server 分享给别人

---

## 什么时候选 MCP？

| 场景 | 建议 |
|------|------|
| Agent 需要频繁接入外部服务（文件、数据库、GitHub、Slack）| **用 MCP** |
| 团队/项目需要复用工具，或直接吃社区生态 | **用 MCP** |
| 需要灵活切换不同 LLM 供应商 | **用 MCP** |
| 只是几个固定工具的简单脚本 | Tool Call 也可以 |

---

## 一句话总结

**传统 Tool Call 的问题在于"每个服务都要手搓适配器，每个模型都要改格式"；MCP 的价值在于"统一接口、一次接入、处处复用"，让 Agent 扩展像插 USB 一样简单。**
