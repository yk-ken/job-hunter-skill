# 设计文档：候选岗位三分类机制

日期：2026-04-08

## 背景

Job Hunter 定时任务自动搜索、筛选、评分岗位并写入 `data/job-candidates.csv`。用户人工审核候选列表后，需要对岗位进行分类管理。现有的排除机制（操作 D）处理"不合适"的岗位，但通过审核的岗位也存在两类不同性质的机会：

1. **练手岗位** — 某些方面不完全理想（如距离远、薪资偏低），但值得面试以积累经验
2. **目标岗位** — 各方面（薪资、地点、工作内容等）都符合预期，真心想去

用户需要一个机制将候选岗位分为三档：排除、练手、目标，分别归档到独立文件，候选列表仅保留未审核岗位。

## 核心设计

### 数据文件

新增两个归档文件，格式与 `job-excluded.csv` 同构：

| 文件 | 用途 |
|------|------|
| `data/job-practice.csv` | 练手岗位归档 |
| `data/job-target.csv` | 心仪目标岗位归档 |

列格式（在 `job-candidates.csv` 基础上追加两列）：

```
编号,岗位名称,公司名,薪资,地点,发布日期,经验要求,技能要求,匹配度分数,匹配度星级,BOSS姓名,BOSS职位,HR活跃度,公司主营业务,公司发展状况,社保人数,链接,security_id,备注,分类日期
```

- **编号**：各文件自身的顺序编号，从 1 开始递增
- **备注**：用户标记时口述的分类原因，如"薪资偏低但技术栈对口，拿来练手"。用户可跳过此步骤，此时填「无」
- **分类日期**：标记当天的日期，格式 `YYYY-MM-DD`

### meta.json 新增字段

```json
{
  "practice_count": 0,
  "target_count": 0,
  "next_practice_number": 1,
  "next_target_number": 1
}
```

### 交互流程

统一为操作 E（岗位分类），覆盖练手和目标两种分类。

#### 触发方式

| 用户说 | 动作 |
|--------|------|
| 「练手编号X」 | 分类为练手 |
| 「目标编号X」 | 分类为目标 |
| 「标记编号X」 | 询问用户分类为练手还是目标 |

#### 流程步骤

```
步骤 1  用户说「练手/目标/标记 编号X」
步骤 2  我从 job-candidates.csv 读取该岗位，展示确认：
          编号：X  岗位：xxx  公司：xxx  薪资：xxx  地点：xxx
        （如果是「标记」且未指定类型，此时询问练手还是目标）
步骤 3  我问「备注一下分类原因？（可直接回车跳过）」
步骤 4  用户口述原因或跳过
步骤 5  执行变更：
          - 追加到 job-practice.csv 或 job-target.csv
          - 从 job-candidates.csv 移除该行
          - 剩余行重编号（按原顺序重新编号为 1, 2, 3, ...）
          - 更新 meta.json：
            - total_candidates 减 1，next_candidate_number 同步调整
            - practice_count 或 target_count 加 1
            - next_practice_number 或 next_target_number 加 1
            - updated_at 更新为当前 ISO 时间
步骤 6  输出确认：
          已将「xxx @ xxx」标记为练手/目标
          备注：xxx
          候选列表剩余 N 个岗位
```

### 查看列表

#### 操作 F：查看练手列表

触发：`/job-hunter practice`

展示格式：

```
练手岗位（共 N 个）：

| # | 岗位名称 | 公司 | 薪资 | 备注 | 分类日期 |
|---|---------|------|------|------|---------|
| 1 | xxx     | xxx  | xxx  | 薪资偏低但技术栈对口 | 2026-04-08 |
```

文件不存在或仅有表头行时：

```
练手列表为空，暂无标记为练手的岗位。
```

#### 操作 G：查看目标列表

触发：`/job-hunter target`

展示格式与练手列表一致，标题改为「目标岗位」。

### 状态展示增强

操作 C（`/job-hunter status`）增加分类统计：

```
Job Hunter 状态：运行中

  定时任务 ID：xxx
  搜索间隔：每 30 分钟
  候选列表：5 个待审核
  练手岗位：3 个
  目标岗位：2 个
  已排除：4 个
  上次运行时间：xxx

管理命令：
  /job-hunter stop     — 停止搜索
  /job-hunter start    — 重启搜索（需先 stop）
  /job-hunter practice — 查看练手列表
  /job-hunter target   — 查看目标列表
  /job-hunter exclude  — 查看排除列表
```

## 涉及的文件变更

### 1. SKILL.md

**触发条件表**新增三行：

| 命令 | 操作 |
|------|------|
| 用户说「练手编号X」 | 触发分类流程，分类为练手 |
| 用户说「目标编号X」 | 触发分类流程，分类为目标 |
| 用户说「标记编号X」 | 触发分类流程，询问分类类型 |

**新增章节**：
- 操作 E — 岗位分类（练手/目标的统一流程）
- 操作 F — 查看练手列表
- 操作 G — 查看目标列表

**操作 C（status）增强**：增加练手/目标计数和管理命令提示。

**步骤 1.5 老用户升级检查**：必要文件清单新增 `job-practice.csv` 和 `job-target.csv`。

**meta.json 字段说明**新增 `practice_count`、`target_count`、`next_practice_number`、`next_target_number`。

### 2. prompts/intake.md

**第四步：初始化运行数据**新增：
- 4.1.6 创建练手文件（写入表头行）
- 4.1.7 创建目标文件（写入表头行）

**meta.json 初始内容**新增四个字段（practice_count: 0, target_count: 0, next_practice_number: 1, next_target_number: 1）。

**确认信息**更新：

```
运行数据已初始化：
  - data/job-candidates.csv — 候选岗位记录文件（CSV 格式，可用 Excel 打开）
  - data/job-excluded.csv — 排除岗位归档文件
  - data/job-practice.csv — 练手岗位归档文件
  - data/job-target.csv — 目标岗位归档文件
  - data/meta.json — 运行元数据
```

### 3. prompts/filter.md

不改动。练手/目标岗位的 `security_id` 已包含在 `recorded_job_ids` 中（通过候选列表写入时已记录），不会在后续搜索中重复推荐。

### 4. prompts/scorer.md

不改动。

## 不涉及的内容

- 不修改搜索策略（`prompts/search-strategy.md`）
- 不修改通知模板（`prompts/notifications.md`）
- 不修改筛选逻辑（`prompts/filter.md`）
- 不新增定时任务或 cron 作业
- 分类操作不触发搜索条件更新（无反馈闭环）
