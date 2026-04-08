---
name: job-hunter
description: "自动从 Boss 直聘发现、筛选、记录合适岗位。使用 /job-hunter 启动定时求职助手。"
---

# Job Hunter Skill — 主入口

本文件是 job-hunter skill 的主入口。Claude 在用户触发 `/job-hunter` 时读取此文件，按照以下逻辑执行完整流程。

---

## 触发条件

| 命令 | 操作 |
|------|------|
| `/job-hunter` | 等同于 `/job-hunter start`，启动定时搜索 |
| `/job-hunter start` | 启动定时搜索任务 |
| `/job-hunter stop` | 停止定时搜索任务，数据保留 |
| `/job-hunter status` | 查看当前运行状态 |
| `/job-hunter exclude` | 查看排除列表 |
| 用户说「排除编号X」 | 触发排除流程（自然语言触发） |
| `/job-hunter practice` | 查看练手列表 |
| `/job-hunter target` | 查看目标列表 |
| 用户说「练手编号X」 | 触发分类流程，分类为练手 |
| 用户说「目标编号X」 | 触发分类流程，分类为目标 |
| 用户说「标记编号X」 | 触发分类流程，询问分类类型 |

---

## 主入口逻辑

收到 `/job-hunter` 命令后，按以下顺序执行：

### 步骤 0：工作目录检查

检查当前工作目录（CWD）下是否存在 `data/` 目录：

- **data/ 已存在** → 继续步骤 1（识别为已有用户的工作目录）
- **data/ 不存在** → 使用 AskUserQuestion 询问用户：

```
当前目录不是 Job Hunter 的工作目录。
数据文件（画像、候选岗位）将保存在当前目录的 data/ 下。

请确认：
1. 在当前目录继续 — 数据将保存在 {CWD}/data/
2. 我想先切换到其他目录
```

如果用户选择 2，提示用户使用 `cd` 切换到目标目录后重新运行 `/job-hunter`，然后结束。
如果用户选择 1，继续步骤 1。

> **重要**：所有 `data/` 路径（如 data/job-profile.md、data/meta.json）均相对于当前工作目录（CWD）解析。模板文件通过 `${CLAUDE_SKILL_DIR}/prompts/` 从 skill 安装目录读取。两者分离。

### 步骤 1：检查是否为新用户

读取 `data/job-profile.md` 是否存在：

- **不存在** → 进入「新用户引导流程」（步骤 2）
- **存在** → 继续步骤 1.5

### 步骤 1.5：老用户升级检查

当 `data/job-profile.md` 存在时（老用户），检查 `data/` 目录下是否缺少必要文件。

当前必要文件清单：

| 文件 | 说明 |
|------|------|
| `data/job-candidates.csv` | 候选岗位记录 |
| `data/job-excluded.csv` | 排除岗位归档 |
| `data/job-practice.csv` | 练手岗位归档 |
| `data/job-target.csv` | 目标岗位归档 |
| `data/job-profile.md` | 用户求职画像 |
| `data/meta.json` | 运行元数据 |

检查逻辑：

1. 逐项检查上述文件是否存在
2. 如果全部存在 → 跳到步骤 3
3. 如果有缺失 → 自动创建缺失文件（使用默认初始内容），并通知用户：

```
检测到以下文件缺失，已自动补充：

  [已创建] {缺失文件名} — {文件说明}

这可能是因为 skill 更新引入了新文件。你的现有数据未受影响。
```

各缺失文件的默认初始内容：

- `data/job-candidates.csv`：写入表头行（`编号,岗位名称,公司名,...,security_id`）
- `data/job-excluded.csv`：写入表头行（`编号,岗位名称,公司名,...,排除原因,排除日期`）
- `data/job-practice.csv`：写入表头行（`编号,岗位名称,公司名,...,备注,分类日期`）
- `data/job-target.csv`：写入表头行（`编号,岗位名称,公司名,...,备注,分类日期`）
- `data/meta.json`：不自动创建（缺少此文件说明引导流程未完成，应视为异常，提示用户重新运行引导）

此外，检查 `data/meta.json` 是否缺少以下字段，如果缺少则追加默认值：
- `practice_count`: 0
- `target_count`: 0
- `next_practice_number`: 1
- `next_target_number`: 1

检查完成后，继续步骤 3。

### 步骤 2：新用户引导流程

读取并执行 `${CLAUDE_SKILL_DIR}/prompts/intake.md` 中的完整引导流程。

该流程包含：
1. 环境检测（Node.js、opencli、Browser Bridge、Boss 直聘登录）
2. 分步问答收集求职画像（4 组问题）
3. 生成 `data/job-profile.md`
4. 初始化 `data/job-candidates.csv`、`data/job-excluded.csv` 和 `data/meta.json`

**环境检测未通过时**：读取 `${CLAUDE_SKILL_DIR}/prompts/notifications.md` 中的「环境检测未通过」模板，填充检测结果后输出。不继续后续流程。

**引导完成后**：自动进入步骤 3。

### 步骤 3：读取运行状态

读取 `data/meta.json`，获取以下字段：
- `cron_job_id`：当前定时任务 ID（null 表示未运行）
- `run_count`：已运行次数
- `total_candidates`：已记录岗位总数
- `last_run_time`：上次运行时间
- `search_interval_minutes`：搜索间隔
- `next_candidate_number`：下一个岗位编号
- `practice_count`：已标记为练手的岗位数
- `target_count`：已标记为目标的岗位数
- `next_practice_number`：下一个练手编号
- `next_target_number`：下一个目标编号

### 步骤 4：根据参数执行操作

根据用户传入的参数（argument）判断操作类型。

---

## 操作 A：start / 无参数 — 启动定时任务

### A1. 检查是否已在运行

读取 `data/meta.json` 的 `cron_job_id`：
- 如果不为 null，说明已有定时任务在运行。向用户提示：

```
Job Hunter 已在运行中（任务 ID：{cron_job_id}）。
如需重启，请先运行 /job-hunter stop，再运行 /job-hunter start。
```

然后结束。

### A2. 读取搜索间隔配置

读取 `data/meta.json` 的 `search_interval_minutes`（默认 30）。

### A3. 搜索频率安全检查

根据间隔值执行不同逻辑：

**间隔 < 10 分钟**：拒绝并强制修正：

```
搜索间隔不能低于 10 分钟，这是为了避免触发 Boss 直聘风控。
已将间隔设置为最低安全值 10 分钟。
如需调整，请编辑 data/meta.json 中的 search_interval_minutes 字段。
```

将 `search_interval_minutes` 更新为 10，然后继续。

**间隔 10-14 分钟**（中风险）：读取 `${CLAUDE_SKILL_DIR}/prompts/notifications.md` 中的「搜索频率警告」模板，填充变量：
- `{interval}` = 当前间隔值
- `{risk_level}` = `中风险（10-14分钟）`

输出警告，使用 AskUserQuestion 询问用户是否确认。如果用户拒绝，将间隔修正为 30 分钟。

**间隔 15-29 分钟**（低风险）：读取 `${CLAUDE_SKILL_DIR}/prompts/notifications.md` 中的「搜索频率警告」模板，填充变量：
- `{interval}` = 当前间隔值
- `{risk_level}` = `低风险（15-29分钟）`

输出警告，使用 AskUserQuestion 询问用户是否确认。如果用户拒绝，将间隔修正为 30 分钟。

**间隔 >= 30 分钟**：安全，无需警告，直接继续。

### A4. 创建定时任务

使用 CronCreate 创建定时任务，参数如下：

- **durable**: `true`（最长运行 7 天）
- **间隔**: `data/meta.json` 中的 `search_interval_minutes`（分钟）

CronCreate 的 prompt 内容为下方「定时任务执行指令」章节的完整内容。

### A5. 更新 meta.json

CronCreate 创建成功后，更新 `data/meta.json`：
- `cron_job_id` = 创建返回的任务 ID
- `updated_at` = 当前 ISO 时间

### A6. 输出启动成功提示

读取 `${CLAUDE_SKILL_DIR}/prompts/notifications.md` 中的「定时任务启动成功」模板，填充变量：
- `{interval}` = `search_interval_minutes`
- `{keywords_count}` = `search_keywords` 数组长度

输出填充后的提示。

---

## 操作 B：stop — 停止定时任务

### B1. 检查是否有运行中的任务

读取 `data/meta.json` 的 `cron_job_id`：
- 如果为 null，向用户提示：

```
Job Hunter 当前未在运行，无需停止。
```

然后结束。

### B2. 停止任务

使用 CronDelete 删除 `cron_job_id` 对应的定时任务。

### B3. 更新 meta.json

- `cron_job_id` = null
- `updated_at` = 当前 ISO 时间

### B4. 输出停止确认

```
Job Hunter 已停止。

你的数据已保留，下次运行 /job-hunter start 即可恢复搜索。
已记录的 {total_candidates} 个候选岗位仍在 data/job-candidates.csv 中。
```

---

## 操作 C：status — 查看状态

读取 `data/meta.json`，输出状态信息：

**如果 cron_job_id 不为 null（运行中）**：

> 注意：`excluded_count` 从 `data/job-excluded.csv` 行数计算，`practice_count` 和 `target_count` 从 `data/meta.json` 读取。

```
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

**如果 cron_job_id 为 null（已停止）**：

```
Job Hunter 状态：已停止

  候选列表：{total_candidates} 个待审核
  练手岗位：{practice_count} 个
  目标岗位：{target_count} 个
  已排除：{excluded_count} 个
  上次运行时间：{last_run_time 或 "从未运行"}

运行 /job-hunter start 开始搜索。
```

**如果 last_run_time 不为 null 且距离 created_at + 7 天不足 24 小时**：

在状态信息后追加到期提醒。读取 `${CLAUDE_SKILL_DIR}/prompts/notifications.md` 中的「任务即将到期」模板，填充：
- `{remaining_hours}` = floor((created_at + 7天 - 当前时间) / 小时)

---

## 操作 D：排除岗位

当用户说「排除编号X」或表达类似意图时，执行以下流程。

### D1. 确认岗位信息

从 `data/job-candidates.csv` 中读取用户指定编号的岗位，展示完整信息给用户确认。

> **注意**：CSV 中「链接」列存储的是 HYPERLINK 公式格式（如 `"=HYPERLINK(""https://..."",""点击查看"")"`），展示给用户时应提取其中的实际 URL，不要展示公式原文。提取方式：从公式中定位第一个 `http` 开头的 URL 部分。

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

---

## 操作 F：岗位分类

当用户说「练手编号X」「目标编号X」或「标记编号X」时，执行以下流程。

### F1. 确认岗位信息

从 `data/job-candidates.csv` 中读取用户指定编号的岗位，展示完整信息给用户确认。

> **注意**：CSV 中「链接」列存储的是 HYPERLINK 公式格式（如 `"=HYPERLINK(""https://..."",""点击查看"")"`），展示给用户时应提取其中的实际 URL，不要展示公式原文。提取方式：从公式中定位第一个 `http` 开头的 URL 部分。

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

---

## 定时任务执行指令

以下是 CronCreate 创建的定时任务每次执行时的完整指令。这段内容作为 CronCreate 的 prompt 参数传入。

---

```
你是 Job Hunter 定时任务执行器。每次被触发时，严格按照以下 9 个步骤执行。所有文件路径基于 skill 安装目录。

## 步骤 1：读取配置

读取以下两个文件：
- data/job-profile.md — 用户求职画像
- data/meta.json — 运行元数据

如果任一文件不存在或无法读取，输出错误信息并终止本轮执行。

## 步骤 2：执行搜索

读取 ${CLAUDE_SKILL_DIR}/prompts/search-strategy.md，按照其中的策略执行岗位搜索：

1. 从 data/meta.json 提取 search_keywords 数组和 search_pages 对象
2. 对每个关键词执行：opencli boss search "{keyword}" -f json --limit 15 --page {该关键词当前页码}
3. 页码处理：有结果则 page+1，返回 0 条则重置为 1 并立即补搜 page 1
4. 执行推荐获取 3 次：opencli boss recommend --limit 10（3 次结果合并去重，每次间隔 3-5 秒随机）
5. 在内存中计算各关键词的页码变化（实际写入在步骤 7）
6. 将所有结果按 security_id 合并去重
7. 与 data/meta.json 的 recorded_job_ids 对比，筛选出新出现的岗位

如果单个关键词搜索失败（退出码 69/77/75），记录错误日志，跳过该关键词，继续其他搜索。
如果所有搜索均失败，输出错误报告，不中断定时任务（下一轮会重试）。

## 步骤 3：获取岗位详情

对步骤 2 中筛选出的每个新岗位，逐一调用：

opencli boss detail --security-id {id} -f json

从返回的 JSON 中提取以下信息：
- 薪资范围（salary）
- 具体工作地址（location / address）
- 发布日期（post_date / publish_time）
- 经验要求（experience）
- 技能要求（skills / tags）
- 公司信息（company_name, company_scale, company_type）
- BOSS 信息（boss_name, boss_title）
- BOSS 活跃时间（active_time）
- 岗位名称（job_name）
- security_id
- job_id（用于生成链接）

如果单个岗位详情获取失败，跳过该岗位，记录错误，继续处理下一个。

## 步骤 4：筛选

读取 ${CLAUDE_SKILL_DIR}/prompts/filter.md，按照其中的规则执行筛选：

输入：步骤 3 获取的岗位详情列表 + data/job-profile.md + data/meta.json + data/job-excluded.csv

按以下顺序执行筛选（任一步骤不通过则排除）：
1. ID 去重：security_id 在 recorded_job_ids 中 → 排除
1.5. 排除列表：security_id 在 job-excluded.csv 中 → 排除
2. 排除关键词：岗位名/描述含 exclude_keywords → 排除
3. 城市区域：不匹配画像中的城市/区域偏好 → 排除
4. 薪资下限：低于画像最低薪资 → 排除
5. 休息制度：不符合画像最低要求 → 排除
6. HR 活跃度：active_time 为"2周内活跃"或更低频 → 排除
7. 公司规模：低于最低要求 → 不排除但标记 SCALE_PENALTY

将筛选通过的岗位和对应的标记传递给步骤 5。

## 步骤 5：评分排序

读取 ${CLAUDE_SKILL_DIR}/prompts/scorer.md，按照其中的规则计算每个通过筛选岗位的综合匹配度分数（满分 100）：

评分维度：
- 发布日期（25分）：越新分数越高，1天内=25，2天=23，3天=20，5天=15，7天=12，10天=7，14天=3，超过14天=0，无日期=13
- 薪资匹配（20分）：与画像薪资期望的重合程度
- 技术栈匹配（20分）：画像技术栈与岗位要求的重叠度
- 地点匹配（15分）：优先区域=15，同城市非优先=10，远程=12，不匹配=0
- HR 活跃度（10分）：刚刚活跃=10，今日活跃=8，3日内活跃=5，本周活跃=2，无信息=5
- 公司规模（10分）：10000人以上=10，1000-9999人=9，500-999人=8，100-499人=6，50-99人=4，20-49人=3，0-19人=2，信息缺失=5

星级映射：
- 90-100：★★★★★
- 75-89：★★★★☆
- 60-74：★★★☆☆
- 40-59：★★☆☆☆
- 40以下：★☆☆☆☆

按总分降序排列，同分按发布日期降序，同分同日期按薪资匹配度降序。

## 步骤 6：记录新岗位

将通过筛选并评分的岗位追加到 data/job-candidates.csv。

### CSV 格式

文件使用 CSV 格式（逗号分隔），第一行为表头，后续每行一个岗位：

```csv
编号,岗位名称,公司名,薪资,地点,发布日期,经验要求,技能要求,匹配度分数,匹配度星级,BOSS姓名,BOSS职位,HR活跃度,公司主营业务,公司发展状况,社保人数,链接,security_id
```

### 写入规则

1. 如果文件不存在或仅有表头行，直接追加数据行
2. 每个岗位写入一行，字段用逗号分隔
3. 字段中如含逗号、引号或换行，用双引号包裹整个字段
4. 编号使用 data/meta.json 中的 next_candidate_number，每记录一个岗位后编号递增 1
5. 按评分降序排列写入
6. **链接列使用 HYPERLINK 公式**，使 Excel/WPS 打开时可直接点击跳转。格式为：
   ```
   "=HYPERLINK(""https://www.zhipin.com/job_detail/xxx.html"",""点击查看"")"
   ```
   整个公式用双引号包裹，公式内部的双引号用 `""` 转义。该字段为 CSV 倒数第二列（security_id 之前）

### 字段缺失默认值

| 字段 | 默认值 |
|------|--------|
| 具体地址缺失 | 仅显示「城市·区域」 |
| 发布日期缺失 | 未知 |
| 经验要求缺失 | 未知 |
| BOSS 信息缺失 | 未知 |
| HR 活跃度缺失 | 未知 |
| 薪资为「面议」时 | 面议 |
| 公司主营业务缺失 | 未查询到 |
| 公司发展状况缺失 | 未查询到 |
| 社保人数缺失 | 未查询到 |

### CSV 行示例

```csv
12,全栈工程师(AI Agent),星辰科技,15-23K,广州·天河区,2026-04-03,3-5年,Vue,Python,MySQL,82,★★★★☆,张明,技术总监,刚刚活跃,人工智能应用软件开发,A轮融资,156,"=HYPERLINK(""https://www.zhipin.com/job_detail/abc123.html"",""点击查看"")",sec_xxx
```

## 步骤 7：更新 meta.json

更新 data/meta.json 的以下字段：

- recorded_job_ids：将本轮通过筛选的所有岗位的 security_id 追加到数组中（不包括被筛选排除的岗位，避免重复推荐）
- search_pages：将步骤 2 中计算好的各关键词页码变化写入（有结果的 +1，空结果重置为 1 并补搜后的页码）
- last_run_time：当前 ISO 时间
- run_count：原值 + 1
- total_candidates：原值 + 本轮新记录的岗位数
- next_candidate_number：原值 + 本轮新记录的岗位数
- updated_at：当前 ISO 时间

使用 Edit 工具更新 meta.json，保留其他字段不变。

## 步骤 8：输出本轮报告

读取 ${CLAUDE_SKILL_DIR}/prompts/notifications.md 中的「每轮搜索报告」模板。

如果本轮有新岗位（new_count > 0）：

使用模板 5a，填充：
- {date} = 当前日期（YYYY-MM-DD）
- {new_count} = 本轮新记录的岗位数
- {job_summaries} = 每个新岗位的一行摘要（按匹配分降序），格式：
  #{number}  {job_name} @ {company} | {salary} | {location} | 发布于 {post_date} | HR{active_time} | {company_scale} | 匹配 {score}分
- {total_count} = meta.json 中的 total_candidates

如果本轮无新岗位（new_count = 0）：

使用模板 5b，填充：
- {date} = 当前日期
- {searched_count} = 本轮搜索返回的岗位总数（去重前）
- {total_count} = meta.json 中的 total_candidates

## 步骤 9：检查到期时间

读取 data/meta.json 的 created_at 字段，计算定时任务是否接近 7 天上限：

remaining_hours = floor((created_at + 7天 - 当前时间) / 小时)

如果 0 < remaining_hours <= 24：

读取 ${CLAUDE_SKILL_DIR}/prompts/notifications.md 中的「任务即将到期」模板，填充：
- {remaining_hours} = 计算得到的剩余小时数

在本轮报告之后输出到期提醒。
```

---

## 岗位记录格式参考

写入 `data/job-candidates.csv` 的每个岗位为一行 CSV，列顺序如下：

```
编号,岗位名称,公司名,薪资,地点,发布日期,经验要求,技能要求,匹配度分数,匹配度星级,BOSS姓名,BOSS职位,HR活跃度,公司主营业务,公司发展状况,社保人数,链接,security_id
```

如果某个字段信息缺失，使用以下默认值：
- 地点缺失：显示「未知」
- 发布日期缺失：显示「未知」
- 经验要求缺失：显示「未知」
- BOSS 姓名缺失：显示「未知」
- BOSS 职位缺失：显示「未知」
- HR 活跃度缺失：显示「未知」
- 薪资为「面议」时：显示「面议」
- 公司主营业务缺失：显示「未查询到」
- 公司发展状况缺失：显示「未查询到」
- 社保人数缺失：显示「未查询到」
- 链接字段：使用 `=HYPERLINK("url","点击查看")` 公式格式，用双引号包裹，内部双引号用 `""` 转义
- 字段中含逗号或引号时：用双引号包裹整个字段
