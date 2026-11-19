# Toolkit HTTPS 连接与认证排查

本指南面向 Toolkit CLI 的网络连接。首次创建管理员账号和取得设备公开证书，按
[HTTPS 与首次配对指南](SECURITY_TRANSPORT_zh.md)操作；两步验证设置见
[TOTP 指南](TWO_STEP_zh.md)。浏览器已信任证书或已登录，不代表 Toolkit 自动拥有该证书文件或浏览器会话。
以下命令从仓库根目录运行；示例 IP、用户名和证书路径均需替换成已确认的设备资料。

## 指定设备和证书

Toolkit 的 `network` 命令默认走 UART；通过 HTTPS 读取配置时，必须指定
`--transport network --device`。`EXOANCHOR_BASE_URL` 是 MCP 的设置，Toolkit CLI 不读取它。
目标接受 IPv4 地址及可选端口，裸地址默认 HTTPS 443；hostname、IPv6 和 API 路径不适用。
详细参数边界见 [CLI 指南](TOOLKIT_CLI_RECOVERY_zh.md)。

Toolkit 通过 `EXOANCHOR_TLS_CERTIFICATE_FILE` 读取配对 PEM，没有独立的指纹参数或 `pair` 子命令。
证书文件必须是包含单张设备证书的 ASCII PEM，不是证书链 bundle 或设备私钥。
先确认该文件来自预期设备，再设置路径：

```sh
export EXOANCHOR_TLS_CERTIFICATE_FILE="$PWD/exoanchor-device.pem"
```

| 模式 | 客户端检查 | 地址变化后的含义 |
| --- | --- | --- |
| 未设置配对文件 | 使用默认 CA 信任来源，并校验证书中的 hostname/IP。 | 自签名设备不会仅因为浏览器曾接受它就自动获得 Toolkit 信任。 |
| 设置配对文件 | 以 PEM 作为信任输入，保留证书链、签名和有效期检查；发送请求前，还核对整张叶证书 DER 的 SHA-256。此模式不按 hostname 匹配。 | 同一证书可配合新的 DHCP 地址使用，但仍需更新 `--device`。更换或重新签发证书后，即便公钥相同，也不能假定原配对仍匹配。 |

可用下面的本地命令查看证书主题、签发者、有效期和摘要；它不连接设备，也不证明文件来源正确：

```sh
openssl x509 -in ./exoanchor-device.pem -noout \
  -subject -issuer -dates -fingerprint -sha256
```

准备好账号密码及必要的一次性验证码后，使用下面的命令检查实际 HTTPS 连接：

```sh
toolkit/.venv/bin/exoanchor-toolkit network show \
  --transport network --device https://192.0.2.10 \
  --username YOUR_DEVICE_ACCOUNT --timeout 10 --json
```

此命令读取网络运行状态和配置，不修改网络配置；但它可能为完成读取而登录、创建认证会话并消耗一次性输入。
成功后核对返回的设备身份。UDP 发现成功、命令参数解析通过或本地证书检查通过，都不能代替这次连接检查。
设备请求默认绕过系统 HTTP 代理且不跟随重定向；旧 IP 不会通过跳转自动变成新 IP。

## 凭据、一次性码与重试

| 输入 | Toolkit CLI 的来源 |
| --- | --- |
| 用户名 | `--username`，默认 `admin`；填写首次设置时创建的实际账号，没有对应的用户名环境变量。 |
| 密码 | 非空的 `--password` 优先，否则读取 `EXOANCHOR_DEVICE_PASSWORD`。 |
| TOTP 或恢复码 | 非空的 `--otp-code` 优先，否则读取 `EXOANCHOR_OTP_CODE`。Toolkit CLI 不读取 MCP 的 `EXOANCHOR_OTP_CODE_FILE`。 |

当前 CLI 没有 `login`、证书下载子命令或 `--token` 参数，认证随设备请求按需发生。
“enter its username and password” 是错误提示，不表示随后会弹出交互密码输入框。
不要把实际密码、动态码、恢复码或 token 放入共享命令记录；通过环境提供凭据时，也不要把环境内容输出到日志。

客户端在发送登录请求前就清空对象内待发送的 OTP；请求失败也不会保留这份输入供自动重试。
这不表示失败请求必然在设备端消耗了验证码，更不表示父 shell 的环境变量已被清空。
重新执行 CLI 会重新读取环境；重试前必须更新或移除旧的 `EXOANCHOR_OTP_CODE`，
不能反复运行仍携带旧码的命令。曾提供 OTP 的同一客户端需要重新登录时，没有新码会直接提示会话过期。

GET 收到 401 时，仅在已提供密码且允许重试的情况下登录，再重试原 GET 一次；第二次 401 不继续循环。
JSON/binary POST 在有密码且没有 token 时先认证；业务 POST 已发出后收到 401，不会自动登录再重放。
对修改设备的操作，认证错误或超时后应先读取实际状态，再决定是否重试。
OTA 各阶段的判断见 [Release 与恢复指南](TOOLKIT_RELEASE_RECOVERY_zh.md)。

`network show --timeout` 默认 5 秒，控制单次设备 HTTP 请求预算，须为有限正数。
登录的初始提交、等待和状态轮询另共享 30 秒预算，每次登录请求使用普通超时与剩余登录预算中的较小值。
因此增大 `--timeout` 不会延长 30 秒登录预算；一条 CLI 命令还可能包含多个请求，不能把该参数当作整个命令的总时限。
