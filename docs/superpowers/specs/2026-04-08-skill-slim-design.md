# 设计文档：SKILL.md 瘦身为路由器

日期：2026-04-08

## 背景

SKILL.md 当前约 620 行，每次触发 `/job-hunter` 时全量加载到上下文。但实际每次只执行其中一个操作（如 status、exclude、classify 等），大量指令内容是冗余的。特别是定时任务执行指令独占 ~200 行，但仅在 `/job-hunter start` 创建任务时使用一次。

需要将 SKILL.md 瘦身为路由器，操作详情按需加载，减少上下文占用。

## 核心设计

### SKILL.md 保留内容（~120 行）

- 触发条件表
- 步骤 0：工作目录检查
- 步骤 1-1.5：新用户判断 + 升级检查
- 步骤 2：新用户引导（委托给 intake.md，已有）
- 步骤 3：读取运行状态
- 步骤 4：路由分发（按参数读取对应操作文件）

步骤 4 的路由逻辑：

| 参数 | 读取文件 |
|------|---------|
| start / 无参数 | `prompts/operations/start.md` |
| stop | `prompts/operations/stop.md` |
| status | `prompts/operations/status.md` |
| exclude | `prompts/operations/view-exclude.md` |
| practice | `prompts/operations/view-practice.md` |
| target | `prompts/operations/view-target.md` |
| update | `prompts/operations/update.md` |
| 用户说「排除编号X」 | `prompts/operations/exclude.md` |
| 用户说「练手/目标/标记 编号X」 | `prompts/operations/classify.md` |

所有路径使用 `${CLAUDE_SKILL_DIR}/prompts/operations/` 前缀。

### 新增文件

```
prompts/
├── operations/
│   ├── start.md          ← 操作 A：启动定时任务（~40 行）
│   ├── stop.md           ← 操作 B：停止定时任务（~20 行）
│   ├── status.md         ← 操作 C：查看运行状态（~30 行）
│   ├── exclude.md        ← 操作 D：排除岗位（~80 行）
│   ├── view-exclude.md   ← 操作 E：查看排除列表（~15 行）
│   ├── classify.md       ← 操作 F：岗位分类（~80 行）
│   ├── view-practice.md  ← 操作 G：查看练手列表（~15 行）
│   ├── view-target.md    ← 操作 H：查看目标列表（~15 行）
│   └── update.md         ← 操作 I：更新 Skill（~15 行）
└── cron-task.md          ← 定时任务执行指令 + 格式参考（~225 行）
```

每个操作文件是独立完整的指令，包含它所需的所有上下文信息。

### 从 SKILL.md 删除的内容

- 操作 A-I 的全部详细流程
- 定时任务执行指令的全部内容（迁移到 `prompts/cron-task.md`）
- 岗位记录格式参考（迁移到 `prompts/cron-task.md`，仅定时任务写入 CSV 时需要）

### 不修改的文件

- `prompts/intake.md`
- `prompts/filter.md`
- `prompts/scorer.md`
- `prompts/notifications.md`
- `prompts/search-strategy.md`

## 效果

- SKILL.md：~620 行 → ~120 行（减少 ~80%）
- 每次触发上下文占用：~620 行 → ~120 行 + 当前操作的文件（15-225 行）
- 最大节省场景：`/job-hunter status` 只需 ~150 行（原来 620 行）
