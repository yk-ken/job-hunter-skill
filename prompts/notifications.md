# 模板化提示

所有关键节点的提示语定义在此，确保每次输出质量一致、不遗漏。

---

## 1. 定时任务启动成功

**触发环节**：环节 4（CronCreate 成功后）

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

**变量说明**：

| 变量 | 来源 | 示例值 |
|------|------|--------|
| `{interval}` | data/meta.json → search_interval_minutes | `30` |
| `{keywords_count}` | data/meta.json → search_keywords 的长度 | `4` |

---

## 2. 搜索频率警告

**触发条件**：用户设定 search_interval_minutes < 30 分钟时触发

```
搜索频率警告：当前设定的间隔为 {interval} 分钟

风险等级：{risk_level}
  - 频繁请求 Boss 直聘可能触发平台风控
  - 可能导致账号被限制访问
  - 建议间隔不低于 30 分钟

是否确认使用 {interval} 分钟间隔？（y/n）
```

**变量说明**：

| 变量 | 来源 | 示例值 |
|------|------|--------|
| `{interval}` | 用户设定的 search_interval_minutes | `20` |
| `{risk_level}` | 根据间隔映射（见下表） | `低风险（15-29分钟）` |

**风险等级映射**：

| 间隔范围 | 风险等级 | 处理方式 |
|----------|---------|---------|
| 15 - 29 分钟 | `低风险（15-29分钟）` | 输出警告，询问确认 |
| 10 - 14 分钟 | `中风险（10-14分钟）` | 输出警告，需用户二次确认 |
| < 10 分钟 | `高风险（<10分钟）` | 拒绝设置，强制最低 10 分钟，不使用此模板 |

**注意**：当间隔 < 10 分钟时，不使用此模板，直接拒绝并提示：

```
搜索间隔不能低于 10 分钟，这是为了避免触发 Boss 直聘风控。
已将间隔设置为最低安全值 10 分钟。
如需调整，请编辑 data/meta.json 中的 search_interval_minutes 字段。
```

---

## 3. 任务即将到期

**触发条件**：定时任务运行接近 7 天时（如第 6 天），在某一轮搜索报告后追加

```
Job Hunter 提醒：定时任务即将在约 {remaining_hours} 小时后到期（durable 任务最长 7 天）

到期后不会丢失数据，但搜索将停止。
如需继续，请在到期后运行 /job-hunter start 重新启动。
```

**变量说明**：

| 变量 | 来源 | 示例值 |
|------|------|--------|
| `{remaining_hours}` | 当前时间 - data/meta.json → created_at 计算剩余小时数 | `18` |

**计算方式**：
```
remaining_hours = floor((created_at + 7天 - now) / 小时)
```

当 `remaining_hours <= 24` 且 `remaining_hours > 0` 时触发此提示。

---

## 4. 环境检测未通过

**触发环节**：环节 0（首次运行环境检测时，任一检测项未通过）

```
环境检测未通过，以下问题需要解决：

  [{status_node}] Node.js .............. {result_node}
  [{status_opencli}] opencli .............. {result_opencli}
  [{status_bridge}] Browser Bridge ....... {result_bridge}
  [{status_boss}] Boss 直聘登录 ........ {result_boss}

说明：[SKIP] 表示该项的前置依赖缺失，已跳过检测。

{installation_guides}

请解决以上问题后重新运行 /job-hunter
```

**变量说明**：

| 变量 | 来源 | 可能值 |
|------|------|--------|
| `{status_node}` | `node --version` 执行结果 | `✅` 或 `❌` |
| `{result_node}` | 版本号或错误信息 | `已安装 (v20.x.x)` / `未安装或版本低于 20` |
| `{status_opencli}` | `opencli --version` 执行结果 | `✅`、`❌` 或 `[SKIP]`（Node.js 缺失时跳过） |
| `{result_opencli}` | 版本号或错误信息 | `已安装 (v1.5.0)` / `未安装` / `跳过 — 前置依赖 Node.js 缺失` |
| `{status_bridge}` | `opencli doctor` 执行结果 | `✅`、`❌` 或 `[SKIP]`（opencli 缺失时跳过） |
| `{result_bridge}` | 连接状态 | `已连接` / `未连接` / `跳过 — 前置依赖 opencli 缺失` |
| `{status_boss}` | `opencli boss search "测试" --limit 1` 执行结果 | `✅`、`❌` 或 `[SKIP]`（Browser Bridge 缺失时跳过） |
| `{result_boss}` | 登录状态 | `已登录` / `未登录` / `跳过 — 前置依赖 Browser Bridge 缺失` |
| `{installation_guides}` | 仅包含未通过项的安装指引（见下方） | — |

**逐项安装指引（仅未通过项展示）**：

当 Node.js 未安装或版本低于 20 时，追加：
```
【安装 Node.js】
  需要 Node.js >= 20，推荐使用 nvm 安装：
    nvm install --lts
    nvm use --lts
  或直接从 https://nodejs.org 下载 LTS 版本。
  安装后运行 node --version 验证版本 >= 20。
```

当 opencli 未安装时，追加：
```
【安装 opencli】
  推荐使用 npm 全局安装：
    npm install -g opencli
  安装后运行 opencli --version 验证。
```

当 Browser Bridge 未连接时，追加：
```
【安装 Browser Bridge Chrome 扩展】
  1. 在 Chrome 浏览器中访问 Chrome 网上应用店，搜索 "Browser Bridge"
  2. 点击"添加到 Chrome"安装扩展
  3. 安装完成后，点击扩展图标，按照提示完成初始化配置
  4. 确认扩展状态为"已连接"
  5. 运行 opencli doctor 验证连接状态
```

当 Boss 直聘未登录时，追加：
```
【登录 Boss 直聘】
  1. 在 Chrome 浏览器中打开 https://www.zhipin.com
  2. 使用手机扫码或其他方式登录
  3. 登录成功后，确认页面显示已登录状态
  4. 重新运行 opencli boss search "测试" --limit 1 验证
```

**完整示例（全部未通过）**：

```
环境检测未通过，以下问题需要解决：

  [❌] opencli .............. 未安装
  [❌] Browser Bridge ....... 未连接
  [❌] Boss 直聘登录 ........ 未登录

【安装 opencli】
  推荐使用 npm 全局安装：
    npm install -g opencli
  安装后运行 opencli --version 验证。

【安装 Browser Bridge Chrome 扩展】
  1. 在 Chrome 浏览器中访问 Chrome 网上应用店，搜索 "Browser Bridge"
  2. 点击"添加到 Chrome"安装扩展
  3. 安装完成后，点击扩展图标，按照提示完成初始化配置
  4. 确认扩展状态为"已连接"
  5. 运行 opencli doctor 验证连接状态

【登录 Boss 直聘】
  1. 在 Chrome 浏览器中打开 https://www.zhipin.com
  2. 使用手机扫码或其他方式登录
  3. 登录成功后，确认页面显示已登录状态
  4. 重新运行 opencli boss search "测试" --limit 1 验证

请解决以上问题后重新运行 /job-hunter
```

**部分通过示例**：

```
环境检测未通过，以下问题需要解决：

  [✅] opencli .............. 已安装 (v1.5.0)
  [❌] Browser Bridge ....... 未连接
  [✅] Boss 直聘登录 ........ 已登录

【安装 Browser Bridge Chrome 扩展】
  1. 在 Chrome 浏览器中访问 Chrome 网上应用店，搜索 "Browser Bridge"
  2. 点击"添加到 Chrome"安装扩展
  3. 安装完成后，点击扩展图标，按照提示完成初始化配置
  4. 确认扩展状态为"已连接"
  5. 运行 opencli doctor 验证连接状态

请解决以上问题后重新运行 /job-hunter
```

---

## 5. 每轮搜索报告

**触发环节**：环节 10（每轮搜索完成后）

### 5a. 有新岗位时

```
Job Hunter 本轮报告（{date}）

发现 {new_count} 个新匹配岗位，已记录到 data/job-candidates.md

{job_summaries}

累计已记录 {total_count} 个候选岗位。
```

**变量说明**：

| 变量 | 来源 | 示例值 |
|------|------|--------|
| `{date}` | 当前日期 | `2026-04-05` |
| `{new_count}` | 本轮新增岗位数量 | `3` |
| `{job_summaries}` | 每个新岗位的一行摘要（按匹配度降序） | 见下方格式 |
| `{total_count}` | data/meta.json → total_candidates | `15` |

**岗位摘要格式**（每个岗位一行，按匹配分降序排列）：

```
  #{number}  {job_name} @ {company} | {salary} | {location} | 发布于 {post_date} | 匹配 {score}分
```

**示例输出**：

```
Job Hunter 本轮报告（2026-04-05）

发现 3 个新匹配岗位，已记录到 data/job-candidates.md

  #12  全栈工程师(AI Agent) @ 星辰科技 | 15-23K | 广州·天河区 | 发布于 2026-04-03 | 匹配 82分
  #13  Python工程师（AI Agent开发）@ 四一技术 | 15-30K | 广州·海珠区 | 发布于 2026-04-04 | 匹配 78分
  #14  Go后端开发工程师 @ 云端网络 | 18-25K | 广州·天河区 | 发布于 2026-04-04 | 匹配 65分

累计已记录 14 个候选岗位。
```

### 5b. 无新岗位时

```
Job Hunter 本轮报告（{date}）

本轮未发现新的匹配岗位。

已扫描 {searched_count} 个岗位，均已在之前记录或不符合筛选条件。

累计已记录 {total_count} 个候选岗位。
```

**变量说明**：

| 变量 | 来源 | 示例值 |
|------|------|--------|
| `{date}` | 当前日期 | `2026-04-05` |
| `{searched_count}` | 本轮搜索返回的岗位总数（去重前） | `45` |
| `{total_count}` | data/meta.json → total_candidates | `15` |

**示例输出**：

```
Job Hunter 本轮报告（2026-04-05）

本轮未发现新的匹配岗位。

已扫描 45 个岗位，均已在之前记录或不符合筛选条件。

累计已记录 15 个候选岗位。
```
