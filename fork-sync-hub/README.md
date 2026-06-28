# sync-fork

自动同步上游仓库的 tags 和分支到你的 fork。

本仓库同时提供两种使用方式：

- **方案 A（单仓库）**：在每个 fork 里放一份 workflow，调用 `SourceDive/sync-fork` action
- **方案 B（中心化）**：在本仓库用 `sync-all-forks.yml` 统一调度所有 fork（推荐）

## 前置条件

需要一个 GitHub Personal Access Token (PAT)，权限勾选 `repo` + `workflow`。

建议在 Organization `SourceDive` 级别创建 Secret `SYNC_FORK_TOKEN`，所有仓库共享。

Settings → Secrets and variables → Actions → New organization secret

## 方案 B：中心化同步所有 Fork（推荐）

本仓库已配置 `.github/workflows/sync-all-forks.yml`，会读取 `forks.json` 中的矩阵配置，每天自动同步所有 fork。

### 首次启用

1. 在 Organization 或本仓库配置 `SYNC_FORK_TOKEN`
2. 确认 `forks.json` 包含你要同步的 fork 列表
3. 到 Actions 页手动触发 **Sync All Forks** 验证

### 新增 Fork 后

运行脚本重新生成配置：

```bash
python3 scripts/generate-forks-config.py
git add forks.json
git commit -m "chore: update forks.json"
git push
```

或手动在 `forks.json` 追加一条：

```json
{
  "fork": "SourceDive/your-repo",
  "upstream": "owner/upstream-repo",
  "target_branch": "main"
}
```

### 手动触发选项

- `fork_filter`：只同步名称匹配的 fork（如 `spring-ai`）
- `sync_tags` / `sync_branches`：控制同步内容

### 启用中心化后

各 fork 仓库里的 `.github/workflows/sync-fork.yml` 可以删除，避免重复同步。

## 方案 A：单仓库同步

将 `templates/sync-fork.yml` 复制到 fork 仓库的 `.github/workflows/` 目录，修改 `upstream_repo`：

```yaml
name: Sync Fork from Upstream
on:
  schedule:
    - cron: '0 0 * * *'
  workflow_dispatch:
jobs:
  sync:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      actions: write
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: SourceDive/sync-fork@v1.1
        with:
          upstream_repo: spring-projects/spring-ai
          github_token: ${{ secrets.SYNC_FORK_TOKEN }}
          sync_tags: 'true'
          sync_branches: 'true'
          target_branch: main
```

## Action 参数

| 参数 | 必填 | 默认值 | 说明 |
|---|---|---|---|
| `upstream_repo` | 是 | — | 上游仓库，格式 `owner/repo` |
| `github_token` | 是 | — | PAT，需 `repo` + `workflow` 权限 |
| `sync_tags` | 否 | `true` | 是否同步 tags |
| `sync_branches` | 否 | `false` | 是否同步分支 |
| `target_branch` | 否 | `main` | 要同步的分支，多个用逗号分隔 |
