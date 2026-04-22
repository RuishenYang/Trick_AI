## 模块 6：AgentLoop（`loop.py`）

### 怎么设计的

AgentLoop 是整个 Agent 系统的顶层编排器，是"产品层"的唯一入口。它把引擎层（AgentRunner）、信息层（ContextBuilder）、工具层（ToolRegistry）、会话层（SessionManager）、内存层（Consolidator/Dream/AutoCompact）、子代理层（SubagentManager）全部组装在一起，对外暴露一个简单的 `run()` 方法。

**6.1 初始化：组装所有子系统**

`__init__()` 创建了 Agent 系统运行所需的全部组件：

```python
self.context = ContextBuilder(workspace, ...)       # 信息总装线
self.sessions = SessionManager(workspace)            # 会话管理
self.tools = ToolRegistry()                          # 工具注册表
self.runner = AgentRunner(provider)                  # ReAct 引擎
self.subagents = SubagentManager(...)                # 子代理管理
self.consolidator = Consolidator(...)                # 轻轻量内存合并
self.auto_compact = AutoCompact(...)                 # 自动压缩
self.dream = Dream(...)                              # 深度内存处理
self.commands = CommandRouter()                      # 斜杠命令路由
```

然后调用 `_register_default_tools()` 注册所有内置工具：

| 工具 | 条件 |
|---|---|
| ReadFileTool, WriteFileTool, EditFileTool, ListDirTool | 始终注册 |
| GlobTool, GrepTool | 始终注册 |
| NotebookEditTool | 始终注册 |
| ExecTool | `exec_config.enable` |
| WebSearchTool, WebFetchTool | `web_config.enable` |
| MessageTool | 始终注册（回调为 `bus.publish_outbound`） |
| SpawnTool | 始终注册 |
| CronTool | `cron_service` 存在 |
| MyTool | `tools_config.my.enable` |

还有几个关键的数据结构用于运行时管理：

```python
self._active_tasks: dict[str, list[asyncio.Task]]   # session_key → 活跃任务列表
self._session_locks: dict[str, asyncio.Lock]         # session_key → 会话锁
self._pending_queues: dict[str, asyncio.Queue]       # session_key → 中途注入队列
self._concurrency_gate: asyncio.Semaphore            # 跨会话并发限制（默认 3）
```

**6.2 主循环 `run()` —— 消费消息总线**

```python
async def run(self):
    self._running = True
    await self._connect_mcp()          # 连接 MCP 服务器
    while self._running:
        msg = await self.bus.consume_inbound()  # 阻塞等待消息
        # 1. 优先级命令（如 /stop）立即处理
        # 2. 如果该会话已有活跃任务 → 路由到 pending_queue（中途注入）
        # 3. 否则 → 创建新 asyncio.Task 分派处理
```

三个关键设计：

1. **优先级命令**：`/stop` 等命令不经过会话处理，立即执行
2. **中途注入路由**：如果会话已有活跃任务，新消息放入 `pending_queue` 而不是创建竞争任务
3. **异步任务分派**：每条消息创建一个 `asyncio.Task`，任务完成后自动从 `_active_tasks` 中移除

**6.3 消息分派 `_dispatch()` —— 会话隔离与并发控制**

```python
async def _dispatch(self, msg):
    lock = self._session_locks.setdefault(session_key, asyncio.Lock())
    gate = self._concurrency_gate or nullcontext()
    pending = asyncio.Queue(maxsize=20)
    self._pending_queues[session_key] = pending

    async with lock, gate:          # 会话内串行，跨会话限并发
        # 设置流式回调（on_stream / on_stream_end）
        # 调用 _process_message()
        # 发布响应到 bus
```

关键设计：
- **`async with lock`**：同一会话内的消息串行处理，防止并发修改会话状态
- **`async with gate`**：跨会话并发限制（Semaphore，默认 3），防止同时处理太多请求导致 OOM
- **pending_queue 注册**：在 `_pending_queues` 中注册队列，让后续同会话消息可以注入
- **finally 块**：任务结束后，把 pending_queue 中残留的消息重新发布到 bus，不会丢失

**6.4 消息处理 `_process_message()` —— 完整的处理流水线**

这是 AgentLoop 最长的方法，实现了从收到消息到返回响应的完整流水线：

```
1. 会话恢复
   → get_or_create(session_key)
   → _restore_runtime_checkpoint()   恢复崩溃前的中间状态
   → _restore_pending_user_turn()    恢复未完成的用户轮次

2. 自动压缩
   → auto_compact.prepare_session()  如果会话过期，压缩旧消息

3. 斜杠命令
   → commands.dispatch()             如果是命令，直接返回结果

4. 内存合并
   → consolidator.maybe_consolidate_by_tokens()  token 预算触发合并

5. 上下文构建
   → context.build_messages()        组装完整消息列表

6. 用户消息提前持久化
   → session.add_message("user", msg.content)
   → _mark_pending_user_turn()
   → sessions.save()
   （防止进程崩溃丢失用户消息）

7. 运行 Agent 循环
   → _run_agent_loop()               委托给 AgentRunner

8. 保存轮次
   → _save_turn()                    保存新消息到会话
   → _clear_runtime_checkpoint()     清除检查点
   → sessions.save()                 持久化会话
   → _schedule_background(consolidator.maybe_consolidate_by_tokens())  后台合并

9. 返回响应
   → OutboundMessage(channel, chat_id, content)
```

**6.5 Agent 循环委托 `_run_agent_loop()`**

这个方法是把 AgentLoop 的"产品层关注点"和 AgentRunner 的"引擎层关注点"连接起来的桥梁：

1. 创建 `_LoopHook`（AgentLoop 自己的 Hook 实现），处理流式输出、进度推送、工具调用日志
2. 如果有 `_extra_hooks`，用 `CompositeHook` 组合
3. 定义 `_checkpoint()` 回调——把 Runner 的检查点持久化到 session metadata
4. 定义 `_drain_pending()` 回调——从 pending_queue 中取出中途注入的消息
5. 构造 `AgentRunSpec`，调用 `self.runner.run(spec)`
6. 返回结果

**6.6 `_LoopHook` —— AgentLoop 的内置 Hook**

`_LoopHook` 是 AgentLoop 对 Hook 机制的具体使用，实现了 5 个钩子：

| 钩子 | 做什么 |
|---|---|
| `on_stream()` | 增量式流式输出：维护 `_stream_buf`，用 `strip_think()` 过滤 `<think>` 标签，只推送增量部分 |
| `on_stream_end()` | 调用通道的 `on_stream_end` 回调，清空 `_stream_buf` |
| `before_iteration()` | 更新 `_current_iteration`（供 `/stop` 等命令查询） |
| `before_execute_tools()` | 推送工具调用进度（`_tool_hint`），记录工具调用日志，设置工具上下文（channel/chat_id/message_id） |
| `after_iteration()` | 记录 token 用量日志 |
| `finalize_content()` | 用 `strip_think()` 过滤 `<think>` 标签 |

注意 `_LoopHook` 的 `_reraise=True`——它是系统关键 Hook，失败必须向上传播。

**6.7 崩溃恢复机制**

AgentLoop 实现了两层崩溃恢复：

**第一层：Runtime Checkpoint（运行时检查点）**

在 AgentRunner 执行过程中，每次工具调用前后都会通过 `checkpoint_callback` 持久化中间状态到 session metadata：

```python
checkpoint = {
    "phase": "awaiting_tools",           # 当前阶段
    "iteration": 3,                      # 当前迭代
    "model": "gpt-4",                    # 模型
    "assistant_message": {...},          # assistant 消息
    "completed_tool_results": [...],     # 已完成的工具结果
    "pending_tool_calls": [...],         # 待执行的工具调用
}
```

进程崩溃后，`_restore_runtime_checkpoint()` 把这些中间状态物化到 session history 中：
- 已完成的工具结果正常追加
- 未完成的工具调用补一个 "Error: Task interrupted" 结果

**第二层：Pending User Turn（待处理用户轮次）**

用户消息在 Agent 循环开始前就提前持久化（`_mark_pending_user_turn()`）。如果进程在 Agent 循环中崩溃，下次恢复时 `_restore_pending_user_turn()` 发现用户消息没有对应的 assistant 响应，补一个 "Error: Task interrupted" 响应。

**6.8 会话保存 `_save_turn()`**

保存轮次时有几个关键处理：
- **跳过已提前持久化的用户消息**（`skip` 参数）
- **截断过长的工具结果**（超过 `max_tool_result_chars`）
- **剥离 runtime context**（用户消息中的 `[Runtime Context...]` 块不持久化，因为下次时间不同）
- **替换 base64 图片为占位符**（图片数据太大，持久化用路径引用代替）
- **跳过空 assistant 消息**（会污染会话上下文）

**6.9 系统消息处理**

`_process_message()` 对 `channel == "system"` 的消息有特殊处理：
- 系统消息的 `chat_id` 格式为 `"channel:chat_id"`，从中解析出真正的通道和聊天 ID
- 子代理结果（`sender_id == "subagent"`）在 prompt 组装前持久化到 session history，防止重复注入
- 子代理内容作为 `current_role="assistant"` 而非 `"user"` 注入

**6.10 其他入口**

`process_direct()` —— 一个同步式入口，不经过消息总线，直接构造 `InboundMessage` 调用 `_process_message()`。用于 CLI 等不需要异步消息总线的场景。

### 为什么设计

**为什么 AgentLoop 要组装这么多子系统？**

AgentLoop 是"产品层"——它不实现任何核心算法，但负责把所有子系统组装成可运行的产品。这遵循了"组合优于继承"的原则：每个子系统（Runner、Context、Memory、Skills、Subagent）都是独立的，AgentLoop 把它们"粘"在一起。

**为什么需要会话锁（`_session_locks`）？**

同一会话可能同时收到多条消息（用户快速连发）。没有锁的话，两个任务并发修改同一个 session 的 messages，会导致数据损坏。锁确保同一会话内串行处理。

**为什么需要并发门（`_concurrency_gate`）？**

不同会话可以并发，但无限制并发可能导致 OOM（每个会话都持有大量消息和 LLM 连接）。Semaphore 限制同时处理的会话数（默认 3），超出则排队等待。

**为什么 pending_queue 的残留消息要重新发布到 bus？**

任务结束后，pending_queue 中可能还有未被 AgentRunner 消费的消息（比如 Agent 循环已经结束，但用户又发了一条）。如果直接丢弃，用户消息就丢了。重新发布到 bus 让它们作为新消息被正常处理。

**为什么用户消息要提前持久化？**

如果进程在 Agent 循环中崩溃（OOM、SIGKILL、自重启），runtime checkpoint 保存了 assistant/tool 的中间状态，但**不包含触发这次循环的用户消息本身**。提前持久化确保崩溃恢复后，用户消息不会丢失。

**为什么 `_save_turn` 要剥离 runtime context？**

runtime context 包含当前时间、通道 ID 等瞬态信息。持久化到 session history 没有意义——下次加载时时间已经变了。剥离它节省存储空间，也避免旧的时间信息污染新对话。

**为什么需要两层崩溃恢复？**

- **Runtime Checkpoint**：恢复"Agent 正在执行工具"的中间状态——已完成的工具结果保留，未完成的标记为中断
- **Pending User Turn**：恢复"用户发了消息但 Agent 还没开始处理"的状态——补一个中断响应

两层覆盖了两种崩溃场景：Agent 循环中崩溃（有 checkpoint）和 Agent 循环前崩溃（只有用户消息）。

**为什么 `_LoopHook` 要增量式流式输出？**

LLM 流式返回的 token 可能包含 `<think>...</think>` 标签（某些模型的推理过程）。`_LoopHook` 维护一个 `_stream_buf` 累积全部内容，用 `strip_think()` 过滤后，只推送增量部分给通道。这样用户看到的是干净的、逐字出现的输出，而不是突然跳出的 `<think>` 标签内容。

**为什么子代理结果要在 prompt 组装前持久化？**

子代理结果通过系统消息到达，如果不在 prompt 组装前写入 session history，ContextBuilder 的 `_merge_message_content()` 可能把它和相邻的同角色消息合并，导致子代理结果在 session history 中"消失"——LLM 能看到（在合并后的 prompt 中），但持久化时找不到。

### 在系统中充当什么角色

**AgentLoop 是整个 Agent 系统的"总指挥"——它不实现任何核心算法，但负责编排所有子系统的协作。**

```
消息总线（MessageBus）
    ↓ 消费消息
AgentLoop.run()
    ↓ 分派
AgentLoop._dispatch()
    ↓ 会话锁 + 并发门
AgentLoop._process_message()
    ↓ 流水线：
    ├─ SessionManager        → 会话恢复/保存
    ├─ AutoCompact           → 过期会话压缩
    ├─ CommandRouter         → 斜杠命令处理
    ├─ Consolidator          → token 预算合并
    ├─ ContextBuilder        → 上下文组装
    ├─ AgentRunner           → ReAct 循环执行
    ├─ _save_turn()          → 轮次保存
    └─ Consolidator(后台)    → 后台合并
    ↓
OutboundMessage → MessageBus → 通道 → 用户
```

具体角色：
- **对通道层**：AgentLoop 是通道唯一需要交互的对象——通道只需把消息发到 bus，AgentLoop 负责处理和响应
- **对 AgentRunner**：AgentLoop 是 Runner 的调用方，负责准备 Spec、提供 Hook/Checkpoint/Injection 回调、处理 Result
- **对 SessionManager**：AgentLoop 管理会话的完整生命周期——创建、恢复、保存、压缩
- **对崩溃恢复**：AgentLoop 实现了两层恢复机制，确保进程崩溃后用户不丢失消息和上下文
- **对并发控制**：AgentLoop 通过会话锁和并发门，在正确性和性能之间取得平衡

AgentLoop 是"引擎层"和"外部世界"之间的**编排层**——外部世界只需要发消息和收响应，所有复杂性（会话、内存、工具、崩溃恢复、并发控制）都封装在 AgentLoop 内部。

---
