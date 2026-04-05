# Job Hunter Skill 实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 实现 job-hunter-skill 的所有核心文件，使 `/job-hunter` 命令可完整运行。

**Architecture:** Claude Code Skill 架构。SKILL.md 为入口，读取 prompts/ 下的模板文件驱动各环节行为。数据存储在 data/ 目录。通过 CronCreate 创建定时任务，通过 opencli CLI 与 Boss 直聘交互。

**Tech Stack:** Claude Code Skill (SKILL.md + markdown prompts), opencli CLI, CronCreate/CronDelete

---

## 依赖关系

```
Task 1 (intake.md) ──┐
Task 2 (search)  ────┤
Task 3 (filter)  ────┼──→ Task 6 (SKILL.md) ──→ Task 8 (README.md)
Task 4 (scorer)  ────┤
Task 5 (notifications)┘

Task 7 (INSTALL.md) — 独立，可与 Task 6 并行
```

**Task 1-5 完全独立，可并行执行。** Task 6 依赖 Task 1-5 全部完成。Task 7-8 依赖 Task 6。

---

## Task 1: prompts/intake.md — 新用户引导模板

**Files:**
- Create: `prompts/intake.md`

**Description:** 新用户首次运行时的交互式问答模板。通过 AskUserQuestion 收集求职画像信息，生成 data/job-profile.md。

- [ ] **Step 1: 创建 intake.md**

创建 `prompts/intake.md`，内容为 Claude 执行新用户引导时的完整指令，包括：

1. **分步问答设计**（每步用 AskUserQuestion，最多4个问题一组）：
   - 第1组：城市、优先区域、排除区域、工作年限
   - 第2组：期望薪资范围（最低/期望/最高）、岗位方向（多选）
   - 第3组：技术栈（自由输入）、休息制度要求
   - 第4组：公司类型偏好、公司规模要求、搜索关键词建议

2. **生成画像文件指令**：
   - 将收集的信息按 PRD 5.1 节的格式写入 `data/job-profile.md`
   - 如果用户没提供搜索关键词，根据岗位方向+城市自动生成默认关键词

3. **初始化运行数据指令**：
   - 创建 `data/job-candidates.md`，写入标题 `# 候选岗位列表`
   - 创建 `data/meta.json`，初始结构：
     ```json
     {
       "created_at": "<当前ISO时间>",
       "updated_at": "<当前ISO时间>",
       "last_run_time": null,
       "cron_job_id": null,
       "run_count": 0,
       "total_candidates": 0,
       "next_candidate_number": 1,
       "search_interval_minutes": 30,
       "recorded_job_ids": [],
       "exclude_keywords": [],
       "search_keywords": ["<从画像提取>"]
     }
     ```

4. **环境检测指令**（在问答之前执行）：
   - 检测 `opencli --version`：未安装则提示 `npm install -g @jackwener/opencli`，询问用户是否安装
   - 检测 `opencli doctor`：Browser Bridge 未连接则给出安装步骤
   - 检测 Boss 登录状态：`opencli boss search "测试" --limit 1`，失败则提示用户在 Chrome 登录 zhipin.com
   - 所有检测项逐项展示结果，不自动安装任何东西

- [ ] **Step 2: 检查文件格式**

验证 intake.md 的问答流程覆盖了 PRD 中 job-profile.md 的所有字段。

---

## Task 2: prompts/search-strategy.md — 搜索策略模板

**Files:**
- Create: `prompts/search-strategy.md`

**Description:** 定义定时任务执行时的搜索策略。Claude 读取此文件决定如何调用 opencli 搜索岗位。

- [ ] **Step 1: 创建 search-strategy.md**

创建 `prompts/search-strategy.md`，内容包括：

1. **搜索来源**：
   - `opencli boss search "关键词" -f json --limit 30`：按关键词搜索
   - `opencli boss recommend --limit 20`：获取推荐岗位
   - 两种来源合并，按 security_id 去重

2. **关键词策略**：
   - 从 `data/meta.json` 的 `search_keywords` 读取关键词列表
   - 每个关键词执行一次 `opencli boss search`
   - 关键词格式：`"{岗位方向} {城市}"`

3. **搜索结果处理**：
   - 从 JSON 结果中提取：job_name, company_name, salary, location, security_id, job_id
   - 与 `data/meta.json` 的 `recorded_job_ids` 对比，过滤已记录的岗位
   - 对新岗位收集 security_id 列表，传递给详情获取环节

4. **错误处理**：
   - opencli 命令失败（退出码 69/77/75）：记录错误，跳过该关键词，继续其他搜索
   - 所有搜索都失败：输出错误报告，不中断定时任务

- [ ] **Step 2: 验证策略与 PRD 环节 5 一致**

---

## Task 3: prompts/filter.md — 筛选规则模板

**Files:**
- Create: `prompts/filter.md`

**Description:** 定义岗位筛选的硬条件规则。Claude 读取此文件和 job-profile.md 决定哪些岗位进入评分环节。

- [ ] **Step 1: 创建 filter.md**

创建 `prompts/filter.md`，内容包括：

1. **筛选流程**（按顺序执行，任一条件不满足则排除）：

   ```
   输入：搜索结果列表 + data/job-profile.md + data/meta.json

   Step 1: ID 去重
     - 检查 security_id 是否在 meta.json.recorded_job_ids 中
     - 已存在 → 排除

   Step 2: 排除关键词
     - 检查岗位名称和描述是否包含 meta.json.exclude_keywords 中的关键词
     - 匹配 → 排除

   Step 3: 城市区域
     - 检查地点是否匹配 job-profile 的优先区域
     - 检查地点是否在排除区域中
     - 不匹配或被排除 → 排除

   Step 4: 薪资下限
     - 解析薪资范围（如 "15-25K" → 15000）
     - 低于 job-profile 最低薪资 → 排除

   Step 5: 休息制度
     - 岗位明确标注"单休"且用户最低要求为"单双休"
     - 单休且用户不接受单休 → 排除

   Step 6: 公司规模（软条件）
     - 公司规模低于最低要求且非开发主体公司
     - 不排除，但标记为降低评分
   ```

2. **薪资解析规则**：
   - `"15-25K"` → 最低 15000，最高 25000
   - `"15-25K·13薪"` → 同上，额外标记有年终奖
   - `"面议"` → 跳过薪资筛选，进入评分环节

3. **地点匹配规则**：
   - 精确匹配优先：岗位地点包含优先区域名称
   - 宽松匹配：岗位地点包含城市名称且不在排除区域
   - 远程岗位：标记为"支持远程"的不受地点限制

4. **输出格式**：
   - 通过筛选的岗位列表（含标记字段）
   - 被排除的岗位列表及排除原因（用于调试）

- [ ] **Step 2: 验证规则与 PRD 第六节一致**

---

## Task 4: prompts/scorer.md — 评分排序模板

**Files:**
- Create: `prompts/scorer.md`

**Description:** 定义岗位评分和排序规则。Claude 读取此文件为通过筛选的岗位计算匹配度分数。

- [ ] **Step 1: 创建 scorer.md**

创建 `prompts/scorer.md`，内容包括：

1. **评分维度与权重**（满分 100）：

   ```
   发布日期（30分）：
     - 1天内：30
     - 2天：27
     - 3天：24
     - 5天：18
     - 7天：14
     - 10天：8
     - 14天：4
     - 超过14天：0（排除）
     - 无发布日期：15（默认中等偏上，不过度惩罚）

   薪资匹配（25分）：
     - 薪资范围完全在期望内：25
     - 期望薪资在岗位范围内：25
     - 部分重叠：按重叠比例 × 25
     - 低于最低期望：0

   技术栈匹配（20分）：
     - 计算画像技术栈与岗位要求的重叠度
     - 重叠比例 × 20

   地点匹配（15分）：
     - 在优先区域：15
     - 在城市内但非优先区域：10
     - 支持远程：12
     - 不在城市：0

   公司规模（10分）：
     - 匹配偏好类型（外企/大厂等）：10
     - 100人以上：8
     - 50-99人：5
     - 50人以下（开发主体）：3
     - 50人以下（非开发主体）：1
   ```

2. **星级映射**：
   - 90-100：★★★★★
   - 75-89：★★★★☆
   - 60-74：★★★☆☆
   - 40-59：★★☆☆☆
   - 40以下：★☆☆☆☆

3. **排序规则**：
   - 按总分降序
   - 同分时按发布日期降序（新的优先）

4. **输出格式**：
   - 评分后的岗位列表，每个岗位附带分数和星级

- [ ] **Step 2: 验证规则与 PRD 第七节一致**

---

## Task 5: prompts/notifications.md — 模板化提示

**Files:**
- Create: `prompts/notifications.md`

**Description:** 所有关键节点的用户提示模板。Claude 在特定环节读取对应模板输出，确保提示质量一致。

- [ ] **Step 1: 创建 notifications.md**

创建 `prompts/notifications.md`，包含以下 5 套模板：

**模板 1：定时任务启动成功**（环节 4 完成后）
```
Job Hunter 已启动！

运行配置：
  - 搜索间隔：每 {search_interval_minutes} 分钟
  - 搜索关键词：{keywords_count} 组
  - 搜索来源：Boss 直聘（search + recommend）

注意事项：
  1. 定时任务仅在 Claude Code 运行期间执行，关闭终端/退出 Claude Code 后任务暂停
  2. durable 任务最长持续 7 天，到期后需要重新运行 /job-hunter start
  3. 岗位记录文件：data/job-candidates.md，可随时查看

管理命令：
  /job-hunter status   — 查看运行状态
  /job-hunter stop     — 停止搜索
  /job-hunter start    — 恢复搜索
```

**模板 2：搜索频率警告**（用户设置间隔 < 30 分钟时）
```
搜索频率警告：当前设定的间隔为 {interval} 分钟

风险等级：{risk_level}
  - 频繁请求 Boss 直聘可能触发平台风控
  - 可能导致账号被限制访问
  - 建议间隔不低于 30 分钟

是否确认使用 {interval} 分钟间隔？（y/n）
```

风险等级映射：
- 15-29 分钟："低风险"
- 10-14 分钟："中风险 — 强烈建议不低于 30 分钟"

**模板 3：任务即将到期**（运行接近 7 天时追加）
```
Job Hunter 提醒：定时任务即将在约 {remaining_hours} 小时后到期（durable 任务最长 7 天）

到期后不会丢失数据，但搜索将停止。
如需继续，请在到期后运行 /job-hunter start 重新启动。
```

**模板 4：环境检测未通过**（环境检测失败时）
```
环境检测未通过，以下问题需要解决：

  [{status}] opencli .............. {result}
  [{status}] Browser Bridge ....... {result}
  [{status}] Boss 直聘登录 ........ {result}

{针对每项缺失的具体安装/配置指引}

请解决以上问题后重新运行 /job-hunter
```

状态标记：`✅` 已就绪 / `❌` 未通过 / `⚠️` 部分就绪

**模板 5：每轮搜索报告**（环节 10）
```
Job Hunter 本轮报告（{date}）

发现 {new_count} 个新匹配岗位，已记录到 data/job-candidates.md

{对每个新岗位按匹配度排序的一行摘要}
  #N  岗位名 @ 公司 | 薪资 | 地点 | 发布于{date} | 匹配{score}分

累计已记录 {total_count} 个候选岗位。
```

如果本轮无新岗位：
```
Job Hunter 本轮报告（{date}）

本轮未发现新的匹配岗位。累计已记录 {total_count} 个候选岗位。
```

- [ ] **Step 2: 验证模板覆盖 PRD 第十三节所有场景**

---

## Task 6: SKILL.md — 主入口

**Files:**
- Create: `SKILL.md`

**Depends on:** Task 1, 2, 3, 4, 5

**Description:** Skill 主入口文件。包含 frontmatter、完整流程定义、定时任务 prompt 内容。这是整个 skill 的核心文件。

- [ ] **Step 1: 创建 SKILL.md frontmatter**

```yaml
---
name: job-hunter
description: "自动从 Boss 直聘发现、筛选、记录合适岗位。使用 /job-hunter 启动定时求职助手。"
---
```

- [ ] **Step 2: 编写入口逻辑**

SKILL.md 主体内容，包含：

1. **触发条件**：
   - `/job-hunter` 或 `/job-hunter start` — 启动
   - `/job-hunter stop` — 停止
   - `/job-hunter status` — 查看状态

2. **主入口分支逻辑**：
   ```
   1. 读取 data/job-profile.md 是否存在
   2. 不存在 → 执行 prompts/intake.md 的引导流程
   3. 存在 → 读取 data/meta.json 获取运行状态
   4. 根据 argument 判断操作：
      - start / 无参数：创建 CronCreate 定时任务
      - stop：CronDelete 停止任务
      - status：读取 meta.json 显示状态
   ```

- [ ] **Step 3: 编写 CronCreate prompt**

CronCreate 的 prompt 内容是定时任务每次执行时的完整指令：

```
1. 读取 data/job-profile.md 和 data/meta.json
2. 读取 prompts/search-strategy.md，执行搜索
3. 对新岗位调用 opencli boss detail 获取详情
4. 读取 prompts/filter.md，执行筛选
5. 读取 prompts/scorer.md，执行评分排序
6. 将新岗位追加到 data/job-candidates.md
7. 更新 data/meta.json（recorded_job_ids、last_run_time、run_count）
8. 读取 prompts/notifications.md 的每轮报告模板，输出结果
```

- [ ] **Step 4: 编写搜索频率配置逻辑**

在创建定时任务前：
```
1. 读取 meta.json 的 search_interval_minutes
2. 如果 < 30：
   - 读取 prompts/notifications.md 的频率警告模板
   - 输出警告，询问用户确认
   - < 10 分钟：拒绝，强制最低 10 分钟
3. 用户确认后创建 CronCreate
```

- [ ] **Step 5: 编写启动成功提示**

创建定时任务成功后，读取 `prompts/notifications.md` 的启动成功模板，填充变量并输出。

- [ ] **Step 6: 提交**

验证 SKILL.md 引用了所有 prompt 文件，逻辑闭环。

---

## Task 7: INSTALL.md — 安装说明

**Files:**
- Create: `INSTALL.md`

**Description:** 用户安装此 skill 的说明文档。

- [ ] **Step 1: 创建 INSTALL.md**

内容包含：

1. **前置条件**：
   - Claude Code 已安装
   - Node.js >= 20
   - Chrome 浏览器
   - opencli 已安装（`npm install -g @jackwener/opencli`）
   - Browser Bridge Chrome 扩展已安装

2. **安装 skill**：
   ```bash
   # 方式一：安装到当前项目
   mkdir -p .claude/skills
   git clone https://github.com/yk-ken/job-hunter-skill .claude/skills/job-hunter

   # 方式二：全局安装（所有项目可用）
   git clone https://github.com/yk-ken/job-hunter-skill ~/.claude/skills/job-hunter
   ```

3. **验证安装**：
   ```bash
   # 在 Claude Code 中输入
   /job-hunter
   ```

4. **首次使用**：
   - 运行 `/job-hunter` 后会自动进入环境检测和新用户引导
   - 按提示完成即可

---

## Task 8: README.md — 项目介绍

**Files:**
- Create: `README.md`

**Depends on:** Task 6

**Description:** GitHub 项目首页。

- [ ] **Step 1: 创建 README.md**

内容包含：

1. **标题和简介**：一句话概括 + 核心价值
2. **亮点**：
   - 自动搜索 + 筛选 + 记录
   - 30 分钟自动运行
   - 智能评分（发布日期权重）
   - 新用户开箱即用
   - 持续优化搜索策略
3. **快速开始**：安装 → 运行 → 3 步
4. **功能列表**：
   - 求职画像引导
   - 多关键词搜索
   - 智能筛选（薪资/地点/休息制度）
   - 综合评分排序
   - 发布日期新鲜度追踪
   - 搜索频率可配置
   - 环境自动检测
5. **命令列表**：/job-hunter, /job-hunter start, /job-hunter stop, /job-hunter status
6. **文件结构**：与 PRD 一致
7. **依赖说明**：opencli, Claude Code
8. **License**：MIT

---

## 并行执行策略

```
并行批次 1（Task 1-5，同时派发 5 个 agent）：
  Agent A → prompts/intake.md
  Agent B → prompts/search-strategy.md
  Agent C → prompts/filter.md
  Agent D → prompts/scorer.md
  Agent E → prompts/notifications.md

批次 1 全部完成后 →

并行批次 2（Task 6-8，同时派发 3 个 agent）：
  Agent F → SKILL.md
  Agent G → INSTALL.md（独立，可与 F 并行）
  Agent H → README.md（依赖 F 完成，但内容可先行起草）
```

**审查策略**：每个 Task 完成后做规格合规审查（对照 PRD 对应章节），确认无遗漏后标记完成。

---

## Self-Review Checklist

- [ ] PRD 第一节（产品概述）→ Task 6 覆盖
- [ ] PRD 第二节（用户流程）→ Task 6 覆盖（入口逻辑 + CronCreate prompt）
- [ ] PRD 第三节（文件结构）→ 所有 Task 覆盖
- [ ] PRD 第四节（文件与环节对照）→ 所有 Task 覆盖
- [ ] PRD 第五节（数据文件格式）→ Task 1 覆盖（job-profile.md + meta.json 初始化）
- [ ] PRD 第六节（筛选规则）→ Task 3 覆盖
- [ ] PRD 第七节（评分规则）→ Task 4 覆盖
- [ ] PRD 第八节（frontmatter）→ Task 6 覆盖
- [ ] PRD 第九节（opencli 依赖）→ Task 2, 6 覆盖
- [ ] PRD 第十节（Claude Code 依赖）→ Task 6 覆盖
- [ ] PRD 第十一节（约束与边界 + 频率安全 + 环境检测）→ Task 1, 5, 6 覆盖
- [ ] PRD 第十三节（模板化提示）→ Task 5 覆盖
- [ ] PRD 第十四节（搜索间隔配置）→ Task 6 覆盖
- [ ] 无 placeholder：检查所有 Task 的代码块是否包含实际内容
