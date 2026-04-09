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
   编号,岗位名称,公司名,链接,薪资,地点,发布日期,经验要求,技能要求,匹配度分数,匹配度星级,BOSS姓名,BOSS职位,HR活跃度,公司主营业务,公司发展状况,社保人数,security_id,排除原因,排除日期
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
