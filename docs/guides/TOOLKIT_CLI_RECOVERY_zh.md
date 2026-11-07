# Toolkit CLI 参数、预演与网络配置恢复

本指南帮助操作者区分发现设备、读取配置、预演烧录与实际修改设备的命令。
下载、包校验和 OTA 分块续传的回归范围见 [Toolkit 网络 OTA 验证](TOOLKIT_OTA_VALIDATION_zh.md)；
首次连接 HTTPS 设备前，按 [HTTPS 与首次配对指南](SECURITY_TRANSPORT_zh.md)准备账号和配对证书。

## 准备与目标选择

以下命令从仓库根目录运行。首次准备环境可能需要下载 Python 和依赖：

```sh
uv sync --locked --project toolkit --python 3.12
toolkit/.venv/bin/exoanchor-toolkit --help
toolkit/.venv/bin/exoanchor-toolkit network configure --help
toolkit/.venv/bin/exoanchor-toolkit firmware flash --help
```

各命令的默认传输方式不同，`--port` 也不是通用的设备选择参数：

| 命令 | 默认传输与端口参数 | 用途 |
| --- | --- | --- |
| `discover` | 默认 `network`；`--port` 是 UDP 发现端口。UART 发现使用 `--transport uart --serial-port`。 | LAN 探测或通过诊断 UART0 读取身份；成功发现不等于已经验证 HTTPS 配对。 |
| `network show` | 默认 `uart`；UART 使用 `--port`，HTTP API 使用 `--transport network --device`。 | 读取当前及已存网络配置，不是 OTA 上传状态命令。 |
| `firmware flash` | 默认 `uart`；UART 使用 `--port`，网络 OTA 必须显式指定 `--transport network --device`。 | 预演或执行固件更新，具体边界见下节。 |

下面是读取设备信息的示例；替换文档用 IPv4 地址和占位串口后再使用：

```sh
toolkit/.venv/bin/exoanchor-toolkit discover --transport network --json
toolkit/.venv/bin/exoanchor-toolkit discover --transport uart \
  --serial-port /dev/EXAMPLE_UART --json
toolkit/.venv/bin/exoanchor-toolkit network show --transport network \
  --device https://192.0.2.10 --json
```

`--device` 接受 IPv4 地址及可选端口；裸地址默认 HTTPS 443。
它不接受 hostname、IPv6、`/api/...` 路径、URL 内的账号密码、query 或 fragment。
`network` 的传输参数可以放在子命令前或后；为了便于检查，上述示例统一放在后面。
设备密码可通过 `EXOANCHOR_DEVICE_PASSWORD` 提供，第二因素可通过
`EXOANCHOR_OTP_CODE` 提供；不要把实际凭据写入共享命令记录。
