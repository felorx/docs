# Felorx Sync Node Panel 应用身份迁移计划

## 目标

- 将 Flutter 包名 `puupee_sync_node_panel` 迁移为 `felorx_sync_node_panel`。
- 将英文应用显示名 `Puupee Sync Node Panel` 迁移为 `Felorx Sync Node Panel`。
- 将应用 bundle id 从 `com.puupee.sync_node_panel` / `com.puupee.sync-node-panel` 迁移为 `com.felorx.sync_node_panel` / `com.felorx.sync-node-panel`，包含 dev/debug/test 变体。
- 更新 Android、iOS、macOS、Linux、Windows 配置、文档、路由 imports、URL scheme 和默认 client id。

## 非目标

- 不移动 `apps/sync_node_panel` 目录；`sync_node_panel` 已是单数应用 slug。
- 不重命名 `PuupeeSyncServer`、`PuupeeSharedLocalizations` 等共享类型名。
- 不重命名 `puupee_shared`、`puupee_ui`、`puupee_utilities` 等共享依赖包。

## 步骤

1. 更新 `apps/sync_node_panel` 内包名、imports、manifest、desktop runner、bundle id、URL scheme 和英文显示名。
2. 重新生成路由/l10n 相关输出，并保持对 `felorx_sync_node` 的依赖。
3. 对应用分析、官网类型检查和旧身份残留扫描进行验证。
4. 分别提交 docs 计划与顶层 Sync Node Panel 应用身份迁移。
