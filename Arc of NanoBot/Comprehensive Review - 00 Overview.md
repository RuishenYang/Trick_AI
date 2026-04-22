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
