# SKILL.md 独立路由器重构 客现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 将 SKILL.md 从 ~620 行瘦身为 ~120 行路由器，操作详情按需加载，减少上下文占用。

**Architecture:** SKILL.md 仅保留触发表、路由和前3 步预处理，步骤 4 按参数读取对应的操作文件。新增 10 个操作文件 + 1 个定时任务文件。格式参考并入定时任务文件。

cron-task.md 内容作为 CronCreate 的 prompt 参数直接传入。

**Tech Stack:** 纯 Markdown 文档，无代码依赖。

---

## 文件变更概览

| 文件 | 操作 | 行数变化 |
|------|------|---------|
| `SKILL.md` | 重写 | 620 → ~120 行 |
| `prompts/cron-task.md` | 新建 | ~225 行（从 SKILL.md 提取） |
| `prompts/operations/start.md` | 新建 | ~40 行 |
| `prompts/operations/stop.md` | 新建 | ~20 行 |
| `prompts/operations/status.md` | 新建 | ~30 行 |
| `prompts/operations/exclude.md` | 新建 | ~80 行 |
| `prompts/operations/view-exclude.md` | 新建 | ~15 行 |
| `prompts/operations/classify.md` | 新建 | ~80 行 |
| `prompts/operations/view-practice.md` | 新建 | ~15 行 |
| `prompts/operations/view-target.md` | 新建 | ~15 行 |
| `prompts/operations/update.md` | 新建 | ~15 行 |
| `README.md` | 修改 | 更新文件结构说明 |

---

## Task 1: 创建 prompts/cron-task.md

这是最大的块（~225 行），从 SKILL.md 提取定时任务执行指令 + 格式参考。独立于其他任务。

**Files:**
- Create: `prompts/cron-task.md`

- [ ] **Step 1: 从 SKILL.md 提取内容**

从当前 SKILL.md 中找到以下两个部分，合并为一个文件：

1. `## 定时任务执行指令` 章节（从 `以下是 CronCreate 创建的定时任务...` 到代码块结束标记 ` ``` ）的全部内容
2. `## 岗位记录格式参考` 章节（从 `写入 data/job-candidates.csv 的...` 到文件末尾）

写入 `prompts/cron-task.md`。

- [ ] **Step 2: 验证**

用 Read 工具读取 `prompts/cron-task.md`，确认内容完整、格式正确。

- [ ] **Step 3: 提交**

```bash
git add prompts/cron-task.md
git commit -m "refactor: extract cron task instructions to separate file"
```

---

## Task 2: 创建 9 个操作文件

这些文件从 SKILL.md 提取。由于它们互不依赖，可以用多个子代理并行创建。

**Files:**
- Create: `prompts/operations/start.md`
- Create: `prompts/operations/stop.md`
- Create: `prompts/operations/status.md`
- Create: `prompts/operations/exclude.md`
- Create: `prompts/operations/view-exclude.md`
- Create: `prompts/operations/classify.md`
- Create: `prompts/operations/view-practice.md`
- Create: `prompts/operations/view-target.md`
- Create: `prompts/operations/update.md`

- [ ] **Step 1: 创建目录**

```bash
mkdir -p prompts/operations
```

- [ ] **Step 2: 创建各操作文件**

从当前 SKILL.md 中提取各操作的完整内容（包含操作标题、子步骤、代码块等），逐个写入文件。每个文件是独立的完整指令。

具体提取对应关系：

| 溵盖文件 | SKILL.md 原文 |
|---------|---------------|
| `operations/start.md` | `## 操作 A：start / 无参数 — 启动定时任务` 的全部内容（A1-A6） |
| `operations/stop.md` | `## 操作 B：stop — 停止定时任务` 的全部内容（B1-B4） |
| `operations/status.md` | `## 操作 C：status — 查看状态` 的全部内容（含两种状态模板 + 到期提醒） |
| `operations/exclude.md` | `## 操作 D：排除岗位` 的全部内容（D1-D5） |
| `operations/view-exclude.md` | `## 操作 E：查看排除列表` 的全部内容 |
| `operations/classify.md` | `## 操作 F：岗位分类` 的全部内容（F1-F4） |
| `operations/view-practice.md` | `## 操作 G：查看练手列表` 的全部内容 |
| `operations/view-target.md` | `## 操作 H：查看目标列表` 的全部内容 |
| `operations/update.md` | `## 操作 I：更新 Skill` 的全部内容 |

每个文件的格式： 直接从 SKILL.md 复制对应章节内容，去掉操作标题的 `## 操作 X：xxx` 前缀（因为文件名已经说明了用途）。

- [ ] **Step 3: 验证**

用 Bash 检查所有文件是否存在：
```bash
ls -la prompts/operations/
```

- [ ] **Step 4: 提交**

```bash
git add prompts/operations/
git commit -m "refactor: extract 9 operation files from SKILL.md"
```

---

## Task 3: 重写 SKILL.md 为路由器
依赖 Task 1 和 Task 2 完成后执行。保留前 3 步预处理 + 路由表，替换操作详情为按需加载。

**Files:**
- Modify: `SKILL.md`（全文件重写）

- [ ] **Step 1: 環写 SKILL.md**

保留的内容（原样保留）：
- frontmatter（---name: job-hunter... ... ---）
- 标题 `# Job Hunter Skill — 主入口`
- 触发条件表
- `## 主入口逻辑` 到 `步骤 4` 之前的所有内容（步骤 0、1、1.5、2、3）

替换的内容：
- `## 步骤 4：根据参数执行操作` 替换为路由表

删除的内容：
- 操作 A-I 的全部详细流程（`## 操作 A` 到 `## 操作 I` 结束）
- `## 定时任务执行指令` 全部内容
- `## 岗位记录格式参考` 全部内容

步骤 4 的新路由表：

```markdown
### 步骤 4：根据参数执行操作
根据用户传入的参数（argument）判断操作类型，读取对应的操作文件执行：

| 参数 | 读取文件 |
|------|---------|
| start / 无参数 | `${CLAUDE_SKILL_DIR}/prompts/operations/start.md` |
| stop | `${CLAUDE_SKILL_DIR}/prompts/operations/stop.md` |
| status | `${CLAUDE_SKILL_DIR}/prompts/operations/status.md` |
| exclude | `${CLAUDE_SKILL_DIR}/prompts/operations/view-exclude.md` |
| practice | `${CLAUDE_SKILL_DIR}/prompts/operations/view-practice.md` |
| target | `${CLAUDE_SKILL_DIR}/prompts/operations/view-target.md` |
| update | `${CLAUDE_SKILL_DIR}/prompts/operations/update.md` |
| 用户说「排除编号X」 | `${CLAUDE_SKILL_DIR}/prompts/operations/exclude.md` |
| 用户说「练手/目标/标记 编号X」 | `${CLAUDE_SKILL_DIR}/prompts/operations/classify.md` |
```

- [ ] **Step 2: 验证**

用 Read 工具读取重写后的 SKILL.md，确认：
- 路由表完整
- 无残留的操作详细内容
- 总行数约120 行

- [ ] **Step 3: 提交**

```bash
git add SKILL.md
git commit -m "refactor: slim SKILL.md to router with on-demand operation loading"
```

---

## Task 4: 更新 README.md 文件结构
与 Task 3 可并行执行（不同文件）。

**Files:**
- Modify: `README.md`

- [ ] **Step 1: 更新文件结构说明**

在 README.md 的 skill 安装目录文件结构部分（`job-hunter-skill/` 代码块），更新 `prompts/` 目录展示：

```markdown
job-hunter-skill/
├── SKILL.md                 # 主入口，路由分发
├── prompts/                 # 各环节行为模板
│   ├── intake.md            #   新用户引导问答
│   ├── search-strategy.md   #   搜索策略
│   ├── filter.md            #   筛选规则
│   ├── scorer.md            #   评分规则
│   ├── cron-task.md         #   定时任务执行指令
│   ├── notifications.md     #   模板化提示
│   └── operations/          #   按需加载的操作指令
│       ├── start.md         #   启动定时任务
│       ├── stop.md          #   停止定时任务
│       ├── status.md        #   查看状态
│       ├── exclude.md       #   排除岗位
│       ├── view-exclude.md   #   查看排除列表
│       ├── classify.md      #   寡练手/目标分类
│       ├── view-practice.md  #   查看练手列表
│       ├── view-target.md   #   查看目标列表
│       └── update.md        #   更新 Skill
```

- [ ] **Step 2: 提交**

```bash
git add README.md
git commit -m "docs: update file structure in README for slim router"
```

---

## Task 5: 更新步骤 1.5 升级检查
依赖 Task 3 执行。SKILL.md 猩身后，升级检查需要包含新增的文件。

**Files:**
- Modify: `SKILL.md`（步骤 1.5 区域）

- [ ] **Step 1: 更新必要文件清单**

在步骤 1.5 的必要文件清单表格中追加：

```markdown
| `data/job-practice.csv` | 练手岗位归档 |
| `data/job-target.csv` | 目标岗位归档 |
```

在默认初始内容说明中追加 `prompts/cron-task.md` 和 `prompts/operations/` 目录说明。

不修改默认初始内容，因为它们是新目录不是文件。

但可以在注释中说明："新版本的 prompt 文件（如 cron-task.md、operations/） 不自动创建，会自动在首次使用时由操作流程按需加载。"

。

- [ ] **Step 2: 提交**

```bash
git add SKILL.md
git commit -m "refactor: update upgrade check for new file structure"
```

---

## Task 6: 最终验证与推送

依赖所有任务完成。

- [ ] **Step 1: 一致性检查**

Grep 搜索 `${CLAUDE_SKILL_DIR}/prompts/operations/`，确认所有 9 个文件路径在 SKILL.md 的路由表中正确出现。Grep 搜索 `${CLAUDE_SKILL_DIR}/prompts/cron-task.md`，确认 start.md 操作中引用了 cron-task.md。

Grep 搜索 `## 操作`， SKILL.md， 确认无残留的操作详细内容。

Grep 搜索 `## 定时任务执行指令`， SKILL.md, 确认已移除。Grep 搜索 `## 岗位记录格式参考` 在 SKILL.md, 确认已移除。

- [ ] **Step 2: 推送**

```bash
git push
```

## 并行执行策略

```
Task 1 (cron-task.md) ─┐ Task 2 (operations/) ─┐ Task 3 (重写 SKILL.md) ─→ Task 5 ─→ Task 6
                     ↖ Task 4 (README.md, 与 Task 3 并行) ─↘
```

- Task 1 和 Task 2 可以并行（创建新文件）
- Task 3 必须等待 Task 1 和 Task 2 完成
- Task 4 可以和 Task 3 并行
- Task 5 必须等待 Task 3 完成
- Task 6 在最后执行
