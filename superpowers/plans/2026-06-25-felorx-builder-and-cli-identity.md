# Felorx Builder 与 Builder CLI 迁移计划

## 目标

- 将 Flutter 应用包名 `puupee_builder` 迁移为 `felorx_builder`。
- 将工具包目录与包名 `packages/tools/puupee_builder_cli` 迁移为 `packages/tools/felorx_builder_cli`。
- 将 CLI bin 文件、可执行名和默认命令从 `puupee-builder` / `puupee_builder_cli` 迁移为 `felorx-builder` / `felorx_builder_cli`。
- 将 Builder 的显示名、bundle id、URL scheme 和平台配置迁移到 Felorx。

## 非目标

- 不在本批次重命名 `puupee_cli`、`felorx_api_client`、`puupee_shared` 等其他共享包。
- 不改变构建业务逻辑，只同步标识、路径、导入和测试期望。
- 不迁移历史设计文档中的旧路径，除非它们是当前命令或 workspace 配置。

## 步骤

1. 更新顶层 workspace、install 脚本、构建脚本中的 Builder CLI 路径和可执行名。
2. 迁移 `apps/builder` 的包名、imports、平台标识、默认 CLI 路径和文案。
3. 迁移 `packages/tools/puupee_builder_cli` 目录、pubspec、bin 文件、imports、测试和 README。
4. 移动 Android package 路径到 `com/felorx/builder`。
5. 重新生成 Builder 应用代码，运行应用和 CLI 的分析/测试以及旧身份扫描。
6. 分别提交 docs 计划与顶层 Builder/CLI 迁移。
