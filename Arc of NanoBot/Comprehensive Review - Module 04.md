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
