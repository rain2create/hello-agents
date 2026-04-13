# MCP（Model Context Protocol）简明指南

MCP 是 Anthropic 提出的开放协议，目标是**统一 Agent 与外部工具/资源的通信方式 ！！！**。你可以把它理解为 Agent 世界的 "USB-C"：一个接口，通吃各类服务。

---

## 核心架构

MCP 采用三层架构：

| 层级 | 角色 | 职责 |
|------|------|------|
| **Host** | 应用入口（如 Claude Desktop）| 接收用户输入、管理对话流 |
| **Client** | 内置在 Host 中的 MCP 客户端 | 连接 Server、转发请求、接收结果 |
| **Server** | 外部服务进程 | 执行具体操作（查文件、调 API、读数据库）|

**完整链路**：用户提问 → Host → 模型分析 → 需要工具 → MCP Client → MCP Server → 执行 → 返回结果 → 模型生成答案。

---

## 支持的传输方式

HelloAgents 的 `MCPClient` 支持 5 种传输模式：

| 传输方式 | 适用场景 | 示例 |
|---------|---------|------|
| **Memory** | 单元测试、快速原型 | 无需配置，直连内置演示服务器 |
| **Stdio** | 本地开发、Python/NPX 脚本 | `python my_mcp_server.py` |
| **HTTP** | 生产环境、远程微服务 | `http://api.example.com/mcp` |
| **SSE** | 实时通信、流式处理 | `http://localhost:8080/sse` |
| **StreamableHTTP** | 需要双向流式通信的 HTTP 场景 | `http://localhost:8080/mcp` |

---

## 直接使用 MCPClient，到对应的server里去找和调用工具！！！

```python
import asyncio
from hello_agents.protocols import MCPClient

async def demo():
    # Stdio 方式连接本地 MCP 服务     可以自己写脚本 自定义工具 然后用mcp通过stdio连接本地 （本地开发）
    client = MCPClient(["python", "my_mcp_server.py"])
    
    async with client:
        # 1. 发现工具
        tools = await client.list_tools()
        print([t['name'] for t in tools])
        
        # 2. 调用工具
        result = await client.call_tool("read_file", {"path": "README.md"})
        print(result)

asyncio.run(demo())
```

---

## 在 Agent 中使用（推荐）

HelloAgents 提供 `MCPTool` 包装器，可将 MCP Server 的**所有工具自动展开**为 Agent 可用的独立工具。

```python
from hello_agents import SimpleAgent, HelloAgentsLLM
from hello_agents.tools import MCPTool

agent = SimpleAgent(name="助手", llm=HelloAgentsLLM())

# 方式1：内置演示服务器（自动提供 add/subtract/multiply 等工具）  如果不给参数 那就是hello-agent包里定义的一个内置服务器，提供简单的工具来使用
mcp_tool = MCPTool()
agent.add_tool(mcp_tool)

# 方式2：连接外部 MCP 服务器（需指定唯一 name，防止工具名冲突）
fs_tool = MCPTool(
    name="fs",
    server_command=["npx", "-y", "@modelcontextprotocol/server-filesystem", "."]
)
agent.add_tool(fs_tool)

# Agent 会自动识别并调用
response = agent.run("请读取 my_README.md 并总结内容")
print(response)
```

> **自动展开机制**：`MCPTool(name="fs")` 会将 Server 提供的 `read_file`、`write_file` 等工具，自动注册为 `fs_read_file`、`fs_write_file`，Agent 可直接调用。

---

## 一句话总结

**MCP 让 Agent 接入外部服务变得像插 USB-C 一样简单，不再需要为每个 API 手写适配器。**
