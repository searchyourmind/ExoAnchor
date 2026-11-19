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
