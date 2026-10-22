# DeepSeek Harness · macOS 26 Intel 桌面版构建

用 GitHub Actions 把上游 [deepseek-ai/deepseek-harness](https://github.com/deepseek-ai/deepseek-harness)
编译成 **macOS 26 / Intel (x64)** 桌面安装包。

本仓库只保存 workflow。源码在运行时从上游远程仓库拉取，不做任何镜像或改动。

## 逻辑

每天 22:00（UTC+8）跑一个 Linux 检查任务，只有上游有新 commit 时才编译。

判断不需要额外记账：**Release 的 tag 就是记录**。

```
desktop-macos-x64-<上游 commit 前 12 位>
```

- 该 tag 的 Release 存在 → 这个 commit 已经构建并发布过 → 跳过
- 不存在 → 构建，成功后用这个 tag 发布 Release

所以"已构建"和"产物在哪"是同一件事，不会出现状态与实际产物不一致。
**删除某个 Release 就等于要求重建那个 commit。**

## 使用

**自动执行**：每天北京时间 22:00。上游没变只跑约 20 秒的检查；有新 commit
才完整编译（约 40 分钟），产出 DMG + Release。

GitHub 的 cron 用 UTC，`0 14 * * *` 即 UTC+8 的 22:00。定时任务为尽力调度，
队列繁忙时会延后；且只在默认分支（`main`）生效。仓库连续 60 天无提交活动时
GitHub 会自动暂停定时任务。

**手动执行**：Actions → **Build Desktop · macOS 26 Intel (x64)** → Run workflow。

| 参数 | 说明 |
| --- | --- |
| `source_ref` | 上游分支或标签，默认 `master` |
| `build_version` | `auto` 按日期编号（默认）；`product` 用上游清单版本；或显式版本 |

产物在 run 的 **Artifacts** 里，同时发布为 **Releases**。

## 构建内容

两个 job：`check`（ubuntu，约 20 秒）把上游 ref 解析成具体 commit 并查 Release；
`build`（`macos-26-intel`）只在需要时运行，检出的正是 `check` 解析出的那个
commit——检查与构建之间即使分支又推进，也不会构建没检查过的代码。

- Runner：`macos-26-intel`（标准托管 runner，Intel x64，macOS 26）。
- 命令：上游自带的 `pnpm run package:desktop:mac:x64:unsigned`，即上游的
  `mac-x64` 目标 —— `--unsigned` 走临时签名（`identity: '-'`），
  不需要 Apple 签名证书、公证凭据或上传凭据。
- 运行前会回收 runner 上本构建用不到的工具链（Android SDK、iOS 模拟器、
  .NET、GHC、Homebrew 缓存等）以腾出磁盘；只有在空间仍不足时才移除 Xcode
  并切到 Command Line Tools（`cc`、`codesign`、`hdiutil` 都由 CLT 提供）。
- 构建后会核对产物架构、`LSMinimumSystemVersion` 与签名信息，并写出 SHA-256。

## 已知边界

- **未公证。** 产物是临时签名，下载后首次启动会被 Gatekeeper 拦截，需要
  `xattr -dr com.apple.quarantine "/Applications/DeepSeek Harness.app"`。
  正式分发需要 Apple Developer ID 证书与公证凭据，本 workflow 不使用任何密钥。
- 产物是 x64：Intel Mac 原生运行，Apple Silicon 走 Rosetta 2。
- 上游的签名/公证打包走自托管 runner，本 workflow 不使用那条路径。
