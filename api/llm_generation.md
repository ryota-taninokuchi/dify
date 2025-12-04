# LLM Generation Detail 存储方案

## 需求背景

需要存储 LLM 生成的详细内容，包括：

- `content`：模型输出的文本
- `reasoning_content`：模型的推理过程（可能有多段）
- `tool_calls`：工具调用记录

对于 Workflow/Chatflow，这会体现为 LLM 节点新增一个 `generation` 输出变量；对于 Basic App，则是把这些信息关联到 Message 上。

核心挑战在于，这些内容在生成时是**交错出现**的：

```
content_1 → reasoning_1 → content_2 → tool_call_1 → content_3
```

流式输出按时间顺序展示即可，但存储后再展示需要保留这个顺序信息。

---

## 应用类型分析

Dify 有 5 种应用类型，存储方式有所不同：

| 应用类型   | 有 Message | 有 WorkflowRun | Pipeline 类                           |
| ---------- | ---------- | -------------- | ------------------------------------- |
| Completion | ✅         | ❌             | `EasyUIBasedGenerateTaskPipeline`     |
| Chat       | ✅         | ❌             | `EasyUIBasedGenerateTaskPipeline`     |
| Agent-Chat | ✅         | ❌             | `EasyUIBasedGenerateTaskPipeline`     |
| Chatflow   | ✅         | ✅             | `AdvancedChatAppGenerateTaskPipeline` |
| Workflow   | ❌         | ✅             | `WorkflowAppGenerateTaskPipeline`     |

前 4 种都有 Message，只有 Workflow 没有。因此设计思路为：

- 有 Message 的应用 → 用 `message_id` 关联
- Workflow → 用 `workflow_run_id + node_id` 关联

---

## 表结构设计

```python
class LLMGenerationDetail(Base):
    """
    存储 LLM 生成的详细内容，包括推理过程和工具调用。

    关联方式二选一：
    - 有 Message 的应用：用 message_id（一对一）
    - Workflow：用 workflow_run_id + node_id（一个 run 可能有多个 LLM 节点）
    """
    __tablename__ = "llm_generation_details"
    __table_args__ = (
        sa.PrimaryKeyConstraint("id", name="llm_generation_detail_pkey"),
        sa.Index("idx_llm_generation_detail_message", "message_id"),
        sa.Index("idx_llm_generation_detail_workflow", "workflow_run_id", "node_id"),
    )

    id: Mapped[str] = mapped_column(StringUUID, default=lambda: str(uuid4()))
    tenant_id: Mapped[str] = mapped_column(StringUUID, nullable=False)
    app_id: Mapped[str] = mapped_column(StringUUID, nullable=False)

    # 关联字段，二选一
    message_id: Mapped[str | None] = mapped_column(StringUUID, nullable=True, unique=True)
    workflow_run_id: Mapped[str | None] = mapped_column(StringUUID, nullable=True)
    node_id: Mapped[str | None] = mapped_column(String(255), nullable=True)

    # 核心数据，JSON 字符串
    reasoning_content: Mapped[str | None] = mapped_column(LongText)  # ["推理1", "推理2", ...]
    tool_calls: Mapped[str | None] = mapped_column(LongText)  # [{name, arguments, result}, ...]
    sequence: Mapped[str | None] = mapped_column(LongText)  # 顺序信息

    created_at: Mapped[datetime] = mapped_column(DateTime, nullable=False, server_default=func.current_timestamp())
```

### sequence 字段格式

用于记录展示顺序。content 用 start/end 表示在原文中的位置，reasoning 和 tool_call 用 index 指向数组下标：

```json
[
  { "type": "content", "start": 0, "end": 100 },
  { "type": "reasoning", "index": 0 },
  { "type": "content", "start": 100, "end": 200 },
  { "type": "tool_call", "index": 0 },
  { "type": "content", "start": 200, "end": 350 }
]
```

---

## 各应用类型的存储与查询

### Completion / Chat

直接调用 LLM，流程最简单。

- **写入位置**：`EasyUIBasedGenerateTaskPipeline._save_message()`
- **数据来源**：`llm_result` 对象
- **查询方式**：`WHERE message_id = ?`

目前这两种应用可能还没有 reasoning_content 和 tool_calls（取决于 LLM 是否支持），可以先预留字段。

### Agent-Chat

Agent 模式会多轮调用 LLM，每轮的思考和工具调用存储在 `MessageAgentThought` 表中。需要在保存时将这些记录合并为一条 `LLMGenerationDetail`。

- **写入位置**：`EasyUIBasedGenerateTaskPipeline._save_message()`
- **数据来源**：合并 `MessageAgentThought` 记录
- **查询方式**：`WHERE message_id = ?`

合并逻辑：

```python
def _save_agent_generation_detail(self, session: Session, message: Message):
    # 获取所有轮次，按 position 排序
    thoughts = (
        session.query(MessageAgentThought)
        .where(MessageAgentThought.message_id == message.id)
        .order_by(MessageAgentThought.position.asc())
        .all()
    )

    if not thoughts:
        return

    reasoning_content = []
    tool_calls = []
    sequence = []

    for thought in thoughts:
        # 思考内容
        if thought.thought:
            reasoning_content.append(thought.thought)
            sequence.append({"type": "reasoning", "index": len(reasoning_content) - 1})

        # 工具调用
        if thought.tool:
            tool_calls.append({
                "name": thought.tool,
                "arguments": thought.tool_input or "",
                "result": thought.observation or ""
            })
            sequence.append({"type": "tool_call", "index": len(tool_calls) - 1})

        # 回答内容
        if thought.answer:
            # 处理 content 的位置...

    detail = LLMGenerationDetail(
        message_id=message.id,
        reasoning_content=json.dumps(reasoning_content),
        tool_calls=json.dumps(tool_calls),
        sequence=json.dumps(sequence),
        ...
    )
    session.add(detail)
```

### Chatflow

走 Workflow 引擎，但最终产出 Message。Answer 来自 `_task_state.answer`，如果 Answer 节点引用了多个 LLM 节点的 generation，需要合并。

- **写入位置**：`AdvancedChatAppGenerateTaskPipeline._save_message()`
- **数据来源**：`_task_state` 中的合并结果
- **查询方式**：`WHERE message_id = ?`

### Workflow

没有 Message，每个 LLM 节点执行完都存一条 Detail。

- **写入位置**：`LLMNode._run()` 节点执行完成时
- **数据来源**：单个 LLM 节点的输出
- **查询方式**：
  - 查某个节点：`WHERE workflow_run_id = ? AND node_id = ?`
  - 查所有 LLM 节点：`WHERE workflow_run_id = ?`

```python
detail = LLMGenerationDetail(
    message_id=None,
    workflow_run_id=self._workflow_run_id,
    node_id=self._node_id,
    reasoning_content=json.dumps(reasoning_contents),
    tool_calls=json.dumps(tool_calls),
    sequence=json.dumps(sequence),
    ...
)
```

---

## Workflow/Chatflow 示例

### Chatflow：存储时合并，读取时直接用

Chatflow 有 Message，generation_detail 在存储时就已经按 answer 的引用顺序合并好了。

```
[Start] → [LLM_1] → [LLM_2] → [Answer]
                                 ↓
              answer = {{#llm_1.generation#}} + {{#llm_2.generation#}}
```

**写入**：在 `_save_message()` 时，按 answer 的引用顺序合并所有 LLM 的 generation

```python
# 合并后存储
LLMGenerationDetail(
    message_id=message.id,
    workflow_run_id=None,
    node_id=None,
    reasoning_content='["llm_1的推理", "llm_2的推理"]',  # 已按顺序合并
    tool_calls='[{...}, {...}]',  # 已按顺序合并
    sequence='[...]',  # 已按顺序合并
)
```

**读取**：直接在 message 接口返回，不需要额外处理

```python
# 现有 message 接口扩展返回 generation_detail
{
    "id": "msg_123",
    "answer": "最终回答...",
    "generation_detail": {
        "reasoning_content": ["llm_1的推理", "llm_2的推理"],
        "tool_calls": [...],
        "sequence": [...]
    }
}
```

前端拿到就直接按 sequence 顺序展示，不需要关心数据来自哪个节点。

### Workflow：按节点分别存储，在节点详情 API 中自动附加

Workflow 没有 Message，每个 LLM 节点各存一条 Detail。

```
[Start] → [LLM_1] → [LLM_2] → [End]
```

**写入**：每个 LLM 节点执行完成时存一条

```python
# LLM_1 执行完成
LLMGenerationDetail(
    workflow_run_id="run_123",
    node_id="llm_1",
    ...
)

# LLM_2 执行完成
LLMGenerationDetail(
    workflow_run_id="run_123",
    node_id="llm_2",
    ...
)
```

**读取**：在节点执行详情 API 中，后端自动查询并附加 `generation_detail` 字段

```python
# 节点详情接口内部逻辑
def get_node_execution(node_execution_id):
    node_execution = ...

    # 如果是 LLM 节点，自动附加 generation_detail
    if node_execution.node_type == NodeType.LLM:
        generation_detail = (
            db.session.query(LLMGenerationDetail)
            .filter_by(
                workflow_run_id=node_execution.workflow_run_id,
                node_id=node_execution.node_id
            )
            .first()
        )
        if generation_detail:
            node_execution.generation_detail = generation_detail.to_dict()

    return node_execution
```

---

## API 设计

### 有 Message 的应用（Completion / Chat / Agent-Chat / Chatflow）

不新增接口，直接在现有 message 接口中返回 `generation_detail` 字段：

```json
{
  "id": "msg_123",
  "answer": "最终回答...",
  "generation_detail": {
    "reasoning_content": ["推理内容1", "推理内容2"],
    "tool_calls": [{ "name": "search", "arguments": "{}", "result": "..." }],
    "sequence": [
      { "type": "content", "start": 0, "end": 100 },
      { "type": "reasoning", "index": 0 },
      { "type": "tool_call", "index": 0 }
    ]
  }
}
```

### Workflow

Workflow 没有 Message，不新增接口。在现有的**节点执行详情接口**中，后端自动附加 `generation_detail` 字段：

```
GET /apps/{app_id}/workflow-runs/{run_id}/node-executions/{node_execution_id}
```

响应：

```json
{
  "id": "node_exec_123",
  "node_id": "llm_1",
  "node_type": "llm",
  "inputs": {...},
  "outputs": {
    "text": "最终回答...",
    "reasoning_content": "推理内容..."
  },
  "execution_metadata": {...},

  "generation_detail": {
    "reasoning_content": ["推理内容1", "推理内容2"],
    "tool_calls": [{ "name": "search", "arguments": "{}", "result": "..." }],
    "sequence": [
      { "type": "content", "start": 0, "end": 100 },
      { "type": "reasoning", "index": 0 },
      { "type": "tool_call", "index": 0 }
    ]
  }
}
```

`generation_detail` 字段由后端从 `LLMGenerationDetail` 表查询后附加，只有 LLM 节点才会有此字段。

### 前端判断逻辑

前端无需判断 `node_type`，只需检查响应中是否有 `generation_detail` 字段：

- **有** `generation_detail` → 渲染 generation 详情面板
- **没有** `generation_detail` → 不展示

这样设计的好处：

1. **安全可信**：`generation_detail` 来自后端查表，不是从 `outputs` 解析，无法被用户伪造
2. **逻辑简单**：前端不需要额外验证，有就展示，没有就不展示
3. **减少请求**：不需要单独调用接口获取 generation 信息

---

## 总结

### Completion / Chat

- **关联字段**：`message_id`
- **写入位置**：`EasyUIBasedGenerateTaskPipeline._save_message()`
- **查询方式**：Message 接口自动附加 `generation_detail`

### Agent-Chat

- **关联字段**：`message_id`
- **写入位置**：`EasyUIBasedGenerateTaskPipeline._save_message()`（合并 AgentThought）
- **查询方式**：Message 接口自动附加 `generation_detail`

### Chatflow

- **关联字段**：`message_id`
- **写入位置**：`AdvancedChatAppGenerateTaskPipeline._save_message()`
- **查询方式**：Message 接口自动附加 `generation_detail`

### Workflow

- **关联字段**：`workflow_run_id + node_id`
- **写入位置**：`LLMNode._run()` 节点执行完成时
- **查询方式**：节点执行详情接口自动附加 `generation_detail`

---

## 数据流

### 有 Message 的应用

```
请求 → Pipeline → LLM 调用 → 保存 Message
                       │
                       └──→ 保存 LLMGenerationDetail (message_id)

API 响应：Message 接口自动附加 generation_detail 字段
```

### Workflow

```
请求 → WorkflowRun → LLM_1 节点 → 保存 NodeExecution
            │              │
            │              └──→ 保存 LLMGenerationDetail (workflow_run_id, node_id="llm_1")
            │
            └──→ LLM_2 节点 → 保存 NodeExecution
                       │
                       └──→ 保存 LLMGenerationDetail (workflow_run_id, node_id="llm_2")

API 响应：节点执行详情接口自动附加 generation_detail 字段
```

---

## 设计要点

### 统一存储

所有应用类型都使用 `LLMGenerationDetail` 表存储，保持一致性。

### 自动附加

后端在返回数据时自动查询并附加 `generation_detail` 字段，前端无需额外请求。

### 安全可信

`generation_detail` 来自后端查表，不是从 `outputs` 解析，无法被用户通过 Code 节点等方式伪造。

### 前端判断简单

前端只需检查响应中是否有 `generation_detail` 字段，有就展示，没有就不展示。不需要判断 `node_type` 或验证数据结构。
