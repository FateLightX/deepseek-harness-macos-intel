# DeepSeek Harness · macOS 26 Intel 桌面版构建

用 GitHub Actions 把上游 [deepseek-ai/deepseek-harness](https://github.com/deepseek-ai/deepseek-harness)
编译成 **macOS 26 / Intel (x64)** 桌面安装包。

本仓库只保存 workflow。源码在运行时从上游远程仓库拉取，不做任何镜像或改动。

## 使用

1. 打开 **Actions → Build Desktop · macOS 26 Intel (x64) → Run workflow**。
2. 填写参数：

   | 参数 | 说明 |
   | --- | --- |
   | `source_ref` | 上游分支或标签，例如 `master`、`dsh-v0.2.1-alpha.4` |
   | `build_version` | `auto` 按日期自动编号；`product` 用上游清单版本；也可显式填写受支持的版本 |
   | `publish_release` | 构建成功后创建 GitHub Release，附 DMG 与 SHA-256 |

3. 产物在 run 的 **Artifacts** 里（`deepseek-harness-<版本>-mac-x64-unsigned.dmg`），
   勾选 `publish_release` 时同时出现在 **Releases**。

## 构建内容

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
