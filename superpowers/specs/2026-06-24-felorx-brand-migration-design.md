# Felorx 品牌与命名迁移设计

## 背景

仓库当前以 Puupee/Puupees 为品牌、包名前缀和核心模型名称，并在 Flutter 子应用、Dart 包、Next.js website、.NET API、K8s、构建工具、扩展和文档中广泛使用 `puupee.com`、`com.puupee`、`puupee_*`、`Puupee`、`Puupees` 与复数应用名。

目标是统一迁移到 Felorx 品牌：

- 域名统一为 `felorx.com`。
- Bundle id 前缀统一为 `com.felorx`。
- 明显复数应用名改为单数。
- Dart 包名从 `puupee_*` 迁移到 `felorx_*`，核心同步包从 `puupee_sync` 迁移到 `felorx_sync`。
- Puupee 数据模型、功能模型、类名和 API 命名最终迁移到 Felorx 命名。

## 范围

### 域名与外部地址

替换运行配置、默认配置和面向用户入口中的 `puupee.com`：

- website 默认站点域名、认证地址、API 地址、robots、Vercel/Nginx/部署文档。
- API appsettings、AuthServer、HttpApi.Host、DbMigrator 中的公开 URL、CORS、redirect URL、issuer/audience 配置。
- Flutter 应用 `env.dart`、共享包 env、CLI、构建工具、扩展、K8s ingress 和容器配置。
- 邮箱、发布者 URL、安装器元数据中的品牌域名改为 `felorx.com`。

### Bundle id 与平台包标识

将所有应用和平台工程中的 `com.puupee` 改为 `com.felorx`：

- Android namespace、applicationId、Kotlin/Java MainActivity 包路径。
- iOS/macOS `PRODUCT_BUNDLE_IDENTIFIER`、entitlements、flavorizr 配置。
- Linux `APPLICATION_ID`。
- Windows installer、Runner.rc、模板和构建生成器默认值。
- OHOS 包名配置。
- 子应用模板、生成脚本、构建工具测试预期。

### 复数应用改单数

确认迁移以下明显复数应用，`ops` 等缩写或领域词保持不变：

| 旧应用目录/键 | 新应用目录/键 | 旧包名 | 新包名 | 英文展示名示例 |
| --- | --- | --- | --- | --- |
| `todos` | `todo` | `puupee_todos` | `felorx_todo` | `Puupee Todos` -> `Felorx Todo` |
| `billings` | `billing` | `puupee_billings` | `felorx_billing` | `Puupee Billings` -> `Felorx Billing` |
| `photos` | `photo` | `puupee_photos` | `felorx_photo` | `Puupee Photos` -> `Felorx Photo` |
| `notes` | `note` | `puupee_notes` | `felorx_note` | `Puupee Notes` -> `Felorx Note` |
| `messages` | `message` | `puupee_messages` | `felorx_message` | `Puupee Messages` -> `Felorx Message` |
| `boxes` | `box` | `puupee_boxes` | `felorx_box` | `Puupee Boxes` -> `Felorx Box` |

迁移内容包括：

- `apps/<old>` 目录重命名为 `apps/<new>`。
- 根 `pubspec.yaml` workspace、`.puupee/builder.yaml`、`.vscode/launch.json`、Codemagic/GitHub Actions、构建脚本、文档路径引用。
- 子应用 `pubspec.yaml` 的 `name`、`description`、URL scheme、依赖引用和测试引用。
- 路由路径与 website app 数据中面向用户的 app slug，保持业务 content type `todo` 等原本已经单数的值不被误改。
- 历史 tag 和 artifact 文件名默认值从 `puupee_<plural>` 迁移到 `felorx_<singular>`，同时在构建工具中按需保留兼容读取旧文件名的逻辑。

### Dart 包迁移

分批迁移 Dart 包名、目录和导入：

- 第一批：同步与 API 相关包，`packages/sync/puupee_sync` -> `packages/sync/felorx_sync`，`puupee_sync_api` -> `felorx_sync_api`。
- 第二批：公共包，`puupee_shared`、`puupee_ui`、`puupee_upgrader`、`puupee_sdui`、`puupee_utilities`、`puupee_annotations`、`puupee_generator`。
- 第三批：工具包，`puupee_cli`、`puupee_builder_cli`、`puupee_icons`、SDK generator/client。
- 核心包 `packages/core/puupee` 迁移到 `packages/core/felorx`，并更新导出库名、生成器、imports 和依赖约束。

为降低风险，每批提交都必须让 `dart pub get`/`melos bootstrap` 能解析 workspace，并用 `rg 'package:puupee'` 与 `rg 'puupee_'` 检查残留。

### 核心模型/API 命名

最后迁移语义层命名：

- Dart 模型：`Puupee` -> `Felorx`，`Puupees` 表类 -> `Felorxes` 或明确约定的复数表类，`PuupeeContentTypes` -> `FelorxContentTypes`。
- Content type 前缀：`application/vnd.puupee.*` -> `application/vnd.felorx.*`。
- Drift 生成代码通过 build_runner 重新生成，不手工维护 `database.g.dart`。
- .NET API 项目、命名空间、模块、DbContext、权限组、错误码、资源文件和 appsettings 从 `Puupees`/`Puupee` 迁移到 `Felorx`。
- 数据库表名和已有数据迁移需要单独 migration，确保旧表和旧 content type 可升级到新命名。

## 分阶段实施

### 阶段 1：域名与品牌可见配置

只替换域名、发布者、网站名和配置默认值，不改目录名、包名或模型名。

验证：

- `cd website && npm test -- --runInBand`
- `cd website && npm run type-check`
- `rg 'puupee\\.com'` 检查运行配置残留，并记录历史文档中的可接受残留。

提交建议：

- `refactor(website): 迁移 Felorx 域名配置`
- `refactor(api): 迁移 Felorx 服务域名配置`
- `refactor(k8s): 迁移 Felorx 入口域名`

### 阶段 2：Bundle id 前缀

将平台工程、模板和构建生成器中的 `com.puupee` 改为 `com.felorx`。目录名和 Dart 包名仍保持现状，避免同时触发 import 断裂。

验证：

- `rg 'com\\.puupee' apps packages templates scripts codemagic.yaml pubspec.yaml`
- 针对构建工具运行 `cd packages/tools/puupee_builder_cli && dart test`
- 对代表性子应用运行 `cd apps/todos && dart analyze`

提交建议：

- `refactor(apps): 迁移 Felorx 应用包名前缀`
- `refactor(puupee_builder_cli): 迁移 Felorx 打包标识默认值`

### 阶段 3：复数应用改单数

逐个应用迁移目录、workspace、构建键、路由、展示名、平台包尾段和测试引用。每个应用单独提交，优先从 `todos -> todo` 开始，用它验证迁移脚本和检查清单。

验证：

- `melos bootstrap`
- `cd apps/todo && dart analyze`
- `cd apps/todo && flutter test`
- `rg 'apps/todos|\\btodos\\b|Puupee Todos|puupee_todos|com\\.felorx\\.todos'` 检查残留，并区分历史文档残留。

提交建议：

- `refactor(todo): 将待办应用改为 Felorx Todo`
- 后续应用按同样粒度提交。

### 阶段 4：Dart 包名迁移

按依赖拓扑从底层到上层迁移包目录、`pubspec.yaml name`、imports、exports、生成器配置和测试引用。先迁移同步包，再迁移共享包和工具包。

验证：

- `melos bootstrap`
- `melos run analyze`
- 迁移包内 `dart test` 或 `flutter test`
- `rg 'package:puupee_|puupee_sync|puupee_shared|puupee_utilities'` 检查残留。

提交建议：

- `refactor(felorx_sync): 迁移同步包命名`
- `refactor(felorx_shared): 迁移共享包命名`

### 阶段 5：核心模型与 API 命名

最后迁移模型和服务端命名空间，因为这会影响生成代码、数据库迁移、OpenAPI、客户端生成包和业务错误码。

验证：

- `cd packages/core/felorx && dart run build_runner build --delete-conflicting-outputs`
- `cd packages/core/felorx && dart test`
- `cd api && dotnet build`
- `cd api && dotnet test`
- 重新生成 API 客户端后运行相关 Dart 测试。

提交建议：

- `refactor(felorx): 迁移核心数据模型命名`
- `refactor(api): 迁移 Felorx API 命名空间`
- `build(felorx_sync_api): 重新生成 Felorx 同步客户端`

## 兼容策略

- 服务端对旧 `application/vnd.puupee.*` content type 提供一次性 migration，并在读取层短期兼容旧值。
- 构建工具对旧 artifact/tag 文件名保留只读兼容，生成新 artifact 时使用 `felorx_<singular>`。
- OAuth client id 和已有 App Store/Bundle id 变更可能需要外部控制台同步；代码迁移只负责仓库内配置。
- 历史文档、旧设计稿和 changelog 可在运行代码迁移后单独扫尾，不阻塞构建验证。

## 风险与缓解

- 风险：全仓替换误改业务单数 `todo` 或历史文档。
  缓解：按阶段、按路径白名单替换，并用 `rg` 对关键旧名做残留审查。

- 风险：目录重命名导致 Dart workspace 和 imports 同时断裂。
  缓解：先更新 workspace 和 pubspec，再更新 imports，最后运行 `melos bootstrap`。

- 风险：Drift 生成代码和手写模型不一致。
  缓解：只改源表和模型定义，通过 build_runner 重新生成。

- 风险：.NET 项目/命名空间迁移产生大量文件移动。
  缓解：API 命名独立阶段处理，先 `dotnet build`，再生成客户端。

- 风险：已有用户数据中的旧 content type、bundle id 或 app name 不可识别。
  缓解：服务端和本地数据库 migration 中显式映射旧值到新值。

## 完成标准

- `rg 'puupee\\.com|com\\.puupee|Puupee Todos|puupee_todos|apps/todos'` 在运行代码和配置中无非预期残留。
- `website` 类型检查和测试通过。
- 受影响 Flutter workspace 能 bootstrap，代表性应用和核心包分析/测试通过。
- API 能 build/test，通过后重新生成客户端。
- 所有新生成的应用名、包名、bundle id、域名和 content type 使用 Felorx 命名。
