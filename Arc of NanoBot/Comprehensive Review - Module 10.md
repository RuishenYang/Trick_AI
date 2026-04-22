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
