# Telemetry 内核模块

本目录对应 `src/kernel/telemetry`，提供本地遥测事件收集与云端遥测所需的身份状态基础能力。app 层的云端发送器负责把受控的聚合窗口发送出去；本模块不负责 HTTP 传输。

## 模块边界

- `TelemetryConfig` 控制通用本地遥测收集，默认关闭。
- 云端身份状态由 `cloud/identity.py` 管理，包括稳定客户端实例标识、同意状态与安装凭证元数据。
- 同意状态为 `unknown`、`granted` 或 `revoked`；云端客户端仅在状态为 `granted` 时才会采集或发送。
- 身份状态使用 kernel 的 `JSONStore` 持久化；遥测待发送窗口由 app 层客户端保留在内存中，不写入本地持久化队列。

## 与 app 层的关系

- app 层装配与运行行为见 [app/cloud_telemetry](../app/cloud_telemetry.md)。
- CoreConfig 中的 `cloud_telemetry` 配置节决定客户端启用开关、身份状态目录、待发送队列上限及发送时间参数。

## 代码映射

- 本地遥测配置：`src/kernel/telemetry/config.py`
- 云端身份状态：`src/kernel/telemetry/cloud/identity.py`
- app 层云端客户端：`src/app/cloud_telemetry/client.py`

## 维护约束

- 不要把 app 层的 HTTP、后台任务或运行时装配逻辑写入本模块的 API 契约。
- 不要将身份状态中可能出现的凭证值写入文档、示例或排障输出。
