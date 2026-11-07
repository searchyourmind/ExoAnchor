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

## 烧录预演与实际写入的区别

`firmware flash --dry-run` 会先打开并校验本地包，再按所选传输构造更新计划。
网络预演示例：

```sh
toolkit/.venv/bin/exoanchor-toolkit firmware flash ./firmware-package \
  --transport network --device https://192.0.2.10 --dry-run --json
```

网络预演检查包的板型是否在 OTA 允许列表内，并检查目标地址和超时参数。
它也会初始化 TLS 配置，因此设置了配对证书文件时，该文件必须能被正常读取。
它不连接或登录设备，不读取设备 capability，不建立上传 session，也不验证目标可达性。
JSON 中的 `network-ota` 是计划描述，不是另一个可以直接执行的 shell 命令。

UART 预演仍要求完整的本地身份资料：

```sh
toolkit/.venv/bin/exoanchor-toolkit firmware flash ./firmware-package \
  --transport uart --port /dev/EXAMPLE_UART \
  --identity-registry ./BOARD_IDENTITY_REGISTRY.toml \
  --board-record OPERATOR_CONFIRMED_RECORD --dry-run --json
```

将端口、注册表路径和记录名替换为已确认的设备资料；不要把示例值当成身份记录。
即使是预演，UART 烧录也拒绝 `auto`、空端口和通配符，并核对注册表授权及包的板型。
预演不会探测实际 CH343 序列号、eFuse MAC 或芯片 revision；实际写入前才会核对这些值。
UART 实际写入后还会等待同一设备重新枚举，并核对其运行中的固件版本和板型。

- `--dry-run --json` 输出结构化计划；实际 `flash --json` 仍输出进度文本，不能当作 JSON 结果解析。
- `--no-reboot` 只影响网络 OTA；它不能阻止 UART 烧录流程中的复位。
- `--yes` 只跳过交互确认，不能跳过包校验、板型或身份检查。

真机烧录前的身份确认与验收条件见 [固件修改验证规程](FIRMWARE_VALIDATION_zh.md)。
