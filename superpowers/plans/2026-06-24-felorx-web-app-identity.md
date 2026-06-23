# Felorx Web 应用身份迁移计划

## 目标

- 将 Flutter 包名 `puupee_web` 迁移为 `felorx_web`。
- 将英文应用显示名 `Puupee Web` 迁移为 `Felorx Web`。
- 将应用 bundle id 从 `com.puupee.web` 迁移为 `com.felorx.web`，包含 dev/debug/test 变体。
- 更新构建、文档、官网数据、桌面 runner、installer、测试 imports 和脚本中的 Web 应用身份。

## 非目标

- 不移动 `apps/web` 目录；`web` 已是单数应用 slug。
- 不把 Next.js `website` 工程本身当作本步骤的 Web 子应用改造对象。
- 不在本步骤修改 Web 业务模型、数据库模型、同步模型或服务端内容类型。
- 不修改通用 `puupee_shared`、`puupee_ui`、`puupee_utilities` 依赖包名。

## 步骤

1. 更新 `apps/web` 内包名、imports、manifest、desktop runner、installer、bundle id、URL scheme 和英文显示名。
2. 更新官网应用数据、构建脚本、Codemagic 文档和相关模板引用。
3. 重新生成路由/Provider 相关输出，对应用测试、官网类型检查和旧身份残留扫描进行验证。
4. 分别提交 docs 计划、website 数据与顶层 Web 应用身份迁移。
