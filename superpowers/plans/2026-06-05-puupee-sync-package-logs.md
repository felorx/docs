# puupee_sync Package Logs Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Migrate `puupee_sync/lib` business logs from raw Talker usage to `PuupeeLogger` module keys and scene tags.

**Architecture:** Extend the existing `PuupeeLogger` infrastructure with package-wide sync module factories, then migrate files by subsystem. Keep message text and behavior stable while changing only the log transport, module key, scene, and named exception arguments.

**Tech Stack:** Dart, Talker, `puupee_utilities`, `puupee_sync`, `dart test`, `dart analyze`.

---

## File Structure

- Modify `packages/puupee_utilities/lib/talker.dart`: add module keys, scene constants, logger factories, and key registration.
- Modify `packages/puupee_utilities/test/talker_test.dart`: assert every sync module is registered and each factory writes the expected key.
- Create `packages/puupee_sync/test/puupee_sync_logging_test.dart`: package-level logger contract tests for representative modules.
- Modify `packages/puupee_sync/lib/src/manager_interface.dart`: expose `PuupeeLogger` instead of `Talker`.
- Modify node files: `sync_node_manager.dart`, `lan_sync_node_discoverer.dart`, `connection_quality_checker.dart`, `repos/sync_node_config_repo.dart`, `sync_node.dart`.
- Modify client files: `sync_client.dart`, `sync_client_manager.dart`, `api_client.dart`.
- Modify storage and transfer files: `storage/storage_manager.dart`, `storage/cloud_storage.dart`, `storage/lan_storage.dart`, `storage/local_storage.dart`, `file_transfer_manager.dart`, `cloud_transfer_impl.dart`, `transfer_manager.dart`.
- Modify auth, CRDT, content event, and MCP files: `auth_strategies.dart`, `pairing_auth.dart`, `pairing_service.dart`, `crdt.dart`, `postgres_crdt.dart`, `content_events.dart`, `mcp_server.dart`.

## Task 1: Extend PuupeeLogger Modules

**Files:**
- Modify: `packages/puupee_utilities/lib/talker.dart`
- Modify: `packages/puupee_utilities/test/talker_test.dart`

- [ ] **Step 1: Write failing tests for all module registrations and factories**

Add tests that expect these modules to be registered: `sync-server`, `sync-node`, `sync-client`, `sync-storage`, `sync-transfer`, `sync-auth`, `sync-crdt`, `sync-mcp`. Add factory tests that create a silent Talker backend and assert one log from each factory has the expected `key`.

- [ ] **Step 2: Run the tests and verify RED**

Run: `cd packages/puupee_utilities && dart test test/talker_test.dart`

Expected: FAIL because new module constants and factories do not exist yet.

- [ ] **Step 3: Implement module constants, factories, and key registration**

Add the missing `PuupeeLogModule` constants, `PuupeeLogger` factories, and include all module keys in `registerPuupeeTalkerKeys`.

- [ ] **Step 4: Run tests and analyzer**

Run:

```bash
cd packages/puupee_utilities
dart test test/talker_test.dart
dart analyze lib/talker.dart test/talker_test.dart
```

Expected: all tests pass and analyzer reports no issues.

- [ ] **Step 5: Commit**

```bash
git add packages/puupee_utilities/lib/talker.dart packages/puupee_utilities/test/talker_test.dart
git commit -m "feat(puupee_utilities): 注册同步包日志模块"
```

## Task 2: Add puupee_sync Logger Contract Tests

**Files:**
- Create: `packages/puupee_sync/test/puupee_sync_logging_test.dart`

- [ ] **Step 1: Write failing package logger tests**

Create tests that instantiate each representative logger factory with `createTalker(settings: TalkerSettings(useConsoleLogs: false))`, emit one log with a scene, and assert `history.single.key`, `history.single.message`, `history.single.exception`, and `history.single.stackTrace` where relevant.

- [ ] **Step 2: Run the tests and verify RED**

Run: `cd packages/puupee_sync && dart test test/puupee_sync_logging_test.dart`

Expected: FAIL until Task 1 factories exist in the checked-out code.

- [ ] **Step 3: Keep the tests focused on logger contracts**

Do not instantiate network services, databases, mDNS, or storage adapters in this test. This test documents module-to-key mapping and scene formatting only.

- [ ] **Step 4: Run tests**

Run: `cd packages/puupee_sync && dart test test/puupee_sync_logging_test.dart`

Expected: all tests pass after Task 1 is implemented.

- [ ] **Step 5: Commit**

```bash
git add packages/puupee_sync/test/puupee_sync_logging_test.dart
git commit -m "test(puupee_sync): 覆盖同步包日志模块"
```

## Task 3: Migrate Node and Client Logs

**Files:**
- Modify: `packages/puupee_sync/lib/src/manager_interface.dart`
- Modify: `packages/puupee_sync/lib/src/sync_node_manager.dart`
- Modify: `packages/puupee_sync/lib/src/lan_sync_node_discoverer.dart`
- Modify: `packages/puupee_sync/lib/src/connection_quality_checker.dart`
- Modify: `packages/puupee_sync/lib/src/repos/sync_node_config_repo.dart`
- Modify: `packages/puupee_sync/lib/src/sync_node.dart`
- Modify: `packages/puupee_sync/lib/src/api_client.dart`
- Modify: `packages/puupee_sync/lib/src/sync_client.dart`
- Modify: `packages/puupee_sync/lib/src/sync_client_manager.dart`

- [ ] **Step 1: Migrate logger types**

Replace `Talker` imports and getters with `PuupeeLogger`. Use `PuupeeLogger.syncNode()` for node files and `PuupeeLogger.syncClient()` for client files.

- [ ] **Step 2: Add scenes**

Use `PuupeeLogScene.lifecycle`, `discovery`, `config`, `network`, `http`, and `crdt` based on the message context. Every migrated logger call in these files must include `scene:`.

- [ ] **Step 3: Convert positional exceptions**

Convert calls like `logger.error('Sync failed', e, stackTrace)` to `logger.error('Sync failed', scene: ..., exception: e, stackTrace: stackTrace)`.

- [ ] **Step 4: Run targeted tests and analyzer**

Run:

```bash
cd packages/puupee_sync
dart test test/sync_node_manager_test.dart test/lan_discovery_test.dart test/lan_discovery_service_entry_test.dart test/puupee_sync_logging_test.dart
dart analyze lib/src/manager_interface.dart lib/src/sync_node_manager.dart lib/src/lan_sync_node_discoverer.dart lib/src/connection_quality_checker.dart lib/src/repos/sync_node_config_repo.dart lib/src/sync_node.dart lib/src/api_client.dart lib/src/sync_client.dart lib/src/sync_client_manager.dart test/puupee_sync_logging_test.dart
```

Expected: tests pass and analyzer reports no issues.

- [ ] **Step 5: Commit**

```bash
git add packages/puupee_sync/lib/src/manager_interface.dart packages/puupee_sync/lib/src/sync_node_manager.dart packages/puupee_sync/lib/src/lan_sync_node_discoverer.dart packages/puupee_sync/lib/src/connection_quality_checker.dart packages/puupee_sync/lib/src/repos/sync_node_config_repo.dart packages/puupee_sync/lib/src/sync_node.dart packages/puupee_sync/lib/src/api_client.dart packages/puupee_sync/lib/src/sync_client.dart packages/puupee_sync/lib/src/sync_client_manager.dart
git commit -m "refactor(puupee_sync): 迁移节点和客户端日志"
```

## Task 4: Migrate Storage and Transfer Logs

**Files:**
- Modify: `packages/puupee_sync/lib/src/storage/storage_manager.dart`
- Modify: `packages/puupee_sync/lib/src/storage/cloud_storage.dart`
- Modify: `packages/puupee_sync/lib/src/storage/lan_storage.dart`
- Modify: `packages/puupee_sync/lib/src/storage/local_storage.dart`
- Modify: `packages/puupee_sync/lib/src/file_transfer_manager.dart`
- Modify: `packages/puupee_sync/lib/src/cloud_transfer_impl.dart`
- Modify: `packages/puupee_sync/lib/src/transfer_manager.dart`

- [ ] **Step 1: Migrate storage loggers**

Use `PuupeeLogger.syncStorage()` in storage adapters, storage manager, file transfer manager, and cloud transfer implementation.

- [ ] **Step 2: Migrate transfer manager logger**

Use `PuupeeLogger.syncTransfer()` in `transfer_manager.dart`.

- [ ] **Step 3: Add scenes and named exception arguments**

Use `storage`, `network`, `transfer`, `lifecycle`, and `config` scenes. Convert all positional exception arguments to named `exception:` and `stackTrace:`.

- [ ] **Step 4: Run targeted tests and analyzer**

Run:

```bash
cd packages/puupee_sync
dart test test/storage_manager_test.dart test/transfer_manager_test.dart test/puupee_sync_logging_test.dart
dart analyze lib/src/storage/storage_manager.dart lib/src/storage/cloud_storage.dart lib/src/storage/lan_storage.dart lib/src/storage/local_storage.dart lib/src/file_transfer_manager.dart lib/src/cloud_transfer_impl.dart lib/src/transfer_manager.dart test/puupee_sync_logging_test.dart
```

Expected: tests pass and analyzer reports no issues.

- [ ] **Step 5: Commit**

```bash
git add packages/puupee_sync/lib/src/storage/storage_manager.dart packages/puupee_sync/lib/src/storage/cloud_storage.dart packages/puupee_sync/lib/src/storage/lan_storage.dart packages/puupee_sync/lib/src/storage/local_storage.dart packages/puupee_sync/lib/src/file_transfer_manager.dart packages/puupee_sync/lib/src/cloud_transfer_impl.dart packages/puupee_sync/lib/src/transfer_manager.dart
git commit -m "refactor(puupee_sync): 迁移存储和传输日志"
```

## Task 5: Migrate Auth, CRDT, Content Event, and MCP Logs

**Files:**
- Modify: `packages/puupee_sync/lib/src/auth_strategies.dart`
- Modify: `packages/puupee_sync/lib/src/pairing_auth.dart`
- Modify: `packages/puupee_sync/lib/src/pairing_service.dart`
- Modify: `packages/puupee_sync/lib/src/crdt.dart`
- Modify: `packages/puupee_sync/lib/src/postgres_crdt.dart`
- Modify: `packages/puupee_sync/lib/src/content_events.dart`
- Modify: `packages/puupee_sync/lib/src/mcp_server.dart`

- [ ] **Step 1: Migrate auth and pairing loggers**

Use `PuupeeLogger.syncAuth()` in auth and pairing files. Use `auth`, `pairing`, and `cache` scenes.

- [ ] **Step 2: Migrate CRDT and content event loggers**

Use `PuupeeLogger.syncCrdt()` in CRDT, Postgres CRDT, and content event files. Use `crdt`, `db`, and `content-event` scenes.

- [ ] **Step 3: Migrate MCP logger**

Use `PuupeeLogger.syncMcp()` in `mcp_server.dart`. Use `mcp`, `auth`, and `lifecycle` scenes.

- [ ] **Step 4: Run targeted tests and analyzer**

Run:

```bash
cd packages/puupee_sync
dart test test/auth_strategies_test.dart test/pairing_auth_test.dart test/pairing_token_validator_test.dart test/content_events_test.dart test/crdt_merge_test.dart test/puupee_sync_logging_test.dart
dart analyze lib/src/auth_strategies.dart lib/src/pairing_auth.dart lib/src/pairing_service.dart lib/src/crdt.dart lib/src/postgres_crdt.dart lib/src/content_events.dart lib/src/mcp_server.dart test/puupee_sync_logging_test.dart
```

Expected: tests pass and analyzer reports no issues.

- [ ] **Step 5: Commit**

```bash
git add packages/puupee_sync/lib/src/auth_strategies.dart packages/puupee_sync/lib/src/pairing_auth.dart packages/puupee_sync/lib/src/pairing_service.dart packages/puupee_sync/lib/src/crdt.dart packages/puupee_sync/lib/src/postgres_crdt.dart packages/puupee_sync/lib/src/content_events.dart packages/puupee_sync/lib/src/mcp_server.dart
git commit -m "refactor(puupee_sync): 迁移认证 CRDT 和 MCP 日志"
```

## Task 6: Final Audit and Verification

**Files:**
- Inspect: `packages/puupee_sync/lib`
- Inspect: `packages/puupee_utilities/lib/talker.dart`

- [ ] **Step 1: Search for raw Talker usage**

Run:

```bash
rg -n "package:talker/talker.dart|\\btalker\\.|\\bTalker\\b" packages/puupee_sync/lib packages/puupee_utilities/lib/talker.dart
```

Expected: raw Talker usage remains only in `packages/puupee_utilities/lib/talker.dart` and legitimate test/helper contexts outside `puupee_sync/lib`; `sync_server.dart` and other `puupee_sync/lib` files should not import `package:talker/talker.dart`.

- [ ] **Step 2: Check every migrated logger call has a scene**

Run a script that scans `packages/puupee_sync/lib` for `logger.info/debug/warning/error/critical/verbose` and `_logger.info/debug/warning/error/critical/verbose` calls without `scene:`.

Expected: zero missing scene calls in `puupee_sync/lib`.

- [ ] **Step 3: Run final tests and analyzer**

Run:

```bash
cd packages/puupee_utilities
dart test test/talker_test.dart
dart analyze lib/talker.dart test/talker_test.dart

cd ../puupee_sync
dart test test/puupee_sync_logging_test.dart test/sync_server_logging_test.dart test/sync_node_manager_test.dart test/lan_discovery_test.dart test/lan_discovery_service_entry_test.dart test/storage_manager_test.dart test/transfer_manager_test.dart test/auth_strategies_test.dart test/pairing_auth_test.dart test/pairing_token_validator_test.dart test/content_events_test.dart test/crdt_merge_test.dart
dart analyze lib test/puupee_sync_logging_test.dart test/sync_server_logging_test.dart
```

Expected: all selected tests pass and analyzer reports no issues.

- [ ] **Step 4: Commit any final fixes**

If the audit reveals missed files or formatting-only changes, fix and commit them with:

```bash
git add packages/puupee_sync packages/puupee_utilities
git commit -m "fix(puupee_sync): 补齐同步包日志迁移"
```
