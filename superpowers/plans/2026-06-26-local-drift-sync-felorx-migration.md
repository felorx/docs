# 本地 Drift 与同步协议 Felorx 数据库迁移实施方案

> 给执行代理：实现本方案时必须使用 `superpowers:subagent-driven-development`（推荐）或 `superpowers:executing-plans` 逐项执行。任务使用 `- [ ]` 复选框记录进度。

**目标：** 在服务端 EF Core 数据库迁移完成之后，将本地 Drift / sync-node 存储中的旧品牌物理命名迁移到 Felorx 命名，并让现有客户端数据库、sync changeset、老路由和老协议 key 可以无缝过渡。

**核心架构：** 以 v17 作为本地 Drift schema 边界。v17 之后内部物理存储和 CRDT 白名单统一使用 `felorxes`、`user_felorxes`、`tag_felorxes`、`felorx_id`；旧库升级时迁移数据并创建旧名兼容视图；同步协议新增显式 v2 输出 Felorx key，同时继续接收并默认输出旧 key，直到所有旧客户端升级完成。

**技术栈：** Dart、Flutter、Drift、SQLite、本地 sync server、`sql_crdt`、`crdt_sync`、生成的 `felorx_sync_api` 客户端。

---

## 范围与非目标

**本方案范围：**

- `packages/core/felorx` 的 Drift 表定义、schemaVersion、迁移、快照、数据库诊断工具。
- `packages/sync/felorx_sync` 的 CRDT 表白名单、sync server SQL、MCP server SQL、sync client 初始同步/全量同步协议。
- `packages/api/felorx_sync_api` 的 changeset 响应模型与生成脚本。
- sync-node SQLite 数据库的自动 v17 迁移。
- sync-node PostgreSQL 数据库的独立 SQL 迁移脚本或启动前迁移入口。
- 完整测试：旧 v16 SQLite 数据升级、新库创建、兼容视图读写、sync 协议 v1/v2、MCP 查询、同步客户端请求头、生成 API 客户端。

**本方案非目标：**

- `.puupee` / `.puupee.yaml` builder 配置目录兼容。
- `puupee-cos://`、`.puupeenote`、历史导入导出扩展名。
- `repos/puupees/puupee-apps` 本机路径兜底。
- 文档里描述历史实现的旧名。
- 已完成的 `api` ABP/EF Core 服务端主数据库迁移。

这些非目标需要后续独立清理，不能混入本地数据库迁移提交。

## 当前状态摘要

- Drift Dart 类型已经语义化为 `Felorx`：`Felorxes`、`UserFelorxes`、`TagFelorxes`。
- Drift 物理表名仍是旧名：
  - `Felorxes.tableName => 'puupees'`
  - `UserFelorxes.tableName => 'user_puupees'`
  - `TagFelorxes.tableName => 'tag_puupees'`
- Drift 物理列仍有旧名：
  - `user_puupees.puupee_id`
  - `tag_puupees.puupee_id`
  - `transfer_tasks.puupee_id`
- `FelorxDatabase` 暴露了兼容 getter/typedef：
  - `puupees => felorxes`
  - `userPuupees => userFelorxes`
  - `tagPuupees => tagFelorxes`
  - `typedef Puupee = Felorx`
- `FelorxSqlCrdt.getTables()` 仍返回旧 CRDT 表名：`puupees`、`user_puupees`、`tags`、`tag_puupees`。
- `sync_server.dart` 已有新路由 `/sync/felorxes`，但 SQL、changeset key 和默认响应仍以旧表名为内部存储名。
- `sync_server.dart` 已能把入站 `felorxes` / `user_felorxes` / `tag_felorxes` 映射成旧 key，说明协议兼容层已经有雏形。
- 生成的 `felorx_sync_api` 响应模型仍用旧 JSON key：
  - `GetChangeset200Response.felorxes` 序列化为 `puupees`
  - `GetChangeset200Response.userFelorxes` 序列化为 `user_puupees`
  - `LoginInit200Response.felorxes` 序列化为 `puupees`

## 迁移命名矩阵

| 类型 | 旧名 | 新名 | 处理方式 |
| --- | --- | --- | --- |
| 主内容表 | `puupees` | `felorxes` | v17 迁移为真实表，旧名保留兼容视图 |
| 用户关联表 | `user_puupees` | `user_felorxes` | v17 迁移为真实表，`puupee_id` 改为 `felorx_id`，旧名保留兼容视图 |
| 标签关联表 | `tag_puupees` | `tag_felorxes` | v17 迁移为真实表，`puupee_id` 改为 `felorx_id`，旧名保留兼容视图 |
| 传输任务列 | `transfer_tasks.puupee_id` | `transfer_tasks.felorx_id` | v17 重建表并迁移数据，兼容 getter 只保留在模型扩展中 |
| 传输任务索引 | `idx_transfer_tasks_puupee_id` | `idx_transfer_tasks_felorx_id` | v17 删除旧索引并创建新索引 |
| changeset key | `puupees` | `felorxes` | 内部统一新名；协议 v1 输出旧名，协议 v2 输出新名 |
| changeset key | `user_puupees` | `user_felorxes` | 内部统一新名；协议 v1 输出旧名，协议 v2 输出新名 |
| changeset key | `tag_puupees` | `tag_felorxes` | 内部统一新名；协议 v1 输出旧名，协议 v2 输出新名 |
| changeset 字段 | `puupee_id` | `felorx_id` | 内部统一新名；协议 v1 输出旧字段，协议 v2 输出新字段 |

---

## Task 1：先写本地数据库迁移测试

**文件：**

- 新建：`packages/core/felorx/test/local_brand_migration_test.dart`
- 可复用：`packages/core/felorx/test/migrations_test.dart`
- 可复用：`packages/core/felorx/lib/src/migrations/snapshots/snapshot_v16.dart`

- [ ] 新增 `local_brand_migration_test.dart`，用 `NativeDatabase.memory()` 创建空数据库。

- [ ] 测试 `MigrationV17` 从 v16 旧表升级到新表。测试步骤固定为：
  1. 执行 `SnapshotV16().sqliteStatements` 创建 v16 结构。
  2. 插入一条 `puupees` 数据，至少包含 `id`、`content_type`、`creator_id`、`title`。
  3. 插入一条 `user_puupees` 数据，至少包含 `user_id`、`puupee_id`、`created_at`。
  4. 插入一条 `tags` 数据，至少包含 `id`、`name`、`creator_id`、`created_at`。
  5. 插入一条 `tag_puupees` 数据，至少包含 `tag_id`、`puupee_id`、`created_at`。
  6. 插入一条 `transfer_tasks` 数据，至少包含 `puupee_id`、`type`、`status`、`creator_id`、`created_at`。
  7. 执行 `MigrationV17().upgrade(Migrator(database), database, false)`。
  8. 断言真实表存在：`felorxes`、`user_felorxes`、`tag_felorxes`、`transfer_tasks`。
  9. 断言旧真实表不存在：`sqlite_master WHERE type='table' AND name IN ('puupees','user_puupees','tag_puupees')` 返回空。
  10. 断言旧兼容视图存在：`sqlite_master WHERE type='view' AND name IN ('puupees','user_puupees','tag_puupees')` 返回 3 条。
  11. 断言 `user_felorxes`、`tag_felorxes`、`transfer_tasks` 有 `felorx_id` 列且没有 `puupee_id` 列。
  12. 断言迁移前插入的数据仍能从新表读取。
  13. 断言旧视图能读取同一批数据，并将 `felorx_id` 映射为 `puupee_id`。

- [ ] 测试旧名兼容视图的写入能力：
  1. `INSERT INTO puupees (id, content_type, creator_id, title) VALUES (...)` 后，`felorxes` 能读到同一行。
  2. `UPDATE puupees SET title = ... WHERE id = ...` 后，`felorxes.title` 更新。
  3. `DELETE FROM puupees WHERE id = ...` 后，`felorxes` 对应行被物理删除。
  4. `INSERT INTO user_puupees (user_id, puupee_id, created_at) VALUES (...)` 后，`user_felorxes.felorx_id` 正确。
  5. `UPDATE user_puupees SET position = 9 WHERE user_id = ... AND puupee_id = ...` 后，新表同步。
  6. `DELETE FROM user_puupees WHERE user_id = ... AND puupee_id = ...` 后，新表删除。
  7. `INSERT INTO tag_puupees (tag_id, puupee_id, created_at) VALUES (...)` 后，`tag_felorxes.felorx_id` 正确。

- [ ] 测试新库创建：
  1. 把 `dbVersion` 升到 17 后创建 `FelorxDatabase(NativeDatabase.memory(), isServer: false)`。
  2. 第一次查询触发 onCreate。
  3. 断言真实表为新名。
  4. 断言旧名兼容视图存在。
  5. 断言 `SnapshotComparator.compare(database, 17, false).isMatch == true`。

- [ ] 测试 `DatabaseVersionManager.validateDatabaseIntegrity()` 支持新真实表，并在兼容期接受旧视图存在。

- [ ] 测试 `DatabaseVersionManager.getDatabaseStats()` 默认返回新 key：
  - `felorxes_count`
  - `user_felorxes_count`
  - `tag_felorxes_count`

- [ ] 测试 `getDatabaseStats()` 在兼容期继续返回旧 key：
  - `puupees_count`
  - `user_puupees_count`
  - `tag_puupees_count`

- [ ] 先运行测试，确认在未实现 v17 时失败：

```bash
cd packages/core/felorx
flutter test test/local_brand_migration_test.dart
```

**预期失败：** `MigrationV17`、`SnapshotV17` 或新表名尚不存在。

---

## Task 2：增加本地品牌存储迁移工具

**文件：**

- 新建：`packages/core/felorx/lib/src/migrations/branding/local_brand_database_names.dart`
- 新建：`packages/core/felorx/lib/src/migrations/branding/local_brand_compatibility_views.dart`
- 新建：`packages/core/felorx/test/local_brand_compatibility_views_test.dart`

- [ ] 新增 `LocalBrandDatabaseNames`，集中维护物理命名映射：

```dart
class LocalBrandDatabaseNames {
  static const felorxes = 'felorxes';
  static const legacyPuupees = 'puupees';

  static const userFelorxes = 'user_felorxes';
  static const legacyUserPuupees = 'user_puupees';

  static const tagFelorxes = 'tag_felorxes';
  static const legacyTagPuupees = 'tag_puupees';

  static const felorxId = 'felorx_id';
  static const legacyPuupeeId = 'puupee_id';
}
```

- [ ] 新增 `LocalBrandCompatibilityViews.createSqliteViews(FelorxDatabase database)`，只在 SQLite 客户端执行。

- [ ] `createSqliteViews` 必须先删除旧视图和旧 trigger：
  - `DROP VIEW IF EXISTS puupees`
  - `DROP VIEW IF EXISTS user_puupees`
  - `DROP VIEW IF EXISTS tag_puupees`
  - `DROP TRIGGER IF EXISTS puupees_legacy_insert`
  - `DROP TRIGGER IF EXISTS puupees_legacy_update`
  - `DROP TRIGGER IF EXISTS puupees_legacy_delete`
  - `DROP TRIGGER IF EXISTS user_puupees_legacy_insert`
  - `DROP TRIGGER IF EXISTS user_puupees_legacy_update`
  - `DROP TRIGGER IF EXISTS user_puupees_legacy_delete`
  - `DROP TRIGGER IF EXISTS tag_puupees_legacy_insert`
  - `DROP TRIGGER IF EXISTS tag_puupees_legacy_update`
  - `DROP TRIGGER IF EXISTS tag_puupees_legacy_delete`

- [ ] `createSqliteViews` 创建旧名视图：

```sql
CREATE VIEW puupees AS SELECT * FROM felorxes;

CREATE VIEW user_puupees AS
SELECT
  user_id,
  felorx_id AS puupee_id,
  position,
  created_at,
  deleted_at,
  is_favorite,
  readable,
  writable,
  deletable,
  creatable,
  node_id,
  hlc,
  modified
FROM user_felorxes;

CREATE VIEW tag_puupees AS
SELECT
  tag_id,
  felorx_id AS puupee_id,
  created_at,
  deleted_at,
  node_id,
  hlc,
  modified
FROM tag_felorxes;
```

- [ ] 为三张旧名视图创建 `INSTEAD OF INSERT`、`INSTEAD OF UPDATE`、`INSTEAD OF DELETE` trigger。

- [ ] 对 `puupees` 视图，trigger 通过 `PRAGMA table_info('felorxes')` 动态读取列清单生成 SQL，避免手写 80 多个字段。生成规则：
  - INSERT：`INSERT INTO felorxes (<all columns>) VALUES (NEW.<column>...)`
  - UPDATE：`UPDATE felorxes SET <column> = NEW.<column> ... WHERE id = OLD.id`
  - DELETE：`DELETE FROM felorxes WHERE id = OLD.id`

- [ ] 对 `user_puupees` 视图，trigger 显式映射 `puupee_id` 与 `felorx_id`：

```sql
INSERT INTO user_felorxes (
  user_id, felorx_id, position, created_at, deleted_at,
  is_favorite, readable, writable, deletable, creatable,
  node_id, hlc, modified
) VALUES (
  NEW.user_id, NEW.puupee_id, COALESCE(NEW.position, 0), NEW.created_at, NEW.deleted_at,
  COALESCE(NEW.is_favorite, 0), COALESCE(NEW.readable, 1), COALESCE(NEW.writable, 1),
  COALESCE(NEW.deletable, 1), COALESCE(NEW.creatable, 1),
  NEW.node_id, NEW.hlc, NEW.modified
);
```

- [ ] 对 `tag_puupees` 视图，trigger 显式映射 `puupee_id` 与 `felorx_id`：

```sql
INSERT INTO tag_felorxes (
  tag_id, felorx_id, created_at, deleted_at, node_id, hlc, modified
) VALUES (
  NEW.tag_id, NEW.puupee_id, NEW.created_at, NEW.deleted_at, NEW.node_id, NEW.hlc, NEW.modified
);
```

- [ ] 测试 `LocalBrandCompatibilityViews.createSqliteViews` 可以重复执行两次，第二次不报错且视图/trigger 仍可用。

---

## Task 3：实现 Drift v17 物理迁移

**文件：**

- 修改：`packages/core/felorx/lib/src/version.dart`
- 修改：`packages/core/felorx/lib/src/tables/felorx.dart`
- 修改：`packages/core/felorx/lib/src/tables/user_felorx.dart`
- 修改：`packages/core/felorx/lib/src/tables/tag.dart`
- 修改：`packages/core/felorx/lib/src/tables/transfer_task.dart`
- 新建：`packages/core/felorx/lib/src/migrations/migration_v17.dart`
- 修改：`packages/core/felorx/lib/src/migrations/migration_registry.dart`
- 新建：`packages/core/felorx/lib/src/migrations/snapshots/snapshot_v17.dart`
- 修改：`packages/core/felorx/lib/src/migrations/snapshots/snapshot_registry.dart`
- 修改：`packages/core/felorx/lib/src/migration_strategy.dart`
- 修改：`packages/core/felorx/lib/src/database_version.dart`
- 生成：`packages/core/felorx/lib/src/database.g.dart`

- [ ] 将 `dbVersion` 从 16 改为 17。

- [ ] 将 Drift 表定义改为新物理名：

```dart
class Felorxes extends Table {
  @override
  String get tableName => 'felorxes';
}

class UserFelorxes extends Table {
  @override
  String get tableName => 'user_felorxes';

  TextColumn get felorxId => text().named('felorx_id').withLength(max: 64)();

  @override
  Set<Column> get primaryKey => {userId, felorxId};
}

class TagFelorxes extends Table {
  @override
  String get tableName => 'tag_felorxes';

  TextColumn get felorxId => text().named('felorx_id').withLength(max: 64)();

  @override
  Set<Column> get primaryKey => {tagId, felorxId};
}

class TransferTasks extends Table {
  TextColumn get felorxId => text().named('felorx_id').withLength(max: 64)();

  @override
  Set<Column> get primaryKey => {felorxId, type, storageId, creatorId};
}
```

- [ ] 保留兼容 getter/typedef，但明确它们只兼容 Dart API，不再代表物理表名：

```dart
$FelorxesTable get puupees => felorxes;
$UserFelorxesTable get userPuupees => userFelorxes;
$TagFelorxesTable get tagPuupees => tagFelorxes;
```

- [ ] 在生成的数据类上补扩展 getter，给未迁移完的调用方一个短期兼容层：

```dart
extension UserFelorxLegacyCompat on UserFelorx {
  String get puupeeId => felorxId;
}

extension TagFelorxLegacyCompat on TagFelorx {
  String get puupeeId => felorxId;
}

extension TransferTaskLegacyCompat on TransferTask {
  String get puupeeId => felorxId;
}
```

- [ ] `MigrationV17.upgrade` 对 SQLite 执行：
  1. `DROP VIEW IF EXISTS puupees`
  2. `DROP VIEW IF EXISTS user_puupees`
  3. `DROP VIEW IF EXISTS tag_puupees`
  4. 如果 `puupees` 表存在且 `felorxes` 表不存在，执行 `ALTER TABLE puupees RENAME TO felorxes`
  5. 如果 `user_puupees` 表存在，重建为 `user_felorxes`，把 `puupee_id` 复制到 `felorx_id`
  6. 如果 `tag_puupees` 表存在，重建为 `tag_felorxes`，把 `puupee_id` 复制到 `felorx_id`
  7. 如果 `transfer_tasks` 有 `puupee_id` 列，重建 `transfer_tasks`，把 `puupee_id` 复制到 `felorx_id`
  8. `DROP INDEX IF EXISTS idx_transfer_tasks_puupee_id`
  9. `CREATE INDEX IF NOT EXISTS idx_transfer_tasks_felorx_id ON transfer_tasks(felorx_id)`
  10. 保留现有 `idx_transfer_tasks_type`、`idx_transfer_tasks_status`、`idx_transfer_tasks_storage_id`、`idx_transfer_tasks_storage_type`
  11. 调用 `LocalBrandCompatibilityViews.createSqliteViews(database)`

- [ ] `MigrationV17.upgrade` 必须幂等：
  - 已经有新表时不得覆盖数据。
  - 新旧表同时存在且旧表还有数据时必须抛出 `StateError`，禁止猜测合并。
  - 新旧表同时存在但旧表为空时删除旧表再创建旧名视图。

- [ ] 更新 `DatabaseMigrationStrategy._onCreate`：
  - snapshot 创建完真实表后调用 `LocalBrandCompatibilityViews.createSqliteViews(database)`。
  - 仅 `!isServer` 时执行 SQLite 兼容视图逻辑。

- [ ] 新建 `SnapshotV17`：
  - `sqliteTableNames` 使用 `felorxes`、`user_felorxes`、`tag_felorxes`。
  - `sqliteStatements` 中真实表 SQL 使用 `felorx_id`。
  - 不把兼容视图写进 `sqliteStatements`，避免 `SnapshotComparator` 把视图当表结构差异。

- [ ] 更新 snapshot registry：
  - import `snapshot_v17.dart`
  - 注册 `17: () => SnapshotV17()`

- [ ] 更新 migration registry：
  - import `migration_v17.dart`
  - 在 `MigrationRegistry.getMigrations()` 末尾加入 `MigrationV17()`

- [ ] 更新 `DatabaseVersionManager.validateDatabaseIntegrity()`：
  - 必需真实表改为 `felorxes`、`user_felorxes`、`tags`、`tag_felorxes`。
  - 兼容期额外检查旧名视图存在，但视图缺失只能 warning，不能阻断新库使用。

- [ ] 更新 `getDatabaseStats()`：
  - 新 key 读取新真实表。
  - 旧 key 使用相同计数回填，保持诊断 JSON 向后兼容。

- [ ] 运行生成：

```bash
cd packages/core/felorx
dart run build_runner build --delete-conflicting-outputs
```

- [ ] 运行核心测试：

```bash
cd packages/core/felorx
flutter test test/local_brand_migration_test.dart
flutter test test/migrations_test.dart test/migration_strategy_test.dart test/database_version_test.dart test/database_utils_test.dart
dart analyze
```

- [ ] 第一次提交：

```bash
git add packages/core/felorx
git commit -m "feat(felorx): 迁移本地数据库物理命名"
```

---

## Task 4：将内部 CRDT 与 sync server SQL 切换到新物理名

**文件：**

- 修改：`packages/sync/felorx_sync/lib/src/crdt.dart`
- 修改：`packages/sync/felorx_sync/lib/src/sync_server.dart`
- 修改：`packages/sync/felorx_sync/lib/src/mcp_server.dart`
- 修改：`packages/sync/felorx_sync/lib/src/sync_client.dart`
- 修改：`packages/sync/felorx_sync/lib/src/transfer_manager.dart`
- 修改：`packages/sync/felorx_sync/lib/src/repos/transfer_task_repo.dart`
- 按需修改：`packages/sync/felorx_sync/lib/src/repos/**`
- 按需修改：`packages/sync/felorx_sync/test/**`

- [ ] `FelorxSqlCrdt.getTables()` 改为：

```dart
Future<Iterable<String>> getTables() async =>
    Future.value(['felorxes', 'user_felorxes', 'tags', 'tag_felorxes']);
```

- [ ] `FelorxSqlCrdt.getTableKeys()` 改为：

```dart
case 'felorxes':
  return ['id'];
case 'user_felorxes':
  return ['user_id', 'felorx_id'];
case 'tags':
  return ['id'];
case 'tag_felorxes':
  return ['tag_id', 'felorx_id'];
```

- [ ] `FelorxDriftCrdt.put()` 的 `onDatasetChanged` 改为 `['felorxes']`。

- [ ] 保留入站 legacy changeset 支持。把现有 `_normalizeFelorxChangesetPayload` 改成“旧名到新名”：
  - `puupees` -> `felorxes`
  - `user_puupees` -> `user_felorxes`
  - `tag_puupees` -> `tag_felorxes`
  - record 中 `puupee_id` / `puupeeId` -> `felorx_id`

- [ ] `sync_server.dart` 所有内部 SQL 改用新物理名：
  - `puupees` -> `felorxes`
  - `user_puupees` -> `user_felorxes`
  - `tag_puupees` -> `tag_felorxes`
  - `puupee_id` -> `felorx_id`

- [ ] `_queries()` 返回新内部 key：
  - `felorxes`
  - `user_felorxes`
  - `tags`
  - `tag_felorxes`

- [ ] `_initChangesetTables` 和 `_initChangesetTableKeys` 改为新 key。

- [ ] `_normalizeFelorxOrderBy()` 改成 legacy 输入兼容：

```dart
return orderBy
    .replaceAll(RegExp(r'\bpuupees\.'), 'felorxes.')
    .replaceAll(RegExp(r'\buser_puupees\.'), 'user_felorxes.')
    .replaceAll(RegExp(r'\btag_puupees\.'), 'tag_felorxes.');
```

- [ ] `DbCreator.create()` 中初始化字段清单改查新表：
  - SQLite：`pragma_table_info('felorxes')`
  - PostgreSQL：`information_schema.columns WHERE table_name = 'felorxes'`

- [ ] `McpServer` 内部 SQL 全部切到新名，但 orderBy 继续接受旧表前缀并归一化到新表前缀。

- [ ] 所有 `TransferTasksCompanion(puupeeId: ...)` 改为 `TransferTasksCompanion(felorxId: ...)`。

- [ ] 所有 `db.transferTasks.puupeeId` 改为 `db.transferTasks.felorxId`。

- [ ] 所有 `UserFelorxesCompanion(puupeeId: ...)` 改为 `UserFelorxesCompanion(felorxId: ...)`。

- [ ] 所有 `TagFelorxesCompanion(puupeeId: ...)` 改为 `TagFelorxesCompanion(felorxId: ...)`。

- [ ] 运行：

```bash
cd packages/sync/felorx_sync
flutter test
dart analyze
```

- [ ] 第二次提交：

```bash
git add packages/sync/felorx_sync packages/core/felorx
git commit -m "feat(sync): 切换 CRDT 存储到 Felorx 命名"
```

---

## Task 5：增加 sync changeset 协议 v2，并保持 v1 默认兼容

**文件：**

- 修改：`packages/sync/felorx_sync/lib/src/sync_server.dart`
- 修改：`packages/sync/felorx_sync/lib/src/sync_client.dart`
- 修改：`packages/sync/felorx_sync/lib/src/api_client.dart`
- 修改：`packages/sync/felorx_sync/test/sync_server_route_naming_test.dart`
- 新建：`packages/sync/felorx_sync/test/sync_protocol_version_test.dart`

- [ ] 新增协议版本常量：

```dart
const felorxSyncProtocolHeader = 'x-felorx-sync-protocol';
const felorxSyncProtocolV2 = '2';
```

- [ ] 新增请求判断：

```dart
bool _wantsFelorxProtocolV2(Request request) {
  return request.headers[felorxSyncProtocolHeader] == felorxSyncProtocolV2 ||
      request.queryParameters['syncProtocol'] == '2' ||
      request.queryParameters['syncProtocol'] == 'felorx-v2';
}
```

- [ ] 新增出站 changeset 映射：

```dart
Map<String, dynamic> _serializeFelorxChangesetForRequest(
  Request request,
  Map<String, dynamic> changeset,
) {
  if (_wantsFelorxProtocolV2(request)) return changeset;
  return _toLegacyFelorxChangeset(changeset);
}
```

- [ ] `_toLegacyFelorxChangeset` 规则：
  - key `felorxes` 输出为 `puupees`
  - key `user_felorxes` 输出为 `user_puupees`
  - key `tag_felorxes` 输出为 `tag_puupees`
  - record 字段 `felorx_id` 输出为 `puupee_id`
  - 其他 key 和字段保持不变

- [ ] `_loginInit()` 非分页响应使用 `_serializeFelorxChangesetForRequest`。

- [ ] `_loginInit()` 分页响应中的 `changeset` 使用 `_serializeFelorxChangesetForRequest`，`nextCursor` 和 `hasMore` 不变。

- [ ] `_getChangeset()` 响应使用 `_serializeFelorxChangesetForRequest`。

- [ ] WebSocket changeset 如果仍由 `crdt_sync` 直接处理，先保持内部新名；若现有旧客户端仍走 WebSocket，需要在 `CrdtSyncClient` 层加入 mapIncoming/mapOutgoing，或者保留旧名兼容视图直到 WebSocket 客户端全部升级。

- [ ] `ApiClient.send()` 默认给本仓库新客户端加请求头：

```dart
felorxSyncProtocolHeader: felorxSyncProtocolV2,
```

- [ ] `FelorxSyncClient.validateRecord` 中的表名判断改为 `user_felorxes`。

- [ ] 新增测试：
  - 无 header 请求 `/sync/changeset` 返回旧 key `puupees`、`user_puupees`。
  - header `x-felorx-sync-protocol: 2` 请求 `/sync/changeset` 返回新 key `felorxes`、`user_felorxes`。
  - `POST /sync/felorxes` 接收新 key 与 `felorx_id`。
  - `POST /sync/felorxes` 接收旧 key 与 `puupee_id`。
  - `GET /sync/changeset/init?limit=...` 分页响应在 v1/v2 下 key 正确。

- [ ] 第三次提交：

```bash
git add packages/sync/felorx_sync
git commit -m "feat(sync): 增加 Felorx changeset 协议 v2"
```

---

## Task 6：更新生成的 sync API 客户端与 Swagger

**文件：**

- 修改：`packages/sync/felorx_sync/scripts/generate_swagger.dart`
- 修改：`packages/sync/felorx_sync/scripts/generate_api_client.dart`
- 修改：`packages/sync/felorx_sync/swagger.json`
- 修改：`packages/api/felorx_sync_api/lib/src/model/get_changeset200_response.dart`
- 修改：`packages/api/felorx_sync_api/lib/src/model/get_changeset200_response.g.dart`
- 修改：`packages/api/felorx_sync_api/lib/src/model/login_init200_response.dart`
- 修改：`packages/api/felorx_sync_api/lib/src/model/login_init200_response.g.dart`
- 修改：`packages/api/felorx_sync_api/doc/GetChangeset200Response.md`
- 修改：`packages/api/felorx_sync_api/doc/LoginInit200Response.md`
- 修改：`packages/api/felorx_sync_api/test/felorx_semantic_alias_test.dart`

- [ ] Swagger 为 `/sync/changeset` 和 `/sync/changeset/init` 增加 header 参数：
  - name: `x-felorx-sync-protocol`
  - in: `header`
  - required: `false`
  - schema enum: `["1", "2"]`
  - description: `2 returns Felorx-named changeset keys; omitted keeps legacy Puupee keys.`

- [ ] Swagger v2 schema 使用新 key：
  - `felorxes`
  - `user_felorxes`
  - `tag_felorxes`

- [ ] 文档明确：默认响应仍是 legacy key；新客户端应发送 header 获取 v2。

- [ ] 生成脚本中的旧替换逻辑改为：
  - Dart 属性名仍为 `felorxes`、`userFelorxes`、`tagFelorxes`
  - `@JsonKey` 对 v2 模型使用 `felorxes`、`user_felorxes`、`tag_felorxes`
  - 如果需要兼容 v1 响应，新增手写 `fromJson` fallback，而不是只靠单一 `@JsonKey`

- [ ] `GetChangeset200Response.fromJson` 支持：
  - 优先读 `felorxes`
  - 缺失时读 `puupees`
  - 优先读 `user_felorxes`
  - 缺失时读 `user_puupees`
  - 优先读 `tag_felorxes`
  - 缺失时读 `tag_puupees`

- [ ] `toJson()` 默认输出新 key；如果需要输出 v1，调用 sync server 的兼容映射，不让生成模型承担协议降级。

- [ ] 更新测试：
  - 旧 JSON key 可以反序列化。
  - 新 JSON key 可以反序列化。
  - `toJson()` 输出新 JSON key。

- [ ] 运行：

```bash
cd packages/api/felorx_sync_api
dart test
dart analyze
```

- [ ] 第四次提交：

```bash
git add packages/sync/felorx_sync packages/api/felorx_sync_api
git commit -m "feat(sync-api): 支持 Felorx changeset key"
```

---

## Task 7：sync-node PostgreSQL 独立迁移脚本

**文件：**

- 新建：`packages/sync/felorx_sync/lib/src/branding/sync_node_postgres_brand_migration_sql.dart`
- 新建：`packages/sync/felorx_sync/test/sync_node_postgres_brand_migration_sql_test.dart`
- 新建：`packages/sync/felorx_sync/docs/sync-node-database-felorx-migration-runbook.md`

- [ ] 新增 SQL 生成器，覆盖 sync-node PostgreSQL 存储表：
  - `puupees` -> `felorxes`
  - `user_puupees` -> `user_felorxes`
  - `tag_puupees` -> `tag_felorxes`
  - `user_felorxes.puupee_id` -> `felorx_id`
  - `tag_felorxes.puupee_id` -> `felorx_id`
  - `transfer_tasks.puupee_id` -> `felorx_id`
  - `idx_transfer_tasks_puupee_id` -> `idx_transfer_tasks_felorx_id`

- [ ] PostgreSQL SQL 必须在事务中运行，并在迁移前锁表：

```sql
BEGIN;
LOCK TABLE puupees, user_puupees, tag_puupees IN ACCESS EXCLUSIVE MODE;
```

- [ ] PostgreSQL SQL 必须在新旧真实表同时存在且旧表非空时 fail-fast：

```sql
DO $$
BEGIN
  IF to_regclass('public.puupees') IS NOT NULL
     AND to_regclass('public.felorxes') IS NOT NULL
     AND EXISTS (SELECT 1 FROM public.puupees LIMIT 1) THEN
    RAISE EXCEPTION 'Both puupees and felorxes contain migration sources.';
  END IF;
END $$;
```

- [ ] PostgreSQL SQL 创建 legacy 兼容 view：
  - `puupees`
  - `user_puupees`
  - `tag_puupees`

- [ ] PostgreSQL compatibility view 可以只保证读取；写入兼容不作为 PostgreSQL sync-node 回滚策略。PostgreSQL 回滚必须依赖迁移前 `pg_dump`。

- [ ] Runbook 必须包含：
  - 停止所有 sync-node 写入进程。
  - `pg_dump -Fc` 备份。
  - `pg_restore --list` 校验备份可读。
  - 运行迁移 SQL。
  - 校验新表计数。
  - 校验旧 view 计数。
  - 回滚只允许先确认备份文件存在，再 drop/recreate database。

- [ ] 测试 SQL 字符串覆盖以上行为，不在默认测试中启动 PostgreSQL 容器。

- [ ] 第五次提交：

```bash
git add packages/sync/felorx_sync
git commit -m "docs(sync): 增加 sync-node 数据库品牌迁移手册"
```

---

## Task 8：全量验证与旧名扫描

- [ ] Core 包：

```bash
cd packages/core/felorx
flutter test
dart analyze
```

- [ ] Sync 包：

```bash
cd packages/sync/felorx_sync
flutter test
dart analyze
```

- [ ] Sync API 包：

```bash
cd packages/api/felorx_sync_api
dart test
dart analyze
```

- [ ] 旧名扫描只允许以下兼容/文档位置保留旧名：
  - 本地 v1-v16 migration/snapshot 历史文件。
  - `LocalBrandDatabaseNames` 和兼容视图/trigger。
  - sync 协议 v1 映射函数和测试。
  - sync-node PostgreSQL runbook。
  - `FelorxDatabase` 兼容 typedef/getter。
  - API 模型旧 JSON fallback 测试。

- [ ] 执行扫描：

```bash
rg -n "puupees|puupee|Puupee|Puupees|user_puupees|tag_puupees|puupee_id" \
  packages/core/felorx packages/sync/felorx_sync packages/api/felorx_sync_api
```

- [ ] 对扫描结果逐项分类：`history`、`compatibility`、`test-compatibility`、`docs-runbook`、`must-fix`。

- [ ] 所有 `must-fix` 清零后提交：

```bash
git add packages/core/felorx packages/sync/felorx_sync packages/api/felorx_sync_api
git commit -m "test(sync): 覆盖 Felorx 本地数据库迁移"
```

---

## 回滚策略

**SQLite 客户端：**

- 自动迁移前，宿主应用应复用 `DatabaseUtils.backupDatabase()` 对真实 DB 文件做一次 timestamp 备份。
- v17 迁移失败时：
  1. 关闭当前 Drift 连接。
  2. 用备份文件替换数据库文件。
  3. 删除同名 `-wal` 和 `-shm`。
  4. 恢复旧版本应用或重新运行修复后的 v17。
- 不支持“应用降级后继续写新 v17 数据库”作为正式回滚方式。旧名视图和 trigger 只是兼容窗口，不替代备份恢复。

**sync-node PostgreSQL：**

- 只允许从迁移前 `pg_dump -Fc` 备份恢复。
- 禁止在生产库上执行手写反向 rename 作为回滚。
- 回滚脚本必须使用明确的确认变量，确认数据库名和备份文件名完全匹配后才能 drop database。

## 提交节奏

建议至少 5 个提交，避免一次性巨量改动：

1. `feat(felorx): 迁移本地数据库物理命名`
2. `feat(sync): 切换 CRDT 存储到 Felorx 命名`
3. `feat(sync): 增加 Felorx changeset 协议 v2`
4. `feat(sync-api): 支持 Felorx changeset key`
5. `docs(sync): 增加 sync-node 数据库品牌迁移手册`

每个提交前必须运行对应包的最小测试；最终提交前运行 Task 8 全量验证。
