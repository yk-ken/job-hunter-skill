# 岗位三分类机制 实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 为 Job Hunter 添加候选岗位三分类机制，用户可将候选岗位标记为「练手」或「目标」，分别归档到独立 CSV 文件。

**Architecture:** 复用现有排除机制的模式（从候选列表移除 → 追加到归档文件 → 重编号 → 更新 meta.json），新增统一分类操作和两个查看列表操作。练手/目标分类无反馈闭环，不影响搜索条件。

**Tech Stack:** 纯 prompt 驱动（Markdown 文档修改），无代码依赖。

---

## 文件变更概览

| 文件 | 操作 | 职责 |
|------|------|------|
| `SKILL.md` | 修改 | 触发表新增、升级检查扩展、状态增强、新增操作 E/F/G |
| `prompts/intake.md` | 修改 | 新增文件初始化步骤、meta.json 新字段 |

以下两个文件**不修改**：`prompts/filter.md`（security_id 已在 recorded_job_ids 中）、`prompts/scorer.md`。

---

## Task 1: SKILL.md — 触发表与升级检查

**Files:**
- Modify: `SKILL.md:12-21`（触发条件表）
- Modify: `SKILL.md:57-88`（步骤 1.5 升级检查）

- [ ] **Step 1: 触发条件表新增三行**

在 `SKILL.md` 第 21 行（`用户说「排除编号X」...` 之后）追加：

```markdown
| `/job-hunter practice` | 查看练手列表 |
| `/job-hunter target` | 查看目标列表 |
| 用户说「练手编号X」 | 触发分类流程，分类为练手 |
| 用户说「目标编号X」 | 触发分类流程，分类为目标 |
| 用户说「标记编号X」 | 触发分类流程，询问分类类型 |
```

- [ ] **Step 2: 升级检查必要文件清单新增两项**

在 `SKILL.md` 步骤 1.5 的必要文件清单表格（第 62-68 行区域），在 `data/meta.json` 行之后追加：

```markdown
| `data/job-practice.csv` | 练手岗位归档 |
| `data/job-target.csv` | 目标岗位归档 |
```

在默认初始内容说明（第 84-88 行区域），追加：

```markdown
- `data/job-practice.csv`：写入表头行（`编号,岗位名称,公司名,...,备注,分类日期`）
- `data/job-target.csv`：写入表头行（`编号,岗位名称,公司名,...,备注,分类日期`）
```

- [ ] **Step 3: 验证**

用 Read 工具读取 `SKILL.md` 第 12-25 行和第 57-95 行，确认修改正确、格式对齐。

- [ ] **Step 4: 提交**

```bash
git add SKILL.md
git commit -m "feat: add trigger table entries and upgrade check for job classification"
```

---

## Task 2: SKILL.md — 状态展示增强

**Files:**
- Modify: `SKILL.md:106-114`（步骤 3 meta.json 字段列表）
- Modify: `SKILL.md:226-261`（操作 C status 输出模板）

- [ ] **Step 1: 步骤 3 新增 meta.json 字段说明**

在 `SKILL.md` 步骤 3 的字段列表（第 114 行之后），追加：

```markdown
- `practice_count`：已标记为练手的岗位数
- `target_count`：已标记为目标的岗位数
- `next_practice_number`：下一个练手编号
- `next_target_number`：下一个目标编号
```

- [ ] **Step 2: 运行中状态模板增强**

将操作 C 运行中的输出模板（约第 232-245 行区域）替换为：

```markdown
Job Hunter 状态：运行中

  定时任务 ID：{cron_job_id}
  搜索间隔：每 {search_interval_minutes} 分钟
  搜索关键词：{search_keywords 的数量} 组
  已运行次数：{run_count}
  候选列表：{total_candidates} 个待审核
  练手岗位：{practice_count} 个
  目标岗位：{target_count} 个
  已排除：{excluded_count} 个
  上次运行时间：{last_run_time}

管理命令：
  /job-hunter stop     — 停止搜索
  /job-hunter start    — 重启搜索（需先 stop）
  /job-hunter practice — 查看练手列表
  /job-hunter target   — 查看目标列表
  /job-hunter exclude  — 查看排除列表
```

- [ ] **Step 3: 已停止状态模板增强**

将操作 C 已停止的输出模板替换为：

```markdown
Job Hunter 状态：已停止

  候选列表：{total_candidates} 个待审核
  练手岗位：{practice_count} 个
  目标岗位：{target_count} 个
  已排除：{excluded_count} 个
  上次运行时间：{last_run_time 或 "从未运行"}

运行 /job-hunter start 开始搜索。
```

注意：`excluded_count` 需要从 `data/job-excluded.csv` 读取行数（不含表头），不是 meta.json 的字段。

- [ ] **Step 4: 验证**

用 Read 工具读取修改区域，确认字段名、占位符、缩进正确。

- [ ] **Step 5: 提交**

```bash
git add SKILL.md
git commit -m "feat: enhance status display with classification counts"
```

---

## Task 3: SKILL.md — 新增操作 E/F/G

**Files:**
- Modify: `SKILL.md:352` 区域（操作 E 查看排除列表之后，定时任务执行指令之前）

这是最大的改动块。在现有操作 E（查看排除列表）之后、「定时任务执行指令」之前插入三个新操作。

- [ ] **Step 1: 新增操作 E — 岗位分类**

在 `SKILL.md` 操作 E（查看排除列表）结束后（`---` 分隔线处），追加操作 E 改为操作 F（原查看排除列表），然后新增分类操作。具体操作：

1. 将现有的「操作 E：查看排除列表」保持不变（它已经是查看排除列表）
2. 在其后追加新的操作 F（岗位分类）、操作 G（查看练手列表）、操作 H（查看目标列表）

等等——根据设计文档，编号需要重新规划。当前已有操作 A-E，新增操作应接续：

- 操作 A: start
- 操作 B: stop
- 操作 C: status
- 操作 D: 排除岗位
- 操作 E: 查看排除列表
- **操作 F: 岗位分类（新增）**
- **操作 G: 查看练手列表（新增）**
- **操作 H: 查看目标列表（新增）**

在操作 E 结束后的 `---` 分隔线之后、「定时任务执行指令」的 `---` 之前，插入以下内容：

```markdown
## 操作 F：岗位分类

当用户说「练手编号X」「目标编号X」或「标记编号X」时，执行以下流程。

### F1. 确认岗位信息

从 `data/job-candidates.csv` 中读取用户指定编号的岗位，展示完整信息给用户确认。

> **注意**：CSV 中「链接」列存储的是 HYPERLINK 公式格式（如 `"=HYPERLINK(""https://..."",""点击查看"")"`），展示给用户时应提取其中的实际 URL，不要展示公式原文。

```
确认分类以下岗位：
  编号：{number}
  岗位：{job_name}
  公司：{company}
  薪资：{salary}
  地点：{location}
```

如果指定编号不存在，提示用户并终止。

如果用户说的是「标记编号X」（未指定分类），使用 AskUserQuestion 询问：

```
分类为：
1. 练手 — 值得面试但非首选
2. 目标 — 各方面都符合预期
```

### F2. 询问备注

使用 AskUserQuestion 询问用户：

```
备注一下分类原因？（可直接回车跳过）
```

如果用户跳过（空回复或直接回车），备注字段填「无」。

### F3. 执行分类

执行以下变更：

1. **创建归档文件**（如果目标文件不存在）：
   - 练手 → `data/job-practice.csv`，写入表头行：
     ```
     编号,岗位名称,公司名,薪资,地点,发布日期,经验要求,技能要求,匹配度分数,匹配度星级,BOSS姓名,BOSS职位,HR活跃度,公司主营业务,公司发展状况,社保人数,链接,security_id,备注,分类日期
     ```
   - 目标 → `data/job-target.csv`，写入表头行（格式同上）

2. **追加到归档文件**：
   读取目标 CSV 现有行数（不含表头），新行的编号 = 现有行数 + 1。将该岗位的完整信息 + 备注 + 当天日期（YYYY-MM-DD）追加一行。

3. **从候选列表移除**：
   从 `data/job-candidates.csv` 中删除该编号对应的行。

4. **重排候选列表编号**：
   将 `data/job-candidates.csv` 剩余行按原顺序重新编号为 1, 2, 3, ...。

5. **更新 meta.json**：
   - `total_candidates` 减 1
   - `next_candidate_number` = 候选列表新总行数 + 1
   - 练手：`practice_count` 加 1，`next_practice_number` 加 1
   - 目标：`target_count` 加 1，`next_target_number` 加 1
   - `updated_at` = 当前 ISO 时间

### F4. 输出分类确认

```
已将岗位标记为{练手/目标}：
  {job_name} @ {company} | {salary} | {location}
  备注：{note}

候选列表剩余 {total_candidates} 个岗位。
{练手/目标}列表共 {分类计数} 个岗位。
```

---

## 操作 G：查看练手列表

当用户运行 `/job-hunter practice` 时，读取 `data/job-practice.csv`。

如果文件不存在或仅有表头行：

```
练手列表为空，暂无标记为练手的岗位。
```

如果有记录，以表格形式展示：

```
练手岗位（共 {count} 个）：

| # | 岗位名称 | 公司 | 薪资 | 备注 | 分类日期 |
|---|---------|------|------|------|---------|
| 1 | xxx     | xxx  | xxx  | xxx  | xxx     |
```

---

## 操作 H：查看目标列表

当用户运行 `/job-hunter target` 时，读取 `data/job-target.csv`。

如果文件不存在或仅有表头行：

```
目标列表为空，暂无标记为目标的岗位。
```

如果有记录，以表格形式展示：

```
目标岗位（共 {count} 个）：

| # | 岗位名称 | 公司 | 薪资 | 备注 | 分类日期 |
|---|---------|------|------|------|---------|
| 1 | xxx     | xxx  | xxx  | xxx  | xxx     |
```
```

- [ ] **Step 2: 验证**

用 Read 工具读取插入区域（操作 E 结束到定时任务执行指令之间），确认：
- 操作 F/G/H 结构完整
- 编号与已有操作 A-E 不冲突
- 三个新操作之间有 `---` 分隔线

- [ ] **Step 3: 提交**

```bash
git add SKILL.md
git commit -m "feat: add classify, practice list, and target list operations"
```

---

## Task 4: prompts/intake.md — 新用户引导初始化

**Files:**
- Modify: `prompts/intake.md:280-340` 区域（第四步初始化运行数据）

此任务与 Task 1-3 修改不同文件，**可并行执行**。

- [ ] **Step 1: 新增练手文件和目标文件创建步骤**

在 `prompts/intake.md` 的「4.1.5 创建排除列表文件」之后，追加：

```markdown
### 4.1.6 创建练手文件

写入文件路径：`data/job-practice.csv`

文件内容（CSV 表头行）：

```csv
编号,岗位名称,公司名,薪资,地点,发布日期,经验要求,技能要求,匹配度分数,匹配度星级,BOSS姓名,BOSS职位,HR活跃度,公司主营业务,公司发展状况,社保人数,链接,security_id,备注,分类日期
```

### 4.1.7 创建目标文件

写入文件路径：`data/job-target.csv`

文件内容（CSV 表头行）：

```csv
编号,岗位名称,公司名,薪资,地点,发布日期,经验要求,技能要求,匹配度分数,匹配度星级,BOSS姓名,BOSS职位,HR活跃度,公司主营业务,公司发展状况,社保人数,链接,security_id,备注,分类日期
```
```

- [ ] **Step 2: meta.json 初始内容新增字段**

在 `prompts/intake.md` 的 meta.json 初始内容模板中，在 `"exclude_keywords": [],` 之后追加：

```json
  "practice_count": 0,
  "target_count": 0,
  "next_practice_number": 1,
  "next_target_number": 1,
```

- [ ] **Step 3: 更新确认信息**

将确认信息（约第 335 行区域）更新为：

```markdown
运行数据已初始化：
  - data/job-candidates.csv — 候选岗位记录文件（CSV 格式，可用 Excel 打开）
  - data/job-excluded.csv — 排除岗位归档文件
  - data/job-practice.csv — 练手岗位归档文件
  - data/job-target.csv — 目标岗位归档文件
  - data/meta.json — 运行元数据
```

- [ ] **Step 4: 验证**

用 Read 工具读取 `prompts/intake.md` 修改区域，确认格式正确、meta.json 字段完整。

- [ ] **Step 5: 提交**

```bash
git add prompts/intake.md
git commit -m "feat: add practice and target file initialization in intake flow"
```

---

## 并行执行策略

```
Task 1 (SKILL.md 触发+升级) ──┐
                               ├─→ Task 3 (SKILL.md 操作F/G/H) ─→ 最终验证 ─→ push
Task 4 (intake.md)         ────┘
Task 2 (SKILL.md 状态增强) ────┘
```

- **Task 1 和 Task 4** 修改不同文件，可完全并行
- **Task 2 和 Task 3** 与 Task 1 同文件但不同区域，可串行快速执行
- 建议：Task 1 → Task 2 → Task 3 串行（同文件避免冲突），Task 4 并行

## 最终验证

所有任务完成后：

- [ ] **全文一致性检查**：Grep 搜索 `job-practice.csv` 和 `job-target.csv`，确认在 SKILL.md、intake.md 中引用一致
- [ ] **meta.json 字段检查**：确认 `practice_count`、`target_count`、`next_practice_number`、`next_target_number` 在 SKILL.md 步骤 3、步骤 1.5 升级检查、intake.md 初始化中都有出现
- [ ] **操作编号检查**：确认操作 A-H 无重复无遗漏

- [ ] **推送**

```bash
git push
```
