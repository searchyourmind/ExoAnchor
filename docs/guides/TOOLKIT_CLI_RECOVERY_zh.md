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

## 网络配置的确认和恢复

`network configure` 执行“读取原身份 → 暂存 → apply → 验证 → commit → 再读取”的流程。
下面的示例会实际修改设备网络；确认目标和预期配置后才运行：

```sh
toolkit/.venv/bin/exoanchor-toolkit network configure \
  --transport network --device https://192.0.2.10 \
  --mode dhcp --verify-timeout 20 --json
```

静态模式改用 `--mode static --address 192.0.2.20 --netmask 255.255.255.0`，
并按实际网络填写可选的 gateway 和 DNS。DHCP 重新发现可添加
`--discovery-subnet 192.0.2.0/24`；这些地址均为文档示例，不能直接代入真实网络。

确认保存前，客户端要求设备报告与修改前相同的 `device_id`、已连接状态和待确认状态；
静态模式还必须报告指定 IP，并等地址冲突检查离开中间状态。
HTTP 静态模式通过新 IP 验证，DHCP 模式检查原地址并尝试重新发现同一 `device_id`。
验证窗口 `--verify-timeout` 默认 20 秒，允许 0.1–110 秒；它不修改固件的 120 秒待确认回滚计时器，
也不是整个 CLI 调用的严格总耗时上限。

手动 `network dhcp/static/hostname --apply --commit` 会在 apply 后直接请求 commit，
不会执行 `network configure` 的重新发现和就绪检查。`--commit` 必须与 `--apply` 一起使用。

| 失败阶段 | 后续操作 |
| --- | --- |
| 缺少 `device_id` 或参数校验失败 | 确认身份与输入；不会进入 apply。 |
| apply 后验证窗口内没有获得可确认的状态 | 客户端不调用 commit，交给设备的待确认回滚机制；先等待并检查原地址或重新发现设备。CLI 已退出不等于设备已完成恢复。 |
| commit 响应或 commit 后的最终读取失败 | 配置可能已保存，不能套用“尚未 commit”的结论；重新读取设备配置，确认结果后再操作。 |

恢复后，可用 `network show` 检查实际运行配置和设备身份；仍无法通过网络访问时，
通过已确认的诊断 UART0 读取：

```sh
toolkit/.venv/bin/exoanchor-toolkit network show --transport uart \
  --port /dev/EXAMPLE_UART --json
```

`network rollback` 请求恢复 last-good 配置；`network reset --yes` 恢复出厂 DHCP，二者含义不同。
读取状态和确认目标前，不把 reset 当作自动重试步骤。

## 本机可复现的检查范围

准备好 Toolkit 环境后，从仓库根目录运行以下既有测试：

```sh
(
  cd toolkit &&
  .venv/bin/python -m unittest \
    tests.test_cli tests.test_network_configure tests.test_uart_recovery -v
)
```

这些测试使用模拟设备和时钟，验证参数解析、配置确认顺序、失败时不 commit、
串口重新枚举后的同设备匹配及版本检查；不执行真实设备网络变更或烧录。
记录测试所用的源码版本、Python 版本、模块和结果；真机连通、回滚时间及启动结果需另外验证。
实现可对照 [CLI](../../toolkit/exoanchor_toolkit/cli.py)、
[网络配置流程](../../toolkit/exoanchor_toolkit/network_configure.py)和
[UART 运行确认](../../toolkit/exoanchor_toolkit/uart_recovery.py)。
