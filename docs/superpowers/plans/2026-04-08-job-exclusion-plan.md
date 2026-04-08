# 候选岗位排除机制 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 为 job-hunter skill 增加候选岗位排除机制，支持人工审核后排除不合适岗位，并将排除原因反馈到搜索条件优化中。

**Architecture:** 三个独立的文件修改任务，互不依赖，可完全并行执行。每个任务修改一个文件的不同部分。

**Tech Stack:** Markdown skill 文件（SKILL.md、prompts/*.md），CSV 数据文件。

---

## File Structure

| 文件 | 操作 | 职责 |
|------|------|------|
| `prompts/intake.md` | 修改 | 新用户引导流程中初始化 `job-excluded.csv` |
| `prompts/filter.md` | 修改 | 筛选流程增加排除列表检查（Step 1.5） |
| `SKILL.md` | 修改 | 触发条件表 + 操作 D 章节 + 定时任务步骤 4 更新 |

三个任务完全独立，可并行执行。

---

### Task 1: 更新 intake.md — 初始化排除列表文件

**Files:**
- Modify: `prompts/intake.md`

- [ ] **Step 1: 在第四步 4.1 之后插入 4.1.5 小节**

在 `prompts/intake.md` 的 `### 4.2 创建运行元数据文件` 之前，插入以下内容：

```markdown
### 4.1.5 创建排除列表文件

写入文件路径：`data/job-excluded.csv`

文件内容（CSV 表头行）：

```csv
编号,岗位名称,公司名,薪资,地点,发布日期,经验要求,技能要求,匹配度分数,匹配度星级,BOSS姓名,BOSS职位,HR活跃度,公司主营业务,公司发展状况,社保人数,链接,security_id,排除原因,排除日期
```
```

- [ ] **Step 2: 更新确认信息**

在 `prompts/intake.md` 中找到确认信息部分：

```
运行数据已初始化：
  - data/job-candidates.csv — 候选岗位记录文件（CSV 格式，可用 Excel 打开）
  - data/meta.json — 运行元数据
```

替换为：

```
运行数据已初始化：
  - data/job-candidates.csv — 候选岗位记录文件（CSV 格式，可用 Excel 打开）
  - data/job-excluded.csv — 排除岗位归档文件
  - data/meta.json — 运行元数据
```

- [ ] **Step 3: 提交**

```bash
git add prompts/intake.md
git commit -m "feat: initialize job-excluded.csv in intake flow"
```

---

### Task 2: 更新 filter.md — 增加排除列表检查

**Files:**
- Modify: `prompts/filter.md`

- [ ] **Step 1: 在输入来源表新增一行**

找到：

```markdown
| `data/meta.json` | 运行元数据（`recorded_job_ids`、`exclude_keywords`） |
```

在其后面插入：

```markdown
| `data/job-excluded.csv` | 已排除岗位的 `security_id` 列表 |
```

- [ ] **Step 2: 在 Step 1 之后插入 Step 1.5**

找到 `### Step 1: ID 去重` 的完整内容，在其结束（`### Step 2: 排除关键词` 之前）后插入：

```markdown
### Step 1.5: 排除列表检查

读取 `data/job-excluded.csv` 中所有 `security_id`（最后一列之前的列）。

- 岗位的 `security_id` 在排除列表中 → 排除，原因标记 `EXCLUDED`
- 不在排除列表中 → 通过，继续下一步

如果 `job-excluded.csv` 不存在或为空（仅表头行），跳过此步骤。
```

- [ ] **Step 3: 在排除原因枚举值列表中新增 EXCLUDED**

找到：

```markdown
- `DUPLICATE` — 已记录过的岗位
```

在其后插入：

```markdown
- `EXCLUDED` — 用户人工排除的岗位
```

- [ ] **Step 4: 在执行注意事项新增第 7 条**

找到执行注意事项的第 6 条：

```markdown
6. 每轮筛选结束后，将通过的岗位的 `security_id` 追加到 `meta.json.recorded_job_ids`，防止下一轮重复推荐
```

在其后插入：

```markdown
7. `job-excluded.csv` 中的岗位在 Step 1.5 被排除，不会进入候选列表。`recorded_job_ids` 已包含这些 ID，此步骤作为额外安全网
```

- [ ] **Step 5: 提交**

```bash
git add prompts/filter.md
git commit -m "feat: add excluded list check in filter pipeline"
```

---

### Task 3: 更新 SKILL.md — 排除操作入口与定时任务增强

**Files:**
- Modify: `SKILL.md`

- [ ] **Step 1: 触发条件表新增排除命令行**

找到触发条件表：

```markdown
| `/job-hunter status` | 查看当前运行状态 |
```

在其后插入：

```markdown
| `/job-hunter exclude` | 查看排除列表 |
| 用户说「排除编号X」 | 触发排除流程（自然语言触发） |
```

- [ ] **Step 2: 在操作 C 之后插入操作 D**

找到操作 C（status）的完整内容结束位置，即 `---` 分隔线和 `## 定时任务执行指令` 之间，插入操作 D：

```markdown
## 操作 D：排除岗位

当用户说「排除编号X」或表达类似意图时，执行以下流程。

### D1. 确认岗位信息

从 `data/job-candidates.csv` 中读取用户指定编号的岗位，展示完整信息给用户确认：

```
确认排除以下岗位：
  编号：{number}
  岗位：{job_name}
  公司：{company}
  薪资：{salary}
  地点：{location}
```

如果指定编号不存在，提示用户并终止。

### D2. 询问排除原因

使用 AskUserQuestion 询问用户：

```
排除原因是什么？
```

等待用户回复。

### D3. 确认条件更新方案

根据用户提供的排除原因，分析并使用 AskUserQuestion 询问条件更新方式：

```
是否将此排除原因应用到搜索条件？请选择：

1. 加入排除关键词 — 将「{提取的关键词}」加入 meta.json 的 exclude_keywords，以后所有含该词的岗位直接被筛掉
2. 调整画像偏好 — 更新 data/job-profile.md 的相关偏好（如区域、薪资等）
3. 仅排除此岗位 — 不更新任何条件，只排除这一个
```

等待用户选择。根据选择更新对应文件：

**选择 1**：将提取的关键词追加到 `meta.json` 的 `exclude_keywords` 数组。
**选择 2**：根据原因更新 `data/job-profile.md` 的对应字段。
**选择 3**：不更新条件文件。

如果原因映射关系不明确，向用户追问具体要调整什么。

### D4. 执行排除

执行以下变更：

1. **创建排除列表文件**（如果 `data/job-excluded.csv` 不存在）：
   写入表头行：
   ```
   编号,岗位名称,公司名,薪资,地点,发布日期,经验要求,技能要求,匹配度分数,匹配度星级,BOSS姓名,BOSS职位,HR活跃度,公司主营业务,公司发展状况,社保人数,链接,security_id,排除原因,排除日期
   ```

2. **追加到排除列表**：
   读取 `data/job-excluded.csv` 现有行数（不含表头），新行的编号 = 现有行数 + 1。将该岗位的完整信息 + 排除原因 + 当天日期（YYYY-MM-DD）追加一行到 `data/job-excluded.csv`。

3. **从候选列表移除**：
   从 `data/job-candidates.csv` 中删除该编号对应的行。

4. **重排候选列表编号**：
   将 `data/job-candidates.csv` 剩余行按原顺序重新编号为 1, 2, 3, ...。

5. **更新 meta.json**：
   - `total_candidates` 减 1
   - `next_candidate_number` = 候选列表新总行数 + 1
   - `updated_at` = 当前 ISO 时间

### D5. 输出排除确认

```
已排除岗位：
  {job_name} @ {company} | {salary} | {location}
  排除原因：{reason}
  条件更新：{更新内容摘要，如"已将'外包'加入排除关键词"或"仅排除此岗位"}

候选列表剩余 {total_candidates} 个岗位。
排除列表共 {excluded_count} 个岗位。
```

---

## 操作 E：查看排除列表

当用户运行 `/job-hunter exclude` 时，读取 `data/job-excluded.csv`。

如果文件不存在或仅有表头行：

```
排除列表为空，暂无被排除的岗位。
```

如果有排除记录，以表格形式展示：

```
排除列表（共 {count} 个岗位）：

| # | 岗位名称 | 公司 | 薪资 | 排除原因 | 排除日期 |
|---|---------|------|------|---------|---------|
| 1 | xxx     | xxx  | xxx  | xxx     | xxx     |
```
```

- [ ] **Step 3: 更新定时任务步骤 4 的筛选描述**

找到定时任务执行指令中的步骤 4：

```markdown
## 步骤 4：筛选

读取 ${CLAUDE_SKILL_DIR}/prompts/filter.md，按照其中的规则执行筛选：

输入：步骤 3 获取的岗位详情列表 + data/job-profile.md + data/meta.json
```

替换为：

```markdown
## 步骤 4：筛选

读取 ${CLAUDE_SKILL_DIR}/prompts/filter.md，按照其中的规则执行筛选：

输入：步骤 3 获取的岗位详情列表 + data/job-profile.md + data/meta.json + data/job-excluded.csv
```

- [ ] **Step 4: 更新定时任务步骤 4 的筛选步骤列表**

找到步骤 4 中的筛选步骤列表：

```markdown
按以下顺序执行筛选（任一步骤不通过则排除）：
1. ID 去重：security_id 在 recorded_job_ids 中 → 排除
2. 排除关键词：岗位名/描述含 exclude_keywords → 排除
```

替换为：

```markdown
按以下顺序执行筛选（任一步骤不通过则排除）：
1. ID 去重：security_id 在 recorded_job_ids 中 → 排除
1.5. 排除列表：security_id 在 job-excluded.csv 中 → 排除
2. 排除关键词：岗位名/描述含 exclude_keywords → 排除
```

- [ ] **Step 5: 提交**

```bash
git add SKILL.md
git commit -m "feat: add job exclusion workflow and update filter step in SKILL.md"
```
