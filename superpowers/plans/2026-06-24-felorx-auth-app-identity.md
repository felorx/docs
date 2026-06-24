# Felorx Auth 应用身份迁移计划

## 目标

- 将 Flutter 包名 `puupee_auth` 迁移为 `felorx_auth`。
- 将英文应用显示名 `Puupee Auth` 迁移为 `Felorx Auth`。
- 将应用 bundle id 从 `com.puupee.auth` 迁移为 `com.felorx.auth`，包含 dev/debug/test 变体。
- 更新构建、文档、桌面 runner、installer、测试 imports、snap desktop、manifest 和脚本中的 Auth 应用身份。

## 非目标

- 不移动 `apps/auth` 目录；`auth` 已是单数应用 slug。
- 不在本步骤修改 Auth 业务模型、数据库模型、同步模型或服务端内容类型。
- 不修改通用 `puupee_shared`、`puupee_ui`、`puupee_utilities` 依赖包名。

## 步骤

1. 更新 `apps/auth` 内包名、imports、manifest、desktop runner、installer、bundle id、URL scheme 和英文显示名。
2. 更新构建脚本、snap desktop、文档和相关测试引用。
3. 重新生成路由/Provider 相关输出，对应用测试、官网类型检查和旧身份残留扫描进行验证。
4. 分别提交 docs 计划与顶层 Auth 应用身份迁移。
