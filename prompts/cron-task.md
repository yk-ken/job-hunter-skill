# 定时任务执行指令

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
