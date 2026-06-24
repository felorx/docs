# Felorx DevPanel 应用身份迁移计划

## 目标

- 将 Flutter 包名 `puupee_devpanel` 迁移为 `felorx_devpanel`。
- 将英文应用显示名 `Puupee DevPanel` 迁移为 `Felorx DevPanel`。
- 将 URL scheme 从 `puupeedevpanel` 迁移为 `felorxdevpanel`。
- 更新 macOS、Windows 配置、路由 imports、默认 client id 和用户可见文案。

## 非目标

- 不移动 `apps/devpanel` 目录；`devpanel` 已是单数应用 slug。
- 不修改本地仓库路径探测逻辑中对 `puupees/puupee-apps` 目录名的兼容。
- 不重命名 `puupee_shared`、`puupee_ui` 等共享依赖包。

## 步骤

1. 更新 `apps/devpanel` 内包名、imports、desktop runner、URL scheme、默认 client id 和英文显示名。
2. 重新生成路由/Provider 相关输出。
3. 对应用分析、官网类型检查和旧身份残留扫描进行验证。
4. 分别提交 docs 计划与顶层 DevPanel 应用身份迁移。
