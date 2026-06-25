# Felorx Sync Node 包身份迁移计划

## 目标

- 将 Dart 包名 `puupee_sync_node` 迁移为 `felorx_sync_node`。
- 将英文应用/服务名 `Puupee Sync Node` 迁移为 `Felorx Sync Node`。
- 将默认客户端 ID `Puupee_Sync_Node` 迁移为 `Felorx_Sync_Node`。
- 更新 CLI 入口、测试 imports、文档和默认参数中的 Sync Node 身份。

## 非目标

- 不移动 `apps/sync_node` 目录；`sync_node` 已是单数服务 slug。
- 不在本步骤重命名 `PuupeeSyncServer`、`PuupeeDbConfig`、`FelorxApiClient` 等共享同步/API 类型。
- 不重命名 `puupee_connect`、`puupee_shared`、`puupee_sync` 等共享依赖包。

## 步骤

1. 更新 `apps/sync_node` 内包名、imports、默认节点名、默认 client id 和英文品牌文案。
2. 保留共享同步包类型名，避免与后续横切包迁移混淆。
3. 对包分析、包测试、官网类型检查和旧身份残留扫描进行验证。
4. 分别提交 docs 计划与顶层 Sync Node 包身份迁移。
