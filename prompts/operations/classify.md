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
     编号,岗位名称,公司名,链接,薪资,地点,发布日期,经验要求,技能要求,匹配度分数,匹配度星级,BOSS姓名,BOSS职位,HR活跃度,公司主营业务,公司发展状况,社保人数,security_id,备注,分类日期
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
