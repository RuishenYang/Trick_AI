## 模块 3：AgentHook 生命周期钩子（`hook.py`）

### 怎么设计的

这个模块由三个部分组成：

**1. AgentHookContext —— 每次迭代的可变状态包**

```python
@dataclass(slots=True)
class AgentHookContext:
    iteration: int                    # 当前是第几轮迭代
    messages: list[dict]              # 当前对话消息列表
    response: LLMResponse | None      # LLM 的响应
    usage: dict[str, int]             # token 用量统计
    tool_calls: list[ToolCallRequest] # 本轮 LLM 请求的工具调用
    tool_results: list[Any]           # 工具执行结果
    tool_events: list[dict]           # 工具执行事件日志
    final_content: str | None         # 最终输出内容
    stop_reason: str | None           # 停止原因
    error: str | None                 # 错误信息
```

这是一个 `slots=True` 的 dataclass，轻量且不可动态添加属性。它把一次迭代的**所有关键状态**打包在一起，作为 Hook 方法的唯一输入。

**2. AgentHook —— 钩子抽象基类**

定义了 6 个生命周期方法，全部有默认空实现：

```
before_iteration(ctx)        → 每次调用 LLM 之前
on_stream(ctx, delta)        → 流式输出每个 token 片段
on_stream_end(ctx, resuming) → 流式输出结束
before_execute_tools(ctx)    → 执行工具之前
after_iteration(ctx)         → 每次迭代结束之后
finalize_content(ctx, content) → 最终内容后处理（同步方法）
```

还有一个 `wants_streaming()` → 声明是否需要流式回调（默认 False）。

每个 Hook 实例有一个 `_reraise` 标志：默认 False（错误被吞掉），设为 True 则错误向上传播。

**3. CompositeHook —— 组合钩子**

继承自 `AgentHook`，持有一个 `hooks: list[AgentHook]`，按顺序委托给每个子钩子。

关键设计是**错误隔离**：所有 async 方法通过 `_for_each_hook_safe()` 执行，逻辑是：

```python
for h in self._hooks:
    if h._reraise:
        await h.method(...)      # 必须成功，否则抛异常
    else:
        try:
            await h.method(...)  # 失败只 log，不影响其他 hook
        except Exception:
            logger.exception(...)
```

唯一的例外是 `finalize_content()`——它是同步的管道式处理，**不做错误隔离**：

```python
for h in self._hooks:
    content = h.finalize_content(ctx, content)  # 链式传递，bug 应该暴露
```

### 为什么设计

**为什么需要 Hook 机制？**

AgentRunner 的核心循环是固定的：调 LLM → 执行工具 → 反馈结果 → 重复。但不同使用场景需要不同的"附加行为"：
- CLI 通道需要流式输出到终端
- Web 通道需要通过 WebSocket 推送进度
- 调试模式需要记录每轮的工具调用
- 计费系统需要统计 token 用量

如果把这些都写进 AgentRunner，Runner 会变成一个巨大的上帝类。Hook 机制让这些**横切关注点**从 Runner 中解耦出来——Runner 只管核心循环，Hook 负责所有"旁边的事"。

**为什么用 Context 对象而不是直接传参数？**

6 个钩子方法如果每个都传不同的参数，签名会很混乱。用一个 Context dataclass：
1. **签名统一**：所有方法都是 `async def method(ctx)`
2. **扩展方便**：加新状态只需改 Context，不用改所有 Hook 的签名
3. **Hook 可修改状态**：因为 Context 是可变的，Hook 可以修改 `messages`、`tool_calls` 等，影响后续行为

**为什么 CompositeHook 要做错误隔离？**

这是防御性设计。用户可能注册自定义 Hook，如果某个 Hook 抛异常，不应该让整个 Agent 循环崩溃。错误隔离确保：
- 一个坏 Hook 不影响其他 Hook
- 一个坏 Hook 不影响核心循环
- 错误被 log 记录，可事后排查

但 `finalize_content` 不隔离——因为它是内容变换管道，如果变换逻辑有 bug，应该立即暴露而不是静默吞掉。

**为什么 `_reraise` 标志存在？**

有些 Hook 是系统内置的（如流式输出 Hook），它们的失败是致命的，应该向上传播。`_reraise=True` 让这些关键 Hook 的错误不被吞掉。

### 在系统中充当什么角色

**AgentHook 是 AgentRunner 的"侧信道"——不影响核心循环逻辑，但在循环的每个关键节点插入自定义行为。**

它在 ReAct 循环中的位置：

```
before_iteration()     ← LLM 调用前（可修改 messages、注入额外上下文）
    ↓
调用 LLM
    ↓
on_stream() / on_stream_end()  ← 流式输出（推送到 UI）
    ↓
before_execute_tools() ← 工具执行前（可修改 tool_calls、拦截调用）
    ↓
执行工具
    ↓
after_iteration()      ← 迭代结束（记录日志、统计用量、触发通知）
    ↓
... 重复 ...
    ↓
finalize_content()     ← 最终输出前（内容后处理，如脱敏、格式化）
```

具体角色：
- **对 AgentRunner**：Runner 在循环的每个节点调 Hook，Hook 是 Runner 唯一的外部扩展点
- **对通道层**：通道通过注册 Hook 实现流式输出、进度推送等 UI 行为
- **对监控系统**：通过 Hook 收集 usage、tool_events 等指标
- **CompositeHook** 是"多通道并存"的基础——CLI Hook + Web Hook + 计费 Hook 可以同时工作互不干扰

Hook 机制让 AgentRunner 保持纯粹（只管循环逻辑），同时让所有"旁路行为"可插拔、可组合、可隔离。

**具体场景举例**：用户在 CLI 中让 Agent "读取 config.yaml 并修改端口为 8080"。

没有 Hook 时，用户输入后屏幕卡住 10 秒，然后突然出现最终结果——LLM 的思考过程、工具调用、执行结果全部在 Runner 内部默默发生。

有了 Hook 后，CLIStreamHook 在 `on_stream()` 中逐字打印 LLM 输出，在 `before_execute_tools()` 中显示工具调用进度；BillingHook 在 `after_iteration()` 中累加 token 用量；AuditHook 在 `after_iteration()` 中记录写文件操作到审计日志。这些行为通过 CompositeHook 组合，互不干扰，且任何一个失败不会让 Agent 循环崩溃。

Hook 的本质是**观察者模式 + 责任链**的混合体：观察者模式让多个监听者响应同一事件，责任链让 finalize_content 可以链式修改内容。而错误隔离让它比传统观察者模式更健壮。

---
