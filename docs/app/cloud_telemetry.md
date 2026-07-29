# 云端遥测客户端

本文档对应 `src/app/cloud_telemetry`。该模块位于 app 层：它装配 kernel 提供的身份存储和本地遥测能力，并在 Bot 运行期间管理云端遥测发送循环。

## 启用条件

初始化云端遥测运行时时，客户端只有同时满足以下条件才会实际启用：

1. `cloud_telemetry.client_enabled` 为真；
2. 本地身份状态的同意状态为 `granted`。

首次初始化会建立本地身份状态；其默认同意状态为 `unknown`，因此即使 `client_enabled` 的默认值为真，客户端也不会发送数据，直到同意状态被明确设为 `granted`。`revoked` 同样不会发送。

## 生命周期

- Bot 的 kernel 初始化末段会初始化云端遥测运行时。
- `Bot.run()` 启动后，仅在客户端实际启用时才启动后台发送循环。
- `Bot.shutdown()` 会停止该循环并清理全局运行时引用。

运行时可通过 `get_cloud_telemetry_status_summary()` 获取本地状态摘要。摘要包含初始化状态、同意状态、客户端是否启用、待发送窗口计数和字节数，以及最近一次发送状态；不要将其中可能出现的错误文本或身份相关信息作为凭证展示。

## 数据与传输边界

客户端按心跳窗口采集以下聚合信息：

- kernel telemetry 的窗口摘要与域聚合；
- LLM 统计摘要及请求名聚合；
- WatchDog、任务管理器和流循环管理器的运行状态统计；
- 仅限 warning、error、critical 级别的受控诊断事件。

诊断事件的 `summary` 会保留前 200 个字符，并在原值超长时追加省略号；属性仅保留 `domain` 和 `entity_id`。待发送窗口仅保留在进程内存中，并由最大窗口数和总字节数限制；达到限制时，最旧窗口会被丢弃。身份状态则独立保存在 `cloud_telemetry.identity_storage_dir` 指定的目录中。

发送前客户端会注册安装实例，之后以批量心跳方式发送窗口。收到无效安装凭证响应时，客户端会清除本地安装凭证，并在后续发送时重新注册。若服务端将实例标为 `suspended`，客户端将停止采集并终止后台发送循环。

## 配置

`CoreConfig.cloud_telemetry` 提供下列配置项：

- `client_enabled`：是否允许客户端发送能力，默认 `true`；仍须取得明确同意才会实际启用。
- `identity_storage_dir`：本地身份状态目录，默认 `data/cloud_telemetry/state`。
- `pending_queue_max_bytes`：内存待发送窗口总字节上限，默认 `524288`。
- `pending_queue_max_windows`：内存待发送窗口数量上限，默认 `128`。
- `default_heartbeat_interval_seconds`：默认心跳间隔，默认 `300.0` 秒；服务端响应可更新后续间隔。
- `send_timeout_seconds`：单次发送超时，默认 `10.0` 秒。

HTTP 客户端的环境信任行为复用 `CoreConfig.advanced.trust_env`。

## 代码映射

- 运行时装配与状态摘要：`src/app/cloud_telemetry/runtime.py`
- 发送队列与客户端：`src/app/cloud_telemetry/client.py`
- 身份存储装配：`src/app/cloud_telemetry/bootstrap.py`
- Bot 生命周期接入：`src/app/runtime/bot.py`
- 同意状态与身份存储：`src/kernel/telemetry/cloud/identity.py`

## 相关文档

- [app runtime](./runtime.md)
- [core 配置](../core/config/core_config.md)
- [telemetry 内核模块](../telemetry/README.md)
