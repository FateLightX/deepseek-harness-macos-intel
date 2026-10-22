# DeepSeek Harness · macOS 26 Intel 桌面版构建

把上游 [deepseek-ai/deepseek-harness](https://github.com/deepseek-ai/deepseek-harness)
的最新 Release 编译成 **macOS 26 / Intel (x64)** 桌面安装包。

本仓库只保存 workflow。源码在运行时从上游远程仓库拉取，不做镜像、不改动。

## 逻辑

每天 22:00（UTC+8）先跑一个约 10 秒的 Linux 检查，**只有上游发了新 release 才编译**。

1. 取上游最新的 `dsh-v*` Release（不区分 rc / dev / alpha，只取最新）
2. 由 tag 推出桌面版版本：`dsh-v0.2.1-rc.2` → `0.2.1-rc.2`
3. 查本仓库是否已有 Release `desktop-macos-x64-0.2.1-rc.2`
   - **有** → 跳过，不占用 macOS runner
   - **没有** → 检出该 tag 编译，成功后发布同 tag 的 Release

Release tag 本身就是"已构建"的记录，没有额外标记、状态文件或变量。
**删掉某个 Release = 要求重建那个版本。**

## 使用

**自动执行**：每天北京时间 22:00。上游没新 release 只跑约 10 秒检查。

GitHub 的 cron 用 UTC，`0 14 * * *` 即 UTC+8 的 22:00。定时任务为尽力调度，
队列繁忙时会延后，且只在默认分支（`main`）生效。仓库连续 60 天无提交活动时
GitHub 会自动暂停定时任务。

**手动执行**：Actions → **Build Desktop · macOS 26 Intel (x64)** → Run workflow。
`release_tag` 留空取最新 release；填 `dsh-v0.2.1-rc.1` 可补构建指定版本。

## 构建内容

| job | runner | 作用 |
| --- | --- | --- |
| `check` | ubuntu | 取最新 release、推出目标 tag、判断是否需要构建 |
| `build` | `macos-26-intel` | 仅在需要时运行，检出 release tag 编译 |

- 版本号直接来自 tag，无需任何版本参数；编译前会断言检出代码的 `version`
  与 tag 推导值一致，避免发布错版本。
- 命令用上游自带的 `pnpm run package:desktop:mac:x64:unsigned`（`mac-x64` 目标），
  `--unsigned` 走临时签名，不需要 Apple 证书、公证或上传凭据。
- 运行前回收 runner 上本构建用不到的工具链（Android SDK、iOS 模拟器、.NET、
  GHC、Homebrew 缓存）腾磁盘；仅当空间仍不足时才移除 Xcode 并切到
  Command Line Tools（`cc`、`codesign`、`hdiutil` 都由 CLT 提供）。
- 构建后核对架构、`LSMinimumSystemVersion`、签名与产物版本号，并写出 SHA-256。

## 已知边界

- **未公证。** 产物为临时签名，下载后首次启动会被 Gatekeeper 拦截，需执行
  `xattr -dr com.apple.quarantine "/Applications/DeepSeek Harness.app"`。
  正式分发需要 Apple Developer ID 证书与公证凭据。
- 产物是 x64：Intel Mac 原生运行，Apple Silicon 走 Rosetta 2。
- 两次运行之间上游连发多个 release 时，只构建最新的一个，不补历史。
