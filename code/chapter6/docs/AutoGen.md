# AutoGen 框架简介

## 是什么？

AutoGen 是微软开源的多智能体协作框架，核心理念是**"以对话驱动协作"**。它让多个 AI 智能体像人类团队一样通过对话协作完成任务。

---

## 核心架构（实际使用版）

```
┌─────────────────────────────────────────────────────────────────┐
│                     AutoGen 架构                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   用户任务                                                       │
│      │                                                          │
│      ▼                                                          │
│   ┌───────────────────────────────────────┐                     │
│   │         Selector（选择器）              │◄─────────────┐    │
│   │    LLM 根据对话内容动态选择下一位发言者  │              │    │
│   └───────────┬───────────────────────────┘              │    │
│               │                                          │    │
│       ┌───────┴───────┬───────────────┐                  │    │
│       │               │               │                  │    │
│       ▼               ▼               ▼                  │    │
│   ┌─────────┐   ┌───────────┐   ┌───────────┐            │    │
│   │Assistant│   │UserProxy  │   │  其他角色  │            │    │
│   │ Agent   │◄─►│  Agent    │   │   Agent   │            │    │
│   │ (思考者) │   │ (执行者)   │   │           │            │    │
│   └────┬────┘   └─────┬─────┘   └───────────┘            │    │
│        │              │                                  │    │
│        │         ┌────┘                                  │    │
│        │         │                                       │    │
│        │         ▼                                       │    │
│        │    ┌─────────────┐                              │    │
│        └───►│  工具执行    │                              │    │
│             │ 代码运行/调用 │                              │    │
│             └─────────────┘                              │    │
│                                                          │    │
│   循环逻辑：问题解决 ◄─────────────────────────────────────┘    │
│   - 执行失败 → 返回 Assistant 重新思考                          │
│   - 审查不通过 → Selector 选择 Engineer 修改                    │
│   - 测试不通过 → Selector 选择对应角色处理                      │
│                                                                 │
│   终止：任务完成 / 达到最大轮数 / 用户输入 TERMINATE             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**实际循环示例**：
```
Assistant: 写代码
    ↓
UserProxy: 执行，报错：模块不存在
    ↓
Selector: 判断应该 Assistant 修复
    ↓
Assistant: 添加导入语句，重新提交
    ↓
UserProxy: 执行成功
    ↓
Assistant: 任务完成！
    ↓
TERMINATE
```

---

## 三大核心组件

| 组件 | 作用 | 类比 |
|------|------|------|
| `AssistantAgent` | 负责思考、规划、生成代码/文本 | 大脑 |
| `UserProxyAgent` | 代表用户，执行代码/工具，反馈结果 | 手脚 |
| `SelectorGroupChat` | **根据内容动态选择**下一个发言者 | 智能主持人 |

---

## 快速上手示例

### 1. 安装

```bash
pip install autogen-agentchat autogen-ext[openai]
```

### 2. 双智能体协作（含循环修复）

```python
import asyncio
from autogen_ext.models.openai import OpenAIChatCompletionClient
from autogen_agentchat.agents import AssistantAgent, UserProxyAgent
from autogen_agentchat.teams import SelectorGroupChat  # 用 Selector 不是 RoundRobin
from autogen_agentchat.conditions import TextMentionTermination
from autogen_agentchat.ui import Console

# 1. 配置模型
model_client = OpenAIChatCompletionClient(
    model="gpt-4o",
    api_key="your-api-key",
)

# 2. 创建智能体
assistant = AssistantAgent(
    name="Coder",
    model_client=model_client,
    system_message="""你是Python专家，负责写代码。
如果执行报错，分析错误并修复代码，重新提交。
""",
)

user_proxy = UserProxyAgent(
    name="Executor",
    description="执行代码并返回结果（成功/报错）",
)

# 3. 组建团队（用 SelectorGroupChat 实现动态路由）
team = SelectorGroupChat(
    participants=[assistant, user_proxy],
    model_client=model_client,  # Selector 用 LLM 判断下一位是谁
    termination_condition=TextMentionTermination("TERMINATE"),
    max_turns=10,
)

# 4. 运行
async def main():
    task = "写一个Python函数，计算斐波那契数列前10项"
    await Console(team.run_stream(task=task))
    # 可能的过程：
    # Round 1: Coder写代码 → Executor执行报错（没导入模块）
    # Round 2: Selector判断应该 Coder 修复 → Coder修复 → Executor执行成功
    # Round 3: Coder确认完成 → TERMINATE

asyncio.run(main())
```

---

## 多智能体团队示例（实际工作流）

软件开发团队，**审查/测试失败可以回退到对应角色**：

```
                          ┌──────────────────────────┐
                          │     Selector（选择器）    │
                          │  LLM 根据对话内容决定下一位 │
                          └─────────────┬────────────┘
                                        │
        ┌───────────────────────────────┼───────────────────────────────┐
        │                               │                               │
        ▼                               ▼                               ▼
┌──────────────┐              ┌────────────────┐              ┌──────────────┐
│  产品经理     │              │     工程师      │              │  代码审查员   │
│ ProductManager│              │   Engineer      │              │ CodeReviewer  │
└───────┬──────┘              └───────┬────────┘              └───────┬──────┘
        │                             │                               │
        │  输出需求                    │  编写代码                      │  审查
        │                             │                               │
        │                             │                               │
        │                      ┌──────┴────────┐                      │
        │                      │               │                      │
        │                      ▼               ▼                      │
        │                ┌──────────┐    ┌──────────┐                 │
        │                │ 审查通过  │    │ 审查不通过│ ◄───────────────┘
        │                │          │    │          │
        │                └────┬─────┘    └────┬─────┘
        │                     │               │
        │                     │               └──────────► 直接回退到 Engineer
        │                     ▼               （Selector 选择 Engineer 发言）
        │              ┌──────────────┐
        │              │   用户代理    │
        │              │  UserProxy   │
        │              └───────┬──────┘
        │                      │
        │                 ┌────┴────────┐
        │                 │             │
        │                 ▼             ▼
        │           ┌──────────┐  ┌──────────┐
        │           │ 测试通过  │  │ 测试不通过│
        │           │          │  │          │
        │           └────┬─────┘  └────┬─────┘
        │                │             │
        │                │             └──────────► 回退到 Engineer
        │                │             （Selector 选择 Engineer）
        │                ▼
        │          TERMINATE
        │
        └────────────────────────────────────────────────────────►
              如果需求有问题，也会回退到 PM 澄清
```

```python
from autogen_agentchat.agents import AssistantAgent, UserProxyAgent
from autogen_agentchat.teams import SelectorGroupChat
from autogen_agentchat.conditions import TextMentionTermination

# 创建角色（系统消息引导回退逻辑）
product_manager = AssistantAgent(
    name="ProductManager",
    model_client=model_client,
    system_message="""你是产品经理。
- 分析需求，输出需求文档
- 如果工程师有疑问，澄清需求
- 完成后说'请 Engineer 实现'""",
)

engineer = AssistantAgent(
    name="Engineer", 
    model_client=model_client,
    system_message="""你是工程师，负责编写代码。
- 完成后说'请 Reviewer 检查'
- 如果 Reviewer 发现问题，修复后重新提交
- 如果测试失败，修复后重新提交""",
)

code_reviewer = AssistantAgent(
    name="CodeReviewer",
    model_client=model_client,
    system_message="""你是代码审查员。
- 如果有问题，说'Engineer，请修复：具体问题xxx'
- 如果通过，说'请 UserProxy 测试'""",
)

user_proxy = UserProxyAgent(
    name="UserProxy",
    description="""测试代码。
- 如果测试失败，说'Engineer，测试不通过，原因：xxx'
- 如果通过，说 TERMINATE"""
)

# 用 SelectorGroupChat（动态路由，不是死板轮询）
team = SelectorGroupChat(
    participants=[product_manager, engineer, code_reviewer, user_proxy],
    model_client=model_client,  # Selector 用 LLM 判断该谁发言
    termination_condition=TextMentionTermination("TERMINATE"),
    max_turns=20,
)
```

**实际运行过程（含回退）**：

```
Round 1: ProductManager
  "需求：比特币价格显示应用。功能：1.实时价格 2.24h趋势..."
  
Round 2: Engineer
  "我写好了代码，用 requests 调用 CoinGecko API..."
  
Round 3: CodeReviewer
  "发现两个问题：1. 缺少错误处理 2. 没有 API key 管理。
   Engineer，请修复这两个问题。"
  
Round 4: Engineer （Selector 判断 Engineer 该发言，不是 PM！）
  "已修复：添加了 try-except 和 .env 配置..."
  
Round 5: CodeReviewer
  "审查通过，请 UserProxy 测试"
  
Round 6: UserProxy
  "运行报错：ModuleNotFoundError: No module named 'requests'
   Engineer，测试不通过，需要添加依赖安装说明或检查导入"
   
Round 7: Engineer （又回退到 Engineer）
  "修复：添加了 pip install requests 到文档，并添加了导入检查..."
  
Round 8: UserProxy
  "测试通过，功能正常。TERMINATE"
```

---

## 对话调度方式对比

| 方式 | 机制 | 适用场景 | 回退支持 |
|------|------|----------|----------|
| `RoundRobinGroupChat` | 固定顺序 A→B→C→A | 简单流水线 | ❌ 死板 |
| `SelectorGroupChat` | **LLM 动态选择** | **复杂协作（推荐）** | ✅ **灵活回退** |
| `Swarm` | 去中心化自主决定 | 复杂自组织 | ✅ 灵活 |

**实际项目推荐用 `SelectorGroupChat`**

---

## 关键概念总结

```
┌─────────────────────────────────────────────────────────────────┐
│                    AutoGen 实际工作流程                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 定义角色 → 系统消息中包含"出问题找谁"的指引                   │
│       │                                                         │
│       ▼                                                         │
│  2. 组建 SelectorGroupChat → LLM 动态路由                       │
│       │                                                         │
│       ▼                                                         │
│  3. 分配任务                                                    │
│       │                                                         │
│       ▼                                                         │
│  4. 循环协作（动态路由，不是死板轮询）                           │
│     ┌─────────────────────────────────────────────────────┐     │
│     │                                                     │     │
│     │  Agent A 发言 ──► Selector 分析内容                 │     │
│     │                      │                              │     │
│     │                      ▼                              │     │
│     │              判断下一位该谁                         │     │
│     │                      │                              │     │
│     │         ┌────────────┼────────────┐                 │     │
│     │         │            │            │                 │     │
│     │         ▼            ▼            ▼                 │     │
│     │     继续 A        转到 B        转到 C              │     │
│     │   (自迭代)      (正常流转)    (回退处理)            │     │
│     │                                                     │     │
│     │   ┌─────────────────────────────────────────┐       │     │
│     │   │  示例流转：                              │       │     │
│     │   │  PM → Engineer → Reviewer ─┬─► Engineer │       │     │
│     │   │                             │   (回退)   │       │     │
│     │   │                            ▼            │       │     │
│     │   │                          UserProxy ─┬─► Engineer │   │     │
│     │   │                             (通过)  │   (回退)   │   │     │
│     │   │                                    ▼            │   │     │
│     │   │                                 TERMINATE       │   │     │
│     │   └─────────────────────────────────────────┘       │     │
│     │                                                     │     │
│     └─────────────────────────────────────────────────────┘     │
│                                                                 │
│  5. 终止条件满足 → 结束                                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 核心循环的本质（实际版）

```
单智能体（Coder ↔ Executor）：
┌──────────────────────────────────────────────────────┐
│                                                      │
│   Coder: 写代码                                      │
│      ↓                                               │
│   Executor: 执行                                     │
│      │                                               │
│      ├─► 失败 ──► Selector 选 Coder 修复 ──► 重试   │
│      │                                               │
│      └─► 成功 ──► Coder 确认完成                    │
│                      ↓                               │
│                  TERMINATE                           │
│                                                      │
└──────────────────────────────────────────────────────┘

多智能体（动态路由）：
┌──────────────────────────────────────────────────────┐
│                                                      │
│   PM ──► Engineer ──► Reviewer                       │
│              ▲            │                          │
│              │            ├─ 审查不通过               │
│              │            │                          │
│              │            ▼                          │
│              └────── 回退（Selector 路由）            │
│                                                      │
│   Engineer ──► UserProxy                             │
│      ▲            │                                  │
│      │            ├─ 测试不通过                       │
│      │            │                                  │
│      │            ▼                                  │
│      └────── 回退（Selector 路由）                    │
│                                                      │
│   特点：不是死板轮询，LLM 根据内容判断该谁处理        │
│                                                      │
└──────────────────────────────────────────────────────┘
```

---

## 优势与局限

**✅ 优势：**
- Selector 动态路由，灵活回退，接近真实团队协作
- 自动循环修复，直到问题解决
- 代码执行报错 → 自动回到 Coder 修复
- 审查/测试不通过 → 回到对应角色处理

**⚠️ 局限：**
- Selector 也有判断错误的时候（需要好的模型）
- 循环次数可能不可控（需设置 max_turns）
- 调试需要跟踪复杂的路由历史

---

## 官方资源

- GitHub: https://github.com/microsoft/autogen
- 文档: https://microsoft.github.io/autogen/
- 推荐：`SelectorGroupChat` 比 `RoundRobinGroupChat` 更适合实际项目
