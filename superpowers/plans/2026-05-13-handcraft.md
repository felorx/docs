# 小汪手工 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the new `handcraft` Flutter sub-application so users can create synced handcraft projects, upload a style image and fabric image, fill or estimate dimensions, and run a complete mock generation flow for preview image, pattern image, and process video reference.

**Architecture:** Scaffold `apps/puupee/handcraft` as a first-class Puupee sub-application using `runMyApp`, typed `go_router`, Riverpod annotation providers, shadcn Flutter UI, and Android project conventions copied from the existing Puupee apps. Keep all persistence behind `HandcraftProjectRepo`, keep generation behind `HandcraftGenerationService`, and use an in-memory first-version data source shaped for later Puupee/Sync and object-storage replacement.

**Tech Stack:** Dart 3.8, Flutter, shadcn_flutter, go_router with go_router_builder, Riverpod annotation, hooks_riverpod, puupee_shared, puupee_ui, build_runner, Android Kotlin/Java 17.

---

## File Structure

- Modify `pubspec.yaml`: add `apps/puupee/handcraft` to the workspace list near other Puupee apps.
- Create `apps/puupee/handcraft/pubspec.yaml`: Flutter package definition for `puupee_handcraft`.
- Create `apps/puupee/handcraft/.metadata`: Flutter app metadata so app-level tests and tooling recognize the project.
- Create `apps/puupee/handcraft/lib/env.dart`: `HandcraftEnvConfig`.
- Create `apps/puupee/handcraft/lib/main.dart`: startup through `runMyApp`.
- Create `apps/puupee/handcraft/lib/router.dart`: typed shell routes for studio, projects, assets, and settings.
- Generate `apps/puupee/handcraft/lib/router.g.dart`.
- Create `apps/puupee/handcraft/lib/components/adaptive_handcraft_shell.dart`: desktop sidebar and mobile bottom navigation.
- Create `apps/puupee/handcraft/lib/components/handcraft_project_card.dart`: project history card.
- Create `apps/puupee/handcraft/lib/components/handcraft_media_input_card.dart`: path-based media input card.
- Create `apps/puupee/handcraft/lib/components/handcraft_dimension_editor.dart`: item type and dimension editor.
- Create `apps/puupee/handcraft/lib/components/handcraft_generation_panel.dart`: stage and result preview panel.
- Create `apps/puupee/handcraft/lib/models/handcraft_project.dart`: project entity, enums, dimension helpers, and copy helpers.
- Create `apps/puupee/handcraft/lib/repo/handcraft_project_repo.dart`: repo interface and in-memory implementation.
- Create `apps/puupee/handcraft/lib/services/handcraft_generation_service.dart`: mock generation service.
- Create `apps/puupee/handcraft/lib/providers/handcraft_providers.dart`: repo, project stream, current project, and generation controller providers.
- Generate `apps/puupee/handcraft/lib/providers/handcraft_providers.g.dart`.
- Create `apps/puupee/handcraft/lib/pages/handcraft_studio_page.dart`: mobile wizard and desktop workbench.
- Create `apps/puupee/handcraft/lib/pages/handcraft_projects_page.dart`: project history list.
- Create `apps/puupee/handcraft/lib/pages/handcraft_assets_page.dart`: lightweight media reference list.
- Create `apps/puupee/handcraft/README.md`: first-version scope and verification commands.
- Create tests under `apps/puupee/handcraft/test/`: env, model, generation service, repo/provider, and page smoke tests.
- Create Android files under `apps/puupee/handcraft/android/`: Gradle config, manifest, localized strings, Kotlin `MainActivity`, launch resources, FileProvider XML, and launcher assets copied from an existing Puupee app.

---

### Task 1: Scaffold App Package And EnvConfig

**Files:**
- Modify: `pubspec.yaml`
- Create: `apps/puupee/handcraft/pubspec.yaml`
- Create: `apps/puupee/handcraft/.metadata`
- Create: `apps/puupee/handcraft/lib/env.dart`
- Create: `apps/puupee/handcraft/lib/main.dart`
- Test: `apps/puupee/handcraft/test/env_test.dart`

- [ ] **Step 1: Write the failing EnvConfig test**

Create `apps/puupee/handcraft/test/env_test.dart`:

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:puupee_handcraft/env.dart';

void main() {
  test('HandcraftEnvConfig provides required split layout defaults', () {
    final env = HandcraftEnvConfig();

    expect(env.appId, 'handcraft');
    expect(env.appTitle, '小汪手工');
    expect(env.defaultLeftFlex, 0.5);
    expect(env.defaultRightFlex, 0.5);
    expect(env.defaultLeftMinWidth, 250);
    expect(env.defaultRightMinWidth, 400);
    expect(env.defaultLeftSize, isNull);
    expect(env.defaultRightSize, isNull);
  });
}
```

- [ ] **Step 2: Run the test and verify it fails because the app package does not exist**

Run:

```bash
cd apps/puupee/handcraft
flutter test test/env_test.dart --no-pub
```

Expected: command cannot run yet because `apps/puupee/handcraft` or `pubspec.yaml` is missing.

- [ ] **Step 3: Add the app to the workspace**

Modify root `pubspec.yaml` workspace list by inserting this entry after `apps/puupee/builder`:

```yaml
  - apps/puupee/handcraft
```

- [ ] **Step 4: Create the Flutter package manifest**

Create `apps/puupee/handcraft/pubspec.yaml`:

```yaml
name: puupee_handcraft
resolution: workspace
version: 0.1.0
publish_to: none
description: 小汪手工 - 布料成品效果、裁剪图和制作过程模拟生成工具

environment:
  sdk: ^3.8.1

dependencies:
  flutter:
    sdk: flutter
  flutter_localizations:
    sdk: flutter

  puupee_shared: ^0.0.41
  puupee_ui: ^0.0.3+3
  puupee_utilities: ^0.0.13+3
  puupee: ^1.1.3
  puupee_sync: ^0.0.30+2

  flutter_riverpod: ^2.5.1
  hooks_riverpod: ^2.5.1
  riverpod: ^2.5.1
  riverpod_annotation: ^2.3.5

  go_router: ^14.2.0
  shadcn_flutter: ^0.0.52

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^6.0.0
  build_runner: ^2.4.15
  riverpod_generator: ^2.6.5
  go_router_builder: 2.8.2

flutter:
  uses-material-design: true
  generate: true
```

- [ ] **Step 5: Create Flutter metadata**

Create `apps/puupee/handcraft/.metadata`:

```yaml
# This file tracks properties of this Flutter project.
# Used by Flutter tool to assess capabilities and perform upgrades etc.
#
# This file should be version controlled and should not be manually edited.

version:
  revision: "adc901062556672b4138e18a4dc62a4be8f4b3c2"
  channel: "stable"

project_type: app

migration:
  platforms:
    - platform: root
      create_revision: adc901062556672b4138e18a4dc62a4be8f4b3c2
      base_revision: adc901062556672b4138e18a4dc62a4be8f4b3c2
    - platform: android
      create_revision: adc901062556672b4138e18a4dc62a4be8f4b3c2
      base_revision: adc901062556672b4138e18a4dc62a4be8f4b3c2

  unmanaged_files:
    - 'lib/main.dart'
```

- [ ] **Step 6: Implement EnvConfig**

Create `apps/puupee/handcraft/lib/env.dart`:

```dart
import 'package:puupee_shared/env.dart';

/// 小汪手工应用环境配置。
class HandcraftEnvConfig extends EnvConfig {
  HandcraftEnvConfig()
    : super(
        env: 'production',
        appId: 'handcraft',
        appTitle: '小汪手工',
        apiUrl: const String.fromEnvironment(
          'API_URL',
          defaultValue: 'https://api.puupee.com',
        ),
        authUrl: const String.fromEnvironment(
          'AUTH_URL',
          defaultValue: 'https://auth.puupee.com',
        ),
        authClientId: const String.fromEnvironment(
          'AUTH_CLIENT_ID',
          defaultValue: 'Puupee_Sync_Node',
        ),
        feedbackUrl: const String.fromEnvironment(
          'FEEDBACK_URL',
          defaultValue: 'https://feedback.puupee.com',
        ),
        defaultLeftFlex: 0.5,
        defaultRightFlex: 0.5,
        defaultLeftMinWidth: 250,
        defaultRightMinWidth: 400,
      );
}
```

- [ ] **Step 7: Implement the app entry**

Create `apps/puupee/handcraft/lib/main.dart`:

```dart
import 'package:go_router/go_router.dart';
import 'package:puupee_handcraft/env.dart';
import 'package:puupee_handcraft/router.dart';
import 'package:puupee_shared/app/startup.dart';

/// 小汪手工应用入口。
void main() async {
  await runMyApp(
    env: HandcraftEnvConfig(),
    createRouter: (ref) =>
        GoRouter(initialLocation: '/handcraft', routes: $appRoutes),
  );
}
```

This file will not analyze until Task 5 creates `router.dart`.

- [ ] **Step 8: Run the focused EnvConfig test**

Run:

```bash
cd /Users/j/repos/puupees/puupee-apps/apps/puupee/handcraft
flutter test test/env_test.dart --no-pub
```

Expected: PASS for `HandcraftEnvConfig provides required split layout defaults`.

- [ ] **Step 9: Commit the scaffold and EnvConfig**

Run:

```bash
git add pubspec.yaml apps/puupee/handcraft/pubspec.yaml apps/puupee/handcraft/.metadata apps/puupee/handcraft/lib/env.dart apps/puupee/handcraft/lib/main.dart apps/puupee/handcraft/test/env_test.dart
git commit -m "feat(handcraft): 创建小汪手工应用骨架"
```

---

### Task 2: Add Project Model And Dimension Logic

**Files:**
- Create: `apps/puupee/handcraft/lib/models/handcraft_project.dart`
- Test: `apps/puupee/handcraft/test/handcraft_project_test.dart`

- [ ] **Step 1: Write the failing model tests**

Create `apps/puupee/handcraft/test/handcraft_project_test.dart`:

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:puupee_handcraft/models/handcraft_project.dart';

void main() {
  test('create initializes a draft project with local media upload defaults', () {
    final now = DateTime(2026, 5, 13, 9);
    final project = HandcraftProject.create(
      id: 'project-1',
      title: '帆布托特包',
      itemType: HandcraftItemType.bag,
      now: now,
    );

    expect(project.status, HandcraftProjectStatus.draft);
    expect(project.currentStage, HandcraftGenerationStage.ready);
    expect(project.styleUploadStatus, HandcraftUploadStatus.localOnly);
    expect(project.fabricUploadStatus, HandcraftUploadStatus.localOnly);
    expect(project.resultUploadStatus, HandcraftUploadStatus.localOnly);
    expect(project.createdAt, now);
    expect(project.updatedAt, now);
    expect(project.hasRequiredInputs, isFalse);
  });

  test('copyWith updates media paths and input readiness', () {
    final project = HandcraftProject.create(
      id: 'project-1',
      title: '帆布托特包',
      itemType: HandcraftItemType.bag,
      now: DateTime(2026, 5, 13, 9),
    ).copyWith(
      styleImageLocalPath: '/tmp/bag.jpg',
      fabricImageLocalPath: '/tmp/canvas.jpg',
    );

    expect(project.hasRequiredInputs, isTrue);
    expect(project.styleImageLocalPath, '/tmp/bag.jpg');
    expect(project.fabricImageLocalPath, '/tmp/canvas.jpg');
  });

  test('estimated dimensions use item specific defaults', () {
    final bag = HandcraftDimensions.estimatedFor(HandcraftItemType.bag);
    final clothing = HandcraftDimensions.estimatedFor(
      HandcraftItemType.clothing,
    );
    final other = HandcraftDimensions.estimatedFor(HandcraftItemType.other);

    expect(bag.mode, HandcraftDimensionMode.estimated);
    expect(bag.values['widthCm'], 32);
    expect(bag.values['seamAllowanceCm'], 1);
    expect(clothing.values['chestCm'], 96);
    expect(clothing.values['sleeveCm'], 58);
    expect(other.values['heightCm'], 24);
  });

  test('effectiveDimensions keeps manual values when provided', () {
    final project = HandcraftProject.create(
      id: 'project-1',
      title: '衬衫',
      itemType: HandcraftItemType.clothing,
      now: DateTime(2026, 5, 13, 9),
    ).copyWith(
      dimensions: const HandcraftDimensions(
        mode: HandcraftDimensionMode.manual,
        values: {'lengthCm': 70, 'chestCm': 102},
      ),
    );

    expect(project.effectiveDimensions.mode, HandcraftDimensionMode.manual);
    expect(project.effectiveDimensions.values['lengthCm'], 70);
    expect(project.effectiveDimensions.values['chestCm'], 102);
  });
}
```

- [ ] **Step 2: Run the model tests and verify they fail**

Run:

```bash
cd /Users/j/repos/puupees/puupee-apps/apps/puupee/handcraft
flutter test test/handcraft_project_test.dart --no-pub
```

Expected: FAIL because `models/handcraft_project.dart` does not exist.

- [ ] **Step 3: Implement the project model**

Create `apps/puupee/handcraft/lib/models/handcraft_project.dart` with these public APIs:

```dart
enum HandcraftItemType { bag, clothing, other }

enum HandcraftProjectStatus { draft, generating, completed, failed }

enum HandcraftGenerationStage {
  ready,
  renderingPreview,
  draftingPattern,
  makingVideo,
  completed,
}

enum HandcraftUploadStatus { localOnly, queued, uploading, uploaded, failed }

enum HandcraftDimensionMode { manual, estimated }

class HandcraftDimensions {
  const HandcraftDimensions({required this.mode, required this.values});

  final HandcraftDimensionMode mode;
  final Map<String, num> values;

  factory HandcraftDimensions.estimatedFor(HandcraftItemType itemType) {
    return switch (itemType) {
      HandcraftItemType.bag => const HandcraftDimensions(
        mode: HandcraftDimensionMode.estimated,
        values: {
          'widthCm': 32,
          'bodyHeightCm': 28,
          'bottomDepthCm': 10,
          'handleLengthCm': 46,
          'seamAllowanceCm': 1,
        },
      ),
      HandcraftItemType.clothing => const HandcraftDimensions(
        mode: HandcraftDimensionMode.estimated,
        values: {
          'lengthCm': 68,
          'chestCm': 96,
          'shoulderCm': 40,
          'sleeveCm': 58,
          'seamAllowanceCm': 1.2,
        },
      ),
      HandcraftItemType.other => const HandcraftDimensions(
        mode: HandcraftDimensionMode.estimated,
        values: {
          'widthCm': 24,
          'heightCm': 24,
          'depthCm': 6,
          'seamAllowanceCm': 1,
        },
      ),
    };
  }

  HandcraftDimensions copyWith({
    HandcraftDimensionMode? mode,
    Map<String, num>? values,
  }) {
    return HandcraftDimensions(
      mode: mode ?? this.mode,
      values: Map.unmodifiable(values ?? this.values),
    );
  }
}

class HandcraftProject {
  const HandcraftProject({
    required this.id,
    required this.title,
    required this.itemType,
    required this.createdAt,
    required this.updatedAt,
    required this.status,
    required this.currentStage,
    required this.styleUploadStatus,
    required this.fabricUploadStatus,
    required this.resultUploadStatus,
    this.styleImageLocalPath,
    this.styleImageRemoteUrl,
    this.fabricImageLocalPath,
    this.fabricImageRemoteUrl,
    this.previewImageLocalPath,
    this.previewImageRemoteUrl,
    this.patternImageLocalPath,
    this.patternImageRemoteUrl,
    this.processVideoLocalPath,
    this.processVideoRemoteUrl,
    this.dimensions,
    this.generationPrompt,
    this.generationSeed,
    this.errorMessage,
  });

  factory HandcraftProject.create({
    required String id,
    required String title,
    required HandcraftItemType itemType,
    DateTime? now,
  }) {
    final timestamp = now ?? DateTime.now();
    return HandcraftProject(
      id: id,
      title: title,
      itemType: itemType,
      createdAt: timestamp,
      updatedAt: timestamp,
      status: HandcraftProjectStatus.draft,
      currentStage: HandcraftGenerationStage.ready,
      styleUploadStatus: HandcraftUploadStatus.localOnly,
      fabricUploadStatus: HandcraftUploadStatus.localOnly,
      resultUploadStatus: HandcraftUploadStatus.localOnly,
    );
  }

  final String id;
  final String title;
  final HandcraftItemType itemType;
  final DateTime createdAt;
  final DateTime updatedAt;
  final HandcraftProjectStatus status;
  final HandcraftGenerationStage currentStage;
  final String? styleImageLocalPath;
  final String? styleImageRemoteUrl;
  final String? fabricImageLocalPath;
  final String? fabricImageRemoteUrl;
  final String? previewImageLocalPath;
  final String? previewImageRemoteUrl;
  final String? patternImageLocalPath;
  final String? patternImageRemoteUrl;
  final String? processVideoLocalPath;
  final String? processVideoRemoteUrl;
  final HandcraftUploadStatus styleUploadStatus;
  final HandcraftUploadStatus fabricUploadStatus;
  final HandcraftUploadStatus resultUploadStatus;
  final HandcraftDimensions? dimensions;
  final String? generationPrompt;
  final int? generationSeed;
  final String? errorMessage;

  bool get hasRequiredInputs =>
      styleImageLocalPath != null &&
      styleImageLocalPath!.trim().isNotEmpty &&
      fabricImageLocalPath != null &&
      fabricImageLocalPath!.trim().isNotEmpty;

  HandcraftDimensions get effectiveDimensions =>
      dimensions ?? HandcraftDimensions.estimatedFor(itemType);

  HandcraftProject copyWith({
    String? title,
    HandcraftItemType? itemType,
    DateTime? updatedAt,
    HandcraftProjectStatus? status,
    HandcraftGenerationStage? currentStage,
    Object? styleImageLocalPath = _sentinel,
    Object? styleImageRemoteUrl = _sentinel,
    Object? fabricImageLocalPath = _sentinel,
    Object? fabricImageRemoteUrl = _sentinel,
    Object? previewImageLocalPath = _sentinel,
    Object? previewImageRemoteUrl = _sentinel,
    Object? patternImageLocalPath = _sentinel,
    Object? patternImageRemoteUrl = _sentinel,
    Object? processVideoLocalPath = _sentinel,
    Object? processVideoRemoteUrl = _sentinel,
    HandcraftUploadStatus? styleUploadStatus,
    HandcraftUploadStatus? fabricUploadStatus,
    HandcraftUploadStatus? resultUploadStatus,
    Object? dimensions = _sentinel,
    Object? generationPrompt = _sentinel,
    Object? generationSeed = _sentinel,
    Object? errorMessage = _sentinel,
  }) {
    return HandcraftProject(
      id: id,
      title: title ?? this.title,
      itemType: itemType ?? this.itemType,
      createdAt: createdAt,
      updatedAt: updatedAt ?? DateTime.now(),
      status: status ?? this.status,
      currentStage: currentStage ?? this.currentStage,
      styleImageLocalPath: _valueOrCurrent(
        styleImageLocalPath,
        this.styleImageLocalPath,
      ),
      styleImageRemoteUrl: _valueOrCurrent(
        styleImageRemoteUrl,
        this.styleImageRemoteUrl,
      ),
      fabricImageLocalPath: _valueOrCurrent(
        fabricImageLocalPath,
        this.fabricImageLocalPath,
      ),
      fabricImageRemoteUrl: _valueOrCurrent(
        fabricImageRemoteUrl,
        this.fabricImageRemoteUrl,
      ),
      previewImageLocalPath: _valueOrCurrent(
        previewImageLocalPath,
        this.previewImageLocalPath,
      ),
      previewImageRemoteUrl: _valueOrCurrent(
        previewImageRemoteUrl,
        this.previewImageRemoteUrl,
      ),
      patternImageLocalPath: _valueOrCurrent(
        patternImageLocalPath,
        this.patternImageLocalPath,
      ),
      patternImageRemoteUrl: _valueOrCurrent(
        patternImageRemoteUrl,
        this.patternImageRemoteUrl,
      ),
      processVideoLocalPath: _valueOrCurrent(
        processVideoLocalPath,
        this.processVideoLocalPath,
      ),
      processVideoRemoteUrl: _valueOrCurrent(
        processVideoRemoteUrl,
        this.processVideoRemoteUrl,
      ),
      styleUploadStatus: styleUploadStatus ?? this.styleUploadStatus,
      fabricUploadStatus: fabricUploadStatus ?? this.fabricUploadStatus,
      resultUploadStatus: resultUploadStatus ?? this.resultUploadStatus,
      dimensions: _valueOrCurrent(dimensions, this.dimensions),
      generationPrompt: _valueOrCurrent(
        generationPrompt,
        this.generationPrompt,
      ),
      generationSeed: _valueOrCurrent(generationSeed, this.generationSeed),
      errorMessage: _valueOrCurrent(errorMessage, this.errorMessage),
    );
  }
}

const Object _sentinel = Object();

T? _valueOrCurrent<T>(Object? value, T? current) {
  if (identical(value, _sentinel)) {
    return current;
  }
  return value as T?;
}
```

- [ ] **Step 4: Run the model tests**

Run:

```bash
cd /Users/j/repos/puupees/puupee-apps/apps/puupee/handcraft
flutter test test/handcraft_project_test.dart --no-pub
```

Expected: PASS.

- [ ] **Step 5: Commit the model**

Run:

```bash
git add apps/puupee/handcraft/lib/models/handcraft_project.dart apps/puupee/handcraft/test/handcraft_project_test.dart
git commit -m "feat(handcraft): 添加手工作品模型"
```

---

### Task 3: Add Mock Generation Service

**Files:**
- Create: `apps/puupee/handcraft/lib/services/handcraft_generation_service.dart`
- Test: `apps/puupee/handcraft/test/handcraft_generation_service_test.dart`

- [ ] **Step 1: Write the failing service tests**

Create `apps/puupee/handcraft/test/handcraft_generation_service_test.dart`:

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:puupee_handcraft/models/handcraft_project.dart';
import 'package:puupee_handcraft/services/handcraft_generation_service.dart';

void main() {
  HandcraftProject readyProject() {
    return HandcraftProject.create(
      id: 'project-1',
      title: '帆布托特包',
      itemType: HandcraftItemType.bag,
      now: DateTime(2026, 5, 13, 9),
    ).copyWith(
      styleImageLocalPath: '/tmp/bag.jpg',
      fabricImageLocalPath: '/tmp/canvas.jpg',
    );
  }

  test('mock generation completes preview, pattern, and video references', () async {
    final stages = <HandcraftGenerationStage>[];
    final service = MockHandcraftGenerationService(delay: Duration.zero);

    final result = await service.generate(
      readyProject(),
      onStageChanged: stages.add,
    );

    expect(result.status, HandcraftProjectStatus.completed);
    expect(result.currentStage, HandcraftGenerationStage.completed);
    expect(result.previewImageLocalPath, 'mock://handcraft/project-1/preview.png');
    expect(result.patternImageLocalPath, 'mock://handcraft/project-1/pattern.png');
    expect(result.processVideoLocalPath, 'mock://handcraft/project-1/process.mp4');
    expect(result.resultUploadStatus, HandcraftUploadStatus.localOnly);
    expect(result.generationPrompt, contains('帆布托特包'));
    expect(result.generationSeed, isNotNull);
    expect(stages, [
      HandcraftGenerationStage.renderingPreview,
      HandcraftGenerationStage.draftingPattern,
      HandcraftGenerationStage.makingVideo,
      HandcraftGenerationStage.completed,
    ]);
  });

  test('mock generation rejects missing input media', () async {
    final service = MockHandcraftGenerationService(delay: Duration.zero);
    final project = HandcraftProject.create(
      id: 'project-1',
      title: '帆布托特包',
      itemType: HandcraftItemType.bag,
      now: DateTime(2026, 5, 13, 9),
    );

    expect(
      () => service.generate(project),
      throwsA(isA<HandcraftGenerationException>()),
    );
  });

  test('mock generation can fail at a requested stage', () async {
    final service = MockHandcraftGenerationService(
      delay: Duration.zero,
      failAtStage: HandcraftGenerationStage.draftingPattern,
    );

    await expectLater(
      service.generate(readyProject()),
      throwsA(
        isA<HandcraftGenerationException>().having(
          (error) => error.stage,
          'stage',
          HandcraftGenerationStage.draftingPattern,
        ),
      ),
    );
  });
}
```

- [ ] **Step 2: Run the service tests and verify they fail**

Run:

```bash
cd /Users/j/repos/puupees/puupee-apps/apps/puupee/handcraft
flutter test test/handcraft_generation_service_test.dart --no-pub
```

Expected: FAIL because `services/handcraft_generation_service.dart` does not exist.

- [ ] **Step 3: Implement the generation service**

Create `apps/puupee/handcraft/lib/services/handcraft_generation_service.dart`:

```dart
import 'dart:async';

import 'package:puupee_handcraft/models/handcraft_project.dart';

abstract class HandcraftGenerationService {
  Future<HandcraftProject> generate(
    HandcraftProject project, {
    FutureOr<void> Function(HandcraftGenerationStage stage)? onStageChanged,
  });
}

class HandcraftGenerationException implements Exception {
  const HandcraftGenerationException(this.message, {this.stage});

  final String message;
  final HandcraftGenerationStage? stage;

  @override
  String toString() {
    final stageText = stage == null ? '' : ' at ${stage!.name}';
    return 'HandcraftGenerationException$stageText: $message';
  }
}

class MockHandcraftGenerationService implements HandcraftGenerationService {
  const MockHandcraftGenerationService({
    this.delay = const Duration(milliseconds: 320),
    this.failAtStage,
  });

  final Duration delay;
  final HandcraftGenerationStage? failAtStage;

  @override
  Future<HandcraftProject> generate(
    HandcraftProject project, {
    FutureOr<void> Function(HandcraftGenerationStage stage)? onStageChanged,
  }) async {
    if (!project.hasRequiredInputs) {
      throw const HandcraftGenerationException('请先上传款式图和布料图');
    }

    var current = project.copyWith(
      status: HandcraftProjectStatus.generating,
      errorMessage: null,
    );

    current = await _advance(
      current,
      HandcraftGenerationStage.renderingPreview,
      onStageChanged,
    );
    current = current.copyWith(
      previewImageLocalPath: 'mock://handcraft/${project.id}/preview.png',
    );

    current = await _advance(
      current,
      HandcraftGenerationStage.draftingPattern,
      onStageChanged,
    );
    current = current.copyWith(
      patternImageLocalPath: 'mock://handcraft/${project.id}/pattern.png',
    );

    current = await _advance(
      current,
      HandcraftGenerationStage.makingVideo,
      onStageChanged,
    );
    current = current.copyWith(
      processVideoLocalPath: 'mock://handcraft/${project.id}/process.mp4',
    );

    current = await _advance(
      current,
      HandcraftGenerationStage.completed,
      onStageChanged,
    );

    return current.copyWith(
      status: HandcraftProjectStatus.completed,
      currentStage: HandcraftGenerationStage.completed,
      resultUploadStatus: HandcraftUploadStatus.localOnly,
      generationPrompt: _buildPrompt(current),
      generationSeed: current.id.hashCode.abs(),
      errorMessage: null,
    );
  }

  Future<HandcraftProject> _advance(
    HandcraftProject project,
    HandcraftGenerationStage stage,
    FutureOr<void> Function(HandcraftGenerationStage stage)? onStageChanged,
  ) async {
    if (failAtStage == stage) {
      throw HandcraftGenerationException('模拟生成在 ${stage.name} 阶段失败', stage: stage);
    }
    if (delay > Duration.zero) {
      await Future<void>.delayed(delay);
    }
    await onStageChanged?.call(stage);
    return project.copyWith(currentStage: stage);
  }

  String _buildPrompt(HandcraftProject project) {
    final dimensions = project.effectiveDimensions.values.entries
        .map((entry) => '${entry.key}=${entry.value}')
        .join(', ');
    return '使用 ${project.fabricImageLocalPath} 的面料制作 ${project.title}，尺寸参数：$dimensions';
  }
}
```

- [ ] **Step 4: Run the service tests**

Run:

```bash
cd /Users/j/repos/puupees/puupee-apps/apps/puupee/handcraft
flutter test test/handcraft_generation_service_test.dart --no-pub
```

Expected: PASS.

- [ ] **Step 5: Commit the service**

Run:

```bash
git add apps/puupee/handcraft/lib/services/handcraft_generation_service.dart apps/puupee/handcraft/test/handcraft_generation_service_test.dart
git commit -m "feat(handcraft): 添加模拟生成服务"
```

---

### Task 4: Add Repo And Riverpod Providers

**Files:**
- Create: `apps/puupee/handcraft/lib/repo/handcraft_project_repo.dart`
- Create: `apps/puupee/handcraft/lib/providers/handcraft_providers.dart`
- Generate: `apps/puupee/handcraft/lib/providers/handcraft_providers.g.dart`
- Test: `apps/puupee/handcraft/test/handcraft_project_repo_test.dart`
- Test: `apps/puupee/handcraft/test/handcraft_providers_test.dart`

- [ ] **Step 1: Write the failing repo tests**

Create `apps/puupee/handcraft/test/handcraft_project_repo_test.dart`:

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:puupee_handcraft/models/handcraft_project.dart';
import 'package:puupee_handcraft/repo/handcraft_project_repo.dart';

void main() {
  test('in-memory repo creates, updates, and streams projects', () async {
    final repo = InMemoryHandcraftProjectRepo(
      clock: () => DateTime(2026, 5, 13, 9),
      idFactory: () => 'project-1',
    );
    final emissions = <List<HandcraftProject>>[];
    final sub = repo.watchProjects().listen(emissions.add);

    final created = await repo.createProject(
      title: '帆布托特包',
      itemType: HandcraftItemType.bag,
    );
    final updated = await repo.saveProject(
      created.copyWith(styleImageLocalPath: '/tmp/bag.jpg'),
    );

    expect(updated.styleImageLocalPath, '/tmp/bag.jpg');
    expect(await repo.getProject('project-1'), updated);
    expect(emissions.last.single.id, 'project-1');

    await sub.cancel();
    repo.dispose();
  });

  test('repo filters projects by status', () async {
    final repo = InMemoryHandcraftProjectRepo(
      clock: () => DateTime(2026, 5, 13, 9),
      idFactory: () => 'project-1',
    );

    final created = await repo.createProject(
      title: '帆布托特包',
      itemType: HandcraftItemType.bag,
    );
    await repo.saveProject(
      created.copyWith(status: HandcraftProjectStatus.completed),
    );

    final completed = await repo.listProjects(
      status: HandcraftProjectStatus.completed,
    );
    final drafts = await repo.listProjects(status: HandcraftProjectStatus.draft);

    expect(completed, hasLength(1));
    expect(drafts, isEmpty);

    repo.dispose();
  });
}
```

- [ ] **Step 2: Implement the repo**

Create `apps/puupee/handcraft/lib/repo/handcraft_project_repo.dart`:

```dart
import 'dart:async';

import 'package:puupee_handcraft/models/handcraft_project.dart';

abstract class HandcraftProjectRepo {
  Future<HandcraftProject> createProject({
    required String title,
    required HandcraftItemType itemType,
  });

  Future<HandcraftProject?> getProject(String id);

  Future<List<HandcraftProject>> listProjects({
    HandcraftProjectStatus? status,
  });

  Stream<List<HandcraftProject>> watchProjects({
    HandcraftProjectStatus? status,
  });

  Future<HandcraftProject> saveProject(HandcraftProject project);
}

typedef HandcraftClock = DateTime Function();
typedef HandcraftIdFactory = String Function();

class InMemoryHandcraftProjectRepo implements HandcraftProjectRepo {
  InMemoryHandcraftProjectRepo({
    HandcraftClock? clock,
    HandcraftIdFactory? idFactory,
    List<HandcraftProject> seedProjects = const [],
  }) : _clock = clock ?? DateTime.now,
       _idFactory = idFactory ?? _defaultIdFactory {
    for (final project in seedProjects) {
      _projects[project.id] = project;
    }
    _emit();
  }

  final HandcraftClock _clock;
  final HandcraftIdFactory _idFactory;
  final Map<String, HandcraftProject> _projects = {};
  final _controller = StreamController<List<HandcraftProject>>.broadcast();

  @override
  Future<HandcraftProject> createProject({
    required String title,
    required HandcraftItemType itemType,
  }) async {
    final project = HandcraftProject.create(
      id: _idFactory(),
      title: title,
      itemType: itemType,
      now: _clock(),
    );
    _projects[project.id] = project;
    _emit();
    return project;
  }

  @override
  Future<HandcraftProject?> getProject(String id) async {
    return _projects[id];
  }

  @override
  Future<List<HandcraftProject>> listProjects({
    HandcraftProjectStatus? status,
  }) async {
    return _filtered(status);
  }

  @override
  Stream<List<HandcraftProject>> watchProjects({
    HandcraftProjectStatus? status,
  }) async* {
    yield _filtered(status);
    yield* _controller.stream.map((projects) {
      if (status == null) {
        return projects;
      }
      return projects
          .where((project) => project.status == status)
          .toList(growable: false);
    });
  }

  @override
  Future<HandcraftProject> saveProject(HandcraftProject project) async {
    final updated = project.copyWith(updatedAt: _clock());
    _projects[project.id] = updated;
    _emit();
    return updated;
  }

  void dispose() {
    _controller.close();
  }

  List<HandcraftProject> _filtered(HandcraftProjectStatus? status) {
    final projects = _projects.values.toList(growable: false)
      ..sort((a, b) => b.updatedAt.compareTo(a.updatedAt));
    if (status == null) {
      return projects;
    }
    return projects
        .where((project) => project.status == status)
        .toList(growable: false);
  }

  void _emit() {
    if (!_controller.isClosed) {
      _controller.add(_filtered(null));
    }
  }
}

String _defaultIdFactory() {
  return 'handcraft-${DateTime.now().microsecondsSinceEpoch}';
}
```

- [ ] **Step 3: Run repo tests**

Run:

```bash
cd /Users/j/repos/puupees/puupee-apps/apps/puupee/handcraft
flutter test test/handcraft_project_repo_test.dart --no-pub
```

Expected: PASS.

- [ ] **Step 4: Write the failing provider tests**

Create `apps/puupee/handcraft/test/handcraft_providers_test.dart`:

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:puupee_handcraft/models/handcraft_project.dart';
import 'package:puupee_handcraft/providers/handcraft_providers.dart';
import 'package:puupee_handcraft/repo/handcraft_project_repo.dart';
import 'package:puupee_handcraft/services/handcraft_generation_service.dart';

void main() {
  test('studio controller creates a draft and completes mock generation', () async {
    final repo = InMemoryHandcraftProjectRepo(
      clock: () => DateTime(2026, 5, 13, 9),
      idFactory: () => 'project-1',
    );
    final container = ProviderContainer(
      overrides: [
        handcraftProjectRepoProvider.overrideWithValue(repo),
        handcraftGenerationServiceProvider.overrideWithValue(
          const MockHandcraftGenerationService(delay: Duration.zero),
        ),
      ],
    );
    addTearDown(container.dispose);
    addTearDown(repo.dispose);

    final controller = container.read(handcraftStudioControllerProvider.notifier);
    final draft = await container.read(handcraftStudioControllerProvider.future);

    expect(draft.status, HandcraftProjectStatus.draft);

    await controller.updateInputs(
      styleImageLocalPath: '/tmp/bag.jpg',
      fabricImageLocalPath: '/tmp/canvas.jpg',
    );
    await controller.generate();

    final completed = await container.read(handcraftStudioControllerProvider.future);

    expect(completed.status, HandcraftProjectStatus.completed);
    expect(completed.previewImageLocalPath, isNotNull);
    expect(await repo.getProject('project-1'), completed);
  });
}
```

- [ ] **Step 5: Implement providers**

Create `apps/puupee/handcraft/lib/providers/handcraft_providers.dart`:

```dart
import 'package:puupee_handcraft/models/handcraft_project.dart';
import 'package:puupee_handcraft/repo/handcraft_project_repo.dart';
import 'package:puupee_handcraft/services/handcraft_generation_service.dart';
import 'package:riverpod_annotation/riverpod_annotation.dart';

part 'handcraft_providers.g.dart';

@Riverpod(keepAlive: true)
HandcraftProjectRepo handcraftProjectRepo(Ref ref) {
  final repo = InMemoryHandcraftProjectRepo();
  ref.onDispose(() {
    if (repo is InMemoryHandcraftProjectRepo) {
      repo.dispose();
    }
  });
  return repo;
}

@riverpod
HandcraftGenerationService handcraftGenerationService(Ref ref) {
  return const MockHandcraftGenerationService();
}

@riverpod
Stream<List<HandcraftProject>> handcraftProjects(
  Ref ref, {
  HandcraftProjectStatus? status,
}) {
  return ref.watch(handcraftProjectRepoProvider).watchProjects(status: status);
}

@riverpod
class HandcraftStudioController extends _$HandcraftStudioController {
  @override
  Future<HandcraftProject> build() async {
    final repo = ref.watch(handcraftProjectRepoProvider);
    final existing = await repo.listProjects(status: HandcraftProjectStatus.draft);
    if (existing.isNotEmpty) {
      return existing.first;
    }
    return repo.createProject(title: '新的手工作品', itemType: HandcraftItemType.bag);
  }

  Future<void> updateInputs({
    String? styleImageLocalPath,
    String? fabricImageLocalPath,
  }) async {
    final project = await future;
    final updated = project.copyWith(
      styleImageLocalPath: styleImageLocalPath ?? project.styleImageLocalPath,
      fabricImageLocalPath: fabricImageLocalPath ?? project.fabricImageLocalPath,
      errorMessage: null,
    );
    state = AsyncData(await ref.read(handcraftProjectRepoProvider).saveProject(updated));
  }

  Future<void> updateDetails({
    String? title,
    HandcraftItemType? itemType,
    HandcraftDimensions? dimensions,
  }) async {
    final project = await future;
    final updated = project.copyWith(
      title: title,
      itemType: itemType,
      dimensions: dimensions,
      errorMessage: null,
    );
    state = AsyncData(await ref.read(handcraftProjectRepoProvider).saveProject(updated));
  }

  Future<void> generate() async {
    final project = await future;
    if (project.status == HandcraftProjectStatus.generating) {
      return;
    }

    final repo = ref.read(handcraftProjectRepoProvider);
    final service = ref.read(handcraftGenerationServiceProvider);
    try {
      state = AsyncData(
        await repo.saveProject(
          project.copyWith(
            status: HandcraftProjectStatus.generating,
            currentStage: HandcraftGenerationStage.renderingPreview,
            errorMessage: null,
          ),
        ),
      );
      final result = await service.generate(
        await future,
        onStageChanged: (stage) async {
          final current = await future;
          state = AsyncData(
            await repo.saveProject(current.copyWith(currentStage: stage)),
          );
        },
      );
      state = AsyncData(await repo.saveProject(result));
    } on HandcraftGenerationException catch (error) {
      final failed = project.copyWith(
        status: HandcraftProjectStatus.failed,
        currentStage: error.stage ?? project.currentStage,
        errorMessage: error.message,
      );
      state = AsyncData(await repo.saveProject(failed));
    }
  }
}
```

- [ ] **Step 6: Generate Riverpod code**

Run:

```bash
cd /Users/j/repos/puupees/puupee-apps/apps/puupee/handcraft
dart run build_runner build --delete-conflicting-outputs
```

Expected: creates `lib/providers/handcraft_providers.g.dart`.

- [ ] **Step 7: Run provider tests**

Run:

```bash
cd /Users/j/repos/puupees/puupee-apps/apps/puupee/handcraft
flutter test test/handcraft_providers_test.dart --no-pub
```

Expected: PASS.

- [ ] **Step 8: Commit repo and providers**

Run:

```bash
git add apps/puupee/handcraft/lib/repo/handcraft_project_repo.dart apps/puupee/handcraft/lib/providers/handcraft_providers.dart apps/puupee/handcraft/lib/providers/handcraft_providers.g.dart apps/puupee/handcraft/test/handcraft_project_repo_test.dart apps/puupee/handcraft/test/handcraft_providers_test.dart
git commit -m "feat(handcraft): 添加作品仓库和状态管理"
```

---

### Task 5: Add Routing, Shell, Pages, And Components

**Files:**
- Create: `apps/puupee/handcraft/lib/router.dart`
- Generate: `apps/puupee/handcraft/lib/router.g.dart`
- Create: `apps/puupee/handcraft/lib/components/adaptive_handcraft_shell.dart`
- Create: `apps/puupee/handcraft/lib/components/handcraft_project_card.dart`
- Create: `apps/puupee/handcraft/lib/components/handcraft_media_input_card.dart`
- Create: `apps/puupee/handcraft/lib/components/handcraft_dimension_editor.dart`
- Create: `apps/puupee/handcraft/lib/components/handcraft_generation_panel.dart`
- Create: `apps/puupee/handcraft/lib/pages/handcraft_studio_page.dart`
- Create: `apps/puupee/handcraft/lib/pages/handcraft_projects_page.dart`
- Create: `apps/puupee/handcraft/lib/pages/handcraft_assets_page.dart`
- Test: `apps/puupee/handcraft/test/handcraft_page_smoke_test.dart`

- [ ] **Step 1: Write a failing page smoke test**

Create `apps/puupee/handcraft/test/handcraft_page_smoke_test.dart`:

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:hooks_riverpod/hooks_riverpod.dart';
import 'package:puupee_handcraft/pages/handcraft_studio_page.dart';
import 'package:shadcn_flutter/shadcn_flutter.dart';

void main() {
  testWidgets('studio page shows primary workflow text', (tester) async {
    await tester.pumpWidget(
      const ProviderScope(
        child: ShadcnApp(home: HandcraftStudioPage()),
      ),
    );

    await tester.pumpAndSettle();

    expect(find.text('小汪手工'), findsOneWidget);
    expect(find.text('款式图'), findsWidgets);
    expect(find.text('布料图'), findsWidgets);
    expect(find.text('生成效果'), findsOneWidget);
  });
}
```

- [ ] **Step 2: Run the smoke test and verify it fails**

Run:

```bash
cd /Users/j/repos/puupees/puupee-apps/apps/puupee/handcraft
flutter test test/handcraft_page_smoke_test.dart --no-pub
```

Expected: FAIL because pages and components do not exist.

- [ ] **Step 3: Implement the adaptive shell**

Create `apps/puupee/handcraft/lib/components/adaptive_handcraft_shell.dart` using the same structure as `AdaptiveInventoryShell`: `AdaptiveLayout` for desktop, bottom navigation for mobile, app icon `HomeIcon(appId: 'handcraft')`, title `小汪手工`, and four menu items:

```dart
static const List<AdaptiveMenuItem> _menuItems = [
  AdaptiveMenuItem(label: '创作', icon: Icons.auto_fix_high),
  AdaptiveMenuItem(label: '作品', icon: Icons.collections_bookmark_outlined),
  AdaptiveMenuItem(label: '素材', icon: Icons.perm_media_outlined),
  AdaptiveMenuItem(
    label: '我的',
    icon: Icons.person_outline,
    position: MenuItemPosition.bottom,
  ),
];
```

Map indexes to these routes:

```dart
switch (index) {
  case 0:
    context.go('/handcraft');
    break;
  case 1:
    context.go('/handcraft/projects');
    break;
  case 2:
    context.go('/handcraft/assets');
    break;
  case 3:
    context.go('/handcraft/settings');
    break;
}
```

- [ ] **Step 4: Implement reusable cards and editors**

Create the four component files with focused responsibilities:

```dart
// handcraft_project_card.dart
class HandcraftProjectCard extends StatelessWidget {
  const HandcraftProjectCard({super.key, required this.project, this.onPressed});

  final HandcraftProject project;
  final VoidCallback? onPressed;

  @override
  Widget build(BuildContext context) {
    final theme = Theme.of(context);
    return Card(
      child: GestureDetector(
        onTap: onPressed,
        behavior: HitTestBehavior.opaque,
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text(project.title, style: theme.typography.h4),
            const SizedBox(height: 6),
            Text(
              '${project.itemType.label} · ${project.status.label}',
              style: theme.typography.small.copyWith(
                color: theme.colorScheme.mutedForeground,
              ),
            ),
          ],
        ),
      ),
    );
  }
}
```

```dart
// handcraft_media_input_card.dart
class HandcraftMediaInputCard extends StatelessWidget {
  const HandcraftMediaInputCard({
    super.key,
    required this.title,
    required this.path,
    required this.onChanged,
  });

  final String title;
  final String? path;
  final ValueChanged<String> onChanged;

  @override
  Widget build(BuildContext context) {
    final controller = TextEditingController(text: path ?? '');
    return Card(
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Text(title, style: Theme.of(context).typography.h4),
          const SizedBox(height: 10),
          TextField(
            controller: controller,
            placeholder: '输入本地图片路径',
            onSubmitted: onChanged,
          ),
        ],
      ),
    );
  }
}
```

```dart
// handcraft_dimension_editor.dart
class HandcraftDimensionEditor extends StatelessWidget {
  const HandcraftDimensionEditor({
    super.key,
    required this.project,
    required this.onChanged,
  });

  final HandcraftProject project;
  final ValueChanged<HandcraftDimensions> onChanged;

  @override
  Widget build(BuildContext context) {
    final dimensions = project.effectiveDimensions;
    return Card(
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Text('尺寸', style: Theme.of(context).typography.h4),
          const SizedBox(height: 10),
          for (final entry in dimensions.values.entries)
            Padding(
              padding: const EdgeInsets.only(bottom: 8),
              child: Row(
                children: [
                  Expanded(child: Text(entry.key)),
                  Text('${entry.value} cm'),
                ],
              ),
            ),
        ],
      ),
    );
  }
}
```

```dart
// handcraft_generation_panel.dart
class HandcraftGenerationPanel extends StatelessWidget {
  const HandcraftGenerationPanel({
    super.key,
    required this.project,
    required this.onGenerate,
  });

  final HandcraftProject project;
  final VoidCallback onGenerate;

  @override
  Widget build(BuildContext context) {
    final isGenerating = project.status == HandcraftProjectStatus.generating;
    return Card(
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Text('生成效果', style: Theme.of(context).typography.h4),
          const SizedBox(height: 10),
          Text('当前阶段：${project.currentStage.label}'),
          const SizedBox(height: 12),
          PrimaryButton(
            onPressed: isGenerating || !project.hasRequiredInputs ? null : onGenerate,
            child: Text(isGenerating ? '生成中' : '生成效果'),
          ),
          const SizedBox(height: 12),
          Text('效果图：${project.previewImageLocalPath ?? '尚未生成'}'),
          Text('裁剪图：${project.patternImageLocalPath ?? '尚未生成'}'),
          Text('过程视频：${project.processVideoLocalPath ?? '尚未生成'}'),
          if (project.errorMessage != null) Text(project.errorMessage!),
        ],
      ),
    );
  }
}
```

Also add label extensions to `handcraft_project.dart`:

```dart
extension HandcraftItemTypeLabel on HandcraftItemType {
  String get label => switch (this) {
    HandcraftItemType.bag => '包',
    HandcraftItemType.clothing => '衣服',
    HandcraftItemType.other => '其他',
  };
}

extension HandcraftProjectStatusLabel on HandcraftProjectStatus {
  String get label => switch (this) {
    HandcraftProjectStatus.draft => '草稿',
    HandcraftProjectStatus.generating => '生成中',
    HandcraftProjectStatus.completed => '已完成',
    HandcraftProjectStatus.failed => '失败',
  };
}

extension HandcraftGenerationStageLabel on HandcraftGenerationStage {
  String get label => switch (this) {
    HandcraftGenerationStage.ready => '准备',
    HandcraftGenerationStage.renderingPreview => '生成成品效果图',
    HandcraftGenerationStage.draftingPattern => '生成裁剪图',
    HandcraftGenerationStage.makingVideo => '生成过程视频',
    HandcraftGenerationStage.completed => '完成',
  };
}
```

- [ ] **Step 5: Implement pages**

Create `HandcraftStudioPage` as a `ConsumerWidget`. Use `context.isMobile` to choose a single-column mobile wizard and a two-column desktop workbench. Both layouts must include `HandcraftMediaInputCard` for `款式图` and `布料图`, `HandcraftDimensionEditor`, and `HandcraftGenerationPanel`.

Create `HandcraftProjectsPage` as a `ConsumerWidget` that watches `handcraftProjectsProvider()` and renders `HandcraftProjectCard` for each project, with empty text `还没有手工作品`.

Create `HandcraftAssetsPage` as a `ConsumerWidget` that watches all projects and lists non-empty local and remote media fields under headings `输入素材` and `生成结果`.

- [ ] **Step 6: Implement typed routes and settings routes**

Create `apps/puupee/handcraft/lib/router.dart` with a `TypedStatefulShellRoute<HandcraftMainStatefulShellRoute>` and four branches:

```dart
TypedGoRoute<HandcraftStudioRoute>(path: '/handcraft')
TypedGoRoute<HandcraftProjectsRoute>(path: '/handcraft/projects')
TypedGoRoute<HandcraftAssetsRoute>(path: '/handcraft/assets')
TypedShellRoute<HandcraftSettingsShellRoute>(routes: [
  TypedGoRoute<HandcraftSettingsRoute>(path: '/handcraft/settings'),
  TypedGoRoute<HandcraftSettingsAccountRoute>(path: '/handcraft/settings/account'),
  TypedGoRoute<HandcraftSettingsStorageRoute>(path: '/handcraft/settings/storage'),
  TypedGoRoute<HandcraftSettingsSyncRoute>(path: '/handcraft/settings/sync'),
  TypedGoRoute<HandcraftSettingsDataRoute>(path: '/handcraft/settings/data'),
  TypedGoRoute<HandcraftSettingsDevicesRoute>(path: '/handcraft/settings/devices'),
  TypedGoRoute<HandcraftSettingsDeveloperRoute>(path: '/handcraft/settings/developer'),
  TypedGoRoute<HandcraftSettingsAboutRoute>(path: '/handcraft/settings/about'),
  TypedGoRoute<HandcraftSettingsFeedbackRoute>(path: '/handcraft/settings/feedback'),
  TypedGoRoute<HandcraftSettingsFeedbackSuccessRoute>(path: '/handcraft/settings/feedback/success'),
  TypedGoRoute<HandcraftSettingsFeedbackListRoute>(path: '/handcraft/settings/feedback/list'),
  TypedGoRoute<HandcraftSettingsFeedbackDetailRoute>(path: '/handcraft/settings/feedback/detail/:id'),
  TypedGoRoute<HandcraftSettingsRuntimeInfoRoute>(path: '/handcraft/settings/runtime-info'),
  TypedGoRoute<HandcraftSettingsRecycleBinRoute>(path: '/handcraft/settings/recycle-bin'),
  TypedGoRoute<HandcraftSettingsSubscriptionRoute>(path: '/handcraft/settings/subscription'),
])
```

For settings page classes, reuse the same shared settings page imports and mobile-empty behavior used by `apps/puupee/inventory/lib/router.dart`, replacing route class names and paths with `Handcraft`.

- [ ] **Step 7: Generate route and provider code**

Run:

```bash
cd /Users/j/repos/puupees/puupee-apps/apps/puupee/handcraft
dart run build_runner build --delete-conflicting-outputs
```

Expected: creates or updates `lib/router.g.dart` and `lib/providers/handcraft_providers.g.dart`.

- [ ] **Step 8: Run the page smoke test**

Run:

```bash
cd /Users/j/repos/puupees/puupee-apps/apps/puupee/handcraft
flutter test test/handcraft_page_smoke_test.dart --no-pub
```

Expected: PASS.

- [ ] **Step 9: Commit UI and routing**

Run:

```bash
git add apps/puupee/handcraft/lib/router.dart apps/puupee/handcraft/lib/router.g.dart apps/puupee/handcraft/lib/components apps/puupee/handcraft/lib/pages apps/puupee/handcraft/lib/models/handcraft_project.dart apps/puupee/handcraft/test/handcraft_page_smoke_test.dart
git commit -m "feat(handcraft): 添加创作工作台和导航页面"
```

---

### Task 6: Add Android Project

**Files:**
- Create: `apps/puupee/handcraft/android/.gitignore`
- Create: `apps/puupee/handcraft/android/app/build.gradle.kts`
- Create: `apps/puupee/handcraft/android/app/proguard-rules.pro`
- Create: `apps/puupee/handcraft/android/app/src/debug/AndroidManifest.xml`
- Create: `apps/puupee/handcraft/android/app/src/main/AndroidManifest.xml`
- Create: `apps/puupee/handcraft/android/app/src/main/kotlin/com/puupee/handcraft/MainActivity.kt`
- Create: `apps/puupee/handcraft/android/app/src/main/res/values/strings.xml`
- Create: `apps/puupee/handcraft/android/app/src/main/res/values-zh/strings.xml`
- Create: `apps/puupee/handcraft/android/app/src/main/res/values/styles.xml`
- Create: `apps/puupee/handcraft/android/app/src/main/res/values-night/styles.xml`
- Create: `apps/puupee/handcraft/android/app/src/main/res/drawable/launch_background.xml`
- Create: `apps/puupee/handcraft/android/app/src/main/res/drawable-v21/launch_background.xml`
- Create: `apps/puupee/handcraft/android/app/src/main/res/xml/file_paths.xml`
- Create launcher icon assets copied from `apps/puupee/inventory/android/app/src/main/res/mipmap-*`
- Create: `apps/puupee/handcraft/android/app/src/profile/AndroidManifest.xml`
- Create: `apps/puupee/handcraft/android/build.gradle.kts`
- Create: `apps/puupee/handcraft/android/gradle.properties`
- Create: `apps/puupee/handcraft/android/settings.gradle.kts`
- Create: `apps/puupee/handcraft/android/gradle/wrapper/gradle-wrapper.properties`

- [ ] **Step 1: Copy Android files from inventory**

Run:

```bash
mkdir -p apps/puupee/handcraft
rsync -a \
  --exclude='.gradle' \
  --exclude='local.properties' \
  --exclude='gradlew' \
  --exclude='gradlew.bat' \
  --exclude='gradle/wrapper/gradle-wrapper.jar' \
  --exclude='app/src/main/java' \
  apps/puupee/inventory/android/ \
  apps/puupee/handcraft/android/
```

Expected: `apps/puupee/handcraft/android/app/src/main/AndroidManifest.xml` exists.

- [ ] **Step 2: Rename Android package paths**

Run:

```bash
mkdir -p apps/puupee/handcraft/android/app/src/main/kotlin/com/puupee/handcraft
mv apps/puupee/handcraft/android/app/src/main/kotlin/com/puupee/inventory/MainActivity.kt \
  apps/puupee/handcraft/android/app/src/main/kotlin/com/puupee/handcraft/MainActivity.kt
find apps/puupee/handcraft/android -type d -empty -delete
```

Replace `apps/puupee/handcraft/android/app/src/main/kotlin/com/puupee/handcraft/MainActivity.kt` with:

```kotlin
package com.puupee.handcraft

import io.flutter.embedding.android.FlutterActivity

class MainActivity : FlutterActivity()
```

- [ ] **Step 3: Update Android Gradle config**

In `apps/puupee/handcraft/android/app/build.gradle.kts`, replace:

```kotlin
namespace = "com.puupee.inventory"
applicationId = "com.puupee.inventory"
```

with:

```kotlin
namespace = "com.puupee.handcraft"
applicationId = "com.puupee.handcraft"
```

Verify the file still contains:

```kotlin
kotlin {
    jvmToolchain(17)
}

compileOptions {
    sourceCompatibility = JavaVersion.VERSION_17
    targetCompatibility = JavaVersion.VERSION_17
    isCoreLibraryDesugaringEnabled = true
}

dependencies {
    coreLibraryDesugaring("com.android.tools:desugar_jdk_libs:2.1.5")
}
```

- [ ] **Step 4: Update localized app names**

Replace `apps/puupee/handcraft/android/app/src/main/res/values/strings.xml` with:

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <string name="app_name">Puupee Handcraft</string>
</resources>
```

Replace `apps/puupee/handcraft/android/app/src/main/res/values-zh/strings.xml` with:

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <string name="app_name">小汪手工</string>
</resources>
```

- [ ] **Step 5: Confirm manifest keeps Puupee requirements**

Check `apps/puupee/handcraft/android/app/src/main/AndroidManifest.xml` contains:

```xml
<application
    android:label="@string/app_name"
    android:name="${applicationName}"
    android:icon="@mipmap/ic_launcher">
```

and FileProvider:

```xml
<provider
    android:name="androidx.core.content.FileProvider"
    android:authorities="${applicationId}.fileprovider"
    android:exported="false"
    android:grantUriPermissions="true">
    <meta-data
        android:name="android.support.FILE_PROVIDER_PATHS"
        android:resource="@xml/file_paths" />
</provider>
```

- [ ] **Step 6: Remove local platform junk**

Run:

```bash
rm -rf apps/puupee/handcraft/android/.gradle \
  apps/puupee/handcraft/android/local.properties \
  apps/puupee/handcraft/android/app/src/main/java \
  apps/puupee/handcraft/android/gradle/wrapper/gradle-wrapper.jar \
  apps/puupee/handcraft/android/gradlew \
  apps/puupee/handcraft/android/gradlew.bat
```

Expected: none of those paths remain.

- [ ] **Step 7: Run Android file existence checks**

Run:

```bash
test -f apps/puupee/handcraft/android/app/src/main/AndroidManifest.xml
test -f apps/puupee/handcraft/android/app/src/main/kotlin/com/puupee/handcraft/MainActivity.kt
test -f apps/puupee/handcraft/android/app/src/main/res/values/strings.xml
test -f apps/puupee/handcraft/android/app/src/main/res/values-zh/strings.xml
```

Expected: all commands exit 0.

- [ ] **Step 8: Commit Android platform files**

Run:

```bash
git add apps/puupee/handcraft/android
git commit -m "build(handcraft): 添加 Android 平台配置"
```

---

### Task 7: Add README And Run Final Verification

**Files:**
- Create: `apps/puupee/handcraft/README.md`
- Verify generated files: `apps/puupee/handcraft/lib/router.g.dart`, `apps/puupee/handcraft/lib/providers/handcraft_providers.g.dart`

- [ ] **Step 1: Create the app README**

Create `apps/puupee/handcraft/README.md`:

```markdown
# 小汪手工

小汪手工是布料成品效果、尺寸裁剪图和制作过程视频的模拟生成工具。

## 第一版范围

- 上传款式图和布料图的本地路径。
- 选择包、衣服或其他物品类型。
- 填写目标尺寸；未填写时使用模拟估算尺寸。
- 使用本地模拟服务生成成品效果图、裁剪图和过程视频引用。
- 保存同步型作品历史的数据形状，预留远程媒体 URL 和上传状态字段。

## 开发

```bash
cd apps/puupee/handcraft
dart run build_runner build --delete-conflicting-outputs
dart analyze
flutter test
flutter build apk --debug --target-platform android-arm64 --no-pub
```
```

- [ ] **Step 2: Run code generation**

Run:

```bash
cd /Users/j/repos/puupees/puupee-apps/apps/puupee/handcraft
dart run build_runner build --delete-conflicting-outputs
```

Expected: finishes successfully and generated files are current.

- [ ] **Step 3: Run focused analyze**

Run:

```bash
cd /Users/j/repos/puupees/puupee-apps
dart analyze apps/puupee/handcraft/lib/env.dart apps/puupee/handcraft/lib/main.dart apps/puupee/handcraft/test/env_test.dart
```

Expected: `No issues found!`

- [ ] **Step 4: Run all handcraft tests**

Run:

```bash
cd /Users/j/repos/puupees/puupee-apps/apps/puupee/handcraft
flutter test --no-pub
```

Expected: all tests pass.

- [ ] **Step 5: Build debug Android APK**

Run:

```bash
cd /Users/j/repos/puupees/puupee-apps/apps/puupee/handcraft
flutter build apk --debug --target-platform android-arm64 --no-pub
```

Expected: debug APK build succeeds.

- [ ] **Step 6: Confirm no local platform junk is staged**

Run:

```bash
git status --short
```

Expected: no entries for:

```text
apps/puupee/handcraft/android/.gradle
apps/puupee/handcraft/android/local.properties
apps/puupee/handcraft/android/gradle/wrapper/gradle-wrapper.jar
apps/puupee/handcraft/android/gradlew
apps/puupee/handcraft/android/gradlew.bat
```

- [ ] **Step 7: Commit README and verification updates**

Run:

```bash
git add apps/puupee/handcraft/README.md apps/puupee/handcraft/lib/router.g.dart apps/puupee/handcraft/lib/providers/handcraft_providers.g.dart
git commit -m "docs(handcraft): 添加应用说明和生成文件"
```

If generated files were already committed in earlier tasks and only README changed, run:

```bash
git add apps/puupee/handcraft/README.md
git commit -m "docs(handcraft): 添加应用说明"
```

---

## Self-Review Notes

- Spec coverage: this plan covers app scaffold, workspace registration, EnvConfig, Android setup, model fields, local media paths, remote URL fields, upload statuses, manual or estimated dimensions, mock preview/pattern/video generation, project history repo, Riverpod state, studio/projects/assets/settings navigation, tests, analyze, and APK build.
- Scope: real AI generation, object storage upload, precise pattern-making algorithms, and complete cross-device media sync remain outside first-version implementation.
- Type consistency: model enums and class names use the `Handcraft` prefix across tests, service, repo, providers, routes, and pages.
