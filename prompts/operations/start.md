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
