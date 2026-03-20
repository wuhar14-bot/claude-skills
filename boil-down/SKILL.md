---
name: boil-down
description: Boil down current session to notes and update daily canvas/thought file. Use when user says "boil down", "boil down session", "浓缩session", "总结这个对话", or wants to summarize the conversation into session notes.
---

# Boil Down Session Skill

## IMPORTANT REMINDERS

1. **Use PowerShell on Windows** - not bash syntax like `if not exist`
2. **Session summary + file links together** - keep them in one block, not separated
3. **同一文件多次修改合并** - 若同一文件经历多次修改，合并为一条记录，用顿号分隔变化点，时间用范围表示

将当前对话浓缩为session笔记，并自动更新daily canvas和thought文件。

## 触发条件

当用户说以下内容时自动触发：
- "boil down"
- "boil down session"
- "浓缩session"
- "总结这个对话"
- "创建session笔记"

## 执行步骤

### 1. 自动检测Session编号

**从thought文件检测当前已有session数量：**

```powershell
$today = Get-Date -Format 'yyyy-MM-dd'
$month = Get-Date -Format 'yyyy-MM'
$thought_file = "E:\claude-code\2025_new_me_copy\thought-recording\thoughts_$today.md"
```

读取thought文件，查找所有 `**Session N**:` 格式的行，获取最大N值，新session编号 = N + 1

### 2. 创建Session笔记文件

**文件路径：**
```
E:\claude-code\2025_new_me_copy\focus-sessions\[YYYY-MM]\[YYYY-MM-DD]\session [N].md
```

**笔记格式（遵循boil_down_guide）：**

```markdown
project: [DETECTED — see Project Detection Rules below]

# Session [N] - YYYY-MM-DD

## 会话总结

本次会话主要完成了以下任务：
1. 任务描述1（关键数据）
2. 任务描述2（结果）
3. 任务描述3（统计）

---

### 1. 任务名称

**问题**：描述问题或任务需求

**解决方案**：
- 步骤1：具体操作
- 步骤2：具体操作

**结论**：关键发现或要点

---

## 生成/修改的文件

- `E:\claude-code\path\to\file1.md` - 文件描述 | HH:MM | prompt: 用户指令概述
- `E:\claude-code\path\to\file2.md` - 文件描述 | HH:MM | prompt: 同上
```

**CRITICAL**: 文件链接列表必须放在session笔记文件中（而非thought文件），作为最后一个section。

### 3. 更新Thought文件

**添加Session简介（保持简洁）：**

在Sessions部分添加：

**⚠️ ORDER CHECK（添加后必须检查）**: 添加完session条目后，必须核对Thought文件中所有Session的排列顺序是否为升序（Session 1, 2, 3, …）。若新添加的Session编号低于文件末尾已有的Session，则必须将其插入到正确位置，**不能直接追加到末尾**。
```markdown
**Session N**: 简要描述本次会话主要内容（约20字）
📋 Session Note: `E:\claude-code\2025_new_me_copy\focus-sessions\YYYY-MM\YYYY-MM-DD\session N.md`
🎯 Main Output: `E:\path\to\main\output\file.ext`（若本session无明确文件产出则省略此行）
🤝 Hand-off File: `E:\claude-code\2025_new_me_copy\focus-sessions\YYYY-MM\YYYY-MM-DD\handoff_session_N.md`
- bullet point 1：关键操作或成果（含具体数据/文件名）
- bullet point 2：关键操作或成果
- bullet point 3：关键操作或成果
- bullet point 4：关键操作或成果（4-6条，视任务复杂度）
```

**CRITICAL**: Thought文件中每个Session包含5样东西：
1. **标题行**：`**Session N**: 约20字简介`
2. **Session笔记链接**：用 📋 标记
3. **主要输出文件**：用 🎯 标记，本次session最终交付的核心文件路径（仅一行）
4. **Hand-off文件链接**：用 🤝 标记
5. **Bullet point列表**：4-6条，每条一句话，包含具体文件名/数据，不写段落句子

**CRITICAL**: 文件列表（生成/修改的文件）放在session笔记文件中，不要放在thought文件里。Thought文件只保留简要总结，详细内容和文件列表都在session note中。

### 4. 生成 Hand-off 文件

**文件路径：**
```
E:\claude-code\2025_new_me_copy\focus-sessions\[YYYY-MM]\[YYYY-MM-DD]\handoff_session_[N].md
```

**Hand-off文件格式：**

```markdown
# Hand-off: Session N — YYYY-MM-DD

## 任务描述

本次会话完成了（1-2句话概括核心任务和背景）。

## 主要输出文件

- `E:\path\to\main\output.ext` — 核心交付物说明

## 涉及文件路径

- `E:\path\to\file1.md` — 文件用途
- `E:\path\to\file2.py` — 文件用途
（列出所有关键文件，5条以内）

## 上下文与注意事项

- 关键决策或约束（如有）
- 已知问题或边界条件

## 未完成 / 下一步

- [ ] 待续任务1（若有）
- [ ] 待续任务2（若有）
（无待续任务则写"本session任务已全部完成"）
```

**要点：**
- Hand-off文件是给**下一个agent**看的，面向快速上手，不需要过程细节
- 总行数控制在30行以内
- 文件路径全部用反引号包裹

### 5. 同步 BACKLOG.md

**文件路径：** `E:\claude-code\2025_new_me_copy\BACKLOG.md`

**执行逻辑：**

1. 读取刚生成的 handoff 文件的 `## 未完成 / 下一步` 部分，提取所有 `- [ ]` 条目
2. 若该部分为"本session任务已全部完成"，则跳过，无需修改 BACKLOG
3. 根据 session 的 `project:` 标签，定位 BACKLOG 中对应的 Section：

| Project 标签 | BACKLOG Section |
|:---|:---|
| `Thesis` / `CoNova` | `## AIS / 浙一研究` |
| `HMRF` | `## HMRF 2025（iNurseOT）` |
| `LBP_System` | （如无该 section，新建） |
| `ABC` | `## ABC（anythingbutclimbing.com）` |
| `Skills` / `System` | `## vibe-pet` 或新建 `## 工具/系统` section |
| `Other` / 其他 | 在文件末尾新建对应名称的 section |

4. 将 handoff 的 `- [ ]` 条目逐条检查：
   - 若 BACKLOG 对应 section 中已有**语义相同**的条目（即使文字略有出入），跳过（不重复添加）
   - 若是新条目，追加到该 section 末尾
5. 若本 session 明确完成了 BACKLOG 中某条目，将其**移入 `## Archive`**（而非打 `[x]`）：
   - 在 Archive 中找到或新建对应项目子标题（如 `### Hotel Inspection`）
   - 将条目改写为简洁一行 + ` ✓`，去掉子行路径说明
   - 从原 section 中删除该条目；若 section 因此清空，删除整个 section
6. 更新文件顶部的 `> 上次更新：YYYY-MM-DD`
7. **若新建了 section**，同步更新顶部目录：
   - 在 `**目录**` 下追加一行 `- [新Section名称](#anchor)`
   - anchor 规则：全小写，空格→`-`，括号/句号/斜杠等特殊字符删除
   - 示例：`## PhD 延期申请` → `- [PhD 延期申请](#phd-延期申请)`
   - 目录顺序与文件中的 section 顺序保持一致，`## Archive` 始终放最后

**格式规范：**
- 新增条目格式保持 `- [ ] 任务描述` 一行，可附关键路径作为缩进子行
- 不删除 BACKLOG 中已有的 `[~]` 等待中条目
- Archive 条目格式：`- 简洁描述 ✓`（无 checkbox，无子行）

---

## Session笔记编写规范

### 会话总结部分

- 使用编号列表（1-8项，根据任务数量调整）
- 每项一句话概括核心任务
- 括号内包含关键数据
- 动作导向：用"修复、创建、解决、更新、生成"等动词开头

### 详细描述部分

每个任务包含以下要素（至少3个）：

1. **问题/任务**：用户请求或发现的问题
2. **根本原因**（如适用）：问题的深层原因
3. **解决方案/操作**：具体步骤列表
4. **数据/计算**（如适用）：相关数值
5. **结论/经验教训**（如适用）：关键发现

### 格式规范

- **加粗**：用于标签和关键术语
- `代码块`：用于文件名、命令、代码片段
- 使用 `→` 表示变化：`70卡→110卡`
- **总行数不超过100行**
- 全部使用中文（除专业术语和文件名）

---

## 文件链接格式

```markdown
- `E:\claude-code\path\to\file.md` - 文件描述（关键数据） | HH:MM | prompt: 用户指令概述
```

**格式要素：**
| 要素 | 说明 | 示例 |
|:---|:---|:---|
| 文件路径 | 完整本地路径，用反引号包裹 | `E:\claude-code\xxx\file.md` |
| 简要说明 | 一句话描述文件内容/用途 | 患者列表生成规则文档 |
| 生成时间 | 精确到分钟（HH:MM） | 12:25 |
| prompt | 触发生成的用户指令概述 | 创建guide文档 |

**相同prompt：** 多个文件由同一prompt生成时，可用"同上"简化

**同一文件多次修改：** 合并为一条，用顿号分隔变化点，时间用范围
```markdown
# WRONG - 重复列出同一文件
- `E:\xxx\file.md` - 创建v2版本 | 21:00
- `E:\xxx\file.md` - 新增2.1节 | 21:05
- `E:\xxx\file.md` - 修改格式 | 21:10

# CORRECT - 合并为一条
- `E:\xxx\file.md` - 创建v2：新增2.1节、修改格式、细分利益方 | 21:00-21:15
```

---

## Project Detection Rules

当创建 session 文件时，根据会话内容（关键词匹配）自动填写 `project:` 标签。

**优先级从上到下，第一个匹配的 project 胜出：**

| Project | 关键词 / 模式（任意一个匹配即可） |
|:---|:---|
| `Thesis` | thesis, LaTeX, overleaf, .tex, dissertation, Schroth, PSSE, AIS, chapter1-7 |
| `HMRF` | HMRF, hmrf, iNurseOT, EvalMethods, Objectives_v2 |
| `Vaccine` | 疫苗, 联苗, 联合疫苗, 褚教授, 政策杠杆, vaccine, 联苗报告 |
| `CoNova` | CoNova, ConovaMed, X光, 脊柱侧弯筛查, 患者招募, RGBD, 浙一, 浙二, 邵逸夫 |
| `LBP_System` | low back pain system, LBP, mediapipe service, LowBackPain, 核心稳定性训练 |
| `Climbing` | 攀岩, climbing, spraywall, bouldering, lead climbing |
| `CTDP` | CTDP, React Native, Expo, Vercel CTDP |
| `Skills` | nano banana, PackyAPI, SKILL.md, boil-down skill, create skill |
| `System` | graph viewer, thought recording, thought file, focus-session格式, daily tracker, 每日追踪, 营养追踪, daily canvas, 创建每日, server.py, 笔记系统, hand-off, 工作流, workflow improvement |
| `ABC` | abc-website, abc\\, Chalkemon, ABC.*电商 |
| `PhD` | PhD extension, study plan, Donna, 延长申请 |
| `Other` | 会话内容零散、无明确主题（如闲聊、一次性临时任务） |

**如果没有关键词匹配：**
1. 使用 `project: Other`
2. **不要自己发明新项目名**——这会导致 graph viewer 出现孤立的单连接节点
3. 如果会话确实代表一个重要新方向（5+ sessions 可能性高），在 session 笔记的注释里说明，等用户确认后再更新 canonical list

**CRITICAL**: `project:` 必须是文件第一行，单独一行，格式严格为 `project: HMRF`（冒号后一个空格，无引号）。

---

## 质量检查清单

完成后检查：
- [ ] Session编号正确（从thought文件自动检测）
- [ ] **Session顺序正确**：Thought文件中所有Session条目按编号升序排列（若新Session编号较低，需插入到正确位置而非追加末尾）
- [ ] **`project:` 标签已写入 session 文件第一行**（格式：`project: HMRF`）
- [ ] 会话总结项数合理
- [ ] 详细描述包含5要素（至少3个）
- [ ] Thought文件Session简介约20字
- [ ] 文件链接包含路径+说明+时间+prompt
- [ ] 总行数不超过100行
- [ ] Thought文件包含 🎯 主要输出行（有文件产出时）
- [ ] Hand-off文件已生成（路径正确，30行以内）
- [ ] Thought文件包含 🤝 Hand-off链接
- [ ] BACKLOG.md 已同步（新增未完成条目 / 标记完成条目 / 更新日期）
- [ ] **若新建了 BACKLOG section，顶部目录已同步追加对应链接**