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

## 按 OTA 失败阶段决定下一步

本节只适用于 `firmware flash --transport network`。保留本次使用的包、目标信息和完整日志；
恢复前重新执行本地 `firmware verify`，按 [OTA 验证规程](TOOLKIT_OTA_VALIDATION_zh.md)
核对设备身份与 `/api/ota/upload/status`，再判断是否存在匹配的活动会话。
下表描述当前客户端行为；操作者仍需根据设备实际状态决定后续操作。

| 阶段 | 客户端行为 | 恢复时的判断 |
| --- | --- | --- |
| 初始状态检查 | 读取 capability 和上传状态；活动会话须有匹配的镜像大小/摘要、有效 session ID 和范围内 offset，才会续传。 | 不匹配时停止，确认现有上传用途；客户端不会自动取消其他会话。 |
| `start` 请求失败或响应不合法 | 当前调用停止，不自动重发或查询状态恢复。 | 设备可能已经创建会话。先核对状态；之后同一包的新 `flash` 调用会检查活动会话，匹配时才续传。 |
| `chunk` 的 `post_binary` 调用抛出 `ToolkitError`（例如 HTTP、读取或 JSON 解析错误） | 查询一次 status；同一会话的 offset 确认前进且未超过本块结尾时，从确认位置继续。 | 无进展、状态查询失败或会话变化都会停止，不无限重试。 |
| `chunk` 已返回 JSON 对象，但 `ok` / `offset` 校验失败 | 停止，不进入上一行的自动 status 恢复流程。 | 不能把所有 chunk 错误都当作自动续传；先检查同一包的活动会话及 offset。 |
| `finish` 失败、响应丢失或 digest 不符 | 停止，不请求 reboot，也不自动重发 finish。 | 设备可能已经处理 finish；先查状态，不把响应失败当成安装一定未发生。 |
| `reboot` 拒绝或响应丢失 | 报错，不自动重发 reboot。 | 镜像可能已完成安装。检查设备可达性、是否重启及实际运行版本，再决定后续操作。 |

再次执行 `flash` 不是无条件续传：若设备报告没有活动会话，客户端会开始新上传。
重复已经完成的更新也可能再次上传，不能把整条命令当作幂等重试，尤其是 finish/reboot 结果不明时。

当前 CLI 没有 `firmware status`、`firmware resume` 或 `firmware abort` 子命令。
`network show` 查询网络配置，不查询 OTA 状态。不要把不存在的命令加入恢复脚本；
应通过已授权的设备界面或 API 工具检查 OTA 状态。

CLI 在 `update()` 整体返回后才打印上传成功信息。如果 reboot 阶段报错，
即使 finish 已成功，终端也可能没有 “OTA image uploaded” 或 “SHA-256 verified” 字样。
缺少这些文字不能证明设备未接收镜像。
`--no-reboot` 表示需要另行重启激活；“Device reboot requested”只说明请求被确认，
二者都不等于新镜像已成功启动。

## 检查依据与验证范围

Release 选择、缓存和本地包行为可对照
[releases.py](../../toolkit/exoanchor_toolkit/releases.py)；
OTA 各阶段和命令输出分别见
[http_device.py](../../toolkit/exoanchor_toolkit/http_device.py)与
[cli.py](../../toolkit/exoanchor_toolkit/cli.py)。

准备好 Toolkit 环境后，从仓库根目录运行相关既有测试：

```sh
(
  cd toolkit &&
  .venv/bin/python -m unittest \
    tests.test_releases tests.test_release_transport \
    tests.test_ota_workflow tests.test_http_transport_failures -v
)
```

这些测试使用合成包、替换后的传输或本地 HTTP 服务，覆盖 Release 选择、下载与缓存复验、
chunk 响应丢失恢复、非法响应拒绝及 finish/reboot 拒绝等行为；不访问真实 GitHub Release，
也不执行物理设备烧录。第一页查询限制、坏缓存不自动替换、成功信息延后输出，
以及 start/finish/reboot 响应丢失时不自动重放的说明来自实现检查，
不能写成这些场景均已有专门回归测试。真机启动与恢复结果需按
[固件验证规程](FIRMWARE_VALIDATION_zh.md)另外记录。
