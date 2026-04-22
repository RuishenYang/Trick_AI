# Arc of NANOBOT

> 本文档记录 nanobot agent 系统的架构拆解，聚焦三个核心问题：**怎么设计的**、**为什么设计**、**在系统中充当什么角色**。

---

## 整体架构概览

```
用户消息 → MessageBus → AgentLoop → SessionManager（会话隔离）
  → ContextBuilder（组装提示+历史+内存+技能）
  → AgentRunner（ReAct 循环）
    → LLM 调用 → 工具执行 → 结果反馈 → 重复
  → 响应 → MessageBus → 通道 → 用户
```

分层关系：

```
AgentLoop（产品层：会话管理、通道路由、命令处理）
    ↓ 委托
AgentRunner（引擎层：ReAct 循环、上下文治理、工具执行、容错）
    ↓ 依赖
LLMProvider（模型层：API 调用）
ToolRegistry（工具层：注册、校验、执行）
AgentHook（扩展层：流式输出、计费、审计）
```

---

## 模块 1：Tool 基类与 Schema 系统（`tools/base.py` + `tools/schema.py`）

### 怎么设计的

这个模块由三层构成：

**第一层：Schema 抽象基类**（`base.py:21-114`）

`Schema` 是一个 ABC，定义了两个核心能力：
- `to_json_schema()` → 将自身序列化为 JSON Schema 字典（给 LLM 看的格式）
- `validate_value(value)` → 校验一个值是否符合 schema

它还提供了两个**静态方法**作为所有子类共享的基础设施：
- `validate_json_schema_value(val, schema)` → 递归校验任意 JSON Schema 片段，支持 type/enum/minimum/maxLength/minItems/required 等约束
- `fragment(value)` → 统一归一化：如果传入的是 Schema 实例就调 `to_json_schema()`，如果已经是 dict 就直接返回

**第二层：具体 Schema 类型**（`schema.py`）

6 个具体子类，每个对应 JSON Schema 的一个类型：

| 类 | JSON Schema type | 特有约束 |
|---|---|---|
| `StringSchema` | string | min_length, max_length, enum |
| `IntegerSchema` | integer | minimum, maximum, enum |
| `NumberSchema` | number | minimum, maximum, enum |
| `BooleanSchema` | boolean | default |
| `ArraySchema` | array | items（子元素 schema）, min_items, max_items |
| `ObjectSchema` | object | properties, required, additional_properties |

每个都支持 `nullable`（生成 `["type", "null"]` 联合类型）。

还有一个便捷函数 `tool_parameters_schema()`，一行代码构建完整的 `{"type": "object", "properties": {...}, "required": [...]}`。

**第三层：Tool 抽象基类**（`base.py:117-243`）

`Tool` 是所有工具的 ABC，定义了 5 个抽象成员 + 3 个可选属性 + 3 个方法：

```
抽象（必须实现）：
  name: str          → 工具名（LLM 调用时用）
  description: str   → 工具描述（LLM 决定是否调用时用）
  parameters: dict   → JSON Schema（告诉 LLM 参数格式）
  execute(**kwargs)  → 实际执行逻辑

可选属性：
  read_only          → 是否无副作用（默认 False）
  concurrency_safe   → 是否可并行（默认 = read_only and not exclusive）
  exclusive          → 是否必须独占执行（默认 False）

方法：
  cast_params()      → 执行前的类型转换（如 "123" → 123）
  validate_params()  → 执行前的参数校验
  to_schema()        → 生成 OpenAI function calling 格式
```

**第四层：`@tool_parameters` 装饰器**（`base.py:246-279`）

这是一个类装饰器，让你用声明式方式定义 parameters，而不必手写 `@property`：

```python
@tool_parameters({
    "type": "object",
    "properties": {"path": {"type": "string"}},
    "required": ["path"],
})
class ReadFileTool(Tool):
    ...
```

它做了三件事：
1. `deepcopy` 冻结 schema 防止意外修改
2. 注入 `parameters` property（每次返回深拷贝）
3. 从 `__abstractmethods__` 中移除 `parameters`，这样子类就不会因为没实现 `parameters` 而报错

### 为什么设计

**为什么需要 Schema 系统？**

LLM 的 function calling 协议要求工具参数以 **JSON Schema** 格式声明。但纯手写 JSON Schema dict 有两个问题：
1. **没有校验**：手写 dict 无法在 Python 侧验证 LLM 传回的参数是否合法
2. **没有复用**：每个工具重复写 `{"type": "string", "description": "..."}` 很冗余

Schema 类系统解决了这两个问题：Python 侧有类型安全的构造器 + 校验方法，同时能序列化为 LLM 需要的 JSON Schema dict。

**为什么需要 `cast_params`？**

LLM 返回的参数类型不一定准确——可能把整数返回成字符串 `"123"`，或把布尔返回成 `"true"`。`cast_params` 在校验前做宽容的类型转换，避免因 LLM 的格式偏差导致工具执行失败。

**为什么需要 `read_only` / `concurrency_safe` / `exclusive`？**

当 LLM 在一次响应中返回多个工具调用时，系统需要决定哪些可以并行执行：
- `read_only=True` 的工具（如读文件、搜索）无副作用，可以并行
- `exclusive=True` 的工具（如写文件）必须独占
- `concurrency_safe` 是综合判断：只读且非独占 → 可并行

**为什么用装饰器而不是直接写 property？**

减少样板代码。没有装饰器的话，每个 Tool 子类都要写：
```python
@property
def parameters(self):
    return {"type": "object", "properties": {...}}
```
装饰器一行搞定，且自动处理深拷贝和抽象方法集合。

### 在系统中充当什么角色

**Tool + Schema 是整个 Agent 系统的"能力原子"。**

整个 Agent 的工作循环本质上是：LLM 思考 → 决定调用哪个工具 → 系统执行工具 → 把结果反馈给 LLM。Tool 就是这个循环中"执行"这一步的唯一载体。

具体来说：
- **对 LLM**：`to_schema()` 生成 function calling 的接口描述，LLM 据此知道有哪些工具可用、每个工具需要什么参数
- **对 AgentRunner**：`execute()` 是实际执行入口，`cast_params()` + `validate_params()` 是执行前的安全网
- **对 ToolRegistry**：Tool 实例是注册和查找的基本单元
- **Schema** 是 Tool 的"契约"——定义了 Tool 与 LLM 之间的参数协议

没有 Tool + Schema，Agent 就是一个只能说话不能做事的 LLM。它们是 Agent 从"语言模型"变成"智能体"的关键桥梁。

---

## 模块 2：ToolRegistry（`tools/registry.py`）

### 怎么设计的

`ToolRegistry` 是一个非常精炼的注册中心，核心数据结构就一个：

```python
self._tools: dict[str, Tool] = {}   # name → Tool 实例
self._cached_definitions: list | None = None  # 缓存的工具定义列表
```

它提供两组能力：

**第一组：注册/查找（CRUD）**

| 方法 | 作用 |
|---|---|
| `register(tool)` | 以 `tool.name` 为键存入字典，**同时清空缓存** |
| `unregister(name)` | 弹出工具，**同时清空缓存** |
| `get(name)` | 按名查找，返回 Tool 实例或 None |
| `has(name)` | 是否已注册 |

**第二组：定义生成与执行（核心业务）**

`get_definitions()` → 生成给 LLM 的工具定义列表，有两个关键设计：
1. **排序策略**：内置工具按名字排序在前，MCP 工具（`mcp_` 前缀）排序在后
2. **缓存**：结果缓存到 `_cached_definitions`，直到下次 register/unregister 才失效

`prepare_call(name, params)` → 执行前的"准备"步骤，返回三元组 `(tool, cast_params, error)`：
1. 特殊处理 `write_file`/`read_file` 的参数类型错误（LLM 有时传 list 而非 dict）
2. 查找工具，找不到则返回错误 + 可用工具列表
3. 调用 `tool.cast_params()` 做类型转换
4. 调用 `tool.validate_params()` 做参数校验
5. 全部通过则返回 `(tool, cast_params, None)`

`execute(name, params)` → 完整的执行流程：
1. 调用 `prepare_call()` 做准备
2. 如果准备阶段有错误，直接返回错误 + 提示
3. 调用 `tool.execute(**params)`
4. 如果结果以 `"Error"` 开头，追加提示
5. 异常捕获，返回错误信息 + 提示

其中 `_HINT = "\n\n[Analyze the error above and try a different approach.]"` 是一个固定的错误后缀，引导 LLM 自行分析错误并换策略。

### 为什么设计

**为什么需要 Registry 模式而不是直接用 dict？**

如果只是存取 Tool，一个 dict 就够了。但 Registry 封装了三个关键逻辑：

1. **定义生成与缓存**：`get_definitions()` 每次被调用时（每次 LLM 调用都需要），如果工具集没变就不必重新生成。缓存让 LLM prompt 中的工具定义部分保持稳定，减少 token 波动。

2. **排序策略**：内置工具在前、MCP 工具在后，且各自按名字排序。这确保了 LLM 每次看到的工具列表顺序一致。LLM 对 prompt 顺序敏感——如果工具定义每次顺序不同，LLM 的行为可能不稳定。

3. **prepare → execute 的两阶段设计**：`prepare_call` 把"准备"和"执行"分开。这让调用方（AgentRunner）可以在执行前拿到校验结果，决定是否继续。同时 `execute()` 方法是一个完整的"一站式"入口，简单场景直接调就行。

**为什么错误信息要追加 `_HINT`？**

这是 ReAct 循环的关键设计。当工具执行失败时，错误信息不是简单地返回给 LLM，而是附带一个提示"分析上面的错误并尝试不同的方法"。这引导 LLM 进入"反思"模式——不是盲目重试，而是分析原因后换策略。这是 ReAct 模式中"Reason"的体现。

**为什么 `write_file`/`read_file` 有特殊处理？**

某些 LLM（尤其是较弱的模型）在 function calling 时会把参数传成位置参数（list）而非关键字参数（dict）。这是已知的 LLM 行为偏差，Registry 在这里做了防御性处理。

### 在系统中充当什么角色

**ToolRegistry 是 Agent 系统中"工具世界"的入口和门面。**

它在系统中的位置可以这样理解：

```
LLM 返回 tool_call(name, params)
        ↓
   ToolRegistry.prepare_call()   ← 查找 + 转换 + 校验（门卫）
        ↓
   ToolRegistry.execute()        ← 执行 + 错误处理（执行者）
        ↓
   结果返回给 AgentRunner → 喂回 LLM
```

具体角色：
- **对 AgentLoop**：AgentLoop 在初始化时创建 Registry，注册所有默认工具
- **对 AgentRunner**：Runner 在每次迭代中通过 Registry 查找和执行工具
- **对 ContextBuilder**：通过 `get_definitions()` 获取工具定义，注入到 LLM 的 prompt 中
- **对 MCP 集成**：MCP 工具通过 `register()` 动态加入，通过 `unregister()` 动态移除
- **对 Subagent**：子代理创建自己的 Registry 子集（不含 message/spawn 等工具）

Registry 是"工具层"和"循环层"之间的**解耦点**——Runner 不需要知道有哪些工具、工具怎么校验、工具怎么排序，它只跟 Registry 交互。

---

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

## 模块 4：ContextBuilder（`context.py`）

### 怎么设计的

ContextBuilder 做一件事：**把散落在各处的信息组装成 LLM 能理解的消息列表**。它有两个核心方法，分别构建 system prompt 和完整消息列表。

**方法 1：`build_system_prompt()` —— 构建系统提示**

系统提示由 6 个部分按顺序拼接，用 `---` 分隔：

```
1. Identity（身份）        → 模板渲染：你是谁、工作目录、运行时环境、平台策略
2. Bootstrap Files（引导文件）→ 从工作目录读取 AGENTS.md / SOUL.md / USER.md / TOOLS.md
3. Memory（记忆）          → 从 MEMORY.md 读取长期记忆
4. Active Skills（常驻技能）→ 加载 "always" 模式的 SKILL.md 全文
5. Skills Summary（技能摘要）→ 列出其他可用技能的名称+描述（不加载全文，省 token）
6. Recent History（近期历史）→ 从 history.jsonl 读取最近 50 条 Dream 处理过的历史摘要
```

**方法 2：`build_messages()` —— 构建完整消息列表**

```
[
  {role: "system",  content: build_system_prompt() 的结果},
  ...history,                                    ← 对话历史
  {role: "user",   content: runtime_ctx + 用户消息 + 图片},  ← 当前消息
]
```

其中 runtime context 是注入到用户消息前面的元数据块：

```
[Runtime Context — metadata only, not instructions]
Current Time: 2026-04-22 15:30:00
Channel: telegram
Chat ID: 12345
[/Runtime Context]
```

还有几个辅助方法：

| 方法 | 作用 |
|---|---|
| `_get_identity()` | 渲染身份模板（工作目录、OS、Python 版本、平台策略） |
| `_build_runtime_context()` | 构建运行时元数据块（时间、通道、会话摘要） |
| `_load_bootstrap_files()` | 读取工作目录中的 4 个引导文件 |
| `_build_user_content()` | 处理用户消息 + 图片（base64 编码） |
| `_merge_message_content()` | 合并同角色连续消息（某些 LLM 不允许连续同角色消息） |
| `add_tool_result()` | 向消息列表追加工具结果 |
| `add_assistant_message()` | 向消息列表追加助手消息 |

### 为什么设计

**为什么系统提示要分 6 个部分？**

每个部分解决不同的问题：

| 部分 | 解决什么问题 |
|---|---|
| Identity | LLM 需要知道"我是谁"——名字、能力边界、工作目录、当前平台。没有这些，LLM 不知道文件路径怎么写、命令怎么执行 |
| Bootstrap Files | 用户自定义的持久指令。AGENTS.md 是项目级指令（"这个项目用 React"），SOUL.md 是人格指令，USER.md 是用户偏好，TOOLS.md 是工具使用规则 |
| Memory | 跨会话的长期记忆。LLM 需要记住"用户喜欢用 TypeScript"这种跨对话信息 |
| Active Skills | 某些技能需要始终激活（如代码规范检查），全文注入确保 LLM 完整理解 |
| Skills Summary | 其他技能只给名称+描述，LLM 按需加载。省 token——如果把所有技能全文注入，system prompt 会非常长 |
| Recent History | Dream 处理过的历史摘要，让 LLM 了解"之前做过什么" |

**为什么 runtime context 注入在用户消息而不是 system prompt？**

注意这行注释：`[Runtime Context — metadata only, not instructions]`

这是**信任边界**的设计。System prompt 中的内容 LLM 会当作"指令"执行，但 runtime context 是元数据（当前时间、通道 ID），不是指令。放在用户消息中：
1. 语义正确——"现在是几点"是事实信息，不是指令
2. 安全——防止 LLM 把 "Channel: telegram" 当作指令执行
3. 灵活——每轮用户消息可以带不同的 runtime context（时间在变）

**为什么要有 `_merge_message_content`？**

某些 LLM 提供商（如 Anthropic）不允许连续两条同角色消息。当 history 的最后一条已经是 user 消息，而当前消息也是 user 消息时，需要合并成一条而不是追加。这个方法处理了字符串和 content block 两种格式的合并。

**为什么引导文件是 AGENTS.md / SOUL.md / USER.md / TOOLS.md 这四个？**

这是**关注点分离**：
- `AGENTS.md`：项目级——"这是 React 项目，用 pnpm"
- `SOUL.md`：人格级——"你是一个严谨的工程师"
- `USER.md`：用户级——"我喜欢中文回复"
- `TOOLS.md`：工具级——"执行 shell 命令前先确认"

不同粒度的指令分开管理，修改一个不影响其他。

**为什么 Memory 要检查 `_is_template_content`？**

如果用户没有自定义 MEMORY.md，它就是模板默认内容。注入默认模板内容没有信息量，反而浪费 token。所以先检查是否和模板一致，一致就跳过。

### 在系统中充当什么角色

**ContextBuilder 是 Agent 系统的"信息总装线"——把所有分散的信息源汇聚成 LLM 一次调用所需的完整输入。**

它在系统中的位置：

```
各信息源（散落各处）：
  Identity 模板    ─┐
  AGENTS.md        ─┤
  SOUL.md          ─┤
  USER.md          ─┤
  TOOLS.md         ─┤
  MEMORY.md        ─┤──→ ContextBuilder.build_system_prompt() ──→ system prompt
  SkillsLoader     ─┤
  history.jsonl    ─┘

  对话历史          ─┐
  当前用户消息      ─┤──→ ContextBuilder.build_messages() ──→ 完整消息列表
  图片/媒体         ─┤                                      ↓
  runtime context   ─┘                              传给 LLM Provider
```

具体角色：
- **对 AgentRunner**：Runner 在每次迭代前调 `build_messages()` 获取完整输入
- **对 MemoryStore**：ContextBuilder 是 Memory 的消费者，读取记忆和历史
- **对 SkillsLoader**：ContextBuilder 是 Skills 的消费者，加载技能内容
- **对用户**：引导文件是用户自定义 Agent 行为的主要入口——改 AGENTS.md 就能改变 Agent 的项目认知

ContextBuilder 是"信息层"和"执行层"之间的**汇聚点**——所有信息在这里汇合，然后统一交给 LLM。

---

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

## 模块 7：三层内存系统（`memory.py`）

### 怎么设计的

这个模块实现了三个层级的内存处理，从轻到重：**MemoryStore**（纯文件 I/O）→ **Consolidator**（轻量合并）→ **Dream**（深度处理）。

**7.1 MemoryStore —— 纯文件 I/O 层**

MemoryStore 是最底层的内存原语，只做文件读写，不做任何 LLM 调用。它管理 5 个文件：

| 文件 | 用途 | 格式 |
|---|---|---|
| `memory/MEMORY.md` | 长期记忆事实 | Markdown |
| `memory/history.jsonl` | 历史摘要条目 | JSONL（每行一个 JSON 对象） |
| `SOUL.md` | 人格定义 | Markdown |
| `USER.md` | 用户偏好 | Markdown |
| `memory/.cursor` / `memory/.dream_cursor` | 游标（记录处理进度） | 纯文本整数 |

核心方法：

| 方法 | 作用 |
|---|---|
| `read_memory()` / `write_memory()` | 读写 MEMORY.md |
| `read_soul()` / `write_soul()` | 读写 SOUL.md |
| `read_user()` / `write_user()` | 读写 USER.md |
| `get_memory_context()` | 读取 MEMORY.md 并格式化为注入上下文的文本 |
| `append_history(entry)` | 追加一条历史摘要到 history.jsonl，返回自增 cursor |
| `read_unprocessed_history(since_cursor)` | 读取 cursor 之后的所有未处理条目 |
| `compact_history()` | 如果条目超过 `max_history_entries`（默认 1000），丢弃最旧的 |
| `get_last_dream_cursor()` / `set_last_dream_cursor()` | 读写 Dream 处理进度游标 |

**history.jsonl 的格式**：每行一个 JSON 对象：

```json
{"cursor": 42, "timestamp": "2026-04-22 15:30", "content": "用户修改了 config.yaml 中的端口配置"}
```

cursor 是自增整数，用于 Consolidator 和 Dream 跟踪处理进度。

**遗留迁移**：`_maybe_migrate_legacy_history()` 在首次启动时，如果存在旧的 `HISTORY.md`（Markdown 格式），会解析并转换为 `history.jsonl`，然后备份旧文件。迁移后 dream_cursor 设为最后一条，避免 Dream 重放整个历史。

**GitStore 集成**：MemoryStore 持有一个 `GitStore` 实例，跟踪 `SOUL.md`、`USER.md`、`memory/MEMORY.md` 的变更，供 Dream 的 `_annotate_with_ages()` 使用。

**7.2 Consolidator —— 轻轻量 token 预算触发合并**

Consolidator 的职责是：当会话的 prompt token 数接近上下文窗口上限时，把旧消息摘要化后存入 history.jsonl，然后从 session.messages 中删除，释放 token 空间。

核心方法 `maybe_consolidate_by_tokens()` 的流程：

```
1. 估算当前 prompt token 数
2. 如果 < budget → 不需要合并，返回
3. 如果 >= budget → 进入合并循环（最多 5 轮）：
   a. 计算需要移除的 token 数（estimated - target，target = budget/2）
   b. pick_consolidation_boundary() → 找到一个用户轮次边界，移除足够多的旧 token
   c. _cap_consolidation_boundary() → 限制单次最多合并 60 条消息
   d. archive() → 调用 LLM 摘要化旧消息，追加到 history.jsonl
   e. 更新 session.last_consolidated 游标
   f. 重新估算 token 数
4. 将最后的摘要存入 session.metadata["_last_summary"]，供 AutoCompact 使用
```

**`pick_consolidation_boundary()`** 的关键设计：只在**用户轮次边界**处切割。即只在一个 user 消息之前截断，不会在 assistant/tool 消息中间截断。这确保被保留的消息始终以 user 消息开头，符合 LLM 的协议要求。

**`archive()`** 方法：调用 LLM 对旧消息做摘要。如果 LLM 调用失败，降级为 `raw_archive()`——直接把原始消息格式化后追加到 history.jsonl，不丢失数据。

**并发安全**：Consolidator 用 `weakref.WeakValueDictionary` 为每个 session 维护一个 `asyncio.Lock`，防止同一会话的合并操作并发执行。

**7.3 Dream —— 深度定时内存处理**

Dream 是最重量级的内存处理器，由 CronService 定时触发（不是每次对话都触发）。它做两件事：

1. **分析历史**：从 history.jsonl 中读取未处理的条目，用 LLM 分析哪些信息值得长期记忆
2. **编辑文件**：委托 AgentRunner 使用 read_file/edit_file 工具，对 MEMORY.md、SOUL.md、USER.md 做增量编辑

**两阶段流程 `run()`**：

```
Phase 1: Analyze（纯 LLM 调用，无工具）
  输入：
    - 未处理的 history.jsonl 条目
    - 当前 MEMORY.md 内容（带行龄标注）
    - 当前 SOUL.md / USER.md 内容
  输出：
    - 分析结果文本（哪些信息需要记住、哪些需要更新、哪些过时了）

Phase 2: Edit（AgentRunner + 工具调用）
  输入：
    - Phase 1 的分析结果
    - 当前文件内容
    - 已有技能列表（防止重复创建）
  工具：
    - read_file：读取文件
    - edit_file：编辑文件
    - write_file：创建新技能文件（仅限 skills/ 目录）
  输出：
    - 工具调用事件列表（哪些文件被修改了）
```

**行龄标注 `_annotate_with_ages()`**：Dream 在 Phase 1 的输入中，对 MEMORY.md 的每一行标注"距上次修改多少天"。超过 14 天（`_STALE_THRESHOLD_DAYS`）的行会标注 `← 30d` 这样的后缀。这让 LLM 知道哪些记忆已经过时，可以清理或更新。

行龄数据来自 `GitStore.line_ages()`——通过 git blame 计算每行的修改时间。如果 git 不可用或行数不匹配（工作区有未提交修改），则跳过标注。

**工具注册**：Dream 创建自己的 ToolRegistry，只包含 read_file、edit_file、write_file 三个工具。write_file 限制只能写入 `skills/` 目录，防止 Dream 意外修改其他文件。

**游标推进**：无论 Phase 2 是否成功，Dream 都会推进 dream_cursor。这避免 Phase 1 失败后反复重试同一批历史。

**Git 自动提交**：如果 Phase 2 产生了文件修改，Dream 会通过 `GitStore.auto_commit()` 自动提交，提交信息包含时间戳和变更摘要。

**7.4 三层协作关系**

```
对话进行中：
  AgentRunner 执行工具 → 产生消息
  AgentLoop._save_turn() → 保存到 session.messages
  AgentLoop._schedule_background(consolidator.maybe_consolidate_by_tokens())
    → Consolidator 检查 token 预算
    → 如果超预算 → archive() → LLM 摘要 → 追加到 history.jsonl
    → 从 session.messages 中删除旧消息

定时触发（CronService）：
  Dream.run()
    → 读取 history.jsonl 中未处理条目
    → Phase 1: LLM 分析
    → Phase 2: AgentRunner + 工具 → 编辑 MEMORY.md / SOUL.md / USER.md
    → 推进 dream_cursor
    → git auto-commit

下次对话：
  ContextBuilder.build_system_prompt()
    → MemoryStore.read_memory() → 读取 MEMORY.md（Dream 更新过的长期记忆）
    → MemoryStore.read_unprocessed_history() → 读取 Dream 未处理的近期历史
    → 注入到 system prompt
```

### 为什么设计

**为什么需要三层而不是一层？**

三层解决三个不同的问题：

| 层 | 解决什么问题 | 触发条件 | 代价 |
|---|---|---|---|
| MemoryStore | 数据持久化——内存数据不能只存在内存中 | 每次写入 | 几乎无（纯文件 I/O） |
| Consolidator | token 预算——会话太长时必须压缩，否则超出上下文窗口 | prompt token 接近上限 | 一次 LLM 调用（摘要化） |
| Dream | 长期记忆——跨会话的知识需要从对话中提取并持久化 | 定时（CronService） | 多次 LLM 调用 + 工具调用 |

如果只有 MemoryStore，会话无限增长最终超出上下文窗口。如果只有 Consolidator，长期记忆只能存在 session.messages 中，新会话无法继承旧会话的知识。Dream 把知识从"会话级"提升到"项目级"（MEMORY.md），让所有会话共享。

**为什么 Consolidator 只在用户轮次边界切割？**

如果在 assistant 消息或 tool result 中间切割，保留的消息可能以 assistant 或 tool 角色开头，违反 LLM 的协议要求（第一条非 system 消息必须是 user）。只在 user 消息前切割确保协议合法性。

**为什么 Consolidator 的 target 是 budget/2 而不是 budget？`

合并到 budget/2 而不是 budget，给后续对话留出增长空间。如果合并到 budget，下一次对话几轮迭代后又会触发合并，频繁合并浪费 LLM 调用。合并到一半，后续对话可以增长一倍才需要再次合并。

**为什么 Dream 要分两阶段？**

Phase 1（分析）和 Phase 2（编辑）分离有两个好处：
1. **成本控制**：Phase 1 是单次 LLM 调用，成本固定。Phase 2 可能多轮工具调用，成本较高。如果 Phase 1 判断"不需要修改"，可以跳过 Phase 2。
2. **关注点分离**：Phase 1 只做"思考"（分析哪些信息重要），Phase 2 只做"行动"（编辑文件）。这符合 ReAct 模式的 Reason → Act 分离。

**为什么 Dream 的 write_file 只能写 skills/ 目录？**

Dream 的 Phase 2 有 AgentRunner 的完整工具调用能力。如果不限制写入范围，LLM 可能意外修改源代码或配置文件。限制到 skills/ 目录确保 Dream 只能创建/修改技能文件，这是一种**最小权限**设计。

**为什么 Dream 无论 Phase 2 是否成功都推进 cursor？**

如果 Phase 2 失败后不推进 cursor，下次 Dream 触发时会重试同一批历史。如果失败是系统性的（如 LLM 服务不可用），会无限重试。推进 cursor 确保失败不会阻塞后续历史的处理。丢失的只是一次处理机会，历史数据仍在 history.jsonl 中。

**为什么行龄标注用 git blame？**

MEMORY.md 中的信息有时效性——"项目使用 React 18"可能一个月后过时了。但 LLM 不知道每条记忆是什么时候写入的。git blame 提供精确的每行修改时间，让 LLM 能判断哪些记忆过时了、需要更新或删除。

**为什么 Consolidator 降级为 raw_archive 而不是丢弃？**

当 LLM 摘要化失败时，选择保留原始消息（格式化后存入 history.jsonl）而不是丢弃。数据丢失是不可逆的——一旦丢弃，那些对话信息就永远消失了。raw_archive 虽然不如 LLM 摘要精炼，但至少保留了信息。

### 在系统中充当什么角色

**三层内存系统是 Agent 的"记忆器官"——让 Agent 不仅能记住当前对话，还能跨会话积累知识。**

```
短期记忆（会话级）：
  session.messages → Consolidator → history.jsonl（摘要化）

长期记忆（项目级）：
  history.jsonl → Dream → MEMORY.md / SOUL.md / USER.md（知识提取）

记忆消费：
  ContextBuilder → MemoryStore → 注入 system prompt
```

具体角色：
- **MemoryStore**：是所有内存操作的底层——ContextBuilder 从它读取记忆，Consolidator 和 Dream 向它写入历史
- **Consolidator**：是"短期记忆管理器"——确保会话不超出 token 预算，把旧消息摘要化后归档
- **Dream**：是"长期记忆构建器"——从历史摘要中提取知识，写入持久化文件，让所有会话共享

三层内存系统让 Agent 具备了类似人类的记忆能力：工作记忆（当前对话）、短期记忆（近期历史摘要）、长期记忆（项目知识）。没有它，Agent 每次对话都从零开始，无法积累经验。

---

## 模块 8：SkillsLoader（`skills.py`）

### 怎么设计的

SkillsLoader 负责发现和加载 SKILL.md 文件——这些文件教 Agent 如何使用特定工具或执行特定任务。

**技能的文件结构**：每个技能是一个目录，包含一个 `SKILL.md` 文件：

```
skills/
  code-review/
    SKILL.md
  testing/
    SKILL.md
```

**两个技能来源**：
- `workspace/skills/`：用户自定义的工作区技能
- `nanobot/skills/`：内置技能（随 nanobot 分发）

工作区技能**优先于**同名内置技能（`list_skills()` 中 `skip_names=workspace_names`）。

**SKILL.md 的格式**：YAML frontmatter + Markdown 正文：

```markdown
---
description: Code review best practices
metadata:
  nanobot:
    always: true
    requires:
      bins: [git]
      env: []
---

## Code Review Guidelines
...（技能正文，教 Agent 如何做代码审查）
```

**核心方法**：

| 方法 | 作用 |
|---|---|
| `list_skills(filter_unavailable=True)` | 列出所有可用技能，工作区优先，过滤不满足依赖的 |
| `load_skill(name)` | 按名加载技能全文 |
| `load_skills_for_context(names)` | 加载多个技能，去掉 frontmatter，格式化后用 `---` 分隔 |
| `build_skills_summary(exclude)` | 构建技能摘要列表（名称+描述+路径+可用性），用于渐进式加载 |
| `get_always_skills()` | 获取标记为 `always=true` 且满足依赖的技能名列表 |
| `get_skill_metadata(name)` | 解析 YAML frontmatter，返回元数据 dict |

**依赖检查 `_check_requirements()`**：技能可以声明依赖：
- `bins`：需要的 CLI 命令（如 `git`），用 `shutil.which()` 检查
- `env`：需要的环境变量，用 `os.environ.get()` 检查

不满足依赖的技能在 `list_skills()` 中被过滤，在 `build_skills_summary()` 中标记为 `(unavailable: ...)`。

**OpenClaw 兼容 `_parse_nanobot_metadata()`**：

```python
payload = data.get("nanobot", data.get("openclaw", {}))
```

先找 `nanobot` 键，找不到就找 `openclaw` 键。这确保 OpenClaw 格式的 SKILL.md 无需修改就能被 nanobot 解析。

**frontmatter 处理**：`_strip_frontmatter()` 用正则剥离 `---` 包裹的 YAML 块，只保留 Markdown 正文给 LLM。`get_skill_metadata()` 用 `yaml.safe_load()` 解析 frontmatter。

### 为什么设计

**为什么技能是 Markdown 文件而不是 Python 代码？**

技能是"知识"而不是"逻辑"——它们教 LLM 如何做某件事（如"代码审查时关注这些点"），而不是实现可执行逻辑。Markdown 是 LLM 最自然的理解格式，也方便用户阅读和编辑。

**为什么需要 frontmatter？**

frontmatter 存储机器可读的元数据（description、always、requires），正文存储 LLM 可读的指令。两者分离：元数据用于 SkillsLoader 的过滤和排序逻辑，正文用于注入 LLM 的 prompt。

**为什么工作区技能优先于内置技能？**

用户可能想覆盖内置技能的行为（如自定义代码审查规则）。工作区优先让用户只需在 `skills/` 目录下创建同名目录即可覆盖，不需要修改内置文件。

**为什么有 `always` 模式和渐进式加载？**

- `always=true` 的技能始终注入 system prompt（如安全规范），确保 LLM 每次都遵守
- 其他技能只给名称+描述（`build_skills_summary()`），LLM 按需用 `read_file` 加载全文

这避免了把所有技能全文注入 system prompt 导致 token 浪费。

**为什么需要依赖检查？**

某些技能依赖特定工具（如 git-review 技能需要 `git` CLI）。如果依赖不满足，注入技能全文也没用——LLM 会尝试调用不存在的工具。依赖检查在加载前就过滤掉不可用的技能，避免误导 LLM。

**为什么保留 OpenClaw 兼容？**

nanobot 的技能系统继承自 OpenClaw。已有的 OpenClaw 技能库（社区贡献的 SKILL.md 文件）无需修改就能在 nanobot 中使用，降低迁移成本。

### 在系统中充当什么角色

**SkillsLoader 是 Agent 的"能力目录"——告诉 LLM 有哪些技能可用，按需加载技能内容。**

```
技能发现：
  SkillsLoader.list_skills() → 发现所有可用技能
  SkillsLoader.get_always_skills() → 获取常驻技能

技能消费（在 ContextBuilder 中）：
  build_system_prompt()
    → Active Skills: load_skills_for_context(always_skills)  ← 常驻技能全文注入
    → Skills Summary: build_skills_summary(exclude=always)   ← 其他技能只给摘要
```

具体角色：
- **对 ContextBuilder**：ContextBuilder 是 SkillsLoader 的消费者，在构建 system prompt 时调用
- **对 Dream**：Dream 在 Phase 2 中列出已有技能，防止重复创建
- **对用户**：用户通过在 `skills/` 目录下创建 SKILL.md 来扩展 Agent 能力，无需写代码

SkillsLoader 是"能力层"和"信息层"之间的**桥梁**——技能文件是静态的 Markdown，SkillsLoader 把它们变成 LLM 可消费的 prompt 内容。

---

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

## 模块 10：AutoCompact（`autocompact.py`）

### 怎么设计的

AutoCompact 主动压缩空闲会话，减少下次激活时的 token 成本和延迟。

**核心参数**：
- `session_ttl_minutes`：会话过期时间（分钟），0 表示禁用
- `_RECENT_SUFFIX_MESSAGES = 8`：压缩时保留最近 8 条消息

**10.1 过期检查 `check_expired()`**

在 AgentLoop 主循环的空闲时段（`bus.consume_inbound()` 超时 1 秒时）调用：

```python
def check_expired(self, schedule_background, active_session_keys=()):
    for info in sessions.list_sessions():
        if key in self._archiving: continue          # 正在归档中，跳过
        if key in active_session_keys: continue      # 有活跃任务，跳过
        if self._is_expired(info.updated_at):        # 检查是否过期
            self._archiving.add(key)
            schedule_background(self._archive(key))  # 后台归档
```

两个跳过条件：
1. 正在归档中（`_archiving` 集合）——防止重复归档
2. 有活跃任务（`active_session_keys`）——不能压缩正在使用的会话

**10.2 归档 `_archive()`**

```python
async def _archive(key):
    session = sessions.get_or_create(key)
    archive_msgs, kept_msgs = _split_unconsolidated(session)
    
    if archive_msgs:
        summary = await consolidator.archive(archive_msgs)  # LLM 摘要化
        _summaries[key] = (summary, last_active)             # 缓存摘要
    
    session.messages = kept_msgs          # 只保留最近消息
    session.last_consolidated = 0         # 重置合并游标
    sessions.save(session)
```

`_split_unconsolidated()` 的逻辑：从未合并的消息尾部，保留最近 8 条（`_RECENT_SUFFIX_MESSAGES`），其余归档。使用 `Session.retain_recent_legal_suffix()` 确保保留的消息协议合法（以 user 消息开头）。

**10.3 会话准备 `prepare_session()`**

当用户重新激活一个被压缩的会话时调用：

```python
def prepare_session(session, key) -> (Session, str | None):
    # 如果正在归档或已过期，重新加载会话
    if key in self._archiving or self._is_expired(session.updated_at):
        session = sessions.get_or_create(key)
    
    # 取出摘要（优先从内存缓存，其次从 session metadata）
    entry = self._summaries.pop(key, None)
    if entry:
        return session, _format_summary(entry[0], entry[1])
    if "_last_summary" in session.metadata:
        return session, _format_summary(...)
    
    return session, None  # 无摘要
```

`_format_summary()` 格式化摘要为：`"Inactive for X minutes.\nPrevious conversation summary: ..."`

这个摘要会被 ContextBuilder 注入到 runtime context 中，让 LLM 知道之前的对话内容。

### 为什么设计

**为什么需要 AutoCompact 而不是只靠 Consolidator？**

Consolidator 是**被动**的——只在 prompt token 接近上限时触发。AutoCompact 是**主动**的——在会话空闲时提前压缩。

区别在于：
- Consolidator 压缩的是"当前会话中太长的部分"，目的是让当前对话继续
- AutoCompact 压缩的是"整个空闲会话"，目的是让下次对话更快启动

没有 AutoCompact，一个长时间空闲的会话下次激活时，需要把所有旧消息加载到 prompt 中，浪费 token 和增加延迟。AutoCompact 提前把旧消息摘要化，下次只需注入摘要。

**为什么保留最近 8 条消息？**

完全清空会话会让 LLM 失去所有上下文。保留最近 8 条消息让 LLM 知道"最后在做什么"，用户可以自然地继续对话。8 条是一个经验值——足够保留最后一轮用户-助手交互的上下文，又不会太多导致 token 浪费。

**为什么摘要要同时存在内存和 session metadata 中？**

- **内存缓存 `_summaries`**：进程没重启时，直接从内存取，速度快
- **session metadata `_last_summary`**：进程重启后，内存缓存丢失，从持久化的 metadata 中恢复

两种存储确保摘要不会因进程重启而丢失。

**为什么 `prepare_session()` 要从 metadata 中清除 `_last_summary`？**

`_last_summary` 是一次性的——注入给 LLM 后就不需要了。如果不清除，摘要会一直留在 metadata 中，每次加载会话都重复注入。`pop()` 确保摘要只注入一次。

**为什么 `check_expired()` 在空闲时段调用而不是定时器？**

AgentLoop 的主循环用 `asyncio.wait_for(bus.consume_inbound(), timeout=1.0)` 等待消息。超时意味着没有新消息，是检查过期会话的好时机。这比单独的定时器更简单，也避免了定时器和消息处理的并发问题。

### 在系统中充当什么角色

**AutoCompact 是 Agent 的"会话管家"——在会话空闲时主动整理，让下次使用更高效。**

```
会话生命周期：
  活跃 → 用户发消息 → AgentLoop 处理
  空闲 → 超过 TTL → AutoCompact.check_expired()
    → _archive() → Consolidator.archive() → LLM 摘要
    → session.messages = 最近 8 条
    → 摘要存入 _summaries 和 metadata
  重新激活 → prepare_session()
    → 取出摘要 → 注入 runtime context
    → LLM 看到摘要 + 最近 8 条消息 → 无缝继续对话
```

具体角色：
- **对 AgentLoop**：AgentLoop 在空闲时调 `check_expired()`，在处理消息前调 `prepare_session()`
- **对 Consolidator**：AutoCompact 委托 Consolidator 做实际的 LLM 摘要化
- **对 SessionManager**：AutoCompact 修改 session.messages 和 metadata，通过 SessionManager 持久化
- **对 ContextBuilder**：prepare_session() 返回的摘要被注入 runtime context

AutoCompact 是"内存层"和"会话层"之间的**优化器**——它不改变 Agent 的功能，但通过主动压缩减少 token 成本和启动延迟。

---
