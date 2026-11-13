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
