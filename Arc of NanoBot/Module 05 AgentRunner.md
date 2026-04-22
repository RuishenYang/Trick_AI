## 模块 5：AgentRunner（`runner.py`）

### 怎么设计的

AgentRunner 是整个系统最核心的模块，实现了完整的 ReAct 循环。它由三个数据类 + 一个主类组成。

**5.1 两个数据类：AgentRunSpec 和 AgentRunResult**

**AgentRunSpec** —— 一次运行的配置包（输入）：

| 字段 | 含义 |
|---|---|
| `initial_messages` | 初始消息列表（system + history + user） |
| `tools` | ToolRegistry 实例 |
| `model` | 模型名 |
| `max_iterations` | 最大迭代次数 |
| `max_tool_result_chars` | 工具结果最大字符数 |
| `hook` | 生命周期钩子 |
| `concurrent_tools` | 是否允许工具并行执行 |
| `fail_on_tool_error` | 工具错误是否致命 |
| `context_window_tokens` | 上下文窗口大小 |
| `injection_callback` | 中途注入消息的回调（用于 mid-turn injection） |
| `checkpoint_callback` | 检查点回调（用于崩溃恢复） |

**AgentRunResult** —— 一次运行的结果包（输出）：

| 字段 | 含义 |
|---|---|
| `final_content` | 最终文本内容 |
| `messages` | 完整消息历史 |
| `tools_used` | 使用过的工具名列表 |
| `usage` | token 用量统计 |
| `stop_reason` | 停止原因（completed / max_iterations / error / tool_error / empty_final_response） |
| `had_injections` | 是否有中途注入 |

**5.2 主循环 `run()` 方法**

这是整个 Agent 系统的心脏。按流程图的方式拆解：

```
for iteration in range(max_iterations):

    ┌─────────────────────────────────────────────┐
    │ 1. 上下文治理（Context Governance）           │
    │    messages_for_model = messages 的安全副本   │
    │    → drop_orphan_tool_results()  清除孤儿    │
    │    → backfill_missing_tool_results() 补缺     │
    │    → microcompact()  旧工具结果摘要化         │
    │    → apply_tool_result_budget()  截断过长结果  │
    │    → snip_history()  历史裁剪适配上下文窗口   │
    │    → 再次 drop_orphan + backfill（裁剪可能产生新孤儿）│
    └─────────────────────────────────────────────┘
                    ↓
    ┌─────────────────────────────────────────────┐
    │ 2. Hook: before_iteration()                  │
    └─────────────────────────────────────────────┘
                    ↓
    ┌─────────────────────────────────────────────┐
    │ 3. 调用 LLM                                  │
    │    如果 hook.wants_streaming() → 流式调用     │
    │    否则 → 普通调用                            │
    └─────────────────────────────────────────────┘
                    ↓
    ┌─────────────────────────────────────────────┐
    │ 4. 分支判断                                   │
    │                                              │
    │  A) LLM 返回了工具调用（should_execute_tools）│
    │     → 保存 assistant 消息到 history           │
    │     → Hook: before_execute_tools()           │
    │     → 执行工具（_execute_tools）              │
    │     → 将工具结果追加到 history                │
    │     → 如果有致命错误 → break                  │
    │     → 检查中途注入（injection）               │
    │     → Hook: after_iteration()                │
    │     → continue（进入下一轮迭代）              │
    │                                              │
    │  B) LLM 返回了纯文本（最终响应）              │
    │     → finalize_content()                     │
    │     → 处理空响应重试                          │
    │     → 处理输出截断恢复                        │
    │     → 检查中途注入                            │
    │     → Hook: on_stream_end()                  │
    │     → break（循环结束）                       │
    └─────────────────────────────────────────────┘

循环结束后 → 返回 AgentRunResult
```

**5.3 上下文治理（5 个方法）**

这是 Runner 最精巧的部分，每次迭代前对消息列表做"手术"，确保发给 LLM 的消息合法且不超 token 预算：

| 方法 | 做什么 | 为什么 |
|---|---|---|
| `_drop_orphan_tool_results()` | 删除没有对应 tool_call 的 tool result 消息 | 历史裁剪可能把 assistant 消息删了，但它的 tool result 还在，LLM 会报错 |
| `_backfill_missing_tool_results()` | 为有 tool_call 但没 tool result 的补一个占位结果 | 同上，assistant 消息在但 tool result 被删了，LLM 要求每个 tool_call 都有对应 result |
| `_microcompact()` | 把旧的"可压缩"工具结果替换为一行摘要 | read_file/exec 等工具结果可能很长，旧的不需要全文，省 token |
| `_apply_tool_result_budget()` | 截断超长的工具结果 | 单个工具结果可能几十万字符，必须截断 |
| `_snip_history()` | 从旧到新裁剪历史消息，适配上下文窗口 | 对话太长时，保留 system + 最近的消息，丢弃旧的 |

注意：这些操作都在 `messages_for_model`（副本）上进行，**不修改原始 `messages`**。注释说得很清楚：

> Keep the persisted conversation untouched. Context governance may repair or compact historical messages for the model, but those synthetic edits must not shift the append boundary used later when the caller saves only the new turn.

**5.4 工具执行（3 个方法）**

`_execute_tools()` → 分批执行工具：
- `_partition_tool_batches()` 把工具调用分成批次：`concurrency_safe` 的工具放在同一批（可并行），非安全的独占一批
- 并行批用 `asyncio.gather()` 并发执行
- 非并行批顺序执行

`_run_tool()` → 执行单个工具：
1. 检查重复外部查找（防止 LLM 反复调用同一个 web_search）
2. 调用 `prepare_call()` 做参数准备
3. 执行 `tool.execute(**params)`
4. 错误处理：区分 `fail_on_tool_error`（致命 vs 非致命）
5. 返回三元组 `(result, event, fatal_error)`

`_normalize_tool_result()` → 结果归一化：
1. `ensure_nonempty_tool_result()` 确保非空
2. `maybe_persist_tool_result()` 大结果写入临时文件，消息中只留引用
3. 超长截断

**5.5 中途注入（Mid-turn Injection）**

`_try_drain_injections()` / `_drain_injections()` —— 这是"对话中追话"的机制：

当 Agent 正在处理一个请求时，用户又发了一条消息，这条消息不会丢失，而是通过 `injection_callback` 注入到当前对话中。Runner 在三个时机检查注入：
1. 工具执行后
2. 最终响应后
3. 错误后

注入有上限保护：每轮最多 3 条（`_MAX_INJECTIONS_PER_TURN`），最多 5 轮注入循环（`_MAX_INJECTION_CYCLES`），防止无限循环。

**5.6 其他容错机制**

| 机制 | 常量 | 做什么 |
|---|---|---|
| 空响应重试 | `_MAX_EMPTY_RETRIES = 2` | LLM 返回空内容时重试，最多 2 次 |
| 截断恢复 | `_MAX_LENGTH_RECOVERIES = 3` | `finish_reason="length"` 时追加"请继续"消息，最多 3 次 |
| 最终化重试 | `_request_finalization_retry()` | 空响应重试耗尽后，不带工具再调一次 LLM，强制生成文本 |
| 重复查找拦截 | `repeated_external_lookup_error()` | LLM 反复调用同一个 web_search 同一个参数时拦截 |

### 为什么设计

**为什么上下文治理要 5 步而不是简单截断？**

简单截断（只保留最近 N 条消息）会导致 LLM 协议违规：
- 删了 assistant 消息但没删它的 tool result → LLM 报错"tool result 没有 tool_call"
- 删了 tool result 但没删 assistant 的 tool_call → LLM 报错"tool_call 没有 result"
- 删了 system 消息 → LLM 不知道自己是谁

5 步治理确保发给 LLM 的消息**始终协议合法**：无孤儿、无缺失、不超预算。

**为什么 microcompact 只压缩特定工具？**

`_COMPACTABLE_TOOLS = {"read_file", "exec", "grep", "glob", "web_search", "web_fetch", "list_dir"}`

这些工具的结果是"查阅型"的——LLM 看过一次就够了，旧结果不需要全文。但 `write_file`、`edit_file` 的结果不能压缩——LLM 可能需要回顾"我刚才写了什么"。

**为什么工具分批执行？**

LLM 可能一次返回多个工具调用。如果都是只读的（read_file + grep），可以并行加速。但如果有写操作（edit_file），必须独占执行，否则并行写文件会冲突。`_partition_tool_batches()` 按安全性分批，兼顾性能和正确性。

**为什么需要中途注入？**

没有注入机制时，如果 Agent 正在执行一个 10 步的任务，用户在第 3 步时说"等等，换个方式"——这条消息只能等 Agent 完成后才处理，浪费了 7 步的计算。注入机制让用户可以在 Agent 工作时实时干预。

**为什么 `messages_for_model` 和 `messages` 要分开？**

`messages` 是持久化的对话历史，`messages_for_model` 是给 LLM 的"视图"。治理操作（裁剪、压缩、补缺）只影响视图，不影响持久化。这样保存对话时，只追加新消息，不会因为治理修改了旧消息而导致保存逻辑混乱。

### 在系统中充当什么角色

**AgentRunner 是 ReAct 循环的"引擎"——它不关心"谁在用"（CLI/Web/Discord），不关心"用什么模型"（GPT/Claude/GLM），只关心"怎么循环"。**

```
AgentLoop（产品层：会话管理、通道路由、命令处理）
    ↓ 委托
AgentRunner（引擎层：ReAct 循环、上下文治理、工具执行、容错）
    ↓ 依赖
LLMProvider（模型层：API 调用）
ToolRegistry（工具层：注册、校验、执行）
AgentHook（扩展层：流式输出、计费、审计）
```

AgentRunner 是"引擎层"的唯一模块。它把 ReAct 循环的所有复杂性（上下文治理、容错、注入、检查点）封装在一个 `run()` 方法中，让上层的 AgentLoop 只需要准备 Spec、调用 run、处理 Result。

---
