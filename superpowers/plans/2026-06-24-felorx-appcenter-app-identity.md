# Felorx AppCenter 应用身份迁移计划

## 目标

- 将 Flutter 包名 `puupee_appcenter` 迁移为 `felorx_appcenter`。
- 将英文应用显示名 `Puupee AppCenter` 迁移为 `Felorx AppCenter`。
- 将应用 bundle id 从 `com.puupee.appcenter` 迁移为 `com.felorx.appcenter`，包含 dev/debug/test 变体。
- 更新构建、文档、桌面 runner、测试 imports、manifest 和 URL scheme 中的 AppCenter 应用身份。

## 非目标

- 不移动 `apps/appcenter` 目录；`appcenter` 已是单数应用 slug。
- 不在本步骤修改通用共享包、CLI、服务端发布系统或 `puupee_shared` 等依赖名。
- 不改应用中心查询的历史/外部包名清单，除非它们直接表示 AppCenter 自身身份。

## 步骤

1. 更新 `apps/appcenter` 内包名、imports、manifest、desktop runner、bundle id、URL scheme 和英文显示名。
2. 更新 AppCenter 文档和用户可见文案中的 Felorx 品牌表达。
3. 重新生成路由相关输出，对应用分析、官网类型检查和旧身份残留扫描进行验证。
4. 分别提交 docs 计划与顶层 AppCenter 应用身份迁移。
