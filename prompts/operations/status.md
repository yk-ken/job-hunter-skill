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
