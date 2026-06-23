# Felorx Photo 应用身份迁移计划

## 目标

- 将 `apps/photos` 子应用目录迁移为 `apps/photo`。
- 将 Flutter 包名 `puupee_photos` 迁移为 `felorx_photo`。
- 将英文应用显示名 `Puupee Photos` 迁移为 `Felorx Photo`。
- 将应用 bundle id 从 `com.puupee.photos` 迁移为 `com.felorx.photo`，包含 dev/debug/test 变体。
- 将构建、文档、官网数据和路由里的应用 slug 从 `photos` 迁移为 `photo`。

## 非目标

- 不在本步骤修改照片、相册、人脸、地点等业务模型。
- 不在本步骤修改 `Products.photos`、数据库模型、同步模型或服务端内容类型；这些归入后续 Felorx 模型迁移。
- 不强行修改中文自然语言“照片”或普通业务变量名 `photos`。

## 步骤

1. 移动 `apps/photos` 到 `apps/photo`，更新 workspace、builder、脚本、文档和官网数据引用。
2. 更新 Flutter/Dart 包名、imports、manifest、desktop runner、installer、bundle id 和英文显示名。
3. 重新生成路由/Provider 相关输出，对应用、官网类型检查和旧身份残留扫描进行验证。
4. 分别提交 docs 计划、website 数据与顶层 Photo 应用身份迁移。
