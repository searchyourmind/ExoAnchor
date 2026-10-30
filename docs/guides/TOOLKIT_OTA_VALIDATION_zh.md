# Toolkit 下载与 OTA 客户端验证

本规程验证主机端的 Release 下载、固件包 manifest 校验、网络 OTA 上传和错误恢复。
测试使用临时目录、合成镜像及绑定 `127.0.0.1` 的 HTTP 服务，不连接 GitHub 或物理设备。
它不证明设备端 Flash 写入、启动确认、回滚、断电恢复或固件自行通过 URL 拉取 manifest 的行为。

## 运行离线回归

以下命令均从仓库根目录开始。先准备 Toolkit 和整仓主机测试依赖：

```bash
uv sync --locked --project toolkit --python 3.12
uv pip install --python toolkit/.venv/bin/python \
  -r device/ESP32P4/firmware/v0.86.6-dev/tests/host/requirements.txt
```

环境准备可能需要下载 Python 和依赖；“离线”指依赖就绪后的回归测试，
不表示首次安装无需网络。仅运行 Toolkit 测试时，完成第一条 `uv sync` 即可；
第二条安装的是整仓主机测试所需的额外依赖。

仅运行下载、上传和 JSON 边界回归：

```bash
(
  cd toolkit &&
  .venv/bin/python -m unittest \
    tests.test_ota_workflow tests.test_json_limits \
    tests.test_firmware tests.test_http_transport_failures \
    tests.test_gui_transport tests.test_release_transport -v
)
```

括号使目录切换只在子 shell 内生效，执行完仍停留在仓库根目录；可继续运行
下面的整仓命令，不会误把解释器路径拼成 `toolkit/toolkit/.venv/bin/python`。

整仓检查从仓库根目录运行；所需 C 编译器、Node.js、OpenSSL 和 `pkg-config`
见[固件验证规程](FIRMWARE_VALIDATION_zh.md)及[传输验证说明](SECURITY_TRANSPORT_zh.md)：

```bash
PYTHON="$PWD/toolkit/.venv/bin/python" \
MCP_PYTHON="$PWD/toolkit/.venv/bin/python" \
TOOLKIT_PYTHON="$PWD/toolkit/.venv/bin/python" \
  ./scripts/check-all.sh
```

Python 兼容性变更应分别使用 3.10、3.11、3.12 运行 Toolkit 全部测试和
`device/ESP32P4/firmware/security-tests/test_tls_clients.py`。后者建立真实的本地
TLS 连接，检查 Toolkit、MCP 和诊断客户端只向通过证书校验的对端发送凭据。
准备环境时可替换上述 `--python` 版本；依赖安装完成后测试本身无需外网。

## 验证内容

| 路径 | 核对结果 |
| --- | --- |
| Release → ZIP → manifest → OTA | 下载器计算归档 SHA-256，仅在 Release asset 提供 digest 时与其比较；各 segment 始终按 manifest 核对长度与 SHA-256。完整上传的字节与合成镜像一致；三轮上传均独立完成；再次下载复用经过复验的缓存。 |
| 归档或 segment 损坏 | 返回明确错误，临时下载被清理，不发布可用缓存，也不开始 OTA。 |
| 校验后文件变化 | 在任何设备请求前，重新核对实际上传字节的长度与 SHA-256；文件增长、截断或等长修改均停止。 |
| 部分写入或整块响应丢失 | 通过真实 socket 断连模拟；先查询状态，只从同一 session、大小、摘要对应的已确认 offset 继续，不能盲目重发原块。 |
| 断连且没有进展 | 当前调用失败并停止，不无限重试；之后的新调用可查询并续传匹配的活动会话。 |
| 会话或板型不匹配 | 不开始新上传、不覆盖其他活动会话、不自动 abort。 |
| 状态字段不合法 | 拒绝布尔值冒充 offset、空或非字符串 session ID、无进展或越界的 offset、非正整数 chunk size。 |
| 新会话分块限制 | 使用 start 响应给出的 chunk size，同时维持客户端每块最多 64 KiB 的限制。 |
| finish 或 reboot 拒绝 | finish 必须同时返回 `ok: true` 和匹配的 SHA-256 才能请求重启；reboot 未确认时不能报告请求成功。 |
| Release 重定向 | HTTP 308 在 Python 3.10 上也能处理；跨 origin 跳转移除凭据，HTTPS 不能降级为 HTTP。 |

JSON 接收入口在解析后、交给业务代码前，以非递归遍历限制为最多 64 层对象/数组。
根对象或数组计一层，字符串里的括号不计层数。限制覆盖 Toolkit 和 MCP 的设备响应、
MCP stdio 消息、GUI 请求、Release 元数据和固件 manifest；本地 manifest 及共用读取
函数处理的构建 JSON 文件最多 1 MiB。
各 HTTP 入口保留各自原有的字节限制，解释器自身的语法、整数长度或递归异常仍会转换为
对应 API 的错误。该策略不依赖某一 Python 版本恰好抛出 `RecursionError`，也不修改全局递归上限。

## 设备 HTTP 失败的定向排查

设备 API 与 Release 下载使用不同的跳转规则，登录总期限也不等同于一次
HTTP 请求的期限。遇到以下问题时，先按对应分支定位：

| 现象 | 当前客户端行为与对应回归 |
| --- | --- |
| 设备 API 返回 301、302、303、307 或 308 | 不跟随跳转，即使目标是同一 origin；不会向跳转地址转发凭据或重放请求。核对显式设备地址及代理配置，见 `tests.test_http_targets`。Release 下载允许受限跳转的结果不能推定到设备 API。 |
| POST 返回 401 | 不在重新登录后自动重放写请求；先检查设备状态再决定后续操作。GET 可在重新认证后重试一次，见已列入首组命令的 `tests.test_http_transport_failures`。 |
| 异步登录超时 | 30 秒总预算包含初始提交、轮询和等待；增大单次请求的 `timeout` 不会延长登录总预算。超时后到达的成功响应不能替换原 token，见 `tests.test_http_login_budget`。 |
| 已收到响应头，但正文停顿或长度不符 | 正文读取仍受本次请求期限和大小限制；持续少量字节不能无限延长期限。声明了 `Content-Length` 的响应若被截断，即使已收到的内容是合法 JSON 也会被拒绝，见 `tests.test_http_response_deadline`。 |

在仓库根目录运行额外的目标地址、登录预算与响应正文回归：

```bash
(
  cd toolkit &&
  .venv/bin/python -m unittest \
    tests.test_http_targets tests.test_http_login_budget \
    tests.test_http_response_deadline -v
)
```

这组测试使用本地 HTTP 服务或模拟时钟，不需要设备凭据。记录实际使用的
Python 版本、模块列表和结果；某一个解释器上的通过记录不能替代 3.10、3.11、
3.12 的逐版本兼容性验证，也不能写成整仓检查通过。

## 本地固件包校验

准备好 Toolkit 环境后，可在仓库根目录运行下列命令。将示例路径替换成实际
下载或构建得到的包；三种输入任选一种，无需全部执行：

```bash
toolkit/.venv/bin/exoanchor-toolkit firmware verify ./firmware-package
toolkit/.venv/bin/exoanchor-toolkit firmware verify ./firmware-package/exoanchor-firmware.json
toolkit/.venv/bin/exoanchor-toolkit firmware verify ./firmware-package.zip --json
```

`package` 是位置参数，支持包目录、manifest 文件或 ZIP；不使用 `--manifest`。
目录中的 manifest 必须名为 `exoanchor-firmware.json`，ZIP 中必须恰有一个该文件。
这些命令只读取本地包，ZIP 会解压到临时目录；不发现、登录、烧录或重启设备。

普通输出中的 `Verification: PASS` 表示包校验成功；`--json` 输出包信息而不打印
该文字，应同时记录退出码。JSON 包含版本、板型和各 segment 的长度、SHA-256。
ZIP 输出中的 `manifest` 是临时解压路径，命令结束后即被清理；复现时应保留
原 ZIP 路径和文件本身，不能把临时路径当成下次校验的输入。

本地 `verify` 核对文件与包内 manifest 的一致性，不验证发布者的数字签名，
也不证明镜像与已连接的实物匹配。校验成功不能代替可信来源确认或真机启动验证。

## 中断后的排查顺序

1. 保留原始固件包并重新运行 Toolkit 的 `firmware verify`。文件不再匹配 manifest 时，
   先恢复或重新获取可信包，不继续上传。
2. 确认目标身份、板型和配对证书后，检查 `/api/ota/upload/status` 的
   `active`、`session_id`、`size`、`sha256`、`offset`。日志中不记录密码、token 或 Cookie。
3. 若活动会话对应同一包，重新执行原上传操作会从报告的 offset 恢复；状态不匹配时停止，
   由操作者确认现有会话用途后再决定是否显式取消。客户端不会替操作者取消其他上传。
4. finish 或 reboot 的响应丢失时，不把“不知道结果”记为“操作未发生”，也不自动重放。
   先检查设备状态。`--no-reboot` 只表示镜像待激活；“Device reboot requested”只表示
   请求获确认，均不代表新镜像已成功启动。

保存源码版本、Python 版本、命令、退出码和完整日志。真实设备的启动、回滚及断电验证
应单独记录，并遵循[固件验证规程](FIRMWARE_VALIDATION_zh.md)的设备身份与恢复条件。
