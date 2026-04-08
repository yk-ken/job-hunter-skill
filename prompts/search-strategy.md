# 搜索策略

本文件定义定时任务每轮执行时的岗位搜索逻辑。Claude 在环节 5 读取此文件，按以下策略搜索 Boss 直聘岗位。

---

## 一、搜索来源

使用两个来源获取岗位，合并后统一处理：

| 来源 | 命令 | 说明 |
|------|------|------|
| 关键词搜索 | `opencli boss search "{关键词}" -f json --limit 15 --page {页码}` | 按关键词精确搜索，每个关键词返回最多 15 条，页码逐轮递增 |
| 推荐岗位 | `opencli boss recommend --limit 10` × 3 次 | Boss 直聘个性化推荐，调用 3 次合并去重，最多 30 条 |

**合并规则**：将所有来源的结果合并为一个数组，按 `security_id` 去重（保留首次出现的记录）。

---

## 二、关键词策略

关键词从 `data/meta.json` 的 `search_keywords` 数组读取。

示例（meta.json 中）：

```json
{
  "search_keywords": [
    "全栈开发 广州",
    "AI开发 广州",
    "Go后端 广州",
    "Python 广州"
  ]
}
```

**执行方式**：

1. 读取 `data/meta.json`，提取 `search_keywords` 数组和 `search_pages` 对象
2. 关键词格式为 `"{岗位方向} {城市}"`，直接作为 opencli search 的查询参数使用
3. 每个关键词携带各自的页码（从 `search_pages` 中读取，首次为 1）
4. 所有关键词搜索完成后，执行 3 次 recommend 获取推荐岗位，合并去重

**页码轮换策略**：

每个关键词独立追踪页码，存储在 `meta.json` 的 `search_pages` 对象中：

```json
{
  "search_pages": {
    "全栈开发 广州": 3,
    "AI开发 广州": 1
  }
}
```

- 搜索返回有结果 → 该关键词 `page + 1`
- 搜索返回 0 条结果 → 该关键词 `page` 重置为 1，**立即用 page 1 补搜一次**，确保本轮不跑空。如果补搜 page 1 也返回 0 条，保持 page=1 不再重试，记录日志。如果补搜有结果，page 更新为 2（下次从第 2 页开始）。补搜前等待 2 秒。
- 如果某个关键词在 `search_pages` 中不存在，默认从第 1 页开始

**分批并行策略**：

为避免短时间大量请求触发平台风控，搜索采用分批执行：
- 关键词搜索每 **2 个**为一组，组内两个命令并行执行
- 每组完成后 **等待 5 秒**再执行下一组
- `recommend` 调用 **3 次**，单独作为最后一批，串行执行，每次之间 **随机等待 3-5 秒**

**伪代码**：

```
all_results = []
pages = meta.search_pages  # {"全栈开发 广州": 3, "AI开发 广州": 1, ...}

# 分批执行关键词搜索，每批 2 个并行
keywords = meta.search_keywords
for i in range(0, len(keywords), 2):
    batch = keywords[i:i+2]
    batch_results = parallel([
        run(f"opencli boss search \"{kw}\" -f json --limit 15 --page {pages.get(kw, 1)}")
        for kw in batch
    ])

    # 页码处理：有结果 +1，空结果重置为 1 并补搜
    for idx, kw in enumerate(batch):
        if len(batch_results[idx]) > 0:
            pages[kw] = pages.get(kw, 1) + 1
            all_results.append(batch_results[idx])
        else:
            pages[kw] = 1
            sleep(2)  # 补搜前短暂延迟
            retry = run(f"opencli boss search \"{kw}\" -f json --limit 15 --page 1")
            if len(retry) > 0:
                pages[kw] = 2
            # 补搜也返回 0 条时保持 page=1，不再重试
            all_results.append(retry)

    if i + 2 < len(keywords):
        sleep(5)  # 批次间等待 5 秒

# recommend 调用 3 次，随机间隔 3-5 秒
for _ in range(3):
    recommend_result = run("opencli boss recommend --limit 10")
    if recommend_result is not None:
        all_results.append(recommend_result)
    else:
        log("recommend 调用失败，继续尝试剩余次数")
    sleep(random(3, 5))

# 更新 meta.json 的 search_pages
meta.search_pages = pages

merged = deduplicate(all_results, key="security_id")
```

---

## 三、搜索结果处理

### 3.1 字段提取

从每个搜索结果的 JSON 中提取以下字段：

| 字段 | 用途 |
|------|------|
| `job_name` | 岗位名称 |
| `company_name` | 公司名称 |
| `salary` | 薪资范围（如 "15-23K"） |
| `location` | 工作地点 |
| `security_id` | 岗位唯一标识（用于去重和详情获取） |
| `job_id` | 岗位 ID（用于去重和链接生成） |

### 3.2 去重过滤

将提取的结果与 `data/meta.json` 的 `recorded_job_ids` 数组对比：

1. 遍历合并后的搜索结果
2. 对每个结果，检查其 `security_id` 是否存在于 `recorded_job_ids` 中
3. 如果已存在，跳过（已记录过的岗位不再重复处理）
4. 如果不存在，标记为新岗位，加入本轮新岗位列表

### 3.3 输出

过滤后产生两个输出：

- **新岗位 security_id 列表**：传递给环节 6（获取岗位详情），逐一调用 `opencli boss detail --security-id {id} -f json`
- **本轮搜索摘要**：记录本轮搜索统计（各关键词返回数量、推荐返回数量、去重后数量、过滤已记录后数量）

---

## 四、错误处理

### 4.1 单个关键词搜索失败

opencli 命令可能返回以下错误退出码：

| 退出码 | 含义 | 处理方式 |
|--------|------|---------|
| 69 | Browser Bridge 未连接 | 记录错误日志，跳过该关键词，继续其他搜索 |
| 77 | Chrome 未登录或会话过期 | 记录错误日志，跳过该关键词，继续其他搜索 |
| 75 | 网络或平台错误 | 记录错误日志，跳过该关键词，继续其他搜索 |

**处理原则**：单个关键词失败不阻塞整个搜索流程。记录失败的关键词和错误信息，继续执行剩余关键词的搜索和 recommend。

### 4.2 recommend 失败

如果单次 `opencli boss recommend` 失败，记录错误日志，继续尝试剩余次数。如果全部 3 次均失败，跳过推荐来源，仅使用关键词搜索的结果。

### 4.3 所有搜索均失败

如果所有关键词搜索和 recommend 全部失败：

1. 输出错误报告，包含每个失败命令的错误信息
2. 不中断定时任务（下一轮执行时会重新尝试）
3. 在本轮摘要中标注"本轮搜索全部失败"

### 4.4 错误日志格式

每个失败的搜索记录以下信息：

```
搜索失败：opencli boss search "{keyword}" -f json --limit 15
退出码：{exit_code}
错误输出：{stderr}
时间：{timestamp}
```

---

## 五、频率与风控

- 关键词搜索每 2 个并行执行，批次间等待 5 秒，降低短时间内大量请求的风险
- recommend 3 次串行执行，每次间隔随机 3-5 秒
- 搜索间隔由 `data/meta.json` 的 `search_interval_minutes` 控制（定时任务层面）
- 默认间隔 30 分钟，低于此值会触发警告（见 prompts/notifications.md）
- 单轮搜索的 opencli 调用次数 = `len(search_keywords) + 3`（关键词数 + 推荐 3 次），注意总调用量

---

## 六、注意事项

- opencli 返回的 JSON 结构可能因 Boss 直聘页面改版而变化，提取字段时应做容错处理（字段缺失时跳过该条记录）
- `security_id` 是去重和详情获取的核心字段，必须确保正确提取
- 每轮搜索完成后，无论成功与否，都应生成本轮摘要供用户查看
