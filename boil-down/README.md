# boil-down

一个用于 [Claude Code](https://claude.ai/code) 的 skill，让你在对话结束时说一句 **"boil down"**，Claude 就会自动把整个会话整理成结构化的笔记。

---

## 它解决什么问题

和 AI 一起工作时，一次会话可能做了很多事情——调 bug、写代码、分析数据、做决策。但一旦关掉对话窗口，上下文就消失了。下次开新会话，又得从头解释一遍。

**boil-down** 的作用是：在你说完"boil down"的瞬间，把整个会话自动转化成可以持久保存的笔记，让你（或下一个 AI）可以无缝接续。

---

## 说"boil down"之后会发生什么

Claude 读完整个对话，然后按顺序生成**四个文件**：

---

### 第一个文件 — 会话笔记（Session Note）

**路径：** `NOTES_DIR/sessions/YYYY-MM/YYYY-MM-DD/session-N.md`

这是本次会话的详细记录，包含：

- **编号摘要**（1–8 条）：本次完成了什么，附关键数据和结果
- **逐任务拆解**：每个任务的问题是什么、根本原因、解决步骤、结论
- **文件清单**：本次创建或修改的每个文件，附说明、操作时间、触发该操作的用户指令

同一个文件被多次修改时，会合并成一条记录。

摘要示例：
```
1. 修复用户注册接口的邮箱验证 bug（影响新注册流程）
2. 重构数据库查询逻辑，平均响应时间从 800ms 降至 120ms
3. 编写单元测试 12 个，覆盖核心业务逻辑
```

文件第一行是 `project:` 标签，Claude 根据对话内容自动检测（例如 `project: Research`、`project: WebApp`）。你可以在 `SKILL.md` 里自定义检测规则，匹配自己的项目。

---

### 第二个文件 — 每日日志（Daily Log）

**路径：** `NOTES_DIR/daily/thoughts-YYYY-MM-DD.md`

这是一天的流水账。每次 boil down，Claude 就在这个文件里追加一个简短的 block。文件不存在时自动创建。

每个 block 长这样：

```
**Session 1**: 修复登录 bug，优化查询性能，补充单元测试
📋 Session Note: `NOTES_DIR/sessions/2025-01-15/session-1.md`
🎯 Main Output: `NOTES_DIR/sessions/2025-01-15/api/auth_service.py`
🤝 Hand-off: `NOTES_DIR/sessions/2025-01-15/handoff-session-1.md`
- 定位并修复邮箱验证逻辑错误：正则表达式未处理带加号的邮箱地址
- 重构数据库查询：引入索引 + 批量查询，响应时间 800ms → 120ms
- 新增单元测试 12 个，覆盖登录、注册、密码重置三个模块
- 更新 API 文档，补充错误码说明
```

一眼就能看到当天每个 session 做了什么、产出在哪里。

---

### 第三个文件 — 交接文件（Hand-off）

**路径：** `NOTES_DIR/sessions/YYYY-MM/YYYY-MM-DD/handoff-session-N.md`

这个文件是**写给下一个 AI agent 的**（或者下次开新会话的你）。刻意控制在 30 行以内，只写下一个人需要知道的东西，不写过程细节。

内容结构：
- **任务描述**：1–2 句话，说清楚做了什么、背景是什么
- **主要产出**：核心交付文件和当前状态
- **关键文件**：最多 5 个文件路径，每个一行说明用途
- **注意事项**：重要的约束、决策、已知问题
- **未完成 / 下一步**：checkbox 列表，列出还没做完的事

示例：
```markdown
## 未完成 / 下一步

- [ ] 将分析结果整理成正式报告（参考模板格式）
- [ ] 补充统计检验（需安装 scipy）
- [ ] 更新项目 README，同步最新结论
```

如果本次没有未完成的事，就写：*"本 session 任务已全部完成。"*

---

### 第四个文件 — 待办同步（Backlog Sync）

**路径：** `NOTES_DIR/backlog.md`

Claude 读取交接文件里的未完成事项，同步到统一的待办列表。逻辑如下：

- 交接文件的 `- [ ]` 条目 → 追加到 `backlog.md` 对应项目的 section
- 如果已有语义相同的条目 → 跳过，不重复添加
- 如果本次 session 明确完成了 backlog 里的某条 → 移入 `## Archive` section，加 ✓ 标记
- 更新文件顶部的"上次更新"时间戳

如果本次没有未完成事项，`backlog.md` 不做任何修改。

---

## 整体流程一览

```
你说："boil down"
        │
        ▼
Claude 读完整个对话
        │
        ├──► session-N.md              （详细笔记：摘要 + 任务拆解 + 文件清单）
        │
        ├──► thoughts-YYYY-MM-DD.md    （每日日志：追加 6 行简短 block）
        │
        ├──► handoff-session-N.md      （交接文件：≤30 行，写给下一个 agent）
        │
        └──► backlog.md                （待办同步：新增未完成项 / 归档已完成项）
```

---

## 文件结构

```
NOTES_DIR/
├── sessions/
│   └── 2025-01/
│       └── 2025-01-15/
│           ├── session-1.md
│           ├── handoff-session-1.md
│           ├── session-2.md
│           └── handoff-session-2.md
├── daily/
│   ├── thoughts-2025-01-14.md
│   └── thoughts-2025-01-15.md
└── backlog.md
```

全部是纯 Markdown 文件，不依赖任何特定 app。VS Code、Obsidian、Typora，或者任何文本编辑器都可以打开。

---

## 使用方法

正常和 Claude 工作。做完了，说一句：

> **"Boil down"**

Claude 会生成四个文件，并告诉你保存在哪里。

---

> 安装与配置说明见 [SKILL.md](./SKILL.md) 顶部。

## License

MIT
