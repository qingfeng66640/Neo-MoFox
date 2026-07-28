# 云端遥测客户端

本文档对应 `src/app/cloud_telemetry`：app 层对本地身份、待发送窗口队列和云端遥测客户端的装配入口。该模块只装配 app 层客户端；通用遥测与 LLM 统计仍分别由 kernel 层初始化。

## 公开入口

从 `src.app.cloud_telemetry` 可导入以下公开对象：

- `CloudTelemetryFoundation`：本地基础设施装配结果，包含 `core_config` 与 `identity_store`。
- `CloudTelemetryClient`、`CloudTelemetryClientConfig`：客户端及其配置对象。
- `CloudTelemetryPendingQueue`、`CloudTelemetryQueueState`：待发送窗口队列及其状态。
- `build_cloud_telemetry_identity_store(core_config=None)`：按 `CoreConfig.cloud_telemetry.identity_storage_dir` 构建本地身份存储。
- `initialize_cloud_telemetry_foundation(core_config=None)`：装配并返回仅含本地身份服务的 `CloudTelemetryFoundation`。
- `initialize_cloud_telemetry_runtime(foundation=None)`：装配全局 runtime 与客户端，并返回 foundation；参数可为 `CloudTelemetryFoundation`、`CoreConfig` 或 `None`。
- `get_cloud_telemetry_runtime()`、`get_cloud_telemetry_client()`：获取当前全局 foundation 或客户端；未初始化时返回 `None`。
- `get_cloud_telemetry_status_summary()`：异步返回本地状态摘要。
- `close_cloud_telemetry_runtime()`：停止已启动的客户端循环并清除全局 runtime 引用。

## 启用前提

`CoreConfig.cloud_telemetry` 提供客户端开关、本地身份状态目录、队列容量、默认心跳间隔和发送超时设置。

客户端的有效发送状态同时依赖以下条件：

1. `cloud_telemetry.client_enabled` 为真；
2. 本地身份状态的同意状态为 `granted`。

运行时初始化仍会在未同意或关闭客户端开关时创建本地 foundation 和客户端对象，但客户端的 `enabled` 状态为假，`Bot.run()` 不会启动后台发送循环。身份状态由配置的本地身份目录维护。

## Bot 生命周期中的装配

`Bot._initialize_kernel()` 在通用遥测、LLM 统计及其他 kernel 服务初始化后调用 `initialize_cloud_telemetry_runtime(self.config)`。因此，云端遥测 runtime 在 core 组件和插件加载前完成本地装配。

进入 `Bot.run()` 时，运行时取得全局客户端；仅当客户端存在且 `enabled` 为真时调用 `CloudTelemetryClient.start()`。后台循环通过 task manager 创建守护任务，周期性采集本地遥测与 LLM 统计窗口，并执行注册或批量心跳发送。

`Bot.shutdown()` 在关闭 LLM 统计数据库前调用 `close_cloud_telemetry_runtime()`。该调用会停止客户端后台循环，并清除全局 client 与 foundation 引用。

## 状态摘要

`get_cloud_telemetry_status_summary()` 可在初始化前后调用。未初始化时返回稳定的默认摘要，例如：

- `initialized` 为 `false`；
- `client_enabled` 为 `false`；
- `client_instance_id` 为 `null`；
- `credential_present` 为 `false`；
- 待发送窗口数与待发送字节数为 `0`；
- `loop_running` 为 `false`。

初始化后，摘要还反映本地身份及队列状态，包括：

- 同意状态与 IP 保留选择；
- 安装凭据是否存在、上次注册时间和凭据过期时间；
- 待发送窗口数、字节数、上次发送状态、错误和发送时间；
- 下次心跳间隔、后台循环是否运行；
- 实例状态、最近拒绝原因和最近窗口处理结果。

状态摘要只报告状态，不包含安装凭据值。

## 发送结果、暂停与队列行为

`CloudTelemetryClient.run_once()` 的正常返回值包含 `ok` 与 `reason`。当前实现中：

- 未获同意、客户端未启用、已记录实例暂停、注册后仍未取得安装凭据、安装凭据无效时，分别返回 `consent_not_granted`、`client_disabled`、`instance_suspended`、`registration_failed`、`invalid_install_credential`；
- 没有待发送窗口时返回 `{"ok": true, "reason": "idle"}`；
- 批量心跳处理完成时返回 `{"ok": true, "reason": "sent"}`，并附带 `accepted`、`duplicate` 与 `rejected` 计数。

这些 `reason` 是当前 `run_once()` 的业务结果，不是远端协议的永久枚举。注册、HTTP 状态校验、响应解析或网络传输异常会直接向调用方抛出；后台循环会捕获此类异常，将错误写入 `last_send_error`，并在下一个心跳间隔继续尝试，不会直接终止 Bot 主运行循环。

服务端在批量心跳响应中首次报告 `suspended` 时，当轮仍按 `sent` 完成，并在结果中携带 `instance_status: "suspended"`；客户端同时记录实例状态与逐窗口拒绝原因。后台循环会先等待下一个心跳间隔，后续迭代发现已记录的暂停状态后，`run_once()` 返回 `instance_suspended` 并停止循环，此后不再采集或继续轮询。

待发送窗口和队列状态只保留在当前进程内，进程退出或 runtime 重建后不会恢复。队列受最大窗口数和最大字节数约束，入队后会从最旧窗口开始修剪；单个新窗口本身超过字节限制时，也可能不会被保留。服务端逐窗口确认状态为 `accepted`、`duplicate` 或 `rejected_permanent` 时，对应窗口会出队；`rejected_retryable` 会保留以供后续发送。

状态摘要中的 `last_rejection_reason` 仅表示最近一次服务端逐窗口拒绝原因；注册、HTTP、网络等发送异常应查看 `last_send_error`。`instance_status`、`last_window_results`、待发送窗口数和字节数可与这两个字段结合用于排查发送状态。

## 相关实现与测试

- 实现：`src/app/cloud_telemetry/__init__.py`、`src/app/cloud_telemetry/bootstrap.py`、`src/app/cloud_telemetry/runtime.py`、`src/app/cloud_telemetry/client.py`
- runtime 与状态测试：`test/app/test_cloud_telemetry_runtime.py`
- foundation 测试：`test/app/test_cloud_telemetry_bootstrap.py`
- 客户端队列、发送与后台循环测试：`test/app/test_cloud_telemetry_client.py`
- 后端集成与暂停状态测试：`test/app/test_cloud_telemetry_backend_integration.py`
- Bot 生命周期调用点：`src/app/runtime/bot.py`
