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
