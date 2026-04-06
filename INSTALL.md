# Job Hunter Skill 安装指南

---

## 1. 前置条件

在安装 job-hunter-skill 之前，请确保以下环境已就绪：

| 依赖 | 版本要求 | 说明 |
|------|---------|------|
| Claude Code | 最新版 | [安装指引](https://docs.anthropic.com/en/docs/claude-code) |
| Node.js | >= 20 | `node --version` 检查 |
| Chrome 浏览器 | 最新版 | 用于 Browser Bridge 扩展 |
| opencli | >= 1.5.0 | 通过 npm 全局安装 |

安装 opencli：

```bash
npm install -g @jackwener/opencli
```

安装后验证：

```bash
opencli --version
```

---

## 2. 安装 Skill

### 方式一：安装到当前项目（推荐）

在你的 git 仓库根目录下执行：

```bash
mkdir -p .claude/skills
git clone https://github.com/yk-ken/job-hunter-skill .claude/skills/job-hunter
```

安装后，Skill 仅对该项目生效。

### 方式二：全局安装

所有项目均可使用：

```bash
git clone https://github.com/yk-ken/job-hunter-skill ~/.claude/skills/job-hunter
```

---

## 3. Browser Bridge Chrome 扩展安装

opencli 通过 Browser Bridge 与 Chrome 通信，需要安装对应的 Chrome 扩展。

### 步骤

1. 从 [opencli Releases](https://github.com/jackwener/opencli/releases) 下载 `opencli-extension.zip`

2. 将 zip 文件解压到任意目录（记住这个路径）

3. 打开 Chrome，地址栏输入 `chrome://extensions/`

4. 开启右上角的「开发者模式」

5. 点击「加载已解压的扩展程序」，选择第 2 步解压的目录

6. 扩展安装完成后，运行以下命令验证连接：

   ```bash
   opencli doctor
   ```

   如果输出显示连接成功，说明 Browser Bridge 已就绪。

---

## 4. 验证安装

在 Claude Code 中输入：

```
/job-hunter
```

Skill 会自动进入环境检测流程，逐项检查以下内容：

- opencli 是否已安装
- Browser Bridge 扩展是否已连接
- Chrome 是否已登录 Boss 直聘

所有检测项通过后，即可正常使用。

---

## 5. 首次使用

运行 `/job-hunter` 后，如果是新用户，Skill 会自动进入引导流程：

1. 通过交互式问答收集求职画像（城市、薪资、技术栈等）
2. 生成求职画像文件
3. 创建定时搜索任务

按提示逐步完成即可。后续运行 `/job-hunter` 会自动读取已有的求职画像，直接进入搜索流程。

---

## 常见问题

### opencli doctor 提示连接失败

- 确认 Chrome 浏览器已打开
- 确认 Browser Bridge 扩展已启用（chrome://extensions/ 中检查）
- 尝试重启 Chrome 后再次运行 `opencli doctor`

### Boss 直聘登录检测失败

- 在 Chrome 中手动访问 [zhipin.com](https://www.zhipin.com) 并登录
- 登录成功后重新运行 `/job-hunter`

### Node.js 版本不满足

- 使用 [nvm](https://github.com/nvm-sh/nvm)（Linux/macOS）或 [nvm-windows](https://github.com/coreybutler/nvm-windows)（Windows）管理 Node.js 版本
- 升级到 Node.js >= 20 后重新验证
