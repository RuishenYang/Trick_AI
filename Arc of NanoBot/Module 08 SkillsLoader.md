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
