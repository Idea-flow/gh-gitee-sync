# GitHub → Gitee 仓库同步（测试）

通过 GitHub Actions 手动触发，将指定 GitHub 仓库同步到 Gitee，用于验证同步链路是否可用。

## 工作原理

使用 [Yikun/hub-mirror-action](https://github.com/Yikun/hub-mirror-action) 将指定仓库同步到同名的 Gitee 仓库（不存在则自动创建）。

## 工作流配置说明

当前工作流文件 `.github/workflows/sync-to-gitee.yml` 中使用了以下配置项：

| 配置项 | 来源 | 说明 |
|--------|------|------|
| `vars.GH_USERNAME` | GitHub Variables | 你的 GitHub 用户名 |
| `vars.GITEE_USERNAME` | GitHub Variables | 你的 Gitee 用户名 |
| `secrets.GITEE_PRIVATE_KEY` | GitHub Secrets | Gitee 账号的 SSH 私钥 |
| `secrets.GITEE_TOKEN` | GitHub Secrets | Gitee 个人访问令牌 |
| `static_list: 'zimeiti'` | 硬编码 | 当前仅同步 `zimeiti` 这一个仓库 |

## 配置步骤

### 1. 准备 Gitee 的凭据

**GITEE_TOKEN**：在 [Gitee 个人设置 → 私人令牌](https://gitee.com/profile/personal_access_tokens) 创建，勾选 `projects` 权限。

**GITEE_PRIVATE_KEY**：生成 SSH 密钥对，将**公钥**添加到 [Gitee SSH 公钥](https://gitee.com/profile/sshkeys)：

```bash
ssh-keygen -t ed25519 -C "gitee-sync" -f ~/.ssh/id_ed25519_gitee
# 查看公钥并添加到 Gitee
cat ~/.ssh/id_ed25519_gitee.pub
```

### 2. 在 GitHub 仓库配置

将本项目推送到 GitHub 后，进入 **Settings → Secrets and variables → Actions**，分两部分配置：

**Secrets（机密）** — 切换到 Secrets 标签，添加：

| Name | Value |
|------|-------|
| `GITEE_PRIVATE_KEY` | SSH **私钥**内容，即 `~/.ssh/id_ed25519_gitee` 的全文 |
| `GITEE_TOKEN` | Gitee 私人令牌的字符串 |

**Variables（变量）** — 切换到 Variables 标签，添加：

| Name | Value |
|------|-------|
| `GH_USERNAME` | 你的 GitHub 用户名，如 `wangpenglong` |
| `GITEE_USERNAME` | 你的 Gitee 用户名 |

### 3. 手动触发验证

进入 GitHub 仓库 **Actions** 页面，选择 **Sync zimeiti to Gitee** 工作流，点击 **Run workflow** → 绿色按钮，等待执行完成，检查日志是否成功。

## 常见问题

**Q：这样做会被 GitHub 封号吗？**
A：不会。GitHub 允许通过 Actions 向外部服务推送代码，这是合法的自动化操作。镜像自己的仓库属于正常使用行为，不会被封号。但需要注意：不要滥用 GitHub Actions 免费额度（公开仓库无限额度，私有仓库有月度限制），不要频繁触发大规模同步。

**Q：Gitee 仓库已存在但不同名？**
A：不同名需手动在 Gitee 创建后，用 `static_list` 指定映射关系。

**Q：同步失败怎么办？**
A：进入 Actions 日志查看详细错误。常见原因：Secrets 未配置或配置错误、Gitee SSH 公钥未添加、Gitee Token 权限不足。
