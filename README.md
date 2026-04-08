# Job Hunter Skill

> 让 AI 帮你刷 Boss，你只管挑心仪的岗位。

![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg) ![Claude Code](https://img.shields.io/badge/Claude%20Code-Skill-blueviolet) ![AgentSkills](https://img.shields.io/badge/AgentSkills-Standard-green)

自动从 Boss 直聘发现、筛选、记录匹配岗位的 Claude Code Skill。
建立求职画像后，每 30 分钟自动搜索新岗位，智能评分排序，持续追加到候选清单。

[前置准备](#前置准备) · [快速开始](#快速开始) · [命令](#命令) · [工作流程](#工作流程)

---

## 亮点

- **全自动搜索** — 搜索、详情获取、筛选、评分、记录，每 30 分钟自动运行一轮
- **智能评分排序** — 6 维加权评分（发布日期、薪资、技术栈、地点、HR 活跃度、公司规模），按匹配度排序
- **三分类管理** — 人工审核后将候选岗位分为练手、目标、排除三档，独立归档
- **新用户开箱即用** — 交互式引导建立求职画像，无需手动配置任何文件
- **环境自动检测** — 首次运行自动检查 opencli、Browser Bridge、Boss 登录状态，缺失项给出安装指引
- **搜索频率可配置** — 支持自定义搜索间隔，低于安全阈值时自动警告，防止触发平台风控
- **持续优化** — 画像、排除规则、搜索策略均可随时调整，越用越精准

---

## 前置准备

| 依赖 | 说明 | 安装方式 |
|------|------|---------|
| Claude Code | AI 编码助手 | [官方安装指南](https://docs.anthropic.com/en/docs/claude-code) |
| Node.js >= 20 | JavaScript 运行时 | [nodejs.org](https://nodejs.org) |
| opencli | Boss 直聘 CLI 工具 | `npm install -g @jackwener/opencli` |
| Chrome + Browser Bridge | 浏览器自动化 | 详见 [INSTALL.md](INSTALL.md#3-browser-bridge-chrome-扩展安装) |

> 安装完 opencli 后，运行 `opencli doctor` 验证连接状态。首次运行 `/job-hunter` 也会自动检测环境。

---

## 快速开始

```bash
# 1. 安装 Skill（全局安装，所有项目可用）
git clone https://github.com/yk-ken/job-hunter-skill ~/.claude/skills/job-hunter

# 2. 创建你的工作目录（画像、岗位等数据保存在这里）
mkdir ~/job-search && cd ~/job-search

# 3. 在此目录启动 Claude Code，运行
/job-hunter
```

首次运行会自动检测环境并引导你建立求职画像（城市、薪资、技术栈等），完成后立即创建定时搜索任务。

详细的安装步骤和故障排除，请查看 [INSTALL.md](INSTALL.md)。

---

## 命令

| 命令 | 说明 |
|------|------|
| `/job-hunter` 或 `/job-hunter start` | 启动定时搜索任务 |
| `/job-hunter stop` | 停止搜索（数据保留，可随时恢复） |
| `/job-hunter status` | 查看运行状态和各分类统计 |
| `/job-hunter update` | 更新 Skill 到最新版本 |
| `/job-hunter exclude` | 查看排除列表 |
| `/job-hunter practice` | 查看练手列表 |
| `/job-hunter target` | 查看目标列表 |

**自然语言操作**（直接跟 Claude 说即可）：

| 说法 | 说明 |
|------|------|
| 「排除编号X」 | 将岗位从候选列表移除，可反馈优化搜索条件 |
| 「练手编号X」 | 将岗位标记为练手，移入练手归档 |
| 「目标编号X」 | 将岗位标记为心仪目标，移入目标归档 |
| 「标记编号X」 | 询问分类为练手还是目标（先确认岗位再选择分类） |

---

## 工作流程

```
搜索岗位（多关键词 + 推荐）
       |
       v
获取详情（薪资、地址、发布日期、技能要求...）
       |
       v
智能筛选（地点、薪资、休息制度、排除词、去重）
       |
       v
综合评分（发布日期 25% + 薪资 20% + 技术栈 20% + 地点 15% + HR活跃度 10% + 公司规模 10%）
       |
       v
记录到候选清单（按匹配度排序，持续追加）
       |
       v
输出本轮报告（新增岗位摘要 + 累计统计）
       |
       v
人工审核 → 三分类
  ├─ 排除 → job-excluded.csv（可反馈优化搜索条件）
  ├─ 练手 → job-practice.csv（值得面试但非首选）
  └─ 目标 → job-target.csv（各方面都符合预期）
```

每轮循环间隔默认 30 分钟，在 Claude Code 运行期间持续执行。

---

## 文件结构

**Skill 安装目录**（`~/.claude/skills/job-hunter/`）：

```
job-hunter-skill/
├── SKILL.md                 # 主入口，路由分发
├── prompts/                 # 各环节行为模板
│   ├── intake.md            #   新用户引导问答
│   ├── search-strategy.md   #   搜索策略
│   ├── filter.md            #   筛选规则
│   ├── scorer.md            #   评分规则
│   ├── cron-task.md         #   定时任务执行指令
│   ├── notifications.md     #   模板化提示
│   └── operations/          #   按需加载的操作指令
│       ├── start.md         #   启动定时任务
│       ├── stop.md          #   停止定时任务
│       ├── status.md        #   查看状态
│       ├── exclude.md       #   排除岗位
│       ├── view-exclude.md  #   查看排除列表
│       ├── classify.md      #   练手/目标分类
│       ├── view-practice.md #   查看练手列表
│       ├── view-target.md   #   查看目标列表
│       └── update.md        #   更新 Skill
├── docs/
│   ├── PRD.md               #   产品需求文档
│   └── plan.md              #   实施计划
├── INSTALL.md               #   安装说明
├── .gitignore
├── LICENSE                  #   MIT
└── README.md                #   本文件
```

**用户工作目录**（如 `~/job-search/`）：

```
job-search/                  ← 你的工作目录（启动 Claude Code 的位置）
└── data/
    ├── job-profile.md       #   你的求职画像
    ├── job-candidates.csv   #   候选岗位列表（待审核，CSV 格式，可用 Excel 打开）
    ├── job-excluded.csv     #   排除岗位归档
    ├── job-practice.csv     #   练手岗位归档
    ├── job-target.csv       #   目标岗位归档
    └── meta.json            #   运行元数据（去重列表、排除规则、分类计数、定时任务ID）
```

> 模板文件从 skill 安装目录读取，用户数据保存在工作目录。两者分离，互不影响。

---

## Star History

![Star History Chart](https://api.star-history.com/svg?repos=yk-ken/job-hunter-skill&type=Date)

---

## License

[MIT](LICENSE)
