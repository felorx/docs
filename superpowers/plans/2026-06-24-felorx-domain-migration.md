# Felorx Domain Migration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Migrate the repository's runtime domains, public URLs, CDN defaults, and related tests from `puupee.com` to `felorx.com` without changing package names, app directories, bundle ids, or core model names.

**Architecture:** This is phase 1 of the Felorx migration. It intentionally changes only URL/domain configuration and brand-facing publisher contact values; semantic names such as `Puupee`, `Puupees`, `puupee_*`, `Puupees_Website`, app ids, and content types remain unchanged in this phase.

**Tech Stack:** Next.js/Jest/TypeScript, .NET ABP appsettings and tests, Flutter/Dart env configs, Kubernetes YAML, browser extension TypeScript, shell scripts, Inno Setup.

---

## Scope Boundaries

Change these values everywhere they are runtime defaults, public URLs, deployment settings, tests, or user-facing docs:

```text
puupee.com -> felorx.com
www.puupee.com -> www.felorx.com
dev.puupee.com -> dev.felorx.com
api.puupee.com -> api.felorx.com
dev.api.puupee.com -> dev.api.felorx.com
auth.puupee.com -> auth.felorx.com
dev.auth.puupee.com -> dev.auth.felorx.com
feedback.puupee.com -> feedback.felorx.com
sync.puupee.com -> sync.felorx.com
sentry.puupee.com -> sentry.felorx.com
dev.sentry.puupee.com -> dev.sentry.felorx.com
app-assets.puupee.com -> app-assets.felorx.com
avatar.puupee.com -> avatar.felorx.com
user-files.puupee.com -> user-files.felorx.com
support@puupee.com -> support@felorx.com
puupee@puupee.com -> felorx@felorx.com
```

Keep these names unchanged in this phase:

```text
Puupee
Puupees
Puupees_Website
Puupee_Sync_Node
puupee
puupee_*
package:puupee_*
apps/todos
com.puupee
application/vnd.puupee.*
```

Keep explicit legacy compatibility values when the surrounding code is handling old data. The current known allowed legacy value is:

```text
website/src/lib/api/apps.ts: LEGACY_BAD_ASSET_HOSTS contains cdn.puupee.com
```

## Domain File Inventory

Before editing, regenerate the working inventory:

```bash
rg -l --hidden \
  --glob '!**/.git/**' \
  --glob '!**/build/**' \
  --glob '!**/.dart_tool/**' \
  --glob '!**/node_modules/**' \
  'puupee\.com|auth\.puupee\.com|api\.puupee\.com|cdn\.puupee\.com|support@puupee\.com' \
  website api apps packages extends k8s templates scripts .puupee.yaml docker-compose.yaml codemagic.yaml pubspec.yaml .vscode
```

Expected: command exits 0 and prints domain-bearing files. Do not include `docs/superpowers/specs/2026-06-24-felorx-brand-migration-design.md` in phase 1 edits.

### Task 1: Website Runtime URLs And Tests

**Files:**
- Modify: `website/src/lib/config/api.ts`
- Modify: `website/src/lib/config/__tests__/api.test.ts`
- Modify: `website/src/lib/auth/__tests__/browser-oauth.test.ts`
- Modify: `website/src/lib/auth/__tests__/urls.test.ts`
- Modify: `website/src/lib/api/apps.ts`
- Modify: `website/.env.example`
- Modify: `website/nginx.conf`
- Modify: `website/vercel.json`
- Modify: `website/public/robots.txt`
- Modify: `website/src/app/__tests__/page.test.tsx`
- Modify: `website/src/components/apps/__tests__/AlipayCheckout.test.tsx`
- Modify: `website/src/components/apps/__tests__/PayPalCheckout.test.tsx`
- Modify: `website/docs/deployment.md`
- Modify: `website/docs/quick-start-deployment.md`
- Modify: `website/docs/static-assets.md`
- Modify: `website/README.md`

- [ ] **Step 1: Update website domain defaults**

Run this scoped replacement:

```bash
perl -0pi -e '
s#https://dev\.api\.puupee\.com#https://dev.api.felorx.com#g;
s#https://api\.puupee\.com#https://api.felorx.com#g;
s#https://dev\.auth\.puupee\.com#https://dev.auth.felorx.com#g;
s#https://auth\.puupee\.com#https://auth.felorx.com#g;
s#https://dev\.puupee\.com#https://dev.felorx.com#g;
s#https://www\.puupee\.com#https://www.felorx.com#g;
s#https://puupee\.com#https://felorx.com#g;
s#app-assets\.puupee\.com#app-assets.felorx.com#g;
s#avatar\.puupee\.com#avatar.felorx.com#g;
s#user-files\.puupee\.com#user-files.felorx.com#g;
s#\*\.puupee\.com#*.felorx.com#g;
s#www\.puupee\.com#www.felorx.com#g;
s#dev\.puupee\.com#dev.felorx.com#g;
s#puupee\.com#felorx.com#g;
' \
website/src/lib/config/api.ts \
website/src/lib/config/__tests__/api.test.ts \
website/src/lib/auth/__tests__/browser-oauth.test.ts \
website/src/lib/auth/__tests__/urls.test.ts \
website/.env.example \
website/nginx.conf \
website/vercel.json \
website/public/robots.txt \
website/src/app/__tests__/page.test.tsx \
website/src/components/apps/__tests__/AlipayCheckout.test.tsx \
website/src/components/apps/__tests__/PayPalCheckout.test.tsx \
website/docs/deployment.md \
website/docs/quick-start-deployment.md \
website/docs/static-assets.md \
website/README.md
```

Expected: the listed files no longer contain `puupee.com`.

- [ ] **Step 2: Update AppAssets default without removing legacy CDN guard**

Edit `website/src/lib/api/apps.ts` so the default host changes but legacy guard remains:

```ts
const DEFAULT_APP_ASSETS_CDN_DOMAIN = 'app-assets.felorx.com';
const LEGACY_BAD_ASSET_HOSTS = new Set(['cdn.puupee.com']);
```

Expected: `DEFAULT_APP_ASSETS_CDN_DOMAIN` uses Felorx; `LEGACY_BAD_ASSET_HOSTS` still contains `cdn.puupee.com`.

- [ ] **Step 3: Verify website unit tests**

Run:

```bash
cd website && npm test -- --runInBand
```

Expected: Jest exits 0.

- [ ] **Step 4: Verify website TypeScript**

Run:

```bash
cd website && npm run type-check
```

Expected: TypeScript exits 0.

- [ ] **Step 5: Commit website changes**

Run:

```bash
git add website/src/lib/config/api.ts website/src/lib/config/__tests__/api.test.ts website/src/lib/auth/__tests__/browser-oauth.test.ts website/src/lib/auth/__tests__/urls.test.ts website/src/lib/api/apps.ts website/.env.example website/nginx.conf website/vercel.json website/public/robots.txt website/src/app/__tests__/page.test.tsx website/src/components/apps/__tests__/AlipayCheckout.test.tsx website/src/components/apps/__tests__/PayPalCheckout.test.tsx website/docs/deployment.md website/docs/quick-start-deployment.md website/docs/static-assets.md website/README.md
git commit -m "refactor(website): 迁移 Felorx 域名配置"
```

Expected: commit succeeds and contains only website phase 1 changes.

### Task 2: API Appsettings And API Tests

**Files:**
- Modify: `api/src/Puupees.AuthServer/appsettings.json`
- Modify: `api/src/Puupees.AuthServer/appsettings.Development.json`
- Modify: `api/src/Puupees.AuthServer/appsettings.Production.json`
- Modify: `api/src/Puupees.AuthServer/appsettings.Testing.json`
- Modify: `api/src/Puupees.AuthServer/ApiKeyLogin/ApiKeyTokenGrant.cs`
- Modify: `api/src/Puupees.AuthServer/PhoneNumberLogin/SmsTokenGrant.cs`
- Modify: `api/src/Puupees.HttpApi.Host/appsettings.json`
- Modify: `api/src/Puupees.HttpApi.Host/appsettings.Development.json`
- Modify: `api/src/Puupees.HttpApi.Host/appsettings.Production.json`
- Modify: `api/src/Puupees.HttpApi.Host/appsettings.Testing.json`
- Modify: `api/src/Puupees.DbMigrator/appsettings.json`
- Modify: `api/src/Puupees.DbMigrator/appsettings.Production.json`
- Modify: `api/src/Puupees.DbMigrator/appsettings.Testing.json`
- Modify: `api/test/Puupees.EntityFrameworkCore.Tests/EntityFrameworkCore/PuupeesEntityFrameworkCoreTestModule.cs`
- Modify: `api/test/Puupees.EntityFrameworkCore.Tests/Subscriptions/SubscriptionAlipayAppServiceTests.cs`

- [ ] **Step 1: Update API domains**

Run this scoped replacement:

```bash
perl -0pi -e '
s#https://test\.api\.puupee\.com#https://test.api.felorx.com#g;
s#https://test\.puupee\.com#https://test.felorx.com#g;
s#https://dev\.api\.puupee\.com#https://dev.api.felorx.com#g;
s#https://api\.puupee\.com#https://api.felorx.com#g;
s#https://dev\.auth\.puupee\.com#https://dev.auth.felorx.com#g;
s#https://auth\.puupee\.com#https://auth.felorx.com#g;
s#https://dev\.puupee\.com#https://dev.felorx.com#g;
s#https://www\.puupee\.com#https://www.felorx.com#g;
s#https://puupee\.com#https://felorx.com#g;
s#user-files\.puupee\.com#user-files.felorx.com#g;
s#app-assets\.puupee\.com#app-assets.felorx.com#g;
s#avatar\.puupee\.com#avatar.felorx.com#g;
s#\*\.puupee\.com#*.felorx.com#g;
s#www\.puupee\.com#www.felorx.com#g;
s#dev\.puupee\.com#dev.felorx.com#g;
s#test\.puupee\.com#test.felorx.com#g;
s#puupee@felorx\.com#felorx@felorx.com#g;
s#@puupee\.com#@felorx.com#g;
s#puupee\.com#felorx.com#g;
' \
api/src/Puupees.AuthServer/appsettings.json \
api/src/Puupees.AuthServer/appsettings.Development.json \
api/src/Puupees.AuthServer/appsettings.Production.json \
api/src/Puupees.AuthServer/appsettings.Testing.json \
api/src/Puupees.AuthServer/ApiKeyLogin/ApiKeyTokenGrant.cs \
api/src/Puupees.AuthServer/PhoneNumberLogin/SmsTokenGrant.cs \
api/src/Puupees.HttpApi.Host/appsettings.json \
api/src/Puupees.HttpApi.Host/appsettings.Development.json \
api/src/Puupees.HttpApi.Host/appsettings.Production.json \
api/src/Puupees.HttpApi.Host/appsettings.Testing.json \
api/src/Puupees.DbMigrator/appsettings.json \
api/src/Puupees.DbMigrator/appsettings.Production.json \
api/src/Puupees.DbMigrator/appsettings.Testing.json \
api/test/Puupees.EntityFrameworkCore.Tests/EntityFrameworkCore/PuupeesEntityFrameworkCoreTestModule.cs \
api/test/Puupees.EntityFrameworkCore.Tests/Subscriptions/SubscriptionAlipayAppServiceTests.cs
```

Expected: API URL defaults and test expectations use Felorx domains. Database names, namespaces, project names, OAuth client ids, and error codes remain unchanged.

- [ ] **Step 2: Verify API domain residuals**

Run:

```bash
rg -n 'puupee\.com' api/src api/test
```

Expected: no output. If output appears, inspect each line and only keep a value if it is explicitly testing legacy Puupee input.

- [ ] **Step 3: Build API**

Run:

```bash
cd api && dotnet build
```

Expected: build exits 0.

- [ ] **Step 4: Run targeted API tests**

Run:

```bash
cd api && dotnet test test/Puupees.EntityFrameworkCore.Tests/Puupees.EntityFrameworkCore.Tests.csproj
```

Expected: test project exits 0.

- [ ] **Step 5: Commit API changes**

Run:

```bash
git add api/src/Puupees.AuthServer/appsettings.json api/src/Puupees.AuthServer/appsettings.Development.json api/src/Puupees.AuthServer/appsettings.Production.json api/src/Puupees.AuthServer/appsettings.Testing.json api/src/Puupees.AuthServer/ApiKeyLogin/ApiKeyTokenGrant.cs api/src/Puupees.AuthServer/PhoneNumberLogin/SmsTokenGrant.cs api/src/Puupees.HttpApi.Host/appsettings.json api/src/Puupees.HttpApi.Host/appsettings.Development.json api/src/Puupees.HttpApi.Host/appsettings.Production.json api/src/Puupees.HttpApi.Host/appsettings.Testing.json api/src/Puupees.DbMigrator/appsettings.json api/src/Puupees.DbMigrator/appsettings.Production.json api/src/Puupees.DbMigrator/appsettings.Testing.json api/test/Puupees.EntityFrameworkCore.Tests/EntityFrameworkCore/PuupeesEntityFrameworkCoreTestModule.cs api/test/Puupees.EntityFrameworkCore.Tests/Subscriptions/SubscriptionAlipayAppServiceTests.cs
git commit -m "refactor(api): 迁移 Felorx 服务域名配置"
```

Expected: commit succeeds and contains only API phase 1 changes.

### Task 3: Flutter App Env Defaults And Shared Env

**Files:**
- Modify: `packages/common/puupee_shared/lib/env.dart`
- Modify: `packages/common/puupee_shared/test/env_legal_urls_test.dart`
- Modify: `packages/common/puupee_shared/test/features/login/qr_scan_page_test.dart`
- Modify: `packages/common/puupee_shared/lib/models/sync_node_panel_config.dart`
- Modify: `packages/common/puupee_shared/lib/features/settings/sync/storage_config_edit_page.dart`
- Modify: `packages/common/puupee_shared/lib/providers/cdn_domain.dart`
- Modify: `packages/common/puupee_shared/lib/features/settings/about/provider.dart`
- Modify: `packages/common/puupee_shared/lib/components/file_upload/single_file_upload_widget.dart`
- Modify: each `apps/*/lib/env.dart` that contains `puupee.com`
- Modify: `apps/powery/lib/pages/powery_settings_widgets.dart`
- Modify: `apps/developer/lib/apps/list_view_item.dart`
- Modify: `apps/developer/lib/apps/details/app_locale_assets.dart`
- Modify: `apps/devpanel/lib/services/devpanel_project_service.dart`
- Modify: `apps/ops/lib/providers/connect_provider.dart`
- Modify: `apps/ops/test/providers/connect_provider_test.dart`
- Modify: `apps/sync_node/lib/sync_node_arg_parser.dart`

- [ ] **Step 1: Update shared env and app env defaults**

Run this scoped replacement:

```bash
perl -0pi -e '
s#https://dev\.api\.puupee\.com#https://dev.api.felorx.com#g;
s#https://api\.puupee\.com#https://api.felorx.com#g;
s#https://dev\.auth\.puupee\.com#https://dev.auth.felorx.com#g;
s#https://auth\.puupee\.com#https://auth.felorx.com#g;
s#https://feedback\.puupee\.com#https://feedback.felorx.com#g;
s#https://dev\.puupee\.com#https://dev.felorx.com#g;
s#https://puupee\.com#https://felorx.com#g;
s#sentry\.puupee\.com#sentry.felorx.com#g;
s#app-assets\.puupee\.com#app-assets.felorx.com#g;
s#avatar\.puupee\.com#avatar.felorx.com#g;
s#user-files\.puupee\.com#user-files.felorx.com#g;
s#puupee\.com#felorx.com#g;
' \
packages/common/puupee_shared/lib/env.dart \
packages/common/puupee_shared/test/env_legal_urls_test.dart \
packages/common/puupee_shared/test/features/login/qr_scan_page_test.dart \
packages/common/puupee_shared/lib/models/sync_node_panel_config.dart \
packages/common/puupee_shared/lib/features/settings/sync/storage_config_edit_page.dart \
packages/common/puupee_shared/lib/providers/cdn_domain.dart \
packages/common/puupee_shared/lib/features/settings/about/provider.dart \
packages/common/puupee_shared/lib/components/file_upload/single_file_upload_widget.dart \
apps/*/lib/env.dart \
apps/powery/lib/pages/powery_settings_widgets.dart \
apps/developer/lib/apps/list_view_item.dart \
apps/developer/lib/apps/details/app_locale_assets.dart \
apps/devpanel/lib/services/devpanel_project_service.dart \
apps/ops/lib/providers/connect_provider.dart \
apps/ops/test/providers/connect_provider_test.dart \
apps/sync_node/lib/sync_node_arg_parser.dart
```

Expected: Flutter runtime env defaults use Felorx domains. App ids and app titles remain unchanged in this phase.

- [ ] **Step 2: Format touched Dart files**

Run:

```bash
dart format packages/common/puupee_shared/lib/env.dart packages/common/puupee_shared/test/env_legal_urls_test.dart packages/common/puupee_shared/test/features/login/qr_scan_page_test.dart packages/common/puupee_shared/lib/models/sync_node_panel_config.dart packages/common/puupee_shared/lib/features/settings/sync/storage_config_edit_page.dart packages/common/puupee_shared/lib/providers/cdn_domain.dart packages/common/puupee_shared/lib/features/settings/about/provider.dart packages/common/puupee_shared/lib/components/file_upload/single_file_upload_widget.dart apps/*/lib/env.dart apps/powery/lib/pages/powery_settings_widgets.dart apps/developer/lib/apps/list_view_item.dart apps/developer/lib/apps/details/app_locale_assets.dart apps/devpanel/lib/services/devpanel_project_service.dart apps/ops/lib/providers/connect_provider.dart apps/ops/test/providers/connect_provider_test.dart apps/sync_node/lib/sync_node_arg_parser.dart
```

Expected: formatter exits 0.

- [ ] **Step 3: Verify shared package tests**

Run:

```bash
cd packages/common/puupee_shared && flutter test test/env_legal_urls_test.dart test/features/login/qr_scan_page_test.dart
```

Expected: selected tests exit 0.

- [ ] **Step 4: Verify representative app analysis**

Run:

```bash
cd apps/todos && dart analyze
```

Expected: analyzer exits 0 or only reports pre-existing warnings unrelated to domain strings. If it reports errors from domain changes, fix them before committing.

- [ ] **Step 5: Commit Flutter env changes**

Run:

```bash
git add packages/common/puupee_shared/lib/env.dart packages/common/puupee_shared/test/env_legal_urls_test.dart packages/common/puupee_shared/test/features/login/qr_scan_page_test.dart packages/common/puupee_shared/lib/models/sync_node_panel_config.dart packages/common/puupee_shared/lib/features/settings/sync/storage_config_edit_page.dart packages/common/puupee_shared/lib/providers/cdn_domain.dart packages/common/puupee_shared/lib/features/settings/about/provider.dart packages/common/puupee_shared/lib/components/file_upload/single_file_upload_widget.dart apps/*/lib/env.dart apps/powery/lib/pages/powery_settings_widgets.dart apps/developer/lib/apps/list_view_item.dart apps/developer/lib/apps/details/app_locale_assets.dart apps/devpanel/lib/services/devpanel_project_service.dart apps/ops/lib/providers/connect_provider.dart apps/ops/test/providers/connect_provider_test.dart apps/sync_node/lib/sync_node_arg_parser.dart
git commit -m "refactor(puupee_shared): 迁移 Felorx 客户端域名配置"
```

Expected: commit succeeds and contains only Flutter env/default URL changes.

### Task 4: Dart Tooling, Sync Packages, And Builder Defaults

**Files:**
- Modify: `packages/sync/puupee_connect/lib/src/host/connect_host_config.dart`
- Modify: `packages/sync/puupee_connect/test/host/connect_host_config_test.dart`
- Modify: `packages/sync/puupee_connect/test/host/connect_host_cli_defaults_test.dart`
- Modify: `packages/sync/puupee_connect/test/host/connect_host_cli_output_test.dart`
- Modify: `packages/sync/puupee_sync/lib/src/sync_server_config.dart`
- Modify: `packages/sync/puupee_sync/lib/src/sync_server.dart`
- Modify: `packages/sync/puupee_sync/bin/mcp_config_generator.dart`
- Modify: `packages/sync/puupee_sync/test/core_models_test.dart`
- Modify: `packages/sync/puupee_sync/pubspec.yaml`
- Modify: `packages/sync/puupee_sync/README.md`
- Modify: `packages/sync/puupee_sync/docs/mcp_installation.md`
- Modify: `packages/sync/puupee_sync/docs/mcp_authentication.md`
- Modify: `packages/sync/puupee_sync/docs/mcp_server.md`
- Modify: `packages/sync/puupee_sync/docs/mcp_web_config_generator.html`
- Modify: `packages/common/puupee_shared/pubspec.yaml`
- Modify: `packages/common/puupee_upgrader/pubspec.yaml`
- Modify: `packages/tools/puupee_cli/lib/src/config/puupee_config.dart`
- Modify: `packages/tools/puupee_cli/test/config_test.dart`
- Modify: `packages/tools/puupee_cli/example/api_key_login_example.sh`
- Modify: `packages/tools/puupee_cli/lib/src/commands/store_app_asset_command.dart`
- Modify: `packages/tools/puupee_builder_cli/lib/src/packaging_config_generator.dart`
- Modify: `packages/tools/puupee_icons/pubspec.yaml`
- Modify: `packages/api/puupee_sdk_generator/lib/src/swagger_downloader.dart`
- Modify: `packages/api/puupee_sdk_generator/bin/puupee_sdk_generator.dart`
- Modify: `packages/api/puupee_sdk_generator/README.md`
- Modify: `packages/api/puupee_sdk_generator/swagger.json`
- Modify: `packages/api/puupee_api_client/README.md`
- Modify: `packages/api/puupee-api-go/README.md`
- Modify: `packages/api/puupee-api-go/api/openapi.yaml`

- [ ] **Step 1: Update tool and package domain defaults**

Run this scoped replacement:

```bash
perl -0pi -e '
s#https://dev\.api\.puupee\.com#https://dev.api.felorx.com#g;
s#https://api\.puupee\.com#https://api.felorx.com#g;
s#https://dev\.auth\.puupee\.com#https://dev.auth.felorx.com#g;
s#https://auth\.puupee\.com#https://auth.felorx.com#g;
s#https://sync\.puupee\.com#https://sync.felorx.com#g;
s#wss://dev\.api\.puupee\.com#wss://dev.api.felorx.com#g;
s#wss://api\.puupee\.com#wss://api.felorx.com#g;
s#https://dev\.puupee\.com#https://dev.felorx.com#g;
s#https://puupee\.com#https://felorx.com#g;
s#app-assets\.puupee\.com#app-assets.felorx.com#g;
s#support@puupee\.com#support@felorx.com#g;
s#puupee\.com#felorx.com#g;
' \
packages/sync/puupee_connect/lib/src/host/connect_host_config.dart \
packages/sync/puupee_connect/test/host/connect_host_config_test.dart \
packages/sync/puupee_connect/test/host/connect_host_cli_defaults_test.dart \
packages/sync/puupee_connect/test/host/connect_host_cli_output_test.dart \
packages/sync/puupee_sync/lib/src/sync_server_config.dart \
packages/sync/puupee_sync/lib/src/sync_server.dart \
packages/sync/puupee_sync/bin/mcp_config_generator.dart \
packages/sync/puupee_sync/test/core_models_test.dart \
packages/sync/puupee_sync/pubspec.yaml \
packages/sync/puupee_sync/README.md \
packages/sync/puupee_sync/docs/mcp_installation.md \
packages/sync/puupee_sync/docs/mcp_authentication.md \
packages/sync/puupee_sync/docs/mcp_server.md \
packages/sync/puupee_sync/docs/mcp_web_config_generator.html \
packages/common/puupee_shared/pubspec.yaml \
packages/common/puupee_upgrader/pubspec.yaml \
packages/tools/puupee_cli/lib/src/config/puupee_config.dart \
packages/tools/puupee_cli/test/config_test.dart \
packages/tools/puupee_cli/example/api_key_login_example.sh \
packages/tools/puupee_cli/lib/src/commands/store_app_asset_command.dart \
packages/tools/puupee_builder_cli/lib/src/packaging_config_generator.dart \
packages/tools/puupee_icons/pubspec.yaml \
packages/api/puupee_sdk_generator/lib/src/swagger_downloader.dart \
packages/api/puupee_sdk_generator/bin/puupee_sdk_generator.dart \
packages/api/puupee_sdk_generator/README.md \
packages/api/puupee_sdk_generator/swagger.json \
packages/api/puupee_api_client/README.md \
packages/api/puupee-api-go/README.md \
packages/api/puupee-api-go/api/openapi.yaml
```

Expected: runtime defaults, pubspec homepage values, generated docs, and test expectations use Felorx domains. Package names remain unchanged.

- [ ] **Step 2: Format touched Dart tooling files**

Run:

```bash
dart format packages/sync/puupee_connect/lib/src/host/connect_host_config.dart packages/sync/puupee_connect/test/host/connect_host_config_test.dart packages/sync/puupee_connect/test/host/connect_host_cli_defaults_test.dart packages/sync/puupee_connect/test/host/connect_host_cli_output_test.dart packages/sync/puupee_sync/lib/src/sync_server_config.dart packages/sync/puupee_sync/lib/src/sync_server.dart packages/sync/puupee_sync/bin/mcp_config_generator.dart packages/sync/puupee_sync/test/core_models_test.dart packages/tools/puupee_cli/lib/src/config/puupee_config.dart packages/tools/puupee_cli/test/config_test.dart packages/tools/puupee_builder_cli/lib/src/packaging_config_generator.dart packages/api/puupee_sdk_generator/lib/src/swagger_downloader.dart packages/api/puupee_sdk_generator/bin/puupee_sdk_generator.dart
```

Expected: formatter exits 0.

- [ ] **Step 3: Run targeted Dart package tests**

Run:

```bash
cd packages/sync/puupee_connect && dart test test/host/connect_host_config_test.dart test/host/connect_host_cli_defaults_test.dart test/host/connect_host_cli_output_test.dart
```

Expected: selected connect tests exit 0.

Run:

```bash
cd packages/tools/puupee_cli && dart test test/config_test.dart
```

Expected: CLI config tests exit 0.

- [ ] **Step 4: Commit tooling changes**

Run:

```bash
git add packages/sync/puupee_connect/lib/src/host/connect_host_config.dart packages/sync/puupee_connect/test/host/connect_host_config_test.dart packages/sync/puupee_connect/test/host/connect_host_cli_defaults_test.dart packages/sync/puupee_connect/test/host/connect_host_cli_output_test.dart packages/sync/puupee_sync/lib/src/sync_server_config.dart packages/sync/puupee_sync/lib/src/sync_server.dart packages/sync/puupee_sync/bin/mcp_config_generator.dart packages/sync/puupee_sync/test/core_models_test.dart packages/sync/puupee_sync/pubspec.yaml packages/sync/puupee_sync/README.md packages/sync/puupee_sync/docs/mcp_installation.md packages/sync/puupee_sync/docs/mcp_authentication.md packages/sync/puupee_sync/docs/mcp_server.md packages/sync/puupee_sync/docs/mcp_web_config_generator.html packages/common/puupee_shared/pubspec.yaml packages/common/puupee_upgrader/pubspec.yaml packages/tools/puupee_cli/lib/src/config/puupee_config.dart packages/tools/puupee_cli/test/config_test.dart packages/tools/puupee_cli/example/api_key_login_example.sh packages/tools/puupee_cli/lib/src/commands/store_app_asset_command.dart packages/tools/puupee_builder_cli/lib/src/packaging_config_generator.dart packages/tools/puupee_icons/pubspec.yaml packages/api/puupee_sdk_generator/lib/src/swagger_downloader.dart packages/api/puupee_sdk_generator/bin/puupee_sdk_generator.dart packages/api/puupee_sdk_generator/README.md packages/api/puupee_sdk_generator/swagger.json packages/api/puupee_api_client/README.md packages/api/puupee-api-go/README.md packages/api/puupee-api-go/api/openapi.yaml
git commit -m "refactor(puupee_cli): 迁移 Felorx 工具域名配置"
```

Expected: commit succeeds and contains only tooling/package domain changes.

### Task 5: Extension, K8s, Docker, VS Code, Templates, And Installers

**Files:**
- Modify: `extends/puupee-fav-extension/src/lib/auth.ts`
- Modify: `extends/puupee-fav-extension/src/lib/oauth.ts`
- Modify: `extends/puupee-fav-extension/src/options/App.tsx`
- Modify: `extends/puupee-fav-extension/manifest.json`
- Modify: `.puupee.yaml`
- Modify: `docker-compose.yaml`
- Modify: `.vscode/launch.json`
- Modify: `templates/inno_setup_template.iss`
- Modify: `scripts/create_app_configs.sh`
- Modify: `k8s/apps/**`
- Modify: each `apps/*/windows/installer.iss` that contains `puupee.com`

- [ ] **Step 1: Update extension, local config, K8s, templates, and installers**

Run this scoped replacement:

```bash
perl -0pi -e '
s#https://dev\.api\.puupee\.com#https://dev.api.felorx.com#g;
s#https://api\.puupee\.com#https://api.felorx.com#g;
s#https://dev\.auth\.puupee\.com#https://dev.auth.felorx.com#g;
s#https://auth\.puupee\.com#https://auth.felorx.com#g;
s#https://feedback\.puupee\.com#https://feedback.felorx.com#g;
s#https://dev\.sentry\.puupee\.com#https://dev.sentry.felorx.com#g;
s#https://sentry\.puupee\.com#https://sentry.felorx.com#g;
s#wss://dev\.api\.puupee\.com#wss://dev.api.felorx.com#g;
s#wss://api\.puupee\.com#wss://api.felorx.com#g;
s#https://dev\.puupee\.com#https://dev.felorx.com#g;
s#https://www\.puupee\.com#https://www.felorx.com#g;
s#https://puupee\.com#https://felorx.com#g;
s#dev\.api\.puupee\.com#dev.api.felorx.com#g;
s#api\.puupee\.com#api.felorx.com#g;
s#dev\.auth\.puupee\.com#dev.auth.felorx.com#g;
s#auth\.puupee\.com#auth.felorx.com#g;
s#dev\.sentry\.puupee\.com#dev.sentry.felorx.com#g;
s#sentry\.puupee\.com#sentry.felorx.com#g;
s#stock\.puupee\.com#stock.felorx.com#g;
s#livememo\.puupee\.com#livememo.felorx.com#g;
s#\*\.puupee\.com#*.felorx.com#g;
s#puupee\.com#felorx.com#g;
' \
extends/puupee-fav-extension/src/lib/auth.ts \
extends/puupee-fav-extension/src/lib/oauth.ts \
extends/puupee-fav-extension/src/options/App.tsx \
extends/puupee-fav-extension/manifest.json \
.puupee.yaml \
docker-compose.yaml \
.vscode/launch.json \
templates/inno_setup_template.iss \
scripts/create_app_configs.sh \
k8s/apps/cash-mm/base/ingress.yaml \
k8s/apps/cash-asmr-green/base/ingress.yaml \
k8s/apps/cash-asmr-blue/base/ingress.yaml \
k8s/apps/cash-lesson/base/ingress.yaml \
k8s/apps/livememo/base/ingress.yaml \
k8s/apps/livememo/overlays/prod/kustomization.yaml \
k8s/apps/puupee-sentry/base/ingress.yaml \
k8s/apps/puupee-sentry/overlays/prod/kustomization.yaml \
k8s/apps/puupee/base/sync-server-development.yaml \
k8s/apps/puupee/base/ingress.yaml \
k8s/apps/puupee/overlays/prod/kustomization.yaml \
k8s/apps/stock/base/ingress.yaml \
k8s/apps/stock/overlays/prod/kustomization.yaml \
apps/*/windows/installer.iss
```

Expected: listed config/deployment files use Felorx domains. App names, app ids, K8s app directory names, and image names remain unchanged.

- [ ] **Step 2: Verify JSON/YAML syntax for edited config files**

Run:

```bash
python3 -m json.tool .vscode/launch.json >/tmp/felorx-launch-json-check.txt
python3 -m json.tool extends/puupee-fav-extension/manifest.json >/tmp/felorx-extension-manifest-check.txt
```

Expected: both commands exit 0.

Run:

```bash
rg -n 'puupee\.com' .puupee.yaml docker-compose.yaml .vscode/launch.json templates/inno_setup_template.iss scripts/create_app_configs.sh extends k8s apps/*/windows/installer.iss
```

Expected: no output.

- [ ] **Step 3: Commit config/deployment changes**

Run:

```bash
git add extends/puupee-fav-extension/src/lib/auth.ts extends/puupee-fav-extension/src/lib/oauth.ts extends/puupee-fav-extension/src/options/App.tsx extends/puupee-fav-extension/manifest.json .puupee.yaml docker-compose.yaml .vscode/launch.json templates/inno_setup_template.iss scripts/create_app_configs.sh k8s/apps/cash-mm/base/ingress.yaml k8s/apps/cash-asmr-green/base/ingress.yaml k8s/apps/cash-asmr-blue/base/ingress.yaml k8s/apps/cash-lesson/base/ingress.yaml k8s/apps/livememo/base/ingress.yaml k8s/apps/livememo/overlays/prod/kustomization.yaml k8s/apps/puupee-sentry/base/ingress.yaml k8s/apps/puupee-sentry/overlays/prod/kustomization.yaml k8s/apps/puupee/base/sync-server-development.yaml k8s/apps/puupee/base/ingress.yaml k8s/apps/puupee/overlays/prod/kustomization.yaml k8s/apps/stock/base/ingress.yaml k8s/apps/stock/overlays/prod/kustomization.yaml apps/*/windows/installer.iss
git commit -m "refactor(k8s): 迁移 Felorx 部署域名配置"
```

Expected: commit succeeds and contains only extension/config/deployment/installer domain changes.

### Task 6: Repository-Wide Residual Audit

**Files:**
- Inspect all files reported by the residual commands.
- Modify only runtime/config/test/doc files that should have been covered by Tasks 1-5.
- Do not modify historical design files under `docs/superpowers/specs`, `docs/superpowers/plans`, `.kiro/specs`, or `dev/spec` unless a line is actively used by a build, test, or deployment command.

- [ ] **Step 1: Audit remaining `puupee.com` in runtime paths**

Run:

```bash
rg -n --hidden \
  --glob '!**/.git/**' \
  --glob '!**/build/**' \
  --glob '!**/.dart_tool/**' \
  --glob '!**/node_modules/**' \
  'puupee\.com' \
  website api apps packages extends k8s templates scripts .puupee.yaml docker-compose.yaml codemagic.yaml pubspec.yaml .vscode
```

Expected: only explicitly allowed legacy compatibility lines remain. The known allowed line is `website/src/lib/api/apps.ts` containing `cdn.puupee.com`.

- [ ] **Step 2: Audit accidental semantic renames**

Run:

```bash
git diff --name-only HEAD~5..HEAD | xargs rg -n 'com\.felorx|application/vnd\.felorx|package:felorx_|FelorxContentTypes|class Felorx|apps/todo|felorx_todo' || true
```

Expected: no output. Phase 1 must not introduce bundle id, package, app directory, or model-name migrations.

- [ ] **Step 3: Check working tree only contains expected changes**

Run:

```bash
git status --short
```

Expected: only pre-existing unrelated `vela` remains modified, or the working tree is clean if that file was already handled outside this work. Do not stage or revert `vela`.

- [ ] **Step 4: Final phase 1 verification summary**

Run:

```bash
git log --oneline -5
```

Expected: the latest phase 1 commits correspond to website, API, Flutter/shared env, tooling, and config/deployment domain migrations.

## Self-Review Checklist

- Every task keeps semantic names unchanged.
- Every test expectation changed from Puupee domain to Felorx domain is covered by a targeted test command.
- Legacy `cdn.puupee.com` remains only as an old-data guard in `website/src/lib/api/apps.ts`.
- No step stages or reverts unrelated `vela` changes.
- Phase 1 ends with runtime/config/test files on Felorx domains and no bundle id/package/model migration.
