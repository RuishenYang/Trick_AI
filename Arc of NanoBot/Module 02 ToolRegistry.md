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
