# ContextBuilder 上下文构建器

ContextBuilder 是 HelloAgents 框架的上下文工程核心组件，通过 GSSC 流水线实现智能化的上下文管理。

---

## 核心架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                     ContextBuilder                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌────────────┐  │
│  │   Gather    │→│   Select    │→│  Structure  │→│  Compress  │  │
│  │   信息汇集   │  │   智能选择   │  │   结构化    │  │   压缩     │  │
│  │             │  │             │  │             │  │            │  │
│  │ • 系统指令   │  │ • 相关性评分 │  │ • 分区组织   │  │ • 分区压缩  │  │
│  │ • 记忆检索   │  │ • 新近性评分 │  │ • 模板生成   │  │ • 截断处理  │  │
│  │ • RAG检索   │  │ • 贪心选择   │  │             │  │            │  │
│  │ • 对话历史   │  │             │  │             │  │            │  │
│  │ • 自定义包   │  │             │  │             │  │            │  │
│  └─────────────┘  └─────────────┘  └─────────────┘  └────────────┘  │
│         ↑                                                           │
│    ┌────┴────┐                                                      │
│    │ 配置管理 │                                                      │
│    │ContextConfig                                                     │
│    │• max_tokens                                                      │
│    │• reserve_ratio                                                   │
│    │• recency_weight                                                  │
│    │• relevance_weight                                                │
│    └─────────┘                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## GSSC 流水线详解

### 完整流程图

```
┌──────────────────────────────────────────────────────────────────────┐
│                         GSSC 流水线                                   │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. GATHER 信息汇集                                                   │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │  多源信息收集（容错设计）                                        │  │
│  │                                                                │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │  │
│  │  │ 系统指令      │  │  记忆系统     │  │  RAG系统      │         │  │
│  │  │ (最高优先级)  │  │  MemoryTool  │  │  RAGTool     │         │  │
│  │  │              │  │  • search    │  │  • search    │         │  │
│  │  │ relevance:   │  │  • limit: 10 │  │  • limit: 5  │         │  │
│  │  │   1.0        │  │              │  │              │         │  │
│  │  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘         │  │
│  │         └─────────────────┼─────────────────┘                  │  │
│  │                           ↓                                    │  │
│  │  ┌──────────────┐  ┌──────────────┐                           │  │
│  │  │  对话历史     │  │  自定义包     │                           │  │
│  │  │  最近N条     │  │  ContextPacket│                           │  │
│  │  └──────────────┘  └──────────────┘                           │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                              ↓                                       │
│  2. SELECT 智能选择                                                   │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │  分离系统指令 → 计算综合分数 → 贪心选择                           │  │
│  │                                                                │  │
│  │  综合分数 = relevance_weight × 相关性 + recency_weight × 新近性  │  │
│  │                                                                │  │
│  │  相关性计算: 关键词重叠 / 向量相似度                              │  │
│  │  新近性计算: 指数衰减 (24小时内高分)                              │  │
│  │                                                                │  │
│  │  ┌─────────────────────────────────────────────────────┐       │  │
│  │  │  排序: 按 combined_score 降序                        │       │  │
│  │  │  选择: 贪心填充直到达到 token 预算                    │       │  │
│  │  │  过滤: 低于 min_relevance 阈值的不选                  │       │  │
│  │  └─────────────────────────────────────────────────────┘       │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                              ↓                                       │
│  3. STRUCTURE 结构化输出                                              │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │  按类型分组 → 构建模板 → 格式化输出                               │  │
│  │                                                                │  │
│  │  ┌──────────────────────────────────────────────────────────┐  │  │
│  │  │  [Role & Policies]                                       │  │  │
│  │  │  系统指令 / Agent角色定义                                 │  │  │
│  │  │                                                          │  │  │
│  │  │  [Task]                                                  │  │  │
│  │  │  用户当前查询                                            │  │  │
│  │  │                                                          │  │  │
│  │  │  [Evidence]                                              │  │  │
│  │  │  RAG检索结果 / 外部知识                                   │  │  │
│  │  │                                                          │  │  │
│  │  │  [Context]                                               │  │  │
│  │  │  对话历史 / 相关记忆                                      │  │  │
│  │  │                                                          │  │  │
│  │  │  [Output]                                                │  │  │
│  │  │  期望输出格式和要求                                       │  │  │
│  │  └──────────────────────────────────────────────────────────┘  │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                              ↓                                       │
│  4. COMPRESS 压缩处理                                                 │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │  IF 上下文超出 token 预算:                                       │  │
│  │    • 分区压缩（保持结构完整）                                     │  │
│  │    • 简单截断（保留高价值分区）                                   │  │
│  │    • 标记 [内容已压缩]                                           │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                              ↓                                       │
│                       返回优化后的上下文                               │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 核心数据结构

### ContextPacket 信息包

```python
from dataclasses import dataclass
from datetime import datetime

@dataclass
class ContextPacket:
    """候选信息包"""
    content: str              # 信息内容
    timestamp: datetime       # 时间戳
    token_count: int          # Token 数量
    relevance_score: float    # 相关性分数 (0.0-1.0)
    metadata: Dict[str, Any]  # 元数据 (type, priority, etc.)
```

### ContextConfig 配置

```python
@dataclass
class ContextConfig:
    """上下文构建配置"""
    max_tokens: int = 3000        # 最大 token 数量
    reserve_ratio: float = 0.2     # 系统指令预留比例
    min_relevance: float = 0.1     # 最低相关性阈值
    enable_compression: bool = True
    recency_weight: float = 0.3    # 新近性权重
    relevance_weight: float = 0.7  # 相关性权重
```

---

## 使用示例

### 基础用法

```python
from hello_agents.context import ContextBuilder, ContextConfig
from hello_agents.tools import MemoryTool, RAGTool

# 初始化
memory_tool = MemoryTool(user_id="user123")
rag_tool = RAGTool(knowledge_base_path="./kb")

config = ContextConfig(
    max_tokens=3000,
    reserve_ratio=0.2,
    min_relevance=0.2
)

builder = ContextBuilder(
    memory_tool=memory_tool,
    rag_tool=rag_tool,
    config=config
)

# 构建上下文
context = builder.build(
    user_query="如何优化Pandas内存占用？",
    conversation_history=history,
    system_instructions="你是一位资深的Python数据工程顾问..."
)
```

### 与 Agent 集成

```python
from hello_agents import SimpleAgent

class ContextAwareAgent(SimpleAgent):
    def __init__(self, name: str, llm, **kwargs):
        super().__init__(name=name, llm=llm)
        
        self.context_builder = ContextBuilder(
            memory_tool=MemoryTool(user_id=kwargs.get("user_id")),
            rag_tool=RAGTool(knowledge_base_path=kwargs.get("kb_path")),
            config=ContextConfig(max_tokens=4000)
        )
    
    def run(self, user_input: str) -> str:
        # 自动构建优化上下文
        optimized_context = self.context_builder.build(
            user_query=user_input,
            conversation_history=self.conversation_history,
            system_instructions=self.system_prompt
        )
        
        messages = [
            {"role": "system", "content": optimized_context},
            {"role": "user", "content": user_input}
        ]
        return self.llm.invoke(messages)
```

---

## 关键参数说明

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `max_tokens` | int | 3000 | 上下文最大 token 数量 |
| `reserve_ratio` | float | 0.2 | 为系统指令预留的比例 |
| `min_relevance` | float | 0.1 | 信息入选的最低相关性 |
| `recency_weight` | float | 0.3 | 新近性评分权重 |
| `relevance_weight` | float | 0.7 | 相关性评分权重 |

---

## 输出模板示例

```
================================================================================
[Role & Policies]
你是一位资深的Python数据工程顾问。你的回答需要:
1) 提供具体可行的建议
2) 解释技术原理
3) 给出代码示例

[Task]
如何优化Pandas的内存占用?

[Evidence]
Pandas内存优化的核心策略包括:
1. 使用合适的数据类型(如category代替object)
2. 分块读取大文件...
---
数据类型优化可以显著减少内存占用...

[Context]
user: 我正在开发一个数据分析工具
assistant: 很好!数据分析工具通常需要处理大量数据...
记忆: 用户正在开发数据分析工具,使用Python和Pandas

[Output]
请基于以上信息,提供准确、有据的回答。
================================================================================
```

---

## 工作流程图（简化版）

```
多源信息输入          评分排序              模板组装              输出上下文
┌─────────┐        ┌─────────┐          ┌─────────┐          ┌─────────┐
│ 系统指令 │        │         │          │[Role &  │          │         │
│ 记忆    │───────▶│ 综合评分 │─────────▶│ Task    │─────────▶│ 结构化  │
│ RAG结果 │        │ 贪心选择 │          │Evidence │          │ 上下文  │
│ 对话历史 │        │         │          │Context  │          │         │
└─────────┘        └─────────┘          │ Output] │          └─────────┘
                                        └─────────┘
```

---

## 关键要点总结

1. **GSSC 流水线**：Gather → Select → Structure → Compress

2. **评分机制**：综合分数 = 0.7×相关性 + 0.3×新近性

3. **系统指令保护**：预留 20% token 预算，确保不被挤占

4. **输出结构**：固定分区模板，便于调试和扩展

5. **压缩兜底**：超限时分区压缩，保持结构完整

---

*详见 chapter9 代码示例：01_context_builder_basic.py, 02_context_builder_with_agent.py*
