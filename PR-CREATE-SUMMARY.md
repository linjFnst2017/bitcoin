# 创建 PR 流程总结

## 背景

在 Claude Code（VSCode 扩展环境）中，通过 Git 命令 + GitHub REST API 完成了从提交代码到创建 Pull Request 的全流程。

> **注意：** 本次操作**没有使用 MCP 服务器**（虽然配置了 GitHub MCP 服务器，但未启用）。直接通过内置的 `Bash` 工具执行了 git 命令和 curl 命令。

---

## 流程步骤

### 1. 提交代码到本地仓库

```bash
git add <文件>
git commit -m "提交信息"
```

### 2. 创建特性分支

由于 PR 需要从特性分支提交，不能直接从 `master` 发起：

```bash
git checkout -b feat/add-project-config
```

### 3. 推送到远程仓库

```bash
git push -u origin feat/add-project-config
```

### 4. 回退 master 分支（如果已误提交到 master）

如果 commit 已经推送到了 `master`，需要回退并强制推送：

```bash
git checkout master
git reset --hard <上一个commit的hash>
git push --force origin master
```

### 5. 创建 PR

两种方式：

#### 方式 A：使用 curl 调用 GitHub REST API

```bash
curl -X POST \
  -H "Authorization: token <GITHUB_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "PR 标题",
    "body": "PR 描述",
    "head": "feat/your-branch",
    "base": "master"
  }' \
  https://api.github.com/repos/<用户名>/<仓库名>/pulls
```

#### 方式 B：浏览器手动创建

打开 GitHub 提供的链接：
`https://github.com/<用户名>/<仓库名>/pull/new/<分支名>`

---

## 关键工具说明

| 工具 | 作用 | 来源 |
|------|---------|---------|
| `git` | 本地版本控制、推送代码到远程 | 系统安装的 Git |
| `curl` | 命令行 HTTP 客户端，调用 GitHub REST API | Git Bash 内置 |
| GitHub API | `POST /repos/{owner}/{repo}/pulls` 创建 PR | GitHub REST API v3 |
| Git Credential Manager | 安全存储 GitHub token，自动处理认证 | Windows 系统凭据管理 |

---

## MCP 方式创建 PR（实际操作流程）

以下是通过配置并使用 GitHub MCP 服务器创建 PR 的完整流程：

### 1. 配置 MCP 服务器

在项目 `.claude/mcp.json` 中定义 GitHub MCP 服务器：

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      }
    }
  }
}
```

### 2. 设置 GITHUB_TOKEN 环境变量

在 `.claude/settings.local.json` 中添加：

```json
{
  "env": {
    "GITHUB_TOKEN": "gho_xxxxxxxxxxxxxxxxxxxx"
  }
}
```

Token 可从 Windows Git Credential Manager 获取：
```bash
git credential fill <<< $'protocol=https\nhost=github.com\n'
```

### 3. 确认 MCP 服务器状态

```bash
claude mcp list
```

### 4. 通过对话直接创建 PR

配置完成后，MCP 工具会自动加载。只需在对话中提出请求：

> "从 feat/mcp-demo 分支创建一个 PR 到 master"

MCP 的 GitHub 工具会自动调用 GitHub API 完成创建。

---

## MCP 与本次操作的对比

| 方式 | 说明 |
|------|---------|
| **git + curl（本次使用）** | 直接通过 Bash 执行命令行工具，不需要 MCP 服务器 |
| **GitHub MCP 服务器** | 如果配置并启用，可直接通过 MCP 工具调用 GitHub API，无需手动拼 curl 命令 |
| **gh CLI** | GitHub 官方命令行工具，`gh pr create` 一行命令完成 |

如果用 MCP 创建 PR，流程会更简洁：

```
# 在 Claude Code 中直接对话即可，MCP 工具会自动处理 API 调用
"帮我从这个分支创建一个 PR"
```

---

## 认证方式

- **Git push**：使用 Windows Git Credential Manager 中保存的凭据
- **GitHub API**：从凭据管理器中读取到 token（`gho_` 前缀的 OAuth token），通过 `Authorization` 请求头发送
- token 存储在 **Windows 凭据管理器**（控制面板 → 凭据管理器 → Windows 凭据 → `git:https://github.com`）
