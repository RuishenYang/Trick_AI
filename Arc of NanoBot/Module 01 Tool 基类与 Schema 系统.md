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
