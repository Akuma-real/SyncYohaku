# SyncYohaku

同步、构建并发布 `innei-dev/Yohaku` 的自动化仓库。

## 工作流职责

`.github/workflows/mirror.yml` 会按定时任务、手动触发或
`repository_dispatch` 检查上游 `main`：

1. 读取本仓 `build_hash` 中记录的已发布上游提交。
2. 检查上游 `innei-dev/Yohaku` 是否出现新提交。
3. 预检上游源码是否处于可构建或可临时适配状态。
4. 将上游 `main` 同步到 GitHub 目标仓 `Akuma-real/Yohaku` 和 CNB 目标仓
   `june-ink/yohaku`。
5. 构建并推送镜像：
   - `ghcr.io/akuma-real/yohaku:latest`
   - `ghcr.io/akuma-real/yohaku:<short-sha>`
   - `docker.cnb.cool/june-ink/yohaku:latest`
   - `docker.cnb.cool/june-ink/yohaku:<short-sha>`
6. 构建发布成功后，将完整上游 SHA 写回 `build_hash`。

## 上游构建预检

上游当前可能出现 workspace 结构暂时不完整的情况。例如
`apps/web` 依赖 `@yohaku/design-system@workspace:*`，但该包没有被根目录
`pnpm-workspace.yaml` 纳入 workspace。

本仓不修改上游仓库来绕过这类问题。工作流会先检出上游指定提交、初始化
submodule 和 LFS，然后检查 `@yohaku/design-system` 是否对根 pnpm workspace
可见。

如果该包存在于 `design-oss/design-system`，但尚未被根 workspace 纳入，Docker 构建前
会只在 GitHub Actions 的临时构建目录中向 `pnpm-workspace.yaml` 追加该路径。这个补丁
不会提交到上游，也不会推送到目标仓；目标仓仍保持对上游 `main` 的原样同步。

如果依赖包完全缺失，后续同步、Docker 构建和 `build_hash` 更新都会跳过，工作流保持成功
结束，并在日志中输出原因。下一次定时任务会继续检查同一个上游提交。

## 分支策略

工作流只同步目标仓的 `main` 分支，不会删除目标仓中的其他远程分支。

## 必需 Secrets

- `PAT`: 读取上游仓库、推送 GitHub 目标仓、登录 GHCR。
- `CNB_TOKEN`: 推送 CNB Git 仓库并登录 CNB 镜像仓库。
