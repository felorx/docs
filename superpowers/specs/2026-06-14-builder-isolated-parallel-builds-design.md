# Builder 独立并行构建设计

## 背景

Builder 当前有两条构建入口：

- 桌面 GUI：`apps/builder/lib/services/build_service.dart` 已按单个制品创建任务，但构建 workspace 仍按项目根目录复用。
- CLI：`packages/tools/puupee_builder_cli` 的 `build` 和 `build-all` 已有并发参数，但任务粒度主要按应用划分；同一应用内多个平台或多个制品仍可能在同一进程内顺序构建，一个 target 失败会中断后续 target。

目标是让不同应用、不同平台、不同制品类型完全独立构建。每个构建都在自己的项目副本中运行，失败只影响当前构建单元，其他构建继续执行。

## 目标

1. 构建单元统一定义为 `app + platform + artifactType`。
2. 每个构建单元使用独立 workspace，目录结构为：

   ```text
   <builder-workspace-root>/<app>/<platform>/<artifactType>
   ```

3. 每个 workspace 都从项目源代码 clone 或同步而来，构建进程的工作目录和 CLI `--working-dir` 都指向该 workspace。
4. 所有构建单元并行调度，并通过已有并发数量设置控制同时运行数量。
5. 单个构建单元失败不会取消、阻塞或污染其他构建单元。
6. 最终输出成功和失败汇总；只要存在失败，CLI 入口退出码为 1。

## 非目标

- 不重构 fastforge、Flutter、Docker 或 native build 的内部执行命令。
- 不改变 `.puupee/builder.yaml` 的应用名规则，配置键继续使用服务端中横线应用名。
- 不改变 GUI 构建记录创建流程；每个制品仍然单独创建一个 BuildRecord。
- 不清理旧构建目录，也不实现 workspace 生命周期管理策略。

## 方案

采用统一构建单元方案。GUI 和 CLI 都将配置解析成一组独立构建单元，然后进入并发队列。每个构建单元只携带一个平台和一个制品类型，构建命令也只传一个 `--targets` 值。

### 核心模型

在 CLI 内新增或复用一个清晰模型，例如：

```dart
class BuildUnit {
  final String app;
  final String platform;
  final String artifactType;
}
```

`BuildTarget` 已表达 `platform + artifactType`，因此实现时可选择扩展现有模型，避免引入重复概念。但调度层必须显式以 `app + BuildTarget` 作为任务身份。

### Workspace 规则

新增或整理 workspace 路径解析逻辑，输入为 `projectRootDir/app/platform/artifactType`，输出为稳定目录：

```text
<root>/<safeApp>/<safePlatform>/<safeArtifactType>
```

`root` 继续优先使用 `PUUPEE_BUILDER_WORKSPACE_DIR`；未配置时沿用现有默认目录。路径片段需要做保守清洗，避免空值、路径分隔符和 Windows 不安全字符进入目录名。

GUI 侧的 `BuildService` 将 `_buildWorkspaceDirFor(projectRootDir)` 改为按任务维度解析：

```dart
_buildWorkspaceDirFor(
  projectRootDir: projectRootDir,
  appName: appName,
  platform: platform,
  artifactType: target,
)
```

锁粒度为最终 workspace 目录。同一个 `app/platform/artifactType` 不会并发写同一目录，不同构建单元之间不会共享目录。

### CLI 调度

`BuildCommand`：

- 解析所有 `appsToBuild`。
- 对每个应用通过 `BuildTargetResolver.resolveTargets` 得到平台和制品。
- 将每个 `app + target` 拆成独立构建单元。
- 无论是单应用多 target 还是多应用，都通过同一个并发调度器执行。
- 子进程调用时显式传单个 `--platform` 和单个 `--targets`。

`BuildAllCommand`：

- 根据当前系统和默认策略生成构建单元。
- macOS 上 Android APK 和 macOS DMG 是两个独立构建单元，不再让 DMG 依赖 Android APK 成功。
- 每个构建单元独立进入 semaphore 队列。

### BuildCoordinator 行为

`BuildCoordinator.executeBuild` 保持可以接收 `List<BuildTarget>`，但调度层正常只传单个 target。为兼容直接调用多 target 的情况，内部不应在第一个 target 失败时立即终止整个方法，而应：

- 为每个 target 独立 try/catch。
- 失败时更新该 target 的 BuildRecord 为 failed。
- 记录失败并继续下一个 target。
- 方法末尾如存在失败，再抛出汇总异常。

这样即使未来有旧调用一次传多个 target，也能满足“单独失败不影响其他构建”的要求。

### 失败隔离

单元失败包括 clone、同步、submodule、构建、产物解析、归档、发布等阶段。失败处理规则：

- 当前单元标记 failed，并记录错误信息。
- 释放并发槽位和 workspace 锁。
- 其他单元不取消、不等待失败单元额外处理、不共用失败单元目录。
- 总结中按 `app platform artifactType` 输出失败项。

### 产物归档

归档根目录继续使用 `--archive-root` 或 GUI 的项目根目录。构建产物从独立 workspace 解析，再复制到项目根目录下既有 `dist/<app>/<platform>/<version>/...` 结构。这个设计让构建隔离，不改变用户查找产物的位置。

### 测试策略

先按 TDD 增加测试，再实现：

1. CLI 单元拆分测试：多个 app、多个 platform、多个 targets 会生成多个独立构建单元。
2. CLI 并发失败隔离测试：一个构建单元失败时，其他单元仍被执行，最终返回失败汇总。
3. BuildAll macOS 任务拆分测试：Android APK 和 macOS DMG 是两个平级任务，DMG 不依赖 APK 成功。
4. GUI workspace 路径测试：`app/platform/artifactType` 生成三层目录，且不同 artifactType 不共享目录。
5. BuildCoordinator 多 target 兼容测试：第一个 target 失败后仍执行后续 target，并在最后抛汇总异常。

最终验证命令：

```bash
cd packages/tools/puupee_builder_cli && dart test
cd packages/tools/puupee_builder_cli && dart analyze
cd apps/builder && dart analyze
```

如果 GUI service 的测试需要 Flutter 环境，再补充：

```bash
cd apps/builder && flutter test
```

## 风险与缓解

- 并行 clone 会增加磁盘和 IO 压力。通过 `--jobs` 和 GUI 最大并发设置控制。
- 同一 BuildRecord ID 不应复用于多个构建单元。GUI 当前已经每个 target 创建独立记录；CLI 传入 `--build-record-id` 时只适合单 target 场景，实施时需要保留校验或明确错误提示。
- 多进程同时使用同一个 pub cache 仍可能有外部工具层面的锁竞争。本次只隔离项目 workspace，不隔离全局缓存。
- 旧脚本 `scripts/build_all.dart` 和 `scripts/build.dart` 是包装入口，实施时应确认是否继续保留，至少不要让它们绕开 CLI 的独立构建能力。

## 验收标准

- 选择同一应用的多个制品时，每个制品有独立构建目录。
- 选择多个应用、多个平台、多个制品时，构建按配置的并发数并行运行。
- 任一构建单元失败后，已排队或运行中的其他构建单元继续完成。
- 日志、BuildRecord 状态和产物路径均对应单个构建单元。
- CLI 输出汇总能看到每个失败构建单元，且存在失败时退出码为 1。
