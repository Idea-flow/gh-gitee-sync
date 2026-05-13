# GitHub → Gitee 全仓库镜像同步

通过 GitHub Actions 自动将 **GitHub 账号下所有仓库** 同步到 Gitee，支持定时 + 手动触发。

## 工作原理

使用 [Yikun/hub-mirror-action](https://github.com/Yikun/hub-mirror-action) 自动发现 GitHub 账号下的所有仓库，同步到同名的 Gitee 仓库（不存在则自动创建）。

## 使用步骤

### 1. 准备 Gitee 令牌与 SSH 密钥

- **GITEE_TOKEN**：在 [Gitee 个人设置 → 私人令牌](https://gitee.com/profile/personal_access_tokens) 创建，勾选 `projects` 权限
- **GITEE_PRIVATE_KEY**：生成 SSH 密钥对，将**公钥**添加到 [Gitee SSH 公钥](https://gitee.com/profile/sshkeys)

生成 SSH 密钥（如果还没有）：

```bash
ssh-keygen -t ed25519 -C "gitee-sync" -f ~/.ssh/id_ed25519_gitee
```

### 2. 在 GitHub 仓库配置 Secrets

将本仓库推送到 GitHub 后，进入 **Settings → Secrets and variables → Actions**，添加：

| Name | Value |
|------|-------|
| `GITEE_PRIVATE_KEY` | SSH 私钥内容（`~/.ssh/id_ed25519_gitee` 的全文） |
| `GITEE_TOKEN` | Gitee 私人令牌 |

### 3. 配置 Variables（可选但推荐）

在同一个页面切换到 **Variables** 标签，添加：

| Name | Value |
|------|-------|
| `GITHUB_USERNAME` | 你的 GitHub 用户名 |
| `GITEE_USERNAME` | 你的 Gitee 用户名 |

> 如果不配置 Variables，需直接在 workflow 文件中硬编码用户名。

### 4. 手动触发验证

进入 GitHub 仓库 **Actions** 页面，选择 **Sync all repos from GitHub to Gitee** 工作流，点击 **Run workflow** 手动触发一次，检查是否成功。

## 同步策略

| 项目 | 说明 |
|------|------|
| 触发方式 | 定时（每 30 分钟）+ 手动触发 |
| 同步内容 | 所有分支、标签、提交记录 |
| 同步方向 | GitHub → Gitee（单向） |
| 黑名单 | 如需排除某些仓库，修改 `black_list` 参数 |
| 白名单 | 如需只同步特定仓库，用 `static_list` 或 `white_list` |

## 常见问题

**Q：Gitee 仓库已存在但不同名？**
A：hub-mirror-action 会自动匹配同名仓库，不同名需手动在 Gitee 创建后，用 `static_list` 指定映射。

**Q：同步失败怎么办？**
A：检查 Secrets 是否正确配置，以及 Gitee 公钥是否已添加。可在 Actions 日志中查看详细错误。

**Q：只想同步部分仓库？**
A：取消注释 `static_list` 或 `white_list` 参数，填入仓库名（逗号分隔）。
