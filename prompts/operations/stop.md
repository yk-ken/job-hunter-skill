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
