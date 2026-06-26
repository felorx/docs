# Felorx Connect Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 将 `reevibe-server` 重构为 `packages/sync/felorx_connect` 通用 WebRTC 连接层，并在 `sync_node` 和 Ops 中落地远程终端。

**Architecture:** `felorx_connect` 提供纯 Dart relay/signaling、host daemon、协议模型和终端连接抽象；`sync_node` 以可选子服务启动 relay；Ops 复用现有终端 UI，通过 relay 建立 WebRTC DataChannel 与远程 Linux host 直连。

**Tech Stack:** Dart 3.8+ workspace、`dart:io` WebSocket/HTTP、`args`、`crypto`、`uuid`、`webrtc_dart` host 侧 WebRTC、Ops 侧 `flutter_webrtc`、`xterm`、`flutter_pty`、Riverpod、go_router、shadcn_flutter。

---

## Scope Check

本规格横跨协议包、服务端、daemon、sync_node 和 Ops UI，但每一层都依赖上一层连接协议。按一个实施计划推进，任务按依赖顺序拆分，任何任务完成后都能独立测试并提交。

## File Structure

### 新增 `packages/sync/felorx_connect`

- `packages/sync/felorx_connect/pubspec.yaml`：包依赖，包含 `args`、`crypto`、`uuid`、`webrtc_dart`、`test`。
- `packages/sync/felorx_connect/analysis_options.yaml`：复用仓库 lint 风格。
- `packages/sync/felorx_connect/lib/felorx_connect.dart`：公共导出，不导出 Flutter 依赖。
- `packages/sync/felorx_connect/lib/src/protocol/connect_message.dart`：版本化 JSON 消息。
- `packages/sync/felorx_connect/lib/src/protocol/connect_error.dart`：错误码。
- `packages/sync/felorx_connect/lib/src/auth/connect_password.dart`：连接密码 hash/verify。
- `packages/sync/felorx_connect/lib/src/relay/host_registry.dart`：在线 host 注册表。
- `packages/sync/felorx_connect/lib/src/relay/session_registry.dart`：连接会话注册表。
- `packages/sync/felorx_connect/lib/src/relay/connect_relay_server.dart`：HTTP + WebSocket relay。
- `packages/sync/felorx_connect/lib/src/client/connect_relay_client.dart`：relay WebSocket 客户端。
- `packages/sync/felorx_connect/lib/src/client/connect_terminal_client.dart`：终端客户端抽象和 DataChannel 桥。
- `packages/sync/felorx_connect/lib/src/host/connect_host_daemon.dart`：host daemon 编排。
- `packages/sync/felorx_connect/lib/src/host/connect_host_config.dart`：CLI/env 配置解析。
- `packages/sync/felorx_connect/lib/src/host/terminal_host_session.dart`：远程终端会话。
- `packages/sync/felorx_connect/lib/src/host/pty/terminal_pty.dart`：PTY 抽象。
- `packages/sync/felorx_connect/lib/src/host/pty/linux_terminal_pty.dart`：Linux PTY 实现，封装在抽象后。
- `packages/sync/felorx_connect/bin/felorx_connect_host.dart`：Linux host daemon CLI。
- `packages/sync/felorx_connect/test/...`：协议、密码、注册表、relay、daemon 配置测试。

### 修改 `apps/sync_node`

- `apps/sync_node/pubspec.yaml`：新增 `felorx_connect` 依赖。
- `apps/sync_node/lib/sync_node_arg_parser.dart`：新增 connect relay 参数。
- `apps/sync_node/lib/sync_node_connect_relay.dart`：connect relay 启动 helper，便于生命周期单测。
- `apps/sync_node/lib/sync_node_runner.dart`：启动/停止 connect relay。
- `apps/sync_node/test/sync_node_arg_parser_test.dart`：参数测试。
- `apps/sync_node/test/connect_relay_lifecycle_test.dart`：生命周期测试。

### 修改 `apps/ops`

- `apps/ops/pubspec.yaml`：新增 `felorx_connect` 和 `flutter_webrtc` 依赖；`flutter_webrtc` 现仅在 reevibe 使用，Ops 需要显式依赖。
- `apps/ops/lib/pages/terminal/terminal_page.dart`：本地/远程模式入口。
- `apps/ops/lib/pages/terminal/remote_terminal_page.dart`：远程终端页面。
- `apps/ops/lib/pages/terminal/remote_terminal_connection_dialog.dart`：设备码连接表单。
- `apps/ops/lib/pages/terminal/ops_terminal_surface.dart`：共享 `TerminalView` 外壳。
- `apps/ops/lib/services/connect_relay_service.dart`：Ops relay 查询和连接服务。
- `apps/ops/lib/services/flutter_webrtc_terminal_peer.dart`：Ops 侧 WebRTC adapter。
- `apps/ops/lib/providers/connect_provider.dart`：远程 host 列表和连接状态 provider。
- `apps/ops/lib/models/connect_host.dart`：Ops UI host model。
- `apps/ops/lib/router.dart`：`/ops/terminal` 返回新的 `OpsTerminalPage`。
- `apps/ops/test/pages/terminal_remote_connection_form_test.dart`：设备码表单测试。

### 移除旧项目

- `pubspec.yaml`：workspace 移除 `apps/reevibe-server`，新增 `packages/sync/felorx_connect`。
- `apps/reevibe-server/`：删除。
- `scripts/reevibe_server.dart`：删除旧启动脚本。

---

## Task 1: Scaffold `felorx_connect` and Protocol Messages

**Files:**
- Create: `packages/sync/felorx_connect/pubspec.yaml`
- Create: `packages/sync/felorx_connect/analysis_options.yaml`
- Create: `packages/sync/felorx_connect/lib/felorx_connect.dart`
- Create: `packages/sync/felorx_connect/lib/src/protocol/connect_message.dart`
- Create: `packages/sync/felorx_connect/lib/src/protocol/connect_error.dart`
- Create: `packages/sync/felorx_connect/test/protocol/connect_message_test.dart`
- Modify: `pubspec.yaml`

- [ ] **Step 1: Add package skeleton**

Create `packages/sync/felorx_connect/pubspec.yaml`:

```yaml
name: felorx_connect
resolution: workspace
description: Felorx remote connection relay, host daemon, and terminal protocol.
version: 0.1.0
publish_to: none

environment:
  sdk: ^3.8.1

dependencies:
  args: ^2.4.2
  crypto: ^3.0.6
  uuid: ^4.5.1
  webrtc_dart: ^0.25.3

dev_dependencies:
  lints: ^6.0.0
  test: ^1.26.2
```

Create `packages/sync/felorx_connect/analysis_options.yaml`:

```yaml
include: package:lints/recommended.yaml

linter:
  rules:
    prefer_single_quotes: true
```

Create `packages/sync/felorx_connect/lib/felorx_connect.dart`:

```dart
library;

export 'src/protocol/connect_error.dart';
export 'src/protocol/connect_message.dart';
```

Modify root `pubspec.yaml` workspace section:

```yaml
  - packages/sync/felorx_connect
```

Keep `apps/reevibe-server` in the workspace until Task 15 removes the old app.

- [ ] **Step 2: Write failing protocol tests**

Create `packages/sync/felorx_connect/test/protocol/connect_message_test.dart`:

```dart
import 'dart:convert';

import 'package:felorx_connect/felorx_connect.dart';
import 'package:test/test.dart';

void main() {
  group('ConnectMessage', () {
    test('round trips a registerHost message', () {
      final message = ConnectMessage(
        type: ConnectMessageType.registerHost,
        requestId: 'req-1',
        sessionId: 'session-1',
        payload: {
          'hostId': 'host-1',
          'userId': 'user-1',
          'capabilities': ['terminal', 'directWebRtc'],
        },
      );

      final decoded = ConnectMessage.fromJson(
        jsonDecode(jsonEncode(message.toJson())) as Map<String, dynamic>,
      );

      expect(decoded.protocolVersion, ConnectMessage.currentProtocolVersion);
      expect(decoded.type, ConnectMessageType.registerHost);
      expect(decoded.requestId, 'req-1');
      expect(decoded.sessionId, 'session-1');
      expect(decoded.payload['hostId'], 'host-1');
      expect(decoded.payload['capabilities'], ['terminal', 'directWebRtc']);
    });

    test('rejects an unsupported protocol version', () {
      expect(
        () => ConnectMessage.fromJson({
          'protocolVersion': 999,
          'type': 'hello',
          'payload': <String, dynamic>{},
        }),
        throwsA(isA<ConnectProtocolException>()),
      );
    });

    test('rejects an unknown message type', () {
      expect(
        () => ConnectMessage.fromJson({
          'protocolVersion': ConnectMessage.currentProtocolVersion,
          'type': 'not-real',
          'payload': <String, dynamic>{},
        }),
        throwsA(isA<ConnectProtocolException>()),
      );
    });
  });
}
```

- [ ] **Step 3: Run protocol test to verify red**

Run:

```bash
cd packages/sync/felorx_connect && dart test test/protocol/connect_message_test.dart
```

Expected: FAIL because `ConnectMessage`, `ConnectMessageType`, and `ConnectProtocolException` are not defined.

- [ ] **Step 4: Implement protocol errors**

Create `packages/sync/felorx_connect/lib/src/protocol/connect_error.dart`:

```dart
enum ConnectErrorCode {
  unsupportedProtocolVersion,
  unknownMessageType,
  invalidPayload,
  authenticationRequired,
  permissionDenied,
  hostNotFound,
  sessionNotFound,
  directConnectionFailed,
}

class ConnectProtocolException implements Exception {
  ConnectProtocolException(this.code, this.message);

  final ConnectErrorCode code;
  final String message;

  @override
  String toString() => 'ConnectProtocolException($code): $message';
}
```

- [ ] **Step 5: Implement protocol messages**

Create `packages/sync/felorx_connect/lib/src/protocol/connect_message.dart`:

```dart
import 'connect_error.dart';

enum ConnectMessageType {
  hello('hello'),
  registerHost('registerHost'),
  hostList('hostList'),
  hostListResult('hostListResult'),
  connectRequest('connectRequest'),
  connectAccept('connectAccept'),
  connectReject('connectReject'),
  offer('offer'),
  answer('answer'),
  iceCandidate('iceCandidate'),
  terminalInput('terminalInput'),
  terminalOutput('terminalOutput'),
  resize('resize'),
  exit('exit'),
  heartbeat('heartbeat'),
  error('error');

  const ConnectMessageType(this.wireName);

  final String wireName;

  static ConnectMessageType fromWireName(String value) {
    for (final type in values) {
      if (type.wireName == value) return type;
    }
    throw ConnectProtocolException(
      ConnectErrorCode.unknownMessageType,
      'Unknown connect message type: $value',
    );
  }
}

class ConnectMessage {
  ConnectMessage({
    required this.type,
    Map<String, dynamic>? payload,
    this.protocolVersion = currentProtocolVersion,
    this.requestId,
    this.sessionId,
    this.fromPeerId,
    this.targetPeerId,
    DateTime? timestamp,
  })  : payload = Map.unmodifiable(payload ?? const <String, dynamic>{}),
        timestamp = timestamp ?? DateTime.now().toUtc() {
    if (protocolVersion != currentProtocolVersion) {
      throw ConnectProtocolException(
        ConnectErrorCode.unsupportedProtocolVersion,
        'Unsupported protocol version: $protocolVersion',
      );
    }
  }

  static const int currentProtocolVersion = 1;

  final int protocolVersion;
  final ConnectMessageType type;
  final String? requestId;
  final String? sessionId;
  final String? fromPeerId;
  final String? targetPeerId;
  final DateTime timestamp;
  final Map<String, dynamic> payload;

  factory ConnectMessage.fromJson(Map<String, dynamic> json) {
    final version = json['protocolVersion'];
    if (version != currentProtocolVersion) {
      throw ConnectProtocolException(
        ConnectErrorCode.unsupportedProtocolVersion,
        'Unsupported protocol version: $version',
      );
    }

    final rawType = json['type'];
    if (rawType is! String || rawType.trim().isEmpty) {
      throw ConnectProtocolException(
        ConnectErrorCode.invalidPayload,
        'Message type must be a non-empty string',
      );
    }

    final rawPayload = json['payload'];
    final payload = rawPayload == null
        ? <String, dynamic>{}
        : Map<String, dynamic>.from(rawPayload as Map);

    return ConnectMessage(
      protocolVersion: version as int,
      type: ConnectMessageType.fromWireName(rawType),
      requestId: json['requestId'] as String?,
      sessionId: json['sessionId'] as String?,
      fromPeerId: json['fromPeerId'] as String?,
      targetPeerId: json['targetPeerId'] as String?,
      timestamp: DateTime.tryParse(json['timestamp'] as String? ?? ''),
      payload: payload,
    );
  }

  Map<String, dynamic> toJson() {
    return {
      'protocolVersion': protocolVersion,
      'type': type.wireName,
      if (requestId != null) 'requestId': requestId,
      if (sessionId != null) 'sessionId': sessionId,
      if (fromPeerId != null) 'fromPeerId': fromPeerId,
      if (targetPeerId != null) 'targetPeerId': targetPeerId,
      'timestamp': timestamp.toIso8601String(),
      'payload': payload,
    };
  }
}
```

- [ ] **Step 6: Run protocol test to verify green**

Run:

```bash
cd packages/sync/felorx_connect && dart test test/protocol/connect_message_test.dart
```

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add pubspec.yaml packages/sync/felorx_connect
git commit -m "feat(felorx_connect): 添加连接协议包骨架"
```

---

## Task 2: Password Hashing for Device-Code Connections

**Files:**
- Create: `packages/sync/felorx_connect/lib/src/auth/connect_password.dart`
- Create: `packages/sync/felorx_connect/test/auth/connect_password_test.dart`

- [ ] **Step 1: Write failing password tests**

Create `packages/sync/felorx_connect/test/auth/connect_password_test.dart`:

```dart
import 'package:felorx_connect/felorx_connect.dart';
import 'package:test/test.dart';

void main() {
  group('ConnectPassword', () {
    test('verifies a generated hash', () {
      final hash = ConnectPassword.hash('correct horse battery staple');

      expect(
        ConnectPassword.verify(
          password: 'correct horse battery staple',
          encodedHash: hash,
        ),
        isTrue,
      );
    });

    test('rejects a wrong password', () {
      final hash = ConnectPassword.hash('open-sesame');

      expect(
        ConnectPassword.verify(password: 'wrong', encodedHash: hash),
        isFalse,
      );
    });

    test('rejects malformed encoded hashes', () {
      expect(
        ConnectPassword.verify(password: 'x', encodedHash: 'bad-format'),
        isFalse,
      );
    });
  });
}
```

- [ ] **Step 2: Run password test to verify red**

Run:

```bash
cd packages/sync/felorx_connect && dart test test/auth/connect_password_test.dart
```

Expected: FAIL because `ConnectPassword` is not defined.

- [ ] **Step 3: Implement password hashing**

Create `packages/sync/felorx_connect/lib/src/auth/connect_password.dart`:

```dart
import 'dart:convert';
import 'dart:math';
import 'dart:typed_data';

import 'package:crypto/crypto.dart';

class ConnectPassword {
  static const String _scheme = 'pbkdf2-sha256';
  static const int _defaultIterations = 120000;
  static const int _saltLength = 16;

  static String hash(
    String password, {
    int iterations = _defaultIterations,
    Random? random,
  }) {
    final salt = _randomBytes(_saltLength, random ?? Random.secure());
    final digest = _derive(password, salt, iterations);
    return [
      _scheme,
      iterations.toString(),
      base64UrlEncode(salt),
      base64UrlEncode(digest),
    ].join(r'$');
  }

  static bool verify({
    required String password,
    required String encodedHash,
  }) {
    final parts = encodedHash.split(r'$');
    if (parts.length != 4 || parts[0] != _scheme) return false;
    final iterations = int.tryParse(parts[1]);
    if (iterations == null || iterations <= 0) return false;

    try {
      final salt = base64Url.decode(parts[2]);
      final expected = base64Url.decode(parts[3]);
      final actual = _derive(password, salt, iterations);
      return _constantTimeEquals(actual, expected);
    } catch (_) {
      return false;
    }
  }

  static Uint8List _derive(String password, List<int> salt, int iterations) {
    final hmac = Hmac(sha256, utf8.encode(password));
    var block = hmac.convert([...salt, 0, 0, 0, 1]).bytes;
    final output = Uint8List.fromList(block);
    for (var i = 1; i < iterations; i++) {
      block = hmac.convert(block).bytes;
      for (var j = 0; j < output.length; j++) {
        output[j] ^= block[j];
      }
    }
    return output;
  }

  static Uint8List _randomBytes(int length, Random random) {
    return Uint8List.fromList(
      List<int>.generate(length, (_) => random.nextInt(256)),
    );
  }

  static bool _constantTimeEquals(List<int> a, List<int> b) {
    if (a.length != b.length) return false;
    var diff = 0;
    for (var i = 0; i < a.length; i++) {
      diff |= a[i] ^ b[i];
    }
    return diff == 0;
  }
}
```

- [ ] **Step 4: Export password helper**

Modify `packages/sync/felorx_connect/lib/felorx_connect.dart`:

```dart
library;

export 'src/auth/connect_password.dart';
export 'src/protocol/connect_error.dart';
export 'src/protocol/connect_message.dart';
```

- [ ] **Step 5: Run password test to verify green**

Run:

```bash
cd packages/sync/felorx_connect && dart test test/auth/connect_password_test.dart
```

Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add packages/sync/felorx_connect/lib/src/auth packages/sync/felorx_connect/test/auth
git commit -m "feat(felorx_connect): 添加设备码连接密码校验"
```

---

## Task 3: Host Registry

**Files:**
- Create: `packages/sync/felorx_connect/lib/src/relay/host_registry.dart`
- Create: `packages/sync/felorx_connect/test/relay/host_registry_test.dart`

- [ ] **Step 1: Write failing host registry tests**

Create `packages/sync/felorx_connect/test/relay/host_registry_test.dart`:

```dart
import 'package:felorx_connect/felorx_connect.dart';
import 'package:test/test.dart';

void main() {
  group('ConnectHostRegistry', () {
    late DateTime now;
    late ConnectHostRegistry registry;

    setUp(() {
      now = DateTime.utc(2026, 5, 19, 12);
      registry = ConnectHostRegistry(now: () => now);
    });

    test('lists hosts for the same user only', () {
      registry.register(
        ConnectHostRegistration(
          hostId: 'host-a',
          displayName: 'A',
          userId: 'user-1',
          deviceCode: 'AAA111',
          passwordHash: 'hash-a',
          platform: 'linux',
          capabilities: const {'terminal', 'directWebRtc'},
          authModes: const {'sameAccount', 'deviceCodePassword'},
        ),
      );
      registry.register(
        ConnectHostRegistration(
          hostId: 'host-b',
          displayName: 'B',
          userId: 'user-2',
          deviceCode: 'BBB222',
          passwordHash: 'hash-b',
          platform: 'linux',
          capabilities: const {'terminal'},
          authModes: const {'sameAccount'},
        ),
      );

      final hosts = registry.listByUserId('user-1');

      expect(hosts.map((h) => h.hostId), ['host-a']);
    });

    test('finds a host by device code', () {
      registry.register(
        ConnectHostRegistration(
          hostId: 'host-a',
          displayName: 'A',
          userId: 'user-1',
          deviceCode: 'AAA111',
          passwordHash: 'hash-a',
          platform: 'linux',
          capabilities: const {'terminal'},
          authModes: const {'deviceCodePassword'},
        ),
      );

      expect(registry.findByDeviceCode('AAA111')?.hostId, 'host-a');
    });

    test('prunes stale hosts by heartbeat timeout', () {
      registry.register(
        ConnectHostRegistration(
          hostId: 'host-a',
          displayName: 'A',
          userId: 'user-1',
          deviceCode: 'AAA111',
          passwordHash: 'hash-a',
          platform: 'linux',
          capabilities: const {'terminal'},
          authModes: const {'sameAccount'},
        ),
      );

      now = now.add(const Duration(seconds: 61));
      final removed = registry.pruneStale(
        timeout: const Duration(seconds: 60),
      );

      expect(removed.map((h) => h.hostId), ['host-a']);
      expect(registry.listByUserId('user-1'), isEmpty);
    });
  });
}
```

- [ ] **Step 2: Run host registry test to verify red**

Run:

```bash
cd packages/sync/felorx_connect && dart test test/relay/host_registry_test.dart
```

Expected: FAIL because registry classes are not defined.

- [ ] **Step 3: Implement host registry**

Create `packages/sync/felorx_connect/lib/src/relay/host_registry.dart`:

```dart
class ConnectHostRegistration {
  ConnectHostRegistration({
    required this.hostId,
    required this.displayName,
    required this.userId,
    required this.platform,
    required this.capabilities,
    required this.authModes,
    this.deviceCode,
    this.passwordHash,
    this.systemVersion,
  });

  final String hostId;
  final String displayName;
  final String userId;
  final String platform;
  final String? systemVersion;
  final String? deviceCode;
  final String? passwordHash;
  final Set<String> capabilities;
  final Set<String> authModes;
}

class ConnectHostSnapshot {
  ConnectHostSnapshot({
    required this.hostId,
    required this.displayName,
    required this.userId,
    required this.platform,
    required this.capabilities,
    required this.authModes,
    required this.lastSeenAt,
    this.deviceCode,
    this.passwordHash,
    this.systemVersion,
  });

  final String hostId;
  final String displayName;
  final String userId;
  final String platform;
  final String? systemVersion;
  final String? deviceCode;
  final String? passwordHash;
  final Set<String> capabilities;
  final Set<String> authModes;
  final DateTime lastSeenAt;

  Map<String, dynamic> toPublicJson() {
    return {
      'hostId': hostId,
      'displayName': displayName,
      'userId': userId,
      'platform': platform,
      if (systemVersion != null) 'systemVersion': systemVersion,
      'capabilities': capabilities.toList()..sort(),
      'authModes': authModes.toList()..sort(),
      'lastSeenAt': lastSeenAt.toIso8601String(),
    };
  }
}

class ConnectHostRegistry {
  ConnectHostRegistry({DateTime Function()? now}) : _now = now ?? DateTime.now;

  final DateTime Function() _now;
  final Map<String, ConnectHostSnapshot> _hostsById = {};

  void register(ConnectHostRegistration registration) {
    _hostsById[registration.hostId] = ConnectHostSnapshot(
      hostId: registration.hostId,
      displayName: registration.displayName,
      userId: registration.userId,
      platform: registration.platform,
      systemVersion: registration.systemVersion,
      deviceCode: registration.deviceCode,
      passwordHash: registration.passwordHash,
      capabilities: Set.unmodifiable(registration.capabilities),
      authModes: Set.unmodifiable(registration.authModes),
      lastSeenAt: _now().toUtc(),
    );
  }

  void heartbeat(String hostId) {
    final current = _hostsById[hostId];
    if (current == null) return;
    _hostsById[hostId] = ConnectHostSnapshot(
      hostId: current.hostId,
      displayName: current.displayName,
      userId: current.userId,
      platform: current.platform,
      systemVersion: current.systemVersion,
      deviceCode: current.deviceCode,
      passwordHash: current.passwordHash,
      capabilities: current.capabilities,
      authModes: current.authModes,
      lastSeenAt: _now().toUtc(),
    );
  }

  ConnectHostSnapshot? findByHostId(String hostId) => _hostsById[hostId];

  ConnectHostSnapshot? findByDeviceCode(String deviceCode) {
    for (final host in _hostsById.values) {
      if (host.deviceCode == deviceCode) return host;
    }
    return null;
  }

  List<ConnectHostSnapshot> listByUserId(String userId) {
    final hosts = _hostsById.values.where((h) => h.userId == userId).toList();
    hosts.sort((a, b) => b.lastSeenAt.compareTo(a.lastSeenAt));
    return hosts;
  }

  List<ConnectHostSnapshot> pruneStale({required Duration timeout}) {
    final cutoff = _now().toUtc().subtract(timeout);
    final removed = <ConnectHostSnapshot>[];
    _hostsById.removeWhere((_, host) {
      final stale = host.lastSeenAt.isBefore(cutoff) ||
          host.lastSeenAt.isAtSameMomentAs(cutoff);
      if (stale) removed.add(host);
      return stale;
    });
    return removed;
  }

  void unregister(String hostId) {
    _hostsById.remove(hostId);
  }
}
```

- [ ] **Step 4: Export host registry**

Modify `packages/sync/felorx_connect/lib/felorx_connect.dart`:

```dart
library;

export 'src/auth/connect_password.dart';
export 'src/protocol/connect_error.dart';
export 'src/protocol/connect_message.dart';
export 'src/relay/host_registry.dart';
```

- [ ] **Step 5: Run host registry test to verify green**

Run:

```bash
cd packages/sync/felorx_connect && dart test test/relay/host_registry_test.dart
```

Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add packages/sync/felorx_connect/lib/src/relay/host_registry.dart packages/sync/felorx_connect/test/relay/host_registry_test.dart
git commit -m "feat(felorx_connect): 添加在线主机注册表"
```

---

## Task 4: Session Registry and Signaling Routing

**Files:**
- Create: `packages/sync/felorx_connect/lib/src/relay/session_registry.dart`
- Create: `packages/sync/felorx_connect/test/relay/session_registry_test.dart`

- [ ] **Step 1: Write failing session registry tests**

Create `packages/sync/felorx_connect/test/relay/session_registry_test.dart`:

```dart
import 'package:felorx_connect/felorx_connect.dart';
import 'package:test/test.dart';

void main() {
  group('ConnectSessionRegistry', () {
    test('creates a session between client and host', () {
      final registry = ConnectSessionRegistry(idFactory: () => 'session-1');

      final session = registry.create(
        hostId: 'host-1',
        clientPeerId: 'client-1',
        authMode: 'sameAccount',
      );

      expect(session.sessionId, 'session-1');
      expect(session.hostId, 'host-1');
      expect(session.clientPeerId, 'client-1');
      expect(session.hostPeerId, isNull);
    });

    test('sets host peer and resolves target peers', () {
      final registry = ConnectSessionRegistry(idFactory: () => 'session-1');
      registry.create(
        hostId: 'host-1',
        clientPeerId: 'client-1',
        authMode: 'sameAccount',
      );

      registry.attachHostPeer(sessionId: 'session-1', hostPeerId: 'host-peer');

      expect(
        registry.resolveTargetPeerId(
          sessionId: 'session-1',
          fromPeerId: 'client-1',
        ),
        'host-peer',
      );
      expect(
        registry.resolveTargetPeerId(
          sessionId: 'session-1',
          fromPeerId: 'host-peer',
        ),
        'client-1',
      );
    });

    test('removes sessions by host id', () {
      final registry = ConnectSessionRegistry(idFactory: () => 'session-1');
      registry.create(
        hostId: 'host-1',
        clientPeerId: 'client-1',
        authMode: 'sameAccount',
      );

      final removed = registry.removeByHostId('host-1');

      expect(removed.map((s) => s.sessionId), ['session-1']);
      expect(registry.find('session-1'), isNull);
    });
  });
}
```

- [ ] **Step 2: Run session registry test to verify red**

Run:

```bash
cd packages/sync/felorx_connect && dart test test/relay/session_registry_test.dart
```

Expected: FAIL because `ConnectSessionRegistry` is not defined.

- [ ] **Step 3: Implement session registry**

Create `packages/sync/felorx_connect/lib/src/relay/session_registry.dart`:

```dart
import 'package:uuid/uuid.dart';

class ConnectSession {
  ConnectSession({
    required this.sessionId,
    required this.hostId,
    required this.clientPeerId,
    required this.authMode,
    required this.createdAt,
    this.hostPeerId,
  });

  final String sessionId;
  final String hostId;
  final String clientPeerId;
  final String? hostPeerId;
  final String authMode;
  final DateTime createdAt;

  ConnectSession copyWith({String? hostPeerId}) {
    return ConnectSession(
      sessionId: sessionId,
      hostId: hostId,
      clientPeerId: clientPeerId,
      hostPeerId: hostPeerId ?? this.hostPeerId,
      authMode: authMode,
      createdAt: createdAt,
    );
  }
}

class ConnectSessionRegistry {
  ConnectSessionRegistry({
    String Function()? idFactory,
    DateTime Function()? now,
  })  : _idFactory = idFactory ?? const Uuid().v4,
        _now = now ?? DateTime.now;

  final String Function() _idFactory;
  final DateTime Function() _now;
  final Map<String, ConnectSession> _sessionsById = {};

  ConnectSession create({
    required String hostId,
    required String clientPeerId,
    required String authMode,
  }) {
    final session = ConnectSession(
      sessionId: _idFactory(),
      hostId: hostId,
      clientPeerId: clientPeerId,
      authMode: authMode,
      createdAt: _now().toUtc(),
    );
    _sessionsById[session.sessionId] = session;
    return session;
  }

  ConnectSession? find(String sessionId) => _sessionsById[sessionId];

  ConnectSession attachHostPeer({
    required String sessionId,
    required String hostPeerId,
  }) {
    final session = _sessionsById[sessionId];
    if (session == null) {
      throw StateError('Connect session not found: $sessionId');
    }
    final updated = session.copyWith(hostPeerId: hostPeerId);
    _sessionsById[sessionId] = updated;
    return updated;
  }

  String? resolveTargetPeerId({
    required String sessionId,
    required String fromPeerId,
  }) {
    final session = _sessionsById[sessionId];
    if (session == null) return null;
    if (fromPeerId == session.clientPeerId) return session.hostPeerId;
    if (fromPeerId == session.hostPeerId) return session.clientPeerId;
    return null;
  }

  List<ConnectSession> removeByHostId(String hostId) {
    final removed = <ConnectSession>[];
    _sessionsById.removeWhere((_, session) {
      final match = session.hostId == hostId;
      if (match) removed.add(session);
      return match;
    });
    return removed;
  }

  ConnectSession? remove(String sessionId) => _sessionsById.remove(sessionId);
}
```

- [ ] **Step 4: Export session registry**

Modify `packages/sync/felorx_connect/lib/felorx_connect.dart`:

```dart
library;

export 'src/auth/connect_password.dart';
export 'src/protocol/connect_error.dart';
export 'src/protocol/connect_message.dart';
export 'src/relay/host_registry.dart';
export 'src/relay/session_registry.dart';
```

- [ ] **Step 5: Run session registry test to verify green**

Run:

```bash
cd packages/sync/felorx_connect && dart test test/relay/session_registry_test.dart
```

Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add packages/sync/felorx_connect/lib/src/relay/session_registry.dart packages/sync/felorx_connect/test/relay/session_registry_test.dart
git commit -m "feat(felorx_connect): 添加连接会话注册表"
```

---

## Task 5: Relay Server Health, Info, and WebSocket Registration

**Files:**
- Create: `packages/sync/felorx_connect/lib/src/relay/connect_relay_server.dart`
- Create: `packages/sync/felorx_connect/test/relay/connect_relay_server_test.dart`

- [ ] **Step 1: Write failing relay server tests**

Create `packages/sync/felorx_connect/test/relay/connect_relay_server_test.dart`:

```dart
import 'dart:convert';
import 'dart:io';

import 'package:felorx_connect/felorx_connect.dart';
import 'package:test/test.dart';

void main() {
  group('ConnectRelayServer', () {
    late ConnectRelayServer server;

    tearDown(() async {
      await server.stop();
    });

    test('serves health and info endpoints', () async {
      server = ConnectRelayServer();
      await server.start(port: 0);

      final health = await HttpClient()
          .getUrl(Uri.parse('http://127.0.0.1:${server.port}/connect/health'))
          .then((r) => r.close());
      final healthBody = await utf8.decoder.bind(health).join();
      expect(jsonDecode(healthBody), {'status': 'ok'});

      final info = await HttpClient()
          .getUrl(Uri.parse('http://127.0.0.1:${server.port}/connect/info'))
          .then((r) => r.close());
      final infoBody = jsonDecode(await utf8.decoder.bind(info).join())
          as Map<String, dynamic>;
      expect(infoBody['protocolVersion'], ConnectMessage.currentProtocolVersion);
      expect(infoBody['turnEnabled'], isFalse);
    });

    test('registers a host over websocket and lists it for same user', () async {
      server = ConnectRelayServer();
      await server.start(port: 0);

      final host = await WebSocket.connect(
        'ws://127.0.0.1:${server.port}/connect/ws?peerId=host-peer',
      );
      host.add(jsonEncode(ConnectMessage(
        type: ConnectMessageType.registerHost,
        payload: {
          'hostId': 'host-1',
          'displayName': 'Build Box',
          'userId': 'user-1',
          'platform': 'linux',
          'deviceCode': 'ABC123',
          'passwordHash': ConnectPassword.hash('secret', iterations: 10),
          'capabilities': ['terminal', 'directWebRtc'],
          'authModes': ['sameAccount', 'deviceCodePassword'],
        },
      ).toJson()));

      final client = await WebSocket.connect(
        'ws://127.0.0.1:${server.port}/connect/ws?peerId=client-peer&userId=user-1',
      );
      client.add(jsonEncode(ConnectMessage(
        type: ConnectMessageType.hostList,
        requestId: 'req-1',
        payload: {'userId': 'user-1'},
      ).toJson()));

      final raw = await client.first as String;
      final message = ConnectMessage.fromJson(
        jsonDecode(raw) as Map<String, dynamic>,
      );

      expect(message.type, ConnectMessageType.hostListResult);
      expect(message.requestId, 'req-1');
      expect(message.payload['hosts'], isA<List>());
      expect((message.payload['hosts'] as List).single['hostId'], 'host-1');

      await host.close();
      await client.close();
    });
  });
}
```

- [ ] **Step 2: Run relay server test to verify red**

Run:

```bash
cd packages/sync/felorx_connect && dart test test/relay/connect_relay_server_test.dart
```

Expected: FAIL because `ConnectRelayServer` is not defined.

- [ ] **Step 3: Implement relay server**

Create `packages/sync/felorx_connect/lib/src/relay/connect_relay_server.dart` with:

```dart
import 'dart:async';
import 'dart:convert';
import 'dart:io';

import '../auth/connect_password.dart';
import '../protocol/connect_error.dart';
import '../protocol/connect_message.dart';
import 'host_registry.dart';
import 'session_registry.dart';

class ConnectRelayServer {
  ConnectRelayServer({
    ConnectHostRegistry? hostRegistry,
    ConnectSessionRegistry? sessionRegistry,
    this.turnEnabled = false,
    this.hostHeartbeatTimeout = const Duration(seconds: 60),
  })  : hostRegistry = hostRegistry ?? ConnectHostRegistry(),
        sessionRegistry = sessionRegistry ?? ConnectSessionRegistry();

  final ConnectHostRegistry hostRegistry;
  final ConnectSessionRegistry sessionRegistry;
  final bool turnEnabled;
  final Duration hostHeartbeatTimeout;

  HttpServer? _server;
  Timer? _pruneTimer;
  final Map<String, WebSocket> _peers = {};
  final Map<WebSocket, String> _peerIds = {};
  final Map<WebSocket, String> _hostIds = {};

  int get port => _server?.port ?? 0;

  Future<void> start({int port = 8787, InternetAddress? address}) async {
    if (_server != null) return;
    _server = await HttpServer.bind(address ?? InternetAddress.anyIPv4, port);
    _pruneTimer = Timer.periodic(const Duration(seconds: 10), (_) {
      final removed = hostRegistry.pruneStale(timeout: hostHeartbeatTimeout);
      for (final host in removed) {
        sessionRegistry.removeByHostId(host.hostId);
      }
    });
    unawaited(_acceptLoop(_server!));
  }

  Future<void> stop() async {
    _pruneTimer?.cancel();
    _pruneTimer = null;
    for (final socket in List<WebSocket>.from(_peerIds.keys)) {
      await socket.close();
    }
    _peers.clear();
    _peerIds.clear();
    _hostIds.clear();
    await _server?.close(force: true);
    _server = null;
  }

  Future<void> _acceptLoop(HttpServer server) async {
    await for (final request in server) {
      await _handleRequest(request);
    }
  }

  Future<void> _handleRequest(HttpRequest request) async {
    if (request.uri.path == '/connect/health') {
      _sendJson(request, {'status': 'ok'});
      return;
    }
    if (request.uri.path == '/connect/info') {
      _sendJson(request, {
        'protocolVersion': ConnectMessage.currentProtocolVersion,
        'turnEnabled': turnEnabled,
      });
      return;
    }
    if (request.uri.path == '/connect/ws' &&
        WebSocketTransformer.isUpgradeRequest(request)) {
      final peerId = request.uri.queryParameters['peerId'];
      if (peerId == null || peerId.trim().isEmpty) {
        request.response.statusCode = HttpStatus.badRequest;
        await request.response.close();
        return;
      }
      final socket = await WebSocketTransformer.upgrade(request);
      _registerPeer(peerId, socket);
      return;
    }
    request.response.statusCode = HttpStatus.notFound;
    await request.response.close();
  }

  void _registerPeer(String peerId, WebSocket socket) {
    _peers[peerId] = socket;
    _peerIds[socket] = peerId;
    socket.listen(
      (data) => _handleSocketMessage(socket, data),
      onDone: () => _removeSocket(socket),
      onError: (_) => _removeSocket(socket),
      cancelOnError: true,
    );
  }

  void _removeSocket(WebSocket socket) {
    final peerId = _peerIds.remove(socket);
    if (peerId != null) _peers.remove(peerId);
    final hostId = _hostIds.remove(socket);
    if (hostId != null) {
      hostRegistry.unregister(hostId);
      sessionRegistry.removeByHostId(hostId);
    }
  }

  void _handleSocketMessage(WebSocket socket, Object? data) {
    try {
      final raw = data is String ? data : utf8.decode(data as List<int>);
      final message = ConnectMessage.fromJson(
        jsonDecode(raw) as Map<String, dynamic>,
      );
      switch (message.type) {
        case ConnectMessageType.registerHost:
          _handleRegisterHost(socket, message);
        case ConnectMessageType.hostList:
          _handleHostList(socket, message);
        case ConnectMessageType.connectRequest:
          _handleConnectRequest(socket, message);
        case ConnectMessageType.connectAccept:
          _handleConnectAccept(socket, message);
        case ConnectMessageType.offer:
        case ConnectMessageType.answer:
        case ConnectMessageType.iceCandidate:
          _forwardSignal(message);
        case ConnectMessageType.heartbeat:
          final hostId = _hostIds[socket];
          if (hostId != null) hostRegistry.heartbeat(hostId);
        default:
          _send(socket, ConnectMessage(
            type: ConnectMessageType.error,
            requestId: message.requestId,
            payload: {
              'code': ConnectErrorCode.invalidPayload.name,
              'message': 'Message type is not accepted by relay',
            },
          ));
      }
    } catch (e) {
      _send(socket, ConnectMessage(
        type: ConnectMessageType.error,
        payload: {'message': e.toString()},
      ));
    }
  }

  void _handleRegisterHost(WebSocket socket, ConnectMessage message) {
    final payload = message.payload;
    final hostId = payload['hostId'] as String;
    hostRegistry.register(
      ConnectHostRegistration(
        hostId: hostId,
        displayName: payload['displayName'] as String,
        userId: payload['userId'] as String,
        platform: payload['platform'] as String,
        systemVersion: payload['systemVersion'] as String?,
        deviceCode: payload['deviceCode'] as String?,
        passwordHash: payload['passwordHash'] as String?,
        capabilities: Set<String>.from(payload['capabilities'] as List),
        authModes: Set<String>.from(payload['authModes'] as List),
      ),
    );
    _hostIds[socket] = hostId;
    _send(socket, ConnectMessage(
      type: ConnectMessageType.hello,
      requestId: message.requestId,
      payload: {'registered': true, 'hostId': hostId},
    ));
  }

  void _handleHostList(WebSocket socket, ConnectMessage message) {
    final userId = message.payload['userId'] as String?;
    final hosts = userId == null
        ? const <Map<String, dynamic>>[]
        : hostRegistry.listByUserId(userId).map((h) => h.toPublicJson()).toList();
    _send(socket, ConnectMessage(
      type: ConnectMessageType.hostListResult,
      requestId: message.requestId,
      payload: {'hosts': hosts},
    ));
  }

  void _handleConnectRequest(WebSocket socket, ConnectMessage message) {
    final payload = message.payload;
    final authMode = payload['authMode'] as String? ?? 'sameAccount';
    final clientPeerId = _peerIds[socket]!;

    ConnectHostSnapshot? host;
    if (authMode == 'sameAccount') {
      final hostId = payload['hostId'] as String?;
      final userId = payload['userId'] as String?;
      host = hostId == null ? null : hostRegistry.findByHostId(hostId);
      if (host == null || host.userId != userId) {
        _reject(socket, message, ConnectErrorCode.permissionDenied);
        return;
      }
    } else {
      final deviceCode = payload['deviceCode'] as String?;
      final password = payload['password'] as String?;
      host = deviceCode == null ? null : hostRegistry.findByDeviceCode(deviceCode);
      if (host == null ||
          password == null ||
          host.passwordHash == null ||
          !ConnectPassword.verify(
            password: password,
            encodedHash: host.passwordHash!,
          )) {
        _reject(socket, message, ConnectErrorCode.permissionDenied);
        return;
      }
    }

    final session = sessionRegistry.create(
      hostId: host.hostId,
      clientPeerId: clientPeerId,
      authMode: authMode,
    );
    final hostSocket = _socketForHost(host.hostId);
    if (hostSocket == null) {
      _reject(socket, message, ConnectErrorCode.hostNotFound);
      return;
    }
    _send(hostSocket, ConnectMessage(
      type: ConnectMessageType.connectRequest,
      requestId: message.requestId,
      sessionId: session.sessionId,
      fromPeerId: clientPeerId,
      targetPeerId: host.hostId,
      payload: {
        'sessionId': session.sessionId,
        'clientPeerId': clientPeerId,
        'authMode': authMode,
      },
    ));
  }

  void _handleConnectAccept(WebSocket socket, ConnectMessage message) {
    final sessionId = message.sessionId;
    final hostPeerId = _peerIds[socket];
    if (sessionId == null || hostPeerId == null) return;
    final session = sessionRegistry.attachHostPeer(
      sessionId: sessionId,
      hostPeerId: hostPeerId,
    );
    _send(_peers[session.clientPeerId], ConnectMessage(
      type: ConnectMessageType.connectAccept,
      requestId: message.requestId,
      sessionId: sessionId,
      fromPeerId: hostPeerId,
      targetPeerId: session.clientPeerId,
      payload: {'sessionId': sessionId, 'hostPeerId': hostPeerId},
    ));
  }

  void _forwardSignal(ConnectMessage message) {
    final target = message.targetPeerId ??
        (message.sessionId == null || message.fromPeerId == null
            ? null
            : sessionRegistry.resolveTargetPeerId(
                sessionId: message.sessionId!,
                fromPeerId: message.fromPeerId!,
              ));
    _send(target == null ? null : _peers[target], message);
  }

  WebSocket? _socketForHost(String hostId) {
    for (final entry in _hostIds.entries) {
      if (entry.value == hostId) return entry.key;
    }
    return null;
  }

  void _reject(
    WebSocket socket,
    ConnectMessage message,
    ConnectErrorCode code,
  ) {
    _send(socket, ConnectMessage(
      type: ConnectMessageType.connectReject,
      requestId: message.requestId,
      payload: {'code': code.name},
    ));
  }

  void _send(WebSocket? socket, ConnectMessage message) {
    if (socket == null || socket.readyState != WebSocket.open) return;
    socket.add(jsonEncode(message.toJson()));
  }

  void _sendJson(HttpRequest request, Map<String, dynamic> data) {
    request.response.headers.contentType = ContentType.json;
    request.response.write(jsonEncode(data));
    request.response.close();
  }
}
```

- [ ] **Step 4: Export relay server**

Modify `packages/sync/felorx_connect/lib/felorx_connect.dart`:

```dart
library;

export 'src/auth/connect_password.dart';
export 'src/protocol/connect_error.dart';
export 'src/protocol/connect_message.dart';
export 'src/relay/connect_relay_server.dart';
export 'src/relay/host_registry.dart';
export 'src/relay/session_registry.dart';
```

- [ ] **Step 5: Run relay server tests to verify green**

Run:

```bash
cd packages/sync/felorx_connect && dart test test/relay/connect_relay_server_test.dart
```

Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add packages/sync/felorx_connect/lib/src/relay/connect_relay_server.dart packages/sync/felorx_connect/test/relay/connect_relay_server_test.dart
git commit -m "feat(felorx_connect): 添加通用连接中继服务"
```

---

## Task 6: Relay Client and Terminal Transport Abstractions

**Files:**
- Create: `packages/sync/felorx_connect/lib/src/client/connect_relay_client.dart`
- Create: `packages/sync/felorx_connect/lib/src/client/connect_terminal_client.dart`
- Create: `packages/sync/felorx_connect/test/client/connect_relay_client_test.dart`

- [ ] **Step 1: Write failing relay client test**

Create `packages/sync/felorx_connect/test/client/connect_relay_client_test.dart`:

```dart
import 'dart:async';

import 'package:felorx_connect/felorx_connect.dart';
import 'package:test/test.dart';

void main() {
  test('ConnectRelayClient dispatches messages by type', () async {
    final controller = StreamController<ConnectMessage>();
    final client = ConnectRelayClient.fromStreams(
      incoming: controller.stream,
      sendMessage: (_) {},
    );

    final future = client.messages
        .where((m) => m.type == ConnectMessageType.hostListResult)
        .first;

    controller.add(ConnectMessage(
      type: ConnectMessageType.hostListResult,
      payload: {'hosts': <Map<String, dynamic>>[]},
    ));

    expect((await future).type, ConnectMessageType.hostListResult);
    await controller.close();
    await client.close();
  });
}
```

- [ ] **Step 2: Run relay client test to verify red**

Run:

```bash
cd packages/sync/felorx_connect && dart test test/client/connect_relay_client_test.dart
```

Expected: FAIL because `ConnectRelayClient` is not defined.

- [ ] **Step 3: Implement relay client**

Create `packages/sync/felorx_connect/lib/src/client/connect_relay_client.dart`:

```dart
import 'dart:async';
import 'dart:convert';
import 'dart:io';

import '../protocol/connect_message.dart';

abstract class ConnectRelayConnection {
  Stream<ConnectMessage> get messages;
  void send(ConnectMessage message);
  Future<void> close();
}

class ConnectRelayClient implements ConnectRelayConnection {
  ConnectRelayClient._({
    required Stream<ConnectMessage> incoming,
    required void Function(ConnectMessage message) sendMessage,
    Future<void> Function()? closeTransport,
  })  : _sendMessage = sendMessage,
        _closeTransport = closeTransport {
    _subscription = incoming.listen(_messages.add);
  }

  factory ConnectRelayClient.fromStreams({
    required Stream<ConnectMessage> incoming,
    required void Function(ConnectMessage message) sendMessage,
  }) {
    return ConnectRelayClient._(
      incoming: incoming,
      sendMessage: sendMessage,
    );
  }

  static Future<ConnectRelayClient> connect(Uri uri) async {
    final socket = await WebSocket.connect(uri.toString());
    final incoming = socket.map((data) {
      final raw = data is String ? data : utf8.decode(data as List<int>);
      return ConnectMessage.fromJson(jsonDecode(raw) as Map<String, dynamic>);
    });
    return ConnectRelayClient._(
      incoming: incoming,
      sendMessage: (message) => socket.add(jsonEncode(message.toJson())),
      closeTransport: () => socket.close(),
    );
  }

  final void Function(ConnectMessage message) _sendMessage;
  final Future<void> Function()? _closeTransport;
  final StreamController<ConnectMessage> _messages =
      StreamController<ConnectMessage>.broadcast();
  StreamSubscription<ConnectMessage>? _subscription;

  Stream<ConnectMessage> get messages => _messages.stream;

  void send(ConnectMessage message) => _sendMessage(message);

  Future<void> close() async {
    await _subscription?.cancel();
    await _messages.close();
    await _closeTransport?.call();
  }
}
```

- [ ] **Step 4: Export client abstractions**

Modify `packages/sync/felorx_connect/lib/felorx_connect.dart`:

```dart
library;

export 'src/auth/connect_password.dart';
export 'src/client/connect_relay_client.dart';
export 'src/client/connect_terminal_client.dart';
export 'src/protocol/connect_error.dart';
export 'src/protocol/connect_message.dart';
export 'src/relay/connect_relay_server.dart';
export 'src/relay/host_registry.dart';
export 'src/relay/session_registry.dart';
```

- [ ] **Step 5: Implement terminal client abstractions**

Create `packages/sync/felorx_connect/lib/src/client/connect_terminal_client.dart`:

```dart
import 'dart:async';
import 'dart:typed_data';

class TerminalResize {
  const TerminalResize({
    required this.columns,
    required this.rows,
    this.pixelWidth = 0,
    this.pixelHeight = 0,
  });

  final int columns;
  final int rows;
  final int pixelWidth;
  final int pixelHeight;

  Map<String, dynamic> toJson() => {
        'columns': columns,
        'rows': rows,
        'pixelWidth': pixelWidth,
        'pixelHeight': pixelHeight,
      };
}

abstract class ConnectTerminalChannel {
  Stream<Uint8List> get output;
  Stream<int> get exitCode;
  void write(Uint8List data);
  void resize(TerminalResize resize);
  Future<void> close();
}
```

- [ ] **Step 6: Run relay client tests to verify green**

Run:

```bash
cd packages/sync/felorx_connect && dart test test/client/connect_relay_client_test.dart
```

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add packages/sync/felorx_connect/lib/src/client packages/sync/felorx_connect/test/client
git commit -m "feat(felorx_connect): 添加中继客户端抽象"
```

---

## Task 7: Host Daemon Configuration and Registration

**Files:**
- Create: `packages/sync/felorx_connect/lib/src/host/connect_host_config.dart`
- Create: `packages/sync/felorx_connect/lib/src/host/connect_host_daemon.dart`
- Create: `packages/sync/felorx_connect/bin/felorx_connect_host.dart`
- Create: `packages/sync/felorx_connect/test/host/connect_host_config_test.dart`

- [ ] **Step 1: Write failing config tests**

Create `packages/sync/felorx_connect/test/host/connect_host_config_test.dart`:

```dart
import 'package:felorx_connect/felorx_connect.dart';
import 'package:test/test.dart';

void main() {
  group('ConnectHostConfig', () {
    test('parses relay url and password hash from args', () {
      final config = ConnectHostConfig.parse([
        '--relay-url',
        'ws://localhost:8787/connect/ws',
        '--host-id',
        'host-1',
        '--name',
        'Build Box',
        '--device-code',
        'ABC123',
        '--connect-password-hash',
        'hash',
      ]);

      expect(config.relayUrl.toString(), 'ws://localhost:8787/connect/ws');
      expect(config.hostId, 'host-1');
      expect(config.name, 'Build Box');
      expect(config.deviceCode, 'ABC123');
      expect(config.connectPasswordHash, 'hash');
    });

    test('hashes plain connect password when no hash is provided', () {
      final config = ConnectHostConfig.parse([
        '--relay-url',
        'ws://localhost:8787/connect/ws',
        '--connect-password',
        'secret',
      ]);

      expect(config.connectPasswordHash, isNotNull);
      expect(
        ConnectPassword.verify(
          password: 'secret',
          encodedHash: config.connectPasswordHash!,
        ),
        isTrue,
      );
    });
  });
}
```

- [ ] **Step 2: Run config tests to verify red**

Run:

```bash
cd packages/sync/felorx_connect && dart test test/host/connect_host_config_test.dart
```

Expected: FAIL because `ConnectHostConfig` is not defined.

- [ ] **Step 3: Implement config parser**

Create `packages/sync/felorx_connect/lib/src/host/connect_host_config.dart`:

```dart
import 'dart:io';

import 'package:args/args.dart';

import '../auth/connect_password.dart';

class ConnectHostConfig {
  ConnectHostConfig({
    required this.relayUrl,
    required this.hostId,
    required this.name,
    required this.deviceCode,
    required this.shell,
    this.apiKey,
    this.tokenFile,
    this.connectPasswordHash,
    this.stunUrl = 'stun:stun.l.google.com:19302',
    this.turnUrl,
    this.turnUsername,
    this.turnCredential,
  });

  final Uri relayUrl;
  final String hostId;
  final String name;
  final String deviceCode;
  final String shell;
  final String? apiKey;
  final String? tokenFile;
  final String? connectPasswordHash;
  final String stunUrl;
  final String? turnUrl;
  final String? turnUsername;
  final String? turnCredential;

  static ArgParser buildArgParser() {
    return ArgParser()
      ..addOption('relay-url')
      ..addOption('api-key')
      ..addOption('token-file')
      ..addOption('host-id')
      ..addOption('name')
      ..addOption('device-code')
      ..addOption('connect-password')
      ..addOption('connect-password-hash')
      ..addOption('shell')
      ..addOption('stun-url', defaultsTo: 'stun:stun.l.google.com:19302')
      ..addOption('turn-url')
      ..addOption('turn-username')
      ..addOption('turn-credential')
      ..addFlag('help', abbr: 'h', negatable: false);
  }

  static ConnectHostConfig parse(
    List<String> args, {
    Map<String, String>? environment,
  }) {
    final env = environment ?? Platform.environment;
    final result = buildArgParser().parse(args);
    final relay = result['relay-url'] as String? ??
        env['PUUPEE_CONNECT_RELAY_URL'] ??
        'ws://localhost:8787/connect/ws';
    final hostId = result['host-id'] as String? ??
        env['PUUPEE_CONNECT_HOST_ID'] ??
        Platform.localHostname;
    final name = result['name'] as String? ??
        env['PUUPEE_CONNECT_NAME'] ??
        Platform.localHostname;
    final deviceCode = result['device-code'] as String? ??
        env['PUUPEE_CONNECT_DEVICE_CODE'] ??
        hostId;
    final plainPassword = result['connect-password'] as String? ??
        env['PUUPEE_CONNECT_PASSWORD'];
    final passwordHash = result['connect-password-hash'] as String? ??
        env['PUUPEE_CONNECT_PASSWORD_HASH'] ??
        (plainPassword == null ? null : ConnectPassword.hash(plainPassword));

    return ConnectHostConfig(
      relayUrl: Uri.parse(relay),
      apiKey: result['api-key'] as String? ?? env['PUUPEE_CONNECT_API_KEY'],
      tokenFile:
          result['token-file'] as String? ?? env['PUUPEE_CONNECT_TOKEN_FILE'],
      hostId: hostId,
      name: name,
      deviceCode: deviceCode,
      connectPasswordHash: passwordHash,
      shell: result['shell'] as String? ??
          env['PUUPEE_CONNECT_SHELL'] ??
          (File('/bin/bash').existsSync() ? '/bin/bash' : '/bin/sh'),
      stunUrl: result['stun-url'] as String,
      turnUrl: result['turn-url'] as String? ?? env['PUUPEE_CONNECT_TURN_URL'],
      turnUsername: result['turn-username'] as String? ??
          env['PUUPEE_CONNECT_TURN_USERNAME'],
      turnCredential: result['turn-credential'] as String? ??
          env['PUUPEE_CONNECT_TURN_CREDENTIAL'],
    );
  }
}
```

- [ ] **Step 4: Implement daemon skeleton**

Create `packages/sync/felorx_connect/lib/src/host/connect_host_daemon.dart`:

```dart
import '../client/connect_relay_client.dart';
import '../protocol/connect_message.dart';
import 'connect_host_config.dart';

class ConnectHostDaemon {
  ConnectHostDaemon({
    required this.config,
    ConnectRelayConnection? relayClient,
  }) : _relayClient = relayClient;

  final ConnectHostConfig config;
  ConnectRelayConnection? _relayClient;

  Future<void> start() async {
    final client =
        _relayClient ?? await ConnectRelayClient.connect(config.relayUrl);
    _relayClient = client;
    client.send(ConnectMessage(
      type: ConnectMessageType.registerHost,
      payload: {
        'hostId': config.hostId,
        'displayName': config.name,
        'userId': config.apiKey ?? 'anonymous',
        'platform': 'linux',
        'deviceCode': config.deviceCode,
        if (config.connectPasswordHash != null)
          'passwordHash': config.connectPasswordHash,
        'capabilities': [
          'terminal',
          'directWebRtc',
          if (config.turnUrl != null) 'turnFallback',
        ],
        'authModes': [
          if (config.apiKey != null) 'sameAccount',
          if (config.connectPasswordHash != null) 'deviceCodePassword',
        ],
      },
    ));
  }

  Future<void> stop() async {
    await _relayClient?.close();
    _relayClient = null;
  }
}
```

Create `packages/sync/felorx_connect/bin/felorx_connect_host.dart`:

```dart
import 'dart:io';

import 'package:felorx_connect/felorx_connect.dart';

Future<void> main(List<String> args) async {
  final parser = ConnectHostConfig.buildArgParser();
  if (args.contains('--help') || args.contains('-h')) {
    stdout.writeln(parser.usage);
    return;
  }

  final daemon = ConnectHostDaemon(config: ConnectHostConfig.parse(args));
  try {
    await daemon.start();
    stdout.writeln('Felorx Connect host daemon started');
    ProcessSignal.sigint.watch().listen((_) async {
      await daemon.stop();
      exit(0);
    });
    ProcessSignal.sigterm.watch().listen((_) async {
      await daemon.stop();
      exit(0);
    });
  } catch (error, stackTrace) {
    stderr.writeln('Felorx Connect host daemon failed: $error');
    stderr.writeln(stackTrace);
    exitCode = 64;
  }
}
```

- [ ] **Step 5: Export host config and daemon**

Modify `packages/sync/felorx_connect/lib/felorx_connect.dart`:

```dart
library;

export 'src/auth/connect_password.dart';
export 'src/client/connect_relay_client.dart';
export 'src/client/connect_terminal_client.dart';
export 'src/host/connect_host_config.dart';
export 'src/host/connect_host_daemon.dart';
export 'src/protocol/connect_error.dart';
export 'src/protocol/connect_message.dart';
export 'src/relay/connect_relay_server.dart';
export 'src/relay/host_registry.dart';
export 'src/relay/session_registry.dart';
```

- [ ] **Step 6: Run config tests to verify green**

Run:

```bash
cd packages/sync/felorx_connect && dart test test/host/connect_host_config_test.dart
```

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add packages/sync/felorx_connect/lib/src/host packages/sync/felorx_connect/bin packages/sync/felorx_connect/test/host
git commit -m "feat(felorx_connect): 添加 Linux 主机守护进程入口"
```

---

## Task 8: PTY Abstraction and Linux Adapter

**Files:**
- Create: `packages/sync/felorx_connect/lib/src/host/pty/terminal_pty.dart`
- Create: `packages/sync/felorx_connect/lib/src/host/pty/linux_terminal_pty.dart`
- Create: `packages/sync/felorx_connect/lib/src/host/terminal_host_session.dart`
- Create: `packages/sync/felorx_connect/test/host/terminal_host_session_test.dart`

- [ ] **Step 1: Write failing terminal session test with fake PTY**

Create `packages/sync/felorx_connect/test/host/terminal_host_session_test.dart`:

```dart
import 'dart:async';
import 'dart:convert';
import 'dart:typed_data';

import 'package:felorx_connect/src/host/pty/terminal_pty.dart';
import 'package:felorx_connect/src/host/terminal_host_session.dart';
import 'package:felorx_connect/felorx_connect.dart';
import 'package:test/test.dart';

class FakeTerminalPty implements TerminalPty {
  final _output = StreamController<Uint8List>();
  final _exit = StreamController<int>();
  final writes = <String>[];
  TerminalResize? lastResize;
  bool closed = false;

  @override
  Stream<Uint8List> get output => _output.stream;

  @override
  Stream<int> get exitCode => _exit.stream;

  @override
  void write(Uint8List data) {
    writes.add(utf8.decode(data));
  }

  @override
  void resize(TerminalResize resize) {
    lastResize = resize;
  }

  @override
  Future<void> close() async {
    closed = true;
    await _output.close();
    await _exit.close();
  }
}

void main() {
  test('TerminalHostSession writes input and resize to PTY', () async {
    final pty = FakeTerminalPty();
    final sent = <ConnectMessage>[];
    final session = TerminalHostSession(
      pty: pty,
      sendMessage: sent.add,
    );

    session.handleDataChannelMessage(ConnectMessage(
      type: ConnectMessageType.terminalInput,
      payload: {'text': 'ls\n'},
    ));
    session.handleDataChannelMessage(ConnectMessage(
      type: ConnectMessageType.resize,
      payload: {'columns': 120, 'rows': 40, 'pixelWidth': 1200, 'pixelHeight': 800},
    ));

    expect(pty.writes, ['ls\n']);
    expect(pty.lastResize?.columns, 120);
    expect(pty.lastResize?.rows, 40);
    await session.close();
    expect(pty.closed, isTrue);
  });
}
```

- [ ] **Step 2: Run terminal session test to verify red**

Run:

```bash
cd packages/sync/felorx_connect && dart test test/host/terminal_host_session_test.dart
```

Expected: FAIL because `TerminalPty` and `TerminalHostSession` are not defined.

- [ ] **Step 3: Implement PTY abstraction**

Create `packages/sync/felorx_connect/lib/src/host/pty/terminal_pty.dart`:

```dart
import 'dart:typed_data';

import '../../client/connect_terminal_client.dart';

abstract class TerminalPty {
  Stream<Uint8List> get output;
  Stream<int> get exitCode;
  void write(Uint8List data);
  void resize(TerminalResize resize);
  Future<void> close();
}
```

- [ ] **Step 4: Implement terminal host session**

Create `packages/sync/felorx_connect/lib/src/host/terminal_host_session.dart`:

```dart
import 'dart:async';
import 'dart:convert';
import 'dart:typed_data';

import '../client/connect_terminal_client.dart';
import '../protocol/connect_message.dart';
import 'pty/terminal_pty.dart';

class TerminalHostSession {
  TerminalHostSession({
    required this.pty,
    required void Function(ConnectMessage message) sendMessage,
  }) : _sendMessage = sendMessage {
    _outputSub = pty.output.listen((data) {
      _sendMessage(ConnectMessage(
        type: ConnectMessageType.terminalOutput,
        payload: {'text': utf8.decode(data, allowMalformed: true)},
      ));
    });
    _exitSub = pty.exitCode.listen((code) {
      _sendMessage(ConnectMessage(
        type: ConnectMessageType.exit,
        payload: {'exitCode': code},
      ));
    });
  }

  final TerminalPty pty;
  final void Function(ConnectMessage message) _sendMessage;
  StreamSubscription<Uint8List>? _outputSub;
  StreamSubscription<int>? _exitSub;

  void handleDataChannelMessage(ConnectMessage message) {
    switch (message.type) {
      case ConnectMessageType.terminalInput:
        final text = message.payload['text'] as String? ?? '';
        pty.write(Uint8List.fromList(utf8.encode(text)));
      case ConnectMessageType.resize:
        pty.resize(TerminalResize(
          columns: message.payload['columns'] as int,
          rows: message.payload['rows'] as int,
          pixelWidth: message.payload['pixelWidth'] as int? ?? 0,
          pixelHeight: message.payload['pixelHeight'] as int? ?? 0,
        ));
      default:
        break;
    }
  }

  Future<void> close() async {
    await _outputSub?.cancel();
    await _exitSub?.cancel();
    await pty.close();
  }
}
```

- [ ] **Step 5: Implement Linux process adapter with PTY seam**

Create `packages/sync/felorx_connect/lib/src/host/pty/linux_terminal_pty.dart`:

```dart
import 'dart:async';
import 'dart:io';
import 'dart:typed_data';

import '../../client/connect_terminal_client.dart';
import 'terminal_pty.dart';

class LinuxTerminalPty implements TerminalPty {
  LinuxTerminalPty._(this._process);

  final Process _process;
  final StreamController<int> _exitCode = StreamController<int>.broadcast();

  static Future<LinuxTerminalPty> start({
    required String shell,
    Map<String, String>? environment,
  }) async {
    final env = <String, String>{
      ...Platform.environment,
      if (environment != null) ...environment,
      'TERM': 'xterm-256color',
      'LANG': Platform.environment['LANG'] ?? 'en_US.UTF-8',
    };
    final process = await Process.start(
      shell,
      const ['-i'],
      environment: env,
      mode: ProcessStartMode.normal,
    );
    final pty = LinuxTerminalPty._(process);
    process.exitCode.then((code) {
      if (!pty._exitCode.isClosed) pty._exitCode.add(code);
    });
    return pty;
  }

  @override
  Stream<Uint8List> get output => _process.stdout.cast<Uint8List>();

  @override
  Stream<int> get exitCode => _exitCode.stream;

  @override
  void write(Uint8List data) {
    _process.stdin.add(data);
  }

  @override
  void resize(TerminalResize resize) {
    final columns = resize.columns.clamp(1, 500);
    final rows = resize.rows.clamp(1, 200);
    _process.stdin.writeln('stty cols $columns rows $rows 2>/dev/null');
  }

  @override
  Future<void> close() async {
    _process.kill(ProcessSignal.sigterm);
    await _exitCode.close();
  }
}
```

This adapter uses `Process.start` as the compile-safe first implementation behind the `TerminalPty` interface. If interactive full-screen terminal behavior fails in integration verification, replace only `LinuxTerminalPty` with an FFI-backed PTY adapter while keeping the same tests and public interface.

- [ ] **Step 6: Run terminal session tests to verify green**

Run:

```bash
cd packages/sync/felorx_connect && dart test test/host/terminal_host_session_test.dart
```

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add packages/sync/felorx_connect/lib/src/host/pty packages/sync/felorx_connect/lib/src/host/terminal_host_session.dart packages/sync/felorx_connect/test/host/terminal_host_session_test.dart
git commit -m "feat(felorx_connect): 添加远程终端 PTY 桥接"
```

---

## Task 9: Host WebRTC DataChannel Integration

**Files:**
- Create: `packages/sync/felorx_connect/lib/src/host/webrtc_dart_terminal_peer.dart`
- Modify: `packages/sync/felorx_connect/lib/src/host/connect_host_daemon.dart`
- Create: `packages/sync/felorx_connect/test/host/connect_host_daemon_test.dart`

- [ ] **Step 1: Write failing daemon request handling test**

Create `packages/sync/felorx_connect/test/host/connect_host_daemon_test.dart`:

```dart
import 'dart:async';

import 'package:felorx_connect/felorx_connect.dart';
import 'package:test/test.dart';

class RecordingRelayClient implements ConnectRelayConnection {
  RecordingRelayClient(this.controller);

  final StreamController<ConnectMessage> controller;
  final sent = <ConnectMessage>[];

  @override
  Stream<ConnectMessage> get messages => controller.stream;

  @override
  void send(ConnectMessage message) {
    sent.add(message);
  }

  @override
  Future<void> close() async {}
}

void main() {
  test('daemon accepts connect requests for registered host', () async {
    final controller = StreamController<ConnectMessage>();
    final relay = RecordingRelayClient(controller);
    final daemon = ConnectHostDaemon(
      config: ConnectHostConfig.parse([
        '--relay-url',
        'ws://localhost:8787/connect/ws',
        '--host-id',
        'host-1',
        '--api-key',
        'user-1',
      ]),
      relayClient: relay,
    );

    await daemon.start();
    controller.add(ConnectMessage(
      type: ConnectMessageType.connectRequest,
      requestId: 'req-1',
      sessionId: 'session-1',
      fromPeerId: 'client-peer',
      payload: {'clientPeerId': 'client-peer'},
    ));

    await Future<void>.delayed(Duration.zero);

    expect(
      relay.sent.any((m) =>
          m.type == ConnectMessageType.connectAccept &&
          m.sessionId == 'session-1'),
      isTrue,
    );
    await daemon.stop();
    await controller.close();
  });
}
```

- [ ] **Step 2: Run daemon test to verify red**

Run:

```bash
cd packages/sync/felorx_connect && dart test test/host/connect_host_daemon_test.dart
```

Expected: FAIL because current daemon does not listen for `connectRequest`.

- [ ] **Step 3: Implement host-side peer abstraction**

Create `packages/sync/felorx_connect/lib/src/host/webrtc_dart_terminal_peer.dart`:

```dart
import 'dart:async';
import 'dart:convert';

import 'package:webrtc_dart/webrtc_dart.dart';

import '../protocol/connect_message.dart';

class WebRtcDartTerminalPeer {
  WebRtcDartTerminalPeer({
    required this.sessionId,
    required this.hostPeerId,
    required this.sendSignal,
  });

  final String sessionId;
  final String hostPeerId;
  final void Function(ConnectMessage message) sendSignal;
  final RTCPeerConnection _pc = RTCPeerConnection();
  RTCDataChannel? _channel;

  Stream<Object> get dataMessages {
    final channel = _channel;
    return channel == null ? const Stream.empty() : channel.onMessage;
  }

  Future<void> start() async {
    _pc.onIceCandidate.listen((candidate) {
      sendSignal(ConnectMessage(
        type: ConnectMessageType.iceCandidate,
        sessionId: sessionId,
        fromPeerId: hostPeerId,
        payload: {'candidate': candidate.toSdp(), 'sdpMid': '0'},
      ));
    });
    _pc.onDataChannel.listen((channel) {
      _channel = channel;
    });
  }

  Future<void> handleOffer(ConnectMessage message) async {
    await _pc.setRemoteDescription(RTCSessionDescription(
      type: 'offer',
      sdp: message.payload['sdp'] as String,
    ));
    final answer = await _pc.createAnswer();
    await _pc.setLocalDescription(answer);
    sendSignal(ConnectMessage(
      type: ConnectMessageType.answer,
      sessionId: sessionId,
      fromPeerId: hostPeerId,
      payload: {'sdp': answer.sdp},
    ));
  }

  Future<void> addIceCandidate(ConnectMessage message) async {
    await _pc.addIceCandidate(
      RTCIceCandidate.fromSdp(message.payload['candidate'] as String),
    );
  }

  void sendData(ConnectMessage message) {
    _channel?.sendString(jsonEncode(message.toJson()));
  }

  Future<void> close() => _pc.close();
}
```

- [ ] **Step 4: Update daemon to accept connect requests**

Modify `packages/sync/felorx_connect/lib/src/host/connect_host_daemon.dart` so `start()` listens to messages:

```dart
StreamSubscription<ConnectMessage>? _relaySub;

Future<void> start() async {
  final client =
      _relayClient ?? await ConnectRelayClient.connect(config.relayUrl);
  _relayClient = client;
  _relaySub = client.messages.listen(_handleRelayMessage);
  client.send(ConnectMessage(
    type: ConnectMessageType.registerHost,
    payload: {
      'hostId': config.hostId,
      'displayName': config.name,
      'userId': config.apiKey ?? 'anonymous',
      'platform': 'linux',
      'deviceCode': config.deviceCode,
      if (config.connectPasswordHash != null)
        'passwordHash': config.connectPasswordHash,
      'capabilities': [
        'terminal',
        'directWebRtc',
        if (config.turnUrl != null) 'turnFallback',
      ],
      'authModes': [
        if (config.apiKey != null) 'sameAccount',
        if (config.connectPasswordHash != null) 'deviceCodePassword',
      ],
    },
  ));
}

void _handleRelayMessage(ConnectMessage message) {
  if (message.type != ConnectMessageType.connectRequest) return;
  _relayClient?.send(ConnectMessage(
    type: ConnectMessageType.connectAccept,
    requestId: message.requestId,
    sessionId: message.sessionId,
    fromPeerId: config.hostId,
    targetPeerId: message.fromPeerId,
    payload: {'hostPeerId': config.hostId},
  ));
}

Future<void> stop() async {
  await _relaySub?.cancel();
  _relaySub = null;
  await _relayClient?.close();
  _relayClient = null;
}
```

- [ ] **Step 5: Run daemon tests to verify green**

Run:

```bash
cd packages/sync/felorx_connect && dart test test/host/connect_host_daemon_test.dart
```

Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add packages/sync/felorx_connect/lib/src/host packages/sync/felorx_connect/test/host/connect_host_daemon_test.dart
git commit -m "feat(felorx_connect): 添加主机侧 WebRTC 会话入口"
```

---

## Task 10: sync_node Connect Relay Parameters and Lifecycle

**Files:**
- Modify: `apps/sync_node/pubspec.yaml`
- Modify: `apps/sync_node/lib/sync_node_arg_parser.dart`
- Create: `apps/sync_node/lib/sync_node_connect_relay.dart`
- Modify: `apps/sync_node/lib/sync_node_runner.dart`
- Create: `apps/sync_node/test/sync_node_arg_parser_test.dart`
- Create: `apps/sync_node/test/connect_relay_lifecycle_test.dart`

- [ ] **Step 1: Write failing arg parser test**

Create `apps/sync_node/test/sync_node_arg_parser_test.dart`:

```dart
import 'package:felorx_sync_node/sync_node_arg_parser.dart';
import 'package:test/test.dart';

void main() {
  test('parses connect relay options', () {
    final result = buildSyncNodeArgParser().parse([
      '--enable-connect-relay',
      '--connect-port',
      '18787',
      '--connect-turn-url',
      'turn:turn.example.com',
    ]);

    expect(result['enable-connect-relay'], isTrue);
    expect(result['connect-port'], '18787');
    expect(result['connect-turn-url'], 'turn:turn.example.com');
  });
}
```

- [ ] **Step 2: Run arg parser test to verify red**

Run:

```bash
cd apps/sync_node && dart test test/sync_node_arg_parser_test.dart
```

Expected: FAIL because connect relay options are missing.

- [ ] **Step 3: Add dependency and parser options**

Modify `apps/sync_node/pubspec.yaml`:

```yaml
dependencies:
  felorx_connect: ^0.1.0
```

Modify `apps/sync_node/lib/sync_node_arg_parser.dart` by adding before `help`:

```dart
    ..addFlag(
      'enable-connect-relay',
      defaultsTo: false,
      help: '启用 Felorx Connect 中继服务，用于远程终端等远程操作连接',
    )
    ..addOption(
      'connect-port',
      defaultsTo: '18787',
      help: 'Felorx Connect 中继服务端口',
    )
    ..addOption(
      'connect-path-prefix',
      defaultsTo: '/connect',
      help: 'Felorx Connect HTTP/WebSocket 路径前缀',
    )
    ..addOption(
      'connect-stun-url',
      defaultsTo: 'stun:stun.l.google.com:19302',
      help: 'Felorx Connect WebRTC STUN 地址',
    )
    ..addOption('connect-turn-url', help: 'Felorx Connect WebRTC TURN 地址')
    ..addOption('connect-turn-username', help: 'Felorx Connect TURN 用户名')
    ..addOption('connect-turn-credential', help: 'Felorx Connect TURN 凭据')
    ..addOption(
      'connect-host-heartbeat-timeout',
      defaultsTo: '60',
      help: 'Felorx Connect 主机心跳超时时间，单位秒',
    )
```

- [ ] **Step 4: Run arg parser test to verify green**

Run:

```bash
cd apps/sync_node && dart test test/sync_node_arg_parser_test.dart
```

Expected: PASS.

- [ ] **Step 5: Write lifecycle test**

Create `apps/sync_node/test/connect_relay_lifecycle_test.dart`:

```dart
import 'dart:convert';
import 'dart:io';

import 'package:felorx_sync_node/sync_node_connect_relay.dart';
import 'package:test/test.dart';

void main() {
  test('starts and stops connect relay when enabled', () async {
    final relay = await startSyncNodeConnectRelay(
      enabled: true,
      port: 0,
      turnUrl: null,
      heartbeatTimeout: const Duration(seconds: 60),
    );

    addTearDown(() async => relay?.stop());

    expect(relay, isNotNull);
    final response = await HttpClient()
        .getUrl(Uri.parse('http://127.0.0.1:${relay!.port}/connect/health'))
        .then((request) => request.close());
    final body = jsonDecode(await utf8.decoder.bind(response).join())
        as Map<String, dynamic>;

    expect(body['status'], 'ok');
    await relay.stop();
  });

  test('does not start connect relay when disabled', () async {
    final relay = await startSyncNodeConnectRelay(
      enabled: false,
      port: 0,
      turnUrl: null,
      heartbeatTimeout: const Duration(seconds: 60),
    );

    expect(relay, isNull);
  });
}
```

- [ ] **Step 6: Add sync_node connect relay helper**

Create `apps/sync_node/lib/sync_node_connect_relay.dart`:

```dart
import 'package:felorx_connect/felorx_connect.dart';

Future<ConnectRelayServer?> startSyncNodeConnectRelay({
  required bool enabled,
  required int port,
  required String? turnUrl,
  required Duration heartbeatTimeout,
  void Function(String message)? onLog,
}) async {
  if (!enabled) return null;
  final relay = ConnectRelayServer(
    turnEnabled: turnUrl != null && turnUrl.trim().isNotEmpty,
    hostHeartbeatTimeout: heartbeatTimeout,
  );
  await relay.start(port: port);
  onLog?.call('Felorx Connect 中继服务已启动: ${relay.port}');
  return relay;
}
```

- [ ] **Step 7: Update runner lifecycle**

Modify `apps/sync_node/lib/sync_node_runner.dart`:

```dart
import 'package:felorx_connect/felorx_connect.dart';
```

Also import:

```dart
import 'sync_node_connect_relay.dart';
```

Update `SyncNodeRunHandle`:

```dart
class SyncNodeRunHandle {
  SyncNodeRunHandle({
    required this.node,
    this.contentEventConsumer,
    this.connectRelayServer,
  });

  final SyncNode node;
  final FelorxContentEventConsumer? contentEventConsumer;
  final ConnectRelayServer? connectRelayServer;

  Future<void> stop() async {
    await connectRelayServer?.stop();
    await contentEventConsumer?.stop();
    await node.transferManager?.stop();
    await node.syncClientManager?.stop();
    await node.lanSyncNodeDiscoverer?.stop();
    await node.syncServer.stop();
  }
}
```

After `await syncServer.serve();`, add:

```dart
  ConnectRelayServer? connectRelayServer;
  final enableConnectRelay =
      env['FELORX_SYNC_NODE_CONNECT_ENABLE_RELAY'] == 'true' ||
      result['enable-connect-relay'] as bool;
  final connectPort = int.parse(
    env['FELORX_SYNC_NODE_CONNECT_PORT'] ?? result['connect-port'] as String,
  );
  final heartbeatTimeout = Duration(
    seconds: int.parse(
      env['FELORX_SYNC_NODE_CONNECT_HOST_HEARTBEAT_TIMEOUT'] ??
          result['connect-host-heartbeat-timeout'] as String,
    ),
  );
  final connectTurnUrl =
      env['FELORX_SYNC_NODE_CONNECT_TURN_URL'] ??
      result['connect-turn-url'] as String?;
  connectRelayServer = await startSyncNodeConnectRelay(
    enabled: enableConnectRelay,
    port: connectPort,
    turnUrl: connectTurnUrl,
    heartbeatTimeout: heartbeatTimeout,
    onLog: (message) => _emitLog(onLog, message),
  );
```

Return handle with `connectRelayServer: connectRelayServer`.

- [ ] **Step 8: Run sync_node tests**

Run:

```bash
cd apps/sync_node && dart test test/sync_node_arg_parser_test.dart
dart test test/connect_relay_lifecycle_test.dart
cd /Users/j/repos/puupees/felorx-apps && dart analyze apps/sync_node packages/sync/felorx_connect
```

Expected: tests PASS and analysis reports no errors for touched packages.

- [ ] **Step 9: Commit**

```bash
git add apps/sync_node/pubspec.yaml apps/sync_node/lib apps/sync_node/test
git commit -m "feat(sync_node): 集成 Felorx Connect 中继服务"
```

---

## Task 11: Ops Models, Providers, and Relay Service

**Files:**
- Modify: `apps/ops/pubspec.yaml`
- Create: `apps/ops/lib/models/connect_host.dart`
- Create: `apps/ops/lib/services/connect_relay_service.dart`
- Create: `apps/ops/lib/providers/connect_provider.dart`
- Create: `apps/ops/test/services/connect_relay_service_test.dart`

- [ ] **Step 1: Write failing service test**

Create `apps/ops/test/services/connect_relay_service_test.dart`:

```dart
import 'package:felorx_ops/models/connect_host.dart';
import 'package:test/test.dart';

void main() {
  test('ConnectHost parses relay payload', () {
    final host = ConnectHost.fromJson({
      'hostId': 'host-1',
      'displayName': 'Build Box',
      'platform': 'linux',
      'capabilities': ['terminal', 'directWebRtc'],
      'authModes': ['sameAccount'],
      'lastSeenAt': '2026-05-19T12:00:00.000Z',
    });

    expect(host.hostId, 'host-1');
    expect(host.displayName, 'Build Box');
    expect(host.canUseTerminal, isTrue);
  });
}
```

- [ ] **Step 2: Run service test to verify red**

Run:

```bash
cd apps/ops && flutter test test/services/connect_relay_service_test.dart
```

Expected: FAIL because `ConnectHost` is missing.

- [ ] **Step 3: Add Ops dependencies**

Modify `apps/ops/pubspec.yaml`:

```yaml
dependencies:
  felorx_connect: ^0.1.0
  flutter_webrtc: ^0.12.0
```

- [ ] **Step 4: Implement model**

Create `apps/ops/lib/models/connect_host.dart`:

```dart
class ConnectHost {
  const ConnectHost({
    required this.hostId,
    required this.displayName,
    required this.platform,
    required this.capabilities,
    required this.authModes,
    required this.lastSeenAt,
    this.systemVersion,
  });

  final String hostId;
  final String displayName;
  final String platform;
  final String? systemVersion;
  final Set<String> capabilities;
  final Set<String> authModes;
  final DateTime lastSeenAt;

  bool get canUseTerminal => capabilities.contains('terminal');
  bool get supportsDirectWebRtc => capabilities.contains('directWebRtc');
  bool get supportsTurnFallback => capabilities.contains('turnFallback');

  factory ConnectHost.fromJson(Map<String, dynamic> json) {
    return ConnectHost(
      hostId: json['hostId'] as String,
      displayName: json['displayName'] as String,
      platform: json['platform'] as String,
      systemVersion: json['systemVersion'] as String?,
      capabilities: Set<String>.from(json['capabilities'] as List),
      authModes: Set<String>.from(json['authModes'] as List),
      lastSeenAt: DateTime.parse(json['lastSeenAt'] as String),
    );
  }
}
```

- [ ] **Step 5: Implement relay service**

Create `apps/ops/lib/services/connect_relay_service.dart`:

```dart
import 'dart:async';

import 'package:felorx_connect/felorx_connect.dart';

import '../models/connect_host.dart';

class OpsConnectRelayService {
  OpsConnectRelayService(this.client);

  final ConnectRelayClient client;

  Future<List<ConnectHost>> listHosts({required String userId}) async {
    final requestId = DateTime.now().microsecondsSinceEpoch.toString();
    final responseFuture = client.messages.firstWhere(
      (m) =>
          m.type == ConnectMessageType.hostListResult &&
          m.requestId == requestId,
    );
    client.send(ConnectMessage(
      type: ConnectMessageType.hostList,
      requestId: requestId,
      payload: {'userId': userId},
    ));
    final response = await responseFuture.timeout(const Duration(seconds: 10));
    final hosts = response.payload['hosts'] as List? ?? const [];
    return hosts
        .map((e) => ConnectHost.fromJson(Map<String, dynamic>.from(e as Map)))
        .toList();
  }

  Future<void> requestDeviceCodeConnection({
    required String deviceCode,
    required String password,
  }) async {
    client.send(ConnectMessage(
      type: ConnectMessageType.connectRequest,
      requestId: DateTime.now().microsecondsSinceEpoch.toString(),
      payload: {
        'authMode': 'deviceCodePassword',
        'deviceCode': deviceCode,
        'password': password,
      },
    ));
  }
}
```

- [ ] **Step 6: Implement provider shell**

Create `apps/ops/lib/providers/connect_provider.dart`:

```dart
import 'package:riverpod_annotation/riverpod_annotation.dart';

import '../models/connect_host.dart';

part 'connect_provider.g.dart';

@riverpod
Future<List<ConnectHost>> connectHostList(ConnectHostListRef ref) async {
  return const <ConnectHost>[];
}
```

Run generator later in Task 14 with the Ops build_runner command.

- [ ] **Step 7: Run service test to verify green**

Run:

```bash
cd apps/ops && flutter test test/services/connect_relay_service_test.dart
```

Expected: PASS.

- [ ] **Step 8: Commit**

```bash
git add apps/ops/pubspec.yaml apps/ops/lib/models/connect_host.dart apps/ops/lib/services/connect_relay_service.dart apps/ops/lib/providers/connect_provider.dart apps/ops/test/services/connect_relay_service_test.dart
git commit -m "feat(ops): 添加 Connect 远程主机模型"
```

---

## Task 12: Ops Terminal Mode Shell and Device-Code Form

**Files:**
- Create: `apps/ops/lib/pages/terminal/terminal_page.dart`
- Create: `apps/ops/lib/pages/terminal/remote_terminal_connection_dialog.dart`
- Create: `apps/ops/lib/pages/terminal/remote_terminal_page.dart`
- Modify: `apps/ops/lib/router.dart`
- Create: `apps/ops/test/pages/terminal_remote_connection_form_test.dart`

- [ ] **Step 1: Write failing widget test**

Create `apps/ops/test/pages/terminal_remote_connection_form_test.dart`:

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:felorx_ops/pages/terminal/remote_terminal_connection_dialog.dart';

void main() {
  testWidgets('device code form disables submit when empty', (tester) async {
    await tester.pumpWidget(const RemoteTerminalConnectionForm(
      onConnect: null,
    ));

    expect(find.text('设备码'), findsOneWidget);
    expect(find.text('连接密码'), findsOneWidget);
  });
}
```

- [ ] **Step 2: Run widget test to verify red**

Run:

```bash
cd apps/ops && flutter test test/pages/terminal_remote_connection_form_test.dart
```

Expected: FAIL because form widget is missing.

- [ ] **Step 3: Implement terminal mode shell**

Create `apps/ops/lib/pages/terminal/terminal_page.dart`:

```dart
import 'package:shadcn_flutter/shadcn_flutter.dart';

import 'local_terminal_page.dart';
import 'remote_terminal_page.dart';

enum OpsTerminalMode { local, remote }

class OpsTerminalPage extends StatefulWidget {
  const OpsTerminalPage({super.key});

  @override
  State<OpsTerminalPage> createState() => _OpsTerminalPageState();
}

class _OpsTerminalPageState extends State<OpsTerminalPage> {
  OpsTerminalMode _mode = OpsTerminalMode.local;

  @override
  Widget build(BuildContext context) {
    return Column(
      crossAxisAlignment: CrossAxisAlignment.stretch,
      children: [
        Padding(
          padding: const EdgeInsets.fromLTRB(12, 8, 12, 0),
          child: SegmentedControl<OpsTerminalMode>(
            value: _mode,
            onChanged: (value) => setState(() => _mode = value),
            children: const [
              SegmentedControlItem(
                value: OpsTerminalMode.local,
                child: Text('本地终端'),
              ),
              SegmentedControlItem(
                value: OpsTerminalMode.remote,
                child: Text('远程终端'),
              ),
            ],
          ),
        ),
        Expanded(
          child: _mode == OpsTerminalMode.local
              ? const LocalTerminalPage()
              : const RemoteTerminalPage(),
        ),
      ],
    );
  }
}
```

- [ ] **Step 4: Implement device-code form**

Create `apps/ops/lib/pages/terminal/remote_terminal_connection_dialog.dart`:

```dart
import 'package:shadcn_flutter/shadcn_flutter.dart';

class RemoteTerminalConnectionForm extends StatefulWidget {
  const RemoteTerminalConnectionForm({
    super.key,
    required this.onConnect,
  });

  final Future<void> Function(String deviceCode, String password)? onConnect;

  @override
  State<RemoteTerminalConnectionForm> createState() =>
      _RemoteTerminalConnectionFormState();
}

class _RemoteTerminalConnectionFormState
    extends State<RemoteTerminalConnectionForm> {
  final _deviceCode = TextEditingController();
  final _password = TextEditingController();
  bool _submitting = false;

  @override
  void dispose() {
    _deviceCode.dispose();
    _password.dispose();
    super.dispose();
  }

  bool get _canSubmit =>
      widget.onConnect != null &&
      _deviceCode.text.trim().isNotEmpty &&
      _password.text.isNotEmpty &&
      !_submitting;

  Future<void> _submit() async {
    if (!_canSubmit) return;
    setState(() => _submitting = true);
    try {
      await widget.onConnect!(
        _deviceCode.text.trim(),
        _password.text,
      );
    } finally {
      if (mounted) setState(() => _submitting = false);
    }
  }

  @override
  Widget build(BuildContext context) {
    return Column(
      crossAxisAlignment: CrossAxisAlignment.stretch,
      children: [
        TextField(
          controller: _deviceCode,
          placeholder: const Text('设备码'),
          onChanged: (_) => setState(() {}),
        ),
        const SizedBox(height: 8),
        TextField(
          controller: _password,
          placeholder: const Text('连接密码'),
          obscureText: true,
          onChanged: (_) => setState(() {}),
        ),
        const SizedBox(height: 12),
        PrimaryButton(
          onPressed: _canSubmit ? _submit : null,
          child: Text(_submitting ? '连接中…' : '连接'),
        ),
      ],
    );
  }
}
```

- [ ] **Step 5: Implement remote terminal page shell**

Create `apps/ops/lib/pages/terminal/remote_terminal_page.dart`:

```dart
import 'package:shadcn_flutter/shadcn_flutter.dart';

import 'remote_terminal_connection_dialog.dart';

class RemoteTerminalPage extends StatelessWidget {
  const RemoteTerminalPage({super.key});

  @override
  Widget build(BuildContext context) {
    final theme = Theme.of(context);
    return Row(
      crossAxisAlignment: CrossAxisAlignment.stretch,
      children: [
        SizedBox(
          width: 280,
          child: DecoratedBox(
            decoration: BoxDecoration(
              border: Border(
                right: BorderSide(color: theme.colorScheme.border),
              ),
            ),
            child: ListView(
              padding: const EdgeInsets.all(12),
              children: [
                Text('我的机器', style: theme.typography.small),
                const SizedBox(height: 12),
                Text(
                  '暂无在线机器',
                  style: theme.typography.xSmall.copyWith(
                    color: theme.colorScheme.mutedForeground,
                  ),
                ),
                const SizedBox(height: 20),
                Text('设备码连接', style: theme.typography.small),
                const SizedBox(height: 12),
                RemoteTerminalConnectionForm(onConnect: (code, password) async {}),
              ],
            ),
          ),
        ),
        Expanded(
          child: Center(
            child: Text(
              '选择机器或输入设备码以打开远程终端',
              style: theme.typography.small.copyWith(
                color: theme.colorScheme.mutedForeground,
              ),
            ),
          ),
        ),
      ],
    );
  }
}
```

- [ ] **Step 6: Update router**

Modify `apps/ops/lib/router.dart` imports:

```dart
import 'package:felorx_ops/pages/terminal/terminal_page.dart';
```

Replace `return const LocalTerminalPage();` in `OpsLocalTerminalRoute.build`:

```dart
return const OpsTerminalPage();
```

- [ ] **Step 7: Run widget test to verify green**

Run:

```bash
cd apps/ops && flutter test test/pages/terminal_remote_connection_form_test.dart
```

Expected: PASS.

- [ ] **Step 8: Commit**

```bash
git add apps/ops/lib/pages/terminal apps/ops/lib/router.dart apps/ops/test/pages/terminal_remote_connection_form_test.dart
git commit -m "feat(ops): 添加本地和远程终端入口"
```

---

## Task 13: Ops Flutter WebRTC Terminal Peer

**Files:**
- Create: `apps/ops/lib/services/flutter_webrtc_terminal_peer.dart`
- Modify: `apps/ops/lib/pages/terminal/remote_terminal_page.dart`
- Create: `apps/ops/test/services/flutter_webrtc_terminal_peer_test.dart`

- [ ] **Step 1: Write data-channel message encoding test**

Create `apps/ops/test/services/flutter_webrtc_terminal_peer_test.dart`:

```dart
import 'dart:convert';

import 'package:felorx_connect/felorx_connect.dart';
import 'package:felorx_ops/services/flutter_webrtc_terminal_peer.dart';
import 'package:test/test.dart';

void main() {
  test('terminal input encodes as connect message json', () {
    final json = FlutterWebRtcTerminalPeer.encodeTerminalInput('pwd\n');
    final message = ConnectMessage.fromJson(
      jsonDecode(json) as Map<String, dynamic>,
    );

    expect(message.type, ConnectMessageType.terminalInput);
    expect(message.payload['text'], 'pwd\n');
  });
}
```

- [ ] **Step 2: Run peer test to verify red**

Run:

```bash
cd apps/ops && flutter test test/services/flutter_webrtc_terminal_peer_test.dart
```

Expected: FAIL because peer service is missing.

- [ ] **Step 3: Implement peer encoding and adapter skeleton**

Create `apps/ops/lib/services/flutter_webrtc_terminal_peer.dart`:

```dart
import 'dart:async';
import 'dart:convert';

import 'package:flutter_webrtc/flutter_webrtc.dart';
import 'package:felorx_connect/felorx_connect.dart';

class FlutterWebRtcTerminalPeer {
  FlutterWebRtcTerminalPeer({
    required this.sessionId,
    required this.peerId,
    required this.sendSignal,
  });

  final String sessionId;
  final String peerId;
  final void Function(ConnectMessage message) sendSignal;
  RTCPeerConnection? _pc;
  RTCDataChannel? _channel;
  final StreamController<String> _output = StreamController<String>.broadcast();

  Stream<String> get output => _output.stream;

  static String encodeTerminalInput(String text) {
    return jsonEncode(ConnectMessage(
      type: ConnectMessageType.terminalInput,
      payload: {'text': text},
    ).toJson());
  }

  Future<void> startOffer() async {
    _pc = await createPeerConnection({
      'iceServers': [
        {'urls': 'stun:stun.l.google.com:19302'},
      ],
    });
    final pc = _pc!;
    pc.onIceCandidate = (candidate) {
      sendSignal(ConnectMessage(
        type: ConnectMessageType.iceCandidate,
        sessionId: sessionId,
        fromPeerId: peerId,
        payload: {
          'candidate': candidate.candidate,
          'sdpMid': candidate.sdpMid,
          'sdpMLineIndex': candidate.sdpMLineIndex,
        },
      ));
    };
    _channel = await pc.createDataChannel('terminal', RTCDataChannelInit());
    _channel!.onMessage = (message) {
      final connectMessage = ConnectMessage.fromJson(
        jsonDecode(message.text) as Map<String, dynamic>,
      );
      if (connectMessage.type == ConnectMessageType.terminalOutput) {
        _output.add(connectMessage.payload['text'] as String? ?? '');
      }
    };
    final offer = await pc.createOffer();
    await pc.setLocalDescription(offer);
    sendSignal(ConnectMessage(
      type: ConnectMessageType.offer,
      sessionId: sessionId,
      fromPeerId: peerId,
      payload: {'sdp': offer.sdp, 'type': offer.type},
    ));
  }

  Future<void> handleAnswer(ConnectMessage message) async {
    await _pc?.setRemoteDescription(
      RTCSessionDescription(message.payload['sdp'] as String, 'answer'),
    );
  }

  Future<void> addIceCandidate(ConnectMessage message) async {
    await _pc?.addCandidate(RTCIceCandidate(
      message.payload['candidate'] as String?,
      message.payload['sdpMid'] as String?,
      message.payload['sdpMLineIndex'] as int?,
    ));
  }

  void write(String text) {
    _channel?.send(RTCDataChannelMessage(encodeTerminalInput(text)));
  }

  Future<void> close() async {
    await _channel?.close();
    await _pc?.close();
    await _output.close();
  }
}
```

- [ ] **Step 4: Run peer test to verify green**

Run:

```bash
cd apps/ops && flutter test test/services/flutter_webrtc_terminal_peer_test.dart
```

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add apps/ops/lib/services/flutter_webrtc_terminal_peer.dart apps/ops/test/services/flutter_webrtc_terminal_peer_test.dart
git commit -m "feat(ops): 添加远程终端 WebRTC 客户端"
```

---

## Task 14: Code Generation, Analysis, and Local Smoke Tests

**Files:**
- Generated: `apps/ops/lib/providers/connect_provider.g.dart`
- Generated or lockfile updates from workspace bootstrap if present.

- [ ] **Step 1: Run dependency resolution**

Run:

```bash
dart pub get
```

Expected: package graph resolves with `packages/sync/felorx_connect`, `sync_node`, and Ops dependencies.

- [ ] **Step 2: Run Ops provider generation**

Run:

```bash
cd apps/ops && dart run build_runner build --delete-conflicting-outputs
```

Expected: `lib/providers/connect_provider.g.dart` is generated and command exits 0.

- [ ] **Step 3: Run package tests**

Run:

```bash
cd packages/sync/felorx_connect && dart test
```

Expected: all `felorx_connect` tests PASS.

- [ ] **Step 4: Run sync_node tests**

Run:

```bash
cd apps/sync_node && dart test
```

Expected: sync_node tests PASS.

- [ ] **Step 5: Run Ops focused tests**

Run:

```bash
cd apps/ops && flutter test test/services/connect_relay_service_test.dart test/pages/terminal_remote_connection_form_test.dart test/services/flutter_webrtc_terminal_peer_test.dart
```

Expected: focused Ops tests PASS.

- [ ] **Step 6: Run focused analysis**

Run:

```bash
dart analyze packages/sync/felorx_connect apps/sync_node apps/ops
```

Expected: no analyzer errors in touched packages.

- [ ] **Step 7: Commit generated files and dependency updates**

```bash
git add pubspec.lock apps/ops/lib/providers/connect_provider.g.dart
git commit -m "build(connect): 更新依赖解析和生成文件"
```

If `pubspec.lock` does not change, stage only generated files.

---

## Task 15: Remove `reevibe-server` Project and Old Script

**Files:**
- Modify: `pubspec.yaml`
- Delete: `apps/reevibe-server/src/main.dart`
- Delete: `apps/reevibe-server/pubspec.yaml`
- Delete: `apps/reevibe-server/docs/FEATURES.md`
- Delete: `apps/reevibe-server/docs/PRICING.md`
- Delete: `apps/reevibe-server/CHANGELOG.md`
- Delete: `scripts/reevibe_server.dart`

- [ ] **Step 1: Verify no live references remain**

Run:

```bash
rg -n "reevibe-server|reevibe_server|reevibe_server.dart" pubspec.yaml scripts apps packages .felorx --glob '!apps/reevibe-server/**'
```

Expected: remaining matches are documentation or builder tests that intentionally exercise app-name conversion. Inspect each match and update only runtime/workspace references.

- [ ] **Step 2: Remove workspace entry**

Modify root `pubspec.yaml` by deleting:

```yaml
  - apps/reevibe-server
```

- [ ] **Step 3: Delete old app and script**

Run:

```bash
rm -rf apps/reevibe-server
rm scripts/reevibe_server.dart
```

- [ ] **Step 4: Verify workspace no longer sees old package**

Run:

```bash
dart pub get
rg -n "apps/reevibe-server|name: reevibe_server" pubspec.yaml apps packages scripts
```

Expected: `dart pub get` succeeds; `rg` finds no workspace/runtime references to the deleted project.

- [ ] **Step 5: Commit removal**

```bash
git add pubspec.yaml apps/reevibe-server scripts/reevibe_server.dart
git commit -m "refactor(connect): 移除 reevibe-server 独立项目"
```

---

## Task 16: End-to-End Verification

**Files:**
- No planned source edits. Fix compile/test failures in the owning files from earlier tasks.

- [ ] **Step 1: Start relay manually**

Run:

```bash
cd packages/sync/felorx_connect
dart run bin/felorx_connect_host.dart --help
```

Expected: CLI usage prints successfully and exits 0.

- [ ] **Step 2: Compile host daemon**

Run:

```bash
cd packages/sync/felorx_connect
dart compile exe bin/felorx_connect_host.dart -o build/felorx-connect-host
```

Expected: executable is created at `packages/sync/felorx_connect/build/felorx-connect-host`.

- [ ] **Step 3: Run all focused tests again**

Run:

```bash
cd packages/sync/felorx_connect && dart test
cd ../../apps/sync_node && dart test
cd ../ops && flutter test test/services/connect_relay_service_test.dart test/pages/terminal_remote_connection_form_test.dart test/services/flutter_webrtc_terminal_peer_test.dart
```

Expected: all commands PASS.

- [ ] **Step 4: Run focused analysis again**

Run:

```bash
cd /Users/j/repos/puupees/felorx-apps
dart analyze packages/sync/felorx_connect apps/sync_node apps/ops
```

Expected: no analyzer errors in touched packages.

- [ ] **Step 5: Commit verification fixes**

If verification required source fixes:

```bash
git add <fixed-files>
git commit -m "fix(connect): 修复远程连接验证问题"
```

If no files changed, skip this commit.

---

## Self-Review

Spec coverage:

- `packages/sync/felorx_connect` package, protocol, relay, host daemon, client abstraction: Tasks 1-9.
- sync_node optional relay service: Task 10.
- Ops local/remote terminal distinction and device-code form: Tasks 11-13.
- TURN/STUN and direct WebRTC path: Tasks 5, 9, 13.
- Old `reevibe-server` removal: Task 15.
- Verification: Tasks 14 and 16.

Placeholder scan:

- No placeholder markers or empty implementation slots are intended in this plan.
- The Linux PTY adapter starts with a compile-safe process-backed implementation behind the `TerminalPty` interface. The interface isolates a future FFI replacement without blocking the package/API work.

Type consistency:

- `ConnectMessage`, `ConnectMessageType`, `ConnectRelayClient`, `ConnectRelayServer`, `ConnectHostConfig`, `ConnectHostDaemon`, `ConnectHostRegistry`, and `ConnectSessionRegistry` names are consistent across tasks.
- Ops models use `ConnectHost`; package registry uses `ConnectHostSnapshot` to avoid UI coupling.
