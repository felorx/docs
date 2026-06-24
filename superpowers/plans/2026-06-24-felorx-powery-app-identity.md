# Felorx Powery 应用身份迁移计划

## 目标

- 将 Flutter 包名 `puupee_powery` 迁移为 `felorx_powery`。
- 将英文应用显示名 `Puupee Powery` 迁移为 `Felorx Powery`。
- 将 URL scheme 从 `puupeepowery` 迁移为 `felorxpowery`。
- 将 bundle id、MethodChannel、macOS privileged helper 标识从 `com.puupee.powery` 迁移为 `com.felorx.powery`。

## 非目标

- 不移动 `apps/powery` 目录；`powery` 已是单数应用 slug。
- 不重命名 `puupee_shared`、`puupee_ui`、`puupee_utilities` 等共享依赖包。
- 不改动 `puupee-image`、`PuupeeSyncServer` 等平台级模型/服务名。

## 步骤

1. 更新 `apps/powery` 内包名、imports、desktop runner、URL scheme、默认 client id 和英文显示名。
2. 同步更新 macOS helper 的 bundle id、Mach service、SMC channel 与 plist 文件名引用。
3. 重新生成路由/Provider 相关输出。
4. 对应用分析、测试、官网类型检查和旧身份残留扫描进行验证。
5. 分别提交 docs 计划与顶层 Powery 应用身份迁移。
