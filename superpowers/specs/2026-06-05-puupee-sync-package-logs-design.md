# puupee_sync Package 自定义日志迁移设计

## 背景

第一阶段已经在 `packages/puupee_utilities/lib/talker.dart` 中建立 `PuupeeTalkerLog` 与 `PuupeeLogger`，并将 `packages/puupee_sync/lib/src/sync_server.dart` 迁移到 `sync-server` 自定义 Talker key。这样可以在现有 `TalkerScreen` 中单独筛选同步服务器日志。

`puupee_sync` 其它核心类仍大量直接使用全局 `talker` 或 `Talker` 类型，例如 `sync_node_manager.dart`、`lan_sync_node_discoverer.dart`、`sync_client_manager.dart`、`storage/*`、`transfer_manager.dart`、`auth_strategies.dart`、`pairing_auth.dart`、`content_events.dart` 和 `mcp_server.dart`。这些日志仍落入普通等级 key，导致 Talker UI 无法按同步子系统筛选。

## 目标

- 将 `puupee_sync/lib` 内业务日志统一迁移到 `PuupeeLogger`。
- 允许用户在 Talker UI 中按同步子系统筛选日志，例如 `sync-node`、`sync-client`、`sync-storage`。
- 保留现有日志文本、等级、异常和堆栈语义。
- 给每条迁移后的日志补充 scene，便于在模块筛选后继续搜索。
- 避免每个类都创建 Talker key，控制筛选项数量。

## 非目标

- 不重写 Talker UI。
- 不重构同步业务逻辑、网络逻辑、存储逻辑或数据库逻辑。
- 不迁移测试文件中为了检查 Talker 行为而直接创建 `Talker` 的用法。
- 不引入新日志依赖。
- 不将 token、密钥、认证 header 或请求体敏感内容新增到日志中。

## 模块划分

采用“子系统 key + scene 标签”的结构。

- `sync-server`：同步服务器，第一阶段已完成。
- `sync-node`：节点发现、节点配置、连接质量和节点模型。
- `sync-client`：同步客户端、客户端管理和同步 API 请求。
- `sync-storage`：存储器、文件传输适配、局域网/云端/本地存储。
- `sync-transfer`：上传下载任务队列和 TransferTask 调度。
- `sync-auth`：JWT、API key、配对 token 和配对服务。
- `sync-crdt`：CRDT、Postgres CRDT 和内容事件。
- `sync-mcp`：MCP server。

## 场景标签

复用第一阶段已有 scene，并补充第二阶段需要的通用 scene：

- `lifecycle`：启动、停止、初始化、关闭。
- `discovery`：mDNS 注册、发现、节点变更通知。
- `config`：节点配置、保存配置、最后同步时间。
- `network`：HTTP/API 请求、连接质量、下载、上传前的连接检查。
- `storage`：存储器创建、上传、下载、删除、统计。
- `transfer`：上传下载任务、队列、重试、进度。
- `auth`：JWT、API key、认证服务器验证。
- `pairing`：配对 token、公钥解析、配对认证。
- `crdt`：CRDT merge、changeset、Postgres CRDT。
- `content-event`：内容事件处理。
- `mcp`：MCP 连接和命令。
- `cache`：缓存读写、缓存公钥、清理。

## API 变更

在 `PuupeeLogModule` 中补齐第二阶段模块 key，并在 `registerPuupeeTalkerKeys` 中注册这些 key。`PuupeeLogger` 增加对应工厂：

- `PuupeeLogger.syncNode()`
- `PuupeeLogger.syncClient()`
- `PuupeeLogger.syncStorage()`
- `PuupeeLogger.syncTransfer()`
- `PuupeeLogger.syncAuth()`
- `PuupeeLogger.syncCrdt()`
- `PuupeeLogger.syncMcp()`

保留现有 `PuupeeLogger.syncServer()`，避免破坏第一阶段行为。

## 迁移策略

迁移时按文件职责选择模块：

- 节点类使用 `PuupeeLogger.syncNode()`。
- 客户端类使用 `PuupeeLogger.syncClient()`。
- 存储类使用 `PuupeeLogger.syncStorage()`。
- 任务调度类 `transfer_manager.dart` 使用 `PuupeeLogger.syncTransfer()`。
- 认证与配对类使用 `PuupeeLogger.syncAuth()`。
- CRDT 和内容事件类使用 `PuupeeLogger.syncCrdt()`。
- MCP 类使用 `PuupeeLogger.syncMcp()`。

原来 `logger.error('msg', e, stackTrace)` 这类 Talker 位置参数调用改为 `logger.error('msg', exception: e, stackTrace: stackTrace)`。只有确实没有堆栈的调用传 `exception`，不为了迁移额外捕获堆栈。

## 验收标准

- `puupee_sync/lib` 中除 `sync_server.dart` 既有自定义 logger 外，不再有业务代码直接依赖 `package:talker/talker.dart` 或全局 `talker.*` 输出日志。
- 所有迁移后的日志都通过 `PuupeeLogger` 写入自定义 key。
- 迁移后的日志调用都带有 `scene:`。
- `PuupeeLogger` 测试覆盖所有新模块 key 的注册与工厂。
- `puupee_sync` 新增日志迁移测试，覆盖至少 `sync-node`、`sync-client`、`sync-storage`、`sync-transfer`、`sync-auth`、`sync-crdt`、`sync-mcp`。
- 相关 `dart test` 与 `dart analyze` 通过。

## 风险

- `puupee_sync` 日志调用数量多，机械迁移容易漏掉位置参数异常。通过搜索和测试检查兜底。
- 部分文件通过 `ManagerInterface` 暴露 `logger` getter。该接口需要从 `Talker` 改为 `PuupeeLogger`，实现类同步调整。
- `sync_client_manager.dart` 当前使用独立 `Talker()`，迁移后会进入全局 Talker；这是符合统一筛选目标的行为变化。
