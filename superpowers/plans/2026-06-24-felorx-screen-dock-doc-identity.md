# Felorx Screen Dock 文档身份迁移计划

## 目标

- 将 `apps/screen_dock/dev/PRD.md` 中的 `Puupee Screen Dock` 迁移为 `Felorx Screen Dock`。
- 将规划包名从 `puupee_screen_dock` / `com.puupee.screen_dock` 迁移为 `felorx_screen_dock` / `com.felorx.screen_dock`。
- 将规划中的投屏协议、Sender SDK、Cast、Relay、Sync 等生态命名迁移到 Felorx。

## 非目标

- 不创建 `apps/screen_dock` Flutter 工程；当前目录只有 PRD。
- 不重命名共享包或 CLI 命令。

## 步骤

1. 更新 PRD 内品牌、包名、bundle id、协议发现名和 well-known 路径。
2. 运行旧标识扫描与 diff 空白检查。
3. 分别提交 docs 计划与顶层 Screen Dock 文档身份迁移。
