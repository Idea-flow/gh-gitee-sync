# GitHub → Gitee 全仓库镜像同步

通过 GitHub Actions 将 **GitHub 账号下所有仓库** 自动同步到 Gitee，支持定时（每小时）+ 手动触发。

## 工作原理

使用 [Yikun/hub-mirror-action](https://github.com/Yikun/hub-mirror-action) 自动发现 GitHub 账号下所有仓库，同步到同名的 Gitee 仓库（不存在则自动创建）。

## 工作流配置说明

当前工作流文件 `.github/workflows/sync-to-gitee.yml` 中使用了以下配置项：

| 配置项 | 来源 | 说明 |
|--------|------|------|
| `vars.GH_USERNAME` | GitHub Variables | 你的 GitHub 用户名 |
| `vars.GITEE_USERNAME` | GitHub Variables | 你的 Gitee 用户名 |
| `secrets.GITEE_PRIVATE_KEY` | GitHub Secrets | Gitee 账号的 SSH 私钥 |
| `secrets.GITEE_TOKEN` | GitHub Secrets | Gitee 个人访问令牌 |

## 配置步骤

### 1. 准备 Gitee 的凭据

**GITEE_TOKEN**：在 [Gitee 个人设置 → 私人令牌](https://gitee.com/profile/personal_access_tokens) 创建，勾选 `projects` 权限。

**GITEE_PRIVATE_KEY**：生成一组 SSH 密钥（包含一个私钥和一个公钥），GitHub Actions 用私钥连接 Gitee，Gitee 用公钥来验证身份。

① 打开终端，执行以下命令生成密钥对：

```bash
ssh-keygen -t ed25519 -C "gitee-sync" -f ./id_ed25519_gitee
```

② 执行过程中会提示输入密码，**直接按回车跳过**（不要设置密码，否则 Actions 无法自动使用）：

```
Enter passphrase (empty for no passphrase):   ← 直接回车
Enter same passphrase again:                  ← 直接回车
```

③ 命令执行完后，当前目录下会多出两个文件：
- `id_ed25519_gitee` — **私钥**（绝不能泄露），稍后填入 GitHub Secrets
- `id_ed25519_gitee.pub` — **公钥**（可以公开），需要添加到 Gitee

> 这两个文件生成在当前目录，用完记得不要误提交到 Git。

④ 查看公钥内容，复制它：

```bash
cat ./id_ed25519_gitee.pub
```

会输出类似这样的内容（全部复制）：

```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAINmRqP0f3... gitee-sync
```

⑤ 打开 [Gitee SSH 公钥设置页面](https://gitee.com/profile/sshkeys)，点击 **添加公钥**，将刚才复制的内容粘贴到"公钥"输入框，"标题"随便填（如 `github-sync`），点击确定。

⑥ **同样把公钥添加到 GitHub**（因为同步私有仓库需要用 SSH 方式克隆）：
打开 [GitHub SSH Keys 设置页面](https://github.com/settings/keys)，点击 **New SSH key**，标题填 `gitee-sync`，将同样的公钥内容粘贴进去，点击 Add SSH key。

> **为什么要这么做？** 由于 `zimeiti` 是私有仓库，我们需要用 SSH 方式克隆代码（而非 HTTPS）。这把 SSH 密钥同时用于两件事：从 GitHub **克隆**代码（GitHub 验证身份）+ 推送到 Gitee（Gitee 验证身份）。所以公钥要同时添加到 GitHub 和 Gitee。

### 2. 在 GitHub 仓库配置

将本项目推送到 GitHub 后，进入 **Settings → Secrets and variables → Actions**，分两部分配置：

**Secrets（机密）** — 切换到 Secrets 标签，添加：

| Name | Value |
|------|-------|
| `GITEE_PRIVATE_KEY` | SSH **私钥**内容，即 `./id_ed25519_gitee` 的全文 |
| `GITEE_TOKEN` | Gitee 私人令牌的字符串 |

**Variables（变量）** — 切换到 Variables 标签，添加：

| Name | Value |
|------|-------|
| `GH_USERNAME` | 你的 GitHub 用户名，如 `Idea-flow` |
| `GITEE_USERNAME` | 你的 Gitee 用户名 |

### 3. 手动触发验证

进入 GitHub 仓库 **Actions** 页面，选择 **Sync all repos to Gitee** 工作流，点击 **Run workflow** → 绿色按钮，等待执行完成，检查日志是否成功。

> 后续 GitHub 仓库有新的推送时，每小时会自动同步一次，也可以随时手动触发。

## 同步策略配置

### 全量强制同步

默认是增量同步（只推送新增提交），如需让 Gitee 与 GitHub **完全一致**（包括删除的分支、回退的提交等），在 workflow 中加 `force_update: true`：

```yaml
with:
  force_update: true
```

加上后会用 `git push -f` 强制覆盖 Gitee 仓库，**谨慎使用**。

### 禁止同步某些仓库

在 workflow 的 `with` 中添加 `black_list` 参数，多个仓库名用逗号分隔：

```yaml
with:
  black_list: 'repo-archive,repo-deprecated,test-playground'
```

### 只允许同步某些仓库

用 `white_list` 或 `static_list` 指定白名单：

```yaml
with:
  white_list: 'repo1,repo2'
```

> **三个参数优先级**：`static_list` > `black_list` > `white_list`。`static_list` 会完全替代自动发现的仓库列表，仅同步列出的仓库。

## 常见问题

**Q：这样做会被 GitHub 封号吗？**
A：不会。GitHub 允许通过 Actions 向外部服务推送代码，这是合法的自动化操作。镜像自己的仓库属于正常使用行为，不会被封号。但需要注意：不要滥用 GitHub Actions 免费额度（公开仓库无限额度，私有仓库有月度限制），不要频繁触发大规模同步。

**Q：Gitee 仓库已存在但不同名？**
A：不同名需手动在 Gitee 创建后，用 `static_list` 指定映射关系。

**Q：同步失败怎么办？**
A：进入 Actions 日志查看详细错误。常见原因：Secrets 未配置或配置错误、Gitee SSH 公钥未添加、Gitee Token 权限不足。
