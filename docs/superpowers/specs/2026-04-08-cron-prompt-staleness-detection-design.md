# 定时任务 Prompt 变更检测

日期：2026-04-08

## 背景

Job Hunter 的定时任务通过 CronCreate 创建，创建时 prompt 文本被快照到任务中。Skill 更新后，已创建的定时任务仍使用旧 prompt 执行，行为与最新版本不一致。需要一种机制在 `/job-hunter start` 时自动检测 prompt 是否已变更，并提示用户重建。

## 方案

在创建定时任务时保存 prompt 快照文件，启动时与当前 prompt 对比检测变更。

### 改动范围

| 文件 | 变更内容 |
|------|---------|
| `prompts/operations/start.md` | A1 步骤加入 prompt 比对逻辑；A5 步骤加入快照写入 |
| `data/cron-prompt-snapshot.md` | 新增，存储创建时使用的 prompt 文本 |

其他文件（`SKILL.md`、`prompts/cron-task.md`、其他 operations 文件）无需改动。

### start.md A1：变更检测

当 `cron_job_id` 不为 null 时，执行以下逻辑：

1. 读取 `${CLAUDE_SKILL_DIR}/prompts/cron-task.md`
2. 提取其中 code block 内的内容（排除前后的说明文字和「岗位记录格式参考」等非 prompt 部分）
3. 将 `${CLAUDE_SKILL_DIR}` 替换为 skill 实际安装路径（与创建时处理一致）
4. 读取 `data/cron-prompt-snapshot.md`
   - 如果文件不存在（定时任务在此功能上线前创建），视为「不一致」
5. 逐字对比两段文本

结果分支：

- **一致** → 现有行为："Job Hunter 已在运行中（任务 ID：{cron_job_id}）。如需重启，请先运行 /job-hunter stop，再运行 /job-hunter start。"
- **不一致**（包括快照文件不存在的情况）→ 提示用户：
  ```
  Skill 已更新，当前定时任务的执行逻辑与最新版本不一致。
  建议重建定时任务以使用最新逻辑。是否重建？
  ```
  - 用户确认 → 自动执行 stop 操作，然后继续 start 的后续步骤（A2-A6），整个过程无需用户再次干预
  - 用户拒绝 → 保持旧任务继续运行，不更新快照，结束

### start.md A5：快照写入

在 CronCreate 创建成功后、更新 meta.json 时，增加一步：

将实际传给 CronCreate 的 prompt 文本（已提取、已替换路径的完整内容）写入 `data/cron-prompt-snapshot.md`。

### prompt 提取规则

`cron-task.md` 的结构为：

```
# 定时任务执行指令         ← 标题（非 prompt）
说明文字...               ← 说明（非 prompt）
---                       ← 分隔线（非 prompt）
```                       ← code block 开始
你是 Job Hunter ...       ← 实际 prompt 开始
...9 个步骤...
```                       ← code block 结束（实际 prompt 结束）
---                       ← 分隔线（非 prompt）
## 岗位记录格式参考        ← 参考信息（非 prompt）
...
```

提取规则：只取第一个 `` ``` `` 到第二个 `` ``` `` 之间的内容（不含 ``` 标记本身）。

### 为什么不使用哈希

- `cron-task.md` 包含非 prompt 内容（标题、说明、参考信息），哈希整个文件会因非 prompt 部分的修改产生误判
- 只对 prompt 部分做哈希需要额外工具（sha256sum / Node.js crypto），引入不必要的外部依赖
- Claude 自身具备文本比对能力，直接对比两段文本即可，无需引入中间工具

## 不涉及的内容

- 不修改 `SKILL.md` 主入口
- 不修改 `prompts/cron-task.md`
- 不修改 stop / status / exclude / classify 等其他操作文件
- 不在 meta.json 中新增字段
