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
| `/job-hunter update` | 更新 Job Hunter Skill 到最新版本 |
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
