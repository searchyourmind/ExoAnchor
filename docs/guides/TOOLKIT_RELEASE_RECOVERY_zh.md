# Toolkit Release 选择、缓存与 OTA 故障恢复

本指南说明怎样记录可复查的下载结果，以及网络 OTA 中断后如何判断下一步。
命令从仓库根目录运行；Python 环境、设备参数和 HTTPS 配对准备见
[Toolkit CLI 指南](TOOLKIT_CLI_RECOVERY_zh.md)。示例中的 `RELEASE_TAG` 和
`/PATH/TO/exoanchor-firmware.json` 必须替换成查询结果的 tag 和下载结果中的 manifest 路径。

## 选择并记录 Release

先查询可用固件包，再指定完整 tag 下载：

```sh
toolkit/.venv/bin/exoanchor-toolkit firmware versions \
  --repository wxffxx/ExoAnchor --json
toolkit/.venv/bin/exoanchor-toolkit firmware download RELEASE_TAG \
  --repository wxffxx/ExoAnchor --output ./firmware-downloads --json
```

`versions` 只读取 GitHub 元数据并筛选符合要求的 Release 与 asset，
不会下载 ZIP 或校验其中的镜像。`download` 才下载、解包并校验固件包。
默认仓库是 `wxffxx/ExoAnchor`；记录中显式保留仓库名，避免把 fork 和上游混用。

| 选择方式 | 当前行为与边界 |
| --- | --- |
| `versions` | 只请求 `/releases?per_page=30` 的第一页，没有分页；过滤后按 `published_at` 倒序排列，不按语义版本号排序。 |
| `download RELEASE_TAG` | 在同一页过滤结果中精确匹配完整 tag，不单独调用按 tag 查询的 API。固定 tag 便于记录选择，但不保证历史版本始终可查询或 asset 永远不变。 |
| `latest` | 下载当前查询所得且符合筛选条件的最新发布项；下次执行可能选中另一版本。 |
| `--stable-only` | 三种命令默认都包含 prerelease；此开关才排除 prerelease。它不是设备兼容性或真机稳定性的验证。 |

例如，仅在希望选择当前查询结果中的最新非 prerelease 时运行：

```sh
toolkit/.venv/bin/exoanchor-toolkit firmware latest \
  --repository wxffxx/ExoAnchor --stable-only \
  --output ./firmware-downloads --json
```

候选 Release 必须不是 draft，并且恰好有一个满足固件包命名和元数据检查的
`exoanchor-firmware-*.zip` asset；GitHub 自动生成的源码归档不等于固件包。
`no stable ... release found` 只说明当前查询结果没有符合条件的稳定固件包。
`firmware release ... is not available` 也可能是 tag 位于后续页面、被 stable 筛选排除，
或 asset 不合格；先核对 Release 页面与筛选条件，不据此断定 tag 不存在。

保存下载结果中的 `repository`、`release.tag`、asset ID/digest、包的 board/version、
`package.manifest`、`directory`、`cached` 和 `archive_sha256`，同时记录 Toolkit 源码版本及命令。
包的 SHA-256 检查证明文件与所检查摘要一致，不是发布者数字签名验证；
`Verification: PASS` 也不证明目标板型匹配、烧录成功或新固件已启动。

## 区分缓存复验与离线使用

`download` 和 `latest` 每次都先在线查询 Release 元数据，再检查本地缓存。
缓存命中只省略 asset 的 ZIP 下载；这两个命令没有离线开关。
若 GitHub 查询失败，即使包目录还在，下载命令也可能无法完成。

`--output` 指定的是父目录，实际包保存在自动命名的子目录中：asset 文件名去掉 `.zip`，
再附加元数据中 SHA-256 十六进制摘要的前 12 位；没有 digest 时使用 asset ID。
以 JSON 的 `directory` 或文本的 `Package:` 路径为准，不自行拼接版本目录。
未指定 `--output` 时使用用户缓存目录，macOS 上是
`~/Library/Caches/exoanchor-toolkit/firmware`。成功下载保留解包目录，临时原始 ZIP 会被清理。

| 下载结果 | 实际执行的检查 | `archive_sha256` 的含义 |
| --- | --- | --- |
| `cached: false` | 检查下载大小，计算 ZIP 摘要；元数据提供 digest 时还会比对；解包后校验 manifest、segment 长度/摘要及包版本。 | 本次下载 ZIP 的计算值。 |
| `cached: true` | 重新读取缓存 manifest，校验 segment 长度/摘要，并核对包版本与 tag 对应版本。 | 来自当前元数据 digest；未提供 digest 时为空字符串。不是重新计算原 ZIP 的结果。 |

已有缓存损坏、manifest 数量不正确、目标为符号链接或包版本不匹配时，客户端会报错，
不会自动删除、覆盖或重新下载该目录。先保留错误信息和目录用于排查；需要重新下载时，
可指定一个新的父目录，下载成功后再决定如何处理旧缓存：

```sh
toolkit/.venv/bin/exoanchor-toolkit firmware download RELEASE_TAG \
  --repository wxffxx/ExoAnchor --output ./firmware-recovery-downloads --json
```

准备好 Python 依赖后，使用已有本地目录、manifest 或 ZIP 的 `inspect` / `verify`
不查询 GitHub。下面的路径应替换为成功下载 JSON 结果中的 `package.manifest`；
ZIP 可能将 manifest 放在嵌套子目录中，此时不能直接把最外层 `directory` 作为包目录：

```sh
toolkit/.venv/bin/exoanchor-toolkit firmware verify \
  /PATH/TO/exoanchor-firmware.json --json
```

这一步可离线检查本地包，但不能替代设备连接、身份核对或更新后的启动确认。
