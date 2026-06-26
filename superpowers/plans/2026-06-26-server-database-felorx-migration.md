# Server Database Felorx Migration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Migrate existing server databases from legacy Puupee/Puupees database artifacts to Felorx artifacts without data loss, with PostgreSQL production coverage first and compatibility views for the transition window.

**Architecture:** Add an idempotent EF Core data migration that renames legacy server tables from `Puupee*` or `Puupees*` to `Felorx*`, renames PostgreSQL indexes/constraints/sequences that still contain the legacy brand, normalizes legacy enum/string data, and creates legacy-name PostgreSQL views for short-term rollback/compatibility. Keep local Drift tables (`puupees`, `user_puupees`, `tag_puupees`) and sync JSON keys out of this server-first phase; they need a separate protocol/local-database migration.

**Tech Stack:** .NET 8, ABP Framework, EF Core migrations, PostgreSQL/Npgsql for production, SQLite for fast migration harness tests, Testcontainers PostgreSQL for full integration tests.

---

## Scope And Non-Goals

**In scope for this plan:**
- Server EF Core database artifacts under `api/src/Felorx.EntityFrameworkCore`.
- Existing production databases that may still have `Puupee*` or `Puupees*` tables while the current code expects `Felorx*`.
- Existing legacy publisher data where `FelorxAppReleases.Publisher = 'Puupee'`.
- Migration discovery, because the current hand-written `20260626081500_RenameLegacyPublisherToFelorx.cs` file has no `.Designer.cs` and no `[Migration]` attribute.
- PostgreSQL compatibility views named `Puupee*` and `Puupees*` pointing at renamed `Felorx*` tables for one release window.

**Out of scope for this plan:**
- Local Flutter/Drift tables named `puupees`, `user_puupees`, `tag_puupees`.
- Sync protocol JSON keys named `puupees`, `user_puupees`, `puupee`, or `puupee_id`.
- Website npm package name `puupee-api`.
- Coding/Kubernetes registry namespace `code2code-docker.pkg.coding.net/puupees/...`.

Those are deliberate compatibility surfaces and should be migrated in later, separate plans.

## Files

- Modify: `api/src/Felorx.EntityFrameworkCore/Migrations/20260626081500_RenameLegacyPublisherToFelorx.cs`
- Create: `api/src/Felorx.EntityFrameworkCore/Migrations/Branding/LegacyBrandDatabaseNames.cs`
- Create: `api/src/Felorx.EntityFrameworkCore/Migrations/Branding/LegacyBrandMigrationSql.cs`
- Create: `api/src/Felorx.EntityFrameworkCore/Migrations/20260627090000_RenameLegacyDatabaseArtifactsToFelorx.cs`
- Modify: `api/test/Felorx.EntityFrameworkCore.Tests/Felorx.EntityFrameworkCore.Tests.csproj`
- Create: `api/test/Felorx.EntityFrameworkCore.Tests/EntityFrameworkCore/Branding/ManualMigrationDiscoveryTests.cs`
- Create: `api/test/Felorx.EntityFrameworkCore.Tests/EntityFrameworkCore/Branding/LegacyBrandMigrationSqlTests.cs`
- Create: `api/test/Felorx.EntityFrameworkCore.Tests/EntityFrameworkCore/Branding/LegacyBrandSqliteMigrationTests.cs`
- Create: `api/test/Felorx.EntityFrameworkCore.Tests/EntityFrameworkCore/Branding/LegacyBrandPostgreSqlMigrationTests.cs`
- Create: `api/docs/server-database-felorx-migration-runbook.md`

## Migration Artifact Map

The server migration must cover every current Felorx-owned table configured in `FelorxDbContext` with `FelorxConsts.DbTablePrefix`:

```text
FelorxAiModels
FelorxAiProviders
FelorxAppAssets
FelorxAppDependencies
FelorxAppFeatureLocales
FelorxAppFeatures
FelorxAppFeedbacks
FelorxAppLocales
FelorxAppPlanPrices
FelorxAppPricingItems
FelorxAppPricings
FelorxAppReleases
FelorxAppRunRecords
FelorxAppSdkRelations
FelorxAppSdks
FelorxAppTesters
FelorxAppUserScores
FelorxAppleNotificationRecords
FelorxApps
FelorxBuildRecords
FelorxCreditAccounts
FelorxCreditLedgerEntries
FelorxCreditOrders
FelorxCreditPackages
FelorxDeployRecords
FelorxDevices
FelorxMembers
FelorxMessageSourceCategories
FelorxMessageSourceRouteSubs
FelorxMessageSourceRoutes
FelorxMessageSources
FelorxMessageTemplateReleases
FelorxMessageTemplates
FelorxStorageObjects
FelorxStoreProductMappings
FelorxSubscriptionContracts
FelorxSubscriptionEntitlements
FelorxSubscriptionOrders
FelorxSubscriptions
FelorxUserStorages
FelorxVerifyReceiptRecords
FelorxVerifyReceiptResults
```

For each table, support both legacy aliases:

```text
Puupee<suffix>  -> Felorx<suffix>
Puupees<suffix> -> Felorx<suffix>
```

Example:

```text
PuupeeAppReleases  -> FelorxAppReleases
PuupeesAppReleases -> FelorxAppReleases
```

---

### Task 1: Make Existing Manual Data Migration Discoverable

**Files:**
- Modify: `api/src/Felorx.EntityFrameworkCore/Migrations/20260626081500_RenameLegacyPublisherToFelorx.cs`
- Create: `api/test/Felorx.EntityFrameworkCore.Tests/EntityFrameworkCore/Branding/ManualMigrationDiscoveryTests.cs`

- [ ] **Step 1: Write the failing discovery test**

Create `api/test/Felorx.EntityFrameworkCore.Tests/EntityFrameworkCore/Branding/ManualMigrationDiscoveryTests.cs`:

```csharp
using Microsoft.EntityFrameworkCore.Migrations;
using Microsoft.Extensions.DependencyInjection;
using Shouldly;
using Xunit;

namespace Felorx.EntityFrameworkCore.Branding;

[Collection(FelorxTestConsts.CollectionDefinitionName)]
public class ManualMigrationDiscoveryTests : FelorxEntityFrameworkCoreTestBase
{
    [Fact]
    public void RenameLegacyPublisherToFelorxMigration_Is_Discoverable_By_EfCore()
    {
        var migrationsAssembly = ServiceProvider.GetRequiredService<IMigrationsAssembly>();

        migrationsAssembly.Migrations.Keys.ShouldContain(
            "20260626081500_RenameLegacyPublisherToFelorx");
    }
}
```

- [ ] **Step 2: Run the test and verify it fails**

Run:

```bash
cd api
dotnet test test/Felorx.EntityFrameworkCore.Tests/Felorx.EntityFrameworkCore.Tests.csproj --filter FullyQualifiedName~ManualMigrationDiscoveryTests
```

Expected before the fix:

```text
Failed!  - Failed: 1
ShouldContain ... "20260626081500_RenameLegacyPublisherToFelorx"
```

- [ ] **Step 3: Add EF migration attributes to the existing manual migration**

Modify `api/src/Felorx.EntityFrameworkCore/Migrations/20260626081500_RenameLegacyPublisherToFelorx.cs` to include attributes directly on the migration class:

```csharp
using Microsoft.EntityFrameworkCore.Infrastructure;
using Microsoft.EntityFrameworkCore.Migrations;
using Felorx.EntityFrameworkCore;

#nullable disable

namespace Felorx.Migrations
{
    /// <inheritdoc />
    [DbContext(typeof(FelorxDbContext))]
    [Migration("20260626081500_RenameLegacyPublisherToFelorx")]
    public partial class RenameLegacyPublisherToFelorx : Migration
    {
        /// <inheritdoc />
        protected override void Up(MigrationBuilder migrationBuilder)
        {
            migrationBuilder.Sql(
                """UPDATE "FelorxAppReleases" SET "Publisher" = 'Felorx' WHERE "Publisher" = 'Puupee';""");
        }

        /// <inheritdoc />
        protected override void Down(MigrationBuilder migrationBuilder)
        {
            migrationBuilder.Sql(
                """UPDATE "FelorxAppReleases" SET "Publisher" = 'Puupee' WHERE "Publisher" = 'Felorx';""");
        }
    }
}
```

- [ ] **Step 4: Run the discovery test and verify it passes**

Run:

```bash
cd api
dotnet test test/Felorx.EntityFrameworkCore.Tests/Felorx.EntityFrameworkCore.Tests.csproj --filter FullyQualifiedName~ManualMigrationDiscoveryTests
```

Expected:

```text
Passed!  - Failed: 0
```

- [ ] **Step 5: Commit**

```bash
git add api/src/Felorx.EntityFrameworkCore/Migrations/20260626081500_RenameLegacyPublisherToFelorx.cs \
  api/test/Felorx.EntityFrameworkCore.Tests/EntityFrameworkCore/Branding/ManualMigrationDiscoveryTests.cs
git commit -m "fix(api): 确保手写品牌数据迁移可发现"
```

---

### Task 2: Add Shared Legacy Brand Table Map

**Files:**
- Create: `api/src/Felorx.EntityFrameworkCore/Migrations/Branding/LegacyBrandDatabaseNames.cs`
- Create: `api/test/Felorx.EntityFrameworkCore.Tests/EntityFrameworkCore/Branding/LegacyBrandMigrationSqlTests.cs`

- [ ] **Step 1: Write the table map coverage test**

Create `api/test/Felorx.EntityFrameworkCore.Tests/EntityFrameworkCore/Branding/LegacyBrandMigrationSqlTests.cs`:

```csharp
using System.Linq;
using Felorx.EntityFrameworkCore.Migrations.Branding;
using Shouldly;
using Xunit;

namespace Felorx.EntityFrameworkCore.Branding;

public class LegacyBrandMigrationSqlTests
{
    private static readonly string[] ExpectedFelorxTables =
    [
        "FelorxAiModels",
        "FelorxAiProviders",
        "FelorxAppAssets",
        "FelorxAppDependencies",
        "FelorxAppFeatureLocales",
        "FelorxAppFeatures",
        "FelorxAppFeedbacks",
        "FelorxAppLocales",
        "FelorxAppPlanPrices",
        "FelorxAppPricingItems",
        "FelorxAppPricings",
        "FelorxAppReleases",
        "FelorxAppRunRecords",
        "FelorxAppSdkRelations",
        "FelorxAppSdks",
        "FelorxAppTesters",
        "FelorxAppUserScores",
        "FelorxAppleNotificationRecords",
        "FelorxApps",
        "FelorxBuildRecords",
        "FelorxCreditAccounts",
        "FelorxCreditLedgerEntries",
        "FelorxCreditOrders",
        "FelorxCreditPackages",
        "FelorxDeployRecords",
        "FelorxDevices",
        "FelorxMembers",
        "FelorxMessageSourceCategories",
        "FelorxMessageSourceRouteSubs",
        "FelorxMessageSourceRoutes",
        "FelorxMessageSources",
        "FelorxMessageTemplateReleases",
        "FelorxMessageTemplates",
        "FelorxStorageObjects",
        "FelorxStoreProductMappings",
        "FelorxSubscriptionContracts",
        "FelorxSubscriptionEntitlements",
        "FelorxSubscriptionOrders",
        "FelorxSubscriptions",
        "FelorxUserStorages",
        "FelorxVerifyReceiptRecords",
        "FelorxVerifyReceiptResults"
    ];

    [Fact]
    public void NewTableNames_Cover_All_Felorx_Owned_Tables()
    {
        LegacyBrandDatabaseNames.NewTableNames.ShouldBe(ExpectedFelorxTables);
    }

    [Fact]
    public void LegacyTablePairs_Include_Puupee_And_Puupees_Aliases_For_Each_Table()
    {
        var pairs = LegacyBrandDatabaseNames.LegacyTablePairs.ToList();

        pairs.Count.ShouldBe(ExpectedFelorxTables.Length * 2);
        pairs.ShouldContain(x => x.LegacyName == "PuupeeAppReleases" && x.NewName == "FelorxAppReleases");
        pairs.ShouldContain(x => x.LegacyName == "PuupeesAppReleases" && x.NewName == "FelorxAppReleases");
        pairs.ShouldContain(x => x.LegacyName == "PuupeeCreditAccounts" && x.NewName == "FelorxCreditAccounts");
        pairs.ShouldContain(x => x.LegacyName == "PuupeesCreditAccounts" && x.NewName == "FelorxCreditAccounts");
    }
}
```

- [ ] **Step 2: Run the test and verify it fails because the helper does not exist**

Run:

```bash
cd api
dotnet test test/Felorx.EntityFrameworkCore.Tests/Felorx.EntityFrameworkCore.Tests.csproj --filter FullyQualifiedName~LegacyBrandMigrationSqlTests
```

Expected:

```text
error CS0234: The type or namespace name 'Migrations.Branding' does not exist
```

- [ ] **Step 3: Add the shared table map**

Create `api/src/Felorx.EntityFrameworkCore/Migrations/Branding/LegacyBrandDatabaseNames.cs`:

```csharp
using System.Collections.Generic;
using System.Linq;

namespace Felorx.EntityFrameworkCore.Migrations.Branding;

internal sealed record LegacyBrandTablePair(string LegacyName, string NewName);

internal static class LegacyBrandDatabaseNames
{
    private const string NewPrefix = "Felorx";

    private static readonly string[] LegacyPrefixes = ["Puupee", "Puupees"];

    public static readonly string[] NewTableNames =
    [
        "FelorxAiModels",
        "FelorxAiProviders",
        "FelorxAppAssets",
        "FelorxAppDependencies",
        "FelorxAppFeatureLocales",
        "FelorxAppFeatures",
        "FelorxAppFeedbacks",
        "FelorxAppLocales",
        "FelorxAppPlanPrices",
        "FelorxAppPricingItems",
        "FelorxAppPricings",
        "FelorxAppReleases",
        "FelorxAppRunRecords",
        "FelorxAppSdkRelations",
        "FelorxAppSdks",
        "FelorxAppTesters",
        "FelorxAppUserScores",
        "FelorxAppleNotificationRecords",
        "FelorxApps",
        "FelorxBuildRecords",
        "FelorxCreditAccounts",
        "FelorxCreditLedgerEntries",
        "FelorxCreditOrders",
        "FelorxCreditPackages",
        "FelorxDeployRecords",
        "FelorxDevices",
        "FelorxMembers",
        "FelorxMessageSourceCategories",
        "FelorxMessageSourceRouteSubs",
        "FelorxMessageSourceRoutes",
        "FelorxMessageSources",
        "FelorxMessageTemplateReleases",
        "FelorxMessageTemplates",
        "FelorxStorageObjects",
        "FelorxStoreProductMappings",
        "FelorxSubscriptionContracts",
        "FelorxSubscriptionEntitlements",
        "FelorxSubscriptionOrders",
        "FelorxSubscriptions",
        "FelorxUserStorages",
        "FelorxVerifyReceiptRecords",
        "FelorxVerifyReceiptResults"
    ];

    public static IReadOnlyList<LegacyBrandTablePair> LegacyTablePairs =>
        NewTableNames
            .SelectMany(newName =>
            {
                var suffix = newName[NewPrefix.Length..];
                return LegacyPrefixes.Select(prefix => new LegacyBrandTablePair(prefix + suffix, newName));
            })
            .ToArray();

    public static string ToFelorxIdentifier(string identifier)
    {
        return identifier
            .Replace("Puupees", NewPrefix)
            .Replace("Puupee", NewPrefix);
    }

    public static string ToPrimaryLegacyIdentifier(string identifier)
    {
        return identifier.Replace(NewPrefix, "Puupee");
    }
}
```

- [ ] **Step 4: Run the table map tests and verify they pass**

Run:

```bash
cd api
dotnet test test/Felorx.EntityFrameworkCore.Tests/Felorx.EntityFrameworkCore.Tests.csproj --filter FullyQualifiedName~LegacyBrandMigrationSqlTests
```

Expected:

```text
Passed!  - Failed: 0
```

- [ ] **Step 5: Commit**

```bash
git add api/src/Felorx.EntityFrameworkCore/Migrations/Branding/LegacyBrandDatabaseNames.cs \
  api/test/Felorx.EntityFrameworkCore.Tests/EntityFrameworkCore/Branding/LegacyBrandMigrationSqlTests.cs
git commit -m "test(api): 固定服务端品牌表迁移映射"
```

---

### Task 3: Add Idempotent PostgreSQL And SQLite SQL Builders

**Files:**
- Create: `api/src/Felorx.EntityFrameworkCore/Migrations/Branding/LegacyBrandMigrationSql.cs`
- Modify: `api/test/Felorx.EntityFrameworkCore.Tests/EntityFrameworkCore/Branding/LegacyBrandMigrationSqlTests.cs`

- [ ] **Step 1: Add failing SQL generation tests**

Append these tests to `LegacyBrandMigrationSqlTests`:

```csharp
[Fact]
public void PostgresUpSql_Renames_Tables_Identifiers_And_Creates_Legacy_Views()
{
    var sql = LegacyBrandMigrationSql.BuildPostgresUpSql();

    sql.ShouldContain("ALTER TABLE %I RENAME TO %I");
    sql.ShouldContain("CREATE OR REPLACE VIEW %I AS SELECT * FROM %I");
    sql.ShouldContain("ALTER INDEX %I RENAME TO %I");
    sql.ShouldContain("RENAME CONSTRAINT %I TO %I");
    sql.ShouldContain("UPDATE \"FelorxAppReleases\" SET \"Publisher\" = 'Felorx'");
    sql.ShouldContain("PuupeeAppReleases");
    sql.ShouldContain("PuupeesAppReleases");
    sql.ShouldContain("FelorxAppReleases");
}

[Fact]
public void SqliteUpStatements_Rename_Legacy_Tables_And_Normalize_Publisher()
{
    var sql = LegacyBrandMigrationSql.BuildSqliteUpStatements().ToList();

    sql.ShouldContain("""ALTER TABLE "PuupeeAppReleases" RENAME TO "FelorxAppReleases";""");
    sql.ShouldContain("""ALTER TABLE "PuupeesAppReleases" RENAME TO "FelorxAppReleases";""");
    sql.ShouldContain("""UPDATE "FelorxAppReleases" SET "Publisher" = 'Felorx' WHERE "Publisher" = 'Puupee';""");
}

[Fact]
public void PostgresDownSql_Drops_Legacy_Views_And_Renames_To_Primary_Legacy_Prefix()
{
    var sql = LegacyBrandMigrationSql.BuildPostgresDownSql();

    sql.ShouldContain("DROP VIEW IF EXISTS %I");
    sql.ShouldContain("ALTER TABLE %I RENAME TO %I");
    sql.ShouldContain("UPDATE \"PuupeeAppReleases\" SET \"Publisher\" = 'Puupee'");
}
```

- [ ] **Step 2: Run the SQL tests and verify they fail because the SQL builder does not exist**

Run:

```bash
cd api
dotnet test test/Felorx.EntityFrameworkCore.Tests/Felorx.EntityFrameworkCore.Tests.csproj --filter FullyQualifiedName~LegacyBrandMigrationSqlTests
```

Expected:

```text
error CS0103: The name 'LegacyBrandMigrationSql' does not exist
```

- [ ] **Step 3: Add the SQL builder**

Create `api/src/Felorx.EntityFrameworkCore/Migrations/Branding/LegacyBrandMigrationSql.cs`:

```csharp
using System.Collections.Generic;
using System.Linq;
using System.Text;

namespace Felorx.EntityFrameworkCore.Migrations.Branding;

internal static class LegacyBrandMigrationSql
{
    public static string BuildPostgresUpSql()
    {
        var values = BuildPostgresValues();

        return $$"""
DO $$
DECLARE
    item record;
    legacy_table_exists boolean;
    new_table_exists boolean;
    new_identifier text;
BEGIN
    FOR item IN SELECT * FROM (VALUES
{{values}}
    ) AS t(legacy_name, new_name) LOOP
        SELECT EXISTS (
            SELECT 1 FROM pg_class c
            JOIN pg_namespace n ON n.oid = c.relnamespace
            WHERE n.nspname = current_schema()
              AND c.relname = item.legacy_name
              AND c.relkind IN ('r', 'p')
        ) INTO legacy_table_exists;

        SELECT EXISTS (
            SELECT 1 FROM pg_class c
            JOIN pg_namespace n ON n.oid = c.relnamespace
            WHERE n.nspname = current_schema()
              AND c.relname = item.new_name
              AND c.relkind IN ('r', 'p')
        ) INTO new_table_exists;

        IF legacy_table_exists AND new_table_exists THEN
            RAISE EXCEPTION 'Both legacy table % and new table % exist. Resolve duplicate tables before Felorx migration.', item.legacy_name, item.new_name;
        END IF;

        IF legacy_table_exists AND NOT new_table_exists THEN
            EXECUTE format('ALTER TABLE %I RENAME TO %I', item.legacy_name, item.new_name);
        END IF;
    END LOOP;

    FOR item IN
        SELECT c.relname
        FROM pg_class c
        JOIN pg_namespace n ON n.oid = c.relnamespace
        WHERE n.nspname = current_schema()
          AND c.relkind IN ('i', 'I')
          AND (c.relname LIKE '%Puupee%' OR c.relname LIKE '%Puupees%')
    LOOP
        new_identifier := replace(replace(item.relname, 'Puupees', 'Felorx'), 'Puupee', 'Felorx');
        IF NOT EXISTS (
            SELECT 1 FROM pg_class c
            JOIN pg_namespace n ON n.oid = c.relnamespace
            WHERE n.nspname = current_schema()
              AND c.relname = new_identifier
        ) THEN
            EXECUTE format('ALTER INDEX %I RENAME TO %I', item.relname, new_identifier);
        END IF;
    END LOOP;

    FOR item IN
        SELECT c.relname
        FROM pg_class c
        JOIN pg_namespace n ON n.oid = c.relnamespace
        WHERE n.nspname = current_schema()
          AND c.relkind = 'S'
          AND (c.relname LIKE '%Puupee%' OR c.relname LIKE '%Puupees%')
    LOOP
        new_identifier := replace(replace(item.relname, 'Puupees', 'Felorx'), 'Puupee', 'Felorx');
        IF NOT EXISTS (
            SELECT 1 FROM pg_class c
            JOIN pg_namespace n ON n.oid = c.relnamespace
            WHERE n.nspname = current_schema()
              AND c.relname = new_identifier
        ) THEN
            EXECUTE format('ALTER SEQUENCE %I RENAME TO %I', item.relname, new_identifier);
        END IF;
    END LOOP;

    FOR item IN
        SELECT con.conname, con.conrelid::regclass AS table_name
        FROM pg_constraint con
        JOIN pg_namespace n ON n.oid = con.connamespace
        WHERE n.nspname = current_schema()
          AND (con.conname LIKE '%Puupee%' OR con.conname LIKE '%Puupees%')
    LOOP
        new_identifier := replace(replace(item.conname, 'Puupees', 'Felorx'), 'Puupee', 'Felorx');
        IF NOT EXISTS (
            SELECT 1 FROM pg_constraint con
            JOIN pg_namespace n ON n.oid = con.connamespace
            WHERE n.nspname = current_schema()
              AND con.conname = new_identifier
        ) THEN
            EXECUTE format('ALTER TABLE %s RENAME CONSTRAINT %I TO %I', item.table_name, item.conname, new_identifier);
        END IF;
    END LOOP;

    IF EXISTS (
        SELECT 1 FROM information_schema.tables
        WHERE table_schema = current_schema()
          AND table_name = 'FelorxAppReleases'
    ) THEN
        EXECUTE 'UPDATE "FelorxAppReleases" SET "Publisher" = ''Felorx'' WHERE "Publisher" = ''Puupee''';
    END IF;

    FOR item IN SELECT * FROM (VALUES
{{values}}
    ) AS t(legacy_name, new_name) LOOP
        IF EXISTS (
            SELECT 1 FROM pg_class c
            JOIN pg_namespace n ON n.oid = c.relnamespace
            WHERE n.nspname = current_schema()
              AND c.relname = item.new_name
              AND c.relkind IN ('r', 'p')
        ) THEN
            EXECUTE format('DROP VIEW IF EXISTS %I', item.legacy_name);
            EXECUTE format('CREATE OR REPLACE VIEW %I AS SELECT * FROM %I', item.legacy_name, item.new_name);
        END IF;
    END LOOP;
END $$;
""";
    }

    public static string BuildPostgresDownSql()
    {
        var values = BuildPostgresValues();

        return $$"""
DO $$
DECLARE
    item record;
    primary_legacy_name text;
BEGIN
    FOR item IN SELECT * FROM (VALUES
{{values}}
    ) AS t(legacy_name, new_name) LOOP
        EXECUTE format('DROP VIEW IF EXISTS %I', item.legacy_name);
    END LOOP;

    FOR item IN SELECT DISTINCT new_name FROM (VALUES
{{values}}
    ) AS t(legacy_name, new_name) LOOP
        primary_legacy_name := replace(item.new_name, 'Felorx', 'Puupee');
        IF EXISTS (
            SELECT 1 FROM pg_class c
            JOIN pg_namespace n ON n.oid = c.relnamespace
            WHERE n.nspname = current_schema()
              AND c.relname = item.new_name
              AND c.relkind IN ('r', 'p')
        ) AND NOT EXISTS (
            SELECT 1 FROM pg_class c
            JOIN pg_namespace n ON n.oid = c.relnamespace
            WHERE n.nspname = current_schema()
              AND c.relname = primary_legacy_name
        ) THEN
            EXECUTE format('ALTER TABLE %I RENAME TO %I', item.new_name, primary_legacy_name);
        END IF;
    END LOOP;

    IF EXISTS (
        SELECT 1 FROM information_schema.tables
        WHERE table_schema = current_schema()
          AND table_name = 'PuupeeAppReleases'
    ) THEN
        EXECUTE 'UPDATE "PuupeeAppReleases" SET "Publisher" = ''Puupee'' WHERE "Publisher" = ''Felorx''';
    END IF;
END $$;
""";
    }

    public static IReadOnlyList<string> BuildSqliteUpStatements()
    {
        var statements = new List<string>();

        foreach (var pair in LegacyBrandDatabaseNames.LegacyTablePairs)
        {
            statements.Add($"""
ALTER TABLE "{pair.LegacyName}" RENAME TO "{pair.NewName}";
""");
        }

        statements.Add("""UPDATE "FelorxAppReleases" SET "Publisher" = 'Felorx' WHERE "Publisher" = 'Puupee';""");

        return statements;
    }

    public static IReadOnlyList<string> BuildSqliteDownStatements()
    {
        return LegacyBrandDatabaseNames.NewTableNames
            .Select(name => $"""ALTER TABLE "{name}" RENAME TO "{LegacyBrandDatabaseNames.ToPrimaryLegacyIdentifier(name)}";""")
            .Append("""UPDATE "PuupeeAppReleases" SET "Publisher" = 'Puupee' WHERE "Publisher" = 'Felorx';""")
            .ToArray();
    }

    private static string BuildPostgresValues()
    {
        var builder = new StringBuilder();
        var pairs = LegacyBrandDatabaseNames.LegacyTablePairs;

        for (var i = 0; i < pairs.Count; i++)
        {
            var pair = pairs[i];
            builder.Append("        ('");
            builder.Append(pair.LegacyName);
            builder.Append("', '");
            builder.Append(pair.NewName);
            builder.Append("')");
            if (i < pairs.Count - 1)
            {
                builder.Append(',');
            }
            builder.AppendLine();
        }

        return builder.ToString().TrimEnd();
    }
}
```

- [ ] **Step 4: Run the SQL tests and verify they pass**

Run:

```bash
cd api
dotnet test test/Felorx.EntityFrameworkCore.Tests/Felorx.EntityFrameworkCore.Tests.csproj --filter FullyQualifiedName~LegacyBrandMigrationSqlTests
```

Expected:

```text
Passed!  - Failed: 0
```

- [ ] **Step 5: Commit**

```bash
git add api/src/Felorx.EntityFrameworkCore/Migrations/Branding/LegacyBrandMigrationSql.cs \
  api/test/Felorx.EntityFrameworkCore.Tests/EntityFrameworkCore/Branding/LegacyBrandMigrationSqlTests.cs
git commit -m "feat(api): 生成旧品牌数据库迁移 SQL"
```

---

### Task 4: Add The Server Database Rename Migration

**Files:**
- Create: `api/src/Felorx.EntityFrameworkCore/Migrations/20260627090000_RenameLegacyDatabaseArtifactsToFelorx.cs`
- Modify: `api/test/Felorx.EntityFrameworkCore.Tests/EntityFrameworkCore/Branding/ManualMigrationDiscoveryTests.cs`

- [ ] **Step 1: Extend the discovery test for the new migration**

Add this assertion to `RenameLegacyPublisherToFelorxMigration_Is_Discoverable_By_EfCore`:

```csharp
migrationsAssembly.Migrations.Keys.ShouldContain(
    "20260627090000_RenameLegacyDatabaseArtifactsToFelorx");
```

- [ ] **Step 2: Run the discovery test and verify it fails**

Run:

```bash
cd api
dotnet test test/Felorx.EntityFrameworkCore.Tests/Felorx.EntityFrameworkCore.Tests.csproj --filter FullyQualifiedName~ManualMigrationDiscoveryTests
```

Expected:

```text
Failed!  - Failed: 1
ShouldContain ... "20260627090000_RenameLegacyDatabaseArtifactsToFelorx"
```

- [ ] **Step 3: Add the migration**

Create `api/src/Felorx.EntityFrameworkCore/Migrations/20260627090000_RenameLegacyDatabaseArtifactsToFelorx.cs`:

```csharp
using System;
using Microsoft.EntityFrameworkCore.Infrastructure;
using Microsoft.EntityFrameworkCore.Migrations;
using Felorx.EntityFrameworkCore;
using Felorx.EntityFrameworkCore.Migrations.Branding;

#nullable disable

namespace Felorx.Migrations
{
    /// <inheritdoc />
    [DbContext(typeof(FelorxDbContext))]
    [Migration("20260627090000_RenameLegacyDatabaseArtifactsToFelorx")]
    public partial class RenameLegacyDatabaseArtifactsToFelorx : Migration
    {
        /// <inheritdoc />
        protected override void Up(MigrationBuilder migrationBuilder)
        {
            if (migrationBuilder.ActiveProvider.Contains("Npgsql", StringComparison.OrdinalIgnoreCase))
            {
                migrationBuilder.Sql(LegacyBrandMigrationSql.BuildPostgresUpSql());
                return;
            }

            if (migrationBuilder.ActiveProvider.Contains("Sqlite", StringComparison.OrdinalIgnoreCase))
            {
                foreach (var statement in LegacyBrandMigrationSql.BuildSqliteUpStatements())
                {
                    migrationBuilder.Sql(statement, suppressTransaction: false);
                }
                return;
            }

            throw new NotSupportedException(
                $"Provider {migrationBuilder.ActiveProvider} is not supported by Felorx legacy brand database migration.");
        }

        /// <inheritdoc />
        protected override void Down(MigrationBuilder migrationBuilder)
        {
            if (migrationBuilder.ActiveProvider.Contains("Npgsql", StringComparison.OrdinalIgnoreCase))
            {
                migrationBuilder.Sql(LegacyBrandMigrationSql.BuildPostgresDownSql());
                return;
            }

            if (migrationBuilder.ActiveProvider.Contains("Sqlite", StringComparison.OrdinalIgnoreCase))
            {
                foreach (var statement in LegacyBrandMigrationSql.BuildSqliteDownStatements())
                {
                    migrationBuilder.Sql(statement, suppressTransaction: false);
                }
                return;
            }

            throw new NotSupportedException(
                $"Provider {migrationBuilder.ActiveProvider} is not supported by Felorx legacy brand database migration.");
        }
    }
}
```

- [ ] **Step 4: Run the discovery test and verify it passes**

Run:

```bash
cd api
dotnet test test/Felorx.EntityFrameworkCore.Tests/Felorx.EntityFrameworkCore.Tests.csproj --filter FullyQualifiedName~ManualMigrationDiscoveryTests
```

Expected:

```text
Passed!  - Failed: 0
```

- [ ] **Step 5: Commit**

```bash
git add api/src/Felorx.EntityFrameworkCore/Migrations/20260627090000_RenameLegacyDatabaseArtifactsToFelorx.cs \
  api/test/Felorx.EntityFrameworkCore.Tests/EntityFrameworkCore/Branding/ManualMigrationDiscoveryTests.cs
git commit -m "feat(api): 添加服务端旧品牌数据库迁移"
```

---

### Task 5: Add SQLite Migration Harness Tests

**Files:**
- Create: `api/test/Felorx.EntityFrameworkCore.Tests/EntityFrameworkCore/Branding/LegacyBrandSqliteMigrationTests.cs`

- [ ] **Step 1: Write SQLite migration tests**

Create `api/test/Felorx.EntityFrameworkCore.Tests/EntityFrameworkCore/Branding/LegacyBrandSqliteMigrationTests.cs`:

```csharp
using System;
using System.Threading.Tasks;
using Felorx.EntityFrameworkCore.Migrations.Branding;
using Microsoft.Data.Sqlite;
using Shouldly;
using Xunit;

namespace Felorx.EntityFrameworkCore.Branding;

public class LegacyBrandSqliteMigrationTests
{
    [Fact]
    public async Task SqliteUp_Renames_Legacy_Table_And_Preserves_Data()
    {
        await using var connection = new SqliteConnection("Data Source=:memory:");
        await connection.OpenAsync();

        await ExecuteAsync(connection, """
CREATE TABLE "PuupeeAppReleases" (
    "Id" TEXT NOT NULL PRIMARY KEY,
    "AppId" TEXT NOT NULL,
    "Publisher" TEXT NULL,
    "VersionCode" INTEGER NOT NULL
);
""");
        await ExecuteAsync(connection, """
INSERT INTO "PuupeeAppReleases" ("Id", "AppId", "Publisher", "VersionCode")
VALUES ('release-1', 'app-1', 'Puupee', 42);
""");

        foreach (var statement in LegacyBrandMigrationSql.BuildSqliteUpStatements())
        {
            try
            {
                await ExecuteAsync(connection, statement);
            }
            catch (SqliteException exception) when (
                exception.SqliteErrorCode == 1 &&
                exception.Message.Contains("no such table", StringComparison.OrdinalIgnoreCase))
            {
                // The harness only creates the table required by this test.
            }
        }

        var legacyExists = await ScalarAsync<long>(connection, """
SELECT COUNT(*) FROM sqlite_master WHERE type = 'table' AND name = 'PuupeeAppReleases';
""");
        var felorxExists = await ScalarAsync<long>(connection, """
SELECT COUNT(*) FROM sqlite_master WHERE type = 'table' AND name = 'FelorxAppReleases';
""");
        var publisher = await ScalarAsync<string>(connection, """
SELECT "Publisher" FROM "FelorxAppReleases" WHERE "Id" = 'release-1';
""");

        legacyExists.ShouldBe(0);
        felorxExists.ShouldBe(1);
        publisher.ShouldBe("Felorx");
    }

    [Fact]
    public async Task SqliteDown_Renames_Felorx_Table_Back_To_Primary_Legacy_Name()
    {
        await using var connection = new SqliteConnection("Data Source=:memory:");
        await connection.OpenAsync();

        await ExecuteAsync(connection, """
CREATE TABLE "FelorxAppReleases" (
    "Id" TEXT NOT NULL PRIMARY KEY,
    "AppId" TEXT NOT NULL,
    "Publisher" TEXT NULL,
    "VersionCode" INTEGER NOT NULL
);
""");
        await ExecuteAsync(connection, """
INSERT INTO "FelorxAppReleases" ("Id", "AppId", "Publisher", "VersionCode")
VALUES ('release-1', 'app-1', 'Felorx', 42);
""");

        foreach (var statement in LegacyBrandMigrationSql.BuildSqliteDownStatements())
        {
            try
            {
                await ExecuteAsync(connection, statement);
            }
            catch (SqliteException exception) when (
                exception.SqliteErrorCode == 1 &&
                exception.Message.Contains("no such table", StringComparison.OrdinalIgnoreCase))
            {
                // The harness only creates the table required by this test.
            }
        }

        var legacyExists = await ScalarAsync<long>(connection, """
SELECT COUNT(*) FROM sqlite_master WHERE type = 'table' AND name = 'PuupeeAppReleases';
""");
        var publisher = await ScalarAsync<string>(connection, """
SELECT "Publisher" FROM "PuupeeAppReleases" WHERE "Id" = 'release-1';
""");

        legacyExists.ShouldBe(1);
        publisher.ShouldBe("Puupee");
    }

    private static async Task ExecuteAsync(SqliteConnection connection, string sql)
    {
        await using var command = connection.CreateCommand();
        command.CommandText = sql;
        await command.ExecuteNonQueryAsync();
    }

    private static async Task<T> ScalarAsync<T>(SqliteConnection connection, string sql)
    {
        await using var command = connection.CreateCommand();
        command.CommandText = sql;
        var result = await command.ExecuteScalarAsync();
        return (T)Convert.ChangeType(result!, typeof(T));
    }
}
```

- [ ] **Step 2: Run SQLite harness tests and verify they pass**

Run:

```bash
cd api
dotnet test test/Felorx.EntityFrameworkCore.Tests/Felorx.EntityFrameworkCore.Tests.csproj --filter FullyQualifiedName~LegacyBrandSqliteMigrationTests
```

Expected:

```text
Passed!  - Failed: 0
```

- [ ] **Step 3: Commit**

```bash
git add api/test/Felorx.EntityFrameworkCore.Tests/EntityFrameworkCore/Branding/LegacyBrandSqliteMigrationTests.cs
git commit -m "test(api): 覆盖旧品牌表 SQLite 迁移"
```

---

### Task 6: Add PostgreSQL Integration Tests For Real Production Semantics

**Files:**
- Modify: `api/test/Felorx.EntityFrameworkCore.Tests/Felorx.EntityFrameworkCore.Tests.csproj`
- Create: `api/test/Felorx.EntityFrameworkCore.Tests/EntityFrameworkCore/Branding/LegacyBrandPostgreSqlMigrationTests.cs`

- [ ] **Step 1: Add test dependencies**

Modify `api/test/Felorx.EntityFrameworkCore.Tests/Felorx.EntityFrameworkCore.Tests.csproj`:

```xml
<ItemGroup>
  <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.8.0" />
  <PackageReference Include="Npgsql" Version="8.0.0" />
  <PackageReference Include="Testcontainers.PostgreSql" Version="3.9.0" />
</ItemGroup>
```

Keep the existing `Microsoft.NET.Test.Sdk` reference in this item group; do not create a duplicate package reference.

- [ ] **Step 2: Write PostgreSQL integration tests**

Create `api/test/Felorx.EntityFrameworkCore.Tests/EntityFrameworkCore/Branding/LegacyBrandPostgreSqlMigrationTests.cs`:

```csharp
using System;
using System.Threading.Tasks;
using Felorx.EntityFrameworkCore.Migrations.Branding;
using Npgsql;
using Shouldly;
using Testcontainers.PostgreSql;
using Xunit;

namespace Felorx.EntityFrameworkCore.Branding;

public sealed class LegacyBrandPostgreSqlMigrationTests : IAsyncLifetime
{
    private readonly PostgreSqlContainer _postgres = new PostgreSqlBuilder()
        .WithImage("postgres:16-alpine")
        .WithDatabase("felorx_migration_test")
        .WithUsername("postgres")
        .WithPassword("postgres")
        .Build();

    public async Task InitializeAsync()
    {
        await _postgres.StartAsync();
    }

    public async Task DisposeAsync()
    {
        await _postgres.DisposeAsync();
    }

    [Fact]
    public async Task PostgresUp_Renames_Table_Index_Constraint_And_Creates_Updatable_Legacy_View()
    {
        await using var connection = new NpgsqlConnection(_postgres.GetConnectionString());
        await connection.OpenAsync();

        await ExecuteAsync(connection, """
CREATE TABLE "PuupeeAppReleases" (
    "Id" text NOT NULL,
    "AppId" text NOT NULL,
    "Publisher" text NULL,
    "VersionCode" integer NOT NULL,
    CONSTRAINT "PK_PuupeeAppReleases" PRIMARY KEY ("Id")
);
CREATE INDEX "IX_PuupeeAppReleases_AppId" ON "PuupeeAppReleases" ("AppId");
INSERT INTO "PuupeeAppReleases" ("Id", "AppId", "Publisher", "VersionCode")
VALUES ('release-1', 'app-1', 'Puupee', 42);
""");

        await ExecuteAsync(connection, LegacyBrandMigrationSql.BuildPostgresUpSql());

        (await ExistsAsync(connection, "PuupeeAppReleases", "r")).ShouldBeFalse();
        (await ExistsAsync(connection, "FelorxAppReleases", "r")).ShouldBeTrue();
        (await ExistsAsync(connection, "PuupeeAppReleases", "v")).ShouldBeTrue();
        (await ScalarAsync<string>(connection, """SELECT "Publisher" FROM "FelorxAppReleases" WHERE "Id" = 'release-1';"""))
            .ShouldBe("Felorx");
        (await ScalarAsync<long>(connection, """
SELECT COUNT(*) FROM pg_indexes WHERE indexname = 'IX_FelorxAppReleases_AppId';
""")).ShouldBe(1);
        (await ScalarAsync<long>(connection, """
SELECT COUNT(*) FROM pg_constraint WHERE conname = 'PK_FelorxAppReleases';
""")).ShouldBe(1);

        await ExecuteAsync(connection, """
INSERT INTO "PuupeeAppReleases" ("Id", "AppId", "Publisher", "VersionCode")
VALUES ('release-2', 'app-2', 'Felorx', 43);
""");

        (await ScalarAsync<long>(connection, """SELECT COUNT(*) FROM "FelorxAppReleases";"""))
            .ShouldBe(2);
    }

    [Fact]
    public async Task PostgresUp_Is_Idempotent_After_First_Run()
    {
        await using var connection = new NpgsqlConnection(_postgres.GetConnectionString());
        await connection.OpenAsync();

        await ExecuteAsync(connection, """
CREATE TABLE "PuupeesApps" (
    "Id" text NOT NULL,
    "Name" text NOT NULL,
    CONSTRAINT "PK_PuupeesApps" PRIMARY KEY ("Id")
);
INSERT INTO "PuupeesApps" ("Id", "Name") VALUES ('app-1', 'Felorx Todo');
""");

        await ExecuteAsync(connection, LegacyBrandMigrationSql.BuildPostgresUpSql());
        await ExecuteAsync(connection, LegacyBrandMigrationSql.BuildPostgresUpSql());

        (await ExistsAsync(connection, "FelorxApps", "r")).ShouldBeTrue();
        (await ExistsAsync(connection, "PuupeesApps", "v")).ShouldBeTrue();
        (await ScalarAsync<string>(connection, """SELECT "Name" FROM "FelorxApps" WHERE "Id" = 'app-1';"""))
            .ShouldBe("Felorx Todo");
    }

    [Fact]
    public async Task PostgresUp_Fails_Fast_When_Legacy_And_New_Tables_Both_Exist()
    {
        await using var connection = new NpgsqlConnection(_postgres.GetConnectionString());
        await connection.OpenAsync();

        await ExecuteAsync(connection, """
CREATE TABLE "PuupeeApps" ("Id" text NOT NULL PRIMARY KEY);
CREATE TABLE "FelorxApps" ("Id" text NOT NULL PRIMARY KEY);
""");

        var exception = await Should.ThrowAsync<PostgresException>(
            () => ExecuteAsync(connection, LegacyBrandMigrationSql.BuildPostgresUpSql()));

        exception.MessageText.ShouldContain("Both legacy table");
    }

    private static async Task ExecuteAsync(NpgsqlConnection connection, string sql)
    {
        await using var command = connection.CreateCommand();
        command.CommandText = sql;
        await command.ExecuteNonQueryAsync();
    }

    private static async Task<bool> ExistsAsync(NpgsqlConnection connection, string name, string relkind)
    {
        var count = await ScalarAsync<long>(connection, """
SELECT COUNT(*)
FROM pg_class c
JOIN pg_namespace n ON n.oid = c.relnamespace
WHERE n.nspname = current_schema()
  AND c.relname = @name
  AND c.relkind = @relkind;
""", ("name", name), ("relkind", relkind));
        return count == 1;
    }

    private static async Task<T> ScalarAsync<T>(
        NpgsqlConnection connection,
        string sql,
        params (string Name, object Value)[] parameters)
    {
        await using var command = connection.CreateCommand();
        command.CommandText = sql;
        foreach (var parameter in parameters)
        {
            command.Parameters.AddWithValue(parameter.Name, parameter.Value);
        }

        var result = await command.ExecuteScalarAsync();
        return (T)Convert.ChangeType(result!, typeof(T));
    }
}
```

- [ ] **Step 3: Run PostgreSQL integration tests and verify they pass**

Run:

```bash
cd api
dotnet test test/Felorx.EntityFrameworkCore.Tests/Felorx.EntityFrameworkCore.Tests.csproj --filter FullyQualifiedName~LegacyBrandPostgreSqlMigrationTests
```

Expected:

```text
Passed!  - Failed: 0
```

If Docker is not available, do not mark this task complete. Run it in CI or on a machine with Docker and record the passing output in the PR.

- [ ] **Step 4: Commit**

```bash
git add api/test/Felorx.EntityFrameworkCore.Tests/Felorx.EntityFrameworkCore.Tests.csproj \
  api/test/Felorx.EntityFrameworkCore.Tests/EntityFrameworkCore/Branding/LegacyBrandPostgreSqlMigrationTests.cs
git commit -m "test(api): 覆盖旧品牌表 PostgreSQL 无缝迁移"
```

---

### Task 7: Add Production Runbook

**Files:**
- Create: `api/docs/server-database-felorx-migration-runbook.md`

- [ ] **Step 1: Create the runbook**

Create `api/docs/server-database-felorx-migration-runbook.md`:

```markdown
# Server Database Felorx Migration Runbook

## Scope

This runbook migrates server PostgreSQL database artifacts from legacy `Puupee*` / `Puupees*` names to `Felorx*` names. It does not migrate local Drift tables or sync protocol JSON keys.

## Preflight

1. Confirm the target service build contains migration `20260627090000_RenameLegacyDatabaseArtifactsToFelorx`.
2. Stop write traffic or put the API in maintenance mode.
3. Create a PostgreSQL backup:

```bash
pg_dump "$CONNECTION_STRING" --format=custom --file=felorx-pre-brand-migration.dump
```

4. Check for duplicate legacy/new tables:

```sql
SELECT legacy.relname AS legacy_table, newer.relname AS new_table
FROM pg_class legacy
JOIN pg_namespace n ON n.oid = legacy.relnamespace
JOIN pg_class newer ON newer.relnamespace = legacy.relnamespace
WHERE n.nspname = current_schema()
  AND legacy.relkind IN ('r', 'p')
  AND newer.relkind IN ('r', 'p')
  AND (
    newer.relname = replace(legacy.relname, 'Puupees', 'Felorx')
    OR newer.relname = replace(legacy.relname, 'Puupee', 'Felorx')
  )
  AND legacy.relname <> newer.relname;
```

The query must return zero rows before migration.

## Migration

Run:

```bash
cd api
dotnet run --project src/Felorx.DbMigrator/Felorx.DbMigrator.csproj --environment Production
```

## Postflight Verification

Check new table exists:

```sql
SELECT COUNT(*) FROM information_schema.tables
WHERE table_schema = current_schema()
  AND table_name = 'FelorxAppReleases';
```

Expected: `1`.

Check legacy table is now a view:

```sql
SELECT relkind FROM pg_class WHERE relname = 'PuupeeAppReleases';
```

Expected: `v`.

Check publisher data:

```sql
SELECT COUNT(*) FROM "FelorxAppReleases" WHERE "Publisher" = 'Puupee';
```

Expected: `0`.

Check API smoke endpoints:

```bash
curl -f https://api.felorx.com/health-status
curl -f https://api.felorx.com/swagger/v1/swagger.json
```

## Rollback

If migration fails before completion, restore the backup:

```bash
dropdb "$DATABASE_NAME"
createdb "$DATABASE_NAME"
pg_restore --dbname="$CONNECTION_STRING" --clean --if-exists felorx-pre-brand-migration.dump
```

If the migration completes but the new application must roll back to the previous release, keep the database in the migrated state and use the `Puupee*` compatibility views for the rollback window. If a full physical rollback is required, run EF `database update` to the migration before `20260627090000_RenameLegacyDatabaseArtifactsToFelorx`, then restore from backup if the old service requires physical `Puupee*` tables for migration tooling.

## Compatibility Window Exit

After one stable release, remove `Puupee*` and `Puupees*` compatibility views in a separate cleanup migration. Do not remove them in the same release as the table rename.
```

- [ ] **Step 2: Run Markdown grep checks**

Run:

```bash
rg -n "Puupee\\*|Puupees\\*|Felorx\\*" api/docs/server-database-felorx-migration-runbook.md
```

Expected: the command prints the intentional runbook references.

- [ ] **Step 3: Commit**

```bash
git add api/docs/server-database-felorx-migration-runbook.md
git commit -m "docs(api): 增加服务端数据库品牌迁移手册"
```

---

### Task 8: Full Verification And Regression Sweep

**Files:**
- No file changes.

- [ ] **Step 1: Run focused EF migration tests**

Run:

```bash
cd api
dotnet test test/Felorx.EntityFrameworkCore.Tests/Felorx.EntityFrameworkCore.Tests.csproj --filter "FullyQualifiedName~Branding"
```

Expected:

```text
Passed!  - Failed: 0
```

- [ ] **Step 2: Run the full EF Core test project**

Run:

```bash
cd api
dotnet test test/Felorx.EntityFrameworkCore.Tests/Felorx.EntityFrameworkCore.Tests.csproj
```

Expected:

```text
Passed!  - Failed: 0
```

- [ ] **Step 3: Run migration list verification**

Run:

```bash
cd api
dotnet ef migrations list --project src/Felorx.EntityFrameworkCore/Felorx.EntityFrameworkCore.csproj --startup-project src/Felorx.HttpApi.Host/Felorx.HttpApi.Host.csproj
```

Expected output contains:

```text
20260626081500_RenameLegacyPublisherToFelorx
20260627090000_RenameLegacyDatabaseArtifactsToFelorx
```

- [ ] **Step 4: Run strong old-brand scan for server code**

Run:

```bash
rg -n "Puupee|Puupees|puupee|puupees" api/src api/test \
  -g '!**/bin/**' \
  -g '!**/obj/**' \
  -g '!**/wwwroot/libs/**'
```

Expected remaining matches are only:

```text
api/src/Felorx.EntityFrameworkCore/Migrations/20260626081500_RenameLegacyPublisherToFelorx.cs
api/src/Felorx.EntityFrameworkCore/Migrations/20260627090000_RenameLegacyDatabaseArtifactsToFelorx.cs
api/src/Felorx.EntityFrameworkCore/Migrations/Branding/LegacyBrandDatabaseNames.cs
api/src/Felorx.EntityFrameworkCore/Migrations/Branding/LegacyBrandMigrationSql.cs
api/test/Felorx.EntityFrameworkCore.Tests/EntityFrameworkCore/Branding/*
```

- [ ] **Step 5: Commit verification-only updates if any were needed**

If no files changed, do not commit. If a verification fix was required, commit only those files:

```bash
git status --short
git add <changed-files>
git commit -m "test(api): 完成服务端数据库品牌迁移验证"
```

---

## Deployment Sequence

1. Merge and deploy code containing all tasks above to staging.
2. Run `dotnet run --project src/Felorx.DbMigrator/Felorx.DbMigrator.csproj --environment Testing` against a staging copy of production.
3. Run staging smoke checks against `https://api.felorx.com` equivalent staging endpoints.
4. Take production backup with `pg_dump --format=custom`.
5. Put production API into maintenance mode or stop write workers.
6. Run `Felorx.DbMigrator` in production.
7. Start the new Felorx API/AuthServer release.
8. Verify table names, compatibility views, publisher data, login, app listing, subscription listing, and app release download APIs.
9. Keep compatibility views for one release.
10. Create a later cleanup migration to drop `Puupee*` / `Puupees*` views after production telemetry confirms no old service/code path reads them.

## Self-Review

- Spec coverage: The plan prioritizes server DB migration, preserves existing data, includes rollback/compatibility, and defines complete focused + integration tests.
- Placeholder scan: The plan contains no unresolved placeholder markers or unspecified "write tests" steps.
- Type consistency: Helper namespace is `Felorx.EntityFrameworkCore.Migrations.Branding`; tests use the same namespace and existing `InternalsVisibleTo("Felorx.EntityFrameworkCore.Tests")` already present in `api/src/Felorx.EntityFrameworkCore/Properties/AssemblyInfo.cs`.
- Known risk: PostgreSQL updatable compatibility views are covered by Testcontainers. SQLite is only a fast rename harness and does not prove old-service rollback behavior.
