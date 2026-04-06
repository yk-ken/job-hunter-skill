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
- **存在** → 跳到「步骤 3：读取运行状态」

### 步骤 2：新用户引导流程

读取并执行 `${CLAUDE_SKILL_DIR}/prompts/intake.md` 中的完整引导流程。

该流程包含：
1. 环境检测（Node.js、opencli、Browser Bridge、Boss 直聘登录）
2. 分步问答收集求职画像（4 组问题）
3. 生成 `data/job-profile.md`
4. 初始化 `data/job-candidates.md` 和 `data/meta.json`

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
已记录的 {total_candidates} 个候选岗位仍在 data/job-candidates.md 中。
```

---

## 操作 C：status — 查看状态

读取 `data/meta.json`，输出状态信息：

**如果 cron_job_id 不为 null（运行中）**：

```
Job Hunter 状态：运行中

  定时任务 ID：{cron_job_id}
  搜索间隔：每 {search_interval_minutes} 分钟
  搜索关键词：{search_keywords 的数量} 组
  已运行次数：{run_count}
  已记录岗位：{total_candidates} 个
  上次运行时间：{last_run_time}

管理命令：
  /job-hunter stop     — 停止搜索
  /job-hunter start    — 重启搜索（需先 stop）
```

**如果 cron_job_id 为 null（已停止）**：

```
Job Hunter 状态：已停止

  已记录岗位：{total_candidates} 个
  上次运行时间：{last_run_time 或 "从未运行"}

运行 /job-hunter start 开始搜索。
```

**如果 last_run_time 不为 null 且距离 created_at + 7 天不足 24 小时**：

在状态信息后追加到期提醒。读取 `${CLAUDE_SKILL_DIR}/prompts/notifications.md` 中的「任务即将到期」模板，填充：
- `{remaining_hours}` = floor((created_at + 7天 - 当前时间) / 小时)

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

1. 从 data/meta.json 提取 search_keywords 数组
2. 对每个关键词执行：opencli boss search "{keyword}" -f json --limit 30
3. 执行推荐获取：opencli boss recommend --limit 20
4. 将所有结果按 security_id 合并去重
5. 与 data/meta.json 的 recorded_job_ids 对比，筛选出新出现的岗位

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

输入：步骤 3 获取的岗位详情列表 + data/job-profile.md + data/meta.json

按以下顺序执行筛选（任一步骤不通过则排除）：
1. ID 去重：security_id 在 recorded_job_ids 中 → 排除
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
- 公司规模（10分）：匹配偏好=10，100人以上=8，50-99人=5，50人以下(开发主体)=3，其他=1

星级映射：
- 90-100：★★★★★
- 75-89：★★★★☆
- 60-74：★★★☆☆
- 40-59：★★☆☆☆
- 40以下：★☆☆☆☆

按总分降序排列，同分按发布日期降序，同分同日期按薪资匹配度降序。

## 步骤 6：记录新岗位

将通过筛选并评分的岗位追加到 data/job-candidates.md。

按日期分组，最新的日期在最前面。同一日期内的岗位按评分降序排列。

每个岗位的记录格式如下：

```markdown
### #N 岗位名称 — 公司名
- 薪资：XX-XXK
- 地点：城市·区域·具体地址（如有）
- 发布日期：YYYY-MM-DD
- 经验要求：X-X年
- 技能：技能1, 技能2, 技能3
- 匹配度：★★★★☆（XX分）
- BOSS：姓名 · 职位 · 刚刚活跃
- 链接：https://www.zhipin.com/job_detail/{job_id}.html
- security_id：xxx
```

其中 #N 使用 data/meta.json 中的 next_candidate_number，每记录一个岗位后编号递增 1。BOSS 行末尾附带 HR 活跃度信息（active_time 字段）。

新日期分组的格式：

```markdown
---

## YYYY-MM-DD 发现

### #N ...
```

将新日期分组插入到文件的「# 候选岗位列表」标题之后、已有日期分组之前。

## 步骤 7：更新 meta.json

更新 data/meta.json 的以下字段：

- recorded_job_ids：将本轮通过筛选的所有岗位的 security_id 追加到数组中（不包括被筛选排除的岗位，避免重复推荐）
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
  #{number}  {job_name} @ {company} | {salary} | {location} | 发布于 {post_date} | HR{active_time} | 匹配 {score}分
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

写入 `data/job-candidates.md` 的每个岗位必须包含以下所有字段，格式严格如下：

```markdown
### #N 岗位名称 — 公司名
- 薪资：XX-XXK
- 地点：城市·区域·具体地址（如有）
- 发布日期：YYYY-MM-DD
- 经验要求：X-X年
- 技能：技能1, 技能2, 技能3
- 匹配度：★★★★☆（XX分）
- BOSS：姓名 · 职位 · 刚刚活跃
- 链接：https://www.zhipin.com/job_detail/{job_id}.html
- security_id：xxx
```

如果某个字段信息缺失，使用以下默认值：
- 具体地址缺失：仅显示「城市·区域」
- 发布日期缺失：显示「未知」
- 经验要求缺失：显示「未知」
- BOSS 信息缺失：显示「未知 · 未知 · 未知」
- HR 活跃度缺失：BOSS 行末尾显示「未知」
- 薪资为「面议」时：显示「面议」
