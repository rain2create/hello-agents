# NoteTool 结构化笔记工具

NoteTool 是 HelloAgents 框架的长时程任务记忆组件，使用 Markdown + YAML 格式实现结构化的外部记忆管理。
主要是为了实现项目化的记忆模块；而不是像memorytool一样对话式的，可以做长期的记忆模块导入

---

## 核心架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                         NoteTool                                    │
│                                                                     │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐  ┌─────────────┐  │
│  │ Create  │ │  Read   │ │ Update  │ │ Delete  │  │   Summary   │  │
│  │ 创建笔记 │ │ 读取笔记 │ │ 更新笔记 │ │ 删除笔记 │  │   统计摘要   │  │
│  │  (增)   │ │  (查)   │ │  (改)   │ │  (删)   │  │   (统计)     │  │
│  └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘  └──────┬──────┘  │
│       └───────────┴───────────┴───────────┘             │         │
│                         │                               │         │
│                         ▼                               ▼         │
│              ┌─────────────────────┐    ┌──────────────────────┐  │
│              │   笔记文件 (.md)     │    │   统计报告 (JSON)     │  │
│              │   • YAML Front Matter│    │   • 总数/类型/标签    │  │
│              │   • Markdown 正文    │    │   • 项目进度概览      │  │
│              └─────────────────────┘    └──────────────────────┘  │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │                      存储层                                      ││
│  │  • Markdown + YAML 格式                                        ││
│  │  • 文件名即ID: note_YYYYMMDD_HHMMSS_X.md                        ││
│  │  • notes_index.json 索引文件                                    ││
│  └─────────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────────┘
```

---

## 存储格式详解

### 笔记文件格式

```markdown
---
id: note_20250119_153000_0
title: 项目进展 - 第一阶段
type: task_state
tags: [refactoring, phase1, backend]
created_at: 2025-01-19T15:30:00
updated_at: 2025-01-19T15:30:00
---

# 项目进展 - 第一阶段

## 完成情况

已完成数据模型层的重构,主要改动包括:

1. 统一了实体类的命名规范
2. 引入了类型提示,提升代码可维护性
3. 优化了数据库查询性能

## 测试覆盖

- 单元测试覆盖率: 85%
- 集成测试覆盖率: 70%
```

### 索引文件结构

```json
{
  "note_20250119_153000_0": {
    "id": "note_20250119_153000_0",
    "title": "项目进展 - 第一阶段",
    "type": "task_state",
    "tags": ["refactoring", "phase1", "backend"],
    "created_at": "2025-01-19T15:30:00",
    "updated_at": "2025-01-19T15:30:00",
    "file_path": "./notes/note_20250119_153000_0.md"
  }
}
```

---

## 核心操作流程

```
┌──────────────────────────────────────────────────────────────────────┐
│                        NoteTool 工作流程                              │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  CREATE 创建笔记                                                      │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │  1. 生成唯一ID: note_YYYYMMDD_HHMMSS_X                          │  │
│  │  2. 构建 YAML Front Matter 元数据                                │  │
│  │  3. 组合 Markdown 内容                                          │  │
│  │  4. 写入文件: {workspace}/{note_id}.md                          │  │
│  │  5. 更新 notes_index.json                                       │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                              ↓                                       │
│  SEARCH/LIST 检索笔记                                                 │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │  • list: 读取索引文件，返回所有笔记列表                          │  │
│  │  • search: 按 title/content/type/tags 关键词匹配                │  │
│  │  • filter: 按 type 或 tags 筛选                                 │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                              ↓                                       │
│  READ 读取笔记                                                        │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │  1. 从索引获取文件路径                                           │  │
│  │  2. 解析 YAML Front Matter                                       │  │
│  │  3. 提取 Markdown 正文                                           │  │
│  │  4. 返回结构化数据                                               │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                              ↓                                       │
│  UPDATE 更新笔记                                                      │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │  1. 读取原笔记                                                   │  │
│  │  2. 更新指定字段                                                 │  │
│  │  3. 更新 updated_at 时间戳                                       │  │
│  │  4. 重写文件 + 更新索引                                          │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                              ↓                                       │
│  DELETE 删除笔记                                                      │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │  1. 从索引获取文件路径                                           │  │
│  │  2. 删除物理文件                                                 │  │
│  │  3. 从索引中移除                                                 │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                              ↓                                       │
│  SUMMARY 统计摘要                                                     │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │  1. 读取索引文件                                                 │  │
│  │  2. 统计笔记总数 / 类型分布 / 标签频次                            │  │
│  │  3. 生成结构化摘要报告                                            │  │
│  │  4. 返回: {"total_notes": N, "by_type": {...}, "by_tags": {...}} │  │
│  └────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 核心操作

### 1. create - 创建笔记

```python
from hello_agents.tools import NoteTool

notes = NoteTool(workspace="./notes")

# 创建任务状态笔记
notes.run({
    "action": "create",
    "title": "重构项目 - 第一阶段",
    "content": "已完成数据模型层的重构...",
    "note_type": "task_state",
    "tags": ["refactoring", "phase1"]
})
```

### 2. list/search - 检索笔记

```python
# 列出所有笔记
all_notes = notes.run({"action": "list"})

# 搜索笔记
results = notes.run({
    "action": "search",
    "query": "重构",
    "limit": 5
})

# 按类型筛选
tasks = notes.run({
    "action": "list",
    "note_type": "task_state"
})
```

### 3. read - 读取笔记

```python
note = notes.run({
    "action": "read",
    "note_id": "note_20250119_153000_0"
})
# 返回: {"id": "...", "title": "...", "content": "...", "type": "...", ...}
```

### 4. update - 更新笔记

```python
notes.run({
    "action": "update",
    "note_id": "note_20250119_153000_0",
    "title": "更新后的标题",
    "content": "更新后的内容"
})
```

### 5. delete - 删除笔记

```python
notes.run({
    "action": "delete",
    "note_id": "note_20250119_153000_0"
})
```

### 6. summary - 统计摘要 ⭐

获取笔记库的整体统计信息，用于项目进度概览和长时程任务追踪。

```python
summary = notes.run({"action": "summary"})
```

**返回结构：**

```json
{
  "total_notes": 12,
  "by_type": {
    "task_state": 4,
    "conclusion": 2,
    "blocker": 3,
    "action": 2,
    "reference": 1
  },
  "by_tags": {
    "refactoring": 5,
    "backend": 4,
    "urgent": 2,
    "optimization": 3
  },
  "recent_activity": {
    "last_24h": 3,
    "last_7d": 8
  }
}
```

**使用场景：**

```python
# 场景 1：项目进度概览
summary = notes.run({"action": "summary"})
print(f"项目共有 {summary['total_notes']} 条笔记")
print(f"待解决问题: {summary['by_type'].get('blocker', 0)} 个")
print(f"已完成结论: {summary['by_type'].get('conclusion', 0)} 个")

# 场景 2：长时程任务检查
if summary['by_type'].get('action', 0) > 5:
    print("警告：积压行动项过多，建议优先处理！")

# 场景 3：生成项目报告
report = {
    "project_status": "进行中",
    "total_notes": summary['total_notes'],
    "open_issues": summary['by_type'].get('blocker', 0),
    "completed_tasks": summary['by_type'].get('conclusion', 0)
}
```

---

## 笔记类型说明

| 类型 | 用途 | 示例场景 |
|------|------|----------|
| `task_state` | 任务状态记录 | 项目进度、阶段总结 |
| `conclusion` | 结论/总结 | 调研结论、会议纪要 |
| `blocker` | 阻塞问题 | 技术难点、依赖问题 |
| `action` | 行动计划 | TODO、下一步行动 |
| `reference` | 参考资料 | 文献、链接、引用 |
| `general` | 通用笔记 | 其他信息 |

---

## 与 ContextBuilder 集成

```python
# 在 Agent 中集成笔记到上下文
def run(self, user_input: str) -> str:
    # 1. 检索相关笔记
    relevant_notes = self.note_tool.run({
        "action": "search",
        "query": user_input,
        "limit": 3
    })
    
    # 2. 转换为 ContextPacket
    note_packets = []
    for note in relevant_notes:
        note_packets.append(ContextPacket(
            content=note['content'],
            timestamp=note['updated_at'],
            token_count=self._count_tokens(note['content']),
            relevance_score=0.7,
            metadata={"type": "note", "note_type": note['type']}
        ))
    
    # 3. 构建上下文时传入
    context = self.context_builder.build(
        user_query=user_input,
        custom_packets=note_packets,
        ...
    )
```

---

## 长时程任务工作流

```
Day 1                    Day 2                    Day 3
  │                        │                        │
  ▼                        ▼                        ▼
┌─────────┐            ┌─────────┐            ┌─────────┐
│探索代码库 │            │分析问题  │            │规划重构  │
│         │            │         │            │         │
│• 记录    │            │• 读取    │            │• 读取    │
│  task_state│           │  昨日笔记 │            │  所有笔记 │
│• 标记    │            │• 补充    │            │• 创建    │
│  blocker │            │  conclusion│           │  action  │
└────┬────┘            └────┬────┘            └────┬────┘
     │                      │                      │
     │               ┌──────┴──────┐              │
     │               ▼             ▼              │
     │          ┌─────────┐   ┌─────────┐        │
     │          │ summary │   │ summary │        │
     │          │ 检查点1  │   │ 检查点2  │        │
     │          └────┬────┘   └────┬────┘        │
     │               │             │             │
     └───────────────┴─────────────┴─────────────┘
                            │
                            ▼
                    ┌───────────────┐
                    │  一周后检查    │
                    │               │
                    │ • 读取历史笔记  │
                    │ • summary 统计 │  ← 项目全局视图
                    │   - 总进度     │
                    │   - 阻塞问题   │
                    │   - 待办事项   │
                    │ • 更新状态     │
                    └───────────────┘
```

---

## 目录结构示例

```
project/
├── notes/
│   ├── note_20250119_143000_0.md    # 项目初始化
│   ├── note_20250119_153000_1.md    # 第一阶段完成
│   ├── note_20250120_092000_2.md    # 依赖问题记录
│   ├── note_20250120_164500_3.md    # 技术方案结论
│   └── notes_index.json             # 索引文件
└── src/
    └── ...
```

---

## 关键要点总结

1. **格式优势**：Markdown + YAML，人类可读 + 机器可解析

2. **轻量设计**：文件系统存储，无需数据库，天然支持 Git 版本控制

3. **六大类型**：task_state / conclusion / blocker / action / reference / general

4. **索引管理**：notes_index.json 实现快速检索，避免遍历文件

5. **统计摘要**：`summary` 功能提供项目全局视图，支持进度追踪

6. **长时程支持**：跨会话状态保持，项目式任务追踪

---

*详见 chapter9 代码示例：03_note_tool_operations.py, 04_note_tool_integration.py*
