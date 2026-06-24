# Felorx Asset Kiln 应用身份迁移计划

## 目标

- 将 Flutter 包名 `puupee_asset_kiln` 迁移为 `felorx_asset_kiln`。
- 将 URL scheme、默认 client id 和生成配置中的 Puupee 前缀迁移为 Felorx。
- 将 Android、iOS、macOS、Linux 配置中的应用标识改为 `com.felorx.asset_kiln` 前缀。

## 非目标

- 不移动 `apps/asset_kiln` 目录；`asset_kiln` 已是单数应用 slug。
- 不重命名 `puupee_shared`、`puupee_ui` 等共享依赖包。
- 不调整未迁移的 CLI 命令名或共享模型名。

## 步骤

1. 更新 `apps/asset_kiln` 内包名、imports、mobile/desktop runner、URL scheme 和默认 client id。
2. 将 Android Kotlin package 路径从 `com/puupee/asset_kiln` 移到 `com/felorx/asset_kiln`。
3. 重新生成路由/Provider 相关输出。
4. 对应用分析、测试、官网类型检查和旧身份残留扫描进行验证。
5. 分别提交 docs 计划与顶层 Asset Kiln 应用身份迁移。
