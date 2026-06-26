# Builder Isolated Parallel Builds Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make every builder run operate on one independent `app + platform + artifactType` unit, each with its own cloned workspace and failure-isolated parallel execution.

**Architecture:** Add small CLI-side primitives for build units, scheduling, workspace paths, and workspace preparation, then wire GUI and CLI entrypoints through those primitives. GUI keeps its BuildRecord-per-target behavior, while CLI `build`, CLI `build-all`, and legacy wrapper scripts stop running multi-target work inside one shared source tree.

**Tech Stack:** Dart 3.8, `package:test`, `package:args`, `package:path`, Flutter service code in `apps/builder`, existing `felorx_builder_cli` services.

---

## File Structure

- Create `packages/tools/felorx_builder_cli/lib/src/models/build_unit.dart`: immutable identity for one app/platform/artifact build.
- Create `packages/tools/felorx_builder_cli/lib/src/models/build_unit_result.dart`: success/failure record for scheduler summaries.
- Create `packages/tools/felorx_builder_cli/lib/src/utils/semaphore.dart`: shared concurrency primitive used by build and build-all commands.
- Create `packages/tools/felorx_builder_cli/lib/src/services/build_unit_planner.dart`: expands apps and resolved targets into build units.
- Create `packages/tools/felorx_builder_cli/lib/src/services/build_unit_scheduler.dart`: runs build units with a semaphore and never cancels siblings on failure.
- Create `packages/tools/felorx_builder_cli/lib/src/services/build_workspace_path_resolver.dart`: sanitizes and joins `app/platform/artifactType`.
- Create `packages/tools/felorx_builder_cli/lib/src/services/build_workspace_service.dart`: clones or syncs source project into the unit workspace.
- Create `packages/tools/felorx_builder_cli/lib/src/services/build_unit_process_command_builder.dart`: builds child-process args for a single isolated unit.
- Create `packages/tools/felorx_builder_cli/lib/src/services/build_target_failure_isolator.dart`: keeps `BuildCoordinator` compatible when called with several targets.
- Modify `packages/tools/felorx_builder_cli/lib/felorx_builder_cli.dart`: export new model/services needed by GUI.
- Modify `packages/tools/felorx_builder_cli/lib/src/build_command.dart`: add orchestrator mode and hidden isolated-unit mode.
- Modify `packages/tools/felorx_builder_cli/lib/src/build_all_command.dart`: build unit list first, then schedule independent units.
- Modify `packages/tools/felorx_builder_cli/lib/src/services/build_coordinator.dart`: isolate target failures inside direct multi-target calls.
- Modify `apps/builder/lib/services/build_service.dart`: use the shared workspace path resolver and include app/platform/artifactType in workspace paths.
- Modify `scripts/build_all.dart`: delegate to CLI `build-all` so the legacy wrapper no longer serializes its own build graph.
- Test `packages/tools/felorx_builder_cli/test/build_unit_planner_test.dart`.
- Test `packages/tools/felorx_builder_cli/test/build_unit_scheduler_test.dart`.
- Test `packages/tools/felorx_builder_cli/test/build_workspace_path_resolver_test.dart`.
- Test `packages/tools/felorx_builder_cli/test/build_workspace_service_test.dart`.
- Test `packages/tools/felorx_builder_cli/test/build_unit_process_command_builder_test.dart`.
- Test `packages/tools/felorx_builder_cli/test/build_target_failure_isolator_test.dart`.
- Test `packages/tools/felorx_builder_cli/test/build_all_units_test.dart`.
- Test `apps/builder/test/services/build_service_workspace_test.dart`.

---

### Task 1: Build Unit Model, Planner, and Scheduler

**Files:**
- Create: `packages/tools/felorx_builder_cli/lib/src/models/build_unit.dart`
- Create: `packages/tools/felorx_builder_cli/lib/src/models/build_unit_result.dart`
- Create: `packages/tools/felorx_builder_cli/lib/src/utils/semaphore.dart`
- Create: `packages/tools/felorx_builder_cli/lib/src/services/build_unit_planner.dart`
- Create: `packages/tools/felorx_builder_cli/lib/src/services/build_unit_scheduler.dart`
- Modify: `packages/tools/felorx_builder_cli/lib/felorx_builder_cli.dart`
- Test: `packages/tools/felorx_builder_cli/test/build_unit_planner_test.dart`
- Test: `packages/tools/felorx_builder_cli/test/build_unit_scheduler_test.dart`

- [ ] **Step 1: Write the failing planner test**

```dart
import 'package:felorx_builder_cli/src/config_parser.dart';
import 'package:felorx_builder_cli/src/services/build_target_resolver.dart';
import 'package:felorx_builder_cli/src/services/build_unit_planner.dart';
import 'package:test/test.dart';

void main() {
  test('为多个应用和多个制品生成独立构建单元', () {
    final config = BuilderConfig(
      apps: {
        'todo': AppConfig(
          name: 'todo',
          localName: 'todo',
          platforms: {
            'android': PlatformConfig(
              platform: 'android',
              targets: {
                'apk': ['felorx'],
                'aab': ['felorx'],
              },
            ),
          },
        ),
        'note': AppConfig(
          name: 'note',
          localName: 'note',
          platforms: {
            'macos': PlatformConfig(
              platform: 'macos',
              targets: {
                'dmg': ['felorx'],
              },
            ),
          },
        ),
      },
    );

    final units = BuildUnitPlanner(
      targetResolver: BuildTargetResolver(),
    ).plan(
      apps: ['todo', 'note'],
      config: config,
      platformFromArgs: null,
      targetsStr: null,
    );

    expect(
      units.map((unit) => unit.toString()).toList(),
      [
        'todo:android:aab',
        'todo:android:apk',
        'note:macos:dmg',
      ],
    );
  });

  test('命令行 targets 逗号列表被拆成多个单制品构建单元', () {
    final units = BuildUnitPlanner(
      targetResolver: BuildTargetResolver(),
    ).plan(
      apps: ['todo'],
      config: null,
      platformFromArgs: 'android',
      targetsStr: 'apk,aab',
    );

    expect(
      units.map((unit) => unit.toString()).toList(),
      ['todo:android:apk', 'todo:android:aab'],
    );
  });
}
```

- [ ] **Step 2: Run planner test to verify it fails**

Run:

```bash
cd packages/tools/felorx_builder_cli && dart test test/build_unit_planner_test.dart
```

Expected: FAIL because `BuildUnitPlanner` does not exist.

- [ ] **Step 3: Write the failing scheduler test**

```dart
import 'dart:async';

import 'package:felorx_builder_cli/src/models/build_target.dart';
import 'package:felorx_builder_cli/src/models/build_unit.dart';
import 'package:felorx_builder_cli/src/services/build_unit_scheduler.dart';
import 'package:test/test.dart';

void main() {
  test('一个构建单元失败后其他构建单元仍继续执行', () async {
    final scheduler = BuildUnitScheduler();
    final units = [
      BuildUnit(app: 'ok-a', target: BuildTarget(platform: 'android', artifactType: 'apk')),
      BuildUnit(app: 'bad', target: BuildTarget(platform: 'android', artifactType: 'apk')),
      BuildUnit(app: 'ok-b', target: BuildTarget(platform: 'macos', artifactType: 'dmg')),
    ];
    final executed = <String>[];

    final results = await scheduler.run(
      units: units,
      jobs: 2,
      runUnit: (unit) async {
        executed.add(unit.toString());
        await Future<void>.delayed(const Duration(milliseconds: 10));
        if (unit.app == 'bad') {
          throw Exception('boom');
        }
      },
    );

    expect(executed, containsAll(units.map((unit) => unit.toString())));
    expect(results, hasLength(3));
    expect(results.where((result) => result.success), hasLength(2));
    expect(results.where((result) => !result.success).single.unit.app, 'bad');
    expect(results.where((result) => !result.success).single.errorMessage, contains('boom'));
  });

  test('并发数量不超过 jobs', () async {
    final scheduler = BuildUnitScheduler();
    final units = List.generate(
      5,
      (index) => BuildUnit(
        app: 'app-$index',
        target: BuildTarget(platform: 'android', artifactType: 'apk'),
      ),
    );
    var running = 0;
    var maxRunning = 0;

    await scheduler.run(
      units: units,
      jobs: 2,
      runUnit: (unit) async {
        running += 1;
        maxRunning = running > maxRunning ? running : maxRunning;
        await Future<void>.delayed(const Duration(milliseconds: 20));
        running -= 1;
      },
    );

    expect(maxRunning, 2);
  });
}
```

- [ ] **Step 4: Run scheduler test to verify it fails**

Run:

```bash
cd packages/tools/felorx_builder_cli && dart test test/build_unit_scheduler_test.dart
```

Expected: FAIL because `BuildUnitScheduler` does not exist.

- [ ] **Step 5: Implement the model, planner, and scheduler**

Implement `build_unit.dart`:

```dart
import 'build_target.dart';

class BuildUnit {
  final String app;
  final BuildTarget target;

  BuildUnit({required this.app, required this.target});

  String get platform => target.platform;
  String get artifactType => target.artifactType;

  @override
  String toString() => '$app:${target.platform}:${target.artifactType}';

  @override
  bool operator ==(Object other) =>
      identical(this, other) ||
      other is BuildUnit &&
          runtimeType == other.runtimeType &&
          app == other.app &&
          target == other.target;

  @override
  int get hashCode => app.hashCode ^ target.hashCode;
}
```

Implement `build_unit_result.dart`:

```dart
import 'build_unit.dart';

class BuildUnitResult {
  final BuildUnit unit;
  final bool success;
  final String? errorMessage;

  BuildUnitResult.success(this.unit) : success = true, errorMessage = null;

  BuildUnitResult.failure(this.unit, Object error)
    : success = false,
      errorMessage = error.toString();
}
```

Implement `utils/semaphore.dart` by moving the existing `Semaphore` class out of `build_all_command.dart`:

```dart
import 'dart:async';

class Semaphore {
  final int _maxCount;
  int _currentCount;
  final _waitQueue = <Completer<void>>[];

  Semaphore(this._maxCount) : _currentCount = _maxCount;

  int get maxCount => _maxCount;

  Future<void> acquire() async {
    if (_currentCount > 0) {
      _currentCount--;
      return;
    }

    final completer = Completer<void>();
    _waitQueue.add(completer);
    return completer.future;
  }

  void release() {
    if (_waitQueue.isNotEmpty) {
      final completer = _waitQueue.removeAt(0);
      completer.complete();
    } else {
      _currentCount++;
      if (_currentCount > _maxCount) {
        _currentCount = _maxCount;
      }
    }
  }
}
```

Implement `build_unit_planner.dart`:

```dart
import '../config_parser.dart';
import '../models/build_unit.dart';
import 'build_target_resolver.dart';

class BuildUnitPlanner {
  final BuildTargetResolver targetResolver;

  BuildUnitPlanner({required this.targetResolver});

  List<BuildUnit> plan({
    required List<String> apps,
    required BuilderConfig? config,
    required String? platformFromArgs,
    required String? targetsStr,
  }) {
    final units = <BuildUnit>[];
    for (final app in apps) {
      final appConfig = config?.getApp(app);
      final targets = targetResolver.resolveTargets(
        app: app,
        platformFromArgs: platformFromArgs,
        targetsStr: targetsStr,
        appConfig: appConfig,
      );
      for (final target in targets) {
        units.add(BuildUnit(app: app, target: target));
      }
    }
    return units;
  }
}
```

Implement `build_unit_scheduler.dart`:

```dart
import '../models/build_unit.dart';
import '../models/build_unit_result.dart';
import '../utils/semaphore.dart';

class BuildUnitScheduler {
  Future<List<BuildUnitResult>> run({
    required List<BuildUnit> units,
    required int jobs,
    required Future<void> Function(BuildUnit unit) runUnit,
  }) async {
    if (jobs < 1) {
      throw ArgumentError.value(jobs, 'jobs', 'must be greater than 0');
    }

    final semaphore = Semaphore(jobs);
    final results = <BuildUnitResult>[];
    final tasks = <Future<void>>[];

    for (final unit in units) {
      tasks.add(
        semaphore.acquire().then((_) async {
          try {
            await runUnit(unit);
            results.add(BuildUnitResult.success(unit));
          } catch (error) {
            results.add(BuildUnitResult.failure(unit, error));
          } finally {
            semaphore.release();
          }
        }),
      );
    }

    await Future.wait(tasks);
    return results;
  }
}
```

Add exports to `felorx_builder_cli.dart`:

```dart
export 'src/models/build_unit.dart';
export 'src/models/build_unit_result.dart';
export 'src/services/build_unit_planner.dart';
export 'src/services/build_unit_scheduler.dart';
```

- [ ] **Step 6: Run tests to verify they pass**

Run:

```bash
cd packages/tools/felorx_builder_cli && dart test test/build_unit_planner_test.dart test/build_unit_scheduler_test.dart
```

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add packages/tools/felorx_builder_cli/lib/src/models/build_unit.dart \
  packages/tools/felorx_builder_cli/lib/src/models/build_unit_result.dart \
  packages/tools/felorx_builder_cli/lib/src/utils/semaphore.dart \
  packages/tools/felorx_builder_cli/lib/src/services/build_unit_planner.dart \
  packages/tools/felorx_builder_cli/lib/src/services/build_unit_scheduler.dart \
  packages/tools/felorx_builder_cli/lib/felorx_builder_cli.dart \
  packages/tools/felorx_builder_cli/test/build_unit_planner_test.dart \
  packages/tools/felorx_builder_cli/test/build_unit_scheduler_test.dart
git commit -m "feat(builder): 增加独立构建单元调度"
```

---

### Task 2: Workspace Path Resolver and Clone Service

**Files:**
- Create: `packages/tools/felorx_builder_cli/lib/src/services/build_workspace_path_resolver.dart`
- Create: `packages/tools/felorx_builder_cli/lib/src/services/build_workspace_service.dart`
- Modify: `packages/tools/felorx_builder_cli/lib/felorx_builder_cli.dart`
- Test: `packages/tools/felorx_builder_cli/test/build_workspace_path_resolver_test.dart`
- Test: `packages/tools/felorx_builder_cli/test/build_workspace_service_test.dart`

- [ ] **Step 1: Write the failing path resolver test**

```dart
import 'dart:io';

import 'package:path/path.dart' as path;
import 'package:felorx_builder_cli/src/services/build_workspace_path_resolver.dart';
import 'package:test/test.dart';

void main() {
  test('生成 app/platform/artifactType 三层构建目录', () {
    final resolver = BuildWorkspacePathResolver();
    final root = path.join(Directory.systemTemp.path, 'felorx-test-workspaces');

    final workspace = resolver.resolve(
      root: root,
      appName: 'sync-node',
      platform: 'windows',
      artifactType: 'exe',
    );

    expect(workspace, path.join(root, 'sync-node', 'windows', 'exe'));
  });

  test('路径片段会替换分隔符和 Windows 非法字符', () {
    final resolver = BuildWorkspacePathResolver();

    final workspace = resolver.resolve(
      root: '/tmp/root',
      appName: r'bad/app:name',
      platform: r'win\dows',
      artifactType: 'pkg*prod',
    );

    expect(workspace, path.join('/tmp/root', 'bad_app_name', 'win_dows', 'pkg_prod'));
  });
}
```

- [ ] **Step 2: Run path resolver test to verify it fails**

Run:

```bash
cd packages/tools/felorx_builder_cli && dart test test/build_workspace_path_resolver_test.dart
```

Expected: FAIL because `BuildWorkspacePathResolver` does not exist.

- [ ] **Step 3: Write the failing workspace clone test**

```dart
import 'dart:io';

import 'package:path/path.dart' as path;
import 'package:felorx_builder_cli/src/models/build_target.dart';
import 'package:felorx_builder_cli/src/models/build_unit.dart';
import 'package:felorx_builder_cli/src/services/build_workspace_service.dart';
import 'package:test/test.dart';

void main() {
  test('为构建单元 clone 源项目到独立 workspace', () async {
    final tempDir = await Directory.systemTemp.createTemp('felorx-workspace-service-');
    addTearDown(() async {
      if (await tempDir.exists()) {
        await tempDir.delete(recursive: true);
      }
    });

    final source = Directory(path.join(tempDir.path, 'source'))..createSync();
    await File(path.join(source.path, 'pubspec.yaml')).writeAsString('name: source\n');
    await Process.run('git', ['init'], workingDirectory: source.path);
    await Process.run('git', ['add', '.'], workingDirectory: source.path);
    await Process.run(
      'git',
      ['-c', 'user.email=test@example.com', '-c', 'user.name=Test', 'commit', '-m', 'init'],
      workingDirectory: source.path,
    );

    final workspaceRoot = path.join(tempDir.path, 'workspaces');
    final unit = BuildUnit(
      app: 'todo',
      target: BuildTarget(platform: 'android', artifactType: 'apk'),
    );

    final workspace = await BuildWorkspaceService().prepareWorkspace(
      sourceProjectDir: source.path,
      workspaceRoot: workspaceRoot,
      unit: unit,
    );

    expect(workspace, path.join(workspaceRoot, 'todo', 'android', 'apk'));
    expect(await File(path.join(workspace, 'pubspec.yaml')).readAsString(), 'name: source\n');
    expect(await Directory(path.join(workspace, '.git')).exists(), isTrue);
  });
}
```

- [ ] **Step 4: Run workspace clone test to verify it fails**

Run:

```bash
cd packages/tools/felorx_builder_cli && dart test test/build_workspace_service_test.dart
```

Expected: FAIL because `BuildWorkspaceService` does not exist.

- [ ] **Step 5: Implement path resolver and clone service**

Implement `build_workspace_path_resolver.dart`:

```dart
import 'package:path/path.dart' as path;

class BuildWorkspacePathResolver {
  String resolve({
    required String root,
    required String appName,
    required String platform,
    required String artifactType,
  }) {
    return path.join(
      root,
      sanitizeSegment(appName),
      sanitizeSegment(platform),
      sanitizeSegment(artifactType),
    );
  }

  String sanitizeSegment(String value) {
    final trimmed = value.trim();
    final normalized = trimmed.isEmpty ? 'unknown' : trimmed;
    final replaced = normalized.replaceAll(RegExp(r'[<>:"/\\|?*\x00-\x1F]+'), '_');
    final collapsed = replaced.replaceAll(RegExp(r'_+'), '_');
    final withoutEdgeDots = collapsed.replaceAll(RegExp(r'^\.+|\.+$'), '');
    return withoutEdgeDots.isEmpty ? 'unknown' : withoutEdgeDots;
  }
}
```

Implement `build_workspace_service.dart`:

```dart
import 'dart:io';

import '../models/build_unit.dart';
import 'build_workspace_path_resolver.dart';

class BuildWorkspaceService {
  final BuildWorkspacePathResolver pathResolver;

  BuildWorkspaceService({BuildWorkspacePathResolver? pathResolver})
    : pathResolver = pathResolver ?? BuildWorkspacePathResolver();

  Future<String> prepareWorkspace({
    required String sourceProjectDir,
    required String workspaceRoot,
    required BuildUnit unit,
    void Function(String message)? onLog,
  }) async {
    final workspaceDir = pathResolver.resolve(
      root: workspaceRoot,
      appName: unit.app,
      platform: unit.platform,
      artifactType: unit.artifactType,
    );
    final workspace = Directory(workspaceDir);
    await workspace.parent.create(recursive: true);

    if (!await workspace.exists()) {
      onLog?.call('克隆项目到构建目录: $workspaceDir');
      await _runGit(
        ['clone', '--local', '--no-hardlinks', sourceProjectDir, workspaceDir],
        workingDirectory: workspace.parent.path,
        errorMessage: '克隆项目失败',
      );
    } else if (!await _isGitWorkspace(workspaceDir)) {
      await workspace.delete(recursive: true);
      await _runGit(
        ['clone', '--local', '--no-hardlinks', sourceProjectDir, workspaceDir],
        workingDirectory: workspace.parent.path,
        errorMessage: '重新克隆项目失败',
      );
    }

    await _syncToSourceHead(
      sourceProjectDir: sourceProjectDir,
      workspaceDir: workspaceDir,
      onLog: onLog,
    );
    return workspaceDir;
  }

  Future<void> _syncToSourceHead({
    required String sourceProjectDir,
    required String workspaceDir,
    void Function(String message)? onLog,
  }) async {
    final head = await _gitOutput(
      ['rev-parse', 'HEAD'],
      workingDirectory: sourceProjectDir,
      errorMessage: '无法读取源项目 HEAD',
    );
    await _runGit(
      ['remote', 'set-url', 'origin', sourceProjectDir],
      workingDirectory: workspaceDir,
      errorMessage: '设置构建目录 origin 失败',
    );
    await _runGit(
      ['fetch', '--prune', 'origin', '+refs/heads/*:refs/remotes/origin/*', '+refs/tags/*:refs/tags/*'],
      workingDirectory: workspaceDir,
      errorMessage: '同步源项目 refs 失败',
    );
    await _runGit(
      ['checkout', '--detach', head],
      workingDirectory: workspaceDir,
      errorMessage: '切换构建目录 HEAD 失败',
    );
    await _runGit(
      ['reset', '--hard'],
      workingDirectory: workspaceDir,
      errorMessage: '重置构建目录失败',
    );
    await _runGit(
      ['clean', '-fd'],
      workingDirectory: workspaceDir,
      errorMessage: '清理构建目录失败',
    );
    onLog?.call('构建目录已同步到 ${head.length > 7 ? head.substring(0, 7) : head}');
  }

  Future<bool> _isGitWorkspace(String workspaceDir) async {
    final result = await Process.run(
      'git',
      ['rev-parse', '--is-inside-work-tree'],
      workingDirectory: workspaceDir,
    );
    return result.exitCode == 0 && result.stdout.toString().trim() == 'true';
  }

  Future<String> _gitOutput(
    List<String> args, {
    required String workingDirectory,
    required String errorMessage,
  }) async {
    final result = await Process.run('git', args, workingDirectory: workingDirectory);
    if (result.exitCode != 0) {
      throw Exception('$errorMessage: ${result.stderr.toString().trim()}');
    }
    return result.stdout.toString().trim();
  }

  Future<void> _runGit(
    List<String> args, {
    required String workingDirectory,
    required String errorMessage,
  }) async {
    final result = await Process.run('git', args, workingDirectory: workingDirectory);
    if (result.exitCode != 0) {
      throw Exception('$errorMessage: ${result.stderr.toString().trim()}');
    }
  }
}
```

Add exports:

```dart
export 'src/services/build_workspace_path_resolver.dart';
export 'src/services/build_workspace_service.dart';
```

- [ ] **Step 6: Run workspace tests to verify they pass**

Run:

```bash
cd packages/tools/felorx_builder_cli && dart test test/build_workspace_path_resolver_test.dart test/build_workspace_service_test.dart
```

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add packages/tools/felorx_builder_cli/lib/src/services/build_workspace_path_resolver.dart \
  packages/tools/felorx_builder_cli/lib/src/services/build_workspace_service.dart \
  packages/tools/felorx_builder_cli/lib/felorx_builder_cli.dart \
  packages/tools/felorx_builder_cli/test/build_workspace_path_resolver_test.dart \
  packages/tools/felorx_builder_cli/test/build_workspace_service_test.dart
git commit -m "feat(builder): 增加独立构建目录服务"
```

---

### Task 3: GUI BuildService Workspace Isolation

**Files:**
- Modify: `apps/builder/lib/services/build_service.dart`
- Create: `apps/builder/test/services/build_service_workspace_test.dart`

- [ ] **Step 1: Write the failing GUI workspace test**

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:path/path.dart' as path;
import 'package:felorx_builder/services/build_service.dart';

void main() {
  test('BuildService 为不同制品生成不同构建目录', () {
    final service = BuildService(
      maxConcurrentBuilds: 1,
      felorxBuilderPath: 'felorx-builder',
    );

    final apkWorkspace = service.resolveBuildWorkspaceDir(
      projectRootDir: '/repo/felorx-apps',
      appName: 'todo',
      platform: 'android',
      artifactType: 'apk',
    );
    final aabWorkspace = service.resolveBuildWorkspaceDir(
      projectRootDir: '/repo/felorx-apps',
      appName: 'todo',
      platform: 'android',
      artifactType: 'aab',
    );

    expect(apkWorkspace, endsWith(path.join('todo', 'android', 'apk')));
    expect(aabWorkspace, endsWith(path.join('todo', 'android', 'aab')));
    expect(apkWorkspace, isNot(aabWorkspace));
  });
}
```

- [ ] **Step 2: Run GUI workspace test to verify it fails**

Run:

```bash
cd apps/builder && flutter test test/services/build_service_workspace_test.dart
```

Expected: FAIL because `resolveBuildWorkspaceDir` does not exist.

- [ ] **Step 3: Modify BuildService to include app/platform/artifactType in workspace paths**

In `apps/builder/lib/services/build_service.dart`, import the shared resolver:

```dart
import 'package:felorx_builder_cli/felorx_builder_cli.dart'
    show ArtifactArchiveService, ArtifactPathResolver, BuildWorkspacePathResolver, PlatformMapper;
```

Add the testable resolver method:

```dart
  @visibleForTesting
  String resolveBuildWorkspaceDir({
    required String projectRootDir,
    required String appName,
    required String platform,
    required String artifactType,
  }) {
    final projectWorkspaceRoot = _buildWorkspaceRootForProject(projectRootDir);
    return BuildWorkspacePathResolver().resolve(
      root: projectWorkspaceRoot,
      appName: appName,
      platform: platform,
      artifactType: artifactType,
    );
  }
```

Rename the old project-level method:

```dart
  String _buildWorkspaceRootForProject(String projectRootDir) {
    final projectName = path.basename(path.normalize(projectRootDir));
    final hash = _stableHash(projectRootDir);
    return path.join(_buildWorkspaceParentPath(), '${projectName}_$hash');
  }
```

In `executeBuild`, replace the workspace selection:

```dart
      final artifactType = targets.isEmpty ? 'unknown' : targets.first;
      buildWorkspaceDir = resolveBuildWorkspaceDir(
        projectRootDir: projectRootDir,
        appName: appName,
        platform: platform,
        artifactType: artifactType,
      );
```

Keep `_acquireBuildWorkspace(buildWorkspaceDir)` as-is so the lock is now per unit directory.

- [ ] **Step 4: Run GUI workspace test to verify it passes**

Run:

```bash
cd apps/builder && flutter test test/services/build_service_workspace_test.dart
```

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add apps/builder/lib/services/build_service.dart apps/builder/test/services/build_service_workspace_test.dart
git commit -m "feat(builder): 按制品隔离 GUI 构建目录"
```

---

### Task 4: BuildCoordinator Target Failure Isolation

**Files:**
- Create: `packages/tools/felorx_builder_cli/lib/src/services/build_target_failure_isolator.dart`
- Modify: `packages/tools/felorx_builder_cli/lib/src/services/build_coordinator.dart`
- Test: `packages/tools/felorx_builder_cli/test/build_target_failure_isolator_test.dart`

- [ ] **Step 1: Write the failing failure-isolator test**

```dart
import 'package:felorx_builder_cli/src/models/build_target.dart';
import 'package:felorx_builder_cli/src/services/build_target_failure_isolator.dart';
import 'package:test/test.dart';

void main() {
  test('target 失败后继续运行后续 target 并返回失败列表', () async {
    final isolator = BuildTargetFailureIsolator();
    final targets = [
      BuildTarget(platform: 'android', artifactType: 'apk'),
      BuildTarget(platform: 'android', artifactType: 'aab'),
      BuildTarget(platform: 'macos', artifactType: 'dmg'),
    ];
    final executed = <String>[];
    final markedFailures = <String>[];

    final failures = await isolator.run(
      targets: targets,
      runTarget: (target) async {
        executed.add(target.toString());
        if (target.artifactType == 'aab') {
          throw Exception('aab failed');
        }
      },
      onFailure: (target, error) async {
        markedFailures.add('${target.toString()}:$error');
      },
    );

    expect(executed, ['android:apk', 'android:aab', 'macos:dmg']);
    expect(failures, hasLength(1));
    expect(failures.single.target.artifactType, 'aab');
    expect(markedFailures.single, contains('aab failed'));
  });
}
```

- [ ] **Step 2: Run failure-isolator test to verify it fails**

Run:

```bash
cd packages/tools/felorx_builder_cli && dart test test/build_target_failure_isolator_test.dart
```

Expected: FAIL because `BuildTargetFailureIsolator` does not exist.

- [ ] **Step 3: Implement the failure isolator**

```dart
import '../models/build_target.dart';

class BuildTargetFailure {
  final BuildTarget target;
  final Object error;

  BuildTargetFailure({required this.target, required this.error});
}

class BuildTargetFailureIsolator {
  Future<List<BuildTargetFailure>> run({
    required List<BuildTarget> targets,
    required Future<void> Function(BuildTarget target) runTarget,
    required Future<void> Function(BuildTarget target, Object error) onFailure,
  }) async {
    final failures = <BuildTargetFailure>[];
    for (final target in targets) {
      try {
        await runTarget(target);
      } catch (error) {
        failures.add(BuildTargetFailure(target: target, error: error));
        await onFailure(target, error);
      }
    }
    return failures;
  }
}
```

- [ ] **Step 4: Wrap the BuildCoordinator target loop with the isolator**

In `BuildCoordinator.executeBuild`, keep setup steps 1-6 before the target loop. Replace the raw `for (final target in buildTargets)` body with:

```dart
      final failures = await BuildTargetFailureIsolator().run(
        targets: buildTargets,
        runTarget: (target) async {
          await _executeSingleTarget(
            app: app,
            target: target,
            trigger: trigger,
            normalizedEnvironment: normalizedEnvironment,
            autoCreateRelease: effectiveAutoCreateRelease,
            releasePublisher: effectiveReleasePublisher,
            releaseChannel: releaseChannel,
            releaseNotes: releaseNotes,
            bootstrap: bootstrap,
            versionInfo: versionInfo,
            gitInfo: gitInfo,
            config: config,
            existingBuildRecord: existingBuildRecord,
            appId: appId,
            appInfo: appInfo,
            buildRecordService: buildRecordService,
            publishService: publishService,
            buildResults: buildResults,
            buildRecords: buildRecords,
          );
        },
        onFailure: (target, error) async {
          Logger.error('构建目标 $target 失败: $error');
        },
      );

      if (failures.isNotEmpty) {
        final failedLabels = failures.map((failure) => failure.target.toString()).join(', ');
        throw Exception('部分构建目标失败: $failedLabels');
      }
```

Extract the previous per-target body into `_executeSingleTarget`. Preserve existing BuildRecord failed updates inside that method so each failed target is marked before the exception reaches the isolator.

- [ ] **Step 5: Run failure-isolator test**

Run:

```bash
cd packages/tools/felorx_builder_cli && dart test test/build_target_failure_isolator_test.dart
```

Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add packages/tools/felorx_builder_cli/lib/src/services/build_target_failure_isolator.dart \
  packages/tools/felorx_builder_cli/lib/src/services/build_coordinator.dart \
  packages/tools/felorx_builder_cli/test/build_target_failure_isolator_test.dart
git commit -m "refactor(builder): 隔离多制品构建失败"
```

---

### Task 5: CLI Build Command Orchestrator Mode

**Files:**
- Create: `packages/tools/felorx_builder_cli/lib/src/services/build_unit_process_command_builder.dart`
- Modify: `packages/tools/felorx_builder_cli/lib/src/build_command.dart`
- Test: `packages/tools/felorx_builder_cli/test/build_unit_process_command_builder_test.dart`

- [ ] **Step 1: Write the failing child command builder test**

```dart
import 'package:felorx_builder_cli/src/models/build_target.dart';
import 'package:felorx_builder_cli/src/models/build_unit.dart';
import 'package:felorx_builder_cli/src/services/build_unit_process_command_builder.dart';
import 'package:test/test.dart';

void main() {
  test('子进程命令只包含一个平台和一个制品并启用 isolated-unit', () {
    final args = BuildUnitProcessCommandBuilder().buildArguments(
      unit: BuildUnit(
        app: 'todo',
        target: BuildTarget(platform: 'android', artifactType: 'apk'),
      ),
      configPath: 'builder.yaml',
      trigger: 'Manual',
      environment: 'prod',
      workingDir: '/tmp/workspaces/todo/android/apk',
      archiveRootDir: '/repo/source',
      fastforgePath: 'fastforge',
      flutterPath: 'flutter',
      verbose: true,
    );

    expect(args, contains('--isolated-unit'));
    expect(args, containsAllInOrder(['--app', 'todo']));
    expect(args, containsAllInOrder(['--platform', 'android']));
    expect(args, containsAllInOrder(['--targets', 'apk']));
    expect(args, containsAllInOrder(['--working-dir', '/tmp/workspaces/todo/android/apk']));
    expect(args, containsAllInOrder(['--archive-root', '/repo/source']));
    expect(args, isNot(contains('apk,aab')));
  });
}
```

- [ ] **Step 2: Run child command builder test to verify it fails**

Run:

```bash
cd packages/tools/felorx_builder_cli && dart test test/build_unit_process_command_builder_test.dart
```

Expected: FAIL because `BuildUnitProcessCommandBuilder` does not exist.

- [ ] **Step 3: Implement child command builder**

Implement `build_unit_process_command_builder.dart` with a complete argument builder:

```dart
import '../models/build_unit.dart';

class BuildUnitProcessCommandBuilder {
  List<String> buildArguments({
    required BuildUnit unit,
    required String configPath,
    required String trigger,
    required String environment,
    required String workingDir,
    required String archiveRootDir,
    required String fastforgePath,
    required String flutterPath,
    required bool verbose,
    String? tag,
    bool autoCreateRelease = false,
    String? releasePublisher,
    String? releaseChannel,
    String? releaseNotes,
    bool bootstrap = false,
    bool skipBuildRecord = false,
    String? buildRecordId,
    String? versionFromArgs,
    String? apiUrl,
    String? accessToken,
    String? tokenType,
    String? syncApiUrl,
    String? deviceId,
    String? userId,
  }) {
    final args = <String>[
      'run',
      'packages/tools/felorx_builder_cli/bin/felorx_builder_cli.dart',
      'build',
      '--isolated-unit',
      '--app',
      unit.app,
      '--config',
      configPath,
      '--platform',
      unit.platform,
      '--targets',
      unit.artifactType,
      '--trigger',
      trigger,
      '--environment',
      environment,
      '--working-dir',
      workingDir,
      '--archive-root',
      archiveRootDir,
      '--fastforge-path',
      fastforgePath,
      '--flutter-path',
      flutterPath,
    ];

    if (tag != null) args.addAll(['--tag', tag]);
    if (autoCreateRelease) args.add('--auto-create-release');
    if (releasePublisher != null) args.addAll(['--release', releasePublisher]);
    if (releaseChannel != null) args.addAll(['--release-channel', releaseChannel]);
    if (releaseNotes != null) args.addAll(['--release-notes', releaseNotes]);
    if (bootstrap) args.add('--bootstrap');
    if (verbose) args.add('--verbose');
    if (skipBuildRecord) args.add('--skip-build-record');
    if (buildRecordId != null) args.addAll(['--build-record-id', buildRecordId]);
    if (versionFromArgs != null) args.addAll(['--version', versionFromArgs]);
    if (apiUrl != null) args.addAll(['--api-url', apiUrl]);
    if (accessToken != null) args.addAll(['--access-token', accessToken]);
    if (tokenType != null) args.addAll(['--token-type', tokenType]);
    if (syncApiUrl != null) args.addAll(['--sync-api-url', syncApiUrl]);
    if (deviceId != null) args.addAll(['--device-id', deviceId]);
    if (userId != null) args.addAll(['--user-id', userId]);

    return args;
  }
}
```

- [ ] **Step 4: Modify BuildCommand argument parser and execution flow**

In `BuildCommand` constructor, add a hidden flag:

```dart
      ..addFlag(
        'isolated-unit',
        help: '内部参数：当前进程只执行一个已隔离 workspace 中的构建单元',
        defaultsTo: false,
        hide: true,
      );
```

In `run`, read:

```dart
    final isolatedUnit = argResults!['isolated-unit'] as bool;
```

Change flow:

- If `isolatedUnit` is true, require one app, one platform, and one target; initialize API client and call `BuildCoordinator.executeBuild` directly with a single `BuildTarget`.
- If `isolatedUnit` is false, resolve all `BuildUnit`s with `BuildUnitPlanner`.
- If `buildRecordId != null && units.length != 1`, print an error and `exit(1)`.
- Prepare each workspace with `BuildWorkspaceService`.
- Run child processes through `BuildUnitScheduler` and `BuildUnitProcessCommandBuilder`.
- Print failed units and `exit(1)` after all results finish when any result failed.

Use this exact summary condition:

```dart
    final failedResults = results.where((result) => !result.success).toList();
    if (failedResults.isNotEmpty) {
      Logger.error('以下构建单元失败:');
      for (final result in failedResults) {
        Logger.error('  ${result.unit}: ${result.errorMessage}');
      }
      exit(1);
    }
```

- [ ] **Step 5: Run child command builder test**

Run:

```bash
cd packages/tools/felorx_builder_cli && dart test test/build_unit_process_command_builder_test.dart
```

Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add packages/tools/felorx_builder_cli/lib/src/services/build_unit_process_command_builder.dart \
  packages/tools/felorx_builder_cli/lib/src/build_command.dart \
  packages/tools/felorx_builder_cli/test/build_unit_process_command_builder_test.dart
git commit -m "feat(builder): 让 build 命令按构建单元隔离执行"
```

---

### Task 6: CLI Build-All Independent Unit Scheduling

**Files:**
- Modify: `packages/tools/felorx_builder_cli/lib/src/build_all_command.dart`
- Test: `packages/tools/felorx_builder_cli/test/build_all_units_test.dart`

- [ ] **Step 1: Write the failing build-all unit test**

```dart
import 'package:felorx_builder_cli/src/build_all_command.dart';
import 'package:test/test.dart';

void main() {
  test('macOS build-all 为每个应用生成 android apk 和 macos dmg 平级构建单元', () {
    final units = buildAllUnitsForOperatingSystem(
      apps: ['todo', 'note'],
      operatingSystem: 'macos',
    );

    expect(
      units.map((unit) => unit.toString()).toList(),
      [
        'todo:android:apk',
        'todo:macos:dmg',
        'note:android:apk',
        'note:macos:dmg',
      ],
    );
  });

  test('Windows build-all 为每个应用生成 windows exe 构建单元', () {
    final units = buildAllUnitsForOperatingSystem(
      apps: ['todo', 'note'],
      operatingSystem: 'windows',
    );

    expect(
      units.map((unit) => unit.toString()).toList(),
      ['todo:windows:exe', 'note:windows:exe'],
    );
  });
}
```

- [ ] **Step 2: Run build-all unit test to verify it fails**

Run:

```bash
cd packages/tools/felorx_builder_cli && dart test test/build_all_units_test.dart
```

Expected: FAIL because `buildAllUnitsForOperatingSystem` does not exist.

- [ ] **Step 3: Add build-all unit generation helper**

In `build_all_command.dart`, add:

```dart
List<BuildUnit> buildAllUnitsForOperatingSystem({
  required List<String> apps,
  required String operatingSystem,
}) {
  final units = <BuildUnit>[];
  for (final app in apps) {
    if (operatingSystem == 'macos') {
      units.add(BuildUnit(app: app, target: BuildTarget(platform: 'android', artifactType: 'apk')));
      units.add(BuildUnit(app: app, target: BuildTarget(platform: 'macos', artifactType: 'dmg')));
    } else if (operatingSystem == 'windows') {
      units.add(BuildUnit(app: app, target: BuildTarget(platform: 'windows', artifactType: 'exe')));
    } else {
      throw UnsupportedError('不支持的操作系统: $operatingSystem');
    }
  }
  return units;
}
```

Import:

```dart
import 'models/build_target.dart';
import 'models/build_unit.dart';
import 'services/build_unit_process_command_builder.dart';
import 'services/build_unit_scheduler.dart';
import 'services/build_workspace_service.dart';
import 'utils/semaphore.dart';
```

Remove the old inline `Semaphore` class from `build_all_command.dart`; `BuildAllCommand` should use `utils/semaphore.dart` through `BuildUnitScheduler`.

- [ ] **Step 4: Replace build-all app-level scheduling with unit-level scheduling**

In `BuildAllCommand.run`, after app discovery:

```dart
    final units = buildAllUnitsForOperatingSystem(
      apps: apps,
      operatingSystem: Platform.operatingSystem,
    );
```

Print unit count and jobs:

```dart
    print('构建单元数: ${units.length}');
    print('并行数: $jobs');
```

Replace `_buildAppsInParallel` usage with:

```dart
    final sourceRoot = Directory.current.path;
    final workspaceRoot = _buildWorkspaceRootForProject(sourceRoot);
    final scheduler = BuildUnitScheduler();
    final workspaceService = BuildWorkspaceService();
    final commandBuilder = BuildUnitProcessCommandBuilder();

    final results = await scheduler.run(
      units: units,
      jobs: jobs,
      runUnit: (unit) async {
        final workspaceDir = await workspaceService.prepareWorkspace(
          sourceProjectDir: sourceRoot,
          workspaceRoot: workspaceRoot,
          unit: unit,
          onLog: Logger.verbose,
        );
        final args = commandBuilder.buildArguments(
          unit: unit,
          configPath: 'builder.yaml',
          trigger: finalTrigger,
          environment: finalEnvironment,
          workingDir: workspaceDir,
          archiveRootDir: sourceRoot,
          fastforgePath: 'fastforge',
          flutterPath: 'flutter',
          releasePublisher: releasePublisher,
          verbose: verbose,
        );
        final exitCode = await runCommandWithRealtimeOutput('dart', args, runInShell: true);
        if (exitCode != 0) {
          throw Exception('退出码: $exitCode');
        }
      },
    );
```

Add helper near the command class:

```dart
String _buildWorkspaceRootForProject(String projectRootDir) {
  final projectName = path.basename(path.normalize(projectRootDir));
  final hash = _stableHash(projectRootDir);
  return path.join(_buildWorkspaceParentPath(), '${projectName}_$hash');
}
```

Use the same `_stableHash` and workspace parent rules as GUI `BuildService`.

- [ ] **Step 5: Run build-all unit test**

Run:

```bash
cd packages/tools/felorx_builder_cli && dart test test/build_all_units_test.dart
```

Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add packages/tools/felorx_builder_cli/lib/src/build_all_command.dart \
  packages/tools/felorx_builder_cli/test/build_all_units_test.dart
git commit -m "feat(builder): 让 build-all 按构建单元并行执行"
```

---

### Task 7: Legacy Wrapper Scripts Delegate to CLI

**Files:**
- Modify: `scripts/build_all.dart`

- [ ] **Step 1: Replace serial wrapper logic with CLI delegation**

Rewrite `scripts/build_all.dart` to parse the existing public flags and forward them to CLI `build-all`:

```dart
#!/usr/bin/env dart

import 'dart:convert';
import 'dart:io';

String formatCommand(String executable, List<String> arguments) {
  final parts = [executable, ...arguments];
  return parts
      .map((part) => part.contains(RegExp(r'\s')) ? '"${part.replaceAll('"', '\\"')}"' : part)
      .join(' ');
}

Future<int> runCommandWithRealtimeOutput(
  String executable,
  List<String> arguments, {
  bool runInShell = false,
}) async {
  print('执行命令: ${formatCommand(executable, arguments)}');
  final process = await Process.start(
    executable,
    arguments,
    runInShell: runInShell,
    mode: ProcessStartMode.normal,
  );
  process.stdout.transform(const Utf8Decoder()).listen(stdout.write);
  process.stderr.transform(const Utf8Decoder()).listen(stderr.write);
  return process.exitCode;
}

void main(List<String> args) async {
  final forwarded = <String>[
    'run',
    'packages/tools/felorx_builder_cli/bin/felorx_builder_cli.dart',
    'build-all',
  ];

  for (var i = 0; i < args.length; i++) {
    final arg = args[i];
    switch (arg) {
      case '--release':
        forwarded.addAll(['--release', 'felorx']);
        break;
      case '--environment':
      case '--trigger':
      case '--jobs':
      case '-j':
        if (i + 1 >= args.length || args[i + 1].startsWith('--')) {
          stderr.writeln('错误: $arg 需要提供一个值');
          exit(1);
        }
        forwarded.add(arg);
        forwarded.add(args[++i]);
        break;
      case '--verbose':
      case '--help':
      case '-h':
        forwarded.add(arg);
        break;
      default:
        if (arg.startsWith('--')) {
          stderr.writeln('错误: 未知参数: $arg');
          exit(1);
        }
    }
  }

  final exitCode = await runCommandWithRealtimeOutput(
    'dart',
    forwarded,
    runInShell: true,
  );
  exit(exitCode);
}
```

- [ ] **Step 2: Analyze the wrapper**

Run:

```bash
dart analyze scripts/build_all.dart
```

Expected: PASS.

- [ ] **Step 3: Commit**

```bash
git add scripts/build_all.dart
git commit -m "refactor(builder): 让旧 build-all 脚本委托 CLI"
```

---

### Task 8: Full Verification

**Files:**
- Read: `docs/superpowers/specs/2026-06-14-builder-isolated-parallel-builds-design.md`
- Verify changed code and tests.

- [ ] **Step 1: Run CLI tests**

Run:

```bash
cd packages/tools/felorx_builder_cli && dart test
```

Expected: PASS with all builder CLI tests passing.

- [ ] **Step 2: Run CLI analyzer**

Run:

```bash
cd packages/tools/felorx_builder_cli && dart analyze
```

Expected: PASS with no errors.

- [ ] **Step 3: Run GUI workspace test**

Run:

```bash
cd apps/builder && flutter test test/services/build_service_workspace_test.dart
```

Expected: PASS.

- [ ] **Step 4: Run GUI analyzer**

Run:

```bash
cd apps/builder && dart analyze
```

Expected: PASS with no errors.

- [ ] **Step 5: Re-read acceptance criteria**

Check each item from `docs/superpowers/specs/2026-06-14-builder-isolated-parallel-builds-design.md`:

- Same app with multiple artifacts maps to different workspaces.
- Multiple apps/platforms/artifacts are scheduled with configured concurrency.
- Failed units do not stop queued or running units.
- Logs, BuildRecord state, and artifact path remain scoped to one unit.
- CLI summary lists failed units and exits with code 1 when any failed.

- [ ] **Step 6: Commit final verification fixes if any**

If verification required code fixes, stage only files changed by this implementation:

```bash
git status --short
git add packages/tools/felorx_builder_cli/lib/felorx_builder_cli.dart \
  packages/tools/felorx_builder_cli/lib/src/build_all_command.dart \
  packages/tools/felorx_builder_cli/lib/src/build_command.dart \
  packages/tools/felorx_builder_cli/lib/src/models/build_unit.dart \
  packages/tools/felorx_builder_cli/lib/src/models/build_unit_result.dart \
  packages/tools/felorx_builder_cli/lib/src/services/build_coordinator.dart \
  packages/tools/felorx_builder_cli/lib/src/services/build_target_failure_isolator.dart \
  packages/tools/felorx_builder_cli/lib/src/services/build_unit_planner.dart \
  packages/tools/felorx_builder_cli/lib/src/services/build_unit_process_command_builder.dart \
  packages/tools/felorx_builder_cli/lib/src/services/build_unit_scheduler.dart \
  packages/tools/felorx_builder_cli/lib/src/services/build_workspace_path_resolver.dart \
  packages/tools/felorx_builder_cli/lib/src/services/build_workspace_service.dart \
  packages/tools/felorx_builder_cli/lib/src/utils/semaphore.dart \
  packages/tools/felorx_builder_cli/test/build_all_units_test.dart \
  packages/tools/felorx_builder_cli/test/build_target_failure_isolator_test.dart \
  packages/tools/felorx_builder_cli/test/build_unit_planner_test.dart \
  packages/tools/felorx_builder_cli/test/build_unit_process_command_builder_test.dart \
  packages/tools/felorx_builder_cli/test/build_unit_scheduler_test.dart \
  packages/tools/felorx_builder_cli/test/build_workspace_path_resolver_test.dart \
  packages/tools/felorx_builder_cli/test/build_workspace_service_test.dart \
  apps/builder/lib/services/build_service.dart \
  apps/builder/test/services/build_service_workspace_test.dart \
  scripts/build_all.dart
git commit -m "fix(builder): 修正独立并行构建验证问题"
```

If no fixes were needed, do not create an empty commit.
