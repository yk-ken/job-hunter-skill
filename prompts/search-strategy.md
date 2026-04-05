# 搜索策略

本文件定义定时任务每轮执行时的岗位搜索逻辑。Claude 在环节 5 读取此文件，按以下策略搜索 Boss 直聘岗位。

---

## 一、搜索来源

使用两个来源获取岗位，合并后统一处理：

| 来源 | 命令 | 说明 |
|------|------|------|
| 关键词搜索 | `opencli boss search "{关键词}" -f json --limit 30` | 按关键词精确搜索，每个关键词返回最多 30 条 |
| 推荐岗位 | `opencli boss recommend --limit 20` | Boss 直聘个性化推荐，最多 20 条 |

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

1. 读取 `data/meta.json`，提取 `search_keywords` 数组
2. 遍历数组中的每个关键词，对每个关键词执行一次搜索命令
3. 关键词格式为 `"{岗位方向} {城市}"`，直接作为 opencli search 的查询参数使用
4. 所有关键词搜索完成后，再执行一次 recommend 获取推荐岗位

**伪代码**：

```
all_results = []

for keyword in meta.search_keywords:
    result = run("opencli boss search \"{keyword}\" -f json --limit 30")
    all_results.append(result)

recommend_result = run("opencli boss recommend --limit 20")
all_results.append(recommend_result)

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

如果 `opencli boss recommend` 失败，跳过推荐来源，仅使用关键词搜索的结果。记录推荐获取失败的错误信息。

### 4.3 所有搜索均失败

如果所有关键词搜索和 recommend 全部失败：

1. 输出错误报告，包含每个失败命令的错误信息
2. 不中断定时任务（下一轮执行时会重新尝试）
3. 在本轮摘要中标注"本轮搜索全部失败"

### 4.4 错误日志格式

每个失败的搜索记录以下信息：

```
搜索失败：opencli boss search "{keyword}" -f json --limit 30
退出码：{exit_code}
错误输出：{stderr}
时间：{timestamp}
```

---

## 五、频率与风控

- 每个关键词搜索之间不需要额外延迟（单轮内连续执行）
- 搜索间隔由 `data/meta.json` 的 `search_interval_minutes` 控制（定时任务层面）
- 默认间隔 30 分钟，低于此值会触发警告（见 prompts/notifications.md）
- 单轮搜索的 opencli 调用次数 = `len(search_keywords) + 1`（关键词数 + 推荐），注意总调用量

---

## 六、注意事项

- opencli 返回的 JSON 结构可能因 Boss 直聘页面改版而变化，提取字段时应做容错处理（字段缺失时跳过该条记录）
- `security_id` 是去重和详情获取的核心字段，必须确保正确提取
- 每轮搜索完成后，无论成功与否，都应生成本轮摘要供用户查看
