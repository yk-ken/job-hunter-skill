# Job Hunter Skill — 产品需求文档 v1.0

---

## 一、产品概述

**job-hunter-skill** 是一个运行在 Claude Code 上的自动化求职助手 Skill。

用户通过交互式引导建立求职画像，Skill 自动创建定时任务，周期性地从 Boss 直聘发现、筛选、记录匹配的岗位。用户只需查看持续增长的候选岗位文档，从中挑选心仪的岗位投递即可。

**核心价值**：减少用户手动搜索和筛选岗位的时间，提高求职效率和命中率。

---

## 二、用户流程

```
用户触发 /job-hunter
        │
        ├── [环节 0] 环境检测
        │     检测运行环境是否满足条件：
        │     · opencli 是否已安装（opencli --version）
        │     · Browser Bridge 扩展是否已连接（opencli doctor）
        │     · Chrome 是否已登录 Boss 直聘
        │     如有缺失：
        │     · 告知用户需要安装什么
        │     · 询问用户是否愿意安装（不得未经同意自动安装）
        │     · 需要人工操作的部分（如配置 Chrome 扩展）给出明确步骤指引
        │     所有环境就绪后进入下一步
        │
        ├── [分支 A] 新用户（data/job-profile.md 不存在）
        │       │
        │       ├── [环节 1] 交互式引导录入
        │       │     读取 prompts/intake.md
        │       │     通过 AskUserQuestion 收集求职画像
        │       │     · 城市/区域偏好（可多选）
        │       │     · 排除区域
        │       │     · 期望薪资范围
        │       │     · 岗位方向（全栈/后端/前端/AI 等）
        │       │     · 技术栈
        │       │     · 工作年限
        │       │     · 休息制度要求
        │       │     · 公司类型偏好
        │       │     · 公司规模要求
        │       │     · 搜索关键词建议（用户可自定义）
        │       │
        │       ├── [环节 2] 生成求职画像文件
        │       │     → data/job-profile.md
        │       │
        │       ├── [环节 3] 初始化运行数据
        │       │     → data/job-candidates.md（空文件）
        │       │     → data/meta.json（初始状态）
        │       │
        │       └── ↓ 进入环节 4
        │
        └── [分支 B] 老用户（data/job-profile.md 已存在）
                │
                ├── [环节 2'] 读取求职画像
                │     ← data/job-profile.md
                │
                ├── [环节 3'] 读取运行状态
                │     ← data/meta.json（去重列表、排除规则等）
                │
                └── ↓ 进入环节 4
        │
├── [环节 4] 创建定时任务
│     从 data/meta.json 读取 search_interval（默认 30 分钟）
│     CronCreate（durable: true，按用户设定的间隔执行）
│     创建成功后，向用户输出标准提示（见「十三、模板化提示」）
│     以下为定时任务的每次执行内容：
│     │
│     ├── [环节 5] 搜索岗位
│     │     ← prompts/search-strategy.md（搜索策略）
│     │     执行多关键词搜索：
│     │     · opencli boss search "关键词1" -f json --limit 30
│     │     · opencli boss search "关键词2" -f json --limit 30
│     │     · opencli boss recommend --limit 20
│     │     合并去重结果
│     │
│     ├── [环节 6] 获取岗位详情
│     │     对新出现的岗位调用：
│     │     · opencli boss detail --security-id xxx -f json
│     │     提取：薪资范围、具体工作地址、发布日期、经验要求、
│     │           技能要求、公司信息、BOSS 信息等
│     │
│     ├── [环节 7] 筛选
│     │     ← prompts/filter.md（筛选规则）
│     │     ← data/job-profile.md（画像）
│     │     ← data/meta.json（去重列表 + 排除规则）
│     │     硬条件过滤：
│     │     · 地点匹配
│     │     · 薪资范围
│     │     · 休息制度
│     │     · 关键词排除
│     │     · ID 去重（已记录的不再重复）
│     │
│     ├── [环节 8] 评分排序
│     │     ← prompts/scorer.md（评分规则）
│     │     综合评分维度：
│     │     · 发布日期（越新权重越高）
│     │     · 薪资匹配度
│     │     · 技术栈匹配度
│     │     · 公司规模匹配度
│     │     · 地点匹配度
│     │     按综合得分降序排列
│     │
│     ├── [环节 9] 记录新岗位
│     │     → data/job-candidates.md（追加新岗位）
│     │     → data/meta.json（更新去重列表、运行时间）
│     │
│     └── [环节 10] 向用户报告
│           输出本轮摘要：
│           "本轮发现 N 个新岗位，已记录。"
│           列出每个新岗位的一行摘要
│
├── [环节 11] 持续优化（用户随时触发）
│     · 更新画像：直接编辑 data/job-profile.md
│     · 添加排除规则：更新 data/meta.json 的 exclude 字段
│     · 调整搜索策略：更新 prompts/search-strategy.md
│
├── [环节 12] 停止 — /job-hunter stop
│     CronDelete 取消定时任务
│     数据文件保留，下次 /job-hunter start 可恢复
│
└── [环节 13] 查看状态 — /job-hunter status
      显示：运行中/已停止、已记录岗位总数、上次运行时间
```

---

## 三、文件结构

```
job-hunter-skill/
├── SKILL.md                        # 主入口，完整流程定义
├── docs/
│   └── PRD.md                      # 本文档（产品需求文档）
├── prompts/
│   ├── intake.md                   # 新用户引导问答模板
│   ├── search-strategy.md          # 搜索策略（关键词组合、来源）
│   ├── filter.md                   # 筛选规则定义
│   ├── scorer.md                   # 评分排序规则定义
│   └── notifications.md            # 模板化提示（启动/停止/警告）
├── data/                           # 运行时用户数据（gitignored）
│   ├── .gitkeep
│   ├── job-profile.md              # 用户求职画像
│   ├── job-candidates.md           # 候选岗位列表（持续追加）
│   └── meta.json                   # 运行元数据
├── .gitignore
├── LICENSE                         # MIT
├── INSTALL.md                      # 安装说明
└── README.md                       # 项目介绍
```

---

## 四、文件与环节对照

| 文件 | 创建时机 | 读取环节 | 更新时机 |
|------|---------|---------|---------|
| `SKILL.md` | 安装时 | 环节 0（入口） | — |
| `prompts/intake.md` | 安装时 | 环节 1（新用户引导） | — |
| `prompts/search-strategy.md` | 安装时 | 环节 5（搜索策略） | 环节 11（用户优化） |
| `prompts/filter.md` | 安装时 | 环节 7（筛选规则） | 环节 11（用户优化） |
| `prompts/scorer.md` | 安装时 | 环节 8（评分规则） | 环节 11（用户优化） |
| `prompts/notifications.md` | 安装时 | 环节 4/10/12/13（模板化提示） | — |
| `data/job-profile.md` | 环节 2（新用户） | 环节 4/7/8 | 环节 11（用户修改） |
| `data/job-candidates.md` | 环节 3（新用户） | 环节 7（去重参考） | 环节 9（追加新岗位） |
| `data/meta.json` | 环节 3（新用户） | 环节 3'/7（去重列表） | 环节 9/11 |

---

## 五、数据文件格式

### 5.1 data/job-profile.md — 求职画像

```markdown
# 求职画像

## 基本信息
- 城市：广州
- 优先区域：天河区、海珠区
- 排除区域：（无）
- 工作年限：5年

## 薪资期望
- 最低：15K
- 期望：18K
- 最高：22K

## 岗位方向
- 全栈开发
- 后端开发
- AI 应用开发

## 技术栈
- Java, Go, Python
- Vue
- MySQL, Redis
- Docker

## 休息制度
- 最低：单双休
- 期望：双休

## 公司偏好
- 类型优先级：外企 > 中大型企业（100人+）> 创业公司
- 最低规模：50人（以开发为主体）

## 搜索关键词
- 全栈开发 广州
- AI开发 广州
- Go后端 广州
- Python 广州
```

### 5.2 data/job-candidates.md — 候选岗位

按日期分组，每个岗位包含完整信息：

```markdown
# 候选岗位列表

---

## 2026-04-05 发现

### #1 全栈工程师(AI Agent/全球化产品) — 广州市天河区星辰科技
- 薪资：15-23K
- 地点：广州·天河区·珠江新城（支持远程）
- 发布日期：2026-04-03
- 经验要求：3-5年
- 技能：全栈无侧重, Vue, Python, 全栈项目经验
- 匹配度：★★★★☆（82分）
- BOSS：王智胜 · 总经理
- 链接：https://www.zhipin.com/job_detail/xxx
- security_id：swf0JZEvV1_0...

### #2 Python工程师（AI Agent开发）— 四一技术
- 薪资：15-30K
- 地点：广州·海珠区·琶洲
- 发布日期：2026-04-04
- 经验要求：5-10年
- 技能：Scrapy, AI Agent, Docker, Django, Python, Flask
- 匹配度：★★★★☆（78分）
- BOSS：卓先生 · 负责人
- 链接：https://www.zhipin.com/job_detail/xxx
- security_id：d3DiIohaNak...

---

## 2026-04-04 发现

### #3 ...
```

### 5.3 data/meta.json — 运行元数据

```json
{
  "created_at": "2026-04-05T10:00:00Z",
  "updated_at": "2026-04-05T20:30:00Z",
  "last_run_time": "2026-04-05T20:30:00Z",
  "cron_job_id": "job-hunter-search",
  "run_count": 21,
  "total_candidates": 15,
  "next_candidate_number": 16,
  "search_interval_minutes": 30,
  "recorded_job_ids": [
    "28d2dcad70c1b7de0nZ-29u9E1JZ",
    "5047ee0d92fb5e440nR92dm4FldS"
  ],
  "exclude_keywords": ["测试", "QA", "实习"],
  "search_keywords": [
    "全栈开发 广州",
    "AI开发 广州",
    "Go后端 广州",
    "Python 广州"
  ]
}
```

---

## 六、筛选规则（prompts/filter.md 概要）

按优先级从高到低执行硬条件过滤：

| 维度 | 规则 | 处理方式 |
|------|------|---------|
| ID 去重 | security_id 在 meta.json.recorded_job_ids 中 | 直接排除 |
| 排除关键词 | 岗位名/描述含 meta.json.exclude_keywords | 直接排除 |
| 城市区域 | 不在偏好区域，或在排除区域 | 直接排除 |
| 薪资下限 | 低于 job-profile 最低薪资 | 直接排除 |
| 休息制度 | 明确标注"单休"且用户不接受 | 直接排除 |
| 公司规模 | 低于最低要求（非开发主体公司） | 降低评分 |

筛选后的岗位进入评分环节。

---

## 七、评分规则（prompts/scorer.md 概要）

综合评分（满分 100 分）：

| 维度 | 权重 | 评分依据 |
|------|------|---------|
| **发布日期** | **30%** | 1天内=满分，每多1天扣分，超过14天大幅扣分 |
| 薪资匹配 | 25% | 与期望薪资范围的吻合程度 |
| 技术栈匹配 | 20% | 画像技术栈与岗位要求的重叠度 |
| 地点匹配 | 15% | 是否在优先区域，有具体地址加分 |
| 公司规模 | 10% | 符合偏好类型加分 |

**发布日期衰减规则**（30分维度）：
- 1天内发布：30分
- 2天：27分
- 3天：24分
- 5天：18分
- 7天：14分
- 10天：8分
- 14天：4分
- 超过14天：0分（建议排除）

星级对应：
- 90-100分：★★★★★
- 75-89分：★★★★☆
- 60-74分：★★★☆☆
- 40-59分：★★☆☆☆
- 40分以下：★☆☆☆☆

---

## 八、SKILL.md frontmatter

```yaml
---
name: job-hunter
description: "自动从 Boss 直聘发现、筛选、记录合适岗位。使用 /job-hunter 启动定时求职助手。"
---
```

仅使用 `name` 和 `description` 两个官方字段。

---

## 九、opencli 依赖

本 Skill 依赖 [opencli](https://github.com/jackwener/opencli) 提供以下命令：

| 命令 | 用途 | 环节 |
|------|------|------|
| `opencli boss search` | 搜索职位 | 环节 5 |
| `opencli boss recommend` | 获取推荐岗位 | 环节 5 |
| `opencli boss detail` | 获取岗位详情 | 环节 6 |

**前置条件**：
- opencli >= 1.5.0 已安装
- Browser Bridge Chrome 扩展已安装并连接
- Chrome 已登录 Boss 直聘（zhipin.com）

---

## 十、Claude Code 依赖

本 Skill 使用以下 Claude Code 原生能力：

| 能力 | 用途 | 环节 |
|------|------|------|
| `CronCreate` | 创建定时任务 | 环节 4 |
| `CronDelete` | 停止定时任务 | 环节 12 |
| `AskUserQuestion` | 交互式引导录入 | 环节 1 |
| `Read` / `Write` / `Edit` | 读写数据文件 | 全流程 |
| `Bash` | 执行 opencli 命令 | 环节 5/6 |

---

## 十一、约束与边界

- 定时任务仅在 Claude Code 运行期间执行（CronCreate 限制）
- durable 任务最大持续 7 天，到期需重新启动
- Boss 直聘搜索频率不宜过短，避免触发风控
- 每次搜索结果上限约 30 条（opencli 限制）
- 岗位详情依赖 Boss 页面结构，结构变化可能导致详情获取失败
- 发布日期依赖 Boss 页面展示，部分岗位可能无发布日期

### 搜索频率安全阈值

| 间隔 | 风险等级 | 说明 |
|------|---------|------|
| >= 30 分钟 | 安全 | 推荐默认值 |
| 15-29 分钟 | 低风险 | 可接受，但需提醒用户 |
| 10-14 分钟 | 中风险 | 触发警告，需用户二次确认 |
| < 10 分钟 | 高风险 | 拒绝设置，强制最低 10 分钟 |

用户设置间隔 < 30 分钟时，输出警告提示（见「十三、模板化提示」）。

### 环境检测清单

新用户首次运行时，自动检测以下环境：

| 检测项 | 检测方式 | 缺失处理 |
|--------|---------|---------|
| Node.js >= 20 | `node --version` | 提示安装，不自动执行 |
| opencli 已安装 | `opencli --version` | 询问是否安装（需用户同意） |
| Browser Bridge 连接 | `opencli doctor` | 提示安装 Chrome 扩展（人工操作） |
| Chrome 已登录 Boss 直聘 | `opencli boss search "测试" --limit 1` | 提示用户在 Chrome 中登录 zhipin.com |

**原则**：
- 不得在未经用户同意的情况下自动安装任何软件
- 需要人工操作的步骤（如安装 Chrome 扩展、登录网站），给出清晰的步骤指引
- 检测结果逐项展示，让用户了解当前状态

---

## 十二、路线图

### P0 — MVP
- [ ] SKILL.md 主流程
- [ ] prompts/intake.md 新用户引导
- [ ] prompts/search-strategy.md 搜索策略
- [ ] prompts/filter.md 筛选规则
- [ ] prompts/scorer.md 评分规则
- [ ] prompts/notifications.md 模板化提示
- [ ] 环境检测（opencli、Browser Bridge、Boss 登录）
- [ ] 定时任务创建/停止/状态查询
- [ ] 搜索频率配置 + 安全警告
- [ ] 岗位搜索 → 详情获取 → 筛选 → 评分 → 记录 完整闭环

### P1 — 体验优化
- [ ] 支持多平台（LinkedIn、V2EX 招聘等）
- [ ] 搜索关键词自动优化（根据历史命中率调整权重）
- [ ] 岗位变更追踪（已记录岗位状态变化检测）

### P2 — 高级功能
- [ ] 通勤时间计算（集成地图 API，可选功能）
- [ ] 自动打招呼（对高匹配度岗位自动发送问候）
- [ ] 数据可视化（生成 HTML 岗位看板）
- [ ] 面试准备辅助（根据岗位要求生成复习清单）

---

## 十三、模板化提示

所有关键节点的提示语必须模板化，存储在 `prompts/notifications.md`，确保每次输出质量一致、不遗漏。

### 13.1 定时任务启动成功

```
Job Hunter 已启动！

运行配置：
  - 搜索间隔：每 {interval} 分钟
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

### 13.2 搜索频率警告

当用户设定间隔 < 30 分钟时触发：

```
搜索频率警告：当前设定的间隔为 {interval} 分钟

风险等级：{risk_level}
  - 频繁请求 Boss 直聘可能触发平台风控
  - 可能导致账号被限制访问
  - 建议间隔不低于 30 分钟

是否确认使用 {interval} 分钟间隔？（y/n）
```

### 13.3 任务即将到期（7天限制）

在任务运行接近 7 天时（如第 6 天），在某一轮搜索报告后追加：

```
Job Hunter 提醒：定时任务即将在约 {remaining_hours} 小时后到期（durable 任务最长 7 天）

到期后不会丢失数据，但搜索将停止。
如需继续，请在到期后运行 /job-hunter start 重新启动。
```

### 13.4 环境检测未通过

```
环境检测未通过，以下问题需要解决：

  [{status}] opencli .............. {result}
  [{status}] Browser Bridge ....... {result}
  [{status}] Boss 直聘登录 ........ {result}

{针对每项缺失的具体指引}

请解决以上问题后重新运行 /job-hunter
```

### 13.5 每轮搜索报告

```
Job Hunter 本轮报告（{date}）

发现 {new_count} 个新匹配岗位，已记录到 data/job-candidates.md

{对每个新岗位的一行摘要，按匹配度排序}
  #N  岗位名 @ 公司 | 薪资 | 地点 | 发布于{date} | 匹配{score}分

累计已记录 {total_count} 个候选岗位。
```

---

## 十四、搜索间隔配置

`data/meta.json` 中的 `search_interval_minutes` 字段控制搜索频率：

```json
{
  "search_interval_minutes": 30
}
```

**规则**：
- 默认值：30 分钟
- 用户可在初始化时设定，也可随时通过编辑 meta.json 修改
- 修改后需重新启动定时任务（`/job-hunter stop` → `/job-hunter start`）才生效
- 最低允许值：10 分钟（硬限制，低于此值拒绝设置）
- 10-14 分钟：高风险警告，需用户二次确认
- 15-29 分钟：低风险提示
- >= 30 分钟：安全，无需额外提示
