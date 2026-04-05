# Job Hunter Skill

> 让 AI 帮你刷 Boss，你只管挑心仪的岗位。

自动从 Boss 直聘发现、筛选、记录匹配岗位的 Claude Code Skill。
建立求职画像后，每 30 分钟自动搜索新岗位，智能评分排序，持续追加到候选清单。

[快速开始](#快速开始) · [命令](#命令) · [工作流程](#工作流程) · [安装](#安装)

---

## 亮点

- **全自动搜索** — 搜索、详情获取、筛选、评分、记录，每 30 分钟自动运行一轮
- **智能评分排序** — 发布日期权重 30%（越新分越高），结合薪资、技术栈、地点、公司规模综合打分
- **新用户开箱即用** — 交互式引导建立求职画像，无需手动配置任何文件
- **环境自动检测** — 首次运行自动检查 opencli、Browser Bridge、Boss 登录状态，缺失项给出安装指引
- **搜索频率可配置** — 支持自定义搜索间隔，低于安全阈值时自动警告，防止触发平台风控
- **持续优化** — 画像、排除规则、搜索策略均可随时调整，越用越精准

---

## 快速开始

```bash
# 1. 安装 Skill
git clone https://github.com/yk-ken/job-hunter-skill ~/.claude/skills/job-hunter

# 2. 在 Claude Code 中运行
/job-hunter

# 3. 按提示完成画像 → 自动开始搜索
```

首次运行会自动检测环境并引导你建立求职画像（城市、薪资、技术栈等），完成后立即创建定时搜索任务。

---

## 命令

| 命令 | 说明 |
|------|------|
| `/job-hunter` 或 `/job-hunter start` | 启动定时搜索任务 |
| `/job-hunter stop` | 停止搜索（数据保留，可随时恢复） |
| `/job-hunter status` | 查看运行状态和已记录岗位数 |

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
综合评分（发布日期 30% + 薪资 25% + 技术栈 20% + 地点 15% + 公司规模 10%）
       |
       v
记录到候选清单（按匹配度排序，持续追加）
       |
       v
输出本轮报告（新增岗位摘要 + 累计统计）
```

每轮循环间隔默认 30 分钟，在 Claude Code 运行期间持续执行。

---

## 安装

### 前置准备

| 依赖 | 说明 | 安装方式 |
|------|------|---------|
| Claude Code | AI 编码助手 | [官方安装指南](https://docs.anthropic.com/en/docs/claude-code) |
| Node.js >= 20 | JavaScript 运行时 | [nodejs.org](https://nodejs.org) |
| opencli | 网站 CLI 工具 | `npm install -g @anthropic-ai/opencli` |
| Chrome + Browser Bridge | 浏览器自动化 | [详细安装指引](INSTALL.md#browser-bridge-扩展安装) |

> 首次运行 `/job-hunter` 会自动检测以上环境，缺失项会给出安装指引。

### 安装 Skill

```bash
# 方式一：全局安装（推荐，所有项目可用）
git clone https://github.com/yk-ken/job-hunter-skill ~/.claude/skills/job-hunter

# 方式二：安装到当前项目
mkdir -p .claude/skills
git clone https://github.com/yk-ken/job-hunter-skill .claude/skills/job-hunter
```

### 验证安装

在 Claude Code 中输入 `/job-hunter`，首次运行会自动进入环境检测和新用户引导。

详细的安装步骤和故障排除，请查看 [INSTALL.md](INSTALL.md)。

---

## 文件结构

```
job-hunter-skill/
├── SKILL.md                 # 主入口，完整流程定义
├── prompts/                 # 各环节行为模板
│   ├── intake.md            #   新用户引导问答
│   ├── search-strategy.md   #   搜索策略（关键词组合、来源）
│   ├── filter.md            #   筛选规则（硬条件过滤）
│   ├── scorer.md            #   评分规则（多维加权打分）
│   └── notifications.md     #   模板化提示（启动/停止/警告/报告）
├── data/                    # 运行时用户数据（gitignored）
│   ├── job-profile.md       #   求职画像
│   ├── job-candidates.md    #   候选岗位列表（持续追加）
│   └── meta.json            #   运行元数据（去重列表、排除规则）
├── docs/
│   ├── PRD.md               #   产品需求文档
│   └── plan.md              #   实施计划
├── INSTALL.md               #   安装说明
├── .gitignore
├── LICENSE                  #   MIT
└── README.md                #   本文件
```

---

## Star History

[![Star History Chart](https://api.star-history.com/svg?repos=yk-ken/job-hunter-skill&type=Date)](https://star-history.com/#yk-ken/job-hunter-skill&Date)

---

## License

[MIT](LICENSE)
