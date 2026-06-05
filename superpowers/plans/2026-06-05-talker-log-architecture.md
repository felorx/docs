# Talker Log Architecture Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a Talker-based custom logging layer so `sync_server.dart` writes logs under a filterable `sync-server` key with searchable scene labels.

**Architecture:** Add a focused logging model and logger facade to `puupee_utilities`, then migrate only `packages/puupee_sync/lib/src/sync_server.dart` to use it. The global `talker` remains the single logging backend, and the existing developer `TalkerScreen(talker: talker)` remains the UI.

**Tech Stack:** Dart 3.8, `talker` 5.1.17, Dart `test`, `puupee_utilities`, `puupee_sync`.

---

## File Structure

- Create `packages/puupee_utilities/test/talker_test.dart`
  - Unit tests for `PuupeeTalkerLog`, `PuupeeLogger`, key registration, and `createTalker(settings:)`.
- Modify `packages/puupee_utilities/lib/talker.dart`
  - Define module/scene constants, custom log model, logger facade, key registration, and fix `createTalker`.
- Modify `packages/puupee_utilities/lib/puupee_utilities.dart`
  - Already exports `talker.dart`; verify no extra export is needed.
- Modify `packages/puupee_sync/lib/src/sync_server.dart`
  - Replace direct `Talker` usage with `PuupeeLogger.syncServer()`.
  - Migrate existing `_logger.info/warning/error` calls to named parameters and scene labels.
- Create `packages/puupee_sync/test/sync_server_logging_test.dart`
  - Lightweight tests proving sync-server logs use key `sync-server` and scene-prefixed messages.

---

### Task 1: Add Puupee Talker Infrastructure Tests

**Files:**
- Create: `packages/puupee_utilities/test/talker_test.dart`
- Modify: `packages/puupee_utilities/lib/talker.dart`

- [ ] **Step 1: Write the failing infrastructure tests**

Create `packages/puupee_utilities/test/talker_test.dart`:

```dart
import 'package:puupee_utilities/talker.dart';
import 'package:talker/talker.dart';
import 'package:test/test.dart';

void main() {
  group('PuupeeTalkerLog', () {
    test('应该写入模块 key、等级和场景前缀', () {
      final stackTrace = StackTrace.current;
      final exception = StateError('boom');

      final log = PuupeeTalkerLog(
        '处理配对请求失败',
        module: PuupeeLogModule.syncServer,
        scene: PuupeeLogScene.pairing,
        logLevel: LogLevel.error,
        exception: exception,
        stackTrace: stackTrace,
      );

      expect(log.key, equals(PuupeeLogModule.syncServer));
      expect(log.title, equals(PuupeeLogModule.syncServer));
      expect(log.logLevel, equals(LogLevel.error));
      expect(log.message, equals('[pairing] 处理配对请求失败'));
      expect(log.exception, same(exception));
      expect(log.stackTrace, same(stackTrace));
    });

    test('没有场景时不添加空前缀', () {
      final log = PuupeeTalkerLog(
        '同步服务器已停止',
        module: PuupeeLogModule.syncServer,
        logLevel: LogLevel.info,
      );

      expect(log.message, equals('同步服务器已停止'));
    });
  });

  group('PuupeeLogger', () {
    test('应该通过 logCustom 写入 Talker history', () {
      final loggerBackend = createTalker(
        settings: TalkerSettings(useConsoleLogs: false),
      );
      final logger = PuupeeLogger.syncServer(backend: loggerBackend);

      logger.warning(
        'API Key 验证失败',
        scene: PuupeeLogScene.auth,
        exception: const FormatException('bad api key'),
        stackTrace: StackTrace.empty,
      );

      expect(loggerBackend.history, hasLength(1));
      final log = loggerBackend.history.single as PuupeeTalkerLog;
      expect(log.key, equals(PuupeeLogModule.syncServer));
      expect(log.logLevel, equals(LogLevel.warning));
      expect(log.message, equals('[auth] API Key 验证失败'));
      expect(log.exception, isA<FormatException>());
      expect(log.stackTrace, same(StackTrace.empty));
    });

    test('createTalker 应该保留传入 settings', () {
      final loggerBackend = createTalker(
        settings: TalkerSettings(
          useConsoleLogs: false,
          titles: const {PuupeeLogModule.syncServer: 'Sync Server'},
        ),
      );

      expect(loggerBackend.settings.useConsoleLogs, isFalse);
      expect(
        loggerBackend.settings.getTitleByKey(PuupeeLogModule.syncServer),
        equals('Sync Server'),
      );
    });

    test('registerPuupeeTalkerKeys 应该注册 sync-server key', () {
      final loggerBackend = Talker(
        settings: TalkerSettings(
          useConsoleLogs: false,
          titles: const {},
        ),
      );

      registerPuupeeTalkerKeys(loggerBackend);

      expect(
        loggerBackend.settings.registeredKeys,
        contains(PuupeeLogModule.syncServer),
      );
      expect(
        loggerBackend.settings.getTitleByKey(PuupeeLogModule.syncServer),
        equals(PuupeeLogModule.syncServer),
      );
    });
  });
}
```

- [ ] **Step 2: Run the new test to verify it fails**

Run:

```bash
cd packages/puupee_utilities
dart test test/talker_test.dart
```

Expected: FAIL with missing symbols such as `PuupeeTalkerLog`, `PuupeeLogModule`, `PuupeeLogScene`, `PuupeeLogger`, or `registerPuupeeTalkerKeys`.

- [ ] **Step 3: Implement the minimal logging infrastructure**

Replace `packages/puupee_utilities/lib/talker.dart` with:

```dart
import 'package:talker/talker.dart';

abstract final class PuupeeLogModule {
  static const syncServer = 'sync-server';
  static const syncClient = 'sync-client';
  static const syncStorage = 'sync-storage';
  static const syncAuth = 'sync-auth';
  static const syncMcp = 'sync-mcp';
}

abstract final class PuupeeLogScene {
  static const lifecycle = 'lifecycle';
  static const db = 'db';
  static const mq = 'mq';
  static const http = 'http';
  static const storage = 'storage';
  static const pairing = 'pairing';
  static const auth = 'auth';
  static const mcp = 'mcp';
  static const cache = 'cache';
  static const crdt = 'crdt';
  static const contentEvent = 'content-event';
  static const version = 'version';
  static const appInfo = 'app-info';
}

class PuupeeTalkerLog extends TalkerLog {
  PuupeeTalkerLog(
    String message, {
    required String module,
    String? scene,
    Object? exception,
    StackTrace? stackTrace,
    LogLevel? logLevel,
  }) : super(
         _formatMessage(message, scene),
         key: module,
         title: module,
         exception: exception,
         stackTrace: stackTrace,
         logLevel: logLevel,
       );

  static String _formatMessage(String message, String? scene) {
    final normalizedScene = scene?.trim();
    if (normalizedScene == null || normalizedScene.isEmpty) {
      return message;
    }
    return '[$normalizedScene] $message';
  }
}

class PuupeeLogger {
  const PuupeeLogger({
    required this.talker,
    required this.module,
  });

  factory PuupeeLogger.syncServer({Talker? backend}) {
    return PuupeeLogger(
      talker: backend ?? talker,
      module: PuupeeLogModule.syncServer,
    );
  }

  final Talker talker;
  final String module;

  void critical(
    Object? message, {
    String? scene,
    Object? exception,
    StackTrace? stackTrace,
  }) {
    _log(
      message,
      scene: scene,
      exception: exception,
      stackTrace: stackTrace,
      logLevel: LogLevel.critical,
    );
  }

  void debug(
    Object? message, {
    String? scene,
    Object? exception,
    StackTrace? stackTrace,
  }) {
    _log(
      message,
      scene: scene,
      exception: exception,
      stackTrace: stackTrace,
      logLevel: LogLevel.debug,
    );
  }

  void error(
    Object? message, {
    String? scene,
    Object? exception,
    StackTrace? stackTrace,
  }) {
    _log(
      message,
      scene: scene,
      exception: exception,
      stackTrace: stackTrace,
      logLevel: LogLevel.error,
    );
  }

  void info(
    Object? message, {
    String? scene,
    Object? exception,
    StackTrace? stackTrace,
  }) {
    _log(
      message,
      scene: scene,
      exception: exception,
      stackTrace: stackTrace,
      logLevel: LogLevel.info,
    );
  }

  void verbose(
    Object? message, {
    String? scene,
    Object? exception,
    StackTrace? stackTrace,
  }) {
    _log(
      message,
      scene: scene,
      exception: exception,
      stackTrace: stackTrace,
      logLevel: LogLevel.verbose,
    );
  }

  void warning(
    Object? message, {
    String? scene,
    Object? exception,
    StackTrace? stackTrace,
  }) {
    _log(
      message,
      scene: scene,
      exception: exception,
      stackTrace: stackTrace,
      logLevel: LogLevel.warning,
    );
  }

  void _log(
    Object? message, {
    required LogLevel logLevel,
    String? scene,
    Object? exception,
    StackTrace? stackTrace,
  }) {
    talker.logCustom(
      PuupeeTalkerLog(
        message?.toString() ?? '',
        module: module,
        scene: scene,
        exception: exception,
        stackTrace: stackTrace,
        logLevel: logLevel,
      ),
    );
  }
}

Talker createTalker({TalkerSettings? settings}) {
  final instance = Talker(settings: settings);
  registerPuupeeTalkerKeys(instance);
  return instance;
}

void registerPuupeeTalkerKeys(Talker talker) {
  talker.settings.registerKeys(const [PuupeeLogModule.syncServer]);
  talker.settings.titles[PuupeeLogModule.syncServer] =
      PuupeeLogModule.syncServer;
}

Talker talker = createTalker();
```

- [ ] **Step 4: Run the test to verify it passes**

Run:

```bash
cd packages/puupee_utilities
dart test test/talker_test.dart
```

Expected: PASS.

- [ ] **Step 5: Run formatter**

Run:

```bash
dart format lib/talker.dart test/talker_test.dart
```

Expected: formatter exits with code 0.

- [ ] **Step 6: Commit infrastructure**

Run:

```bash
git add packages/puupee_utilities/lib/talker.dart packages/puupee_utilities/test/talker_test.dart
git commit -m "feat(puupee_utilities): 添加 Talker 自定义日志基础设施"
```

Expected: commit succeeds.

---

### Task 2: Add Sync Server Logging Test

**Files:**
- Create: `packages/puupee_sync/test/sync_server_logging_test.dart`
- Modify: `packages/puupee_sync/lib/src/sync_server.dart`

- [ ] **Step 1: Write a sync-server logger behavior test**

Create `packages/puupee_sync/test/sync_server_logging_test.dart`:

```dart
import 'package:puupee_utilities/puupee_utilities.dart';
import 'package:talker/talker.dart';
import 'package:test/test.dart';

void main() {
  group('sync-server 日志', () {
    test('应该写入 sync-server key 和 lifecycle 场景', () {
      final loggerBackend = createTalker(
        settings: TalkerSettings(useConsoleLogs: false),
      );
      final logger = PuupeeLogger.syncServer(backend: loggerBackend);

      logger.info(
        'Serving at http://0.0.0.0:8080',
        scene: PuupeeLogScene.lifecycle,
      );

      expect(loggerBackend.history, hasLength(1));
      final log = loggerBackend.history.single;
      expect(log.key, equals(PuupeeLogModule.syncServer));
      expect(log.logLevel, equals(LogLevel.info));
      expect(log.message, equals('[lifecycle] Serving at http://0.0.0.0:8080'));
    });

    test('应该传递错误对象和堆栈', () {
      final loggerBackend = createTalker(
        settings: TalkerSettings(useConsoleLogs: false),
      );
      final logger = PuupeeLogger.syncServer(backend: loggerBackend);
      final exception = Exception('storage failed');

      logger.error(
        '本地上传失败',
        scene: PuupeeLogScene.storage,
        exception: exception,
        stackTrace: StackTrace.empty,
      );

      final log = loggerBackend.history.single;
      expect(log.key, equals(PuupeeLogModule.syncServer));
      expect(log.logLevel, equals(LogLevel.error));
      expect(log.message, equals('[storage] 本地上传失败'));
      expect(log.exception, same(exception));
      expect(log.stackTrace, same(StackTrace.empty));
    });
  });
}
```

- [ ] **Step 2: Run the test**

Run:

```bash
cd packages/puupee_sync
dart test test/sync_server_logging_test.dart
```

Expected: PASS if Task 1 is complete. If it fails because `puupee_sync` does not see the local workspace version of `puupee_utilities`, run `melos bootstrap` from the repository root and rerun the test.

- [ ] **Step 3: Format the new test**

Run:

```bash
dart format test/sync_server_logging_test.dart
```

Expected: formatter exits with code 0.

- [ ] **Step 4: Commit the test**

Run:

```bash
git add packages/puupee_sync/test/sync_server_logging_test.dart
git commit -m "test(puupee_sync): 覆盖 sync-server 自定义日志"
```

Expected: commit succeeds.

---

### Task 3: Migrate sync_server.dart Logger Field and Imports

**Files:**
- Modify: `packages/puupee_sync/lib/src/sync_server.dart`

- [ ] **Step 1: Remove direct Talker import**

In `packages/puupee_sync/lib/src/sync_server.dart`, remove:

```dart
import 'package:talker/talker.dart';
```

Keep:

```dart
import 'package:puupee_utilities/puupee_utilities.dart';
```

- [ ] **Step 2: Replace the logger field**

Change:

```dart
final Talker _logger = talker;
```

To:

```dart
final PuupeeLogger _logger = PuupeeLogger.syncServer();
```

- [ ] **Step 3: Run analyzer to see call-site errors**

Run:

```bash
cd packages/puupee_sync
dart analyze lib/src/sync_server.dart
```

Expected: FAIL with errors for positional exception/stackTrace arguments on `_logger.warning` and `_logger.error`. These errors drive Task 4.

- [ ] **Step 4: Commit only if analyzer shows no unrelated issues after Task 4**

Do not commit this task alone if `sync_server.dart` does not compile. The commit for this field/import change happens together with Task 4.

---

### Task 4: Migrate sync_server.dart Log Calls by Scene

**Files:**
- Modify: `packages/puupee_sync/lib/src/sync_server.dart`

- [ ] **Step 1: Convert app-info logs**

Change:

```dart
_logger.warning('解析 App 信息失败: $appName', e, stackTrace);
```

To:

```dart
_logger.warning(
  '解析 App 信息失败: $appName',
  scene: PuupeeLogScene.appInfo,
  exception: e,
  stackTrace: stackTrace,
);
```

- [ ] **Step 2: Convert lifecycle logs**

Use `scene: PuupeeLogScene.lifecycle` for these messages:

```dart
_logger.info('Sync server already started');
_logger.info('Serving at http://${server.address.host}:${server.port}');
_logger.info('同步服务器已停止');
_logger.info('关闭空闲连接: (${client.peerId?.short ?? "unknown"})');
_logger.info('已关闭 ${idleClients.length} 个空闲连接');
_logger.warning('关闭客户端连接失败: $e');
```

Converted examples:

```dart
_logger.info(
  'Sync server already started',
  scene: PuupeeLogScene.lifecycle,
);
```

```dart
_logger.warning(
  '关闭客户端连接失败: $e',
  scene: PuupeeLogScene.lifecycle,
);
```

- [ ] **Step 3: Convert database and CRDT logs**

Use `scene: PuupeeLogScene.db` for:

```dart
_logger.info('Fields: $fields');
_logger.error('Failed to open Postgres database.', e, stackTrace);
```

Use `scene: PuupeeLogScene.crdt` for:

```dart
_logger.info('[validateRecord] Record creator is local user');
```

Converted error:

```dart
_logger.error(
  'Failed to open Postgres database.',
  scene: PuupeeLogScene.db,
  exception: e,
  stackTrace: stackTrace,
);
```

- [ ] **Step 4: Convert MQ and content event logs**

Use `scene: PuupeeLogScene.mq` for:

```dart
_logger.warning('Failed to initialize AMQP exchange', e, stackTrace);
```

Use `scene: PuupeeLogScene.contentEvent` for:

```dart
_logger.error('发布 Puupee content event 失败', e, stackTrace);
```

Converted examples:

```dart
_logger.warning(
  'Failed to initialize AMQP exchange',
  scene: PuupeeLogScene.mq,
  exception: e,
  stackTrace: stackTrace,
);
```

```dart
_logger.error(
  '发布 Puupee content event 失败',
  scene: PuupeeLogScene.contentEvent,
  exception: e,
  stackTrace: stackTrace,
);
```

- [ ] **Step 5: Convert MCP logs**

Use `scene: PuupeeLogScene.mcp` for:

```dart
_logger.info('MCP 服务器端点已注册: GET/POST /mcp (需要认证)');
_logger.info('MCP 服务器端点: http://${server.address.host}:${server.port}/mcp');
```

Converted example:

```dart
_logger.info(
  'MCP 服务器端点已注册: GET/POST /mcp (需要认证)',
  scene: PuupeeLogScene.mcp,
);
```

- [ ] **Step 6: Convert HTTP request/response logs**

Use `scene: PuupeeLogScene.http` inside `_logNonSuccessRequests` and health/request handling for:

```dart
_logger.error('健康检查失败: $e');
_logger.error('读取请求体用于错误日志失败: ${request.method} ${request.requestedUri}', e, stackTrace);
_logger.warning(_formatNonSuccessLog(...));
_logger.error(_formatExceptionLog(...), e, stackTrace);
```

Converted examples:

```dart
_logger.error(
  '健康检查失败: $e',
  scene: PuupeeLogScene.http,
);
```

```dart
_logger.error(
  _formatExceptionLog(
    request: bufferedRequest.request,
    requestBody: bufferedRequest.body,
    elapsed: DateTime.now().difference(startedAt),
  ),
  scene: PuupeeLogScene.http,
  exception: e,
  stackTrace: stackTrace,
);
```

- [ ] **Step 7: Convert storage logs**

Use `scene: PuupeeLogScene.storage` for file upload/download/storage credential logs, including:

```dart
_logger.warning('rapidCode hash 计算异常', e, stackTrace);
_logger.warning('清理临时目录失败: ${dir.path} - $e');
_logger.error('onFileLocalUploaded 回调失败', e, stackTrace);
_logger.warning('onFileLocalProgress 回调失败', e, stackTrace);
_logger.error('本地上传失败', e, stackTrace);
_logger.error('读取文件失败: $e');
_logger.warning('cdnDomain 为 null 且没有默认值: usage=$usage');
_logger.warning('存储空间已满: $errorMessage');
_logger.warning('API 业务错误: $errorCode - $errorMessage');
_logger.warning('解析错误响应失败: $parseError');
_logger.error('获取文件凭证失败: ${e.message}', e, e.stackTrace);
_logger.error('获取文件凭证时发生未预期的错误', e, stackTrace);
_logger.error('Error getting user storage', e, stackTrace);
```

Converted example:

```dart
_logger.error(
  '获取文件凭证失败: ${e.message}',
  scene: PuupeeLogScene.storage,
  exception: e,
  stackTrace: e.stackTrace,
);
```

- [ ] **Step 8: Convert auth and Puupee API logs**

Use `scene: PuupeeLogScene.auth` for:

```dart
_logger.info('配对码认证，跳过权限检查: puupeeId=$puupeeId');
_logger.error('检查 Puupee 访问权限失败: $e', e, stackTrace);
_logger.warning('请求 User-Agent 版本校验失败: $userAgent', e);
```

Use `scene: PuupeeLogScene.http` for Puupee CRUD endpoint failures:

```dart
_logger.error('获取 Puupee 失败: $e', e, stackTrace);
_logger.error('查询 Puupees 集合失败: $e', e, stackTrace);
_logger.error('更新 Puupee 失败: $e', e, stackTrace);
_logger.error('删除 Puupee 失败: $e', e, stackTrace);
_logger.error('统计 Puupees 失败: $e', e, stackTrace);
_logger.error('批量创建 Puupees 失败: $e', e, stackTrace);
_logger.error('批量更新 Puupees 失败: $e', e, stackTrace);
_logger.error('批量删除 Puupees 失败: $e', e, stackTrace);
```

Converted auth warning:

```dart
_logger.warning(
  '请求 User-Agent 版本校验失败: $userAgent',
  scene: PuupeeLogScene.auth,
  exception: e,
);
```

- [ ] **Step 9: Convert pairing logs**

Use `scene: PuupeeLogScene.pairing` for:

```dart
_logger.error('处理配对请求失败: $e', e, stackTrace);
_logger.warning('接收端 token 验证失败: ${verificationResult.error}');
_logger.info('接收端 token 验证成功，接收端 nodeId: $receiverNodeId');
_logger.warning('解析或验证接收端公钥失败: $e');
_logger.info('已保存接收端配对 token: $receiverNodeId');
_logger.info('已保存接收端公钥: $receiverNodeId');
_logger.error('处理配对确认失败: $e', e, stackTrace);
```

Converted example:

```dart
_logger.error(
  '处理配对确认失败: $e',
  scene: PuupeeLogScene.pairing,
  exception: e,
  stackTrace: stackTrace,
);
```

- [ ] **Step 10: Convert cache logs**

Use `scene: PuupeeLogScene.cache` for:

```dart
_logger.info('清理了 ${keysToRemove.length} 个过期缓存项');
```

Converted:

```dart
_logger.info(
  '清理了 ${keysToRemove.length} 个过期缓存项',
  scene: PuupeeLogScene.cache,
);
```

- [ ] **Step 11: Find and convert every remaining call**

Run:

```bash
rg -n "_logger\\.(info|warning|error|debug|verbose|critical)\\([^\\n]*, [^\\n]*, [^\\n]*\\)" packages/puupee_sync/lib/src/sync_server.dart
```

Expected: no output.

Run:

```bash
rg -n "_logger\\." packages/puupee_sync/lib/src/sync_server.dart
```

Expected: every result has `scene:` in the call block.

- [ ] **Step 12: Format and analyze**

Run:

```bash
cd packages/puupee_sync
dart format lib/src/sync_server.dart
dart analyze lib/src/sync_server.dart
```

Expected: formatter exits with code 0 and analyzer exits with code 0.

- [ ] **Step 13: Commit sync_server migration**

Run:

```bash
git add packages/puupee_sync/lib/src/sync_server.dart
git commit -m "refactor(puupee_sync): 使用 sync-server 自定义日志"
```

Expected: commit succeeds.

---

### Task 5: Run Focused Tests and Package Analysis

**Files:**
- No source files expected.

- [ ] **Step 1: Run puupee_utilities tests**

Run:

```bash
cd packages/puupee_utilities
dart test test/talker_test.dart
dart analyze lib/talker.dart test/talker_test.dart
```

Expected: both commands exit with code 0.

- [ ] **Step 2: Run puupee_sync focused tests**

Run:

```bash
cd packages/puupee_sync
dart test test/sync_server_logging_test.dart
dart test test/sync_server_test.dart
dart analyze lib/src/sync_server.dart test/sync_server_logging_test.dart
```

Expected: all commands exit with code 0.

- [ ] **Step 3: Run repository-level targeted search checks**

Run:

```bash
rg -n "final Talker _logger = talker" packages/puupee_sync/lib/src/sync_server.dart
```

Expected: no output.

Run:

```bash
rg -n "import 'package:talker/talker.dart';" packages/puupee_sync/lib/src/sync_server.dart
```

Expected: no output.

Run:

```bash
rg -n "PuupeeLogger.syncServer" packages/puupee_sync/lib/src/sync_server.dart
```

Expected: one output line for the `_logger` field.

- [ ] **Step 4: Commit verification notes only if files changed**

If no files changed during verification, do not commit. If formatter changed files, run:

```bash
git status --short
git add packages/puupee_utilities packages/puupee_sync
git commit -m "chore(puupee_sync): 格式化 sync-server 日志迁移"
```

Expected: commit succeeds only when there are formatter changes.

---

### Task 6: Final Review Before PR or Handoff

**Files:**
- Review only.

- [ ] **Step 1: Inspect final diff**

Run:

```bash
git status --short
git log --oneline -n 5
```

Expected: working tree is clean, and recent commits include:

```text
feat(puupee_utilities): 添加 Talker 自定义日志基础设施
test(puupee_sync): 覆盖 sync-server 自定义日志
refactor(puupee_sync): 使用 sync-server 自定义日志
```

- [ ] **Step 2: Confirm spec coverage**

Open `docs/superpowers/specs/2026-06-05-talker-log-architecture-design.md` and confirm:

- `sync-server` is the Talker key.
- Scene labels stay in message prefixes.
- Existing `TalkerScreen(talker: talker)` remains unchanged.
- Only `sync_server.dart` is migrated in the first implementation pass.
- Sensitive request/body behavior is not expanded.

- [ ] **Step 3: Prepare handoff summary**

Use this summary shape:

```text
已完成 Talker 自定义日志第一阶段：
- 在 puupee_utilities 增加 PuupeeTalkerLog / PuupeeLogger / sync-server key 注册。
- 修正 createTalker(settings:) 传参。
- 将 sync_server.dart 迁移到 sync-server 模块日志，并按 scene 标注核心日志。
- 增加 puupee_utilities 与 puupee_sync 的聚焦测试。

验证：
- cd packages/puupee_utilities && dart test test/talker_test.dart
- cd packages/puupee_utilities && dart analyze lib/talker.dart test/talker_test.dart
- cd packages/puupee_sync && dart test test/sync_server_logging_test.dart
- cd packages/puupee_sync && dart test test/sync_server_test.dart
- cd packages/puupee_sync && dart analyze lib/src/sync_server.dart test/sync_server_logging_test.dart
```
