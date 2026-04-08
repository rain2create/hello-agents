# LangGraph 框架简介

## 是什么？

LangGraph 是 LangChain 生态的扩展库，用**"图结构"**来编排智能体工作流。与对话驱动不同，它将任务流程建模为**节点（Nodes）**和**边（Edges）**，像画流程图一样构建智能体。

---

## 核心架构

```
┌─────────────────────────────────────────────────────────────────┐
│                     LangGraph 结构                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌───────────────────────────────────────────────────────┐     │
│   │                   State (全局状态)                     │     │
│   │     {messages: [], count: 0, result: ...}             │     │
│   └─────────────────────────┬─────────────────────────────┘     │
│                             │                                   │
│                             ▼                                   │
│                    ┌────────────────┐                           │
│                    │   Node 1       │                           │
│                    │  调用LLM思考    │                           │
│                    └───────┬────────┘                           │
│                            │                                    │
│                            ▼                                    │
│                     ┌──────────────┐                            │
│                     │   条件边      │                            │
│                     │  需要工具?    │                            │
│                     └──────┬───────┘                            │
│                            │                                    │
│              ┌─────────────┴─────────────┐                      │
│              │                           │                      │
│              ▼                           ▼                      │
│    ┌──────────────────┐      ┌──────────────────┐              │
│    │   Node 2         │      │   Node 3         │              │
│    │  执行工具         │      │  输出结果        │              │
│    │  (Tool Node)     │      │  (END)           │              │
│    └────────┬─────────┘      └──────────────────┘              │
│             │                                                   │
│             └──────────────────►┌──────────────────┐            │
│                                 │   回到 Node 1    │            │
│                                 │   继续思考       │            │
│                                 └──────────────────┘            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 三大核心要素

| 要素 | 作用 | 示例 |
|------|------|------|
| **State** | 全局状态，所有节点共享的数据 | `{"messages": [], "count": 0}` |
| **Node** | 执行具体任务的函数 | 调用LLM、执行工具、处理数据 |
| **Edge** | 连接节点，控制流程走向 | 普通边、条件边（if/else） |

---

## 快速上手示例

### 1. 安装

```bash
pip install langgraph langchain langchain-openai
```

### 2. 最简单的 ReAct 智能体

```python
from typing import TypedDict, List
from langgraph.graph import StateGraph, END
from langgraph.prebuilt import ToolNode
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool

# ========== 1. 定义状态 ==========
class AgentState(TypedDict):
    messages: List[dict]  # 对话历史

# ========== 2. 定义工具 ==========
@tool
def search(query: str) -> str:
    """搜索工具"""
    return f"搜索'{query}'的结果：LangGraph是一个智能体框架"

@tool  
def calculator(expression: str) -> str:
    """计算工具"""
    return str(eval(expression))

tools = [search, calculator]

# ========== 3. 定义节点 ==========
llm = ChatOpenAI(model="gpt-4o", api_key="your-key")
llm_with_tools = llm.bind_tools(tools)

def agent_node(state: AgentState):
    """智能体节点：调用LLM决定下一步"""
    messages = state["messages"]
    response = llm_with_tools.invoke(messages)
    return {"messages": messages + [response]}

def should_continue(state: AgentState) -> str:
    """条件边：判断是否继续调用工具"""
    last_message = state["messages"][-1]
    # 如果有工具调用，继续；否则结束
    if last_message.tool_calls:
        return "tools"
    return "end"

# ========== 4. 构建图 ==========
workflow = StateGraph(AgentState)

# 添加节点
workflow.add_node("agent", agent_node)
workflow.add_node("tools", ToolNode(tools))

# 添加边
workflow.set_entry_point("agent")  # 入口
workflow.add_conditional_edges(
    "agent",
    should_continue,           # 条件函数
    {"tools": "tools", "end": END}
)
workflow.add_edge("tools", "agent")  # 工具执行完回到agent

# 编译
app = workflow.compile()

# ========== 5. 运行 ==========
result = app.invoke({
    "messages": [{"role": "user", "content": "搜索LangGraph是什么，然后2+8等于几？"}]
})
print(result["messages"][-1].content)
```

---

## 流程可视化

```
┌─────────────────────────────────────────────────────────────────┐
│                      ReAct 工作流                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│     ┌─────────┐                                               │
│     │  开始   │                                               │
│     └────┬────┘                                               │
│          │                                                     │
│          ▼                                                     │
│   ┌────────────────┐                                           │
│   │   Agent 节点   │◄──────────────────────┐                   │
│   │  (LLM 思考)    │                       │                   │
│   └───────┬────────┘                       │                   │
│           │                                │                   │
│           ▼                                │                   │
│    ┌──────────────┐                        │                   │
│    │   需要工具?   │                        │                   │
│    └──────┬───────┘                        │                   │
│           │                                │                   │
│     ┌─────┴─────┐                          │                   │
│     │           │                          │                   │
│     ▼           ▼                          │                   │
│  ┌──────┐   ┌────────┐                     │                   │
│  │  是  │   │   否   │                     │                   │
│  └──┬───┘   └───┬────┘                     │                   │
│     │           │                          │                   │
│     ▼           ▼                          │                   │
│ ┌────────┐  ┌────────┐                     │                   │
│ │ Tool   │  │ 输出结果│                     │                   │
│ │ 节点   │  │  (END) │                     │                   │
│ └───┬────┘  └────────┘                     │                   │
│     │                                      │                   │
│     └──────────────────────────────────────┘                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 进阶示例：带循环的 Reflection

实现自我反思和优化：

```python
from typing import TypedDict

class ReflectionState(TypedDict):
    draft: str      # 草稿内容
    feedback: str   # 反思意见
    iterations: int # 迭代次数

def writer_node(state: ReflectionState):
    """撰写节点"""
    if state["iterations"] == 0:
        draft = "写一篇关于AI的文章初稿..."
    else:
        draft = f"根据反馈改进: {state['feedback']}"
    return {"draft": draft, "iterations": state["iterations"] + 1}

def reflector_node(state: ReflectionState):
    """反思节点"""
    feedback = f"审查草稿并提出改进建议..."
    return {"feedback": feedback}

def should_continue(state: ReflectionState) -> str:
    """判断是否继续优化"""
    if state["iterations"] < 3:
        return "reflect"
    return "end"

# 构建工作流
workflow = StateGraph(ReflectionState)
workflow.add_node("writer", writer_node)
workflow.add_node("reflector", reflector_node)

workflow.set_entry_point("writer")
workflow.add_conditional_edges(
    "writer",
    should_continue,
    {"reflect": "reflector", "end": END}
)
workflow.add_edge("reflector", "writer")

app = workflow.compile()
```

**流程图：**

```
┌─────────────────────────────────────────────────────────────────┐
│                   Reflection 迭代优化                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌─────────┐                                                  │
│   │  撰写   │─────► 生成初稿                                   │
│   │ Writer  │                                                  │
│   └────┬────┘                                                  │
│        │                                                        │
│        │ iterations < 3?                                        │
│        ▼                                                        │
│   ┌──────────┐                                                  │
│   │ 条件判断  │─────► 是 → 继续优化                             │
│   └────┬─────┘         否 → 结束                                │
│        │                                                        │
│        ▼                                                        │
│   ┌──────────┐                                                  │
│   │  反思    │─────► 提出改进意见                                │
│   │ Reflector│                                                  │
│   └────┬─────┘                                                  │
│        │                                                        │
│        └──────────────────────► 回到 Writer (迭代+1)            │
│                                                                 │
│   循环3次后:                                                    │
│   Writer → Reflector → Writer → Reflector → Writer → END       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 常用模式对比

```
普通 Chain (单向)                    LangGraph (可循环)
┌──────────────────────────┐       ┌──────────────────────────┐
│  输入 → A → B → C → 输出  │       │  输入 → A → B → 条件判断   │
│                          │       │            ↓             │
│  只能单向流动             │       │      是 ←┴─→ 否          │
│  不能回头                 │       │      ↓      ↓            │
│                          │       │      C      D            │
│                          │       │      └──────┘            │
│                          │       │           │              │
│                          │       │           ▼              │
│                          │       │      可以回到 A          │
└──────────────────────────┘       └──────────────────────────┘
```

| 特性 | 普通 Chain | LangGraph |
|------|-----------|-----------|
| 流程方向 | 单向线性 | 任意跳转 |
| 循环支持 | ❌ | ✅ |
| 条件分支 | 有限 | 灵活 |
| 状态管理 | 无 | 全局State |
| 适用场景 | 简单流水线 | 复杂智能体 |

---

## 关键概念总结

```
┌─────────────────────────────────────────┐
│          LangGraph 工作流程              │
├─────────────────────────────────────────┤
│                                         │
│  1. 定义 State → 确定需要传递的数据      │
│       │                                 │
│       ▼                                 │
│  2. 创建 Nodes → 每个节点是一个函数      │
│       │                                 │
│       ▼                                 │
│  3. 连接 Edges → 普通边 + 条件边         │
│       │                                 │
│       ▼                                 │
│  4. 编译 Graph → workflow.compile()     │
│       │                                 │
│       ▼                                 │
│  5. 执行调用   → app.invoke(state)      │
│                                         │
└─────────────────────────────────────────┘
```

---

## 优势与局限

**✅ 优势：**
- 流程可视化，逻辑清晰
- 原生支持循环和条件分支
- 状态集中管理，易于追踪
- 与 LangChain 生态无缝集成

**⚠️ 局限：**
- 需要预先设计好流程图
- 相比对话驱动，灵活性稍低
- 学习曲线略陡

---

## 官方资源

- GitHub: https://github.com/langchain-ai/langgraph
- 文档: https://langchain-ai.github.io/langgraph/
