## 模块 9：SubagentManager（`subagent.py`）

### 怎么设计的

SubagentManager 管理后台子代理的生成、执行和结果通知。

**9.1 数据结构**

```python
@dataclass(slots=True)
class SubagentStatus:
    task_id: str              # 唯一任务 ID（UUID 前 8 位）
    label: str                # 显示标签
    task_description: str     # 任务描述
    started_at: float         # 启动时间（monotonic）
    phase: str                # 当前阶段（initializing/awaiting_tools/tools_completed/final_response/done/error）
    iteration: int            # 当前迭代
    tool_events: list         # 工具事件
    usage: dict               # token 用量
    stop_reason: str | None   # 停止原因
    error: str | None         # 错误信息
```

运行时状态：

```python
self._running_tasks: dict[str, asyncio.Task]     # task_id → asyncio.Task
self._task_statuses: dict[str, SubagentStatus]   # task_id → 实时状态
self._session_tasks: dict[str, set[str]]         # session_key → {task_id, ...}
```

**9.2 生成子代理 `spawn()`**

```python
async def spawn(task, label, origin_channel, origin_chat_id, session_key) -> str:
    task_id = uuid.uuid4()[:8]
    status = SubagentStatus(...)
    bg_task = asyncio.create_task(self._run_subagent(...))
    # 注册清理回调：任务完成后从 _running_tasks 和 _task_statuses 中移除
    return "Subagent [label] started (id: task_id). I'll notify you when it completes."
```

**9.3 执行子代理 `_run_subagent()`**

子代理有自己独立的 AgentRunner 和 ToolRegistry，与主代理的关键区别：

| 方面 | 主代理 | 子代理 |
|---|---|---|
| 工具集 | 全部工具（含 message、spawn） | 无 message、无 spawn（防递归） |
| max_iterations | 用户配置 | 固定 15 |
| fail_on_tool_error | False | True（工具错误致命） |
| System prompt | 完整身份+记忆+技能 | 精简的子代理专用 prompt |
| 结果通知 | 直接返回给用户 | 通过消息总线通知主代理 |

子代理的工具集：ReadFileTool、WriteFileTool、EditFileTool、ListDirTool、GlobTool、GrepTool、ExecTool（可选）、WebSearchTool/WebFetchTool（可选）。**没有 MessageTool 和 SpawnTool**——子代理不能给用户发消息，也不能生成子子代理。

**9.4 结果通知 `_announce_result()`**

子代理完成后，通过消息总线发送一条 `InboundMessage`：

```python
msg = InboundMessage(
    channel="system",           # 系统消息
    sender_id="subagent",       # 标识为子代理
    chat_id="channel:chat_id",  # 原始通道和聊天 ID
    content=announce_content,   # 格式化的结果
    session_key_override=override,  # 路由到主代理的会话
    metadata={
        "injected_event": "subagent_result",
        "subagent_task_id": task_id,
    },
)
```

`session_key_override` 确保结果路由到主代理的 pending_queue（中途注入），而不是作为独立任务分派。

**9.5 `_SubagentHook`**

子代理的 Hook 实现，做两件事：
- `before_execute_tools()`：记录工具调用日志
- `after_iteration()`：更新 SubagentStatus 的实时状态（iteration、tool_events、usage、error）

**9.6 其他方法**

| 方法 | 作用 |
|---|---|
| `cancel_by_session(session_key)` | 取消某会话的所有子代理，返回取消数量 |
| `get_running_count()` | 当前运行中的子代理总数 |
| `get_running_count_by_session()` | 某会话的运行中子代理数 |

### 为什么设计

**为什么子代理不能有 MessageTool 和 SpawnTool？**

- **无 MessageTool**：子代理是后台任务，不应该直接给用户发消息。所有用户交互通过主代理进行。如果子代理能发消息，用户会收到来源不明的通知，且主代理无法控制消息格式。
- **无 SpawnTool**：防止递归生成——子代理生成子子代理，子子代理又生成子子子代理，可能导致无限递归和资源耗尽。

**为什么子代理的 `fail_on_tool_error=True`？**

主代理的工具错误是非致命的（LLM 会分析错误并换策略），但子代理是无人值守的后台任务。如果工具持续失败，子代理应该快速终止而不是无限重试，浪费 token 和时间。

**为什么结果通过消息总线而不是直接回调？**

子代理是异步后台任务，完成时主代理可能正在处理其他消息。通过消息总线：
1. **解耦**：子代理不需要持有主代理的引用
2. **正确路由**：`session_key_override` 确保结果路由到正确的会话
3. **中途注入**：如果主代理正在处理同一会话，结果会进入 pending_queue 而不是创建竞争任务

**为什么子代理有独立的 ToolRegistry 而不是共享主代理的？**

子代理的工具集是主代理的子集（无 message/spawn）。如果共享 Registry，子代理的 `unregister` 会影响主代理。独立 Registry 确保隔离。

**为什么 SubagentStatus 需要实时更新？**

主代理可能需要查询子代理的进度（如用户问"后台任务进展如何"）。`_SubagentHook.after_iteration()` 实时更新 status，让主代理可以随时获取当前迭代、工具事件、token 用量等信息。

### 在系统中充当什么角色

**SubagentManager 是 Agent 的"并行执行器"——让主代理可以同时处理多个任务。**

```
主代理处理用户消息：
  "帮我重构 auth 模块，同时写单元测试"
    ↓
  主代理决定：
    1. 自己处理重构（需要交互）
    2. spawn 子代理写测试（后台执行）
    ↓
  SubagentManager.spawn("写 auth 模块单元测试")
    → 创建独立 AgentRunner + ToolRegistry
    → asyncio.create_task(_run_subagent)
    ↓
  主代理继续处理重构...
    ↓
  子代理完成 → _announce_result()
    → InboundMessage(channel="system", sender_id="subagent")
    → 消息总线 → 主代理的 pending_queue → 中途注入
    ↓
  主代理看到子代理结果，综合后回复用户
```

具体角色：
- **对 SpawnTool**：SpawnTool 调用 `SubagentManager.spawn()` 生成子代理
- **对 AgentLoop**：AgentLoop 在 `_process_message()` 中处理子代理的系统消息
- **对消息总线**：子代理结果通过总线路由到主代理
- **对并发控制**：`cancel_by_session()` 让 `/stop` 命令能取消所有子代理

SubagentManager 是"单代理"和"多代理"之间的**桥梁**——它让一个 Agent 可以派生出多个并行执行的子 Agent，同时保持结果汇聚和资源隔离。

---
